---
name: alarmmanager
description: >
  AlarmManager for time-based task scheduling in Android.
  Load this skill when scheduling exact-time alarms, implementing
  reminder notifications, triggering actions at a specific time,
  or handling cases where WorkManager's timing precision is insufficient.
---

# AlarmManager

## Overview
AlarmManager schedules tasks to run at a specific time, even if the app is not running. It is the correct tool when **exact timing** matters — reminder alarms, calendar events, scheduled notifications. For flexible/deferrable work, WorkManager is preferred. Since Android 12, exact alarms require explicit permission.

---

## Core Principles

- Use AlarmManager only when **exact timing** is required — use WorkManager for flexible scheduling
- Request `SCHEDULE_EXACT_ALARM` or `USE_EXACT_ALARM` permission for exact alarms on Android 12+
- Always use `PendingIntent` with `FLAG_IMMUTABLE` — required on Android 12+
- Re-schedule alarms after device reboot via `BOOT_COMPLETED` receiver
- Cancel alarms when they are no longer needed — they persist across app restarts

---

## Permissions

```xml
<!-- AndroidManifest.xml -->

<!-- For exact alarms — user must grant in settings (Android 12+) -->
<uses-permission android:name="android.permission.SCHEDULE_EXACT_ALARM" />

<!-- OR for specific use cases (clock, calendar apps) — always granted -->
<uses-permission android:name="android.permission.USE_EXACT_ALARM" />

<!-- For BroadcastReceiver to survive reboot -->
<uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED" />
```

---

## Scheduling an Exact Alarm

```kotlin
// ✅ Schedule exact alarm
class ReminderScheduler @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private val alarmManager = context.getSystemService(AlarmManager::class.java)

    fun scheduleReminder(reminderId: Long, triggerAtMillis: Long) {
        val intent = Intent(context, ReminderReceiver::class.java).apply {
            putExtra(ReminderReceiver.EXTRA_REMINDER_ID, reminderId)
        }

        val pendingIntent = PendingIntent.getBroadcast(
            context,
            reminderId.toInt(),
            intent,
            PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
        )

        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
            if (alarmManager.canScheduleExactAlarms()) {
                alarmManager.setExactAndAllowWhileIdle(
                    AlarmManager.RTC_WAKEUP,
                    triggerAtMillis,
                    pendingIntent
                )
            }
            // else: redirect user to settings
        } else {
            alarmManager.setExactAndAllowWhileIdle(
                AlarmManager.RTC_WAKEUP,
                triggerAtMillis,
                pendingIntent
            )
        }
    }

    fun cancelReminder(reminderId: Long) {
        val intent = Intent(context, ReminderReceiver::class.java)
        val pendingIntent = PendingIntent.getBroadcast(
            context,
            reminderId.toInt(),
            intent,
            PendingIntent.FLAG_NO_CREATE or PendingIntent.FLAG_IMMUTABLE
        )
        pendingIntent?.let { alarmManager.cancel(it) }
    }
}
```

---

## BroadcastReceiver

```kotlin
// ✅ Receiver triggered by alarm
class ReminderReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        val reminderId = intent.getLongExtra(EXTRA_REMINDER_ID, -1L)
        if (reminderId == -1L) return

        // Show notification — short work only in onReceive
        NotificationHelper.showReminder(context, reminderId)
    }

    companion object {
        const val EXTRA_REMINDER_ID = "reminder_id"
    }
}

// Register in manifest
// <receiver android:name=".ReminderReceiver" android:exported="false" />
```

---

## Reschedule After Reboot

```kotlin
// ✅ Reboot receiver — reschedules all pending alarms
class BootReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        if (intent.action != Intent.ACTION_BOOT_COMPLETED) return

        // Reschedule all pending reminders from local DB
        val scope = CoroutineScope(SupervisorJob() + Dispatchers.IO)
        scope.launch {
            val scheduler = ReminderScheduler(context)
            reminderRepository.getPendingReminders().forEach { reminder ->
                scheduler.scheduleReminder(reminder.id, reminder.triggerAt)
            }
        }
    }
}

// Register in manifest
// <receiver android:name=".BootReceiver" android:exported="true">
//     <intent-filter>
//         <action android:name="android.intent.action.BOOT_COMPLETED" />
//     </intent-filter>
// </receiver>
```

---

## Directing User to Exact Alarm Settings

```kotlin
// ✅ On Android 12+ — direct user to grant exact alarm permission
fun requestExactAlarmPermission(activity: Activity) {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
        val alarmManager = activity.getSystemService(AlarmManager::class.java)
        if (!alarmManager.canScheduleExactAlarms()) {
            val intent = Intent(Settings.ACTION_REQUEST_SCHEDULE_EXACT_ALARM)
            activity.startActivity(intent)
        }
    }
}
```

---

## Anti-Patterns

- Using AlarmManager for network sync or deferrable tasks — use WorkManager
- Not rescheduling alarms after reboot — alarms are lost on device restart
- Using `FLAG_MUTABLE` for PendingIntent on Android 12+ — SecurityException
- Using `set()` instead of `setExactAndAllowWhileIdle()` — alarm delayed in Doze mode
- Doing heavy work in `onReceive` — it runs on the main thread; delegate to a service or WorkManager

---

## Related Skills
- `workmanager` — preferred for deferrable background work
- `notification` — showing notifications from the alarm receiver
- `broadcast-receiver` — BroadcastReceiver fundamentals
- `background-processing` — choosing the right scheduling tool
