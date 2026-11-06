---
nexus: 1
id: 01a1ad84-4c6f-77f2-bc67-f9359da26862
kind: spec
title: umbra — architecture
status: active
project: 33fba8bb-d3e8-450e-9b9d-26848c5d8c1a
topic: architecture
created: "2026-11-05T21:01:31Z"
updated: "2026-11-05T22:52:47Z"
source: import
---

_Created by NEXUS. The structure below is read from the repository; the summary is left for you or a reviewed draft._

## Structure (verified from the file system)

- `android/` — 11 entries
- `assets/` — 2 entries
- `fonts/` — 7 entries
- `ios/` — 6 entries
- `lib/` — 10 entries
- `test/` — 3 entries
- `web/` — 4 entries

Conventional entry points present: `lib/main.dart`.

## Architecture summary

_Drafted by Claude in a NEXUS session and accepted on 2026-11-05. Facts marked as inferred still need checking._

# Umbra / Novčanik — architecture

Review status (2026-11-05): verified claims, 5 attack rounds, last round applied by the orchestrator; see overview.md. Unconfirmed by the owner.

HEAD `726e195`, version `1.4.0+14`. One Flutter codebase with two experiences (classic "Umbra", light "Novčanik").

## How this was verified

- Grades: **100** executed (command, probe or saved run) · **80** read at source (file:line) · **50** derived (chain stated). Grade-20 items are in the open questions, not here.
- `(verified: ...)` = read at source or executed. `(inferred: ...)` = derived.
- Two readers ran throw-away Dart probes from a scratch folder against the real `lib/` code. They used a mock HTTP client and no network. Those results are marked [100].
- Terms (rc=1 = search found no match, FAB, IKOF/PIB/crtd, back-fill) are defined in overview.md, Terms.
- Nothing was built; no tracked file changed (see overview.md).

## 1. Layers at a glance

```
lib/main.dart, lib/main_light.dart      entry points -> boot()
lib/app.dart                            UmbraApp (MaterialApp), AppFrame (text scale, width)
lib/state/ledger_controller.dart        LedgerController (ChangeNotifier): all state + writes
lib/state/demo_data.dart                demo seed (button gated by UMBRA_DEMO)
lib/data/                               models, AppDatabase (sembast), conditional DB opener
lib/services/                           pure domain functions + fiscal HTTP + notifications + OCR
lib/screens/                            16 screen files, 17 public screen classes (PriceItemScreen shares prices_screen.dart), plus the private `_RecurringEditor` route; pushed with plain MaterialPageRoute
lib/widgets/                            UmbraScope + widget kit, review/price/entry widgets
lib/theme/                              AppTheme, UmbraTokens, SeniorTextScaler, UmbraFlavor
lib/l10n/l10n.dart                      hand-written L10n (English / Serbian Latin)
```
(verified: git ls-files lib; file heads) [80]

There is no state-management package, no router package and no platform channels (verified: pubspec.yaml dependencies; grep `MethodChannel|EventChannel` rc=1) [80].

Runtime dependencies and where they are used (verified: pubspec.yaml:30-54; grep `package:<name>/` over lib) [100]:

| Package | Job | Used in |
|---|---|---|
| sembast, sembast_web | local database (IO / IndexedDB) | lib/data/app_database.dart, db_open_io.dart, db_open_web.dart |
| mobile_scanner | live QR scan | lib/screens/scan_screen.dart |
| image_picker | photo / gallery pick | lib/screens/scan_screen.dart |
| google_mlkit_text_recognition | on-device OCR | lib/services/ocr/ocr_io.dart |
| http | fiscal lookup POST | lib/services/fiscal_receipt.dart |
| flutter_local_notifications, timezone, flutter_timezone | reminder scheduling | lib/services/notifications_service.dart |
| permission_handler | notification permission request | lib/services/notifications_service.dart (called from ledger_controller.dart:202 and :217) |
| path_provider, path | documents/temp directories | lib/data/db_open_io.dart; ocr_io.dart (path_provider only) |
| package_info_plus | app version in Settings | lib/screens/settings_screen.dart |
| uuid | entity ids | lib/state/ledger_controller.dart |
| intl | date formatting | lib/l10n/l10n.dart, lib/main.dart and several screens/widgets |
| flutter_localizations (SDK) | Material/Cupertino widget strings and date pickers | lib/app.dart:3,35-44 |
| camera | declared, never imported | — |
| cupertino_icons | declared, never used (no `CupertinoIcons` in lib) | — |

