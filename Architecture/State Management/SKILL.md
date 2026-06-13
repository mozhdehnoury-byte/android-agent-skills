---
name: state-management
description: >
  State management patterns for Android with Compose and ViewModel.
  Load this skill when designing UI state models, choosing between
  StateFlow vs SharedFlow vs Channel, managing loading/error/success states,
  handling state persistence, or combining multiple state sources.
---

# State Management

## Overview

State management defines where state lives, how it flows to the UI, and how it changes. In Android with Compose, state lives in the ViewModel as `StateFlow`, flows to composables via `collectAsStateWithLifecycle()`, and changes only through explicit events or function calls. One-time events (navigation, toasts) use `Channel` — not `StateFlow`.

---

## Core Principles

- **Single source of truth** — one `StateFlow<UiState>` per screen
- **Immutable state** — always produce a new state via `copy()`, never mutate
- **StateFlow for persistent state** — last value replayed to new collectors
- **Channel for one-time events** — not replayed, not missed when buffered
- **`collectAsStateWithLifecycle()`** — lifecycle-aware collection in Compose

---

## State Shape Patterns

```kotlin
// ✅ Pattern 1: Sealed class — for screens with distinct modes
sealed interface ProductDetailUiState {
    data object Loading : ProductDetailUiState
    data class Success(val product: Product, val isFavorite: Boolean) : ProductDetailUiState
    data class Error(val message: String) : ProductDetailUiState
}

// ✅ Pattern 2: Data class — for screens with parallel state fields
data class ProductListUiState(
    val isLoading: Boolean = false,
    val products: List<Product> = emptyList(),
    val error: String? = null,
    val searchQuery: String = "",
    val selectedFilter: Filter = Filter.All,
    val totalCount: Int = 0
)

// When to use which:
// Sealed class → screen has mutually exclusive modes (loading OR content OR error)
// Data class → screen has multiple independent fields shown simultaneously
```

---

## StateFlow Patterns

```kotlin
// ✅ Simple StateFlow
private val _state = MutableStateFlow(ProductListUiState())
val state: StateFlow<ProductListUiState> = _state.asStateFlow()

// Update with copy()
_state.update { it.copy(isLoading = true) }

// ✅ StateFlow from Flow (derived state)
val state: StateFlow<ProductListUiState> =
    repository.observeProducts()
        .map { products -> ProductListUiState(products = products) }
        .catch { e -> emit(ProductListUiState(error = e.message)) }
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5_000),  // cancel 5s after UI gone
            initialValue = ProductListUiState(isLoading = true)
        )

// ✅ Combining multiple sources
val state: StateFlow<DashboardUiState> = combine(
    userRepository.observeCurrentUser(),
    orderRepository.observeRecentOrders(),
    notificationRepository.observeUnreadCount()
) { user, orders, unreadCount ->
    DashboardUiState(
        userName = user.name,
        recentOrders = orders,
        unreadNotifications = unreadCount
    )
}.stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), DashboardUiState())
```

---

## One-Time Events: Channel

```kotlin
// ✅ Use Channel for events that happen exactly once
sealed interface ProductEvent {
    data object NavigateToCart : ProductEvent
    data class ShowSnackbar(val message: String) : ProductEvent
    data class NavigateToDetail(val productId: String) : ProductEvent
}

private val _events = Channel<ProductEvent>(Channel.BUFFERED)
val events: Flow<ProductEvent> = _events.receiveAsFlow()

fun onAddToCart(product: Product) {
    viewModelScope.launch {
        addToCartUseCase(product).fold(
            onSuccess = { _events.send(ProductEvent.ShowSnackbar("Added to cart")) },
            onFailure = { _events.send(ProductEvent.ShowSnackbar("Failed to add")) }
        )
    }
}

// ✅ Collect events in Compose
LaunchedEffect(Unit) {
    viewModel.events.collect { event ->
        when (event) {
            is ProductEvent.NavigateToCart -> navController.navigate(CartRoute)
            is ProductEvent.ShowSnackbar -> snackbarHost.showSnackbar(event.message)
            is ProductEvent.NavigateToDetail -> navController.navigate(DetailRoute(event.productId))
        }
    }
}
```

---

## StateFlow vs SharedFlow vs Channel

|                     | StateFlow  | SharedFlow   | Channel            |
| ------------------- | ---------- | ------------ | ------------------ |
| Replays last value  | ✅ Always   | Configurable | ❌ Never            |
| Has initial value   | ✅ Required | ❌ No         | ❌ No               |
| Multiple collectors | ✅          | ✅            | ❌ Single consumer  |
| Use for             | UI state   | Event bus    | One-time UI events |

```kotlin
// ✅ StateFlow — UI state (always has a value, replays to new subscribers)
val uiState: StateFlow<MyState>

// ✅ Channel — one-time events (navigation, toast) — won't miss events when buffered
val events: Flow<MyEvent> = _channel.receiveAsFlow()

// ✅ SharedFlow — multi-cast events to multiple collectors (rare in UI layer)
val sharedEvents = MutableSharedFlow<Event>(extraBufferCapacity = 1)
```

---

## Loading / Error / Success

```kotlin
// ✅ Async operation pattern
fun loadProduct(id: String) {
    viewModelScope.launch {
        _state.update { it.copy(isLoading = true, error = null) }
        getProductUseCase(id).fold(
            onSuccess = { product ->
                _state.update { it.copy(isLoading = false, product = product) }
            },
            onFailure = { error ->
                _state.update { it.copy(isLoading = false, error = error.message ?: "Unknown error") }
            }
        )
    }
}

// ✅ Collecting in Compose
@Composable
fun ProductScreen(viewModel: ProductViewModel = hiltViewModel()) {
    val state by viewModel.state.collectAsStateWithLifecycle()

    when {
        state.isLoading  -> LoadingIndicator()
        state.error != null -> ErrorView(state.error, onRetry = viewModel::retry)
        state.product != null -> ProductDetail(state.product)
    }
}
```

---

## Persisted State

```kotlin
// ✅ Persist UI state across process death via SavedStateHandle
@HiltViewModel
class SearchViewModel @Inject constructor(
    savedStateHandle: SavedStateHandle,
    private val searchUseCase: SearchUseCase
) : ViewModel() {

    // ✅ Query persists across process death
    var searchQuery by savedStateHandle.saveable { mutableStateOf("") }
        private set

    // ✅ StateFlow backed by SavedStateHandle
    private val _query = savedStateHandle.getStateFlow("query", "")
    val results: StateFlow<List<Result>> = _query
        .debounce(300)
        .flatMapLatest { searchUseCase(it) }
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), emptyList())

    fun onQueryChanged(query: String) {
        savedStateHandle["query"] = query
    }
}
```

---

## Anti-Patterns

- Exposing `MutableStateFlow` publicly — always expose as `StateFlow`
- Using `StateFlow` for navigation events — they replay on resubscription (double navigation)
- Multiple `StateFlow` for one screen — combine into one state class
- Mutating state directly (`_state.value.list.add(...)`) — use `copy()` and `update()`
- `collectAsState()` instead of `collectAsStateWithLifecycle()` — leaks collection in background

---

## Related Skills

- `mvvm` — ViewModel pattern using state management
- `mvi` — MVI pattern with strict state management
- `unidirectional-data-flow` — the underlying principle
- `reactive-state-management` — Flow operators for state derivation
- `savedstatehandle` — persisting state across process death
- `side-effect-management` — managing effects alongside state
