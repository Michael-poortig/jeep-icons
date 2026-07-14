# Troubleshooting & FAQ — Jeep ZJ "Vantabeast"

No repo has a dedicated TROUBLESHOOTING.md/FAQ.md — this content is
distilled from HANDOVER*.md files, BOOT-LOOP-HANDOVER.md,
TODO-REMEDIATION.md, and code comments across all 5 repos. This owner
tracks known issues in structured markdown handover docs, not inline
TODO/FIXME comments — a grep across `jeep-digital-cluster` source found
zero code-comment hits, only markdown-doc hits.

## CLOSED — Boot-loop / brownout (main ESP32 DevKit), closed 2026-07-06

**Symptom**: repeating `[EXP] MCP23017 absent` / `[IMU] MPU6050 absent` /
`Brownout detector was triggered`, alternating `POWERON_RESET`/
`SW_CPU_RESET` forever, on every boot across firmware v1.7.1–v1.7.7.

**Root cause**: hardware defect **inside the ESP32-WROOM module itself**
on this specific unit (failed on-module decoupling cap or damaged RF/power
section) — pinned to the exact call chain
`WiFi.softAPdisconnect(true)` → `WiFi.mode(WIFI_STA)`, i.e. right when
ESP-IDF's WiFi calibration/power-up burst (~300-500mA) runs. Ruled out
with real tests: USB cable/port, external 5V/2A+ PSU, cross-contamination
from the CAM, GPIO boot-strap conflicts (real but separate, fixed
v1.7.2), core 2.0.x vs 3.x, general board defect (a *different* firmware
booted clean on the same unit), WiFi TX power level, and 3 different bulk
capacitors across 3.3V/GND.

**Resolution**: physical board replacement (unreachable in software).
This is why `USE_EXT_HUB_WIFI_GATEWAY=1` (gateway mode) is currently
active — it's a permanent architectural workaround until a replacement
board with a working radio is installed, not a temporary diagnostic
state.

## FIXED — v1.7.7 stale retained MQTT gear reading on connect

Root cause: `_lastPublish` stayed `0` for the first 5 minutes after
construction, so the heartbeat gate never forced an immediate full-state
publish right after a fresh `connect()` — a stale retained
`cedra/jeep/sensor/gear/state` value could be shown to any new subscriber
for up to 5 minutes. Fixed with a one-shot `_justConnected` flag in
`mqtt_client.cpp`/`.h`.

## FIXED (historical, "don't reintroduce") — `MQTT_BASE` redefinition bug

`main.cpp` used to have an unguarded `#define MQTT_BASE "jeep"` placed
*after* `config.h`'s correct `#define MQTT_BASE "cedra/jeep"`, silently
redefining the macro — this made the command handler match the wrong
topic prefix, silently dropping every switch/button command except
lock/unlock (which worked by payload-matching coincidence). No existing
test caught it because the native test env excludes `main.cpp`. Already
fixed; flagged so nobody reintroduces a second `#define MQTT_BASE`
anywhere in `main.cpp`.

## FIXED — GPIO12/15 boot-strapping pin conflict (v1.7.2)

`air_compressor` (GPIO12) and `pass_win_up` (GPIO15) originally sat on
ESP32 boot-strapping pins, causing an *earlier, different* boot-loop.
Migrated to MCP23017 Port A bits 6-7. Required a physical rewire — if
anyone ever proposes moving a relay back onto a native GPIO, check it's
not 0/2/12/15 first.

## FIXED — v1.7.3 → v1.7.4 AsyncTCP/ESPAsyncWebServer migration

`platformio.ini` originally had **no version pin** on
`platform = espressif32`, so a fresh clone resolved to whatever was
newest (core 3.x/ESP-IDF 5.x), which broke the unmaintained
`me-no-dev/AsyncTCP`+`ESPAsyncWebServer` libs (`Guru Meditation Error` on
`ws.onEvent`, only compatible with old core 2.0.x). v1.7.3 was an interim
pin-back to core 2.0.17; v1.7.4 was the real fix — swapped to the
actively-maintained `esp32async` forks (drop-in compatible) and re-pinned
forward to `espressif32@7.0.1`. **This is the repo's own precedent** for
what to do when a pinned dependency goes stale — see the
dependency-freshness skill.

