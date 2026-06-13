---
name: nested-navigation
description: >
  Nested navigation graphs in Jetpack Compose Navigation.
  Load this skill when organizing routes into feature-scoped graphs,
  implementing tab-based navigation with independent back stacks,
  isolating feature module navigation, or structuring auth vs main flows.
---

# Nested Navigation

## Overview
Nested navigation graphs group related destinations into a sub-graph with its own start destination. This enables feature isolation, independent back stacks per tab, and clean separation between flows (e.g. auth flow vs main flow).

---

## Core Principles

- Each feature or flow gets its own nested graph — never dump all routes into one flat NavHost
- Nested graphs have their own `startDestination` — the entry point is the only public route
- Tab-based navigation uses `saveState`/`restoreState` to preserve independent back stacks
- Auth flow and main flow must be separate nested graphs — not mixed in the same graph
- Never navigate directly to an internal route of another feature's graph

---

## Basic Nested Graph

```kotlin
// ✅ Define feature graph routes
@Serializable data object UserGraph
@Serializable data object UserListRoute
@Serializable data class UserDetailRoute(val userId: String)
@Serializable data class UserEditRoute(val userId: String)

// ✅ Register nested graph in NavHost
NavHost(navController = navController, startDestination = UserGraph) {

    navigation<UserGraph>(startDestination = UserListRoute) {

        composable<UserListRoute> {
            UserListScreen(
                onNavigateToDetail = { userId ->
                    navController.navigate(UserDetailRoute(userId))
                }
            )
        }

        composable<UserDetailRoute> { backStackEntry ->
            val route: UserDetailRoute = backStackEntry.toRoute()
            UserDetailScreen(
                userId = route.userId,
                onNavigateToEdit = {
                    navController.navigate(UserEditRoute(route.userId))
                },
                onNavigateBack = { navController.popBackStack() }
            )
        }

        composable<UserEditRoute> { backStackEntry ->
            val route: UserEditRoute = backStackEntry.toRoute()
            UserEditScreen(
                userId = route.userId,
                onNavigateBack = { navController.popBackStack() }
            )
        }
    }
}
```

---

## Auth vs Main Flow

```kotlin
// ✅ Separate auth and main flows as nested graphs
@Serializable data object AuthGraph
@Serializable data object LoginRoute
@Serializable data object RegisterRoute

@Serializable data object MainGraph
@Serializable data object HomeRoute
@Serializable data object SettingsRoute

@Composable
fun AppNavHost(
    navController: NavHostController,
    isAuthenticated: Boolean
) {
    NavHost(
        navController = navController,
        startDestination = if (isAuthenticated) MainGraph else AuthGraph
    ) {

        navigation<AuthGraph>(startDestination = LoginRoute) {
            composable<LoginRoute> {
                LoginScreen(
                    onLoginSuccess = {
                        navController.navigate(MainGraph) {
                            popUpTo(AuthGraph) { inclusive = true }  // ✅ clear auth stack
                        }
                    },
                    onNavigateToRegister = {
                        navController.navigate(RegisterRoute)
                    }
                )
            }
            composable<RegisterRoute> {
                RegisterScreen(
                    onRegisterSuccess = {
                        navController.navigate(MainGraph) {
                            popUpTo(AuthGraph) { inclusive = true }
                        }
                    },
                    onNavigateBack = { navController.popBackStack() }
                )
            }
        }

        navigation<MainGraph>(startDestination = HomeRoute) {
            composable<HomeRoute> { HomeScreen() }
            composable<SettingsRoute> { SettingsScreen() }
        }
    }
}
```

---

## Tab Navigation with Independent Back Stacks

```kotlin
// ✅ Each tab is a nested graph with its own back stack
@Serializable data object HomeGraph
@Serializable data object HomeRootRoute

@Serializable data object SearchGraph
@Serializable data object SearchRootRoute

@Serializable data object ProfileGraph
@Serializable data object ProfileRootRoute

@Composable
fun MainScreen() {
    val navController = rememberNavController()

    Scaffold(
        bottomBar = {
            BottomNavBar(navController = navController)
        }
    ) { padding ->
        NavHost(
            navController = navController,
            startDestination = HomeGraph,
            modifier = Modifier.padding(padding)
        ) {
            navigation<HomeGraph>(startDestination = HomeRootRoute) {
                composable<HomeRootRoute> { HomeScreen() }
                // more home destinations...
            }

            navigation<SearchGraph>(startDestination = SearchRootRoute) {
                composable<SearchRootRoute> { SearchScreen() }
                // more search destinations...
            }

            navigation<ProfileGraph>(startDestination = ProfileRootRoute) {
                composable<ProfileRootRoute> { ProfileScreen() }
                // more profile destinations...
            }
        }
    }
}

// ✅ Bottom nav switching with state save/restore
fun NavController.switchTab(route: Any) {
    navigate(route) {
        popUpTo(graph.findStartDestination().id) {
            saveState = true     // ✅ save current tab's back stack
        }
        launchSingleTop = true
        restoreState = true      // ✅ restore destination tab's back stack
    }
}
```

---

## Popping to Graph Start

```kotlin
// ✅ Pop back to the start of a nested graph
navController.popBackStack(route = UserListRoute, inclusive = false)

// ✅ Pop the entire graph off the stack
navController.popBackStack(route = UserGraph, inclusive = true)
```

---

## Anti-Patterns

- Flat NavHost with all routes at the same level — hard to maintain and reason about
- Navigating directly to an internal route of another feature graph from outside it
- Not using `saveState`/`restoreState` for tab navigation — back stack lost on tab switch
- Not clearing the auth graph when navigating to main — user can back-navigate to login
- Sharing a single `NavController` across nested `NavHost` composables

---

## Related Skills
- `navigation` — core navigation setup and route definitions
- `deep-navigation` — deep links into nested graphs
- `compose-navigation` — transitions and animations between graphs
