---
name: analytics
description: >
  Firebase Analytics setup and event tracking for Android.
  Load this skill when implementing event tracking, screen tracking,
  user properties, conversion funnels, or integrating analytics
  with the app's architecture.
---

# Analytics

## Overview
Firebase Analytics provides free, unlimited event tracking for Android apps. It automatically tracks some events (first_open, session_start) and allows custom events. Events appear in the Firebase console and can be forwarded to Google Analytics, BigQuery, and Ads.

---

## Core Principles

- Analytics must be **disabled in debug builds** — dev events pollute production data
- Track **meaningful user actions** — not every tap, only business-relevant events
- Use a **centralized analytics wrapper** — never call Firebase directly from UI
- Define an **event taxonomy** before implementation — consistent naming is critical
- User PII (name, email) must **never** be sent as event parameters

---

## Setup

```kotlin
dependencies {
    implementation(platform(libs.firebase.bom))
    implementation(libs.firebase.analytics)
}
```

---

## Analytics Wrapper

```kotlin
// ✅ Centralized wrapper — never call FirebaseAnalytics directly from UI/ViewModel
interface Analytics {
    fun logEvent(event: AnalyticsEvent)
    fun setUserId(userId: String?)
    fun setUserProperty(name: String, value: String?)
    fun logScreenView(screenName: String, screenClass: String)
}

class FirebaseAnalyticsImpl @Inject constructor(
    @ApplicationContext context: Context
) : Analytics {

    private val firebaseAnalytics = FirebaseAnalytics.getInstance(context)

    override fun logEvent(event: AnalyticsEvent) {
        firebaseAnalytics.logEvent(event.name, event.toBundle())
    }

    override fun setUserId(userId: String?) {
        firebaseAnalytics.setUserId(userId)
    }

    override fun setUserProperty(name: String, value: String?) {
        firebaseAnalytics.setUserProperty(name, value)
    }

    override fun logScreenView(screenName: String, screenClass: String) {
        firebaseAnalytics.logEvent(FirebaseAnalytics.Event.SCREEN_VIEW) {
            param(FirebaseAnalytics.Param.SCREEN_NAME, screenName)
            param(FirebaseAnalytics.Param.SCREEN_CLASS, screenClass)
        }
    }
}

// ✅ No-op implementation for debug builds
class NoOpAnalytics @Inject constructor() : Analytics {
    override fun logEvent(event: AnalyticsEvent) = Unit
    override fun setUserId(userId: String?) = Unit
    override fun setUserProperty(name: String, value: String?) = Unit
    override fun logScreenView(screenName: String, screenClass: String) = Unit
}
```

---

## Event Taxonomy

```kotlin
// ✅ Sealed class for type-safe event definitions
sealed class AnalyticsEvent(val name: String) {

    abstract fun toBundle(): Bundle

    // Auth events
    object Login : AnalyticsEvent("login") {
        override fun toBundle() = bundleOf(
            FirebaseAnalytics.Param.METHOD to "email"
        )
    }

    object Logout : AnalyticsEvent("logout") {
        override fun toBundle() = Bundle.EMPTY
    }

    data class SignUp(val method: String) : AnalyticsEvent("sign_up") {
        override fun toBundle() = bundleOf(
            FirebaseAnalytics.Param.METHOD to method
        )
    }

    // Content events
    data class ViewItem(val itemId: String, val itemName: String, val category: String)
        : AnalyticsEvent(FirebaseAnalytics.Event.VIEW_ITEM) {
        override fun toBundle() = bundleOf(
            FirebaseAnalytics.Param.ITEM_ID to itemId,
            FirebaseAnalytics.Param.ITEM_NAME to itemName,
            FirebaseAnalytics.Param.ITEM_CATEGORY to category
        )
    }

    data class Search(val query: String) : AnalyticsEvent(FirebaseAnalytics.Event.SEARCH) {
        override fun toBundle() = bundleOf(
            FirebaseAnalytics.Param.SEARCH_TERM to query
        )
    }

    // Error events
    data class ErrorOccurred(val errorType: String, val errorMessage: String)
        : AnalyticsEvent("error_occurred") {
        override fun toBundle() = bundleOf(
            "error_type" to errorType,
            "error_message" to errorMessage.take(100)  // max 100 chars
        )
    }

    // Feature usage
    data class FeatureUsed(val featureName: String) : AnalyticsEvent("feature_used") {
        override fun toBundle() = bundleOf("feature_name" to featureName)
    }
}
```

