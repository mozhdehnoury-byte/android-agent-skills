---
name: memory-leak-prevention
description: >
  Memory leak detection and prevention in Android.
  Load this skill when investigating memory leaks, auditing code for
  common leak patterns, using LeakCanary, fixing Context leaks,
  or ensuring proper cleanup of listeners and callbacks.
---

# Memory Leak Prevention

## Overview
A memory leak occurs when an object is no longer needed but cannot be garbage collected because something still holds a reference to it. On Android, the most common leaks involve holding a reference to an Activity or Fragment beyond their lifecycle. LeakCanary is the standard tool for detection.

---

## Core Principles

- Never store `Activity` or `Fragment` references in long-lived objects (ViewModel, singleton)
- Always unregister listeners, observers, and callbacks in the symmetric lifecycle callback
- Use `applicationContext` in singletons — not `Activity` context
- Clear ViewBinding references in `onDestroyView`
- Use `WeakReference` only as a last resort — fix the design instead

---

## LeakCanary Setup

```toml
# libs.versions.toml
[libraries]
leakcanary = { module = "com.squareup.leakcanary:leakcanary-android", version = "2.14" }
```

```kotlin
// build.gradle.kts — debug only
dependencies {
    debugImplementation(libs.leakcanary)
}
// No code needed — LeakCanary auto-installs via ContentProvider
```

---

## Common Leak Patterns and Fixes

```kotlin
// ❌ Storing Activity in a singleton — leak
object ImageLoader {
    var context: Context? = null  // holds Activity reference
}

// ✅ Use ApplicationContext
object ImageLoader {
    lateinit var appContext: Context

    fun init(context: Context) {
        appContext = context.applicationContext  // safe
    }
}

// ❌ Anonymous listener holding Activity reference
class UserActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        someManager.setListener(object : SomeListener {
            override fun onEvent() {
                updateUi()  // captures Activity implicitly
            }
        })
        // listener never removed — Activity leaked
    }
}

// ✅ Remove listener in onDestroy
class UserActivity : AppCompatActivity() {
    private val listener = SomeListener { updateUi() }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        someManager.addListener(listener)
    }

    override fun onDestroy() {
        someManager.removeListener(listener)
        super.onDestroy()
    }
}
```

---

## ViewBinding Leak

```kotlin
// ❌ Binding held past onDestroyView — Fragment leak
class UserFragment : Fragment() {
    private lateinit var binding: FragmentUserBinding  // never cleared

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        binding = FragmentUserBinding.bind(view)
    }
    // Fragment view destroyed but binding still holds reference to views
}

// ✅ Clear binding in onDestroyView
class UserFragment : Fragment() {
    private var _binding: FragmentUserBinding? = null
    private val binding get() = _binding!!

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        _binding = FragmentUserBinding.bind(view)
    }

    override fun onDestroyView() {
        _binding = null  // ✅ clear reference
        super.onDestroyView()
    }
}
```

---

## Coroutine Scope Leak

```kotlin
// ❌ Custom scope not cancelled — coroutine leak
class MyManager {
    private val scope = CoroutineScope(Dispatchers.IO)

    fun start() {
        scope.launch { doWork() }
    }
    // scope never cancelled — coroutines run forever
}

// ✅ Cancel scope when no longer needed
class MyManager {
    private val scope = CoroutineScope(SupervisorJob() + Dispatchers.IO)

    fun start() {
        scope.launch { doWork() }
    }

    fun stop() {
        scope.cancel()  // ✅ cancels all children
    }
}
```

---

## Handler and Runnable Leak

```kotlin
// ❌ Handler posting to Activity that may be destroyed
class UserActivity : AppCompatActivity() {
    private val handler = Handler(Looper.getMainLooper())

    override fun onStart() {
        super.onStart()
        handler.postDelayed({ updateUi() }, 5_000)  // captures Activity
    }
    // runnable runs even after Activity destroyed
}

// ✅ Cancel pending callbacks in onStop
class UserActivity : AppCompatActivity() {
    private val handler = Handler(Looper.getMainLooper())
    private val updateRunnable = Runnable { updateUi() }

    override fun onStart() {
        super.onStart()
        handler.postDelayed(updateRunnable, 5_000)
    }

    override fun onStop() {
        handler.removeCallbacks(updateRunnable)  // ✅
        super.onStop()
    }
}
```

---

## ViewModel Holding View Reference

```kotlin
// ❌ ViewModel holding Composable lambda or View reference
class UserViewModel : ViewModel() {
    var onSuccess: (() -> Unit)? = null  // holds UI reference — leak
}

// ✅ Use events via Channel or SharedFlow — no direct reference
class UserViewModel : ViewModel() {
    private val _events = Channel<UserEvent>(Channel.BUFFERED)
    val events = _events.receiveAsFlow()
}
```

---

## Anti-Patterns

- Passing `Activity` context to a singleton or ViewModel
- Not removing `BroadcastReceiver` registered dynamically
- Not unregistering `SensorManager` or `LocationManager` listeners
- Storing references to Composable lambdas in ViewModel
- Using `registerForActivityResult` outside of `onCreate` or composable

---

## Related Skills
- `lifecycle` — symmetric resource acquisition and release
- `viewmodel` — ViewModel scope and what it should not hold
- `coroutine` — scope cancellation to prevent coroutine leaks
- `compose` — DisposableEffect for Compose-specific cleanup
