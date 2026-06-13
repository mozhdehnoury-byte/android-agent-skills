---
name: side-effect-management
description: >
  Side effect handling patterns in Android/Kotlin across all layers.
  Load this skill when dealing with one-time events, navigation triggers,
  analytics calls, toast/snackbar messages, or any operation that causes
  observable change outside the current scope.
---

# Side Effect Management

## Overview

A side effect is any operation that changes state or causes observable behavior outside the current function's scope — navigation, showing a toast, logging, analytics, network calls triggered by UI events. Managing side effects correctly prevents duplication, memory leaks, and lifecycle bugs.

---

## Core Principles

- Side effects must be **lifecycle-aware** — never fire after the UI is gone
- **One-time events** (navigation, snackbar) must never be modeled as persistent state
- Side effects in ViewModel are communicated to UI via **SharedFlow events**
- Side effects in Compose use `LaunchedEffect`, `SideEffect`, or `DisposableEffect`
- Never trigger side effects inside `map {}`, `combine {}`, or composable body directly

---

## Layer Responsibility

| Layer        | Side Effects                                | How                                    |
| ------------ | ------------------------------------------- | -------------------------------------- |
| UI (Compose) | Navigation, show dialog, request permission | `LaunchedEffect`, collect `SharedFlow` |
| ViewModel    | Emit one-time events to UI                  | `MutableSharedFlow`                    |
| UseCase      | None — pure logic only                      | —                                      |
| Repository   | Logging, cache invalidation                 | Contained within repository            |
| Data Source  | Network call, DB write                      | Result returned, not side-effected     |

---

## ViewModel — One-Time Events via SharedFlow

```kotlin
// ✅ Sealed class for typed events
sealed class UserDetailEvent {
    data class NavigateToEdit(val userId: String) : UserDetailEvent()
    data class ShowSnackbar(val message: String) : UserDetailEvent()
    object NavigateBack : UserDetailEvent()
    object ShowDeleteConfirmation : UserDetailEvent()
}

@HiltViewModel
class UserDetailViewModel @Inject constructor(
    private val repository: UserRepository
) : ViewModel() {

    private val _state = MutableStateFlow(UserDetailUiState())
    val state: StateFlow<UserDetailUiState> = _state.asStateFlow()

    // ✅ SharedFlow for events — replay = 0, never persisted
    private val _events = MutableSharedFlow<UserDetailEvent>()
    val events: SharedFlow<UserDetailEvent> = _events.asSharedFlow()

    fun onEditClicked() {
        viewModelScope.launch {
            _events.emit(UserDetailEvent.NavigateToEdit(state.value.userId))
        }
    }

    fun onDeleteClicked() {
        viewModelScope.launch {
            _events.emit(UserDetailEvent.ShowDeleteConfirmation)
        }
    }

    fun onDeleteConfirmed() {
        viewModelScope.launch {
            repository.deleteUser(state.value.userId)
                .onSuccess {
                    _events.emit(UserDetailEvent.NavigateBack)
                }
                .onFailure { error ->
                    _events.emit(UserDetailEvent.ShowSnackbar(error.message ?: "Error"))
                }
        }
    }
}
```

---

## UI — Collecting Events

```kotlin
@Composable
fun UserDetailScreen(
    navController: NavController,
    viewModel: UserDetailViewModel = hiltViewModel()
) {
    val state by viewModel.state.collectAsStateWithLifecycle()
    val snackbarHostState = remember { SnackbarHostState() }
    val lifecycleOwner = LocalLifecycleOwner.current

    // ✅ Collect one-time events with lifecycle awareness
    LaunchedEffect(Unit) {
        viewModel.events
            .flowWithLifecycle(lifecycleOwner.lifecycle, Lifecycle.State.STARTED)
            .collect { event ->
                when (event) {
                    is UserDetailEvent.NavigateToEdit ->
                        navController.navigate(EditUserRoute(event.userId))
                    is UserDetailEvent.ShowSnackbar ->
                        snackbarHostState.showSnackbar(event.message)
                    is UserDetailEvent.NavigateBack ->
                        navController.navigateUp()
                    is UserDetailEvent.ShowDeleteConfirmation ->
                        // show dialog via local state
                        showDeleteDialog = true
                }
            }
    }

    Scaffold(snackbarHost = { SnackbarHost(snackbarHostState) }) { padding ->
        UserDetailContent(
            state = state,
            onEditClick = viewModel::onEditClicked,
            onDeleteClick = viewModel::onDeleteClicked,
            modifier = Modifier.padding(padding)
        )
    }
}
```

