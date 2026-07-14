# Build / Test / Run Commands

## cedra-firmware-jeep-security (ESP32/PlatformIO)

```
pio test -e native                              # Unity host-side tests (294 cases across 19 suites)
pio run -e esp32                                # Main ESP32 DevKit build
pio run -e ext_hub                              # ESP32-CAM build
pio run -e esp32 --target upload --upload-port COM6
pio run -e ext_hub --target upload --upload-port COM7
```
Run all three (`native`, `esp32`, `ext_hub`) before committing — see this
repo's own CLAUDE.md. If `pio` on PATH errors with
`ModuleNotFoundError: No module named 'requirements'`, it's a broken shim
— use `<platformio-home>/penv/Scripts/pio` instead.

CI (`.github/workflows/firmware-ci.yml`): `test` job runs
`pio test -e native`; `build` job builds only the `esp32` matrix entry
(not `ext_hub`) and produces a merged `firmware-full.bin`; `release` job
(main branch only) cuts a GitHub Release.

## jeep-digital-cluster (nested at `Jeep-Digital-Cluster/jeep-digital-cluster/`)

Frontend:
```
npm run dev              # vite dev server, http://localhost:3000
npm run build             # vite build
npm run preview            # vite preview
npm test / npm run test:watch   # vitest run / vitest
npm run ota:fetch          # pull latest private-GitHub-release OTA asset (needs GITHUB_OWNER/GITHUB_REPO/GITHUB_TOKEN)
```
Backend (`can-backend/`):
```
npm run dev:sim            # CAN_MODE=simulator — no hardware needed
npm run dev                 # CAN_MODE=esp32 — real hardware over serial
npm run build                # tsc
npm start / start:socketcan / start:serial / start:sim   # compiled dist/ variants
npm test / npm run test:watch
```
Typical local dev: Terminal 1 `cd can-backend && npm install && npm run dev:sim`,
Terminal 2 `npm install && npm run dev` (frontend), then point Chromium at
`http://localhost:3000`.

Baseline test counts at last audit: backend 180/180, frontend 52/52.

## jeep-control-panel (Android/Gradle)

```
./gradlew :app:testDebugUnitTest :app:lintDebug --stacktrace   # unit tests + lint (what CI runs)
./gradlew assembleDebug
./gradlew assembleRelease
./gradlew assembleDebug -PversionCodeOverride=<N>               # pin versionCode (CI uses $GITHUB_RUN_NUMBER)
```
No instrumented (`androidTest`) run is wired into CI. CI
(`.github/workflows/android-ci.yml`): `test` → `build` (matrix
debug/release) → `release` (master push only, tags
`v<versionName>-b<run_number>`, creates a GitHub Release with the APK).

## jeep-icons

No build tooling — pure asset repo. Updating an icon means manually
converting the source PNG to `.webp` and copying it into
`jeep-control-panel/app/src/main/res/drawable-nodpi/relay_<name>.webp`
(prefix `relay_`, filenames otherwise match). No script automates this.

## Home-Assistant

Config-only repo, currently near-empty on disk. To recover the real
Jeep-relevant config: `git show 909448c:Files/configuration.yaml` and
`git show 909448c:Files/automations.yaml` (that commit's `Files/` dir also
contains a bunch of non-Jeep HACS-integration source and a runtime DB that
shouldn't be re-added — cherry-pick just the two YAML files' Jeep-relevant
content if reconstituting this repo).
