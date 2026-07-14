# Drafting a Code-Change Request for Another Claude Code Session

Use this when the user wants to hand off an actual code change to a fresh
Claude Code session working in one of the 5 repos (this skill itself
should not make the change — it's reference material, not an editor for
repos it's not currently open in).

## Template

```
Target repo: <exact repo name from repo-map.md, e.g. jeep-digital-cluster>
Branch: <existing feature branch if one is in flight, otherwise let the session create one>

Context:
<1-3 sentences: what problem this solves and why, written so a session with
zero prior context understands the motivation — not just the mechanical ask>

Requested change:
<concrete description of the change>

Relevant files (from this project's audit — verify against current source,
this may be stale):
<file paths, from repo-map.md / hardware-inventory.md / mqtt-topics-and-sync.md as applicable>

Cross-repo considerations:
<anything the other session should know that lives outside its own repo —
e.g. "the firmware's relay allowlist in gpio.h must stay in sync with this
change" or "this touches the cedra/jeep vs jeep MQTT topic-base question,
see mqtt-topics-and-sync.md in the jeep-technician skill">

Acceptance criteria:
<how to know the change is done — tests to run, build commands from
build-commands.md, manual verification steps>
```

## Rules when drafting

- **Never include actual secret values** (WiFi/MQTT/AP passwords, Android
  keystore password) in the draft, even as "here's the current value for
  reference." Point to the file they live in instead (see
  `configuration.md`). A drafted request is exactly the kind of artifact
  that gets pasted into a PR description or issue — don't let it become a
  new leak vector.
- If the change spans more than one repo (e.g. renaming a relay, which
  touches firmware `gpio.h` + `can-backend/relays.ts` + Android
  `RelayCatalog.kt`), say so explicitly and either draft one request per
  repo or make clear in a single request which repo is authoritative and
  which need to follow.
- If the change touches the known MQTT topic-namespace mismatch
  (`mqtt-topics-and-sync.md`), flag that it's an open question the owner
  hasn't resolved yet — don't have the other session "fix" it without the
  owner confirming which side is correct.
- If the change touches a pinned dependency (`platformio.ini`,
  `package.json`, Gradle version catalog, GitHub Actions versions),
  mention the dependency-freshness policy (pin explicitly, verify
  actively-maintained, don't float) so the receiving session applies it.
- Reference the actual build/test commands from `build-commands.md` in the
  acceptance criteria rather than inventing new ones.
- Keep the request self-contained — the receiving session has no memory of
  this conversation, only what's written here plus its own repo.
