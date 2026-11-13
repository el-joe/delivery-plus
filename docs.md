# MASTER PROMPT — Multi-Vehicle Ride & Delivery Platform

> How to use: put this file in the root of an empty workspace and tell Claude:
> "Read MASTER_PROMPT.md and start Phase 0." Work one phase per session.

---

## 1. Your role

You are the lead engineer building a production-grade ride-hailing and goods-delivery platform for the Egyptian market, similar to inDrive but covering every vehicle type (passengers and cargo). You own architecture, code quality, and delivery. I am the technical lead (senior Laravel developer); I review your work and make product decisions.

Build it in phases (section 13). At the end of each phase, stop, summarize what was built, how to run and test it, and wait for my approval before starting the next phase. Do not try to build everything in one pass.

## 2. Product summary

- One mobile app with two completely different interfaces depending on account type: **Rider** (customer) and **Driver**.
- One admin dashboard (web) that controls everything: users, drivers, vehicles, trips, pricing, finance, support, settings.
- Riders request a trip or a delivery, choose a vehicle type, and see a suggested fare calculated from the admin's per-kilometer price. Nearby drivers accept that fare or send a counter-offer (bid). The rider picks an offer, the trip is tracked live, then paid and rated.
- Languages: Arabic (default, full RTL) and English. Currency: EGP. Timezone: Africa/Cairo.

### Vehicle types (seeded, fully manageable by admin without an app update)

| Key | Arabic | Service |
|---|---|---|
| private_car | ملاكي | passengers |
| tuktuk | توكتوك | passengers |
| motorcycle | موتوسيكل | passengers / small parcels |
| microbus | ميكروباص | passengers (group) |
| quarter_truck | ربع نقل | cargo |
| half_truck | نصف نقل | cargo |
| truck | نقل | cargo |
| furniture_truck | نقل عفش | cargo |
| heavy_truck | نقل ثقيل | cargo |
| heavy_equipment | معدات ثقيلة | cargo / equipment |

Each type has: names (ar/en), icon, image, service kind (passenger / cargo / both), max passengers, max load (kg), required driver documents, pricing fields (section 6), commission rate, active flag, sort order.

## 3. Hard constraints (never violate)

**Backend and dashboard**
- Latest stable Laravel and PHP, MySQL 8, Redis (cache, queues, driver geo index).
- **Never use Livewire. Never use Filament, Inertia, Vue, or React.** The dashboard is Blade + Tailwind CSS + plain vanilla JavaScript (ES modules, built with Vite).
- Allowed dashboard libraries: Yajra DataTables (server-side), Select2, ApexCharts, Flatpickr, SweetAlert2, Google Maps JS. Because DataTables and Select2 require jQuery, jQuery is allowed only as their dependency; all custom code is vanilla JS.
- Architecture: Controllers → Form Requests → DTOs → Services → Repositories (interfaces + Eloquent implementations) → Models. API Resources for every response.
- Auth: Laravel Sanctum tokens for the mobile API; session guard `admin` for the dashboard.

**Mobile**
- Flutter (latest stable), one codebase, Android and iOS, a single app containing both Rider and Driver experiences.

**General**
- Do not add a dependency that is not listed here without asking me first.
- Verify the current versions and APIs of every package against its official documentation before using it; do not rely on memory.
- No secrets in the repository. Everything configurable through `.env` and the admin settings module.
- No placeholder code, no `TODO` left behind inside a finished phase, no fake data paths in production code.

## 4. Accounts and authentication

- On the first screen the user chooses **Rider** or **Driver**. That choice decides which interface the app loads after login.
- Register: name, phone (required, Egyptian format, unique), email (optional, unique), password, account type.
- Login: one identifier field that accepts **email or phone**, plus password, plus account type.
- One `users` table with a `type` column (`rider`, `driver`). Driver-specific data lives in `driver_profiles`. Admins are a separate `admins` table and guard.
- Phone verification by OTP is implemented behind a setting (`auth.require_phone_verification`) with an `SmsProvider` interface and a log driver for development. Forgot/reset password uses the same OTP channel.
- Token handling: Sanctum access token with device name, logout from current device and from all devices, FCM device token registration endpoint.
- Rate limiting on all auth endpoints. Passwords hashed with the framework default.
- Account states — rider: active, suspended, banned. Driver: pending_documents, under_review, approved, rejected, suspended, banned, documents_expired. A driver can log in but cannot go online until approved.

