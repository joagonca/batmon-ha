# Dropping MQTT — exposing batmon entities natively from the HA integration

> **Prerequisite:** this plan only makes sense **after** `HA_INT_MIGRATION.md` is complete. It assumes the project is already running as a custom HA integration (`custom_components/batmon/`), with one `ActiveBluetoothDataUpdateCoordinator` per BMS feeding a refactored `BmsSampler`. Do not attempt this as a standalone change against the add-on — there is nowhere to host the entities.

The current MQTT bridge does double duty: it is the integration's transport *and* its entity-discovery mechanism (via HA's MQTT integration's auto-discovery topic `homeassistant/...`). Once we own the integration, we can register entities directly with HA, removing the MQTT broker, the paho client, the discovery republish loop, the dedup cache, the action queue, and the `mqtt_last_publish_time` watchdog.

This document plans that removal.

---

## 1. What `mqtt_util.py` actually does today

To know what we are replacing, first an inventory of obligations:

| Today (`bmslib/mqtt_util.py`) | What it provides |
|---|---|
| `mqtt_single_out()` (dedup + significant-digit rounding) | Sends one value to one MQTT topic |
| `sample_desc` field registry | Maps each BMS field to `device_class`, `state_class`, unit, precision, `expire_after` |
| Discovery publisher (5-min loop in `sampling.py:388-405`) | Tells HA's MQTT integration to create `sensor`/`binary_sensor`/`switch` entities |
| `mqtt_message_handler` (paho thread) → `_message_queue` → `mqtt_process_action_queue` (asyncio) | Routes inbound switch commands back to `bms.set_switch()` |
| `mqtt_last_publish_time` | Watchdog signal for the outer process |
| `round_to_n` + custom JSON encoder | Compact payloads |
| Per-cell voltage dedup + 1 % random refresh (`sinks.py`) | Cardinality control for InfluxDB |
| Availability via `expire_after` on each discovery message | Marks entities unavailable on stalls |

Every one of those concerns has a direct, idiomatic HA equivalent.

---

## 2. Target shape

```
custom_components/batmon/
├── __init__.py            # async_setup_entry: build coordinator, forward [sensor, binary_sensor, switch]
├── coordinator.py         # produces a typed BmsState dataclass on each update
├── entity.py              # BatmonEntity base (CoordinatorEntity[Coordinator])
├── sensor.py              # SensorEntityDescription table + entity factory
├── binary_sensor.py       # MOSFET status etc.
├── switch.py              # charge/discharge enable switches
├── number.py              # (optional) tunables like over_power threshold
└── diagnostics.py
```

