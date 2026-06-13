---
name: paging
description: >
  Jetpack Paging 3 setup and patterns for Android.
  Load this skill when loading large datasets incrementally, implementing
  infinite scroll, paginating network or database results, or combining
  paged network data with a local Room cache.
---

# Paging

## Overview
Jetpack Paging 3 enables loading large datasets in pages — from network, database, or both. It integrates with Room, Retrofit, and Compose. The library handles load state, error retry, and list diffing automatically.

---

## Core Principles

- Use Paging when a list **could grow unbounded** — don't load everything at once
- **PagingSource** defines how to load a single page
- **RemoteMediator** coordinates network + Room cache for offline-first paging
- **PagingData** is consumed only once — use `cachedIn(viewModelScope)` to survive rotation
- Never collect `PagingData` directly in ViewModel — use `collectAsLazyPagingItems` in Compose

---

## Setup

```toml
[versions]
paging = "3.3.2"

[libraries]
paging-runtime = { module = "androidx.paging:paging-runtime", version.ref = "paging" }
paging-compose  = { module = "androidx.paging:paging-compose", version.ref = "paging" }
paging-room     = { module = "androidx.room:room-paging", version.ref = "room" }
```

---

## PagingSource — Network Only

```kotlin
// ✅ Network-only PagingSource
class UserPagingSource(
    private val api: UserApi,
    private val query: String
) : PagingSource<Int, UserDto>() {

    override suspend fun load(params: LoadParams<Int>): LoadResult<Int, UserDto> {
        val page = params.key ?: 1

        return try {
            val response = api.getUsers(
                query = query,
                page = page,
                pageSize = params.loadSize
            )

            LoadResult.Page(
                data = response.users,
                prevKey = if (page == 1) null else page - 1,
                nextKey = if (response.users.isEmpty()) null else page + 1
            )
        } catch (e: IOException) {
            LoadResult.Error(e)
        } catch (e: HttpException) {
            LoadResult.Error(e)
        }
    }

    override fun getRefreshKey(state: PagingState<Int, UserDto>): Int? {
        return state.anchorPosition?.let { anchor ->
            state.closestPageToPosition(anchor)?.prevKey?.plus(1)
                ?: state.closestPageToPosition(anchor)?.nextKey?.minus(1)
        }
    }
}
```

---

## Repository — Network-Only Paging

```kotlin
class UserRepository @Inject constructor(
    private val api: UserApi,
    private val mapper: UserMapper
) {
    fun searchUsers(query: String): Flow<PagingData<User>> = Pager(
        config = PagingConfig(
            pageSize = 20,
            prefetchDistance = 5,
            enablePlaceholders = false
        ),
        pagingSourceFactory = { UserPagingSource(api, query) }
    ).flow.map { pagingData ->
        pagingData.map { mapper.toDomain(it) }
    }
}
```

---

## Room + Paging (Database-Only)

```kotlin
// ✅ Room generates PagingSource automatically
@Dao
interface UserDao {
    @Query("SELECT * FROM users ORDER BY name ASC")
    fun pagingSource(): PagingSource<Int, UserEntity>
}

// Repository
class UserRepository @Inject constructor(private val userDao: UserDao) {
    fun observeUsersPaged(): Flow<PagingData<User>> = Pager(
        config = PagingConfig(pageSize = 20),
        pagingSourceFactory = { userDao.pagingSource() }
    ).flow.map { it.map(UserEntity::toDomain) }
}
```

---

## RemoteMediator — Network + Room Cache

```kotlin
// ✅ RemoteMediator: network fills Room, Room provides paged data
@OptIn(ExperimentalPagingApi::class)
class UserRemoteMediator(
    private val api: UserApi,
    private val db: AppDatabase
) : RemoteMediator<Int, UserEntity>() {

    override suspend fun load(
        loadType: LoadType,
        state: PagingState<Int, UserEntity>
    ): MediatorResult {

        val page = when (loadType) {
            LoadType.REFRESH -> 1
            LoadType.PREPEND -> return MediatorResult.Success(endOfPaginationReached = true)
            LoadType.APPEND  -> {
                val lastItem = state.lastItemOrNull()
                    ?: return MediatorResult.Success(endOfPaginationReached = false)
                getNextPageForItem(lastItem)
            }
        }

        return try {
            val response = api.getUsers(page = page, pageSize = state.config.pageSize)

            db.withTransaction {
                if (loadType == LoadType.REFRESH) {
                    db.userDao().deleteAll()
                }
                db.userDao().upsertAll(response.users.map { it.toEntity() })
            }

            MediatorResult.Success(endOfPaginationReached = response.users.isEmpty())
        } catch (e: IOException) {
            MediatorResult.Error(e)
        } catch (e: HttpException) {
            MediatorResult.Error(e)
        }
    }

    private suspend fun getNextPageForItem(item: UserEntity): Int {
        // derive page number from item position
        return (db.userDao().getItemIndex(item.id) / 20) + 2
    }
}

// ✅ Pager with RemoteMediator
@OptIn(ExperimentalPagingApi::class)
fun usersPaged(): Flow<PagingData<User>> = Pager(
    config = PagingConfig(pageSize = 20),
    remoteMediator = UserRemoteMediator(api, db),
    pagingSourceFactory = { db.userDao().pagingSource() }
).flow.map { it.map(UserEntity::toDomain) }
```

---

## ViewModel

```kotlin
@HiltViewModel
class UserListViewModel @Inject constructor(
    private val repository: UserRepository
) : ViewModel() {

    private val _query = MutableStateFlow("")

    // ✅ cachedIn — survives rotation without re-fetching
    val users: Flow<PagingData<User>> = _query
        .debounce(300)
        .flatMapLatest { query -> repository.searchUsers(query) }
        .cachedIn(viewModelScope)

    fun onQueryChanged(query: String) { _query.value = query }
}
```

---

## Compose UI

```kotlin
@Composable
fun UserListScreen(viewModel: UserListViewModel = hiltViewModel()) {
    val users = viewModel.users.collectAsLazyPagingItems()

    LazyColumn {
        items(
            count = users.itemCount,
            key = users.itemKey { it.id }
        ) { index ->
            val user = users[index]
            if (user != null) UserCard(user = user)
            else UserCardPlaceholder()
        }

        // ✅ Load state handling
        when (val state = users.loadState.append) {
            is LoadState.Loading -> item { LoadingIndicator() }
            is LoadState.Error   -> item {
                ErrorItem(
                    message = state.error.message ?: "Error",
                    onRetry = { users.retry() }
                )
            }
            else -> Unit
        }
    }

    // ✅ Refresh state
    when (users.loadState.refresh) {
        is LoadState.Loading -> FullScreenLoading()
        is LoadState.Error   -> FullScreenError(onRetry = { users.refresh() })
        else -> Unit
    }
}
```

---

## Anti-Patterns

- Loading entire dataset without Paging when the list can grow unbounded
- Not using `cachedIn(viewModelScope)` — re-fetches from page 1 on rotation
- Collecting `PagingData` in ViewModel with `collect {}` — it's a one-shot stream
- Missing load state handling — no loading indicator or error state shown
- Using `enablePlaceholders = true` without proper placeholder UI

---

## Related Skills
- `room` — Room PagingSource generation
- `retrofit` — network data source for PagingSource
- `repository-pattern` — Pager lives in repository
- `compose` — LazyColumn with LazyPagingItems
- `cache-strategy` — offline-first paging with RemoteMediator
