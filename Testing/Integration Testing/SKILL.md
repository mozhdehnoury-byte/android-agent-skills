---
name: integration-testing
description: >
  Integration testing for Android — testing multiple components together.
  Load this skill when testing Repository with a real Room database,
  testing ViewModel with real UseCases, verifying data flow across layers,
  or using Hilt for test dependency injection.
---

# Integration Testing

## Overview
Integration tests verify that multiple components work correctly together. Common targets: Repository + Room (real DB), ViewModel + UseCase + fake Repository, or full data layer with in-memory database. These run on a device or emulator (androidTest) or on JVM with Robolectric.

---

## Core Principles

- Integration tests sit **between unit and UI tests** — test real wiring, fake I/O
- Use **in-memory Room databases** — fast, no disk I/O, auto-cleaned
- Use **Hilt testing** — swap real implementations with fakes via `@TestInstallIn`
- Run on **JVM with Robolectric** where possible — faster than device
- Keep integration tests **focused** — one subsystem per test class

---

## Dependencies

```kotlin
// build.gradle.kts
dependencies {
    // Instrumented tests (androidTest)
    androidTestImplementation(libs.hilt.android.testing)
    kspAndroidTest(libs.hilt.compiler)
    androidTestImplementation(libs.androidx.test.runner)
    androidTestImplementation(libs.androidx.test.rules)
    androidTestImplementation(libs.kotlinx.coroutines.test)
    androidTestImplementation(libs.turbine)

    // JVM tests with Robolectric (test)
    testImplementation(libs.robolectric)
    testImplementation(libs.androidx.test.core)
}
```

---

## Repository + Room Integration Test

```kotlin
// ✅ Test Repository with real in-memory Room database
@RunWith(AndroidJUnit4::class)
class UserRepositoryIntegrationTest {

    private lateinit var database: AppDatabase
    private lateinit var dao: UserDao
    private lateinit var api: UserApiService
    private lateinit var repository: UserRepositoryImpl

    @Before
    fun setup() {
        val context = ApplicationProvider.getApplicationContext<Context>()
        database = Room.inMemoryDatabaseBuilder(context, AppDatabase::class.java)
            .allowMainThreadQueries()   // ✅ allowed in tests only
            .build()
        dao = database.userDao()
        api = mockk()
        repository = UserRepositoryImpl(dao, api)
    }

    @After
    fun teardown() {
        database.close()
    }

    @Test
    fun `getUser caches API response in local database`() = runTest {
        // Arrange
        val dto = UserDto(id = "1", name = "Ali", email = "ali@test.com", role = "member")
        coEvery { api.getUser("1") } returns dto

        // Act — first call fetches from API
        val result = repository.getUser("1")

        // Assert
        assertThat(result.isSuccess).isTrue()
        assertThat(result.getOrNull()?.name).isEqualTo("Ali")

        // Verify cached — second call should not hit API
        repository.getUser("1")
        coVerify(exactly = 1) { api.getUser("1") }

        // Verify in DB
        val cached = dao.getById("1")
        assertThat(cached).isNotNull()
        assertThat(cached?.name).isEqualTo("Ali")
    }

    @Test
    fun `observeUsers emits updates when database changes`() = runTest {
        // Arrange
        repository.observeUsers().test {
            assertThat(awaitItem()).isEmpty()

            // Act — insert directly into DB
            dao.insert(UserEntity(id = "1", name = "Ali", email = "ali@test.com", role = "member"))

            // Assert — Flow emits new value
            val users = awaitItem()
            assertThat(users).hasSize(1)
            assertThat(users[0].name).isEqualTo("Ali")

            cancelAndIgnoreRemainingEvents()
        }
    }
}
```

---

## Hilt Integration Test

