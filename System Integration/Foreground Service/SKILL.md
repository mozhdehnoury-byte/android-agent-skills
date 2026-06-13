---
name: foreground-service
description: >
  Foreground Service implementation in Android.
  Load this skill when running long-running user-visible operations like
  media playback, file upload/download, location tracking, or any task
  that must run while the app is in background with a visible notification.
---

# Foreground Service

## Overview
A Foreground Service runs in the background while showing a persistent notification. It survives app backgrounding and is used for operations the user is actively aware of — music playback, navigation, ongoing uploads. Since Android 14, foreground service types must be declared explicitly.

---

## Core Principles

- Always show a **meaningful notification** — it's required and user-facing
- Declare the correct **foreground service type** in the manifest
- Stop the service when the work is done — `stopSelf()` or `stopForeground()`
- Use `CoroutineScope` inside the service for async work
- Prefer WorkManager for deferrable work — ForegroundService is for user-visible ongoing tasks

---

## Manifest Declaration

```xml
<!-- AndroidManifest.xml -->
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_DATA_SYNC" />
<!-- Add the type matching your use case -->

<application>
    <service
        android:name=".service.UploadService"
        android:foregroundServiceType="dataSync"
        android:exported="false" />
</application>
```

---

## Service Types Reference

| Type | Use Case |
|------|----------|
| `dataSync` | Upload / download / sync |
| `mediaPlayback` | Music / podcast playback |
| `location` | Navigation, location tracking |
| `camera` | Video recording |
| `microphone` | Audio recording |
| `connectedDevice` | Bluetooth device communication |

---

## Basic Foreground Service

```kotlin
// ✅ Foreground service with coroutine scope
class UploadService : Service() {

    private val scope = CoroutineScope(SupervisorJob() + Dispatchers.IO)
    private var uploadJob: Job? = null

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        when (intent?.action) {
            ACTION_START -> {
                val fileUri = intent.getStringExtra(EXTRA_FILE_URI) ?: run {
                    stopSelf()
                    return START_NOT_STICKY
                }
                startForegroundWithNotification()
                startUpload(fileUri)
            }
            ACTION_CANCEL -> cancelUpload()
        }
        return START_NOT_STICKY
    }

    private fun startForegroundWithNotification() {
        val notification = buildNotification("Starting upload…")
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
            startForeground(NOTIFICATION_ID, notification, ServiceInfo.FOREGROUND_SERVICE_TYPE_DATA_SYNC)
        } else {
            startForeground(NOTIFICATION_ID, notification)
        }
    }

    private fun startUpload(fileUri: String) {
        uploadJob = scope.launch {
            try {
                uploadRepository.upload(Uri.parse(fileUri)) { progress ->
                    updateNotification("Uploading… $progress%")
                }
                updateNotification("Upload complete")
                delay(2_000)
            } catch (e: CancellationException) {
                updateNotification("Upload cancelled")
            } catch (e: Exception) {
                updateNotification("Upload failed: ${e.message}")
            } finally {
                stopForeground(STOP_FOREGROUND_REMOVE)
                stopSelf()
            }
        }
    }

    private fun cancelUpload() {
        uploadJob?.cancel()
    }

    override fun onDestroy() {
        scope.cancel()
        super.onDestroy()
    }

    override fun onBind(intent: Intent?): IBinder? = null

    private fun buildNotification(text: String): Notification {
        createNotificationChannel()
        return NotificationCompat.Builder(this, CHANNEL_ID)
            .setContentTitle("File Upload")
            .setContentText(text)
            .setSmallIcon(R.drawable.ic_upload)
            .setOngoing(true)
            .build()
    }

    private fun updateNotification(text: String) {
        val manager = getSystemService(NotificationManager::class.java)
        manager.notify(NOTIFICATION_ID, buildNotification(text))
    }

    private fun createNotificationChannel() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            val channel = NotificationChannel(
                CHANNEL_ID, "Upload", NotificationManager.IMPORTANCE_LOW
            )
            getSystemService(NotificationManager::class.java).createNotificationChannel(channel)
        }
    }

    companion object {
        const val ACTION_START = "action_start"
        const val ACTION_CANCEL = "action_cancel"
        const val EXTRA_FILE_URI = "file_uri"
        private const val NOTIFICATION_ID = 1001
        private const val CHANNEL_ID = "upload_channel"

        fun startIntent(context: Context, fileUri: Uri): Intent =
            Intent(context, UploadService::class.java).apply {
                action = ACTION_START
                putExtra(EXTRA_FILE_URI, fileUri.toString())
            }

        fun cancelIntent(context: Context): Intent =
            Intent(context, UploadService::class.java).apply {
                action = ACTION_CANCEL
            }
    }
}
```

---

## Starting the Service

```kotlin
// ✅ Start from ViewModel via event
class UploadViewModel : ViewModel() {
    private val _events = Channel<UploadEvent>(Channel.BUFFERED)
    val events = _events.receiveAsFlow()

    fun onUploadClick(fileUri: Uri) {
        viewModelScope.launch {
            _events.send(UploadEvent.StartService(fileUri))
        }
    }
}

// ✅ Handle in composable / activity
LaunchedEffect(Unit) {
    viewModel.events.collect { event ->
        when (event) {
            is UploadEvent.StartService -> {
                val intent = UploadService.startIntent(context, event.fileUri)
                context.startForegroundService(intent)
            }
        }
    }
}
```

---

## Anti-Patterns

- Starting a ForegroundService for short tasks — use coroutines or WorkManager
- Not calling `stopSelf()` when work completes — service stays alive indefinitely
- Missing `foregroundServiceType` in manifest on Android 10+ — SecurityException
- Doing heavy work on the main thread inside `onStartCommand` — use a coroutine scope
- Not cancelling the coroutine scope in `onDestroy` — coroutine leak

---

## Related Skills
- `workmanager` — deferrable background work without a notification requirement
- `notification` — building and updating notifications
- `background-processing` — choosing the right background mechanism
- `coroutine` — coroutine scope inside services
