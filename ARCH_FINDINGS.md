# batmon-ha — Architecture & Implementation Findings

A Home Assistant add-on that reads from a variety of battery BMS devices (mainly BLE, some wired-serial) and bridges their data into MQTT for Home Assistant. This document captures a top-down analysis of how the system works.

## 1. High-level architecture

```
config.yaml (HA options) ──┐
addon_main.sh ──env vars──►  main.py (asyncio)
                             ├─ bt_discovery (multi-adapter)
                             ├─ construct_bms()  ─► BtBms / VirtualGroupBms instances
                             ├─ paho MQTT client (in/out)
                             ├─ background_loop (coroutine) + background_thread (suicide watchdog)
                             └─ BmsSampler[] ── fetch_loop() ──► BMS → integrators → MQTT/sinks
```

The pipeline per BMS is: **BLE/serial fetch → BmsSample → Downsampler/Integrators → MQTT (+InfluxDB/Telemetry sinks)**, with HA Discovery messages republished periodically.

## 2. Entry point (`main.py:119-415`)

- **Modes**: `pair-only` (one-shot BLE pairing via a separate venv), `skip-discovery` (debug), normal.
- **BT prep**: optional power-cycle via `bluetoothctl`, then parallel `bt_discovery` across every `hci*` adapter found in `/sys/class/bluetooth` plus any adapters named in config (`main.py:147-149`).
- **Device construction**: each entry in `user_config['devices']` → `construct_bms()`; virtual group BMSes get their member references resolved against `bms_by_name` (`main.py:198-215`).
- **MQTT bootstrap**: env vars from `addon_main.sh` (which uses `bashio` to fetch HA Supervisor MQTT credentials) overlay user config; client connects and `loop_start()`s on a paho thread.
- **Sampler creation** (`main.py:280-292`): one `BmsSampler` per BMS, configured with publish period, sample expiry, calibration factor, sinks, optional algorithm, and optional group.
- **Concurrency**: `parallel_fetch` ⇒ one `fetch_loop` per BMS; otherwise a single serial loop shuffles tasks each tick. The outer `while not shutdown` recovers from cancelled tasks (Bleak quirk).
- **Double watchdog** (`main.py:90-116`): an async `background_loop` AND a daemon Python thread both check `mqtt_last_publish_time()`. If MQTT silence exceeds `wd_timeout` (max(5 min, 4·sample_period)), they set `shutdown` and the thread eventually calls `exit_process(True, True)` — surviving an asyncio deadlock that Bleak occasionally triggers.

## 3. Sampling core (`bmslib/sampling.py`)

`BmsSampler.__call__` is the heart of the system. On each tick:

1. **Connect/keep alive** the underlying `BtBms` (`sampling.py:214-231`); first sample also calls `fetch_device_info()`.
2. **Fetch** a `BmsSample`; reject as `SampleExpiredError` if older than `max(expire_after_seconds, MIN_VALUE_EXPIRY)` (`sampling.py:241-247`).
3. **Apply calibration** (`current_calibration_factor`) and `invert_current`.
4. **Feed integrators** in `pwmath.py` (`sampling.py:257-281`):
   - `current_integrator` → Ah (trapezoidal),
   - `power_integrator` → kWh, plus split charge/discharge variants,
   - `cycle_integrator` → SoC half-cycles via `DiffAbsSum`,
   - `charge_integrator` → tracks redistribution.

   All integrators **reject gaps** larger than `dt_max` (no interpolation across stalls) so accumulated values can't blow up on reconnect.
5. **Group aggregation** (`sampling.py:270-272`): pushes sample to the parent `BmsGroup` so a `VirtualGroupBms` can aggregate (mean voltage, summed currents, capacity-weighted SoC).
6. **Algorithm hook** (`sampling.py:283-304`): currently `SocAlgorithm` only — hysteresis-driven charge enable/disable with a calibration phase that forces a 100 % SoC touch periodically. Switch commands are pushed to the BMS, and algorithm state is persisted to JSON on disk.
7. **Publish gating** by `PeriodicBoolSignal`s plus power-jump detection (`sampling.py:347-405`): publishes immediately on >15 % power transients (held high-frequency for 4 s) or when crossing `over_power`, otherwise throttles to `publish_period`. Every 5 min republishes HA Discovery + a fresh sample.

Supporting:

- **`Downsampler`** averages high-frequency samples before publishing.
- **`LHQ`** (low-pass hysteresis quantiser) smooths quantised temperature steps and survives NaNs by holding the last good value.
- **`BatteryTracker`** (`tracker.py`) records the cell that has been simultaneously the most-empty and most-full — that's the weakest cell limiting pack capacity.

