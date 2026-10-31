# BAR Android: Agent Instructions

## Project
BAR (Book A Ride) is a ride-hailing platform for Meru, Kenya. This repo is one Gradle multi-module
project with two apps: rider `dev.korryr.bar.rider` and driver `dev.korryr.bar.driver`.

Stack: Kotlin, Jetpack Compose, Material 3, Hilt (KSP), Navigation 3, Room, DataStore, Retrofit,
OkHttp, kotlinx.serialization. compileSdk and targetSdk 36. The backend lives in a separate repo
(`bar-backend`); its OpenAPI spec is the API contract.

## Read on demand (never read all of these upfront)
- `docs/ANDROID_PLAN.md`: lanes and tasks. Find the task ID before writing code.
- `docs/ANDROID_PRD.md`: requirements (AND-* IDs), architecture, security, design system.
- `docs/decisions.md`: settled decisions. Do not reopen them.
  Read only the sections that matter for the current task.

## How to work
1. Identify the task ID and requirement IDs. If there is none, ask.
2. For a change touching more than 3 files, write a short plan first and wait for approval.
3. Build the smallest working vertical slice. One concern per change.
4. Before finishing, run the checks below, fix what you broke, then report: what changed, what you
   verified, what is left.
5. Ask instead of guessing: unclear requirement, missing API field, unsure library API. Check the
   version in `gradle/libs.versions.toml` and the official docs. Never invent APIs.
6. No scope creep: no drive-by refactors, renames, or reformatting of code you were not asked to touch.

## Commands
```
./gradlew :app-rider:assembleDebug
./gradlew :app-driver:assembleDebug
./gradlew testDebugUnitTest
./gradlew lintDebug detekt spotlessCheck
./gradlew spotlessApply
./gradlew :app-rider:assembleStaging :app-driver:assembleStaging
```
Both apps must build after every change. Some tasks only exist after lane AL0; if a task is
missing, say so instead of skipping it silently.

## Architecture (non-negotiable)
- Feature-based MVVM with unidirectional data flow. Feature layout: `ui/`, `domain/`, `data/`, `di/`, `navigation/`.
- ViewModel exposes one `StateFlow<UiState>`, accepts one `onAction(UiAction)`, and sends one-off events through a `UiEffect` channel. UI uses `collectAsStateWithLifecycle()`.
- No `Context`, composables, or Android framework types in ViewModels, use cases, or repositories (except `SavedStateHandle`).
- Repositories are the single source of truth. Map DTOs to domain models at the data boundary. Errors are `AppError`.
- Coroutines: inject dispatchers, no `GlobalScope`, no blocking calls, respect cancellation.
- Hilt with KSP only. Never KAPT. Never apply `org.jetbrains.kotlin.android` (AGP has built-in Kotlin).
- Navigation 3: keys are `@Serializable` `NavKey`s in `:core:navigation`; each app owns its back stack; features expose entries only. Pass IDs, never sensitive data.
- Module rules: features depend on core only; no feature to feature; rider and driver modules never depend on each other; core never depends on features or apps. Anything needing background location, a foreground service, or camera permission lives only in driver modules.
- Everything is `internal` unless it must be public.
- The server decides prices, eligibility, ride state, and payment status. Never compute or trust them on the client.
- Dependency and plugin versions live only in `gradle/libs.versions.toml`. Never add a dependency without asking.

## Where things go
- Reusable UI: `:core:designsystem` (primitives) or `:core:ui` (shared composites)
- New screen: the feature module, `ui/`
- API call: feature `data/`, using the generated client from `:core:network`
- Navigation key: `:core:navigation`
- Shared domain model: `:core:model`
- Test fakes and helpers: `:core:testing`

