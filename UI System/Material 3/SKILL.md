---
name: material3
description: >
  Material Design 3 theming, components, and color system for Android/Compose.
  Load this skill when setting up M3 theme, using M3 components, defining
  custom color schemes, typography, or shapes, or supporting dynamic color.
---

# Material 3

## Overview
Material Design 3 (M3) is Google's latest design system, built into Jetpack Compose via `androidx.compose.material3`. It provides a comprehensive color system, typography scale, shape system, and a full set of UI components. M3 supports dynamic color (Android 12+) and dark/light themes natively.

---

## Core Principles

- Always use **M3 semantic color tokens** — never hardcode colors
- Define **one MaterialTheme** at the root — never nest multiple themes
- Use M3 components before building custom — they handle accessibility and state
- Support **dark mode** from day one — design the color scheme for both
- Use **dynamic color** where available (API 31+) with a fallback palette

---

## Theme Setup

```kotlin
// ✅ Theme definition
@Composable
fun AppTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    dynamicColor: Boolean = true,
    content: @Composable () -> Unit
) {
    val colorScheme = when {
        dynamicColor && Build.VERSION.SDK_INT >= Build.VERSION_CODES.S -> {
            val context = LocalContext.current
            if (darkTheme) dynamicDarkColorScheme(context)
            else dynamicLightColorScheme(context)
        }
        darkTheme -> DarkColorScheme
        else      -> LightColorScheme
    }

    MaterialTheme(
        colorScheme = colorScheme,
        typography = AppTypography,
        shapes = AppShapes,
        content = content
    )
}

// ✅ Apply at root
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            AppTheme {
                AppNavHost()
            }
        }
    }
}
```

---

## Color Scheme

```kotlin
// ✅ Define light and dark color schemes
private val LightColorScheme = lightColorScheme(
    primary = Purple40,
    onPrimary = Color.White,
    primaryContainer = Purple90,
    onPrimaryContainer = Purple10,
    secondary = PurpleGrey40,
    onSecondary = Color.White,
    secondaryContainer = PurpleGrey90,
    onSecondaryContainer = PurpleGrey10,
    tertiary = Pink40,
    background = Color(0xFFFFFBFE),
    surface = Color(0xFFFFFBFE),
    onBackground = Color(0xFF1C1B1F),
    onSurface = Color(0xFF1C1B1F),
    error = Color(0xFFB3261E)
)

private val DarkColorScheme = darkColorScheme(
    primary = Purple80,
    onPrimary = Purple20,
    primaryContainer = Purple30,
    onPrimaryContainer = Purple90,
    // ...
)
```

---

## Color Token Usage

```kotlin
// ✅ Always use MaterialTheme.colorScheme tokens
@Composable
fun PrimaryButton(text: String, onClick: () -> Unit) {
    Button(
        onClick = onClick,
        colors = ButtonDefaults.buttonColors(
            containerColor = MaterialTheme.colorScheme.primary,
            contentColor = MaterialTheme.colorScheme.onPrimary
        )
    ) {
        Text(text)
    }
}

// ✅ Surface with correct tonal elevation
Surface(
    color = MaterialTheme.colorScheme.surface,
    tonalElevation = 2.dp
) { ... }

// ❌ Never hardcode colors
Text(text = "Hello", color = Color(0xFF6200EE))
```

### M3 Color Roles Reference

| Role | On Role | Use For |
|------|---------|---------|
| `primary` | `onPrimary` | Main action, FAB |
| `primaryContainer` | `onPrimaryContainer` | Selected state, chips |
| `secondary` | `onSecondary` | Supporting actions |
| `tertiary` | `onTertiary` | Contrasting accents |
| `surface` | `onSurface` | Cards, sheets, dialogs |
| `surfaceVariant` | `onSurfaceVariant` | Input fields, chips |
| `background` | `onBackground` | Screen background |
| `error` | `onError` | Error states |

---

## Typography

