# BAR Android Delivery Plan (Lanes)

| | |
|---|---|
| **Document** | Delivery plan for both Android apps, organized in lanes |
| **Repo** | `bar-android` (rider: `dev.korryr.bar.rider`, driver: `dev.korryr.bar.driver`) |
| **Companion docs** | `ANDROID_PRD.md` (what to build), `PLAN.md` (backend lanes), `PRD.md`, `ADMIN_PRD.md` |
| **Version** | 0.1 |
| **Date** | 2026-10-30 |

One plan, three tracks. Every task is labeled **[S]** shared (core and shared features),
**[R]** rider app only, or **[D]** driver app only. Requirement IDs (`AND-...`) point to `ANDROID_PRD.md`.

---

## 0. How to use this plan

1. **Work one lane at a time, in order.** Lane numbers `AL0` to `AL11` follow the backend lanes
   (`L1` to `L14`); a screen is built when its API contract is published and stable on staging.
2. **Mock first, switch later.** If the backend lane is not ready, build against a fake
   repository or a mock server behind the same interface, then switch to the real API when the
   OpenAPI contract is published. Do not wait idle.
3. **One task = one branch = one PR.** Branch: `feat/al3-document-capture`. The PR lists the task
   ID and requirement IDs and ticks the PR checklist (security, accessibility, R8, manifest).
4. **Priority tags:** **(P0)** launch, **(P1)** should be at launch, **(P2)** later. Cut P1 first, never P0.
5. **Both apps must build and pass CI at all times,** even when only one app changed.
6. **Unfinished features** stay behind a feature flag or are simply not wired into navigation in
   release builds.
7. **Every lane ends the same way:**
    - [ ] All P0 tasks ticked (P1 ticked or consciously deferred in writing)
    - [ ] Unit, UI, and accessibility tests green; no new LeakCanary or lint findings
    - [ ] `staging` build (R8 on) smoke tested on a real device
    - [ ] Merged manifests checked (rider has no driver-only permissions)
    - [ ] Tag `rider-v0.<lane>.0` and/or `driver-v0.<lane>.0` for each app that changed
    - [ ] Status table updated; five-line retro in `docs/retro.md`
8. **Sizes** (one developer, part-time): **S** about 1 week, **M** about 2, **L** about 3, **XL** 4 to 5.

---

## 1. Lane map and status

| Lane | Name | Backend needs | New modules | Rider | Driver | Size | Status |
|---|---|---|---|---|---|---|---|
| AL0 | Foundation | L1 | 9 Stage 1 modules, `build-logic` | shell | shell | M | [ ] |
| AL1 | Authentication | L2 | `core:security`, `core:preferences`, `feature:profile` | yes | yes | L | [ ] |
| AL2 | Platform control | L3 | none | yes | yes | S | [ ] |
| AL3 | Driver onboarding | L4 | `core:database`, `feature-driver:onboarding` | tech debt | main work | L | [ ] |
| AL4 | Notifications and pricing | L5, L6 | `feature:notifications`, `core:location`, `core:observability`, `feature-rider:home` | main work | channels | L | [ ] |
| AL5 | Location and realtime | L7 | `core:realtime`, `feature-driver:home` | subscribe | main work | L | [ ] |
| AL6 | Wallet | L8 | `feature-driver:wallet` | receipts | main work | S | [ ] |
| AL7 | Rides (both apps) | L9 | `feature-rider:trip`, `feature-rider:history`, `feature-driver:trip`, `feature-driver:earnings` | main work | main work | XL | [ ] |
| AL8 | Top-up | L10 | none | polish | main work | M | [ ] |
| AL9 | Safety | L11 | `feature:safety`, (chat) | yes | yes | M | [ ] |
| AL10 | Ratings, promos, support | L12 | `feature:ratings`, `feature:support`, `feature-rider:promos` | yes | yes | L | [ ] |
| AL11 | Hardening and launch | L14 | `benchmark` | yes | yes | L | [ ] |
| AL12 | Post-launch backlog | | | | | n/a | [ ] |

**Splitting between two people:** from AL3 onward the rider and driver tracks are mostly
independent (AL3, AL5, AL6, AL8 are driver-heavy; AL4 is rider-heavy). Do [S] tasks first and
together, then split by [R] and [D].

**Solo order inside a lane:** [S] first, then the app whose backend contract is ready, then the other.

