---
name: mvvm
description: >
  Model-View-ViewModel pattern for Android with Jetpack Compose.
  Load this skill when structuring a screen with ViewModel, defining
  UI state, handling user events, managing one-time side effects,
  or wiring ViewModel to Compose UI.
---

# MVVM

## Overview

MVVM separates UI (View) from business logic (ViewModel) via observable state. In Compose, the ViewModel exposes `StateFlow` for UI state and a `Channel` or `SharedFlow` for one-time events. The composable observes state and delegates all actions to the ViewModel.

---

## Core Principles

- ViewModel exposes **one UI state** as `StateFlow` — never multiple disconnected flags
- One-time events (navigation, toasts) go through a `Channel` — not `StateFlow`
- Composables are **stateless** — they receive state and emit events only
- ViewModel has **no reference** to View, Context, or composable
- User actions are modeled as a sealed `UiEvent` class — not individual functions for simple screens

---

## UI State Model

```kotlin
// ✅ Single sealed state per screen
sealed interface UserDetailUiState {
    data object Loading : UserDetailUiState
    data class Success(val user: User) : UserDetailUiState
    data class Error(val message: String) : UserDetailUiState
}

// ✅ Or data class for screens with multiple fields
data class UserListUiState(
    val isLoading: Boolean = false,
    val users: List<User> = emptyList(),
    val error: String? = null,
    val searchQuery: String = ""
)
```

---

## One-Time Events

```kotlin
// ✅ Events that happen once — not persistent state
sealed interface UserDetailEvent {
    data object NavigateBack : UserDetailEvent
    data class ShowToast(val message: String) : UserDetailEvent
    data class NavigateToEdit(val userId: String) : UserDetailEvent
}
```

---

## ViewModel

```kotlin
// ✅ Standard ViewModel structure
@HiltViewModel
class UserDetailViewModel @Inject constructor(
    savedStateHandle: SavedStateHandle,
    private val getUserUseCase: GetUserUseCase,
    private val deleteUserUseCase: DeleteUserUseCase
) : ViewModel() {

    private val userId: String = checkNotNull(savedStateHandle["userId"])

    // UI state
    private val _state = MutableStateFlow<UserDetailUiState>(UserDetailUiState.Loading)
    val state: StateFlow<UserDetailUiState> = _state.asStateFlow()

    // One-time events
    private val _events = Channel<UserDetailEvent>(Channel.BUFFERED)
    val events: Flow<UserDetailEvent> = _events.receiveAsFlow()

    init {
        loadUser()
    }

    fun loadUser() {
        viewModelScope.launch {
            _state.value = UserDetailUiState.Loading
            getUserUseCase(userId).fold(
                onSuccess = { _state.value = UserDetailUiState.Success(it) },
                onFailure = { _state.value = UserDetailUiState.Error(it.message ?: "Error") }
            )
        }
    }

    fun onDeleteClick() {
        viewModelScope.launch {
            deleteUserUseCase(userId).fold(
                onSuccess = { _events.send(UserDetailEvent.NavigateBack) },
                onFailure = { _events.send(UserDetailEvent.ShowToast("Delete failed")) }
            )
        }
    }

    fun onEditClick() {
        viewModelScope.launch {
            _events.send(UserDetailEvent.NavigateToEdit(userId))
        }
    }
}
```

---

## Composable

```kotlin
// ✅ Stateless composable — observes state, delegates actions
@Composable
fun UserDetailScreen(
    onNavigateBack: () -> Unit,
    onNavigateToEdit: (String) -> Unit,
    viewModel: UserDetailViewModel = hiltViewModel()
) {
    val state by viewModel.state.collectAsStateWithLifecycle()
    val context = LocalContext.current

    // ✅ Collect one-time events
    LaunchedEffect(Unit) {
        viewModel.events.collect { event ->
            when (event) {
                is UserDetailEvent.NavigateBack -> onNavigateBack()
                is UserDetailEvent.ShowToast -> Toast.makeText(context, event.message, Toast.LENGTH_SHORT).show()
                is UserDetailEvent.NavigateToEdit -> onNavigateToEdit(event.userId)
            }
        }
    }

    UserDetailContent(
        state = state,
        onDeleteClick = viewModel::onDeleteClick,
        onEditClick = viewModel::onEditClick,
        onRetryClick = viewModel::loadUser
    )
}

// ✅ Pure content composable — no ViewModel reference
@Composable
private fun UserDetailContent(
    state: UserDetailUiState,
    onDeleteClick: () -> Unit,
    onEditClick: () -> Unit,
    onRetryClick: () -> Unit
) {
    when (state) {
        is UserDetailUiState.Loading -> LoadingState()
        is UserDetailUiState.Success -> UserDetailBody(state.user, onDeleteClick, onEditClick)
        is UserDetailUiState.Error   -> ErrorState(state.message, onRetryClick)
    }
}
```

---

## Shared ViewModel (Between Screens)

```kotlin
// ✅ Shared ViewModel scoped to NavBackStackEntry
@Composable
fun ParentScreen(navController: NavController) {
    val parentEntry = remember(navController) {
        navController.getBackStackEntry(ParentRoute)
    }
    val sharedViewModel: SharedViewModel = hiltViewModel(parentEntry)
}
```

---

## Anti-Patterns

- Multiple `StateFlow` properties for one screen — use a single state class
- Using `StateFlow` for navigation events — use `Channel` instead (won't replay)
- Passing `Context` into ViewModel — use `ApplicationContext` only when necessary via `@ApplicationContext`
- Calling `viewModel.someFlow.collect {}` without `collectAsStateWithLifecycle` in Compose
- Business logic inside composables — belongs in ViewModel or use case
- ViewModel referencing a specific composable or Fragment

---

## Related Skills

- `clean-architecture` — layer structure ViewModel sits within
- `state-management` — advanced state patterns
- `side-effect-management` — handling side effects from ViewModel
- `use-case-design` — what the ViewModel delegates to
- `savedstatehandle` — persisting ViewModel state across process death
