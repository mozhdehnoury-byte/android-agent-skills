---
name: hilt
description: >
  Hilt dependency injection setup and patterns for Android.
  Load this skill when setting up Hilt, defining modules, injecting into
  Android components, scoping dependencies, or providing third-party libraries.
---

# Hilt

## Overview
Hilt is Android's recommended DI framework built on top of Dagger. It reduces Dagger boilerplate by providing standard components and scopes for Android classes. Hilt generates DI code at compile time — no reflection, no runtime overhead.

---

## Core Principles

- **One scope per lifetime** — match dependency scope to its actual lifetime
- Never inject into constructors of Android framework classes — use `@Inject` on fields or use Hilt entry points
- Prefer **constructor injection** over field injection everywhere possible
- Modules provide dependencies the constructor can't — third-party libs, interfaces, context-dependent types
- **`@Singleton`** is application-scoped — use sparingly, only for truly shared state

---

## Setup

```toml
[versions]
hilt = "2.51.1"

[libraries]
hilt-android         = { module = "com.google.dagger:hilt-android", version.ref = "hilt" }
hilt-compiler        = { module = "com.google.dagger:hilt-compiler", version.ref = "hilt" }
hilt-navigation-compose = { module = "androidx.hilt:hilt-navigation-compose", version = "1.2.0" }

[plugins]
hilt = { id = "com.google.dagger.hilt.android", version.ref = "hilt" }
```

```kotlin
// build.gradle.kts (root)
plugins {
    alias(libs.plugins.hilt) apply false
}

// build.gradle.kts (app)
plugins {
    alias(libs.plugins.hilt)
    alias(libs.plugins.ksp)
}

dependencies {
    implementation(libs.hilt.android)
    ksp(libs.hilt.compiler)
    implementation(libs.hilt.navigation.compose)
}
```

```kotlin
// ✅ Application class — required
@HiltAndroidApp
class MyApplication : Application()
```

---

## Constructor Injection

```kotlin
// ✅ Preferred — inject via constructor
class UserRepository @Inject constructor(
    private val userDao: UserDao,
    private val userApi: UserApi,
    private val mapper: UserMapper
) {
    // Hilt provides all dependencies automatically
}

class UserMapper @Inject constructor() {
    fun toDomain(entity: UserEntity): User = TODO()
}

// ✅ ViewModel — use @HiltViewModel
@HiltViewModel
class UserListViewModel @Inject constructor(
    private val repository: UserRepository,
    private val savedStateHandle: SavedStateHandle
) : ViewModel()
```

---

## Hilt Modules

```kotlin
// ✅ @Module + @InstallIn — provides dependencies Hilt can't construct automatically

// Singleton module — app-scoped dependencies
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {

    @Provides
    @Singleton
    fun provideOkHttpClient(): OkHttpClient =
        OkHttpClient.Builder()
            .connectTimeout(30, TimeUnit.SECONDS)
            .build()

    @Provides
    @Singleton
    fun provideRetrofit(okHttpClient: OkHttpClient): Retrofit =
        Retrofit.Builder()
            .baseUrl("https://api.example.com/")
            .client(okHttpClient)
            .addConverterFactory(json.asConverterFactory("application/json".toMediaType()))
            .build()

    @Provides
    @Singleton
    fun provideUserApi(retrofit: Retrofit): UserApi =
        retrofit.create(UserApi::class.java)
}

@Module
@InstallIn(SingletonComponent::class)
object DatabaseModule {

    @Provides
    @Singleton
    fun provideDatabase(@ApplicationContext context: Context): AppDatabase =
        Room.databaseBuilder(context, AppDatabase::class.java, "app_db")
            .addMigrations(MIGRATION_1_2)
            .build()

    @Provides
    fun provideUserDao(database: AppDatabase): UserDao = database.userDao()

    @Provides
    fun provideOrderDao(database: AppDatabase): OrderDao = database.orderDao()
}
```

---

## Binding Interfaces

```kotlin
// ✅ Bind interface to implementation
@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {

    @Binds
    @Singleton
    abstract fun bindUserRepository(
        impl: UserRepositoryImpl
    ): UserRepository

    @Binds
    abstract fun bindAnalytics(
        impl: FirebaseAnalyticsImpl
    ): Analytics
}

// ✅ @Binds is more efficient than @Provides for interface binding
// Use @Provides only when you need to call a constructor or factory
```

