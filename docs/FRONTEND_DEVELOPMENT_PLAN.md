# Lazyfit frontend — Flutter reference & development plan

This document is a **long-lived reference** for building the Lazyfit Flutter app: how it connects to the backend, how to organise code, what to build in what order, and where to learn more. Primary focus today is **Android**; **iOS** and **web** are planned targets with notes below.

**Source of truth for the backend product and API:**

- Handoff summary: **`Lazyfit`** repo, [`docs/PROJECT_CONTINUITY.md`](../../Lazyfit/docs/PROJECT_CONTINUITY.md). Relative links assume **`Lazyfit`** and **`Lazyfit_Front`** are **siblings** under one parent folder (adjust if your checkout layout differs). In the frontend devcontainer, the backend bind-mount is **`/workspaces/Lazyfit`**.
- HTTP contract: [`docs/openapi.yaml`](../../Lazyfit/docs/openapi.yaml).

---

## 1. What you are building (aligned with PROJECT_CONTINUITY)

Lazyfit is a **workout recommendation + logging** experience backed by HTTP APIs:

- **Exercise catalog** — moves plus muscle taxonomy and UI/progression metadata.
- **Session recommendation** — `GET /recommendations/session`: if the user has a **premade program** selected, the response comes from program templates plus per-user overrides; otherwise the **dynamic recommender** is used.
- **Session logging** — `POST /sessions` persists a completed workout and advances tracking (including premade-program progression behaviour when applicable).
- **Programs** — list premade programs, select via `POST /user/program-selection`, adjust starting prescription via `POST /user/program-overrides`.
- **Dev auth today** — `GET /login` sets a **`lazyfit-mock-session`** cookie; protected routes expect that cookie. Production auth is not the same surface yet — design the client so auth can evolve (centralised HTTP layer).

Important idea from continuity: **prescription vs actual** is stored at log time (recommended sets/reps vs what the user did) so progression rules stay explainable later.

---

## 2. Flutter principles (quick, practical)

Flutter UIs are a **tree of widgets**. Your job is mostly: **composition** (small widgets), **state** (where data lives), and **async** (loading data without freezing the UI).

### Widget tree and composition

- Almost everything visible is a `Widget`.
- Prefer **many small widgets** over one enormous `build` method — easier to read, test, and reuse.
- **Composition over inheritance**: build custom UI by nesting `Column`, `ListView`, `Text`, themed buttons, etc.

### Stateless vs stateful vs “state above”

- **`StatelessWidget`**: immutable; no internal state that changes over time — ideal when all data is passed in as constructor arguments.
- **`StatefulWidget` + `State`**: for **local** UI state (expanded/collapsed, current tab index, form field controllers tied to this screen only).
- **Lift state up**: if two sibling widgets need the same changing data (e.g. selected program ID), hold that state in a **common ancestor** or in **app-wide state** (see below).

### Async and the UI isolate

- Network and disk I/O return **`Future`**s (or **`Stream`**s). Never block the UI thread waiting on HTTP.
- Typical screen pattern: **loading → error → data**. Use `FutureBuilder`, `StreamBuilder`, or your state-management solution to drive rebuilds.

### State management (pick one philosophy for app-wide concerns)

For **tiny** widgets, `setState` is fine. For **session, user, catalog cache, navigation guards**, pick **one** app-wide approach and stick to it:

| Approach | Typical use |
|----------|--------------|
| **Provider** (`provider` package) | Simple: expose `ChangeNotifier` / `ValueNotifier` above `MaterialApp`. |
| **Riverpod** (`flutter_riverpod`) | Very explicit dependencies and testability; steeper curve, scales well. |

Recommendation for a beginner willing to learn one dependency: **start with Provider** + a small number of providers; migrate to Riverpod later if complexity grows.

### Navigation

- **Imperative**: `Navigator.push` / `pop` — quick for demos, harder to deep-link later.
- **Declarative routing**: **`go_router`** maps paths to screens and URL state — **recommended** early if you plan web and bookmarks.

### Theming

- Use **`ThemeData`** (Material 3: `ThemeData(useMaterial3: true)`) so colours and typography stay consistent.
- Put shared constants in **`lib/core/theme/`** (spacing, radius) rather than scattering magic numbers.

### Testing (layers)

- **Unit tests**: pure Dart (e.g. parsing, mappers).
- **Widget tests**: pump a screen, assert widgets and taps.
- **Integration tests**: full app driver (later). Mock HTTP at the **repository** boundary for stable tests.