## 2. Entry points and boot

- `lib/main.dart:32` is `main() => boot()`. `lib/main_light.dart:4-7` sets `UmbraFlavor.isLight = true`, then calls the same `boot()`. `isLight` is a mutable static (`static var isLight = false;`), not a const or a dart-define (verified: lib/theme/umbra_flavor.dart:4) [80].
- `boot()` order (verified: lib/main.dart:10-29) [80]:
  1. `ensureInitialized`.
  2. Date formatting for `en`. `sr_Latn` and `sr` are loaded inside a try that swallows errors.
  3. Portrait-up lock and edge-to-edge UI.
  4. Register the Inter font licence.
  5. `NotificationService.instance.initialize()`.
  6. `createLedger()`: open the DB, build `LedgerController`, await `load()`.
  7. If reminders are on, `syncSchedule(enabled: true)`.
  8. `runApp(UmbraApp(controller))`.
- `load()` finishes before `runApp`, so the `_Boot` spinner is never shown in production (inferred: lib/app.dart:46-47 + ledger_controller.dart:32-40) [50].
- There is no try around `initialize()` or `createLedger()`. A throw there stops `runApp`, and the native splash stays on screen (inferred: lib/main.dart:24-29; notifications_service.dart:15-32) [50].

## 3. App shell

- `UmbraApp` = `UmbraScope` > `ListenableBuilder(controller)` > `MaterialApp`. Every `notifyListeners` rebuilds the `MaterialApp` (verified: lib/app.dart:21-51) [80].
  - `title` = `UmbraFlavor.appTitle`. `themeMode` comes from `settings.appearance`. `locale` comes from `settings.locale`.
  - `home` = `_Boot` while loading, else `ShellScreen` if onboarded, else `OnboardingScreen`.
  - So finishing onboarding or erasing data swaps the root `home` widget with no route push. Erase and 'load test data' also pop the Settings route (settings_screen.dart:92,112) [80].
- `AppFrame` (verified: lib/app.dart:58-93) [80]:
  - Clamps the system text scale to 0.9-1.3.
  - In the light flavor it clamps to 1.0-1.3 and wraps the result in `SeniorTextScaler`. That scaler enlarges small text by up to 26% (lib/theme/app_theme.dart:473-485).
  - Above 500 px width it centres a 430 px column.
- The 1.75x senior cap can never bind; see open questions (UI) [50].
- `ShellScreen` is an `IndexedStack` with 5 tabs: 0 Home, 1 History, 2 Balance, 3 Prices, 4 Review. The FAB opens a capture sheet (verified: lib/screens/shell_screen.dart:27-143) [80].
- Navigation is plain `MaterialPageRoute` pushes. There are no named routes (verified: grep of push sites) [100].
- The main push graph [100]:
  - Shell (Add sheet) → ScanScreen, AddEntry(expense), RecurringScreen, SharedScreen (classic) (shell_screen.dart:94-141).
  - Home → Settings, Shared (classic), Recurring, EntryDetail, plus light quick tiles.
  - Shared → EntryDetail (shared_screen.dart:138).
  - Recurring → _RecurringEditor (recurring_screen.dart:96,102).
  - History → EntryDetail.
  - Balance → AddEntry(income), EntryDetail.
  - Prices → PriceItemScreen.
  - Review → MonthDetailScreen.
  - EntryDetail → PriceItemScreen.
  - Scan → FiscalCodeScreen / ReceiptReviewScreen.

## 4. State: LedgerController

File: `lib/state/ledger_controller.dart`.

