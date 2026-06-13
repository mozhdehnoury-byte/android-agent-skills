---
name: use-case-design
description: >
  Use Case design for Android Clean Architecture.
  Load this skill when creating UseCase classes, deciding what logic
  belongs in a UseCase vs Repository vs ViewModel, structuring UseCase
  inputs and outputs, or composing multiple UseCases together.
---

# Use Case Design

## Overview

A UseCase (also called Interactor) encapsulates a single business operation. It lives in the Domain layer, depends only on Repository interfaces and domain models, and is called by the ViewModel. ViewModels never call Repositories directly — they always go through a UseCase.

---

## Core Principles

- **One UseCase, one operation** — `GetUserUseCase`, not `UserUseCase`
- Lives in **Domain layer** — pure Kotlin, no Android imports
- Depends on **Repository interfaces** — not implementations
- Called by **ViewModel only** — never from UI or another UseCase (compose at ViewModel level)
- Returns **Result<T>** for suspend operations, **Flow<T>** for reactive streams
- **No state** — UseCases are stateless; state lives in ViewModel

---

## Standard UseCase (Suspend)

```kotlin
// ✅ Suspend UseCase — one operation, invokable via operator fun
class GetUserUseCase @Inject constructor(
    private val userRepository: UserRepository
) {
    suspend operator fun invoke(userId: String): Result<User> =
        userRepository.getUser(userId)
}

class DeleteUserUseCase @Inject constructor(
    private val userRepository: UserRepository
) {
    suspend operator fun invoke(userId: String): Result<Unit> =
        userRepository.deleteUser(userId)
}

class CreateOrderUseCase @Inject constructor(
    private val orderRepository: OrderRepository,
    private val cartRepository: CartRepository,
    private val inventoryRepository: InventoryRepository
) {
    suspend operator fun invoke(cartId: String, userId: String): Result<Order> =
        runCatching {
            val cart = cartRepository.getCart(cartId).getOrThrow()
            require(cart.items.isNotEmpty()) { "Cannot create order from empty cart" }

            // Business rule: validate inventory before placing order
            cart.items.forEach { item ->
                val available = inventoryRepository.getStock(item.productId).getOrThrow()
                require(available >= item.quantity) {
                    "Insufficient stock for ${item.productName}"
                }
            }

            val order = orderRepository.create(cart.toOrderRequest(userId)).getOrThrow()
            cartRepository.clear(cartId)
            order
        }
}
```

---

## Reactive UseCase (Flow)

```kotlin
// ✅ Flow UseCase — for ongoing observation
class ObserveUsersUseCase @Inject constructor(
    private val userRepository: UserRepository
) {
    operator fun invoke(): Flow<List<User>> =
        userRepository.observeUsers()
}

// ✅ Flow UseCase with transformation
class ObserveActiveUsersUseCase @Inject constructor(
    private val userRepository: UserRepository
) {
    operator fun invoke(): Flow<List<User>> =
        userRepository.observeUsers()
            .map { users -> users.filter { it.status == UserStatus.ACTIVE } }
            .distinctUntilChanged()
}

// ✅ Flow UseCase combining multiple sources
class ObserveDashboardUseCase @Inject constructor(
    private val userRepository: UserRepository,
    private val orderRepository: OrderRepository
) {
    operator fun invoke(userId: String): Flow<DashboardData> =
        combine(
            userRepository.observeUser(userId),
            orderRepository.observeRecentOrders(userId)
        ) { user, orders ->
            DashboardData(
                userName = user?.name ?: "",
                recentOrderCount = orders.size,
                pendingOrderCount = orders.count { it.status == OrderStatus.PENDING }
            )
        }
}
```

---

## UseCase with Parameters

```kotlin
// ✅ Simple primitive params — pass directly
class SearchProductsUseCase @Inject constructor(
    private val repository: ProductRepository
) {
    suspend operator fun invoke(query: String, filter: ProductFilter): Result<List<Product>> =
        repository.search(query, filter)
}

// ✅ Complex params — use a dedicated Params data class
class PlaceOrderUseCase @Inject constructor(
    private val repository: OrderRepository
) {
    data class Params(
        val cartId: String,
        val shippingAddressId: String,
        val paymentMethodId: String,
        val couponCode: String? = null
    )

    suspend operator fun invoke(params: Params): Result<Order> =
        runCatching {
            // apply coupon if present
            val discount = params.couponCode?.let {
                repository.validateCoupon(it).getOrThrow()
            }
            repository.placeOrder(params.toRequest(discount)).getOrThrow()
        }
}

// ViewModel usage
fun onPlaceOrder() {
    viewModelScope.launch {
        val params = PlaceOrderUseCase.Params(
            cartId = state.value.cartId,
            shippingAddressId = state.value.selectedAddressId,
            paymentMethodId = state.value.selectedPaymentId,
            couponCode = state.value.couponCode.takeIf { it.isNotBlank() }
        )
        placeOrderUseCase(params).fold(
            onSuccess = { _events.send(Event.NavigateToConfirmation(it.id)) },
            onFailure = { _state.update { s -> s.copy(error = it.message) } }
        )
    }
}
```

---

## ViewModel Usage

```kotlin
// ✅ ViewModel composes UseCases — never calls Repository directly
@HiltViewModel
class UserDetailViewModel @Inject constructor(
    savedStateHandle: SavedStateHandle,
    private val getUserUseCase: GetUserUseCase,
    private val updateUserUseCase: UpdateUserUseCase,
    private val deleteUserUseCase: DeleteUserUseCase
) : ViewModel() {

    private val userId: String = checkNotNull(savedStateHandle["userId"])

    val state: StateFlow<UserDetailUiState> =
        getUserUseCase.asFlow(userId)   // extension if needed
            .map { result ->
                result.fold(
                    onSuccess = { UserDetailUiState.Success(it) },
                    onFailure = { UserDetailUiState.Error(it.message ?: "Error") }
                )
            }
            .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), UserDetailUiState.Loading)

    fun onDeleteClick() {
        viewModelScope.launch {
            deleteUserUseCase(userId).fold(
                onSuccess = { _events.send(Event.NavigateBack) },
                onFailure = { _events.send(Event.ShowError(it.message ?: "Delete failed")) }
            )
        }
    }
}
```

---

## UseCase Package Structure

```
domain/
└── usecase/
    ├── user/
    │   ├── GetUserUseCase.kt
    │   ├── GetUsersUseCase.kt
    │   ├── ObserveUsersUseCase.kt
    │   ├── CreateUserUseCase.kt
    │   ├── UpdateUserUseCase.kt
    │   └── DeleteUserUseCase.kt
    ├── order/
    │   ├── PlaceOrderUseCase.kt
    │   ├── CancelOrderUseCase.kt
    │   └── ObserveOrdersUseCase.kt
    └── auth/
        ├── LoginUseCase.kt
        └── LogoutUseCase.kt
```

---

## Anti-Patterns

- ViewModel calling Repository directly — always go through UseCase
- UseCase with multiple unrelated responsibilities — split into separate UseCases
- UseCase holding state — stateless only; state belongs in ViewModel
- UseCase importing Android framework classes — Domain layer is pure Kotlin
- Generic `UserUseCase` with many methods — one class, one operation
- UseCase calling another UseCase — compose at ViewModel level instead

---

## Related Skills

- `clean-architecture` — layer structure UseCase sits within
- `mvvm` — ViewModel calling UseCases
- `repository-pattern` — Repository interfaces UseCase depends on
- `domain-modeling` — domain models UseCase operates on
- `side-effect-management` — handling side effects triggered by UseCases
