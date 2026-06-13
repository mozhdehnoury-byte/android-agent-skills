---
name: bottom-sheet-pattern
description: >
  Bottom sheet and side sheet patterns in Jetpack Compose with Material 3.
  Load this skill when implementing modal bottom sheets, persistent bottom
  sheets, side sheets, sheet state management, nested scrolling inside sheets,
  or handling keyboard interaction with sheets.
---

# Bottom Sheet / Side Sheet Patterns

## Overview
Bottom sheets and side sheets are surface containers that slide into view from the edge of the screen. M3 provides `ModalBottomSheet` and `BottomSheetScaffold`. Side sheets are handled via custom `AnimatedVisibility` or `Drawer` depending on the use case.

---

## Core Principles

- Use `ModalBottomSheet` for transient, overlay content (actions, pickers)
- Use `BottomSheetScaffold` for persistent sheets that coexist with screen content
- Always hoist `SheetState` to the caller — never manage it inside the sheet
- Add `imePadding()` inside sheet content when it contains input fields
- Never put a bottom sheet inside another bottom sheet

---

## Modal Bottom Sheet

```kotlin
// ✅ Basic modal bottom sheet
@Composable
fun UserActionsSheet(
    onDismiss: () -> Unit,
    onEdit: () -> Unit,
    onDelete: () -> Unit
) {
    val sheetState = rememberModalBottomSheetState(
        skipPartiallyExpanded = true   // go directly to full height
    )

    ModalBottomSheet(
        onDismissRequest = onDismiss,
        sheetState = sheetState
    ) {
        Column(
            modifier = Modifier
                .fillMaxWidth()
                .padding(bottom = AppSpacing.lg)
        ) {
            ListItem(
                headlineContent = { Text("Edit") },
                leadingContent = { Icon(Icons.Default.Edit, null) },
                modifier = Modifier.clickable { onEdit(); onDismiss() }
            )
            ListItem(
                headlineContent = { Text("Delete") },
                leadingContent = { Icon(Icons.Default.Delete, null) },
                modifier = Modifier.clickable { onDelete(); onDismiss() }
            )
        }
    }
}

// ✅ Show/hide from parent
@Composable
fun UserScreen() {
    var showSheet by remember { mutableStateOf(false) }

    Scaffold { padding ->
        // screen content
    }

    if (showSheet) {
        UserActionsSheet(
            onDismiss = { showSheet = false },
            onEdit = { /* ... */ },
            onDelete = { /* ... */ }
        )
    }
}
```

---

## Sheet with Input Fields (Keyboard)

```kotlin
// ✅ imePadding + windowInsetsPadding inside sheet with text fields
@Composable
fun AddItemSheet(
    onDismiss: () -> Unit,
    onAdd: (String) -> Unit
) {
    val sheetState = rememberModalBottomSheetState(skipPartiallyExpanded = true)
    var text by remember { mutableStateOf("") }
    val focusRequester = remember { FocusRequester() }

    ModalBottomSheet(
        onDismissRequest = onDismiss,
        sheetState = sheetState,
        windowInsets = WindowInsets.ime   // ✅ sheet moves above keyboard
    ) {
        Column(
            modifier = Modifier
                .fillMaxWidth()
                .padding(AppSpacing.md)
                .imePadding()             // ✅ extra safety for content inside
        ) {
            OutlinedTextField(
                value = text,
                onValueChange = { text = it },
                label = { Text("Item name") },
                modifier = Modifier
                    .fillMaxWidth()
                    .focusRequester(focusRequester),
                keyboardOptions = KeyboardOptions(imeAction = ImeAction.Done),
                keyboardActions = KeyboardActions(onDone = { onAdd(text) })
            )
            Spacer(Modifier.height(AppSpacing.md))
            Button(
                onClick = { onAdd(text) },
                modifier = Modifier.fillMaxWidth()
            ) {
                Text("Add")
            }
        }
    }

    LaunchedEffect(Unit) {
        focusRequester.requestFocus()
    }
}
```

