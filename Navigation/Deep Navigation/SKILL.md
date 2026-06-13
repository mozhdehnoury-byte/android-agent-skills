---
name: deep-navigation
description: >
  Deep link handling in Android with Jetpack Compose Navigation.
  Load this skill when implementing URI-based deep links, handling
  incoming intents from notifications or external apps, defining
  deep link patterns, or testing deep link navigation.
---

# Deep Navigation

## Overview
Deep links allow the app to navigate directly to a specific screen from an external source — a notification, a web URL, another app, or an NFC tag. Navigation Compose supports deep links via `deepLinks` parameter on composable destinations.

---

## Core Principles

- Define deep link URIs in a central constants object — never inline strings
- Always verify deep links in the manifest with `android:autoVerify="true"` for App Links
- Handle deep link arguments the same way as regular nav arguments — via `toRoute()`
- Deep links must be **tested with adb** before release
- Provide fallback navigation when a deep link destination requires authentication

---

## Manifest Setup

```xml
<!-- AndroidManifest.xml -->
<activity
    android:name=".MainActivity"
    android:exported="true">

    <intent-filter android:autoVerify="true">
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="https"
              android:host="example.com"
              android:pathPrefix="/users" />
    </intent-filter>

    <!-- Custom scheme for internal deep links -->
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="myapp" />
    </intent-filter>

</activity>
```

---

## Route with Deep Link

```kotlin
// ✅ Define deep link URIs centrally
object DeepLinks {
    const val BASE = "https://example.com"
    const val USER_DETAIL = "$BASE/users/{userId}"
    const val ORDER_DETAIL = "$BASE/orders/{orderId}"

    // Custom scheme
    const val NOTIFICATION_BASE = "myapp://notification"
    const val NOTIFICATION_DETAIL = "$NOTIFICATION_BASE/{notificationId}"
}

// ✅ Attach deep links to composable destination
composable<UserDetailRoute>(
    deepLinks = listOf(
        navDeepLink<UserDetailRoute>(
            basePath = "https://example.com/users"
        )
    )
) { backStackEntry ->
    val route: UserDetailRoute = backStackEntry.toRoute()
    UserDetailScreen(userId = route.userId)
}

// ✅ Legacy string-based deep link (if not using type-safe routes)
composable(
    route = "user/{userId}",
    deepLinks = listOf(
        navDeepLink {
            uriPattern = DeepLinks.USER_DETAIL
        }
    ),
    arguments = listOf(navArgument("userId") { type = NavType.StringType })
) { backStackEntry ->
    val userId = backStackEntry.arguments?.getString("userId") ?: return@composable
    UserDetailScreen(userId = userId)
}
```

---

## Handling Incoming Intent in Activity

```kotlin
// ✅ Pass intent to NavHost for deep link handling
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            AppTheme {
                val navController = rememberNavController()
                AppNavHost(navController = navController)
            }
        }
    }
}

// Navigation Compose handles intent automatically when
// NavHost is set up — no manual intent parsing needed
```

---

## Notification Deep Links

```kotlin
// ✅ Build PendingIntent with deep link URI
fun buildNotificationIntent(context: Context, userId: String): PendingIntent {
    val deepLinkUri = Uri.parse("https://example.com/users/$userId")

    val intent = Intent(Intent.ACTION_VIEW, deepLinkUri).apply {
        setPackage(context.packageName)
        flags = Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_CLEAR_TOP
    }

    return TaskStackBuilder.create(context).run {
        addNextIntentWithParentStack(intent)
        getPendingIntent(
            userId.hashCode(),
            PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
        )!!
    }
}
```

---

## Auth Guard for Deep Links

```kotlin
// ✅ Redirect unauthenticated deep link to login
@Composable
fun AppNavHost(navController: NavHostController) {
    val authState by authViewModel.state.collectAsStateWithLifecycle()

    NavHost(navController = navController, startDestination = HomeRoute) {

        composable<UserDetailRoute> { backStackEntry ->
            if (authState !is AuthState.Authenticated) {
                LaunchedEffect(Unit) {
                    navController.navigate(LoginRoute) {
                        // store original destination for post-login redirect
                        navController.currentBackStackEntry
                            ?.savedStateHandle
                            ?.set("pending_route", "user_detail")
                    }
                }
            } else {
                val route: UserDetailRoute = backStackEntry.toRoute()
                UserDetailScreen(userId = route.userId)
            }
        }
    }
}
```

---

## Testing Deep Links with ADB

```bash
# Test HTTPS deep link
adb shell am start \
  -W -a android.intent.action.VIEW \
  -d "https://example.com/users/123" \
  com.example.app

# Test custom scheme
adb shell am start \
  -W -a android.intent.action.VIEW \
  -d "myapp://notification/456" \
  com.example.app
```

---

## Anti-Patterns

- Inline URI strings in `navDeepLink {}` — define in a constants object
- Not adding `autoVerify="true"` for HTTPS App Links — won't be verified by Google
- Parsing the intent URI manually in `onCreate` — Navigation Compose handles this
- Not handling the unauthenticated deep link case — sends user to a broken state
- Using deep links for in-app navigation between features — use regular routes

---

## Related Skills
- `navigation` — core navigation setup
- `nested-navigation` — deep links into nested graphs
- `notification` — building notifications with deep link intents
- `manifest` — intent filter configuration
