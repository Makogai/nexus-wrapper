---
nexus: 1
id: 01a1ad84-4c6d-768d-bf6f-0918f3940d1d
kind: spec
title: umbra — overview
status: active
project: 33fba8bb-d3e8-450e-9b9d-26848c5d8c1a
topic: overview
created: "2026-11-05T21:01:31Z"
updated: "2026-11-05T22:52:50Z"
source: import
---

_Drafted by Claude in a NEXUS session and accepted on 2026-11-05. Facts marked as inferred still need checking._

# Umbra / Novčanik — overview

Review status (2026-11-05): drafted from claims that survived independent verification, then attacked for 5 rounds (27, 18, 13, 15, 11 findings). The round-5 findings were applied and re-read by the orchestrator; no sixth attack round ran. Unconfirmed by the owner.

Repo: `D:/code/umbra`, branch `main` at `726e195`, app version `1.4.0+14` (verified: pubspec.yaml:19) [80].

## Terms

- **flavor:** which app experience you get: Umbra (classic) or Novčanik (light). It is set by two separate switches, the Android productFlavor (`--flavor`) and the Dart entry point (`lib/main_light.dart` sets `UmbraFlavor.isLight`); see Flavors. The "light" flavor means Novčanik, not the light colour theme.
- **IKOF/IKOM:** the 32-hex invoice code (`iic` in the QR and code). **PIB:** the seller's tax number (`tin`). **crtd:** the receipt's creation time (`dateTimeCreated` in the POST) (verified: lib/services/fiscal_receipt.dart:49-51,167-169; lib/l10n/l10n.dart:356-362) [80].
- **grep rc=1:** the search found no match.
- **FAB:** the floating Add button.
- **back-fill:** posting the missed months of a monthly charge.

## How this was verified

- Five readers mapped the repo at HEAD `726e195` (boot/data/platform, receipt pipeline, domain logic, UI, tests/build/history). An independent verifier re-checked every claim at source. Nothing here comes from a refuted claim.
- Grades: **100** = executed (a command, a probe, or a saved test/analyze run); **80** = read at source (file:line given); **50** = derived (the chain is written out, not run). Grade-20 items are not stated as facts. They are listed in the open questions (task log / questions.md).
- Inline markers: `(verified: file:line)` means read at source or executed. `(inferred: ...)` means derived, with the chain given.
- Nothing was built or installed. No tracked file was changed. `flutter test` did write gitignored artifacts (build/test_cache, .dart_tool/version), and `git status` showed only the existing untracked graphify-out/ [100].
- `flutter analyze` was run once. `flutter test` was run three times: once plain, once with `--reporter expanded` to list test names, and once on `test/widget_test.dart` alone. All runs were on 2026-11-05 and the outputs are saved in `D:/code/RIA/_tickets/UMBRA-LEARN/evidence/` [100].
- Probe = a throw-away Dart script run from a scratch folder against the real lib/ code, with a mock HTTP client and no network. The probe scripts were not kept, so probe-backed [100] claims cannot be re-run from saved evidence.
- `graphify-out/GRAPH_REPORT.md` was read. It was built in cluster-only mode from commit `726e195b` (1771 nodes, 64 unnamed communities) [100]. It gives no named structure. It undercounts the fan-in of `LedgerController`, because every screen reaches the controller through `UmbraScope.of(context)` (verified: lib/widgets/umbra_widgets.dart:12-24) [80]. Do not use it as a dependency map.

## Purpose

Umbra is a private spending tracker for one person's phone. Balance, entries and price history stay in a local database on the device. Receipts from Montenegro are read by scanning the fiscal QR code and asking the Montenegrin tax site for the ticket (verified: README.md:3; lib/services/fiscal_receipt.dart:149-172) [80].

## Who it is for

- **Umbra (classic flavor):** a single private user on their own phone (README.md:3). It has every feature, including "For them" (items bought for someone else) (verified: README.md:3; lib/screens/shell_screen.dart:129) [80].
- **Novčanik (light flavor):** older users. The latest commit made it "easier for older people": larger text, bigger targets, labelled buttons, explainer text on tabs, and a quick-action grid on Home. "For them" is hidden (verified: lib/theme/app_theme.dart:174; lib/screens/home_screen.dart:85-116; git show 726e195) [80].
- The UI and copy are aimed at Montenegro: Serbian strings use ijekavian wording (the Montenegrin form: Cijene, mjesec, not Serbian Ekavian Cene, mesec), and the fiscal lookup goes to a host ending in `tax.gov.me` (normally mapr.tax.gov.me) (verified: lib/l10n/l10n.dart:103,114; lib/services/fiscal_receipt.dart:36-63) [80].

## Main features, as the user sees them

