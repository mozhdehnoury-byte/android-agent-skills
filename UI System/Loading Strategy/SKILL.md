---
name: loading-strategy
description: >
  Strategies for showing loading states in Android/Compose applications.
  Load this skill when implementing skeleton screens, shimmer effects,
  pull-to-refresh, progressive loading, pagination loading indicators,
  or deciding between spinner vs skeleton vs placeholder approaches.
---

# Loading Strategy

## Overview
Loading strategy defines how the UI communicates data-fetching progress to the user. The choice between spinner, skeleton, shimmer, and placeholder affects perceived performance significantly. A consistent, well-defined loading strategy prevents layout shifts and provides a smoother user experience.

---

## Core Principles

- Use **skeleton/shimmer** for initial content load — never a full-screen spinner
- Use **inline spinner** for actions (button submit, pull-to-refresh)
- Never block the entire screen with a loading overlay unless the action is destructive and non-cancellable
- Loading state must be part of the UI state model — never a separate boolean flag
- Skeleton dimensions must match the real content dimensions to prevent layout shift

---

## UI State Model

```kotlin
// ✅ Loading is part of a sealed state — not a separate flag
sealed interface UiState<out T> {
    data object Loading : UiState<Nothing>
    data class Success<T>(val data: T) : UiState<T>
    data class Error(val message: String) : UiState<Nothing>
}

// ✅ In ViewModel
class UserListViewModel : ViewModel() {
    private val _state = MutableStateFlow<UiState<List<User>>>(UiState.Loading)
    val state: StateFlow<UiState<List<User>>> = _state.asStateFlow()

    fun loadUsers() {
        viewModelScope.launch {
            _state.value = UiState.Loading
            _state.value = repository.getUsers().fold(
                onSuccess = { UiState.Success(it) },
                onFailure = { UiState.Error(it.message ?: "Unknown error") }
            )
        }
    }
}
```

---

## Skeleton Screen

```kotlin
// ✅ Skeleton matches real item dimensions
@Composable
fun UserItemSkeleton(modifier: Modifier = Modifier) {
    Row(
        modifier = modifier
            .fillMaxWidth()
            .padding(horizontal = AppSpacing.md, vertical = AppSpacing.sm),
        horizontalArrangement = Arrangement.spacedBy(AppSpacing.md)
    ) {
        // Avatar placeholder
        Box(
            modifier = Modifier
                .size(40.dp)
                .clip(CircleShape)
                .shimmer()
        )
        Column(verticalArrangement = Arrangement.spacedBy(AppSpacing.xs)) {
            // Name placeholder
            Box(
                modifier = Modifier
                    .fillMaxWidth(0.5f)
                    .height(16.dp)
                    .clip(MaterialTheme.shapes.small)
                    .shimmer()
            )
            // Subtitle placeholder
            Box(
                modifier = Modifier
                    .fillMaxWidth(0.35f)
                    .height(12.dp)
                    .clip(MaterialTheme.shapes.small)
                    .shimmer()
            )
        }
    }
}

// ✅ Shimmer modifier using animated brush
@Composable
fun Modifier.shimmer(): Modifier {
    val transition = rememberInfiniteTransition(label = "shimmer")
    val translateAnim by transition.animateFloat(
        initialValue = 0f,
        targetValue = 1000f,
        animationSpec = infiniteRepeatable(
            animation = tween(durationMillis = 1200, easing = LinearEasing),
            repeatMode = RepeatMode.Restart
        ),
        label = "shimmer_translate"
    )
    val brush = Brush.linearGradient(
        colors = listOf(
            MaterialTheme.colorScheme.surfaceVariant,
            MaterialTheme.colorScheme.surface,
            MaterialTheme.colorScheme.surfaceVariant
        ),
        start = Offset(translateAnim - 200f, 0f),
        end = Offset(translateAnim, 0f)
    )
    return this.background(brush)
}
```

---

## List Loading

```kotlin
// ✅ Show skeleton list during initial load
@Composable
fun UserListScreen(viewModel: UserListViewModel = hiltViewModel()) {
    val state by viewModel.state.collectAsStateWithLifecycle()

    when (state) {
        is UiState.Loading -> {
            LazyColumn {
                items(6) { UserItemSkeleton() }  // fixed count matches expected content
            }
        }
        is UiState.Success -> {
            LazyColumn {
                items((state as UiState.Success).data) { user ->
                    UserItem(user)
                }
            }
        }
        is UiState.Error -> {
            ErrorState(message = (state as UiState.Error).message)
        }
    }
}
```

---

## Pull-to-Refresh

```kotlin
// ✅ Pull-to-refresh with PullToRefreshBox (M3)
@Composable
fun UserListScreen(viewModel: UserListViewModel = hiltViewModel()) {
    val state by viewModel.state.collectAsStateWithLifecycle()
    val isRefreshing = state is UiState.Loading && viewModel.isRefreshing

    PullToRefreshBox(
        isRefreshing = isRefreshing,
        onRefresh = viewModel::refresh
    ) {
        LazyColumn { ... }
    }
}
```

---

## Button Loading State

```kotlin
// ✅ Inline loading indicator in action button
@Composable
fun SubmitButton(
    isLoading: Boolean,
    onClick: () -> Unit
) {
    Button(
        onClick = onClick,
        enabled = !isLoading
    ) {
        AnimatedContent(targetState = isLoading, label = "submit_btn") { loading ->
            if (loading) {
                CircularProgressIndicator(
                    modifier = Modifier.size(18.dp),
                    strokeWidth = 2.dp,
                    color = MaterialTheme.colorScheme.onPrimary
                )
            } else {
                Text("Submit")
            }
        }
    }
}
```

---

## Pagination Loading

```kotlin
// ✅ Footer loading indicator for paginated lists
@Composable
fun PaginatedList(
    items: List<Item>,
    isLoadingMore: Boolean,
    onLoadMore: () -> Unit
) {
    LazyColumn {
        items(items) { ItemRow(it) }

        if (isLoadingMore) {
            item {
                Box(
                    modifier = Modifier
                        .fillMaxWidth()
                        .padding(AppSpacing.md),
                    contentAlignment = Alignment.Center
                ) {
                    CircularProgressIndicator(modifier = Modifier.size(24.dp))
                }
            }
        }
    }

    // Trigger load more when near end
    LaunchedEffect(items.size) {
        if (items.isNotEmpty()) onLoadMore()
    }
}
```

---

## Anti-Patterns

- Full-screen `CircularProgressIndicator` for content loading — use skeleton instead
- Using a separate `isLoading: Boolean` flag alongside data state — use sealed state
- Skeleton with wrong dimensions — causes layout shift when content appears
- Not disabling interactive elements during loading — allows double submissions
- Showing spinner on every recomposition — tie to actual async state only

---

## Related Skills
- `empty-state-strategy` — what to show when loading succeeds but data is empty
- `state-management` — UI state modeling
- `compose` — animation and layout fundamentals
- `paging` — pagination with Paging 3 library
