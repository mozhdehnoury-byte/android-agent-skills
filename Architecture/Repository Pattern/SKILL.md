---
name: repository-pattern
description: >
  Repository pattern for Android data access abstraction.
  Load this skill when implementing a Repository, defining its interface,
  coordinating between local and remote data sources, handling caching logic,
  or structuring the data layer of a feature.
---

# Repository Pattern

## Overview

The Repository pattern abstracts data sources behind a single interface. The ViewModel and UseCases never know whether data comes from Room, Retrofit, DataStore, or memory cache — they only interact with the Repository interface defined in the Domain layer. The implementation in the Data layer coordinates between local and remote sources.

---

## Core Principles

- Repository **interface** lives in Domain — pure Kotlin, no Android dependencies
- Repository **implementation** lives in Data — knows about Room, Retrofit, etc.
- Repository is the **only entry point** to data — no direct DAO or API calls from UseCases
- Repositories return **Domain models** — never DTOs or Entities
- One repository per **aggregate root** — not per screen or per DAO

---

## Interface (Domain Layer)

```kotlin
// ✅ Interface in domain — clean, no framework dependencies
interface UserRepository {
    // Reactive — for UI observation
    fun observeUsers(): Flow<List<User>>
    fun observeUser(id: String): Flow<User?>

    // Suspend — for one-time reads and writes
    suspend fun getUser(id: String): Result<User>
    suspend fun getUsers(): Result<List<User>>
    suspend fun createUser(user: User): Result<User>
    suspend fun updateUser(user: User): Result<Unit>
    suspend fun deleteUser(id: String): Result<Unit>

    // Domain operations
    suspend fun searchUsers(query: String): Result<List<User>>
    suspend fun syncUsers(): Result<Unit>
}
```

---

## Implementation (Data Layer)

```kotlin
// ✅ Implementation in data layer — coordinates sources
class UserRepositoryImpl @Inject constructor(
    private val userDao: UserDao,
    private val userApiService: UserApiService,
    private val dispatcher: CoroutineDispatcher = Dispatchers.IO
) : UserRepository {

    // ✅ Observe local DB — UI updates automatically when DB changes
    override fun observeUsers(): Flow<List<User>> =
        userDao.observeAll()
            .map { entities -> entities.map { it.toDomain() } }
            .flowOn(dispatcher)

    // ✅ Cache-first read
    override suspend fun getUser(id: String): Result<User> = withContext(dispatcher) {
        runCatching {
            userDao.getById(id)?.toDomain()
                ?: fetchAndCacheUser(id)
        }
    }

    // ✅ Write to local + remote
    override suspend fun createUser(user: User): Result<User> = withContext(dispatcher) {
        runCatching {
            val dto = userApiService.createUser(user.toCreateRequest())
            val created = dto.toDomain()
            userDao.insert(created.toEntity())
            created
        }
    }

    override suspend fun updateUser(user: User): Result<Unit> = withContext(dispatcher) {
        runCatching {
            userApiService.updateUser(user.id, user.toUpdateRequest())
            userDao.upsert(user.toEntity())
        }
    }

    override suspend fun deleteUser(id: String): Result<Unit> = withContext(dispatcher) {
        runCatching {
            userApiService.deleteUser(id)
            userDao.deleteById(id)
        }
    }

    override suspend fun syncUsers(): Result<Unit> = withContext(dispatcher) {
        runCatching {
            val dtos = userApiService.getUsers()
            val entities = dtos.map { it.toDomain().toEntity() }
            userDao.replaceAll(entities)
        }
    }

    private suspend fun fetchAndCacheUser(id: String): User {
        val dto = userApiService.getUser(id)
        val domain = dto.toDomain()
        userDao.insert(domain.toEntity())
        return domain
    }
}
```

---

## Hilt Binding

```kotlin
// ✅ Bind interface to implementation
@Module
@InstallIn(SingletonComponent::class)
abstract class UserRepositoryModule {

    @Binds
    @Singleton
    abstract fun bindUserRepository(
        impl: UserRepositoryImpl
    ): UserRepository
}
```

---

## Repository Strategies

```kotlin
// ✅ Strategy 1: Network-first (always fresh data)
override suspend fun getUser(id: String): Result<User> = runCatching {
    val dto = userApiService.getUser(id)
    val domain = dto.toDomain()
    userDao.upsert(domain.toEntity())
    domain
}

// ✅ Strategy 2: Cache-first (fast, offline-capable)
override suspend fun getUser(id: String): Result<User> = runCatching {
    userDao.getById(id)?.toDomain() ?: run {
        val dto = userApiService.getUser(id)
        val domain = dto.toDomain()
        userDao.insert(domain.toEntity())
        domain
    }
}

// ✅ Strategy 3: Cache-then-network (show cached, refresh in background)
override fun observeUserWithRefresh(id: String): Flow<User> = flow {
    userDao.getById(id)?.toDomain()?.let { emit(it) }
    val refreshed = userApiService.getUser(id).toDomain()
    userDao.upsert(refreshed.toEntity())
    emit(refreshed)
}
```

---

## Fake Repository for Testing

```kotlin
// ✅ Fake implementation for unit tests — no Room, no Retrofit
class FakeUserRepository : UserRepository {
    private val users = MutableStateFlow<List<User>>(emptyList())
    var shouldThrow = false

    override fun observeUsers(): Flow<List<User>> = users

    override suspend fun getUser(id: String): Result<User> {
        if (shouldThrow) return Result.failure(Exception("Network error"))
        return users.value.find { it.id == id }
            ?.let { Result.success(it) }
            ?: Result.failure(Exception("Not found"))
    }

    override suspend fun createUser(user: User): Result<User> {
        users.update { it + user }
        return Result.success(user)
    }

    fun emit(userList: List<User>) {
        users.value = userList
    }
}
```

---

## Anti-Patterns

- DAO called directly from ViewModel or UseCase — always go through Repository
- Repository returning Entity or DTO — always map to Domain model before returning
- One Repository per screen — one Repository per aggregate root (User, Order, Product)
- Repository containing UI logic — Repository only handles data access and mapping
- Not using `flowOn(dispatcher)` — DB and network operations must not run on Main thread

---

## Related Skills

- `clean-architecture` — where Repository fits in layer structure
- `dao` — DAO patterns the Repository coordinates
- `dto-mapping` — mapping between DTO/Entity and Domain model
- `use-case-design` — UseCases that call Repositories
- `cache-strategy` — cache invalidation and expiry policies
- `offline-first` — Repository design for offline-capable apps
