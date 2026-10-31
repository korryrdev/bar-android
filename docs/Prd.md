# BAR Android: Product Requirements Document (Rider and Driver Apps)

| | |
|---|---|
| **Document** | Android PRD for both BAR apps (shared platform plus per-app requirements) |
| **Repo** | `bar-android` (one Gradle multi-module project, two applications) |
| **Apps** | Rider: `dev.korryr.bar.rider` · Driver: `dev.korryr.bar.driver` |
| **Companion docs** | `PRD.md` (backend), `PLAN.md` (backend lanes), `ADMIN_PRD.md` |
| **Version** | 0.1 (draft) |
| **Date** | 2026-10-30 |
| **Status** | Draft for review by both founders |

**One document or two?** One. About 80% of the work (architecture, design system, security,
networking, build, testing) is shared, and two separate documents would drift apart. Sections
5 to 7 hold the requirements that differ per app.

**How to read this.** Requirement IDs start with `AND-`. Priorities: **P0** launch, **P1** should
be there at launch, **P2** later. Library versions are *pinned in the version catalog at project
start*; where this document names a version line, verify it on the official release page first,
because it moves quickly.

**Interpretations used** (correct me if wrong): "pwds" means people with disabilities, so that
section covers accessibility; "version center" means the Gradle version catalog; "trace" means
performance tracing and observability.

---

## 1. Overview

### 1.1 Purpose
Two native Android apps on one shared platform:
- **BAR Rider:** request a ride, track it live, pay, rate, stay safe.
- **BAR Driver:** apply and get verified, go online, receive offers, run trips, track earnings and the commission wallet.

The apps are thin clients: the server decides prices, eligibility, ride state, and payment
status. The apps must stay correct and usable on cheap phones and weak networks.

### 1.2 Goals
| ID | Goal |
|---|---|
| ADG1 | Fast, reliable core flows (request, accept, track, complete) on low-end Android phones |
| ADG2 | Security and privacy by default (tokens, location, documents, payments) |
| ADG3 | One design system, expressive and cheerful, usable by everyone including people with disabilities |
| ADG4 | A modular codebase two people can work in without stepping on each other |
| ADG5 | Releases that are small, observable, and reversible |
| ADG6 | Stay resilient to bad networks, process death, and aggressive battery management |

### 1.3 Non-goals (MVP)
iOS, Wear OS, Android Auto, tablet-optimized layouts, in-app turn-by-turn navigation (use an
external navigation app for drivers), offline ride requests, in-app card payments, widgets.

### 1.4 Targets (to be measured, then enforced in CI where possible)
| Area | Target |
|---|---|
| Cold start (mid-range phone) | under 2 s to first interactive screen; under 3.5 s on a 2 GB device |
| Jank (frames over 16 ms or 33 ms during core flows) | under 5% |
| Download size | Rider under 20 MB, Driver under 25 MB (Play download size) |
| Crash-free sessions | 99.5% or better, well under Play's bad-behavior thresholds |
| Memory | No leaks found by LeakCanary in the core flows; peak under 250 MB on a 3 GB device |
| Driver battery (online, screen off) | Measured per release; budget set after first field data |
| Time to show ride state after reconnect | under 2 s |

---

## 2. Apps and responsibilities

| Capability | Rider | Driver |
|---|---|---|
| Login with phone OTP, profile, settings | Yes | Yes |
| Request ride, fare quote, ride tracking | Yes | No |
| Driver application, document capture, vehicle | No | Yes |
| Online or offline, receive and answer offers | No | Yes |
| Active trip (arrive, PIN, start, complete, cash confirm) | View only | Drives it |
| Foreground location service and background location | No (foreground only) | Yes (only while online) |
| Wallet and M-PESA top-up | No (cash-first MVP) | Yes |
| SOS, trip share, emergency contacts | Yes | SOS yes, contacts optional |
| Ratings, support, notifications inbox | Yes | Yes |
| Promos | Yes | No |
| Earnings | No | Yes |

Both apps share the same sign-in, networking, design system, session handling, observability,
safety, and support code.

---

## 3. Platform and tooling baseline

### 3.1 Targets
| Item | Decision |
|---|---|
| `targetSdk` / `compileSdk` | **36** (Android 16). Google Play requires new apps and updates to target API 36 since 31 Aug 2026 |
| `minSdk` | **26** recommended (Android 8.0): consistent notification channels, background limits, adaptive icons. Decide using Play Console device data from your existing apps (AND-D1) |
| Language | Kotlin only |
| UI | Jetpack Compose only; no XML layouts except the manifest, themes bootstrap, and splash |
| JDK | 21 |
| Form factor | Phones, portrait first; the UI must still survive rotation and resizing without losing state |

Android 16 behaviour to design for from day one (verify against the current behavior-change list):
- **Predictive back is enforced** for apps targeting 36; `onBackPressed()` and legacy key events are not delivered. Use Navigation 3 and `BackHandler`/`PredictiveBackHandler` correctly.
- **Edge-to-edge is mandatory**; handle system bar and keyboard insets on every screen and over the map.
- **Orientation and resizability locks are ignored on large screens**; the UI must not break in other sizes.

### 3.2 Build tooling
| Concern | Decision |
|---|---|
| Android Gradle Plugin | 9.x line. **Built-in Kotlin**: do not apply `org.jetbrains.kotlin.android`; remove it from module files and the catalog if the wizard adds it |
| Annotation processing | **KSP only** (Hilt, Room); no KAPT. Re-check KSP, Hilt, and AGP compatibility on every AGP bump |
| Dependency injection | Hilt |
| Compose | Compose BOM from the version catalog; Compose compiler is the Kotlin compiler plugin |
| Navigation | **Navigation 3** (`androidx.navigation3`), stable 1.1.x line. Built-in deep link matching is still in the 1.2 alpha line, so MVP deep links are handled by our own small parser |
| Material | Material 3. See section 8.1 for the Expressive decision |
| Persistence | Room (structured data), DataStore (preferences) |
| Network | OkHttp, Retrofit, kotlinx.serialization |
| Images | Coil |
| Logging | Timber (debug only) plus the crash reporter |
| Static analysis | Android Lint, detekt, ktlint (via Spotless) |

### 3.3 The "version center": the Gradle version catalog
- **Location:** `gradle/libs.versions.toml` at the project root, inside the `gradle` folder. It is not under a `kotlin` folder.
- Gradle loads it automatically as `libs`. Sections: `[versions]`, `[libraries]`, `[bundles]`, `[plugins]`.
- **Rules:** every dependency and plugin version lives only here; modules reference `libs.xxx`; no hard-coded versions in module files; use bundles for groups that always go together (for example Compose UI, networking, testing).
- Enable Gradle dependency verification and the type-safe project accessors (`projects.core.network`).
- Updates go through reviewed PRs (Renovate or Dependabot), one concern per PR.
- An annotated skeleton is in Appendix B.

### 3.4 Convention plugins (`build-logic`)
An included build named `build-logic` provides plugins so module files stay tiny:
`bar.android.application`, `bar.android.library`, `bar.android.feature` (library plus Compose plus Hilt plus lifecycle plus navigation), `bar.android.compose`, `bar.android.hilt`, `bar.jvm.library` (pure Kotlin), and `bar.android.test`. They own SDK versions, Java and Kotlin settings, lint and detekt configuration, test options, and R8 defaults, so changing a setting happens in one place.

---

## 4. Architecture

### 4.1 Feature-based MVVM with unidirectional data flow
Each feature owns its own screens, state, and data. Inside a feature module:

```
feature/xyz/
  ui/       Screen composables, ViewModel, UiState, UiAction, UiEffect
  domain/   Use cases (only where they add value), feature models
  data/     Repository implementation, remote and local data sources, mappers
  di/       Hilt module
  navigation/  entries and registration (public surface of the feature)
```

