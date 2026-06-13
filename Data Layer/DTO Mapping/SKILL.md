---
name: dto-mapping
description: >
  DTO to domain model mapping patterns for Android.
  Load this skill when writing mappers between API responses (DTOs),
  database entities, and domain models — or when designing the data
  layer boundary in a clean architecture project.
---

# DTO Mapping

## Overview
DTO (Data Transfer Object) mapping is the translation between data representations across layer boundaries. API responses (DTOs), database rows (Entities), and domain models are separate types — mapping between them keeps each layer independent and testable.

---

## Core Principles

- **Domain models are the language of business logic** — pure Kotlin, no framework annotations
- **DTOs mirror the API contract** — annotated with `@Serializable`, snake_case fields
- **Entities mirror the DB schema** — annotated with `@Entity`, `@ColumnInfo`
- Mapping always happens at the **repository layer** — never in ViewModel or UseCase
- Mappers are **pure functions** — no side effects, no dependencies

---

## Layer Boundaries

```
Network (JSON)          DTO             @Serializable
       ↓ mapper
Domain Layer         Domain Model       Pure Kotlin data class
       ↓ mapper
Database (SQLite)       Entity          @Entity, @ColumnInfo
```

---

## Domain Model

```kotlin
// ✅ Pure Kotlin — no framework dependencies
data class User(
    val id: String,
    val name: String,
    val email: String,
    val status: UserStatus,
    val createdAt: Instant
)

enum class UserStatus { ACTIVE, INACTIVE, PENDING }
```

---

## DTO (API Response)

```kotlin
// ✅ Mirrors API contract — annotated for serialization
@Serializable
data class UserDto(
    @SerialName("user_id") val id: String,
    @SerialName("full_name") val name: String,
    @SerialName("email_address") val email: String,
    @SerialName("account_status") val status: String,
    @SerialName("created_timestamp") val createdAt: Long
)
```

---

## Entity (Database Row)

```kotlin
// ✅ Mirrors DB schema — annotated for Room
@Entity(tableName = "users")
data class UserEntity(
    @PrimaryKey val id: String,
    @ColumnInfo(name = "full_name") val name: String,
    @ColumnInfo(name = "email") val email: String,
    @ColumnInfo(name = "status") val status: String,
    @ColumnInfo(name = "created_at") val createdAt: Long
)
```

---

## Mapper — Extension Functions (Simple)

```kotlin
// ✅ Extension function mapper — concise for simple cases
fun UserDto.toDomain(): User = User(
    id = id,
    name = name,
    email = email,
    status = UserStatus.valueOf(status.uppercase()),
    createdAt = Instant.ofEpochMilli(createdAt)
)

fun UserEntity.toDomain(): User = User(
    id = id,
    name = name,
    email = email,
    status = UserStatus.valueOf(status),
    createdAt = Instant.ofEpochMilli(createdAt)
)

fun User.toEntity(): UserEntity = UserEntity(
    id = id,
    name = name,
    email = email,
    status = status.name,
    createdAt = createdAt.toEpochMilli()
)

fun User.toDto(): UserDto = UserDto(
    id = id,
    name = name,
    email = email,
    status = status.name.lowercase(),
    createdAt = createdAt.toEpochMilli()
)
```

---

## Mapper — Class-Based (Testable, Injectable)

```kotlin
// ✅ Class mapper — preferred when mapping has dependencies or is complex
class UserMapper @Inject constructor(
    private val clock: Clock = Clock.systemUTC()
) {
    fun toDomain(dto: UserDto): User = User(
        id = dto.id,
        name = dto.name.trim(),
        email = dto.email.lowercase(),
        status = mapStatus(dto.status),
        createdAt = Instant.ofEpochMilli(dto.createdAt)
    )

    fun toDomain(entity: UserEntity): User = User(
        id = entity.id,
        name = entity.name,
        email = entity.email,
        status = UserStatus.valueOf(entity.status),
        createdAt = Instant.ofEpochMilli(entity.createdAt)
    )

    fun toEntity(domain: User): UserEntity = UserEntity(
        id = domain.id,
        name = domain.name,
        email = domain.email,
        status = domain.status.name,
        createdAt = domain.createdAt.toEpochMilli()
    )

    private fun mapStatus(raw: String): UserStatus = when (raw.lowercase()) {
        "active"   -> UserStatus.ACTIVE
        "inactive" -> UserStatus.INACTIVE
        else       -> UserStatus.PENDING
    }
}
```

---

## Repository Using Mapper

```kotlin
class UserRepository @Inject constructor(
    private val userApi: UserApi,
    private val userDao: UserDao,
    private val userMapper: UserMapper
) {

    fun observeUsers(): Flow<List<User>> =
        userDao.observeAll()
            .map { entities -> entities.map(userMapper::toDomain) }

    suspend fun refreshUsers() {
        val dtos = userApi.getUsers()
        val entities = dtos.map(userMapper::toEntity)
        userDao.upsertAll(entities)
    }

    suspend fun createUser(user: User): User {
        val dto = userApi.createUser(user.toDto())
        val entity = userMapper.toEntity(userMapper.toDomain(dto))
        userDao.upsert(entity)
        return userMapper.toDomain(dto)
    }
}
```

---

## Handling Nullable / Missing Fields

```kotlin
// ✅ Provide sensible defaults — never return null for required domain fields
fun UserDto.toDomain(): User = User(
    id = id,
    name = name.takeIf { it.isNotBlank() } ?: "Unknown",
    email = email,
    status = runCatching { UserStatus.valueOf(status.uppercase()) }
        .getOrDefault(UserStatus.PENDING),
    createdAt = if (createdAt > 0) Instant.ofEpochMilli(createdAt) else Instant.now()
)
```

---

## Testing Mappers

```kotlin
// ✅ Mappers are pure — easy to unit test without mocks
class UserMapperTest {

    private val mapper = UserMapper()

    @Test
    fun `DTO maps to domain correctly`() {
        val dto = UserDto(
            id = "1",
            name = "Ali Rezaei",
            email = "ali@example.com",
            status = "active",
            createdAt = 1_700_000_000_000L
        )

        val domain = mapper.toDomain(dto)

        assertEquals("1", domain.id)
        assertEquals("Ali Rezaei", domain.name)
        assertEquals(UserStatus.ACTIVE, domain.status)
    }

    @Test
    fun `domain maps to entity and back`() {
        val user = User("1", "Ali", "ali@example.com", UserStatus.ACTIVE, Instant.now())
        val entity = mapper.toEntity(user)
        val restored = mapper.toDomain(entity)

        assertEquals(user.id, restored.id)
        assertEquals(user.status, restored.status)
    }
}
```

---

## Anti-Patterns

- Using DTOs directly in ViewModel or Compose — couples UI to API contract
- Using Entities in domain or UI layer — couples business logic to DB schema
- Mapping in ViewModel — mapping belongs in repository/data layer
- Not handling unknown enum values — crashes when API adds a new status
- Giant mapper files — split by domain entity
- Returning `null` for required domain fields — use defaults or `Result<T>`

---

## Related Skills
- `serialization` — DTO serialization setup
- `room` — Entity design and Room setup
- `repository-pattern` — repository using mappers
- `entity-design` — designing domain entities
- `domain-modeling` — domain model design principles
