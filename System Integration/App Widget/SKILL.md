---
name: app-widget
description: >
  Home screen App Widget implementation in Android.
  Load this skill when building home screen widgets, updating widget
  content, handling widget interactions, using Glance for Compose-based
  widgets, or configuring widget metadata.
---

# App Widget

## Overview
App Widgets are miniature app views displayed on the home screen or lock screen. They update periodically and respond to user interactions. Modern widgets are best built with **Glance** (Jetpack Compose-based API). Legacy widgets use `RemoteViews`.

---

## Core Principles

- Use **Glance** for new widgets — it provides a Compose-like API over RemoteViews
- Widgets run in a separate process — keep updates lightweight and fast
- Update widgets via `AppWidgetManager` — not direct state mutation
- Provide a `configure` activity if the widget needs user setup
- Minimize update frequency — excessive updates drain battery

---

## Glance Widget (Recommended)

```kotlin
// ✅ Glance widget — Compose-like API
class StatsWidget : GlanceAppWidget() {

    override suspend fun provideGlance(context: Context, id: GlanceId) {
        // Load data before composing
        val stats = statsRepository.getStats()

        provideContent {
            StatsWidgetContent(stats)
        }
    }
}

@Composable
fun StatsWidgetContent(stats: Stats) {
    Column(
        modifier = GlanceModifier
            .fillMaxSize()
            .background(GlanceTheme.colors.widgetBackground)
            .padding(16.dp),
        verticalAlignment = Alignment.CenterVertically
    ) {
        Text(
            text = stats.title,
            style = TextStyle(
                color = GlanceTheme.colors.onSurface,
                fontSize = 16.sp,
                fontWeight = FontWeight.Bold
            )
        )
        Spacer(modifier = GlanceModifier.height(8.dp))
        Text(
            text = stats.value,
            style = TextStyle(
                color = GlanceTheme.colors.primary,
                fontSize = 32.sp
            )
        )
        Button(
            text = "Refresh",
            onClick = actionRunCallback<RefreshAction>()
        )
    }
}
```

---

## Glance Action

```kotlin
// ✅ Handle widget button tap
class RefreshAction : ActionCallback {
    override suspend fun onAction(
        context: Context,
        glanceId: GlanceId,
        parameters: ActionParameters
    ) {
        // Trigger data refresh then update widget
        statsRepository.refresh()
        StatsWidget().update(context, glanceId)
    }
}
```

---

## Glance Widget Receiver

```kotlin
// ✅ AppWidgetReceiver for Glance widgets
class StatsWidgetReceiver : GlanceAppWidgetReceiver() {
    override val glanceAppWidget: GlanceAppWidget = StatsWidget()
}
```

---

## Widget Metadata (XML)

```xml
<!-- res/xml/stats_widget_info.xml -->
<appwidget-provider
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:minWidth="180dp"
    android:minHeight="110dp"
    android:targetCellWidth="2"
    android:targetCellHeight="2"
    android:updatePeriodMillis="1800000"
    android:initialLayout="@layout/widget_loading"
    android:description="@string/widget_description"
    android:previewImage="@drawable/widget_preview"
    android:widgetCategory="home_screen" />
```

---

## Manifest Registration

```xml
<receiver
    android:name=".widget.StatsWidgetReceiver"
    android:exported="true">
    <intent-filter>
        <action android:name="android.appwidget.action.APPWIDGET_UPDATE" />
    </intent-filter>
    <meta-data
        android:name="android.appwidget.provider"
        android:resource="@xml/stats_widget_info" />
</receiver>
```

---

## Updating Widget from App

```kotlin
// ✅ Update all instances of a widget
suspend fun updateAllWidgets(context: Context) {
    val manager = GlanceAppWidgetManager(context)
    val glanceIds = manager.getGlanceIds(StatsWidget::class.java)
    glanceIds.forEach { glanceId ->
        StatsWidget().update(context, glanceId)
    }
}

// ✅ Trigger update from WorkManager for periodic refresh
class WidgetUpdateWorker(context: Context, params: WorkerParameters) :
    CoroutineWorker(context, params) {
    override suspend fun doWork(): Result {
        updateAllWidgets(applicationContext)
        return Result.success()
    }
}
```

---

## Anti-Patterns

- Doing heavy work directly in `provideGlance` on the main thread — use `withContext(Dispatchers.IO)`
- Setting `updatePeriodMillis` too low — minimum enforced is 30 minutes
- Using `RemoteViews` for new widgets when Glance is available — harder to maintain
- Not providing a preview image — widget looks broken in the picker
- Keeping large bitmaps in widget state — causes TransactionTooLargeException

---

## Related Skills
- `glance` — detailed Glance API patterns
- `workmanager` — scheduling periodic widget updates
- `notification` — alternative for time-sensitive updates
- `compose` — Compose fundamentals that inform Glance's API
