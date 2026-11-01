# Tires Air Pressure Management System — DIY Technical Specification

**Project Type:** Open-Source DIY (Do-It-Yourself) Off-Road Hardware & Mobile/Automotive Software Suite  
**Version:** Firmware `v1.0.x` (`Software_VER = 2.00`) | iOS `v2.1.0` | Android `v2.1.0`  
**Author:** Nir Azulay  

---

## 1. System Overview & Pneumatic Architecture

The **Tires Air Pressure Management System** is an open-source **DIY** automated pneumatic closed-loop controller and multi-platform mobile/automotive application suite designed to inflate or deflate up to four vehicle tires simultaneously through a shared manifold using off-the-shelf components.

```text
                       ┌──────────────────────┐
   Air Tank / Source ──► Y1: Inflate Solenoid │
                       │  (12V NC, Relay 1)   ├──┐
                       └──────────────────────┘  │
                                                 ▼
                       ┌──────────────────────┐  ┌─────────────────────────────┐
   Atmosphere (Vent) ◄── Y2: Deflate Solenoid │◄─┤ Shared Pneumatic Manifold   ├──► 4 Tires
                       │  (12V NC, Relay 2)   │  │ (Equalized Shared Volume)   │
                       └──────────────────────┘  └──────────────┬──────────────┘
                                                                │
                                                                ▼
                                                 ┌─────────────────────────────┐
                                                 │ U3: 0.5–4.5V Pressure       │
                                                 │ Transducer (1/8" NPT)       │
                                                 └──────────────┬──────────────┘
                                                                │ 10kΩ / 10kΩ Divider + 100nF RC
                                                                ▼
                                                 ┌─────────────────────────────┐
                                                 │ ESP32 Controller (ADC1)     │
                                                 │ BLE GATT + Async Web Server │
                                                 └──────┬───────────────┬──────┘
                                           BLE 5.0 GATT │               │ Wi-Fi STA / SoftAP HTTP
                                        ┌───────────────┴───┐       ┌───┴───────────────────┐
                                        ▼                   ▼       ▼                       ▼
                                  iOS & CarPlay     Android & Auto  Web UI (/configurations) OTA (/update)
```

### Core Operating Principle
Because open air flows across the manifold and tire hoses when a solenoid valve (`Y1` or `Y2`) is energized, the manifold pressure during active airflow does not equal static tire pressure. The controller therefore executes a **Pulse-Wait-Measure** cycle:
1. Open the appropriate valve (`Inflate` or `Deflate`) for a dynamically scaled pulse interval (`MaxSecForStatus` scaled by the relative distance from `targetPressure`).
2. Close both valves (`StopAir()`).
3. Wait `MessureWait` seconds for manifold and tire pressures to equalize.
4. Sample the analog pressure sensor using a 30-sample trimmed mean filter, requiring consecutive readings to stabilize within `3.0 PSI` (`kStablePsi`).
5. Repeat until `|targetPressure - currentPressure| <= Precision`, then close both valves and broadcast a `target_reached` BLE notification.

---

## 2. Hardware & Electrical Specification

### 2.1 Microcontroller & Relay Board
- **Target Board:** `ESP32 Relay X2` (`esp32dev`, ESP32-D0WD-V3 @ 240 MHz, 320 KB SRAM, 4 MB Flash).
- **Power Supply:** `12 V DC` (minimum `2 A`, protected by a `2 A` inline fuse) connected to board **VCC** and both relay `COM` terminals.
- **5 V Rail:** Onboard regulator output (`5 V` pin) powers the ratiometric pressure transducer.
- **Flyback Protection:** `1N4007` diode across each 12 V solenoid coil (cathode/band toward relay `NO`).

### 2.2 Configurable GPIO Pin Assignments
All hardware pins are software-configurable and validated on boot:

| Logical Function | Setting Key | Default GPIO | Raptor-Air-MNG (`155`) | Valid Range | Notes |
|---|---|---|---|---|---|
| **Inflate Valve (`Y1`)** | `RelayCompressorPin` | `GPIO 16` | `GPIO 17` | `1..39` | Active `HIGH` (`LOW` at boot & stop) |
| **Deflate Valve (`Y2`)** | `RelayFreePin` | `GPIO 17` | `GPIO 16` | `1..39` (`!= RelayCompressorPin`) | Active `HIGH` (`LOW` at boot & stop) |
| **Pressure Sensor (`U3`)** | `SensorPin` | `GPIO 34` | `GPIO 32` | `32..39` (ADC1 only) | ADC2 pins (`<32`) are auto-migrated to `GPIO 34` because Wi-Fi disables ADC2 |