---

## Usage in ViewModel

```kotlin
// ✅ Log events from ViewModel — not from Compose/UI
@HiltViewModel
class ProductViewModel @Inject constructor(
    private val repository: ProductRepository,
    private val analytics: Analytics
) : ViewModel() {

    fun onProductViewed(product: Product) {
        analytics.logEvent(
            AnalyticsEvent.ViewItem(
                itemId = product.id,
                itemName = product.name,
                category = product.category
            )
        )
    }

    fun onSearchPerformed(query: String) {
        analytics.logEvent(AnalyticsEvent.Search(query))
    }
}
```

---

## Screen Tracking

```kotlin
// ✅ Track screen views automatically via Navigation
@Composable
fun AppNavHost(navController: NavHostController, analytics: Analytics) {

    val currentEntry by navController.currentBackStackEntryAsState()

    LaunchedEffect(currentEntry) {
        currentEntry?.destination?.route?.let { route ->
            analytics.logScreenView(
                screenName = route,
                screenClass = route
            )
        }
    }

    NavHost(navController, startDestination = HomeRoute) {
        // routes...
    }
}
```

---

## User Properties

```kotlin
// ✅ Set stable user attributes for segmentation
class UserAnalyticsManager @Inject constructor(private val analytics: Analytics) {

    fun onUserLoggedIn(user: User) {
        analytics.setUserId(user.id)
        analytics.setUserProperty("account_type", user.accountType)
        analytics.setUserProperty("subscription_plan", user.plan)
        // ❌ Never set PII
        // analytics.setUserProperty("email", user.email)  // WRONG
    }

    fun onUserLoggedOut() {
        analytics.setUserId(null)
    }
}
```

---

## Hilt Binding by Build Type

```kotlin
// ✅ Inject NoOp in debug, real impl in release
@Module
@InstallIn(SingletonComponent::class)
abstract class AnalyticsModule {

    @Binds
    @Singleton
    abstract fun bindAnalytics(
        // Switch based on build type
        impl: FirebaseAnalyticsImpl  // or NoOpAnalytics in debug flavor
    ): Analytics
}

// ✅ Better — use BuildConfig
@Module
@InstallIn(SingletonComponent::class)
object AnalyticsModule {

    @Provides
    @Singleton
    fun provideAnalytics(@ApplicationContext context: Context): Analytics =
        if (BuildConfig.DEBUG) NoOpAnalytics()
        else FirebaseAnalyticsImpl(context)
}
```

---

## Event Naming Rules

```
✅ snake_case — user_signed_up, product_viewed, checkout_started
✅ Max 40 characters for event name
✅ Max 25 parameters per event
✅ Max 100 characters for parameter value (strings)
❌ No spaces, no special characters except underscore
❌ No reserved prefixes: firebase_, google_, ga_
❌ No PII in any parameter
```

---

## Anti-Patterns

- Calling `FirebaseAnalytics` directly from Composable — not testable, hard to disable
- Logging analytics in debug builds — pollutes production dashboards
- Sending PII (email, name, phone) as event parameters — compliance violation
- Tracking every user interaction — noise, hard to find signal
- Inconsistent event names (`user_login` vs `login_user` vs `userLogin`) — breaks funnels
- No event taxonomy document — events drift over time, become meaningless

---

## Related Skills
- `firebase` — Firebase core setup
- `crashlytics` — crash reporting alongside analytics
- `remote-config` — feature flags informed by analytics
- `side-effect-management` — analytics as a side effect in ViewModel
