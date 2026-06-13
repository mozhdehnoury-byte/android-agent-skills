---
name: secure-networking
description: >
  Secure networking practices for Android apps.
  Load this skill when configuring TLS, implementing certificate pinning,
  setting up Network Security Config, handling authentication headers securely,
  preventing cleartext traffic, or hardening OkHttp/Retrofit for production.
---

# Secure Networking

## Overview
Secure networking ensures that data in transit is protected from interception and tampering. The key concerns are: enforcing TLS, pinning certificates against MITM attacks, protecting auth credentials in requests, and preventing accidental cleartext traffic.

---

## Core Principles

- **TLS only** — no cleartext HTTP in production; enforce via Network Security Config
- **Certificate pinning** for high-security endpoints — prevents MITM even with compromised CAs
- **Auth tokens in headers** — never in URL query parameters (logged by servers/proxies)
- **Token refresh** handled transparently via OkHttp Authenticator
- **Sensitive headers stripped** on redirects — never forward auth to different domains

---

## Network Security Config

```xml
<!-- res/xml/network_security_config.xml -->
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>

    <!-- ✅ Block all cleartext traffic in production -->
    <base-config cleartextTrafficPermitted="false">
        <trust-anchors>
            <certificates src="system" />
        </trust-anchors>
    </base-config>

    <!-- ✅ Allow cleartext only for debug builds (localhost) -->
    <debug-overrides>
        <trust-anchors>
            <certificates src="system" />
            <certificates src="user" />   <!-- allows Charles/Burp on debug -->
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

## OkHttp Security Configuration

```kotlin
// ✅ Production OkHttpClient
fun buildSecureOkHttpClient(
    authInterceptor: AuthInterceptor,
    certificatePinner: CertificatePinner? = null
): OkHttpClient {
    return OkHttpClient.Builder()
        .connectTimeout(30, TimeUnit.SECONDS)
        .readTimeout(30, TimeUnit.SECONDS)
        .writeTimeout(30, TimeUnit.SECONDS)
        .addInterceptor(authInterceptor)
        .apply { certificatePinner?.let { certificatePinner(it) } }
        .authenticator(TokenRefreshAuthenticator())
        .build()
}

// ✅ Auth interceptor — adds token to every request
class AuthInterceptor @Inject constructor(
    private val securePreferences: SecurePreferences
) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val token = securePreferences.authToken
        val request = if (token != null) {
            chain.request().newBuilder()
                .header("Authorization", "Bearer $token")
                .build()
        } else {
            chain.request()
        }
        return chain.proceed(request)
    }
}

// ✅ Token refresh on 401
class TokenRefreshAuthenticator @Inject constructor(
    private val securePreferences: SecurePreferences,
    private val authApi: AuthApiService
) : Authenticator {
    override fun authenticate(route: Route?, response: Response): Request? {
        // Prevent infinite loops
        if (response.request.header("Authorization") == null) return null
        if (responseCount(response) >= 2) return null

        val refreshToken = securePreferences.refreshToken ?: return null

        val newToken = runBlocking {
            authApi.refreshToken(refreshToken).getOrNull()
        } ?: return null

        securePreferences.authToken = newToken.accessToken
        securePreferences.refreshToken = newToken.refreshToken

        return response.request.newBuilder()
            .header("Authorization", "Bearer ${newToken.accessToken}")
            .build()
    }

    private fun responseCount(response: Response): Int {
        var count = 1
        var prior = response.priorResponse
        while (prior != null) { count++; prior = prior.priorResponse }
        return count
    }
}
```

---

## Certificate Pinning

```kotlin
// ✅ Certificate pinning via OkHttp
fun buildCertificatePinner(): CertificatePinner {
    return CertificatePinner.Builder()
        // Pin the leaf certificate + backup pin for rotation
        .add("api.example.com", "sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=")
        .add("api.example.com", "sha256/BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB=") // backup
        .build()
}

// ✅ Getting the pin value for your certificate:
// $ openssl s_client -connect api.example.com:443 | openssl x509 -pubkey -noout |
//   openssl pkey -pubin -outform der | openssl dgst -sha256 -binary | base64
```

---

## Logging Interceptor (Debug Only)

```kotlin
// ✅ Never log request bodies in release — may contain tokens or PII
val loggingInterceptor = HttpLoggingInterceptor().apply {
    level = if (BuildConfig.DEBUG) {
        HttpLoggingInterceptor.Level.BODY
    } else {
        HttpLoggingInterceptor.Level.NONE   // ✅ no logging in production
    }
}

// ✅ Redact sensitive headers in logs
val loggingInterceptor = HttpLoggingInterceptor().apply {
    level = HttpLoggingInterceptor.Level.HEADERS
    redactHeader("Authorization")
    redactHeader("Cookie")
    redactHeader("Set-Cookie")
}
```

---

## Hilt Setup

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {

    @Provides
    @Singleton
    fun provideCertificatePinner(): CertificatePinner = buildCertificatePinner()

    @Provides
    @Singleton
    fun provideOkHttpClient(
        authInterceptor: AuthInterceptor,
        certificatePinner: CertificatePinner
    ): OkHttpClient = buildSecureOkHttpClient(authInterceptor, certificatePinner)

    @Provides
    @Singleton
    fun provideRetrofit(okHttpClient: OkHttpClient): Retrofit =
        Retrofit.Builder()
            .baseUrl(BuildConfig.API_BASE_URL)
            .client(okHttpClient)
            .addConverterFactory(json.asConverterFactory("application/json".toMediaType()))
            .build()
}
```

---

## Anti-Patterns

- Passing auth tokens as URL query params — visible in server logs, browser history, proxies
- `cleartextTrafficPermitted="true"` in production manifest — allows HTTP interception
- Logging request/response bodies in release — leaks tokens and PII to logcat
- No token refresh logic — forces users to re-login on every token expiry
- Trusting user-installed CAs in production — enables corporate proxy / MITM attacks
- Hard-coding API keys or secrets in source code — use BuildConfig from CI environment variables

---

## Related Skills
- `certificate-pinning` — in-depth pinning setup and rotation strategy
- `retrofit` — Retrofit configuration
- `okhttp` — OkHttp interceptors and configuration
- `authentication` — auth flow and token management
- `network-security-config` — XML-based network security policy