### 2.3 Analog Front-End & Signal Conditioning
- **Transducer Output:** `0.5 V` at `0 PSI` to `4.5 V` at `SensorFullScalePSI` (typically `50 PSI` or `100 PSI`).
- **Voltage Divider (`R5 = 10 kΩ`, `R6 = 10 kΩ`, `C3 = 100 nF`):** Scales the `0.5–4.5 V` sensor signal by `0.5×` to `0.25–2.25 V` at the ESP32 `ADC1` input (`12-bit`, `0–4095` counts over `0–3.3 V`).
- **Digital Filtering (`readPressureFromSensor()`):**
  - Collects `NUM_SAMPLES = 30` ADC samples (`analogRead(settings.SensorPin)`) with a `2 ms` inter-sample delay (aborting early if an OTA update, settings save, or reboot is pending).
  - Sorts the 30 samples in ascending order and computes the arithmetic mean of the middle 30% (`indices 10..18`, i.e., 35% to 65% trimmed mean) to reject transient spikes and EMI noise.

### 2.4 Pressure Calculation Methods
The controller supports two calibration formulas selected by `PressureCalcMethod`:

#### Method 1: Raw ADC Linear Interpolation (`PressureCalcMethod = 1`)
Uses `SensorValueAt0PSI` ($ADC_0$) and `SensorValueAt50PSI` ($ADC_{FS}$) as raw 12-bit ADC counts (`0–4095`) across `SensorFullScalePSI` ($P_{FS}$):
- Fault check: Invalid if $P_{FS} \le 0$, $ADC_{FS} = ADC_0$, or $\text{sensorVal} + 150 < ADC_0$.
- Formula:
  \[
  P_{\text{PSI}} = \frac{\text{sensorVal} - ADC_0}{(ADC_{FS} - ADC_0) / P_{FS}}
  \]

#### Method 2: Voltage Divider Calibration (`PressureCalcMethod = 2`)
Uses `SensorValueAt0PSI` ($V_0$) and `SensorValueAt50PSI` ($V_{FS}$) as voltages (`0.0–3.3 V`) measured at the ESP32 ADC pin after the 2:1 divider:
- ADC pin voltage: $V_{\text{ADC}} = \text{sensorVal} \times \frac{3.3}{4095.0}$
- Sensor wire voltage: $V_{\text{sensor}} = 2.0 \times V_{\text{ADC}}$
- Fault check: Invalid if $P_{FS} \le 0$, $V_{FS} = V_0$, $V_0 \notin [0, 3.3]$, $V_{FS} \notin [0, 3.3]$, or $V_{\text{ADC}} < 0.05\text{ V}$ (disconnected sensor wire).
- Formula:
  \[
  P_{\text{PSI}} = \left(V_{\text{sensor}} - 2 V_0\right) \times \frac{P_{FS}}{2 (V_{FS} - V_0)} = \left(V_{\text{ADC}} - V_0\right) \times \frac{P_{FS}}{V_{FS} - V_0}
  \]

#### Clamping & Over-Range Check (`applyPressureLimits()`)
- Negative values near zero are clamped to `0.0 PSI`.
- If $P_{\text{PSI}} > 1.05 \times P_{FS}$, `pressureValid` is set to `false`.

---

## 3. Closed-Loop Control & Safety State Machine

### 3.1 Operation Modes
- `1` (`Stop`): Both relays `LOW`. Idle monitoring loop reads pressure every `100` ticks (`~1.0 s`) without pulsing valves.
- `2` (`Inflate`): `RelayCompressorPin = HIGH`, `RelayFreePin = LOW` (in `SimulationMode`, both physical relays stay `LOW` while simulated pressure increments by `+0.5 PSI` per cycle).
- `3` (`Deflate`): `RelayCompressorPin = LOW`, `RelayFreePin = HIGH` (in `SimulationMode`, both physical relays stay `LOW` while simulated pressure decrements by `-0.3 PSI` per cycle).

