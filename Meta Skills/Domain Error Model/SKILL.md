---
name: domain-error-model
description: >
  Domain error hierarchy design for Android Clean Architecture.
  Load this skill when designing the sealed class error hierarchy,
  deciding how granular errors should be, structuring errors by domain
  context, or defining the contract between data and domain layers.
---

# Domain Error Model

## Overview
The domain error model is a sealed class hierarchy that represents all possible failure states in the application, using the language of the business — not HTTP codes or SQL exceptions. It lives in the domain layer and is the single contract between data (which produces errors) and presentation (which displays them).

---

## Core Principles

- **Pure Kotlin** — no Android, Retrofit, or Room imports
- **Sealed hierarchy** — exhaustive, forces handling at every call site
- **Business language** — `Unauthorized` not `Http401`, `NotFound` not `Http404`
- **Granularity by need** — only as specific as the UI needs to differentiate behavior
- **Stable** — rarely changes; presentation adapts to it, not the other way

---

## Full Domain Error Hierarchy

```kotlin
// domain/error/AppError.kt

sealed interface AppError {

    // ── Network errors ──────────────────────────────────────────────────
    sealed interface Network : AppError {
        /** Device has no internet connection */
        data object NoConnection : Network

        /** Request timed out */
        data object Timeout : Network

        /** Server returned 5xx */
        data class ServerError(val code: Int) : Network

        /** 401 — session expired or token invalid */
        data object Unauthorized : Network

        /** 403 — user lacks permission */
        data object Forbidden : Network

        /** 404 — resource does not exist on server */
        data object NotFound : Network

        /** 429 — rate limited */
        data object RateLimited : Network
    }

    // ── Data / business errors ───────────────────────────────────────────
    sealed interface Data : AppError {
        /** Requested entity does not exist locally */
        data object NotFound : Data

        /** Operation would violate a uniqueness constraint */
        data class Conflict(val field: String) : Data

        /** Input failed business validation */
        data class Validation(val errors: Map<String, String>) : Data

        /** Operation not allowed in current state */
        data class InvalidState(val reason: String) : Data
    }

    // ── Storage errors ───────────────────────────────────────────────────
    sealed interface Storage : AppError {
        /** Device storage is full */
        data object DiskFull : Storage

        /** Local database or file is corrupted */
        data object Corrupted : Storage

        /** Unclassified storage error */
        data class Unknown(val cause: Throwable) : Storage
    }

    // ── Auth / security errors ───────────────────────────────────────────
    sealed interface Auth : AppError {
        /** User is not authenticated */
        data object NotAuthenticated : Auth

        /** Biometric / PIN verification failed */
        data object BiometricFailed : Auth

        /** Account is suspended or locked */
        data object AccountSuspended : Auth
    }

    // ── Unexpected / unclassified ────────────────────────────────────────
    data class Unexpected(val cause: Throwable) : AppError
}
```

---

## Wrapping as Exception (for Result interop)

```kotlin
// ✅ Wrap AppError in an exception so it can travel through Result<T>
class AppException(val error: AppError) : Exception(error.toString())

// Extension to lift AppError into Result failure
fun <T> Result.Companion.domainFailure(error: AppError): Result<T> =
    failure(AppException(error))

// Extension to extract AppError from Result failure
fun <T> Result<T>.appErrorOrNull(): AppError? =
    exceptionOrNull()?.let { if (it is AppException) it.error else null }

// ✅ Usage in UseCase
class GetUserUseCase @Inject constructor(
    private val repository: UserRepository
) {
    suspend operator fun invoke(id: String): Result<User> =
        repository.getUser(id).fold(
            onSuccess = { user ->
                if (user.status == UserStatus.DELETED)
                    Result.domainFailure(AppError.Data.InvalidState("User is deleted"))
                else
                    Result.success(user)
            },
            onFailure = { Result.failure(it) }
        )
}
```

---

