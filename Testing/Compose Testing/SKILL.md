---
name: compose-testing
description: >
  Jetpack Compose UI testing APIs and patterns.
  Load this skill when using ComposeTestRule, finding nodes by semantics,
  testing animations, asserting accessibility properties, verifying
  recomposition behavior, or debugging failing Compose tests.
---

# Compose Testing

## Overview
Compose Testing uses a semantic tree — a parallel representation of the UI — to find and interact with composables. Unlike Espresso's view hierarchy, Compose tests work through `SemanticsNode`s which expose accessibility properties. Tests are deterministic because `ComposeTestRule` controls the clock.

---

## Core Principles

- Tests interact through the **semantic tree** — not the visual layout
- **`testTag`** is the most reliable locator — use for non-text UI elements
- **`contentDescription`** serves double duty — accessibility and testability
- `ComposeTestRule` **controls the clock** — animations and async are deterministic
- Compose tests are **synchronous by default** — the rule awaits idle state after each action

---

## Test Rule Setup

```kotlin
// ✅ For isolated composable tests (no Activity needed)
@get:Rule
val composeRule = createComposeRule()

// ✅ For full Activity/NavHost tests
@get:Rule
val composeRule = createAndroidComposeRule<MainActivity>()

// ✅ For Hilt + Activity tests
@HiltAndroidTest
class MyScreenTest {
    @get:Rule(order = 0) val hiltRule = HiltAndroidRule(this)
    @get:Rule(order = 1) val composeRule = createAndroidComposeRule<MainActivity>()
}
```

---

## Semantic Matchers Reference

```kotlin
// ── Text ──────────────────────────────────────────────────────────────────
hasText("Submit")                          // exact match
hasText("ubmit", substring = true)         // substring
hasText("submit", ignoreCase = true)       // case-insensitive

// ── Tags ──────────────────────────────────────────────────────────────────
hasTestTag("my_button")

// ── Accessibility ─────────────────────────────────────────────────────────
hasContentDescription("Back")
hasContentDescription("icon", substring = true)

// ── Role / Type ───────────────────────────────────────────────────────────
isDialog()
isPopup()
isFocused()
isEnabled()
isNotEnabled()
hasClickAction()
hasScrollAction()
isSelected()
isToggleable()
isOn()
isOff()

// ── Hierarchy ─────────────────────────────────────────────────────────────
hasParent(hasTestTag("card"))
hasAnyChild(hasText("Ali"))
hasAnySibling(hasText("Submit"))

// ── Combining ─────────────────────────────────────────────────────────────
hasText("Ali") and isEnabled()
hasText("Ali") or hasTestTag("ali_node")
```

---

## Finding Nodes

```kotlin
// Single node — throws if 0 or 2+ found
composeRule.onNodeWithText("Submit")
composeRule.onNodeWithTag("loading_indicator")
composeRule.onNodeWithContentDescription("Close")
composeRule.onNode(hasText("Ali") and isEnabled())

// Multiple nodes
composeRule.onAllNodesWithText("Item")
composeRule.onAllNodesWithTag("list_item")
composeRule.onAllNodes(hasClickAction())

// Indexed access
composeRule.onAllNodesWithTag("list_item")[0]
composeRule.onAllNodesWithTag("list_item").onFirst()
composeRule.onAllNodesWithTag("list_item").onLast()
```

---

## Actions

```kotlin
val node = composeRule.onNodeWithTag("my_button")

// Click
node.performClick()
node.performLongClick()
node.performDoubleClick()

// Text input
node.performTextInput("Hello")
node.performTextClearance()
node.performTextReplacement("New text")

// Scroll
node.performScrollTo()                     // scroll this node into view
node.performScrollToIndex(10)              // scroll LazyList to index
node.performScrollToKey("item_key")        // scroll LazyList to keyed item
node.performScrollToNode(hasText("Target"))

// Gestures
node.performTouchInput { swipeLeft() }
node.performTouchInput { swipeUp() }
node.performTouchInput { click(center) }

// IME
node.performImeAction()
```

---

## Assertions