- **Onboarding** (3 pages): language and theme, then balance, optional monthly income, currency format, then reminders (verified: lib/screens/onboarding_screen.dart:68-190) [80].
- **Five tabs:** Home, History, Balance, Prices, Review. A floating Add button opens a sheet: Scan receipt, Add expense, Monthly charges, and (classic only) For them (verified: lib/screens/shell_screen.dart:37-57,89-143) [80].
- **Home**: current balance, spent this month, spending by store, For-them card (classic), upcoming monthly charges, recent entries; light adds a 2x2 quick-action grid (verified: lib/screens/home_screen.dart:86-230,312-358) [80].
- **Scan a receipt:** live QR scan; take a photo or pick one from the gallery (QR first, then on-device text recognition); type the fiscal code by hand; or open a local Demo ticket. Every path ends on an editable review screen before saving (verified: lib/screens/scan_screen.dart:45-55,94-182,384-400) [80]. The Scan screen Demo ticket is always available, even in release builds; UMBRA_DEMO only shows the Settings button that replaces all data with demo data (verified: scan_screen.dart:214-221,394-399 have no `isDemoBuild` check; lib/state/demo_data.dart:7; lib/screens/settings_screen.dart:233) [80].
- **Manual entry** of an expense or income, with optional line items (verified: lib/screens/add_entry_screen.dart:17-37) [80].
- **Entry detail**: date, source (scan / by hand / monthly charge), note, items, For-them marking (classic), delete. There is no edit: a correction means delete and re-add (verified: lib/screens/entry_detail_screen.dart:23-29,45-50,105-144,219; architecture §4) [80].
- **History** with search over titles and item names and an All/Spending/Funds filter (verified: lib/screens/history_screen.dart:33-96) [80].
- **Balance** timeline per month: starting balance, funds added, spending, end balance (verified: lib/screens/balance_screen.dart:88-210) [80].
- **Prices:** compares the last price of the same product across stores and lets the user link names of the same product (verified: comparison at lib/screens/prices_screen.dart:64-120 and lib/services/price_advisor.dart:34-60; linking in PriceItemScreen at prices_screen.dart:169-310, `linkPriceItems` at :206, and price_advisor.dart:327-375) [80].
- **Review:** month view (bar chart, month detail, top stores and items, biggest charge) and week view (verified: lib/screens/review_screen.dart:45-213) [80]. The engine still works out a score out of 100 and a headline (review_engine.dart:383-470), but no screen has shown it since 551966c (the 2026-10-26 commit that added flavors and For them). The monthly reminder on the 1st (notification id 13, notifications_service.dart:84-86) and the settings/onboarding text (l10n.dart:271,297,312) still promise "a score". The score labels at l10n.dart:245-251 (`monthScore`, `outOf100`, `addIncomeForScore`) are now unused (grep outside lib/l10n rc=1) [80].
- **Monthly charges** (recurring income or expense). Each due charge is dated 09:00 on its day of month and is posted the next time the app loads, or when a charge is added or edited. There is no background timer (verified: lib/state/ledger_controller.dart:37,419,429,438-469; lib/services/recurring.dart:3-33) [80].
- **For them** (classic only): mark items bought for someone else, see what they owe, and mark it paid, optionally adding the payment to the balance (verified: lib/screens/shared_screen.dart:86-119; lib/state/ledger_controller.dart:334-378) [80].
- **Reminders:** three fixed local notifications: daily 20:30, Sunday 10:00, 1st of the month 10:15 (verified: lib/services/notifications_service.dart:43-93) [80].
- **Settings:** starting balance and typical monthly income, theme (System/Light/Dark), language (English / Serbian Latin), currency display, reminders, erase everything, app version (verified: lib/screens/settings_screen.dart:128-290; starting balance and income at :129-147) [80]. Typical monthly income is never posted or added to the balance; see architecture §4 for how balance is computed.

## Flavors

| | classic | light |
|---|---|---|
| Name shown | Umbra | Novčanik |
| Android applicationId | `app.umbra.umbra` | `app.umbra.umbra.light` |
| Dart entry point | `lib/main.dart` | `lib/main_light.dart` (sets `UmbraFlavor.isLight = true`) |
| Database | `umbra.db` / web `umbra` | `umbra_light.db` / web `umbra_light` |
| Default theme | System | Light theme |
| Senior sizing, "about" text, quick actions | no | yes |
| For them | yes | hidden |

Sources: android/app/build.gradle.kts:29-39 (applicationId); android/app/src/light/res/values/strings.xml:3 (app_name Novčanik); android/app/src/{classic,light}/res/values*/colors.xml (launcher colours); lib/main_light.dart:4-7; lib/theme/umbra_flavor.dart:1-11; lib/theme/app_theme.dart:174-189 (verified) [80].

