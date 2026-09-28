# Cross-Platform Parity Testing (Android + iOS)

Use the **same Maestro YAML flow** against both the Android APK and the iOS
app to verify flow parity: if the journey passes on Android but fails on
iOS, a UI/flow difference exists — the "missed UI" problem.

## Core principle: text-first matching

Maestro matches against the **accessibility tree** (what is visible on
screen), so most elements work cross-platform with no code changes:

```yaml
- tapOn: "Sign in"
- assertVisible: "Welcome back"
```

**Start every parity journey with text matching only.** Reach for IDs
later, only for elements where text matching fails — never audit/align
component IDs up front.

## Shared structure

```
.maestro/
├── shared/
│   ├── login-journey.yaml        # one file, both platforms
│   └── platform-dialogs/
│       ├── android.yaml          # conditional per-OS steps
│       └── ios.yaml
├── critical/                     # subset for PR gate
└── config/
    ├── android.yaml              # appId wrapper
    └── ios.yaml
```

## Per-platform differences and how to abstract them

### 1. appId / bundle ID

```yaml
# config/android.yaml
appId: com.example.app
---
- runFlow: ../shared/login-journey.yaml

# config/ios.yaml
appId: com.example.app.iOS
---
- runFlow: ../shared/login-journey.yaml
```

Or a single flow with env vars: `appId: ${APP_ID}` and
`maestro test -e APP_ID=... login-journey.yaml`.

### 2. System dialogs (permission text differs per OS)

Prefer the cross-platform automatic path:

```yaml
- launchApp:
    clearState: true
    permissions: all: allow     # handled by Maestro on both platforms
```

Only tap dialogs manually when testing the deny path — then branch:

```yaml
- runFlow:
    when:
      platform: Android
    commands:
      - tapOn: "Don't allow"
- runFlow:
    when:
      platform: iOS
    commands:
      - tapOn: "Don't Allow"
```

### 3. When text matching breaks: fix the element, minimally

| Symptom | Fix (per element only) |
|---|---|
| Icon-only button (no text) | `contentDescription` (Android) / `accessibilityLabel` (iOS) — required for screen readers anyway |
| Dynamic text ("$42.00") | regex: `tapOn: { text: "\\$\\d+\\.\\d{2}" }` or an ID on that element |
| Copy intentionally differs per platform | branch with `when: platform:` |
| Duplicate texts in a list | structural matchers (`below:`/`above:`) or an ID on that element |

**Anti-over-engineering rule**: do not add IDs "for stability" everywhere.
Typically only 10–20% of elements need explicit identifiers. When you do
add them, use the **same name on both platforms** (`login_button` as
Compose `testTag` and SwiftUI `accessibilityIdentifier`).

### 4. Platform-conditional behavior in the app itself

```yaml
- runFlow:
    when:
      visible: ".*Android-specific copy.*"
    commands:
      - tapOn: "Continue"
- runFlow:
    when:
      visible: ".*iOS-specific copy.*"
    commands:
      - tapOn: "Continue"
```

Prefer this (`when: visible`) over `when: platform` when the divergence is
content, not chrome — it doubles as a parity assertion for that copy.

## Running parity in CI

```yaml
# Same flow, two jobs
jobs:
  parity-android:
    steps: [boot emulator, install APK, maestro test .maestro/config/android.yaml]
  parity-ios:
    runs-on: macos-latest
    steps: [boot simulator, install app, maestro test .maestro/config/ios.yaml]
```

Both must be green. A one-platform failure is a **parity bug**, not a test
bug — file it to the platform that diverged.

## What parity flows do NOT verify

- Visual/design consistency (spacing, fonts, theme) → use screenshot
  testing per platform (Roborazzi/Shot on Android, snapshot tests on iOS).
- Component-level behavior → in-app tests (Espresso/Compose / XCUITest).
- Performance.

Parity journeys verify: **the user can accomplish the same goals with the
same visible flow on both platforms.**

## iOS-side notes (for Android-first authors)

- Simulator runs need macOS runners (or Maestro Cloud).
- iOS animations are slower; prefer `waitForAnimationToEnd` +
  assertion timeouts over text-flakiness workarounds.
- iOS keyboard behaviors differ (autocorrect may alter typed text) —
  disable autocorrect on test inputs or assert with regex.
