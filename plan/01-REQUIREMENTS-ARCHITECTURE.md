# Requirements and technical design (normative)

## Functional requirements / defaults
R01 — Floating overlay: retain existing movable window; show GPS speed in km/h, GPS stale/valid state, recording status, optional road speed limit and most severe active user rule. All taps non-obstructive, reposition drag, persistent coordinates. Do not hide custom alerts when OSM limit is unavailable.
R02 — Rules: support 1..10 individually enabled rules; each has immutable UUID, label, thresholdKmh (1..250), durationMs (0..120000), severity INFO/WARNING/CRITICAL, soundMode NONE/BEEP/TTS, repeatCooldownMs (>= 0), enabled. Sample defaults **examples only**: 60 km/h / 10s, 80 / 5s, 100 / 3s. UI must let user edit all fields. Threshold uses strict `speedKmh > thresholdKmh`; equal means no violation. Units internally normalized to km/h.
R03 — Time duration: violation starts at timestamp of first **accepted fresh location sample** with speed strictly over threshold. Trigger at/after its continuous duration only while valid speed evidence continues. Missing/stale GPS must never accumulate elapsed time. On stale/invalid fix **reset pending durations** and suppress newly triggered warnings; re-arm after fresh samples. Invalid means missing speed, mock? See policy below. Avoid constructing fake speed=0 from missing speed.
R04 — Multi-rule behavior: all rules are evaluated independently, even if several thresholds cross simultaneously; overlay displays highest severity active rule (tie: higher threshold, then rule UUID), while audio arbitration emits at most one sound per evaluation tick using same priority. Persist rule trigger events for every qualifying rule. Lower-level pending rules continue timing. A rule triggers once per contiguous violation interval, then may repeat if configured repeatCooldownMs>0; once speed <= threshold it rearms. Simultaneous notifications are coalesced to highest audible priority but other events are logged.
R05 — Timing: use `Location.elapsedRealtimeNanos` or monotonic elapsed clock; never wall clock for elapsed violation. Detect out-of-order/duplicate fixes and drop duplicates for evaluation; use wall UTC timestamps only for export. Do not count time over GPS gaps beyond a configurable max gap (default 3s). Avoid wall-clock jumps.
R06 — GPS quality: require `hasSpeed`, accurate/fresh coordinate, usable elapsedRealtime and accuracy setting (default <=25m, configurable), reject invalid values / negative speed; keep raw and filtered speed separate. Never allow the same fused+GPS event to be double counted. If GPS is lost show GPS LOST and persist gap event.
R07 — Telemetry: start/stop trip explicitly. While active, persist a timestamped trackpoint about every configurable 10s (choices 1,5,10,15,30,60); persist initial/last points and gap boundaries; include original fix time, UTC sample time, latitude, longitude, position accuracy, provider, raw GPS speed, filtered speed, bearing if present, altitude if present, derived longitudinal acceleration (m/s²), derivation quality and source. Derive acceleration from consecutive distinct valid location speeds and timestamps: `(v2_ms-v1_ms)/delta_seconds` for positive delta and reasonable gap; if unavailable use NULL, not 0. GPS-derived longitudinal acceleration is not 3-axis accelerometer data. Physical accelerometer/harsh braking optional later, not v1.
R08 — Local DB: Room/SQLite with `Trip`, `TrackPoint`, `AlertRule`, `AlertEvent`, `GpsGapEvent`, `SyncQueue` only when sync is implemented. Persist trips atomically, indexed by trip/time; WAL supported. No automatic deletion by default. Handle application kill and recovery of unfinished trips (mark INTERRUPTED on next launch; never silently fabricate data).
R09 — Export: CSV with UTF-8 header, RFC4180 quoting, UTC ISO8601, fixed decimal '.' and units in headings or schema metadata; optional GPX 1.1 tracks with extensions for speed/acceleration, valid XML escaping. Use Android Storage Access Framework ACTION_CREATE_DOCUMENT or share via FileProvider; never broad storage permission merely for export. Export must work offline.
R10 — Optional server: after local and export are proven, opt-in HTTPS upload to user-configured endpoint. Batches with trip UUID, point UUID, retry/idempotency keys, last success cursor and durable queue. Never delete local points on upload success. TLS validation enabled; no insecure cleartext; tokens in appropriate secure storage; redact diagnostics. No server required for basic app.
R11 — Privacy: local-only default; user controls recording and export. Show foreground recording indicator. Optional retention; delete all trip data control with confirmation. No secretly running telemetry.
R12 — Japan: km/h defaults, timezone-independent UTC storage and optional Asia/Tokyo display, airplane/offline-capable; do not infer statutory road limit from country or OSM. Existing fallback OSM road-limit logic stays independent from absolute custom alerts. Do not assume speed camera database legality.
R13 — Driving UX: high contrast, large digits, minimal interaction during driving; settings only when parked; optional volume/mute persisted.

