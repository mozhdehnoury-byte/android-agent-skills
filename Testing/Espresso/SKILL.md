---
name: espresso
description: >
  Espresso UI testing for Android View-based UI.
  Load this skill when writing instrumented tests for Views, Fragments,
  or Activities, using ViewMatchers and ViewActions, testing RecyclerView,
  or interacting with dialogs and menus.
---

# Espresso

## Overview
Espresso is Android's UI testing framework for View-based UI. It synchronizes automatically with the UI thread and AsyncTasks, making tests deterministic. Espresso operates through three components: `onView()` to find views, `ViewActions` to interact, and `ViewAssertions` to verify.

---

## Core Principles

- Espresso **auto-syncs** with the main thread — no `Thread.sleep()` needed
- Find views with **`ViewMatcher`** — prefer `withId()` and `withContentDescription()`
- **`IdlingResource`** for custom async operations that Espresso can't detect
- **Hilt** replaces real dependencies for isolated UI tests
- Espresso works alongside Compose tests in mixed View/Compose apps

---

## Dependencies

```kotlin
// build.gradle.kts
dependencies {
    androidTestImplementation(libs.espresso.core)
    androidTestImplementation(libs.espresso.contrib)      // RecyclerView, DrawerLayout
    androidTestImplementation(libs.espresso.intents)      // intent verification
    androidTestImplementation(libs.androidx.test.rules)
    androidTestImplementation(libs.androidx.test.runner)
}
```

---

## Basic Test Structure

```kotlin
// ✅ Standard Espresso test
@RunWith(AndroidJUnit4::class)
@HiltAndroidTest
class UserListActivityTest {

    @get:Rule(order = 0) val hiltRule = HiltAndroidRule(this)
    @get:Rule(order = 1) val activityRule = ActivityScenarioRule(UserListActivity::class.java)

    @Inject lateinit var fakeRepository: FakeUserRepository

    @Before
    fun setup() { hiltRule.inject() }

    @Test
    fun showsUserNameInList() {
        fakeRepository.emit(listOf(
            User(id = "1", name = "Ali Rezaei", email = Email("ali@test.com"))
        ))

        onView(withText("Ali Rezaei")).check(matches(isDisplayed()))
    }

    @Test
    fun clickingUserOpensDetail() {
        fakeRepository.emit(listOf(
            User(id = "1", name = "Ali Rezaei", email = Email("ali@test.com"))
        ))

        onView(withText("Ali Rezaei")).perform(click())

        onView(withId(R.id.user_detail_title)).check(matches(isDisplayed()))
    }
}
```

---

## ViewMatchers Reference

```kotlin
// By ID
onView(withId(R.id.submit_button))

// By text
onView(withText("Submit"))
onView(withText(R.string.submit))
onView(withText(containsString("ubmit")))   // Hamcrest matcher

// By content description
onView(withContentDescription("Back"))
onView(withContentDescription(R.string.back))

// By hint
onView(withHint("Enter email"))

// By tag
onView(withTagKey(R.id.my_tag))
onView(withTagValue(equalTo("my_tag_value")))

// By type
onView(isAssignableFrom(EditText::class.java))

// Combining matchers
onView(allOf(withId(R.id.name_field), isDisplayed()))
onView(allOf(withText("Ali"), withParent(withId(R.id.user_card))))
```

---

## ViewActions

```kotlin
val view = onView(withId(R.id.my_view))

// Click
view.perform(click())
view.perform(longClick())
view.perform(doubleClick())

// Text input
view.perform(typeText("Hello"))
view.perform(clearText())
view.perform(replaceText("New text"))

// Keyboard
view.perform(pressImeActionButton())
view.perform(pressBack())
view.perform(closeSoftKeyboard())

// Scroll
view.perform(scrollTo())

// Swipe
view.perform(swipeLeft())
view.perform(swipeRight())
view.perform(swipeUp())
view.perform(swipeDown())
```

---

## ViewAssertions

