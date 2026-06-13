---
name: rendering-performance
description: >
  UI rendering performance optimization in Android.
  Load this skill when diagnosing dropped frames, reducing overdraw,
  optimizing custom view drawing, improving scroll performance,
  or profiling with GPU Overdraw and Frame Pacing tools.
---

# Rendering Performance

## Overview
Android targets 60fps (16ms per frame) or 120fps (8ms per frame) on high-refresh devices. Dropped frames cause visible jank. The main causes are: work on the main thread during frame rendering, overdraw, expensive layout passes, and large hierarchies. The key tool is the **System Trace** and **GPU Overdraw** visualizer.

---

## Core Principles

- Keep the **main thread free** during frame rendering — offload work to coroutines
- Avoid **overdraw** — drawing the same pixel multiple times wastes GPU time
- Flatten view hierarchies — deep nesting causes expensive measure/layout passes
- Avoid `invalidate()` calls that trigger full redraws — invalidate only dirty regions
- Use hardware acceleration — it's on by default, don't disable it

---

## Frame Budget

```
60fps target  → 16ms per frame
120fps target → 8ms per frame

Frame work breakdown:
  Input handling     ~2ms
  Animation         ~2ms
  Measure/Layout    ~4ms
  Draw              ~4ms
  Sync & Upload     ~2ms
  GPU work          ~2ms
```

---

## Detecting Jank

```kotlin
// ✅ FrameMetrics API — detect dropped frames programmatically
class PerformanceMonitor(private val activity: Activity) {

    private val frameMetricsListener = Window.OnFrameMetricsAvailableListener { _, frameMetrics, _ ->
        val totalDuration = frameMetrics.getMetric(FrameMetrics.TOTAL_DURATION) / 1_000_000L // ns to ms
        if (totalDuration > 16) {
            Timber.w("Slow frame: ${totalDuration}ms")
        }
    }

    fun start() {
        activity.window.addOnFrameMetricsAvailableListener(
            frameMetricsListener,
            Handler(Looper.getMainLooper())
        )
    }

    fun stop() {
        activity.window.removeOnFrameMetricsAvailableListener(frameMetricsListener)
    }
}
```

---

## Reducing Overdraw

```kotlin
// ✅ Remove redundant backgrounds
// In Compose — don't set background on every layer
@Composable
fun Screen() {
    // ❌ Both Scaffold and Column have background — overdraw
    Scaffold {
        Column(modifier = Modifier.background(Color.White)) { ... }
    }

    // ✅ Only one background needed — Scaffold handles it
    Scaffold {
        Column { ... }
    }
}

// ✅ In XML theme — remove window background if screen has its own
// android:windowBackground="@null" in theme (if screen fills the window)
```

---

## Compose Rendering Tips

```kotlin
// ✅ Use graphicsLayer for hardware-accelerated transformations
Box(
    modifier = Modifier
        .graphicsLayer {
            scaleX = scale
            scaleY = scale
            alpha = opacity
            // GPU-accelerated — doesn't trigger recomposition
        }
)

// ✅ Avoid clip + shadow on the same composable — expensive
// Instead, use separate layers
Box(
    modifier = Modifier
        .shadow(elevation = 4.dp, shape = RoundedCornerShape(8.dp))
        // shadow handles clipping internally
)

// ✅ Use drawWithCache for custom drawing that doesn't change every frame
Modifier.drawWithCache {
    val path = Path()
    path.addRoundRect(...)  // computed once, reused across frames
    onDrawBehind {
        drawPath(path, color = Color.Blue)
    }
}
```

---

## Main Thread Protection

```kotlin
// ✅ Never block main thread during scroll/animation
@Composable
fun UserList(viewModel: UserListViewModel) {
    val users by viewModel.users.collectAsStateWithLifecycle()

    LazyColumn {
        items(users, key = { it.id }) { user ->
            // ❌ Don't do I/O or heavy computation here
            // val data = File(user.avatarPath).readBytes()  // blocks main thread

            // ✅ Data already loaded — just render
            UserItem(user = user)
        }
    }
}
```

---

## Custom View — Dirty Region Invalidation

```kotlin
// ✅ Invalidate only the changed region — not the entire view
class ProgressView(context: Context) : View(context) {
    private var progress = 0f
    private val dirtyRect = Rect()

    fun setProgress(value: Float) {
        val oldRight = (progress * width).toInt()
        progress = value
        val newRight = (progress * width).toInt()

        // Invalidate only the changed horizontal strip
        dirtyRect.set(minOf(oldRight, newRight), 0, maxOf(oldRight, newRight), height)
        invalidate(dirtyRect)  // ✅ partial invalidation
    }
}
```

---

## Profiling Tools

```
GPU Overdraw (Developer Options → Debug GPU overdraw):
  Blue   = 1x overdraw  (acceptable)
  Green  = 2x overdraw  (minor issue)
  Pink   = 3x overdraw  (investigate)
  Red    = 4x+ overdraw (fix immediately)

System Trace (Android Studio Profiler → CPU → System Trace):
  - See exact frame timing
  - Identify which method causes the long frame
  - Find binder calls, lock contention on main thread
```

---

## Anti-Patterns

- Performing disk or network I/O on the main thread — causes frame drops
- Deep nested `ConstraintLayout` inside `ConstraintLayout` — expensive measure passes
- Setting background on every layout level — causes overdraw
- Calling `invalidate()` on every frame when nothing changed
- Using `wrap_content` on `RecyclerView` — measures all items unnecessarily

---

## Related Skills
- `compose-optimization` — Compose-specific recomposition and stability
- `allocation-optimization` — reducing GC pauses during rendering
- `benchmark` — measuring frame rendering with Macrobenchmark
- `anr-prevention` — keeping the main thread free from blocking work
