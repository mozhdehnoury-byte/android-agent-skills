---
name: ui-testing
description: >
  UI testing for Android apps running on device or emulator.
  Load this skill when writing instrumented UI tests, testing navigation
  flows, verifying screen content under different states, or setting up
  the UI test infrastructure with Hilt and test rules.
---

# UI Testing

## Overview
UI tests verify the full user experience by running on a real device or emulator. They interact with the UI through semantic actions and verify the rendered output. In Compose, the primary tool is `ComposeTestRule`. For View-based UI, Espresso is used. Both can be combined in the same test suite.

---

## Core Principles

- UI tests are **slow** — run on CI only; prefer unit + integration tests for logic
- Use **semantic properties** for finding components — `contentDescription`, `testTag`
- **Fake the data layer** via Hilt — never hit real network in UI tests
- **One scenario per test** — a test that fails should point to exactly one thing
- Use **`StateFlow` seeding** to set up UI state before assertions

---

## Dependencies

```kotlin
// build.gradle.kts
dependencies {
    androidTestImplementation(libs.androidx.compose.ui.test.junit4)
    androidTestImplementation(libs.hilt.android.testing)
    kspAndroidTest(libs.hilt.compiler)
    androidTestImplementation(libs.androidx.test.runner)
    debugImplementation(libs.androidx.compose.ui.test.manifest)
}
```

---

## Compose UI Test Basics

```kotlin
// ✅ Basic Compose UI test
@HiltAndroidTest
class UserListScreenTest {

    @get:Rule(order = 0)
    val hiltRule = HiltAndroidRule(this)

    @get:Rule(order = 1)
    val composeRule = createAndroidComposeRule<MainActivity>()

    @Inject lateinit var fakeRepository: FakeUserRepository

    @Before
    fun setup() { hiltRule.inject() }

    @Test
    fun showsLoadingInitially() {
        composeRule.onNodeWithTag("loading_indicator").assertIsDisplayed()
    }

    @Test
    fun showsUsersWhenLoaded() {
        // Seed data
        val users = listOf(User(id = "1", name = "Ali Rezaei", email = Email("ali@test.com")))
        fakeRepository.emit(users)

        composeRule
            .onNodeWithText("Ali Rezaei")
            .assertIsDisplayed()
    }

    @Test
    fun showsEmptyStateWhenNoUsers() {
        fakeRepository.emit(emptyList())

        composeRule
            .onNodeWithTag("empty_state")
            .assertIsDisplayed()
    }

    @Test
    fun navigatesToDetailOnUserClick() {
        val users = listOf(User(id = "1", name = "Ali Rezaei", email = Email("ali@test.com")))
        fakeRepository.emit(users)

        composeRule
            .onNodeWithText("Ali Rezaei")
            .performClick()

        composeRule
            .onNodeWithTag("user_detail_screen")
            .assertIsDisplayed()
    }
}
```

---

## Test Tags

```kotlin
// ✅ Add testTag to composables for reliable test targeting
@Composable
fun LoadingIndicator(modifier: Modifier = Modifier) {
    CircularProgressIndicator(
        modifier = modifier.testTag("loading_indicator")
    )
}

@Composable
fun UserListContent(
    state: UserListUiState,
    onUserClick: (String) -> Unit
) {
    when (state) {
        is UserListUiState.Loading -> LoadingIndicator()
        is UserListUiState.Success -> LazyColumn(
            modifier = Modifier.testTag("user_list")
        ) {
            items(state.users, key = { it.id }) { user ->
                UserRow(
                    user = user,
                    onClick = { onUserClick(user.id) },
                    modifier = Modifier.testTag("user_item_${user.id}")
                )
            }
        }
        is UserListUiState.Error -> ErrorView(
            message = state.message,
            modifier = Modifier.testTag("error_view")
        )
    }
}
```

---

## Testing Isolated Composables (No Activity)

