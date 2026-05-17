# Converting batmon-ha to a Home Assistant Integration — using HA's Bluetooth stack

**Goal:** convert the current standalone HA *add-on* (a separate Docker container that talks to BMS devices over its own Bleak stack and bridges to HA via MQTT) into a HA *integration* (a Python package under `custom_components/batmon/`) that connects to BMSes through Home Assistant's built-in `bluetooth` integration. The headline win is that any device reachable from an **ESPHome Bluetooth proxy** becomes automatically reachable, with no per-host BlueZ adapter on the HA server required.

This document covers **only** the BLE-transport migration. MQTT remains in place for now (replacing it is the follow-up in `DITCH_MQTT.md`).

---

## 1. Why this is non-trivial

The current codebase is built around the assumption that it **owns the Bluetooth subsystem**:

- It manages adapters: `bt_controllers_hci`, `bt_power`, `bt_power_cycle`, AppArmor DBus rules to `org.bluez`.
- It owns the scanner: `bmslib/scan.py`'s `get_shared_scanner`, `_stop_loop`, per-adapter cache, `resolve_address`.
- It owns connection serialisation: `ConnectLock`, `_connect_with_scanner`, exponential reconnect.
- It uses a **patched Bleak** (built into a second venv) to support BLE pairing with PSK.
- Discovery is driven by `bt_discovery()` returning a flat device list that `construct_bms()` consumes.
- It runs its **own watchdogs**, both async and threaded, and `exit_process(suicide)` on failures.

Inside HA, almost none of that is allowed or desirable: HA's `bluetooth` integration owns the adapters and the BlueZ DBus session, multiplexes Bleak-style connections across local adapters *and* ESPHome proxies, and reloads integrations on failure. The integration must surrender control of those layers and become a guest on HA's BT stack.

---

## 2. Target architecture

```
custom_components/batmon/
├── manifest.json          # declares dependency on `bluetooth`, BT matchers, requirements
├── __init__.py            # async_setup_entry: build coordinator, forward platforms
├── config_flow.py         # discovery (Bluetooth) + user-initiated flow
├── const.py
├── coordinator.py         # ActiveBluetoothDataUpdateCoordinator per device
├── bms/                   # the existing bmslib/models/* adapters, refactored
│   ├── __init__.py        # construct_bms() rewritten to accept a BLEDevice
│   ├── base.py            # slimmed-down BtBms (no scanner, no ConnectLock)
│   ├── jbd.py … sok.py    # mostly unchanged protocol logic
├── algorithm.py           # SocAlgorithm (moved over, mostly intact)
├── pwmath.py              # integrators (unchanged)
├── group.py               # virtual group (unchanged conceptually)
├── store.py               # rewritten on top of homeassistant.helpers.storage.Store
├── sensor.py              # placeholder; only used in the MQTT-still-on-the-wire mode
└── strings.json / translations/en.json
```

In this milestone, `mqtt_util.py` is kept verbatim and called from the coordinator's update callback, so the *outward* HA-visible behaviour (auto-discovery topics, sensor entities created by HA's MQTT integration) does not change.

---

## 3. Required code changes

### 3.1 `manifest.json`

```jsonc
{
  "domain": "batmon",
  "name": "BatMon BMS",
  "version": "x.y.z",
  "iot_class": "local_polling",
  "dependencies": ["bluetooth"],
  "requirements": [
    "bleak-retry-connector>=3.5",
    "paho-mqtt>=1.6",          // until we ditch MQTT
    "aiobmsble>=…"             // already a runtime dep
  ],
  "bluetooth": [
    { "manufacturer_id": 0x0A0A },                  // example per BMS family
    { "service_uuid": "0000fff0-0000-1000-8000-00805f9b34fb" },
    { "local_name": "JK-*" },
    { "local_name": "SOK-*" }
  ],
  "config_flow": true
}
```

Each BMS family contributes one or more matchers so HA dispatches a `BluetoothServiceInfoBleak` to our `config_flow.async_step_bluetooth` automatically when a device or proxy advertises it.

### 3.2 Replace the scanner/connection layer (`bmslib/bt.py` + `bmslib/scan.py`)

Everything tied to local adapter management is deleted:

