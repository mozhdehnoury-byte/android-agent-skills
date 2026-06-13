---
name: multipart-upload
description: >
  Multipart file upload in Android using Retrofit or Ktor.
  Load this skill when uploading images, videos, or files to a server,
  tracking upload progress, handling large file uploads, or sending
  mixed form data with files.
---

# Multipart Upload

## Overview

Multipart upload sends files and form data in a single HTTP request using `multipart/form-data` encoding. On Android, files are typically picked from the gallery or camera, converted to a `MultipartBody.Part`, and sent via Retrofit or Ktor. Progress tracking requires a custom `RequestBody` wrapper.

---

## Core Principles

- Never read the full file into memory — stream from `Uri` using `ContentResolver`
- Track upload progress via a custom `RequestBody` wrapper — not polling
- Run uploads in a `CoroutineWorker` for long-running or background uploads
- Compress images before upload when size matters — respect server limits
- Show progress in the UI via `StateFlow` — cancel is a first-class operation

---

## Retrofit Multipart Upload

```kotlin
// ✅ API interface
interface MediaApi {
    @Multipart
    @POST("users/{id}/avatar")
    suspend fun uploadAvatar(
        @Path("id") userId: String,
        @Part avatar: MultipartBody.Part
    ): AvatarDto

    @Multipart
    @POST("posts")
    suspend fun createPost(
        @Part("title") title: RequestBody,
        @Part("description") description: RequestBody,
        @Part image: MultipartBody.Part
    ): PostDto
}

// ✅ Build MultipartBody.Part from Uri
fun Context.uriToMultipartPart(uri: Uri, partName: String): MultipartBody.Part {
    val mimeType = contentResolver.getType(uri) ?: "application/octet-stream"
    val fileName = uri.getFileName(contentResolver) ?: "upload"

    val requestBody = object : RequestBody() {
        override fun contentType() = mimeType.toMediaType()

        override fun contentLength(): Long {
            return contentResolver.openFileDescriptor(uri, "r")?.statSize ?: -1
        }

        override fun writeTo(sink: BufferedSink) {
            contentResolver.openInputStream(uri)?.use { input ->
                sink.writeAll(input.source())
            }
        }
    }

    return MultipartBody.Part.createFormData(partName, fileName, requestBody)
}

fun Uri.getFileName(contentResolver: ContentResolver): String? {
    return contentResolver.query(this, null, null, null, null)?.use { cursor ->
        val index = cursor.getColumnIndex(OpenableColumns.DISPLAY_NAME)
        cursor.moveToFirst()
        cursor.getString(index)
    }
}
```

---

## Progress Tracking

```kotlin
// ✅ RequestBody wrapper with progress callback
class ProgressRequestBody(
    private val delegate: RequestBody,
    private val onProgress: (Int) -> Unit
) : RequestBody() {

    override fun contentType() = delegate.contentType()
    override fun contentLength() = delegate.contentLength()

    override fun writeTo(sink: BufferedSink) {
        val total = contentLength()
        var uploaded = 0L

        val progressSink = object : ForwardingSink(sink) {
            override fun write(source: Buffer, byteCount: Long) {
                super.write(source, byteCount)
                uploaded += byteCount
                val progress = if (total > 0) ((uploaded * 100) / total).toInt() else 0
                onProgress(progress)
            }
        }

        delegate.writeTo(progressSink.buffer())
    }
}

// ✅ Use in ViewModel
class UploadViewModel @Inject constructor(
    private val mediaRepository: MediaRepository
) : ViewModel() {

    private val _progress = MutableStateFlow(0)
    val progress: StateFlow<Int> = _progress.asStateFlow()

    private val _state = MutableStateFlow<UploadState>(UploadState.Idle)
    val state: StateFlow<UploadState> = _state.asStateFlow()

    private var uploadJob: Job? = null

    fun upload(uri: Uri) {
        uploadJob = viewModelScope.launch {
            _state.value = UploadState.Uploading
            mediaRepository.uploadFile(uri) { progress ->
                _progress.value = progress
            }.fold(
                onSuccess = { _state.value = UploadState.Success(it) },
                onFailure = { _state.value = UploadState.Error(it.message ?: "Upload failed") }
            )
        }
    }

    fun cancel() {
        uploadJob?.cancel()
        _state.value = UploadState.Idle
        _progress.value = 0
    }
}

sealed interface UploadState {
    data object Idle : UploadState
    data object Uploading : UploadState
    data class Success(val url: String) : UploadState
    data class Error(val message: String) : UploadState
}
```

---

## Image Compression Before Upload

```kotlin
// ✅ Compress bitmap before upload
fun Uri.compressToByteArray(
    context: Context,
    maxWidth: Int = 1024,
    maxHeight: Int = 1024,
    quality: Int = 85
): ByteArray {
    val bitmap = BitmapFactory.decodeStream(context.contentResolver.openInputStream(this))
    val scaled = Bitmap.createScaledBitmap(
        bitmap,
        maxWidth.coerceAtMost(bitmap.width),
        maxHeight.coerceAtMost(bitmap.height),
        true
    )
    return ByteArrayOutputStream().also { out ->
        scaled.compress(Bitmap.CompressFormat.JPEG, quality, out)
    }.toByteArray()
}
```

---

## Ktor Multipart Upload

```kotlin
// ✅ Ktor multipart upload
suspend fun uploadAvatar(userId: String, fileBytes: ByteArray, fileName: String): AvatarDto {
    return client.submitFormWithBinaryData(
        url = "users/$userId/avatar",
        formData = formData {
            append("avatar", fileBytes, Headers.build {
                append(HttpHeaders.ContentType, "image/jpeg")
                append(HttpHeaders.ContentDisposition, "filename=$fileName")
            })
        }
    ).body()
}
```

---

## Anti-Patterns

- Loading the entire file into a `ByteArray` before upload — causes OOM for large files
- Not showing progress for uploads over 1MB — user has no feedback
- No cancel mechanism — user is stuck waiting
- Not compressing images — wastes bandwidth and hits server size limits
- Blocking the main thread during file reading — use coroutines with IO dispatcher

---

## Related Skills

- `retrofit` — Retrofit client setup
- `ktor` — Ktor client for KMP
- `workmanager` — background upload for large or resumable uploads
- `camera` — capturing images to upload
- `filesystem` — reading files from device storage
