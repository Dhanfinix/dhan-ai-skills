# Compose UI Testing

In-process testing for Jetpack Compose screens using the semantics tree.
Same layer as Espresso (screen/component, fast, fakes injected), just a
different matcher API.

## Dependencies (only if not present)

```kotlin
// libs.versions.toml
androidx-compose-ui-test-junit4 = { module = "androidx.compose.ui:ui-test-junit4", version.ref = "composeBom" }
androidx-compose-ui-test-manifest = { module = "androidx.compose.ui:ui-test-manifest", version.ref = "composeBom" }
```

## Two setups: component vs full-app

**Component test (fastest, no device needed with Robolectric):**

```kotlin
class LoginContentTest {
    @get:Rule val composeRule = createComposeRule()

    @Test fun errorShownWhenPasswordEmpty() {
        composeRule.setContent {
            AppTheme { LoginContent(onLogin = { _, _ -> }) }  // the composable directly
        }
        composeRule.onNodeWithTag("email").performTextInput("user@test.dev")
        composeRule.onNodeWithText(stringResource(R.string.sign_in)).performClick() // see strings note
        composeRule.onNodeWithTag("error").assertIsDisplayed()
    }
}
```

Use a `ComponentActivity` in `setContent` when the composable needs resources:
`composeRule.activity.setContent { ... }` or the `ComponentActivity` guidance
from the Compose docs.

**Full-app / navigation test (instrumented or Robolectric):**

```kotlin
@get:Rule val composeRule = createAndroidComposeRule<MainActivity>()
```

## Matcher strategy (in priority order)

1. **Role + semantics**: `onNode(hasRole(Role.Button).and(hasText("Submit")))`
2. **Text**: `onNodeWithText("...")` / `onNodeWithText(R.string.x)` — fine for
   stable copy
3. **testTag**: only when #1–2 would need >3 combined matchers, or for
   dynamic content with no meaningful text:

```kotlin
composeRule.onNodeWithTag("cart_total").assertTextContains("$42.00")
```

If the project uses `testTagsAsResourceId = true` (needed for Maestro/
UiAutomator), tags become `resource-id` and are visible cross-process too.

## Interaction cheat sheet

| Purpose | Idiom |
|---|---|
| Type | `performTextInput("...")` |
| Click | `performClick()` |
| Scroll lazy list to node | `performScrollToNode(hasTestTag("x"))` |
| Scroll to index | `performScrollToIndex(n)` (avoid in committed tests) |
| Swipe | `performTouchInput { swipeRight() }` |
| Assert displayed | `assertIsDisplayed()` |
| Assert enabled/disabled | `assertIsEnabled()` / `assertIsNotEnabled()` |
| Assert selected (tab) | `assertisSelected()` |
| Count nodes | `onAllNodesWithTag("item").fetchSemanticsNodes().size` |
| Wait for async | `composeRule.waitUntil(timeoutMillis) { onAllNodesWithTag("loaded").fetchSemanticsNodes().isNotEmpty() }` |

## State & recomposition

- Pass ViewModel state as parameters to screen composables; test the
  composable with different state values directly (loading / error / empty /
  success) — no network needed.
- Use `DeviceConfigurationOverride` (font scale, window size, dark theme)
  for adaptation checks — see Compose testing common patterns.
- Verify state restoration after config change if the screen holds
  `rememberSaveable` state: `composeRule.activity.recreate()`.

## Main clock (animations & coroutines)

```kotlin
composeRule.mainClock.advanceTimeBy(500)   // control animation time
composeRule.mainClock.autoAdvance = false  // freeze for deterministic anims
```

Prefer this over disabling animations globally.

## Anti-patterns

- `onNodeAt(2)` positional access → breaks on any layout change.
- Asserting internals of deeply nested layout → assert on user-visible
  semantics instead.
- `testTag` on every node "just in case" → noise; add tags with purpose.
- Testing private child composables in isolation unless they're reused.
