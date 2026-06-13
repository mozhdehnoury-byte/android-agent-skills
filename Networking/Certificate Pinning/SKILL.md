---
name: certificate-pinning
description: >
  SSL/TLS certificate pinning for Android to prevent MITM attacks.
  Load this skill when implementing certificate pinning via OkHttp,
  configuring network security config, managing pin rotation,
  or handling pinning failures gracefully.
---

# Certificate Pinning

## Overview
Certificate pinning ensures the app only communicates with servers whose TLS certificate matches a known pinned value. This prevents man-in-the-middle attacks even if a rogue CA is trusted by the OS. Android supports pinning via OkHttp's `CertificatePinner` or the Network Security Config XML.

---

## Core Principles

- Always pin the **backup pin** alongside the primary — enables rotation without a forced update
- Use **public key pinning** (SPKI hash), not certificate pinning — survives certificate renewal
- Never pin in debug builds — breaks traffic inspection with Charles/Proxyman
- Have a **pin rotation plan** before shipping — expired pins = broken app
- Test pinning failures explicitly — they should fail closed, not open

---

## OkHttp Certificate Pinner

```kotlin
// ✅ CertificatePinner with primary + backup pin
@Provides
@Singleton
fun provideCertificatePinner(): CertificatePinner {
    return CertificatePinner.Builder()
        .add(
            "api.example.com",
            "sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=",  // primary
            "sha256/BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB="   // backup
        )
        .add(
            "cdn.example.com",
            "sha256/CCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCC=",
            "sha256/DDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDD="
        )
        .build()
}

// ✅ Attach to OkHttpClient — skip in debug
@Provides
@Singleton
fun provideOkHttpClient(
    certificatePinner: CertificatePinner,
    @IsDebug isDebug: Boolean
): OkHttpClient {
    return OkHttpClient.Builder()
        .apply {
            if (!isDebug) certificatePinner(certificatePinner)
        }
        .build()
}
```

---

## Network Security Config (XML)

```xml
<!-- res/xml/network_security_config.xml -->
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <domain-config>
        <domain includeSubdomains="true">api.example.com</domain>
        <pin-set expiration="2026-01-01">
            <!-- Primary pin -->
            <pin digest="SHA-256">AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=</pin>
            <!-- Backup pin -->
            <pin digest="SHA-256">BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB=</pin>
        </pin-set>
    </domain-config>

    <!-- Debug: allow cleartext and trust user CAs -->
    <debug-overrides>
        <trust-anchors>
            <certificates src="user"/>
            <certificates src="system"/>
        </trust-anchors>
    </debug-overrides>
</network-security-config>
```

```xml
<!-- AndroidManifest.xml -->
<application
    android:networkSecurityConfig="@xml/network_security_config"
    ...>
```

---

## Extracting the Pin

```bash
# Extract SPKI hash from a live server
openssl s_client -connect api.example.com:443 -servername api.example.com \
  </dev/null 2>/dev/null \
  | openssl x509 -pubkey -noout \
  | openssl pkey -pubin -outform DER \
  | openssl dgst -sha256 -binary \
  | openssl enc -base64

# Or from a certificate file
openssl x509 -in cert.pem -pubkey -noout \
  | openssl pkey -pubin -outform DER \
  | openssl dgst -sha256 -binary \
  | openssl enc -base64
```

---

## Pinning Failure Handling

```kotlin
// ✅ Catch SSLPeerUnverifiedException — pinning failure
suspend fun safeApiCall(call: suspend () -> T): Result<T> = runCatching {
    call()
}.recoverCatching { throwable ->
    when (throwable) {
        is SSLPeerUnverifiedException -> throw CertificatePinningException(
            "Certificate pinning failure — possible MITM attack"
        )
        else -> throw throwable
    }
}

class CertificatePinningException(message: String) : SecurityException(message)
```

---

## Pin Rotation Strategy

```
1. Generate new certificate/key pair
2. Extract SPKI hash of new cert → backup pin
3. Ship app update with both old (primary) + new (backup) pins
4. Wait for adoption (monitor crash-free rate)
5. Deploy new certificate to server
6. Ship next app update with new cert as primary + newer backup
```

---

## Anti-Patterns

- Pinning in debug builds — breaks all proxy-based debugging tools
- Pinning without a backup pin — one certificate renewal breaks the app
- Catching `SSLPeerUnverifiedException` silently — must fail closed, log the incident
- Using certificate pinning (full cert hash) instead of public key pinning — breaks on renewal
- Not setting an `expiration` in Network Security Config — no forced rotation reminder

---

## Related Skills
- `okhttp` — OkHttp client where pinner is attached
- `secure-networking` — broader TLS and network security
- `network-security-config` — XML-based network security configuration
- `certificate-transparency` — complementary server validation mechanism
