---
name: filesystem
description: >
  Android filesystem — internal storage, external storage, cache, and file sharing.
  Load this skill when reading or writing files, sharing files between apps,
  choosing the right storage location, or handling storage permissions.
---

# Filesystem

## Overview

Android provides several storage locations with different visibility, persistence, and permission requirements. Choosing the right location depends on whether files are private to the app, shareable with other apps, or user-visible. Scoped Storage (API 29+) significantly restricts direct filesystem access.

---

## Core Principles

- Default to **internal storage** — private, no permissions needed
- Use **external storage** only for files the user should access via file manager
- Never use `Environment.getExternalStorageDirectory()` — deprecated, requires dangerous permission
- Use `FileProvider` for sharing files with other apps — never expose raw file paths
- Cache files can be deleted by the system at any time — don't rely on them for critical data

---

## Storage Locations

| Location           | API                             | Permission               | Cleared on Uninstall | Visible to User |
| ------------------ | ------------------------------- | ------------------------ | -------------------- | --------------- |
| `filesDir`         | `context.filesDir`              | None                     | ✅ Yes                | ❌ No            |
| `cacheDir`         | `context.cacheDir`              | None                     | ✅ Yes                | ❌ No            |
| `externalFilesDir` | `context.getExternalFilesDir()` | None (API 19+)           | ✅ Yes                | ✅ Yes           |
| `externalCacheDir` | `context.externalCacheDir`      | None                     | ✅ Yes                | ✅ Yes           |
| MediaStore         | Via ContentResolver             | `READ_MEDIA_*` (API 33+) | ❌ No                 | ✅ Yes           |

---

## Internal Storage

```kotlin
// ✅ Write to internal storage
fun writeFile(context: Context, fileName: String, content: String) {
    val file = File(context.filesDir, fileName)
    file.writeText(content)
}

// ✅ Read from internal storage
fun readFile(context: Context, fileName: String): String? {
    val file = File(context.filesDir, fileName)
    return if (file.exists()) file.readText() else null
}

// ✅ Subdirectory in internal storage
fun getAppDirectory(context: Context, dirName: String): File {
    return File(context.filesDir, dirName).also { it.mkdirs() }
}

// ✅ Write binary data
fun writeBinaryFile(context: Context, fileName: String, bytes: ByteArray) {
    File(context.filesDir, fileName).writeBytes(bytes)
}
```

---

## Cache Storage

```kotlin
// ✅ Use cache for temporary/reproducible data
fun cacheImage(context: Context, name: String, bitmap: Bitmap) {
    val file = File(context.cacheDir, name)
    file.outputStream().use { out ->
        bitmap.compress(Bitmap.CompressFormat.JPEG, 90, out)
    }
}

// ✅ Check cache before downloading
fun getCachedFile(context: Context, name: String): File? {
    val file = File(context.cacheDir, name)
    return if (file.exists()) file else null
}

// ✅ Limit cache size manually if needed
fun trimCache(context: Context, maxSizeBytes: Long) {
    val cacheDir = context.cacheDir
    val files = cacheDir.listFiles()?.sortedBy { it.lastModified() } ?: return
    var totalSize = files.sumOf { it.length() }
    for (file in files) {
        if (totalSize <= maxSizeBytes) break
        totalSize -= file.length()
        file.delete()
    }
}
```

---

## External App-Specific Storage

```kotlin
// ✅ Write to app-specific external directory (no permission needed)
fun writeToExternalFiles(context: Context, fileName: String, content: String) {
    val dir = context.getExternalFilesDir(Environment.DIRECTORY_DOCUMENTS)
        ?: return // external storage may not be available
    File(dir, fileName).writeText(content)
}

// ✅ Check external storage availability
fun isExternalStorageWritable(): Boolean =
    Environment.getExternalStorageState() == Environment.MEDIA_MOUNTED
```

---

## File Sharing with FileProvider

```xml
<!-- AndroidManifest.xml -->
<provider
    android:name="androidx.core.content.FileProvider"
    android:authorities="${applicationId}.fileprovider"
    android:exported="false"
    android:grantUriPermissions="true">
    <meta-data
        android:name="android.support.FILE_PROVIDER_PATHS"
        android:resource="@xml/file_paths" />
</provider>
```

```xml
<!-- res/xml/file_paths.xml -->
<paths>
    <files-path name="internal_files" path="." />
    <cache-path name="cache" path="." />
    <external-files-path name="external_files" path="." />
</paths>
```

```kotlin
// ✅ Share a file via FileProvider
fun shareFile(context: Context, file: File) {
    val uri = FileProvider.getUriForFile(
        context,
        "${context.packageName}.fileprovider",
        file
    )

    val intent = Intent(Intent.ACTION_SEND).apply {
        type = "application/pdf"
        putExtra(Intent.EXTRA_STREAM, uri)
        addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)
    }

    context.startActivity(Intent.createChooser(intent, "Share File"))
}
```

---

## Reading Files from Assets

```kotlin
// ✅ Read from assets/
fun readAsset(context: Context, fileName: String): String {
    return context.assets.open(fileName).bufferedReader().use { it.readText() }
}

// ✅ Read JSON config from assets
fun readJsonAsset(context: Context, fileName: String): JsonObject {
    val content = readAsset(context, fileName)
    return Json.parseToJsonElement(content).jsonObject
}
```

---

## Coroutine-Safe File I/O

```kotlin
// ✅ Always perform file I/O on Dispatchers.IO
suspend fun writeFileSafely(context: Context, fileName: String, content: String) {
    withContext(Dispatchers.IO) {
        File(context.filesDir, fileName).writeText(content)
    }
}

suspend fun readFileSafely(context: Context, fileName: String): String? {
    return withContext(Dispatchers.IO) {
        val file = File(context.filesDir, fileName)
        if (file.exists()) file.readText() else null
    }
}
```

---

## Anti-Patterns

- Using `Environment.getExternalStorageDirectory()` — deprecated since API 29
- File I/O on the main thread — causes ANR
- Exposing raw `file://` URIs to other apps — crashes on API 24+, use FileProvider
- Storing sensitive data in external storage — accessible by other apps
- Relying on cache files for critical data — system can delete them anytime
- Not checking external storage availability before writing
- Hardcoding absolute paths — they differ across devices and API levels

---

## Related Skills

- `file-storage` — higher-level file storage patterns
- `binary-storage` — storing binary/media files
- `encrypted-database` — encrypting sensitive stored data
- `workmanager` — background file operations
- `datastore` — structured key-value persistence
