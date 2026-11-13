# Delivery Plus — Team Execution Plan

> Owner: Project Manager / Business Analyst (orchestrator)
> Source of truth for requirements: [`docs.md`](docs.md) (the Master Prompt). This file is about **how** the team builds it: who does what, in what order, with which quality gates.
> Live status lives in `docs/PROGRESS.md`. Every agent reads it first and updates it last.

---

## 1. Environment baseline (verified 2026-11-14)

| Item | State | Consequence |
|---|---|---|
| `backend/` | Fresh Laravel **13.x** skeleton, PHP **8.3.36**, Pest 4, Pint, Laravel Boost (MCP + skills), `backend/CLAUDE.md` guidelines | Build in place. Follow Boost rules (`search-docs`, `make:` commands, `pint --dirty`). |
| `application/` | Fresh Flutter **3.47.9** / Dart 3.13.7 skeleton, package name `application` | Build in place. Android SDK present; iOS cannot be built on this Linux host (code only). |
| MySQL | 8.0.46 running locally, **root without password is refused** | **Blocker:** need a DB user + two databases (`delivery_plus`, `delivery_plus_testing`). | Tech lead is creating it in `backend/.env`.
| SQLite | `pdo_sqlite` **not installed** | Tests run against MySQL test DB. |
| Redis | 7.0.15, `PONG` | Cache, queues, GEO index usable locally. |
| Docker | **Not installed** | `docker-compose.yml` is written and documented but cannot be smoke-tested here. Local dev uses native services. |
| Node | 20.20 / npm 10.8 | Vite + Tailwind build for dashboard. |
| Design | Figma MCP + Canva MCP connected | Design track (section 5) produces real Figma files and Canva assets. |

## 2. Product decisions (open questions from docs.md §15)

**Confirmed by the tech lead (2026-11-14):** all defaults accepted. Run mode: all phases run straight through; stop only for blockers.

1. One phone = one account = one type (rider **or** driver).
2. Price = suggested fare (per-km) + bidding within admin `min_bid_percent`/`max_bid_percent`.
3. OTP built, toggled by `auth.require_phone_verification` (off in dev).
4. Multi-zone ready; one zone seeded (Greater Cairo).
5. Online payments: Paymob, Phase 9.
6. SMS: `SmsProvider` interface + `log` driver until a provider is chosen.
7. Driver may own several vehicles of different types; one active at a time.
8. Cash commission debited from driver wallet; negative limit blocks going online.

Additional working assumptions (low-risk, easy to change):
- Money stored as **integer piastres** (`unsignedBigInteger`) with a `Money` value object.
- Geo: `decimal(10,7)` lat/lng columns + MySQL `POINT SRID 4326` with spatial index on `zones.area` and trip pickup; Redis GEO for live drivers.
- Primary brand color placeholder `#0E9F6E` (green) until Figma tokens are delivered.

## 3. The team (agent roster)

Each role runs as a subagent with a fixed brief. The orchestrator (PM/BA) never writes feature code; it plans, briefs, reviews, and gates.

| Role | Agent | Owns | Must never |
|---|---|---|---|
| **PM / Business Analyst** | orchestrator (main session) | Scope, acceptance criteria, sequencing, PROGRESS.md, decisions log, final report | Merge a phase that fails a gate |
| **Database Designer / Optimizer** | `general-purpose` | ERD (`docs/ERD.md`, Mermaid), migrations, indexes, spatial columns, factories/seeders, query/N+1 review | Store money as float; skip FKs/indexes |
| **Senior Backend Developer** | `general-purpose` | Laravel API, services, repositories, DTOs, enums, events, jobs, Reverb, Sanctum, Scribe docs, Pest tests | Put queries/logic in controllers; add unlisted packages |
| **Senior Frontend Developer** | `general-purpose` | Admin dashboard: Blade + Tailwind + vanilla JS (Vite), `x-admin.*` components, Yajra DataTables, Select2, ApexCharts, Flatpickr, SweetAlert2, Google Maps | Use Livewire/Filament/Inertia/Vue/React; write custom jQuery |
| **Senior Flutter/Dart Developer** | `general-purpose` | Mobile app: Riverpod (codegen), go_router role shells, Dio, freezed, maps, realtime, push, l10n/RTL, flavors | Mix rider/driver routes; hard-code strings |
| **UI/UX Designer** | `general-purpose` + Figma/Canva MCP | Design tokens, `DESIGN_BRIEF.md`, `DESIGN_ASSETS.md`, Figma file (dashboard + mobile), Canva illustrations | Ship values that aren't in the token set |
| **Problem Solver** | `general-purpose` | Called when a gate fails or an agent is blocked: root-cause, minimal fix, writes the lesson into PROGRESS.md | Paper over failures (skipping/deleting tests) |
| **Senior QA Tester** | `general-purpose` | Writes/extends Pest feature tests & Flutter widget tests from acceptance criteria, exploratory API runs, bug list | Mark green without running the suite |
| **Senior QC (Quality Control)** | `general-purpose` (read-mostly) | Audits each phase against docs.md hard constraints & §12 quality bar: architecture rules, security, N+1, i18n, no TODO/placeholder, Pint/Larastan/analyze clean | Approve with open P1/P2 findings |

