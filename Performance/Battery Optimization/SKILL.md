---
name: battery-optimization
description: >
  Battery usage optimization in Android apps.
  Load this skill when reducing background battery drain, handling Doze
  mode, batching network requests, optimizing location usage,
  or profiling energy usage with Android Studio Energy Profiler.
---

# Battery Optimization

## Overview
Battery drain is one of the top reasons users uninstall apps. The main causes are: excessive wake locks, frequent network requests, GPS usage, and background processing that ignores Doze/App Standby. The goal is to do necessary work efficiently and batch or defer work that doesn't need to happen immediately.

---

## Core Principles

- **Batch** network requests — fewer wakeups is better than many small ones
- **Defer** non-urgent work to when the device is charging or on Wi-Fi (WorkManager constraints)
- Release **WakeLocks** as quickly as possible — always in a `finally` block
- Use **passive location** or lower accuracy when precise GPS is not needed
- Never poll — use push (FCM) or `ContentObserver` instead

---

## WorkManager Constraints

```kotlin
// ✅ Defer heavy sync to favorable conditions
val syncRequest = PeriodicWorkRequestBuilder<SyncWorker>(1, TimeUnit.HOURS)
    .setConstraints(
        Constraints.Builder()
            .setRequiredNetworkType(NetworkType.UNMETERED)  // Wi-Fi only
            .setRequiresCharging(true)                       // only when charging
            .setRequiresBatteryNotLow(true)                  // not low battery
            .build()
    )
    .build()
```

---

## Batching Network Requests

```kotlin
// ❌ Individual requests — each wakes radio
suspend fun syncAll() {
    syncUsers()     // wakes radio
    delay(100)
    syncOrders()    // wakes radio again
    delay(100)
    syncProducts()  // wakes radio again
}

// ✅ Single batched request — one radio wakeup
suspend fun syncAll() {
    val batch = syncRepository.syncAll()  // one API call returning all data
    processBatchResult(batch)
}
```

---

## Doze Mode Awareness

```kotlin
// Doze mode restricts:
// - Network access
// - Wake locks
// - AlarmManager (except setAndAllowWhileIdle)
// - JobScheduler / WorkManager (deferred)
//
// Exempt from Doze:
// - FCM high-priority messages
// - ForegroundService
// - WorkManager with setExpedited

// ✅ Expedited work for urgent tasks during Doze
val urgentRequest = OneTimeWorkRequestBuilder<UrgentSyncWorker>()
    .setExpedited(OutOfQuotaPolicy.RUN_AS_NON_EXPEDITED_WORK_REQUEST)
    .build()
```

---

## WakeLock — Use Sparingly

```kotlin
// ✅ Acquire and release WakeLock safely
class WakeLockManager @Inject constructor(
    @ApplicationContext context: Context
) {
    private val powerManager = context.getSystemService(PowerManager::class.java)
    private val wakeLock = powerManager.newWakeLock(
        PowerManager.PARTIAL_WAKE_LOCK,
        "MyApp::SyncWakeLock"
    )

    fun withWakeLock(timeoutMs: Long = 10_000, block: () -> Unit) {
        wakeLock.acquire(timeoutMs)  // always set timeout
        try {
            block()
        } finally {
            if (wakeLock.isHeld) wakeLock.release()  // ✅ always release
        }
    }
}
```

---

## Location Battery Optimization

```kotlin
// ✅ Use lowest accuracy tier that meets the need
val locationRequest = LocationRequest.Builder(
    Priority.PRIORITY_BALANCED_POWER_ACCURACY,  // cell tower + Wi-Fi — less battery than GPS
    intervalMillis = 30_000                      // 30 second interval
)
    .setMinUpdateDistanceMeters(50f)             // only update if moved 50m
    .build()

// ✅ Use PRIORITY_PASSIVE for background — piggyback on other apps' location fixes
val passiveRequest = LocationRequest.Builder(
    Priority.PRIORITY_PASSIVE,
    intervalMillis = 60_000
).build()

// ✅ Stop updates when not needed
override fun onStop() {
    locationClient.removeLocationUpdates(locationCallback)
    super.onStop()
}
```

---

## Polling vs Push

```kotlin
// ❌ Polling — wakes device repeatedly
fun startPolling() {
    viewModelScope.launch {
        while (true) {
            fetchNewMessages()    // unnecessary if no new messages
            delay(30_000)
        }
    }
}

// ✅ FCM push — device wakes only when there's actual data
// Server sends FCM notification → app wakes → fetches only when needed
class MessagingService : FirebaseMessagingService() {
    override fun onMessageReceived(message: RemoteMessage) {
        // Triggered only when server sends a message
        syncMessages()
    }
}
```

---

## Battery Profiling

```
Android Studio → Profiler → Energy:
  - Shows CPU, network, and location wakeups over time
  - Identify: which operations cause the most wakeups
  - Compare before/after optimization

adb shell dumpsys batterystats --reset  // reset stats
adb shell dumpsys batterystats          // dump current stats
```

---

## Anti-Patterns

- Holding WakeLock without a timeout — device never sleeps
- Polling for updates instead of using FCM or WebSocket
- Using `PRIORITY_HIGH_ACCURACY` location for all use cases — drains GPS
- Running WorkManager without constraints — executes even on low battery/metered network
- Ignoring Doze mode — work silently skipped without the app knowing

---

## Related Skills
- `workmanager` — scheduling work with battery-aware constraints
- `background-processing` — choosing the right background mechanism
- `firebase-messaging` — push notifications to replace polling
- `foreground-service` — when continuous background work is truly needed
