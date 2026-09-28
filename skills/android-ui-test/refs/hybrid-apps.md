# Hybrid Apps (Views + Compose)

For apps that mix XML Views and Compose (via `ComposeView`,
`AndroidView`, or gradual migration), tests often need both APIs.

## Choosing the API per screen

- Screen is Compose → Compose test API (`createComposeRule` /
  `createAndroidComposeRule`).
- Screen is Views → Espresso.
- Flow crosses both → pick the API of the *entry* screen and interop, or
  test each screen separately and let a Maestro flow cover the whole path.

## Espresso driving Compose (ComposeView inside Activity/Fragment)

Compose nodes are reachable from Espresso through the view hierarchy, but
prefer `composeTestRule` for assertions on the Compose parts:

```kotlin
@RunWith(AndroidJUnit4::class)
class ProfileFragmentTest {
    @get:Rule val composeRule = createAndroidComposeRule<ProfileActivity>()

    @Test fun savesProfile() {
        // View part
        onView(withId(R.id.nameField)).perform(typeText("Budi"), closeSoftKeyboard())
        // Compose part
        composeRule.onNodeWithText("Save").performClick()
        composeRule.onNodeWithTag("save_success").assertIsDisplayed()
    }
}
```

`createAndroidComposeRule` gives both `onView(...)` (Espresso) and
`onNode...` (Compose) against the same activity — this is the standard
hybrid entry point.

## Synchronization between the two

- Espresso waits for the Compose clock only when `IdlingResource` bridging
  is set up; modern androidx test artifacts handle basic sync, but custom
  async work still needs registration.
- If Compose recomposition is async relative to Espresso checks, use
  `composeRule.waitUntil { ... }` rather than sleeps.
- Keep coroutines on a controllable dispatcher via DI to make idle
  detection reliable.

## testTag vs resource-id for cross-process tools

If Maestro/UiAutomator must also see Compose nodes, add once at the app root
(debug builds only):

```kotlin
CompositionLocalProvider(LocalInspectionMode provides false) { /* not needed here */ }
// at the root composable:
Modifier.semantics { testTagsAsResourceId = true }
```

Then Compose `testTag("login_button")` appears as resource-id
`app.package:id/login_button` in the accessibility tree, usable by both
UiAutomator (`By.res`) and Maestro (`id: login_button`).

Gate with a debug flag so production builds don't expose test metadata:

```kotlin
val rootModifier = if (BuildConfig.DEBUG) {
    Modifier.semantics { testTagsAsResourceId = true }
} else Modifier
```

## Migration checklist for screens moving Views → Compose

1. Write the new Compose screen with semantics/testTags mirroring the old
   view IDs where tests target the same elements.
2. Port the Espresso assertions to Compose matchers (or keep the hybrid
   activity rule during the transition).
3. Keep the old test green until the screen is fully swapped, then delete
   the old test in the same PR.
4. Re-run any Maestro flow covering the screen — accessibility-tree-based
   flows usually survive if IDs/semantics were preserved.
