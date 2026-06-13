---
name: metrics
description: >
  Application metrics and telemetry for Android apps.
  Load this skill when tracking performance metrics, measuring feature
  adoption, monitoring error rates, capturing user behavior signals,
  or building a custom metrics pipeline.
---

# Metrics

## Overview
Metrics provide quantitative signals about app health and user behavior — error rates, response times, feature usage, and performance. On Android, metrics are typically sent via Firebase Analytics, Firebase Performance Monitoring, or a custom backend. The goal is to detect regressions and understand usage patterns before users complain.

---

## Core Principles

- Track **outcomes**, not implementation details — "checkout_completed" not "button_tapped"
- Every metric must be **actionable** — if you can't act on it, don't track it
- Separate **performance metrics** from **business metrics** — different stakeholders
- Avoid tracking PII — user IDs are fine, email addresses are not
- Define metrics **before** shipping a feature — don't instrument retroactively

---

## Firebase Performance Monitoring

```toml
# libs.versions.toml
[libraries]
firebase-perf = { module = "com.google.firebase:firebase-perf-ktx" }
```

```kotlin
// ✅ Custom trace for critical operations
class SyncRepository @Inject constructor(
    private val firebasePerf: FirebasePerformance
) {
    suspend fun syncData(): Result<Unit> {
        val trace = firebasePerf.newTrace("sync_data")
        trace.start()

        return runCatching {
            val result = performSync()
            trace.putMetric("items_synced", result.itemCount.toLong())
            trace.putAttribute("sync_type", result.type)
            result
        }.onSuccess {
            trace.putAttribute("status", "success")
        }.onFailure {
            trace.putAttribute("status", "failure")
            trace.putAttribute("error", it::class.simpleName ?: "unknown")
        }.also {
            trace.stop()
        }.map { Unit }
    }
}
```

---

## Custom Metrics Tracker

```kotlin
// ✅ Typed metrics tracker wrapping Firebase Analytics
@Singleton
class MetricsTracker @Inject constructor(
    private val analytics: FirebaseAnalytics,
    private val firebasePerf: FirebasePerformance
) {
    // ✅ Business event tracking
    fun trackEvent(event: MetricEvent) {
        val bundle = Bundle().apply {
            event.properties.forEach { (key, value) ->
                when (value) {
                    is String -> putString(key, value)
                    is Long   -> putLong(key, value)
                    is Double -> putDouble(key, value)
                    is Int    -> putInt(key, value)
                    is Boolean -> putBoolean(key, value)
                }
            }
        }
        analytics.logEvent(event.name, bundle)
    }

    // ✅ Performance timing
    fun <T> measureTrace(name: String, block: () -> T): T {
        val trace = firebasePerf.newTrace(name)
        trace.start()
        return try {
            block()
        } finally {
            trace.stop()
        }
    }

    // ✅ Async performance timing
    suspend fun <T> measureTraceAsync(name: String, block: suspend () -> T): T {
        val trace = firebasePerf.newTrace(name)
        trace.start()
        return try {
            block()
        } finally {
            trace.stop()
        }
    }
}
```

---

## Metric Events

```kotlin
// ✅ Typed event definitions
sealed class MetricEvent(
    val name: String,
    val properties: Map<String, Any> = emptyMap()
) {
    class FeatureViewed(featureName: String) : MetricEvent(
        name = "feature_viewed",
        properties = mapOf("feature" to featureName)
    )

    class ActionCompleted(action: String, durationMs: Long) : MetricEvent(
        name = "action_completed",
        properties = mapOf("action" to action, "duration_ms" to durationMs)
    )

    class ErrorOccurred(errorType: String, screen: String) : MetricEvent(
        name = "error_occurred",
        properties = mapOf("error_type" to errorType, "screen" to screen)
    )

    class SyncCompleted(itemCount: Int, durationMs: Long, success: Boolean) : MetricEvent(
        name = "sync_completed",
        properties = mapOf(
            "item_count" to itemCount,
            "duration_ms" to durationMs,
            "success" to success
        )
    )
}
```

---

## Error Rate Tracking

```kotlin
// ✅ Track error rates per feature
class UserRepository @Inject constructor(
    private val metrics: MetricsTracker
) {
    suspend fun getUser(id: String): Result<User> {
        val startTime = System.currentTimeMillis()

        return runCatching { api.getUser(id) }
            .onSuccess {
                metrics.trackEvent(MetricEvent.ActionCompleted(
                    action = "fetch_user",
                    durationMs = System.currentTimeMillis() - startTime
                ))
            }
            .onFailure { error ->
                metrics.trackEvent(MetricEvent.ErrorOccurred(
                    errorType = error::class.simpleName ?: "unknown",
                    screen = "user_detail"
                ))
            }
            .map { mapper.toDomain(it) }
    }
}
```

---

## Network Request Metrics

```kotlin
// ✅ Track all network request performance
class MetricsInterceptor @Inject constructor(
    private val metrics: MetricsTracker
) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val start = System.currentTimeMillis()
        val request = chain.request()

        return try {
            val response = chain.proceed(request)
            val duration = System.currentTimeMillis() - start

            metrics.trackEvent(MetricEvent.ActionCompleted(
                action = "network_${request.method.lowercase()}_${request.url.encodedPath}",
                durationMs = duration
            ))
            response
        } catch (e: Exception) {
            metrics.trackEvent(MetricEvent.ErrorOccurred(
                errorType = "network_${e::class.simpleName}",
                screen = request.url.encodedPath
            ))
            throw e
        }
    }
}
```

---

## Anti-Patterns

- Tracking every tap and scroll — creates noise, hard to find signal
- Tracking PII (email, name) as metric properties — privacy violation
- Not defining metric names as constants — typos create duplicate metrics
- Tracking metrics in debug builds — pollutes production dashboards
- No baseline — can't detect regression without knowing the previous value

---

## Related Skills
- `crash-reporting` — crash data alongside error metrics
- `structured-logging` — log events that feed into metrics pipelines
- `analytics` — Firebase Analytics for user behavior
- `observability` — broader observability combining logs, metrics, and traces