| ID | Rule | Pri |
|---|---|---|
| AND-ARCH-1 | Composables are stateless where possible; state is hoisted to a screen-level composable that observes a ViewModel | P0 |
| AND-ARCH-2 | ViewModel exposes one `StateFlow<UiState>` (immutable data class or sealed interface), accepts `UiAction`s through a single `onAction()` function, and emits one-off `UiEffect`s through a `Channel` consumed once | P0 |
| AND-ARCH-3 | State is collected with `collectAsStateWithLifecycle()` | P0 |
| AND-ARCH-4 | No Android framework types, `Context`, or composables inside ViewModels, use cases, or repositories (except `SavedStateHandle`) | P0 |
| AND-ARCH-5 | Repositories are the single source of truth for their data and return `Flow` or a typed `Result`; DTOs are mapped to domain models at the data boundary | P0 |
| AND-ARCH-6 | Domain errors are a sealed `AppError` (network, auth, validation, server code, unknown) mapped from the backend `problem+json` code | P0 |
| AND-ARCH-7 | Coroutines everywhere; dispatchers are injected (never hard-coded); no `GlobalScope`; structured concurrency; cancellation is respected | P0 |
| AND-ARCH-8 | Everything is `internal` by default; each feature exposes only its navigation entry and (if needed) a tiny API | P0 |
| AND-ARCH-9 | Business rules that must be right (prices, eligibility, ride state) live on the server; the client only displays them | P0 |
| AND-ARCH-10 | Every screen has previews for loading, content, empty, and error states | P1 |
| AND-ARCH-11 | Process death is survivable: UI state that matters is in `SavedStateHandle` or recoverable from the server (`GET /v1/state`) | P0 |

### 4.2 Dependency injection (Hilt)
- Each app module has its own `@HiltAndroidApp` Application class and its own Hilt component graph.
- Shared bindings live in core modules; feature bindings live in the feature's `di` package.
- Scopes: `@Singleton` for network, session, database; ViewModel scope for feature state; avoid custom scopes until needed.
- Tests replace modules with Hilt test modules or plain constructor injection with fakes.

### 4.3 Navigation 3
| ID | Requirement | Pri |
|---|---|---|
| AND-NAV-1 | Navigation keys are `@Serializable` types implementing `NavKey`, defined in `:core:navigation` so features never depend on each other | P0 |
| AND-NAV-2 | Each app module owns the back stack (`rememberNavBackStack`) and one `NavDisplay`; features contribute entries through an `entryProvider` extension in their `navigation` package | P0 |
| AND-NAV-3 | Back stack survives configuration change and process death (serializable keys, saved-state and ViewModel-store entry decorators) | P0 |
| AND-NAV-4 | ViewModels are scoped to nav entries; arguments come from the key, injected with Hilt assisted injection | P0 |
| AND-NAV-5 | Predictive back works on every screen and in dialogs and sheets; transitions use the motion scheme (section 8) | P0 |
| AND-NAV-6 | Top-level destinations are separate back stacks (for example Home, History, Profile for the rider) with a defined back behaviour | P1 |
| AND-NAV-7 | Notification taps and app links are parsed by our own `DeepLinkParser` into keys and a correct synthetic back stack; unknown links fall back to Home | P0 |
| AND-NAV-8 | Auth gating: unauthenticated users see only the auth graph; session expiry clears the back stack to login | P0 |
| AND-NAV-9 | Blocking screens (maintenance, forced update) sit above everything and cannot be navigated away from | P0 |
| AND-NAV-10 | Keys and entries never carry sensitive data (tokens, phone numbers); pass IDs only | P0 |

If cross-feature API boundaries grow painful, split a feature into `api` (keys) and `impl`; do
not do it up front.

### 4.4 Module structure

**How many modules?** About **30 at maturity**, but you start with **9** and add a module only
when a task needs it. A module is justified when (a) both apps or several features need the code,
(b) it needs different permissions or lifecycle (driver-only), or (c) build time or ownership
benefits. Never create empty modules in advance.

```
bar-android/
  build-logic/                         (included build, convention plugins)
  gradle/libs.versions.toml            (version catalog)
  app-rider/            app-driver/
  core/
    common  model  designsystem  ui  network  realtime  session
    database  preferences  location  security  observability
    navigation  testing
  feature/                             (shared by both apps)
    auth  profile  safety  support  notifications  ratings
  feature-rider/
    home  trip  history  promos
  feature-driver/
    onboarding  home  trip  earnings  wallet
  benchmark/                           (Macrobenchmark and Baseline Profile generation, later)
```

**Stage 1 (Android lane AL0, 9 modules):** `app-rider`, `app-driver`, `core:common`,
`core:model`, `core:designsystem`, `core:network`, `core:session`, `core:navigation`,
`feature:auth`. Everything else arrives with the backend lane that needs it (section 16).

**Dependency rules**
| Rule | Why |
|---|---|
| App modules depend on the features and core they ship; nothing depends on an app module | Apps stay thin |
| Feature modules depend on core only | No feature-to-feature coupling |
| `feature-rider` and `feature-driver` modules never depend on each other, and shared `feature` modules never depend on either | Permissions and code stay separated per app |
| Core modules never depend on feature or app modules | Clear layering |
| `core:model` and `core:common` are pure Kotlin (JVM) where possible | Fast tests, no Android baggage |
| `core:designsystem` has no network, database, or business logic | Reusable UI primitives |
| Anything declaring background location, foreground service, or camera permissions lives only in driver modules | Manifest merging would otherwise leak permissions into the rider app |

These rules are enforced by an architecture test (for example Konsist or a custom Gradle check)
that fails the build on a violation. The full module catalogue is in Appendix A.

---

## 5. Functional requirements

Backend lane in brackets (see `PLAN.md`).

### 5.1 Shared (both apps)

**Authentication and session [L2, L3]**
| ID | Requirement | Pri |
|---|---|---|
| AND-AUTH-1 | Phone number entry with country code default `+254`, E.164 validation, clear error states | P0 |
| AND-AUTH-2 | OTP entry with auto-fill through the **SMS Retriever or SMS User Consent API** (no `READ_SMS` permission); resend with a visible countdown; attempt-limit and expiry messages | P0 |
| AND-AUTH-3 | Tokens stored securely (section 10); access token refreshed transparently with a single-flight refresh so parallel requests never refresh twice | P0 |
| AND-AUTH-4 | Session expiry or revocation returns the user to login with a clear message and clears local data | P0 |
| AND-AUTH-5 | Logout and "log out of all devices" | P0 |
| AND-AUTH-6 | Profile: name, avatar, language; change phone number through step-up OTP | P1 |
| AND-AUTH-7 | Account deletion request flow (also reachable from a web page for Play policy) | P1 |
| AND-AUTH-8 | Google and email sign-in and identity linking | P2 |

**Platform control [L3]**
| ID | Requirement | Pri |
|---|---|---|
| AND-CFG-1 | On start and on resume, fetch `GET /v1/config` (ETag cached); apply maintenance, minimum version, and flags | P0 |
| AND-CFG-2 | Maintenance screen when config says maintenance or any call returns `503 MAINTENANCE`; includes window end time and a retry action | P0 |
| AND-CFG-3 | Forced update screen on `426` or below minimum version, using Play In-App Updates (immediate) with a Play Store fallback link | P0 |
| AND-CFG-4 | Feature flags read from config; kill switches take effect without an app update | P1 |

