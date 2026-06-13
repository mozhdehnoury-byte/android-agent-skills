---
name: anr-prevention
description: >
  ANR (Application Not Responding) prevention in Android.
  Load this skill when investigating ANR reports, identifying blocking
  main thread operations, using StrictMode to detect violations,
  or ensuring the main thread stays responsive under load.
---

# ANR Prevention

## Overview
An ANR occurs when the main thread is blocked for more than 5 seconds (input) or 10 seconds (broadcast). Android shows a dialog asking the user to wait or close the app. ANRs are caused by I/O, network, database queries, or heavy computation running on the main thread.

---

## Core Principles

- **Nothing blocking** on the main thread — no I/O, no network, no heavy computation
- Use `Dispatchers.IO` for all I/O operations — database, file, network
- Use `Dispatchers.Default` for CPU-heavy work — parsing, sorting, crypto
- Keep `BroadcastReceiver.onReceive()` under 10 seconds — delegate to WorkManager if longer
- Use StrictMode in development to catch violations before they reach users

---

## ANR Triggers

```
Input dispatch timeout:  5 seconds — user touched screen, app didn't respond
Broadcast timeout:      10 seconds (foreground) / 60 seconds (background)
Service timeout:        20 seconds (foreground) / 200 seconds (background)
ContentProvider timeout: 10 seconds

Common causes:
  - SharedPreferences.edit().commit() on main thread (use apply())
  - Room query on main thread (Room throws by default — don't disable this)
  - Bitmap decoding on main thread
  - Heavy JSON parsing on main thread
  - Synchronized lock contention on main thread
  - Binder calls to slow system services on main thread
```

---

## StrictMode — Catch Violations in Development

```kotlin
// ✅ Enable StrictMode in debug builds
class App : Application() {
    override fun onCreate() {
        super.onCreate()

        if (BuildConfig.DEBUG) {
            StrictMode.setThreadPolicy(
                StrictMode.ThreadPolicy.Builder()
                    .detectDiskReads()
                    .detectDiskWrites()
                    .detectNetwork()
                    .detectCustomSlowCalls()
                    .penaltyLog()             // log to Logcat
                    .penaltyFlashScreen()     // visual flash in debug
                    .build()
            )

            StrictMode.setVmPolicy(
                StrictMode.VmPolicy.Builder()
                    .detectLeakedSqlLiteObjects()
                    .detectLeakedClosableObjects()
                    .detectActivityLeaks()
                    .penaltyLog()
                    .build()
            )
        }
    }
}
```

---

## Moving Work off Main Thread

```kotlin
// ❌ Blocking main thread
class UserRepository {
    fun getUser(id: String): User {
        return database.userDao().getUser(id)  // blocks main thread
    }
}

// ✅ Suspend function on IO dispatcher
class UserRepository {
    suspend fun getUser(id: String): User = withContext(Dispatchers.IO) {
        database.userDao().getUser(id)
    }
}

// ❌ SharedPreferences commit on main thread
prefs.edit().putString("key", "value").commit()  // synchronous write

// ✅ apply() — asynchronous write
prefs.edit().putString("key", "value").apply()

// ❌ Heavy computation on main thread
val sorted = largeList.sortedBy { expensiveFunction(it) }  // main thread

// ✅ Move to Default dispatcher
val sorted = withContext(Dispatchers.Default) {
    largeList.sortedBy { expensiveFunction(it) }
}
```

---

## BroadcastReceiver — Delegate Long Work

```kotlin
// ❌ Long work in onReceive — 10 second limit
class SyncReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        performLongSync()  // may exceed 10s — ANR
    }
}

// ✅ Delegate to WorkManager from onReceive
class SyncReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        // onReceive must return quickly — enqueue work
        WorkManager.getInstance(context)
            .enqueue(OneTimeWorkRequestBuilder<SyncWorker>().build())
    }
}
```

---

## Detecting ANRs in Production

```kotlin
// ✅ Firebase Crashlytics reports ANRs automatically
// Check Firebase Console → Crashlytics → ANRs tab

// ✅ Custom ANR watchdog for more detail
class AnrWatchdog(private val timeoutMs: Long = 5_000) : Thread() {
    private val mainHandler = Handler(Looper.getMainLooper())
    @Volatile private var tick = 0

    override fun run() {
        while (!isInterrupted) {
            val prev = tick
            mainHandler.post { tick++ }
            sleep(timeoutMs)
            if (tick == prev) {
                // Main thread hasn't processed the message — potential ANR
                Timber.e("Potential ANR detected — main thread blocked for ${timeoutMs}ms")
            }
        }
    }
}
```

---

## Reading ANR Traces

```bash
# Pull ANR trace from device
adb pull /data/anr/traces.txt

# Key things to look for in traces.txt:
# "main" thread in state BLOCKED or WAIT
# Holding lock: or waiting to lock:
# Long stack traces in android.database or java.io
```

---

## Anti-Patterns

- `SharedPreferences.commit()` on main thread — use `apply()`
- Room `allowMainThreadQueries()` — never enable in production
- `Bitmap.decodeFile()` on main thread — use coroutine with `Dispatchers.IO`
- Long `synchronized {}` blocks on main thread — can cause lock contention ANR
- `Thread.sleep()` on main thread — blocks all UI processing

---

## Related Skills
- `coroutine` — moving work off the main thread
- `background-processing` — delegating work to background
- `rendering-performance` — keeping main thread free during frames
- `workmanager` — delegating BroadcastReceiver work safely
