---
name: channel
description: >
  Kotlin Channel for guaranteed single-consumer message delivery in Android.
  Load this skill when implementing one-time UI events with delivery guarantee,
  building producer-consumer pipelines, handling work queues,
  or choosing between Channel and SharedFlow for event delivery.
---

# Channel

## Overview

`Channel` is a coroutine primitive for communication between coroutines. Unlike `SharedFlow`, a `Channel` guarantees delivery to exactly one consumer and buffers messages when the consumer is not active. On Android, `Channel` is the preferred mechanism for one-time ViewModel events (navigation, toasts) where delivery must be guaranteed.

---

## Core Principles

- Use `Channel` when delivery must be **guaranteed** — events must not be dropped or missed
- Use `SharedFlow` when **broadcasting** to multiple collectors is needed
- Always expose `Channel` as `receiveAsFlow()` — never expose the raw `Channel`
- Use `Channel.BUFFERED` capacity — prevents suspension when collector is temporarily inactive
- One `Channel` per event type, or a sealed class `Channel` for all events from one ViewModel

---

## Basic Channel for UI Events

```kotlin
// ✅ Standard ViewModel event channel
class UserViewModel : ViewModel() {

    private val _events = Channel<UserEvent>(Channel.BUFFERED)
    val events: Flow<UserEvent> = _events.receiveAsFlow()

    fun onSaveSuccess() {
        viewModelScope.launch {
            _events.send(UserEvent.NavigateBack)
        }
    }

    fun onError(message: String) {
        viewModelScope.launch {
            _events.send(UserEvent.ShowSnackbar(message))
        }
    }
}

sealed interface UserEvent {
    data object NavigateBack : UserEvent
    data class ShowSnackbar(val message: String) : UserEvent
    data class NavigateToDetail(val userId: String) : UserEvent
}
```

---

## Collecting in Compose

```kotlin
// ✅ Collect channel events in LaunchedEffect
@Composable
fun UserScreen(
    onNavigateBack: () -> Unit,
    onNavigateToDetail: (String) -> Unit,
    viewModel: UserViewModel = hiltViewModel()
) {
    val snackbarHostState = remember { SnackbarHostState() }

    LaunchedEffect(Unit) {
        viewModel.events.collect { event ->
            when (event) {
                is UserEvent.NavigateBack            -> onNavigateBack()
                is UserEvent.ShowSnackbar            -> snackbarHostState.showSnackbar(event.message)
                is UserEvent.NavigateToDetail        -> onNavigateToDetail(event.userId)
            }
        }
    }

    Scaffold(snackbarHost = { SnackbarHost(snackbarHostState) }) { padding ->
        // screen content
    }
}
```

---

## Channel Capacity Options

```kotlin
// ✅ BUFFERED — default buffer (64 elements), send never suspends unless full
Channel<Event>(Channel.BUFFERED)

// ✅ UNLIMITED — unbounded buffer, send never suspends
Channel<Event>(Channel.UNLIMITED)

// ✅ RENDEZVOUS — no buffer, send suspends until receive
Channel<Event>(Channel.RENDEZVOUS)

// ✅ CONFLATED — keeps only the latest, old values dropped
Channel<Event>(Channel.CONFLATED)

// ✅ Custom capacity
Channel<Event>(capacity = 32)
```

---

## Producer-Consumer Pipeline

```kotlin
// ✅ Work queue with Channel
class ImageProcessingQueue @Inject constructor() {
    private val queue = Channel<ImageTask>(capacity = 32)

    suspend fun enqueue(task: ImageTask) {
        queue.send(task)
    }

    fun startProcessing(scope: CoroutineScope) {
        scope.launch(Dispatchers.Default) {
            for (task in queue) {   // ✅ for loop — processes until channel is closed
                processImage(task)
            }
        }
    }

    fun close() {
        queue.close()
    }
}
```

---

## Channel vs SharedFlow

|               | Channel                 | SharedFlow                     |
| ------------- | ----------------------- | ------------------------------ |
| Consumers     | Single                  | Multiple                       |
| Delivery      | Guaranteed (buffered)   | Not guaranteed (can miss)      |
| Replay        | No                      | Configurable                   |
| Best for      | UI events, work queues  | Broadcast events, status       |
| Missed events | Buffered until consumed | Dropped if no active collector |

---

## trySend vs send

```kotlin
// ✅ send — suspends if buffer full (use inside coroutines)
viewModelScope.launch {
    _events.send(UserEvent.NavigateBack)
}

// ✅ trySend — returns ChannelResult, non-suspending (use outside coroutines)
val result = _events.trySend(UserEvent.NavigateBack)
if (result.isFailure) Timber.w("Event could not be sent — channel full or closed")
```

---

## Closing a Channel

```kotlin
// ✅ Close channel when done producing — consumer loop ends
class DataProducer {
    private val channel = Channel<Data>(Channel.BUFFERED)

    suspend fun produce() {
        try {
            dataList.forEach { channel.send(it) }
        } finally {
            channel.close()  // ✅ always close in finally
        }
    }

    fun consume(scope: CoroutineScope) {
        scope.launch {
            for (data in channel) {  // ends when channel is closed
                process(data)
            }
        }
    }
}
```

---

## Anti-Patterns

- Exposing raw `MutableSharedFlow` or `Channel` — external code can send/close
- Not using `receiveAsFlow()` — exposes Channel API to consumers
- Using `Channel.RENDEZVOUS` for UI events — send suspends if UI isn't collecting
- Using `SharedFlow` when you need guaranteed single delivery — events can be missed
- Forgetting to close the channel in producer — consumer loop hangs forever

---

## Related Skills

- `sharedflow` — broadcast to multiple collectors
- `stateflow` — persistent UI state
- `flow` — cold stream fundamentals
- `coroutine` — structured concurrency and scopes
- `mvvm` — event pattern in ViewModel
