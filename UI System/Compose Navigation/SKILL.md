---
name: compose-navigation
description: >
  Jetpack Compose Navigation setup, patterns, and best practices.
  Load this skill when setting up NavHost, defining routes, navigating between
  screens, passing arguments, handling deep links, or managing the back stack
  in a Compose app.
---

# Compose Navigation

## Overview
Jetpack Compose Navigation provides a type-safe way to navigate between composable screens. Navigation is managed by a `NavController` and defined in a `NavHost`. Arguments, deep links, and back stack management are all handled through the navigation graph.

---

## Core Principles

- `NavController` must be created at the **top level** — never inside nested composables
- Pass `NavController` down **only one level** — deeper screens receive lambdas, not NavController
- Use **type-safe routes** (Navigation 2.8+) — avoid string-based routes
- Navigation logic belongs in the **screen-level composable** — not in ViewModel
- **Never navigate from inside a ViewModel** — emit an event, handle navigation in UI

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

## Type-Safe Routes (Navigation 2.8+)

```kotlin
// ✅ Define routes as serializable objects/data classes
@Serializable
object HomeRoute

@Serializable
object UserListRoute

@Serializable
data class UserDetailRoute(val userId: String)

@Serializable
data class EditUserRoute(val userId: String, val isNew: Boolean = false)
```

---

## NavHost Setup

```kotlin
// ✅ NavHost at top level — in Activity or root composable
@Composable
fun AppNavHost(
    navController: NavHostController = rememberNavController(),
    startDestination: Any = HomeRoute
) {
    NavHost(
        navController = navController,
        startDestination = startDestination
    ) {
        composable<HomeRoute> {
            HomeScreen(
                onNavigateToUsers = { navController.navigate(UserListRoute) }
            )
        }

        composable<UserListRoute> {
            UserListScreen(
                onUserClick = { userId ->
                    navController.navigate(UserDetailRoute(userId))
                },
                onBack = { navController.navigateUp() }
            )
        }

        composable<UserDetailRoute> { backStackEntry ->
            val route = backStackEntry.toRoute<UserDetailRoute>()
            UserDetailScreen(
                userId = route.userId,
                onBack = { navController.navigateUp() },
                onEdit = { navController.navigate(EditUserRoute(route.userId)) }
            )
        }
    }
}
```

---

## Reading Arguments in ViewModel

```kotlin
// ✅ ViewModel reads route args from SavedStateHandle
@HiltViewModel
class UserDetailViewModel @Inject constructor(
    savedStateHandle: SavedStateHandle,
    private val repository: UserRepository
) : ViewModel() {

    private val route = savedStateHandle.toRoute<UserDetailRoute>()
    val userId = route.userId

    val user: StateFlow<User?> = repository
        .observeUser(userId)
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), null)
}
```

---

## Navigation Patterns

```kotlin
// ✅ Navigate and clear back stack (login → home)
navController.navigate(HomeRoute) {
    popUpTo(navController.graph.startDestinationId) { inclusive = true }
}

// ✅ Avoid duplicate destinations
navController.navigate(UserDetailRoute(userId)) {
    launchSingleTop = true
}

// ✅ Navigate up (back)
navController.navigateUp()

// ✅ Pop to specific destination
navController.popBackStack(UserListRoute, inclusive = false)

// ✅ Navigate with result (pass data back)
// Sender
navController.previousBackStackEntry
    ?.savedStateHandle
    ?.set("selected_user_id", userId)
navController.navigateUp()

// Receiver
val selectedUserId = navController.currentBackStackEntry
    ?.savedStateHandle
    ?.getStateFlow<String?>("selected_user_id", null)
    ?.collectAsStateWithLifecycle()
```

---

## Passing NavController Down — Lambda Pattern

```kotlin
// ✅ Only one level deep — screen composables receive lambdas
@Composable
fun AppNavHost(navController: NavHostController) {
    NavHost(navController, startDestination = HomeRoute) {
        composable<HomeRoute> {
            // ✅ Pass lambda, not navController
            HomeScreen(
                onNavigateToUsers = { navController.navigate(UserListRoute) },
                onNavigateToSettings = { navController.navigate(SettingsRoute) }
            )
        }
    }
}

// ✅ Screen receives lambdas — decoupled from navigation
@Composable
fun HomeScreen(
    onNavigateToUsers: () -> Unit,
    onNavigateToSettings: () -> Unit,
    viewModel: HomeViewModel = hiltViewModel()
) {
    // ...
}

// ❌ Don't pass NavController into screen composables
@Composable
fun HomeScreen(navController: NavController) { // tightly coupled
    Button(onClick = { navController.navigate(UserListRoute) }) { ... }
}
```

---

## Deep Links

```kotlin
// ✅ Define deep links in NavHost
composable<UserDetailRoute>(
    deepLinks = listOf(
        navDeepLink<UserDetailRoute>(
            basePath = "https://example.com/users"
        )
    )
) { ... }

// AndroidManifest.xml — declare intent filter
// (see deep-link skill for full manifest setup)
```

---

## Navigation with Hilt

```kotlin
// ✅ hiltViewModel() — scoped to NavBackStackEntry
composable<UserDetailRoute> {
    val viewModel: UserDetailViewModel = hiltViewModel()
    UserDetailScreen(viewModel = viewModel)
}

// ✅ Shared ViewModel across multiple screens
composable<CheckoutRoute> { backStackEntry ->
    val parentEntry = remember(backStackEntry) {
        navController.getBackStackEntry(CheckoutFlowRoute)
    }
    val sharedViewModel: CheckoutViewModel = hiltViewModel(parentEntry)
    CheckoutScreen(viewModel = sharedViewModel)
}
```

---

## Anti-Patterns

- Creating `NavController` inside a non-top-level composable — causes recreation
- Passing `NavController` more than one level deep — tight coupling
- Navigating from ViewModel — emit event, navigate from UI
- Using string-based routes in new code — use type-safe routes
- Not using `launchSingleTop` for bottom nav tabs — duplicates destinations
- Not handling `navigateUp()` return value — can silently fail at root

---

## Related Skills
- `navigation` — general navigation patterns and principles
- `deep-navigation` — nested graphs and deep link handling
- `nested-navigation` — nested NavHost and graphs
- `hilt` — ViewModel injection with hiltViewModel()
- `savedstatehandle` — reading route arguments in ViewModel
