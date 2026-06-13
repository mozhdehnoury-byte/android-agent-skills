---
name: feature-isolation
description: >
  Feature isolation patterns for Android modular projects.
  Load this skill when ensuring features don't leak implementation details,
  designing feature APIs, preventing feature-to-feature coupling,
  or structuring a feature module's internal vs public surface.
---

# Feature Isolation

## Overview

Feature isolation ensures that each feature module exposes only a minimal, stable public API while hiding all implementation details as `internal`. Features never depend on each other directly. Cross-feature communication happens via shared interfaces, callbacks, or a shared core module — never via direct imports.

---

## Core Principles

- **`internal` by default** — everything inside a feature is `internal` unless explicitly needed outside
- **No feature-to-feature imports** — features only import from `:core:*` modules
- **Public API is minimal** — expose only navigation routes, DI modules, and nav graph extensions
- **Feature entry points are composable extensions** — `NavGraphBuilder.featureGraph()`
- **Callbacks over navigation coupling** — features receive lambdas for cross-feature navigation

---

## Feature Public Surface

```kotlin
// ✅ What a feature exposes (public)
// feature/orders/src/main/kotlin/.../

// 1. Navigation contract
object OrdersNavigation {
    const val graphRoute = "orders_graph"
    const val listRoute = "orders/list"
    const val detailRoute = "orders/detail/{orderId}"

    fun detailRoute(orderId: String) = "orders/detail/$orderId"
}

// 2. NavGraph extension — entry point for :app
fun NavGraphBuilder.ordersGraph(
    navController: NavController,
    onNavigateToProduct: (productId: String) -> Unit,   // cross-feature via callback
    onNavigateToProfile: () -> Unit
) {
    navigation(
        startDestination = OrdersNavigation.listRoute,
        route = OrdersNavigation.graphRoute
    ) {
        composable(OrdersNavigation.listRoute) {
            OrderListScreen(                              // internal composable
                onOrderClick = { orderId ->
                    navController.navigate(OrdersNavigation.detailRoute(orderId))
                }
            )
        }
        composable(
            route = OrdersNavigation.detailRoute,
            arguments = listOf(navArgument("orderId") { type = NavType.StringType })
        ) { backStackEntry ->
            val orderId = backStackEntry.arguments?.getString("orderId") ?: return@composable
            OrderDetailScreen(
                orderId = orderId,
                onNavigateToProduct = onNavigateToProduct,
                onNavigateToProfile = onNavigateToProfile
            )
        }
    }
}

// 3. Hilt module (internal — bindings are auto-discovered by Hilt)
@Module
@InstallIn(SingletonComponent::class)
internal abstract class OrdersModule {
    @Binds
    abstract fun bindOrderRepository(impl: OrderRepositoryImpl): OrderRepository
}
```

---

## Internal Implementation

```kotlin
// ✅ Everything implementation-related is internal
internal class OrderRepositoryImpl @Inject constructor(
    private val dao: OrderDao,
    private val api: OrderApiService
) : OrderRepository { ... }

internal class GetOrdersUseCase @Inject constructor(
    private val repository: OrderRepository
) {
    operator fun invoke(): Flow<List<Order>> = repository.observeOrders()
}

@HiltViewModel
internal class OrderListViewModel @Inject constructor(
    private val getOrdersUseCase: GetOrdersUseCase
) : ViewModel() { ... }

@Composable
internal fun OrderListScreen(onOrderClick: (String) -> Unit) { ... }

@Composable
internal fun OrderDetailScreen(
    orderId: String,
    onNavigateToProduct: (String) -> Unit,
    onNavigateToProfile: () -> Unit
) { ... }
```

---

## Cross-Feature Communication Patterns

```kotlin
// ✅ Pattern 1: Callbacks (simplest — for navigation)
// :app wires the callback
ordersGraph(
    navController = navController,
    onNavigateToProduct = { productId ->
        navController.navigate(CatalogNavigation.detailRoute(productId))
    },
    onNavigateToProfile = {
        navController.navigate(ProfileNavigation.graphRoute)
    }
)

// ✅ Pattern 2: Shared interface in :core:domain (for data)
// :core:domain
interface ProductInfoProvider {
    suspend fun getProductSummary(productId: String): Result<ProductSummary>
}

// :feature:catalog implements it
internal class ProductInfoProviderImpl @Inject constructor(
    private val repository: ProductRepository
) : ProductInfoProvider { ... }

// :feature:orders uses it (no knowledge of :feature:catalog)
internal class OrderDetailViewModel @Inject constructor(
    private val productInfoProvider: ProductInfoProvider  // injected, not imported
) : ViewModel() { ... }

// ✅ Pattern 3: Shared ViewModel in :app (for truly global state)
// :app — AuthViewModel scoped to root nav graph
@Composable
fun AppNavHost(navController: NavHostController) {
    val appViewModel: AppViewModel = hiltViewModel()  // scoped to root
    val authState by appViewModel.authState.collectAsStateWithLifecycle()

    ordersGraph(
        navController = navController,
        currentUserId = authState.userId   // passed down as param
    )
}
```

---

## Feature API Contract Checklist

```
✅ Public (exposed to :app):
  - Navigation object with route constants
  - NavGraphBuilder extension function
  - Hilt module (auto-discovered, but still internal annotation on bindings)

❌ Must be internal:
  - All ViewModel classes
  - All composable functions
  - All UseCase classes
  - All Repository implementations
  - All DAO and API service classes
  - All domain model classes (unless in :core:domain)
  - All Hilt @Binds/@Provides methods
```

---

## Detecting Isolation Violations

```kotlin
// ❌ Violation: feature:cart importing from feature:catalog
import com.example.feature.catalog.domain.model.Product  // WRONG

// ✅ Fix: define what cart needs in :core:domain or via an interface
// :core:domain
data class ProductSummary(val id: String, val name: String, val price: Money)

// :feature:cart uses ProductSummary — :feature:catalog maps to it
```

---

## Anti-Patterns

- `public` on internal ViewModels or composables — leaks implementation, creates coupling
- Feature importing another feature's ViewModel — use shared ViewModel in :app instead
- Passing navigation controller deep into composables — use callbacks instead
- Shared mutable state between features via static/companion objects — use proper DI scope
- One feature calling another feature's UseCase directly — go through shared interface in :core

---

## Related Skills

- `modularization` — module structure that enables isolation
- `multi-module-architecture` — full multi-module setup
- `bounded-context` — conceptual boundaries between features
- `hilt` — scoping DI across modules correctly
- `navigation` — wiring navigation between isolated features
