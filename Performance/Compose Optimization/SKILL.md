---
name: compose-optimization
description: >
  Jetpack Compose performance optimization.
  Load this skill when reducing unnecessary recompositions, using
  stability annotations, optimizing LazyList performance, profiling
  with Layout Inspector, or fixing skipped frames in Compose UI.
---

# Compose Optimization

## Overview
Compose performance is dominated by **recomposition** — the process of re-running composables when state changes. Unnecessary recompositions cause dropped frames. The goal is to ensure only the composables that actually depend on changed state are recomposed, using stable types, correct key usage, and proper state scoping.

---

## Core Principles

- Recomposition is triggered by **state reads** — scope reads as narrowly as possible
- Use **stable types** for parameters — unstable types cause recomposition on every parent recompose
- Pass **lambdas** instead of state to leaf composables — reduces recomposition scope
- Use `key()` in `LazyColumn` — helps Compose identify and reuse items correctly
- Profile with **Compose Compiler Metrics** and **Layout Inspector** before optimizing

---

## Stability and @Stable / @Immutable

```kotlin
// ❌ Unstable class — List is not stable (Compose can't verify it won't change)
data class UserListState(
    val users: List<User>,  // mutable List — Compose treats as unstable
    val isLoading: Boolean
)

// ✅ Use ImmutableList or @Immutable annotation
@Immutable
data class UserListState(
    val users: ImmutableList<User>,  // kotlinx.collections.immutable
    val isLoading: Boolean
)

// ✅ Or annotate domain model as stable
@Stable
data class User(
    val id: String,
    val name: String,
    val email: String
)
```

---

## State Scoping — Read State as Low as Possible

```kotlin
// ❌ Reading state high in the tree — recomposes entire subtree
@Composable
fun UserScreen(viewModel: UserViewModel) {
    val state by viewModel.state.collectAsStateWithLifecycle()
    // entire screen recomposes when any state field changes
    UserList(users = state.users, isLoading = state.isLoading)
    SearchBar(query = state.searchQuery)
}

// ✅ Pass lambdas to defer state reads to leaf composables
@Composable
fun UserScreen(viewModel: UserViewModel) {
    UserList(
        users = { viewModel.state.value.users },  // read inside UserList
        isLoading = { viewModel.state.value.isLoading }
    )
}
```

---

## LazyColumn Optimization

```kotlin
// ✅ Always provide stable keys
LazyColumn {
    items(
        items = users,
        key = { user -> user.id }  // ✅ stable key — enables item reuse
    ) { user ->
        UserItem(user = user)
    }
}

// ✅ Use contentType for heterogeneous lists
LazyColumn {
    items(
        items = feedItems,
        key = { it.id },
        contentType = { item -> item::class }  // ✅ different composables for different types
    ) { item ->
        when (item) {
            is FeedItem.Post -> PostItem(item)
            is FeedItem.Ad   -> AdItem(item)
        }
    }
}

// ✅ rememberLazyListState to preserve scroll position
val listState = rememberLazyListState()
LazyColumn(state = listState) { ... }
```

---

## remember and derivedStateOf

```kotlin
// ✅ derivedStateOf — compute derived value only when inputs change
@Composable
fun UserList(users: List<User>, searchQuery: String) {
    val filteredUsers by remember(users, searchQuery) {
        derivedStateOf {
            users.filter { it.name.contains(searchQuery, ignoreCase = true) }
        }
    }
    // filteredUsers recomputes only when users or searchQuery change
}

// ✅ remember expensive computation
@Composable
fun ExpensiveScreen(data: List<Data>) {
    val processed = remember(data) {
        data.map { processExpensive(it) }  // only recomputes when data changes
    }
}
```

---

## Avoiding Lambda Allocations

```kotlin
// ❌ New lambda on every recomposition
@Composable
fun UserItem(user: User, onDelete: (User) -> Unit) {
    Button(onClick = { onDelete(user) }) { ... }  // new lambda every recompose
}

// ✅ remember the lambda
@Composable
fun UserItem(user: User, onDelete: (User) -> Unit) {
    val onClick = remember(user.id) { { onDelete(user) } }
    Button(onClick = onClick) { ... }
}
```

---

## Compose Compiler Metrics

```kotlin
// build.gradle.kts — enable compiler metrics
tasks.withType<KotlinCompile>().configureEach {
    kotlinOptions {
        freeCompilerArgs += listOf(
            "-P", "plugin:androidx.compose.compiler.plugins.kotlin:reportsDestination=${project.buildDir}/compose_metrics",
            "-P", "plugin:androidx.compose.compiler.plugins.kotlin:metricsDestination=${project.buildDir}/compose_metrics"
        )
    }
}
// Then check build/compose_metrics/ for stability reports
```

---

## Anti-Patterns

- Passing unstable `List<T>` to composables — triggers recomposition unnecessarily
- Reading `StateFlow.value` directly in composable body — doesn't trigger recomposition
- No `key` in `LazyColumn` items — items are recreated instead of reused on scroll
- Doing heavy computation directly in composable body — use `remember` or `derivedStateOf`
- Deeply nested composables all reading the same state — scope reads lower in the tree

---

## Related Skills
- `compose` — Compose fundamentals
- `compose-performance` — profiling and measuring Compose performance
- `state-management` — state hoisting and StateFlow patterns
- `allocation-optimization` — reducing object creation in Compose
