# Agent work breakdown and acceptance gates

## Phase 0 — reproduce baseline (MUST complete first)
- Pin branch/head SHA; enumerate all source, resources, tests, gradle and manifest. Examine `DrivingService.kt`, `SettingsManager.kt`, `FloatingSpeedometer.kt`, `MainActivity.kt`, `SettingsActivity.kt`, `LocalSpeedDbManager.kt`, `SpeedLimitRepository.kt`, `ServiceNotification.kt`.
- Run `./gradlew --version`, `./gradlew :app:assembleDebug`, `./gradlew testDebugUnitTest` where supported; record commands, Gradle/JDK/SDK versions and real outcome; no fake PASS. If environment lacks SDK, mark BLOCKED and document missing prerequisites.
- Check existing unit/instrumented tests and baseline screen/service behavior. Audit permissions targetSdk compatibility, backup and FileProvider XML, possible double speed unit conversion in overlays. Inspect settings schema and strings for existing translations.
- Deliver `plan/IMPLEMENTATION_LOG.md` with baseline SHA, findings, risks and signed-off design defaults.
- Gate: known build status and code call graph documented. Do not rewrite unrelated legacy logic.

## Phase 1 — pure Kotlin sample + alert rules engine (no Android UI)
- Create `telemetry/SpeedSample.kt`, `alerts/AlertRule.kt`, `alerts/CustomRuleEngine.kt`, `alerts/AlertDecision.kt` under existing source package (exact path adapted after build audit).
- Isolate valid sample input; monotonic timestamps, UTC timestamps, speed units, data quality; deterministic rule evaluation and independent rule states.
- Implement default seed rules (60/10s, 80/5s, 100/3s) as **user-editable examples** and durable IDs.
- Add unit tests: boundary equality, 1ms before/at duration, overlapping levels, per-rule rearm, repeat/cooldown, GPS gap, out-of-order fixes, duplicates, negative/NaN speed, settings edits mid-event, monotonic vs wall time.
- Gate: all tests pass without emulator, zero Android API dependencies in rule engine. Existing build remains green.

## Phase 2 — integrate live GPS and safe alerts
- Separate `DrivingService` ingest of GPS fixes from its 1-second UI loop; avoid double-ingestion when GPS and fused overlap. Evaluate custom rules only on accepted, fresh, valid fixes.
- Introduce coordinator for sound/visual alerts, prioritize rules; preserve old legacy OSM/camera audio with explicit arbitration. Do NOT issue several beeps simultaneously. Manage AudioTrack/TTS lifecycle correctly.
- Add stale GPS notice and location quality indicator to overlay; keep existing text/sign layout and draggable position. Decide overlay warning style readable over Google Maps.
- Edit settings UI to add rules CRUD, threshold km/h, duration sec, sound, severity, repeat and enable; validate input; use testable repository/config abstraction (SharedPreferences JSON for MVP or Room after phase 3). UI must state rules are absolute speed, NOT road-specific OSM limits.
- Ensure startup/stop/resume/permission revocation/muted operation gracefully clean up.
- Gate: mock provider/emulator verifies 3 rules; existing road limit behavior retained; custom rules work with no OSM DB/network. Screenshot/video evidence where available.

## Phase 3 — trip recorder and local persistence
- Choose Room-compatible stable version with pinned versions after Phase 0 gradle audit; add entities/DAOs/migrations. Prefer single writer on Dispatchers.IO + bounded queue/channel; protect against memory growth and failed writes.
- Trip Start/Stop explicit; service notification shows recording state; recover unfinished trips as INTERRUPTED. Record every configured interval based on monotonic fix timestamps; persist the first and last valid sample and GPS-gap boundaries.
- Compute GPS longitudinal accel from successive unique valid fix speeds in m/s; reject unsuitable delta/gaps and store NULL with reason. Keep raw speed and filtered speed separate; accuracy values nullable when Android provider omits them.
- Database indexes, status transitions, privacy/backup exclusions, lifecycle tests, disk-full handling/notification; catch recoverable failures without infinite retries.
- Gate: no server connection needed; 30-minute airplane-mode test exports expected number of usable points with actual timestamps/gaps and documented tolerance; restart recovery works; no missing trip due to normal activity switching.

## Phase 4 — export, review and operations
- UI: Trip History list with start/end, duration, approximate distance (valid adjacent fixes only), max speed, alert counts, share/export/delete.
- CSV schema version + documented columns in `plan/03-TEST-MATRIX.md`; GPX route with standard trackpoints and namespaced extension values. UTF-8 and locale-invariant decimal.
- SAF exports and FileProvider sharing with correct MIME; Android 10–16 tested. README privacy update, translations minimally English/Thai/Japanese if quality checked, trip UI with safe navigation.
- Gate: CSV round-trip parse validates rows, GPX validates XML, exports offline, deleted trip no longer appears, existing application continues running while Google Maps is foreground.

## Phase 5 — optional server upload (separate PR and disabled by default)
- Scope MVP HTTP API: `POST /v1/trips`, `POST /v1/trips/{tripId}/points:batch`, `POST /v1/trips/{tripId}/events:batch`. Contract should be documented and versioned; implement client only after server contract exists.
- WorkManager with network constraint, exponential backoff and durable cursor; HTTPS only, auth token secure at rest, idempotency by entity UUIDs, bounded batch sizes, redacted logs.
- Avoid uploading without opt-in and clear endpoint display. Do not interpret HTTP timeout as upload failure with safe-to-duplicate writes: retry idempotently.
- Gate: airplane-mode recording, Wi-Fi restoration -> eventual at-least-once delivery without duplicate server records; disable sync prevents future uploads; local retention unchanged.

## Scope control and pull request sequencing
PR1 baseline + design confirmation. PR2 rules pure engine + tests. PR3 GPS integration/alerts/settings/overlay. PR4 persistence + recorder. PR5 export/history/privacy. PR6 optional server sync. No PR may include unexplained refactors or dependency upgrades.

## Definition of done (all mandatory)
- APK builds reproducibly, no lint/static check regressions relative to baseline.
- Deterministic unit tests pass; instrumented tests documented with tested device/API levels.
- Overlay works atop Maps and when screen is locked until user stops session, subject to Android restrictions; foreground notification visible.
- Custom multi-level alerts strictly follow continuous durations and pause/reset on stale GPS.
- Local storage preserves trip points/events with correct timestamps, units, null/quality semantics, offline export.
- Existing speed-limit fetch, camera warning, settings and mute behaviors retain parity or explicit documented change.
- Privacy disclosure correct; MIT attribution preserved.
- Document all unknowns or untested cases honestly in implementation log.