## FIXED — Android: broker rejects connection as "unauthorized"

Root cause: soft-keyboard autocomplete could append an invisible trailing
space to host/username/password fields in Settings, silently corrupting
the password. Fixed by trimming all 4 fields on save
(`SettingsViewModel.save()`). If this symptom recurs, check whether the
trim survived a refactor.

## Build/tooling gotchas

- **`pio` on PATH may be a broken shim**: `ModuleNotFoundError: No module
  named 'requirements'`. Locate the real CLI under
  `<platformio-home>/penv/Scripts/pio` (Windows:
  `~/.platformio/penv/Scripts/pio.exe`).
- **ESP32-CAM GPIO0 (3rd gear input) is a download-mode strap pin** —
  don't leave the Jeep in 3rd gear and power-cycle the CAM, or it'll boot
  into flash-download mode instead of running firmware.
- **Dual-ESP32 OTA relay uses a per-chunk stop-and-wait ack protocol**
  (`ota_relay.*`/`ota_b64.*`) — images built before this protocol landed
  are wire-incompatible. Always flash both boards from the same tree.
- **`USE_EXT_HUB_WIFI_GATEWAY` flips require rebuilding AND reflashing
  BOTH envs** — each personality only speaks its own protocol; flashing
  just one board after a flag flip leaves them unable to talk.
- **Never flash bare `firmware.bin` at offset 0x0/0x1000** — overwrites
  the bootloader, causes `RTCWDT_RTC_RESET` boot-loop. Always flash the
  merged `firmware-full.bin` (or use PlatformIO's normal upload target,
  which handles offsets correctly).

## Open, deferred (not bugs, just known gaps)

- `LAST-KNOWN-GOOD-CONFIG.md` (firmware repo) has plaintext credentials
  committed to git history — flagged, not yet rotated (see
  `configuration.md`'s credentials policy).
- Dashboard (`jeep-digital-cluster`) `tsconfig.json` is deliberately
  non-strict — ~270 strict-mode errors are blocked on a planned
  `clusterConfig.ts` redesign (`Partial<>` skin-overlay type system) that
  hasn't happened yet.
- Dashboard's large `App.tsx` (558 lines) split into smaller files is
  deferred as "largest, riskiest — do last."
- A round of manual bench checks in `jeep-digital-cluster/TODO-REMEDIATION.md`
  were never run as of the last audit: no-MQTT-env fail-fast test, relay
  rate-limit/spam-rejection test, malformed-CAN-frame crash test, 5-minute
  simulator soak test, full demo end-to-end walkthrough.
- `jeep-icons` → `jeep-control-panel` icon deployment is a manual,
  undocumented conversion+copy step (PNG source → `.webp` app resource) —
  no script/CI automates it. If asked to "update an icon," flag that this
  step has to happen by hand unless someone builds automation for it.
- `Home-Assistant` repo is effectively empty on disk (see `repo-map.md`);
  useful content only exists in git history at commit `909448c`.

## FAQ

**Q: Why is the main ESP32 not on WiFi directly?**
A: Confirmed hardware brownout defect on that specific board (see closed
postmortem above). The ESP32-CAM (`ext_hub`) is the gateway until the main
board is physically replaced.

**Q: Why does the dashboard use a different MQTT topic prefix than the
firmware?**
A: Open, unresolved question — see `mqtt-topics-and-sync.md`'s "known open
issue" section. Not something to silently fix.

**Q: The gear display shows no "P" — is that a bug?**
A: No — it's a manual AX15 transmission, there is no Park position. A
code comment in the dashboard confirms "P was auto-trans cruft" from an
earlier assumption.

**Q: Can I flip `USE_EXT_HUB_WIFI_GATEWAY` back to classic mode right
now?**
A: Only once the main board's radio is replaced — flipping it while the
current board is still installed will just re-trigger the brownout.