**Minimum launchable cut:** AL0 to AL9 (P0), the P0 parts of AL10 (ratings and a basic support
form), and AL11.

**Honest timeline:** roughly 30 weeks of Android work for one part-time developer. If you build
the backend and both apps alone, add the two together: realistically **well over a year** for
everything, and **9 to 12 months** for the minimum cut. With two people working in parallel,
expect about half of that. Re-estimate after AL1 using your real speed.

---

## 2. Conventions

| Topic | Rule |
|---|---|
| Module to folder | `:core:network` is `core/network/`; `:feature-rider:home` is `feature-rider/home/` |
| Package names | `dev.korryr.bar.core.network`, `dev.korryr.bar.feature.auth`, `dev.korryr.bar.rider`, `dev.korryr.bar.driver` |
| Branches | `feat/alN-...`, `fix/alN-...`, `chore/...`; short-lived; PRs into protected `main` |
| Commits | Conventional style, for example `feat(auth): add otp screen` |
| Tags | `rider-v0.3.0`, `driver-v0.3.0`, release candidates `-rc1` |
| Version codes | Derived from the CI run number (or semantic version formula), per app |
| PR checklist | Tests, accessibility, no secrets, manifest and permissions, R8-sensitive change smoke-tested on `staging`, threat note |
| Agent rules | Same as the backend: agents open PRs, humans review auth, security, and manifest changes; a shared conventions file (`AGENTS.md` or `CLAUDE.md`) in the repo |

---

## Lane AL0: Foundation

**Size:** M. **Backend needs:** L1 (health endpoint only). **PRD refs:** 3, 4, 8, 12.2, 13.
**Goal:** a clean multi-module base where both apps build, run, and are measured and checked from day one.
You are starting from a wizard-created project with `app-rider` and `app-driver`; this lane turns it into the real structure.

### Tasks
- [ ] **AL0-01 (S, P0)** Record decisions in `docs/decisions.md`: `minSdk` (AND-D1), Expressive option A or B (AND-D2), dynamic color off (AND-D3), font (AND-D4), crash reporter (AND-D8).
- [ ] **AL0-02 (S, P0)** Tidy the wizard project: delete the default `app` module if it still exists; confirm `applicationId` (`dev.korryr.bar.rider`, `dev.korryr.bar.driver`), namespaces, debug suffix; set `compileSdk` and `targetSdk` to 36 and `minSdk` per AL0-01; confirm there is **no `kotlin-android` plugin** (AGP 9 built-in Kotlin).
- [ ] **AL0-03 (S, P0)** Version catalog: move every version and plugin to `gradle/libs.versions.toml`, add bundles, remove hard-coded versions, enable type-safe project accessors and `FAIL_ON_PROJECT_REPOS`.
- [ ] **AL0-04 (S, P0)** `build-logic` included build with convention plugins: application, library, feature, compose, hilt, jvm library, test.
- [ ] **AL0-05 (S, P0)** Create Stage 1 modules as compiling skeletons: `core:common` (JVM), `core:model` (JVM), `core:designsystem`, `core:network`, `core:session`, `core:navigation`, `feature:auth`.
- [ ] **AL0-06 (S, P0)** Hilt in both apps: separate `@HiltAndroidApp` classes, KSP wiring, a trivial injected dependency proving the graph.
- [ ] **AL0-07 (S, P0)** Navigation 3 shell: serializable keys in `core:navigation`, one `NavDisplay` and back stack per app, placeholder home; verify predictive back on an API 36 emulator.
- [ ] **AL0-08 (S, P0)** Design system v0: `BarTheme` (light and dark tokens from the PRD, solid colors, no gradients), bundled font, shapes, `BarMotion` (standard and expressive specs, reduce-motion aware), buttons (primary, secondary, destructive, SOS), text field, status pill, preview catalogue, and a contrast test for the palette.
- [ ] **AL0-09 (S, P0)** Observability baseline (inside `core:common` for now): Timber in debug, redacting logger, correlation ID helper, `androidx.tracing` helper, crash reporter setup with mapping upload.
- [ ] **AL0-10 (S, P0)** Debug tooling: LeakCanary (`debugImplementation`), StrictMode in the debug Application, debug-only component gallery screen.
- [ ] **AL0-11 (S, P0)** Build types: `debug`, `staging`, `release` with suffixes, per-type API base URL, R8 and resource shrinking on `staging` and `release`, starter `proguard-rules.pro` and `consumer-rules.pro` wiring.
- [ ] **AL0-12 (S, P0)** Manifest security: `allowBackup=false`, data extraction rules, explicit `exported`, cleartext off, network security config; a CI check that fails if the rider's merged manifest has any driver-only permission.
- [ ] **AL0-13 (S, P0)** Quality gates: detekt, ktlint through Spotless, Android Lint config, Gradle dependency verification, Renovate or Dependabot.
- [ ] **AL0-14 (S, P0)** Architecture test (Konsist or a Gradle check) enforcing the module dependency rules from the PRD.
- [ ] **AL0-15 (S, P0)** CI on GitHub Actions: per-app path filters, caches, build, tests, lint, scans, size report.
- [ ] **AL0-16 (S, P0)** Signing: separate upload keystores per app, CI secrets, local `~/.gradle/gradle.properties` pattern, recovery notes (keystores stored in two safe places).
- [ ] **AL0-17 (S, P0)** Repo hygiene: README, conventions file, PR template with checklist, `CODEOWNERS`, branch protection.
- [ ] **AL0-18 (S, P1)** OpenAPI client generation: a Gradle task or script that generates models and API interfaces from the backend spec, plus a mock server option (MockWebServer fixtures or a WireMock container).

