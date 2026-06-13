---
name: stateflow
description: >
  StateFlow for UI state management in Android with Kotlin coroutines.
  Load this skill when holding and exposing UI state from a ViewModel,
  converting cold flows to StateFlow, managing state updates safely,
  or replacing LiveData with StateFlow.
---

# StateFlow

## Overview
`StateFlow` is a hot Flow that always holds a current value and replays it to new collectors. It is the standard replacement for `LiveData` in Kotlin-first Android development. ViewModels expose `StateFlow` for UI state, and Compose collects it with `collectAsStateWithLifecycle`.

---

## Core Principles

- Every ViewModel exposes **one** `StateFlow<UiState>` per screen — not multiple separate flows
- `MutableStateFlow` is private — expose as `StateFlow` to prevent external mutation
- Update state atomically with `update {}` — never read-then-write separately
- Use `stateIn` to convert a cold `Flow` from the repository into a `StateFlow`
- `StateFlow` replays the current value — do not use it for one-time events

---

## Basic StateFlow

```kotlin
// ✅ Private mutable, public immutable
class UserViewModel : ViewModel() {

    private val _state = MutableStateFlow<UserUiState>(UserUiState.Loading)
    val state: StateFlow<UserUiState> = _state.asStateFlow()

    fun loadUser(id: String) {
        viewModelScope.launch {
            _state.value = UserUiState.Loading
            repository.getUser(id).fold(
                onSuccess = { _state.value = UserUiState.Success(it) },
                onFailure = { _state.value = UserUiState.Error(it.message ?: "Error") }
            )
        }
    }
}
```

---

## Atomic State Updates

```kotlin
// ✅ update {} — thread-safe atomic update
private val _state = MutableStateFlow(UserListState())

fun onSearchQueryChanged(query: String) {
    _state.update { current ->
        current.copy(searchQuery = query)
    }
}

fun onUserDeleted(userId: String) {
    _state.update { current ->
        current.copy(users = current.users.filter { it.id != userId })
    }
}

// ❌ Non-atomic — race condition possible
_state.value = _state.value.copy(searchQuery = query)
```

---

## stateIn — Convert Repository Flow to StateFlow

```kotlin
// ✅ Convert cold Flow from Room/repository to StateFlow
class UserListViewModel @Inject constructor(
    private val userRepository: UserRepository
) : ViewModel() {

    val state: StateFlow<UserListUiState> = userRepository
        .getUsersFlow()
        .map { users -> UserListUiState(users = users) }
        .catch { emit(UserListUiState(error = it.message)) }
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5_000),  // ✅ stop 5s after last collector
            initialValue = UserListUiState(isLoading = true)
        )
}
```

---

## SharingStarted Options

```kotlin
// ✅ WhileSubscribed(5_000) — recommended for UI state
// Stops upstream 5s after last collector unsubscribes
// Survives configuration changes (Activity recreated within 5s)
.stateIn(
    scope = viewModelScope,
    started = SharingStarted.WhileSubscribed(5_000),
    initialValue = initialState
)

// ✅ Eagerly — starts immediately, never stops
// Use for state that must always be fresh
.stateIn(scope = viewModelScope, started = SharingStarted.Eagerly, initialValue = initialState)

// ✅ Lazily — starts on first collector, never stops
.stateIn(scope = viewModelScope, started = SharingStarted.Lazily, initialValue = initialState)
```

---

## Collecting in Compose

```kotlin
// ✅ collectAsStateWithLifecycle — stops collection when lifecycle is below STARTED
@Composable
fun UserListScreen(viewModel: UserListViewModel = hiltViewModel()) {
    val state by viewModel.state.collectAsStateWithLifecycle()

    when (state) {
        is UserListUiState.Loading -> LoadingIndicator()
        is UserListUiState.Success -> UserList(state.users)
        is UserListUiState.Error   -> ErrorMessage(state.message)
    }
}

// ❌ collectAsState — doesn't respect lifecycle
val state by viewModel.state.collectAsState()
```

---

## Derived State

```kotlin
// ✅ Derive secondary StateFlow from primary
class UserListViewModel : ViewModel() {
    private val _state = MutableStateFlow(UserListState())

    val filteredUsers: StateFlow<List<User>> = _state
        .map { state ->
            state.users.filter { it.name.contains(state.searchQuery, ignoreCase = true) }
        }
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5_000),
            initialValue = emptyList()
        )
}
```

---

## Anti-Patterns

- Using `StateFlow` for one-time events (navigation, toasts) — replays on resubscription
- Exposing `MutableStateFlow` publicly — external code can mutate state
- Non-atomic state updates with `_state.value = _state.value.copy(...)` — race conditions
- Using `SharingStarted.Eagerly` for all flows — keeps upstream alive unnecessarily
- Multiple `StateFlow` properties for one screen — use one state data class

---

## Related Skills
- `sharedflow` — for one-time events that must not replay
- `flow` — cold stream fundamentals and operators
- `mvvm` — ViewModel pattern using StateFlow
- `mvi` — reducer pattern with StateFlow
- `lifecycle` — lifecycle-aware collection
