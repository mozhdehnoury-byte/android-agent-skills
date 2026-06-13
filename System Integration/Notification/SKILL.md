---
name: notification
description: >
  Android notification system implementation.
  Load this skill when creating notification channels, building
  notifications, handling notification permissions, showing progress
  notifications, or adding actions and deep links to notifications.
---

# Notification

## Overview
Android notifications require a notification channel (API 26+), a POST_NOTIFICATIONS permission request (API 33+), and a properly built `Notification` object. Notifications can contain actions, deep link intents, progress indicators, and custom layouts.

---

## Core Principles

- Create notification channels **once** at app startup — not on every notification
- Request `POST_NOTIFICATIONS` permission on Android 13+ before showing notifications
- Use `NotificationCompat` — not the platform `Notification` directly
- Always provide a `contentIntent` — tapping the notification should open the relevant screen
- Group related notifications with a summary notification

---

## Permissions

```xml
<!-- AndroidManifest.xml -->
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
```

```kotlin
// ✅ Request permission on Android 13+
class MainActivity : ComponentActivity() {
    private val requestPermissionLauncher = registerForActivityResult(
        ActivityResultContracts.RequestPermission()
    ) { isGranted ->
        if (!isGranted) {
            // Show rationale or disable notification features
        }
    }

    fun requestNotificationPermission() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
            if (ContextCompat.checkSelfPermission(
                    this, Manifest.permission.POST_NOTIFICATIONS
                ) != PackageManager.PERMISSION_GRANTED
            ) {
                requestPermissionLauncher.launch(Manifest.permission.POST_NOTIFICATIONS)
            }
        }
    }
}
```

---

## Notification Channels

```kotlin
// ✅ Create channels once at app startup
class NotificationChannelSetup @Inject constructor(
    @ApplicationContext private val context: Context
) {
    fun createChannels() {
        if (Build.VERSION.SDK_INT < Build.VERSION_CODES.O) return

        val manager = context.getSystemService(NotificationManager::class.java)

        manager.createNotificationChannel(
            NotificationChannel(
                CHANNEL_MESSAGES,
                "Messages",
                NotificationManager.IMPORTANCE_HIGH
            ).apply {
                description = "New message notifications"
                enableVibration(true)
            }
        )

        manager.createNotificationChannel(
            NotificationChannel(
                CHANNEL_SYNC,
                "Sync",
                NotificationManager.IMPORTANCE_LOW
            ).apply {
                description = "Background sync status"
            }
        )
    }

    companion object {
        const val CHANNEL_MESSAGES = "channel_messages"
        const val CHANNEL_SYNC = "channel_sync"
    }
}
```

---

## Basic Notification

```kotlin
// ✅ Simple notification with deep link intent
fun showMessageNotification(context: Context, message: Message) {
    val deepLinkIntent = Intent(
        Intent.ACTION_VIEW,
        Uri.parse("https://example.com/messages/${message.id}")
    ).apply {
        flags = Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_CLEAR_TOP
    }

    val pendingIntent = TaskStackBuilder.create(context).run {
        addNextIntentWithParentStack(deepLinkIntent)
        getPendingIntent(
            message.id.hashCode(),
            PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
        )
    }

    val notification = NotificationCompat.Builder(context, CHANNEL_MESSAGES)
        .setSmallIcon(R.drawable.ic_message)
        .setContentTitle(message.senderName)
        .setContentText(message.preview)
        .setStyle(NotificationCompat.BigTextStyle().bigText(message.body))
        .setContentIntent(pendingIntent)
        .setAutoCancel(true)
        .setPriority(NotificationCompat.PRIORITY_HIGH)
        .build()

    NotificationManagerCompat.from(context)
        .notify(message.id.hashCode(), notification)
}
```

---

## Notification with Actions

```kotlin
// ✅ Action buttons on notification
val replyIntent = PendingIntent.getBroadcast(
    context,
    0,
    Intent(context, ReplyReceiver::class.java).putExtra("message_id", messageId),
    PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
)

val markReadIntent = PendingIntent.getBroadcast(
    context,
    1,
    Intent(context, MarkReadReceiver::class.java).putExtra("message_id", messageId),
    PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
)

NotificationCompat.Builder(context, CHANNEL_MESSAGES)
    .addAction(R.drawable.ic_reply, "Reply", replyIntent)
    .addAction(R.drawable.ic_check, "Mark Read", markReadIntent)
    .build()
```

---

## Progress Notification

```kotlin
// ✅ Update progress notification
fun showProgressNotification(context: Context, progress: Int, done: Boolean) {
    val notification = NotificationCompat.Builder(context, CHANNEL_SYNC)
        .setSmallIcon(R.drawable.ic_sync)
        .setContentTitle("Uploading file")
        .setContentText(if (done) "Upload complete" else "$progress%")
        .setProgress(100, progress, false)
        .setOngoing(!done)
        .setOnlyAlertOnce(true)   // ✅ don't re-alert on every update
        .build()

    NotificationManagerCompat.from(context).notify(NOTIFICATION_ID_UPLOAD, notification)

    if (done) {
        // Remove progress after delay
        Handler(Looper.getMainLooper()).postDelayed({
            NotificationManagerCompat.from(context).cancel(NOTIFICATION_ID_UPLOAD)
        }, 3_000)
    }
}
```

---

## Grouped Notifications

```kotlin
// ✅ Group multiple notifications with a summary
val GROUP_KEY = "com.example.MESSAGES"

// Individual notifications
messages.forEach { message ->
    val notification = NotificationCompat.Builder(context, CHANNEL_MESSAGES)
        .setSmallIcon(R.drawable.ic_message)
        .setContentTitle(message.sender)
        .setContentText(message.preview)
        .setGroup(GROUP_KEY)
        .build()
    manager.notify(message.id.hashCode(), notification)
}

// Summary notification
val summary = NotificationCompat.Builder(context, CHANNEL_MESSAGES)
    .setSmallIcon(R.drawable.ic_message)
    .setStyle(NotificationCompat.InboxStyle()
        .setSummaryText("${messages.size} new messages"))
    .setGroup(GROUP_KEY)
    .setGroupSummary(true)
    .build()
manager.notify(SUMMARY_ID, summary)
```

---

## Anti-Patterns

- Creating notification channels on every `notify()` call — create once at startup
- Not providing a `contentIntent` — tapping the notification does nothing
- Using `IMPORTANCE_HIGH` for background sync notifications — disrupts the user
- Not setting `FLAG_IMMUTABLE` on `PendingIntent` — crashes on Android 12+
- Showing notifications without requesting `POST_NOTIFICATIONS` on Android 13+

---

## Related Skills
- `deep-navigation` — building deep link intents for notifications
- `foreground-service` — persistent service notification
- `alarmmanager` — triggering notifications at a specific time
- `firebase-messaging` — push notifications via FCM