- It is a `ChangeNotifier` with public mutable `settings`, `entries` (newest first), `recurring`, `priceLinks` and `loading` (verified: :16-30) [80].
- Widgets reach it through `UmbraScope`, an `InheritedNotifier`. `UmbraScope.of` registers a dependency, so any reader rebuilds on every notify. `context.l10n` builds `L10n` from `settings.locale` (verified: lib/widgets/umbra_widgets.dart:12-28) [80].
- **Persisting methods:** load, linkPriceItems, unlinkPriceItem, completeOnboarding, updateSettings, addEntry, updateEntry, setItemForOther, setItemForOtherQty, settleOpenShared, deleteEntry, resetAll, addRecurring, updateRecurring, deleteRecurring.
- **Read-only accessors:** getters balance, monthIncome, priceMemory, sharedOverview and others, and methods monthlyReview([month]), weeklyReview([now]), upcomingCharges([now]) (verified: :42-436) [80].
- **Write ordering is not uniform** (verified: :146-158,196-213,249-283,381-393) [80]:
  - Most mutators await the DB write and then change memory.
  - The price-link methods and `completeOnboarding` change memory first and notify after the save.
  - `updateSettings` alone notifies **before** awaiting the save.
- Every mutation assigns a new list or map. The `priceMemory` cache relies on that: it is reused while `entries` and `priceLinks` are `identical()` (verified: :134-144) [80]. Its 180-day window uses `DateTime.now()` at build time, so it goes stale until the next mutation (verified: price_advisor.dart:146) [80].
- **Balance** = `settings.startingBalance` + every income entry − every expense entry, over all dates, future ones included (verified: :42-48) [80]. `settings.monthlyIncome` is never posted and never added to the balance. It is only a fallback month income when no income is logged that month: in `buildMonthlyReview` (review_engine.dart:243-244) and in the `monthIncome` getter (:88-91), which has no callers (grep) [80].
- All ids are uuid v4 from the controller. The DB assigns no keys (verified: :20,237,266,339,408,450) [80].
- No screen calls `updateEntry`. A saved entry can only be deleted, have an item's For-them quantity changed (`setItemForOtherQty`), or have its open For-them items stamped as settled (`settleOpenShared`, ledger_controller.dart:334-378; shared_screen.dart:119) [80].

## 5. Data layer, data model and persistence

### Storage
- sembast 3.8.10 + sembast_web 2.4.6 (verified: pubspec.lock:744-759) [80].
- `lib/data/db_open.dart:1` is one conditional export: `db_open_io.dart`, or `db_open_web.dart` if `dart.library.html` [80].
  - IO: `databaseFactoryIo` at `<app documents>/<UmbraFlavor.dbFileName>`.
  - Web: `databaseFactoryWeb` (IndexedDB) under `dbWebName` (verified: db_open_io.dart:7-12; db_open_web.dart:5-7) [80].
- File names: classic `umbra.db` / `umbra`, light `umbra_light.db` / `umbra_light`. The classic names equal the initial commit's (verified: fe02b59:lib/data/db_open_io.dart:7, fe02b59:lib/data/db_open_web.dart:4) [80]. Inferred: since the classic applicationId also stayed `app.umbra.umbra` (fe02b59 and 551966c android/app/build.gradle.kts), classic installs kept their data across `551966c` [50].
- `AppDatabase.open({factory, name})` uses `name` only when a factory is injected, as tests do with `databaseFactoryMemory` (verified: lib/data/app_database.dart:18-26) [80].

### Stores (verified: lib/data/app_database.dart:10-96) [80]
| Store | Key | Value |
|---|---|---|
| `entries` | entry id | LedgerEntry map |
| `recurring` | item id | RecurringItem map |
| `settings` | `'app'` (single record) | AppSettings map |
| `price_links` | `'links'` (single record) | `{'links': [{from, to}, ...]}` |

- `loadEntries` sorts by `occurredAt` descending, and `loadRecurring` by `createdAt` ascending.
- Writes are single-record puts and deletes, except `clearAll`, which runs four store-wide `store.delete(db)` calls (app_database.dart:91-96) [80]. There are **no transactions**, so `settleOpenShared`, `clearAll` and recurring back-fill are not atomic (verified: grep `transaction` rc=1) [100].

