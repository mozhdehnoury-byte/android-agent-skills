---
name: sharedflow
description: >
  SharedFlow for hot event streams in Android with Kotlin coroutines.
  Load this skill when broadcasting events to multiple collectors,
  implementing one-time UI events (navigation, toasts, snackbars),
  sharing a stream between multiple observers, or replacing LiveData events.
---

# SharedFlow

## Overview

`SharedFlow` is a hot Flow that broadcasts values to all active collectors. Unlike `StateFlow`, it has no initial value and does not replay the last value by default. It is the correct tool for one-time events (navigation, toasts) and shared event buses.

---

## Core Principles

- Use `SharedFlow` for **events** — things that happen once and don't have a current value
- Use `StateFlow` for **state** — things that have a current value that persists
- Default `replay = 0` is correct for one-time events — never replay navigation events
- Use `Channel` as an alternative for guaranteed delivery when the collector may not be active
- `MutableSharedFlow` is the writable version — expose as `SharedFlow` to consumers

---

## Basic SharedFlow

```kotlin
// ✅ One-time event stream
class UserViewModel : ViewModel() {

    private val _events = MutableSharedFlow<UserEvent>()
    val events: SharedFlow<UserEvent> = _events.asSharedFlow()

    fun onDeleteSuccess() {
        viewModelScope.launch {
            _events.emit(UserEvent.NavigateBack)
        }
    }

    fun onError(message: String) {
        viewModelScope.launch {
            _events.emit(UserEvent.ShowToast(message))
        }
    }
}

sealed interface UserEvent {
    data object NavigateBack : UserEvent
    data class ShowToast(val message: String) : UserEvent
}
```

---

## Collecting SharedFlow in Compose

```kotlin
// ✅ Collect one-time events in LaunchedEffect
@Composable
fun UserScreen(
    onNavigateBack: () -> Unit,
    viewModel: UserViewModel = hiltViewModel()
) {
    val context = LocalContext.current

    LaunchedEffect(Unit) {
        viewModel.events.collect { event ->
            when (event) {
                is UserEvent.NavigateBack -> onNavigateBack()
                is UserEvent.ShowToast   -> Toast.makeText(context, event.message, Toast.LENGTH_SHORT).show()
            }
        }
    }
}
```

---

## Replay and Buffer Configuration

```kotlin
// ✅ replay = 0 — no replay, new collectors miss past events (correct for one-time events)
val events = MutableSharedFlow<Event>(replay = 0)

// ✅ replay = 1 — new collectors get the last event (use for "current status" broadcasts)
val connectionStatus = MutableSharedFlow<ConnectionStatus>(replay = 1)

// ✅ extraBufferCapacity — don't drop events when collectors are slow
val events = MutableSharedFlow<Event>(
    replay = 0,
    extraBufferCapacity = 64,
    onBufferOverflow = BufferOverflow.DROP_OLDEST
)
```

---

## Shared Event Bus

```kotlin
// ✅ App-wide event bus for cross-feature communication
@Singleton
class AppEventBus @Inject constructor() {

    private val _events = MutableSharedFlow<AppEvent>(
        extraBufferCapacity = 32,
        onBufferOverflow = BufferOverflow.DROP_OLDEST
    )
    val events: SharedFlow<AppEvent> = _events.asSharedFlow()

    suspend fun emit(event: AppEvent) {
        _events.emit(event)
    }
}

sealed interface AppEvent {
    data object UserLoggedOut : AppEvent
    data class PushNotificationReceived(val data: Map<String, String>) : AppEvent
}
```

---

## SharedFlow vs Channel vs StateFlow

|                     | SharedFlow       | Channel                    | StateFlow  |
| ------------------- | ---------------- | -------------------------- | ---------- |
| Hot/Cold            | Hot              | Hot                        | Hot        |
| Initial value       | No               | No                         | Yes        |
| Replay              | Configurable     | No                         | Last value |
| Multiple collectors | Yes              | No (one consumer)          | Yes        |
| Delivery guarantee  | No (can miss)    | Yes (buffered)             | N/A        |
| Use for             | Broadcast events | Guaranteed single delivery | UI state   |

---

## tryEmit vs emit

```kotlin
// ✅ emit — suspends if buffer is full (use in coroutines)
viewModelScope.launch {
    _events.emit(UserEvent.NavigateBack)
}

// ✅ tryEmit — returns false if buffer full (use outside coroutines)
val sent = _events.tryEmit(UserEvent.NavigateBack)
if (!sent) Timber.w("Event dropped — buffer full")
```

---

## Anti-Patterns

- Using `SharedFlow` with `replay = 1` for navigation events — navigates again on resubscription
- Using `StateFlow` for one-time events — replays the event on recomposition
- Not setting `extraBufferCapacity` for high-frequency events — events dropped silently
- Collecting `SharedFlow` without lifecycle awareness in Android UI
- Using an app-wide event bus for everything — creates hidden coupling between features

---

## Related Skills

- `stateflow` — hot state holder with current value
- `channel` — guaranteed single-consumer delivery
- `flow` — cold stream fundamentals
- `coroutine` — coroutine scopes for emitting events
- `mvvm` — event pattern in ViewModel
