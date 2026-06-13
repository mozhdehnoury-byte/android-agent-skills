---
name: empty-state-strategy
description: >
  Strategies for displaying empty states in Android/Compose applications.
  Load this skill when building screens that may have no data to show,
  handling empty search results, first-time user experiences, filtered
  lists with no matches, or error states that look like empty states.
---

# Empty State Strategy

## Overview
An empty state is what the user sees when a screen has no content to display. A well-designed empty state communicates clearly why there's nothing to show and what the user can do about it. Poor empty states leave users confused about whether the app is broken or the feature simply has no data yet.

---

## Core Principles

- Every list screen must account for an empty state — it is never optional
- Empty state must communicate **why** it's empty and **what the user can do**
- Distinguish between: no data yet, empty search result, filtered result, and error
- Empty state is a UI state variant — handle it in the state model, not with an `if` in the composable
- Never show a blank screen — always provide context and a call to action when possible

---

## Empty State Types

| Type | Cause | Action |
|------|-------|--------|
| First-time | User has no data yet | Primary CTA to create |
| Empty search | Query returned no results | Clear search / change query |
| Filtered | Active filter hides all items | Clear filters |
| Error | Failed to load | Retry button |
| Offline | No connectivity | Retry when online |

---

## UI State Model

```kotlin
// ✅ Empty is a distinct state, not null data
sealed interface UiState<out T> {
    data object Loading : UiState<Nothing>
    data class Success<T>(val data: T) : UiState<T>
    data object Empty : UiState<Nothing>
    data class Error(val message: String) : UiState<Nothing>
}

// ✅ ViewModel maps domain result to correct state
fun loadItems() {
    viewModelScope.launch {
        _state.value = UiState.Loading
        repository.getItems().fold(
            onSuccess = { items ->
                _state.value = if (items.isEmpty()) UiState.Empty
                               else UiState.Success(items)
            },
            onFailure = { _state.value = UiState.Error(it.message ?: "Error") }
        )
    }
}
```

---

## Empty State Component

```kotlin
// ✅ Reusable empty state component
@Composable
fun EmptyState(
    icon: ImageVector,
    title: String,
    subtitle: String? = null,
    actionLabel: String? = null,
    onAction: (() -> Unit)? = null,
    modifier: Modifier = Modifier
) {
    Column(
        modifier = modifier
            .fillMaxSize()
            .padding(AppSpacing.xl),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        Icon(
            imageVector = icon,
            contentDescription = null,
            modifier = Modifier.size(64.dp),
            tint = MaterialTheme.colorScheme.onSurfaceVariant.copy(alpha = 0.4f)
        )
        Spacer(Modifier.height(AppSpacing.md))
        Text(
            text = title,
            style = MaterialTheme.typography.titleMedium,
            textAlign = TextAlign.Center,
            color = MaterialTheme.colorScheme.onSurface
        )
        if (subtitle != null) {
            Spacer(Modifier.height(AppSpacing.sm))
            Text(
                text = subtitle,
                style = MaterialTheme.typography.bodyMedium,
                textAlign = TextAlign.Center,
                color = MaterialTheme.colorScheme.onSurfaceVariant
            )
        }
        if (actionLabel != null && onAction != null) {
            Spacer(Modifier.height(AppSpacing.lg))
            Button(onClick = onAction) {
                Text(actionLabel)
            }
        }
    }
}
```

---

## Usage Per Type

```kotlin
// ✅ First-time empty state
EmptyState(
    icon = Icons.Outlined.Inbox,
    title = "No items yet",
    subtitle = "Create your first item to get started.",
    actionLabel = "Create Item",
    onAction = onCreateClick
)

// ✅ Empty search result
EmptyState(
    icon = Icons.Outlined.SearchOff,
    title = "No results for \"$query\"",
    subtitle = "Try a different search term.",
    actionLabel = "Clear Search",
    onAction = onClearSearch
)

// ✅ Active filter hides all items
EmptyState(
    icon = Icons.Outlined.FilterAltOff,
    title = "No matches",
    subtitle = "No items match the active filters.",
    actionLabel = "Clear Filters",
    onAction = onClearFilters
)

// ✅ Error state (looks like empty — distinguish clearly)
EmptyState(
    icon = Icons.Outlined.CloudOff,
    title = "Couldn't load items",
    subtitle = "Check your connection and try again.",
    actionLabel = "Retry",
    onAction = onRetry
)
```

---

## Screen Integration

```kotlin
// ✅ Handle all states in one when block
@Composable
fun ItemListScreen(viewModel: ItemListViewModel = hiltViewModel()) {
    val state by viewModel.state.collectAsStateWithLifecycle()

    when (state) {
        is UiState.Loading -> SkeletonList()
        is UiState.Empty   -> EmptyState(
            icon = Icons.Outlined.Inbox,
            title = "Nothing here yet",
            subtitle = "Add your first item.",
            actionLabel = "Add Item",
            onAction = viewModel::onAddClick
        )
        is UiState.Success -> ItemList((state as UiState.Success).data)
        is UiState.Error   -> EmptyState(
            icon = Icons.Outlined.ErrorOutline,
            title = "Something went wrong",
            subtitle = (state as UiState.Error).message,
            actionLabel = "Retry",
            onAction = viewModel::loadItems
        )
    }
}
```

---

## Anti-Patterns

- Showing a blank screen when data is empty — always provide context
- Using `if (list.isEmpty())` inline in a composable instead of a sealed state
- Confusing error state with empty state — user must know why content is missing
- Showing a generic "No data" message without context or action
- Hiding the empty state behind a loading state that never resolves

---

## Related Skills
- `loading-strategy` — what to show while data is being fetched
- `state-management` — modeling UI states as sealed classes
- `error-handling` — error propagation and user-facing error messages
- `compose` — layout and animation fundamentals
