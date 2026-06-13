---
name: unit-testing
description: >
  Unit testing for Android with JUnit5, MockK, and Kotlin coroutines.
  Load this skill when writing unit tests for ViewModels, UseCases,
  Repositories, or any pure logic class, testing coroutines and flows,
  or structuring test classes and assertions.
---

# Unit Testing

## Overview
Unit tests verify a single class or function in isolation. Dependencies are replaced with fakes or mocks. In Android, unit tests run on the JVM (not a device), making them fast. ViewModels, UseCases, Repositories, and domain logic are the primary targets.

---

## Core Principles

- **Test behavior, not implementation** — test what a class does, not how
- **One assertion per test** — focused tests are easier to diagnose
- **AAA structure** — Arrange, Act, Assert
- **Fakes over mocks** — fakes are more maintainable; use mocks for verification
- **Test edge cases** — empty lists, null values, errors, boundary conditions
- **Fast and deterministic** — no network, no disk, no sleep()

---

## Dependencies

```kotlin
// build.gradle.kts (app module)
dependencies {
    testImplementation(libs.junit)
    testImplementation(libs.junit.params)           // parameterized tests
    testImplementation(libs.mockk)                  // mocking
    testImplementation(libs.kotlinx.coroutines.test)
    testImplementation(libs.turbine)                // Flow testing
    testImplementation(libs.truth)                  // Google Truth assertions
}
```

```toml
# libs.versions.toml
[versions]
junit = "5.10.2"
mockk = "1.13.10"
coroutines-test = "1.8.0"
turbine = "1.1.0"
truth = "1.4.2"

[libraries]
junit = { module = "org.junit.jupiter:junit-jupiter", version.ref = "junit" }
junit-params = { module = "org.junit.jupiter:junit-jupiter-params", version.ref = "junit" }
mockk = { module = "io.mockk:mockk", version.ref = "mockk" }
kotlinx-coroutines-test = { module = "org.jetbrains.kotlinx:kotlinx-coroutines-test", version.ref = "coroutines-test" }
turbine = { module = "app.cash.turbine:turbine", version.ref = "turbine" }
truth = { module = "com.google.truth:truth", version.ref = "truth" }
```

---

## Basic Test Structure

```kotlin
// ✅ AAA pattern with JUnit5
class GetUserUseCaseTest {

    // Fake instead of mock for repository
    private val repository = FakeUserRepository()
    private val useCase = GetUserUseCase(repository)

    @Test
    fun `returns user when found`() {
        // Arrange
        val expected = User(id = "1", name = "Ali", email = Email("ali@test.com"))
        repository.emit(listOf(expected))

        // Act
        val result = runBlocking { useCase("1") }

        // Assert
        assertThat(result.isSuccess).isTrue()
        assertThat(result.getOrNull()).isEqualTo(expected)
    }

    @Test
    fun `returns failure when user not found`() {
        // Arrange
        repository.emit(emptyList())

        // Act
        val result = runBlocking { useCase("nonexistent") }

        // Assert
        assertThat(result.isFailure).isTrue()
    }
}
```

---

## Testing ViewModels

```kotlin
// ✅ ViewModel test with TestCoroutineScheduler
class UserListViewModelTest {

    @get:Rule
    val mainDispatcherRule = MainDispatcherRule()  // replaces Main dispatcher

    private val getUsersUseCase = mockk<GetUsersUseCase>()
    private lateinit var viewModel: UserListViewModel

    @BeforeEach
    fun setup() {
        viewModel = UserListViewModel(getUsersUseCase)
    }

    @Test
    fun `state is Loading initially`() {
        // Initial state before any collection
        assertThat(viewModel.state.value).isEqualTo(UserListUiState.Loading)
    }

    @Test
    fun `state is Success when use case returns users`() = runTest {
        // Arrange
        val users = listOf(
            User(id = "1", name = "Ali", email = Email("ali@test.com"))
        )
        every { getUsersUseCase() } returns flowOf(users)

        // Re-create ViewModel after stubbing
        viewModel = UserListViewModel(getUsersUseCase)

        // Act + Assert
        viewModel.state.test {
            assertThat(awaitItem()).isEqualTo(UserListUiState.Loading)
            assertThat(awaitItem()).isEqualTo(UserListUiState.Success(users))
            cancelAndIgnoreRemainingEvents()
        }
    }

    @Test
    fun `state is Error when use case throws`() = runTest {
        // Arrange
        every { getUsersUseCase() } returns flow { throw Exception("Network error") }
        viewModel = UserListViewModel(getUsersUseCase)

        // Act + Assert
        viewModel.state.test {
            skipItems(1)  // skip Loading
            val error = awaitItem()
            assertThat(error).isInstanceOf(UserListUiState.Error::class.java)
            cancelAndIgnoreRemainingEvents()
        }
    }

    @Test
    fun `delete emits NavigateBack event on success`() = runTest {
        // Arrange
        val deleteUseCase = mockk<DeleteUserUseCase>()
        coEvery { deleteUseCase("1") } returns Result.success(Unit)
        viewModel = UserDetailViewModel(savedStateHandle, getUsersUseCase, deleteUseCase)

        // Act + Assert
        viewModel.events.test {
            viewModel.onDeleteClick()
            assertThat(awaitItem()).isEqualTo(UserDetailEvent.NavigateBack)
        }
    }
}
```

