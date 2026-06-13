---
name: state-restoration
description: >
  Complete state restoration strategy for Android — covering UI state,
  scroll position, form data, and Compose rememberSaveable.
  Load this skill when deciding how to persist and restore UI state
  across configuration changes and process death.
---

# State Restoration

## Overview

State restoration ensures the user sees a consistent UI after configuration changes (rotation) or returning after process death. Android provides multiple layers of restoration — from automatic (Compose rememberSaveable) to manual (SavedStateHandle). Choosing the right layer for each piece of state is the key design decision.

---

## Core Principles

- Match the **persistence layer** to the **state lifetime**
- UI-local state → `rememberSaveable`
- ViewModel state that must survive process death → `SavedStateHandle`
- Data loaded from network/DB → don't save, reload from repository
- Never duplicate state across multiple layers — one source of truth per state

---

## State Restoration Layers

| Layer                | Survives Rotation | Survives Process Death | Use For                               |
| -------------------- | ----------------- | ---------------------- | ------------------------------------- |
| `remember`           | ❌                 | ❌                      | Transient UI state (animation, focus) |
| `rememberSaveable`   | ✅                 | ✅                      | Local UI state (expanded, scroll)     |
| `SavedStateHandle`   | ✅                 | ✅                      | ViewModel state (query, selected ID)  |
| `Room` / `DataStore` | ✅                 | ✅                      | Persistent data                       |

---

## Compose — rememberSaveable

```kotlin
// ✅ Simple value — auto-saved
@Composable
fun SearchBar() {
    var query by rememberSaveable { mutableStateOf("") }
    // query survives rotation and process death
    TextField(value = query, onValueChange = { query = it })
}

// ✅ Custom type — requires Saver
data class FilterState(val category: String, val sort: String)

val FilterStateSaver = run {
    val categoryKey = "category"
    val sortKey = "sort"
    mapSaver(
        save = { mapOf(categoryKey to it.category, sortKey to it.sort) },
        restore = { FilterState(it[categoryKey] as String, it[sortKey] as String) }
    )
}

@Composable
fun FilterPanel() {
    var filter by rememberSaveable(stateSaver = FilterStateSaver) {
        mutableStateOf(FilterState("all", "asc"))
    }
}

// ✅ List state — scroll position
@Composable
fun UserList(users: List<User>) {
    val listState = rememberLazyListState()
    // listState scroll position is automatically saved by rememberLazyListState
    LazyColumn(state = listState) {
        items(users) { UserItem(it) }
    }
}
```

---

## Compose — Hoist State to ViewModel

```kotlin
// ✅ When state needs to be shared or outlive a single composable
// → hoist to ViewModel backed by SavedStateHandle

@HiltViewModel
class ProductViewModel @Inject constructor(
    savedStateHandle: SavedStateHandle,
    private val repository: ProductRepository
) : ViewModel() {

    // ✅ Search query — saved across process death
    val query: StateFlow<String> = savedStateHandle
        .getStateFlow("query", "")

    // ✅ Products — reloaded from DB, not saved
    val products: StateFlow<List<Product>> = query
        .flatMapLatest { repository.search(it) }
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), emptyList())

    fun onQueryChanged(q: String) { savedStateHandle["query"] = q }
}

@Composable
fun ProductScreen(viewModel: ProductViewModel = hiltViewModel()) {
    val query by viewModel.query.collectAsStateWithLifecycle()
    val products by viewModel.products.collectAsStateWithLifecycle()

    // UI renders from ViewModel state — no local state needed
}
```

---

## View System — onSaveInstanceState

```kotlin
// ✅ For custom Views that need to save state
class CounterView(context: Context) : View(context) {

    private var count = 0

    override fun onSaveInstanceState(): Parcelable {
        val superState = super.onSaveInstanceState()
        return SavedState(superState, count)
    }

    override fun onRestoreInstanceState(state: Parcelable?) {
        if (state is SavedState) {
            super.onRestoreInstanceState(state.superState)
            count = state.count
        } else {
            super.onRestoreInstanceState(state)
        }
    }

    @Parcelize
    class SavedState(val parcelable: Parcelable?, val count: Int) : BaseSavedState(parcelable)
}
```

---

## Restoration Decision Tree

```
Is this state local to a single composable?
    YES → rememberSaveable
    NO ↓

Is this state loaded from DB or network?
    YES → reload from repository (don't save)
    NO ↓

Is this state user input or a selected ID?
    YES → SavedStateHandle in ViewModel
    NO ↓

Is this critical persistent data?
    YES → Room or DataStore
```

---

## Testing Restoration

```kotlin
// ✅ Test rememberSaveable with StateRestorationTester
@Test
fun searchQuery_survivesStateRestoration() {
    val restorationTester = StateRestorationTester(composeTestRule)

    restorationTester.setContent {
        SearchBar()
    }

    composeTestRule.onNodeWithTag("search_field").performTextInput("kotlin")
    restorationTester.emulateSavedInstanceStateRestore()
    composeTestRule.onNodeWithTag("search_field").assertTextEquals("kotlin")
}

// ✅ Test SavedStateHandle restoration
@Test
fun viewModel_restoresQueryAfterProcessDeath() {
    val handle = SavedStateHandle(mapOf("query" to "android"))
    val vm = ProductViewModel(handle, fakeRepository)
    assertThat(vm.query.value).isEqualTo("android")
}
```

---

## Anti-Patterns

- Using `remember` instead of `rememberSaveable` for user-visible state — lost on rotation
- Saving large lists or bitmaps in `rememberSaveable` or `SavedStateHandle` — size limit and slow
- Duplicating state: saving in both ViewModel and rememberSaveable — two sources of truth
- Not testing restoration — the most common cause of rotation bugs
- Manually saving scroll position when `rememberLazyListState` does it automatically

---

## Related Skills

- `savedstatehandle` — SavedStateHandle in ViewModel
- `process-death-recovery` — surviving system-initiated kills
- `compose` — rememberSaveable and state hoisting in Compose
- `lifecycle` — when restoration occurs in the lifecycle
