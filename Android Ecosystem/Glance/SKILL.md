---
name: glance
description: >
  Jetpack Glance for building App Widgets with Compose-like API.
  Load this skill when creating home screen or lock screen widgets,
  updating widget content, or handling widget interactions.
---

# Glance

## Overview
Jetpack Glance provides a Compose-like API for building Android App Widgets. Instead of XML RemoteViews, Glance lets you write widget UI in a declarative style. Glance is built on top of RemoteViews internally — it translates Composable-like functions to RemoteViews at runtime.

---

## Core Principles

- Glance is **not Compose** — it only supports a subset of Compose-like APIs
- Widget UI runs in a **different process** — no direct state sharing with the app
- Use **GlanceStateDefinition** for widget state — not ViewModel or StateFlow
- Widget updates must be triggered explicitly — they don't auto-update on app state change
- Glance components are a **limited subset** — not all Compose components are available

---

## Setup

```toml
[versions]
glance = "1.1.0"

[libraries]
glance-appwidget = { module = "androidx.glance:glance-appwidget", version.ref = "glance" }
glance-material3 = { module = "androidx.glance:glance-material3", version.ref = "glance" }
```

```kotlin
dependencies {
    implementation(libs.glance.appwidget)
    implementation(libs.glance.material3)
}
```

---

## Widget Definition

```kotlin
// ✅ Define the widget
class MyAppWidget : GlanceAppWidget() {

    override val stateDefinition = MyWidgetStateDefinition

    @Composable
    override fun Content() {
        val state = currentState<MyWidgetState>()
        val context = LocalContext.current

        GlanceTheme {
            MyWidgetContent(
                state = state,
                onRefreshClick = actionRunCallback<RefreshAction>()
            )
        }
    }
}

// ✅ Composable widget UI
@Composable
fun MyWidgetContent(
    state: MyWidgetState,
    onRefreshClick: Action
) {
    Column(
        modifier = GlanceModifier
            .fillMaxSize()
            .background(GlanceTheme.colors.surface)
            .padding(16.dp)
            .cornerRadius(16.dp),
        verticalAlignment = Alignment.CenterVertically
    ) {
        Text(
            text = state.title,
            style = TextStyle(
                color = GlanceTheme.colors.onSurface,
                fontSize = 16.sp,
                fontWeight = FontWeight.Bold
            )
        )

        Spacer(modifier = GlanceModifier.height(8.dp))

        Text(
            text = state.subtitle,
            style = TextStyle(
                color = GlanceTheme.colors.onSurface,
                fontSize = 12.sp
            )
        )

        Spacer(modifier = GlanceModifier.defaultWeight())

        Button(
            text = "Refresh",
            onClick = onRefreshClick
        )
    }
}
```

---

## Widget State

```kotlin
// ✅ Define state model
data class MyWidgetState(
    val title: String = "",
    val subtitle: String = "",
    val isLoading: Boolean = false
)

// ✅ State definition using DataStore
val MyWidgetStateDefinition = object : GlanceStateDefinition<MyWidgetState> {

    override suspend fun getDataStore(
        context: Context,
        fileKey: String
    ): DataStore<MyWidgetState> = DataStoreFactory.create(
        serializer = MyWidgetStateSerializer,
        produceFile = { context.dataStoreFile(fileKey) }
    )

    override fun getLocation(context: Context, fileKey: String): File =
        context.dataStoreFile(fileKey)
}

// ✅ Serializer for widget state
object MyWidgetStateSerializer : Serializer<MyWidgetState> {
    override val defaultValue = MyWidgetState()

    override suspend fun readFrom(input: InputStream): MyWidgetState = try {
        Json.decodeFromString(input.readBytes().decodeToString())
    } catch (e: Exception) {
        defaultValue
    }

    override suspend fun writeTo(t: MyWidgetState, output: OutputStream) {
        output.write(Json.encodeToString(t).toByteArray())
    }
}
```

---

## Widget Receiver

```kotlin
// ✅ GlanceAppWidgetReceiver — entry point for the widget
class MyAppWidgetReceiver : GlanceAppWidgetReceiver() {
    override val glanceAppWidget = MyAppWidget()
}
```