## Per-Feature Error Scoping

```kotlin
// ✅ Feature-specific errors extend AppError — keeps domain focused
sealed interface OrderError : AppError {
    data object CartEmpty : OrderError
    data class InsufficientStock(val productName: String, val available: Int) : OrderError
    data object PaymentDeclined : OrderError
    data class DeliveryUnavailable(val reason: String) : OrderError
}

// ✅ Usage
class PlaceOrderUseCase @Inject constructor(
    private val orderRepository: OrderRepository,
    private val inventoryRepository: InventoryRepository
) {
    suspend operator fun invoke(cartId: String): Result<Order> = runCatching {
        val cart = orderRepository.getCart(cartId).getOrThrow()

        if (cart.items.isEmpty())
            throw AppException(OrderError.CartEmpty)

        cart.items.forEach { item ->
            val stock = inventoryRepository.getStock(item.productId).getOrThrow()
            if (stock < item.quantity)
                throw AppException(OrderError.InsufficientStock(item.name, stock))
        }

        orderRepository.placeOrder(cart).getOrThrow()
    }
}
```

---

## Handling in ViewModel

```kotlin
// ✅ Exhaustive when expression forces handling every error
private fun handleError(error: AppError): String = when (error) {
    is AppError.Network.NoConnection  -> "No internet connection"
    is AppError.Network.Timeout       -> "Request timed out"
    is AppError.Network.Unauthorized  -> "Session expired"
    is AppError.Network.Forbidden     -> "Permission denied"
    is AppError.Network.NotFound      -> "Not found"
    is AppError.Network.ServerError   -> "Server error (${error.code})"
    is AppError.Network.RateLimited   -> "Too many requests"
    is AppError.Data.NotFound         -> "Item not found"
    is AppError.Data.Conflict         -> "Already exists: ${error.field}"
    is AppError.Data.Validation       -> error.errors.values.first()
    is AppError.Data.InvalidState     -> error.reason
    is AppError.Storage.DiskFull      -> "Storage full"
    is AppError.Storage.Corrupted     -> "Data corrupted"
    is AppError.Storage.Unknown       -> "Storage error"
    is AppError.Auth.NotAuthenticated -> "Please log in"
    is AppError.Auth.BiometricFailed  -> "Authentication failed"
    is AppError.Auth.AccountSuspended -> "Account suspended"
    is AppError.Unexpected            -> "Something went wrong"
    // Feature errors
    is OrderError.CartEmpty           -> "Your cart is empty"
    is OrderError.InsufficientStock   -> "Not enough stock for ${error.productName}"
    is OrderError.PaymentDeclined     -> "Payment declined"
    is OrderError.DeliveryUnavailable -> error.reason
}
```

---

## Retry Decision

```kotlin
// ✅ Determine if an error is retryable
fun AppError.isRetryable(): Boolean = when (this) {
    is AppError.Network.NoConnection,
    is AppError.Network.Timeout,
    is AppError.Network.ServerError  -> true
    is AppError.Network.RateLimited  -> true
    else                             -> false
}

// ✅ Determine if error requires re-authentication
fun AppError.requiresReAuth(): Boolean =
    this is AppError.Network.Unauthorized || this is AppError.Auth.NotAuthenticated
```

---

## Anti-Patterns

- `AppError.Unexpected` as the only error type — no granularity; can't make smart UI decisions
- HTTP codes in domain error names (`Http404`, `Http401`) — domain must speak business language
- Domain error importing `retrofit2.HttpException` — breaks domain layer independence
- Flat error list instead of sealed hierarchy — no compile-time exhaustiveness
- Too many error subtypes nobody handles differently — YAGNI; add when UI needs to differentiate

---

## Related Skills
- `error-handling` — propagation strategy across layers
- `error-mapping` — converting raw exceptions to domain errors
- `failure-strategy` — deciding how to respond to each error
- `user-friendly-errors` — displaying errors in UI