### 3.2 Dynamic Pulse Timing (`calculateDelatFactor()`)
During active inflation or deflation, the valve remains open for a number of `10 ms` loop ticks proportional to the remaining pressure error:
\[
\text{Ticks} = \left\lfloor 100 \times \text{MaxSecForStatus} \times \frac{|P_{\text{target}} - P_{\text{current}}|}{\max(P_{\text{target}}, P_{\text{current}})} \right\rfloor
\]
As `currentPressure` approaches `targetPressure`, the valve pulse duration automatically shortens to prevent overshoot.

### 3.3 Settling & Stability Verification (`readCurrentPressure()`)
When `valveSessionActive == true` (active inflate/deflate session):
1. `StopAir()` closes both valves.
2. `waitForPressureToSettle()` waits `MessureWait * 1000` ms.
3. Reads initial `currentPressure`, then performs up to `maxStableReads = 5` additional settle-and-read cycles until two consecutive readings agree within `kStablePsi = 3.0 PSI`.
4. If the reading does not stabilize within 5 attempts, `refuseAir("pressure reading is unstable")` aborts the session and closes both valves.

### 3.4 Safety Interlocks & Watchdogs (`OperateAir()` & `handleStartCommand()`)
The controller enforces six independent hardware/software safety guards:
1. **Unconfigured Sensor Guard:** If `SensorFullScalePSI <= 0` (and not in `SimulationMode`), valves will not open (`refuseAir("sensor full scale is not set")`).
2. **Target Range Guard:** `handleStartCommand` rejects any target where `targetPressure <= 0` or `targetPressure > min(MaxPressurePSI, SensorFullScalePSI)`.
3. **Sensor Fault Guard:** Any invalid reading (`pressureValid == false`, e.g., open circuit `< 0.05 V` or over-scale `> 1.05 × SensorFullScalePSI`) immediately closes both valves (`refuseAir("pressure reading is not usable")`).
4. **Maximum Valve Open Timeout:** If `millis() - valveSessionStartMs >= MaxValveOpenSec * 1000` (default `120 s`), both valves close (`refuseAir("valve open time exceeded")`).
5. **Stall / Leak Watchdog:** Every `kNoProgressMs = 60,000 ms` (`60 s`), the controller checks if pressure has moved at least `kMinProgressPsi = 0.5 PSI` toward `targetPressure`. If not (e.g., compressor off, detached hose, or empty tank), it aborts (`refuseAir("pressure is not changing")`).
6. **Over-Pressure Protection:** If `currentPressure > MaxPressurePSI`, the controller refuses inflation and only permits `DeflateAir()` if `targetPressure + Precision < currentPressure`.

---

## 4. Persistent Configuration Schema (`/spiffs/settings.json`)

Settings are stored on SPIFFS in `/settings.json` and validated at boot and on save by `normalizeSettings()` using a dedicated `StaticJsonDocument<2048> settingsJsonDoc` in `.bss` RAM (avoiding heap fragmentation). Unknown or deprecated keys (`BTPinCode`, `BLESecurityEnabled`, `AutoStart`) are automatically stripped, and missing keys are populated with defaults.

| # | JSON Key | Type | Default | Valid Range / Auto-Repair Rule | Description |
|---|---|---|---|---|---|
| 1 | `DeviceName` | `string` | `"esp32-AirUpDownController"` | `1..31` chars, no `;` | BLE advertising name & SoftAP SSID |
| 2 | `wifissid` | `string` | `""` | `0..64` chars, no `;` | Home/vehicle 2.4 GHz Wi-Fi SSID |
| 3 | `wifipassword` | `string` | `""` | `0..64` chars, no `;` | Home/vehicle Wi-Fi password |
| 4 | `RelayCompressorPin` | `int` | `16` | `1..39` | GPIO driving Inflate relay (`Y1`) |
| 5 | `RelayFreePin` | `int` | `17` | `1..39` (`!= RelayCompressorPin`) | GPIO driving Deflate relay (`Y2`) |
| 6 | `SensorPin` | `int` | `34` | `32..39` (ADC1 only) | GPIO reading divided pressure sensor |
| 7 | `MaxSecForStatus` | `int` | `5` | `1..300` sec | Base time constant for valve pulse scaling |
| 8 | `SensorValueAt0PSI` | `float` | `310.227` | Method 1: `0..4095`, Method 2: `0.0..3.3` | Calibration value at `0 PSI` |
| 9 | `SensorValueAt50PSI` | `float` | `2792.045` | Method 1: `0..4095`, Method 2: `0.0..3.3` | Calibration value at `SensorFullScalePSI` |
| 10 | `AccessCode` | `string` | `""` | `0..32` chars, no `;` | Optional application-layer BLE access code |
| 11 | `MessureWait` | `double` | `2.0` | `0.1..30.0` sec | Settling wait time after closing valves |
| 12 | `Precision` | `double` | `0.5` | `0.05..10.0` PSI | Target deadband (`±Precision` PSI) |
| 13 | `SimulationMode` | `bool` | `false` | `true` / `false` | Enables synthetic pressure simulation |
| 14 | `PressureCalcMethod` | `int` | `1` | `1` (ADC) or `2` (Voltage) | Pressure calculation formula |
| 15 | `MaxPressurePSI` | `float` | `50.0` | `1.0..300.0` PSI | Maximum allowed target pressure |
| 16 | `MaxValveOpenSec` | `int` | `120` | `10..3600` sec | Watchdog timeout for a single session |
| 17 | `SensorFullScalePSI` | `float` | `50.0` | `0.0..500.0` PSI | Full-scale PSI stamped on transducer |
| 18 | `AccessPointPassword` | `string` | `"12345678"` | `8..63` chars, no `;` | WPA2 password for fallback SoftAP |