### Model (lib/data/models.dart) [80]
- `LedgerEntry`: id, type (expense | income), title, amount, occurredAt, source (manual | scan | recurring), notes?, items, recurringId?.
- `LineItem`: id, name, normalizedName, quantity, unitPrice, lineTotal, forOther, forOtherQty, settledAt?, settledId?. Getters: theirsQty, sharedAmount, isOpenShared.
- `AppSettings`: onboarded, startingBalance, currencySymbol `€`, symbolAfter, decimalComma, monthlyIncome?, notificationsEnabled, locale `en`, appearance?.
- `RecurringItem`: id, type, title, amount, dayOfMonth (1-31), createdAt, enabled.
- Encoding: enums by `.name`, dates as epoch ms, items as nested maps.

### Schema evolution
- There is no version and no migration (verified: openDatabase has no version; grep rc=1) [100].
- Old records still load through tolerant `fromMap` defaults (quantity 1, prices 0, settings defaults). But required fields are force-cast, and enums use `values.byName`, which throws on an unknown string. `loadEntries` has no try, so **one bad record fails the whole load**, and with it boot (verified: models.dart:166-183; app_database.dart:43) [80]. The boot consequence is inferred [50].

### Money
- `double` in major units everywhere (verified: models.dart:22-24,124,201,205,277) [80].
- Stored amounts are non-negative. The sign comes from `EntryType` (verified: ledger_controller.dart:242,413,453) [80].
- Display rounding is in `formatMoney` (`toStringAsFixed(2)`, money.dart:3-31). Input is rounded too: AddEntryScreen fills its amount and item fields with `moneyInput` (2 decimals) for the scan total, item prices, qty x unit and the item sum (add_entry_screen.dart:57,66-67,125,176; money.dart:50-53) [80], so amounts saved from the form are usually whole cents (inferred, 50: only prefilled and computed fields are rounded; typed values are saved as parsed, add_entry_screen.dart:180-209). Sums of stored doubles still carry float noise (probe) [100].
- `parseAmount`: when both `,` and `.` appear, the later one is the decimal separator. A lone `,` or `.` is always a decimal point, so `1.234` → 1.234 (verified: money.dart:33-48; probe) [100].

## 6. Services (lib/services/)

The domain services are top-level functions over immutable models with no Flutter imports, so they load in a plain Dart VM (probe) [100]. They are not all pure: see the clock reads below. Only `buildMonthlyReview` reads the clock unconditionally (review_engine.dart:247). `buildPriceMemory`, `parseReceiptText`, `parseReceiptLayout`, `parseAnyReceipt` and `parseFiscalCodesFromText` fall back to `DateTime.now()` when no `now` is passed (price_advisor.dart:146; receipt_parser.dart:43; receipt_layout.dart:43,206; fiscal_receipt.dart:361) [80].

Roles [80, read at source]:

| File | Role |
|---|---|
| `money.dart` | format and parse money, chronological ordering, balances, week start |
| `balance_timeline.dart` | month-by-month balance steps; spending runs merged, each income its own step |
| `review_engine.dart` | month snapshots, monthly and weekly review, score (score computed, not rendered), top stores and items |
| `price_advisor.dart` | price keys, 180-day price memory, cross-store comparison, link suggestions |
| `recurring.dart` | due dates, dedupe, upcoming charges |
| `shared_items.dart` | For-them queries and settlements |
| `item_name.dart` | `normalizeItemName` (stored, but not used for logic) |
| `fiscal_receipt.dart` | QR parsing, fiscal HTTP lookup, JSON mapping, typed codes, demo ticket |
| `receipt_parser.dart`, `receipt_layout.dart` | heuristic text and layout parsers for recognised text |
| `ocr/` | conditional OCR engine (ML Kit on Android/iOS, stub elsewhere) |
| `notifications_service.dart` | singleton over flutter_local_notifications |

