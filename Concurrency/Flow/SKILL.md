---
name: flow
description: >
  Kotlin Flow for reactive streams in Android.
  Load this skill when building cold streams, transforming data pipelines,
  collecting flows with lifecycle awareness, combining multiple flows,
  or converting callbacks to Flow.
---

# Flow

## Overview

Kotlin Flow is a cold, asynchronous stream built on coroutines. It emits values sequentially and is collected in a coroutine. Flow is the standard for reactive data streams in Android — from database queries to UI state. Unlike LiveData, Flow is lifecycle-agnostic and must be collected with lifecycle awareness in the UI layer.

---

## Core Principles

- Flow is **cold** — it only runs when collected
- Collect flows in the UI with `collectAsStateWithLifecycle` (Compose) or `repeatOnLifecycle`
- Use `flowOn` to switch the dispatcher for upstream operators — not for collection
- Prefer `StateFlow` for UI state and `SharedFlow` for events — not plain `Flow`
- Use `callbackFlow` to bridge callback-based APIs to Flow

---

## Basic Flow

```kotlin
// ✅ Cold flow — runs fresh for each collector
fun getUsers(): Flow<List<User>> = flow {
    while (true) {
        emit(repository.getUsers())
        delay(30_000)  // poll every 30s
    }
}

// ✅ Flow from Room — already a Flow
fun getUsersFromDb(): Flow<List<User>> = userDao.getAllUsers()  // Room emits on every change
```

---

## Flow Operators

```kotlin
// ✅ Transform
val userNames: Flow<List<String>> = getUsersFlow()
    .map { users -> users.map { it.name } }

// ✅ Filter
val activeUsers: Flow<List<User>> = getUsersFlow()
    .map { users -> users.filter { it.isActive } }

// ✅ debounce — wait for stable value
val searchResults: Flow<List<User>> = searchQuery
    .debounce(300)
    .distinctUntilChanged()
    .flatMapLatest { query -> searchRepository.search(query) }

// ✅ combine — merge two flows into one
val uiState: Flow<UiState> = combine(usersFlow, filtersFlow) { users, filters ->
    UiState(users = users.filter { filters.matches(it) })
}

// ✅ zip — pair emissions one-to-one
val paired: Flow<Pair<User, Order>> = usersFlow.zip(ordersFlow) { user, order ->
    user to order
}

// ✅ flatMapLatest — cancel previous and switch to latest
val results: Flow<List<Result>> = queryFlow
    .flatMapLatest { query -> searchUseCase(query) }
```

---

## Error Handling

```kotlin
// ✅ catch — handle errors in the stream
getUsersFlow()
    .catch { error ->
        emit(emptyList())  // emit fallback
        Timber.e(error)
    }
    .collect { users -> render(users) }

// ✅ retry
getUsersFlow()
    .retry(3) { cause -> cause is IOException }
    .collect { users -> render(users) }

// ✅ onEach for side effects
getUsersFlow()
    .onEach { users -> Timber.d("Received ${users.size} users") }
    .catch { Timber.e(it) }
    .collect { render(it) }
```

---

## Lifecycle-Aware Collection

```kotlin
// ✅ Compose — collectAsStateWithLifecycle
@Composable
fun UserListScreen(viewModel: UserListViewModel = hiltViewModel()) {
    val state by viewModel.state.collectAsStateWithLifecycle()
    UserListContent(state)
}

// ✅ Fragment — repeatOnLifecycle
viewLifecycleOwner.lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.state.collect { render(it) }
    }
}

// ❌ Never collect without lifecycle awareness in UI
lifecycleScope.launch {
    viewModel.state.collect { render(it) }  // doesn't cancel when backgrounded
}
```

---

## callbackFlow — Bridge Callbacks to Flow

```kotlin
// ✅ Convert callback API to Flow
fun listenToLocation(): Flow<Location> = callbackFlow {
    val callback = object : LocationCallback() {
        override fun onLocationResult(result: LocationResult) {
            result.lastLocation?.let { trySend(it) }
        }
    }

    locationManager.requestUpdates(callback)

    awaitClose {
        locationManager.removeUpdates(callback)  // cleanup on cancellation
    }
}
```

---

## flowOn — Upstream Dispatcher

```kotlin
// ✅ flowOn changes dispatcher for everything upstream of it
val processedData: Flow<List<Item>> = rawDataFlow
    .map { parseHeavy(it) }      // runs on Default
    .filter { it.isValid }        // runs on Default
    .flowOn(Dispatchers.Default)  // applies to map + filter above
    .map { it.toUiModel() }       // runs on collection dispatcher
```

---

## Anti-Patterns

- Collecting flow without lifecycle awareness in Android UI — subscribes even in background
- Using `flow { }` for hot data that should be `StateFlow` or `SharedFlow`
- Calling `.collect {}` directly in a composable body — use `collectAsStateWithLifecycle`
- Using `flatMapMerge` for search/autocomplete — use `flatMapLatest` to cancel previous
- Not calling `awaitClose` in `callbackFlow` — resource leak on cancellation

---

## Related Skills

- `stateflow` — hot state holder for UI state
- `sharedflow` — hot shared stream for events
- `coroutine` — coroutine fundamentals
- `lifecycle` — lifecycle-aware collection patterns
- `reactive-streams` — broader reactive programming concepts