**Safety, support, notifications, ratings [L5, L11, L12]**
| ID | Requirement | Pri |
|---|---|---|
| AND-SAF-1 | SOS button reachable in one or two taps from any active-trip screen; confirm-or-cancel step with a short countdown to avoid accidents; works with the screen locked where Android allows | P0 |
| AND-SAF-2 | Emergency contacts management (contact picker without `READ_CONTACTS`) | P0 |
| AND-SAF-3 | Share trip link through the system share sheet | P0 |
| AND-SAF-4 | "Are you OK?" prompt handling and response (when the backend raises it) | P1 |
| AND-SUP-1 | Report an issue from a ride; see case status and messages | P1 |
| AND-NTF-1 | Notification permission flow (Android 13+) explained before asking; channels per type | P0 |
| AND-NTF-2 | In-app inbox and announcements | P1 |
| AND-RAT-1 | Rate the other party after a trip (1 to 5, optional tags and comment) | P0 |

### 5.2 Rider app

| ID | Requirement | Pri |
|---|---|---|
| AND-R-1 | Home map showing the user's location, pickup and destination selection, and nearby pickup points | P0 |
| AND-R-2 | Place search for destinations through a cost-controlled provider (decision AND-D5) with recent and saved places stored privately | P0 |
| AND-R-3 | Fare quote screen per vehicle type showing the full breakdown and the quote expiry; the app never computes or sends a price | P0 |
| AND-R-4 | Request ride using the quote ID and an idempotency key; one active ride at a time | P0 |
| AND-R-5 | Searching state with cancel; clear no-driver-found outcome with retry | P0 |
| AND-R-6 | Driver assigned: name, photo, plate, vehicle, rating, ETA, route on the map, in-app contact | P0 |
| AND-R-7 | Show the 4-digit start PIN prominently and readable by screen readers | P0 |
| AND-R-8 | Live tracking of the driver; trip progress; arrival and trip-start events | P0 |
| AND-R-9 | Cancel with the fee rules shown before confirming | P0 |
| AND-R-10 | Completion: receipt (fare breakdown), payment method (cash), rating prompt | P0 |
| AND-R-11 | Ride history and receipt detail | P0 |
| AND-R-12 | Promo code entry and applied discount shown in the quote | P1 |
| AND-R-13 | Safety: SOS, share trip, route-deviation prompts, emergency contacts | P0 |
| AND-R-14 | Scheduled rides | P2 |
| AND-R-15 | In-app chat with the driver | P1 |

### 5.3 Driver app

| ID | Requirement | Pri |
|---|---|---|
| AND-D-1 | Driver application wizard: personal details, ID, licence, vehicle, insurance, selfie; save progress as a draft | P0 |
| AND-D-2 | Document capture with **in-app CameraX** (no gallery pick), quality and glare hints, upload through signed URLs with retry and progress, local copy deleted after upload | P0 |
| AND-D-3 | Application status screen: under review, approved, rejected with reason, resubmission request | P0 |
| AND-D-4 | Vehicle management: type (boda, tuk-tuk, car), plate, photos | P0 |
| AND-D-5 | Go online and offline; eligibility explained when blocked (documents expired, wallet below minimum, no location) | P0 |
| AND-D-6 | Foreground location service while online; clear persistent notification; stops immediately when offline | P0 |
| AND-D-7 | Incoming offer UI: pickup area, distance, fare, countdown, large Accept and Decline buttons; sound, vibration, and visual cues | P0 |
| AND-D-8 | After acceptance: exact pickup, rider first name, route preview, one-tap external navigation intent (Google Maps or other installed app) | P0 |
| AND-D-9 | Trip actions: arrived, enter rider PIN to start (attempt limit shown), complete, confirm cash collected | P0 |
| AND-D-10 | Cancel with reason; fee consequences explained | P0 |
| AND-D-11 | Earnings: today, week, per trip with commission and net | P0 |
| AND-D-12 | Wallet: balance, minimum, history, M-PESA top-up via STK Push with pending, success, and failure states | P0 |
| AND-D-13 | Document expiry warnings and renewal flow | P1 |
| AND-D-14 | Battery and background setup guide shown at first go-online, tailored to common phone brands in Kenya | P0 |
| AND-D-15 | Safety: SOS during trips, share trip, driver-side check-in prompts | P0 |

---

## 6. Networking, realtime, and data

### 6.1 Networking
| ID | Requirement | Pri |
|---|---|---|
| AND-NET-1 | Retrofit with OkHttp and kotlinx.serialization. DTOs and API interfaces are **generated from the backend OpenAPI spec**; no hand-written endpoint paths | P0 |
| AND-NET-2 | Interceptors: auth header, `X-App-Version`, `X-App-Platform`, `Accept-Language`, a per-request **correlation ID** (`X-Correlation-Id`) that also appears in client logs and crash reports, and `Idempotency-Key` for write calls that need it | P0 |
| AND-NET-3 | A refresh authenticator handles `401` (single flight, then retry once, then logout) | P0 |
| AND-NET-4 | Errors parsed from `application/problem+json` into `AppError` using the stable `code` (mapping in Appendix F) | P0 |
| AND-NET-5 | Timeouts tuned for weak networks (connect and read), gzip, HTTP/2, small payloads, and retries only for idempotent or keyed requests, with exponential backoff and jitter | P0 |
| AND-NET-6 | No cleartext traffic; certificate pinning per section 10 | P0 |
| AND-NET-7 | Config and static lists cached with ETag | P1 |
| AND-NET-8 | Offline and slow network banners; actions that need the network fail fast with a retry | P0 |

### 6.2 Realtime
| ID | Requirement | Pri |
|---|---|---|
| AND-RT-1 | One WebSocket connection managed in `:core:realtime`, opened only when needed (rider with an active ride, driver while online) | P0 |
| AND-RT-2 | Authenticated at connect; reconnect with backoff and jitter; heartbeat; connection state exposed as a `Flow` | P0 |
| AND-RT-3 | Messages carry sequence numbers; on a gap or reconnect the app calls `GET /v1/state` and rebuilds ride state from the server | P0 |
| AND-RT-4 | Push (FCM) is the fallback when the socket is not connected | P0 |
| AND-RT-5 | Event handling is idempotent: duplicate or out-of-order events never corrupt UI state | P0 |

### 6.3 Local data
| ID | Requirement | Pri |
|---|---|---|
| AND-DATA-1 | Room for ride history cache, drafts (driver application), and the driver's pending location batches; schema exported, migrations tested | P0 |
| AND-DATA-2 | DataStore for settings; **tokens and sensitive values are encrypted** (section 10) | P0 |
| AND-DATA-3 | Logout wipes all user data, caches, and cached images | P0 |
| AND-DATA-4 | Queued writes (for example location batches) are bounded in size and age, and safe to replay | P0 |
| AND-DATA-5 | Money is formatted from minor units with one shared formatter (for example "KES 300") | P0 |

---

## 7. Location, maps, push, and background work

### 7.1 Location
| ID | Requirement | Pri |
|---|---|---|
| AND-LOC-1 | `FusedLocationProviderClient` behind a `LocationSource` interface in `:core:location` (fakeable in tests) | P0 |
| AND-LOC-2 | **Rider:** foreground location only, requested when the map opens; approximate location is accepted with a prompt for precise location when needed | P0 |
| AND-LOC-3 | **Driver:** foreground service with `foregroundServiceType="location"`, running only while online; permission flow is foreground location first, then background, each with an explanation screen | P0 |
| AND-LOC-4 | Driver update cadence follows the backend contract (about 4 s on a trip, about 10 s idle); updates are batched and sent with retry; GPS accuracy mode adapts to state to save battery | P0 |
| AND-LOC-5 | Mock locations are detected (`Location.isMock` where available) and reported as a flag to the backend, not blocked client-side | P1 |
| AND-LOC-6 | Location is never written to logs, crash reports, or analytics | P0 |
| AND-LOC-7 | Permissions and location services states (denied, denied permanently, GPS off) have dedicated, friendly screens | P0 |