Notable behaviour:
- **Two normalisers.** `priceKey` (price_advisor.dart:219-294) folds a fixed set of diacritics, normalises sizes to g/ml and returns sorted unique tokens. Any other letter is stripped, including all Cyrillic, so `Млеко 1л` → `1`, which is too short and is dropped from price memory (verified: probe) [100]. `normalizeItemName` keeps Unicode letters and is stored on `LineItem.normalizedName`, but no logic reads it (verified: grep) [80].
- **Price comparison.** It keeps the latest observation per case-insensitive store and flags "cheaper elsewhere" beyond a 0.005 gap (verified: price_advisor.dart:34-60,185-214; probe) [100]. `topStores` in reviews groups by the exact title instead, so `IDEA` and `Idea` count as two stores there (verified: review_engine.dart:338-348) [80].
- **Spending includes For-them items** at full cost. The shared amount is reported alongside and never subtracted (verified: review_engine.dart:332-336 for the sum; :272,313 for `sharedMarked` reported alongside) [80].
- **Timeline vs headline balance.** Entries dated after the current month count in `balance` but not in the timeline (verified: probe) [100]. `buildMonthSnapshots` returns an empty list when every entry is in the future, while the timeline still returns the current month (verified: probe) [100].

## 7. Main data flows

### 7.1 Manual entry
1. AddEntryScreen collects the amount, title, date (2018 to tomorrow), notes and optional items. In every mode, editing qty or unit recalculates the line total (add_entry_screen.dart:120-127,362). Only in manual mode are the items summed into the amount (:128,170-178) (verified) [80].
2. Save needs amount > 0. It keeps items with a name and an amount > 0, then calls `addEntry` and pops (verified: :180-211) [80].
3. `makeItem` rebuilds each item: it calls `normalizeItemName` and recomputes `unitPrice = lineTotal/qty` (verified: ledger_controller.dart:256-275) [80].
4. `addEntry` applies `abs()`. An empty title becomes the localised Expense/Income (verified: :239-242) [80].

### 7.2 QR fiscal lookup
1. ScanScreen uses mobile_scanner, back camera, QR only. Non-fiscal QR codes are skipped (verified: scan_screen.dart:32-55) [80].
2. `parseFiscalQr` needs a scheme and a host ending in `tax.gov.me`. It merges query and fragment parameters, with the fragment winning (verified: fiscal_receipt.dart:36-63; probe) [100]. It needs `iic`, `tin` and either `crtd` or `dateTimeCreated` (fiscal_receipt.dart:51-52) [80]. (For IKOF, PIB and crtd, see overview.md Terms.)
3. `fetchFiscalReceipt` makes one form POST of `iic`, `dateTimeCreated` and `tin` to `<QR origin>/ic/api/verifyInvoice`, with a 20 s timeout and no retries (verified: :149-172; MockClient probe) [100].
4. Errors map to three hard-coded English messages: could not read, not found (codes can expire after 90 days), need a connection (verified: :174-208; probe) [100].
5. The JSON maps to a `ParsedReceipt` (store, address, items, total, date). Lines with an amount ≤ 0 (discounts, returns) are dropped. Without `priceAfterVat` the price falls back to the VAT-exclusive one (verified: :85-147) [80].
6. `pushReplacement(ReceiptReviewScreen)` follows (verified: scan_screen.dart:224-238) [80].

### 7.3 Photo / gallery (OCR)
1. `pickImage` (max width 2200). `_ingestFile` then runs a cascade (verified: scan_screen.dart:94-182) [80]:
   - a. A QR in the image → fiscal lookup. If that lookup fails, neither OCR nor code entry is tried.
   - b. Recognised text gives full IKOF + PIB + date → automatic lookup at `https://mapr.tax.gov.me`. On failure, a pre-filled FiscalCodeScreen opens.
   - c. Only an IKOF → pre-filled FiscalCodeScreen.
   - d. Otherwise `parseAnyReceipt` runs locally, and review opens with the photo preview.
2. OCR engine (verified: lib/services/ocr/ocr_io.dart:37-116) [80]:
   - The image is normalised to an 1800 px PNG, written to a temp file, and passed to an ML Kit `TextRecognizer(script: latin)`.
   - It retries the original bytes once if the normalised pass is empty.
   - The temp file is deleted afterwards. The result is a list of lines with bounding boxes.
