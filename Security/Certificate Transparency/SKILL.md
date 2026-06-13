---
name: certificate-transparency
description: >
  Certificate Transparency (CT) enforcement for Android apps.
  Load this skill when enforcing CT log verification for TLS connections,
  understanding CT requirements on Android, or configuring OkHttp to
  verify certificates are publicly logged.
---

# Certificate Transparency

## Overview
Certificate Transparency is a public logging framework that requires CAs to log every issued certificate to a public CT log. This makes mis-issued certificates detectable. On Android, CT enforcement is built into the OS (Android 7+) for system CAs, and can be enforced at the app level via OkHttp's `certificatetransparency` library for additional guarantees.

---

## Core Principles

- **CT is enforced by Android OS** for system CA-signed certificates since Android 7+
- App-level CT enforcement provides stricter, per-host control
- CT does not replace certificate pinning — they address different threats
- CT verifies the certificate is in public logs — pinning verifies it's a specific certificate
- Use both for highest security: CT catches mis-issuance; pinning catches MITM

---

## Android OS Built-in CT

```
Android 7.0+ enforces CT for:
  - Certificates signed by public CAs trusted by the system
  - Connections where the certificate chain includes a public CA

Android does NOT enforce CT for:
  - Certificates in the user certificate store
  - Privately issued certificates
  - Certificates added via Network Security Config
```

---

## App-Level CT with OkHttp (certificatetransparency-android)

```kotlin
// dependency: com.appmattus.certificatetransparency:certificatetransparency-android

// ✅ Add CT verification to OkHttp
fun buildCtEnabledOkHttpClient(): OkHttpClient {
    val ctInterceptor = certificateTransparencyInterceptor {
        // ✅ Enforce CT for all hosts
        +("*.*")

        // ✅ Or enforce only for specific hosts
        +("api.example.com")
        +("*.example.com")

        // ✅ Exclude hosts where CT is not applicable (e.g. internal/dev)
        -("internal.dev.example.com")
    }

    return OkHttpClient.Builder()
        .addNetworkInterceptor(ctInterceptor)
        .build()
}

// ✅ Alternative: use CTHostnameVerifier
fun buildCtEnabledClient(): OkHttpClient {
    val ctHostnameVerifier = certificateTransparencyHostnameVerifier {
        +("api.example.com")
    }

    return OkHttpClient.Builder()
        .hostnameVerifier(ctHostnameVerifier)
        .build()
}
```

---

## Network Security Config + CT

```xml
<!-- res/xml/network_security_config.xml -->
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <base-config cleartextTrafficPermitted="false">
        <trust-anchors>
            <!-- ✅ System CAs — Android 7+ enforces CT for these -->
            <certificates src="system" />
        </trust-anchors>
    </base-config>

    <!-- ✅ For debug: allow user-added CAs without CT enforcement -->
    <debug-overrides>
        <trust-anchors>
            <certificates src="system" />
            <certificates src="user" />
        </trust-anchors>
    </debug-overrides>
</network-security-config>
```

---

## CT vs Certificate Pinning

| | Certificate Transparency | Certificate Pinning |
|---|---|---|
| What it verifies | Certificate is publicly logged | Certificate matches expected value |
| Protects against | Mis-issued certificates by CAs | MITM with any valid certificate |
| Maintenance | Low (automatic) | High (pin rotation required) |
| Failure mode | Rejects unlogged certificates | Rejects wrong certificates |
| Use for | All public API connections | High-security endpoints |

---

## Hilt Integration

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {

    @Provides
    @Singleton
    fun provideOkHttpClient(
        authInterceptor: AuthInterceptor,
        certificatePinner: CertificatePinner
    ): OkHttpClient {
        val ctInterceptor = certificateTransparencyInterceptor {
            +("api.example.com")
        }

        return OkHttpClient.Builder()
            .addInterceptor(authInterceptor)
            .addNetworkInterceptor(ctInterceptor)   // ✅ CT after TLS handshake
            .certificatePinner(certificatePinner)
            .build()
    }
}
```

---

## Anti-Patterns

- Assuming OS-level CT is sufficient for all use cases — app-level gives per-host control
- Disabling CT verification in production builds — removes mis-issuance protection
- Using CT as a replacement for certificate pinning — they complement each other
- Not testing CT enforcement — a misconfigured CT check can block all connections silently

---

## Related Skills
- `certificate-pinning` — pinning specific certificates against MITM
- `secure-networking` — overall TLS and network security
- `network-security-config` — Android XML-based network policy
- `okhttp` — OkHttp configuration and interceptors