### 7.2 Maps
| ID | Requirement | Pri |
|---|---|---|
| AND-MAP-1 | Maps Compose with a restricted API key (package name and signing certificate); one key per app; keys not committed (read from `local.properties` or CI secrets) | P0 |
| AND-MAP-2 | Routes are drawn from the **encoded polyline returned by the backend**, so the app does not call a directions API directly | P0 |
| AND-MAP-3 | Markers and polylines are lightweight; map recomposition is minimized; map is released correctly (no leaks) | P0 |
| AND-MAP-4 | Map UI colors follow the brand palette; map style is simple and readable in sunlight | P1 |

### 7.3 Push notifications
| ID | Requirement | Pri |
|---|---|---|
| AND-PUSH-1 | FCM token registered through `POST /v1/devices` on login and on token refresh; removed on logout | P0 |
| AND-PUSH-2 | Data messages handled in a `FirebaseMessagingService` with minimal work; heavy work is handed to WorkManager or the foreground service | P0 |
| AND-PUSH-3 | Notification channels (Appendix E): `offers`, `trip_updates`, `safety`, `payments`, `announcements`, `system`; marketing is off until opt-in | P0 |
| AND-PUSH-4 | Offers arrive as high-priority messages and also through the WebSocket while the driver is online; duplicates are de-duplicated by offer ID | P0 |
| AND-PUSH-5 | Full-screen intent for offers is **not assumed**: Android restricts it, so the offer UI must work from a normal high-priority notification tap (decision AND-D6) | P0 |

### 7.4 Background work
- **Foreground service** only for driver location while online (and an active-trip notification).
- **WorkManager** for deferrable work: retrying uploads (documents), flushing queued writes, token registration.
- No exact alarms, no wake locks beyond what the foreground service needs.
- The driver app guides users through OEM battery settings (Tecno, Infinix, Samsung, Xiaomi) with in-app instructions; it does **not** request the battery-optimization-exemption permission directly unless Play policy is checked (AND-D6).

---

## 8. Design system

### 8.1 Material 3 Expressive decision
Official release tables (late August 2026) show `androidx.compose.material3` **stable at 1.4.0** and
the **1.5.0 alpha line** (alpha 27 on 26 Aug 2026). Third-party tracking notes that most Expressive
APIs (`MaterialExpressiveTheme`, `MotionScheme`, the emphasized type styles, `MaterialShapes`, the
new components) are available in the 1.5 alpha line, and that some remain `@Experimental`. Plan:

| Option | Meaning | Use when |
|---|---|---|
| **A (proposed)** | Use the 1.5 alpha line **inside `:core:designsystem` only**, pinned to an exact version, upgraded deliberately | Pilot and internal builds |
| **B (fallback)** | Stay on stable 1.4.0 and implement `BarMotion`, `BarShapes`, and `BarTypography` ourselves with the same names and semantics as the Expressive APIs | If an alpha blocker appears, or before public launch if no beta or RC exists |

Rules:
1. All screens use **our** wrappers (`BarTheme`, `BarMotion`, `BarShapes`, `BarButton`, ...), never raw Material calls scattered through features. Switching between options is then a one-module change.
2. No alpha UI library ships in the public-launch build unless the lead engineer records an explicit exception in `docs/decisions`.
3. Re-check the release page at the start of each Android lane.

### 8.2 Theme and colors (BAR brand: bright, cute, flat, no gradients)
**No gradients anywhere** (backgrounds, buttons, illustrations, scrims). Depth comes from solid
tonal layers, thin outlines, and spacing, not shadows or blends. Colors come only from theme tokens.

Starting values (generate the full role set with Material Theme Builder from these source colors,
then verify contrast in a test; adjust values, not the rules):

| Role | Light | Dark | Notes |
|---|---|---|---|
| `primary` | `#FFC933` yellow | `#FFC933` | Main actions and highlights |
| `onPrimary` | `#1E1B3A` ink | `#1E1B3A` | **Always dark text on yellow** |
| `primaryContainer` | `#FFE9A8` | `#4A3B00` | Soft highlight areas |
| `secondary` | `#FF6B6B` coral | `#FF8A8A` | Secondary accents |
| `onSecondary` | `#1E1B3A` | `#1E1B3A` | |
| `tertiary` | `#2EC4A0` mint | `#4DDBB8` | Success and positive states |
| `onTertiary` | `#1E1B3A` | `#0F2A24` | |
| `error` | `#E5383B` | `#FF5A5F` | Destructive actions and SOS only |
| `onError` | `#FFFFFF` | `#2B0A0B` | |
| `background` | `#FFFBF0` cream | `#14122A` | |
| `surface` | `#FFFBF0` | `#1E1B3A` ink | |
| `surfaceVariant` | `#F4EEDC` | `#2A2750` | Cards, inputs (solid) |
| `onSurface` | `#1E1B3A` | `#FFFBF0` | |
| `outline` | `#8A86A3` | `#8A86A3` | Thin borders |

| ID | Requirement | Pri |
|---|---|---|
| AND-DES-1 | Light and dark themes both ship; the user can choose system, light, or dark | P1 |
| AND-DES-2 | **Dynamic color (Material You) is off by default** so the brand stays consistent; an optional setting can come later | P0 |
| AND-DES-3 | Text contrast at least 4.5:1 and non-text UI at least 3:1, verified by an automated palette test | P0 |
| AND-DES-4 | Semantic tokens (`success`, `warning`, `danger`, `sos`) mapped to theme roles; red is reserved for danger and SOS | P0 |
| AND-DES-5 | Driver and rider are distinguishable at a glance: same palette, but the driver app uses an ink-colored top area and a "Driver" badge, plus a different launcher icon | P1 |
| AND-DES-6 | Status is never conveyed by color alone (icon and text always accompany it) | P0 |

### 8.3 Typography
- One **bundled** font family (candidates: Nunito or Plus Jakarta Sans, decision AND-D4); no downloadable fonts, to avoid network dependence.
- Use the Material 3 type scale including the emphasized (Expressive) styles for headings and key numbers (fare, ETA, PIN).
- All sizes in `sp`; layouts must hold at 200% font scale (see section 9).
- Swahili and English diacritics render correctly; numbers use tabular figures for fares and balances.

### 8.4 Shapes
- Friendly, rounded: small 12 dp, medium 20 dp, large 28 dp, extra large 40 dp; primary buttons are pill-shaped.
- Shape morphing (`MaterialShapes`) reserved for hero moments: request button to loading to confirmed, offer accept.
- Consistent corner radii between map overlays, sheets, and cards.

### 8.5 Motion (MotionScheme)
| ID | Requirement | Pri |
|---|---|---|
| AND-MOT-1 | `BarMotion` exposes spring-based **spatial** and **effects** specs (fast, default, slow) for each use, modelled on `MotionScheme` | P0 |
| AND-MOT-2 | **Expressive** motion (bouncier springs) only for hero moments: requesting a ride, driver found, offer accepted, trip completed | P1 |
| AND-MOT-3 | **Standard** motion (calm) for lists, navigation, sheets, and utility screens | P0 |
| AND-MOT-4 | Respect the system "remove animations" and animation-scale settings: motion becomes instant fades or none | P0 |
| AND-MOT-5 | Motion never delays a time-critical action (offer accept, SOS) and never runs heavy animations on low-end devices (frame budget checked with benchmarks) | P0 |
| AND-MOT-6 | Predictive back transitions follow the same specs | P0 |

### 8.6 Components and patterns
- Buttons (primary, secondary, tertiary, destructive, SOS), icon buttons, chips, status pills, list items, cards, text fields with OTP and PIN variants, dialogs, modal and persistent bottom sheets, snackbars, top bars, navigation bar.
- Ride-state **bottom sheet** over the map is the main pattern for both apps.
- Expressive loading indicator for waits; skeletons for lists; flat vector illustrations for empty states (solid colors only).
- Icons: Material Symbols (rounded) added as individual vectors; **do not** include the whole extended icons library (size).
- Layout: edge-to-edge with correct insets, 8 dp grid, minimum touch target 48 dp, one-handed reach for primary actions (bottom-aligned).

