---
name: file-storage
description: >
  File storage patterns for Android — saving, reading, and managing files
  across internal storage, external storage, and cache.
  Load this skill when persisting files (images, PDFs, exports, downloads),
  choosing storage location, or managing file lifecycle.
---

# File Storage

## Overview

File storage is used when data doesn't fit neatly into a database or key-value store — images, documents, exports, downloaded content, and binary files. Android provides several storage locations with different visibility, permission, and persistence characteristics.

---

## Core Principles

- Choose the **right storage location** for the file's purpose and visibility
- Always perform file I/O on **Dispatchers.IO** — never main thread
- Use **FileProvider** to share files with other apps — never expose raw paths
- Cache files are **not guaranteed** — don't use for critical data
- Clean up files when they're no longer needed — storage is finite

---

## Storage Location Decision

```
Is the file private to the app?
    YES → Does the system need to delete it automatically?
        YES → cacheDir (temporary)
        NO  → filesDir (permanent internal)
    NO  → Is it app-specific but user-visible?
        YES → getExternalFilesDir() (no permission needed)
        NO  → MediaStore (shared media) or Downloads
```

---

## Writing Files

```kotlin
// ✅ Internal storage — private, permanent
suspend fun saveReport(context: Context, fileName: String, content: ByteArray) {
    withContext(Dispatchers.IO) {
        val file = File(context.filesDir, fileName)
        file.writeBytes(content)
    }
}

// ✅ Subdirectory in internal storage
suspend fun saveInSubdir(context: Context, subdir: String, fileName: String, content: String) {
    withContext(Dispatchers.IO) {
        val dir = File(context.filesDir, subdir).also { it.mkdirs() }
        File(dir, fileName).writeText(content)
    }
}

// ✅ Cache — temporary, system may delete
suspend fun cacheDownload(context: Context, fileName: String, content: ByteArray) {
    withContext(Dispatchers.IO) {
        File(context.cacheDir, fileName).writeBytes(content)
    }
}

// ✅ External app-specific — user-visible, no permission needed
suspend fun saveToExternalDocs(context: Context, fileName: String, content: ByteArray) {
    withContext(Dispatchers.IO) {
        val dir = context.getExternalFilesDir(Environment.DIRECTORY_DOCUMENTS)
            ?: return@withContext
        File(dir, fileName).writeBytes(content)
    }
}
```

---

## Reading Files

```kotlin
// ✅ Read with existence check
suspend fun readFile(context: Context, fileName: String): ByteArray? {
    return withContext(Dispatchers.IO) {
        val file = File(context.filesDir, fileName)
        if (file.exists()) file.readBytes() else null
    }
}

// ✅ Read as stream (large files)
suspend fun readLargeFile(context: Context, fileName: String): Flow<ByteArray> = flow {
    withContext(Dispatchers.IO) {
        val file = File(context.filesDir, fileName)
        file.inputStream().buffered().use { stream ->
            val buffer = ByteArray(8192)
            var bytesRead: Int
            while (stream.read(buffer).also { bytesRead = it } != -1) {
                emit(buffer.copyOf(bytesRead))
            }
        }
    }
}
```

---

## File Management

```kotlin
// ✅ Delete file
suspend fun deleteFile(context: Context, fileName: String): Boolean {
    return withContext(Dispatchers.IO) {
        File(context.filesDir, fileName).delete()
    }
}

// ✅ List files in directory
suspend fun listFiles(context: Context, subdir: String): List<String> {
    return withContext(Dispatchers.IO) {
        File(context.filesDir, subdir)
            .listFiles()
            ?.map { it.name }
            ?: emptyList()
    }
}

// ✅ Get file size
fun getFileSize(context: Context, fileName: String): Long {
    return File(context.filesDir, fileName).length()
}

// ✅ Check if file exists
fun fileExists(context: Context, fileName: String): Boolean {
    return File(context.filesDir, fileName).exists()
}
```

---

## Saving Bitmaps

```kotlin
// ✅ Save bitmap to internal storage
suspend fun saveBitmap(context: Context, bitmap: Bitmap, fileName: String) {
    withContext(Dispatchers.IO) {
        val file = File(context.filesDir, fileName)
        file.outputStream().use { out ->
            bitmap.compress(Bitmap.CompressFormat.JPEG, 90, out)
        }
    }
}

// ✅ Load bitmap
suspend fun loadBitmap(context: Context, fileName: String): Bitmap? {
    return withContext(Dispatchers.IO) {
        val file = File(context.filesDir, fileName)
        if (file.exists()) BitmapFactory.decodeFile(file.absolutePath) else null
    }
}
```

---

## Sharing Files via FileProvider

```kotlin
// ✅ Get shareable URI — never expose raw file:// URI
fun getShareableUri(context: Context, file: File): Uri {
    return FileProvider.getUriForFile(
        context,
        "${context.packageName}.fileprovider",
        file
    )
}

// ✅ Share file
fun shareFile(context: Context, file: File, mimeType: String) {
    val uri = getShareableUri(context, file)
    val intent = Intent(Intent.ACTION_SEND).apply {
        type = mimeType
        putExtra(Intent.EXTRA_STREAM, uri)
        addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)
    }
    context.startActivity(Intent.createChooser(intent, null))
}
```

---

## Cache Management

```kotlin
// ✅ Calculate cache size
suspend fun getCacheSizeBytes(context: Context): Long {
    return withContext(Dispatchers.IO) {
        context.cacheDir.walkTopDown().sumOf { it.length() }
    }
}

// ✅ Clear cache
suspend fun clearCache(context: Context) {
    withContext(Dispatchers.IO) {
        context.cacheDir.deleteRecursively()
    }
}

// ✅ Trim cache to max size (LRU — delete oldest first)
suspend fun trimCache(context: Context, maxBytes: Long) {
    withContext(Dispatchers.IO) {
        val files = context.cacheDir
            .walkTopDown()
            .filter { it.isFile }
            .sortedBy { it.lastModified() }
            .toMutableList()

        var totalSize = files.sumOf { it.length() }
        for (file in files) {
            if (totalSize <= maxBytes) break
            totalSize -= file.length()
            file.delete()
        }
    }
}
```

---

## Anti-Patterns

- File I/O on the main thread — causes ANR
- Storing files in external storage without checking availability
- Using `file://` URI directly with other apps — crashes on API 24+
- Not cleaning up temp files — accumulates storage over time
- Storing sensitive data in external storage — accessible to other apps
- Not checking `file.exists()` before reading — `FileNotFoundException`
- Recursive directory listing on large directories on main thread

---

## Related Skills

- `filesystem` — Android filesystem fundamentals
- `binary-storage` — storing binary/media data
- `encrypted-database` — encrypting sensitive files
- `cache-strategy` — cache invalidation and management
- `workmanager` — background file operations