---

## 5. BLE GATT Protocol Specification

### 5.1 GATT Profile & Transport Layer
- **Service UUID:** `4fafc201-1fb5-459e-8fcc-c5c9c331914b`
- **Notification Characteristic (`TX` from ESP32):** `beb5483e-36e1-4688-b7f5-ea07361b26a8` (`NOTIFY`, CCCD `0x2902`)
- **Command Characteristic (`RX` to ESP32):** `e3223119-9445-4e96-a4a1-85358c4046a2` (`READ | WRITE`)
- **Requested MTU:** `517` bytes
- **Bidirectional Chunking Protocol (`180` bytes):**
  - Because default BLE ATT payloads on some mobile stacks or unnegotiated links are limited to `< 200` bytes while full settings payloads are `~430` bytes, both directions implement `180`-byte chunking:
  - **Mobile → ESP32 (`Command Characteristic`):** Payloads `> 180` bytes are split into `180`-byte slices written with `25 ms` pacing. `MyCharacteristicCallbacks::onWrite` appends incoming chunks into `bleRxBuffer` and only invokes `handleBLEMessage()` once brace depth (`{` vs `}`) balances to `0`.
  - **ESP32 → Mobile (`Notification Characteristic`):** `notifyBLEPayload()` splits outgoing JSON strings `> 180` bytes into `180`-byte notifications with `15 ms` inter-chunk delay. Mobile clients accumulate chunks until a complete JSON object (`{...}`) is formed.

### 5.2 ESP32 → Mobile Notifications

1. **Pressure Status Update:**
   ```json
   {
     "Pressure": "28.5",
     "operationMode": "1",
     "pressureValid": true,
     "rawADC": 1420,
     "voltage": 1.144
   }
   ```
   *(Note: `rawADC` and `voltage` are included when `debugMode` is enabled via Command `7`.)*

2. **Full Settings Notification (`operationMode: 4`):**
   ```json
   {
     "operationMode": 4,
     "DeviceName": "Raptor-Air-MNG",
     "wifissid": "MyNetwork",
     "wifipassword": "MyPassword",
     "RelayCompressorPin": 17,
     "RelayFreePin": 16,
     "SensorPin": 32,
     "MaxSecForStatus": 30,
     "SensorValueAt0PSI": 0.321,
     "SensorValueAt50PSI": 3.140,
     "SensorFullScalePSI": 50.0,
     "MaxPressurePSI": 50.0,
     "MaxValveOpenSec": 120,
     "AccessPointPassword": "12345678",
     "HasAccessCode": false,
     "MessureWait": 1.00,
     "Precision": 0.30,
     "CurrentPressure": 28.5,
     "Software_VER": "v1.0.261101_256",
     "IP": "192.168.10.155",
     "SimulationMode": false,
     "PressureCalcMethod": 2
   }
   ```

3. **Target Reached Notification:**
   ```json
   {
     "type": "target_reached",
     "targetPressure": 32.0,
     "currentPressure": 32.0
   }
   ```

4. **Authentication Status Notification:**
   ```json
   {
     "type": "auth",
     "ok": true,
     "required": false
   }
   ```