## 5. Core domain rules

### 5.1 Trip lifecycle (state machine — enforce transitions in one service class)

`pending` → `offered` → `accepted` → `driver_arrived` → `in_progress` → `completed`

Terminal branches: `cancelled_by_rider`, `cancelled_by_driver`, `cancelled_by_admin`, `expired`.

- Every transition is validated, wrapped in a DB transaction, logged in `trip_status_logs` (actor, from, to, timestamp, location), and fires a domain event that drives notifications and broadcasting.
- A request expires after a configurable duration (default 10 minutes) through a delayed queued job.
- A driver can have only one active trip; a rider can have only one active request. Guard with row locking so two riders cannot accept the same driver and two drivers cannot be accepted on the same trip.
- Cancellation rules and fees are configurable: free before acceptance, fee after acceptance, driver penalty counters after repeated cancellations.
- Trips support `now` and `scheduled` (scheduled is built in a later phase; design the schema for it now).
- Cargo trips carry extra fields: cargo description, approximate weight, helpers needed (yes/no and count), photos, receiver name and phone, proof of delivery (photo or code).

### 5.2 Matching and bidding

- On trip creation, find online, approved drivers of the requested vehicle type within a configurable radius (per vehicle type) using Redis GEO, and notify them through push + realtime channel.
- A driver may accept the rider's price as-is or send one active bid per trip (price + optional note + ETA). Bids expire after a configurable time.
- The rider sees bids live (price, driver rating, completed trips, vehicle, ETA, distance) and accepts one; all other bids are rejected automatically.
- Rider may raise their offered price while the request is pending.

## 6. Pricing engine (admin-controlled)

The admin sets, per vehicle type (with optional per-zone override):

- `base_fare`
- `price_per_km`  ← the main control
- `price_per_minute` (may be 0)
- `minimum_fare`
- `commission_rate` (%)
- `min_bid_percent` and `max_bid_percent` (allowed rider/driver price range around the suggested fare)
- `search_radius_km`

Formula:

```
suggested = max(minimum_fare, base_fare + distance_km * price_per_km + duration_min * price_per_minute)
suggested = suggested * surge_multiplier(zone, time)      // default 1.0
final     = accepted price (rider offer or accepted bid) - promo discount
commission = final_before_discount * commission_rate
driver_net = final_before_discount - commission
```

- Distance and duration come from a `RoutingProvider` interface (Google Routes implementation plus a haversine fallback driver for tests and local development).
- Implement pricing as a pure, fully unit-tested service (`FareCalculator`) that takes a DTO and returns a breakdown DTO. Store the full breakdown as a snapshot on the trip so later price changes never alter old trips.
- Money is stored as integers in piastres (or `decimal(12,2)` handled through a Money value object) — never floats.
- Every pricing change is recorded in an audit log (who, when, old values, new values).

## 7. Payments and wallet

- Methods: cash (MVP), wallet, online card/wallet through a `PaymentGateway` interface (Paymob implementation in a later phase).
- Wallet is a ledger: `wallets` + immutable `wallet_transactions` (type, amount, balance_after, reference morph, description). Balance is never updated without a ledger row, always inside a transaction with a lock.
- Cash trips: the driver collects the full fare, so the platform commission is debited from the driver's wallet. A driver whose wallet falls below a configurable negative limit cannot go online until they top up.
- Driver withdrawal requests (bank / InstaPay / Vodafone Cash) with admin approve/reject flow.
- Promo codes: percent or fixed, max uses, per-user limit, validity dates, restricted vehicle types, minimum fare.

## 8. Backend architecture

