# Release Notes

## v2.2.1 (2026-11-06)

**Bundled Controller Firmware:** `v2.2.1_280`

### What's New Since v2.2.0
- **Synced Versioning:** ESP32 controller firmware now carries the same version as the companion apps (`v2.2.1`, build `280`) instead of a separate date-based number, and is packaged into both apps with `ble/scripts/package_for_ios.py`.
- **OTA Root-Cause Fix (iOS & Android):** Firmware uploads are now paced at flash-erase cadence — one 64 KB burst, then a ~650 ms pause while the controller erases that flash block. Flooding through the erase window lost ACKs on the 2.4 GHz link and TCP retransmission backoff stalled transfers until they timed out; paced uploads now flash the full ~1.7 MB in ~45 s. Also: priming request for controllers on older firmware, CR-boundary-safe multipart chunking, no settings reboot during the simulation-mode guard, preference for the live LAN address over a stale SoftAP address, keep-screen-on while flashing, and a hard error if the controller is still on the old firmware after reboot instead of a false success.
- **Android BLE Stability:** Reassembly of multi-chunk BLE settings notifications, singleton BLE manager/repository so settings survive navigation, and settings load/save fixes.
- **Android Settings Keyboard:** Save/Discard bar stays above the keyboard, keyboard dismisses when tapping outside a field, and fields use a Done action.
- **iOS Improvements:** Adjustable app font size (SettingsManager `AppFontSize`) applied across main and settings screens, with English/Hebrew strings and tests.

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
