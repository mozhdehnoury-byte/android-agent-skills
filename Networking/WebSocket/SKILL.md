---
name: websocket
description: >
  WebSocket communication in Android using OkHttp or Ktor.
  Load this skill when implementing real-time bidirectional communication,
  managing WebSocket connection lifecycle, handling reconnection,
  or processing incoming message streams as Flow.
---

# WebSocket

## Overview
WebSockets provide full-duplex communication over a persistent TCP connection. On Android, OkHttp's `WebSocket` API is the standard for Retrofit-based projects, while Ktor provides native WebSocket support for KMP. Incoming messages are best modeled as a `Flow` to integrate naturally with the reactive architecture.

---

## Core Principles

- Model the WebSocket connection state explicitly — never assume it's always open
- Expose incoming messages as `Flow` — not callbacks
- Handle reconnection with exponential backoff — connections drop unexpectedly
- Close the WebSocket when the lifecycle owner is destroyed
- Never send messages when the connection is not in `OPEN` state

---

## Connection State Model

```kotlin
sealed interface WebSocketState {
    data object Connecting : WebSocketState
    data object Connected : WebSocketState
    data class Message(val data: String) : WebSocketState
    data class Error(val throwable: Throwable) : WebSocketState
    data object Disconnected : WebSocketState
}
```

---

## OkHttp WebSocket

```kotlin
// ✅ WebSocket manager wrapping OkHttp
class WebSocketManager @Inject constructor(
    private val okHttpClient: OkHttpClient,
    private val json: Json
) {
    private var webSocket: WebSocket? = null

    private val _state = MutableSharedFlow<WebSocketState>(
        extraBufferCapacity = 64,
        onBufferOverflow = BufferOverflow.DROP_OLDEST
    )
    val state: SharedFlow<WebSocketState> = _state.asSharedFlow()

    fun connect(url: String) {
        val request = Request.Builder().url(url).build()
        webSocket = okHttpClient.newWebSocket(request, object : WebSocketListener() {

            override fun onOpen(webSocket: WebSocket, response: Response) {
                _state.tryEmit(WebSocketState.Connected)
            }

            override fun onMessage(webSocket: WebSocket, text: String) {
                _state.tryEmit(WebSocketState.Message(text))
            }

            override fun onFailure(webSocket: WebSocket, t: Throwable, response: Response?) {
                _state.tryEmit(WebSocketState.Error(t))
            }

            override fun onClosed(webSocket: WebSocket, code: Int, reason: String) {
                _state.tryEmit(WebSocketState.Disconnected)
            }
        })
        _state.tryEmit(WebSocketState.Connecting)
    }

    fun send(message: String): Boolean {
        return webSocket?.send(message) ?: false
    }

    fun disconnect() {
        webSocket?.close(1000, "Client disconnect")
        webSocket = null
    }
}
```

---

## Ktor WebSocket (KMP)

```kotlin
// ✅ Ktor WebSocket session as Flow
class KtorWebSocketManager @Inject constructor(
    private val client: HttpClient
) {
    fun connect(url: String): Flow<WebSocketState> = callbackFlow {
        try {
            client.webSocket(url) {
                trySend(WebSocketState.Connected)
                for (frame in incoming) {
                    when (frame) {
                        is Frame.Text -> trySend(WebSocketState.Message(frame.readText()))
                        is Frame.Close -> trySend(WebSocketState.Disconnected)
                        else -> Unit
                    }
                }
            }
        } catch (e: Exception) {
            trySend(WebSocketState.Error(e))
        } finally {
            trySend(WebSocketState.Disconnected)
            close()
        }
        awaitClose()
    }
}
```

---

## Reconnection with Backoff

```kotlin
// ✅ Auto-reconnect with exponential backoff
class ReconnectingWebSocket @Inject constructor(
    private val manager: WebSocketManager
) {
    private var reconnectJob: Job? = null

    fun connect(url: String, scope: CoroutineScope) {
        scope.launch {
            manager.state.collect { state ->
                when (state) {
                    is WebSocketState.Error,
                    is WebSocketState.Disconnected -> scheduleReconnect(url, scope)
                    else -> Unit
                }
            }
        }
        manager.connect(url)
    }

    private fun scheduleReconnect(url: String, scope: CoroutineScope) {
        reconnectJob?.cancel()
        reconnectJob = scope.launch {
            var attempt = 0
            while (true) {
                val delay = minOf(1000L * (2.0.pow(attempt)).toLong(), 30_000L)
                delay(delay)
                manager.connect(url)
                attempt++
            }
        }
    }
}
```

---

## Typed Message Handling

```kotlin
// ✅ Parse incoming messages to typed events
@Serializable
sealed interface SocketEvent {
    @Serializable @SerialName("user_updated")
    data class UserUpdated(val user: UserDto) : SocketEvent

    @Serializable @SerialName("notification")
    data class Notification(val message: String) : SocketEvent
}

// ✅ Map raw messages to typed events in ViewModel or repository
val events: Flow<SocketEvent> = webSocketManager.state
    .filterIsInstance<WebSocketState.Message>()
    .mapNotNull { runCatching { json.decodeFromString<SocketEvent>(it.data) }.getOrNull() }
```

---

## Lifecycle Integration

```kotlin
// ✅ Connect/disconnect tied to ViewModel lifecycle
class ChatViewModel @Inject constructor(
    private val webSocket: WebSocketManager
) : ViewModel() {

    init {
        webSocket.connect(BuildConfig.WS_URL)
        viewModelScope.launch {
            webSocket.state.collect { handleState(it) }
        }
    }

    override fun onCleared() {
        webSocket.disconnect()
        super.onCleared()
    }
}
```

---

## Anti-Patterns

- Exposing raw `WebSocketListener` callbacks to the ViewModel — wrap in `Flow`
- Not handling reconnection — mobile networks drop frequently
- Sending messages without checking connection state — silent failures
- Keeping WebSocket open after the user leaves the feature — wastes battery and connection
- Using WebSocket for request/response patterns — use HTTP instead

---

## Related Skills
- `okhttp` — OkHttp client that powers the WebSocket
- `ktor` — Ktor WebSocket for KMP projects
- `flow` — modeling streams of messages
- `server-sent-events` — one-way alternative to WebSocket
- `retry-backoff` — backoff strategy for reconnection
