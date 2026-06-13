---
name: api-contract
description: >
  Defining and consuming API contracts in Android projects.
  Load this skill when documenting API expectations, defining request/response
  DTOs, handling API versioning, managing breaking changes, or aligning
  the data layer with backend contracts.
---

# API Contract

## Overview
An API contract is the formal agreement between client and server about request/response structure, status codes, error formats, and versioning. A well-defined contract prevents runtime surprises, makes the data layer predictable, and provides a stable foundation for mocking in tests.

---

## Core Principles

- Define all request/response shapes as `@Serializable` DTOs — never use `Map<String, Any>`
- Document the expected status codes for every endpoint
- Version the API from day one — breaking changes require a new version
- Use `ignoreUnknownKeys = true` in the Json instance — tolerates additive API changes
- Never couple domain models to the API contract — always map through DTOs

---

## Contract Definition

```kotlin
// ✅ Request DTOs — what we send
@Serializable
data class CreateUserRequest(
    @SerialName("full_name") val name: String,
    @SerialName("email") val email: String,
    @SerialName("password") val password: String
)

@Serializable
data class UpdateUserRequest(
    @SerialName("full_name") val name: String? = null,
    @SerialName("email") val email: String? = null
)

// ✅ Response DTOs — what we receive
@Serializable
data class UserDto(
    @SerialName("id") val id: String,
    @SerialName("full_name") val name: String,
    @SerialName("email") val email: String,
    @SerialName("avatar_url") val avatarUrl: String? = null,
    @SerialName("created_at") val createdAt: Long,
    @SerialName("is_active") val isActive: Boolean = true
)

// ✅ Error response shape
@Serializable
data class ApiErrorResponse(
    @SerialName("code") val code: String,
    @SerialName("message") val message: String,
    @SerialName("details") val details: Map<String, List<String>> = emptyMap()
)
```

---

## Endpoint Contract Documentation

```kotlin
// ✅ Document contract as comments on the API interface
interface UserApi {

    /**
     * GET /users/{id}
     *
     * Success: 200 UserDto
     * Not Found: 404 ApiErrorResponse
     * Unauthorized: 401 ApiErrorResponse
     */
    @GET("users/{id}")
    suspend fun getUser(@Path("id") id: String): UserDto

    /**
     * POST /users
     *
     * Success: 201 UserDto
     * Validation Error: 422 ApiErrorResponse (details contains field errors)
     * Conflict: 409 ApiErrorResponse (email already exists)
     */
    @POST("users")
    suspend fun createUser(@Body request: CreateUserRequest): UserDto

    /**
     * DELETE /users/{id}
     *
     * Success: 204 (no body)
     * Not Found: 404 ApiErrorResponse
     * Forbidden: 403 ApiErrorResponse
     */
    @DELETE("users/{id}")
    suspend fun deleteUser(@Path("id") id: String)
}
```

---

## API Versioning

```kotlin
// ✅ Version in base URL
val retrofit = Retrofit.Builder()
    .baseUrl("https://api.example.com/v1/")
    .build()

// ✅ Version in header (for minor versions)
class ApiVersionInterceptor : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val request = chain.request().newBuilder()
            .addHeader("API-Version", "2024-01-01")
            .build()
        return chain.proceed(request)
    }
}

// ✅ Separate API interfaces per major version
interface UserApiV1 {
    @GET("users/{id}")
    suspend fun getUser(@Path("id") id: String): UserDtoV1
}

interface UserApiV2 {
    @GET("users/{id}")
    suspend fun getUser(@Path("id") id: String): UserDtoV2
}
```

---

## Handling Additive Changes (Non-Breaking)

```kotlin
// ✅ New optional fields — add with default value
@Serializable
data class UserDto(
    @SerialName("id") val id: String,
    @SerialName("name") val name: String,
    // New field added by backend — safe with default
    @SerialName("phone") val phone: String? = null,
    @SerialName("tier") val tier: String = "free"
)
```

---

## Handling Breaking Changes

```kotlin
// ✅ Field renamed — use @SerialName to maintain compatibility
@Serializable
data class UserDto(
    // Backend renamed "username" to "full_name"
    // Keep old mapping until all clients are updated
    @SerialName("full_name") val name: String
)

// ✅ Type changed — custom serializer as bridge
@Serializable
data class OrderDto(
    // Backend changed status from Int to String
    @Serializable(with = OrderStatusSerializer::class)
    @SerialName("status") val status: OrderStatus
)
```

---

## Contract Testing with Fake

```kotlin
// ✅ Fake API implementation for tests — matches the real contract
class FakeUserApi : UserApi {
    val users = mutableListOf<UserDto>()

    override suspend fun getUser(id: String): UserDto {
        return users.firstOrNull { it.id == id }
            ?: throw HttpException(
                Response.error<UserDto>(404, "".toResponseBody())
            )
    }

    override suspend fun createUser(request: CreateUserRequest): UserDto {
        val dto = UserDto(
            id = UUID.randomUUID().toString(),
            name = request.name,
            email = request.email,
            createdAt = System.currentTimeMillis()
        )
        users.add(dto)
        return dto
    }
}
```

---

## Anti-Patterns

- Using `Map<String, Any>` for request/response — loses type safety
- Not documenting expected status codes per endpoint
- Coupling domain models directly to API response structure
- Ignoring `@SerialName` — breaks when API uses snake_case
- No versioning strategy — every backend change becomes a breaking change

---

## Related Skills
- `serialization` — Kotlinx Serialization for DTO definition
- `dto-mapping` — mapping DTOs to domain models
- `rest` — HTTP conventions and status codes
- `retrofit` — implementing the API interface
- `error-handling` — mapping API errors to domain errors
