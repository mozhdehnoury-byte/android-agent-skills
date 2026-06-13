---
name: lifecycle
description: >
  Android Lifecycle management for Activities, Fragments, and Compose.
  Load this skill when working with lifecycle-aware components, collecting
  flows with lifecycle awareness, managing resources tied to lifecycle,
  or avoiding memory leaks caused by lifecycle misuse.
---

# Lifecycle

## Overview

The Android Lifecycle represents the states an Activity, Fragment, or other component goes through from creation to destruction. Lifecycle-awareness ensures that operations start and stop at the right time, preventing memory leaks, crashes, and wasted resources.

---

## Core Principles

- Never hold references to Activity or Fragment in long-lived objects
- Always use lifecycle-aware collection for Flows — never raw `launch {}`
- Release resources in the **symmetric opposite** of where they were acquired
- Prefer `ViewModel` to survive configuration changes — never store UI state in Activity
- Use `repeatOnLifecycle` over `launchWhenStarted` — it properly cancels on stop

---

## Lifecycle States

```
INITIALIZED → CREATED → STARTED → RESUMED
                ↑           ↓
             DESTROYED ← STOPPED
```

| State     | Activity callback | Fragment callback |
| --------- | ----------------- | ----------------- |
| CREATED   | `onCreate()`      | `onViewCreated()` |
| STARTED   | `onStart()`       | `onStart()`       |
| RESUMED   | `onResume()`      | `onResume()`      |
| STOPPED   | `onStop()`        | `onStop()`        |
| DESTROYED | `onDestroy()`     | `onDestroyView()` |

---

## Flow Collection — The Right Way

```kotlin
// ✅ repeatOnLifecycle — cancels collection when below target state
viewLifecycleOwner.lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.state.collect { state ->
            renderState(state)
        }
    }
}

// ✅ Multiple flows — launch inside repeatOnLifecycle
viewLifecycleOwner.lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        launch { viewModel.state.collect { renderState(it) } }
        launch { viewModel.events.collect { handleEvent(it) } }
    }
}

// ✅ Compose — collectAsStateWithLifecycle (preferred over collectAsState)
@Composable
fun MyScreen(viewModel: MyViewModel = hiltViewModel()) {
    val state by viewModel.state.collectAsStateWithLifecycle()
}

// ❌ Wrong — doesn't cancel when app goes to background
lifecycleScope.launch {
    viewModel.state.collect { renderState(it) }
}

// ❌ Wrong — deprecated, doesn't resume collection after stop
lifecycleScope.launchWhenStarted {
    viewModel.state.collect { renderState(it) }
}
```

---

## ViewModel and Lifecycle

```kotlin
// ✅ ViewModel survives configuration changes
class UserViewModel : ViewModel() {
    // State here survives rotation
    private val _state = MutableStateFlow(UserUiState())
    val state: StateFlow<UserUiState> = _state.asStateFlow()

    // ✅ viewModelScope is cancelled when ViewModel is cleared
    fun loadUsers() {
        viewModelScope.launch {
            // safe — auto-cancelled on ViewModel destruction
        }
    }

    // ✅ onCleared — clean up resources
    override fun onCleared() {
        super.onCleared()
        // cancel non-coroutine resources if needed
    }
}

// ✅ Get ViewModel in Fragment
class UserFragment : Fragment() {
    private val viewModel: UserViewModel by viewModels()
    // shared ViewModel across fragments in same Activity
    private val sharedViewModel: SharedViewModel by activityViewModels()
}
```

---

## Lifecycle-Aware Resource Management

```kotlin
// ✅ Register/unregister in symmetric callbacks
class LocationFragment : Fragment() {

    private lateinit var locationManager: LocationManager

    override fun onStart() {
        super.onStart()
        locationManager.startUpdates()  // start in onStart
    }

    override fun onStop() {
        locationManager.stopUpdates()   // stop in onStop
        super.onStop()
    }
}

// ✅ ViewBinding — avoid leaks
class UserFragment : Fragment(R.layout.fragment_user) {

    private var _binding: FragmentUserBinding? = null
    private val binding get() = _binding!!

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        _binding = FragmentUserBinding.bind(view)
    }

    override fun onDestroyView() {
        _binding = null  // ✅ clear binding reference
        super.onDestroyView()
    }
}
```

---

## DefaultLifecycleObserver

```kotlin
// ✅ Use DefaultLifecycleObserver for custom lifecycle-aware components
class AnalyticsTracker(private val analytics: Analytics) : DefaultLifecycleObserver {

    override fun onStart(owner: LifecycleOwner) {
        analytics.startSession()
    }

    override fun onStop(owner: LifecycleOwner) {
        analytics.endSession()
    }
}

// Register in Activity or Fragment
lifecycle.addObserver(AnalyticsTracker(analytics))
```

---

## Compose Lifecycle

```kotlin
// ✅ LaunchedEffect — scoped to composition, cancelled on leave
@Composable
fun UserScreen(userId: String) {
    LaunchedEffect(userId) {
        // runs when userId changes, cancelled when composable leaves
        viewModel.loadUser(userId)
    }
}

// ✅ DisposableEffect — for non-coroutine cleanup
@Composable
fun MapScreen() {
    DisposableEffect(Unit) {
        val listener = registerMapListener()
        onDispose {
            listener.unregister()
        }
    }
}

// ✅ rememberUpdatedState — keep latest value in long-running effect
@Composable
fun Timer(onTick: () -> Unit) {
    val currentOnTick by rememberUpdatedState(onTick)
    LaunchedEffect(Unit) {
        while (true) {
            delay(1_000)
            currentOnTick()
        }
    }
}
```

---

## Anti-Patterns

- Storing Activity/Fragment reference in ViewModel or Repository — causes memory leaks
- Using `lifecycleScope.launch {}` without `repeatOnLifecycle` for Flow collection
- Using `launchWhenStarted` — deprecated, doesn't cancel on stop
- Starting work in `onCreate` that should be in `onStart` — runs even when not visible
- Forgetting to clear ViewBinding in `onDestroyView` — memory leak
- Using `GlobalScope` for any lifecycle-bound work

---

## Related Skills

- `savedstatehandle` — persisting state across process death
- `process-death-recovery` — surviving process kill
- `viewmodel` — state management with ViewModel
- `compose` — Compose-specific lifecycle patterns
- `reactive-streams` — lifecycle-aware Flow collection
