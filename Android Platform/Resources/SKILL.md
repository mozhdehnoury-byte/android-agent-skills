---
name: resources
description: >
  Android resource system — strings, colors, dimensions, drawables, styles, and qualifiers.
  Load this skill when working with res/ directory, defining or accessing resources,
  supporting multiple screen sizes, locales, or themes.
---

# Resources

## Overview

Android resources are external files and static values used by the app — strings, colors, dimensions, drawables, layouts, styles, and more. They live in `res/` and are accessed via generated `R` class. Resources support qualifiers that allow automatic selection based on device configuration.

---

## Core Principles

- Never hardcode strings, colors, or dimensions in code or layouts — always use resources
- Use qualifiers for configuration-specific resources — don't detect configuration in code
- All user-facing strings must be in `strings.xml` — enables localization
- Colors must be defined in `colors.xml` and referenced via theme attributes — not directly
- Dimensions must be in `dimens.xml` — enables density-independent scaling

---

## Directory Structure

```
res/
├── drawable/          ← vector drawables, shape drawables
├── drawable-xxhdpi/   ← raster images (prefer vectors)
├── layout/            ← XML layouts (if using View system)
├── mipmap-*/          ← launcher icons only
├── values/
│   ├── strings.xml
│   ├── colors.xml
│   ├── dimens.xml
│   ├── styles.xml
│   └── themes.xml
├── values-night/      ← dark theme overrides
├── values-fa/         ← Persian strings
├── values-land/       ← landscape overrides
└── xml/               ← network_security_config, file_paths, etc.
```

---

## Strings

```xml
<!-- ✅ strings.xml -->
<resources>
    <!-- Simple string -->
    <string name="app_name">MyApp</string>
    <string name="action_save">Save</string>

    <!-- ✅ Formatted string with argument -->
    <string name="welcome_message">Welcome, %1$s!</string>

    <!-- ✅ Plural strings -->
    <plurals name="item_count">
        <item quantity="one">%d item</item>
        <item quantity="other">%d items</item>
    </plurals>

    <!-- ✅ String array -->
    <string-array name="days_of_week">
        <item>Monday</item>
        <item>Tuesday</item>
    </string-array>
</resources>
```

```kotlin
// ✅ Accessing strings in code
val welcome = getString(R.string.welcome_message, userName)
val itemCount = resources.getQuantityString(R.plurals.item_count, count, count)

// ✅ In Compose
Text(text = stringResource(R.string.welcome_message, userName))
Text(text = pluralStringResource(R.plurals.item_count, count, count))
```

---

## Colors

```xml
<!-- ✅ colors.xml — raw color palette -->
<resources>
    <color name="purple_500">#FF6200EE</color>
    <color name="purple_700">#FF3700B3</color>
    <color name="teal_200">#FF03DAC5</color>
    <color name="white">#FFFFFFFF</color>
    <color name="black">#FF000000</color>
</resources>

<!-- ✅ themes.xml — semantic color attributes referencing palette -->
<style name="Theme.App" parent="Theme.Material3.DayNight">
    <item name="colorPrimary">@color/purple_500</item>
    <item name="colorOnPrimary">@color/white</item>
    <item name="colorSurface">@color/white</item>
</style>

<!-- ✅ values-night/themes.xml — dark theme overrides -->
<style name="Theme.App" parent="Theme.Material3.DayNight">
    <item name="colorPrimary">@color/purple_200</item>
    <item name="colorSurface">@color/black</item>
</style>
```

```kotlin
// ✅ Always access color via theme attribute — not directly
val color = MaterialColors.getColor(view, com.google.android.material.R.attr.colorPrimary)

// ✅ In Compose — use MaterialTheme tokens
Surface(color = MaterialTheme.colorScheme.surface) { ... }

// ❌ Never hardcode or access raw color directly in code
view.setBackgroundColor(Color.parseColor("#FF6200EE"))
```

---

## Dimensions

```xml
<!-- ✅ dimens.xml -->
<resources>
    <dimen name="spacing_small">8dp</dimen>
    <dimen name="spacing_medium">16dp</dimen>
    <dimen name="spacing_large">24dp</dimen>
    <dimen name="corner_radius">12dp</dimen>
    <dimen name="text_size_body">14sp</dimen>
    <dimen name="text_size_title">20sp</dimen>
</resources>
```

```kotlin
// ✅ Accessing dimensions
val spacing = resources.getDimensionPixelSize(R.dimen.spacing_medium)

// ✅ In Compose — use dp/sp directly or define as constants
val SpacingMedium = 16.dp
val TextSizeBody = 14.sp
```

---

## Drawables

```xml
<!-- ✅ Use vector drawables — scale perfectly on all densities -->
<!-- res/drawable/ic_arrow_back.xml -->
<vector xmlns:android="http://schemas.android.com/apk/res/android"
    android:width="24dp"
    android:height="24dp"
    android:viewportWidth="24"
    android:viewportHeight="24"
    android:tint="?attr/colorOnSurface">
    <path android:fillColor="@android:color/white"
        android:pathData="M20,11H7.83l5.59-5.59L12,4l-8,8 8,8 1.41-1.41L7.83,13H20v-2z"/>
</vector>
```

```
// ✅ Raster images — use only when vector isn't suitable
// Place in density buckets:
drawable-mdpi/    (1x)
drawable-hdpi/    (1.5x)
drawable-xhdpi/   (2x)
drawable-xxhdpi/  (3x)
drawable-xxxhdpi/ (4x)
```

---

## Configuration Qualifiers

```
values/              ← default
values-fa/           ← Persian locale
values-night/        ← dark mode
values-land/         ← landscape
values-sw600dp/      ← tablets (600dp+ smallest width)
values-v33/          ← API 33+

layout/              ← default layout
layout-land/         ← landscape layout
layout-sw600dp/      ← tablet layout
```

---

## Accessing Resources in Code

```kotlin
// ✅ From Activity/Fragment
val text = getString(R.string.app_name)
val color = ContextCompat.getColor(this, R.color.purple_500)
val drawable = ContextCompat.getDrawable(this, R.drawable.ic_arrow_back)
val dimen = resources.getDimensionPixelSize(R.dimen.spacing_medium)

// ✅ From ViewModel (via injected Context — use ApplicationContext only)
class MyViewModel(private val appContext: Context) : ViewModel() {
    fun getLabel(): String = appContext.getString(R.string.app_name)
}

// ✅ Compose
Text(stringResource(R.string.app_name))
Icon(painterResource(R.drawable.ic_arrow_back), contentDescription = null)
```

---

## Anti-Patterns

- Hardcoded strings in XML or code — breaks localization
- Accessing colors directly (`R.color.x`) instead of via theme attributes — breaks dark mode
- Using `px` instead of `dp`/`sp` in dimensions — breaks across densities
- Placing launcher icons in `drawable/` instead of `mipmap/` — incorrect scaling
- Using raster images where vectors would work — increases APK size
- Storing sensitive data (keys, tokens) as string resources — visible in decompiled APK
- Duplicate string keys — silent override, hard to debug

---

## Related Skills

- `manifest` — referencing resources in manifest
- `material3` — Material 3 theme and color system
- `rtl` — RTL-specific resource qualifiers
- `design-system` — design tokens as Android resources
