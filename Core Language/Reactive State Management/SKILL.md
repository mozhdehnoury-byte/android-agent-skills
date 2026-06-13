---
name: reactive-state-management
description: >
  Reactive state management patterns for Android using StateFlow and Flow.
  Load this skill when designing how state is produced, transformed, and
  consumed across ViewModel and UI layers in a reactive, observable way.
---

# Reactive State Management

## Overview

Reactive State Management means that UI state is modeled as an observable stream. The UI never pulls state — it reacts to state emissions. State is always immutable, flows in one direction, and is the single source of truth for what the UI renders.

---

## Core Principles

- State is **immutable** — never mutate, always replace via `copy()`
- State flows **one direction** — ViewModel → UI, never reverse
- UI **observes** state — never reads it on demand
- There is **one state object** per screen — not multiple separate fields
- State is **complete** — the UI can render correctly from state alone

---

## State Modeling

```kotlin
// ✅ Single sealed state per screen
data class UserListUiState(
    val users: List<User> = emptyList(),
    val isLoading: Boolean = false,
    val error: String? = null,
    val isEmpty: Boolean = false
)

// ✅ Use sealed class for mutually exclusive states
sealed class ScreenState<out T> {
    object Loading : ScreenState<Nothing>()
    data class Success<T>(val data: T) : ScreenState<T>()
    data class Error(val message: String) : ScreenState<Nothing>()
}
```

---

## ViewModel — Producing State

```kotlin
class UserListViewModel(
    private val getUsersUseCase: GetUsersUseCase
) : ViewModel() {

    // ✅ Private mutable — public immutable
    private val _state = MutableStateFlow(UserListUiState())
    val state: StateFlow<UserListUiState> = _state.asStateFlow()

    // ✅ One-off events via SharedFlow
    private val _events = MutableSharedFlow<UserListEvent>()
    val events: SharedFlow<UserListEvent> = _events.asSharedFlow()

    init {
        loadUsers()
    }

    fun loadUsers() {
        viewModelScope.launch {
            _state.update { it.copy(isLoading = true, error = null) }
            getUsersUseCase()
                .onSuccess { users ->
                    _state.update {
                        it.copy(
                            isLoading = false,
                            users = users,
                            isEmpty = users.isEmpty()
                        )
                    }
                }
                .onFailure { error ->
                    _state.update {
                        it.copy(isLoading = false, error = error.message)
                    }
                }
        }
    }

    fun onUserClicked(userId: String) {
        viewModelScope.launch {
            _events.emit(UserListEvent.NavigateToDetail(userId))
        }
    }
}

// ✅ Events for one-time side effects
sealed class UserListEvent {
    data class NavigateToDetail(val userId: String) : UserListEvent()
    data class ShowSnackbar(val message: String) : UserListEvent()
}
```

---

## Deriving State from Flow

```kotlin
// ✅ stateIn — derive StateFlow from upstream Flow
class UserListViewModel(repository: UserRepository) : ViewModel() {

    val state: StateFlow<UserListUiState> = repository
        .observeUsers()
        .map { users ->
            UserListUiState(
                users = users,
                isEmpty = users.isEmpty()
            )
        }
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5_000),
            initialValue = UserListUiState(isLoading = true)
        )
}
```

---

## Combining Multiple Sources

```kotlin
// ✅ combine — merge multiple streams into one state
class DashboardViewModel(
    userRepository: UserRepository,
    settingsRepository: SettingsRepository
) : ViewModel() {

    val state: StateFlow<DashboardUiState> = combine(
        userRepository.observeCurrentUser(),
        settingsRepository.observeSettings()
    ) { user, settings ->
        DashboardUiState(
            userName = user.name,
            isDarkMode = settings.isDarkMode
        )
    }.stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(5_000),
        initialValue = DashboardUiState()
    )
}
```

---

## UI — Consuming State

```kotlin
// ✅ Compose — collect state
@Composable
fun UserListScreen(viewModel: UserListViewModel = hiltViewModel()) {
    val state by viewModel.state.collectAsStateWithLifecycle()

    // ✅ Collect one-time events
    val lifecycleOwner = LocalLifecycleOwner.current
    LaunchedEffect(Unit) {
        viewModel.events
            .flowWithLifecycle(lifecycleOwner.lifecycle)
            .collect { event ->
                when (event) {
                    is UserListEvent.NavigateToDetail -> navController.navigate(...)
                    is UserListEvent.ShowSnackbar -> snackbarHostState.showSnackbar(...)
                }
            }
    }

    UserListContent(state = state, onUserClick = viewModel::onUserClicked)
}

// ✅ Fragment — collect with repeatOnLifecycle
viewLifecycleOwner.lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.state.collect { state ->
            renderState(state)
        }
    }
}
```

---

## State vs Events

| Type      | Use for                                    | Implementation |
| --------- | ------------------------------------------ | -------------- |
| **State** | Persistent UI data (list, loading, error)  | `StateFlow`    |
| **Event** | One-time actions (navigate, toast, dialog) | `SharedFlow`   |

```kotlin
// ✅ State — persists, survives recomposition
val isLoading: StateFlow<Boolean>

// ✅ Event — fires once, not replayed
val navigationEvent: SharedFlow<NavigationEvent>

// ❌ Don't model navigation in state — it replays on rotation
data class UiState(val navigateTo: String?)  // wrong
```

---

## Anti-Patterns

- Multiple separate `StateFlow` fields per screen — use one state object
- Exposing `MutableStateFlow` from ViewModel — always expose as `StateFlow`
- Collecting state with `lifecycleScope.launch {}` without `repeatOnLifecycle` — leaks in background
- Modeling one-time events as state — use `SharedFlow` for events
- Calling `stateFlow.value` in UI to read state imperatively — always collect reactively
- Triggering side effects inside `map {}` or `combine {}` — use `onEach` or handle in ViewModel

---

## Related Skills

- `stateflow` — StateFlow internals and patterns
- `sharedflow` — SharedFlow for one-time events
- `reactive-streams` — Flow operators and transformation
- `mvvm` — ViewModel and state ownership
- `side-effect-management` — handling side effects in reactive pipelines
