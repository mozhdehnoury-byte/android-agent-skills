---
name: extension-functions-design
description: >
  Design principles and patterns for Kotlin extension functions in Android.
  Load this skill when deciding whether to use an extension function,
  where to place it, how to scope it, and how to avoid common misuse patterns.
---

# Extension Functions Design

## Overview

Extension functions add behavior to existing types without inheritance or modification. They are one of Kotlin's most powerful features but must be used with discipline — overuse leads to scattered, hard-to-discover code.

---

## Core Principles

- Extensions are **discoverable utilities**, not replacements for proper design
- Only extend types **you don't own** or when inheritance is impossible
- If you own the type, **add the method directly** — don't use an extension
- Extensions must be **stateless and pure** — no side effects, no hidden dependencies
- Place extensions **close to where they're used** — not in a global utils file

---

## When to Use Extension Functions

| Situation                                                        | Use Extension?                     |
| ---------------------------------------------------------------- | ---------------------------------- |
| Adding utility to stdlib types (`String`, `List`, `Int`)         | ✅ Yes                              |
| Adding Android-specific helpers to `Context`, `View`, `Fragment` | ✅ Yes                              |
| Adding behavior to a class you own                               | ❌ No — add method directly         |
| Replacing a utility class method                                 | ✅ Yes                              |
| Adding domain logic to a domain model                            | ❌ No — add to the class or UseCase |
| Bypassing encapsulation to access internals                      | ❌ Never                            |

---

## Placement Strategy

```
// ✅ Place extensions next to the type they extend
// If extending Context → ContextExtensions.kt
// If extending String → StringExtensions.kt
// If extending View  → ViewExtensions.kt

// ✅ Place feature-specific extensions inside the feature package
// feature/auth/AuthExtensions.kt — extensions only used in auth
// NOT in a global utils/Extensions.kt that grows forever
```

---

## Stdlib Extensions

```kotlin
// ✅ String utilities
fun String.toSlug(): String =
    lowercase().replace(" ", "-").replace(Regex("[^a-z0-9-]"), "")

fun String.isValidEmail(): Boolean =
    android.util.Patterns.EMAIL_ADDRESS.matcher(this).matches()

fun String?.orEmpty(): String = this ?: ""

// ✅ Number formatting
fun Int.toPx(context: Context): Int =
    (this * context.resources.displayMetrics.density).toInt()

fun Long.toFormattedDate(): String =
    SimpleDateFormat("yyyy-MM-dd", Locale.getDefault()).format(Date(this))

// ✅ Collection utilities
fun <T> List<T>.second(): T = this[1]
fun <T> List<T>.indexOfFirstOrNull(predicate: (T) -> Boolean): Int? =
    indexOfFirst(predicate).takeIf { it >= 0 }
```

---

## Android Platform Extensions

```kotlin
// ✅ Context extensions
fun Context.showToast(message: String, duration: Int = Toast.LENGTH_SHORT) {
    Toast.makeText(this, message, duration).show()
}

fun Context.getColorCompat(@ColorRes colorRes: Int): Int =
    ContextCompat.getColor(this, colorRes)

fun Context.dpToPx(dp: Float): Int =
    TypedValue.applyDimension(TypedValue.COMPLEX_UNIT_DIP, dp, resources.displayMetrics).toInt()

// ✅ View extensions
fun View.show() { visibility = View.VISIBLE }
fun View.hide() { visibility = View.GONE }
fun View.invisible() { visibility = View.INVISIBLE }

fun View.setOnSingleClickListener(action: () -> Unit) {
    var lastClickTime = 0L
    setOnClickListener {
        val now = System.currentTimeMillis()
        if (now - lastClickTime > 500) {
            lastClickTime = now
            action()
        }
    }
}

// ✅ Fragment extensions
fun Fragment.hideKeyboard() {
    val imm = requireContext().getSystemService(InputMethodManager::class.java)
    imm.hideSoftInputFromWindow(requireView().windowToken, 0)
}
```

---

## Flow / Coroutine Extensions

```kotlin
// ✅ Lifecycle-aware collection helper
fun <T> Flow<T>.collectWithLifecycle(
    lifecycleOwner: LifecycleOwner,
    state: Lifecycle.State = Lifecycle.State.STARTED,
    block: suspend (T) -> Unit
) {
    lifecycleOwner.lifecycleScope.launch {
        lifecycleOwner.repeatOnLifecycle(state) {
            collect { block(it) }
        }
    }
}

// ✅ Result extensions
fun <T> Result<T>.onSuccess(action: (T) -> Unit): Result<T> {
    if (isSuccess) action(getOrThrow())
    return this
}
```

---

## Scoping with Companion Objects

```kotlin
// ✅ Scope extension to a specific context using companion object
class UserValidator {
    companion object  // empty companion — allows scoped extensions
}

fun UserValidator.Companion.isValidAge(age: Int): Boolean = age in 18..120

// Usage
UserValidator.isValidAge(25)
```

---

## Anti-Patterns

```kotlin
// ❌ Extending a class you own — add the method directly
class User(val name: String)
fun User.displayName() = name.trim()  // wrong — add to User class

// ❌ Extension that accesses internal state via reflection or casting
fun Any.forceGetField(name: String): Any? { ... }  // bypasses encapsulation

// ❌ Extension with hidden dependencies
fun String.sendToAnalytics() {
    AnalyticsSingleton.track(this)  // hidden side effect — not obvious from signature
}

// ❌ Giant global Extensions.kt file
// utils/Extensions.kt with 500 lines of unrelated extensions
// → split by type and feature

// ❌ Extension that duplicates stdlib
fun List<Int>.mySum() = reduce { acc, i -> acc + i }  // use sum() instead
```

---

## Related Skills

- `kotlin` — core Kotlin language conventions
- `dsl` — extension functions as DSL building blocks
- `reactive-streams` — Flow extension patterns
