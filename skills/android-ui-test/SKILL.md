---
name: android-ui-test
description: >
  Author, fix, and maintain UI tests for Android apps covering both XML Views
  and Jetpack Compose. Implements a layered strategy: Espresso/Compose in-app
  tests as the default, UiAutomator for simple cross-app interactions, and
  Maestro flows for full user journeys. Optimized for AI-agent authoring:
  deterministic tests in the repo, agent self-correction at authoring time.
metadata:
  author: team
  last-updated: '2026-09-28'
  keywords:
  - android
  - ui testing
  - espresso
  - compose testing
  - uiautomator
  - maestro
  - e2e
---

# Android UI Test Author

You are helping author, debug, and maintain UI tests for an Android app that
may contain XML Views, Jetpack Compose, or both (hybrid). Follow the layered
strategy below. **Default to the cheapest layer that can verify the behavior.**

## Step 0: Understand the project

Before writing any test, determine:

1. **UI toolkit**: Views (XML), Compose, or hybrid? Check imports
   (`android.widget`/`androidx.appcompat` vs `androidx.compose.*`), and
   `build.gradle.kts` for `compose-compiler` / buildFeatures.compose.
2. **Existing test stack**: look at `libs.versions.toml` or module build files
   for `espresso-core`, `ui-test-junit4`, `ui-automator`, `mockk`, `hilt`,
   `kaspresso`, Robolectric, etc. **Respect the existing stack** — do not
   introduce a second mocking or DI framework.
3. **Test source sets**: what lives in `test/` (JVM, possibly Robolectric) vs
   `androidTest/` (instrumented)? Follow the project's existing convention.
4. **DI framework**: how do instrumented tests get fakes? (Hilt
   `@HiltAndroidTest` + `HiltAndroidRule`, Koin `loadKoinModules`, or manual
   test doubles via constructor injection.)
5. **CI setup**: how are tests run? (Gradle Managed Devices, GitHub Actions
   emulator, Firebase Test Lab, Maestro Cloud?)

If no UI test infrastructure exists at all, hand off setup to the
`testing-setup` skill if available, then return here for authoring tests.

## Step 1: Classify the behavior to test

Use this decision tree for every behavior under test:

```
Does the behavior stay inside your app's process?
├─ YES → Is it one screen/component's state or logic?
│   ├─ YES → ESPRESSO (Views) or COMPOSE TEST (Compose)     → refs/espresso-views.md
│   │                                                        → refs/compose-testing.md
│   └─ NO (multi-screen flow, all within your app)
│       → Still prefer Espresso/Compose with navigation,
│         or split into per-screen tests. Only escalate
│         to Maestro if setup is genuinely unmanageable.
└─ NO (crosses app boundary: system dialogs, share sheet,
   notifications, OAuth browser, other apps)
    ├─ Is it a simple system interaction (grant/deny permission,
    │   tap a notification, dismiss a dialog)?
│   → YES → GRANT PERMISSION RULE / adb / UIAUTOMATOR      → refs/uiautomator.md
    └─ NO (OAuth round-trip, share to another app and back,
       full real-user journey, deeplink from outside)
       → MAESTRO FLOW                                        → refs/maestro.md

Does the journey need to run on BOTH Android and iOS (parity check,
"missed UI" between platforms)?
├─ YES → One shared Maestro YAML, text-first matching        → refs/cross-platform.md
└─ NO  → Single-platform Maestro flow                        → refs/maestro.md
```

**Escalation rule**: start at the cheapest layer. Escalate only when the
current layer is (a) technically impossible, or (b) measurably flaky
(breaks on refactors that don't change user-visible behavior, or fails
intermittently in CI). Record which layer each journey ended up in.

## Step 2: Make the app testable (selector hygiene)

Before writing tests, verify every UI element the test targets has a stable,
meaningful handle. This serves Espresso, Compose testing, Maestro, and screen
readers simultaneously:

**Compose** — prefer semantics over test tags where possible:
- Add `semantics { contentDescription = ... }` for meaningful elements,
  `Role.Button`, `Role.Tab`, etc.
- Use `Modifier.testTag("...")` only when semantic matchers would need >3
  conditions. If using test tags with UiAutomator/Maestro, ensure
  `Modifier.semantics { testTagsAsResourceId = true }` is set on the root
  (only in debug builds — gate with `BuildConfig.DEBUG`).
- Never use positional index (`onNodeAt(index)`) in committed tests.

