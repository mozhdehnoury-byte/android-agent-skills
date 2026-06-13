---
name: fake-data
description: >
  Fake data and test factory patterns for Android tests.
  Load this skill when building fake repositories, creating test data
  with factory functions, designing object mothers, or maintaining
  consistent test data across a test suite.
---

# Fake Data

## Overview
Fake data refers to two related concerns: fake implementations of interfaces (replacing real dependencies in tests) and test data factories (creating domain/entity objects with sensible defaults). Together they make tests readable, maintainable, and independent of production data.

---

## Core Principles

- **Fakes are real implementations** — they implement the same interface, just in memory
- **Test data factories** provide defaults — only specify what's relevant to the test
- **Centralize test data** — one place to update when models change
- **Named factory variants** for common test scenarios (e.g., `adminUser()`, `suspendedUser()`)
- **Deterministic** — fake data never uses random values unless explicitly testing randomness

---

## Fake Repository

```kotlin
// ✅ Fake repository — in-memory implementation of the real interface
class FakeUserRepository @Inject constructor() : UserRepository {

    private val _users = MutableStateFlow<List<User>>(emptyList())
    var shouldFail = false
    var failureMessage = "Fake failure"

    // ── Reactive ───────────────────────────────────────────────
    override fun observeUsers(): Flow<List<User>> = _users.asStateFlow()

    override fun observeUser(id: String): Flow<User?> =
        _users.map { users -> users.find { it.id == id } }

    // ── Suspend ────────────────────────────────────────────────
    override suspend fun getUser(id: String): Result<User> {
        if (shouldFail) return Result.failure(Exception(failureMessage))
        return _users.value.find { it.id == id }
            ?.let { Result.success(it) }
            ?: Result.failure(Exception("User not found: $id"))
    }

    override suspend fun getUsers(): Result<List<User>> {
        if (shouldFail) return Result.failure(Exception(failureMessage))
        return Result.success(_users.value)
    }

    override suspend fun createUser(user: User): Result<User> {
        if (shouldFail) return Result.failure(Exception(failureMessage))
        _users.update { it + user }
        return Result.success(user)
    }

    override suspend fun updateUser(user: User): Result<Unit> {
        if (shouldFail) return Result.failure(Exception(failureMessage))
        _users.update { users -> users.map { if (it.id == user.id) user else it } }
        return Result.success(Unit)
    }

    override suspend fun deleteUser(id: String): Result<Unit> {
        if (shouldFail) return Result.failure(Exception(failureMessage))
        _users.update { users -> users.filter { it.id != id } }
        return Result.success(Unit)
    }

    // ── Test helpers ───────────────────────────────────────────
    fun emit(users: List<User>) { _users.value = users }
    fun emitOne(user: User) { _users.value = listOf(user) }
    fun clear() { _users.value = emptyList() }
    fun addUser(user: User) { _users.update { it + user } }
}
```

---

## Domain Model Factory

```kotlin
// ✅ Factory object with defaults — only override what matters
object UserFactory {

    fun create(
        id: String = "user-1",
        name: String = "Ali Rezaei",
        email: String = "ali@test.com",
        role: UserRole = UserRole.MEMBER,
        status: UserStatus = UserStatus.ACTIVE,
        createdAt: Instant = Instant.parse("2024-01-01T00:00:00Z")
    ): User = User(
        id = id,
        name = name,
        email = Email(email),
        role = role,
        status = status,
        createdAt = createdAt
    )

    // ✅ Named variants for common scenarios
    fun admin() = create(id = "admin-1", name = "Admin User", role = UserRole.ADMIN)
    fun guest() = create(id = "guest-1", name = "Guest User", role = UserRole.GUEST)
    fun suspended() = create(id = "suspended-1", status = UserStatus.SUSPENDED)

    // ✅ List factory
    fun list(count: Int = 3): List<User> = (1..count).map {
        create(id = "user-$it", name = "User $it", email = "user$it@test.com")
    }
}

// ✅ Usage in tests — only specify what matters
val user = UserFactory.create(name = "Custom Name")
val admin = UserFactory.admin()
val users = UserFactory.list(5)
```

---

## Entity Factory (for Room Tests)

```kotlin
// ✅ Separate factory for DB entities
object UserEntityFactory {

    fun create(
        id: String = "user-1",
        name: String = "Ali Rezaei",
        email: String = "ali@test.com",
        role: String = "member",
        isActive: Boolean = true,
        createdAt: Long = 1704067200000L,
        updatedAt: Long = 1704067200000L
    ): UserEntity = UserEntity(
        id = id, name = name, email = email,
        role = role, isActive = isActive,
        createdAt = createdAt, updatedAt = updatedAt
    )

    fun list(count: Int = 3): List<UserEntity> = (1..count).map {
        create(id = "user-$it", name = "User $it", email = "user$it@test.com")
    }
}
```

---

## DTO Factory (for API Tests)

```kotlin
// ✅ Factory for API response DTOs
object UserDtoFactory {

    fun create(
        id: String = "user-1",
        name: String = "Ali Rezaei",
        email: String = "ali@test.com",
        role: String = "member"
    ): UserDto = UserDto(id = id, name = name, email = email, role = role)

    fun list(count: Int = 3): List<UserDto> = (1..count).map {
        create(id = "user-$it", name = "User $it", email = "user$it@test.com")
    }
}
```

---

## UiState Factory

```kotlin
// ✅ Factories for UI state — useful for Compose Preview and tests
object UserListUiStateFactory {

    fun loading() = UserListUiState.Loading

    fun success(
        users: List<User> = UserFactory.list()
    ) = UserListUiState.Success(users)

    fun error(
        message: String = "Something went wrong"
    ) = UserListUiState.Error(message)

    fun empty() = UserListUiState.Success(emptyList())
}
```

---

## Fake Dispatcher

```kotlin
// ✅ Fake dispatcher for controlling coroutine execution in tests
class FakeDispatcherProvider : DispatcherProvider {
    override val main: CoroutineDispatcher = UnconfinedTestDispatcher()
    override val io: CoroutineDispatcher = UnconfinedTestDispatcher()
    override val default: CoroutineDispatcher = UnconfinedTestDispatcher()
}
```

---

## Test Fixtures in androidTest

```kotlin
// ✅ Shared test fixtures in :core:testing module
// core/testing/src/main/kotlin/com/example/testing/

// Factories
UserFactory.kt
OrderFactory.kt
ProductFactory.kt

// Fakes
FakeUserRepository.kt
FakeOrderRepository.kt
FakeNetworkMonitor.kt

// Rules
MainDispatcherRule.kt
```

---

## Anti-Patterns

- Hardcoding IDs like `"1"` scattered across all tests — use factory defaults
- Creating full objects in every test — use factories with defaults, override only what matters
- Mutable shared state in fakes — each test should reset; use `@BeforeEach` or `clear()`
- Fake that diverges from real implementation — a fake that ignores method contracts is useless
- Using `Random.nextInt()` in factories — non-deterministic; use fixed values

---

## Related Skills
- `unit-testing` — factories used in unit tests
- `integration-testing` — fakes used in integration tests
- `mocking` — when to use fakes vs MockK mocks
- `domain-modeling` — domain models the factories create
