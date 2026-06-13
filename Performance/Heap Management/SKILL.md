---
name: heap-management
description: >
  Android heap memory management and optimization.
  Load this skill when investigating OutOfMemoryError, reducing memory
  footprint, understanding heap limits, managing large objects,
  or profiling memory usage with Android Studio Memory Profiler.
---

# Heap Management

## Overview
Each Android app has a limited heap size (typically 128–512MB depending on device). Exceeding this limit causes `OutOfMemoryError`. Proper heap management involves avoiding large allocations, reusing objects, releasing memory under pressure, and profiling with Android Studio Memory Profiler.

---

## Core Principles

- Never hold large objects (Bitmap, byte arrays) longer than necessary
- Release memory in `onTrimMemory` — the system signals when memory is low
- Use object pools for frequently created/destroyed objects
- Profile memory with **Android Studio Memory Profiler** before optimizing
- Prefer streaming over loading entire files into memory

---

## Heap Limits

```kotlin
// ✅ Check available heap at runtime
val runtime = Runtime.getRuntime()
val maxHeap = runtime.maxMemory()          // max heap the app can use
val usedHeap = runtime.totalMemory() - runtime.freeMemory()
val availableHeap = maxHeap - usedHeap

Timber.d("Heap: used=${usedHeap/1024/1024}MB, max=${maxHeap/1024/1024}MB")

// ✅ Request large heap in manifest (use sparingly)
// android:largeHeap="true" in <application> — last resort
```

---

## onTrimMemory — Release Under Pressure

```kotlin
// ✅ Release non-essential memory when system requests it
class App : Application() {
    override fun onTrimMemory(level: Int) {
        super.onTrimMemory(level)
        when (level) {
            TRIM_MEMORY_UI_HIDDEN -> {
                // App went to background — release UI caches
                imageCache.evictAll()
            }
            TRIM_MEMORY_RUNNING_CRITICAL,
            TRIM_MEMORY_COMPLETE -> {
                // Critical — release everything possible
                imageCache.evictAll()
                dataCache.clear()
            }
        }
    }
}

// ✅ Implement in ViewModel too
class UserListViewModel : ViewModel(), ComponentCallbacks2 {
    override fun onTrimMemory(level: Int) {
        if (level >= ComponentCallbacks2.TRIM_MEMORY_MODERATE) {
            cachedUsers.clear()
        }
    }
    override fun onConfigurationChanged(newConfig: Configuration) {}
    override fun onLowMemory() { cachedUsers.clear() }
}
```

---

## LruCache for Memory Caching

```kotlin
// ✅ LruCache — evicts least-recently-used entries when full
class ImageMemoryCache {
    private val maxMemory = (Runtime.getRuntime().maxMemory() / 1024).toInt()
    private val cacheSize = maxMemory / 8  // use 1/8 of available heap

    private val cache = object : LruCache<String, Bitmap>(cacheSize) {
        override fun sizeOf(key: String, bitmap: Bitmap): Int {
            return bitmap.byteCount / 1024  // size in KB
        }
    }

    fun put(key: String, bitmap: Bitmap) = cache.put(key, bitmap)
    fun get(key: String): Bitmap? = cache.get(key)
    fun evictAll() = cache.evictAll()
}
```

---

## Avoiding Large Allocations

```kotlin
// ❌ Loading entire file into memory
val bytes = File(path).readBytes()  // OOM for large files

// ✅ Stream the file
File(path).inputStream().use { stream ->
    val buffer = ByteArray(8192)  // 8KB buffer
    var bytesRead: Int
    while (stream.read(buffer).also { bytesRead = it } != -1) {
        process(buffer, bytesRead)
    }
}

// ❌ Parsing entire JSON response before using it
val json = response.body?.string()  // loads everything into memory

// ✅ Stream JSON parsing with Kotlinx Serialization
val users = json.decodeFromStream<List<UserDto>>(response.body!!.byteStream())
```

---

## Object Pooling

```kotlin
// ✅ Reuse objects instead of creating new ones in tight loops
class RectPool {
    private val pool = ArrayDeque<RectF>()

    fun acquire(): RectF = pool.removeFirstOrNull() ?: RectF()

    fun release(rect: RectF) {
        rect.setEmpty()
        pool.addLast(rect)
    }
}

// ✅ Usage
val rectPool = RectPool()
val rect = rectPool.acquire()
try {
    // use rect
} finally {
    rectPool.release(rect)
}
```

---

## Profiling Tools

```
Android Studio Memory Profiler:
  - Heap dump: snapshot of all objects in memory
  - Allocation tracking: which code allocates the most
  - GC events: frequency indicates memory pressure

Key metrics to watch:
  - Java Heap: should not grow unboundedly
  - Native Heap: images, bitmaps (decoded natively)
  - Shallow size: size of object itself
  - Retained size: size of object + everything it keeps alive
```

---

## Anti-Patterns

- Holding Bitmap references in static fields — never GC'd
- Caching without size limits — unbounded cache grows until OOM
- Loading full-resolution images for thumbnails — decode at required size
- Creating many small objects in tight loops — GC pressure
- Not implementing `onTrimMemory` — app doesn't release memory under pressure

---

## Related Skills
- `bitmap-optimization` — efficient Bitmap loading and scaling
- `memory-leak-prevention` — preventing reference leaks
- `compose-optimization` — reducing recompositions and allocations
- `startup-optimization` — reducing memory allocations at startup
