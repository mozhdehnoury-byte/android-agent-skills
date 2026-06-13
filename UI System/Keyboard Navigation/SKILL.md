---
name: keyboard-navigation
description: >
  Keyboard and IME (soft keyboard) navigation in Android/Compose.
  Load this skill when managing focus traversal between fields, handling
  IME actions (Next, Done, Search), avoiding content hidden behind the
  keyboard, managing keyboard visibility, or supporting hardware keyboards.
---

# Keyboard Navigation

## Overview
Keyboard navigation covers two concerns: soft keyboard (IME) management for touch devices, and hardware keyboard support for accessibility and productivity. Both require explicit focus management, correct IME action wiring, and ensuring content remains accessible when the keyboard is visible.

---

## Core Principles

- Always set `imeAction` to match the field's role (`Next` for mid-form, `Done`/`Search` for last)
- Never let the keyboard obscure active input fields — use `imePadding()` or `imeNestedScroll()`
- Use `FocusRequester` for programmatic focus — never simulate clicks
- Form fields must chain focus in logical order — left-to-right, top-to-bottom (LTR) or right-to-left (RTL)
- Request focus on first field only after the screen is fully composed

---

## IME Actions

```kotlin
// ✅ Chain fields with Next — last field uses Done/Search/Send
@Composable
fun LoginForm(onSubmit: () -> Unit) {
    val focusManager = LocalFocusManager.current
    var email by remember { mutableStateOf("") }
    var password by remember { mutableStateOf("") }

    Column(verticalArrangement = Arrangement.spacedBy(AppSpacing.md)) {
        OutlinedTextField(
            value = email,
            onValueChange = { email = it },
            label = { Text("Email") },
            keyboardOptions = KeyboardOptions(
                keyboardType = KeyboardType.Email,
                imeAction = ImeAction.Next       // ✅ moves focus to next field
            ),
            keyboardActions = KeyboardActions(
                onNext = { focusManager.moveFocus(FocusDirection.Down) }
            ),
            singleLine = true
        )

        OutlinedTextField(
            value = password,
            onValueChange = { password = it },
            label = { Text("Password") },
            visualTransformation = PasswordVisualTransformation(),
            keyboardOptions = KeyboardOptions(
                keyboardType = KeyboardType.Password,
                imeAction = ImeAction.Done       // ✅ last field — submit
            ),
            keyboardActions = KeyboardActions(
                onDone = {
                    focusManager.clearFocus()
                    onSubmit()
                }
            ),
            singleLine = true
        )
    }
}
```

---

## FocusRequester

```kotlin
// ✅ Programmatic focus after screen appears
@Composable
fun SearchScreen() {
    val focusRequester = remember { FocusRequester() }

    OutlinedTextField(
        modifier = Modifier.focusRequester(focusRequester),
        value = query,
        onValueChange = { query = it },
        keyboardOptions = KeyboardOptions(imeAction = ImeAction.Search),
        keyboardActions = KeyboardActions(onSearch = { onSearch(query) })
    )

    LaunchedEffect(Unit) {
        focusRequester.requestFocus()  // ✅ focus after composition
    }
}

// ✅ Chaining focus manually across fields
@Composable
fun MultiFieldForm() {
    val (firstFocus, secondFocus, thirdFocus) = remember { FocusRequester.createRefs() }

    OutlinedTextField(
        modifier = Modifier.focusRequester(firstFocus),
        keyboardActions = KeyboardActions(onNext = { secondFocus.requestFocus() }),
        keyboardOptions = KeyboardOptions(imeAction = ImeAction.Next),
        ...
    )
    OutlinedTextField(
        modifier = Modifier.focusRequester(secondFocus),
        keyboardActions = KeyboardActions(onNext = { thirdFocus.requestFocus() }),
        keyboardOptions = KeyboardOptions(imeAction = ImeAction.Next),
        ...
    )
    OutlinedTextField(
        modifier = Modifier.focusRequester(thirdFocus),
        keyboardOptions = KeyboardOptions(imeAction = ImeAction.Done),
        ...
    )
}
```

---

## Avoiding Content Hidden Behind Keyboard

```kotlin
// ✅ imePadding on scrollable containers
@Composable
fun FormScreen() {
    Column(
        modifier = Modifier
            .fillMaxSize()
            .verticalScroll(rememberScrollState())
            .imePadding()              // ✅ pushes content above keyboard
            .padding(AppSpacing.md)
    ) {
        // form fields
    }
}

// ✅ WindowInsets approach in Scaffold
Scaffold(
    modifier = Modifier.imePadding()
) { padding ->
    Column(modifier = Modifier.padding(padding)) { ... }
}

// ✅ For LazyColumn — imeNestedScroll + imePadding
LazyColumn(
    modifier = Modifier
        .fillMaxSize()
        .imePadding()
        .imeNestedScroll()
) { ... }
```

---

## Keyboard Visibility Control

```kotlin
// ✅ Show keyboard programmatically (Compose)
val keyboardController = LocalSoftwareKeyboardController.current

Button(onClick = { keyboardController?.show() }) { Text("Open Keyboard") }
Button(onClick = { keyboardController?.hide() }) { Text("Close Keyboard") }

// ✅ Hide keyboard on outside tap
@Composable
fun DismissKeyboardOnTap(content: @Composable () -> Unit) {
    val focusManager = LocalFocusManager.current
    Box(
        modifier = Modifier
            .fillMaxSize()
            .pointerInput(Unit) {
                detectTapGestures(onTap = { focusManager.clearFocus() })
            }
    ) {
        content()
    }
}
```

---

## Keyboard Type Reference

| KeyboardType | Use For |
|---|---|
| `Text` | General text input |
| `Email` | Email addresses |
| `Number` | Numeric only |
| `Phone` | Phone numbers |
| `Password` | Password (masked) |
| `NumberPassword` | PIN / numeric password |
| `Uri` | URLs |
| `Decimal` | Decimal numbers |

---

## Anti-Patterns

- Not setting `imeAction` — keyboard shows a random action button
- Using `ImeAction.Done` on non-last fields — breaks tab/next navigation
- Not adding `imePadding()` to scrollable forms — keyboard hides the active field
- Calling `requestFocus()` outside `LaunchedEffect` — crashes before composition completes
- Using `WindowManager.hideSoftInputFromWindow()` directly — deprecated, use `SoftwareKeyboardController`
- Setting `keyboardType = KeyboardType.Number` for phone numbers — use `Phone` type instead

---

## Related Skills
- `compose` — Compose fundamentals and Modifier system
- `bottom-sheet-pattern` — keyboard interaction with bottom sheets
- `accessibility` — focus order and hardware keyboard support
