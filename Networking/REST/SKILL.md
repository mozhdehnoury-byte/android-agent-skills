---
name: rest
description: >
  REST API design principles and consumption patterns for Android.
  Load this skill when designing API contracts, structuring HTTP requests,
  handling standard HTTP status codes, modeling API responses,
  or understanding REST conventions used in the data layer.
---

# REST

## Overview
REST (Representational State Transfer) is the standard architectural style for HTTP APIs. Understanding REST conventions ensures the data layer handles responses, errors, and status codes correctly and consistently.

---

## Core Principles

- Use the correct HTTP method for each operation — GET, POST, PUT, PATCH, DELETE
- Map HTTP status codes to domain errors — never ignore non-2xx responses
- Use query parameters for filtering/sorting/pagination — path parameters for resource identity
- Requests and responses use consistent naming — snake_case in JSON, camelCase in Kotlin DTOs
- Idempotent operations (GET, PUT, DELETE) are safe to retry — non-idempotent (POST) are not

---

## HTTP Method Mapping

| Method | Use For | Idempotent |
|--------|---------|------------|
| `GET` | Fetch resource(s) | ✅ Yes |
| `POST` | Create new resource | ❌ No |
| `PUT` | Replace entire resource | ✅ Yes |
| `PATCH` | Partial update | ❌ No |
| `DELETE` | Remove resource | ✅ Yes |

---

## Standard Endpoints Pattern

```
GET    /users           → list users
POST   /users           → create user
GET    /users/{id}      → get user by id
PUT    /users/{id}      → replace user
PATCH  /users/{id}      → partial update user
DELETE /users/{id}      → delete user

GET    /users/{id}/posts         → posts belonging to user
POST   /users/{id}/posts         → create post for user
```

---

## HTTP Status Code Handling

```kotlin
// ✅ Map status codes to domain errors
sealed class ApiError : Exception() {
    data class BadRequest(override val message: String) : ApiError()         // 400
    data class Unauthorized(override val message: String) : ApiError()      // 401
    data class Forbidden(override val message: String) : ApiError()         // 403
    data class NotFound(override val message: String) : ApiError()          // 404
    data class Conflict(override val message: String) : ApiError()          // 409
    data class UnprocessableEntity(val errors: Map<String, List<String>>) : ApiError() // 422
    data class TooManyRequests(val retryAfter: Int?) : ApiError()            // 429
    data class ServerError(val code: Int) : ApiError()                      // 5xx
}

// ✅ Map HttpException to ApiError
fun HttpException.toApiError(): ApiError {
    return when (code()) {
        400 -> ApiError.BadRequest(parseErrorMessage())
        401 -> ApiError.Unauthorized(parseErrorMessage())
        403 -> ApiError.Forbidden(parseErrorMessage())
        404 -> ApiError.NotFound(parseErrorMessage())
        409 -> ApiError.Conflict(parseErrorMessage())
        422 -> ApiError.UnprocessableEntity(parseValidationErrors())
        429 -> ApiError.TooManyRequests(response()?.headers()?.get("Retry-After")?.toIntOrNull())
        in 500..599 -> ApiError.ServerError(code())
        else -> ApiError.ServerError(code())
    }
}
```

---

## Pagination Patterns

```kotlin
// ✅ Offset-based pagination
@GET("users")
suspend fun getUsers(
    @Query("page") page: Int,
    @Query("limit") limit: Int = 20
): PagedResponse<UserDto>

@Serializable
data class PagedResponse<T>(
    val data: List<T>,
    val total: Int,
    val page: Int,
    val limit: Int,
    val hasMore: Boolean
)

// ✅ Cursor-based pagination
@GET("feed")
suspend fun getFeed(
    @Query("cursor") cursor: String? = null,
    @Query("limit") limit: Int = 20
): CursorPagedResponse<FeedItemDto>

@Serializable
data class CursorPagedResponse<T>(
    val data: List<T>,
    val nextCursor: String?,
    val hasMore: Boolean
)
```

---

## Standard Response Envelope

```kotlin
// ✅ Consistent API envelope — if the API uses one
@Serializable
data class ApiResponse<T>(
    val success: Boolean,
    val data: T? = null,
    val error: ApiErrorDto? = null,
    val message: String? = null
)

@Serializable
data class ApiErrorDto(
    val code: String,
    val message: String,
    val details: Map<String, List<String>> = emptyMap()
)
```

---

## Query Parameter Conventions

```kotlin
// ✅ Common query parameter patterns
@GET("users")
suspend fun getUsers(
    @Query("q") search: String? = null,         // search
    @Query("sort") sort: String? = "created_at", // sort field
    @Query("order") order: String? = "desc",     // sort direction
    @Query("filter[status]") status: String? = null,  // filtering
    @Query("page") page: Int = 1,
    @Query("limit") limit: Int = 20
): PagedResponse<UserDto>
```

---

## Anti-Patterns

- Using `GET` for operations with side effects — use `POST` or `DELETE`
- Ignoring HTTP status codes — check response codes, don't assume 200
- Putting sensitive data in query parameters — use request body or headers
- Using `POST` for reads — breaks caching and idempotency
- Returning `200 OK` for errors with error details in the body — use correct status codes

---

## Related Skills
- `retrofit` — implementing REST calls with Retrofit
- `api-contract` — formal API contract definition
- `authentication` — auth headers and token handling
- `retry-backoff` — safe retry for idempotent requests
- `dto-mapping` — mapping REST responses to domain models