```kotlin
// ✅ Replace real dependencies with fakes for specific tests

// 1. Define a fake module
@Module
@TestInstallIn(
    components = [SingletonComponent::class],
    replaces = [UserRepositoryModule::class]   // replaces real binding
)
abstract class FakeUserRepositoryModule {
    @Binds
    @Singleton
    abstract fun bindUserRepository(fake: FakeUserRepository): UserRepository
}

// 2. Inject fake into test
@HiltAndroidTest
class UserListViewModelIntegrationTest {

    @get:Rule(order = 0)
    val hiltRule = HiltAndroidRule(this)

    @get:Rule(order = 1)
    val mainDispatcherRule = MainDispatcherRule()

    @Inject lateinit var fakeRepository: FakeUserRepository
    @Inject lateinit var getUsersUseCase: GetUsersUseCase

    private lateinit var viewModel: UserListViewModel

    @Before
    fun setup() {
        hiltRule.inject()
        viewModel = UserListViewModel(getUsersUseCase)
    }

    @Test
    fun `ViewModel reflects repository data changes`() = runTest {
        viewModel.state.test {
            assertThat(awaitItem()).isEqualTo(UserListUiState.Loading)

            val users = listOf(User(id = "1", name = "Ali", email = Email("ali@test.com")))
            fakeRepository.emit(users)

            val success = awaitItem() as UserListUiState.Success
            assertThat(success.users).isEqualTo(users)

            cancelAndIgnoreRemainingEvents()
        }
    }
}
```

---

## Robolectric (JVM-based Android Tests)

```kotlin
// ✅ Run Android-dependent tests on JVM — faster than emulator
@RunWith(RobolectricTestRunner::class)
@Config(sdk = [34])
class UserRepositoryRobolectricTest {

    private lateinit var database: AppDatabase
    private lateinit var repository: UserRepositoryImpl

    @Before
    fun setup() {
        val context = ApplicationProvider.getApplicationContext<Context>()
        database = Room.inMemoryDatabaseBuilder(context, AppDatabase::class.java)
            .allowMainThreadQueries()
            .build()
        repository = UserRepositoryImpl(database.userDao(), mockk())
    }

    @After
    fun teardown() { database.close() }

    @Test
    fun `delete user removes from database`() = runTest {
        // Arrange
        database.userDao().insert(
            UserEntity(id = "1", name = "Ali", email = "ali@test.com", role = "member")
        )

        // Act
        repository.deleteUser("1")

        // Assert
        val result = database.userDao().getById("1")
        assertThat(result).isNull()
    }
}
```

---

## DAO Integration Test

```kotlin
// ✅ Test DAO queries with real Room in-memory database
@RunWith(AndroidJUnit4::class)
class UserDaoTest {

    private lateinit var database: AppDatabase
    private lateinit var dao: UserDao

    @Before
    fun setup() {
        val context = ApplicationProvider.getApplicationContext<Context>()
        database = Room.inMemoryDatabaseBuilder(context, AppDatabase::class.java)
            .allowMainThreadQueries()
            .build()
        dao = database.userDao()
    }

    @After
    fun teardown() { database.close() }

    @Test
    fun `insert and retrieve user`() = runTest {
        val entity = UserEntity(id = "1", name = "Ali", email = "ali@test.com", role = "member")
        dao.insert(entity)

        val retrieved = dao.getById("1")
        assertThat(retrieved).isEqualTo(entity)
    }

    @Test
    fun `observeAll emits empty list initially`() = runTest {
        dao.observeAll().test {
            assertThat(awaitItem()).isEmpty()
            cancelAndIgnoreRemainingEvents()
        }
    }

    @Test
    fun `delete removes entity`() = runTest {
        val entity = UserEntity(id = "1", name = "Ali", email = "ali@test.com", role = "member")
        dao.insert(entity)
        dao.deleteById("1")

        assertThat(dao.getById("1")).isNull()
    }
}
```

---

## Anti-Patterns

- Using real network in integration tests — mock the API layer; only test DB + logic integration
- Not closing the in-memory database in `@After` — causes resource leaks between tests
- `allowMainThreadQueries()` in production code — only acceptable in tests
- Testing too many layers at once — keep each integration test focused on one boundary
- Slow tests due to full Hilt component initialization — use `@UninstallModules` selectively

---

## Related Skills
- `unit-testing` — isolated unit tests for individual classes
- `ui-testing` — Compose UI tests on device/emulator
- `mocking` — MockK patterns used in integration tests
- `fake-data` — test data factories
- `hilt` — Hilt test setup with `@HiltAndroidTest`
- `room` — Room database under test