### Integrator catalogue (`pwmath.py`)

| Meter | Formula | Notes |
|---|---|---|
| `current_integrator` | ∫ I dt / 3600 | Total charge in Ah |
| `power_integrator` | ∫ P dt / 3600000 | Total energy in kWh |
| `power_integrator_discharge` | ∫ max(0,P) dt / … | Discharge energy only |
| `power_integrator_charge` | ∫ \|min(0,P)\| dt / … | Charge energy only |
| `cycle_integrator` | DiffAbsSum of SoC × 0.005 | Half-cycles (100→0 = 1 cycle) |
| `charge_integrator` | DiffAbsSum of Ah changes | Balancing diagnostics |

`DiffAbsSum` accumulates `|Δy|` only if `|Δy| ≤ dy_max` — hysteresis that rejects quantisation noise.

## 4. Persistence (`bmslib/store.py`)

- **Atomic JSON writes** (temp file + `os.replace`) under a single global lock.
- `bms_meter_states.json` — restored at startup so integrators survive restarts.
- `bat_state_{name}.json` — per-BMS algorithm state.
- `load_user_config()` reads `/data/options.json` (HA's options storage) with a fallback to local `options.json`, and silently migrates legacy config layouts.

## 5. MQTT layer (`bmslib/mqtt_util.py`)

- **Topic shape**: `{device_topic}/{soc|cell_voltages|temperatures|mosfet_status|bms|meter|switch}/...`.
- **`mqtt_single_out`** is the gatekeeper for outgoing values: it skips re-publishing identical payloads if less than ~10 s elapsed, updates `_last_publish_time` (the watchdog signal), and applies `round_to_n()` for compact significant-digit rounding via a custom JSON encoder.
- **HA Discovery** is generated from a central `sample_desc` field registry that maps each sample field to `device_class`, `state_class`, unit, precision, and `expire_after`. Discovery is published to `homeassistant/{component}/{device}/{field}/config` every 5 min so HA picks up restarts and new fields. Cell voltage and temperature discoveries are generated dynamically based on what the BMS actually reports.
- **Switches** are published as paired `switch` + `binary_sensor` entities so HA can both control and read.
- **Inbound commands**: `mqtt_message_handler` (paho thread) drops callbacks into a `queue.Queue`; `mqtt_process_action_queue` (async, called from `background_loop`) drains it on the asyncio loop so `bms.set_switch()` runs in the correct context. Subscriptions are registered lazily after the first successful sample.

## 6. Alternative sinks (`bmslib/sinks.py`)

A simple `BmsSampleSink` interface (`publish_sample`, `publish_voltages`, `publish_meters`):

- **`InfluxDBSink`** flattens nested fields, buffers to a 200k-point queue, dedups per-cell voltages with a 1 % random refresh to avoid stale series.
- **`TelemetrySink`** subclasses Influx with anonymised MAC/disk hashing and `silent=True` so telemetry failures never break the add-on.

## 7. Bluetooth transport (`bmslib/bt.py` + `bmslib/scan.py`)

- **`BtBms`** is the abstract base. Concrete adapters under `bmslib/models/` (jbd, daly, sok, ant, jikong, supervolt, victron, litime, …) and the wrapped `aiobmsble` plugins implement `fetch_device_info`, `fetch`, `fetch_voltages`, `fetch_temperatures`. Shared infra:
  - **Global `ConnectLock`** serialises concurrent BLE connects to dodge BlueZ "operation in progress" errors.
  - **`FuturesPool`** (in `bmslib/__init__.py`) gives adapters named async futures for request/response BLE patterns — acquire a future, write the command, the notification callback resolves it.
  - **Robust `start_notify`** tries a list of UUIDs (multi-variant hardware) and stops orphaned subscriptions first.
  - **`_connect_with_scanner`** does exponential-backoff reconnect, restarting the scanner between tries.
  - **PSK pairing**: when a `pin` is set, `client.pair(callback=get_passkey)` is called — this requires a patched Bleak, which is why the Dockerfile builds a **second venv** just for pairing.
- **Shared scanner pool** (`scan.py`): per-adapter `BleakScanner`s, cached with a 60 s idle timeout and a background `_stop_loop` that evicts them.
- **Wired transport** (`bmslib/wired/`): `SerialBleakClientWrapper` adapts pyserial / TCP sockets to the `BleakClient` interface using a blocking receive thread, so adapters can speak the same `start_notify` / `write_gatt_char` API over serial or networked serial.

## 8. Device construction (`bmslib/models/__init__.py`)

`construct_bms()` is a registry-based factory: `type` field → fully qualified class name → `importlib.import_module`. There's a fallback path that wraps `aiobmsble` plugins via `BLE_BMS_wrap.py`, so adding a new BMS doesn't require touching `main.py`. Address resolution converts names to MACs by scanning the discovery list; `"serial"` addresses require an explicit `alias`.

## 9. Groups (`bmslib/group.py`)

- **`BmsGroup`** aggregates the latest sample per member: mean voltage, summed currents/power/charge/capacity, capacity-weighted SoC, concatenated temperatures, **min timestamp** (so age reflects the staleest member).
- **`VirtualGroupBms`** presents a `BtBms`-compatible facade. Its `is_connected` only becomes true once all members have published at least once; `fetch()` waits up to 2 s for that. Switch commands fan out to every member.
- This decouples sampling cadence (per real BMS) from aggregation (a sampler attached to the virtual group publishes the combined view).

## 10. HA add-on packaging

- **`config.yaml`**: HA add-on manifest — version, arch list, options schema, services declaration. Declares the `devices` array and global options (sample_period, publish_period, watchdog, MQTT, InfluxDB, telemetry, expire_values_after, invert_current, concurrent_sampling, bt_power_cycle, keep_alive).
- **`build.yaml`**: per-arch HA base images.
- **`Dockerfile`**: installs python + BlueZ, creates two venvs (`venv_bleak_pairing` for the PSK-capable fork, `venv` for runtime), chmods the entrypoint, `CMD addon_main.sh`.
- **`addon_main.sh`**: bashio fetches MQTT credentials from the HA Supervisor, runs `main.py pair-only` in the pairing venv if PSKs exist, then `main.py` in the runtime venv.
- **`apparmor.txt`**: grants the org.bluez DBus surface, restricts mount/sys/proc, allows `/data` and `/share`.

## Notable design choices and tricks

- **Two watchdogs (coroutine + thread)** to survive Bleak/asyncio deadlocks; HA Supervisor restarts the process.
- **Gap-rejecting integrators** rather than interpolation — accumulated kWh/Ah remain trustworthy across reconnects.
- **Power-transient publishing** (15 % jump) sustains 4 s of high-rate publishing so HA captures load steps.
- **Dedup + significant-digit rounding** on MQTT slashes bandwidth and Influx series cardinality.
- **Lazy MQTT subscription** post-first-sample avoids subscribing for never-connected devices.
- **Two-venv Docker layout** isolates the patched Bleak needed for pairing.
- **Registry/dynamic-import BMS factory** keeps the device-adapter zoo loosely coupled.
- **Atomic JSON state + 30 s flush** keeps coulomb counters and algorithm state durable.
- **Cell tracker** elegantly identifies the pack-limiting cell purely from observed extremes.

## File map

| Area | File | Role |
|---|---|---|
| Orchestrator | `main.py` | startup, discovery, sampler wiring, watchdogs, shutdown |
| Sampling | `bmslib/sampling.py` | `BmsSampler`, `PeriodicBoolSignal`, `Downsampler` |
| Math | `bmslib/pwmath.py` | `Integrator`, `DiffAbsSum`, `LHQ` |
| Algorithm | `bmslib/algorithm.py` | `SocAlgorithm` charge control |
| Groups | `bmslib/group.py` | `BmsGroup`, `VirtualGroupBms`, `sum_parallel` |
| Persistence | `bmslib/store.py` | atomic JSON I/O, config loader |
| Tracker | `bmslib/tracker.py` | weakest-cell detection |
| MQTT/HA | `bmslib/mqtt_util.py` | publish, discovery, inbound queue |
| Sinks | `bmslib/sinks.py` | InfluxDB, telemetry |
| Data model | `bmslib/bms.py`, `bmslib/__init__.py` | `BmsSample`, `FuturesPool`, errors |
| BLE | `bmslib/bt.py`, `bmslib/scan.py` | `BtBms`, scanner pool, power cycling |
| Wired | `bmslib/wired/` | serial/TCP `BleakClient` shim |
| Adapters | `bmslib/models/`, `bmslib/bms_ble/` | per-device protocols |
| Packaging | `Dockerfile`, `addon_main.sh`, `config.yaml`, `build.yaml`, `apparmor.txt` | HA add-on integration |
