---
name: design-system
description: >
  Building and maintaining a custom design system on top of Material 3 in
  Jetpack Compose. Load this skill when defining custom design tokens,
  extending the M3 theme, creating a component library, managing spacing
  scales, or enforcing visual consistency across the app.
---

# Design System

## Overview

A design system extends Material 3 with project-specific tokens (colors, spacing, typography, elevation) and a reusable component library. It acts as the single source of truth for all visual decisions, ensuring consistency and reducing duplication across features.

---

## Core Principles

- Extend M3 — never replace it; build on top of `MaterialTheme`
- Define all tokens once — consume them everywhere via `CompositionLocal`
- Components in the design system must have no business logic
- Every design system component must support preview in both light/dark and LTR/RTL
- Tokens must be sealed objects or typed wrappers — never raw `Dp` or `Color` literals in UI code

---

## Token Structure

```kotlin
// ✅ Spacing tokens
object AppSpacing {
    val xs  = 4.dp
    val sm  = 8.dp
    val md  = 16.dp
    val lg  = 24.dp
    val xl  = 32.dp
    val xxl = 48.dp
}

// ✅ Elevation tokens
object AppElevation {
    val none   = 0.dp
    val low    = 2.dp
    val medium = 4.dp
    val high   = 8.dp
}

// ✅ Icon size tokens
object AppIconSize {
    val sm = 16.dp
    val md = 24.dp
    val lg = 32.dp
}
```

---

## Extending MaterialTheme

```kotlin
// ✅ Custom theme extension via CompositionLocal
data class AppColors(
    val success: Color,
    val onSuccess: Color,
    val warning: Color,
    val onWarning: Color,
    val info: Color,
    val onInfo: Color
)

val LocalAppColors = staticCompositionLocalOf {
    AppColors(
        success = Color(0xFF2E7D32),
        onSuccess = Color.White,
        warning = Color(0xFFF57F17),
        onWarning = Color.Black,
        info = Color(0xFF0277BD),
        onInfo = Color.White
    )
}

// ✅ Access extension colors
val appColors = LocalAppColors.current
Box(modifier = Modifier.background(appColors.success))
```

---

## Theme Provider

```kotlin
// ✅ Provide all custom locals alongside MaterialTheme
@Composable
fun AppTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    content: @Composable () -> Unit
) {
    val appColors = if (darkTheme) darkAppColors() else lightAppColors()

    CompositionLocalProvider(
        LocalAppColors provides appColors
    ) {
        MaterialTheme(
            colorScheme = if (darkTheme) DarkColorScheme else LightColorScheme,
            typography = AppTypography,
            shapes = AppShapes,
            content = content
        )
    }
}
```

---

## Design System Component Pattern

```kotlin
// ✅ Atomic component — no business logic, fully configurable
@Composable
fun AppButton(
    text: String,
    onClick: () -> Unit,
    modifier: Modifier = Modifier,
    enabled: Boolean = true,
    loading: Boolean = false,
    style: AppButtonStyle = AppButtonStyle.Filled
) {
    val colors = when (style) {
        AppButtonStyle.Filled  -> ButtonDefaults.buttonColors()
        AppButtonStyle.Tonal   -> ButtonDefaults.filledTonalButtonColors()
        AppButtonStyle.Outline -> ButtonDefaults.outlinedButtonColors()
    }

    Button(
        onClick = onClick,
        modifier = modifier,
        enabled = enabled && !loading,
        colors = colors
    ) {
        if (loading) {
            CircularProgressIndicator(
                modifier = Modifier.size(AppIconSize.sm),
                strokeWidth = 2.dp
            )
            Spacer(Modifier.width(AppSpacing.sm))
        }
        Text(text)
    }
}

enum class AppButtonStyle { Filled, Tonal, Outline }
```

---

## Typography Extension

```kotlin
// ✅ Custom text styles beyond the M3 scale
object AppTextStyle {
    val code = TextStyle(
        fontFamily = FontFamily.Monospace,
        fontSize = 13.sp,
        lineHeight = 20.sp
    )
    val caption = TextStyle(
        fontSize = 11.sp,
        lineHeight = 16.sp,
        color = Color.Unspecified  // inherit from context
    )
}
```

---

## Component Catalog Preview

```kotlin
// ✅ Every component has a catalog preview
@Preview(name = "Light", uiMode = UI_MODE_NIGHT_NO)
@Preview(name = "Dark", uiMode = UI_MODE_NIGHT_YES)
@Preview(name = "RTL", locale = "fa")
@Composable
private fun AppButtonPreview() {
    AppTheme {
        Column(
            modifier = Modifier.padding(AppSpacing.md),
            verticalArrangement = Arrangement.spacedBy(AppSpacing.sm)
        ) {
            AppButton("Filled", onClick = {}, style = AppButtonStyle.Filled)
            AppButton("Tonal", onClick = {}, style = AppButtonStyle.Tonal)
            AppButton("Loading", onClick = {}, loading = true)
            AppButton("Disabled", onClick = {}, enabled = false)
        }
    }
}
```

---

## Anti-Patterns

- Using raw `Color(0xFF...)` literals in UI composables — use tokens
- Putting business logic inside design system components
- Overriding `MaterialTheme` colors directly via `MaterialTheme.copy()` in a nested scope — use `CompositionLocalProvider` instead
- Defining spacing inline (`Modifier.padding(16.dp)`) — use `AppSpacing.md`
- Mixing M2 and M3 components in the same design system

---

## Related Skills

- `material3` — M3 foundation this system builds on
- `compose` — Compose fundamentals
- `rtl` — directional token and layout requirements
- `resources` — XML resources for legacy interop
