---
name: mvi
description: >
  Model-View-Intent pattern for Android with Jetpack Compose.
  Load this skill when implementing strict unidirectional data flow,
  modeling all user interactions as Intents, using a single immutable
  state per screen, or managing side effects as Effects separate from state.
---

# MVI

## Overview

MVI enforces strict unidirectional data flow: the View emits **Intents**, the ViewModel processes them and produces a new immutable **State**, and one-time **Effects** handle side effects like navigation or toasts. Unlike MVVM, all user interactions are modeled as a sealed `Intent` class — nothing is called imperatively.

---

## Core Principles

- **One immutable State** per screen — never partial updates
- All user actions are **Intents** — sealed class, no direct function calls from UI
- **Effects** are one-time events (navigation, toast) — separate from State
- State transitions are **pure** — `reduce(currentState, intent) → newState`
- ViewModel processes intents sequentially — no race conditions on state

---

## Contract (State + Intent + Effect)

```kotlin
// ✅ Define the full screen contract in one place
object UserListContract {

    data class State(
        val isLoading: Boolean = false,
        val users: List<User> = emptyList(),
        val error: String? = null,
        val searchQuery: String = ""
    )

    sealed interface Intent {
        data object LoadUsers : Intent
        data class SearchChanged(val query: String) : Intent
        data class UserClicked(val userId: String) : Intent
        data class DeleteUser(val userId: String) : Intent
    }

    sealed interface Effect {
        data class NavigateToDetail(val userId: String) : Effect
        data class ShowSnackbar(val message: String) : Effect
    }
}
```

---

## ViewModel

```kotlin
// ✅ MVI ViewModel — processes intents, emits state and effects
@HiltViewModel
class UserListViewModel @Inject constructor(
    private val getUsersUseCase: GetUsersUseCase,
    private val deleteUserUseCase: DeleteUserUseCase
) : ViewModel() {

    private val _state = MutableStateFlow(UserListContract.State())
    val state: StateFlow<UserListContract.State> = _state.asStateFlow()

    private val _effects = Channel<UserListContract.Effect>(Channel.BUFFERED)
    val effects: Flow<UserListContract.Effect> = _effects.receiveAsFlow()

    fun handleIntent(intent: UserListContract.Intent) {
        when (intent) {
            is UserListContract.Intent.LoadUsers       -> loadUsers()
            is UserListContract.Intent.SearchChanged   -> onSearchChanged(intent.query)
            is UserListContract.Intent.UserClicked     -> onUserClicked(intent.userId)
            is UserListContract.Intent.DeleteUser      -> deleteUser(intent.userId)
        }
    }

    private fun loadUsers() {
        viewModelScope.launch {
            _state.update { it.copy(isLoading = true, error = null) }
            getUsersUseCase().fold(
                onSuccess = { users ->
                    _state.update { it.copy(isLoading = false, users = users) }
                },
                onFailure = { error ->
                    _state.update { it.copy(isLoading = false, error = error.message) }
                }
            )
        }
    }

    private fun onSearchChanged(query: String) {
        _state.update { it.copy(searchQuery = query) }
    }

    private fun onUserClicked(userId: String) {
        viewModelScope.launch {
            _effects.send(UserListContract.Effect.NavigateToDetail(userId))
        }
    }

    private fun deleteUser(userId: String) {
        viewModelScope.launch {
            deleteUserUseCase(userId).fold(
                onSuccess = {
                    loadUsers()
                    _effects.send(UserListContract.Effect.ShowSnackbar("User deleted"))
                },
                onFailure = {
                    _effects.send(UserListContract.Effect.ShowSnackbar("Delete failed"))
                }
            )
        }
    }
}
```

---

## Composable

```kotlin
// ✅ View sends intents — never calls ViewModel functions directly
@Composable
fun UserListScreen(
    onNavigateToDetail: (String) -> Unit,
    viewModel: UserListViewModel = hiltViewModel()
) {
    val state by viewModel.state.collectAsStateWithLifecycle()
    val snackbarHostState = remember { SnackbarHostState() }

    // ✅ Collect one-time effects
    LaunchedEffect(Unit) {
        viewModel.effects.collect { effect ->
            when (effect) {
                is UserListContract.Effect.NavigateToDetail ->
                    onNavigateToDetail(effect.userId)
                is UserListContract.Effect.ShowSnackbar ->
                    snackbarHostState.showSnackbar(effect.message)
            }
        }
    }

    // ✅ Trigger initial load via intent
    LaunchedEffect(Unit) {
        viewModel.handleIntent(UserListContract.Intent.LoadUsers)
    }

    Scaffold(snackbarHost = { SnackbarHost(snackbarHostState) }) { padding ->
        UserListContent(
            state = state,
            onIntent = viewModel::handleIntent,
            modifier = Modifier.padding(padding)
        )
    }
}

// ✅ Content composable receives state and onIntent — no ViewModel reference
@Composable
private fun UserListContent(
    state: UserListContract.State,
    onIntent: (UserListContract.Intent) -> Unit,
    modifier: Modifier = Modifier
) {
    Column(modifier = modifier) {
        SearchBar(
            query = state.searchQuery,
            onQueryChange = { onIntent(UserListContract.Intent.SearchChanged(it)) }
        )

        when {
            state.isLoading -> LoadingState()
            state.error != null -> ErrorState(
                message = state.error,
                onRetry = { onIntent(UserListContract.Intent.LoadUsers) }
            )
            state.users.isEmpty() -> EmptyState()
            else -> UserList(
                users = state.users,
                onUserClick = { onIntent(UserListContract.Intent.UserClicked(it)) },
                onDeleteClick = { onIntent(UserListContract.Intent.DeleteUser(it)) }
            )
        }
    }
}
```

---

## MVI vs MVVM — When to Use Each

| Concern        | MVVM                         | MVI                                    |
| -------------- | ---------------------------- | -------------------------------------- |
| Complexity     | Simple to medium screens     | Complex screens with many interactions |
| State tracing  | Multiple StateFlows possible | Single immutable State — easy to trace |
| Testability    | Test individual functions    | Test intent → state transitions        |
| Boilerplate    | Less                         | More (Contract class)                  |
| Predictability | Good                         | Excellent                              |

Use **MVVM** as the default. Reach for **MVI** when a screen has many concurrent interactions and state bugs are hard to trace.

---

## Anti-Patterns

- Calling ViewModel functions directly from UI — use `handleIntent()`
- Mutable state fields inside the State data class — keep State fully immutable
- Sending navigation as State change — navigation is an Effect, not State
- Multiple state flows per screen — one `StateFlow<State>` only
- Processing intents with side effects inside `reduce()` — `reduce()` must be pure if used

---

## Related Skills

- `mvvm` — simpler alternative for straightforward screens
- `unidirectional-data-flow` — the underlying pattern MVI is built on
- `state-management` — state patterns and tools
- `side-effect-management` — handling effects correctly
- `compose` — collecting state in Compose UI
