# Verification matrix, fixtures and release checklist

## Required deterministic fixtures (unit tests)
| ID | Input | Expected |
| --- | --- | --- |
| A01 | rule 80km/h/5s; 80 exactly for 6s | no trigger |
| A02 | 81 at t=0, t=4.999s, t=5s, all fixes valid and gap <=3s | first trigger once at 5s |
| A03 | 81 at t=0 then missing valid GPS until t=6s | no trigger; timer restarts |
| A04 | 81 at t=0, 79 at t=3s, 81 at t=4s, 81 at t=9s (continuous samples) | trigger at t=9s, not earlier |
| A05 | rules 60/10,80/5,100/3; speed 105 on continuous accepted fixes | independent alerts at 3,5,10s; highest priority audio per tick |
| A06 | duplicate same elapsedRealtimeNanos from fused + GPS | evaluate/persist once |
| A07 | out-of-order fix arrives with older monotonic time | ignore for rule state; diagnostic counter |
| A08 | device wall clock jumps +1 hour during event | elapsed duration unaffected |
| A09 | rule edited while pending | reset rule timer |
| A10 | speed unavailable / NaN / negative / accuracy exceeds configured threshold | invalid input, no new warning |
| A11 | duration 0 | trigger on first above-threshold valid fix |
| A12 | repeat cooldown 10s during continuous exceedance | first trigger then repeats no more frequently than 10s |
| A13 | 2 rules trigger same instant, one muted | both alert events persisted, at most one sound |
| A14 | legacy OSM limit unknown, absolute rule 80/5 | custom rule works; legacy speed-limit warning stays inactive |
| A15 | first fix and repeated same cached fix across 1s UI ticks | no false duration accumulation |
| A16 | location time UTC and device timezone transition | stored UTC correct; export independent of locale |
| A17 | acceleration 10m/s -> 12m/s, 5s interval | +0.4m/s²; invalid gap -> NULL |
| A18 | location stops mid-trip | gap recorded, overlay GPS LOST, no fabricated stationary samples |
| A19 | crash/restart while trip ACTIVE | recover as INTERRUPTED and preserve points |
| A20 | CSV quote/comma/newline and Japanese labels | valid UTF-8 RFC4180 parsed round-trip |

For A02/A04/A05, generate intermediate fixes spaced <=3s; two far-apart fixes MUST NOT be interpreted as continuous evidence.

## On-device test scenarios
- Actual Android phone (Samsung and/or rental-car mounted device) with Maps foreground: overlay remains readable and movable.
- Permission workflow: fine denied, coarse-only, approximate location, overlay denied, notifications denied, background permission not granted; clear graceful explanations, no crash.
- Phone screen locked 30+min; GPS in urban tunnels; briefly airplane mode; time-zone Japan; connectivity on/off.
- OSM local DB missing/installed and online Overpass down: absolute rule unchanged.
- Home button/app swipes, service stop, force stop, reboot. Force-stop/reboot automatic resume NOT guaranteed; don't claim otherwise.
- Screen rotation, different display densities, dark theme, battery saver; speech/navigation audio overlap.
- Test exporting 1000+ trackpoints and deleting a trip; verify no precise coords in debug logs.
- Monitor battery, CPU, storage for 2-hour recording; record actual metrics only.

## CSV v1 required columns
`schema_version,trip_id,point_id,fix_time_utc,recorded_at_utc,elapsed_realtime_ns,latitude_deg,longitude_deg,accuracy_m,provider,speed_raw_mps,speed_filtered_kmh,speed_accuracy_mps,bearing_deg,altitude_m,acceleration_longitudinal_mps2,acceleration_source,quality_flags`.
Blank/null for unavailable optional measurements. Export raw speeds in m/s explicitly, filtered in km/h. No inferred zero speed for missing measurement.
Alert events exported separately with `event_id,trip_id,rule_id,time_utc,speed_kmh,threshold_kmh,sustained_ms,severity,kind,sound_played`.
CSV must clearly distinguish GPS-derived acceleration from IMU acceleration.

## Release gates for Japan
[ ] Test APK built and installed on intended Android device.
[ ] All custom rules saved, survive restart, and audio levels audible.
[ ] Overlay coexists with Google Maps; driver does not need to interact while moving.
[ ] Offline Japan road simulation passes; OSM unknown does not disable custom rules.
[ ] Test-trip 30 minutes records local database and successfully exported UTF-8 CSV.
[ ] GPS stale causes alert timers to reset, GPS LOST shown.
[ ] UTC times map correctly to Japan local time (UTC+09:00); no DST assumption in timestamps.
[ ] GPS recording indicator, manual stop, delete and privacy disclosure verified.
[ ] Baseline road-limit/camera features regression tested.
[ ] Unverified scenarios explicitly listed in `plan/IMPLEMENTATION_LOG.md`.

## Sample QA report template
Build SHA:
Device, Android version, battery policy:
Build/test commands and results:
Test case IDs PASS/FAIL/BLOCKED:
GPS mock vs real:
CSV/GPX evidence path (never commit private GPS traces):
Regressions:
Known limitations and release decision:
