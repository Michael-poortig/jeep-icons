# Hardware Inventory — Jeep ZJ "Vantabeast"

Vehicle: Jeep ZJ Grand Cherokee, manual **NP242 Selec-Trac** transfer case
(5 detents: 2H / 4Hi-Part-Time / 4Hi-Full-Time / Neutral / 4Lo) with an
**NP231-based doubler**, **AX15** manual transmission (no Park position —
gear display deliberately has no "P").

Full detail lives in `cedra-firmware-jeep-security/BENCH-TEST-AND-WIRING.md`
(ASCII wiring diagram + a "still not wired" future-work table) and
`HANDOVER.md`/`HANDOVER-V2.7.md`. This file is the quick-lookup summary.

## Boards

| Board | Role | MAC | Notes |
|---|---|---|---|
| ESP32 DevKit (generic, 30-pin) | Main — all Jeep I/O | `F0:24:F9:0E:34:58` | **Confirmed hardware brownout defect** (see troubleshooting-faq.md) — currently kept radio-dark, replacement board pending. Bench: COM6. |
| AI-Thinker ESP32-CAM | `ext_hub` — WiFi/MQTT/OTA gateway (current role) | `84:1F:E8:68:82:B8` | Built with `-DBOARD_HAS_PSRAM=0` to free GPIO16 for a gear-switch input. Bench: COM7. Header pins only: 0, 2, 4, 12, 13, 14, 15, 16. |

## I2C bus (Main ESP32) — SDA GPIO21, SCL GPIO22, 400kHz

| Device | Address | Purpose |
|---|---|---|
| MCP23017 GPIO expander | `0x20` (A0/A1/A2 tied GND) | Port A: mixed I/O (relays + switch sense). Port B: transfer-case/doubler sense, all input. |
| MPU6050 IMU | `0x68` | Pitch/roll only, no compass. Strict `WHO_AM_I==0x68` check — a clone answering `0x98`/`0x70` is rejected by design (TODO[hw] if that's ever seen). |

## CCD bus (Chrysler, 7812.5 baud)

Interface: **2x LM358 op-amps wired as comparators** (already built/in
stock) — squares the CCD bus's ~2.5V-idle analog waveform into UART logic
levels. This is **not** a CDP68HC68S1 or laszlodaniel CCDBusTransceiver
board (a documented correction — some other notes elsewhere say otherwise).
RX on GPIO36 (Serial1), raw-dump TX on GPIO0 (Serial2) to a second
USB-UART dongle for sniffing.

## Relays (16 total, `gpio.cpp` `RELAY_DEFS[]`) — all active-LOW/inverted

| Relay | Pin | Type | Tracked (persisted) |
|---|---|---|---|
| Lock | GPIO 23 | Pulse 500ms | No |
| Unlock | MCP23017 GPA1 | Pulse 500ms | No |
| LED Bar | MCP23017 GPA0 | Toggle | Yes |
| Cube Lights | GPIO 19 | Toggle | Yes |
| Engine Start | GPIO 18 | Pulse 800ms | No |
| Drv Win Down | GPIO 5 | Pulse 1200ms | No |
| Drv Win Up | GPIO 13 | Pulse 1200ms | No |
| Air Compressor | MCP23017 GPA6 | Toggle | Yes |
| Front Diff Lock | GPIO 14 | Toggle | Yes |
| Rear Diff Lock | GPIO 27 | Toggle | Yes |
| Parking Lights | GPIO 26 | Toggle | Yes |
| Headlights | GPIO 25 | Toggle | Yes |
| Hazards | GPIO 33 | Toggle | Yes |
| Fog Lights | GPIO 32 | Toggle | Yes |
| Pass Win Down | GPIO 4 | Pulse 1200ms | No |
| Pass Win Up | MCP23017 GPA7 | Pulse 1200ms | No |

Several relays used to be on native GPIO pins (LED Bar was 21, Unlock was
22, Air Compressor was 12, Pass Win Up was 15) and were migrated to the
MCP23017 in v1.5/v1.7.2 — GPIO12/15 in particular are ESP32
boot-strapping pins and caused a boot loop before the migration (see
troubleshooting-faq.md).

**Diff-lock priming** (implemented independently in both the firmware and
both client apps): turning on a diff lock first energizes the air
compressor and waits ~1500ms before actually engaging the locker relay.
Turning a locker off does *not* turn the compressor back off.

**jeep-control-panel's relay catalog** exposes 14 of these 16 (9 switches
+ 5 buttons) — lock/unlock are excluded from the generic catalog because
the firmware folds them into one dedicated MQTT Lock entity instead. A
unit test (`RelayCatalogTest.kt`) pins these exact 14 names as a fixture
specifically to catch a firmware-side rename before it becomes a silent
MQTT mismatch:
```
Switches: led_bar, cube_lights, air_compressor, front_diff_lock, rear_diff_lock,
          parking_lights, headlights, hazards, fog_lights
Buttons:  engine_start, win_down, win_up, pass_win_down, pass_win_up
```

