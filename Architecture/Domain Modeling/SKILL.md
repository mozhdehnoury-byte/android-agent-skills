---
name: domain-modeling
description: >
  Domain modeling for Android Clean Architecture projects.
  Load this skill when designing domain entities, defining business rules
  in the domain layer, modeling relationships between domain objects,
  choosing between Entity vs Value Object, or structuring the domain package.
---

# Domain Modeling

## Overview

Domain modeling defines the core business concepts of the application as pure Kotlin classes — free of Android framework, database annotations, or network serialization. The domain model is the language of the business, shared across all layers via mapping.

---

## Core Principles

- Domain models are **pure Kotlin** — no `@Entity`, `@SerialName`, `@Parcelize`
- Domain models express **business concepts** — not database tables or API contracts
- **Behavior belongs in domain** — validation, business rules, computations live here
- **Immutable** — use `val`, use `data class`, use `copy()` for changes
- Domain models are the **shared language** — same names used in UI, DB, API (via mapping)

---

## Entity vs Value Object

|           | Entity                   | Value Object               |
| --------- | ------------------------ | -------------------------- |
| Identity  | Has unique `id`          | Defined by its values      |
| Equality  | By `id`                  | By all fields              |
| Lifecycle | Persisted, tracked       | Transient, replaceable     |
| Example   | `User(id=1, name="Ali")` | `Email("ali@example.com")` |

---

## Domain Entity

```kotlin
// ✅ Entity — has identity, represents a business object
data class User(
    val id: String,
    val name: String,
    val email: Email,           // ✅ use Value Object for validated fields
    val role: UserRole,
    val status: UserStatus,
    val createdAt: Instant
) {
    // ✅ Business rules as functions — not in ViewModel
    fun canEdit(actor: User): Boolean =
        actor.id == id || actor.role == UserRole.ADMIN

    fun isActive(): Boolean = status == UserStatus.ACTIVE

    fun withRole(newRole: UserRole): User = copy(role = newRole)
}

data class Order(
    val id: String,
    val customerId: String,
    val items: List<OrderItem>,
    val status: OrderStatus,
    val placedAt: Instant
) {
    val totalAmount: Money
        get() = items.fold(Money.ZERO) { acc, item -> acc + item.subtotal }

    fun canBeCancelled(): Boolean =
        status == OrderStatus.PENDING || status == OrderStatus.PROCESSING

    fun withStatus(newStatus: OrderStatus): Order = copy(status = newStatus)
}
```

---

## Value Object

```kotlin
// ✅ Value Object — validated, no identity
@JvmInline
value class Email(val value: String) {
    init {
        require(value.contains("@")) { "Invalid email: $value" }
        require(value.length <= 254) { "Email too long" }
    }
}

@JvmInline
value class UserId(val value: String) {
    init { require(value.isNotBlank()) { "UserId cannot be blank" } }
}

data class Money(
    val amount: Long,           // stored in smallest unit (cents/rials)
    val currency: Currency
) {
    operator fun plus(other: Money): Money {
        require(currency == other.currency) { "Currency mismatch" }
        return copy(amount = amount + other.amount)
    }

    fun isPositive(): Boolean = amount > 0

    companion object {
        val ZERO = Money(0, Currency.IRR)
    }
}

data class DateRange(
    val start: LocalDate,
    val end: LocalDate
) {
    init { require(!end.isBefore(start)) { "End must be after start" } }

    fun contains(date: LocalDate): Boolean = !date.isBefore(start) && !date.isAfter(end)
    fun durationDays(): Long = ChronoUnit.DAYS.between(start, end)
}
```

---

## Enums and Sealed Classes

```kotlin
// ✅ Simple states — enum
enum class UserRole { ADMIN, MEMBER, GUEST }
enum class UserStatus { ACTIVE, SUSPENDED, DELETED }
enum class OrderStatus { PENDING, PROCESSING, SHIPPED, DELIVERED, CANCELLED }

// ✅ States with data — sealed class
sealed interface PaymentResult {
    data class Success(val transactionId: String, val amount: Money) : PaymentResult
    data class Failed(val reason: String, val code: Int) : PaymentResult
    data object Cancelled : PaymentResult
}

sealed interface ValidationResult {
    data object Valid : ValidationResult
    data class Invalid(val errors: List<String>) : ValidationResult

    fun isValid(): Boolean = this is Valid
}
```

---

## Aggregate Design

```kotlin
// ✅ Aggregate root — owns its children, enforces consistency
data class Cart(
    val id: String,
    val customerId: String,
    val items: List<CartItem> = emptyList()
) {
    val totalAmount: Money
        get() = items.fold(Money.ZERO) { acc, item -> acc + item.subtotal }

    val itemCount: Int get() = items.sumOf { it.quantity }

    fun addItem(product: Product, quantity: Int): Cart {
        require(quantity > 0) { "Quantity must be positive" }
        val existing = items.find { it.productId == product.id }
        return if (existing != null) {
            copy(items = items.map {
                if (it.productId == product.id)
                    it.copy(quantity = it.quantity + quantity)
                else it
            })
        } else {
            copy(items = items + CartItem(product.id, product.name, product.price, quantity))
        }
    }

    fun removeItem(productId: String): Cart =
        copy(items = items.filter { it.productId != productId })

    fun clear(): Cart = copy(items = emptyList())
}

data class CartItem(
    val productId: String,
    val productName: String,
    val unitPrice: Money,
    val quantity: Int
) {
    val subtotal: Money get() = Money(unitPrice.amount * quantity, unitPrice.currency)
}
```

---

## Domain Package Structure

```
domain/
├── model/
│   ├── User.kt
│   ├── Order.kt
│   ├── Cart.kt
│   └── Product.kt
├── value/
│   ├── Email.kt
│   ├── Money.kt
│   ├── DateRange.kt
│   └── UserId.kt
├── repository/
│   ├── UserRepository.kt
│   └── OrderRepository.kt
└── usecase/
    ├── GetUserUseCase.kt
    └── PlaceOrderUseCase.kt
```

---

## Anti-Patterns

- Domain models with `@Entity` or `@SerialName` — domain must not know about persistence or network
- Anemic domain model — models with only `data class` fields and no behavior; business rules scattered in ViewModel
- Using `String` for everything — wrap validated concepts in Value Objects (`Email`, `UserId`)
- Mutable domain models (`var` fields) — always `val` + `copy()`
- Domain model inheriting from framework class — domain is pure Kotlin only

---

## Related Skills

- `entity-design` — designing Room entities that map to domain models
- `value-object` — in-depth Value Object patterns
- `clean-architecture` — where domain sits in the layer structure
- `dto-mapping` — mapping between domain and data layer models
- `use-case-design` — business operations that act on domain models
