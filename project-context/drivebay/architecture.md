---
nexus: 1
id: 01a1b12f-d4ff-7438-abb0-8e20d5dc86ef
kind: spec
title: drivebay — architecture
status: active
project: ec39f27a-4d00-4e5d-9309-0919b95507de
topic: architecture
created: "2026-11-06T14:07:44Z"
updated: "2026-11-06T16:13:10Z"
source: import
---

_Created by NEXUS. The structure below is read from the repository; the summary is left for you or a reviewed draft._

## Structure (verified from the file system)

- `app/` — 13 entries
- `bootstrap/` — 3 entries
- `config/` — 38 entries
- `database/` — 5 entries
- `docker/` — 5 entries
- `docs/` — 12 entries
- `howphotos/` — 16 entries
- `lang/` — 3 entries
- `notification_images/` — 2 entries
- `public/` — 12 entries
- `resources/` — 5 entries
- `routes/` — 5 entries
- `scripts/` — 1 entry
- `storage/` — 3 entries
- `tests/` — 6 entries

Conventional entry points present: `artisan`, `public/index.php`.

## Architecture summary

_Drafted by Claude in a NEXUS session and accepted on 2026-11-06. Facts marked as inferred still need checking._

# DriveBay — architecture summary (draft v3, replaces v1 and v2)

Markers: *(verified: file)* = seen in the code; *(inferred)* = my conclusion; *(MAIW)* = from `D:\code\maiw\memory\drivebay`, not re-checked.

## Components

- **One Laravel 12 app with domain modules.**
  - 21 domains in `app/Domains/{Domain}/` hold services, jobs, console commands, DTOs, events and policies.
  - Models sit in a parallel tree, `app/Models/Domains/{Domain}/Models/`. There are 39 API controllers *(verified: ls)*.
- **Routing** *(verified: bootstrap/app.php, routes/)*:
  - `web.php`: the marketplace closure is mounted at `/` (English) and again at `/sr` with route names prefixed `sr.`.
  - `storefront.php`: dealer hosts.
  - `api.php` → 11 files in `api/v1/`.
- **Middleware** *(verified: bootstrap/app.php, AppServiceProvider)*:
  - `ResolveDealerStorefrontHost` runs first on every web request. It sets a request-wide `StorefrontContext` to marketplace or standalone, or returns 404 for an unknown host.
  - Then `SetLocale`, `EnsureEngagementVisitor` and `HandleInertiaRequests` run. The API adds `SetApiLocale`.
  - API rate limiters: `api-auth` 10/min per IP, `api-search` 60/min, `api-analytics` 120/min, `api-write` 30/min, `api-import-status` 180/min, plus one for the AI fuel estimate.
- **Admin:**
  - Filament 5 panel at `/admin` in SPA mode, open only to staff (`User::canAccessPanel` → `StaffAccessService`).
  - 32 resources and 3 dashboard widgets (attention, import moderation, stats).
  - 10 custom pages, among them platform, watermark and geography settings, taxonomy imports, fuel-price sync and a push tester.
  - `ArtisanCommands` and `DeveloperTools` are `super_admin`-only; ArtisanCommands runs commands from a fixed list through `Artisan::call`, not a shell.
  - *(verified: AdminPanelProvider, User.php, app/Filament/Admin/Pages)*
- **Frontend:**
  - Inertia + Vue: 41 pages, four layouts (App, Guest, Error, DealerStorefront), about 100 components, 24 composables.
  - Translations arrive as shared props passed through `brandify()`; there is no vue-i18n.
  - Only 2 files use axios *(verified: ls, grep, HandleInertiaRequests)*.
- **Helpers:**
  - `brand()` / `brandify()`, with packs in `config/brands/`.
  - `platform_config()`: boolean keys resolve through Pennant flags in global scope; other keys use a stored admin override, falling back to config *(verified)*.

## Data flow

1. **Creating a listing (web)** *(verified: SellerListingController, ListingMediaService, ProcessListingMediaJob, ListingPublishCoordinator)*:
   - `createDraft(CreateListingDTO)` → link to the dealer → sync equipment.
   - **With photos:** the status becomes `processing` and each photo is queued as a `ProcessListingMediaJob` (watermark + variants). When the last asset finishes, `ListingPublishCoordinator` publishes the listing with auto-approve.
   - **Without photos:** it is published immediately.
2. **Publishing** *(verified: ListingService)*:
   - Auto-approve → `active`/`approved`, then the `ListingPublished` event and `SyncListingSearchDocumentJob`.
   - Otherwise → `pending_review` and a notification to staff.
   - The manual publish action auto-approves only `private` users.
   - Statuses are plain strings: `draft`, `processing`, `pending_review`, `active`, `rejected`, `sold`, `archived`.
3. **Imports (Autodiler):** `ListingModerationPipelineService` + `ListingAutoPublishEvaluator` decide, guided by the photo-approval and auto-publish platform settings *(verified)*.
4. **Events:**
   - The only domain event is `ListingPublished`; its listeners are the seller notification and saved-search matches.
   - The observers are `NotificationObserver` (sends a push for in-app rows, per MAIW) and `AdvertisementObserver`.
   - Everything else is direct service calls and jobs *(verified: AppServiceProvider)*.
