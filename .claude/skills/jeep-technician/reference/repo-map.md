# Repo Map — Jeep ZJ "Vantabeast" / CEDRA Systems Project

Audited 2026-07-14. This file replaces a `~/.claude/CLAUDE.md` "full index"
that the firmware repo's CLAUDE.md refers to but which does not actually
exist in any session's environment — this skill (mirrored into all 5
repos) is the real index now.

Never conflate this project with the sibling repo
`Cedra-Client-Cluster-Firmware` — different client, different project, do
not mix code, docs, or credentials between them.

## The 5 repos

| Repo (GitHub) | On-disk path (this environment) | What it is |
|---|---|---|
| `Michael-poortig/cedra-firmware-jeep-security` | `cedra-firmware-jeep-security/` | ESP32/PlatformIO firmware (Arduino framework). The hub of the project — controls relays/switches, reads the CCD bus, talks MQTT to Home Assistant, serves a local PWA webserver. Two hardware build envs: `esp32` (main DevKit) and `ext_hub` (ESP32-CAM). |
| `Michael-poortig/jeep-digital-cluster` | `Jeep-Digital-Cluster/jeep-digital-cluster/` (note: **nested** — the git root is the outer `Jeep-Digital-Cluster/` dir, the actual app is one level down) | React 19 + Vite dashboard ("digital cluster" gauge display) plus a Node/TypeScript `can-backend/` that bridges ESP32 serial telemetry to a WebSocket (for the dashboard) and to MQTT (for Home Assistant). Runs on an in-vehicle Debian mini-PC kiosk. |
| `Michael-poortig/jeep-control-panel` | `jeep-control-panel/` | Native Android/Kotlin/Jetpack Compose mobile app. MQTT client (HiveMQ lib) controlling the same relay set as the firmware, plus lock/unlock. Self-updates from GitHub releases. |
| `Michael-poortig/jeep-icons` | `jeep-icons/` | Pure asset repo — 14 full-resolution PNG icon source files (relay names as filenames), no code. Manually (not automatically) converted to `.webp` and copied into `jeep-control-panel`'s Android resources when icons change. |
| `Michael-poortig/Home-Assistant` | `Home-Assistant/` | The owner's general-purpose Home Assistant config repo (not Jeep-specific). Currently near-empty on disk — a real `/config` snapshot was accidentally committed once and deleted the next commit; Jeep-relevant `configuration.yaml`/`automations.yaml` content is recoverable via `git show 909448c:Files/configuration.yaml` and `git show 909448c:Files/automations.yaml`. |

## Data flow (how the repos talk to each other)

```
Jeep hardware (CCD bus, switches, sensors)
        |
        v
cedra-firmware-jeep-security (ESP32 main + ESP32-CAM ext_hub)
        |  MQTT (cedra/jeep/...)         |  UART0 JSON (gateway-mode bridge)
        v                                 v
Home Assistant  <---MQTT--->  jeep-digital-cluster/can-backend  --WebSocket-->  jeep-digital-cluster (React dashboard)
        ^
        | MQTT (phone sensors: GPS speed, weather, altitude, bearing, g-force, phone pitch/roll)
        |
   HA automations (phone companion app sensors)

jeep-control-panel (Android) <---MQTT (cedra/jeep/...)---> firmware directly (independent of HA/cluster)
```

`jeep-control-panel` and Home Assistant automations both talk to the
firmware's real MQTT namespace `cedra/jeep/...` directly.
`jeep-digital-cluster/can-backend` has its own, separate MQTT bridge that
uses a different topic base — see `mqtt-topics-and-sync.md` for the exact
mismatch and file:line citations.

## Current architecture state (as of this audit)

The main ESP32 DevKit board in the vehicle has a **confirmed hardware
defect** (brownout on WiFi radio init — root-caused, closed investigation,
see `troubleshooting-faq.md`). As a workaround, the firmware currently
runs in **gateway mode** (`USE_EXT_HUB_WIFI_GATEWAY=1` in
`cedra-firmware-jeep-security/src/config.h`): the ESP32-CAM (`ext_hub`)
owns all WiFi/MQTT/OTA, and the main board stays radio-dark, talking to
the CAM only over UART0 JSON. This is load-bearing, not incidental — don't
assume "classic mode" (flag=0) is active unless told the main board's
been physically replaced.

## Deployment / hosting notes

- MQTT broker reachability is Tailscale-only, no TLS (documented
  assumption across firmware, dashboard backend, and mobile app).
- The dashboard (`jeep-digital-cluster`) runs on an in-vehicle **Debian
  mini-PC** kiosk (Chromium), which **superseded** an earlier ChromeOS
  Chromebook plan as of 2026-07-05 — if you see Chromebook/ChromeOS/
  Crostini references in older docs, they're retired history, not current.
- `jeep-digital-cluster`'s owner-machine working copy (per its own
  `LAST-KNOWN-GOOD-CONFIG.md`) lives at a Windows path under a folder
  literally named `active(DO-NOT-PUSH-THIS)` and is a file-based backup,
  not a git checkout — don't be surprised if the owner references a
  Windows path that doesn't match this Linux checkout.