---

## Compose Side Effects

```kotlin
// ✅ LaunchedEffect — coroutine side effect, runs when key changes
@Composable
fun AutoScrollList(scrollToIndex: Int) {
    val listState = rememberLazyListState()

    LaunchedEffect(scrollToIndex) {
        listState.animateScrollToItem(scrollToIndex)
    }
}

// ✅ SideEffect — sync Compose state to external system on every recomposition
@Composable
fun AnalyticsTracker(screenName: String) {
    SideEffect {
        analytics.setCurrentScreen(screenName)
    }
}

// ✅ DisposableEffect — register/unregister listener
@Composable
fun NetworkMonitor(onNetworkChange: (Boolean) -> Unit) {
    val context = LocalContext.current
    DisposableEffect(Unit) {
        val callback = object : ConnectivityManager.NetworkCallback() {
            override fun onAvailable(network: Network) = onNetworkChange(true)
            override fun onLost(network: Network) = onNetworkChange(false)
        }
        val cm = context.getSystemService(ConnectivityManager::class.java)
        cm.registerDefaultNetworkCallback(callback)
        onDispose { cm.unregisterNetworkCallback(callback) }
    }
}

// ✅ rememberCoroutineScope — imperative trigger from event handler
@Composable
fun SaveButton(viewModel: FormViewModel) {
    val scope = rememberCoroutineScope()
    val snackbarHostState = remember { SnackbarHostState() }

    Button(onClick = {
        scope.launch {
            snackbarHostState.showSnackbar("Saved!")
        }
    }) {
        Text("Save")
    }
}
```

---

## Analytics as Side Effect

```kotlin
// ✅ Analytics in ViewModel — after state change, before emitting event
fun onPurchaseCompleted(orderId: String) {
    viewModelScope.launch {
        val result = repository.completeOrder(orderId)
        result.onSuccess {
            analytics.track("purchase_completed", mapOf("order_id" to orderId))
            _events.emit(OrderEvent.NavigateToConfirmation(orderId))
        }
    }
}

// ❌ Analytics inside UseCase — use cases must be pure
class CompleteOrderUseCase {
    operator fun invoke(orderId: String) {
        // ❌ Don't call analytics here — side effect in wrong layer
        analytics.track("purchase_completed")
    }
}
```

---

## Side Effects in Flow Pipelines

```kotlin
// ✅ onEach — non-transforming side effect in flow
fun observeOrders(): Flow<List<Order>> =
    repository.observeOrders()
        .onEach { orders ->
            if (orders.any { it.isUrgent }) {
                notificationManager.showUrgentOrderAlert()
            }
        }

// ✅ onStart / onCompletion — lifecycle side effects
fun fetchUsers(): Flow<List<User>> =
    repository.fetchUsers()
        .onStart { analytics.track("users_fetch_started") }
        .onCompletion { error ->
            if (error == null) analytics.track("users_fetch_success")
        }
        .catch { error ->
            logger.error("Failed to fetch users", error)
            emit(emptyList())
        }
```

---

## Preventing Duplicate Events

```kotlin
// ✅ SharedFlow with replay=0 — events are NOT replayed on new collectors
private val _events = MutableSharedFlow<Event>(
    replay = 0,
    extraBufferCapacity = 1,
    onBufferOverflow = BufferOverflow.DROP_OLDEST
)

// ✅ Channel for guaranteed delivery (but only one collector)
private val _events = Channel<Event>(Channel.BUFFERED)
val events = _events.receiveAsFlow()

// ❌ StateFlow for events — replays last event on rotation
private val _navigateTo = MutableStateFlow<Route?>(null)  // wrong — replays on rotation
```

---

## Anti-Patterns

- Modeling navigation/snackbar as state (`UiState.navigateTo`) — replays on rotation
- Calling `analytics.track()` directly inside a composable body
- Triggering side effects inside `map {}` / `combine {}` operators
- Using `GlobalScope` for event emission — not lifecycle-aware
- Forgetting `flowWithLifecycle` when collecting events — fires in background
- Using `replay = 1` on event SharedFlow — event fires again on re-subscription

---

## Related Skills

- `reactive-state-management` — state vs events distinction
- `compose` — LaunchedEffect, SideEffect, DisposableEffect
- `coroutine` — scope and dispatcher for side effects
- `mvvm` — ViewModel event emission patterns
- `use-case-design` — keeping use cases free of side effects
