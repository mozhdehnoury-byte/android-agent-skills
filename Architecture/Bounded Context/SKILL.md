---
name: bounded-context
description: >
  Bounded Context pattern for Android feature organization.
  Load this skill when defining feature boundaries, deciding what belongs
  in a shared domain vs a feature-specific domain, preventing model pollution
  across features, or designing the conceptual boundaries of a module.
---

# Bounded Context

## Overview

A Bounded Context defines the boundary within which a domain model is consistent and valid. In Android, each feature module (or major feature area) is a Bounded Context — it owns its own models, rules, and language. The same real-world concept (e.g. "User") may exist in multiple contexts with different shapes and responsibilities.

---

## Core Principles

- Each context owns its **own model** — never share a model across contexts if they evolve differently
- **Context map** makes inter-context relationships explicit
- Cross-context communication happens through **well-defined interfaces** — not direct model sharing
- The same word can mean different things in different contexts — that's intentional
- Shared kernel only for concepts that are **truly identical** across contexts

---

## Example: "User" in Multiple Contexts

```kotlin
// ❌ Tempting but wrong — one "User" class used everywhere
data class User(
    val id: String,
    val name: String,
    val email: String,
    val role: UserRole,
    val shippingAddresses: List<Address>,    // only needed in Orders context
    val orderHistory: List<OrderSummary>,    // only needed in Orders context
    val savedCards: List<PaymentCard>,       // only needed in Payment context
    val notificationPreferences: NotificationPrefs // only needed in Notifications
)
// This model becomes a dumping ground — bloated and hard to maintain

// ✅ Correct — each context defines its own model
// Auth context
data class AuthUser(
    val id: String,
    val email: String,
    val role: UserRole,
    val sessionToken: String
)

// Orders context
data class OrderCustomer(
    val id: String,
    val displayName: String,
    val defaultShippingAddress: Address?
)

// Profile context
data class UserProfile(
    val id: String,
    val fullName: String,
    val avatarUrl: String?,
    val bio: String?
)

// Notifications context
data class NotificationRecipient(
    val userId: String,
    val pushToken: String?,
    val preferences: NotificationPreferences
)
```

---

## Context Map

```kotlin
// ✅ Explicit translation between contexts
// When Orders needs user info, it requests it through an interface

// Orders context defines what it needs
interface CustomerInfoProvider {
    suspend fun getCustomer(userId: String): Result<OrderCustomer>
}

// The implementation in :app or a shared module translates
class CustomerInfoProviderImpl @Inject constructor(
    private val userRepository: UserRepository  // Auth/Profile context
) : CustomerInfoProvider {

    override suspend fun getCustomer(userId: String): Result<OrderCustomer> =
        userRepository.getUser(userId).map { user ->
            OrderCustomer(
                id = user.id,
                displayName = user.name,
                defaultShippingAddress = user.addresses.firstOrNull { it.isDefault }
            )
        }
}
```

---

## Feature Module as Bounded Context

```kotlin
// ✅ :feature:orders owns its domain model
// feature/orders/domain/model/
data class Order(val id: String, val customerId: String, val items: List<OrderItem>, ...)
data class OrderItem(val productId: String, val name: String, val quantity: Int, ...)
data class OrderSummary(val id: String, val status: OrderStatus, val totalAmount: Money)

// ✅ :feature:catalog owns its domain model — Product means something different here
// feature/catalog/domain/model/
data class Product(val id: String, val name: String, val description: String, val price: Money, ...)
data class Category(val id: String, val name: String, val products: List<ProductSummary>)

// ✅ :feature:cart owns its domain model — Product in cart context = CartItem
// feature/cart/domain/model/
data class Cart(val id: String, val items: List<CartItem>)
data class CartItem(val productId: String, val productName: String, val unitPrice: Money, val quantity: Int)
// Note: CartItem has productName denormalized — it owns what it needs
```

---

## Shared Kernel

```kotlin
// ✅ Truly shared concepts live in :core:domain
// These are stable and identical across all contexts

// :core:domain/model/shared/
@JvmInline value class UserId(val value: String)
@JvmInline value class ProductId(val value: String)
data class Money(val amount: Long, val currency: Currency)
enum class Currency { IRR, USD, EUR }

// All features import these from :core:domain — not from each other
```

---

## Anti-Corruption Layer

```kotlin
// ✅ Translate external API models without polluting domain
// When integrating a payment gateway, don't let its model bleed in

// External (Zarinpal, Stripe, etc.)
data class PaymentGatewayResponse(
    val status: Int,
    val ref_id: String?,
    val error_code: String?
)

// Anti-corruption layer — translates to domain language
class PaymentGatewayAdapter @Inject constructor(
    private val gateway: PaymentGatewayService
) {
    suspend fun processPayment(request: PaymentRequest): Result<PaymentConfirmation> =
        runCatching {
            val response = gateway.pay(request.toGatewayRequest())
            when (response.status) {
                100 -> PaymentConfirmation(
                    transactionId = response.ref_id ?: error("Missing ref_id"),
                    amount = request.amount
                )
                else -> throw PaymentException(response.error_code ?: "Unknown error")
            }
        }
}
```

---

## When to Split vs Share

| Signal                                                 | Action                                           |
| ------------------------------------------------------ | ------------------------------------------------ |
| Same concept evolves differently in two features       | Split into two models                            |
| Same concept has identical fields and rules everywhere | Put in shared kernel                             |
| Feature A needs data from Feature B                    | Define interface in A, implement in :app         |
| Two features share a DTO from the same API endpoint    | Split into feature-specific domain models anyway |
| Changing one feature's model breaks another feature    | Context boundary is violated — split             |

---

## Anti-Patterns

- One giant `User` class used by auth, profile, orders, notifications — split by context
- Feature modules importing domain models from other feature modules — route through :core or interfaces
- Shared kernel growing to include feature-specific concepts — keep shared kernel minimal and stable
- Skipping the translation layer — directly mapping API DTOs to domain models used by multiple features

---

## Related Skills

- `modularization` — module boundaries that enforce context boundaries
- `feature-isolation` — isolating feature implementation details
- `domain-modeling` — modeling within a bounded context
- `clean-architecture` — layer boundaries within a context
- `repository-pattern` — repository interfaces as context boundaries