Each platform module declares `SensorEntityDescription`/`SwitchEntityDescription` tables built from the existing `sample_desc` registry, so the *mapping of fields to HA metadata* is preserved (we re-use the work, just write it once in the integration's Python instead of as MQTT JSON).

---

## 3. Mapping `mqtt_util.py` onto HA primitives

### 3.1 Per-value publishing → entity `_attr_native_value`

```python
@property
def native_value(self):
    return getattr(self.coordinator.data, self.entity_description.key)
```

The dedup logic in `mqtt_single_out` is **redundant**: HA's state machine already suppresses identical-state writes (no `state_changed` event is fired). Drop it.

### 3.2 Discovery → `SensorEntityDescription`

The `sample_desc` table converts almost field-for-field:

| MQTT discovery key | `SensorEntityDescription` attribute |
|---|---|
| `device_class` | `device_class=SensorDeviceClass.…` |
| `state_class` | `state_class=SensorStateClass.…` |
| `unit_of_measurement` / `native_unit_of_measurement` | `native_unit_of_measurement=…` |
| `suggested_display_precision` | `suggested_display_precision=…` |
| `unique_id` | computed `f"{entry_id}_{key}"` |
| `expire_after` | n/a — replaced by `available` (see 3.5) |
| Device info block | a single `DeviceInfo(...)` shared by all entities of one entry |

The **5-minute discovery republish** disappears. HA persists entity registrations in its own registry; new fields appear when the next coordinator update yields them, by adding the entity at setup time.

### 3.3 Cell voltages and temperatures (dynamic counts)

Today the discovery loop publishes `cell_voltages/1..N` based on what the BMS reports. In the integration, two options:

1. **Per-cell entities.** On the first successful sample, the coordinator knows `n_cells`. The platform `async_setup_entry` waits for `coordinator.async_config_entry_first_refresh()`, then adds N `SensorEntity` instances. New cells appearing later (rare) trigger an `async_add_entities` call from a coordinator-update listener.
2. **One aggregate sensor + attributes.** A single sensor exposes min/max/delta as the state and per-cell voltages as `extra_state_attributes`. Simpler, but per-cell history graphs in HA are lost.

Recommend option 1 with `entity_category=EntityCategory.DIAGNOSTIC` and `entity_registry_enabled_default=False` for individual cells (keeps the device page clean; advanced users opt in). The min/max/delta/median stay as first-class enabled sensors.

### 3.4 Switches → `SwitchEntity`

The inbound MQTT path (`mqtt_message_handler` → `_message_queue` → `mqtt_process_action_queue` → `bms.set_switch`) collapses into:

```python
class BatmonSwitch(CoordinatorEntity, SwitchEntity):
    async def async_turn_on(self, **kw):
        await self.coordinator.bms.set_switch(self.key, True)
        await self.coordinator.async_request_refresh()
    async def async_turn_off(self, **kw): …
```

No paho thread, no `queue.Queue`, no asyncio bridge — the call is already on the event loop. The "publish state right after writing" pattern becomes `async_request_refresh()` (or just trust the next interval).

### 3.5 Availability → coordinator state, not `expire_after`

`expire_after` on each MQTT discovery payload is replaced by:

```python
@property
def available(self) -> bool:
    return super().available and self.coordinator.last_update_success
```

If the BLE link drops, the coordinator marks all entities unavailable in one place. The current per-topic 10-second dedup window and the per-discovery `expire_after_seconds` go away.

### 3.6 Watchdog (`mqtt_last_publish_time`)

Gone. The integration is hosted by HA; if the coordinator fails enough times, HA marks the entry unhealthy and surfaces a Repairs issue. The custom suicide watchdog (already removed in `HA_INT_MIGRATION.md` phase 2) is not reintroduced.

### 3.7 Significant-digit rounding (`round_to_n`)

Replaced by `SensorEntityDescription.suggested_display_precision`. Backend values stay full-precision (HA's recorder will store them efficiently), and the UI rounds for display. The custom JSON encoder and `json_dumps_with_round_n` simply disappear.

### 3.8 InfluxDB and Telemetry sinks (`sinks.py`)

- **InfluxDBSink** — HA's official `influxdb` integration consumes the state machine directly and writes one measurement per entity with proper tags. Drop our sink. Users keep their dashboards by pointing the official integration at the same DB.
- **TelemetrySink** — anonymised telemetry to `tm.fabi.me`. If we want to preserve it, it stays as a small async task subscribed to coordinator updates (`coordinator.async_add_listener`). Otherwise drop, with a clear note in the changelog. Independent of MQTT either way.

### 3.9 Per-cell-voltage dedup + 1 % random refresh

This logic was specifically a workaround for InfluxDB's "drop stale series" behaviour. With HA's recorder this is unnecessary; every state change is timestamped and stored. Drop.

---

## 4. Migration mechanics

### 4.1 Backwards compatibility for existing users

Users have HA automations and dashboards keyed on the **MQTT-discovered entity IDs** (e.g. `sensor.batmon_jbd_xyz_soc_total_voltage`). Native entities will get different unique IDs unless we are careful.

Two strategies:

1. **Adopt the existing unique IDs.** On first run after migration, look up the old MQTT entities by topic-derived unique ID and reuse them — i.e. set the new entity's `_attr_unique_id` to the value that the MQTT integration would have used. The user's automations keep working untouched. Requires careful audit of the MQTT integration's unique-ID derivation rules (essentially `{discovery_id}_{object_id}`).
2. **New unique IDs + a Repairs issue with a one-click "remove old MQTT entities" action.** Cleaner code, mild migration friction.

Recommend (1) for any field that maps 1:1; (2) only for entities that did not exist as MQTT discoveries.

### 4.2 Coexistence period

While the MQTT bridge and the native entities both exist, a config flag `expose_via_mqtt` (default *off* in new entries, *on* in entries migrated from the add-on) lets users keep MQTT topics flowing for the duration of one release, so external consumers (Node-RED flows, Grafana via the MQTT data source, etc.) keep working. The MQTT code path in `mqtt_util.py` is gated behind this flag and removed in the *next* major version.

### 4.3 Files removed once the flag is gone

- `bmslib/mqtt_util.py` — entire file.
- The MQTT branch of `sampling.py` (publish triggers, periodic discovery signals, `mqtt_last_publish_time` references).
- `paho-mqtt` from `requirements.txt` / `manifest.json`.
- `sinks.InfluxDBSink` and `sinks.TelemetrySink` (unless telemetry is intentionally retained).
- MQTT host/user/password options from the config flow.
- The "import MQTT credentials from env" code in `main.py` (already gone in the integration).

### 4.4 Files trimmed but kept

- `sampling.py` — the publish-trigger logic shrinks dramatically. The power-jump fast-path stays, calling `coordinator.async_set_updated_data()` to bypass the next interval. The 5-min discovery republish is deleted.
- `bms.py` `BmsSample` — still the data model, now the coordinator's payload type. Add a `@dataclass`/`TypedDict` annotation so platforms type-check against it.

### 4.5 Files unchanged

- `pwmath.py`, `algorithm.py`, `tracker.py`, `group.py`, `store.py` (the HA-Storage version), every protocol adapter under `bmslib/models/`.

---

## 5. Behavioural deltas to communicate in the changelog

- **Entity IDs may change.** Mitigated by unique-ID reuse where possible (see 4.1).
- **MQTT broker no longer required.** Add-on users who relied on the embedded discovery for non-HA consumers must enable the coexistence flag or use HA's MQTT *statestream* integration.
- **InfluxDB integration changes.** Users move from "BatMon → InfluxDB" to "HA → InfluxDB".
- **`expire_after_seconds`, `over_power`, `concurrent_sampling`, `mqtt_*`, `influxdb_*`, `telemetry`, `watchdog`, `bt_power_cycle`, `keep_alive` options** — most are removed or relocated. New options live on the per-entry options flow: `sample_period`, `algorithm`, `current_calibration`, `invert_current`, `pin`, `over_power`.
- **Per-cell voltage entities are hidden by default.** Users who want them must enable in the entity registry.
- **Faster updates.** Removing the publish/dedup gate means changes hit HA almost immediately; the 1 s typical update is now bounded only by `sample_period`.

---

## 6. Risks

- **MQTT-as-API users.** Anything outside HA that subscribed to BatMon's MQTT topics breaks. The coexistence flag buys one release of grace; document loudly.
- **Unique-ID drift.** Getting the unique-ID reuse wrong leaves users with both old (orphaned) and new entities. Add a migration validation step to `async_migrate_entry`.
- **Cell-count instability.** Some BMSes report a partial cell list during boot. Don't fix the cell count at first refresh — verify with two consecutive samples before adding entities, otherwise we end up with permanently disabled "cell 13" entries.
- **Switch-state echo.** Today, after an MQTT command, the code republishes the new switch state immediately. With `CoordinatorEntity`, the state visible in HA is whatever the next BMS poll returns; if the BMS lags, users see brief stale state. Mitigate with optimistic updates in `async_turn_on`/`async_turn_off`, then reconcile on next refresh.
- **Recorder volume.** Every cell voltage every second is heavy in HA's recorder. Default-disabling per-cell entities (and recommending `recorder:` excludes) keeps things sane.

---

## 7. Suggested order of work

1. Land the integration scaffolding from `HA_INT_MIGRATION.md` (entity-less, MQTT still on).
2. Add `SensorEntityDescription` tables derived from `sample_desc`.
3. Wire entity platforms behind a feature flag (`expose_via_native_entities=True`, MQTT still on); validate parity in a staging HA instance.
4. Implement unique-ID reuse for migrated entries; add `async_migrate_entry` with a Repairs flow for collisions.
5. Replace the switch path; remove `mqtt_message_handler`, `_message_queue`, `mqtt_process_action_queue`.
6. Default `expose_via_mqtt=False` for new installs; keep it on for migrated installs.
7. One release later, delete `mqtt_util.py`, drop `paho-mqtt`, simplify `sampling.py`, remove `sinks.py` (or trim to telemetry only).

The point of no return is step 4 — unique-ID reuse is the only thing that lets users keep their automations intact, and getting it right is the highest-leverage piece of this whole migration.

---

## Phase 2 — Implementation plan: native sensor entities in shadow mode

> **Pre-requisites.** This plan assumes that **Phases 0–3 of `HA_INT_MIGRATION.md` are complete**: the project ships a `custom_components/batmon/` integration with a `manifest.json`, a working config flow, and one `ActiveBluetoothDataUpdateCoordinator` per BMS whose `_async_update` already returns a `BmsSample`-shaped payload. The MQTT bridge is still wired up and emits the legacy topics; this phase only *adds* a parallel native-entity path next to it.

**Objective.** Stand up the first wave of native `SensorEntity` instances driven from the coordinator, with no user-visible regression. At the end of this phase, every static scalar field currently published as MQTT (the `sample_desc` table at `bmslib/mqtt_util.py:135-217`, plus the five meter sensors at `bmslib/mqtt_util.py:333-346`) has a corresponding HA-native sensor entity, but the **MQTT bridge keeps publishing in parallel** so HA still creates its old MQTT-discovered entities and the user's automations keep firing. Native entities live behind a per-entry `expose_via_native_entities` option that defaults to `False` for migrated entries and `True` for fresh installs.

**Non-goals for this phase.** No removal of `mqtt_util.py`. No `paho-mqtt` dropped from `requirements.txt`. No per-cell or per-temperature-sensor entities (their cell count is dynamic — that's Phase 3). No `SwitchEntity` (that's Phase 4, because of the action-queue and optimistic-update concerns). No `async_migrate_entry` for unique-ID reuse (Phase 5). No `expose_via_mqtt=False` default flip (Phase 6). The point of this phase is the *minimum* shippable native-entity surface to validate parity for the 13 static fields.

### Scope of change

The static surface to lift is small and well-defined:

| Source | Count | Notes |
|---|---|---|
| `sample_desc` entries (`mqtt_util.py:135-217`) | 13 | voltage, current, balance_current, soc, power, capacity, cycle_capacity, num_cycles, charge, mos_temperature, uptime, num_samples, etc. |
| Meter sensors (`mqtt_util.py:333-346`) | 5 | total_energy, total_energy_charge, total_energy_discharge, total_charge, total_cycles |
| **Total** | **18 static sensors per BMS** | All map cleanly to a `SensorEntityDescription`. |

No fields are dynamic in count; the entity registry is fully knowable at `async_setup_entry` time. That is what makes this a small, atomic phase.

### Concrete edits

1. **`custom_components/batmon/sensor.py`** (new file).
   - Define a `BATMON_SENSORS: tuple[SensorEntityDescription, ...]` built mechanically from `sample_desc`. Each entry maps:
     - `key` ← the *field name* on `BmsSample` (e.g. `"voltage"`, not `"soc/total_voltage"`). The legacy topic string is irrelevant to the entity; the unique-ID strategy here uses `f"{entry.entry_id}_{description.key}"` for now (Phase 5 swaps in topic-derived IDs).
     - `name`, `icon` from the existing dict.
     - `native_unit_of_measurement` from `unit_of_measurement`.
     - `device_class` mapped via a static dict to `SensorDeviceClass.VOLTAGE` etc. (no `None` → use `getattr(SensorDeviceClass, ..., None)`).
     - `state_class` mapped similarly to `SensorStateClass.*`.
     - `suggested_display_precision` from `precision`.
     - `entity_registry_enabled_default=True` for the 18 static fields.
   - Define a separate `BATMON_METER_SENSORS` tuple for the five meters at `mqtt_util.py:333-346`, with `state_class=SensorStateClass.TOTAL_INCREASING` on the charge/discharge variants exactly as the discovery does today.
   - `async_setup_entry(hass, entry, async_add_entities)`: gate on `entry.options.get("expose_via_native_entities", entry.data.get("created_fresh", False))`. If false, return without adding any entities. If true, instantiate `BatmonSensor(coordinator, description)` for each description and call `async_add_entities`.
   - `class BatmonSensor(CoordinatorEntity[BatmonCoordinator], SensorEntity)`:
     - `_attr_has_entity_name = True`.
     - `_attr_unique_id = f"{coordinator.entry_id}_{description.key}"`.
     - `_attr_device_info` shared with the coordinator's `DeviceInfo` (manufacturer/model/sw_version from `DeviceInfo` at `bmslib/bms.py`).
     - `native_value` reads `getattr(self.coordinator.data.sample, description.key, None)` for the 13 BmsSample fields. For the 5 meter sensors, it reads from a `coordinator.data.meters` dict (already present in the coordinator payload — that's what the existing `publish_meters` call drains).
     - `available` returns `super().available and value is not None and not math.isnan(value)`.

2. **`custom_components/batmon/__init__.py`** — extend the list of forwarded platforms.
   - Add `Platform.SENSOR` to the `PLATFORMS` tuple. (`Platform.SWITCH`, `Platform.BINARY_SENSOR` stay deferred to Phases 4/later.)

3. **`custom_components/batmon/coordinator.py`** — make the data shape explicit.
   - Replace whatever ad-hoc dict the coordinator returns today with a small dataclass:
     ```python
     @dataclass
     class BatmonData:
         sample: BmsSample
         voltages: list[int]
         temperatures: list[float]
         meters: dict[str, float]   # 'total_energy', 'total_energy_charge', etc.
         device_info: DeviceInfo
     ```
     Existing MQTT code keeps reading the same underlying objects; this is a typed-wrapper change only.

4. **`custom_components/batmon/const.py`** — central mapping tables.
   - `STR_TO_DEVICE_CLASS: dict[str, SensorDeviceClass]` covering `voltage`, `current`, `battery`, `power`, `temperature`, `duration`, `energy`.
   - `STR_TO_STATE_CLASS: dict[str, SensorStateClass]` covering `measurement`, `total_increasing`.
   - These are deliberately not re-derived from `mqtt_util.py` at runtime — kept as explicit constants so they're greppable and reviewable.

5. **`custom_components/batmon/config_flow.py`** — surface the feature flag.
   - Add an options-flow step exposing `expose_via_native_entities` (bool, default `False` for entries imported from `options.json`, default `True` for fresh user/bluetooth-discovery setups). One checkbox, one line of `vol.Schema`.
   - On fresh-install entries, set `entry.data["created_fresh"] = True` so the sensor platform picks the right default without the user having to click anything.

6. **`bmslib/mqtt_util.py`** — **no changes**. The MQTT path keeps emitting identical topics. This is what makes the phase reversible.

### `sample_desc` field-name caveat

`sample_desc` keys are MQTT *topic suffixes* (`"soc/total_voltage"`); the `field` sub-key (`"voltage"`) is the actual `BmsSample` attribute. The `SensorEntityDescription.key` in this phase **must** be the `field` value, not the topic suffix — that's what `getattr(self.coordinator.data.sample, key)` needs. Two `sample_desc` entries share the same field in a real edge case (`mosfet_status/capacity_ah` and `soc/capacity` both target `BmsSample.charge`/`capacity`? — verify before final implementation, dedupe or pick one). Capture that during the table-build step; if duplicates exist, keep the first and log a warning at import time.

### Tests

Use `pytest-homeassistant-custom-component`. Three tests:

- **`test_sensor_descriptions_cover_sample_desc`** — assert every `field` in `sample_desc` and every key in the meter dict appears exactly once across `BATMON_SENSORS + BATMON_METER_SENSORS`. Catches drift when `mqtt_util.py` adds a new field.
- **`test_sensors_register_when_flag_on`** — set up a config entry with `expose_via_native_entities=True`, run a coordinator refresh against a fake `BmsSample` populated with synthetic values, assert all 18 expected entity IDs appear in `hass.states.async_all()`.
- **`test_sensors_absent_when_flag_off`** — same setup with the flag `False`, assert *zero* `sensor.batmon_*` entities exist (only MQTT-discovered ones, which the test environment doesn't actually create — so the count is just zero).

For the `device_class` / `state_class` enum mapping, a parameterised test running each `sample_desc` entry through `STR_TO_DEVICE_CLASS` / `STR_TO_STATE_CLASS` and asserting no `KeyError` is cheap insurance.

### Manual verification

In a HA dev container with the integration installed and the flag enabled:

1. Set up one device. Wait one coordinator cycle.
2. Confirm via the HA UI that *both* sets of entities exist: the new `sensor.batmon_<entry>_voltage` (native) and the legacy `sensor.<device>_soc_total_voltage` (MQTT). They should report identical values to within the precision drop (native is full precision, MQTT is `round_to_n` rounded).
3. Toggle the flag off via the options flow; reload the entry; confirm the native entities disappear and the legacy ones remain.
4. Take an Energy dashboard reading: the legacy `total_energy_charge`/`total_energy_discharge` entities and the new ones should both qualify for the Energy dashboard (both have `state_class=total_increasing`, both have `device_class=energy`, both have a `kWh` unit). HA will let you pick either; that's the parity proof.

### Risk surface

- **Duplicate entities on Energy dashboard.** If a user adds both the legacy and native energy sensor to their Energy dashboard during the shadow period, totals double. Mitigation: in the options-flow help text and the release notes, explicitly tell users not to add the new sensors to Energy until they're ready to disable MQTT.
- **Recorder volume doubles during shadow period.** Every value is stored twice. Default the flag to `False` for migrated entries (already in the plan); document the cost; encourage short shadow windows.
- **`available` semantics differ.** Today, MQTT `expire_after` marks entities unavailable after silence. Native entities key on `coordinator.last_update_success`. If the BMS still streams data but a single field becomes NaN, the MQTT entity stays available with the last value; the native entity goes unavailable. That's arguably *better* but is a behaviour change worth flagging in the changelog.
- **DeviceInfo collision.** Native `DeviceInfo` uses `(DOMAIN, device_info.sn or entry.entry_id)` as the identifier. The MQTT integration creates a device under the `mqtt` domain with a different identifier scheme. HA will show *two* devices in the UI for the same physical BMS during the shadow period. Cosmetic, but worth a screenshot in the release notes.
- **Field-name mismatch.** Some `sample_desc.field` values reference attributes that might be `None` on a particular BMS (e.g. `balance_current` on a JBD). The `available` property already handles this; the test must cover at least one BMS where some fields are absent.
- **Coordinator data shape.** Touching `coordinator.py` to introduce `BatmonData` ripples into the still-active MQTT path inside `sampling.py`. Either: (a) keep the MQTT path consuming `coordinator.data.sample` (one-line change), or (b) leave the legacy code reading the raw `BmsSample` and have `BatmonData` wrap it via property. Pick (a); simpler.

### Deliverables and acceptance

- One PR titled `sensor: add native entity platform in shadow mode`.
- Diff confined to `custom_components/batmon/sensor.py` (new), `__init__.py`, `coordinator.py`, `const.py`, `config_flow.py`, and `tests/`. Roughly **250–350 LOC added**, **<20 LOC modified outside the new sensor.py**. Zero lines deleted from `mqtt_util.py`.
- All three new tests pass; existing HA integration tests stay green.
- Manual verification produces a screenshot showing native + MQTT entities side-by-side reporting matching values for at least voltage, current, power, soc, and total_energy.
- `grep -r "paho" custom_components/batmon/` returns zero hits — no native code path imports MQTT.
- Release notes draft included in the PR, with the four risk-surface items above called out for users.

When all five boxes are ticked, Phase 2 is done and Phase 3 (per-cell + per-temperature dynamic entities, which need first-sample cell-count stabilisation) can begin against the same coordinator-driven shadow surface — knowing the static-entity story is already proven.
