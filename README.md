# Tires air pressure

One checkout for the controller and both phone apps. Each part keeps its own repository.

| Folder | What it is | Repository |
|---|---|---|
| `ble` | ESP32 controller | [AirUpDownControllerV2](https://github.com/nir-azulay/AirUpDownControllerV2) |
| `android` | Android app | [TiresAirPressure-Android](https://github.com/nir-azulay/TiresAirPressure-Android) |
| `ios` | iOS app | [TiresAirPressure-iOS](https://github.com/nir-azulay/TiresAirPressure-iOS) |

Clone everything:

```bash
git clone --recurse-submodules git@github.com:nir-azulay/TiresAirPressure.git
```

If the parent is already cloned:

```bash
git submodule update --init --recursive
```

Commit inside the folder you changed, then push that repository. Update the parent only when you want it to point at a new commit of a submodule.

Play internal releases run from the Android repository. TestFlight releases run from the iOS repository. Each one is scheduled for 06:00 UTC on the 1st of the month, and can also be started by hand from that repository's Actions tab. The controller is built in its own repository with PlatformIO.