5. **Search:**
   - `ListingSearchDocument` rows are indexed in Meilisearch.
   - Filterable and sortable attributes are set in `config/scout.php` and take effect only after `composer run search:setup`.
   - Sort scores: freshness, popularity, quality, recommendation and boost *(verified: config/scout.php, ListingSearchDocumentBuilder)*.
6. **Recommendations** *(verified: config/recommendations.php, routes/console.php)*:
   - The listing score is a weighted blend of capped signals: views 0.20, favorites 0.15, leads 0.15, phone 0.10, freshness 0.15, boost 0.15, quality 0.10.
   - Candidates come from content-based (20), collaborative (15) and trending (10) sources, weighted 0.75, 0.65 and 0.55. Similarity has a 0.3 minimum.
   - Everything is rebuilt nightly from 03:00 to 04:00. All tuning lives in config.
7. **Payments:** the gateway comes from `platform_config('billing.gateway')`. It returns Paddle or Stripe only when that provider's keys are set; **otherwise it silently returns `FakePaymentGateway`** *(verified: AppServiceProvider:46-58)*.
8. **Errors:** web 403/404/500/503 render the Inertia `Error` page. The API returns JSON in a `{data, meta}` envelope *(verified)*.

## Boundaries and integrations

- **API:**
  - Sanctum bearer tokens, the `Accept-Language`, `X-Visitor-Id` and `X-Analytics-Source` headers.
  - Scramble exports `docs/api/v1/openapi.json` (122 operations).
  - 9 API controllers reuse `Requests\Web\*` despite the rule that web and API are separate contracts *(verified)*.
- **Datastores:** PostgreSQL 16, Redis 7, Meilisearch 1.12. 149 migrations; 129 tables in the DBML *(verified)*.
- **External services** *(verified: env names in config/, Gateways, Publishers)*:
  - Payments: Stripe and Paddle.
  - Firebase FCM.
  - Sentry and Telescope.
  - Socialite logins: Google, Facebook, Apple.
  - OpenAI, for fuel-consumption estimates.
  - Meta Graph, for Instagram, with fake publishers for dev.
  - Vehicle data: mobile.de, plus CarQuery and NHTSA (legacy).
  - Autodiler and the Montenegro fuel-price feed.
- **Deployment** *(verified)*:
  - One Docker image with three targets: web (php-fpm + nginx), worker (Horizon) and scheduler.
  - Coolify runs the app, Horizon, Postgres, Redis and Meilisearch; the scheduler is a cron entry.
  - Horizon has two supervisors: `default`, and `autodiler-imports` on `redis-imports`.

## Key decisions

- **A modular monolith by domain,** able to move to services later *(verified: docs/architecture/system-overview.md)*.
- **Brands are packs chosen at deploy time** (`BRAND`) *(MAIW; packs verified)*.
- **Locale lives in the URL** for the web and in a header for the API *(verified)*.
- **Long imports get their own queue, connection and supervisor** (KAN-7): timeouts ordered 900 s < 960 s < 1200 s *(verified)*.
- **Publish first, moderate after** for most listings *(code verified; policy inferred)*.
- **Image processing is custom.** Spatie Media Library was removed (KAN-16) *(verified: composer.json)*.
- **Paddle is a third gateway,** not Cashier. **Spatie roles are the source of truth for staff** *(MAIW; code verified)*.

## Risks and open questions

- **Silent fake payments** when a gateway key is missing. No test covers the selection logic *(verified: grep of tests)*.
- **Dealers' manual publish goes to `pending_review`,** while creating a listing auto-approves everyone. Is that intended?
- **The DBML has drifted,** although the repo calls it "the only truth":
  - it lacks app tables that migrations create: `advertisements`, `database_notifications`, `email_verification_codes`, `engagement_campaign_target_*`, `fuel_consumption_estimates`, `recommendation_profile_*`, `user_device_tokens`;
  - its role tables don't match Spatie's (`model_has_roles`, `role_has_permissions`);
  - the migrations still create the unused `media` table.
- **The API docs have drifted** *(verified: comparison of module docs, OpenAPI and routes)*:
  - The module docs leave out about 12 real endpoints: password reset and forgot-password, email verification and resend, account restore, delete and password change, Autodiler import (preview, start, status), dealer review POST, viewing reschedule, and health.
  - `viewing.md` documents `PUT /seller/listings/{publicId}/viewing`, which has no route.
  - OpenAPI (exported 2026-08-03) lacks `POST /account/complete-profile` and `POST /billing/paddle/complete`.
- **SSR in production?** It runs in dev, but the web container runs no Node process *(inferred)*.
- **Test setup:** tests use SQLite in memory with a sync queue, while production uses Postgres and Redis. Coverage is mostly Feature tests: 21 API, 9 Web and about 91 top-level files *(verified)*.
- **Pennant registration:** `AppServiceProvider` defines 13 of the 15 feature classes *(verified)*.
