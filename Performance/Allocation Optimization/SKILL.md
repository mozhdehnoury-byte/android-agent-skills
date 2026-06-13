---
name: allocation-optimization
description: >
  Reducing object allocations and GC pressure in Android.
  Load this skill when optimizing hot code paths, reducing garbage
  collection pauses, minimizing allocations in draw/layout loops,
  or improving frame rate by reducing allocation churn.
---

# Allocation Optimization

## Overview
Excessive object allocation causes frequent garbage collection pauses, which manifest as frame drops and janky animations. On Android, the ART GC is generational and mostly concurrent, but large or frequent allocations in hot code paths (draw loops, scroll, animation) still cause measurable pauses.

---

## Core Principles

- **Never allocate** in `onDraw`, `Canvas` operations, or tight animation loops
- Reuse objects with object pools or `remember {}` in Compose
- Prefer **primitives** over boxed types (`Int` not `Integer`, `IntArray` not `List<Int>`)
- Use `StringBuilder` for string concatenation in loops — not `+` operator
- Profile with **Android Studio Profiler → Memory → Allocation** before optimizing

---

## Common Allocation Hotspots

```kotlin
// ❌ Allocating in draw loop — every frame creates new objects
class CustomView(context: Context) : View(context) {
    override fun onDraw(canvas: Canvas) {
        val paint = Paint()           // allocation every frame ❌
        val rect = RectF(0f, 0f, width.toFloat(), height.toFloat())  // allocation every frame ❌
        canvas.drawRect(rect, paint)
    }
}

// ✅ Pre-allocate outside draw loop
class CustomView(context: Context) : View(context) {
    private val paint = Paint().apply { color = Color.RED }
    private val rect = RectF()

    override fun onDraw(canvas: Canvas) {
        rect.set(0f, 0f, width.toFloat(), height.toFloat())  // reuse — no allocation
        canvas.drawRect(rect, paint)
    }
}
```

---

## String Concatenation in Loops

```kotlin
// ❌ String concatenation creates new String each iteration
var result = ""
items.forEach { result += it.name + ", " }  // O(n²) allocations

// ✅ StringBuilder — single allocation
val sb = StringBuilder()
items.forEach { sb.append(it.name).append(", ") }
val result = sb.toString()

// ✅ Or use joinToString — optimized internally
val result = items.joinToString(", ") { it.name }
```

---

## Avoiding Boxing

```kotlin
// ❌ Boxing primitives in hot path
val counts: Map<String, Int> = mutableMapOf()  // Int boxed to Integer

// ✅ Use SparseArray for Int keys — avoids boxing
val counts = SparseIntArray()

// ✅ Use primitive arrays
val buffer = IntArray(1024)  // no boxing
val floatBuffer = FloatArray(256)

// ❌ forEach on ranges — can box index
for (i in 0..100) { ... }  // optimized by compiler, actually fine

// ❌ Lambda capturing primitives
listOf(1, 2, 3).map { it * 2 }  // boxes Int in some cases
```

---

## Compose Allocation Optimization

```kotlin
// ❌ Creating lambda on every recomposition
@Composable
fun UserList(users: List<User>, onUserClick: (User) -> Unit) {
    users.forEach { user ->
        UserItem(
            user = user,
            onClick = { onUserClick(user) }  // ❌ new lambda every recomposition
        )
    }
}

// ✅ Use remember or stable callbacks
@Composable
fun UserList(users: List<User>, onUserClick: (User) -> Unit) {
    users.forEach { user ->
        val onClick = remember(user.id) { { onUserClick(user) } }
        UserItem(user = user, onClick = onClick)
    }
}

// ❌ Creating objects in composable body
@Composable
fun Chart(data: List<Float>) {
    val path = Path()  // ❌ recreated every recomposition
}

// ✅ remember to reuse
@Composable
fun Chart(data: List<Float>) {
    val path = remember { Path() }  // ✅ created once
}
```

---

## Reusable Buffers

```kotlin
// ✅ Thread-local buffer for repeated operations
private val threadLocalBuffer = ThreadLocal.withInitial { ByteArray(8192) }

fun processData(input: InputStream) {
    val buffer = threadLocalBuffer.get()!!
    var read: Int
    while (input.read(buffer).also { read = it } != -1) {
        process(buffer, read)
    }
}
```

---

## Measuring Allocations

```
Android Studio Profiler → Memory tab:
  1. Start recording allocations
  2. Perform the action to profile (scroll, animation)
  3. Stop recording
  4. Sort by "Alloc Count" — find hotspots

Key signals:
  - Many small allocations in RecyclerView/LazyColumn scroll → reuse objects
  - Repeated allocation of same type → use pool or remember
  - Allocation inside animation frame → move outside animation
```

---

## Anti-Patterns

- Allocating `Paint`, `Path`, `RectF` inside `onDraw` — every frame creates garbage
- String `+` concatenation in loops — use `StringBuilder`
- Creating collections inside tight loops — pre-allocate or reuse
- `List<Int>` when `IntArray` would work — unnecessary boxing
- Allocating large arrays on the main thread — use background thread

---

## Related Skills
- `heap-management` — managing heap size and cache limits
- `compose-optimization` — Compose-specific allocation patterns
- `rendering-performance` — frame rendering and draw optimization
- `benchmark` — measuring allocation count with benchmarks
