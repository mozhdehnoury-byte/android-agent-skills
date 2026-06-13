---
name: clipboard
description: >
  Clipboard read and write operations in Android.
  Load this skill when copying text or URIs to the clipboard,
  reading clipboard content, showing copy confirmation feedback,
  handling clipboard access restrictions on Android 10+,
  or implementing paste functionality.
---

# Clipboard

## Overview
The Android clipboard allows apps to copy and paste text, URIs, and other data. Since Android 10, apps can only read the clipboard when they are in the foreground. Android 13 shows a visual confirmation when an app reads clipboard content. Best practice is to write to clipboard and show immediate in-app feedback rather than reading silently.

---

## Core Principles

- Always show **visual feedback** after copying — toast or snackbar
- Never read clipboard silently in background — it's restricted on Android 10+
- Use `ClipData.newPlainText` for text — `ClipData.newUri` for URIs
- On Android 13+, the system shows a clipboard toast automatically — avoid double-toasting
- Sensitive content (passwords, tokens) should set `ClipDescription.EXTRA_IS_SENSITIVE`

---

## Writing to Clipboard

```kotlin
// ✅ Copy text to clipboard
fun copyToClipboard(context: Context, text: String, label: String = "Copied text") {
    val clipboard = context.getSystemService(ClipboardManager::class.java)
    val clip = ClipData.newPlainText(label, text)
    clipboard.setPrimaryClip(clip)

    // ✅ Show feedback only on Android 12 and below
    // Android 13+ shows system toast automatically
    if (Build.VERSION.SDK_INT <= Build.VERSION_CODES.S_V2) {
        Toast.makeText(context, "Copied", Toast.LENGTH_SHORT).show()
    }
}

// ✅ Copy sensitive content (password, token)
fun copySensitiveToClipboard(context: Context, text: String) {
    val clipboard = context.getSystemService(ClipboardManager::class.java)
    val clip = ClipData.newPlainText("sensitive", text).apply {
        description.extras = PersistableBundle().apply {
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
                putBoolean(ClipDescription.EXTRA_IS_SENSITIVE, true)
            }
        }
    }
    clipboard.setPrimaryClip(clip)
}
```

---

## Reading from Clipboard

```kotlin
// ✅ Read clipboard — only when app is in foreground
fun readFromClipboard(context: Context): String? {
    val clipboard = context.getSystemService(ClipboardManager::class.java)
    val clip = clipboard.primaryClip ?: return null
    if (clip.itemCount == 0) return null
    return clip.getItemAt(0).coerceToText(context).toString()
}

// ✅ Check if clipboard has text before reading
fun hasTextInClipboard(context: Context): Boolean {
    val clipboard = context.getSystemService(ClipboardManager::class.java)
    return clipboard.hasPrimaryClip() &&
        clipboard.primaryClipDescription?.hasMimeType("text/*") == true
}
```

---

## Clipboard in Compose

```kotlin
// ✅ Copy on long press or button tap
@Composable
fun CopyableText(text: String) {
    val context = LocalContext.current
    val clipboardManager = LocalClipboardManager.current

    Text(
        text = text,
        modifier = Modifier.clickable {
            clipboardManager.setText(AnnotatedString(text))
            // Show feedback on Android 12 and below
            if (Build.VERSION.SDK_INT <= Build.VERSION_CODES.S_V2) {
                Toast.makeText(context, "Copied", Toast.LENGTH_SHORT).show()
            }
        }
    )
}

// ✅ Paste button in Compose
@Composable
fun PasteButton(onPaste: (String) -> Unit) {
    val clipboardManager = LocalClipboardManager.current

    Button(onClick = {
        val text = clipboardManager.getText()?.text ?: return@Button
        onPaste(text)
    }) {
        Text("Paste")
    }
}
```

---

## Clipboard Change Listener

```kotlin
// ✅ Listen for clipboard changes (e.g. autofill OTP)
class OtpViewModel : ViewModel() {

    private val clipboardListener = ClipboardManager.OnPrimaryClipChangedListener {
        val text = readFromClipboard(context) ?: return@OnPrimaryClipChangedListener
        if (text.matches(Regex("\\d{6}"))) {
            _otpInput.value = text
        }
    }

    fun registerClipboardListener(clipboard: ClipboardManager) {
        clipboard.addPrimaryClipChangedListener(clipboardListener)
    }

    fun unregisterClipboardListener(clipboard: ClipboardManager) {
        clipboard.removePrimaryClipChangedListener(clipboardListener)
    }
}
```

---

## Anti-Patterns

- Reading clipboard in the background — restricted on Android 10+, throws SecurityException
- Not showing feedback after copy — user doesn't know the action succeeded
- Showing a toast on Android 13+ — the system already shows one, causing double notification
- Storing passwords in clipboard without `EXTRA_IS_SENSITIVE` — shown in clipboard history
- Reading clipboard on every resume — intrusive, user sees system clipboard access notification

---

## Related Skills
- `compose` — `LocalClipboardManager` in Compose
- `share-intent` — sharing content with other apps instead of clipboard