### Tests and security gate
Both apps install side by side with the debug suffix; the merged manifest check passes; the APK
contains no secrets; `staging` (minified) launches; architecture test catches a deliberate violation;
palette contrast test passes.

### Exit criteria
Both apps build, run, and pass CI; design catalogue renders in light, dark, and at 200% font;
the version catalog is the only place with versions; first tags `rider-v0.0.1` and `driver-v0.0.1`.

---

## Lane AL1: Authentication

**Size:** L. **Backend needs:** L2 (OpenAPI v0.2). **PRD refs:** 5.1, 6.1, 10.
**Goal:** a real person can sign in with their phone on staging, and the session survives refresh, restart, and process death.

### Tasks
- [ ] **AL1-01 (S, P0)** `core:security`: Keystore-backed Tink AEAD wrapper, key lifecycle, instrumented tests.
- [ ] **AL1-02 (S, P0)** `core:preferences`: DataStore wrappers and the encrypted token store.
- [ ] **AL1-03 (S, P0)** `core:session`: session manager (tokens, current user, device ID), single-flight refresh coordinator, session events as a `Flow`.
- [ ] **AL1-04 (S, P0)** `core:network` for real: Retrofit and OkHttp, generated auth API, interceptors (auth, app version and platform, language, correlation ID), refresh authenticator, `problem+json` to `AppError` mapping, timeouts and retry rules.
- [ ] **AL1-05 (S, P0)** `feature:auth`: phone entry (+254 default, E.164 validation), OTP screen with SMS Retriever or User Consent auto-fill, resend countdown, attempt and expiry errors, loading and error states, accessibility.
- [ ] **AL1-06 (S, P0)** Auth gating in navigation: auth graph versus main graph, session expiry returns to login, restore after process death.
- [ ] **AL1-07 (S, P0)** `feature:profile` basics: name, language picker, logout, log out everywhere.
- [ ] **AL1-08 (R, P0)** Rider shell: navigation bar (Home placeholder, History placeholder, Profile).
- [ ] **AL1-09 (D, P0)** Driver shell: ink-colored top area, "Driver" badge, own launcher icon, placeholder tabs.
- [ ] **AL1-10 (S, P0)** Localization: English and Swahili strings for auth and profile; per-app language handling.
- [ ] **AL1-11 (S, P1)** Certificate pinning with a backup pin and a documented rotation test (AND-D7).
- [ ] **AL1-12 (S, P1)** Play Integrity token request wired behind a flag (server verifies).
- [ ] **AL1-13 (S, P1)** Phone change with step-up OTP and account deletion request entry.
- [ ] **AL1-14 (S, P0)** Tests: ViewModel tests, **parallel 401 test (many requests, exactly one refresh)**, UI tests with accessibility checks, MockWebServer tests.

