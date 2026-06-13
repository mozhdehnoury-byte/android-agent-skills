---
name: user-friendly-errors
description: >
  Displaying user-friendly error messages in Android Compose UI.
  Load this skill when designing error states in composables,
  writing error string resources, building reusable error components,
  or deciding between inline errors, snackbars, and full error screens.
---

# User Friendly Errors

## Overview
User-friendly errors translate technical failures into clear, actionable messages. The goal is to tell the user what happened, why it matters, and what they can do. Error display varies by context: inline field errors, snackbars for transient failures, and full error screens for blocking failures.

---

## Core Principles

- **Clear language** — no technical jargon; no stack traces shown to users
- **Actionable** — tell the user what to do, not just what went wrong
- **Contextual** — field-level errors inline; screen-level errors as states; transient as snackbars
- **Consistent** — same error type always looks the same across screens
- **Honest** — don't overpromise; "Try again" only when retry might work

---

## Error Display Decision

```
Error type           → Display method
─────────────────────────────────────────────────────
Form validation      → Inline below field
Transient (network)  → Snackbar with action
Blocking (load fail) → Full error state in screen
Permission denied    → Inline message, no retry
Session expired      → Navigate to login
Critical (corrupt)   → Dialog, then navigate away
```

---

## Error String Resources

```xml
<!-- res/values/strings_errors.xml -->
<resources>
    <!-- Network -->
    <string name="error_no_connection">No internet connection. Please check your network.</string>
    <string name="error_timeout">Request timed out. Please try again.</string>
    <string name="error_server">Server error. Please try again later.</string>
    <string name="error_session_expired">Your session has expired. Please log in again.</string>
    <string name="error_no_permission">You don\'t have permission to do this.</string>
    <string name="error_not_found">The item you\'re looking for doesn\'t exist.</string>
    <string name="error_rate_limited">Too many requests. Please wait a moment and try again.</string>

    <!-- Data -->
    <string name="error_item_not_found">Item not found.</string>
    <string name="error_conflict">"%1$s" already exists.</string>
    <string name="error_invalid_state">This action isn\'t available right now.</string>

    <!-- Storage -->
    <string name="error_disk_full">Your device storage is full. Free up space and try again.</string>
    <string name="error_data_corrupted">Some data was corrupted. Please reinstall the app.</string>

    <!-- Generic -->
    <string name="error_unexpected">Something went wrong. Please try again.</string>
    <string name="error_retry">Try Again</string>
    <string name="error_dismiss">Dismiss</string>
</resources>
```

---

## Reusable Error Components

```kotlin
// ✅ Full-screen error state
@Composable
fun FullScreenError(
    message: String,
    onRetry: (() -> Unit)? = null,
    modifier: Modifier = Modifier
) {
    Column(
        modifier = modifier
            .fillMaxSize()
            .padding(horizontal = 32.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        Icon(
            imageVector = Icons.Outlined.ErrorOutline,
            contentDescription = null,
            modifier = Modifier.size(64.dp),
            tint = MaterialTheme.colorScheme.error
        )
        Spacer(Modifier.height(16.dp))
        Text(
            text = message,
            style = MaterialTheme.typography.bodyLarge,
            textAlign = TextAlign.Center,
            color = MaterialTheme.colorScheme.onSurface
        )
        if (onRetry != null) {
            Spacer(Modifier.height(24.dp))
            Button(onClick = onRetry) {
                Text(stringResource(R.string.error_retry))
            }
        }
    }
}

// ✅ Inline error — below a field
@Composable
fun FieldError(
    message: String,
    modifier: Modifier = Modifier
) {
    Text(
        text = message,
        style = MaterialTheme.typography.bodySmall,
        color = MaterialTheme.colorScheme.error,
        modifier = modifier.padding(start = 16.dp, top = 4.dp)
    )
}

// ✅ Error banner — non-blocking top/bottom banner
@Composable
fun ErrorBanner(
    message: String,
    onDismiss: (() -> Unit)? = null,
    modifier: Modifier = Modifier
) {
    Surface(
        modifier = modifier.fillMaxWidth(),
        color = MaterialTheme.colorScheme.errorContainer,
        tonalElevation = 2.dp
    ) {
        Row(
            modifier = Modifier.padding(horizontal = 16.dp, vertical = 12.dp),
            verticalAlignment = Alignment.CenterVertically
        ) {
            Icon(
                imageVector = Icons.Outlined.Warning,
                contentDescription = null,
                tint = MaterialTheme.colorScheme.onErrorContainer,
                modifier = Modifier.size(20.dp)
            )
            Spacer(Modifier.width(12.dp))
            Text(
                text = message,
                style = MaterialTheme.typography.bodyMedium,
                color = MaterialTheme.colorScheme.onErrorContainer,
                modifier = Modifier.weight(1f)
            )
            if (onDismiss != null) {
                IconButton(onClick = onDismiss, modifier = Modifier.size(24.dp)) {
                    Icon(Icons.Default.Close, contentDescription = stringResource(R.string.error_dismiss))
                }
            }
        }
    }
}
```

---

