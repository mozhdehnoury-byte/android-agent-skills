---
name: error-handling
description: >
  Error handling strategy for Android Clean Architecture projects.
  Load this skill when designing how errors propagate across layers,
  using Result<T> for error representation, handling errors in ViewModel,
  or deciding between exceptions and sealed classes for error modeling.
---

# Error Handling

## Overview
Error handling defines how failures travel from the data layer to the UI. In Clean Architecture, errors are represented as `Result<T>` in the domain and data layers, mapped to domain-specific error types, and finally translated to user-friendly messages in the presentation layer. Exceptions are caught at layer boundaries — never leaked into the UI.

---

## Core Principles

- **Catch exceptions at boundaries** — Data layer catches, wraps in `Result.failure()`
- **Domain errors are sealed classes** — not raw exceptions; typed and exhaustive
- **ViewModel translates** domain errors to UI state — never expose raw exceptions to UI
- **Never swallow errors silently** — always log or propagate
- **`runCatching`** for converting exception-throwing code to `Result<T>`

---

## Result<T> Across Layers

```
Data Layer          Domain Layer         Presentation Layer
─────────────────   ──────────────────   ───────────────────
Repository          UseCase              ViewModel
runCatching {   →   Result<T>        →   UiState.Error(msg)
  api.call()    →   .map { }         →
}               →   .mapError { }    →
```

---

## Data Layer — Catching at the Boundary

```kotlin
// ✅ Repository wraps all exceptions in Result
class UserRepositoryImpl @Inject constructor(
    private val api: UserApiService,
    private val dao: UserDao
) : UserRepository {

    override suspend fun getUser(id: String): Result<User> = runCatching {
        dao.getById(id)?.toDomain()
            ?: api.getUser(id).toDomain().also { dao.insert(it.toEntity()) }
    }

    override suspend fun createUser(user: User): Result<User> = runCatching {
        val dto = api.createUser(user.toCreateRequest())
        val created = dto.toDomain()
        dao.insert(created.toEntity())
        created
    }

    // ✅ Flow — use catch operator
    override fun observeUsers(): Flow<List<User>> =
        dao.observeAll()
            .map { entities -> entities.map { it.toDomain() } }
            .catch { e ->
                // Log and emit empty — or rethrow depending on strategy
                Timber.e(e, "Failed to observe users")
                emit(emptyList())
            }
}
```

---

## Domain Layer — Typed Error Model

```kotlin
// ✅ Domain errors — see domain-error-model skill for full detail
sealed interface UserError {
    data object NotFound : UserError
    data object Unauthorized : UserError
    data class NetworkError(val message: String) : UserError
    data class ValidationError(val field: String, val reason: String) : UserError
}

// ✅ UseCase maps raw Result to domain Result
class GetUserUseCase @Inject constructor(
    private val repository: UserRepository
) {
    suspend operator fun invoke(id: String): Result<User> =
        repository.getUser(id)
            .mapCatching { user ->
                // Additional domain validation
                check(user.status != UserStatus.DELETED) { "User has been deleted" }
                user
            }
}
```

---

## Presentation Layer — ViewModel Error Handling

```kotlin
// ✅ ViewModel translates Result to UiState
@HiltViewModel
class UserDetailViewModel @Inject constructor(
    savedStateHandle: SavedStateHandle,
    private val getUserUseCase: GetUserUseCase,
    private val deleteUserUseCase: DeleteUserUseCase
) : ViewModel() {

    private val userId = checkNotNull(savedStateHandle.get<String>("userId"))

    private val _state = MutableStateFlow<UserDetailUiState>(UserDetailUiState.Loading)
    val state: StateFlow<UserDetailUiState> = _state.asStateFlow()

    private val _events = Channel<UserDetailEvent>(Channel.BUFFERED)
    val events: Flow<UserDetailEvent> = _events.receiveAsFlow()

    init { loadUser() }

    fun loadUser() {
        viewModelScope.launch {
            _state.value = UserDetailUiState.Loading
            getUserUseCase(userId).fold(
                onSuccess = { user ->
                    _state.value = UserDetailUiState.Success(user)
                },
                onFailure = { error ->
                    _state.value = UserDetailUiState.Error(
                        message = error.toUserMessage(),
                        canRetry = error.isRetryable()
                    )
                }
            )
        }
    }

    fun onDeleteClick() {
        viewModelScope.launch {
            deleteUserUseCase(userId).fold(
                onSuccess = {
                    _events.send(UserDetailEvent.NavigateBack)
                },
                onFailure = { error ->
                    _events.send(UserDetailEvent.ShowSnackbar(error.toUserMessage()))
                }
            )
        }
    }
}
```

---

## Error Extension Functions

```kotlin
// ✅ Centralized error translation
fun Throwable.toUserMessage(): String = when (this) {
    is HttpException -> when (code()) {
        401 -> "Session expired. Please log in again."
        403 -> "You don't have permission to do this."
        404 -> "The requested item was not found."
        429 -> "Too many requests. Please wait a moment."
        in 500..599 -> "Server error. Please try again later."
        else -> "Something went wrong. Please try again."
    }
    is IOException,
    is SocketTimeoutException -> "No internet connection. Please check your network."
    is CancellationException -> throw this  // never swallow cancellation
    else -> message ?: "An unexpected error occurred."
}

fun Throwable.isRetryable(): Boolean = when (this) {
    is IOException,
    is SocketTimeoutException -> true
    is HttpException -> code() in 500..599
    else -> false
}

// ✅ Result extensions
fun <T> Result<T>.onFailureLog(tag: String = "App"): Result<T> = onFailure { e ->
    if (e !is CancellationException) Timber.tag(tag).e(e)
}

suspend fun <T> Result<T>.orThrowDomain(
    transform: (Throwable) -> Throwable = { it }
): T = fold(
    onSuccess = { it },
    onFailure = { throw transform(it) }
)
```

---

## Flow Error Handling

```kotlin
// ✅ Wrap Flow emissions in Result for error propagation
fun <T> Flow<T>.asResult(): Flow<Result<T>> =
    map { Result.success(it) }
        .catch { emit(Result.failure(it)) }

// ✅ Usage in ViewModel
val state: StateFlow<UserListUiState> =
    getUsersUseCase()
        .asResult()
        .map { result ->
            result.fold(
                onSuccess = { UserListUiState.Success(it) },
                onFailure = { UserListUiState.Error(it.toUserMessage()) }
            )
        }
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), UserListUiState.Loading)
```

---

## Never Swallow CancellationException

```kotlin
// ✅ Always rethrow CancellationException
try {
    someOperation()
} catch (e: CancellationException) {
    throw e   // ✅ must rethrow — coroutine cancellation mechanism depends on this
} catch (e: Exception) {
    handleError(e)
}

// ✅ runCatching is safe — it doesn't swallow CancellationException in Kotlin 1.7+
val result = runCatching { someOperation() }
// But be careful with older Kotlin versions
```

---

## Anti-Patterns

- Catching `Exception` in ViewModel and showing generic message — typed errors give better UX
- Leaking `HttpException` or `IOException` into Domain layer — map at the repository boundary
- Empty `catch` blocks — always log or propagate
- Swallowing `CancellationException` — breaks coroutine cancellation
- Storing error state as `String?` in UiState — use a sealed class for typed error display

---

## Related Skills
- `domain-error-model` — typed domain error hierarchy
- `error-mapping` — mapping between layer-specific errors
- `failure-strategy` — deciding how to respond to different error types
- `user-friendly-errors` — translating technical errors to UI messages
- `state-management` — modeling error state in UiState