5. **Wi-Fi Scan Results Notification:**
   ```json
   {
     "type": "wifi_list",
     "networks": "HomeWiFi|-48;GarageAP|-67;TrailHotspot|-79"
   }
   ```
   *(While scanning is in progress, sends `{"type":"wifi_list","scanning":true}` and automatically pushes the completed `networks` list when `WiFi.scanComplete()` finishes on the main loop).*

### 5.3 Mobile → ESP32 Commands

| Command ID | Name | JSON Payload | Behavior |
|---|---|---|---|
| `1` | **Stop** | `{"command": 1}` | Always accepted (even before auth). Increments `valveCommandEpoch`, sets `valveSessionActive = false`, closes both valves (`StopAir()`). |
| `2` | **Start / Set Target** | `{"command": 2, "targetPressure": 32.0}` | Validates target against `MaxPressurePSI` and `SensorFullScalePSI`, resets watchdog timers, starts closed-loop inflation/deflation. |
| `3` | **Update Settings** | `{"command": 3, "DeviceName": "...", ...}` | Updates any subset of the 18 settings fields (supports `"ClearAccessCode": true` to remove an access code), normalizes, saves `/settings.json` on `loop()`, and reboots after `1.8 s`. |
| `4` | **Simulation Mode** | `{"command": 4, "enabled": true}` | Toggles `SimulationMode`, persists to `/settings.json`, and broadcasts updated settings. |
| `5` | **Log Level** | `{"command": 5, "level": 3}` | Sets runtime serial log verbosity (`0=NONE` .. `5=VERBOSE`). |
| `6` | **Get Settings** | `{"command": 6}` | Triggers an immediate `operationMode: 4` full settings notification. |
| `7` | **Debug Mode** | `{"command": 7, "enabled": true}` | Toggles inclusion of `rawADC` and `voltage` in status notifications and sends an immediate status update. |
| `8` | **Authenticate** | `{"command": 8, "accessCode": "1234"}` | Verifies `accessCode` against `settings.AccessCode`. Replies with `{"type":"auth","ok":...}` and sends settings upon success. |
| `9` | **Scan Wi-Fi** | `{"command": 9, "rescan": true}` | Triggers an asynchronous 2.4 GHz Wi-Fi scan on the ESP32 (`pendingStartWifiScan`) and pushes a `wifi_list` notification when complete. |

---

## 6. Wi-Fi & Embedded HTTP Server Specification

### 6.1 Wi-Fi Connection & SoftAP Fallback (`setupWiFi()`)
1. Configures `WIFI_AP_STA` mode so the fallback SoftAP or STA connection can operate seamlessly.
2. If `settings.wifissid` is non-empty, attempts up to `5` connection retries (`10` polls × `500 ms` per retry, resetting the radio on retry 3).
3. If STA connects, synchronizes UTC+2 time via `NTPClient` (`pool.ntp.org`).
4. If STA fails or `wifissid` is blank, starts a WPA2 SoftAP with SSID `settings.DeviceName` and password `settings.AccessPointPassword` (falling back to `"12345678"` if `< 8` chars) on IP `192.168.4.1`.

### 6.2 HTTP Endpoints (`ESPAsyncWebServer` on Port 80)

