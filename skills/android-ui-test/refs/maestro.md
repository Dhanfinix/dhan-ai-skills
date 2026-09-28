# Maestro for Cross-App Journeys

Declarative YAML flows that drive the built APK from outside the process —
the tool for user journeys that Espresso cannot reach: permission round
trips, share sheets, OAuth browser flows, deeplinks launched externally,
and full end-to-end smoke paths.

**Principle: agents author flows; CI executes them deterministically.**
No AI/LLM at runtime — the YAML is the source of truth.

## Install & run

```bash
brew install maestro   # or curl -Ls "https://get.maestro.dev" | bash
maestro test .maestro/login-journey.yaml      # against booted emulator/device
```

No SDK in the app, no instrumentation dependency — works on the built APK.

## Flow anatomy

```yaml
# .maestro/login-journey.yaml
appId: com.example.app
name: Login journey
---
- launchApp:
    clearState: true                      # deterministic starting point
- runFlow: shared/accept-notifications.yaml  # reusable partial flows
- tapOn:
    id: "email_input"                     # resource-id; Compose testTag via testTagsAsResourceId
- inputText: "user@test.dev"
- tapOn:
    id: "password_input"
- inputText: "s3cret!"
- tapOn: "Sign in"                          # text matcher — fine for stable copy
- assertVisible:
    id: "home_root"
- assertVisible: "Welcome back"
```

## Selector priority

1. `id:` — resource-id (most stable). For Compose, requires
   `testTagsAsResourceId = true` (debug-gated) — see refs/hybrid-apps.md.
2. `text:` — for stable, non-localized or canonical copy.
3. Everything else (index, coordinates) — avoid in committed flows.

## Recipes for the hard cases

### Deeplink (no manual navigation)

```yaml
- launchApp:
    clearState: false
    deepLink: "myapp://product/12345"
- assertVisible:
    id: "product_detail_root"
```

### Permission round trip (deny path)

```yaml
- launchApp:
    clearState: true
    permissions:                             # auto-accept on launch for granted paths
      all: allow
- # for deny-path UX, drop `permissions` and tap the system dialog:
- tapOn: "Don't allow"                       # system dialog; text varies by OS version
```

For "allow" paths always prefer `permissions: all: allow` over tapping —
version-proof.

### Share sheet round trip

```yaml
- tapOn:
    id: "share_button"
- tapOn: "Copy link"                         # system sheet interaction (black-box)
- assertVisible: "Link copied"
```

### OAuth / browser round trip

```yaml
- tapOn: "Continue with Google"
- tapOn:
    text: ".*@example.com"                   # in the browser/account picker
    index: 0
- assertVisible:
    id: "home_root"
```

These browser-side steps are the most fragile in any stack — keep them in
Maestro (re-runnable in seconds) rather than UiAutomator. Use
`runFlow` with `retryTapOnFailure`-style patterns or `- waitForAnimationToEnd`
before flaky system transitions.

### Waiting instead of sleeping

```yaml
- waitForAnimationToEnd
- assertVisible:                             # assertions retry internally
    id: "order_list"
    timeout: 10000
```

Never `runScript` with sleeps.

## Composing flows for a big app

- `runFlow:` partials for shared setup: login, onboarding dismissal,
  permission acceptance. Keep them in `.maestro/shared/`.
- `clearState: true` only when the test needs first-launch behavior;
  default `false` to reuse installed state for speed.
- Environment/config: `- runScript` for dynamic values or use
  `${ENV_VAR}` substitution via Maestro env; avoid hardcoding accounts.
- One journey per file; name files after the user goal
  (`checkout-journey.yaml`), not the screen.

## Tagging flows for CI selection

Keep critical smoke journeys (2–5) in a separate folder (e.g.
`.maestro/critical/`) so CI can run them per-PR while the full suite runs
nightly.

## Anti-patterns

- Index/coordinate taps → break on any layout shift.
- One giant flow testing 20 features → split by user goal.
- AI/LLM decision-making at runtime → kills determinism and costs tokens;
  the agent fixes flows at authoring time, not execution time.
- Hardcoded credentials → use env vars / test accounts.