### Tests and security gate
Tokens are never in plaintext on disk (instrumented check of the app's files); logs contain no
OTP, token, or full phone number; backup disabled; cleartext blocked; OTP is not placed on the clipboard.

### Exit criteria
A real phone signs in on staging in both apps; refresh and logout verified; killing the app and
reopening lands on the right screen; TalkBack walkthrough of auth passes.

---

## Lane AL2: Platform control

**Size:** S. **Backend needs:** L3. **PRD refs:** 5.1 (AND-CFG).
**Goal:** maintenance, forced updates, and kill switches work before there is anything to break.

### Tasks
- [ ] **AL2-01 (S, P0)** `ConfigRepository`: fetch `GET /v1/config` on start and resume with ETag caching.
- [ ] **AL2-02 (S, P0)** Global handling of `503 MAINTENANCE` and `426 UPGRADE_REQUIRED` in the network layer, surfaced as app-wide states.
- [ ] **AL2-03 (S, P0)** Maintenance screen with window end time and retry; sits above all navigation and cannot be dismissed.
- [ ] **AL2-04 (S, P0)** Forced update screen using Play In-App Updates (immediate), with a Play Store link fallback.
- [ ] **AL2-05 (S, P1)** Feature flag provider with typed accessors and safe defaults.
- [ ] **AL2-06 (S, P0)** Tests: each state triggered by a mock server; verify the active-trip exception (maintenance must not strand a user in a trip).

### Exit criteria
Using the admin panel on staging, schedule maintenance and raise the minimum version; both apps
react correctly and recover when it is cleared.

---

## Lane AL3: Driver onboarding

**Size:** L. **Backend needs:** L4. **PRD refs:** 5.3 (AND-D-1 to D-4, D-13), 10 (documents).
**Goal:** a driver can apply, capture documents safely, and see their approval status.
**Rider track:** no feature work; spend the time on design system components, accessibility fixes, and tech debt.

### Tasks
- [ ] **AL3-01 (S, P0)** `core:database` (Room): setup, schema export, first migration test.
- [ ] **AL3-02 (D, P0)** `feature-driver:onboarding` module with the application wizard (personal details, ID, licence, vehicle, insurance, selfie); drafts saved in Room.
- [ ] **AL3-03 (D, P0)** CameraX document capture: in-app only (no gallery), framing, glare and focus hints, retake, preview, `FLAG_SECURE`.
- [ ] **AL3-04 (D, P0)** Upload flow: request signed URL, upload with progress through WorkManager with retry and resume, delete the local copy after success.
- [ ] **AL3-05 (D, P0)** Status screen: under review, approved, rejected with reason, resubmission request with what to fix.
- [ ] **AL3-06 (D, P0)** Vehicle management (type, plate, photos).
- [ ] **AL3-07 (D, P0)** Camera permission flow with an explanation screen; driver-only manifest entries; merged manifest check stays green.
- [ ] **AL3-08 (D, P1)** Document expiry warnings and renewal flow.
- [ ] **AL3-09 (S, P0)** Tests: process death in the middle of an upload, lost connectivity, double submit, draft restore, accessibility of the wizard.
- [ ] **AL3-10 (R, P1)** Tech debt: extract reusable composables into `core:designsystem`, fix accessibility issues found so far.

### Tests and security gate
Document images never land in the gallery or shared storage; the app's cache is cleared after
upload; signed URLs are used once and not logged; camera screen blocks screenshots.

### Exit criteria
On staging, a driver applies and uploads documents, an admin approves in the admin panel, and the
driver app shows the approved state without reinstalling.
EOF
wc -l /mnt/user-data/outputs/ANDROID_PLAN.md
---

## Lane AL4: Notifications and pricing

**Size:** L. **Backend needs:** L5, L6. **PRD refs:** 5.1 (AND-NTF), 5.2 (AND-R-1 to R-3), 7.1, 7.2, 7.3.
**Goal:** push works on real devices, and a rider can pick a route and see a trustworthy fare.

### Tasks
- [ ] **AL4-01 (S, P0)** FCM integration: token registration and removal, `FirebaseMessagingService` doing minimal work, de-duplication of messages.
- [ ] **AL4-02 (S, P0)** Notification channels from the PRD (Appendix E), Android 13 permission flow with an explanation first.
- [ ] **AL4-03 (S, P1)** `feature:notifications`: inbox, announcements, preferences.
- [ ] **AL4-04 (S, P1)** Extract `core:observability` from `core:common` (logging, crash reporter, tracing, metrics).
- [ ] **AL4-05 (S, P0)** `core:location`: `LocationSource` interface, fused implementation, permission state model, friendly denied and GPS-off screens (no background permission here).
- [ ] **AL4-06 (R, P0)** `feature-rider:home` module: Maps Compose with a restricted API key, user location, map recomposition kept light.
- [ ] **AL4-07 (R, P0)** Pickup and destination selection with place search (provider per AND-D5), pickup points, recent and saved places stored privately (Room).
- [ ] **AL4-08 (R, P0)** Fare quote screen per vehicle type: breakdown, expiry countdown, route polyline from the backend, automatic re-quote on `QUOTE_EXPIRED`.
- [ ] **AL4-09 (R, P0)** Accessibility of the map flow: every map action also available as a list, readable summaries of pickup and destination.
- [ ] **AL4-10 (D, P0)** Prepare driver channels (`offers`, `payments`, `system`) and the notification tap routing for document and approval events.
- [ ] **AL4-11 (S, P0)** Tests: notification routing to the correct back stack, permission flows, quote screen states, map leak check with LeakCanary.

### Tests and security gate
No location in logs; Maps key restricted by package name and signing certificate; notification payloads
contain no sensitive data; deep link parser rejects malformed and unknown links.

### Exit criteria
A push for "driver approved" arrives on a real device and opens the right screen; quotes for ten
real Meru routes display with sensible prices.

---

## Lane AL5: Location and realtime

**Size:** L. **Backend needs:** L7. **PRD refs:** 5.3 (AND-D-5, D-6, D-14), 6.2, 7.1, 7.4.
**Goal:** a driver can stay online for hours on a cheap phone and the apps recover from any disconnect.
**Rider track:** subscribe to the realtime connection in a placeholder and verify state recovery.

### Tasks
- [ ] **AL5-01 (S, P0)** `core:realtime`: WebSocket client, authenticated connect, reconnect with backoff and jitter, heartbeat, sequence tracking, gap detection, `GET /v1/state` resync, connection state `Flow`.
- [ ] **AL5-02 (S, P0)** Event handling that is idempotent and order-tolerant; unit tests for duplicate, missing, and reordered events.
- [ ] **AL5-03 (D, P0)** `feature-driver:home` module: online and offline toggle with eligibility explanations (documents, wallet, location).
- [ ] **AL5-04 (D, P0)** Foreground location service (`foregroundServiceType="location"`) with a persistent notification; starts only when online and stops immediately when offline.
- [ ] **AL5-05 (D, P0)** Permission flow: foreground location, then background location, each with an explanation screen; denied and permanently-denied states.
- [ ] **AL5-06 (D, P0)** Location pipeline: update cadence from the backend contract, batching, bounded Room queue with replay, adaptive accuracy for battery.
- [ ] **AL5-07 (D, P1)** Mock-location flag reported to the backend (not blocked on the client).
- [ ] **AL5-08 (D, P0)** Battery and background setup guide for common brands (Tecno, Infinix, Samsung, Xiaomi) shown at the first go-online.
- [ ] **AL5-09 (D, P0)** Driver-only manifest entries (background location, foreground service types) and the merged manifest check for both apps.
- [ ] **AL5-10 (R, P1)** Rider placeholder subscription to realtime to test reconnect and state recovery.
- [ ] **AL5-11 (S, P0)** Tests: service lifecycle, process death, airplane mode toggling, flaky network simulation.
- [ ] **AL5-12 (D, P0)** **Field test:** a driver phone stays online for one hour (then four hours) on real devices from each brand; record battery and drop-outs in `docs/field-tests.md`.

### Tests and security gate
Location never logged; the service cannot be started by other apps; background location requested
only in the driver app; Play declaration draft prepared.

### Exit criteria
Simulated and real drivers appear in the backend's live index for an hour without dropping off;
reconnect and resync verified; battery figures recorded.

---

## Lane AL6: Wallet

**Size:** S. **Backend needs:** L8. **PRD refs:** 5.3 (AND-D-12), 6.3 (AND-DATA-5).
**Goal:** a driver can see their commission balance and history.

### Tasks
- [ ] **AL6-01 (D, P0)** `feature-driver:wallet`: balance, minimum balance explanation, paginated history, statements.
- [ ] **AL6-02 (D, P0)** Eligibility banner on the home screen when the balance is below the minimum.
- [ ] **AL6-03 (S, P0)** Shared money formatter (minor units to "KES 300") with tests for rounding and large values.
- [ ] **AL6-04 (R, P1)** Receipt composable in `core:ui` ready for ride completion.
- [ ] **AL6-05 (D, P0)** `FLAG_SECURE` on the wallet screen; accessibility labels for amounts and statuses.

### Exit criteria
Balance and history shown in the app match the backend ledger for test drivers.

---

## Lane AL7: Rides (both apps)

**Size:** XL. **Backend needs:** L9. **PRD refs:** 5.2 (AND-R-4 to R-11), 5.3 (AND-D-7 to D-11), 6.2.
**Goal:** two phones complete a cash trip from request to receipt, correctly under bad networks and restarts.
This is the heart of the product; expect it to be the longest lane.

### Tasks
- [ ] **AL7-01 (S, P0)** Shared ride state model and reducers driven by server state and realtime events; recovery through `GET /v1/state` on every start and reconnect.
- [ ] **AL7-02 (R, P0)** `feature-rider:trip`: request ride with an idempotency key, searching state with cancel, no-driver outcome with retry.
- [ ] **AL7-03 (R, P0)** Driver assigned screen: photo, plate, vehicle, rating, ETA, route; large readable **start PIN** with screen-reader support.
- [ ] **AL7-04 (R, P0)** Live tracking on the map, arrival and trip-start events, trip progress.
- [ ] **AL7-05 (R, P0)** Cancel flow with the fee shown before confirming.
- [ ] **AL7-06 (R, P0)** Completion screen with receipt and cash payment note.
- [ ] **AL7-07 (R, P0)** `feature-rider:history`: list and receipt detail.
- [ ] **AL7-08 (D, P0)** `feature-driver:trip` and offer UI: pickup area, distance, fare, countdown, large Accept and Decline, sound plus vibration plus visual cues, de-duplication by offer ID.
- [ ] **AL7-09 (D, P0)** After acceptance: exact pickup, rider first name, route preview, one-tap external navigation intent.
- [ ] **AL7-10 (D, P0)** Arrived, PIN entry with attempts shown, start, in-trip view, complete, **confirm cash collected**.
- [ ] **AL7-11 (D, P0)** Cancel with reasons and consequences explained.
- [ ] **AL7-12 (D, P0)** `feature-driver:earnings`: today, week, per trip with commission and net.
- [ ] **AL7-13 (S, P0)** Error handling for every ride error code (Appendix F of the PRD) with friendly recovery.
- [ ] **AL7-14 (S, P0)** Tests: state restoration after process death, duplicate and out-of-order events, offline during each step, two-phone end-to-end script; run the backend **simulator** for load on the driver side.
- [ ] **AL7-15 (S, P0)** Accessibility audit of the whole ride flow with TalkBack on both apps (including the offer countdown behaviour).

### Tests and security gate
Pickup and destination visibility matches server rules; PIN screen blocks screenshots; no duplicate
rides from rapid taps (idempotency verified); no leaks after finishing a trip (LeakCanary).

### Exit criteria
Two real phones complete a cash trip on staging; the commission shows in the driver's wallet;
cancel and no-driver paths work. Upload **internal alpha** builds `rider-v0.7.0` and `driver-v0.7.0` to the Play internal track.

---

## Lane AL8: Top-up

**Size:** M. **Backend needs:** L10. **PRD refs:** 5.3 (AND-D-12), 10.
**Goal:** a driver can top up the commission wallet by M-PESA without confusion.
**Rider track:** polish, accessibility fixes, and a bug bash of the ride flow.

### Tasks
- [ ] **AL8-01 (D, P0)** Top-up screen: amount with limits, phone confirmation, clear explanation of the STK prompt.
- [ ] **AL8-02 (D, P0)** Pending state while waiting for the phone prompt: timeout handling, status from push or realtime with polling fallback, never double-submitting.
- [ ] **AL8-03 (D, P0)** Success, failure, cancelled, and **late success** handling, with notifications.
- [ ] **AL8-04 (D, P0)** Banner and blocking logic for wallet below minimum, with a direct top-up action.
- [ ] **AL8-05 (D, P0)** Tests against the sandbox: success, wrong PIN, timeout, duplicate tap, app killed while pending.
- [ ] **AL8-06 (R, P1)** Bug bash and accessibility fixes from AL7.

### Exit criteria
A real small top-up works end to end; a driver below the minimum cannot go online and sees why.

---

## Lane AL9: Safety

**Size:** M. **Backend needs:** L11. **PRD refs:** 5.1 (AND-SAF).
**Goal:** when something goes wrong, help is one deliberate action away.

### Tasks
- [ ] **AL9-01 (S, P0)** `feature:safety`: SOS button with a short confirm countdown, haptics, reachable from every active-trip screen, usable with TalkBack and with the screen locked where Android allows.
- [ ] **AL9-02 (S, P0)** Emergency contacts management using the system contact picker (no `READ_CONTACTS`).
- [ ] **AL9-03 (S, P0)** Share trip through the system share sheet with the backend link.
- [ ] **AL9-04 (S, P1)** "Are you OK?" prompts and responses, with visual, audible, and haptic cues.
- [ ] **AL9-05 (S, P1)** In-app ride chat with persisted messages and retention (decision AND-D9).
- [ ] **AL9-06 (S, P0)** **Safety drill** with real devices: trigger SOS from both apps, confirm contacts and ops are alerted, verify timings.
- [ ] **AL9-07 (S, P0)** Tests: accidental trigger prevention, offline SOS behaviour (queue and retry visibly), accessibility.

### Exit criteria
A staged SOS from each app reaches contacts and the admin console within the target time; drill recorded.

---

## Lane AL10: Ratings, promos, and support

**Size:** L. **Backend needs:** L12. **PRD refs:** 5.1 (AND-RAT, AND-SUP), 5.2 (AND-R-12).
**Goal:** quality loops and a way to get help.

### Tasks
- [ ] **AL10-01 (S, P0)** `feature:ratings`: rating UI with tags and comment, within the time window, immutable after submit.
- [ ] **AL10-02 (S, P0)** `feature:support`: report an issue from a ride, categories, case status, messages (basic form is P0, conversation view is P1).
- [ ] **AL10-03 (R, P1)** `feature-rider:promos`: code entry, applied discount shown in the quote, promo list.
- [ ] **AL10-04 (D, P1)** Earnings polish: weekly summaries, trip detail, statements.
- [ ] **AL10-05 (S, P1)** Notification inbox integration for case updates.
- [ ] **AL10-06 (S, P0)** Tests and accessibility pass for all new screens.

### Exit criteria
Rate a trip, create a support case, and apply a promo in staging with the admin console reflecting each.

---

## Lane AL11: Hardening and launch readiness

**Size:** L. **Backend needs:** L14. **PRD refs:** 9, 10, 11, 12, 13, 14.
**Goal:** prove both apps are fast, accessible, secure, and shippable, then run the pilot.

### Tasks
- [ ] **AL11-01 (S, P1)** `benchmark` module: Baseline Profiles and Startup Profiles for both apps, Macrobenchmark for cold start and main flows.
- [ ] **AL11-02 (S, P0)** Performance pass: jank, memory, startup, size budget; test on a 2 GB phone and throttled networks.
- [ ] **AL11-03 (S, P0)** Accessibility audit: TalkBack walkthrough, Accessibility Scanner, 200% font screenshots, contrast test, switch and voice access checks; fix findings.
- [ ] **AL11-04 (S, P0)** Security review against PRD section 10: merged manifest audit, APK secret scan, static scan, pinning rotation test, Play Integrity wiring verified, dependency verification and vulnerability scan clean.
- [ ] **AL11-05 (S, P0)** R8 review: remove unnecessary keeps, smoke tests on `staging`, mapping files archived per tag.
- [ ] **AL11-06 (D, P0)** Battery tuning and the extended field test (full day) on each phone brand; background-location declaration video and in-app disclosure for Play.
- [ ] **AL11-07 (S, P0)** Play listings for both apps: screenshots, descriptions in English and Swahili, privacy policy link, data safety form, account deletion path.
- [ ] **AL11-08 (S, P0)** Release rehearsal: internal, then closed testing, then staged rollout plan; rollback and halt procedure written.
- [ ] **AL11-09 (S, P0)** Crash-free soak test with the simulator and real users in the pilot.
- [ ] **AL11-10 (S, P0)** Launch gate with the backend: all P0 done, security and accessibility checks passed, pilot readiness checklist signed.

### Exit criteria
Both apps pass the launch gate; `rider-v1.0.0` and `driver-v1.0.0` are tagged; the staged rollout begins at 5%.

---

## Lane AL12: Post-launch backlog

Ordered by what the pilot teaches you; each item becomes a mini-lane with the same structure.
- [ ] Google sign-in and identity linking, email sign-in (needs backend L15-01 and L15-02)
- [ ] Rider in-app M-PESA payment (per the payments decision)
- [ ] Scheduled rides
- [ ] Masked or proxied calling
- [ ] Optional biometric or device-credential lock
- [ ] Optional dynamic color setting
- [ ] Tablet and large-screen layouts
- [ ] Kimeru language
- [ ] Opt-in accessibility needs note shared with drivers (sensitive data, explicit consent)
- [ ] Late-night rider check-ins and destination change during a trip

---

## Appendix A: Module introduction schedule

| Module | Introduced in |
|---|---|
| `build-logic`, `core:common`, `core:model`, `core:designsystem`, `core:network`, `core:session`, `core:navigation`, `feature:auth` | AL0 |
| `core:security`, `core:preferences`, `feature:profile` | AL1 |
| `core:database`, `feature-driver:onboarding` | AL3 |
| `feature:notifications`, `core:location`, `core:observability`, `feature-rider:home`, `core:ui` (extract when shared composables appear) | AL4 |
| `core:realtime`, `feature-driver:home` | AL5 |
| `feature-driver:wallet` | AL6 |
| `feature-rider:trip`, `feature-rider:history`, `feature-driver:trip`, `feature-driver:earnings` | AL7 |
| `feature:safety` (and chat if separate) | AL9 |
| `feature:ratings`, `feature:support`, `feature-rider:promos` | AL10 |
| `benchmark` | AL11 |
| `core:testing` | When the first shared fake is needed (AL1) |

## Appendix B: Backend contract checkpoints

| Android lane | Needs from backend (see `PLAN.md` lanes) | If not ready |
|---|---|---|
| AL1 | OpenAPI v0.2 (L2) | Mock server with auth fixtures |
| AL2 | Config endpoint (L3) | Static config JSON |
| AL3 | Driver and upload endpoints (L4) | Fake repository, local file upload stub |
| AL4 | Devices, announcements, fare quotes (L5, L6) | Fixture quotes from real route samples |
| AL5 | WebSocket and location ingest (L7) | Local echo server |
| AL6 | Wallet endpoints (L8) | Fake wallet repository |
| AL7 | Rides, offers, state (L9) and the simulator | Scripted ride state machine in a fake |
| AL8 | Top-up and callbacks (L10) | Sandbox and fake status stream |
| AL9 | Safety endpoints (L11) | Fake SOS repository |
| AL10 | Ratings, promos, support (L12) | Fakes |
| AL11 | Production readiness (L14) | n/a |

## Appendix C: Device and test matrix
| Dimension | Coverage |
|---|---|
| Android versions | API 26, 29, 31, 34, 36 (emulators), plus real devices on at least three versions |
| Real devices | A 2 GB low-end phone (Tecno or Infinix), a Samsung A-series, a Xiaomi or Redmi, and a recent Pixel or equivalent |
| Networks | Offline, throttled 3G, flaky Wi-Fi, airplane mode toggling |
| Accessibility | TalkBack on, 200% font, high contrast, animations off, Switch Access sample run |
| Locales | English and Swahili |

## Appendix D: Week-one checklist for AL0 (you are new to multi-module)

1. Commit the current wizard project to Git. All later steps are small commits.
2. Open `gradle/libs.versions.toml` (it already exists from the wizard) and move every version number there; remove the `org.jetbrains.kotlin.android` plugin everywhere if present.
3. Set `compileSdk` and `targetSdk` to 36 and decide `minSdk`; sync and run both apps.
4. Create the `build-logic` folder with one convention plugin (`bar.android.library`), apply it to one new empty module, and confirm it compiles. Then add the others one at a time.
5. Create the Stage 1 modules (`core:common`, `core:model`, `core:designsystem`, `core:network`, `core:session`, `core:navigation`, `feature:auth`) as empty libraries and add them to `settings.gradle.kts`.
6. Make `app-rider` and `app-driver` depend on `core:designsystem` and show one themed screen from it.
7. Add Hilt and Navigation 3 to the apps, then LeakCanary and StrictMode in debug.
8. Add build types (`staging`, `release`) with R8 on, and run the `staging` build on a device.
9. Add the manifest security settings and the merged-manifest permission check.
10. Add detekt, ktlint, and CI last, once the structure stops changing daily.
    Ask for help at any numbered step; each one is small enough to do in a sitting.