| Removed | Replacement |
|---|---|
| `bt_discovery`, `bt_controllers_hci` | `bluetooth.async_discovered_service_info(hass)`; advertisements arrive via the BT matcher |
| `get_shared_scanner`, `_stop_loop`, `SCANNER_TIMEOUT` | HA's central scanner; the integration does not start its own |
| `bt_power`, `bt_power_cycle` | gone — HA owns the radio; user must use the `bluetooth_auto_recovery` integration if needed |
| `ConnectLock` | gone — `bleak_retry_connector.establish_connection` already serialises and retries |
| `_connect_with_scanner`, exponential reconnect loop | `establish_connection(BleakClientWithServiceCache, ble_device, name)` plus `ActiveBluetoothDataUpdateCoordinator` |
| `BtBms.connect()` / `disconnect()` lifecycle | accepts a `BLEDevice` resolved per cycle via `bluetooth.async_ble_device_from_address(hass, address, connectable=True)` |

The `BtBms` base class shrinks to **protocol + state**; transport becomes the caller's job. Its constructor should change from `(address, name, …)` to `(ble_device: BLEDevice, name, …)`, and `connect()` becomes:

```python
self._client = await establish_connection(
    BleakClientWithServiceCache,
    ble_device,
    name,
    disconnected_callback=self._on_disconnect,
    use_services_cache=True,
)
```

`FuturesPool` and `start_notify` stay; they are protocol-level and orthogonal to transport.

### 3.3 Coordinator and entity wiring

Each configured BMS gets one `ActiveBluetoothDataUpdateCoordinator` (HA helper that piggy-backs on advertisement callbacks to *trigger* polls only when the device is in range, and gracefully suspends polling otherwise). The coordinator's `_async_update` calls a stripped-down `BmsSampler.sample()` that:

1. Resolves the `BLEDevice` via `async_ble_device_from_address`.
2. Calls `BtBms.fetch()`/`fetch_voltages()`/`fetch_temperatures()`.
3. Feeds the integrators (unchanged).
4. Returns a dict that downstream consumers (MQTT layer or — later — entity platforms) render.

The current `fetch_loop`, `parallel_fetch`, `background_loop`, and the dual watchdogs go away: HA's coordinator handles scheduling, error tracking (`last_update_success`), and exponential backoff. The "publish on power jump" optimisation in `sampling.py:347-405` can be preserved by short-circuiting the coordinator's interval (call `coordinator.async_set_updated_data()` from inside the sample callback when a transient is detected).

### 3.4 Config flow

Two entry points:

- **`async_step_bluetooth`**: invoked automatically when HA sees an advert that matches a manifest matcher (works equally for local adapter or BT-proxy adverts). Offer the user a friendly setup form with detected device name + protocol guess.
- **`async_step_user`**: manual entry; lets the user pick an already-discovered device from `bluetooth.async_discovered_service_info(hass)` and choose its BMS type + alias + (optional) PIN/PSK + sample period + algorithm.

The existing `devices:` list in `config.yaml`/`options.json` migrates to one HA `ConfigEntry` per device (preferred, gives per-device options/diagnostics) or one entry with sub-devices. Per-device is cleaner; it costs only a one-time importer that reads the old options.json once and creates entries.

Group/virtual BMSes become entries that *reference* other entry IDs in their data — `VirtualGroupBms` then resolves member coordinators via `hass.data[DOMAIN][member_entry_id].coordinator`.

### 3.5 Persistence (`bmslib/store.py`)

Replace:

- `load_user_config` → ConfigEntry data/options + `async_migrate_entry`.
- `bms_meter_states.json` → `homeassistant.helpers.storage.Store` keyed per entry (`f"batmon_meter_{entry_id}"`).
- `bat_state_{name}.json` → same, keyed per entry.

`Store` handles atomic writes and JSON serialisation, so the temp-file + lock dance in the current `store.py` is dropped.

### 3.6 Process model / watchdogs

Remove `background_thread`, `bg_checks`, `exit_process(True, True)`, signal handlers, and the `mqtt_last_publish_time` watchdog. HA itself reloads the entry on repeated coordinator failures; surfacing problems is done via `coordinator.last_exception` and the new `ConfigEntry.async_setup_locks` health pattern.

### 3.7 Adapter selection & BT power cycle

Gone. The user picks adapters via HA's bluetooth integration (per-device "Connectable" routing is automatic). For local-only deployments without proxies, no UX change is visible. The `bt_power_cycle` knob disappears — users needing that should install the official `bluetooth_auto_recovery` integration alongside.

### 3.8 Wired (serial) transport

This is **out of scope** for the BT migration and arguably no longer fits as part of a BLE-focused integration. Two options:

