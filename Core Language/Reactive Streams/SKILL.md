---
name: reactive-streams
description: >
  Reactive Streams patterns and operators for Android development.
  Load this skill when working with data streams, chaining async operations,
  transforming flows, combining multiple sources, or designing reactive pipelines
  across layers.
---

# Reactive Streams

## Overview

Reactive Streams is a standard for asynchronous stream processing with non-blocking backpressure. In Kotlin/Android, this is implemented via `Flow`. This skill covers how to design, transform, and consume reactive streams correctly across all layers.

---

## Core Principles

- Data flows **downward** — from data source to UI, never upward
- Streams are **cold by default** (`Flow`) — only active when collected
- **Never collect a Flow inside another Flow** — use operators instead
- **Never expose `MutableStateFlow` or `MutableSharedFlow`** from public APIs
- Transformation happens in the **middle layers** — not in UI, not in data source

---

## Flow Basics

```kotlin
// ✅ Cold flow — executes only when collected
fun getUsers(): Flow<List<User>> = flow {
    while (true) {
        emit(repository.fetchUsers())
        delay(30_000)
    }
}

// ✅ Flow from suspend function
fun getUserById(id: String): Flow<User> = flow {
    emit(repository.getUser(id))
}

// ✅ Flow from Room (already returns Flow)
@Query("SELECT * FROM users")
fun observeUsers(): Flow<List<UserEntity>>
```

---

## Key Operators

### Transformation

```kotlin
// map — transform each emission
val names: Flow<List<String>> = usersFlow.map { users ->
    users.map { it.name }
}

// flatMapLatest — switch to new flow on each emission (search, user selection)
val results: Flow<List<Result>> = queryFlow.flatMapLatest { query ->
    repository.search(query)
}

// transform — emit multiple values per input
val events: Flow<Event> = actionsFlow.transform { action ->
    emit(Event.Started)
    process(action)
    emit(Event.Completed)
}
```

### Filtering

```kotlin
// filter — conditional emission
val activeUsers = usersFlow.filter { it.isActive }

// distinctUntilChanged — suppress duplicate emissions
val query = searchFlow.distinctUntilChanged()

// debounce — wait for silence (search input)
val debouncedQuery = searchFlow
    .debounce(300)
    .distinctUntilChanged()
```

### Combination

```kotlin
// combine — latest from each source
val uiState: Flow<UiState> = combine(
    usersFlow,
    isLoadingFlow,
    errorFlow
) { users, isLoading, error ->
    UiState(users = users, isLoading = isLoading, error = error)
}

// zip — pair emissions one-to-one
val paired: Flow<Pair<A, B>> = flowA.zip(flowB) { a, b -> a to b }

// merge — emit from all sources as they arrive
val allEvents: Flow<Event> = merge(clickFlow, scrollFlow, keyboardFlow)
```

### Error Handling

```kotlin
// ✅ catch — handle errors without stopping the stream
usersFlow
    .catch { e -> emit(emptyList()) }
    .collect { ... }

// ✅ retry — retry on failure
networkFlow
    .retry(3) { e -> e is IOException }
    .collect { ... }

// ✅ retryWhen — conditional retry with backoff
networkFlow
    .retryWhen { cause, attempt ->
        cause is IOException && attempt < 3
    }
```

---

## Flow Across Layers

```kotlin
// ✅ Repository — expose Flow from data source
class UserRepository {
    fun observeUsers(): Flow<List<User>> =
        localDataSource.observeUsers().map { entities ->
            entities.map { it.toDomain() }
        }
}

// ✅ UseCase — transform and combine
class GetActiveUsersUseCase(private val repository: UserRepository) {
    operator fun invoke(): Flow<List<User>> =
        repository.observeUsers()
            .map { users -> users.filter { it.isActive } }
            .distinctUntilChanged()
}

// ✅ ViewModel — collect into StateFlow
class UserViewModel(useCase: GetActiveUsersUseCase) : ViewModel() {
    val users: StateFlow<List<User>> = useCase()
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5_000),
            initialValue = emptyList()
        )
}

// ✅ UI — collect with lifecycle awareness
viewLifecycleOwner.lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.users.collect { users ->
            adapter.submitList(users)
        }
    }
}
```

---

## stateIn vs shareIn

```kotlin
// ✅ stateIn — convert Flow to StateFlow (single value, replays last)
val state: StateFlow<T> = flow.stateIn(
    scope = viewModelScope,
    started = SharingStarted.WhileSubscribed(5_000),
    initialValue = initialValue
)

// ✅ shareIn — convert Flow to SharedFlow (multicast, configurable replay)
val shared: SharedFlow<T> = flow.shareIn(
    scope = viewModelScope,
    started = SharingStarted.WhileSubscribed(5_000),
    replay = 1
)

// WhileSubscribed(5_000) — keeps upstream alive 5s after last subscriber
// Lazy — starts only when first subscriber appears
// Eagerly — starts immediately
```

---

## Backpressure

```kotlin
// ✅ buffer — decouple producer from consumer
fastProducerFlow
    .buffer(capacity = 64)
    .collect { slowConsumer(it) }

// ✅ conflate — drop intermediate values, keep latest
sensorFlow
    .conflate()
    .collect { render(it) }

// ✅ collectLatest — cancel previous on new emission
searchFlow.collectLatest { query ->
    val results = repository.search(query)
    showResults(results)
}
```

---

## Anti-Patterns

- Collecting Flow inside `viewModelScope.launch {}` directly in UI — use `repeatOnLifecycle`
- Using `GlobalScope` to launch flow collection — always use scoped coroutines
- Nested `collect` calls — use `flatMapLatest` or `combine` instead
- Creating a new Flow on every recomposition in Compose — hoist to ViewModel
- Using `flow.first()` in a loop — use operators instead
- Ignoring backpressure on fast producers — use `buffer`, `conflate`, or `collectLatest`
- Exposing `MutableSharedFlow` — always expose as `SharedFlow` or `Flow`

---

## Related Skills

- `coroutine` — coroutine scope and dispatcher management
- `stateflow` — StateFlow specific patterns
- `sharedflow` — SharedFlow specific patterns
- `state-management` — UI state with reactive streams
- `repository-pattern` — exposing streams from data layer