```kotlin
// ✅ Define custom typography
val AppTypography = Typography(
    displayLarge = TextStyle(
        fontFamily = FontFamily.Default,
        fontWeight = FontWeight.Normal,
        fontSize = 57.sp,
        lineHeight = 64.sp
    ),
    titleLarge = TextStyle(
        fontFamily = FontFamily.Default,
        fontWeight = FontWeight.Normal,
        fontSize = 22.sp,
        lineHeight = 28.sp
    ),
    bodyLarge = TextStyle(
        fontFamily = FontFamily.Default,
        fontWeight = FontWeight.Normal,
        fontSize = 16.sp,
        lineHeight = 24.sp
    ),
    labelSmall = TextStyle(
        fontFamily = FontFamily.Default,
        fontWeight = FontWeight.Medium,
        fontSize = 11.sp,
        lineHeight = 16.sp
    )
)

// ✅ Use typography tokens
Text(text = "Title", style = MaterialTheme.typography.titleLarge)
Text(text = "Body text", style = MaterialTheme.typography.bodyMedium)
Text(text = "Label", style = MaterialTheme.typography.labelSmall)
```

---

## Shapes

```kotlin
// ✅ Define shape scale
val AppShapes = Shapes(
    extraSmall = RoundedCornerShape(4.dp),
    small       = RoundedCornerShape(8.dp),
    medium      = RoundedCornerShape(12.dp),
    large       = RoundedCornerShape(16.dp),
    extraLarge  = RoundedCornerShape(28.dp)
)

// ✅ Use shape tokens
Card(shape = MaterialTheme.shapes.medium) { ... }
Button(shape = MaterialTheme.shapes.extraLarge) { ... }  // pill shape
```

---

## Key M3 Components

```kotlin
// ✅ Buttons
Button(onClick = {}) { Text("Filled") }
OutlinedButton(onClick = {}) { Text("Outlined") }
TextButton(onClick = {}) { Text("Text") }
FilledTonalButton(onClick = {}) { Text("Tonal") }
ElevatedButton(onClick = {}) { Text("Elevated") }

// ✅ FAB
FloatingActionButton(onClick = {}) {
    Icon(Icons.Default.Add, contentDescription = "Add")
}
ExtendedFloatingActionButton(
    text = { Text("New item") },
    icon = { Icon(Icons.Default.Add, contentDescription = null) },
    onClick = {}
)

// ✅ Cards
Card(modifier = Modifier.fillMaxWidth()) { ... }
ElevatedCard { ... }
OutlinedCard { ... }

// ✅ Top App Bar
TopAppBar(
    title = { Text("Screen Title") },
    navigationIcon = {
        IconButton(onClick = onBack) {
            Icon(Icons.Default.ArrowBack, contentDescription = "Back")
        }
    },
    actions = {
        IconButton(onClick = {}) {
            Icon(Icons.Default.MoreVert, contentDescription = "More")
        }
    }
)

// ✅ Navigation Bar
NavigationBar {
    items.forEach { item ->
        NavigationBarItem(
            icon = { Icon(item.icon, contentDescription = null) },
            label = { Text(item.label) },
            selected = currentRoute == item.route,
            onClick = { onNavigate(item.route) }
        )
    }
}

// ✅ Snackbar
val snackbarHostState = remember { SnackbarHostState() }
Scaffold(snackbarHost = { SnackbarHost(snackbarHostState) }) { padding ->
    // content
}
LaunchedEffect(Unit) {
    snackbarHostState.showSnackbar("Item deleted", actionLabel = "Undo")
}
```

---

## Anti-Patterns

- Hardcoding colors instead of using `MaterialTheme.colorScheme` tokens
- Nesting multiple `MaterialTheme` composables — causes inconsistent theming
- Using `androidx.compose.material` (M2) components mixed with M3 — conflicts
- Not providing a non-dynamic fallback color scheme for API < 31
- Using `Color.White`/`Color.Black` directly — breaks dark mode

---

## Related Skills
- `compose` — Compose fundamentals
- `design-system` — extending M3 with custom design tokens
- `resources` — color and style resources for XML interop
- `rtl` — RTL support in M3 components
