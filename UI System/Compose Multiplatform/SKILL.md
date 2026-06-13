---
name: compose-multiplatform
description: >
  Compose Multiplatform (CMP) setup and patterns for sharing UI across
  Android, iOS, Desktop, and Web. Load this skill when building shared
  UI with CMP, handling platform-specific UI differences, or deciding
  what UI code belongs in shared vs platform modules.
---

# Compose Multiplatform

## Overview
Compose Multiplatform (CMP) by JetBrains extends Jetpack Compose to share UI code across Android, iOS, Desktop (JVM), and Web (Wasm). It builds on top of KMP for shared logic and adds a shared UI layer. The same composable functions render natively on each platform.

---

## Core Principles

- Share UI that is **truly identical** across platforms — don't force sharing where platforms differ
- Platform-specific UI (navigation bars, status bars, permissions UI) stays **platform-specific**
- Use `expect/actual` for platform-specific UI components
- Use `rememberX()` patterns for platform resources (images, fonts)
- Test shared UI on **all target platforms** — rendering differences exist

---

## Project Structure

```
project/
├── composeApp/
│   ├── src/
│   │   ├── commonMain/         ← shared UI + business logic
│   │   │   └── kotlin/
│   │   │       ├── App.kt      ← root composable
│   │   │       ├── screens/
│   │   │       └── components/
│   │   ├── androidMain/        ← Android entry point + platform UI
│   │   │   └── kotlin/
│   │   │       └── MainActivity.kt
│   │   ├── iosMain/            ← iOS entry point
│   │   │   └── kotlin/
│   │   │       └── MainViewController.kt
│   │   └── desktopMain/        ← Desktop entry point
│   │       └── kotlin/
│   │           └── main.kt
├── shared/                     ← shared business logic (no UI)
└── iosApp/                     ← Xcode project
```

---

## Root Composable (commonMain)

```kotlin
// commonMain/App.kt
@Composable
fun App() {
    AppTheme {
        // Shared navigation and screens
        AppNavHost()
    }
}
```

---

## Platform Entry Points

```kotlin
// androidMain/MainActivity.kt
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent { App() }
    }
}

// iosMain/MainViewController.kt
fun MainViewController() = ComposeUIViewController { App() }

// desktopMain/main.kt
fun main() = application {
    Window(title = "MyApp", onCloseRequest = ::exitApplication) {
        App()
    }
}
```

---

## Platform-Specific Components

```kotlin
// ✅ expect/actual for platform-specific UI
// commonMain
@Composable
expect fun PlatformStatusBar()

// androidMain
@Composable
actual fun PlatformStatusBar() {
    // Android-specific status bar handling
}

// iosMain
@Composable
actual fun PlatformStatusBar() {
    // iOS safe area insets handling
    Spacer(modifier = Modifier.windowInsetsTopHeight(WindowInsets.safeDrawing))
}

// desktopMain
@Composable
actual fun PlatformStatusBar() {
    // No status bar on desktop
}
```

---

## Resources in CMP

```kotlin
// ✅ Use compose-resources for shared resources
// commonMain/composeResources/
//   drawable/logo.png
//   font/roboto.ttf
//   values/strings.xml

// Access in shared composable
@Composable
fun Logo() {
    Image(
        painter = painterResource(Res.drawable.logo),
        contentDescription = null
    )
}

Text(stringResource(Res.string.app_name))
```

```toml
# libs.versions.toml
[libraries]
compose-resources = { module = "org.jetbrains.compose.components:components-resources" }
```

---

## Navigation in CMP

```kotlin
// ✅ Use Navigation Compose (works in CMP)
// commonMain
@Composable
fun AppNavHost() {
    val navController = rememberNavController()
    NavHost(navController, startDestination = HomeRoute) {
        composable<HomeRoute> {
            HomeScreen(onNavigate = { navController.navigate(DetailRoute(it)) })
        }
        composable<DetailRoute> { DetailScreen() }
    }
}
```

---

## Platform Detection

```kotlin
// ✅ When platform-specific behavior needed in shared code
expect fun getPlatformName(): String

// androidMain
actual fun getPlatformName() = "Android"

// iosMain
actual fun getPlatformName() = "iOS"

// ✅ Use for conditional UI
@Composable
fun PlatformAwareLayout() {
    if (getPlatformName() == "Android") {
        AndroidSpecificLayout()
    } else {
        DefaultLayout()
    }
}
```

---

## Gradle Setup

```kotlin
// composeApp/build.gradle.kts
plugins {
    alias(libs.plugins.kotlin.multiplatform)
    alias(libs.plugins.compose.multiplatform)
    alias(libs.plugins.android.application)
}

kotlin {
    androidTarget()
    iosX64(); iosArm64(); iosSimulatorArm64()
    jvm("desktop")

    sourceSets {
        commonMain.dependencies {
            implementation(compose.runtime)
            implementation(compose.foundation)
            implementation(compose.material3)
            implementation(compose.ui)
            implementation(compose.components.resources)
            implementation(libs.navigation.compose)
        }
        androidMain.dependencies {
            implementation(libs.androidx.activity.compose)
        }
        val desktopMain by getting {
            dependencies {
                implementation(compose.desktop.currentOs)
            }
        }
    }
}
```

---

## What to Share vs Platform-Specific

| UI Element | Share | Platform-Specific |
|------------|-------|------------------|
| Business screens (list, form, detail) | ✅ | |
| Design system components | ✅ | |
| Navigation structure | ✅ | |
| Status bar / navigation bar | | ✅ |
| Permission request UI | | ✅ |
| Platform pickers (date, file) | | ✅ |
| App entry point | | ✅ |

---

## Anti-Patterns

- Importing `android.*` in `commonMain` — breaks iOS/Desktop compilation
- Sharing UI that looks wrong on a platform — prefer platform-specific when UX differs
- Using Android-specific Compose APIs in shared code (e.g., `LocalContext`) — use expect/actual
- Not testing on iOS — rendering differences are common
- Placing platform resources in `commonMain/res/` — use `composeResources/` instead

---

## Related Skills
- `kmp` — Kotlin Multiplatform shared logic
- `compose` — Compose fundamentals
- `compose-navigation` — Navigation in CMP
- `material3` — Material 3 in CMP
