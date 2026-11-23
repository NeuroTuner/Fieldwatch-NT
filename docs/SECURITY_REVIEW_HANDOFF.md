# Fieldwatch-NT — security review and build handoff

Review of upstream Fieldwatch 1.1.21 (commit 5379a20), done in a Claude Code cloud session.
Use this file to continue locally: open Claude Code in this repo and point it here.

## Verdict
Not malware. No telemetry, analytics, ads, dynamic code loading, WebView, GATT/pairing or BLE advertising.
Shipped `dist/Fieldwatch.apk` is consistent with source at class, method-name and string-constant level
(signing cert SHA-256 72f0da99…2609 matches instruction.txt). NOT proven byte-identical: no rebuild was
possible in the cloud sandbox, and the APK was built outside git (`NO_VALID_GIT_FOUND`).
Recommendation: build our own.

## Network egress (complete list)
- `data/CatalogRemote.kt` — HTTPS GET of upstream catalog JSON (OffGridPete/Fieldwatch), user-tap only.
- `data/PathTiles.kt` — OSM tiles, automatic when viewing paths if onlineLookup is on.
- `data/PlaceLookup.kt` — system Geocoder (usually Google), exact coordinates.
- `radio/TakPublisher.kt` — UDP CoT, off by default; always mirrors to 127.0.0.1.

## Findings (paths under app/src/main/java/app/fieldwatch/)
Privacy
1. `domain/Models.kt:571,573` — tagLocation and onlineLookup default ON.
2. `data/ConfigStore.kt:359-367` — migrations V19/V20 force both back ON over user opt-out.
3. `ui/FieldwatchViewModel.kt:2151,2220,2283` — every share export silently copied to getExternalFilesDir(); survives Clear log.
4. Privacy mode (demoMode) doesn't cover logs, file exports or tiles; `domain/Geo.kt:73` masker misses Remote ID lat/lon on separate lines.
5. `data/LogStore.kt` — ~12 MB rolling JSONL, full MAC + lat/lon, no time expiry.

Catalog update (remote JSON, TLS only, no signature, no preview)
6. `domain/SignatureExchange.kt:229-261` — can neuter stock detections (matchAny/rules overwrite persists).
7. `data/ConfigStore.kt:141,151` — pack catalogVersion written to config.version; huge value blocks all future updates and migrations.
8. `domain/SignatureEngine.kt:254-266`, `domain/Models.kt:1016-1029` — glob `*` -> `.*` uncapped; ReDoS measured 16.5 s/match on desktop JVM. Models.glob recompiles regex every call.
9. `domain/SignatureFieldDecoder.kt:173` — non-hex gate value throws outside try; aborts relabel for the batch.

Adversary in radio range (parsers are bounds-safe; issues are resources)
10. `data/DeviceStore.kt:136-139`, `domain/RadioFacts.kt:97-107` — per-device serviceUuids / serviceData unbounded.
11. `domain/SignatureFieldDecoder.kt:52` — decode cache never evicted.
12. `alert/Alerter.kt:139-165` — per-new-MAC notification + vibrate, no rate limit.
13. `data/DeviceStore.kt:245-268` — 900-row table; MAC spam evicts real devices, resets co-travel history.
14. `domain/Sit.kt:350-358` — pinned radios bypass sit cap; `data/SitStore.kt:290` non-atomic write.
15. BLE parsing on main thread; `radio/WifiIeParser.kt:121,128,150` allocations from attacker counts (minor).
Lower: raw SSIDs/names + catalog notes in AI Export prompts (prompt injection); bidi/zero-width not stripped;
XML-illegal control chars can break KML/GPX; Remote ID unauthenticated.

## Proposed changes
Phase 1 build: signed-release CI (keystore in repo secrets), gradle wrapper-validation step,
applicationId `app.fieldwatch.nt` (side-by-side with upstream).
Phase 2 privacy: onlineLookup default off; remove V19/V20 forced re-enable; catalog URL -> this fork (or remove) + size cap;
external export copies opt-in.
Phase 3 robustness (+ unit tests): cap per-device lists; LRU-bound decode cache; non-backtracking glob matcher with cached
compile and star cap; alert rate limit; clamp catalogVersion.
Phase 4 optional: privacy-mode coverage, text sanitizing, atomic sit writes, decode-map validation on import.

Open decisions: which phases (recommend 1-3); build path (CI vs local); catalog source (fork vs remove).

## Unverified
- `gradle/wrapper/gradle-wrapper.jar` SHA-256 2db75c40782f5e8ba1fc278a5574bab070adccb2d21ca5a6e5ed840888448046 —
  compare with https://services.gradle.org/distributions/gradle-8.11.1-wrapper.jar.sha256 before building.
