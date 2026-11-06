---
name: release-manager
description: >-
  Manages store releases for the Tires Air Pressure system across iOS
  (TestFlight), Android (Google Play Console internal track), and ESP32 BLE
  firmware. Use whenever the user asks to publish, release, deploy, or upload
  to TestFlight or Google Play Console, bump release versions, or prepare
  release notes.
---

# Release Manager Skill

Follow this exact checklist whenever the user asks to publish or release to **TestFlight** or **Google Play Console**.

## Release Workflow Checklist

```
Release Progress:
- [ ] Step 1: Package latest ESP32 firmware (if ble/ changed)
- [ ] Step 2: Collect changes since the last TestFlight / Play Store release
- [ ] Step 3: Promote the patch version (X.Y.Z -> X.Y.(Z+1))
- [ ] Step 4: Update release notes with all changes since the last release
- [ ] Step 5: Commit and push submodules + parent repo to origin/main
- [ ] Step 6: Trigger TestFlight / Google Play GitHub Actions workflows
```

---

## Step 1: Sync ESP32 Firmware Package (if `ble/` has changes)

If any firmware or Web UI files under `ble/` were modified since the last package build, run:

```bash
python3 ble/scripts/package_for_ios.py --build
```

This compiles `firmware.bin`, syncs `ios/Resources/FirmwarePackage/`, `android/app/src/main/assets/FirmwarePackage/`, and rebuilds `ble/dist/`.

---

## Step 2: Collect Changes Since Last Release

Identify the previous release commit (e.g. `git -C ios log --grep="Promote version" -n 1` or the last tag/release commit) and inspect all commits across `ios`, `android`, `ble`, and the root repository since that point:

```bash
git -C ios log <last-release-sha>..HEAD --oneline
git -C android log <last-release-sha>..HEAD --oneline
git -C ble log <last-release-sha>..HEAD --oneline
```

Summarize all user-visible features, firmware improvements, bug fixes, and UI updates developed since the last version released to TestFlight.

---

## Step 3: Promote the Patch Version (`X.Y.Z` -> `X.Y.(Z+1)`)

**Every store release MUST promote the patch version first** (e.g. `2.2.0` -> `2.2.1`), leaving major and minor unchanged:

1. **iOS (`ios/`)**:
   - `ios/App/Info.plist`: Update `<key>CFBundleShortVersionString</key>` to `X.Y.(Z+1)`.
   - `ios/TirePressureManager.xcodeproj/project.pbxproj`: Update `MARKETING_VERSION = X.Y.(Z+1);` in both `Debug` and `Release` configurations of the `TirePressureManager` target.
   - `ios/README.md`: Update the version badge in the title.

2. **Android (`android/`)**:
   - `android/app/build.gradle`: Increment `def versionPatch` by `1` and leave `def versionMinor` unchanged.
   - `android/README.md`: Update the version badge in the title.

3. **Root Documentation**:
   - Update the iOS and Android version numbers in `README.md` and `SPECIFICATION.md`.

---

## Step 4: Update Release Notes

Write clear, concise release notes summarizing what was developed since the last TestFlight release into all three release note targets:

1. **TestFlight Release Notes (`ios/fastlane/release_notes.txt`)**:
   - Plain text bullet list passed to `upload_to_testflight(changelog: ...)` in `ios/fastlane/Fastfile`.
   - Include the new app version, bundled ESP32 firmware version (if updated), and bullet points of new features and fixes.

2. **Google Play Internal Release Notes (`android/whatsnew/whatsnew-en-US`)**:
   - Plain text (strictly **<= 500 characters** as required by Google Play Console) consumed by `r0adkll/upload-google-play` via `whatsNewDirectory: whatsnew`.

3. **Repository Release History (`RELEASE_NOTES.md`)**:
   - Prepend a new `## vX.Y.(Z+1) (YYYY-MM-DD)` section at the top of `RELEASE_NOTES.md` in the root repository.

---

## Step 5: Commit and Push in Submodule Order

Commit and push each submodule first, then commit the updated submodule pointers in the root repository:

```bash
# 1. BLE submodule (if modified)
git -C ble add -A && git -C ble commit -m "Release vX.Y.0 firmware package" && git -C ble push origin main

# 2. iOS submodule
git -C ios add -A && git -C ios commit -m "Release vX.Y.Z: promote patch version and update TestFlight release notes" && git -C ios push origin main

# 3. Android submodule
git -C android add -A && git -C android commit -m "Release vX.Y.Z: promote patch version and update Play Store release notes" && git -C android push origin main

# 4. Parent repository
git add ble ios android README.md SPECIFICATION.md RELEASE_NOTES.md
git commit -m "Release vX.Y.0 across iOS, Android, and BLE submodules"
git push origin main
```

---

## Step 6: Trigger Store Workflows

Trigger the GitHub Actions workflows and report the run links to the user:

```bash
# iOS TestFlight
gh workflow run ios-testflight.yml -R nir-azulay/TiresAirPressure-iOS
gh run list -R nir-azulay/TiresAirPressure-iOS -L 2

# Android Google Play Internal Track
gh workflow run android-play.yml -R nir-azulay/TiresAirPressure-Android
gh run list -R nir-azulay/TiresAirPressure-Android -L 2
```