```
app/
  Http/
    Controllers/Api/V1/{Auth,Rider,Driver,Shared}/
    Controllers/Admin/
    Requests/        Resources/        Middleware/
  DTOs/              # readonly classes, built with fromRequest()/fromArray()
  Services/          # business logic, one responsibility each
  Repositories/
    Contracts/       # interfaces
    Eloquent/        # implementations, bound in RepositoryServiceProvider
  Models/            Enums/            Events/          Listeners/
  Jobs/              Notifications/    Policies/        Observers/
  Actions/           # single-purpose operations reused by services
  Support/           # Money, GeoPoint, helpers, ApiResponse
  Integrations/      # Sms/, Routing/, Payment/, Push/ — interface + drivers
  DataTables/        # Yajra classes
```

Rules:
- Controllers are thin: validate (Form Request) → build DTO → call service → return Resource. No queries and no business logic in controllers or Blade.
- Services depend on repository interfaces, never on Eloquent directly. Repositories contain no business rules.
- PHP backed enums for every status and type. Strict types everywhere. Typed properties and return types.
- Policies for authorization; `spatie/laravel-permission` for admin roles and permissions.
- Unified JSON envelope for every API response:
  `{ "success": bool, "message": string, "data": mixed, "errors": object|null, "meta": object|null }`
  with a global exception handler that maps validation, auth, not-found, domain, and server errors to that shape and correct HTTP codes.
- API versioned under `/api/v1`, localized by `Accept-Language` (ar/en), paginated lists, filtering and sorting through query params, idempotency key support on trip creation and payments.
- Realtime: Laravel Reverb with private channels — `trip.{id}`, `rider.{id}`, `driver.{id}`, `drivers.zone.{id}`, and `admin.live-map`. Driver location: app sends location every few seconds; backend writes to Redis GEO, broadcasts on the active trip channel, and persists throttled points to `trip_locations`.
- Push: FCM through a `PushProvider` interface; every notification is also stored in the database (in-app inbox) and is translatable.
- Queues with Horizon, scheduler for expirations and document-expiry reminders (30 days and 3 days before), Sentry-ready error reporting, activity log for admin actions.
- File uploads (documents, avatars, vehicle photos, cargo photos) through the filesystem abstraction, private disk for identity documents with temporary signed URLs.
- API documentation generated with Scribe, plus an exported Postman collection, kept up to date every phase.

### Database (design, then show me the ERD before migrating)

At minimum: `users`, `admins`, `driver_profiles`, `vehicle_types`, `vehicle_type_pricing` (with nullable zone), `vehicle_type_document_requirements`, `vehicles`, `driver_documents`, `zones` (polygon), `surge_rules`, `trips`, `trip_bids`, `trip_status_logs`, `trip_locations`, `trip_cargo_details`, `payments`, `wallets`, `wallet_transactions`, `withdrawal_requests`, `promo_codes`, `promo_code_usages`, `ratings`, `saved_addresses`, `support_tickets`, `support_ticket_messages`, `chat_messages`, `notifications`, `device_tokens`, `cancellation_reasons`, `settings`, `pages` (terms, privacy, FAQ), `audit_logs`, plus permission tables.

Foreign keys, proper indexes (status, type, foreign keys, geo columns, created_at), soft deletes where history matters, factories and seeders for everything (including realistic Egyptian demo data and the vehicle types table above).

## 9. Admin dashboard

Layout: responsive sidebar layout, Arabic RTL by default with English switch, light/dark mode, reusable Blade components (`x-admin.*`: card, table, modal, form inputs, badge, stat, page header, confirm dialog). Tailwind config holds the design tokens from section 11. Each page has its own small vanilla JS module; shared modules for ajax (fetch wrapper with CSRF), toasts, modals, DataTables defaults, Select2 defaults (including ajax search).