## State machine for EACH rule
States DISARMED / PENDING(startMono) / FIRED(lastSoundMono) / GPS_SUSPENDED.
- Good fix with speed <= threshold: DISARMED; clear pending/repeat.
- Good fix speed > threshold from DISARMED: PENDING, start mono=fixTime.
- Good fix while PENDING: if gap > maxGap reset PENDING at current fix; else fire when elapsed >= duration, enter FIRED.
- FIRED with continued good fixes: record repeat only after cooldown >0 reached (or never repeat when 0).
- GPS invalid/stale: GPS_SUSPENDED; clear pending/armed interval; do not alert.
- Good GPS resumption: evaluate as new interval.
- Settings edit mid-drive: reset corresponding rule state to prevent inherited timers; document change in log.
- Foreground/background clock jumps do not change monotonic elapsed logic.

NOTE: Android receives event-based locations; time crossing duration without a new good fix should not automatically trigger. This is intentional conservatism. Audio callback scheduler never uses stale cached speed as evidence.

## Architecture target (suggested, adapt to real code without unnecessary rewrites)
```text
LocationManager/FusedLocationProvider
  -> LocationIngestor (deduplicate, quality, freshness)
  -> SpeedSample {rawKmh, filteredKmh, utc, monotonic, accuracy, provider}
     |-> Existing SpeedLimitRepository -> legacy road-limit warning
     |-> CustomRuleEngine (pure Kotlin) -> AlertEvent -> AlertCoordinator -> audio/overlay
     |-> TripRecorder -> Room (IO coroutine) -> CSV/GPX exporter
                                         -> optional SyncWorker/HTTPS
```
DrivingService remains lifecycle orchestrator: hold location callbacks, foreground notification and scope. Move rule policy into `CustomRuleEngine` and persistence into `TripRecorder`. Do NOT feed custom threshold rules through `calculateOverspeed()`: that uses OSM limits + percentage/fixed tolerance, and currently sound alerts are coupled to `updateOverlay()`. Decouple rendering from alert triggering.

## Existing behavior compatibility and known risks (from source)
- `DrivingService.kt` requests location updates from Android provider at 200ms and fused at 1000ms; fused callback only when primary GPS fix age >5s. Main loop runs at 1000ms.
- `updateLocationAndSpeed()` reads `loc.speed*3.6`, Kalman-filters, broadcasts; verify filter reset on recording startup/GPS loss and preserve separate raw value.
- `hasGpsFix()` treats age <5s and accuracy <=25m as valid for overlay; that is not a sufficient proof of continuously valid threshold samples.
- `updateOverlay()` currently both draws overlay and plays beep based on `calculateOverspeed()`; alert audio must be moved to coordinator without changing existing legacy behavior unexpectedly.
- `SpeedLimitRepository.kt` can return source fallback/unknown; do not reinterpret as user-configured absolute thresholds.
- Manifest app currently has `android:allowBackup=true`; avoid unexpectedly backing up personal GPS trip history: add explicit backup exclusions / policy.
- README currently claims no personal information is stored; once telemetry is implemented, revise README/privacy disclosure and app UI truthfully.
- appId/package `xyz.hvdw.speedalert`: change only with explicit migration strategy; fork installation can collide with upstream existing app.
- Confirm actual compilation and all required dependencies before selecting Room/WorkManager versions, especially Kotlin 1.9.22 and compileSdk 34. Never assume a new library version is compatible.

## Durable database schema — conceptual contracts
Trip(id UUID PK, startedAtUtc, endedAtUtc nullable, status ACTIVE|COMPLETED|INTERRUPTED, deviceTimezone, samplingIntervalSec, appVersion).
TrackPoint(id UUID PK, tripId FK, fixTimeUtc, recordedAtUtc, elapsedRealtimeNanos, latDeg, lonDeg, horizontalAccuracyM, provider, speedRawMps nullable, speedFilteredKmh nullable, speedAccuracyMps nullable, bearingDeg nullable, altitudeM nullable, accelerationLongitudinalMps2 nullable, accelerationSource enum GPS_DERIVED|UNAVAILABLE, qualityFlags bitset); unique(tripId, elapsedRealtimeNanos), index(tripId,fixTimeUtc).
AlertRule(id UUID PK, title, enabled, thresholdKmh, durationMs, severity, soundMode, cooldownMs, sortOrder).
AlertEvent(id UUID PK, tripId FK nullable, ruleId FK nullable, timeUtc, elapsedRealtimeNanos, measuredSpeedKmh, thresholdKmh, sustainedMs, severity, kind FIRST|REPEAT, soundActuallyPlayed, reason).
GpsGapEvent(id UUID PK, tripId, startedUtc, endedUtc nullable, reason).
Migrations: explicit, tested, NO destructive fallback against live trip data.

## Decisions to record during phase 0
- Whether multi-rule input uses raw GPS speed or lightly filtered GPS speed. Recommended default filtered for display, raw validated speed for rule logic; show and document distinction. Consider GPS speed accuracy and avoid overly smoothing alert delays.
- Whether existing legacy road-speed warning and custom rules both produce audio or audio is arbitrated; default coordinator prioritizes CRITICAL custom, then WARNING custom, then legacy, without silencing data events.
- Whether background location permission is needed for manual foreground-only session on target SDK/device; verify Android 10..16 behavior and Google Play policy, do not assume.
- Whether to include Google Play Services dependency in offline build; offline operation requires no network, not absence of Play Services.
