---
name: compose
description: >
  Jetpack Compose fundamentals, patterns, and best practices for Android UI.
  Load this skill when building any Compose UI, designing composable functions,
  managing state in Compose, or optimizing recomposition.
---

# Compose

## Overview
Jetpack Compose is Android's modern declarative UI toolkit. UI is described as functions of state — when state changes, the UI recomposes automatically. Understanding recomposition, state hoisting, and composable design is essential for correct and performant Compose code.

---

## Core Principles

- UI is a **function of state** — never imperatively mutate UI
- **Hoist state** to the lowest common ancestor that needs it
- Keep composables **small and focused** — one responsibility per composable
- **Stateless composables** are easier to test, preview, and reuse
- Avoid side effects inside composable functions — use `LaunchedEffect`, `SideEffect`, `DisposableEffect`

---

## Composable Function Design

```kotlin
// ✅ Stateless composable — accepts state, emits events
@Composable
fun UserCard(
    user: User,
    onUserClick: (String) -> Unit,
    modifier: Modifier = Modifier  // always include modifier parameter
) {
    Card(
        modifier = modifier.clickable { onUserClick(user.id) }
    ) {
        Column(modifier = Modifier.padding(16.dp)) {
            Text(text = user.name, style = MaterialTheme.typography.titleMedium)
            Text(text = user.email, style = MaterialTheme.typography.bodySmall)
        }
    }
}

// ✅ Stateful composable — owns state, delegates to stateless
@Composable
fun UserCardScreen(viewModel: UserViewModel = hiltViewModel()) {
    val state by viewModel.state.collectAsStateWithLifecycle()
    UserCard(
        user = state.user,
        onUserClick = viewModel::onUserClicked
    )
}
```

---

## State Management in Compose

```kotlin
// ✅ Local UI state — remember
@Composable
fun ExpandableCard(title: String, content: String) {
    var isExpanded by remember { mutableStateOf(false) }

    Card(modifier = Modifier.clickable { isExpanded = !isExpanded }) {
        Column {
            Text(title)
            if (isExpanded) Text(content)
        }
    }
}

// ✅ State that survives rotation — rememberSaveable
@Composable
fun SearchBar(onSearch: (String) -> Unit) {
    var query by rememberSaveable { mutableStateOf("") }
    TextField(
        value = query,
        onValueChange = { query = it },
        trailingIcon = {
            IconButton(onClick = { onSearch(query) }) {
                Icon(Icons.Default.Search, contentDescription = null)
            }
        }
    )
}

// ✅ Complex state — hoist to ViewModel
@Composable
fun ProductScreen(viewModel: ProductViewModel = hiltViewModel()) {
    val state by viewModel.state.collectAsStateWithLifecycle()
    ProductContent(
        state = state,
        onAddToCart = viewModel::onAddToCart,
        onQuantityChange = viewModel::onQuantityChange
    )
}
```

---

## State Hoisting Pattern

```kotlin
// ✅ Hoist state up — pass value and onValueChange
@Composable
fun EmailField(
    value: String,
    onValueChange: (String) -> Unit,
    modifier: Modifier = Modifier
) {
    OutlinedTextField(
        value = value,
        onValueChange = onValueChange,
        label = { Text("Email") },
        modifier = modifier
    )
}

// ✅ Owner of state composes and passes down
@Composable
fun RegisterForm() {
    var email by rememberSaveable { mutableStateOf("") }
    var password by rememberSaveable { mutableStateOf("") }

    EmailField(value = email, onValueChange = { email = it })
    PasswordField(value = password, onValueChange = { password = it })
}
```

---

## Side Effects