### 8.7 Components documentation
A `:core:designsystem` preview catalogue (or a debug-only gallery screen) shows every component in light, dark, 200% font, and RTL-agnostic layouts, doubling as a screenshot-test source.

---

## 9. Accessibility (people with disabilities)

Goal: everyone can request or drive a ride safely. Target **WCAG 2.2 AA-equivalent** for mobile.

| ID | Requirement | Pri |
|---|---|---|
| AND-A11Y-1 | **Screen reader (TalkBack):** every actionable and informative element has a meaningful `contentDescription` or semantics label; decorative items are hidden; roles, states (selected, disabled, progress) are exposed | P0 |
| AND-A11Y-2 | Logical focus and reading order; headings marked with heading semantics; dialogs and sheets move and restore focus correctly | P0 |
| AND-A11Y-3 | Important changes are announced (live regions): driver found, driver arriving, trip started, trip ended, errors | P0 |
| AND-A11Y-4 | The start PIN is shown as large text and is readable digit by digit by screen readers | P0 |
| AND-A11Y-5 | Touch targets at least 48 x 48 dp; spacing prevents accidental taps near destructive actions; SOS needs a deliberate action but stays reachable | P0 |
| AND-A11Y-6 | Text scales to at least 200% without clipping or lost actions; critical content never depends on fixed heights | P0 |
| AND-A11Y-7 | Color contrast per AND-DES-3; no color-only meaning | P0 |
| AND-A11Y-8 | **Hearing:** every sound cue (offers, arrival, SOS) has a visual and a haptic equivalent | P0 |
| AND-A11Y-9 | **Vision:** high-contrast legible layouts, no information in images only, map actions available as lists (for example "pickup at ..." text) | P0 |
| AND-A11Y-10 | **Motor:** all actions reachable without precise gestures or dragging; Switch Access and Voice Access work; no action depends on long press only | P0 |
| AND-A11Y-11 | **Cognitive:** short plain-language copy in English and Swahili, one primary action per screen, clear error recovery, no unexplained jargon | P0 |
| AND-A11Y-12 | Timed actions: the driver offer countdown is necessary, so it is announced, shown visually and with haptics, and the Accept button is large and stable; other flows have no unnecessary time limits | P0 |
| AND-A11Y-13 | Reduce-motion and animation-off settings are honored (AND-MOT-4) | P0 |
| AND-A11Y-14 | Optional rider "accessibility needs" note shared with the driver **only with explicit consent** (this is sensitive information); vehicle filtering for accessible vehicles | P2 |

**Verification:** Compose accessibility checks in UI tests, the Accessibility Scanner on each
release, a manual TalkBack walkthrough of the core flows per release, and screenshot tests at 200%
font scale. Findings are tracked as bugs with the same priority as functional defects.

---

## 10. Security

Assume the APK is public and the phone may be rooted, lost, or hostile. Anything inside the app
can be extracted, so secrets and decisions live on the server.

### 10.1 Threats
| Threat | Example | Main controls |
|---|---|---|
| Token theft | Malware or a lost phone reads stored tokens | Keystore-backed encryption, short access tokens, rotating refresh tokens, remote revocation |
| Traffic interception | Hostile Wi-Fi | TLS only, no cleartext, certificate pinning, network security config |
| Reverse engineering and tampering | Modified APK, hooked functions, forged requests | R8, Play Integrity attestation checked on the server, server-side validation of everything |
| Fake location | Spoofed GPS to game fares | Mock-location flag, server plausibility checks |
| Data exposure on device | Screenshots, backups, recents, logs | `FLAG_SECURE`, backups off, no sensitive logs, cleared data on logout |
| Malicious apps | Intent spoofing, overlay attacks | Non-exported components, immutable pending intents, validated deep links |
| Supply chain | Compromised dependency | Dependency verification, pinned versions, reviewed updates |
| Social engineering | Fake driver or rider | PIN, driver details shown, in-app contact only |

### 10.2 Requirements
| ID | Requirement | Pri |
|---|---|---|
| AND-SEC-1 | Tokens and sensitive values are encrypted with a **non-exportable Android Keystore key** (use Tink AEAD with a Keystore-backed master key, with DataStore). `EncryptedSharedPreferences` is not used (deprecated) | P0 |
| AND-SEC-2 | **No secrets in the APK**: no API keys beyond restricted public keys (Maps), no signing secrets, no backend credentials. Restricted keys are limited by package name and signing certificate | P0 |
| AND-SEC-3 | `usesCleartextTraffic="false"` and a network security config trusting system CAs only (debug-only exceptions in the debug source set) | P0 |
| AND-SEC-4 | **Certificate pinning** on our API domain using SPKI pins with a backup pin, and a tested rotation plan; no pinning on third-party SDK hosts (decision AND-D7) | P1 |
| AND-SEC-5 | **Play Integrity API** verdict requested on registration, OTP request, ride request, and driver go-online, verified on the server; failure never blocks silently, it produces a clear message or step-up | P1 |
| AND-SEC-6 | `android:allowBackup="false"` and data extraction rules that exclude everything sensitive | P0 |
| AND-SEC-7 | Every component has an explicit `android:exported`; only launcher, FCM, and deep-link entries are exported; `PendingIntent`s are immutable | P0 |
| AND-SEC-8 | Deep links and notification intents are validated: allow-listed hosts, strict parsing, no trusting extras; no custom URI schemes for sensitive actions | P0 |
| AND-SEC-9 | `FLAG_SECURE` on the PIN screen, document capture screens, and wallet; recents thumbnails hidden for sensitive screens | P0 |
| AND-SEC-10 | **Debug features cannot exist in release builds**: `debuggable=false`, no debug menus, no verbose logging, LeakCanary and StrictMode only in debug | P0 |
| AND-SEC-11 | No sensitive data in logs, crash reports, analytics, or clipboard; the OTP is never copied to the clipboard; custom log wrappers redact phone numbers and tokens | P0 |
| AND-SEC-12 | The server never trusts the client: prices, payment status, eligibility, and ride transitions are validated server-side | P0 |
| AND-SEC-13 | Root and emulator signals are advisory (through Play Integrity), not a hard client-side block; the driver app reports them | P1 |
| AND-SEC-14 | Driver document images are captured in-app, uploaded through short-lived signed URLs, never stored in shared storage or the gallery, and deleted locally after upload | P0 |
| AND-SEC-15 | WebViews are not used in the MVP; if added later they are locked down (no JavaScript bridge, no file access, allow-listed hosts) | P0 |
| AND-SEC-16 | Gradle **dependency verification** enabled; dependency updates reviewed; vulnerable dependencies fail CI | P0 |
| AND-SEC-17 | Auto-logout and local data clearing when the server revokes a session; refresh token reuse leads to logout | P0 |
| AND-SEC-18 | Optional biometric or device-credential lock for the app (rider and driver), P2 | P2 |
| AND-SEC-19 | Release builds are signed with per-app upload keys; signing material never in the repo; keystores backed up in two secure places | P0 |
| AND-SEC-20 | A threat note accompanies each feature ticket, and a security checklist is part of the PR template | P0 |

---

## 11. Privacy and permissions

