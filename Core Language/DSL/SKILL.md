---
name: dsl
description: >
  Kotlin DSL design and implementation patterns for Android development.
  Load this skill when designing builder APIs, configuration blocks,
  type-safe builders, or any fluent API that leverages Kotlin's DSL capabilities.
---

# Kotlin DSL

## Overview

Kotlin DSLs (Domain-Specific Languages) allow expressing configuration, building objects, and defining structure in a readable, type-safe way using lambdas with receivers. Common in Gradle scripts, Compose, Ktor, and custom builders.

---

## Core Principles

- DSLs should **read like natural language** in their domain
- DSLs should be **type-safe** — wrong usage must fail at compile time
- Use `@DslMarker` to prevent implicit receiver leakage
- Keep DSL scope **focused** — one responsibility per receiver class

---

## Basic DSL Pattern

```kotlin
// ✅ Lambda with receiver
class DialogBuilder {
    var title: String = ""
    var message: String = ""
    private val buttons = mutableListOf<Button>()

    fun button(label: String, action: () -> Unit) {
        buttons.add(Button(label, action))
    }

    fun build(): Dialog = Dialog(title, message, buttons)
}

fun dialog(block: DialogBuilder.() -> Unit): Dialog {
    return DialogBuilder().apply(block).build()
}

// Usage
val dialog = dialog {
    title = "Confirm"
    message = "Are you sure?"
    button("Yes") { confirm() }
    button("No") { dismiss() }
}
```

---

## @DslMarker — Preventing Receiver Leakage

```kotlin
// ✅ Always annotate DSL scopes with @DslMarker
@DslMarker
annotation class DialogDsl

@DialogDsl
class DialogBuilder { ... }

@DialogDsl
class ButtonBuilder { ... }

// Without @DslMarker, inner blocks can access outer receivers
// causing confusing and hard-to-debug behavior
```

---

## Type-Safe Builder Pattern

```kotlin
// ✅ Use for building structured/hierarchical data
@DslMarker
annotation class RouteDsl

@RouteDsl
class RouteBuilder {
    private val routes = mutableListOf<Route>()

    fun route(path: String, block: RouteConfig.() -> Unit) {
        routes.add(RouteConfig(path).apply(block).build())
    }

    fun build(): List<Route> = routes
}

@RouteDsl
class RouteConfig(val path: String) {
    var requiresAuth: Boolean = false
    var deepLink: String? = null

    fun build(): Route = Route(path, requiresAuth, deepLink)
}

fun routes(block: RouteBuilder.() -> Unit): List<Route> =
    RouteBuilder().apply(block).build()

// Usage
val appRoutes = routes {
    route("/home") {
        requiresAuth = true
    }
    route("/login") {
        deepLink = "app://login"
    }
}
```

---

## DSL for Configuration

```kotlin
// ✅ Common pattern for library/module configuration
class NetworkConfig {
    var baseUrl: String = ""
    var timeoutMs: Long = 30_000
    var retryCount: Int = 3
    var enableLogging: Boolean = false
}

fun configureNetwork(block: NetworkConfig.() -> Unit): NetworkConfig =
    NetworkConfig().apply(block)

// Usage
val config = configureNetwork {
    baseUrl = "https://api.example.com"
    timeoutMs = 60_000
    enableLogging = BuildConfig.DEBUG
}
```

---

## DSL in Gradle (Convention Plugins)

```kotlin
// ✅ Expose DSL-style API in convention plugins
fun Project.androidLibrary(block: LibraryExtension.() -> Unit) {
    extensions.configure(LibraryExtension::class.java, block)
}

// Usage in module build.gradle.kts
androidLibrary {
    compileSdk = 35
    defaultConfig {
        minSdk = 24
    }
}
```

---

## When to Use DSL

| Use DSL                                            | Don't Use DSL                       |
| -------------------------------------------------- | ----------------------------------- |
| Building complex objects with many optional fields | Simple data classes with few fields |
| Hierarchical/nested configuration                  | One-level flat configuration        |
| Domain has natural language structure              | Technical/algorithmic code          |
| Builder would have 5+ parameters                   | Builder has 1-2 parameters          |

---

## Anti-Patterns

- DSL without `@DslMarker` — allows accidental outer receiver access
- Mutable state escaping the DSL scope — DSL output should be immutable
- DSL that requires calling methods in a specific order — use phases/staged builders instead
- Too many nested levels — more than 3 levels deep becomes unreadable
- Side effects inside DSL blocks — DSL should declare, not execute

---

## Related Skills

- `kotlin` — Kotlin language fundamentals
- `extension-functions-design` — extension functions used in DSL design
- `gradle` — Gradle Kotlin DSL conventions
