---
name: error-mapping
description: >
  Error mapping between layers in Android Clean Architecture.
  Load this skill when converting HTTP exceptions to domain errors,
  mapping Room/SQLite exceptions to domain errors, translating domain
  errors to UI messages, or designing the error translation pipeline.
---

# Error Mapping

## Overview
Error mapping converts layer-specific errors (HTTP status codes, Room exceptions, IO errors) into domain errors at each architectural boundary. The domain layer defines a typed error hierarchy; the data layer maps raw exceptions to it; the presentation layer maps domain errors to user-facing messages.

---

## Core Principles

- **Map at boundaries** — data layer maps to domain errors; ViewModel maps to messages
- **Domain errors know nothing** about HTTP, Room, or Retrofit
- **Exhaustive mapping** — every possible error type has an explicit mapping
- **Preserve context** — include enough info in domain errors to display meaningful messages
- **One mapper per data source** — separate HTTP mapper, Room mapper, etc.

---

## Domain Error Model

```kotlin
// ✅ Domain errors — pure Kotlin, no framework imports
// (See domain-error-model skill for full hierarchy)
sealed interface AppError {
    // Network
    sealed interface Network : AppError {
        data object NoConnection : Network
        data object Timeout : Network
        data class ServerError(val code: Int) : Network
        data object Unauthorized : Network
        data object Forbidden : Network
        data object NotFound : Network
    }
    // Data
    sealed interface Data : AppError {
        data object NotFound : Data
        data class Conflict(val field: String) : Data
        data class Validation(val errors: Map<String, String>) : Data
    }
    // Local storage
    sealed interface Storage : AppError {
        data object DiskFull : Storage
        data object Corrupted : Storage
        data class Unknown(val cause: Throwable) : Storage
    }
    // Unknown
    data class Unexpected(val cause: Throwable) : AppError
}
```

---

## HTTP Error Mapper

```kotlin
// ✅ Map Retrofit/HTTP exceptions to domain errors
object HttpErrorMapper {

    fun map(throwable: Throwable): AppError = when (throwable) {
        is HttpException -> mapHttpException(throwable)
        is IOException   -> mapIoException(throwable)
        is SSLException  -> AppError.Network.NoConnection
        else             -> AppError.Unexpected(throwable)
    }

    private fun mapHttpException(e: HttpException): AppError = when (e.code()) {
        400  -> parseValidationError(e) ?: AppError.Unexpected(e)
        401  -> AppError.Network.Unauthorized
        403  -> AppError.Network.Forbidden
        404  -> AppError.Network.NotFound
        409  -> parseConflictError(e) ?: AppError.Data.Conflict("unknown")
        in 500..599 -> AppError.Network.ServerError(e.code())
        else -> AppError.Unexpected(e)
    }

    private fun mapIoException(e: IOException): AppError = when (e) {
        is SocketTimeoutException,
        is ConnectTimeoutException -> AppError.Network.Timeout
        else                       -> AppError.Network.NoConnection
    }

    private fun parseValidationError(e: HttpException): AppError? = try {
        val body = e.response()?.errorBody()?.string() ?: return null
        val errors = Json.decodeFromString<ValidationErrorDto>(body)
        AppError.Data.Validation(errors.fields)
    } catch (parseException: Exception) {
        null
    }

    private fun parseConflictError(e: HttpException): AppError? = try {
        val body = e.response()?.errorBody()?.string() ?: return null
        val error = Json.decodeFromString<ConflictErrorDto>(body)
        AppError.Data.Conflict(error.field)
    } catch (parseException: Exception) {
        null
    }
}
```

---

## Room Error Mapper

```kotlin
// ✅ Map Room/SQLite exceptions to domain errors
object RoomErrorMapper {

    fun map(throwable: Throwable): AppError = when (throwable) {
        is SQLiteFullException        -> AppError.Storage.DiskFull
        is SQLiteDatabaseCorruptException -> AppError.Storage.Corrupted
        is SQLiteConstraintException  -> parseConstraintError(throwable)
        else                          -> AppError.Storage.Unknown(throwable)
    }

    private fun parseConstraintError(e: SQLiteConstraintException): AppError {
        val message = e.message ?: ""
        return when {
            message.contains("UNIQUE") -> AppError.Data.Conflict(
                extractColumnName(message)
            )
            else -> AppError.Storage.Unknown(e)
        }
    }

    private fun extractColumnName(message: String): String {
        // "UNIQUE constraint failed: users.email" → "email"
        return message.substringAfterLast(".").takeIf { it.isNotBlank() } ?: "unknown"
    }
}
```