| ID | Requirement | Pri |
|---|---|---|
| AND-PRV-1 | Request each permission **in context**, after a short explanation screen; never at app start in bulk | P0 |
| AND-PRV-2 | Minimal permissions per app (Appendix D); no `READ_SMS`, `READ_CONTACTS`, storage, or phone-state permissions | P0 |
| AND-PRV-3 | Rider location is used only while the app is in use; driver location only while online, with a visible persistent notification | P0 |
| AND-PRV-4 | In-app privacy settings: notification preferences, location help, data export and deletion request, links to policy and terms | P1 |
| AND-PRV-5 | Play Data safety form and the **background-location declaration (driver app only)** are prepared and kept in sync with the code and the backend | P0 |
| AND-PRV-6 | Analytics, if any, is minimal, with no personal data and no location; no advertising identifiers collected | P0 |
| AND-PRV-7 | Cached personal data (images, history) respects the retention rules and is removed on logout | P0 |

---

## 12. Performance, tracing, and observability

### 12.1 Performance requirements
| ID | Requirement | Pri |
|---|---|---|
| AND-PERF-1 | **Baseline Profiles and Startup Profiles** generated for both apps (module `:benchmark`) and included in release builds | P1 |
| AND-PERF-2 | **Macrobenchmark** tests for cold start and the main flows (request ride, accept offer) run on demand and nightly where practical; regressions beyond a threshold fail the check | P1 |
| AND-PERF-3 | Compose stability and recomposition reports reviewed; lists use stable keys and `LazyColumn` properly; no heavy work during composition | P0 |
| AND-PERF-4 | `reportFullyDrawn()` called when the first useful screen is ready | P1 |
| AND-PERF-5 | Image loading with Coil using sensible sizes and caches; no full-size images in lists | P0 |
| AND-PERF-6 | R8 full mode and resource shrinking in staging and release builds (section 13) | P0 |
| AND-PERF-7 | Test regularly on a low-end device (2 GB RAM, older Android) and with throttled network | P0 |
| AND-PERF-8 | Size budget (1.4) checked in CI on every release candidate | P1 |

### 12.2 Tracing
| ID | Requirement | Pri |
|---|---|---|
| AND-TRC-1 | Custom trace sections with `androidx.tracing` (`trace("name") { ... }`) around the critical paths: app start, login, fare quote, ride request, offer handling, location pipeline, WebSocket reconnect and resync | P1 |
| AND-TRC-2 | Perfetto and the Android Studio profiler are the standard tools for investigating slow flows; findings go into `docs/performance.md` | P1 |
| AND-TRC-3 | **End-to-end correlation:** the client's `X-Correlation-Id` is sent on every request and attached to crash reports, so one ID links a user report to a backend log line | P0 |
| AND-TRC-4 | Key product timings are measured on the device and sent as anonymous metrics: time to ride request, time to driver assigned, reconnect-to-state time, STK top-up result time | P1 |

### 12.3 Leak and quality tooling
| ID | Requirement | Pri |
|---|---|---|
| AND-QA-1 | **LeakCanary** as a `debugImplementation` dependency in both apps; leaks are triaged like bugs and fixed before a lane ends | P0 |
| AND-QA-2 | LeakCanary instrumentation checks in selected UI tests (no leaks after finishing main flows) | P1 |
| AND-QA-3 | **StrictMode** enabled in debug builds (disk and network on main thread, leaked closables and registrations, cleartext) with penalties logged, and fatal on selected violations in CI builds | P0 |
| AND-QA-4 | Lint, detekt, and ktlint must pass; baselines are not allowed to grow | P0 |

### 12.4 Crash reporting and monitoring
- **Crash reporting:** Firebase Crashlytics by default (already in the Firebase project for FCM); mapping files uploaded on every release build (decision AND-D8: Crashlytics or Sentry).
- Non-fatal errors reported with the correlation ID and `AppError` code, never with personal data.
- **Play Console Android vitals** reviewed after every release; staged rollout is halted on a spike in crashes or ANRs.
- Remote kill switches (feature flags) for risky features.

---

## 13. Build, R8, and release

### 13.1 Build types (no product flavors for rider and driver)
| Build type | App ID suffix | Debuggable | R8 | API environment | Tools |
|---|---|---|---|---|---|
| `debug` | `.debug` | Yes | Off | Local or staging | LeakCanary, StrictMode, Timber, debug gallery |
| `staging` | `.staging` | No | On, same rules as release | Staging | Release-like testing, instrumented smoke tests |
| `release` | none | No | On | Production | Signed, Crashlytics mapping upload |

The environment URL comes from `buildConfigField` per build type. Each app module owns its
`applicationId`, name, icon, and signing config.

### 13.2 R8 and keep rules
| ID | Requirement | Pri |
|---|---|---|
| AND-R8-1 | R8 **full mode** (default in current AGP) with code and resource shrinking enabled for `staging` and `release` | P0 |
| AND-R8-2 | Each library module ships its own `consumer-rules.pro` for anything reflective it contains; each app has one `proguard-rules.pro` for app-wide rules; default optimized rules (`proguard-android-optimize.txt`) are included | P0 |
| AND-R8-3 | **No broad keeps**: `-keep class dev.korryr.** { *; }` and similar are forbidden. Every rule has a comment with the reason, the library or class it protects, and a way to verify it | P0 |
| AND-R8-4 | Most libraries (Hilt, Room, Retrofit, OkHttp, kotlinx.serialization, Coil, Firebase, WorkManager) ship their own consumer rules; we add rules only for gaps found in testing | P0 |
| AND-R8-5 | Keep source file and line number attributes and rename the source file attribute so stack traces can be de-obfuscated with the mapping file | P0 |
| AND-R8-6 | The **`staging` build is tested** with instrumented smoke tests (login, quote, ride flow, serialization of every API model) so missing keep rules are found before production | P0 |
| AND-R8-7 | Mapping files are uploaded to the crash reporter and archived with each release tag | P0 |
| AND-R8-8 | Reflection use is avoided; where unavoidable (for example a third-party SDK), it is documented in `docs/r8.md` | P1 |
| AND-R8-9 | R8 configuration is reviewed whenever AGP, Kotlin, or a major library is upgraded | P0 |

A starter `proguard-rules.pro` is in Appendix C.

### 13.3 Signing, versioning, release
- Separate upload keystores per app; Play App Signing enabled; credentials from `~/.gradle/gradle.properties` locally and CI secrets in the pipeline; `keystore.properties` is git-ignored.
- Each app has its own `versionName` (semantic) and a monotonically increasing `versionCode` derived from the CI run or the semantic version; release tags are per app: `rider-v0.3.0`, `driver-v0.3.0`. A change in `core` builds and tests both apps.
- Release path: internal testing track, then closed testing, then production with **staged rollout** (for example 5%, 20%, 50%, 100%) watching Android vitals.
- Releases ship the Android App Bundle (AAB).
- Each release note lists the backend API version it needs; the backend keeps older app versions working (see backend PRD 4.6).

### 13.4 CI/CD (GitHub Actions)
| Stage | Checks |
|---|---|
| Every PR | Build both apps, unit tests, lint, detekt, ktlint, dependency verification and vulnerability scan, secret scan, architecture test, size report, path filters so a change to only one app builds that app (changes to `core` build both) |
| On merge to `main` | All of the above plus `staging` build, instrumented smoke tests on Gradle Managed Devices, upload to Play internal track |
| Nightly | Instrumented tests on API 26 and API 36, Macrobenchmark (where available), LeakCanary-enabled UI tests |
| On tag | Signed AAB, mapping upload, publish to the chosen track, release notes |

Gradle configuration cache, build cache, and parallel execution enabled; CI caches the Gradle home.

---

## 14. Testing and definition of done