## Snackbar Error Pattern

```kotlin
// ✅ Transient error via snackbar
@Composable
fun ProductScreen(
    viewModel: ProductViewModel = hiltViewModel()
) {
    val snackbarHostState = remember { SnackbarHostState() }
    val context = LocalContext.current

    // Collect one-time error events
    LaunchedEffect(Unit) {
        viewModel.events.collect { event ->
            when (event) {
                is ProductEvent.ShowError -> {
                    val result = snackbarHostState.showSnackbar(
                        message = event.message,
                        actionLabel = if (event.canRetry) context.getString(R.string.error_retry) else null,
                        duration = SnackbarDuration.Long
                    )
                    if (result == SnackbarResult.ActionPerformed && event.canRetry) {
                        viewModel.onRetry()
                    }
                }
            }
        }
    }

    Scaffold(snackbarHost = { SnackbarHost(snackbarHostState) }) { padding ->
        ProductContent(modifier = Modifier.padding(padding))
    }
}
```

---

## Form Validation Errors

```kotlin
// ✅ Inline validation errors on form fields
@Composable
fun EmailField(
    value: String,
    error: String?,
    onValueChange: (String) -> Unit,
    modifier: Modifier = Modifier
) {
    Column(modifier = modifier) {
        OutlinedTextField(
            value = value,
            onValueChange = onValueChange,
            label = { Text("Email") },
            isError = error != null,
            supportingText = {
                if (error != null) {
                    Text(
                        text = error,
                        color = MaterialTheme.colorScheme.error
                    )
                }
            },
            keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Email),
            singleLine = true,
            modifier = Modifier.fillMaxWidth()
        )
    }
}

// ✅ UiState with per-field errors
data class RegistrationUiState(
    val email: String = "",
    val password: String = "",
    val isLoading: Boolean = false,
    val emailError: String? = null,
    val passwordError: String? = null,
    val generalError: String? = null
)
```

---

## AppError → User Message Mapping

```kotlin
// ✅ Centralized mapping — use string resources
@Composable
fun AppError.toMessage(): String = when (this) {
    is AppError.Network.NoConnection  -> stringResource(R.string.error_no_connection)
    is AppError.Network.Timeout       -> stringResource(R.string.error_timeout)
    is AppError.Network.Unauthorized  -> stringResource(R.string.error_session_expired)
    is AppError.Network.Forbidden     -> stringResource(R.string.error_no_permission)
    is AppError.Network.NotFound      -> stringResource(R.string.error_not_found)
    is AppError.Network.ServerError   -> stringResource(R.string.error_server)
    is AppError.Network.RateLimited   -> stringResource(R.string.error_rate_limited)
    is AppError.Data.NotFound         -> stringResource(R.string.error_item_not_found)
    is AppError.Data.Conflict         -> stringResource(R.string.error_conflict, field)
    is AppError.Data.Validation       -> errors.values.first()
    is AppError.Data.InvalidState     -> reason
    is AppError.Storage.DiskFull      -> stringResource(R.string.error_disk_full)
    is AppError.Storage.Corrupted     -> stringResource(R.string.error_data_corrupted)
    is AppError.Storage.Unknown,
    is AppError.Unexpected            -> stringResource(R.string.error_unexpected)
    else                              -> stringResource(R.string.error_unexpected)
}

// ✅ Non-composable mapping (for ViewModel)
fun AppError.toMessageResId(): Int = when (this) {
    is AppError.Network.NoConnection -> R.string.error_no_connection
    is AppError.Network.Timeout      -> R.string.error_timeout
    is AppError.Network.Unauthorized -> R.string.error_session_expired
    // ...
    else                             -> R.string.error_unexpected
}
```

---

## Screen-Level Error Handling Pattern

```kotlin
// ✅ Composable that handles all UiState cases including error
@Composable
fun ProductListScreen(
    viewModel: ProductListViewModel = hiltViewModel()
) {
    val state by viewModel.state.collectAsStateWithLifecycle()

    when (val s = state) {
        is ProductListUiState.Loading -> {
            LoadingIndicator(modifier = Modifier.fillMaxSize())
        }
        is ProductListUiState.Success -> {
            ProductList(
                products = s.products,
                onProductClick = viewModel::onProductClick
            )
        }
        is ProductListUiState.Error -> {
            FullScreenError(
                message = s.message,
                onRetry = if (s.canRetry) viewModel::loadProducts else null
            )
        }
    }
}
```

---

## Anti-Patterns

- Showing raw exception messages (`e.message`) to users — technical and unhelpful
- Same snackbar for every error — `Unauthorized` needs login redirect, not a snackbar
- Error messages without action — "Error occurred" with no retry button frustrates users
- Hardcoding error strings in code — use string resources for localization
- Generic "Something went wrong" for every error — obscures actionable information

---

## Related Skills
- `domain-error-model` — typed errors being displayed
- `error-handling` — how errors reach the UI
- `failure-strategy` — which display method to use per error
- `state-management` — modeling error state in UiState
- `compose` — Compose fundamentals for error composables
- `material3` — error colors and components from Material 3
