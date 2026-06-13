---
name: share-intent
description: >
  Sharing content from Android apps using Intent and ShareSheet.
  Load this skill when sharing text, URLs, images, or files with other
  apps, customizing the share sheet, receiving shared content,
  or implementing the Android Sharesheet API.
---

# Share Intent

## Overview
Android's share system lets apps send and receive content through `Intent.ACTION_SEND`. The system displays a share sheet with compatible apps. Modern apps use the `ShareCompat` API or `rememberLauncherForActivityResult` in Compose for a cleaner integration.

---

## Core Principles

- Use `ShareCompat.IntentBuilder` — not raw `Intent` — for outgoing shares
- Always use `FileProvider` for sharing files — direct `file://` URIs are blocked on Android 7+
- Handle incoming share intents in `onNewIntent` as well as `onCreate`
- Validate incoming share content — treat it as untrusted input
- Use `ChooserIntent` to customize the share sheet title

---

## Sharing Text and URLs

```kotlin
// ✅ Share plain text or URL
fun shareText(context: Context, text: String, title: String = "Share via") {
    val intent = ShareCompat.IntentBuilder(context)
        .setType("text/plain")
        .setText(text)
        .setChooserTitle(title)
        .createChooserIntent()

    context.startActivity(intent)
}

// ✅ Share URL with subject
fun shareUrl(context: Context, url: String, subject: String) {
    val intent = ShareCompat.IntentBuilder(context)
        .setType("text/plain")
        .setText(url)
        .setSubject(subject)
        .setChooserTitle("Share link via")
        .createChooserIntent()

    context.startActivity(intent)
}
```

---

## Sharing Images

```kotlin
// ✅ Share image via FileProvider
fun shareImage(context: Context, imageFile: File) {
    val uri = FileProvider.getUriForFile(
        context,
        "${context.packageName}.fileprovider",
        imageFile
    )

    val intent = ShareCompat.IntentBuilder(context)
        .setType("image/jpeg")
        .setStream(uri)
        .setChooserTitle("Share image via")
        .createChooserIntent()
        .addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)

    context.startActivity(intent)
}
```

---

## FileProvider Setup

```xml
<!-- AndroidManifest.xml -->
<provider
    android:name="androidx.core.content.FileProvider"
    android:authorities="${applicationId}.fileprovider"
    android:exported="false"
    android:grantUriPermissions="true">
    <meta-data
        android:name="android.support.FILE_PROVIDER_PATHS"
        android:resource="@xml/file_paths" />
</provider>
```

```xml
<!-- res/xml/file_paths.xml -->
<paths>
    <cache-path name="cache" path="." />
    <files-path name="files" path="." />
    <external-cache-path name="external_cache" path="." />
</paths>
```

---

## Sharing in Compose

```kotlin
// ✅ Share from Compose
@Composable
fun ShareButton(text: String) {
    val context = LocalContext.current

    Button(onClick = {
        shareText(context, text)
    }) {
        Icon(Icons.Default.Share, contentDescription = null)
        Spacer(Modifier.width(8.dp))
        Text("Share")
    }
}
```

---

## Receiving Shared Content

```kotlin
// ✅ Handle incoming share in Activity
class MainActivity : ComponentActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        handleIncomingShare(intent)
        setContent { AppContent() }
    }

    override fun onNewIntent(intent: Intent) {
        super.onNewIntent(intent)
        handleIncomingShare(intent)
    }

    private fun handleIncomingShare(intent: Intent?) {
        if (intent?.action != Intent.ACTION_SEND) return

        when {
            intent.type == "text/plain" -> {
                val text = intent.getStringExtra(Intent.EXTRA_TEXT) ?: return
                // handle text
            }
            intent.type?.startsWith("image/") == true -> {
                val uri = intent.getParcelableExtra<Uri>(Intent.EXTRA_STREAM) ?: return
                // handle image URI
            }
        }
    }
}

// Register in manifest to receive shares
// <intent-filter>
//     <action android:name="android.intent.action.SEND" />
//     <category android:name="android.intent.category.DEFAULT" />
//     <data android:mimeType="text/plain" />
//     <data android:mimeType="image/*" />
// </intent-filter>
```

---

## Anti-Patterns

- Using `file://` URI directly in share intent — blocked on Android 7+ (FileUriExposedException)
- Not granting `FLAG_GRANT_READ_URI_PERMISSION` for FileProvider URIs — receiving app can't read
- Not validating incoming shared content — treat as untrusted
- Using `startActivity` without a chooser for text sharing — opens default app silently

---

## Related Skills
- `filesystem` — writing files to share via FileProvider
- `manifest` — intent filter for receiving shares
- `deep-link` — handling incoming intents
