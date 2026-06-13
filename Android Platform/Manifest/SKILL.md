---
name: manifest
description: >
  Android Manifest configuration and best practices.
  Load this skill when editing AndroidManifest.xml, declaring components,
  configuring permissions, intent filters, or app-level attributes.
---

# Manifest

## Overview

The AndroidManifest.xml is the entry point configuration of every Android app. It declares all components, permissions, hardware requirements, and app-level behavior. Incorrect manifest configuration causes runtime crashes, security vulnerabilities, and Play Store rejections.

---

## Core Principles

- Declare only what the app **actually needs** — minimal permissions
- Never declare a component that isn't used — increases attack surface
- Always declare `android:exported` explicitly on components with intent filters
- Keep sensitive config (API keys, secrets) out of the manifest — use BuildConfig or encrypted storage
- Use `tools:` namespace for merge directives — never duplicate entries

---

## Application Block

```xml
<application
    android:name=".MyApplication"
    android:icon="@mipmap/ic_launcher"
    android:roundIcon="@mipmap/ic_launcher_round"
    android:label="@string/app_name"
    android:theme="@style/Theme.App"
    android:supportsRtl="true"
    android:allowBackup="false"
    android:fullBackupContent="false"
    android:dataExtractionRules="@xml/data_extraction_rules"
    android:networkSecurityConfig="@xml/network_security_config">

    <!-- components go here -->

</application>
```

---

## Activity Declaration

```xml
<!-- ✅ Main launcher activity -->
<activity
    android:name=".MainActivity"
    android:exported="true"
    android:windowSoftInputMode="adjustResize"
    android:configChanges="orientation|screenSize|keyboardHidden">
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
</activity>

<!-- ✅ Internal activity — not exported -->
<activity
    android:name=".feature.detail.DetailActivity"
    android:exported="false" />

<!-- ✅ Deep link support -->
<activity
    android:name=".MainActivity"
    android:exported="true">
    <intent-filter android:autoVerify="true">
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="https" android:host="example.com" />
    </intent-filter>
</activity>
```

---

## Permissions

```xml
<!-- ✅ Declare only what's needed -->
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />

<!-- ✅ Runtime permissions — still declare in manifest -->
<uses-permission android:name="android.permission.CAMERA" />
<uses-permission android:name="android.permission.READ_MEDIA_IMAGES" />

<!-- ✅ Restrict to specific OS version if not needed on older -->
<uses-permission
    android:name="android.permission.POST_NOTIFICATIONS"
    android:minSdkVersion="33" />

<!-- ✅ Remove permission added by a library -->
<uses-permission
    android:name="android.permission.READ_PHONE_STATE"
    tools:node="remove" />
```

### Permission Groups by Category

| Category      | Permissions                                                                    |
| ------------- | ------------------------------------------------------------------------------ |
| Network       | `INTERNET`, `ACCESS_NETWORK_STATE`, `CHANGE_NETWORK_STATE`                     |
| Location      | `ACCESS_FINE_LOCATION`, `ACCESS_COARSE_LOCATION`, `ACCESS_BACKGROUND_LOCATION` |
| Camera        | `CAMERA`                                                                       |
| Storage       | `READ_MEDIA_IMAGES`, `READ_MEDIA_VIDEO`, `READ_MEDIA_AUDIO`                    |
| Notifications | `POST_NOTIFICATIONS` (API 33+)                                                 |
| Bluetooth     | `BLUETOOTH_SCAN`, `BLUETOOTH_CONNECT` (API 31+)                                |

---

## Services

```xml
<!-- ✅ Foreground service -->
<service
    android:name=".service.DownloadService"
    android:exported="false"
    android:foregroundServiceType="dataSync" />

<!-- ✅ WorkManager worker — no declaration needed -->
<!-- Workers are not components and don't need manifest entries -->
```

---

## Receivers

```xml
<!-- ✅ Boot receiver -->
<receiver
    android:name=".receiver.BootReceiver"
    android:exported="true">
    <intent-filter>
        <action android:name="android.intent.action.BOOT_COMPLETED" />
    </intent-filter>
</receiver>

<!-- ✅ Internal receiver — not exported -->
<receiver
    android:name=".receiver.DownloadReceiver"
    android:exported="false" />
```

---

## Providers

```xml
<!-- ✅ FileProvider for sharing files -->
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

---

## Manifest Merge (Multi-Module)

```xml
<!-- ✅ Override a library's manifest entry -->
<activity
    android:name="com.library.SomeActivity"
    android:exported="false"
    tools:node="merge" />

<!-- ✅ Remove a component added by a library -->
<activity
    android:name="com.library.UnwantedActivity"
    tools:node="remove" />

<!-- ✅ Replace an attribute from a library -->
<application
    android:allowBackup="false"
    tools:replace="android:allowBackup" />
```

---

## Hardware Features

```xml
<!-- ✅ Declare required hardware -->
<uses-feature
    android:name="android.hardware.camera"
    android:required="true" />

<!-- ✅ Declare optional hardware — don't block install -->
<uses-feature
    android:name="android.hardware.camera.autofocus"
    android:required="false" />
```

---

## Anti-Patterns

- `android:exported` not set on components with intent-filters — security risk, required API 31+
- Storing API keys or secrets in `<meta-data>` — visible in decompiled APK
- Declaring `READ_EXTERNAL_STORAGE` on API 33+ — use `READ_MEDIA_*` instead
- Requesting `ACCESS_BACKGROUND_LOCATION` without foreground location first
- Declaring unused permissions — increases permissions dialog burden and audit surface
- Missing `android:supportsRtl="true"` — breaks RTL layouts
- `android:allowBackup="true"` without configuring backup rules — leaks sensitive data

---

## Related Skills

- `deep-link` — intent filter setup for deep links
- `foreground-service` — foreground service declaration
- `network-security-config` — network security configuration
- `resources` — resource references in manifest
