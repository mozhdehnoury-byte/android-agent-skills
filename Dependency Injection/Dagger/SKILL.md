---
name: dagger
description: >
  Dagger 2 dependency injection for Android without Hilt.
  Load this skill when working on a project that uses raw Dagger (not Hilt),
  understanding Hilt's internals, building custom components/subcomponents,
  or migrating from Dagger to Hilt.
---

# Dagger 2

## Overview
Dagger 2 is a compile-time DI framework that generates Java code for dependency injection. Hilt is built on top of Dagger and covers most Android use cases. Use raw Dagger when you need custom component hierarchies, non-standard scopes, or are working on a project that predates Hilt.

---

## Core Principles

- Dagger generates all DI code at **compile time** — no reflection
- **Components** are the bridge between the object graph and the consumer
- **Modules** provide dependencies the constructor can't supply
- **Subcomponents** inherit the parent's bindings and add their own scope
- Prefer Hilt for new Android projects — raw Dagger only when needed

---

## Basic Setup

```toml
[versions]
dagger = "2.51.1"

[libraries]
dagger         = { module = "com.google.dagger:dagger", version.ref = "dagger" }
dagger-compiler = { module = "com.google.dagger:dagger-compiler", version.ref = "dagger" }
```

```kotlin
dependencies {
    implementation(libs.dagger)
    ksp(libs.dagger.compiler)
}
```

---

## Component

```kotlin
// ✅ Application-level component
@Singleton
@Component(modules = [
    NetworkModule::class,
    DatabaseModule::class,
    RepositoryModule::class
])
interface AppComponent {

    // Expose for injection into Application
    fun inject(app: MyApplication)

    // Factory — preferred over Builder for constructor params
    @Component.Factory
    interface Factory {
        fun create(@BindsInstance @ApplicationContext context: Context): AppComponent
    }
}

// ✅ Initialize in Application
class MyApplication : Application() {
    val appComponent: AppComponent by lazy {
        DaggerAppComponent.factory().create(this)
    }

    override fun onCreate() {
        super.onCreate()
        appComponent.inject(this)
    }
}
```

---

## Modules

```kotlin
// ✅ @Module — provides dependencies
@Module
object NetworkModule {

    @Provides
    @Singleton
    fun provideOkHttpClient(): OkHttpClient =
        OkHttpClient.Builder().build()

    @Provides
    @Singleton
    fun provideRetrofit(client: OkHttpClient): Retrofit =
        Retrofit.Builder()
            .baseUrl("https://api.example.com/")
            .client(client)
            .build()

    @Provides
    @Singleton
    fun provideUserApi(retrofit: Retrofit): UserApi =
        retrofit.create(UserApi::class.java)
}

// ✅ @Binds — for interface → implementation binding (abstract module)
@Module
abstract class RepositoryModule {

    @Binds
    @Singleton
    abstract fun bindUserRepository(impl: UserRepositoryImpl): UserRepository
}
```

---

## Scopes

```kotlin
// ✅ Define custom scopes
@Scope
@Retention(AnnotationRetention.RUNTIME)
annotation class ActivityScope

@Scope
@Retention(AnnotationRetention.RUNTIME)
annotation class FragmentScope

// ✅ Use on component and provided dependency
@ActivityScope
@Subcomponent(modules = [ActivityModule::class])
interface ActivityComponent {
    fun inject(activity: MainActivity)

    @Subcomponent.Factory
    interface Factory {
        fun create(): ActivityComponent
    }
}
```

---

## Subcomponents

```kotlin
// ✅ Subcomponent inherits parent bindings, adds its own scope
@ActivityScope
@Subcomponent(modules = [ActivityModule::class])
interface ActivityComponent {
    fun inject(activity: MainActivity)

    @Subcomponent.Factory
    interface Factory {
        fun create(): ActivityComponent
    }
}

// ✅ Register subcomponent in parent
@Module(subcomponents = [ActivityComponent::class])
object AppSubcomponents

// ✅ Create subcomponent from parent
class MainActivity : AppCompatActivity() {
    private lateinit var activityComponent: ActivityComponent

    override fun onCreate(savedInstanceState: Bundle?) {
        activityComponent = (application as MyApplication)
            .appComponent
            .activityComponentFactory()
            .create()
        activityComponent.inject(this)
        super.onCreate(savedInstanceState)
    }
}
```

---

## Constructor Injection

```kotlin
// ✅ Same as Hilt — @Inject on constructor
class UserRepository @Inject constructor(
    private val userDao: UserDao,
    private val userApi: UserApi
)

class UserMapper @Inject constructor()
```

---

## Qualifiers

```kotlin
// ✅ Distinguish multiple bindings of the same type
@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class BaseUrl

@Module
object NetworkModule {

    @Provides
    @Singleton
    @BaseUrl
    fun provideBaseUrl(): String = "https://api.example.com/"

    @Provides
    @Singleton
    fun provideRetrofit(@BaseUrl baseUrl: String): Retrofit =
        Retrofit.Builder().baseUrl(baseUrl).build()
}
```

---

## Lazy and Provider Injection

```kotlin
// ✅ Lazy<T> — defer initialization until first use
class HeavyFeature @Inject constructor(
    private val heavyDep: Lazy<HeavyDependency>
) {
    fun use() {
        heavyDep.get().doWork()  // initialized on first call
    }
}

// ✅ Provider<T> — get a new instance each time
class RequestFactory @Inject constructor(
    private val requestProvider: Provider<NetworkRequest>
) {
    fun createRequest(): NetworkRequest = requestProvider.get()
}
```

---

## Migration to Hilt

```kotlin
// Hilt replaces most Dagger boilerplate:

// Dagger → Hilt equivalents
@Component(modules = [...])
interface AppComponent              →  @HiltAndroidApp on Application

@ActivityScope @Subcomponent(...)
interface ActivityComponent         →  @AndroidEntryPoint on Activity

appComponent.inject(this)           →  Removed (automatic)

@Component.Factory / @Component.Builder →  Removed (Hilt manages)

// Keep raw Dagger when:
// - Custom component hierarchy beyond what Hilt provides
// - Non-Android modules (pure Kotlin/Java libraries)
// - Existing codebase that's too large to migrate at once
```

---

## Anti-Patterns

- Field injection in non-Android classes — use constructor injection
- Creating components outside of Application — breaks the object graph
- Providing mutable singletons — hidden shared state causes bugs
- Deep subcomponent hierarchies — prefer Hilt's flat component model
- Not using `@Binds` for interface bindings — `@Provides` works but is less efficient

---

## Related Skills
- `hilt` — Hilt (recommended for Android)
- `koin` — alternative runtime DI
- `annotation-processing` — KSP for Dagger code generation
