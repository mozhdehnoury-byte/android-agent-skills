---
name: rtl
description: >
  Right-to-Left (RTL) layout support in Android and Jetpack Compose.
  Load this skill when building layouts that must support RTL languages
  (Arabic, Persian, Hebrew), mirroring icons and directions, handling
  bidirectional text, or ensuring correct layout mirroring behavior.
---

# RTL

## Overview
RTL (Right-to-Left) support ensures layouts mirror correctly for languages like Persian, Arabic, and Hebrew. In Compose, this is handled via `LayoutDirection` and logical layout modifiers. Correct RTL support requires using directional-agnostic APIs throughout — never hardcode left/right values.

---

## Core Principles

- Always use **logical layout modifiers** — never physical (`start/end`, not `left/right`)
- Set `android:supportsRtl="true"` in the manifest
- Use `Arrangement.Start`/`Alignment.Start` — not `Left`/`Right`
- Mirror directional icons explicitly using `autoMirrored = true`
- Test in both LTR and RTL using the developer options toggle

---

## Manifest Setup

```xml
<!-- ✅ Required in AndroidManifest.xml -->
<application
    android:supportsRtl="true"
    ... >
```

---

## Compose Layout Direction

```kotlin
// ✅ Read current layout direction
@Composable
fun DirectionAwareIcon() {
    val layoutDirection = LocalLayoutDirection.current
    val isRtl = layoutDirection == LayoutDirection.Rtl
}

// ✅ Force RTL for a subtree
CompositionLocalProvider(LocalLayoutDirection provides LayoutDirection.Rtl) {
    MyLayout()
}

// ✅ Force LTR for specific content (e.g. phone numbers, codes)
CompositionLocalProvider(LocalLayoutDirection provides LayoutDirection.Ltr) {
    Text(text = "+98 912 000 0000")
}
```

---

## Logical Modifiers

```kotlin
// ✅ Use padding start/end — not left/right
Modifier.padding(start = 16.dp, end = 8.dp)

// ✅ Use absolutePadding only when truly position-fixed
Modifier.absolutePadding(left = 0.dp)  // rare — only for non-mirrored content

// ✅ Row arrangement
Row(horizontalArrangement = Arrangement.Start) { ... }  // ✅
Row(horizontalArrangement = Arrangement.End) { ... }    // ✅

// ❌ Never use physical directions in layout
Modifier.padding(left = 16.dp)   // breaks RTL
Modifier.padding(right = 8.dp)   // breaks RTL
```

---

## Icon Mirroring

```kotlin
// ✅ Use autoMirrored for directional icons (arrows, back, forward)
Icon(
    painter = painterResource(R.drawable.ic_arrow_forward),
    contentDescription = null,
    modifier = Modifier.scale(
        scaleX = if (LocalLayoutDirection.current == LayoutDirection.Rtl) -1f else 1f,
        scaleY = 1f
    )
)

// ✅ Better — use autoMirrored in vector drawable XML
// In ic_arrow_forward.xml:
// android:autoMirrored="true"

// ✅ Icons that should NOT be mirrored
Icon(Icons.Default.PlayArrow, contentDescription = null)   // media control — don't mirror
Icon(Icons.Default.Search, contentDescription = null)      // symmetric — don't mirror
```

---

## Bidirectional Text

```kotlin
// ✅ Let the system handle bidi text automatically
Text(text = "Hello سلام")  // system resolves direction per paragraph

// ✅ Force text direction when needed
Text(
    text = userInput,
    textDirection = TextDirection.Content  // auto-detect per content
)

// ✅ Force LTR for codes, emails, URLs
Text(
    text = "user@example.com",
    textDirection = TextDirection.Ltr
)

// ✅ TextField with explicit direction for mixed input
TextField(
    value = value,
    onValueChange = onValueChange,
    keyboardOptions = KeyboardOptions(imeAction = ImeAction.Done),
    modifier = Modifier.semantics { textDirection = TextDirection.Content }
)
```

---

## Alignment Helpers

```kotlin
// ✅ Alignment.Start mirrors automatically
Box(contentAlignment = Alignment.TopStart) { ... }  // top-left in LTR, top-right in RTL
Box(contentAlignment = Alignment.TopEnd) { ... }    // top-right in LTR, top-left in RTL

// ✅ Column horizontal alignment
Column(horizontalAlignment = Alignment.Start) { ... }
Column(horizontalAlignment = Alignment.End) { ... }
```

---

## Previewing RTL

```kotlin
// ✅ Always add an RTL preview alongside LTR
@Preview(name = "LTR", locale = "en")
@Preview(name = "RTL", locale = "fa")
@Composable
fun MyComponentPreview() {
    AppTheme {
        MyComponent()
    }
}
```

---

## Anti-Patterns

- Using `Modifier.padding(left/right)` — breaks RTL mirroring
- Using `Arrangement.AbsoluteLeft` or `AbsoluteRight` in normal layouts
- Hardcoding `LayoutDirection.Ltr` at the root — prevents RTL support
- Not setting `android:supportsRtl="true"` in manifest
- Mirroring media/playback icons (play, pause, record) — they are universal
- Using `Column { Modifier.align(Alignment.AbsoluteLeft) }` — physical alignment

---

## Related Skills
- `compose` — Compose layout fundamentals
- `material3` — M3 component RTL behavior
- `resources` — string direction and locale resources
- `design-system` — design tokens with directional awareness
