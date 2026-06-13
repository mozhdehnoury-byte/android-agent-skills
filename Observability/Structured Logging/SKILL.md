---
name: structured-logging
description: >
  Structured logging for Android apps — consistent, queryable log format.
  Load this skill when building log aggregation pipelines, adding context
  to log events, correlating logs across requests, or making logs
  machine-parseable for log management tools.
---

# Structured Logging

## Overview
Structured logging formats log entries as key-value pairs or JSON instead of free-form strings. This makes logs searchable, filterable, and correlatable in log management systems (Datadog, Elastic, Splunk). On Android, structured logs are typically sent to Crashlytics or a custom logging backend.

---

## Core Principles

- Every log event has a **consistent schema** — event name + context fields
- Include **correlation IDs** to trace a request across multiple log entries
- Log **events**, not state — "user_login_succeeded" not "user is logged in"
- Never include sensitive data in structured logs — same rule as plain logs
- Use a typed log event model — not free-form strings

---

## Log Event Model

```kotlin
// ✅ Typed log event — consistent schema
data class LogEvent(
    val name: String,
    val level: LogLevel,
    val timestamp: Long = System.currentTimeMillis(),
    val userId: String? = null,
    val sessionId: String? = null,
    val properties: Map<String, Any> = emptyMap(),
    val error: Throwable? = null
)

enum class LogLevel { DEBUG, INFO, WARN, ERROR }

// ✅ Event name constants — prevents typos
object LogEvents {
    const val USER_LOGIN_SUCCESS    = "user_login_success"
    const val USER_LOGIN_FAILURE    = "user_login_failure"
    const val SYNC_STARTED          = "sync_started"
    const val SYNC_COMPLETED        = "sync_completed"
    const val SYNC_FAILED           = "sync_failed"
    const val NETWORK_REQUEST       = "network_request"
    const val NETWORK_ERROR         = "network_error"
}
```

---

## Structured Logger

```kotlin
// ✅ Central structured logger
@Singleton
class StructuredLogger @Inject constructor(
    private val sessionManager: SessionManager,
    private val crashlytics: FirebaseCrashlytics
) {
    fun log(
        name: String,
        level: LogLevel = LogLevel.INFO,
        properties: Map<String, Any> = emptyMap(),
        error: Throwable? = null
    ) {
        val event = LogEvent(
            name = name,
            level = level,
            userId = sessionManager.currentUserId,
            sessionId = sessionManager.sessionId,
            properties = properties,
            error = error
        )

        // Debug: log to Logcat
        if (BuildConfig.DEBUG) {
            Timber.tag("StructuredLog").d(event.toLogString())
        }

        // Release: send to Crashlytics
        crashlytics.log(event.toLogString())
        error?.let { crashlytics.recordException(it) }
    }

    private fun LogEvent.toLogString(): String {
        val props = properties.entries.joinToString(", ") { "${it.key}=${it.value}" }
        return "[$level] $name | userId=$userId sessionId=$sessionId | $props"
    }
}
```

---

## Usage in Repository and Use Cases

```kotlin
// ✅ Log structured events at key boundaries
class AuthRepositoryImpl @Inject constructor(
    private val api: AuthApi,
    private val logger: StructuredLogger
) : AuthRepository {

    override suspend fun login(email: String, password: String): Result<User> {
        return runCatching {
            val response = api.login(email, password)
            logger.log(
                name = LogEvents.USER_LOGIN_SUCCESS,
                properties = mapOf("method" to "email")
            )
            mapper.toDomain(response)
        }.onFailure { error ->
            logger.log(
                name = LogEvents.USER_LOGIN_FAILURE,
                level = LogLevel.ERROR,
                properties = mapOf(
                    "method" to "email",
                    "error_type" to error::class.simpleName.orEmpty()
                ),
                error = error
            )
        }
    }
}
```

---

## Network Request Logging

```kotlin
// ✅ Log all network requests with structured context
class StructuredLoggingInterceptor @Inject constructor(
    private val logger: StructuredLogger
) : Interceptor {

    override fun intercept(chain: Interceptor.Chain): Response {
        val request = chain.request()
        val startTime = System.currentTimeMillis()

        return try {
            val response = chain.proceed(request)
            val duration = System.currentTimeMillis() - startTime

            logger.log(
                name = LogEvents.NETWORK_REQUEST,
                properties = mapOf(
                    "method"   to request.method,
                    "path"     to request.url.encodedPath,
                    "status"   to response.code,
                    "duration" to duration
                )
            )
            response
        } catch (e: IOException) {
            logger.log(
                name = LogEvents.NETWORK_ERROR,
                level = LogLevel.ERROR,
                properties = mapOf(
                    "method" to request.method,
                    "path"   to request.url.encodedPath
                ),
                error = e
            )
            throw e
        }
    }
}
```

---

## Session Correlation

```kotlin
// ✅ Generate session ID to correlate logs within a session
@Singleton
class SessionManager @Inject constructor() {
    var currentUserId: String? = null
        private set

    var sessionId: String = generateSessionId()
        private set

    fun onUserLoggedIn(userId: String) {
        currentUserId = userId
        sessionId = generateSessionId()  // new session per login
    }

    fun onUserLoggedOut() {
        currentUserId = null
        sessionId = generateSessionId()
    }

    private fun generateSessionId() = UUID.randomUUID().toString().take(8)
}
```

---

## Crashlytics Key-Value Context

```kotlin
// ✅ Set Crashlytics keys for crash context
fun setCrashlyticsContext(userId: String, plan: String) {
    val crashlytics = FirebaseCrashlytics.getInstance()
    crashlytics.setUserId(userId)
    crashlytics.setCustomKey("user_plan", plan)
    crashlytics.setCustomKey("app_version", BuildConfig.VERSION_NAME)
    crashlytics.setCustomKey("session_id", sessionManager.sessionId)
}
```

---

## Anti-Patterns

- Free-form log strings — not queryable or consistent
- Logging sensitive data (email, tokens) as properties — privacy violation
- No event name constants — typos cause missed log entries
- Logging too many events — drowns out important signals
- Not including session/user context — can't correlate related events

---

## Related Skills
- `logging` — basic Timber setup and log levels
- `crash-reporting` — integrating structured logs with crash reports
- `observability` — metrics and telemetry alongside logs
