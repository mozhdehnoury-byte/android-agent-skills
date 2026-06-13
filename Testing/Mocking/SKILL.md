---
name: mocking
description: >
  Mocking with MockK for Android unit tests.
  Load this skill when creating mocks, stubs, and spies with MockK,
  stubbing suspend functions and Flows, verifying interactions,
  using argument captors, or deciding between mocks and fakes.
---

# Mocking

## Overview
MockK is the idiomatic mocking library for Kotlin. It supports suspend functions, coroutines, extension functions, and object mocking — areas where Mockito struggles with Kotlin code. Use mocks to isolate the class under test from its dependencies and verify interactions.

---

## Core Principles

- **Fakes over mocks** for repositories and data sources — fakes are more maintainable
- **Mocks** for verifying interactions — "was this method called with these arguments?"
- **Stubs** for providing return values — "when called, return this"
- **Never mock what you don't own** — mock your own interfaces, not third-party classes
- `coEvery` / `coVerify` for suspend functions — never use `every`/`verify` for coroutines

---

## Creating Mocks

```kotlin
// ✅ Create mock
val repository = mockk<UserRepository>()

// ✅ Relaxed mock — returns default values without explicit stubs
val repository = mockk<UserRepository>(relaxed = true)

// ✅ Relaxed Unit mock — only relaxes Unit-returning functions
val repository = mockk<UserRepository>(relaxUnitFun = true)

// ✅ Spy — wraps real object, overrides only stubbed methods
val realRepository = UserRepositoryImpl(dao, api)
val spy = spyk(realRepository)
```

---

## Stubbing

```kotlin
// ✅ Stub regular function
every { repository.getCount() } returns 5
every { repository.getCount() } returns 5 andThen 10   // different on subsequent calls
every { repository.getCount() } throws RuntimeException("error")

// ✅ Stub suspend function
coEvery { repository.getUser("1") } returns Result.success(user)
coEvery { repository.getUser("1") } throws IOException("Network error")
coEvery { repository.getUser(any()) } returns Result.failure(Exception("Not found"))

// ✅ Stub Flow
every { repository.observeUsers() } returns flowOf(listOf(user))
every { repository.observeUsers() } returns flow {
    emit(emptyList())
    delay(100)
    emit(listOf(user))
}

// ✅ Stub with argument matching
coEvery { repository.getUser(match { it.startsWith("user_") }) } returns Result.success(user)
coEvery { repository.getUser(any()) } returns Result.failure(Exception())

// ✅ Stub with answer (access arguments)
coEvery { repository.getUser(any()) } answers {
    val id = firstArg<String>()
    Result.success(User(id = id, name = "User $id", email = Email("$id@test.com")))
}

// ✅ Unit-returning suspend — use coJustRun
coJustRun { repository.deleteUser(any()) }

// ✅ Non-suspend Unit — use justRun
justRun { analytics.log(any()) }
```

---

## Verification

```kotlin
// ✅ Verify call was made
verify { repository.getCount() }
coVerify { repository.getUser("1") }

// ✅ Verify call count
verify(exactly = 1) { repository.getCount() }
verify(exactly = 0) { repository.getCount() }   // never called
verify(atLeast = 1) { repository.getCount() }
verify(atMost = 2)  { repository.getCount() }

// ✅ Verify with argument matchers
coVerify { repository.getUser(eq("1")) }
coVerify { repository.getUser(any()) }
coVerify { repository.getUser(match { it.isNotEmpty() }) }

// ✅ Verify order
verifyOrder {
    repository.getCount()
    repository.getUser("1")
}

// ✅ Verify sequence (exactly in order, no other calls)
verifySequence {
    repository.getCount()
    repository.getUser("1")
}

// ✅ Verify no interactions
verify { repository wasNot Called }
confirmVerified(repository)   // all interactions accounted for
```

---

## Argument Captors

```kotlin
// ✅ Capture arguments for complex assertions
val userSlot = slot<User>()
coEvery { repository.createUser(capture(userSlot)) } returns Result.success(user)

viewModel.onCreateUser("Ali", "ali@test.com")

assertThat(userSlot.captured.name).isEqualTo("Ali")
assertThat(userSlot.captured.email.value).isEqualTo("ali@test.com")

// ✅ Capture multiple calls
val capturedIds = mutableListOf<String>()
coEvery { repository.getUser(captureMany = capturedIds) } returns Result.success(user)
```

---

## Mocking Objects and Companions

```kotlin
// ✅ Mock Kotlin object
mockkObject(DateUtils)
every { DateUtils.now() } returns Instant.parse("2024-01-01T00:00:00Z")
// ... test ...
unmockkObject(DateUtils)

// ✅ Mock companion object
mockkObject(User.Companion)
every { User.Companion.create(any()) } returns mockUser

// ✅ Mock top-level functions
mockkStatic("com.example.app.utils.ExtensionsKt")
every { anyString().toFormattedDate() } returns "Jan 1, 2024"
```

---

## Clearing Mocks

```kotlin
// ✅ Clear recorded calls (keep stubs)
clearMocks(repository, answers = false)

// ✅ Clear stubs and calls
clearMocks(repository)

// ✅ Clear all mocks in a test class
@AfterEach
fun teardown() {
    clearAllMocks()
}

// ✅ Unmock objects
@AfterEach
fun teardown() {
    unmockkAll()
}
```

---

## Mocks vs Fakes Decision Guide

| Scenario | Use |
|---|---|
| Repository with simple CRUD | Fake |
| Repository — verify specific API calls | Mock |
| External service (Analytics, Firebase) | Mock |
| Database layer in integration test | Real (in-memory Room) |
| Verifying a method was called N times | Mock |
| Controlling returned data in multiple tests | Fake |

```kotlin
// ✅ Fake — simpler, no verify needed
class FakeUserRepository : UserRepository {
    private val _users = MutableStateFlow<List<User>>(emptyList())
    var shouldFail = false

    override fun observeUsers() = _users.asStateFlow()
    override suspend fun getUser(id: String) =
        if (shouldFail) Result.failure(Exception("error"))
        else _users.value.find { it.id == id }
            ?.let { Result.success(it) }
            ?: Result.failure(Exception("not found"))

    fun emit(users: List<User>) { _users.value = users }
}
```

---

## Anti-Patterns

- `every` for suspend functions — use `coEvery`; `every` will not work correctly
- `verify` for suspend functions — use `coVerify`
- Mocking data classes — just create them directly
- Over-verifying — only verify interactions that matter for the behavior being tested
- Not clearing mocks between tests — stale stubs cause test pollution

---

## Related Skills
- `unit-testing` — full unit test structure using MockK
- `fake-data` — building fake implementations and test data
- `integration-testing` — when to use fakes vs mocks