```kotlin
val view = onView(withId(R.id.result_text))

// Visibility
view.check(matches(isDisplayed()))
view.check(matches(not(isDisplayed())))
view.check(matches(withEffectiveVisibility(Visibility.VISIBLE)))

// State
view.check(matches(isEnabled()))
view.check(matches(not(isEnabled())))
view.check(matches(isChecked()))
view.check(matches(isFocused()))

// Content
view.check(matches(withText("Expected")))
view.check(matches(withText(containsString("partial"))))
view.check(matches(withHint("placeholder")))

// Existence
view.check(matches(isDisplayed()))
view.check(doesNotExist())
```

---

## RecyclerView

```kotlin
// ✅ Scroll to position
onView(withId(R.id.recycler_view))
    .perform(RecyclerViewActions.scrollToPosition<RecyclerView.ViewHolder>(10))

// ✅ Click item at position
onView(withId(R.id.recycler_view))
    .perform(RecyclerViewActions.actionOnItemAtPosition<RecyclerView.ViewHolder>(0, click()))

// ✅ Click item matching a matcher
onView(withId(R.id.recycler_view))
    .perform(RecyclerViewActions.actionOnItem<RecyclerView.ViewHolder>(
        hasDescendant(withText("Ali Rezaei")),
        click()
    ))

// ✅ Assert on item at position
onView(withId(R.id.recycler_view))
    .check(RecyclerViewAssertions.withItemCount(5))
```

---

## Intent Verification (Espresso Intents)

```kotlin
// ✅ Verify outgoing intents
@get:Rule
val intentsRule = IntentsRule()

@Test
fun clickShareOpensShareSheet() {
    onView(withId(R.id.share_button)).perform(click())

    intended(allOf(
        hasAction(Intent.ACTION_SEND),
        hasType("text/plain")
    ))
}

// ✅ Stub incoming intents
intending(hasComponent(CameraActivity::class.java.name))
    .respondWith(ActivityResult(Activity.RESULT_OK, Intent().apply {
        putExtra("photo_uri", "content://mock_uri")
    }))
```

---

## IdlingResource for Custom Async

```kotlin
// ✅ Register IdlingResource for operations Espresso can't detect
class NetworkIdlingResource : IdlingResource {
    private var callback: IdlingResource.ResourceCallback? = null
    private var isIdle = true

    override fun getName() = "NetworkIdlingResource"
    override fun isIdleNow() = isIdle
    override fun registerIdleTransitionCallback(callback: IdlingResource.ResourceCallback) {
        this.callback = callback
    }

    fun setIdle(idle: Boolean) {
        isIdle = idle
        if (idle) callback?.onTransitionToIdle()
    }
}

// Register in test
val idlingResource = NetworkIdlingResource()
IdlingRegistry.getInstance().register(idlingResource)

// Unregister in @After
IdlingRegistry.getInstance().unregister(idlingResource)
```

---

## Mixed Compose + View Testing

```kotlin
// ✅ Espresso and Compose rules can coexist
@RunWith(AndroidJUnit4::class)
class MixedUiTest {

    @get:Rule val activityRule = ActivityScenarioRule(MainActivity::class.java)
    @get:Rule val composeRule = createAndroidComposeRule<MainActivity>()

    @Test
    fun testMixedUI() {
        // Espresso for View
        onView(withId(R.id.toolbar_title)).check(matches(withText("Home")))

        // Compose for Compose content
        composeRule.onNodeWithTag("compose_content").assertIsDisplayed()
    }
}
```

---

## Anti-Patterns

- `Thread.sleep()` for synchronization — use `IdlingResource` or rely on Espresso auto-sync
- Testing with real network — fake the data layer via Hilt
- `onView(withIndex(...))` for RecyclerView items — use `RecyclerViewActions` instead
- Not calling `closeSoftKeyboard()` when keyboard may obscure views — causes view not found errors
- Sharing mutable test state between test methods — each test must be independent

---

## Related Skills
- `ui-testing` — broader UI test setup and Hilt configuration
- `compose-testing` — Compose-specific test APIs
- `integration-testing` — testing below the UI layer
- `hilt` — replacing dependencies in tests
