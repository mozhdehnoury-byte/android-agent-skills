---
name: firebase-messaging
description: >
  Firebase Cloud Messaging (FCM) setup and handling for Android.
  Load this skill when implementing push notifications, handling FCM tokens,
  processing foreground/background messages, or sending data payloads.
---

# Firebase Messaging (FCM)

## Overview
Firebase Cloud Messaging (FCM) is Google's cross-platform messaging solution for push notifications. It delivers notification messages (shown by system) and data messages (handled by app code) to Android devices. FCM tokens identify each device/app installation.

---

## Core Principles

- **Notification messages** — shown automatically by system when app is in background
- **Data messages** — always delivered to `onMessageReceived()` for app to handle
- FCM token can change — always update the server when token refreshes
- Notification permission is required on **API 33+** — `POST_NOTIFICATIONS`
- Handle messages in a **short time** — `onMessageReceived()` has ~10 second limit

---

## Setup

```kotlin
// build.gradle.kts
dependencies {
    implementation(platform(libs.firebase.bom))
    implementation(libs.firebase.messaging)
}
```

```xml
<!-- AndroidManifest.xml -->
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />

<service
    android:name=".messaging.MyFirebaseMessagingService"
    android:exported="false">
    <intent-filter>
        <action android:name="com.google.firebase.MESSAGING_EVENT" />
    </intent-filter>
</service>
```

---

## MessagingService Implementation

```kotlin
// ✅ Handle incoming messages and token refresh
class MyFirebaseMessagingService : FirebaseMessagingService() {

    @Inject
    lateinit var notificationManager: AppNotificationManager

    @Inject
    lateinit var tokenRepository: FcmTokenRepository

    // ✅ Called when a new FCM token is generated
    override fun onNewToken(token: String) {
        super.onNewToken(token)
        // Upload new token to your server
        CoroutineScope(SupervisorJob() + Dispatchers.IO).launch {
            tokenRepository.updateToken(token)
        }
    }

    // ✅ Called for all messages when app is in foreground
    // Called for data-only messages when app is in background
    override fun onMessageReceived(message: RemoteMessage) {
        super.onMessageReceived(message)

        val title = message.notification?.title ?: message.data["title"] ?: return
        val body  = message.notification?.body  ?: message.data["body"]  ?: return
        val type  = message.data["type"]
        val id    = message.data["id"]

        notificationManager.showNotification(
            title = title,
            body = body,
            data = message.data
        )
    }
}
```

---

## FCM Token Management

```kotlin
// ✅ Get current token
class FcmTokenRepository @Inject constructor(
    private val api: UserApi,
    private val prefs: DataStore<Preferences>
) {
    suspend fun getToken(): String? = suspendCoroutine { cont ->
        FirebaseMessaging.getInstance().token
            .addOnSuccessListener { cont.resume(it) }
            .addOnFailureListener { cont.resume(null) }
    }

    suspend fun updateToken(token: String) {
        val saved = prefs.data.first()[FCM_TOKEN_KEY]
        if (saved != token) {
            api.updateFcmToken(token)
            prefs.edit { it[FCM_TOKEN_KEY] = token }
        }
    }

    suspend fun deleteToken() {
        FirebaseMessaging.getInstance().deleteToken().await()
        api.deleteFcmToken()
        prefs.edit { it.remove(FCM_TOKEN_KEY) }
    }

    companion object {
        val FCM_TOKEN_KEY = stringPreferencesKey("fcm_token")
    }
}
```

---

## Notification Permission (API 33+)

```kotlin
// ✅ Request POST_NOTIFICATIONS permission on Android 13+
@Composable
fun NotificationPermissionHandler(onGranted: () -> Unit) {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
        val permissionState = rememberPermissionState(
            Manifest.permission.POST_NOTIFICATIONS
        )
        LaunchedEffect(Unit) {
            if (!permissionState.status.isGranted) {
                permissionState.launchPermissionRequest()
            } else {
                onGranted()
            }
        }
    } else {
        LaunchedEffect(Unit) { onGranted() }
    }
}
```

---

## Showing Notifications

```kotlin
// ✅ NotificationManager wrapper
class AppNotificationManager @Inject constructor(
    private val context: Context
) {
    private val notificationManager =
        context.getSystemService(NotificationManager::class.java)

    init {
        createChannels()
    }

    private fun createChannels() {
        val channel = NotificationChannel(
            CHANNEL_ID_GENERAL,
            "General",
            NotificationManager.IMPORTANCE_DEFAULT
        ).apply {
            description = "General app notifications"
        }
        notificationManager.createNotificationChannel(channel)
    }

    fun showNotification(title: String, body: String, data: Map<String, String>) {
        val intent = Intent(context, MainActivity::class.java).apply {
            flags = Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_CLEAR_TASK
            data.forEach { (key, value) -> putExtra(key, value) }
        }

        val pendingIntent = PendingIntent.getActivity(
            context, 0, intent,
            PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
        )

        val notification = NotificationCompat.Builder(context, CHANNEL_ID_GENERAL)
            .setSmallIcon(R.drawable.ic_notification)
            .setContentTitle(title)
            .setContentText(body)
            .setAutoCancel(true)
            .setContentIntent(pendingIntent)
            .setPriority(NotificationCompat.PRIORITY_DEFAULT)
            .build()

        notificationManager.notify(System.currentTimeMillis().toInt(), notification)
    }

    companion object {
        const val CHANNEL_ID_GENERAL = "general"
    }
}
```

---

## Message Types

```kotlin
// Notification message (server-sent):
// {
//   "notification": { "title": "Hello", "body": "World" },
//   "to": "<fcm_token>"
// }
// → System shows notification automatically when app in background
// → onMessageReceived() called when app in foreground

// Data message (server-sent):
// {
//   "data": { "title": "Hello", "body": "World", "type": "order_update" },
//   "to": "<fcm_token>"
// }
// → onMessageReceived() always called regardless of app state

// ✅ Prefer data messages for full control over display
```

---

## Topic Subscription

```kotlin
// ✅ Subscribe to topics for broadcast messages
suspend fun subscribeToTopic(topic: String) {
    FirebaseMessaging.getInstance().subscribeToTopic(topic).await()
}

suspend fun unsubscribeFromTopic(topic: String) {
    FirebaseMessaging.getInstance().unsubscribeFromTopic(topic).await()
}

// Usage
subscribeToTopic("promotions")
subscribeToTopic("user_${userId}")
```

---

## Anti-Patterns

- Not updating token on `onNewToken()` — server sends to stale token, messages lost
- Long-running work in `onMessageReceived()` — only 10 seconds available; use WorkManager
- Notification messages for data that needs app processing — use data messages
- Not creating notification channels — notifications silently dropped on API 26+
- Not handling `POST_NOTIFICATIONS` permission on API 33+ — no notifications shown

---

## Related Skills
- `firebase` — Firebase core setup
- `notification` — notification channels and display
- `workmanager` — background work triggered by FCM
- `deep-link` — navigating to specific screen from notification tap