3. Parsing (verified: receipt_layout.dart:172-268; receipt_parser.dart:42-509) [80]:
   - A text parser (Serbian/Montenegrin and English/German keywords) always runs. A row-clustering layout parser also runs, and the two results are merged by a score, only when OCR returned 3 or more spans (receipt_layout.dart:211-213).
   - Dates are read D/M/Y, M/D/Y when the middle field is >12, or Y/M/D (receipt_parser.dart:413-453). A receipt with no time gets the current hour and minute, with seconds 0 (receipt_parser.dart:418-440).
4. FiscalCodeScreen (typed codes) checks a 32-hex IKOF and a PIB of at least 8 digits. It always posts to `kMaprOrigin`, with the device's UTC offset in the date, and on success calls `pushAndRemoveUntil(review, isFirst)` (verified: fiscal_code_screen.dart:72-101; fiscal_receipt.dart:290,310-320) [80].

### 7.4 Review and save
- `ReceiptReviewScreen` is a stateless adapter. It returns `AddEntryScreen(type: expense, source: scan)` pre-filled with store, total, date and items. Notes = the address, plus the raw text when OCR found no items (verified: receipt_review_screen.dart:20-66) [80].
- With `source == scan`, items are **never summed into the amount**. The field is labelled "total on ticket", and there is no mismatch warning (verified: add_entry_screen.dart:128,171,264) [80]. The image is never persisted (verified: ledger_controller.dart:225-234) [80].

### 7.5 Prices
- `buildPriceMemory` takes expense line items from the last 180 days, groups them by resolved price key (after user links), and drops keys under 3 characters (verified: price_advisor.dart:140-183; probe) [100]. `latestByStore` then keeps the newest observation per case-insensitive store (verified: price_advisor.dart:34-48,216-217) [80].
- Links are a flat map from key to root, persisted in `price_links` (verified: price_advisor.dart:327-375; app_database.dart:72-89) [80].
- `suggestionsFor` proposes links by word overlap ≥ 0.5. Disjoint sizes score 0 (verified: probe) [100].

### 7.6 Recurring charges and notifications
- **Due date** = 09:00 local on `dayOfMonth`, clamped to the month length (verified: recurring.dart:3-7; probe) [100].
- **Back-fill.** `_postDueRecurring` posts every missed month from `createdAt` to now, with `source: recurring` and the item's type. The creation month counts when the creation **day** is not after the due day; times are ignored (verified: recurring.dart:9-33; probe) [100].
- **When it runs:** only in `load()`, `addRecurring` and `updateRecurring`. The enable switch also goes through `updateRecurring`. There is no timer or lifecycle hook (verified: ledger_controller.dart:37,419,429; grep) [80].
- **Dedupe** is by `recurringId` plus year and month among *current* entries. Inferred: a deleted auto-post comes back on the next launch, and re-enabling an item back-fills all missed months [50].
- **Notifications** (verified: notifications_service.dart:43-93) [80]:
  - `syncSchedule` cancels everything, then schedules ids 11 (daily 20:30), 12 (Sunday 10:00) and 13 (1st of the month, 10:15) on channel `umbra_local`, all `inexactAllowWhileIdle`.
  - The texts are hard-coded English. The channel name and the weekly body say 'Umbra'. There is no tap handler.
- **Time zone** comes from `flutter_timezone`. On failure `tz.local` stays UTC (verified: notifications_service.dart:17-23; timezone package env.dart:56) [80].
- **Permission prompts.** On iOS the first prompt appears at first launch, from the default `DarwinInitializationSettings` in `initialize()` (verified: notifications_service.dart:25-29; plugin source) [80]. Finishing onboarding with reminders on, or turning them on later in Settings, asks through permission_handler (verified: ledger_controller.dart:200-205,214-222) [80].

