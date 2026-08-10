# ScreenLens — Build Report

Project: ScreenLens
Package: `com.screenlens.app`
Version: 1.0.0 (versionCode 1)
Build Date: 2026-08-10 (first green run), report last updated 2026-08-10

## ✅ CI build status: GREEN

**Verified GitHub Actions run:** [`e4edd9e` — run #31404804092](https://github.com/halashasneen-oss/screenlens-android/actions/runs/31404804092) on `main`, completed `2026-08-10T15:43:56Z` with `conclusion: success`, confirmed by fetching the run and its per-step results directly from the GitHub Actions API (not assumed from the push alone).

| Step | Result |
| --- | --- |
| Checkout, JDK 17, Android SDK, Gradle setup | PASS |
| `./gradlew test` (27 unit tests) | **PASS** — 27/27 |
| `./gradlew lint` | **PASS** — 0 errors |
| `./gradlew assembleDebug` | **PASS** |
| Debug APK uploaded | **PASS** — artifact `screenlens-debug-apk`, 72,297,831 bytes, [download](https://github.com/halashasneen-oss/screenlens-android/actions/runs/31404804092/artifacts/9069269708) (GitHub Actions artifacts expire 90 days after upload — 2026-11-08) |
| Release signing secrets check | ran, correctly found none configured |
| `assembleRelease` / `bundleRelease` / release uploads | **SKIPPED** (by design — no `KEYSTORE_BASE64`/`KEYSTORE_PASSWORD`/`KEY_ALIAS`/`KEY_PASSWORD` secrets configured on the repo yet; see README → GitHub Actions / Secrets setup) |

Debug APK: **PASS — produced and uploaded.**
Release APK: **not produced** (no signing secrets configured — expected, not a failure).
Release AAB: **not produced** (same reason).

This was reached after 8 iterative CI-driven fix rounds from the first push (`c4aa3cc`) to this green run (`e4edd9e`), each one diagnosed from real Gradle/AAPT2/Kotlinc/Robolectric output — see the commit history for the full list. Genuine bugs found and fixed along the way (not just this environment's network limitation):

1. `xmlns:app="http://schemas.android.com/apk/res/app"` (wrong URI) in 15 layout/navigation XML files → should be `http://schemas.android.com/apk/res-auto`; broke every `app:` attribute.
2. Missing `@string/settings_title` resource.
3. `InterstitialAdLoadCallback.onAdFailedToLoad` overridden with the wrong parameter type (`AdError` instead of `LoadAdError`).
4. Play Billing's `queryProductDetails` KTX suspend extension called with callback syntax instead of `suspend`/coroutine syntax.
5. `MediaProjectionManager.getMediaProjection()` treated as non-null when the current compileSdk annotates it nullable.
6. CameraX `Preview` has no `surfaceProvider` getter, so `x.surfaceProvider = y` doesn't compile — needed `x.setSurfaceProvider(y)`.
7. Navigation Safe Args reorders a destination's required arguments before its optional ones in the generated Kotlin function regardless of XML declaration order — three call sites were passing a nullable value into what was actually the non-null `sourceType` slot; fixed with named arguments.
8. `QueryProductDetailsResult.productDetailsList` is nullable, not guaranteed non-null.
9. Missing `androidx.test:core` test dependency (`ApplicationProvider` unresolved).
10. Robolectric 4.14 doesn't support simulating API 36 yet — pinned via `robolectric.properties` (`sdk=34`) for unit tests only; the app's real `compileSdk`/`targetSdk` stay 36.
11. `SettingsDataStoreTest`'s cleanup tried to delete DataStore's backing file, which doesn't invalidate DataStore's in-process singleton cache — replaced with a real `SettingsDataStore.clearAll()` call through the live instance.
12. Two real `NewApi` lint errors: `Service#startForeground(int, Notification, int)` only exists on API 29+, called unconditionally with `minSdk 24` — would have thrown `NoSuchMethodError` on API 24-28 devices at runtime, not just failed lint. Also `FOREGROUND_SERVICE_TYPE_SPECIAL_USE` is only meaningful on API 34+.

## Local build (this sandbox)

Still **not possible** in the environment this session ran in — unchanged from earlier in this report's history:

```
$ ./gradlew tasks
FAILURE: Build failed with an exception.
Plugin [id: 'com.android.application', version: '8.9.1', apply: false] was not found ...
```

**Root cause:** this session's outbound network policy blocks `dl.google.com` (HTTP 403 at the proxy). Google's Maven repository and the Android SDK component downloader both resolve through it, so neither AGP, AndroidX/ML Kit dependencies, nor an Android SDK could be fetched here. This is an infrastructure/network-policy limitation of this sandbox, not a defect in the project — confirmed by the fact that the exact same source, built on GitHub Actions (a normal-internet environment), now passes end-to-end.

**Practically: this doesn't matter for the project's build status anymore.** The GitHub Actions result above is real, verified, and green — it is the authoritative build status, and it was reached purely by fixing real compiler/linter/test output, not by working around this sandbox's limitation.

## Tests

27 tests across: `HistoryDaoTest`, `HistoryRepositoryTest`, `SettingsDataStoreTest`, `LanguageCatalogTest`, `HistoryMappingTest`, `HistoryDisplayTest`, `OcrOutcomeTest` (see `README.md` → *Testing*).

Unit Tests: **PASS** — 27/27, verified on GitHub Actions run #31404804092.
Lint: **PASS** — 0 errors, verified on the same run.
Debug APK: **PASS** — produced, verified on the same run.
Release APK: **not attempted** (no signing secrets configured on the repo).
AAB: **not attempted** (same reason).

## GitHub

Repository: https://github.com/halashasneen-oss/screenlens-android
Commit SHA (green build): `e4edd9e076afd60cfdf22b58c89171a558232771`
Workflow run: https://github.com/halashasneen-oss/screenlens-android/actions/runs/31404804092

## Security Scan

Performed before each commit: recursive search across the whole project for
AWS/Google API key patterns, PEM private key headers, OAuth bearer/
Slack-style tokens, and by filename for `*.keystore`, `*.jks`,
`secrets.properties`, `.env`, `google-services.json`, `local.properties`.

Secrets Found: **NO**
Result: **PASS**

`secrets.properties.example` contains only empty placeholder keys, matching
the project's `.gitignore`, which excludes `secrets.properties`, `*.keystore`,
`*.jks`, `local.properties`, and `.idea/`.

## Known Limitations

See `README.md` → *Known limitations* for the full list, summarized:

1. Arabic **OCR** (reading Arabic glyphs from images/screens) is not
   available — ML Kit's on-device Text Recognition only supports
   Latin-alphabet script. Arabic **translation** and the Arabic **app UI**
   both work fully.
2. Premium purchases are architecturally real (Play Billing v7) but
   inactive — no product exists in Play Console yet, so the app correctly
   reports "not configured."
3. No production AdMob/Play Billing identifiers are configured; debug and
   (until secrets are added) release builds use Google's public test IDs.
4. No signed release APK/AAB yet — needs `KEYSTORE_BASE64`, `KEYSTORE_PASSWORD`,
   `KEY_ALIAS`, `KEY_PASSWORD` added as GitHub Secrets (see README).
5. No store screenshots are included (see `store-assets/README.md`) — they
   must come from an actual running build (the debug APK above can now
   produce them).
6. Robolectric's *test-time* simulated SDK is pinned to 34 (`robolectric.properties`)
   because Robolectric 4.14 doesn't ship API 36 support yet — this only
   affects what the JVM unit tests simulate, not the real app, which still
   compiles and targets SDK 36.
