# Release Notes

## v2.2.0 (2026-11-05)

**Bundled Controller Firmware:** `v1.0.261105_270`

### What's New Since v2.1.0
- **Automatic BLE Controller Firmware Version Check:** Compares the connected ESP32 controller's `Software_VER` against the bundled firmware package (`manifest.json`) and displays an update suggestion banner on the main screen and a badge in Settings when a newer build is available.
- **In-App Wi-Fi OTA Upgrade & Maintenance (`FirmwareUpgradeView` / `FirmwareUpgradeScreen`):**
  - **Full Upgrade (Recommended):** Migrates `settings.json`, uploads all 6 SPIFFS static Web UI files, flashes `firmware.bin`, and reboots/reconnects the controller.
  - **Firmware Only (`firmware.bin`):** Streams `firmware.bin` over Wi-Fi to `/doUpdate` with paced 1,436-byte chunks and live progress.
  - **SPIFFS Static Files Only:** Uploads all bundled HTML/CSS/JS Web UI files to `/doUpload` without modifying `settings.json`.
  - **Fix & Migrate `settings.json` Only:** Non-destructively merges the controller's `/spiffs/settings.json` with the bundled 18-key schema (keeping all user calibration, GPIO pins, Wi-Fi, hotspot, and access code settings while adding missing keys and removing deprecated keys).
- **OTA Reliability & Direct Wi-Fi Socket Binding:**
  - Automatically switches the controller to Simulation Mode prior to OTA uploads (and restores the original mode afterward) so ADC sensor sampling never interrupts flash writes.
  - Fixed a signed/unsigned timestamp comparison race in the ESP32 `loop()` (`ble/src/main.cpp`) during `/doUpdate`.
  - Binds HTTP sockets directly to the local Wi-Fi interface (`en0` on iOS, `TRANSPORT_WIFI` `Network` on Android) so upgrades work seamlessly on offline `192.168.4.1` SoftAP networks while mobile data is active.
- **Bilingual Localization:** Full English (`en`) and Hebrew (`he` / `iw`) support for all upgrade flows, dialogs, and status logs.