1. Drop wired support from the integration; users on serial BMSes continue using the add-on or move to a separate `batmon_serial` integration.
2. Keep `SerialBleakClientWrapper` as a non-bluetooth code path *inside* the integration, bypassing the HA bluetooth stack entirely (the coordinator just talks to a serial port). This works but it is an exception to the architecture and complicates `manifest.json` / config flow.

Recommend option 1 for the first cut.

---

## 4. ESPHome Bluetooth Proxy implications

This is the principal motivation for the migration, so worth documenting in detail.

### What works "for free"

When the integration calls `bluetooth.async_ble_device_from_address(hass, address, connectable=True)` and then `establish_connection(...)`, HA transparently picks the best route to the device — a local HCI adapter, or one of the ESPHome proxies. Every read/write/notify subscription is tunnelled. No code in `BtBms`/adapter subclasses needs to know whether it is talking through a proxy.

GATT-only protocols (JBD, JK/JiKong, Daly, Supervolt, Victron, LiTime, ANT) are fully supported.

### What does **not** work

- **BLE pairing with a PIN/PSK.** ESPHome's BT proxy does not proxy the pairing agent — pairing is a local-controller operation. Today the project supports SOK and a few others via a patched Bleak with `client.pair(callback=get_passkey)`. With proxies, those devices must already be paired to the local controller (if HA has one) and routed through it, or pairing must happen out-of-band on the proxy device. **The "two-venv" Dockerfile trick disappears, but PSK support also disappears for proxy-routed devices.** This must be called out clearly in the docs and in the config-flow UX.
- **Bonded/encrypted links.** Same root cause.
- **Bluetooth Classic / SPP.** Not relevant here (none of the BMS adapters use Classic), but worth recording.
- **Aggressive `keep_alive`.** Local Bleak can hold a GATT connection open indefinitely; proxies can hold connections too, but the *budget* (slots × proxy count) is shared with every other HA bluetooth integration. The migration should default `keep_alive=False` for proxy-routed devices and `keep_alive=True` only when the device is reachable via a local adapter. Easy check: `BLEDevice.details.get("source")` (in HA's wrapped device) identifies the proxy that yielded the device — if it points at a proxy, default to drop-and-reconnect per sample.
- **High publish rates.** A 1 s `sample_period` is fine locally, but each round-trip through a proxy is ~80–200 ms. Recommend warning the user (in the config flow's options step) if they pick `sample_period < 3` for a proxy-routed device.
- **Power-jump-triggered immediate publish.** Still works, but the proxy round-trip cost makes the 4-second sustain less valuable. Consider gating that path on local connectivity.

### Connection budget hints

ESPHome proxies advertise their maximum simultaneous-connection count. Surface a diagnostic sensor (or a repair issue) when the user has more BMSes routed through a single proxy than the proxy can hold concurrently.

---

## 5. Risks and open questions

- **PSK devices regression.** Migration breaks PSK pairing in the proxy-routed case. We need explicit doc + config-flow guards, otherwise SOK users (the most common PSK case) will silently fail.
- **Adapter affinity.** Some users today pin a BMS to a specific `hci` adapter via the per-device `adapter:` config. HA exposes "preferred proxy"/"preferred adapter" only via service-side hints. Need a fall-back: either drop the feature or implement an active-connection slot manager that biases `async_ble_device_from_address(... connectable=True)` towards the user's chosen source by polling `bluetooth.async_scanner_devices_by_address`.
- **Notification UUIDs.** Some adapters in `bmslib/models/*` try multiple notify UUIDs (`start_notify` in `bt.py:193-223`); this stays compatible because BLE characteristics are the same regardless of route, but worth a regression pass.
- **AppArmor profile / `bluetoothctl`.** Once the add-on no longer runs, `apparmor.txt`, `addon_main.sh`, the bashio handshake, and `bt_power` go away. The repo can be split: keep the add-on for legacy users, ship the integration via HACS in parallel.
- **`construct_bms` and the `aiobmsble` wrap (`BLE_BMS_wrap.py`).** The factory still works after refactor, but the signature change (BLEDevice instead of address) ripples through every adapter. Plan a single-PR refactor; ~12 adapter files touched, mostly mechanical.
- **Diagnostics & logging.** Move from `bmslib.util.get_logger` to `logging.getLogger(__name__)` and implement `async_get_config_entry_diagnostics` (HA convention).
- **Testing.** Use `pytest-homeassistant-custom-component`; `bluetooth` has a well-developed mock layer (`bluetooth_data_tools`, `BluetoothServiceInfoBleak` fixtures). The adapter protocol logic can largely keep its existing unit tests in `bmslib/test/`.

---

## 6. Suggested migration phases

1. **Phase 0 — Repository split.** Carve out `custom_components/batmon/` skeleton alongside the add-on. Add CI for both.
2. **Phase 1 — Refactor `BtBms` to accept `BLEDevice`.** Mechanical change across all adapters; tests stay green by injecting a mock `BLEDevice`.
3. **Phase 2 — Replace scanner/connect with `establish_connection`.** Drop `scan.py`, `ConnectLock`, `_connect_with_scanner`, `bt_power`, `bt_controllers_hci`.
4. **Phase 3 — Coordinator + config flow.** One entry per device; import `options.json`. MQTT layer still emits identical topics, so HA-side entities created by the MQTT integration remain in place.
5. **Phase 4 — Storage migration.** Move meter and algorithm state into `Store`.
6. **Phase 5 — Proxy hardening.** `keep_alive` heuristics, sample-period guardrails, connection-budget diagnostic, PSK-not-supported warning.
7. **Phase 6 — Cut the add-on.** Once parity is confirmed, deprecate the add-on. (This is also the gate for `DITCH_MQTT.md`.)

The first user-visible win — connecting BMSes through ESP32 BT proxies — lands at the end of **Phase 3**.

---

## Phase 1 — Implementation plan: refactor `BtBms` to accept a `BLEDevice`

**Objective.** Make every adapter able to be constructed with a `bleak.backends.device.BLEDevice` instead of (or in addition to) a string address, without breaking the existing add-on. This phase produces *no* user-visible change; it only introduces the seam that Phase 2 needs to swap out the transport.

**Non-goals for this phase.** No HA scaffolding yet. No removal of `scan.py`, `ConnectLock`, `bt_power`, or `bt_discovery`. No `manifest.json`, no config flow. The add-on must still build and run identically at the end of this phase.

### Scope of change

Today every adapter follows the pattern `__init__(self, address, **kwargs)` → `super().__init__(address, **kwargs)`, and `BtBms.__init__` (`bmslib/bt.py:130`) immediately uses that `address` to build a `BleakClient` via `_create_client()` (`bmslib/bt.py:177-187`), then `_connect_client()` (`bmslib/bt.py:270-324`) re-resolves the address through the shared scanner before connecting.

The seam we want is: **construction takes either a string or a `BLEDevice`**, and **connection does not re-resolve when a `BLEDevice` was given**. That single change is what makes the class re-usable from a future HA coordinator that already has a `BLEDevice` in hand from `bluetooth.async_ble_device_from_address(hass, address, connectable=True)`.

### Concrete edits

1. **`BtBms.__init__` (`bmslib/bt.py:130-176`).**
   - Accept `address_or_device: str | BLEDevice` as the first positional parameter (kept as `address` in the signature for backwards compat — type only widens).
   - Detect the `BLEDevice` case: `isinstance(address_or_device, BLEDevice)` (lazy-import to keep startup cost identical).
   - Store both: `self._ble_device = address_or_device if BLEDevice else None`, `self.address = address_or_device.address if BLEDevice else address_or_device`.
   - When a `BLEDevice` is supplied, skip the `address.startswith('test_')`, `'serial'`, and PSK warning branches that depend on string-shape parsing — those paths only apply to the string-address case.
   - Build `self.client = self._create_client(self._ble_device or self.address)` unchanged afterwards; `BleakClient` already accepts either.

2. **`BtBms._connect_client` (`bmslib/bt.py:270-324`).**
   - Early-return path when `self._ble_device is not None`: do not call `resolve_address`, do not stop the shared scanner, do not reach into `bmslib.scan`. Just `await asyncio.wait_for(self.client.connect(timeout=timeout), timeout=timeout + 2)`.
   - String-address path is unchanged. Goal: the seam exists, the legacy code path is byte-for-byte identical for the add-on.

3. **`BtBms._create_client` (`bmslib/bt.py:177-187`).**
   - Accept a `BLEDevice` directly (it already does — `BleakClient(addr_or_device, ...)`), but stop appending `adapter=...` when `_ble_device` is set. Adapter selection is owned by whoever produced the `BLEDevice`. Inside the add-on path nothing changes.

4. **`bmslib/models/__init__.construct_bms` (`bmslib/models/__init__.py:74-113`).**
   - Add an optional `ble_device: BLEDevice | None = None` parameter.
   - When set, pass it through as the first positional to the resolved adapter class; otherwise behave exactly as today (resolves the name-or-address against the `ble_devices` list from `bt_discovery`).
   - Keep the `address.startswith('#')` skip and the `serial` handling on the string path only.

5. **`bmslib/models/BLE_BMS_wrap.py` (`BMS.__init__` at line 51).**
   - Mirror the same widening — accept a `BLEDevice` as `address` and pass it down to the wrapped `aiobmsble` plugin's connect path.

6. **Adapter subclasses (`bmslib/models/*.py`).**
   - **No code changes required.** Every concrete adapter forwards `address` to `super().__init__()` via `**kwargs`, and they call `self.client.write_gatt_char` / `self.start_notify` — none of them reach into `self.address` for transport purposes. The single grep-able exception is any `self.client.address` reference; those are safe because `BleakClient(BLEDevice)` populates `.address` from the device.

7. **`bmslib/bt.py` module-level.**
   - Add `try: from bleak.backends.device import BLEDevice; except ImportError: BLEDevice = None` near the existing bleak imports, so the `isinstance` check is cheap and safe across bleak versions.

### Tests

Existing tests in `bmslib/test/` already mock at the adapter-protocol level via `BleakDummyClient`. Phase 1 adds:

- **`test_bt_accepts_bledevice`** — construct a `JbdBt` (or any concrete adapter) with a fake `BLEDevice` whose `.address` matches the dummy address. Assert `_ble_device` is set, `_connect_client` does not call `resolve_address`, and the client connects.
- **`test_bt_address_path_unchanged`** — construct with a plain string address. Assert that the scanner-based resolution path still runs (mock `resolve_address` and assert it was called once).
- **`test_construct_bms_with_bledevice`** — call `construct_bms(dev, verbose, ble_devices=[], ble_device=fake)` and assert the resulting instance has `_ble_device is fake`.

Mock `BLEDevice` with a tiny dataclass-like stub if bleak's real class is awkward to instantiate; the `isinstance` check accepts any duck-typed object once we keep our own `_BLEDeviceProtocol` Protocol behind the scenes — but ideally use the real `bleak.backends.device.BLEDevice` for fidelity.

### Manual verification

Run the existing add-on against a real BMS (locally or via the test JK/JBD dummies in `bmslib/models/dummy.py`) before and after the patch:

```bash
python main.py
```

Behaviour, log lines, and MQTT topic output must be identical. The new code path is only exercised by the new tests; no production codepath calls into it yet.

### Risk surface

- **`address` attribute downstream.** `BmsSampler` uses `bms.address` in a few places (`main.py:194`, MQTT device-info construction). Keeping `self.address` populated from `BLEDevice.address` covers this.
- **PSK-aware path.** `_uses_pin` warnings and the `bleak.backends.bluezdbus.agent` import live in the string branch. PSK + `BLEDevice` is a Phase 5 problem; for Phase 1 it is enough that the BLEDevice path doesn't *forbid* PSK — it just skips the bluezdbus pairing-agent warning.
- **`self.client.address`.** Some adapters log `self.client.address`. `BleakClient(BLEDevice)` sets `.address` from the device, so this remains valid.
- **`SerialBleakClientWrapper`.** The `'serial'` branch in `__init__` (line 162-165) only fires on the string path; the `BLEDevice` path skips it. No change to wired transport.
- **`_create_client` `adapter=` kwarg.** Stripping `adapter=` when a `BLEDevice` is present is correct semantically but should be guarded by a regression test that confirms the string path still passes `adapter=` through unchanged.

### Deliverables and acceptance

- One PR titled `bt: accept BLEDevice in BtBms.__init__`.
- Diff confined to `bmslib/bt.py`, `bmslib/models/__init__.py`, `bmslib/models/BLE_BMS_wrap.py`, and `bmslib/test/`. Roughly **80–120 LOC** changed, mostly additive.
- New tests pass; existing test suite green.
- Manual add-on run produces an MQTT trace byte-equivalent to the pre-PR run for at least one real device (operator-verified, attach a screenshot to the PR).
- `git log -p` on the PR shows zero changes inside any `bmslib/models/*.py` adapter implementation file (only `__init__.py` and `BLE_BMS_wrap.py` touched).

When all four boxes are ticked, Phase 1 is done and Phase 2 can begin replacing `_connect_client`'s scanner path with `bleak_retry_connector.establish_connection(...)` for the `BLEDevice` branch — knowing the rest of the codebase already tolerates that shape.
