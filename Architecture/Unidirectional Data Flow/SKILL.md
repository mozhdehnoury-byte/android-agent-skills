---
name: unidirectional-data-flow
description: >
  Unidirectional Data Flow (UDF) pattern for Android.
  Load this skill when designing how state flows from ViewModel to UI,
  how events flow from UI to ViewModel, understanding the UDF cycle,
  or enforcing a single source of truth for UI state.
---

# Unidirectional Data Flow (UDF)

## Overview

UDF is the underlying principle behind both MVVM and MVI. State flows **down** from ViewModel to UI; events flow **up** from UI to ViewModel. The UI never mutates state directly — it only emits events. This creates a predictable, traceable, and testable state cycle.

---

## The UDF Cycle

```
      ┌──────────────────────────────┐
      │           ViewModel          │
      │                              │
      │  event → logic → new state   │
      └──────┬───────────────────────┘
             │ State (StateFlow)
             ▼
      ┌──────────────────────────────┐
      │              UI              │
      │                              │
      │  render(state)               │
      │  user interaction → event ──►│
      └──────────────────────────────┘
```

1. **State flows down** — ViewModel emits `StateFlow`, UI observes and renders
2. **Events flow up** — UI emits events (clicks, input), ViewModel processes them
3. **ViewModel owns state** — UI never mutates state; it requests changes via events
4. **Single source of truth** — one `StateFlow` per screen, not scattered variables

---

## Core Principles

- State is **immutable** — always produce a new state, never mutate in place
- UI is a **pure function of state** — same state always produces same UI
- Events are **the only way** to trigger state changes
- ViewModel is the **single source of truth** for UI state
- No two-way data binding that hides event flow

---

## Implementation

```kotlin
// ✅ State flows down via StateFlow
data class SearchUiState(
    val query: String = "",
    val results: List<Product> = emptyList(),
    val isLoading: Boolean = false,
    val error: String? = null
)

// ✅ Events flow up — all triggers modeled explicitly
sealed interface SearchEvent {
    data class QueryChanged(val query: String) : SearchEvent
    data object SearchSubmitted : SearchEvent
    data object ClearSearch : SearchEvent
    data class FilterSelected(val filter: Filter) : SearchEvent
}

@HiltViewModel
class SearchViewModel @Inject constructor(
    private val searchProductsUseCase: SearchProductsUseCase
) : ViewModel() {

    private val _state = MutableStateFlow(SearchUiState())
    val state: StateFlow<SearchUiState> = _state.asStateFlow()

    fun onEvent(event: SearchEvent) {
        when (event) {
            is SearchEvent.QueryChanged    -> _state.update { it.copy(query = event.query) }
            is SearchEvent.SearchSubmitted -> search(_state.value.query)
            is SearchEvent.ClearSearch     -> _state.update { SearchUiState() }
            is SearchEvent.FilterSelected  -> applyFilter(event.filter)
        }
    }

    private fun search(query: String) {
        viewModelScope.launch {
            _state.update { it.copy(isLoading = true, error = null) }
            searchProductsUseCase(query).fold(
                onSuccess = { results ->
                    _state.update { it.copy(isLoading = false, results = results) }
                },
                onFailure = { error ->
                    _state.update { it.copy(isLoading = false, error = error.message) }
                }
            )
        }
    }
}

// ✅ UI observes state and emits events — never modifies state
@Composable
fun SearchScreen(viewModel: SearchViewModel = hiltViewModel()) {
    val state by viewModel.state.collectAsStateWithLifecycle()

    SearchContent(
        state = state,
        onEvent = viewModel::onEvent
    )
}

@Composable
private fun SearchContent(
    state: SearchUiState,
    onEvent: (SearchEvent) -> Unit
) {
    Column {
        OutlinedTextField(
            value = state.query,
            onValueChange = { onEvent(SearchEvent.QueryChanged(it)) },  // ✅ event up
            trailingIcon = {
                IconButton(onClick = { onEvent(SearchEvent.SearchSubmitted) }) {
                    Icon(Icons.Default.Search, contentDescription = null)
                }
            }
        )

        when {
            state.isLoading  -> CircularProgressIndicator()
            state.error != null -> ErrorMessage(state.error)
            else -> ResultList(state.results)
        }
    }
}
```

---

## State Updates

```kotlin
// ✅ Immutable update via copy()
_state.update { currentState ->
    currentState.copy(
        isLoading = false,
        results = newResults
    )
}

// ✅ update{} is atomic — safe for concurrent updates
// update() uses compareAndSet internally — no race conditions

// ❌ Direct mutation — never do this
_state.value.results.add(item)  // breaks UDF — state is mutated, not replaced
```

---

## Derived State

```kotlin
// ✅ Derive additional state from primary state
val filteredUsers: StateFlow<List<User>> = combine(
    _users,
    _searchQuery
) { users, query ->
    if (query.isBlank()) users
    else users.filter { it.name.contains(query, ignoreCase = true) }
}.stateIn(
    scope = viewModelScope,
    started = SharingStarted.WhileSubscribed(5_000),
    initialValue = emptyList()
)
```

---

## Anti-Patterns

- UI mutating ViewModel state directly — UI only emits events
- Two-way data binding (`var` exposed directly) — breaks traceability
- Multiple independent StateFlows for one screen — hard to reason about combined state
- Calling ViewModel functions with side-effect names from UI ("submit", "navigate") — model as events instead
- Mutable collections inside state — use `List`, not `MutableList`

---

## Related Skills

- `mvvm` — MVVM applies UDF with ViewModel + StateFlow
- `mvi` — MVI is a strict formalization of UDF with Intent/State/Effect
- `state-management` — advanced state patterns
- `reactive-state-management` — Flow operators for deriving state