### 7.7 For them (shared items, classic only)
- An item is shared when `LineItem.forOther` is set, optionally with `forOtherQty` for part of the quantity. `sharedAmount` = theirs × unit price, or a pro-rated `lineTotal` (verified: models.dart:25-49; probe) [100].
- The UI uses only `setItemForOtherQty`. A quantity ≤ 0 unmarks the item. `setItemForOther` has no callers (verified: grep) [80].
- `settleOpenShared` stamps all open items with one `settledAt` and one batch uuid. With "add to balance" it also adds an income entry titled `theyPaidYou` (verified: ledger_controller.dart:334-378; shared_screen.dart:119) [80].
- In the light flavor every entry point is hidden. `SharedScreen` itself has no flag; it is simply unreachable (verified: grep isLight) [80].

## 8. Theme, flavor and localisation

- **Tokens.** `UmbraTokens` is a `ThemeExtension` with const `dark` and `light` palettes. Widgets read it via `context.colors`. Material 3 is built from the tokens, with the Inter fonts (verified: lib/theme/app_theme.dart:8-235) [80].
- **Senior mode** = `AppTheme.senior` = `UmbraFlavor.isLight`. It is not a setting (verified: app_theme.dart:174; grep) [100]. It changes:
  - `clearer()` tokens (higher-contrast `border`, `textMuted` and `textFaint`; everything else is unchanged) (app_theme.dart:112-131);
  - NavigationBar height 84 vs 66;
  - buttons 60/50 vs 52/42;
  - taller rows and segmented controls;
  - labelled buttons in place of icon buttons;
  - "about" text on History/Balance/Prices/Review;
  - a 2x2 quick-action grid on Home (verified: app_theme.dart:112-130,310-331; umbra_widgets.dart:236,246,332,421; home_screen.dart:85-116) [80].
- **Accessibility gaps.** There is no `Semantics` anywhere in `lib/`. `Segmented<T>` is plain `GestureDetector` + `Text` (verified: grep rc=1; umbra_widgets.dart:452-476) [80].
- **Scan screen** always renders dark (verified: scan_screen.dart:243-254) [80].
- **Localisation** is a hand-written const `L10n` class (verified: lib/l10n/l10n.dart:6-19; no l10n.yaml; 281 `t(` lines) [100]:
  - Each string is `t(en, srLatn)`. Locale `sr` gives Serbian Latin; anything else gives English.
  - Alongside it, `MaterialApp` sets `locale: l10n.materialLocale`, `supportedLocales` en and sr-Latn, and the Global Material/Widgets/Cupertino delegates from flutter_localizations, so framework widget strings follow the same setting (verified: lib/app.dart:35-44; lib/l10n/l10n.dart:15-17) [80].
  - The locale comes only from the stored setting. There is no device-locale detection (verified: grep) [80].
  - Fiscal errors and notifications are not localised (verified: fiscal_receipt.dart:176-207; notifications_service.dart:51-86) [80].
  - In-app strings interpolate `UmbraFlavor.appTitle`, so the in-app brand follows the Dart flag, not the Android flavor (verified: l10n.dart:219-220 and others) [80].

## 9. Platform boundaries

- **Two conditional exports with different tests** (verified: lib/data/db_open.dart:1; lib/services/ocr/ocr.dart:2) [80]:
  - DB: io by default, web `if (dart.library.html)`.
  - OCR: stub by default, ML Kit `if (dart.library.io)`.
  - Inferred: under `--wasm`, `dart.library.html` is unavailable, so the io DB opener would be picked and fail, while OCR would correctly use the stub [50].
- **ML Kit** only on Android/iOS: `isSupported = Platform.isAndroid || Platform.isIOS` (verified: ocr_io.dart:11) [80].
  - Android bundles the text-recognition 16.0.1 and barcode-scanning 17.3.0 models (verified: android/app/build.gradle.kts:66; pub-cache mobile_scanner-7.4.1/android/build.gradle.kts:73-79, bundled unless `useUnbundled`; no such property in android/gradle.properties) [80].
  - The `com.google.mlkit.vision.DEPENDENCIES` meta-data is therefore likely inert (inferred) [50].
  - Only the Latin script model is used (verified: lib/services/ocr/ocr_io.dart:79) [80].
