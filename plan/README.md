# Android SpeedAlert — Agent Execution Plan

Repository: https://github.com/KOKThaweesak/Android-SpeedAlert
Baseline branch: main (inspect HEAD before editing)
Status: PLAN ONLY. No production code has been changed by this plan.

## Goal
Extend the existing Android Kotlin app for offline driving in Japan with:
1. Existing draggable floating GPS-speed overlay retained over navigation apps.
2. User-editable multiple independent speed thresholds in km/h, each with continuous exceedance duration, sound/visual severity, cooldown and re-arm behavior.
3. Location/speed/acceleration telemetry sampled for storage approximately every 10 seconds (configurable), while alert evaluation uses location events / ~1-second evaluation and monotonic timing.
4. Local-first, offline-safe durable trip recording, CSV and GPX exports, optional later HTTPS server sync.
5. User-visible recording/GPS health, safe Start/Stop, no blocking UI, privacy by default.

## Code baseline — verified by direct GitHub file inspection, not inference
- `app/src/main/java/xyz/hvdw/speedalert/DrivingService.kt`: Android foreground service, primary LocationManager GPS callback, fallback Fused Location Provider callback, 1-second main loop, Kalman-filtered speed; existing road-limit overspeed logic and beep in `updateOverlay()`.
- `FloatingSpeedometer.kt`: TYPE_APPLICATION_OVERLAY, draggable, two layout modes and mute.
- `SettingsManager.kt`: existing SharedPreferences; road-limit percentage/fixed tolerance and beep behavior.
- `SpeedLimitRepository.kt` and `LocalSpeedDbManager.kt`: existing online/offline road speed-limit lookup (not a reliable authoritative legal limit).
- `AndroidManifest.xml`: fine/coarse/background location, overlay, foreground service and notifications declared; service has location|mediaPlayback FGS type.
- `app/build.gradle.kts`: Kotlin 1.9.22, compileSdk 34, targetSdk 33, minSdk 27, Java 17 bytecode, Kotlin toolchain 21, Play Services Location 21.0.1. App versionName 2.3.1.
- README documents current overlay, camera alerts, local databases, manual foreground-service start and MIT license.
These facts do NOT prove a clean build or functioning target-device permissions. Audit and build first.

## Non-goals / safety
- Do not claim this is a legal speed-limit enforcement system; GPS/OSM may be inaccurate and temporary signs supersede OSM.
- Do not automatically upload GPS history; server sync is opt-in and phase 5.
- Do not bypass Android background location/overlay/battery restrictions.
- Do not break existing camera warnings/OSM speed-limit features.
- Do not implement a launcher home-screen AppWidget; requested 'widget' means **floating overlay above another app**.
- Do not redesign entire app, migrate package name or bump targetSdk without a documented compatibility decision.
- Do not log precise GPS coordinates to ordinary debug logs in release builds.
- Do not promise exact 10-second sensor delivery; persist actual timestamps and gap status.

## Agent operational rules
1. Read all files under `plan/` before changing code.
2. Inspect real files and line references again at current HEAD; document discrepancies in `plan/IMPLEMENTATION_LOG.md`.
3. Make small coherent PRs in the phase order defined in `02-IMPLEMENTATION.md`. Never implement future phases while earlier acceptance gates fail.
4. Preserve current road-limit warning separately from **custom absolute speed rules**; never treat unknown OSM speed limit as zero or as a law.
5. Store speeds internally as m/s or km/h with explicit typed conversions. Prefer m/s for sensor data, km/h for rule UI and CSV exports.
6. Add deterministic JVM tests for all policy logic; do not place policy inside Activity or overlay rendering.
7. Design all persistence for offline-first behavior. Never perform network or database I/O on main thread.
8. No secrets, personal location logs or raw trip database committed.
9. Treat `plan/01-REQUIREMENTS-ARCHITECTURE.md` as product contract; any deviation requires documenting reason and tests.
10. Finish each phase with source changes, test results, device-tested items (or explicit NOT TESTED), and a short implementation log.

## Reading order
1. `plan/01-REQUIREMENTS-ARCHITECTURE.md` — exact behavior, data contracts, edge cases.
2. `plan/02-IMPLEMENTATION.md` — staged tasks, file touchpoints, test and acceptance checklist.
3. `plan/03-TEST-MATRIX.md` — executable/scenario tests, Japan operational checklist.

## Completion definition
A full trip with phone locked and intermittent connectivity records timestamped GPS/speed plus acceleration quality/source; three independent rules can be edited and trigger only after their configured continuous duration; overlay remains functional above Maps; trip exports locally; all essential functions work offline and do not alter existing road-limit and camera warning behavior.
