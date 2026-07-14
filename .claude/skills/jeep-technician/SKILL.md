---
name: jeep-technician
description: Trigger this whenever the user asks about the Jeep ZJ "Vantabeast" project (also called CEDRA / Cedra Systems) or any of its five repos — cedra-firmware-jeep-security, jeep-digital-cluster, jeep-control-panel, jeep-icons, Home-Assistant. Covers hardware inventory and wiring/pinout lookups (ESP32 boards, relays, GPIO, MCP23017, MPU6050, CCD bus), configuration lookups (MQTT topics, platformio.ini, config.h, env vars, role-swap flags), troubleshooting and FAQs (boot loop, brownout, known bugs, build errors), build/test/run commands, and drafting a code-change request to hand off to a fresh Claude Code session working in one of these repos. Use for quick questions that don't require re-reading the full handover docs.
---

# Jeep Technician

Quick-reference skill for the Jeep ZJ "Vantabeast" security/accessory
system project. Answer from the reference files below rather than
re-deriving from source — they were built from a full audit of all 5
repos. If a question needs detail beyond what's here, say so and point to
the authoritative source doc named in the relevant reference file.

Read only the reference file(s) relevant to the question:

| File | Read this when the question is about... |
|---|---|
| `reference/repo-map.md` | "what repo is X in," how the 5 repos relate, where something lives on disk |
| `reference/hardware-inventory.md` | boards, GPIO pins, relays, sensors (IMU/expander), CCD bus, wiring, power |
| `reference/configuration.md` | MQTT broker/ports, env vars, `USE_EXT_HUB_WIFI_GATEWAY` and other build-time flags, where secrets live |
| `reference/mqtt-topics-and-sync.md` | MQTT topic names, Home Assistant entities, the Jeep↔ESP32↔HA↔Cluster signal sync table, the known topic-namespace mismatch |
| `reference/troubleshooting-faq.md` | "why does X happen," known bugs (fixed or open), the boot-loop/brownout postmortem, build errors |
| `reference/build-commands.md` | how to build/test/run any of the 5 repos |
| `reference/code-change-request-template.md` | drafting a request to hand to a different Claude Code session working in one of these repos |

## Ground rules

- **Never repeat actual secret values** (WiFi/MQTT/AP passwords, the
  Android keystore signing password) even though some are already
  committed to one repo's git history — see `configuration.md` for why,
  and never paste them into a drafted code-change request either.
- This project spans 5 repos that may not all be checked out in any given
  session — `repo-map.md` lists exact on-disk paths, but don't assume a
  repo is present; say so if you can't find it.
- Don't conflate this project with the sibling repo
  `Cedra-Client-Cluster-Firmware` — different client, different project.
- Content here reflects a point-in-time audit (see each file's "as of"
  note if present). If asked to make an actual code change, do the change
  in that repo directly — this skill is reference material, not a
  substitute for reading the real source when editing it.
