---
name: configuration
description: >
  Android Configuration changes handling — screen rotation, locale change,
  dark mode, font scale, and other runtime configuration events.
  Load this skill when handling configuration changes, deciding what to retain
  across rotation, or supporting dynamic locale/theme switching.
---

# Configuration

## Overview

Android Configuration represents the current state of the device — screen orientation, locale, font scale, night mode, and more. When configuration changes, Android destroys and recreates the Activity by default. Proper handling ensures seamless UX without data loss or memory leaks.

---

## Core Principles

- Never store UI state in Activity — use ViewModel to survive configuration changes
- Never use `android:configChanges` to avoid recreating Activity unless absolutely necessary
- Handle configuration-dependent resources via the resource qualifier system — not in code
- Use `SavedStateHandle` for state that must survive both configuration change and process death
- Test rotation and locale change explicitly — they expose the most lifecycle bugs

---

## What Happens on Configuration Change

```
User rotates screen
    → Activity.onPause()
    → Activity.onStop()
    → Activity.onDestroy()       ← Activity is destroyed
    → ViewModel stays alive      ← ViewModel is NOT destroyed
    → new Activity.onCreate()
    → new Activity.onStart()
    → new Activity.onResume()
```

---

## ViewModel — Surviving Configuration Change

```kotlin
// ✅ ViewModel survives rotation — store all UI state here
class UserViewModel : ViewModel() {
    private val _state = MutableStateFlow(UserUiState())
    val state: StateFlow<UserUiState> = _state.asStateFlow()
}

// ✅ Activity/Fragment — just observe, never store state locally
class UserFragment : Fragment() {
    private val viewModel: UserViewModel by viewModels()

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        viewLifecycleOwner.lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.state.collect { render(it) }
            }
        }
    }
}
```

---

## Detecting Configuration in Code

```kotlin
// ✅ Read current configuration
val config = resources.configuration

// Night mode
val isNightMode = config.uiMode and Configuration.UI_MODE_NIGHT_MASK ==
        Configuration.UI_MODE_NIGHT_YES

// Orientation
val isLandscape = config.orientation == Configuration.ORIENTATION_LANDSCAPE

// Locale
val locale = config.locales[0]

// Screen size class
val isTablet = config.smallestScreenWidthDp >= 600

// ✅ In Compose
val isSystemInDarkTheme = isSystemInDarkTheme()
val windowSizeClass = calculateWindowSizeClass(activity)
```

---

## Dynamic Locale Change (API 33+)

```kotlin
// ✅ API 33+ — use LocaleManager
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
    getSystemService(LocaleManager::class.java)
        .applicationLocales = LocaleList(Locale.forLanguageTag("fa"))
}

// ✅ API < 33 — use AppCompatDelegate
AppCompatDelegate.setApplicationLocales(
    LocaleListCompat.forLanguageTags("fa")
)
```

---

## Dynamic Dark Mode

```kotlin
// ✅ Change theme at runtime
AppCompatDelegate.setDefaultNightMode(AppCompatDelegate.MODE_NIGHT_YES)
AppCompatDelegate.setDefaultNightMode(AppCompatDelegate.MODE_NIGHT_NO)
AppCompatDelegate.setDefaultNightMode(AppCompatDelegate.MODE_NIGHT_FOLLOW_SYSTEM)

// ✅ Persist preference and apply on app start
class ThemeManager(private val prefs: DataStore<Preferences>) {
    suspend fun applyTheme() {
        val isDark = prefs.data.first()[DARK_MODE_KEY] ?: false
        AppCompatDelegate.setDefaultNightMode(
            if (isDark) AppCompatDelegate.MODE_NIGHT_YES
            else AppCompatDelegate.MODE_NIGHT_FOLLOW_SYSTEM
        )
    }
}
```

---

## configChanges — When to Use

```xml
<!-- ✅ Only for cases where recreation is genuinely harmful -->
<!-- Example: video player, camera, map — recreation causes flicker/reset -->
<activity
    android:name=".PlayerActivity"
    android:configChanges="orientation|screenSize|keyboardHidden" />
```

```kotlin
// ✅ Handle manually when configChanges is declared
override fun onConfigurationChanged(newConfig: Configuration) {
    super.onConfigurationChanged(newConfig)
    if (newConfig.orientation == Configuration.ORIENTATION_LANDSCAPE) {
        adjustLayoutForLandscape()
    }
}

// ❌ Don't use configChanges just to avoid writing proper ViewModel code
```

---

## Window Size Classes (Adaptive UI)

```kotlin
// ✅ Use WindowSizeClass for adaptive layouts
@Composable
fun AdaptiveScreen(windowSizeClass: WindowSizeClass) {
    when (windowSizeClass.widthSizeClass) {
        WindowWidthSizeClass.Compact -> PhoneLayout()
        WindowWidthSizeClass.Medium  -> TabletLayout()
        WindowWidthSizeClass.Expanded -> DesktopLayout()
    }
}

// Setup in Activity
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        val windowSizeClass = calculateWindowSizeClass(this)
        setContent {
            AdaptiveScreen(windowSizeClass)
        }
    }
}
```

---

## Anti-Patterns

- Storing UI state in Activity fields — lost on rotation
- Using `android:configChanges` to avoid ViewModel — just delays the problem
- Reading configuration in ViewModel — ViewModel should be config-independent
- Hardcoding layout logic for orientation in code instead of resource qualifiers
- Not testing rotation — the most common source of state loss bugs
- Using `onRetainNonConfigurationInstance()` — replaced by ViewModel

---

## Related Skills

- `savedstatehandle` — persisting state across process death
- `lifecycle` — Activity/Fragment lifecycle during config change
- `viewmodel` — surviving configuration changes
- `resources` — configuration qualifiers in res/
- `adaptive-ui` — WindowSizeClass and responsive layouts