Important: the Android flavor and the Dart flag are **two separate switches**. The Android flavor sets the app id, the label, the launcher icon resources (byte-identical to classic) and two colours: the launcher background and `umbra_black`, the NormalTheme window background (#F3F6FB in light vs #030814 from main) (verified: android/app/src/light/res/values/colors.xml:3-4; android/app/src/light/res/values-night/colors.xml:3-4; android/app/src/main/res/values/colors.xml:3; android/app/src/main/res/values/styles.xml:6-8; md5 of src/light vs src/classic launcher PNGs and ic_launcher.xml) [80]. The light Dart behaviour comes only from the `lib/main_light.dart` entry point. Nothing in the repo documents this pairing (verified: grep, README.md:11-28) [100]. The git-ignored depfile build/app/intermediates/flutter/lightRelease/flutter_build.d lists lib/main_light.dart [100]. Nothing imports main_light.dart (grep lib test), so it can be there only as the entry point. Inferred: the last light release build used `-t lib/main_light.dart` [50]. This is local build state, not part of the repo.

iOS and web have no light variant in config: iOS has only the `Runner` scheme, and web has no `--flavor` option (verified: ios/Runner.xcodeproj/xcshareddata/xcschemes; `flutter build web -h`) [100].

## Privacy and network stance

- **Local only.** All data lives in a sembast database on the device (IndexedDB on web). There is no account, no sync and no backup or import path (verified: lib/data/db_open_io.dart:7-12; lib/data/db_open_web.dart:5-7; README.md:34 [80]; grep for backup/file_picker/share_plus over lib and pubspec.yaml returns nothing, and `export`/`import` appear in lib only as Dart directives [100]).
- **One outbound call in app code:** `fetchFiscalReceipt`, a single form POST to `<origin>/ic/api/verifyInvoice` (verified: grep over lib, lib/services/fiscal_receipt.dart:149-172) [100]. Three things trigger it: a scanned QR, a QR found in a photo, or typed fiscal codes. It is also triggered **without asking the user** when text recognition finds IKOF, PIB and a date on a photo. (For IKOF, PIB and crtd see Terms.) (verified: lib/screens/scan_screen.dart:149-156; probe) [100].
- **Caveats:**
  - The host comes from the QR. Any host ending in `tax.gov.me`, on any port, is accepted (verified: probe) [100].
  - Native ML Kit is bundled (android/app/build.gradle.kts:66 `com.google.mlkit:text-recognition:16.0.1`; pub-cache mobile_scanner-7.4.1/android/build.gradle.kts:73-79 uses bundled `com.google.mlkit:barcode-scanning:17.3.0` unless `useUnbundled` is set). Whether it sends telemetry was not audited [80 for bundling; telemetry unknown].
  - Web only: CanvasKit and the scanner's zxing-wasm load from CDNs; see architecture §9.
- **Receipt images are not stored.** When no items were found, the raw recognised text can be saved into the entry's notes (verified: lib/screens/receipt_review_screen.dart:39-43; lib/screens/scan_screen.dart:179; lib/screens/add_entry_screen.dart:59,207; `addEntry` has no image field, lib/state/ledger_controller.dart:225-234; the preview is held only in memory, add_entry_screen.dart:246-261) [80].
- The web manifest says "Nothing leaves this device", which contradicts the README's fiscal lookup statement (verified: web/manifest.json:2-8; README.md:3) [80].
- Android permissions: INTERNET, ACCESS_NETWORK_STATE, CAMERA, READ_EXTERNAL_STORAGE (maxSdk 32), READ_MEDIA_IMAGES, POST_NOTIFICATIONS, RECEIVE_BOOT_COMPLETED, VIBRATE. Cleartext HTTP is off (verified: android/app/src/main/AndroidManifest.xml:2-9,16) [80].

## How to run

Prerequisites: Flutter; an Android SDK with platform 37 and JDK 17 (android/app/build.gradle.kts:8,13-14,56); a phone or emulator (README.md:7-8); enough memory for Gradle's 8 GB heap (android/gradle.properties:1) [80]. iOS needs Xcode on a Mac (not tried here). The machine used here has Flutter 3.44.8 stable with Dart 3.12.2 (verified: `flutter --version`) [100]. The SDK constraint is `^3.12.1`, and no Flutter version is pinned (verified: pubspec.yaml:21-22; no .fvm or .tool-versions) [100].

| Goal | Command | Grade |
|---|---|---|
| Classic on Android | `flutter run` (pubspec `default-flavor: classic`) or `flutter run --flavor classic` | 50 (inferred: pubspec.yaml:72-73 + flutter_tools flavor resolution) |
| Light on Android | `flutter run --flavor light -t lib/main_light.dart` | 50 (inferred: flavor and entry point are separate switches; the depfile suggests the last light release build used `-t lib/main_light.dart` [50]) |
| Classic on web | `flutter run -d chrome` | 80 (README.md:27). Whether `default-flavor` affects web was not checked |
| Light on web | `flutter run -d chrome -t lib/main_light.dart` | 50 (inferred: `-t` exists for run [100]; web has no flavor) |
| iOS | Likely **fails**. pubspec forces flavor `classic`, but Xcode has only the `Runner` scheme | 50 (inferred from flutter_tools source; not run, Windows host) |
| Demo data button in Settings (group labelled "Test data") | add `--dart-define=UMBRA_DEMO=true` | 80 (read: lib/state/demo_data.dart:7; lib/screens/settings_screen.dart:233) |

Targets are Android, iOS and web only. There is no desktop target (no windows/, macos/ or linux/ folder), so on a computer use `-d chrome`, as README.md says (verified: `git ls-files` top-level folders; README.md:24-27) [100].

**Warning:** loading demo data erases all entries, monthly charges, price links and settings (except language and theme), then seeds demo data. It asks first, with a "Replace" button. Do not use it on a phone with real data (verified: lib/screens/settings_screen.dart:95-112; lib/state/ledger_controller.dart:386-394) [80].

The entry point can also be passed as a positional file: `flutter run --flavor light lib/main_light.dart` (verified: `flutter run -h`) [100].

On web, text recognition is a stub, and the Demo ticket still works (verified: lib/services/ocr/ocr.dart:1-2; README.md:30) [80].

## How to build

- Classic APK: `flutter build apk --release --flavor classic` (or no flavor, through the default). Light APK: `flutter build apk --release --flavor light -t lib/main_light.dart` (inferred, as above) [50].
- The outputs are named `app-classic-release.apk` / `app-light-release.apk`, not the `app-release.apk` the README names (verified: ls build/app/outputs/flutter-apk; README.md:22) [100].
- APKs are fat, with arm64-v8a, armeabi-v7a and x86_64 libraries, about 114 MB. `--split-per-abi` is available (verified: unzip -l; `flutter build apk -h`) [100].
- Release builds are **signed with the debug key**, with no minify and no shrink. There is no keystore or `key.properties` (verified: android/app/build.gradle.kts:41-51) [80].
- There is no CI and no build scripts (verified: git ls-files) [100].
- **How it is distributed:** as sideloaded APKs (README.md:22) [80]. There is no store listing (inferred: the README names only sideloading, and the release build is debug-signed, build.gradle.kts:41-51) [50]. Settings tells users to install updates over the app and never uninstall, because uninstalling erases all data (settings_screen.dart:248; l10n.dart:259-261) [80]. Because release APKs are debug-signed, an update installs over an existing app only if it was signed with the same debug keystore, normally the one on the build machine. So an APK built on another machine cannot upgrade an install, and reinstalling loses all data because there is no sync (inferred: build.gradle.kts:41-51 + Android signature check) [50].

## How to test and lint

- `flutter test` → **53 passed, 0 failed**, exit 0, run on 2026-11-05 (verified: evidence/2026-11-05-flutter-test.txt:111-112) [100].
  - 42 pure-Dart logic tests in `test/ledger_logic_test.dart`.
  - 2 controller tests on in-memory sembast in `test/widget_test.dart` (despite the name, it has no widget tests).
  - 9 widget tests in `test/screens_test.dart`: one matrix of flavor x locale x data = 8 cases, plus 1 appearance-switch test (verified: test files; evidence) [100].
- `flutter analyze` → **1 info, exit code 1**: `prefer_conditional_assignment` at lib/services/receipt_layout.dart:74 (verified: evidence/2026-11-05-flutter-analyze.txt) [100]. Lint config is stock `flutter_lints` with no extra rules (verified: analysis_options.yaml:10) [80].
- Both commands run an implicit `pub get`. The working tree stayed clean apart from the existing untracked `graphify-out/` (verified: git status) [100].
- Not covered by any test: the fiscal network lookup, on-device text recognition, notification scheduling, the real database openers, and the add-entry, scan, fiscal-code, onboarding, recurring, shared and receipt-review screens (verified: import comparison of lib vs test) [80].

## History in one paragraph

There are 7 linear commits from two authors (verified: git log) [100]:
- The initial author made v1.0.0 on 2026-09-12, then added flavors (Umbra / Novčanik) and "For them" on 2026-10-26.
- A later author made 5 commits on 2026-10-26/27: cross-store prices, the version in Settings, the Balance tab and demo data, a light/dark redesign with the Inter font, and the senior-friendly Novčanik (1.1.0 → 1.4.0) (verified: git show per commit) [100].