## 4. Working protocol (every phase)

```
PM brief ─▶ DB Designer (schema slice) ─▶ Builder(s) ─▶ QA (tests) ─▶ QC (audit) ─┬─▶ PM sign-off ─▶ next phase
                                                                                   └─▶ Problem Solver ─▶ QA/QC re-run
```

1. **Brief.** PM writes the phase brief: goal, files in scope, acceptance criteria (Given/When/Then), out of scope.
2. **Build.** Agents run **one at a time** (no two writers on the same files). Each agent: reads `docs.md`, this file, `docs/PROGRESS.md`, `backend/CLAUDE.md` (backend), relevant skills; verifies package versions (`composer show`, pub.dev, Boost `search-docs`) before use.
3. **Test.** QA runs the full suite and adds missing cases. Report real numbers.
4. **Audit.** QC checks the gate checklist (section 7) and files findings P1–P3.
5. **Fix loop.** P1/P2 findings go to the Problem Solver (or the owning builder) until green. Max 3 loops, then escalate to the tech lead.
6. **Close.** PM updates `docs/PROGRESS.md` (done / next / decisions / known gaps), commits with conventional commits, writes the phase summary.

Handoff contract — each agent ends with: files created/changed, commands run + actual output summary, open issues, next agent's prerequisites.

## 5. Design track (Figma + Canva)

Runs in parallel to Phases 1–2 because it touches no code, and must land before Phase 2 UI and Phase 5 Flutter theming.

| Step | Output | Tool |
|---|---|---|
| D1 Tokens | `design/tokens.json` (colors light/dark, typography Cairo/IBM Plex Sans Arabic, 8pt spacing, radius, elevation) → generated into `tailwind` theme + Flutter `AppTokens` | local files |
| D2 Brief | `DESIGN_BRIEF.md`: every rider, driver, and admin screen: purpose, components, states (loading/empty/error/success) | local file |
| D3 Figma | New Figma file "Delivery Plus — Design System & Screens": variables from tokens, core components (buttons, inputs, cards, bottom sheets, badges, vehicle tiles, bid card), RTL Arabic key screens (role select, login, home map, vehicle selection, offers, tracking, driver incoming request, driver active trip, admin dashboard, pricing page) | Figma MCP |
| D4 Canva | Onboarding illustrations (3), vehicle-type icons/images (10), app icon concepts, splash, empty-state illustrations, admin login hero | Canva MCP |
| D5 Assets | Exports placed under `application/assets/images/...` and `backend/resources/images/...`; `DESIGN_ASSETS.md` (name, size, where used) | local files |

Rule: code consumes **tokens only**; when Figma values change, only `tokens.json` changes.

## 6. Phase plan and agent assignment

