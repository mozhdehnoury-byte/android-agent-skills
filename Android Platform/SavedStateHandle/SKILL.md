---
name: savedstatehandle
description: >
  SavedStateHandle usage in ViewModel for surviving process death and
  configuration changes. Load this skill when persisting ViewModel state
  across process kill, reading navigation arguments, or deciding what
  state needs to be saved vs what can be reloaded.
---

# SavedStateHandle

## Overview

SavedStateHandle is a key-value store provided to ViewModel that survives both configuration changes and process death. Unlike ViewModel which only survives rotation, SavedStateHandle persists through system-initiated process kills and is restored when the user returns to the app.

---

## Core Principles

- Use SavedStateHandle for state that must survive **process death** — not just rotation
- Don't save everything — only save what can't be easily reloaded (user input, scroll position, selected IDs)
- Data stored in SavedStateHandle must be **Parcelable or primitive**
- Prefer `saveable { }` in Compose for local UI state — use SavedStateHandle for ViewModel state
- Navigation arguments are automatically available via SavedStateHandle

---

## What to Save vs What to Reload

| State Type                           | Strategy                               |
| ------------------------------------ | -------------------------------------- |
| User input (text fields, selections) | ✅ SavedStateHandle                     |
| Scroll position                      | ✅ SavedStateHandle or rememberSaveable |
| Selected item ID                     | ✅ SavedStateHandle                     |
| Loaded list from network/DB          | ❌ Reload from repository               |
| Loading/error state                  | ❌ Derive from data loading             |
| Navigation arguments                 | ✅ Read from SavedStateHandle           |

---

## Basic Usage

```kotlin
// ✅ Inject via Hilt — automatic with @HiltViewModel
@HiltViewModel
class UserDetailViewModel @Inject constructor(
    savedStateHandle: SavedStateHandle,
    private val repository: UserRepository
) : ViewModel() {

    // ✅ Read navigation argument (set by NavController)
    private val userId: String = checkNotNull(savedStateHandle["userId"])

    // ✅ StateFlow backed by SavedStateHandle — survives process death
    val searchQuery: StateFlow<String> = savedStateHandle
        .getStateFlow("search_query", initialValue = "")

    fun onSearchQueryChanged(query: String) {
        savedStateHandle["search_query"] = query
    }
}
```

---

## Reading Navigation Arguments

```kotlin
// ✅ Compose Navigation — argument passed via route
// NavHost setup
composable("user/{userId}") { backStackEntry ->
    val viewModel: UserDetailViewModel = hiltViewModel()
    UserDetailScreen(viewModel)
}

// ✅ ViewModel reads it from SavedStateHandle
@HiltViewModel
class UserDetailViewModel @Inject constructor(
    savedStateHandle: SavedStateHandle
) : ViewModel() {
    val userId: String = checkNotNull(savedStateHandle["userId"]) {
        "userId argument is required"
    }
}

// ✅ Type-safe navigation (Compose Navigation 2.8+)
@Serializable
data class UserDetailRoute(val userId: String)

@HiltViewModel
class UserDetailViewModel @Inject constructor(
    savedStateHandle: SavedStateHandle
) : ViewModel() {
    private val route = savedStateHandle.toRoute<UserDetailRoute>()
    val userId = route.userId
}
```

---

## StateFlow from SavedStateHandle

```kotlin
@HiltViewModel
class SearchViewModel @Inject constructor(
    private val savedStateHandle: SavedStateHandle,
    private val repository: SearchRepository
) : ViewModel() {

    // ✅ Backed by SavedStateHandle — survives process death
    val query: StateFlow<String> = savedStateHandle
        .getStateFlow(KEY_QUERY, initialValue = "")

    // ✅ Derive results from the saved query
    val results: StateFlow<List<Result>> = query
        .debounce(300)
        .flatMapLatest { q -> repository.search(q) }
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5_000),
            initialValue = emptyList()
        )

    fun onQueryChanged(newQuery: String) {
        savedStateHandle[KEY_QUERY] = newQuery
    }

    companion object {
        private const val KEY_QUERY = "search_query"
    }
}
```

---

## Saving Complex Types

```kotlin
// ✅ Parcelable types work directly
@Parcelize
data class FilterState(
    val category: String,
    val sortOrder: String,
    val minPrice: Int
) : Parcelable

@HiltViewModel
class ProductViewModel @Inject constructor(
    savedStateHandle: SavedStateHandle
) : ViewModel() {

    var filterState: StateFlow<FilterState> = savedStateHandle
        .getStateFlow("filter", FilterState("all", "asc", 0))

    fun applyFilter(filter: FilterState) {
        savedStateHandle["filter"] = filter
    }
}

// ✅ Primitive types — no Parcelable needed
savedStateHandle["page"] = 1
savedStateHandle["isExpanded"] = true
savedStateHandle["selectedId"] = "user_123"
```

---

## Combining SavedStateHandle with Repository

```kotlin
@HiltViewModel
class UserListViewModel @Inject constructor(
    savedStateHandle: SavedStateHandle,
    private val repository: UserRepository
) : ViewModel() {

    // Persisted: selected filter
    private val selectedFilter = savedStateHandle
        .getStateFlow("filter", "all")

    // Not persisted: loaded data — reload from DB
    val users: StateFlow<List<User>> = selectedFilter
        .flatMapLatest { filter ->
            repository.observeUsers(filter)
        }
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5_000),
            initialValue = emptyList()
        )

    fun onFilterSelected(filter: String) {
        savedStateHandle["filter"] = filter
    }
}
```

---

## Anti-Patterns

- Saving large objects (bitmaps, full lists) in SavedStateHandle — use DB or files instead
- Not using SavedStateHandle for user input — it's lost on process death
- Reading navigation args from `Intent` in Activity when ViewModel is available
- Using `onSaveInstanceState` in Activity/Fragment when SavedStateHandle covers it
- Saving non-Parcelable custom types — causes crash at restore time
- Duplicating state between SavedStateHandle and a separate StateFlow

---

## Related Skills

- `process-death-recovery` — full process death survival strategy
- `state-restoration` — broader state restoration patterns
- `navigation` — passing arguments via NavController
- `lifecycle` — when process death occurs
- `viewmodel` — ViewModel lifecycle and scope