Modules:
1. Dashboard — KPIs (trips, revenue, commission, active drivers, new users), charts by period, latest trips, pending reviews.
2. Live map — online drivers and active trips in realtime, filter by vehicle type and zone.
3. Riders — list, profile, trips, wallet, suspend/ban, notes.
4. Drivers — list, profile, vehicles, documents, earnings, ratings, suspend/ban.
5. Driver verification queue — review each document, approve, reject with reason, request re-upload.
6. Vehicle types — CRUD, required documents per type, images, ordering, activation.
7. **Pricing** — per vehicle type and per zone: base fare, price per km, price per minute, minimum fare, commission, bid range, search radius; with change history.
8. Zones and surge — draw polygons on the map, surge rules by time window.
9. Trips — full list with filters, detail page (timeline, map route, bids, payment, chat), admin cancel, export.
10. Finance — payments, wallet transactions, commission report, withdrawal requests, manual wallet adjustment (with reason, audited).
11. Promo codes.
12. Ratings and reviews.
13. Support tickets — assignment, replies, statuses.
14. Notifications — send push to segment (all riders, all drivers, vehicle type, zone, single user).
15. Content — terms, privacy, FAQ, onboarding slides, banners.
16. Settings — general, auth, trip (expiry, radius, cancellation fees), wallet limits, integrations keys, app versions / force update, maintenance mode.
17. Admins, roles, permissions, audit log.
18. Reports — exportable to Excel and PDF.

Every list is a server-side Yajra DataTable with search, filters, export, and permission-aware actions.

## 10. Flutter app architecture

Feature-first clean architecture:

```
lib/
  app/            # app widget, router, theme, flavors, bootstrap
  core/
    network/      # Dio client, interceptors (auth, locale, logging, retry), ApiResponse<T>, Failure types
    error/        storage/        services/ (location, push, realtime, maps)
    localization/ utils/          widgets/ (shared design-system components)
  features/
    auth/  onboarding/  profile/  wallet/  notifications/  support/  chat/
    rider/   (home, vehicle_selection, trip_request, offers, trip_tracking, trip_summary, history)
    driver/  (onboarding_documents, home, incoming_requests, bidding, active_trip, earnings, vehicles)
  each feature: data/ (datasources, models, repositories impl)
                domain/ (entities, repository contracts, use cases)
                presentation/ (controllers/providers, screens, widgets)
```

- State management: Riverpod (with code generation). Navigation: go_router with a role-based redirect — after login the router loads `RiderShell` or `DriverShell` according to account type; routes of the other role are unreachable.
- Networking: Dio + a typed API layer; models with freezed + json_serializable; a single `ApiResponse<T>` matching the backend envelope; all errors mapped to typed `Failure`s and shown through one consistent UI mechanism; automatic logout on 401; request cancellation; pagination helper.
- Secure token storage with flutter_secure_storage; light local cache for settings and vehicle types.
- Realtime through a `RealtimeService` abstraction (Reverb / Pusher protocol); push with firebase_messaging + local notifications; deep links from notifications to the right screen.
- Maps with google_maps_flutter: place search and autocomplete, pin-on-map picking, route polyline, animated driver marker. Driver background location with a foreground service and the correct Android/iOS permission flows and store-compliant disclosures.
- Localization ar/en with full RTL, theme from the design tokens, light and dark mode, responsive to small screens.
- Flavors: dev, staging, prod with separate base URLs and Firebase configs.
- UX requirements: skeleton loaders, empty states, error states with retry, offline banner, pull to refresh, optimistic updates where safe, large tap targets, a booking flow of at most four steps, drivers can accept or bid with one hand in two taps.
- Force-update and maintenance screens driven by the settings endpoint.