---

## Repository Using Mappers

```kotlin
// ✅ Repository applies mappers at the boundary
class UserRepositoryImpl @Inject constructor(
    private val api: UserApiService,
    private val dao: UserDao
) : UserRepository {

    override suspend fun getUser(id: String): Result<User> =
        runCatching {
            dao.getById(id)?.toDomain()
                ?: api.getUser(id).toDomain().also { dao.insert(it.toEntity()) }
        }.mapFailure { throwable ->
            when (throwable) {
                is HttpException,
                is IOException -> HttpErrorMapper.map(throwable)
                is SQLiteException -> RoomErrorMapper.map(throwable)
                else -> AppError.Unexpected(throwable)
            }
        }

    override suspend fun createUser(user: User): Result<User> =
        runCatching {
            api.createUser(user.toCreateRequest()).toDomain()
        }.mapFailure { HttpErrorMapper.map(it) }
}

// ✅ Extension to map the failure in a Result
fun <T> Result<T>.mapFailure(transform: (Throwable) -> Throwable): Result<T> =
    fold(
        onSuccess = { Result.success(it) },
        onFailure = { Result.failure(transform(it)) }
    )

// Or use domain error as sealed class (not Throwable)
fun <T> Result<T>.mapError(transform: (Throwable) -> AppError): Result<T> =
    recoverCatching { throw AppErrorException(transform(it)) }

class AppErrorException(val error: AppError) : Exception()
```

---

## ViewModel → UI Message Mapping

```kotlin
// ✅ ViewModel maps domain errors to UI strings
fun AppError.toUiMessage(context: Context): String = when (this) {
    is AppError.Network.NoConnection  -> context.getString(R.string.error_no_connection)
    is AppError.Network.Timeout       -> context.getString(R.string.error_timeout)
    is AppError.Network.Unauthorized  -> context.getString(R.string.error_session_expired)
    is AppError.Network.Forbidden     -> context.getString(R.string.error_no_permission)
    is AppError.Network.NotFound      -> context.getString(R.string.error_not_found)
    is AppError.Network.ServerError   -> context.getString(R.string.error_server, code)
    is AppError.Data.NotFound         -> context.getString(R.string.error_item_not_found)
    is AppError.Data.Conflict         -> context.getString(R.string.error_conflict, field)
    is AppError.Data.Validation       -> errors.values.first()
    is AppError.Storage.DiskFull      -> context.getString(R.string.error_disk_full)
    is AppError.Storage.Corrupted     -> context.getString(R.string.error_data_corrupted)
    is AppError.Storage.Unknown,
    is AppError.Unexpected            -> context.getString(R.string.error_unexpected)
}

// ✅ Or without Context — use string keys resolved in Compose
fun AppError.toMessageKey(): Int = when (this) {
    is AppError.Network.NoConnection -> R.string.error_no_connection
    is AppError.Network.Timeout      -> R.string.error_timeout
    // ...
    else                             -> R.string.error_unexpected
}
```

---

## Anti-Patterns

- Mapping HTTP codes in the ViewModel — belongs in the data layer mapper
- Domain error types that import `retrofit2` or `androidx.room` — domain must stay pure
- Catch-all `else -> AppError.Unexpected` without logging — unexpected errors should be tracked
- Showing raw exception messages to users — `e.message` is for developers, not users
- One giant `when(throwable)` across all layers — each layer has its own mapper

---

## Related Skills
- `error-handling` — overall error propagation strategy
- `domain-error-model` — the domain error sealed class hierarchy
- `failure-strategy` — how to respond to each error type
- `user-friendly-errors` — composable/string resource error display
- `repository-pattern` — where mappers are applied
