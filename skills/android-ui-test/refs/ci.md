# CI Wiring for Android UI Tests

Principle: **deterministic tests run on every PR; slow realistic runs nightly.**
No AI decision-making at runtime — the agent only authors/maintains tests.

## Layered CI strategy

| Stage | Runs | Where | Trigger |
|---|---|---|---|
| PR gate | Espresso/Compose (in-app) | JVM (Robolectric) + emulator | every PR |
| PR gate (critical) | 2–5 Maestro journeys | emulator or Maestro Cloud | PR or merge to main |
| Nightly | full Maestro suite + in-app suite | emulator / device farm | cron |
| Weekly (optional) | real-device runs / exploratory agent | Firebase Test Lab | cron |

## Instrumented tests: prefer Gradle Managed Devices

GMD gives reproducible emulators without runner-specific flakiness:

```kotlin
// app/build.gradle.kts
android {
    testOptions {
        managedDevices {
            devices {
                create<com.android.build.api.dsl.ManagedVirtualDevice>("api34") {
                    device = "Pixel 6"
                    apiLevel = 34
                    systemImageSource = "aosp-atd"   // automated test device: faster boot
                }
                groups {
                    create("phoneGroup") { targets.add(devices["api34"]) }
                }
            }
        }
    }
}
```

```bash
./gradlew :app:api34DebugAndroidTest    # boot, run, tear down — reproducible
```

## GitHub Actions (emulator route)

```yaml
jobs:
  ui-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { distribution: temurin, java-version: '17' }
      - uses: gradle/actions/setup-gradle@v4
      - name: Run instrumented tests
        uses: reactivecircus/android-emulator-runner@v2
        with:
          api-level: 34
          arch: x86_64
          script: ./gradlew :app:connectedDebugAndroidTest
```

Or GMD on CI (no emulator-runner action needed): KVM-enabled Linux runners
support GMD; `./gradlew :app:api34DebugAndroidTest` boots its own emulator.

## Maestro in CI

**Option A — on the runner's emulator:**

```yaml
      - name: Boot emulator
        uses: reactivecircus/android-emulator-runner@v2
        with: { api-level: 34, arch: x86_64 }
      - name: Install app
        run: ./gradlew :app:assembleDebug && adb install -r app/build/outputs/apk/debug/app-debug.apk
      - name: Run critical journeys
        run: |
          curl -Ls "https://get.maestro.dev" | bash
          maestro test .maestro/critical/
```

**Option B — Maestro Cloud (no emulator management):**

```bash
maestro cloud --async app-debug.apk .maestro/critical/
```

## Firebase Test Lab (real devices, nightly)

```bash
gcloud firebase test android run \
  --type instrumentation \
  --app app-debug.apk \
  --test app-debug-androidTest.apk \
  --device model=Pixel7,version=34
```

Maestro zip upload to FTL is also supported for real-device journey runs.

## Flakiness policy

1. A test that fails intermittently gets quarantined (annotation/separate
   task), not deleted — with a tracking issue.
2. Retry-on-failure (e.g. `--rerun-tasks` per-test retries) is a diagnostic,
   not a fix; repeated retries hide real bugs.
3. Track flaky rate per layer: if cross-app/UiAutomator tests dominate,
   migrate those journeys to Maestro rather than hardening selectors.

## Secrets & test data

- Test accounts via CI secrets → env vars → Maestro `${ENV}` substitution
  or DI-injected fakes. Never committed.
- Prefer fakes over staging backends for PR runs; reserve staging-backend
  journeys for nightly where their flakiness costs less.
