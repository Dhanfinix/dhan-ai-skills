# Espresso Testing for XML Views

In-process, UI-thread-synchronized UI testing for View-based screens.
Fastest deterministic layer. Lives in `androidTest/` (instrumented) or `test/`
with Robolectric if the project is configured for it.

## Dependencies (only if not present)

```kotlin
// libs.versions.toml
androidx-test-espresso-espresso-core = { module = "androidx.test.espresso:espresso-core", version = "3.7.0" }
androidx-test-rules = { module = "androidx.test:rules", version = "1.7.0" }
androidx-test-runner = { module = "androidx.test:runner", version = "1.7.0" }
androidx-test-ext-junit = { module = "androidx.test.ext:junit", version = "1.3.0" }
```

## Anatomy of a screen test

```kotlin
@RunWith(AndroidJUnit4::class)
class LoginScreenTest {

    @get:Rule val hiltRule = HiltAndroidRule(this)          // if Hilt is used
    @get:Rule val activityRule = ActivityScenarioRule(MainActivity::class.java)

    @Before fun setUp() { hiltRule.inject() }

    @Test fun showsErrorWhenPasswordEmpty() {
        onView(withId(R.id.emailInput)).perform(typeText("user@test.dev"), closeSoftKeyboard())
        onView(withId(R.id.loginButton)).perform(click())
        onView(withId(R.id.errorText)).check(matches(isDisplayed()))
        onView(withId(R.id.loginButton)).check(matches(isEnabled())) // button re-enabled
    }
}
```

## Matcher & action cheat sheet

| Purpose | Idiom |
|---|---|
| Find by ID (preferred) | `onView(withId(R.id.x))` |
| Find by text (stable copy only) | `onView(withText(R.string.sign_in))` |
| Find by contentDescription | `onView(withContentDescription(R.string.back))` |
| Nested/adapter items | `onView(allOf(withId(...), hasDescendant(withText(...))))` |
| RecyclerView row | `onView(withRecyclerView(...)).atPosition(n)` or better, `onItem()` matchers |
| Type | `perform(typeText("..."), closeSoftKeyboard())` |
| Click | `perform(click())` |
| Scroll to | `perform(scrollTo())` (inside ScrollView) / `swipeUp()` |
| Assert displayed | `check(matches(isDisplayed()))` |
| Assert text | `check(matches(withText(...)))` |
| Assert count | `check(matches(hasChildCount(3)))` |
| Assert gone | `check(matches(withEffectiveVisibility(Visibility.GONE)))` |

## Synchronization (replaces all sleeps)

- Espresso auto-syncs with the UI thread and `IdlingResource`-registered work.
- For async work not tracked by the framework (coroutines, custom executors),
  register an `IdlingResource` or use the project's existing idling setup.
- For Compose-in-View hybrid screens, Espresso and `ComposeTestRule` can
  interoperate via `EspressoIdlingResource` — see refs/hybrid-apps.md.

## Intents (still in-process)

For flows that launch another activity *within the app* or a well-known
external intent you want to stub:

```kotlin
Intents.init()
intended(hasAction(Intent.ACTION_VIEW))                       // verify
intending(hasAction(Intent.ACTION_GET_CONTENT))
    .respondWith(Instrumentation.ActivityResult(...))          // stub picker
```

This handles photo pickers and in-app navigation *without* leaving the
instrumentation process. Real cross-app round trips → Maestro instead.

## Permissions

```kotlin
@get:Rule
val permissionRule = GrantPermissionRule.grant(
    Manifest.permission.CAMERA,
    Manifest.permission.POST_NOTIFICATIONS,
)
```

Grant rules cover the "granted" path with zero UiAutomator. Only test the
dialog UX itself (deny / "don't ask again") via refs/uiautomator.md or Maestro.

## State setup

- Inject fakes through DI (Hilt `@TestInstallIn` / test modules or
  `loadKoinModules`) for repository/auth layers.
- Launch into a specific screen state with `ActivityScenario` +
  intent extras, or by driving a fake to the desired state in `@Before`.

## Anti-patterns

- `Thread.sleep()` anywhere → replace with idling or `waitUntil`-style polling.
- `onView(withText("Welcome back!"))` on localized copy → use resource IDs.
- Positional/index-based matching in RecyclerViews that can reorder.
- One test asserting 15 things → split; each test asserts one behavior.
