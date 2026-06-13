---
name: ktor
description: >
  Ktor HTTP client setup and usage for Android and KMP projects.
  Load this skill when using Ktor as the HTTP client, configuring
  plugins, defining typed API calls, handling responses, or building
  a shared networking layer in a Kotlin Multiplatform project.
---

# Ktor

## Overview
Ktor Client is a multiplatform HTTP client built on coroutines. It is the preferred choice for KMP projects where the networking layer is shared between Android and iOS. It uses a plugin-based architecture for features like serialization, auth, logging, and retry.

---

## Core Principles

- Use Ktor when the networking layer must be shared across platforms (KMP)
- Use Retrofit for Android-only projects — Ktor for KMP
- Configure one `HttpClient` instance per app — provide via DI
- All requests are `suspend` functions — no callbacks
- Handle errors at the repository level — wrap in `Result`

---

## Setup

```toml
# libs.versions.toml
[versions]
ktor = "2.3.12"

[libraries]
ktor-client-core         = { module = "io.ktor:ktor-client-core", version.ref = "ktor" }
ktor-client-okhttp       = { module = "io.ktor:ktor-client-okhttp", version.ref = "ktor" }    # Android
ktor-client-darwin       = { module = "io.ktor:ktor-client-darwin", version.ref = "ktor" }    # iOS
ktor-client-content-negotiation = { module = "io.ktor:ktor-client-content-negotiation", version.ref = "ktor" }
ktor-serialization-kotlinx-json = { module = "io.ktor:ktor-serialization-kotlinx-json", version.ref = "ktor" }
ktor-client-logging      = { module = "io.ktor:ktor-client-logging", version.ref = "ktor" }
ktor-client-auth         = { module = "io.ktor:ktor-client-auth", version.ref = "ktor" }
```

---

## Client Configuration

```kotlin
// ✅ Shared HttpClient — configured once
fun createHttpClient(json: Json): HttpClient {
    return HttpClient(OkHttp) {  // use Darwin engine on iOS
        // Serialization
        install(ContentNegotiation) {
            json(json)
        }

        // Logging
        install(Logging) {
            logger = object : Logger {
                override fun log(message: String) {
                    println("Ktor: $message")
                }
            }
            level = if (BuildConfig.DEBUG) LogLevel.BODY else LogLevel.NONE
        }

        // Auth
        install(Auth) {
            bearer {
                loadTokens {
                    BearerTokens(
                        accessToken = tokenStorage.getAccessToken() ?: "",
                        refreshToken = tokenStorage.getRefreshToken() ?: ""
                    )
                }
                refreshTokens {
                    val newTokens = tokenApi.refresh(oldTokens?.refreshToken ?: "")
                    tokenStorage.save(newTokens)
                    BearerTokens(newTokens.accessToken, newTokens.refreshToken)
                }
            }
        }

        // Default request config
        defaultRequest {
            url(BuildConfig.BASE_URL)
            contentType(ContentType.Application.Json)
            header("X-Platform", "android")
        }

        // Timeouts
        install(HttpTimeout) {
            requestTimeoutMillis = 30_000
            connectTimeoutMillis = 15_000
            socketTimeoutMillis = 30_000
        }
    }
}
```

---

## API Calls

```kotlin
// ✅ Typed API functions using Ktor DSL
class UserRemoteDataSource @Inject constructor(
    private val client: HttpClient
) {
    suspend fun getUsers(): List<UserDto> {
        return client.get("users").body()
    }

    suspend fun getUser(id: String): UserDto {
        return client.get("users/$id").body()
    }

    suspend fun createUser(request: CreateUserRequest): UserDto {
        return client.post("users") {
            setBody(request)
        }.body()
    }

    suspend fun updateUser(id: String, request: UpdateUserRequest): UserDto {
        return client.put("users/$id") {
            setBody(request)
        }.body()
    }

    suspend fun deleteUser(id: String) {
        client.delete("users/$id")
    }

    suspend fun searchUsers(query: String, page: Int): PagedResponse<UserDto> {
        return client.get("users") {
            parameter("q", query)
            parameter("page", page)
        }.body()
    }
}
```

---

## Error Handling

```kotlin
// ✅ Wrap Ktor calls in runCatching at the repository level
override suspend fun getUser(id: String): Result<User> = runCatching {
    val dto = remoteDataSource.getUser(id)
    mapper.toDomain(dto)
}.recoverCatching { throwable ->
    when (throwable) {
        is ClientRequestException -> throw ApiException(
            code = throwable.response.status.value,
            message = throwable.response.bodyAsText()
        )
        is ServerResponseException -> throw ServerException(
            code = throwable.response.status.value
        )
        is IOException -> throw NetworkException("No internet")
        else -> throw throwable
    }
}
```

---

## File Upload

```kotlin
// ✅ Multipart upload with Ktor
suspend fun uploadAvatar(userId: String, file: ByteArray, fileName: String): AvatarDto {
    return client.submitFormWithBinaryData(
        url = "users/$userId/avatar",
        formData = formData {
            append("avatar", file, Headers.build {
                append(HttpHeaders.ContentType, "image/jpeg")
                append(HttpHeaders.ContentDisposition, "filename=$fileName")
            })
        }
    ).body()
}
```

---

## KMP Engine Selection

```kotlin
// ✅ Expect/actual for engine selection in KMP
// commonMain
expect fun createEngine(): HttpClientEngine

// androidMain
actual fun createEngine(): HttpClientEngine = OkHttp.create()

// iosMain
actual fun createEngine(): HttpClientEngine = Darwin.create()
```

---

## Anti-Patterns

- Using Ktor in Android-only projects when Retrofit is simpler and more established
- Creating a new `HttpClient` per request — expensive, use singleton
- Not installing `HttpTimeout` — requests can hang indefinitely
- Catching exceptions in the data source — wrap at repository level
- Using string concatenation for URL building — use `parameter()` and path segments

---

## Related Skills
- `retrofit` — alternative for Android-only projects
- `serialization` — Kotlinx Serialization shared Json instance
- `kmp` — KMP project structure and engine selection
- `authentication` — token management with Ktor Auth plugin
- `retry-backoff` — retry plugin for Ktor