Screens — Rider: splash, onboarding, role selection, login, register, OTP, forgot password, home map, destination search, vehicle selection with suggested fares, trip details (cargo fields, notes, price offer, payment method, promo), live offers, driver details, trip tracking (ETA, call, chat, share trip, SOS), trip summary and rating, history, wallet, saved addresses, notifications, profile, support, settings.
Screens — Driver: register, vehicle and document upload, under-review status, home (online toggle, map, today's earnings), incoming request sheet (accept / bid / skip, countdown), navigate to pickup, active trip (arrived, start, complete, proof of delivery), earnings and reports, wallet and withdrawals, my trips, ratings, vehicles and documents renewal, notifications, profile, support.

## 11. Design system (UI/UX)

Final visual design comes from Figma, and illustrations/photos from Canva. Until I hand them over:

- Define design tokens once (colors, typography, spacing, radius, elevation) and use them in both Tailwind config and Flutter `ThemeData` so the Figma values can be dropped in later without touching screens.
- Default direction: clean, modern, high contrast, one strong primary color, Arabic-first typography (Cairo or IBM Plex Sans Arabic), 8pt spacing grid, rounded cards, bottom sheets over the map.
- Keep all images in a clearly named assets structure with placeholder files and a `DESIGN_ASSETS.md` list (name, size, where used) so I can replace them with Canva exports.
- Produce a `DESIGN_BRIEF.md` describing every screen (purpose, components, states) that a designer can follow in Figma.
- If a Figma link or exported frames are provided, match them exactly: spacing, sizes, colors, states.

## 12. Quality bar

- Backend: Pest tests — unit tests for FareCalculator, wallet ledger, trip state machine, bidding rules; feature tests for every API endpoint (happy path, validation, authorization). Larastan and Pint clean.
- Flutter: unit tests for repositories and controllers, widget tests for the main flows, `flutter analyze` clean with strict lints.
- Security: authorization on every endpoint (a user can only touch their own trips, bids, wallet), rate limiting, input validation, mass-assignment protection, signed URLs for private files, masked phone numbers in API responses where the other party should not see them.
- Performance: no N+1 (eager loading, tested), indexed queries, cached settings and vehicle types, queued notifications.
- Docker Compose for local development (app, nginx, mysql, redis, reverb, horizon, mailpit), `.env.example`, a `README.md` with setup steps, and a `docs/` folder (architecture, ERD, API, deployment on Ubuntu VPS with Nginx + Supervisor, release checklist).
- Conventional commits, small focused commits per feature.

## 13. Delivery phases

**Phase 0 — Plan (no feature code).** Read this file, ask me your open questions, then deliver: final ERD, API endpoint list, folder structures, package list with versions, and the phase plan. Wait for approval.

**Phase 1 — Backend foundation.** Project setup, Docker, architecture skeleton, response envelope and exception handling, auth (register, login by email or phone, OTP, password reset, tokens), users, admin auth, roles and permissions, settings, localization, vehicle types and pricing tables with seeders, tests.

**Phase 2 — Dashboard foundation.** Layout, components, admin auth, dashboard shell, riders, drivers, vehicle types, pricing module, verification queue, settings.

**Phase 3 — Driver onboarding and trip core.** Documents, vehicles, zones, routing provider, FareCalculator, trip creation, matching, bidding, state machine, realtime, push notifications, ratings, cancellations, tests.

**Phase 4 — Money.** Wallet ledger, cash commission settlement, promo codes, withdrawals, finance pages and reports in the dashboard.

**Phase 5 — Flutter foundation.** Project structure, theme, localization, networking layer, auth flows, role selection, role-based shells, shared components.

**Phase 6 — Rider app.** All rider screens end to end against the real API.

**Phase 7 — Driver app.** All driver screens end to end, background location, incoming request flow.

**Phase 8 — Dashboard completion.** Live map, trips, support, notifications, content, reports, audit log.

**Phase 9 — Extras and hardening.** Chat, SOS and trip sharing, scheduled trips, online payment gateway, surge, performance and security pass, full docs, deployment guide, store release checklist.

## 14. How to work

- Before coding each phase, post a short plan and the list of files you will create or change.
- Ask when a product decision is unclear; state your assumption and continue only if it is low-risk and easy to change.
- Run tests, static analysis, and formatters before declaring a phase finished, and report the actual results.
- Keep `docs/PROGRESS.md` updated: what is finished, what is next, decisions taken, known gaps. Read it at the start of every session.
- Never silently skip a requirement. If something cannot be done, say so and propose an alternative.
- Code and comments in English; all user-facing text in translation files (ar and en) from day one.

## 15. Open questions to confirm in Phase 0 (defaults in brackets)

1. Can one phone number hold both a rider and a driver account? [No — one account, one type]
2. Is the price pure per-km or per-km suggestion plus bidding? [Suggestion + bidding within admin range]
3. Is OTP phone verification mandatory at launch? [Built, switched on by a setting]
4. Launch scope: one city or multiple zones? [Multi-zone ready, one zone seeded]
5. Online payment gateway provider? [Paymob, Phase 9]
6. SMS provider? [Interface + log driver until chosen]
7. Can a driver own several vehicles of different types? [Yes, one active at a time]
8. Commission collection for cash trips? [Debit from driver wallet with negative limit]