---

## 3. Recommended folder structure (`lib/`)

Use a **layered core** plus **feature folders** — scales well without overwhelming beginners.

```text
lib/
  main.dart                 # Entry: flavors / minimal bootstrap
  app.dart                  # MaterialApp, theme, router root
  core/
    config/                 # AppConfig (baseUrl, flags), build flavors / dart-define
    network/                # ApiClient, interceptors, cookie jar
    routing/                # go_router routes
    theme/                    # ThemeData extensions, spacing tokens
    errors/                   # AppException, message mapping for SnackBars
    utils/                    # Small pure helpers
  data/
    models/                   # DTOs aligned with OpenAPI (hand-written or generated)
    repositories/             # ExercisesRepository, UserRepository, SessionRepository…
  features/
    auth/                     # Dev login, “am I authenticated?”
    catalog/                  # Exercise list / detail (video URL, increments, bump caps)
    program/                  # Program list, selection, overrides UI
    session/                  # “Today” from recommendations — main workout UX
    history/                  # Optional later — when API exposes list/history cleanly
    settings/                 # Dev base URL, about, diagnostics
```

**Why this shape**

- **`core/`**: one place for HTTP, routing, and design tokens — features stay focused on UX.
- **`data/`**: keeps JSON shapes and network calls **out of** widget files — widgets subscribe to repositories or view models instead.
- **`features/`**: each area owns its screens and local widgets — avoids one giant `widgets/` dumping ground.

Optional later:

- **`test/`** mirrors `lib/` paths.
- **`packages/api_client`** if you extract a pure Dart SDK for Lazyfit.

