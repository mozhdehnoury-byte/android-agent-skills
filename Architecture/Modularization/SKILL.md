---
name: modularization
description: >
  App modularization strategy for Android projects.
  Load this skill when deciding how to split a project into modules,
  defining module types and responsibilities, managing module dependencies,
  or planning build performance improvements through modularization.
---

# Modularization

## Overview
Modularization splits the app into independent Gradle modules. Each module has a single responsibility, clear API boundaries, and explicit dependencies. This improves build times (parallel compilation, build cache), enforces architecture boundaries, and enables code reuse across features.

---

## Core Principles

- Modules depend **inward** — feature modules never depend on other feature modules
- Each module exposes a **minimal public API** — use `internal` for implementation details
- **No circular dependencies** between modules
- Core/shared modules are **stable** — feature modules are volatile
- Module boundaries enforce **architecture layers** — a `data` module cannot import a `ui` module

---

## Module Types

```
:app                    — entry point, wires everything together
:core:ui                — shared Compose components, theme, design system
:core:domain            — shared domain models and interfaces
:core:data              — shared repository implementations, database, network
:core:common            — utilities, extensions, base classes
:feature:users          — users feature (presentation + domain + data for this feature)
:feature:products       — products feature
:feature:settings       — settings feature
```

---

## Recommended Structure

```
app/
├── app/                            # :app — Application, MainActivity, DI root
├── core/
│   ├── ui/                         # :core:ui — Theme, shared components
│   ├── domain/                     # :core:domain — Shared models, base UseCases
│   ├── data/                       # :core:data — Network, DB, shared repos
│   ├── common/                     # :core:common — Extensions, utils, Result
│   └── testing/                    # :core:testing — Fakes, test utilities
└── feature/
    ├── users/                      # :feature:users
    │   ├── src/main/
    │   │   ├── presentation/       # Screen, ViewModel, UiState
    │   │   ├── domain/             # Feature-specific UseCases
    │   │   └── data/               # Feature-specific Repository
    │   └── build.gradle.kts
    └── products/                   # :feature:products
```

---

## Dependency Graph

```
:app
 ├── :feature:users
 │    ├── :core:domain
 │    ├── :core:data
 │    └── :core:ui
 ├── :feature:products
 │    ├── :core:domain
 │    ├── :core:data
 │    └── :core:ui
 └── :core:common

# ✅ Allowed
:feature:users → :core:domain
:feature:users → :core:ui
:app → :feature:users

# ❌ Not allowed
:feature:users → :feature:products   # feature-to-feature dependency
:core:data → :feature:users          # core depending on feature
:core:ui → :core:data                # ui depending on data
```

---

## build.gradle.kts per Module

```kotlin
// ✅ Feature module build file
plugins {
    alias(libs.plugins.android.library)
    alias(libs.plugins.kotlin.android)
    alias(libs.plugins.hilt)
    alias(libs.plugins.ksp)
}

android {
    namespace = "com.example.feature.users"
    // shared config via convention plugin
}

dependencies {
    implementation(project(":core:domain"))
    implementation(project(":core:data"))
    implementation(project(":core:ui"))
    implementation(project(":core:common"))

    implementation(libs.hilt.android)
    ksp(libs.hilt.compiler)

    testImplementation(project(":core:testing"))
}
```

---

## Navigation Between Modules

```kotlin
// ✅ Feature modules expose navigation routes as constants
// :feature:users
object UsersNavigation {
    const val ROUTE = "users"
    const val USER_DETAIL_ROUTE = "users/{userId}"

    fun userDetailRoute(userId: String) = "users/$userId"
}

// ✅ :app wires navigation — features don't know about each other
@Composable
fun AppNavHost(navController: NavHostController) {
    NavHost(navController = navController, startDestination = UsersNavigation.ROUTE) {
        usersNavGraph(navController)
        productsNavGraph(navController)
    }
}

// ✅ Each feature exposes a NavGraph extension
fun NavGraphBuilder.usersNavGraph(navController: NavController) {
    navigation(
        startDestination = UsersNavigation.ROUTE,
        route = "users_graph"
    ) {
        composable(UsersNavigation.ROUTE) { UserListScreen(navController) }
        composable(UsersNavigation.USER_DETAIL_ROUTE) { UserDetailScreen(navController) }
    }
}
```

---

## Visibility Control

```kotlin
// ✅ Use internal to hide implementation from other modules
internal class UserRepositoryImpl @Inject constructor(...) : UserRepository

// ✅ Only expose what other modules need
class GetUserUseCase @Inject constructor(...)  // public — used by presentation
internal class UserRemoteDataSource(...)       // internal — implementation detail

// ✅ Hilt: internal classes need explicit binding
@Module
@InstallIn(SingletonComponent::class)
internal abstract class UserModule {
    @Binds
    abstract fun bindUserRepository(impl: UserRepositoryImpl): UserRepository
}
```

---

## Anti-Patterns

- Feature modules depending on each other — use `:app` or a shared `:core` module to mediate
- Putting everything in `:core:common` — keep common truly generic; feature logic belongs in features
- Circular dependencies — will cause Gradle build failure
- Public classes that should be `internal` — leaks implementation details
- One mega-module — defeats purpose of modularization; build times won't improve

---

## Related Skills
- `multi-module-architecture` — detailed multi-module setup and advanced patterns
- `hilt` — Hilt setup across modules
- `gradle` — Gradle configuration for multi-module projects
- `convention-plugin` — sharing build configuration across modules
- `clean-architecture` — layer boundaries that modularization enforces