## Code style
- **KISS.** The simplest thing that works. No speculative abstractions. No interface with a single implementation unless a test or module boundary needs it.
- **DRY with judgment.** Duplicate once, abstract on the third use. Shared code goes to core, never copy-pasted between apps.
- **No comments.** No KDoc, no TODOs, no commented-out code. Make names and structure explain the code. The only exception is R8 rule files, where each rule gets one `# WHY:` line.
- **Size limits:** file under 350 lines (hard limit 500), function or composable under 60 lines, class under 400, line under 100 characters, 5 or fewer parameters (group extras into a state or config object). Split by responsibility before reaching a limit. One primary type or composable per file.
- One public type or composable per file, named after the file. Previews go in a separate `*Previews.kt`.
- Immutable by default: `val`, `data class`, `sealed interface`. Exhaustive `when`, early returns, expression bodies. No `!!`. No `lateinit` unless injected. No magic numbers or strings (use tokens or constants).
- Naming: `*Screen`, `*ViewModel`, `*UiState`, `*UiAction`, `*UiEffect`, `*Repository`, `*Mapper`, use cases as verb phrases.
- Prefer composition over inheritance; small extension functions for helpers.

## Compose and design system
- **Reuse before creating.** Search `:core:designsystem` and `:core:ui` first. Use the `Bar*` wrappers (`BarTheme`, `BarButton`, `BarMotion`, `BarShapes`), never raw Material components scattered through features.
- A reusable composable is stateless: state in, events out, `modifier: Modifier = Modifier` as the first optional parameter, content slots for flexibility, no ViewModel access, no hard-coded strings.
- Used by two features or both apps: `:core:designsystem` or `:core:ui`. Otherwise `ui/components` inside the feature.
- Colors, shapes, typography, motion, and spacing come only from theme tokens. **No gradients. No hard-coded colors.** Red is for danger and SOS only. Text on yellow is always dark ink.
- Motion goes through `BarMotion`, honors reduce-motion, and never delays time-critical actions (offer accept, SOS).
- Every screen handles loading, content, empty, and error states, and has previews for each.
- User text lives in string resources (English and Swahili, plurals via `plurals`). Never hard-code user-facing text.
- Accessibility is part of done: content descriptions, roles, heading semantics, 48 dp touch targets, works at 200% font, no color-only meaning, announcements for ride events.
- Edge-to-edge insets and predictive back on every screen.

## Security and privacy (never violate)
- No secrets, keys, tokens, or credentials in code, resources, tests, logs, or commits. Do not read or edit `local.properties`, keystores, or `.env` files.
- Tokens and sensitive values only through `:core:security` and `:core:session`. Never plain preferences.
- Never log phone numbers, OTPs, tokens, locations, or document data. Use the redacting logger.
- Every manifest component declares `android:exported`. Cleartext off, `allowBackup=false`, permissions requested in context. Never add a driver-only permission to the rider app.
- Validate deep links and intents. No WebView without approval.
- Never weaken lint, detekt, security, or R8 settings to make a build pass.

## R8
No broad keeps of our own packages. Add a keep rule only when a `staging` build test proves it is
needed, with a `# WHY:` line. Reflection needs approval.

## Testing
- Add tests with every change: ViewModel state tests (Turbine, coroutines-test), repository tests with fakes, mapper and deep link tests, Compose UI tests with accessibility checks.
- Prefer fakes over mocks. No `Thread.sleep`. Test names describe behavior, for example `otpExpired_showsResendAction`.
- Never edit, delete, or `@Ignore` a test to make it pass.
- Cover failure paths: offline, process death, duplicate events, 401, 426, 503.

## Ask first (stop and confirm before touching)
`build-logic/`, major bumps in `gradle/libs.versions.toml`, `:core:security`, `:core:session`,
`:core:network` interceptors, manifests and permissions, `proguard-rules.pro`, signing config, CI
workflows, database migrations, anything involving money, wallet, auth, or SOS, deleting files,
adding dependencies.

## Git
- Branch `feat/alN-short-name` or `fix/...`. Conventional commits, for example `feat(auth): add otp screen`.
- Never push to `main`, never force push, never commit generated files or secrets.
- Small PRs that reference the task and requirement IDs. The PR description explains the why (the code has no comments), the tests run, and the risks.

## Communication
Be concise, no filler. State assumptions. If a requirement conflicts with these rules, say so and
propose options instead of picking silently.
