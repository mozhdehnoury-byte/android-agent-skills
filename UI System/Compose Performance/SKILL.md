---
name: compose-performance
description: >
  Compose recomposition optimization and performance best practices.
  Load this skill when diagnosing recomposition issues, optimizing list
  performance, reducing unnecessary work in Compose, or profiling UI jank.
---

# Compose Performance

## Overview
Compose performance is primarily about controlling **recomposition** — the process of re-running composable functions when state changes. Unnecessary recomposition wastes CPU and causes UI jank. Understanding what triggers recomposition and how to minimize its scope is the key to performant Compose UIs.

---

## Core Principles

- Recomposition scope is the **smallest enclosing composable** that reads changed state
- Pass **only the state each composable needs** — not the entire state object
- Use **stable types** — unstable types cause unnecessary recomposition
- Move **heavy computation** out of composition — use `remember` or `derivedStateOf`
- Use `key` in lists — enables smart diffing and avoids full re-render

---

## Stability

```kotlin
// ✅ Stable types — Compose can skip recomposition if inputs unchanged
// - All primitive types (Int, String, Boolean, etc.)
// - Immutable data classes with stable fields
// - @Stable or @Immutable annotated classes

@Immutable  // ✅ Promise to Compose: this object never changes
data class UserUiState(
    val name: String,
    val email: String,
    val isActive: Boolean
)

@Stable  // ✅ Promise: changes are tracked via Snapshot state
class UserPreferences {
    var isDarkMode by mutableStateOf(false)
    var language by mutableStateOf("en")
}

// ❌ Unstable — List<T> is considered unstable by default
data class UserListState(
    val users: List<User>  // triggers recomposition on every emission even if data is same
)

// ✅ Use ImmutableList from kotlinx-collections-immutable
data class UserListState(
    val users: ImmutableList<User>  // stable — Compose can skip recomposition
)
```

---

## Passing Minimal State

```kotlin
// ❌ Passing entire state — entire subtree recomposes on any change
@Composable
fun UserCard(state: UserListUiState) {
    Text(state.selectedUser.name)  // recomposes when anything in state changes
}

// ✅ Pass only what's needed
@Composable
fun UserCard(userName: String, onUserClick: () -> Unit) {
    Text(userName)  // only recomposes when userName changes
}
```

---

## remember and derivedStateOf

```kotlin
// ✅ remember — cache expensive computation across recompositions
@Composable
fun FormattedPrice(priceInCents: Int) {
    val formatted = remember(priceInCents) {
        NumberFormat.getCurrencyInstance().format(priceInCents / 100.0)
    }
    Text(formatted)
}

// ✅ derivedStateOf — compute derived state, recompose only when result changes
@Composable
fun UserList(users: List<User>) {
    val activeUserCount by remember {
        derivedStateOf { users.count { it.isActive } }
    }
    // recomposes only when activeUserCount changes, not on every users update
    Text("Active users: $activeUserCount")
}

// ✅ derivedStateOf for scroll-based state
@Composable
fun ScrollableScreen() {
    val listState = rememberLazyListState()
    val showScrollToTop by remember {
        derivedStateOf { listState.firstVisibleItemIndex > 5 }
    }

    if (showScrollToTop) {
        ScrollToTopButton()
    }
}
```

---

## Lambda Stability

```kotlin
// ❌ Lambda created on each recomposition — causes child recomposition
@Composable
fun UserList(users: List<User>, viewModel: UserViewModel) {
    users.forEach { user ->
        UserCard(
            user = user,
            onClick = { viewModel.onUserClick(user.id) }  // new lambda each time
        )
    }
}

// ✅ Stable lambda reference
@Composable
fun UserList(users: List<User>, onUserClick: (String) -> Unit) {
    users.forEach { user ->
        UserCard(user = user, onClick = { onUserClick(user.id) })
    }
}

// ✅ Use rememberUpdatedState for callbacks that change
@Composable
fun Timer(onTick: () -> Unit) {
    val currentOnTick by rememberUpdatedState(onTick)
    LaunchedEffect(Unit) {
        while (true) {
            delay(1_000)
            currentOnTick()
        }
    }
}
```

---

## List Performance

```kotlin
// ✅ Always provide stable key in LazyColumn/LazyRow
LazyColumn {
    items(
        items = users,
        key = { user -> user.id }  // stable, unique key
    ) { user ->
        UserCard(user = user)
    }
}

// ✅ contentType — helps Compose reuse item compositions
LazyColumn {
    items(
        items = feedItems,
        key = { it.id },
        contentType = { item ->
            when (item) {
                is FeedItem.Post    -> "post"
                is FeedItem.Ad      -> "ad"
                is FeedItem.Header  -> "header"
            }
        }
    ) { item ->
        FeedItemView(item)
    }
}

// ✅ Use Pager for full-screen horizontal paging
HorizontalPager(count = pages.size) { page ->
    PageContent(pages[page])
}
```

---

## Read State at the Lowest Level

```kotlin
// ❌ Read state high up — entire tree recomposes
@Composable
fun Screen(scrollState: ScrollState) {
    val offset = scrollState.value  // read here — Screen recomposes on every scroll
    Column(modifier = Modifier.offset(y = offset.dp)) {
        ExpensiveContent()
    }
}

// ✅ Defer state read to Modifier — no recomposition, only layout pass
@Composable
fun Screen(scrollState: ScrollState) {
    Column(
        modifier = Modifier.graphicsLayer {
            translationY = scrollState.value.toFloat()  // read inside lambda — no recompose
        }
    ) {
        ExpensiveContent()
    }
}
```

---

## Baseline Profiles

```kotlin
// ✅ Generate Baseline Profile to pre-compile critical Compose paths
// See baseline-profile skill for full setup

// Macrobenchmark test
@RunWith(AndroidJUnit4::class)
class BaselineProfileGenerator {
    @get:Rule
    val rule = BaselineProfileRule()

    @Test
    fun generate() = rule.collect(packageName = "com.example.app") {
        startActivityAndWait()
        device.findObject(By.text("Users")).click()
        device.waitForIdle()
    }
}
```

---

## Profiling Tools

```
Layout Inspector → Recomposition counts per composable
Compose Tracing  → detailed timeline in Perfetto
Android Studio Profiler → CPU/frame timeline
```

```kotlin
// ✅ Add Compose tracing for Perfetto
dependencies {
    implementation(libs.androidx.tracing.compose)
}
```

---

## Anti-Patterns

- Reading `scrollState.value` at the top of a composable — causes excessive recomposition
- Using `List<T>` in state — use `ImmutableList<T>` for stable collections
- Creating lambdas inline in items — causes recomposition of every item
- Not providing `key` in `items {}` — full list recompose on any change
- Computing derived values without `remember` — recalculates every recomposition
- Putting heavy logic directly in composable body — use `remember` or move to ViewModel

---

## Related Skills
- `compose` — Compose fundamentals
- `compose-animation` — animation without recomposition overhead
- `baseline-profile` — pre-compiling Compose hot paths
- `benchmark` — measuring Compose performance
- `rendering-performance` — frame timing and jank detection
