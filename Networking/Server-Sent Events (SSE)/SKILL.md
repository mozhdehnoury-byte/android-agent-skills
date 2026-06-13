---
name: server-sent-events
description: >
  Server-Sent Events (SSE) implementation in Android.
  Load this skill when consuming a one-way real-time event stream from
  the server, integrating SSE with AI streaming responses, handling
  reconnection, or modeling SSE as a Flow.
---

# Server-Sent Events (SSE)

## Overview
Server-Sent Events (SSE) is a one-way, server-to-client streaming protocol over HTTP. Unlike WebSockets, SSE is unidirectional and uses standard HTTP — making it simpler and more firewall-friendly. It is commonly used for AI streaming responses (LLMs), live notifications, and real-time dashboards.

---

## Core Principles

- SSE is one-way — use WebSocket if bidirectional communication is needed
- Model the SSE stream as a `Flow` — integrates naturally with coroutines
- Handle reconnection — SSE streams drop on network changes
- Parse the `data:` field of each event — ignore comment lines (starting with `:`)
- Close the connection when the lifecycle owner is destroyed

---

## OkHttp SSE Implementation

```kotlin
// ✅ SSE client wrapping OkHttp EventSource
class SseClient @Inject constructor(
    private val okHttpClient: OkHttpClient
) {
    fun connect(url: String, headers: Map<String, String> = emptyMap()): Flow<SseEvent> =
        callbackFlow {
            val request = Request.Builder()
                .url(url)
                .apply { headers.forEach { (k, v) -> addHeader(k, v) } }
                .addHeader("Accept", "text/event-stream")
                .addHeader("Cache-Control", "no-cache")
                .build()

            val listener = object : EventSourceListener() {
                override fun onEvent(
                    eventSource: EventSource,
                    id: String?,
                    type: String?,
                    data: String
                ) {
                    trySend(SseEvent.Data(id = id, type = type, data = data))
                }

                override fun onFailure(
                    eventSource: EventSource,
                    t: Throwable?,
                    response: Response?
                ) {
                    close(t ?: IOException("SSE connection failed"))
                }

                override fun onClosed(eventSource: EventSource) {
                    trySend(SseEvent.Closed)
                    close()
                }
            }

            val eventSource = EventSources.createFactory(okHttpClient)
                .newEventSource(request, listener)

            awaitClose { eventSource.cancel() }
        }
}

sealed interface SseEvent {
    data class Data(val id: String?, val type: String?, val data: String) : SseEvent
    data object Closed : SseEvent
}
```

---

## Manual SSE Parsing (Without EventSource)

```kotlin
// ✅ Parse SSE stream manually from OkHttp response body
fun parseSSEStream(responseBody: ResponseBody): Flow<String> = flow {
    responseBody.source().use { source ->
        while (!source.exhausted()) {
            val line = source.readUtf8Line() ?: break
            when {
                line.startsWith("data: ") -> emit(line.removePrefix("data: "))
                line.startsWith(":") -> Unit  // comment — ignore
                line.isEmpty() -> Unit        // event separator — ignore
            }
        }
    }
}

// ✅ Usage with Retrofit streaming endpoint
@Streaming
@GET("chat/stream")
suspend fun streamChat(@Query("prompt") prompt: String): ResponseBody
```

---

## AI Streaming Response

```kotlin
// ✅ Stream LLM token-by-token response
class ChatRepository @Inject constructor(
    private val sseClient: SseClient,
    private val tokenStorage: TokenStorage
) {
    fun streamCompletion(prompt: String): Flow<String> {
        return sseClient.connect(
            url = "${BuildConfig.BASE_URL}chat/completions",
            headers = mapOf(
                "Authorization" to "Bearer ${tokenStorage.getAccessToken()}"
            )
        )
        .filterIsInstance<SseEvent.Data>()
        .filter { it.data != "[DONE]" }
        .mapNotNull { event ->
            runCatching {
                json.decodeFromString<CompletionChunkDto>(event.data)
                    .choices.firstOrNull()?.delta?.content
            }.getOrNull()
        }
    }
}

// ✅ Collect in ViewModel — append tokens to build response
class ChatViewModel @Inject constructor(
    private val chatRepository: ChatRepository
) : ViewModel() {

    private val _response = MutableStateFlow("")
    val response: StateFlow<String> = _response.asStateFlow()

    private val _isStreaming = MutableStateFlow(false)
    val isStreaming: StateFlow<Boolean> = _isStreaming.asStateFlow()

    fun sendMessage(prompt: String) {
        viewModelScope.launch {
            _response.value = ""
            _isStreaming.value = true
            chatRepository.streamCompletion(prompt)
                .onCompletion { _isStreaming.value = false }
                .catch { _isStreaming.value = false }
                .collect { token ->
                    _response.value += token
                }
        }
    }
}
```

---

## Reconnection

```kotlin
// ✅ Auto-reconnect on failure with backoff
fun connectWithReconnect(url: String, scope: CoroutineScope): Flow<SseEvent> = flow {
    var attempt = 0
    while (true) {
        try {
            emitAll(sseClient.connect(url))
            attempt = 0  // reset on clean close
        } catch (e: IOException) {
            val delay = minOf(1_000L * (2.0.pow(attempt)).toLong(), 30_000L)
            delay(delay)
            attempt++
        }
    }
}.flowOn(Dispatchers.IO)
```

---

## Anti-Patterns

- Using SSE for bidirectional communication — use WebSocket instead
- Not closing the EventSource on lifecycle destroy — leaks the connection
- Accumulating all SSE events in memory — process and discard each event
- Not handling the `[DONE]` sentinel for AI streams — continues waiting indefinitely
- Blocking the main thread while reading the stream — always on IO dispatcher

---

## Related Skills
- `websocket` — bidirectional alternative to SSE
- `okhttp` — HTTP client for SSE connections
- `flow` — modeling streams with Kotlin Flow
- `retrofit` — streaming response body with `@Streaming`