| Phase | Scope (from docs.md §13) | Agents in order | Exit criteria |
|---|---|---|---|
| **0 Plan** | ERD, API endpoint list, folder structures, package list with verified versions, phase plan | DB Designer → Backend (API list) → Flutter (structure + packages) → QC review | `docs/ERD.md`, `docs/API.md`, `docs/ARCHITECTURE.md`, `docs/PACKAGES.md`, `docs/PROGRESS.md` exist and are consistent |
| **D Design** | Section 5 | UI/UX Designer | Figma file link + Canva assets + token files |
| **1 Backend foundation** | Setup, Docker files, skeleton, envelope + exception handler, auth (register, login email/phone, OTP, reset, tokens, FCM token), admin auth, roles/permissions, settings, localization, vehicle types + pricing tables + seeders | DB Designer → Backend → QA → QC | Pest green, Pint clean, Larastan level ≥ 6 clean, Scribe docs generated |
| **2 Dashboard foundation** | Layout, components, admin auth, shell, riders, drivers, vehicle types, pricing (with history), verification queue, settings | Frontend (+Backend for controllers/DataTables) → QA → QC | Pages render RTL/LTR, light/dark; feature tests for every admin route; `npm run build` OK |
| **3 Trip core** | Documents, vehicles, zones, routing provider, FareCalculator, trip creation, matching (Redis GEO), bidding, state machine, Reverb realtime, push, ratings, cancellations | DB Designer → Backend → Problem Solver (concurrency review) → QA → QC | Unit tests: FareCalculator, state machine, bidding; race-condition tests with row locks |
| **4 Money** | Wallet ledger, cash commission, promo codes, withdrawals, finance pages + reports | DB Designer → Backend → Frontend → QA → QC | Ledger invariants tested; no balance write without ledger row |
| **5 Flutter foundation** | Structure, theme from tokens, l10n/RTL, Dio layer, auth flows, role selection, role shells, shared components | Flutter → QA → QC | `flutter analyze` clean, unit + widget tests green |
| **6 Rider app** | All rider screens against real API | Flutter → QA → QC | Booking flow ≤ 4 steps, widget tests for main flows |
| **7 Driver app** | All driver screens, background location, incoming requests | Flutter → QA → QC | Accept/bid in 2 taps; permission flows documented |
| **8 Dashboard completion** | Live map, trips, support, notifications, content, reports, audit log | Backend → Frontend → QA → QC | All lists Yajra server-side with export |
| **9 Extras & hardening** | Chat, SOS, sharing, scheduled trips, Paymob, surge, perf & security pass, docs, deployment, release checklist | Backend → Flutter → Frontend → Problem Solver (perf/security) → QA → QC | Full suite green, docs complete, release checklist |

## 7. Quality gates (QC checklist per phase)

**Architecture**: Controller → FormRequest → DTO → Service → Repository interface → Model; Resources for every response; enums for every status; `declare(strict_types=1)`.
**Constraints**: no Livewire/Filament/Inertia/Vue/React; jQuery only as DataTables/Select2 dependency; no unlisted package without tech-lead approval.
**Security**: policy on every endpoint, rate limits on auth, mass-assignment protection, private disk + signed URLs, masked phones.
**Data**: FKs + indexes, money as integer piastres, soft deletes where history matters, factories/seeders.
**i18n**: every user-facing string in `lang/ar` + `lang/en` / Flutter ARB; RTL verified.
**Hygiene**: no `TODO`, no placeholder code, no secrets; Pint clean, Larastan clean, `flutter analyze` clean; tests green with actual counts reported.
**Docs**: PROGRESS.md, Scribe/Postman, ERD updated.

Severity: **P1** violates a hard constraint or breaks a flow → blocks. **P2** missing requirement/test → blocks. **P3** polish → logged in known gaps.

## 8. Risk register

| Risk | Mitigation |
|---|---|
| No DB credentials → backend untestable | Blocker raised to tech lead before Phase 1 |
| No Docker locally | Compose file written + documented; native services used for verification |
| iOS cannot build on Linux | Code + config only; flagged in release checklist |
| Third-party keys (Google Maps/Routes, FCM, Paymob, SMS) absent | Interfaces + local drivers (haversine routing, log SMS, log push); keys via `.env`/settings |
| Concurrency bugs in accept/bid | Row locks + dedicated tests + Problem Solver review in Phase 3 |
| Scope size (9 phases) | One phase at a time, gates enforced, PROGRESS.md as memory between sessions |

## 9. Run order (orchestrator script)

1. Phase 0 agents (sequential) → QC → PM sign-off.
2. Design track (Figma/Canva).
3. Phases 1 → 9, each following section 4.
4. Final QA regression + QC audit across the whole system → final report.
