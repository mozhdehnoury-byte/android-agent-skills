---
name: serialization
description: >
  Kotlinx Serialization setup and usage for Android and KMP projects.
  Load this skill when serializing/deserializing JSON or other formats,
  configuring the Json instance, mapping API responses, or handling
  custom serialization logic.
---

# Serialization

## Overview

Kotlinx Serialization is the standard serialization library for Kotlin/Android. It is compile-time safe, KMP-compatible, and integrates natively with Retrofit, Ktor, and DataStore.

---

## Core Principles

- Use `@Serializable` on all data transfer objects (DTOs)
- Never use Gson or Moshi — use Kotlinx Serialization
- Configure a **single shared `Json` instance** per module — never create inline
- Domain models must **not** be `@Serializable` — only DTOs
- Use `@SerialName` when API field names differ from Kotlin naming conventions

---

## Setup

```toml
# libs.versions.toml
[versions]
kotlinx-serialization = "1.7.1"

[libraries]
kotlinx-serialization-json = { module = "org.jetbrains.kotlinx:kotlinx-serialization-json", version.ref = "kotlinx-serialization" }

[plugins]
kotlin-serialization = { id = "org.jetbrains.kotlin.plugin.serialization", version.ref = "kotlin" }
```

```kotlin
// build.gradle.kts
plugins {
    alias(libs.plugins.kotlin.serialization)
}

dependencies {
    implementation(libs.kotlinx.serialization.json)
}
```

---

## Json Instance Configuration

```kotlin
// ✅ Single shared instance — define once, inject everywhere
val json = Json {
    ignoreUnknownKeys = true      // safe for API evolution
    isLenient = false             // strict parsing by default
    encodeDefaults = false        // don't serialize default values
    prettyPrint = false           // compact output in production
    coerceInputValues = true      // handle null → default for non-nullable
}
```

---

## DTO Design

```kotlin
// ✅ DTOs are @Serializable — domain models are not
@Serializable
data class UserDto(
    @SerialName("user_id") val id: String,
    @SerialName("full_name") val name: String,
    @SerialName("email_address") val email: String,
    @SerialName("is_active") val isActive: Boolean = true,
    @SerialName("created_at") val createdAt: Long? = null
)

// ✅ Domain model — clean, no serialization annotations
data class User(
    val id: String,
    val name: String,
    val email: String,
    val isActive: Boolean,
    val createdAt: Long?
)
```

---

## Custom Serializers

```kotlin
// ✅ Use custom serializer for types that can't be annotated
object UUIDSerializer : KSerializer<UUID> {
    override val descriptor = PrimitiveSerialDescriptor("UUID", PrimitiveKind.STRING)
    override fun serialize(encoder: Encoder, value: UUID) = encoder.encodeString(value.toString())
    override fun deserialize(decoder: Decoder): UUID = UUID.fromString(decoder.decodeString())
}

// Apply on property
@Serializable
data class OrderDto(
    @Serializable(with = UUIDSerializer::class)
    val id: UUID
)
```

---

## Sealed Class Serialization

```kotlin
// ✅ Use @JsonClassDiscriminator for polymorphic types
@Serializable
@JsonClassDiscriminator("type")
sealed class EventDto {
    @Serializable
    @SerialName("click")
    data class Click(val x: Int, val y: Int) : EventDto()

    @Serializable
    @SerialName("scroll")
    data class Scroll(val delta: Int) : EventDto()
}
```

---

## Integration with Retrofit

```kotlin
// ✅ Use kotlinx-serialization converter — not Gson
dependencies {
    implementation(libs.retrofit.kotlinx.serialization)
}

val retrofit = Retrofit.Builder()
    .baseUrl(baseUrl)
    .addConverterFactory(json.asConverterFactory("application/json".toMediaType()))
    .build()
```

## Integration with Ktor

```kotlin
// ✅ Use ContentNegotiation plugin
install(ContentNegotiation) {
    json(json) // pass the shared Json instance
}
```

---

## Handling Unknown/Dynamic Fields

```kotlin
// ✅ Use JsonObject for truly dynamic content
@Serializable
data class WebhookDto(
    val event: String,
    val payload: JsonObject  // dynamic structure
)

// Extract typed value from JsonObject
val userId = webhook.payload["user_id"]?.jsonPrimitive?.content
```

---

## Anti-Patterns

- Using Gson or Moshi — not KMP-compatible, less type-safe
- `@Serializable` on domain models — couples domain to transport layer
- Creating `Json {}` inline at call site — inconsistent configuration
- Missing `@SerialName` when API uses snake_case — breaks deserialization
- Using `Any` or `Map<String, Any>` — loses type safety; use `JsonObject`
- Ignoring `ignoreUnknownKeys = true` — crashes on API evolution

---

## Related Skills

- `dto-mapping` — mapping between DTOs and domain models
- `retrofit` — Retrofit setup with serialization converter
- `ktor` — Ktor client setup with serialization plugin
- `kmp` — shared serialization setup in multiplatform projects