```kotlin
val node = composeRule.onNodeWithTag("result_text")

// Existence
node.assertExists()
node.assertDoesNotExist()

// Visibility
node.assertIsDisplayed()
node.assertIsNotDisplayed()

// State
node.assertIsEnabled()
node.assertIsNotEnabled()
node.assertIsFocused()
node.assertIsSelected()
node.assertIsOn()
node.assertIsOff()

// Content
node.assertTextEquals("Exact text")
node.assertTextContains("partial")
node.assertContentDescriptionEquals("desc")
node.assertHasClickAction()

// Count
composeRule.onAllNodesWithTag("item").assertCountEquals(3)
```

---

## Testing State Changes

```kotlin
// ✅ Test that UI updates when state changes
@Test
fun showsErrorAfterFailedLoad() {
    var state: UserListUiState by mutableStateOf(UserListUiState.Loading)

    composeRule.setContent {
        AppTheme {
            UserListContent(state = state, onUserClick = {})
        }
    }

    // Initial loading state
    composeRule.onNodeWithTag("loading_indicator").assertIsDisplayed()

    // Update state
    state = UserListUiState.Error("Network failed")

    // Verify UI updated
    composeRule.onNodeWithText("Network failed").assertIsDisplayed()
    composeRule.onNodeWithTag("loading_indicator").assertDoesNotExist()
}
```

---

## Clock Control and Animations

```kotlin
// ✅ Disable auto-advance for manual clock control
composeRule.mainClock.autoAdvance = false

// Advance by specific duration
composeRule.mainClock.advanceTimeBy(300L)

// Advance until idle
composeRule.mainClock.advanceTimeUntilIdle()

// ✅ Test animated visibility
@Test
fun animatedContentAppearsAfterDelay() {
    var visible by mutableStateOf(false)

    composeRule.setContent {
        AnimatedVisibility(visible = visible) {
            Text("Hello", modifier = Modifier.testTag("animated_text"))
        }
    }

    composeRule.onNodeWithTag("animated_text").assertDoesNotExist()

    visible = true
    composeRule.mainClock.advanceTimeBy(500L)

    composeRule.onNodeWithTag("animated_text").assertIsDisplayed()
}
```

---

## Debugging Failing Tests

```kotlin
// ✅ Print semantic tree to understand what's available
composeRule.onRoot().printToLog("TEST_TAG")

// ✅ Print subtree
composeRule.onNodeWithTag("user_card").printToLog("CARD_TREE")

// Output shows full semantic tree:
// Node #1 at (0.0, 0.0, 411.4, 72.7)px
//  |-Node #2 at (16.0, 12.0, 395.4, 60.7)px
//      Text = "Ali Rezaei"
//      Actions = [OnClick, GetTextLayoutResult]
```

---

## waitUntil

```kotlin
// ✅ Wait for async UI change (max timeout)
composeRule.waitUntil(timeoutMillis = 5_000) {
    composeRule
        .onAllNodesWithTag("user_item")
        .fetchSemanticsNodes()
        .isNotEmpty()
}

// ✅ Wait for node to appear
composeRule.waitUntilAtLeastOneExists(
    hasTestTag("user_item"),
    timeoutMillis = 5_000
)

// ✅ Wait for node to disappear
composeRule.waitUntilDoesNotExist(
    hasTestTag("loading_indicator"),
    timeoutMillis = 5_000
)
```

---

## Anti-Patterns

- Using `Thread.sleep()` for async waits — use `waitUntil` or let the test rule handle idle
- Locating nodes by position index in UI — fragile; use `testTag` or content
- Not wrapping test content in `AppTheme` — may fail if theme-dependent assertions are used
- Asserting on implementation-level composable names — use semantic properties instead
- `autoAdvance = false` without remembering to advance — test hangs indefinitely

---

## Related Skills
- `ui-testing` — full screen UI tests with Activity
- `compose` — Compose fundamentals including `testTag` placement
- `snapshot-testing` — screenshot-based regression testing
- `espresso` — View-based UI testing
- `accessibility` — semantic properties used in both testing and accessibility