---

## MainDispatcherRule

```kotlin
// ✅ Replace Main dispatcher in tests (required for ViewModel tests)
class MainDispatcherRule(
    private val dispatcher: TestDispatcher = UnconfinedTestDispatcher()
) : TestWatcher() {

    override fun starting(description: Description) {
        Dispatchers.setMain(dispatcher)
    }

    override fun finished(description: Description) {
        Dispatchers.resetMain()
    }
}
```

---

## Testing Flows with Turbine

```kotlin
// ✅ Turbine — clean Flow assertion API
@Test
fun `observe users emits list updates`() = runTest {
    val repository = FakeUserRepository()
    val useCase = ObserveUsersUseCase(repository)

    useCase().test {
        // Initial empty state
        assertThat(awaitItem()).isEmpty()

        // Emit new data
        val user = User(id = "1", name = "Ali", email = Email("ali@test.com"))
        repository.emit(listOf(user))
        assertThat(awaitItem()).containsExactly(user)

        // Update existing
        val updated = user.copy(name = "Ali Updated")
        repository.emit(listOf(updated))
        assertThat(awaitItem()).containsExactly(updated)

        cancelAndIgnoreRemainingEvents()
    }
}
```

---

## Testing with MockK

```kotlin
// ✅ MockK for mocking Kotlin classes
class UserRepositoryImplTest {

    private val dao = mockk<UserDao>()
    private val api = mockk<UserApiService>()
    private val repository = UserRepositoryImpl(dao, api)

    @Test
    fun `getUser returns cached value when available`() = runTest {
        // Arrange
        val entity = UserEntity(id = "1", name = "Ali", email = "ali@test.com", role = "member")
        coEvery { dao.getById("1") } returns entity

        // Act
        val result = repository.getUser("1")

        // Assert
        assertThat(result.isSuccess).isTrue()
        coVerify(exactly = 0) { api.getUser(any()) }  // ✅ API not called
    }

    @Test
    fun `getUser fetches from API when not cached`() = runTest {
        // Arrange
        val dto = UserDto(id = "1", name = "Ali", email = "ali@test.com", role = "member")
        coEvery { dao.getById("1") } returns null
        coEvery { api.getUser("1") } returns dto
        coEvery { dao.insert(any()) } just Runs

        // Act
        val result = repository.getUser("1")

        // Assert
        assertThat(result.isSuccess).isTrue()
        coVerify(exactly = 1) { api.getUser("1") }
        coVerify(exactly = 1) { dao.insert(any()) }
    }
}
```

---

## Parameterized Tests

```kotlin
// ✅ Test multiple inputs with one test function
class EmailValidationTest {

    @ParameterizedTest
    @ValueSource(strings = ["ali@test.com", "user@domain.co.uk", "a@b.io"])
    fun `valid email addresses are accepted`(email: String) {
        assertThat(runCatching { Email(email) }.isSuccess).isTrue()
    }

    @ParameterizedTest
    @ValueSource(strings = ["notanemail", "@nodomain", "missing@", ""])
    fun `invalid email addresses throw`(email: String) {
        assertThat(runCatching { Email(email) }.isFailure).isTrue()
    }
}
```

---

## Anti-Patterns

- Testing implementation details — if refactoring breaks tests without changing behavior, tests are wrong
- `Thread.sleep()` in tests — use `runTest` and `TestDispatcher`
- Not resetting mocks between tests — use `@BeforeEach` or `clearAllMocks()`
- Testing private methods — test through the public API instead
- One giant test class — split by the class under test

---

## Related Skills
- `mocking` — MockK patterns and verification
- `fake-data` — building test data with factories
- `integration-testing` — testing multiple components together
- `coroutine` — coroutine dispatcher setup for tests
- `flow` — Flow testing with Turbine
