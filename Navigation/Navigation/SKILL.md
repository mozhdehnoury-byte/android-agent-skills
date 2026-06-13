---
name: navigation
description: >
  Navigation setup and patterns in Jetpack Compose using Navigation Compose.
  Load this skill when setting up the NavHost, defining routes, passing
  arguments between screens, handling back stack, or structuring navigation
  in a single-activity architecture.
---

# Navigation

## Overview
Navigation Compose is the standard navigation library for Jetpack Compose. It manages the back stack, screen transitions, and argument passing within a single-activity architecture. Routes are defined as type-safe objects using Kotlin serialization.

---

## Core Principles

- One `NavHost` per app — never nest multiple NavHosts at the top level
- Routes must be **type-safe** using `@Serializable` data objects/classes — never string literals
- Navigation logic belongs in the **ViewModel or event callbacks** — never inside composables directly
- Pass only **IDs** between screens — never full objects
- Always use `launchSingleTop = true` for bottom navigation items

---

## Setup

```toml
# libs.versions.toml
[versions]
navigation-compose = "2.8.0"

[libraries]
navigation-compose = { module = "androidx.navigation:navigation-compose", version.ref = "navigation-compose" }
```

```kotlin
// build.gradle.kts
dependencies {
    implementation(libs.navigation.compose)
}
```

---

## Type-Safe Routes

```kotlin
// ✅ Define routes as serializable objects/classes
@Serializable
data object HomeRoute

@Serializable
data object SettingsRoute

@Serializable
data class UserDetailRoute(val userId: String)

@Serializable
data class EditUserRoute(val userId: String, val mode: String = "edit")
```

---

## NavHost Setup

```kotlin
// ✅ NavHost at the app root
@Composable
fun AppNavHost(
    navController: NavHostController = rememberNavController()
) {
    NavHost(
        navController = navController,
        startDestination = HomeRoute
    ) {
        composable<HomeRoute> {
            HomeScreen(
                onNavigateToUser = { userId ->
                    navController.navigate(UserDetailRoute(userId))
                }
            )
        }

        composable<UserDetailRoute> { backStackEntry ->
            val route: UserDetailRoute = backStackEntry.toRoute()
            UserDetailScreen(
                userId = route.userId,
                onNavigateBack = { navController.popBackStack() },
                onNavigateToEdit = {
                    navController.navigate(EditUserRoute(route.userId))
                }
            )
        }

        composable<EditUserRoute> { backStackEntry ->
            val route: EditUserRoute = backStackEntry.toRoute()
            EditUserScreen(
                userId = route.userId,
                onNavigateBack = { navController.popBackStack() }
            )
        }

        composable<SettingsRoute> {
            SettingsScreen(
                onNavigateBack = { navController.popBackStack() }
            )
        }
    }
}
```

---

## Bottom Navigation

```kotlin
// ✅ Bottom navigation with launchSingleTop
@Composable
fun AppScaffold(navController: NavHostController) {
    val navBackStackEntry by navController.currentBackStackEntryAsState()
    val currentDestination = navBackStackEntry?.destination

    Scaffold(
        bottomBar = {
            NavigationBar {
                bottomNavItems.forEach { item ->
                    NavigationBarItem(
                        icon = { Icon(item.icon, contentDescription = null) },
                        label = { Text(item.label) },
                        selected = currentDestination?.hierarchy?.any {
                            it.hasRoute(item.route::class)
                        } == true,
                        onClick = {
                            navController.navigate(item.route) {
                                popUpTo(navController.graph.findStartDestination().id) {
                                    saveState = true
                                }
                                launchSingleTop = true   // ✅ avoid duplicates
                                restoreState = true      // ✅ restore tab state
                            }
                        }
                    )
                }
            }
        }
    ) { padding ->
        AppNavHost(
            navController = navController,
            modifier = Modifier.padding(padding)
        )
    }
}
```

---

## Passing Results Back

```kotlin
// ✅ Pass result back via SavedStateHandle
// In destination screen
navController.previousBackStackEntry
    ?.savedStateHandle
    ?.set("selected_item_id", selectedItemId)
navController.popBackStack()

// ✅ Observe in source screen's ViewModel
class SourceViewModel(
    savedStateHandle: SavedStateHandle
) : ViewModel() {
    val selectedItemId = savedStateHandle
        .getStateFlow<String?>("selected_item_id", null)
}
```

---

## Navigation Extensions

```kotlin
// ✅ Extension to reduce boilerplate
fun NavController.navigateSingleTop(route: Any) {
    navigate(route) {
        launchSingleTop = true
    }
}

fun NavController.navigateAndClearStack(route: Any) {
    navigate(route) {
        popUpTo(0) { inclusive = true }
        launchSingleTop = true
    }
}
```

---

## Anti-Patterns

- Using raw string routes — loses type safety and refactoring support
- Passing full objects (Parcelable/Serializable) as nav arguments — pass IDs only
- Calling `navController.navigate()` directly inside a composable body — use callbacks or events
- Nesting `NavHost` inside another `NavHost` at the top level — use nested graphs instead
- Not using `launchSingleTop` for bottom nav — creates duplicate back stack entries
- Holding `NavController` reference in ViewModel — pass navigation events as lambdas

---

## Related Skills
- `compose-navigation` — advanced NavHost patterns and animations
- `deep-navigation` — deep links and external navigation
- `nested-navigation` — nested graphs and feature module navigation
- `savedstatehandle` — passing data between screens