---

## Scopes

```kotlin
// ✅ Scope reference
@Singleton           // lives as long as Application
@ActivityRetainedScoped  // lives as long as ViewModel (survives rotation)
@ActivityScoped      // lives as long as Activity
@FragmentScoped      // lives as long as Fragment
@ViewScoped          // lives as long as View
@ServiceScoped       // lives as long as Service

// ✅ Example — screen-level scope
@Module
@InstallIn(ActivityRetainedComponent::class)
object CartModule {

    @Provides
    @ActivityRetainedScoped
    fun provideCart(): ShoppingCart = ShoppingCart()
}
```

---

## Qualifiers

```kotlin
// ✅ Distinguish multiple bindings of the same type
@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class AuthInterceptorOkHttpClient

@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class LoggingInterceptorOkHttpClient

@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {

    @Provides
    @Singleton
    @AuthInterceptorOkHttpClient
    fun provideAuthOkHttpClient(authInterceptor: AuthInterceptor): OkHttpClient =
        OkHttpClient.Builder()
            .addInterceptor(authInterceptor)
            .build()

    @Provides
    @Singleton
    @LoggingInterceptorOkHttpClient
    fun provideLoggingOkHttpClient(): OkHttpClient =
        OkHttpClient.Builder()
            .addInterceptor(HttpLoggingInterceptor())
            .build()
}

// ✅ Using qualifier at injection site
class UserRepository @Inject constructor(
    @AuthInterceptorOkHttpClient private val okHttpClient: OkHttpClient
)
```

---

## Injecting Into Android Components

```kotlin
// ✅ Activity
@AndroidEntryPoint
class MainActivity : ComponentActivity() {
    // Field injection (only when constructor injection is impossible)
    @Inject lateinit var analytics: Analytics
}

// ✅ Fragment
@AndroidEntryPoint
class UserFragment : Fragment() {
    private val viewModel: UserViewModel by viewModels()
}

// ✅ Compose — hiltViewModel()
@Composable
fun UserScreen(
    viewModel: UserViewModel = hiltViewModel()
) { ... }

// ✅ WorkManager
@HiltWorker
class SyncWorker @AssistedInject constructor(
    @Assisted context: Context,
    @Assisted params: WorkerParameters,
    private val syncEngine: SyncEngine
) : CoroutineWorker(context, params) {
    override suspend fun doWork(): Result = TODO()
}
```

---

## Entry Points (Non-Hilt Classes)

```kotlin
// ✅ Access Hilt graph from non-Hilt class (e.g., ContentProvider, custom View)
@EntryPoint
@InstallIn(SingletonComponent::class)
interface UserRepositoryEntryPoint {
    fun userRepository(): UserRepository
}

// Usage
val entryPoint = EntryPointAccessors.fromApplication(
    context.applicationContext,
    UserRepositoryEntryPoint::class.java
)
val repository = entryPoint.userRepository()
```

---

## Testing with Hilt

```kotlin
// ✅ Replace module in tests
@HiltAndroidTest
class UserRepositoryTest {

    @get:Rule
    val hiltRule = HiltAndroidRule(this)

    @Inject
    lateinit var repository: UserRepository

    @Before
    fun setUp() { hiltRule.inject() }

    @Test
    fun testGetUsers() { TODO() }
}

// ✅ Replace a binding for tests
@UninstallModules(NetworkModule::class)
@HiltAndroidTest
class FakeNetworkTest {

    @Module
    @InstallIn(SingletonComponent::class)
    object FakeNetworkModule {
        @Provides
        fun provideUserApi(): UserApi = FakeUserApi()
    }
}
```

---

## Anti-Patterns

- Field injection (`@Inject lateinit var`) in non-Android classes — use constructor injection
- `@Singleton` on everything — wastes memory, creates hidden shared state
- Putting business logic in Hilt modules — modules only provide/bind, no logic
- Injecting `Context` directly — use `@ApplicationContext` or `@ActivityContext`
- Using `ServiceLocator` pattern alongside Hilt — pick one
- Circular dependencies — redesign with interfaces or lazy injection

---

## Related Skills
- `dagger` — Dagger internals when Hilt isn't sufficient
- `koin` — alternative DI framework
- `annotation-processing` — KSP setup for Hilt code generation
- `viewmodel` — @HiltViewModel pattern
- `workmanager` — @HiltWorker setup