**OpenAPI code generation** (later productivity boost): [`openapi_generator`](https://pub.dev/packages/openapi_generator), [`swagger_dart_code_generator`](https://pub.dev/packages/swagger_dart_code_generator). Not required for v1 — hand-written DTOs are fine until churn stabilises.

---

## 4. High-level development phases (Android first)

Use this as a checklist you can reorder slightly but should not skip for a solid foundation.

### Phase 1 — Project bootstrap

- Run `flutter create .` (or create in a subfolder if you prefer) with **Kotlin** on Android as default.
- Run on **Android emulator or device**; fix any SDK/cmake warnings early.
- Add a **single configuration** for API base URL (e.g. `--dart-define=API_BASE_URL=...` or `flutter_launcher`/flavors). Avoid hard-coded URLs in repositories.

### Phase 2 — HTTP + dev cookie auth

- Implement **`GET /login`** first (dev only), then calls that send the **`lazyfit-mock-session`** cookie.
- Typical stack: **[`dio`](https://pub.dev/packages/dio)** + **[`cookie_jar`](https://pub.dev/packages/cookie_jar)** (persistent jar on mobile).
- **`package:http`** is possible but cookie handling is more manual.

**Flutter web caveat:** browsers enforce **CORS** and cookie rules strictly. Dev cookie auth against `localhost` may require **backend Coordination** (headers, SameSite, etc.). Plan for tokens or explicit web auth later if cookies become painful — not a Flutter limitation alone.

### Phase 3 — Read-only flows (prove end-to-end)

- `GET /exercises` — catalog list + detail (show `instruction_video_url`, increments, bump caps).
- `GET /programs` — list premade programs.
- `GET /user` — show profile + `program_config`.

Focus on **loading / error / empty** states — these patterns repeat everywhere.

### Phase 4 — Program selection & overrides

- `POST /user/program-selection` — select or clear program + starting session index.
- `POST /user/program-overrides` — starting weight / optional sets reps for moves in **current program session**.
- Reflect state from `GET /user` after each mutation.

### Phase 5 — Today’s workout (recommendations)

- `GET /recommendations/session` — primary training screen.
- **Starting weights:** PROJECT_CONTINUITY notes the current gap — you may infer “needs setup” from **`weight_kg == 0`** until a dedicated API exists.

### Phase 6 — Logging

- `POST /sessions` — send performed moves/sets aligned with **`docs/openapi.yaml`** (preserve prescription semantics the API expects).

### Phase 7 — Polish & quality

- Retries/backoff for transient failures, form validation (numeric weight, reps), **TalkBack** / semantics labels, offline messaging (honest “network required” if no offline stack yet).

### Phase 8 — iOS

- Xcode, signing, CocoaPods/Pods as Flutter requires; verify **Simulator** reaches your API (`localhost` differs from simulator — often use LAN IP).
- App Transport Security (**ATS**): HTTPS in production; HTTP only for explicit dev plist exceptions if needed.

### Phase 9 — Web

- `flutter build web`, hosting strategy.
- **Revisit auth**: cookies vs token; CORS vs simple API gateway.

---

## 5. API integration cheat sheet

| Endpoint | Purpose | Typical screen / layer |
|---------|---------|-------------------------|
| `GET /login` | Dev mock login; sets cookie | Auth / onboarding (dev-only entry) |
| `GET /exercises` | Full exercise catalog | Catalog list + detail |
| `GET /programs` | Premade programs | Program picker |
| `POST /user/program-selection` | Select/clear program + session index | Program settings |
| `POST /user/program-overrides` | Per-move starting prescription overrides | Setup before first “real” set |
| `PATCH /user/program` | Legacy/dynamic config fields (`division`, `goal`, …) | Settings (if still used in your UX) |
| `GET /user` | Profile + program config | Home / settings summary |
| `GET /recommendations/session` | Today’s prescribed session | Main workout runner |
| `POST /sessions` | Log completed session | Completion / summary flow |

Detailed schemas: backend [`docs/openapi.yaml`](../../Lazyfit/docs/openapi.yaml).

---

## 6. Multi-platform notes

| Platform | Practical note |
|---------|----------------|
| **Android (current focus)** | Same Dart code path as others. Emulator **→ host machine**: use **`http://10.0.2.2:<port>`** instead of **`http://localhost:<port>`** when the API runs on your development host. Physical device → use host **LAN IP**. |
| **iOS** | Same shared `lib/`; native project under `ios/`. Simulator networking differs from Android; ATS applies to HTTP. |
| **Web** | Shared UI; **`web/`** is packaging. Cookies + CORS + hosting origin — validate early if web is priority. |

The **`android/`**, **`ios/`**, and **`web/`** folders are **shells** — most product code stays in **`lib/`**.

---

## 7. Local backend reminder (optional)

Backend README: Postgres via Docker, **`lazyfit_dev`** network, API on **`8080`** with **`LAZYFIT_DEV=1`** for mock login. Your Flutter app only needs the **reachable base URL** from where the **app binary** runs (emulator vs host vs device).

---

## 8. Learning resources & tooling

Official and high-quality free material:

| Resource | Why use it |
|----------|-------------|
| [Flutter documentation — Get started](https://docs.flutter.dev/learn) | Install, tooling, mental model |
| [Widget catalog](https://docs.flutter.dev/ui/widgets) | Discover layouts and inputs systematically |
| [Dart language documentation](https://dart.dev/learn) — language tour | Null safety, `async`/`await`, classes |
| [Flutter codelabs](https://docs.flutter.dev/codelabs) | Hands-on pathways (Firebase optional; networking patterns still help) |
| [Material Design 3 — Flutter](https://m3.material.io/develop/flutter) | Visual system aligned with `ThemeData(useMaterial3: true)` |
| [Effective Dart](https://dart.dev/effective-dart) | Style & API design conventions |
| [Flutter testing overview](https://docs.flutter.dev/testing) | Unit / widget / integration layering |
| [pub.dev package site](https://pub.dev/) | Prefer packages with maintenance, changelog, Dart 3 compat; check popularity only as a tie-breaker |

Popular packages referenced in this plan (always check latest docs on pub.dev): [`dio`](https://pub.dev/packages/dio), [`cookie_jar`](https://pub.dev/packages/cookie_jar), [`go_router`](https://pub.dev/packages/go_router), [`provider`](https://pub.dev/packages/provider), [`flutter_riverpod`](https://pub.dev/packages/flutter_riverpod).

Community learning (informal):

- Flutter’s **[YouTube](https://www.youtube.com/flutterdev)** for release notes and feature overviews — use alongside official docs for API accuracy.

---

## 9. What this document does not cover

- Designing a full UI/UX specification — iterate with real devices and readability in sunlight/exercise contexts.
- Production auth and account systems — blocked on backend contractual decisions.
- OpenAPI codegen — optional later once schemas stabilise.

When in doubt about behaviour, prefer **`PROJECT_CONTINUITY.md`** for intent and **`openapi.yaml`** for request/response shapes.
