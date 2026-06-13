---
name: deep-link
description: >
  Deep link handling at the Android system level.
  Load this skill when configuring intent filters, handling incoming URIs
  in the Activity, setting up App Links with digital asset links,
  or processing deep links from notifications and external apps.
---

# Deep Link

## Overview
Deep links allow external sources — browsers, other apps, notifications — to open a specific screen inside your app via a URI. Android supports two types: custom scheme deep links (`myapp://`) and App Links (`https://`). App Links require domain verification and are more secure.

---

## Core Principles

- Prefer **App Links** (HTTPS) over custom schemes — they can't be intercepted by other apps
- Verify App Links with `assetlinks.json` hosted on your domain
- Parse the URI in the Activity and delegate to the navigation system — don't handle in the manifest
- Validate all incoming URI parameters — treat them as untrusted input
- Test deep links with `adb` before release

---

## Manifest Intent Filters

```xml
<!-- AndroidManifest.xml -->
<activity
    android:name=".MainActivity"
    android:exported="true"
    android:launchMode="singleTask">

    <!-- App Link (HTTPS — requires domain verification) -->
    <intent-filter android:autoVerify="true">
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data
            android:scheme="https"
            android:host="example.com"
            android:pathPrefix="/users" />
    </intent-filter>

    <!-- Custom scheme (no verification required) -->
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="myapp" />
    </intent-filter>

</activity>
```

---

## App Links Verification (assetlinks.json)

```json
// Hosted at: https://example.com/.well-known/assetlinks.json
[{
  "relation": ["delegate_permission/common.handle_all_urls"],
  "target": {
    "namespace": "android_app",
    "package_name": "com.example.app",
    "sha256_cert_fingerprints": [
      "AA:BB:CC:DD:..."
    ]
  }
}]
```

```bash
# Get your app's SHA-256 fingerprint
keytool -list -v -keystore release.keystore -alias my_alias
```

---

## Handling the Deep Link in Activity

```kotlin
// ✅ Navigation Compose handles deep links automatically
// Just ensure your NavHost is set up with deep links on destinations (see deep-navigation skill)

// ✅ Manual handling if needed
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        handleDeepLinkIntent(intent)
        setContent { AppNavHost() }
    }

    override fun onNewIntent(intent: Intent) {
        super.onNewIntent(intent)
        handleDeepLinkIntent(intent)
    }

    private fun handleDeepLinkIntent(intent: Intent?) {
        val uri = intent?.data ?: return
        // Navigation Compose picks this up automatically via NavHost
        // Manual parsing only needed outside of Navigation Compose
    }
}
```

---

## URI Validation

```kotlin
// ✅ Validate and sanitize deep link parameters
fun parseUserDeepLink(uri: Uri): String? {
    if (uri.host != "example.com") return null
    if (!uri.path.orEmpty().startsWith("/users/")) return null

    val userId = uri.lastPathSegment ?: return null
    if (userId.isBlank() || userId.length > 64) return null
    if (!userId.matches(Regex("[a-zA-Z0-9_-]+"))) return null

    return userId
}
```

---

## Testing Deep Links

```bash
# Test App Link
adb shell am start \
  -W -a android.intent.action.VIEW \
  -d "https://example.com/users/123" \
  com.example.app

# Test custom scheme
adb shell am start \
  -W -a android.intent.action.VIEW \
  -d "myapp://users/123" \
  com.example.app

# Verify App Link association
adb shell pm get-app-links com.example.app
```

---

## Deep Link from Notification

```kotlin
// ✅ Build notification with deep link PendingIntent
fun buildDeepLinkPendingIntent(context: Context, userId: String): PendingIntent {
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

## Anti-Patterns

- Using only custom scheme deep links for sensitive actions — other apps can register the same scheme
- Not verifying App Links — browser asks user to choose the app instead of opening directly
- Not validating URI parameters — path traversal or injection via malicious links
- Not handling `onNewIntent` — deep links sent to an already-running activity are ignored
- Trusting all incoming URIs without validation — treat as untrusted user input

---

## Related Skills
- `deep-navigation` — wiring deep links into Navigation Compose destinations
- `notification` — launching deep links from notifications
- `manifest` — intent filter configuration
- `nested-navigation` — deep linking into nested navigation graphs