| Endpoint | Method | Content-Type | Description |
|---|---|---|---|
| `/` | `GET` | `text/html` | Serves `/index.html` (live pressure gauge, target slider, Inflate/Deflate/Stop controls). |
| `/configurations` | `GET` | `text/html` | Serves `/configurations.html` (`Cache-Control: no-store`), full 18-field configuration UI with Wi-Fi scanner and password Show/Hide toggles. |
| `/update` | `GET` | `text/html` | Serves `/update.html` (OTA `.bin` firmware upload & SPIFFS static file uploader with progress bar). |
| `/fs` | `GET` | `text/html` | Serves `/fs.html` (SPIFFS directory browser with file links). |
| `/about` | `GET` | `text/html` | Serves `/about.html` (system & developer profile page with embedded `/profile.jpg`). |
| `/xml` | `GET` | `text/xml` | Returns live `<xml><currentPressure>...</currentPressure><targetPressure>...</targetPressure><operationMode>...</operationMode></xml>`. |
| `/json` | `GET` | `text/plain` | Returns current settings and state as JSON-like payload. |
| `/settingsValues` | `GET` | `text/plain` | Returns semicolon-delimited 20-field string consumed by `/configurations.html` (`DeviceName;wifissid;wifipassword;RelayCompressorPin;RelayFreePin;SensorPin;SimulationMode;MaxSecForStatus;SensorValueAt0PSI;SensorValueAt50PSI;AccessCode;MessureWait;Precision;PressureCalcMethod;0;VERSION_SHORT;MaxPressurePSI;MaxValveOpenSec;SensorFullScalePSI;AccessPointPassword`). |
| `/scanWifi` | `GET` | `application/json` | Deduplicates and returns nearby 2.4 GHz networks sorted by RSSI (`[{"ssid":"...","rssi":-52,"secure":true},...]`). Supports `?rescan=1`. |
| `/setTargetPressure` | `GET` | `text/plain` | Sets target pressure (`?value=<PSI>`) and starts inflation/deflation. |
| `/stop` | `GET` | `text/plain` | Immediately stops airflow (`StopAir()`). |
| `/saveConfigurations` | `POST` | `text/html` | Parses semicolon-delimited `values` form field, normalizes settings, schedules SPIFFS save (`pendingSaveSettings = true`) and reboot (`pendingRestart = true`), and returns an auto-redirecting Rebooting page. |
| `/savesettings` | `GET/POST` | `text/html` | Updates individual settings parameters by query/form name, saves to `/settings.json`, and reboots. |
| `/doUpdate` | `POST` | `multipart/form-data` | Streams firmware `.bin` or SPIFFS image into `Update.write()`, sets `upgradeInProgress = true` throughout transfer, and reboots upon completion. |
| `/doUpload` | `POST` | `multipart/form-data` | Unlinks target path at `index == 0` and writes uploaded static file to `/spiffs/<filename>`. |
| `/list` | `GET` | `text/plain` | Returns `;`-separated list of files and sizes in `/spiffs`. |
| `/restart` | `GET` | `text/plain` | Schedules a non-blocking device reboot after `500 ms`. |

---

## 7. Mobile & Automotive Applications (`iOS` & `Android`)

### 7.1 Shared Feature Set (`v2.1.0`)
- **Off-Road Dark Theme & 270° Speedometer Gauge:** High-contrast dark UI (`#101318` / `#181C24`) with a custom 270° circular pressure gauge, orange preset chips (`Road`, `Trail`, `Sand`, `Rock`, plus custom presets), and an animated **Target Reached Celebration** overlay with sound and haptic feedback.
- **Multi-Device Auto-Connect:** Stores the 2 most recently connected controllers (`savedDevices`) with per-device `accessCode`, automatically promoting the latest connected controller to priority `0` and scanning/reconnecting in the background on launch.
- **Full Controller Configuration over BLE:** Reads and writes all 18 controller parameters over chunked BLE (`180`-byte chunks), including pin assignments, calibration method & values, safety limits, hotspot password, BLE access code, password visibility toggles, and live ESP32 Wi-Fi network scanning (`command: 9`).
- **Bilingual Localization (`en` / `he`):** Runtime language switching between English (LTR) and Hebrew (RTL) with 100% string key parity.

### 7.2 Automotive Integration
- **Apple CarPlay (`ios/Features/CarPlay/CarPlaySceneDelegate.swift`):** Uses `CPListTemplate` / `CPInformationTemplate` to display live manifold pressure, operation state, preset target triggers, and an emergency Stop button on the vehicle head unit.
- **Android Auto (`android/.../auto/TiresPressureCarAppService.kt` & `MainCarScreen.kt`):** Uses Jetpack ` androidx.car.app` (`PaneTemplate`) to render live pressure, operation status, quick presets, and Stop controls on the Android Auto display.

### 7.3 CI/CD & Release Automation
- **iOS (`nir-azulay/TiresAirPressure-iOS`):** `.github/workflows/ios-testflight.yml` runs `fastlane beta` on `macos-15` (`Xcode 16.4`), auto-incrementing `CFBundleVersion` (`BUILD_NUMBER_BASE + GITHUB_RUN_NUMBER`) and uploading to App Store Connect / TestFlight.
- **Android (`nir-azulay/TiresAirPressure-Android`):** `.github/workflows/android-play.yml` builds a signed release `.aab` with JDK 17 and uploads to the Google Play Console `internal` testing track via `r0adkll/upload-google-play`.
