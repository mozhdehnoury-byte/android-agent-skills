---
name: bitmap-optimization
description: >
  Bitmap loading, scaling, and memory optimization in Android.
  Load this skill when loading images efficiently, scaling bitmaps
  to required dimensions, using inSampleSize, managing Bitmap recycling,
  or reducing native heap usage from image decoding.
---

# Bitmap Optimization

## Overview
Bitmaps are the largest consumers of native heap on Android. A single 12MP photo decoded as ARGB_8888 uses ~48MB. Proper bitmap optimization involves decoding at the required size, choosing the right pixel format, and using a caching library (Coil, Glide) rather than manual management.

---

## Core Principles

- Never decode full-resolution bitmaps for thumbnail display — use `inSampleSize`
- Use **Coil** (or Glide) for image loading — they handle caching, sampling, and lifecycle
- Prefer `RGB_565` for images that don't need transparency — half the memory of `ARGB_8888`
- Recycle bitmaps manually only if not using a caching library
- Load bitmaps on a background thread — never on the main thread

---

## Coil — Recommended Image Loader

```toml
# libs.versions.toml
[libraries]
coil = { module = "io.coil-kt:coil-compose", version = "2.7.0" }
```

```kotlin
// ✅ Load image in Compose with Coil
@Composable
fun UserAvatar(imageUrl: String?, modifier: Modifier = Modifier) {
    AsyncImage(
        model = ImageRequest.Builder(LocalContext.current)
            .data(imageUrl)
            .crossfade(true)
            .size(64, 64)           // ✅ decode at display size — not full res
            .build(),
        contentDescription = null,
        contentScale = ContentScale.Crop,
        placeholder = painterResource(R.drawable.avatar_placeholder),
        error = painterResource(R.drawable.avatar_error),
        modifier = modifier
            .size(64.dp)
            .clip(CircleShape)
    )
}
```

---

## Manual Bitmap Sampling (inSampleSize)

```kotlin
// ✅ Decode at required size without loading full bitmap
fun decodeSampledBitmap(filePath: String, reqWidth: Int, reqHeight: Int): Bitmap {
    // First pass: get dimensions only
    val options = BitmapFactory.Options().apply {
        inJustDecodeBounds = true
    }
    BitmapFactory.decodeFile(filePath, options)

    // Calculate sample size
    options.inSampleSize = calculateInSampleSize(options, reqWidth, reqHeight)
    options.inJustDecodeBounds = false

    return BitmapFactory.decodeFile(filePath, options)
}

fun calculateInSampleSize(options: BitmapFactory.Options, reqWidth: Int, reqHeight: Int): Int {
    val height = options.outHeight
    val width = options.outWidth
    var inSampleSize = 1

    if (height > reqHeight || width > reqWidth) {
        val halfHeight = height / 2
        val halfWidth = width / 2
        while (halfHeight / inSampleSize >= reqHeight && halfWidth / inSampleSize >= reqWidth) {
            inSampleSize *= 2
        }
    }
    return inSampleSize
}
```

---

## Pixel Format Selection

```kotlin
// ✅ RGB_565 — half memory, no transparency
val options = BitmapFactory.Options().apply {
    inPreferredConfig = Bitmap.Config.RGB_565  // 2 bytes/pixel vs 4 bytes for ARGB_8888
}
val bitmap = BitmapFactory.decodeResource(resources, R.drawable.background, options)

// ✅ ARGB_8888 — full quality with alpha
val options = BitmapFactory.Options().apply {
    inPreferredConfig = Bitmap.Config.ARGB_8888
}

// Memory comparison for 1920x1080 image:
// ARGB_8888 = 1920 * 1080 * 4 = ~8MB
// RGB_565   = 1920 * 1080 * 2 = ~4MB
```

---

## Bitmap Reuse (inBitmap)

```kotlin
// ✅ Reuse existing bitmap memory — avoids new allocation
val reusableBitmap = Bitmap.createBitmap(width, height, Bitmap.Config.ARGB_8888)

val options = BitmapFactory.Options().apply {
    inMutable = true
    inBitmap = reusableBitmap  // reuse existing allocation
}
val newBitmap = BitmapFactory.decodeFile(filePath, options)
// newBitmap === reusableBitmap — same memory, new content
```

---

## Compress Before Upload

```kotlin
// ✅ Compress bitmap before sending to server
fun Bitmap.toCompressedByteArray(
    format: Bitmap.CompressFormat = Bitmap.CompressFormat.JPEG,
    quality: Int = 85
): ByteArray {
    return ByteArrayOutputStream().use { out ->
        compress(format, quality, out)
        out.toByteArray()
    }
}

// ✅ Scale down before compressing
fun Uri.toScaledBitmap(context: Context, maxDimension: Int = 1024): Bitmap {
    val original = BitmapFactory.decodeStream(context.contentResolver.openInputStream(this))
    val scale = maxDimension.toFloat() / maxOf(original.width, original.height)
    return if (scale < 1f) {
        Bitmap.createScaledBitmap(
            original,
            (original.width * scale).toInt(),
            (original.height * scale).toInt(),
            true
        ).also { original.recycle() }
    } else {
        original
    }
}
```

---

## Anti-Patterns

- Loading full-resolution images for small thumbnails — massive memory waste
- Decoding bitmaps on the main thread — ANR risk
- Not recycling bitmaps when done (if managing manually) — native heap leak
- Using `ARGB_8888` for all images including opaque backgrounds — unnecessary memory
- Storing decoded bitmaps in static fields — never GC'd

---

## Related Skills
- `heap-management` — overall heap and memory pressure management
- `memory-leak-prevention` — preventing bitmap reference leaks
- `loading-strategy` — showing placeholders while images load
- `allocation-optimization` — reducing GC pressure from bitmap operations