---

## Scrollable Content Inside Sheet

```kotlin
// ✅ Nested scroll in modal bottom sheet
@Composable
fun FilterSheet(
    filters: List<Filter>,
    onDismiss: () -> Unit
) {
    val sheetState = rememberModalBottomSheetState()

    ModalBottomSheet(
        onDismissRequest = onDismiss,
        sheetState = sheetState
    ) {
        LazyColumn(
            modifier = Modifier
                .fillMaxWidth()
                .heightIn(max = 400.dp),  // ✅ cap height for large lists
            contentPadding = PaddingValues(
                start = AppSpacing.md,
                end = AppSpacing.md,
                bottom = AppSpacing.lg
            )
        ) {
            items(filters) { filter ->
                FilterItem(filter)
            }
        }
    }
}
```

---

## Persistent Bottom Sheet (BottomSheetScaffold)

```kotlin
// ✅ Persistent sheet coexists with screen content
@Composable
fun MapScreen() {
    val scaffoldState = rememberBottomSheetScaffoldState(
        bottomSheetState = rememberStandardBottomSheetState(
            initialValue = SheetValue.PartiallyExpanded
        )
    )

    BottomSheetScaffold(
        scaffoldState = scaffoldState,
        sheetContent = {
            Column(
                modifier = Modifier
                    .fillMaxWidth()
                    .padding(AppSpacing.md)
            ) {
                Text("Location Details", style = MaterialTheme.typography.titleMedium)
                // sheet content
            }
        },
        sheetPeekHeight = 120.dp    // ✅ height when partially expanded
    ) { padding ->
        // main screen content (map)
        Box(modifier = Modifier.padding(padding)) { ... }
    }
}
```

---

## Side Sheet (Custom)

```kotlin
// ✅ Side sheet via AnimatedVisibility — for tablets or landscape
@Composable
fun SideSheet(
    visible: Boolean,
    onDismiss: () -> Unit,
    content: @Composable () -> Unit
) {
    AnimatedVisibility(
        visible = visible,
        enter = slideInHorizontally(initialOffsetX = { it }),
        exit = slideOutHorizontally(targetOffsetX = { it })
    ) {
        Surface(
            modifier = Modifier
                .fillMaxHeight()
                .width(320.dp)
                .align(Alignment.End),  // in a Box parent
            tonalElevation = 4.dp,
            shadowElevation = 4.dp
        ) {
            Column(modifier = Modifier.padding(AppSpacing.md)) {
                content()
            }
        }
    }
}

// ✅ Wrap in a Box at screen level
Box(modifier = Modifier.fillMaxSize()) {
    MainContent()
    SideSheet(visible = showSideSheet, onDismiss = { showSideSheet = false }) {
        FilterPanel()
    }
}
```

---

## Programmatic Sheet Control

```kotlin
// ✅ Expand / collapse programmatically
val sheetState = rememberModalBottomSheetState()
val scope = rememberCoroutineScope()

Button(onClick = {
    scope.launch { sheetState.expand() }
}) { Text("Expand") }

Button(onClick = {
    scope.launch { sheetState.hide() }
}) { Text("Hide") }
```

---

## Anti-Patterns

- Managing `SheetState` inside the sheet composable — hoist it to the caller
- Putting a `ModalBottomSheet` inside another `ModalBottomSheet`
- Not adding `imePadding()` or `windowInsets = WindowInsets.ime` in sheets with text fields
- Using `Dialog` instead of `ModalBottomSheet` for action menus on mobile
- Not capping height on scrollable sheet content — sheet can fill the entire screen

---

## Related Skills
- `compose` — Compose layout and animation fundamentals
- `keyboard-navigation` — IME handling inside sheets
- `material3` — M3 surface and elevation tokens
- `navigation` — navigating to/from content shown in sheets
