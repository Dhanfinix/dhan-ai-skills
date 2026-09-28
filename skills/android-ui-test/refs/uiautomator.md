# UiAutomator for Simple Cross-App Interactions

UiAutomator is the escalation path *within* the native stack for system UI
that Espresso cannot reach. Use it narrowly: one or two cross-app steps
inside an otherwise-Espresso test. For full cross-app journeys, use Maestro.

## Dependencies (only if not present)

```kotlin
// libs.versions.toml
androidx-test-uiautomator-uiautomator = { module = "androidx.test.uiautomator:uiautomator", version = "2.3.0" }
```

## Core API

```kotlin
val device = UiDevice.getInstance(InstrumentationRegistry.getInstrumentation())

// Wait for an object — always wait, never assume
val obj = device.wait(Until.findObject(By.res("com.android.systemui", "id")), 5_000)
obj?.click()

// The shell from instrumentation (adb-equivalent, no separate process needed)
device.executeShellCommand("pm grant $packageName android.permission.CAMERA")

// Press system buttons
device.pressBack()          // also dismisses system dialogs
device.openNotification()
```

## Patterns by use case

### Permission dialogs

- "Granted" path → use `GrantPermissionRule` or `adb pm grant` instead of
  tapping the dialog. See refs/espresso-views.md.
- "Denied" path → deny via `adb shell pm revoke` before launching, or tap
  the dialog by its button resource:

```kotlin
val denyButton = device.wait(Until.findObject(By.text("Don't allow")), 5_000)
denyButton?.click()
```

Dialog button text varies across OEMs/versions — match by `By.text` with
fallbacks, or prefer the adb path when the dialog UX itself isn't the target.

### Notifications

```kotlin
// 1. Trigger the notification via the app/fake, then:
device.openNotification()
device.wait(Until.hasObject(By.textContains("Order shipped")), 5_000)
device.findObject(By.textContains("Order shipped")).click()
// 2. Assert the app opened the right screen
onView(withId(R.id.orderDetailRoot)).check(matches(isDisplayed()))
```

### System UI state

```kotlin
device.executeShellCommand("settings put global airplane_mode_on 1") // example
// Broadcast a change and reset afterwards in @After — restore device state!
```

## Mixing with Espresso

A common shape: Espresso for the app part, UiAutomator for the 1–2 system
steps in the middle.

```kotlin
@Test fun deniedNotificationPermissionShowsSettingsHint() {
    // arrange via DI fake + adb revoke
    device.executeShellCommand("pm revoke $packageName android.permission.POST_NOTIFICATIONS")
    // app-side with Espresso
    onView(withId(R.id.enableNotificationsButton)).perform(click())
    // system-side with UiAutomator
    val deny = device.wait(Until.findObject(By.text("Don't allow")), 5_000)!!
    deny.click()
    // back to app-side with Espresso
    onView(withText(R.string.notifications_disabled_hint)).check(isDisplayed())
}
```

## Debugging selectors

- Dump the hierarchy: `device.dumpWindowHierarchy(File("ui.xml"))` or
  `adb shell uiautomator dump` — inspect `resource-id`s before guessing.
- Compose nodes appear as generic elements unless the app sets
  `testTagsAsResourceId = true`; see refs/compose-testing.md.
- `By.res(package, id)` is stricter and more stable than `By.text`.

## Rules of engagement

- Keep UiAutomator steps out of per-screen tests; they belong in flow tests.
- Always `wait(Until..., timeout)` — UiAutomator has no Espresso-style auto-sync.
- Restore device state (grants, settings) in `@After` — CI emulators are shared.
- If a UiAutomator step becomes flaky or version-fragile, escalate that
  *journey* to Maestro instead of hardening selectors forever.