## Switches / sense inputs

| Signal | Location | Notes |
|---|---|---|
| Headlights switch | MCP23017 GPA2 | Active LOW, internal pull-up |
| Parking switch | MCP23017 GPA3 | Active LOW |
| Turn Left (sense only) | MCP23017 GPA4 | Active LOW |
| Turn Right (sense only) | MCP23017 GPA5 | Active LOW |
| T-case LOW range | MCP23017 GPB0 | Active LOW, 30ms debounce |
| T-case NEUTRAL | MCP23017 GPB1 | Active LOW |
| T-case 4WD engaged | MCP23017 GPB2 | Active LOW |
| T-case full-time mode sense | MCP23017 GPB3 | NP242 split sense (v1.6+) |
| Doubler engaged | MCP23017 GPB4 | NP231 sense (v1.6+) |
| Ignition sense | GPIO 17 | Voltage-presence off ignition-switched 12V via divider. Polarity **unconfirmed on bench** (`IGN_SENSE_ACTIVE_LOW` assumed `false`) — TODO[hw]. |
| Physical AP-toggle button | GPIO 16 (main board) | Active LOW, 50ms debounce, 5000ms holdoff |
| Gear switches (ext_hub) | 1st=14, 2nd=13, 3rd=**0** (⚠ download-mode pin — don't power-cycle CAM in 3rd gear), 4th=15, 5th=16, Reverse=2 | All `INPUT_PULLUP`, active LOW, grounded=engaged |

## Battery sensing

GPIO 34 (ADC1_CH6, input-only), 30K+10K voltage divider (4.0x ratio), 10
averaged samples. `displayed_voltage = (pin_voltage/3.3V) × 3.3 × 4.0`. A
100nF cap from pin to GND is recommended for noise smoothing.

## Power

`Jeep 12V IGN → buck converter (12V→5V, e.g. LM2596-class, NOT a linear
regulator) → both ESP32 5V pins + opto-relay module 5V`.

## Not yet wired (future work, per BENCH-TEST-AND-WIRING.md)

High-beam sense, fog sense, e-brake, seatbelt, 6x door-ajar sensors,
fuel/oil/coolant/oil-temp/trans-temp senders, ambient thermistor.
Phone/HA-sourced signals (GPS speed, weather, phone accelerometer/gyro)
need no GPIO — they arrive over MQTT from Home Assistant automations
(see `mqtt-topics-and-sync.md`).

## Calibration TODOs (open, in `src/config.h` and CCD parser)

- `IGN_SENSE_ACTIVE_LOW` polarity unconfirmed.
- `IGN_RUNNING_RPM=400` (CRANKING→RUNNING threshold) uncalibrated.
- `IMU_PITCH_OFFSET_DEG` / `IMU_ROLL_OFFSET_DEG` are placeholders (0.0f),
  pending zeroing on flat ground at install.
- Every CCD scaling formula (ported from the dashboard repo's
  `can-backend/src/protocols/ccd-protocol.ts`) is pending real-bus bench
  confirmation; the `0xB2` DTC byte layout is explicitly speculative.