```kotlin
// ✅ Test a composable in isolation — no Activity, no navigation
class UserCardTest {

    @get:Rule
    val composeRule = createComposeRule()

    @Test
    fun displaysUserName() {
        val user = User(id = "1", name = "Ali Rezaei", email = Email("ali@test.com"))
        var clicked = false

        composeRule.setContent {
            AppTheme {
                UserCard(user = user, onUserClick = { clicked = true })
            }
        }

        composeRule.onNodeWithText("Ali Rezaei").assertIsDisplayed()
        composeRule.onNodeWithText("ali@test.com").assertIsDisplayed()
    }

    @Test
    fun invokesCallbackOnClick() {
        val user = User(id = "1", name = "Ali Rezaei", email = Email("ali@test.com"))
        var clickedId: String? = null

        composeRule.setContent {
            AppTheme {
                UserCard(user = user, onUserClick = { clickedId = it })
            }
        }

        composeRule.onNodeWithText("Ali Rezaei").performClick()
        assertThat(clickedId).isEqualTo("1")
    }
}
```

---

## Common Node Finders

```kotlin
// By text
composeRule.onNodeWithText("Submit")
composeRule.onNodeWithText("Submit", ignoreCase = true)

// By testTag
composeRule.onNodeWithTag("loading_indicator")

// By contentDescription (accessibility)
composeRule.onNodeWithContentDescription("Back")

// By semantic role
composeRule.onNode(hasClickAction())
composeRule.onNode(isDialog())

// By position in list
composeRule.onAllNodesWithTag("user_item").onFirst()
composeRule.onAllNodesWithTag("user_item")[2]

// Combined matchers
composeRule.onNode(hasText("Ali") and hasTestTag("user_item_1"))
```

---

## Common Actions and Assertions

```kotlin
// Actions
node.performClick()
node.performTextInput("test input")
node.performTextClearance()
node.performScrollTo()
node.performScrollToIndex(5)

// Assertions
node.assertIsDisplayed()
node.assertIsNotDisplayed()
node.assertIsEnabled()
node.assertIsNotEnabled()
node.assertTextEquals("Expected")
node.assertTextContains("partial")
node.assertContentDescriptionEquals("desc")
node.assertExists()
node.assertDoesNotExist()
```

---

## Waiting for Async Operations

```kotlin
// ✅ Compose test rule waits for recomposition automatically
// For async operations that need explicit waiting:
composeRule.waitUntil(timeoutMillis = 5_000) {
    composeRule.onAllNodesWithTag("user_item").fetchSemanticsNodes().isNotEmpty()
}

// ✅ Advance time in tests with test dispatcher
composeRule.mainClock.autoAdvance = false
composeRule.mainClock.advanceTimeBy(1_000)
```

---

## Hilt Test Module

```kotlin
// ✅ Replace data layer with fakes for UI tests
@Module
@TestInstallIn(
    components = [SingletonComponent::class],
    replaces = [UserRepositoryModule::class]
)
abstract class FakeUserRepositoryModule {
    @Binds @Singleton
    abstract fun bindUserRepository(fake: FakeUserRepository): UserRepository
}

// ✅ FakeUserRepository is injectable in tests
class FakeUserRepository @Inject constructor() : UserRepository {
    private val _users = MutableStateFlow<List<User>>(emptyList())

    override fun observeUsers(): Flow<List<User>> = _users

    fun emit(users: List<User>) { _users.value = users }
}
```

---

## Anti-Patterns

- Finding nodes by raw text that changes with localization — prefer `testTag`
- Testing implementation details like ViewModel state directly — test visible UI behavior
- Real network calls in UI tests — always fake the data layer
- Not wrapping composables in `AppTheme` in isolated tests — causes theme assertion failures
- Flaky `Thread.sleep()` for async waits — use `waitUntil` or `mainClock`

---

## Related Skills
- `compose-testing` — Compose-specific test APIs in depth
- `integration-testing` — testing layers below UI
- `unit-testing` — fast isolated tests for logic
- `hilt` — Hilt test setup
- `fake-data` — building consistent test data
