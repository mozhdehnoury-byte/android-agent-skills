---
name: entity-design
description: >
  Designing Room database entities for Android.
  Load this skill when defining @Entity classes, choosing primary keys,
  modeling relationships (one-to-many, many-to-many), using type converters,
  handling nullable columns, or mapping between domain models and entities.
---

# Entity Design

## Overview

Room entities are data classes annotated with `@Entity` that map directly to SQLite tables. Entities live in the Data layer and must never leak into Domain or Presentation. The mapping between Entity and Domain model is always explicit.

---

## Core Principles

- Entities live in **Data layer only** — never imported by Domain or Presentation
- Entity fields map to **SQLite columns** — use types SQLite supports natively or via TypeConverter
- Every entity has a **primary key** — prefer `String` UUID over auto-generated `Int` for distributed data
- **Nullable columns** only when the data is truly optional — not as a lazy default
- Entity names reflect **storage** — Domain names reflect **business concepts**

---

## Basic Entity

```kotlin
// ✅ Standard entity
@Entity(tableName = "users")
data class UserEntity(
    @PrimaryKey val id: String,
    @ColumnInfo(name = "full_name") val name: String,
    @ColumnInfo(name = "email_address") val email: String,
    @ColumnInfo(name = "role") val role: String,          // store enum as String
    @ColumnInfo(name = "is_active") val isActive: Boolean,
    @ColumnInfo(name = "created_at") val createdAt: Long, // store Instant as epoch millis
    @ColumnInfo(name = "updated_at") val updatedAt: Long
)

// ✅ Mapping to/from domain
fun UserEntity.toDomain(): User = User(
    id = id,
    name = name,
    email = Email(email),
    role = UserRole.valueOf(role),
    status = if (isActive) UserStatus.ACTIVE else UserStatus.SUSPENDED,
    createdAt = Instant.ofEpochMilli(createdAt)
)

fun User.toEntity(): UserEntity = UserEntity(
    id = id,
    name = name,
    email = email.value,
    role = role.name,
    isActive = status == UserStatus.ACTIVE,
    createdAt = createdAt.toEpochMilli(),
    updatedAt = System.currentTimeMillis()
)
```

---

## Primary Keys

```kotlin
// ✅ String UUID — preferred for distributed/synced data
@Entity(tableName = "orders")
data class OrderEntity(
    @PrimaryKey val id: String = UUID.randomUUID().toString(),
    ...
)

// ✅ Auto-increment Int — for local-only data with no sync
@Entity(tableName = "drafts")
data class DraftEntity(
    @PrimaryKey(autoGenerate = true) val id: Int = 0,
    val content: String,
    val createdAt: Long
)

// ✅ Composite primary key
@Entity(
    tableName = "user_roles",
    primaryKeys = ["user_id", "role_id"]
)
data class UserRoleEntity(
    @ColumnInfo(name = "user_id") val userId: String,
    @ColumnInfo(name = "role_id") val roleId: String,
    @ColumnInfo(name = "assigned_at") val assignedAt: Long
)
```

---

## Relationships

```kotlin
// ✅ One-to-many: Order has many OrderItems
@Entity(tableName = "orders")
data class OrderEntity(
    @PrimaryKey val id: String,
    @ColumnInfo(name = "customer_id") val customerId: String,
    @ColumnInfo(name = "status") val status: String,
    @ColumnInfo(name = "placed_at") val placedAt: Long
)

@Entity(
    tableName = "order_items",
    foreignKeys = [
        ForeignKey(
            entity = OrderEntity::class,
            parentColumns = ["id"],
            childColumns = ["order_id"],
            onDelete = ForeignKey.CASCADE  // delete items when order deleted
        )
    ],
    indices = [Index("order_id")]  // ✅ index foreign key columns
)
data class OrderItemEntity(
    @PrimaryKey val id: String,
    @ColumnInfo(name = "order_id") val orderId: String,
    @ColumnInfo(name = "product_id") val productId: String,
    @ColumnInfo(name = "quantity") val quantity: Int,
    @ColumnInfo(name = "unit_price") val unitPrice: Long
)

// ✅ Relation data class for queries
data class OrderWithItems(
    @Embedded val order: OrderEntity,
    @Relation(
        parentColumn = "id",
        entityColumn = "order_id"
    )
    val items: List<OrderItemEntity>
)

// ✅ Many-to-many via junction table
@Entity(
    tableName = "product_tags",
    primaryKeys = ["product_id", "tag_id"]
)
data class ProductTagCrossRef(
    @ColumnInfo(name = "product_id") val productId: String,
    @ColumnInfo(name = "tag_id") val tagId: String
)

data class ProductWithTags(
    @Embedded val product: ProductEntity,
    @Relation(
        parentColumn = "id",
        entityColumn = "id",
        associateBy = Junction(ProductTagCrossRef::class,
            parentColumn = "product_id",
            entityColumn = "tag_id")
    )
    val tags: List<TagEntity>
)
```

---

## Type Converters

```kotlin
// ✅ TypeConverter for types Room can't store natively
class Converters {

    // List<String>
    @TypeConverter
    fun fromStringList(value: List<String>): String = value.joinToString(",")

    @TypeConverter
    fun toStringList(value: String): List<String> =
        if (value.isBlank()) emptyList() else value.split(",")

    // Enum (store as String, not ordinal — ordinals break on reorder)
    @TypeConverter
    fun fromUserRole(role: UserRole): String = role.name

    @TypeConverter
    fun toUserRole(value: String): UserRole = UserRole.valueOf(value)

    // Instant (store as epoch millis)
    @TypeConverter
    fun fromInstant(instant: Instant?): Long? = instant?.toEpochMilli()

    @TypeConverter
    fun toInstant(value: Long?): Instant? = value?.let { Instant.ofEpochMilli(it) }
}

// ✅ Register converters on the database
@Database(entities = [...], version = 1)
@TypeConverters(Converters::class)
abstract class AppDatabase : RoomDatabase()
```

---

## Indices

```kotlin
// ✅ Index frequently queried columns
@Entity(
    tableName = "products",
    indices = [
        Index(value = ["category_id"]),                          // single column
        Index(value = ["name", "category_id"]),                  // composite
        Index(value = ["sku"], unique = true)                    // unique constraint
    ]
)
data class ProductEntity(
    @PrimaryKey val id: String,
    @ColumnInfo(name = "name") val name: String,
    @ColumnInfo(name = "sku") val sku: String,
    @ColumnInfo(name = "category_id") val categoryId: String,
    @ColumnInfo(name = "price") val price: Long
)
```

---

## Anti-Patterns

- `@Entity` class used in Domain or Presentation layer — entities are Data layer only
- Storing enum as ordinal (`role.ordinal`) — breaks when enum values are reordered; use `name`
- Missing `Index` on foreign key columns — causes full table scan on joins
- `onDelete = ForeignKey.NO_ACTION` on cascading data — leaves orphan rows
- Using `@Ignore` to skip fields instead of proper mapping — creates confusion about what's stored
- Storing complex objects as JSON strings without a TypeConverter — makes querying impossible

---

## Related Skills

- `dao` — DAO patterns for querying entities
- `room` — Room database setup and configuration
- `migration` — handling entity schema changes
- `domain-modeling` — domain models that entities map to
- `dto-mapping` — mapping patterns between layers
- `indexing` — in-depth index strategy for performance
