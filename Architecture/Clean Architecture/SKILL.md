---
name: clean-architecture
description: >
  Clean Architecture layering for Android projects.
  Load this skill when designing layer boundaries, defining dependencies between
  layers, structuring packages, or deciding where logic belongs (UI vs Domain vs Data).
---

# Clean Architecture

## Overview

Clean Architecture organizes code into three concentric layers: **Presentation**, **Domain**, and **Data**. Dependencies flow inward only — outer layers depend on inner layers, never the reverse. The Domain layer is pure Kotlin with no Android dependencies.

---

## Layer Structure

```
app/
├── presentation/       # Compose UI, ViewModels, UiState, UiEvent
│   └── feature/
│       ├── FeatureScreen.kt
│       ├── FeatureViewModel.kt
│       └── FeatureUiState.kt
├── domain/             # Pure Kotlin — UseCases, Entities, Repository interfaces
│   ├── model/
│   │   └── User.kt
│   ├── repository/
│   │   └── UserRepository.kt        # interface only
│   └── usecase/
│       └── GetUserUseCase.kt
└── data/               # Repository implementations, Room, Retrofit, DataStore
    ├── repository/
    │   └── UserRepositoryImpl.kt
    ├── local/
    │   ├── UserDao.kt
    │   └── UserEntity.kt
    ├── remote/
    │   ├── UserApiService.kt
    │   └── UserDto.kt
    └── mapper/
        └── UserMapper.kt
```

---

## Dependency Rule

```
Presentation → Domain ← Data
```

- **Domain** depends on nothing (no Android, no Room, no Retrofit)
- **Presentation** depends on Domain (uses UseCases and models)
- **Data** depends on Domain (implements Repository interfaces)
- **Data** never imports from **Presentation**
- **Presentation** never imports from **Data** directly

---

## Domain Layer

```kotlin
// ✅ Entity — pure Kotlin data class, no annotations
data class User(
    val id: String,
    val name: String,
    val email: String,
    val role: UserRole
)

enum class UserRole { ADMIN, MEMBER, GUEST }

// ✅ Repository interface — defined in Domain, implemented in Data
interface UserRepository {
    suspend fun getUser(id: String): Result<User>
    suspend fun getUsers(): Result<List<User>>
    suspend fun saveUser(user: User): Result<Unit>
    fun observeUsers(): Flow<List<User>>
}

// ✅ UseCase — single responsibility, pure logic
class GetUserUseCase @Inject constructor(
    private val repository: UserRepository
) {
    suspend operator fun invoke(id: String): Result<User> =
        repository.getUser(id)
}

class GetActiveUsersUseCase @Inject constructor(
    private val repository: UserRepository
) {
    operator fun invoke(): Flow<List<User>> =
        repository.observeUsers().map { users ->
            users.filter { it.role != UserRole.GUEST }
        }
}
```

---

## Data Layer

```kotlin
// ✅ Entity — Room-specific, isolated in data layer
@Entity(tableName = "users")
data class UserEntity(
    @PrimaryKey val id: String,
    val name: String,
    val email: String,
    val role: String
)

// ✅ DTO — API response model
data class UserDto(
    @SerialName("id") val id: String,
    @SerialName("full_name") val name: String,
    @SerialName("email_address") val email: String,
    @SerialName("role") val role: String
)

// ✅ Mapper — converts between layers
object UserMapper {
    fun UserDto.toDomain(): User = User(
        id = id,
        name = name,
        email = email,
        role = UserRole.valueOf(role.uppercase())
    )

    fun UserEntity.toDomain(): User = User(
        id = id,
        name = name,
        email = email,
        role = UserRole.valueOf(role.uppercase())
    )

    fun User.toEntity(): UserEntity = UserEntity(
        id = id,
        name = name,
        email = email,
        role = role.name.lowercase()
    )
}

// ✅ Repository implementation — in Data, implements Domain interface
class UserRepositoryImpl @Inject constructor(
    private val api: UserApiService,
    private val dao: UserDao
) : UserRepository {

    override suspend fun getUser(id: String): Result<User> = runCatching {
        val entity = dao.getUserById(id)
        entity?.toDomain() ?: run {
            val dto = api.getUser(id)
            val domain = dto.toDomain()
            dao.insert(domain.toEntity())
            domain
        }
    }

    override fun observeUsers(): Flow<List<User>> =
        dao.observeAll().map { entities -> entities.map { it.toDomain() } }
}
```

---

## Presentation Layer

```kotlin
// ✅ ViewModel uses UseCases — never Repository directly
@HiltViewModel
class UserListViewModel @Inject constructor(
    private val getActiveUsersUseCase: GetActiveUsersUseCase
) : ViewModel() {

    val state: StateFlow<UserListUiState> =
        getActiveUsersUseCase()
            .map { UserListUiState.Success(it) }
            .catch { emit(UserListUiState.Error(it.message ?: "Error")) }
            .stateIn(
                scope = viewModelScope,
                started = SharingStarted.WhileSubscribed(5_000),
                initialValue = UserListUiState.Loading
            )
}
```

---

## Hilt Binding

```kotlin
// ✅ Bind interface to implementation in data module
@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {

    @Binds
    @Singleton
    abstract fun bindUserRepository(
        impl: UserRepositoryImpl
    ): UserRepository
}
```

---

## Anti-Patterns

- Importing `android.*` or `androidx.*` in Domain layer — Domain must be pure Kotlin
- ViewModel calling Repository directly — always go through a UseCase
- Presentation importing Data classes (Entity, DTO) — only Domain models cross into Presentation
- UseCase containing more than one responsibility — one UseCase, one operation
- Repository interface defined in Data layer — interfaces belong in Domain
- Skipping the mapper — never pass DTO or Entity into Domain or Presentation

---

## Related Skills

- `mvvm` — ViewModel and UI state structure
- `use-case-design` — designing UseCases in Domain layer
- `repository-pattern` — Repository interface and implementation patterns
- `dto-mapping` — mapping between layers
- `hilt` — dependency injection wiring
- `domain-modeling` — modeling domain entities
