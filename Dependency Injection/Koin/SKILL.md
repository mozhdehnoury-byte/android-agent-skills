---
name: koin
description: >
  Koin dependency injection setup and patterns for Android and KMP.
  Load this skill when working on a project that uses Koin, setting up
  modules, injecting into Android components, or using Koin in KMP projects.
---

# Koin

## Overview
Koin is a lightweight DI framework for Kotlin. Unlike Dagger/Hilt, Koin is a service locator at runtime — no code generation, no annotation processing. It's simpler to set up and KMP-compatible. The tradeoff is runtime resolution errors instead of compile-time errors.

---

## Core Principles

- Koin resolves dependencies at **runtime** — errors surface at app start, not compile time
- Prefer Hilt for pure Android projects — Koin shines in **KMP** where Hilt isn't available
- Define dependencies in **modules** — one module per feature or layer
- Use `single` for singletons, `factory` for new instances, `viewModel` for ViewModels
- Always run `checkModules()` in tests — catches missing bindings before production

---

## Setup

```toml
[versions]
koin = "3.5.6"
koin-compose = "3.5.6"

[libraries]
koin-android        = { module = "io.insert-koin:koin-android", version.ref = "koin" }
koin-compose        = { module = "io.insert-koin:koin-androidx-compose", version.ref = "koin-compose" }
koin-core           = { module = "io.insert-koin:koin-core", version.ref = "koin" }  # KMP
koin-test           = { module = "io.insert-koin:koin-test", version.ref = "koin" }
```

```kotlin
// build.gradle.kts
dependencies {
    implementation(libs.koin.android)
    implementation(libs.koin.compose)
    testImplementation(libs.koin.test)
}
```

---

## Koin Application Setup

```kotlin
// ✅ Initialize in Application
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        startKoin {
            androidLogger(Level.DEBUG)
            androidContext(this@MyApplication)
            modules(
                networkModule,
                databaseModule,
                repositoryModule,
                viewModelModule
            )
        }
    }
}
```

---

## Defining Modules

```kotlin
// ✅ Network module
val networkModule = module {

    single {
        OkHttpClient.Builder()
            .connectTimeout(30, TimeUnit.SECONDS)
            .build()
    }

    single {
        Retrofit.Builder()
            .baseUrl("https://api.example.com/")
            .client(get())  // get() resolves OkHttpClient
            .addConverterFactory(Json.asConverterFactory("application/json".toMediaType()))
            .build()
    }

    single<UserApi> { get<Retrofit>().create(UserApi::class.java) }
}

// ✅ Database module
val databaseModule = module {

    single {
        Room.databaseBuilder(androidContext(), AppDatabase::class.java, "app_db")
            .addMigrations(MIGRATION_1_2)
            .build()
    }

    single { get<AppDatabase>().userDao() }
    single { get<AppDatabase>().orderDao() }
}

// ✅ Repository module
val repositoryModule = module {
    single<UserRepository> { UserRepositoryImpl(get(), get(), get()) }
    single { UserMapper() }
}

// ✅ ViewModel module
val viewModelModule = module {
    viewModel { UserListViewModel(get()) }
    viewModel { parameters -> UserDetailViewModel(get(), parameters.get()) }
}
```

---

## Scopes

```kotlin
// single     — one instance for the entire app lifetime
// factory    — new instance every time it's requested
// scoped     — one instance per defined scope lifetime

val sessionModule = module {

    // ✅ Scoped to a session
    scope<UserSession> {
        scoped { CartRepository(get()) }
        scoped { CheckoutViewModel(get()) }
    }
}

// ✅ Create and close scope
val scope = getKoin().createScope("session_1", named<UserSession>())
val cart = scope.get<CartRepository>()
scope.close()  // releases scoped instances
```

---

## Injection in Android Components

```kotlin
// ✅ Activity / Fragment — by inject()
class UserFragment : Fragment() {
    private val viewModel: UserListViewModel by viewModel()
    private val repository: UserRepository by inject()
}

// ✅ Compose — koinViewModel()
@Composable
fun UserScreen(
    viewModel: UserListViewModel = koinViewModel()
) { ... }

// ✅ ViewModel with parameters
@Composable
fun UserDetailScreen(userId: String) {
    val viewModel: UserDetailViewModel = koinViewModel(
        parameters = { parametersOf(userId) }
    )
}
```

---

## Qualifiers

```kotlin
// ✅ Named bindings — distinguish same type
val networkModule = module {
    single(named("auth")) {
        OkHttpClient.Builder()
            .addInterceptor(get<AuthInterceptor>())
            .build()
    }
    single(named("logging")) {
        OkHttpClient.Builder()
            .addInterceptor(HttpLoggingInterceptor())
            .build()
    }
}

// ✅ Inject with qualifier
class UserRepository(
    private val client: OkHttpClient = get(named("auth"))
)
```

---

## KMP Setup (commonMain)

```kotlin
// ✅ Koin works in commonMain — no Android dependency
// commonMain
val sharedModule = module {
    single { UserRepository(get(), get()) }
    single { UserMapper() }
    single { HttpClient { /* Ktor config */ } }
}

// androidMain
val androidModule = module {
    single { AndroidLogger() }
    viewModel { UserListViewModel(get()) }
}

// iosMain — use Koin in Swift via KMP
// val iosModule = module { ... }
```

---

## Testing

```kotlin
// ✅ checkModules — verify all bindings resolve
class KoinModuleTest : KoinTest {

    @Test
    fun verifyKoinModules() {
        checkModules {
            modules(networkModule, databaseModule, repositoryModule, viewModelModule)
        }
    }
}

// ✅ Override bindings in tests
@Test
fun testWithFakeDependency() {
    startKoin {
        modules(
            module {
                single<UserRepository> { FakeUserRepository() }
                viewModel { UserListViewModel(get()) }
            }
        )
    }
    // test...
    stopKoin()
}
```

---

## Koin vs Hilt

| | Koin | Hilt |
|--|------|------|
| Error detection | Runtime | Compile-time |
| Setup complexity | Low | Medium |
| KMP support | ✅ Yes | ❌ No |
| Code generation | ❌ No | ✅ Yes |
| Android integration | Good | Excellent |
| Recommended for | KMP, simple apps | Android-only |

---

## Anti-Patterns

- Not calling `checkModules()` in tests — runtime errors reach production
- Using `inject()` (lazy) when `get()` (eager) is needed for initialization order
- Defining all bindings in one giant module — split by feature/layer
- Using `single` for everything — `factory` for stateless dependencies
- Not closing scopes — memory leaks for scoped dependencies

---

## Related Skills
- `hilt` — preferred DI for Android-only projects
- `dagger` — compile-time DI underlying Hilt
- `kmp` — Koin in multiplatform projects
- `viewmodel` — ViewModel injection with Koin
