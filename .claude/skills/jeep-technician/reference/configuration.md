# Configuration Reference — Jeep ZJ "Vantabeast"

## Credentials policy — read this first

WiFi/MQTT/AP passwords are already committed in plaintext to
`cedra-firmware-jeep-security/LAST-KNOWN-GOOD-CONFIG.md`'s git history (a
known, deliberately-deferred issue — "rotate credentials + purge history"
is still open). `jeep-control-panel` separately has a hardcoded Android
keystore signing password in `app/build.gradle.kts` (lower stakes per that
repo's own comment — it's not a credential for any external service, just
a local signing key — but treated the same way here for consistency).

**This file and any code-change request drafted with this skill's help
must never repeat those actual values.** If you need them, go read the
files above directly in that repo. Point to *where* they live; don't copy
them elsewhere — every additional copy is a new place a future leak has to
be found and rotated.

## cedra-firmware-jeep-security — `src/config.h` (Main ESP32)

Non-secret settings (secrets are `WIFI_SSID`/`WIFI_PASS`/`MQTT_USER`/
`MQTT_PASS`/`AP_PASSWORD_AP`, sourced from gitignored `src/secrets.h`,
generated from GitHub Actions repo secrets in CI):

| Setting | Value |
|---|---|
| `FW_VERSION` | `"1.7.8"` |
| `USE_EXT_HUB_WIFI_GATEWAY` | `1` (gateway mode — see below) |
| `MQTT_SERVER` | `100.78.144.7` (Tailscale IP) |
| `MQTT_PORT` | `1883` |
| `MQTT_CLIENT_ID` | `"vantabeast_esp32"` |
| `MQTT_BASE` | `"cedra/jeep"` |
| `HA_DISCOVERY_PREFIX` | `"homeassistant"` |
| `AP_SSID` | `"The Vantabeast ZJ"` |
| `AP_CHANNEL` / `AP_HIDDEN` / `AP_AUTO_OFF_MIN` | `1` / `true` / `10` |
| `OTA_GITHUB_REPO` | `"Michael-poortig/cedra-firmware-jeep-security"` |
| `MDNS_HOSTNAME` | `"jeep"` |
| `I2C_SDA_PIN` / `I2C_SCL_PIN` | `21` / `22` |
| `EXP_I2C_ADDR` | `0x20` |
| `WDT_TIMEOUT_SEC` | `10` |
| `VBATT_PIN` / `VBATT_DIVIDER` | `34` / `4.0f` |
| `CCD_RX_PIN` / `CCD_TX_PIN` / `CCD_BAUD` | `36` / `0` / `7812` |
| `SAVE_DEBOUNCE_MS` | `1000UL` |

**`USE_EXT_HUB_WIFI_GATEWAY`** is the one-line CAM⇄Main role swap:
- `1` (current) = **gateway mode**: main board's radio is hardware-faulty,
  so the ESP32-CAM owns WiFi/AP/MQTT/telnet/OTA; main stays radio-dark,
  exchanges newline-delimited JSON with the CAM over UART0.
- `0` = **classic mode**: main owns WiFi directly; CAM reverts to a plain
  ESP-NOW gear-switch sender. Only usable once a replacement main board
  with a working radio is installed.
- Flipping requires **rebuilding AND reflashing both boards** — each
  personality only speaks its own wire protocol.

## `src/ext_hub_config.h` (ESP32-CAM)

| Setting | Value |
|---|---|
| `EXT_MQTT_CLIENT_ID` | `"vantabeast_esp32cam_gateway"` |
| `EXT_RELAY_HOST` | `"192.168.137.1"` (laptop `can-backend`, Windows ICS default gateway) |
| `EXT_RELAY_PORT` | `8090` |
| `EXT_MQTT_SERVER` | same as `EXT_RELAY_HOST` — CAM can't reach the Tailscale MQTT server directly from the hotspot subnet, so it's proxied through the laptop |
| `EXT_MQTT_PORT` | `1884` (a local mqtt-proxy on the laptop, **not** `1883`) |
| `EXT_AP_SSID` | `"Vantabeast-CAM-Gateway"` |
| `EXT_AP_CHANNEL` | `6` |
| `EXT_WIFI_TX_POWER` | `WIFI_POWER_19_5dBm` (full power — do not reuse Main's brownout-mitigation power cap here) |
| `EXT_TELNET_PORT` | `23` |

## `platformio.ini` (all envs)

```ini
[env:esp32]      platform = espressif32@7.0.1, board = esp32dev
[env:ext_hub]     platform = espressif32@7.0.1, board = esp32cam, -DBOARD_HAS_PSRAM=0
[env:native]      platform = native (host-side Unity tests, CI only)
```
Key `lib_deps`: `knolleary/PubSubClient@^2.8`, `bblanchon/ArduinoJson@^7.0`,
`esp32async/AsyncTCP@^3.4.0` + `esp32async/ESPAsyncWebServer@^3.11.0`
(esp32 env only — the actively-maintained forks that replaced the
unmaintained `me-no-dev` originals in v1.7.4; see troubleshooting-faq.md).
**Versions are pinned deliberately** — see the dependency-freshness skill
and CLAUDE.md for why floating them caused a real incident (v1.7.3).

## jeep-digital-cluster / `can-backend` env vars

| Var | Default | Notes |
|---|---|---|
| `CAN_MODE` | `esp32` | Also: `socketcan`, `serial`, `simulator` |
| `CAN_SERIAL_PORT` | `COM6` (Win) / `/dev/ttyUSB0` (Linux) | |
| `CAN_SERIAL_BAUD` | `115200` | |
| `WS_PORT` / `WS_HOST` | `8080` / `127.0.0.1` | **No auth layer** — documented as Tailscale-only, never expose to LAN/public internet without adding auth |
| `MQTT_BROKER` / `MQTT_USER` / `MQTT_PASS` | none (required) | Backend **fails fast** (`process.exit(1)`) if `MQTT_BRIDGE != 'false'` and any of these are unset — no hardcoded fallback (a previously-fixed vulnerability) |
| `PERSIST_DIR` / `BLACKBOX_DIR` | — | Durable state / USB-stick drive logger, optional |

MQTT topic base here is hardcoded `'jeep'` in `can-backend/src/server.ts`
— **not** `cedra/jeep`. See `mqtt-topics-and-sync.md` for why that's a
flagged, unresolved cross-repo question.

## jeep-control-panel (Android)

No build-time config file — broker host/port/username/password are
entered via in-app Settings and stored in `EncryptedSharedPreferences`
(file `jeep_secure_prefs`). That API was deprecated by Google in April
2025 but is "still published and functional" — the repo owner deliberately
deferred migrating to DataStore+Tink for a personal sideloaded app; revisit
if AndroidX actually removes it rather than just deprecating it. MQTT
client ID is a persisted per-install UUID
(`jeep-control-panel-<UUID>`) — deliberately distinct from the firmware's
`vantabeast_esp32` client ID to avoid a broker-kick collision.

Signing: `signing/jeep-panel.keystore` is checked into the repo on purpose
(so every build/update installs over the last one identically) — alias
`jeeppanel`, password hardcoded in `app/build.gradle.kts` (see credentials
policy above — don't repeat the value here either).