```kotlin
// ✅ LaunchedEffect — coroutine tied to composition
@Composable
fun UserScreen(userId: String, viewModel: UserViewModel = hiltViewModel()) {
    LaunchedEffect(userId) {
        viewModel.loadUser(userId)  // re-runs when userId changes
    }
}

// ✅ DisposableEffect — non-coroutine cleanup
@Composable
fun LifecycleAwareScreen(onResume: () -> Unit) {
    val lifecycleOwner = LocalLifecycleOwner.current
    DisposableEffect(lifecycleOwner) {
        val observer = LifecycleEventObserver { _, event ->
            if (event == Lifecycle.Event.ON_RESUME) onResume()
        }
        lifecycleOwner.lifecycle.addObserver(observer)
        onDispose { lifecycleOwner.lifecycle.removeObserver(observer) }
    }
}

// ✅ SideEffect — sync Compose state to non-Compose code
@Composable
fun AnalyticsScreen(screenName: String) {
    SideEffect {
        analytics.logScreenView(screenName)  // runs on every successful recomposition
    }
}

// ✅ One-time events from ViewModel
@Composable
fun OrderScreen(viewModel: OrderViewModel = hiltViewModel()) {
    val lifecycleOwner = LocalLifecycleOwner.current

    LaunchedEffect(Unit) {
        viewModel.events
            .flowWithLifecycle(lifecycleOwner.lifecycle)
            .collect { event ->
                when (event) {
                    is OrderEvent.NavigateToConfirmation -> navController.navigate(...)
                    is OrderEvent.ShowError -> snackbarHostState.showSnackbar(event.message)
                }
            }
    }
}
```

---

## Modifier Best Practices

```kotlin
// ✅ Always accept modifier as parameter — last before lambdas
@Composable
fun MyComponent(
    text: String,
    modifier: Modifier = Modifier,  // default to Modifier (empty)
    onClick: () -> Unit
) {
    Box(modifier = modifier.clickable { onClick() }) {
        Text(text)
    }
}

// ✅ Order matters — padding before clickable = larger touch target
Box(modifier = Modifier
    .padding(8.dp)
    .clickable { }  // touch area includes padding
)

// ✅ Clickable before padding = smaller touch target
Box(modifier = Modifier
    .clickable { }
    .padding(8.dp)  // padding is outside the touch area
)
```

---

## Lists

```kotlin
// ✅ LazyColumn for long lists — only renders visible items
@Composable
fun UserList(users: List<User>, onUserClick: (String) -> Unit) {
    LazyColumn(
        contentPadding = PaddingValues(16.dp),
        verticalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        items(
            items = users,
            key = { user -> user.id }  // ✅ always provide stable key
        ) { user ->
            UserCard(user = user, onUserClick = onUserClick)
        }
    }
}

// ✅ Column only for short/fixed lists (< ~10 items)
@Composable
fun SettingsMenu(items: List<SettingItem>) {
    Column {
        items.forEach { item ->
            SettingRow(item = item)
        }
    }
}
```

---

## Previews

```kotlin
// ✅ Preview stateless composables
@Preview(showBackground = true)
@Preview(uiMode = Configuration.UI_MODE_NIGHT_YES)
@Composable
private fun UserCardPreview() {
    AppTheme {
        UserCard(
            user = User(id = "1", name = "Ali Rezaei", email = "ali@example.com"),
            onUserClick = {}
        )
    }
}

// ✅ Preview with multiple states
@Preview(name = "Loading")
@Preview(name = "Success")
@Composable
private fun UserScreenPreview() {
    AppTheme {
        UserContent(state = previewState)
    }
}
```

---

## Anti-Patterns

- Side effects directly in composable body — use `LaunchedEffect`/`DisposableEffect`
- Reading `viewModel.state.value` directly — use `collectAsStateWithLifecycle()`
- Not providing `key` in `items {}` — causes incorrect animations and recomposition
- Deeply nested composables — extract to named functions
- Not passing `modifier` parameter — prevents callers from customizing layout
- Creating remember/state inside loops or conditions — violates rules of hooks
- Using `Column` for long scrollable lists — use `LazyColumn`

---

## Related Skills
- `compose-navigation` — navigation between screens
- `compose-performance` — recomposition optimization
- `compose-animation` — animation in Compose
- `material3` — Material 3 components and theming
- `state-management` — state patterns with ViewModel
- `reactive-state-management` — collecting flows in Compose
