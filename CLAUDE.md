# Jeep Icons

Pure asset repo — 14 full-resolution PNG icon source files (filenames
match the relay names used across the Jeep ZJ "Vantabeast" project's MQTT
topics and code, e.g. `front_diff_lock.png`). No code, no build tooling.

Part of a 5-repo personal project; the firmware is
`cedra-firmware-jeep-security`. These icons are manually (not
automatically) converted to `.webp` and copied into
`jeep-control-panel/app/src/main/res/drawable-nodpi/relay_<name>.webp`
whenever they change — no script does this yet.

## Quick reference

For hardware inventory, configuration lookups, troubleshooting/FAQs, build
commands across all 5 repos of this project, or drafting a code-change
request for another Claude Code session, use the `jeep-technician` skill
(`.claude/skills/jeep-technician/SKILL.md`).
