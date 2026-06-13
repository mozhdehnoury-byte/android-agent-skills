---
name: value-object
description: >
  Value Object pattern for Android domain modeling.
  Load this skill when wrapping primitive types with business meaning,
  implementing validation at construction time, designing immutable value types,
  using Kotlin inline/value classes, or eliminating primitive obsession.
---

# Value Object

## Overview

A Value Object is an immutable object defined entirely by its values, not by identity. Two Value Objects with the same values are equal. They encapsulate validation and domain rules at the point of creation, eliminating invalid states and primitive obsession across the codebase.

---

## Core Principles

- **No identity** — equality is based on values, not `id`
- **Immutable** — all fields are `val`; produce new instances instead of mutating
- **Self-validating** — invalid values cannot be constructed; validation in `init`
- **Behavior-rich** — operations and rules belong here, not in the caller
- **Replaces primitives** — prefer `Email` over `String`, `Money` over `Long`

---

## Kotlin Value Class (Inline Class)

```kotlin
// ✅ Single-field value object — use @JvmInline value class (zero overhead)
@JvmInline
value class Email(val value: String) {
    init {
        require(value.isNotBlank()) { "Email cannot be blank" }
        require(value.contains("@")) { "Invalid email format: $value" }
        require(value.length <= 254) { "Email exceeds maximum length" }
    }

    val domain: String get() = value.substringAfter("@")
    val local: String get() = value.substringBefore("@")
}

@JvmInline
value class UserId(val value: String) {
    init { require(value.isNotBlank()) { "UserId cannot be blank" } }
}

@JvmInline
value class PhoneNumber(val value: String) {
    init {
        val digits = value.filter { it.isDigit() }
        require(digits.length in 10..15) { "Invalid phone number: $value" }
    }

    val normalized: String get() = value.filter { it.isDigit() }
}

@JvmInline
value class Percentage(val value: Double) {
    init {
        require(value in 0.0..100.0) { "Percentage must be between 0 and 100: $value" }
    }

    fun toFraction(): Double = value / 100.0
}
```

---

## Multi-Field Value Object

```kotlin
// ✅ Multi-field value object — regular data class
data class Money(
    val amount: Long,         // smallest unit (e.g. cents, rials)
    val currency: Currency
) {
    init {
        require(amount >= 0) { "Money amount cannot be negative: $amount" }
    }

    operator fun plus(other: Money): Money {
        require(currency == other.currency) { "Cannot add different currencies" }
        return copy(amount = amount + other.amount)
    }

    operator fun minus(other: Money): Money {
        require(currency == other.currency) { "Cannot subtract different currencies" }
        require(amount >= other.amount) { "Cannot subtract: result would be negative" }
        return copy(amount = amount - other.amount)
    }

    operator fun times(factor: Int): Money = copy(amount = amount * factor)

    fun isZero(): Boolean = amount == 0L
    fun isPositive(): Boolean = amount > 0L

    companion object {
        fun zero(currency: Currency) = Money(0L, currency)
        fun ofRials(amount: Long) = Money(amount, Currency.IRR)
    }
}

data class Address(
    val street: String,
    val city: String,
    val postalCode: PostalCode,
    val country: Country
) {
    init {
        require(street.isNotBlank()) { "Street cannot be blank" }
        require(city.isNotBlank()) { "City cannot be blank" }
    }

    fun formatted(): String = "$street, $city, ${postalCode.value}, ${country.displayName}"
}

data class DateRange(
    val start: LocalDate,
    val end: LocalDate
) {
    init {
        require(!end.isBefore(start)) { "End date must not be before start date" }
    }

    fun contains(date: LocalDate): Boolean =
        !date.isBefore(start) && !date.isAfter(end)

    fun overlaps(other: DateRange): Boolean =
        !start.isAfter(other.end) && !end.isBefore(other.start)

    fun durationDays(): Long = ChronoUnit.DAYS.between(start, end)
}
```

---

## Safe Construction with Result

```kotlin
// ✅ When validation failure should be handled gracefully (not crash)
@JvmInline
value class Username private constructor(val value: String) {
    companion object {
        fun create(value: String): Result<Username> = runCatching {
            require(value.length in 3..20) { "Username must be 3-20 characters" }
            require(value.all { it.isLetterOrDigit() || it == '_' }) {
                "Username can only contain letters, digits, and underscores"
            }
            Username(value)
        }
    }
}

// Usage
val result = Username.create(input)
result.fold(
    onSuccess = { username -> proceed(username) },
    onFailure = { error -> showValidationError(error.message) }
)
```

---

## Storage Mapping

```kotlin
// ✅ TypeConverter for Value Objects in Room
class ValueObjectConverters {

    @TypeConverter
    fun fromEmail(email: Email?): String? = email?.value

    @TypeConverter
    fun toEmail(value: String?): Email? = value?.let { Email(it) }

    @TypeConverter
    fun fromMoney(money: Money?): String? =
        money?.let { "${it.amount}:${it.currency.name}" }

    @TypeConverter
    fun toMoney(value: String?): Money? = value?.let {
        val (amount, currency) = it.split(":")
        Money(amount.toLong(), Currency.valueOf(currency))
    }
}
```

---

## Serialization Mapping

```kotlin
// ✅ Never serialize Value Objects directly — map at the boundary
data class UserDto(
    @SerialName("id") val id: String,
    @SerialName("email") val email: String,    // raw String in DTO
)

// ✅ Map to domain Value Object in mapper
fun UserDto.toDomain(): User = User(
    id = UserId(id),
    email = Email(email),                       // construct Value Object here
    ...
)
```

---

## Anti-Patterns

- Passing raw `String` for email, phone, ID across the codebase — use Value Objects
- Validating the same constraint in multiple places — validation belongs in the Value Object constructor
- Mutable Value Objects (`var` fields) — Value Objects must be immutable
- Using `@JvmInline value class` for multi-field objects — value classes only support one property
- Throwing generic `IllegalArgumentException` without a message — always include context in the message

---

## Related Skills

- `domain-modeling` — where Value Objects are used within the domain
- `entity-design` — mapping Value Objects to database columns
- `dto-mapping` — mapping between raw DTO strings and Value Objects
- `immutability` — immutability principles in Kotlin
