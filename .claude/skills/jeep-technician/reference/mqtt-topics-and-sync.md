# MQTT Topics & Cross-Repo Signal Sync

## ⚠️ KNOWN OPEN ISSUE — MQTT topic-base mismatch (not fixed, flagged only)

There are currently **three different MQTT topic-base conventions** in
play for what's conceptually the same relay/sensor set. This has never
been consolidated anywhere else — surfacing it is this skill's main
value-add on this topic. Do not silently "fix" this without confirming
with the owner which side is correct and testing on real hardware; it's a
live behavior change on a bridge that talks to a vehicle.

| Source | Topic base | File:line |
|---|---|---|
| Firmware (real, current) | `cedra/jeep` | `cedra-firmware-jeep-security/src/config.h:74` |
| Mobile app (verified matches firmware) | `cedra/jeep` | `jeep-control-panel` `MqttTopics.kt` — code comment explicitly calls a bare `jeep`-prefixed scheme "documented-but-dead-end" |
| HA automations (phone→MQTT bridge) | `cedra/jeep` | recovered `automations.yaml`, git commit `909448c` in `Home-Assistant` |
| HA config, one legacy entity | `jeep` (bare) | recovered `configuration.yaml`, git commit `909448c`: `state_topic: "jeep/ignition"` |
| Dashboard backend (`can-backend`) | `jeep` (bare) | `jeep-digital-cluster/can-backend/src/server.ts:65` — `const MQTT_BASE = 'jeep'`, hardcoded |

The firmware itself once had a bug where `main.cpp` had a stray
`#define MQTT_BASE "jeep"` shadowing the correct `cedra/jeep` — that's
already fixed (see troubleshooting-faq.md), but it may be why the `jeep`
bare-prefix convention exists elsewhere: something built against the
buggy intermediate state and was never updated after the firmware fix
landed. Worth asking the owner whether `can-backend` and the legacy HA
`jeep/ignition` entity are supposed to be migrated to `cedra/jeep`, or are
intentionally a separate namespace.

## Firmware's real topic namespace (`cedra/jeep/...`)

```
cedra/jeep/availability                    retained LWT, "online"/"offline"
cedra/jeep/global_status                   JSON: {status, door, uptime}
cedra/jeep/lock/set  |  .../lock/state     "LOCK"/"UNLOCK"  |  "locked"/"unlocked"
cedra/jeep/switch/<relay>/set              "ON"/"OFF"  (9 toggle relays)
cedra/jeep/switch/<relay>/state            retained
cedra/jeep/button/<relay>/press            "PRESS", momentary (5 pulse relays)
cedra/jeep/sensor/battery/state            float volts
cedra/jeep/sensor/gear/state               raw string (no "P" — manual trans)
cedra/jeep/sensor/tcase/state              transfer-case mode (v1.7+)
cedra/jeep/binary_sensor/doubler/state     (v1.7+)
cedra/jeep/sensor/coolant/state            °C (v1.7.8+)
cedra/jeep/sensor/oil_pressure/state       psi (v1.7.8+)
cedra/jeep/sensor/ignition/state           "off"/"on"/"running" (v1.7.8+)
cedra/jeep/ota/set  |  .../ota/status      OTA trigger ("main"|"cam"|"both") / status
```

HA auto-discovery under `homeassistant/...` prefix, device
`cedra_jeep_vantabeast` ("Jeep Vantabeast", manufacturer "CEDRA Systems").
`ha-dashboard.yaml` (firmware repo root) is a ready-to-import Lovelace
dashboard referencing these exact entity IDs.

## Phone → HA → MQTT bridge (recovered `automations.yaml`, commit `909448c`)

Publishes phone-companion-app sensor data to `cedra/jeep/...`:

```
cedra_gps_speed_to_mqtt        -> cedra/jeep/gps_speed          (phone GPS)
cedra_weather_to_mqtt          -> cedra/jeep/ha/weather_temp, _humidity, _condition, _wind
cedra_altitude_to_mqtt         -> cedra/jeep/ha/altitude        (barometric, from phone pressure sensor)
cedra_bearing_to_mqtt          -> cedra/jeep/ha/bearing         (device_tracker course)
cedra_gforce_to_mqtt           -> cedra/jeep/ha/gforce_x/y/z    (phone linear accel ÷ 9.81)
cedra_phone_orientation_to_mqtt -> cedra/jeep/ha/phone_pitch, _phone_roll
```

These correspond to "needs HA automation" rows in the dashboard's
`docs/sync-checklist.md`, and are the actual working implementation of
that gap — GPS/weather/altitude/bearing/g-force/phone-orientation need no
vehicle GPIO because they arrive over MQTT this way.

## 4-way signal sync reference

`jeep-digital-cluster/docs/sync-checklist.md` (in that repo) is an
existing, well-maintained table of every CCD-bus signal, serial key, MQTT
topic, and sync status (✅/⚠️/❌) across Jeep↔ESP32↔HA↔Cluster — read that
file directly for the authoritative, currently-maintained version rather
than duplicating it here. As of the last audit its open gaps were:
parking-brake wiring not done (hardware), MCP23017 hall-effect calibration
for AX15 gear/transfer-case/doubler position needs bench verification, and
confirming the HA companion app actually publishes phone IMU data to the
expected topics.

## Tracked relay allowlist (must stay in lock-step across 3 places)

`cedra-firmware-jeep-security/src/gpio.h` (`RelayId` enum) ↔
`jeep-digital-cluster/can-backend/src/relays.ts` (dashboard's authoritative
allowlist + rate limiting) ↔ `jeep-control-panel`'s `RelayCatalog.kt` (14
of 16, lock/unlock split into a dedicated entity). If a relay is
renamed/added/removed in one, the other two need updating — this is
exactly the kind of change worth drafting with
`code-change-request-template.md` since it spans repos.