**Views**:
- Set stable `android:id` on every interactive element the tests touch.
- Provide `contentDescription` for icon-only buttons.
- Do not rely on text matching for elements whose copy changes often;
  prefer IDs.

**Cross-layer consistency**: the same semantic/ID should be used by the
in-app test and the Maestro flow that covers the same element.

**Anti-over-engineering rule**: Maestro's primary matchers are text/label
based (accessibility tree), so most cross-platform journeys need **no IDs
at all**. Do not audit or align component IDs up front — add an explicit
identifier only to the specific element where text matching breaks
(icon-only button, dynamic text, duplicated copy), and use the same name
on both platforms when you do. See refs/cross-platform.md.

## Step 3: Author the test

- Views screen state/behavior → follow `refs/espresso-views.md`
- Compose screen state/behavior → follow `refs/compose-testing.md`
- Both toolkits in one flow (hybrid navigation) → `refs/hybrid-apps.md`
- Simple cross-app interactions → `refs/uiautomator.md`
- Full user journeys / deeplinks / share / OAuth → `refs/maestro.md`
- Journey on both Android AND iOS (parity) → `refs/cross-platform.md`
  (text-first matching; per-platform conditionals only for system dialogs
  and intentional copy differences)

### Fakes before mocks

For anything slow or nondeterministic (network, database, auth), prefer
injecting a fake through the DI framework over real backend or mock chains.
A UI test should verify UI behavior, not backend correctness. Only mock
platform callbacks that cannot be faked (e.g. `FusedLocationClient`).

### Determinism rules (non-negotiable)

- Never `Thread.sleep()` / `delay()` in tests. Use idling resources
  (Espresso), `waitUntil` (Compose), or `awaitUntil`/explicit waits
  (UiAutomator). Maestro has `waitForAnimationToEnd` and retry-aware
  assertions — use those instead of `runScript` sleeps.
- Tests must not depend on execution order or device locale (use
  resource IDs, not hardcoded strings, when copy is localized).
- Every test must be independently re-runnable (`@Before` resets state;
  Maestro flows start with a well-defined `launchApp` state).

## Step 4: Agent self-correction loop

When a test you wrote fails, iterate at the cheapest possible loop:

1. **Read the failure output first.** Distinguish: selector not found
   (fix the matcher or add a stable ID), assertion failed (check what
   the state actually was — screenshot/layout dump), or infrastructure
   (emulator boot, install timeout — rerun before changing code).
2. **In-app tests**: fix the test (or the missing test ID) and re-run
   `./gradlew :module:connectedDebugAndroidTest --tests "..."` or the
   Robolectric-equivalent. Prefer the JVM/Robolectric loop for iteration
   speed when available; do a final instrumented run before declaring done.
3. **Maestro flows**: edit the YAML and re-run `maestro test flow.yaml`
   against the already-installed APK — no rebuild needed. Use
   `maestro studio` or a hierarchy dump (`maestro hierarchy`) to find the
   correct selector instead of guessing.
4. **Selector fixes**: prefer adding a stable semantic/ID to production code
   (via debug-only `testTag` or `contentDescription`) over loosening the
   matcher to something fragile.
5. **Do not delete an assertion to make a test pass.** If the behavior
   genuinely changed, confirm with the human or the task description first.
6. Stop after ~5 failed fix attempts on one issue and report what you
   learned (root cause hypothesis, evidence) instead of thrashing.

## Step 5: Run and verify

- JVM/Robolectric (if configured): `./gradlew :module:testDebugUnitTest`
- Instrumented: `./gradlew :module:connectedDebugAndroidTest`
  (single class: `--tests "com.example.LoginScreenTest"`)
- Maestro: `maestro test .maestro/login-journey.yaml`
- Before declaring done: run the whole affected test class/suite, not just
  the single test, to catch state leakage between tests.

## Step 6: CI wiring

Follow `refs/ci.md` for Gradle Managed Devices, GitHub Actions emulator
setup, and Maestro in CI. Key principle: deterministic tests run on every
PR; slow real-device runs happen nightly.

## Non-goals

- Screenshot/paparazzi/Roborazzi testing → see the `testing-setup` skill.
- Unit testing of ViewModels/Repositories → standard unit test practice.
- Writing exploratory agent harnesses (DroidRun/AppAgent) → out of scope.

## Output expectations

When asked to "add UI tests for X", you should deliver:
1. The test file(s) in the correct source set, following project conventions.
2. Any production-code changes needed for stable selectors (clearly listed
   and justified, debug-gated where possible).
3. The exact commands to run the new tests locally.
4. A short note on which layer each behavior was assigned to and why.
