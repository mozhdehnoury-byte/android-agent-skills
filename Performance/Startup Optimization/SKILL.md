---
name: startup-optimization
description: >
  App startup time optimization for Android.
  Load this skill when reducing cold/warm start time, using App Startup
  library, deferring heavy initialization, measuring startup performance,
  or eliminating unnecessary work during application launch.
---

# Startup Optimization

## Overview
App startup time directly impacts user retention. Cold start (process not in memory) is the most expensive. The goal is to do the minimum work needed before the first frame is drawn, and defer everything else. Tools include the App Startup library, baseline profiles, and Macrobenchmark.

---

## Core Principles

- Do **minimal work** in `Application.onCreate()` — defer everything non-essential
- Use **App Startup library** to control and sequence initializer order
- Never run **network calls or disk I/O** synchronously during startup
- Use **Baseline Profiles** to pre-compile critical code paths
- Measure with **Macrobenchmark** — not manual timing

---

## What Happens During Cold Start

```
1. Process creation
2. Application.onCreate()       ← most startup time spent here
3. Activity.onCreate()
4. Layout inflation / Compose composition
5. First frame drawn             ← TTFD (Time To First Display)
```

---

## App Startup Library

```toml
# libs.versions.toml
[libraries]
androidx-startup = { module = "androidx.startup:startup-runtime", version = "1.1.1" }
```

```kotlin
// ✅ Replace manual SDK init in Application.onCreate with Initializers
class TimberInitializer : Initializer<Unit> {
    override fun create(context: Context) {
        if (BuildConfig.DEBUG) Timber.plant(Timber.DebugTree())
    }
    override fun dependencies(): List<Class<out Initializer<*>>> = emptyList()
}

class AnalyticsInitializer : Initializer<Unit> {
    override fun create(context: Context) {
        FirebaseApp.initializeApp(context)
    }
    override fun dependencies(): List<Class<out Initializer<*>>> = emptyList()
}
```

```xml
<!-- AndroidManifest.xml — register initializers -->
<provider
    android:name="androidx.startup.InitializationProvider"
    android:authorities="${applicationId}.androidx-startup"
    android:exported="false">
    <meta-data
        android:name="com.example.TimberInitializer"
        android:value="androidx.startup" />
    <meta-data
        android:name="com.example.AnalyticsInitializer"
        android:value="androidx.startup" />
</provider>
```

---

## Deferring Non-Critical Work

```kotlin
// ✅ Defer non-critical init to after first frame
class App : Application() {
    override fun onCreate() {
        super.onCreate()
        // Only critical init here
        initCrashReporting()

        // Defer everything else
        ProcessLifecycleOwner.get().lifecycle.addObserver(
            object : DefaultLifecycleObserver {
                override fun onStart(owner: LifecycleOwner) {
                    owner.lifecycle.removeObserver(this)
                    initNonCriticalSdks()
                }
            }
        )
    }

    private fun initCrashReporting() {
        FirebaseCrashlytics.getInstance().setCrashlyticsCollectionEnabled(!BuildConfig.DEBUG)
    }

    private fun initNonCriticalSdks() {
        // Analytics, feature flags, etc.
    }
}
```

---

## Lazy Dependency Initialization with Hilt

```kotlin
// ✅ Use @Singleton with lazy injection — initialized on first use
@HiltViewModel
class HomeViewModel @Inject constructor(
    private val analyticsRepository: AnalyticsRepository  // injected lazily
) : ViewModel()

// ✅ Lazy provider for expensive dependencies
@Provides
@Singleton
fun provideExpensiveSdk(
    @ApplicationContext context: Context
): Lazy<ExpensiveSdk> = lazy { ExpensiveSdk.init(context) }
```

---

## Splash Screen API

```kotlin
// ✅ Use SplashScreen API — replaces custom splash Activity
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        val splashScreen = installSplashScreen()

        // Keep splash visible while loading initial data
        splashScreen.setKeepOnScreenCondition {
            !viewModel.isReady.value
        }

        super.onCreate(savedInstanceState)
        setContent { AppContent() }
    }
}
```

```xml
<!-- res/values/themes.xml -->
<style name="Theme.App.Starting" parent="Theme.SplashScreen">
    <item name="windowSplashScreenBackground">@color/primary</item>
    <item name="windowSplashScreenAnimatedIcon">@drawable/ic_logo</item>
    <item name="postSplashScreenTheme">@style/Theme.App</item>
</style>
```

---

## Measuring Startup

```kotlin
// ✅ Macrobenchmark for startup measurement
@RunWith(AndroidJUnit4::class)
class StartupBenchmark {
    @get:Rule
    val benchmarkRule = MacrobenchmarkRule()

    @Test
    fun coldStartup() = benchmarkRule.measureRepeated(
        packageName = "com.example.app",
        metrics = listOf(StartupTimingMetric()),
        iterations = 5,
        startupMode = StartupMode.COLD
    ) {
        pressHome()
        startActivityAndWait()
    }
}
```

---

## StrictMode for Development

```kotlin
// ✅ Enable StrictMode in debug to catch startup violations
if (BuildConfig.DEBUG) {
    StrictMode.setThreadPolicy(
        StrictMode.ThreadPolicy.Builder()
            .detectDiskReads()
            .detectDiskWrites()
            .detectNetwork()
            .penaltyLog()
            .build()
    )
}
```

---

## Anti-Patterns

- Running SharedPreferences reads synchronously in `Application.onCreate()`
- Initializing all SDKs eagerly — most can be deferred or lazy-initialized
- Custom splash Activity — use the SplashScreen API instead
- Not measuring startup before and after optimizations — can't prove improvement
- Doing Room database creation synchronously on the main thread at startup

---

## Related Skills
- `baseline-profile` — pre-compiling critical code paths
- `app-startup` — App Startup library details
- `macrobenchmark` — measuring startup and runtime performance
- `hilt` — lazy dependency injection at startup