| Layer | Scope | Tools |
|---|---|---|
| Unit | ViewModels, use cases, repositories, mappers, formatters, deep link parser, reducers | JUnit, MockK, Turbine, coroutines-test |
| Compose UI | Screens in all states, accessibility checks, keyboard and font scale | Compose test APIs with accessibility checks |
| Screenshot | Design system and key screens in light, dark, 200% font | Compose Preview Screenshot Testing or Roborazzi |
| Integration | Room migrations, network layer against a mock server (MockWebServer) using the backend OpenAPI examples, WebSocket reconnect and resync | MockWebServer, Hilt test |
| Instrumented E2E | Login, quote, ride request, offer accept, trip complete against staging or a mock backend | Gradle Managed Devices |
| Performance | Startup, flows, jank, size | Macrobenchmark |
| Security | Manifest checks (exported components, permissions, cleartext), no secrets in the APK, R8 mapping checks, MobSF-style static scan before release | Scripts and scanners |
| Manual | TalkBack walkthrough, low-end device, weak network, battery behaviour on Tecno, Infinix, Samsung, Xiaomi | Checklist per release |

**Coverage goals:** 80% or more on ViewModels, use cases, and repositories for auth, ride state,
wallet, and offers; UI is covered by scenario and screenshot tests rather than a percentage.

**Definition of done (per ticket)**
- [ ] Requirement IDs referenced; PR small and reviewed
- [ ] Unit and UI tests, including error, offline, and process-death cases
- [ ] Accessibility checked (labels, focus, contrast, 200% font)
- [ ] Previews added for states; strings in resources (English and Swahili)
- [ ] No new lint, detekt, or LeakCanary findings
- [ ] Permissions and manifest changes reviewed (nothing leaks into the other app)
- [ ] R8-sensitive changes smoke tested on `staging`
- [ ] Threat note and security checklist ticked
- [ ] Docs updated

---

## 15. Localization
- English and Swahili at launch; all strings in resources with plurals; lint fails on hard-coded text.
- Per-app language via the system per-app language setting plus an in-app picker (`LocaleManager`); Kimeru can be added later as resource files only.
- Dates, times, numbers, and currency formatted for Kenya (KES; times in East Africa Time on the device; server stays in UTC).
- Copy is short and plain; the same glossary is used in both languages (ride, driver, fare, wallet, top-up).
- Backend-provided messages are localized by the server through `Accept-Language`.

---

## 16. Delivery plan (Android lanes mapped to backend lanes)

Android work follows the backend plan: a screen is built when its API contract is published and
stable on staging. Mock servers cover the gaps.

| Lane | Name | Needs backend | Scope | Exit criteria |
|---|---|---|---|---|
| **AL0** | Foundation | L1 | Create project, `build-logic`, version catalog, Stage 1 modules, design system v0 (theme, motion, buttons, fields), CI, R8 and LeakCanary and StrictMode setup, architecture test | Both apps build, run, and pass CI; `staging` build minifies cleanly |
| **AL1** | Auth | L2 | Phone and OTP flow, session storage, refresh, profile basics, navigation shell, auth gating | Real phone signs in against staging; token refresh and logout verified |
| **AL2** | Platform control | L3 | Config fetch, maintenance and forced-update screens, flags | Maintenance and update behaviours verified end to end |
| **AL3** | Driver onboarding | L4 | Application wizard, CameraX capture, uploads, status screen, vehicle | A driver is approved through the admin and the app shows it |
| **AL4** | Notifications and pricing | L5, L6 | FCM, channels, inbox; rider map, place search, fare quote | Push received on a device; quotes shown for real Meru routes |
| **AL5** | Location and realtime | L7 | `core:location`, `core:realtime`, driver go-online and foreground service, battery setup guide | A driver streams location for an hour on a test phone without being killed |
| **AL6** | Wallet | L8 | Wallet screen, history | Balance and history match the ledger |
| **AL7** | Rides (both apps) | L9 | Request, searching, offer UI, accept, arrive, PIN, start, complete, cancel, receipt, history | Two phones complete a cash trip on staging |
| **AL8** | Top-up | L10 | STK top-up flow, pending and result states | Real small top-up works; gating by minimum balance works |
| **AL9** | Safety | L11 | SOS, trip share, contacts, check-in prompts, chat | SOS drill passes with real devices |
| **AL10** | Quality features | L12 | Ratings, promos, support | Rating, promo, and case flows complete |
| **AL11** | Hardening | L14 | Baseline Profiles, benchmarks, accessibility audit, security review, R8 review, size and battery tuning, Play listing, data safety and background-location declaration | Launch gate passed with the backend |

---

## 17. Risks and open decisions

### 17.1 Risks
| Risk | Impact | Mitigation |
|---|---|---|
| Driver app killed in the background on cheap phones | Drivers appear offline, lose trust | Foreground service, setup guide, testing on real Tecno, Infinix, Samsung, Xiaomi devices, heartbeat monitoring |
| Alpha UI library churn (Expressive on the 1.5 alpha line) | Breaking changes, bugs | Option A and B plan, wrappers in one module, exact pins |
| Too many modules too soon | Slow builds and confusion | Stage 1 has 9 modules; add on demand |
| Play policy on background location and foreground services | Rejection | Driver-only permissions, accurate declaration, in-app disclosure |
| Missing R8 keep rules | Release-only crashes | `staging` build smoke tests, mapping files, rules reviewed on upgrades |
| Weak networks cause duplicate actions | Wrong rides or payments | Idempotency keys, state resync, server as the source of truth |
| Leaked permissions in the rider app through manifest merging | Privacy and review problems | Driver-only modules, merged manifest check in CI |
| Accessibility treated as a late polish | Exclusion and rework | Requirements in every ticket, checks in CI |
| Maps and Places cost | Budget overrun | Backend polylines, caching, quotas, provider choice (AND-D5) |

### 17.2 Open decisions
| # | Decision | Needed by |
|---|---|---|
| AND-D1 | `minSdk`: 26 recommended (the wizard procedure used 24, raise it); decide from Play Console device stats of your existing apps | AL0 |
| AND-D2 | Expressive adoption: Option A (1.5 alpha in `core:designsystem` for the pilot) or Option B (stable 1.4.0 plus our own tokens) | AL0 |
| AND-D3 | Dynamic color: confirm off by default | AL0 |
| AND-D4 | Font family (Nunito, Plus Jakarta Sans, or other) | AL0 |
| AND-D5 | Place search provider (Google Places, backend geocoding with cache, or other) and its cost cap | AL4 |
| AND-D6 | Offer delivery: full-screen intent availability, and whether to request the battery-optimization exemption (check current Play policy) | AL5 |
| AND-D7 | Certificate pinning approach and rotation plan | AL1 |
| AND-D8 | Crash reporting: Crashlytics (default) or Sentry | AL0 |
| AND-D9 | Chat approach for P1 (in-app only; masked calls later) | AL9 |
| AND-D10 | Driver verification extras: liveness check vendor or manual only (matches backend D9) | AL3 |

---

## 18. Appendices