- **Network boundary:** only `fetchFiscalReceipt` (package:http) (verified: grep over lib) [100]. Android `usesCleartextTraffic=false` (verified: android/app/src/main/AndroidManifest.xml:16) [80]. iOS has no ATS exception: ios/Runner/Info.plist has no `NSAppTransportSecurity` key (verified: grep rc=1) [100].
  - Web only: Flutter loads CanvasKit from www.gstatic.com by default (flutter_tools runner/flutter_command.dart:810-815; build_system/targets/web.dart:122), and mobile_scanner 7.4.1 loads zxing-wasm from cdn.jsdelivr.net when the browser has no native BarcodeDetector (mobile_scanner_web.dart:465-489; zxing_wasm_barcode_reader.dart:41-42) [80, package source; not run]. So on web, "nothing leaves this device" is also untrue for these loads.
- **Native shells** are stock: an empty `FlutterActivity`, the default AppDelegate/SceneDelegate, iOS 13.0 (verified: android/app/src/main/kotlin/app/umbra/umbra/MainActivity.kt:5; ios/Runner/AppDelegate.swift:5-15; ios/Runner/SceneDelegate.swift:4-6; ios/Runner.xcodeproj/project.pbxproj:363,489,540 `IPHONEOS_DEPLOYMENT_TARGET = 13.0`) [80].
- **Android build:** compileSdk 37, Java/Kotlin 17, desugaring, `kotlin.incremental=false` (gradle.properties:8; the reason is recorded in the 67f9323 commit message: "disable Kotlin incremental builds so Windows builds work when the pub cache is on another drive" (git show -s 67f9323) [80]), release debug-signed, no minify (verified: android/app/build.gradle.kts:8-67; gradle.properties:8) [80].
- **iOS** has no flavors: only the `Runner` scheme, bundle id `app.umbra.umbra` (project.pbxproj:385), display name "Umbra" (ios/Runner/Info.plist:15-16). `AppIcon-classic` exists but is unused: all three configs use `AppIcon` (verified: project.pbxproj:376,555,577) [80]. Inferred: with pubspec `default-flavor: classic`, iOS builds fail at scheme lookup [50].
- **Orientation:** Dart locks portrait-up. iOS Info.plist still lists landscape (verified: lib/main.dart:16-18; Info.plist:62-74) [80].

## 10. Key decisions as observed in code

These are visible in the code. The reasons are mostly not recorded, so each one is observed, not confirmed. Exceptions: the 726e195 commit message records that Novčanik targets older people, and the 67f9323 message records why `kotlin.incremental=false` (git show -s 726e195; git show -s 67f9323) [80].

1. **Local-only sembast store**, with no account, sync, backup, schema version or migration. The schema evolves through tolerant `fromMap` defaults (verified: §5) [80/100].
2. **Flavor = Android productFlavor + a separate Dart entry point.** It is a mutable static flag, not `appFlavor` or a dart-define. Tests flip it per case (verified: umbra_flavor.dart:4; test/screens_test.dart:60-61) [80].
3. **Separate databases per flavor**, keeping the original classic names (verified: §5) [80]. Inferred: so existing classic data survives [50].
4. **Senior accessibility bound to the Novčanik flavor**, not a user setting (verified: app_theme.dart:174) [80].
5. **Hand-written L10n** with `t(en, srLatn)`, instead of gen-l10n/ARB (verified: §8) [100].
6. **No state or router packages.** One `ChangeNotifier` with write-through persistence, reached through an `InheritedNotifier`, plus plain `MaterialPageRoute` (verified: §3-4) [80].
7. **Flutter-free domain functions** in `lib/services/`, loadable in a plain Dart VM; some read the clock (§6) (verified: probe) [100].
8. **Money as `double`** in major units, rounded to 2 decimals for display and when AddEntryScreen fills its fields (verified: §5) [80].
9. **The fiscal lookup is the only network call.** Its origin comes from the QR, and typed or OCR codes are pinned to `mapr.tax.gov.me` (verified: §7.2-7.3) [100].
10. **Scanned receipts trust the ticket total.** Items are not summed (verified: §7.4) [80].