```xml
<!-- AndroidManifest.xml -->
<receiver
    android:name=".widget.MyAppWidgetReceiver"
    android:exported="true">
    <intent-filter>
        <action android:name="android.appwidget.action.APPWIDGET_UPDATE" />
    </intent-filter>
    <meta-data
        android:name="android.appwidget.provider"
        android:resource="@xml/my_widget_info" />
</receiver>
```

```xml
<!-- res/xml/my_widget_info.xml -->
<appwidget-provider
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:minWidth="250dp"
    android:minHeight="110dp"
    android:targetCellWidth="4"
    android:targetCellHeight="2"
    android:resizeMode="horizontal|vertical"
    android:updatePeriodMillis="1800000"
    android:widgetCategory="home_screen"
    android:initialLayout="@layout/glance_default_loading_layout" />
```

---

## Updating Widget State

```kotlin
// ✅ Update widget from app — e.g., from ViewModel or Worker
suspend fun updateWidget(context: Context, newState: MyWidgetState) {
    // Update state in all widget instances
    GlanceAppWidgetManager(context)
        .getGlanceIds(MyAppWidget::class.java)
        .forEach { glanceId ->
            updateAppWidgetState(context, MyWidgetStateDefinition, glanceId) {
                newState
            }
            MyAppWidget().update(context, glanceId)
        }
}

// ✅ Update from WorkManager for periodic refresh
class WidgetUpdateWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result {
        val data = repository.fetchLatestData()
        updateWidget(
            applicationContext,
            MyWidgetState(title = data.title, subtitle = data.subtitle)
        )
        return Result.success()
    }
}
```

---

## Widget Actions

```kotlin
// ✅ Handle button clicks via ActionCallback
class RefreshAction : ActionCallback {
    override suspend fun onAction(
        context: Context,
        glanceId: GlanceId,
        parameters: ActionParameters
    ) {
        // Update widget state on click
        updateAppWidgetState(context, MyWidgetStateDefinition, glanceId) { state ->
            state.copy(isLoading = true)
        }
        MyAppWidget().update(context, glanceId)

        // Fetch new data
        val newData = repository.fetchData()
        updateAppWidgetState(context, MyWidgetStateDefinition, glanceId) {
            MyWidgetState(title = newData.title, subtitle = newData.subtitle)
        }
        MyAppWidget().update(context, glanceId)
    }
}

// ✅ Open app on widget click
val openAppAction = actionStartActivity<MainActivity>()

// ✅ Open specific screen via deep link
val openDetailAction = actionStartActivity(
    Intent(context, MainActivity::class.java).apply {
        putExtra("destination", "detail")
    }
)
```

---

## Glance Components Reference

```kotlin
// ✅ Available layout components
Row { }
Column { }
Box { }
LazyColumn { }  // for lists

// ✅ Available UI components
Text(text = "Hello")
Button(text = "Click", onClick = action)
Image(provider = ImageProvider(R.drawable.icon), contentDescription = null)
CircularProgressIndicator()
LinearProgressIndicator(progress = 0.5f)
Spacer(modifier = GlanceModifier.height(8.dp))
Switch(checked = true, onCheckedChange = action)
CheckBox(checked = false, onCheckedChange = action)

// ✅ GlanceModifier (not Compose Modifier)
GlanceModifier
    .fillMaxSize()
    .fillMaxWidth()
    .wrapContentSize()
    .padding(16.dp)
    .background(Color.White)
    .clickable(onClick = action)
    .cornerRadius(12.dp)
```

---

## Anti-Patterns

- Using Compose components not available in Glance — runtime crash
- Sharing ViewModel state directly with widget — different processes
- Not calling `widget.update()` after state change — UI stays stale
- Complex animations in widgets — not supported, RemoteViews limitation
- Making widgets too complex — widgets should show summary, not replace the app
- Not handling widget resize — use `SizeMode.Responsive` for multiple sizes

---

## Related Skills
- `compose` — Glance has similar but different API
- `datastore` — widget state persistence
- `workmanager` — periodic widget updates
- `app-widget` — traditional RemoteViews-based widgets