### Appendix A: Module catalogue
| Module | Type | Contents | Depends on |
|---|---|---|---|
| `:app-rider`, `:app-driver` | Application | Application class, `MainActivity`, `NavDisplay`, back stack, manifest, resources (name, icon), Hilt entry, wiring of features | Features and core they ship |
| `:core:common` | JVM library | Result and error types, dispatchers, extensions, money and time utilities | none |
| `:core:model` | JVM library | Domain models shared across features | `common` |
| `:core:designsystem` | Android library | `BarTheme`, tokens, `BarMotion`, `BarShapes`, typography, primitives, icons | `common` |
| `:core:ui` | Android library | Shared composables (ride sheet, status pill, map overlays, error and loading states) | `designsystem`, `model` |
| `:core:network` | Android library | Retrofit and OkHttp setup, generated API client, interceptors, error mapping | `common`, `model`, `session` |
| `:core:realtime` | Android library | WebSocket client, reconnect, sequence handling | `network`, `common` |
| `:core:session` | Android library | Token store, refresh coordination, current user, device ID | `common`, `security`, `preferences` |
| `:core:database` | Android library | Room database, DAOs, migrations | `common`, `model` |
| `:core:preferences` | Android library | DataStore wrappers | `common` |
| `:core:location` | Android library | `LocationSource`, mock detection, no background permission | `common` |
| `:core:security` | Android library | Keystore and Tink wrapper, Play Integrity client, screen protection helpers | `common` |
| `:core:observability` | Android library | Logging wrapper with redaction, crash reporter, trace helpers, metrics | `common` |
| `:core:navigation` | Android library | `NavKey` types, navigator helpers, deep link parser | `model` |
| `:core:testing` | Android library | Fakes, rules, test data, Hilt test helpers | `common`, `model` |
| `:feature:auth` | Feature | Phone, OTP, session gating | core |
| `:feature:profile` | Feature | Profile, settings, language, privacy | core |
| `:feature:safety` | Feature | SOS, contacts, trip share, check-ins | core |
| `:feature:support` | Feature | Issue reporting, cases | core |
| `:feature:notifications` | Feature | Inbox, announcements, preferences, FCM handling | core |
| `:feature:ratings` | Feature | Rating UI and logic | core |
| `:feature-rider:home` | Feature | Map, place search, fare quote, ride request | core |
| `:feature-rider:trip` | Feature | Searching, assigned driver, tracking, completion | core |
| `:feature-rider:history` | Feature | History and receipts | core |
| `:feature-rider:promos` | Feature | Promo entry and list | core |
| `:feature-driver:onboarding` | Feature | Application, capture, status, vehicles | core, camera |
| `:feature-driver:home` | Feature | Online toggle, eligibility, offers, foreground service | core |
| `:feature-driver:trip` | Feature | Navigate, arrive, PIN, start, complete, cash | core |
| `:feature-driver:earnings` | Feature | Earnings screens | core |
| `:feature-driver:wallet` | Feature | Wallet and top-up | core |
| `:benchmark` | Test | Macrobenchmark and Baseline Profile generation | apps |

### Appendix B: Version catalog skeleton (`gradle/libs.versions.toml`)
```toml
[versions]
# Pin the latest STABLE of each at project start; verify on the official release pages.
agp = "<9.x stable>"            # Android Gradle Plugin (built-in Kotlin; no kotlin-android plugin)
kotlin = "<2.x stable>"         # compatible with AGP's bundled Kotlin and KSP
ksp = "<matching KSP>"          # re-check on every AGP bump
composeBom = "<latest stable BOM>"
material3Expressive = "<1.5.0-alphaNN>"   # only referenced by :core:designsystem (Option A)
navigation3 = "<1.1.x stable>"
hilt = "<2.5x>"
room = "<stable>"
retrofit = "<stable>"
okhttp = "<stable>"
kotlinxSerialization = "<stable>"
kotlinxCoroutines = "<stable>"
coil = "<stable>"
leakcanary = "<stable>"
tink = "<stable>"
timber = "<stable>"

[libraries]
# one line per library, using version.ref; group related ones into [bundles]

[bundles]
compose-ui = [...]
networking = [...]
testing-unit = [...]

[plugins]
android-application = { id = "com.android.application", version.ref = "agp" }
android-library = { id = "com.android.library", version.ref = "agp" }
ksp = { id = "com.google.devtools.ksp", version.ref = "ksp" }
hilt = { id = "com.google.dagger.hilt.android", version.ref = "hilt" }
# no kotlin-android and no kapt entries
```
Also in `settings.gradle.kts`: `enableFeaturePreview("TYPESAFE_PROJECT_ACCESSORS")`, `includeBuild("build-logic")`, and `dependencyResolutionManagement { repositoriesMode = FAIL_ON_PROJECT_REPOS }`.

### Appendix C: `proguard-rules.pro` starter (per app)
```proguard
# Keep line numbers for readable crash traces; hide the real file name.
-keepattributes SourceFile,LineNumberTable,*Annotation*,Signature,InnerClasses,EnclosingMethod
-renamesourcefileattribute SourceFile

# Add rules ONLY when a staging smoke test proves they are needed.
# Format every rule as:
#   # WHY: <reason> | PROTECTS: <class or library> | VERIFY: <test that fails without it>
# Forbidden: blanket keeps of our own packages.
```
Library modules put reflective-use rules in `consumer-rules.pro`, referenced by `consumerProguardFiles`.

### Appendix D: Permissions per app
| Permission | Rider | Driver | Notes |
|---|---|---|---|
| `INTERNET`, `ACCESS_NETWORK_STATE` | Yes | Yes | |
| `ACCESS_FINE_LOCATION` / `ACCESS_COARSE_LOCATION` | Yes (foreground) | Yes | Request in context |
| `ACCESS_BACKGROUND_LOCATION` | **No** | Yes | Declared only in driver modules; Play declaration |
| `FOREGROUND_SERVICE`, `FOREGROUND_SERVICE_LOCATION` | No | Yes | Only while online |
| `POST_NOTIFICATIONS` | Yes | Yes | Android 13+ runtime request |
| `CAMERA` | No | Yes | Document capture only |
| `USE_FULL_SCREEN_INTENT` | No | Decision AND-D6 | Restricted on newer Android |
| `READ_SMS`, `READ_CONTACTS`, storage, phone state | No | No | Use SMS Retriever and the contact picker instead |

CI inspects the **merged manifest** of each app and fails if the rider app contains any driver-only permission.

### Appendix E: Notification channels
| Channel | App | Importance | Notes |
|---|---|---|---|
| `offers` | Driver | High, sound and vibration | Offer alerts with deduplication |
| `trip_updates` | Both | High | Driver found, arriving, started, ended |
| `safety` | Both | High | SOS and check-ins |
| `payments` | Driver | Default | Top-up results, low balance |
| `announcements` | Both | Low | Off until opted in |
| `system` | Both | Default | Document expiry, app and account notices |

### Appendix F: Backend error code to UX mapping
| Code | App behaviour |
|---|---|
| `UPGRADE_REQUIRED` | Forced update screen |
| `MAINTENANCE` | Maintenance screen with end time |
| `UNAUTHENTICATED` | Refresh once, else login |
| `RATE_LIMITED` | Friendly wait with countdown |
| `OTP_INVALID`, `OTP_EXPIRED`, `OTP_ATTEMPTS_EXCEEDED` | Field error, resend, or wait guidance |
| `QUOTE_EXPIRED` | Request a new quote automatically and show the changed price |
| `RIDE_ALREADY_ACTIVE` | Open the active ride |
| `NO_DRIVER_FOUND` | Retry or try later |
| `DRIVER_NOT_ELIGIBLE` | Explain the reason (documents, suspension) |
| `WALLET_BELOW_MINIMUM` | Show top-up call to action |
| `PIN_INVALID` | Inline error with attempts left |
| `PAYMENT_PENDING`, `PAYMENT_FAILED` | Pending, retry, or failure states |
| network failure | Offline banner with retry |
| `INTERNAL_ERROR` | Generic message with a reference (correlation ID) |

### Appendix G: Glossary
| Term | Meaning |
|---|---|
| Version catalog | `gradle/libs.versions.toml`, the single place for dependency and plugin versions |
| Convention plugin | A Gradle plugin in `build-logic` that applies shared module settings |
| Nav 3 / NavKey | Navigation 3 library and its serializable destination keys |
| UDF | Unidirectional data flow: state down, actions up |
| R8 | Android's code shrinker, optimizer, and obfuscator |
| Keep rule | An R8 rule that stops a class or member from being removed or renamed |
| Baseline Profile | Precompiled hot paths that speed up startup and runtime |
| MotionScheme | Material 3 Expressive's set of spring-based motion specs |
| FGS | Foreground service |
| SPKI pin | A certificate pin based on the public key |

### Appendix H: Revision history
| Version | Date | Change |
|---|---|---|
| 0.1 | 2026-10-30 | First full draft |