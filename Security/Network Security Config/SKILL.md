---
name: network-security-config
description: >
  Android Network Security Config for controlling TLS and cleartext traffic.
  Load this skill when configuring network security policy via XML,
  blocking cleartext HTTP, trusting custom CAs, defining per-domain rules,
  or allowing debug-only network policies.
---

# Network Security Config

## Overview
Network Security Config is an Android XML-based mechanism for defining a per-app network security policy without code changes. It controls which CAs are trusted, whether cleartext (HTTP) traffic is allowed, certificate pinning, and debug-only overrides. It applies to all HTTP(S) connections made through standard Android networking APIs.

---

## Core Principles

- **Declare in manifest** — `android:networkSecurityConfig` on `<application>`
- **Default behavior** (no config): cleartext allowed on API < 28, blocked on API 28+
- **Block cleartext explicitly** — don't rely on the default
- **Debug overrides** are only active in debug builds — safe to include user CAs there
- **Per-domain rules** override the base config for specific hosts

---

## Full Configuration Reference

```xml
<!-- res/xml/network_security_config.xml -->
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>

    <!-- ✅ Base config — applies to all connections not covered by domain-config -->
    <base-config cleartextTrafficPermitted="false">
        <trust-anchors>
            <certificates src="system" />   <!-- trust system CAs only -->
        </trust-anchors>
    </base-config>

    <!-- ✅ Per-domain config — overrides base-config for specific hosts -->
    <domain-config cleartextTrafficPermitted="false">
        <domain includeSubdomains="true">api.example.com</domain>
        <trust-anchors>
            <certificates src="system" />
        </trust-anchors>
        <!-- Certificate pinning (optional — prefer OkHttp pinning for flexibility) -->
        <pin-set expiration="2026-01-01">
            <pin digest="SHA-256">AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=</pin>
            <pin digest="SHA-256">BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB=</pin>
        </pin-set>
    </domain-config>

    <!-- ✅ Allow cleartext for specific internal/local hosts only -->
    <domain-config cleartextTrafficPermitted="true">
        <domain includeSubdomains="false">10.0.2.2</domain>   <!-- emulator localhost -->
        <domain includeSubdomains="false">192.168.1.1</domain>
    </domain-config>

    <!-- ✅ Debug overrides — ONLY active in debug builds, ignored in release -->
    <debug-overrides>
        <trust-anchors>
            <certificates src="system" />
            <certificates src="user" />     <!-- allows Charles/Burp SSL proxy on debug -->
        </trust-anchors>
    </debug-overrides>

</network-security-config>
```

---

## AndroidManifest.xml

```xml
<application
    android:networkSecurityConfig="@xml/network_security_config"
    android:usesCleartextTraffic="false"   <!-- ✅ belt-and-suspenders for API < 24 -->
    ...>
```

---

## Common Configurations

```xml
<!-- ✅ Minimal production config — block all cleartext, system CAs only -->
<network-security-config>
    <base-config cleartextTrafficPermitted="false">
        <trust-anchors>
            <certificates src="system" />
        </trust-anchors>
    </base-config>
    <debug-overrides>
        <trust-anchors>
            <certificates src="system" />
            <certificates src="user" />
        </trust-anchors>
    </debug-overrides>
</network-security-config>

<!-- ✅ Trust a custom/private CA (e.g. internal enterprise API) -->
<network-security-config>
    <base-config cleartextTrafficPermitted="false">
        <trust-anchors>
            <certificates src="system" />
            <certificates src="@raw/my_ca_cert" />   <!-- place CA cert in res/raw/ -->
        </trust-anchors>
    </base-config>
</network-security-config>

<!-- ✅ Override for one domain with stricter rules -->
<network-security-config>
    <base-config cleartextTrafficPermitted="false">
        <trust-anchors>
            <certificates src="system" />
        </trust-anchors>
    </base-config>
    <domain-config>
        <domain includeSubdomains="true">payments.example.com</domain>
        <pin-set expiration="2026-06-01">
            <pin digest="SHA-256">primaryPinBase64==</pin>
            <pin digest="SHA-256">backupPinBase64==</pin>
        </pin-set>
    </domain-config>
</network-security-config>
```

---

## Certificate Sources

| `src` value | Meaning |
|---|---|
| `system` | Android system CA store |
| `user` | User-installed CAs (avoid in production) |
| `@raw/cert_file` | Bundled CA certificate in `res/raw/` |

---

## Pin Expiration

```xml
<!-- ✅ Always set pin expiration — prevents permanent lockout if pin rotation fails -->
<pin-set expiration="2026-06-01">
    <pin digest="SHA-256">primaryPinBase64==</pin>
    <pin digest="SHA-256">backupPinBase64==</pin>
</pin-set>

<!-- After expiration date, pinning is disabled — connections proceed without pinning -->
<!-- ✅ Better to use OkHttp certificate pinning for runtime flexibility -->
```

---

## Verifying Config is Applied

```bash
# ✅ Test that cleartext is blocked
adb shell am start -a android.intent.action.VIEW \
  -d http://example.com com.example.app

# Expected: NetworkSecurityPolicy blocks the connection
# Logcat: CLEARTEXT communication to example.com not permitted by network security policy

# ✅ Verify config with Network Inspector in Android Studio
# View > Tool Windows > App Inspection > Network Inspector
```

---

## Anti-Patterns

- `cleartextTrafficPermitted="true"` in base-config for production — allows HTTP interception
- `<certificates src="user" />` in base-config (not debug-overrides) — allows proxy attacks
- No `debug-overrides` block — forces developers to modify production config for local testing
- Embedding private keys in `res/raw/` — only embed **CA certificates**, never private keys
- Relying solely on XML pinning — prefer OkHttp pinning which supports easier rotation

---

## Related Skills
- `secure-networking` — overall network security setup
- `certificate-pinning` — OkHttp-based certificate pinning
- `certificate-transparency` — CT enforcement
- `okhttp` — OkHttp configuration
