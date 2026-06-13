---
name: retrofit
description: >
  Retrofit HTTP client setup and usage for Android.
  Load this skill when setting up Retrofit, defining API interfaces,
  configuring converters, adding interceptors, handling responses,
  or integrating Retrofit with Kotlinx Serialization and OkHttp.
---

# Retrofit

## Overview

Retrofit is a type-safe HTTP client for Android that turns API interfaces into callable Kotlin functions. It integrates with OkHttp for the underlying HTTP layer and Kotlinx Serialization for JSON parsing.

---

## Core Principles

- Define one `Retrofit` instance per base URL — use DI to provide it
- API interfaces return `Result<T>` or a sealed type — never raw response objects
- Never call Retrofit from UI or ViewModel directly — go through Repository
- Use `suspend` functions for all API calls — no callbacks or `Call<T>`
- Handle HTTP errors explicitly — don't let them surface as exceptions silently

---

## Setup

```toml
# libs.versions.toml
[versions]
retrofit = "2.11.0"
okhttp = "4.12.0"

[libraries]
retrofit = { module = "com.squareup.retrofit2:retrofit", version.ref = "retrofit" }
retrofit-kotlinx-serialization = { module = "com.jakewharton.retrofit:retrofit2-kotlinx-serialization-converter", version.ref = "retrofit" }
okhttp = { module = "com.squareup.okhttp3:okhttp", version.ref = "okhttp" }
okhttp-logging = { module = "com.squareup.okhttp3:logging-interceptor", version.ref = "okhttp" }
```

```kotlin
// build.gradle.kts
dependencies {
    implementation(libs.retrofit)
    implementation(libs.retrofit.kotlinx.serialization)
    implementation(libs.okhttp)
    implementation(libs.okhttp.logging)
}
```

---

## Retrofit Instance

```kotlin
// ✅ Single Retrofit instance provided via DI
@Provides
@Singleton
fun provideRetrofit(okHttpClient: OkHttpClient, json: Json): Retrofit {
    return Retrofit.Builder()
        .baseUrl(BuildConfig.BASE_URL)
        .client(okHttpClient)
        .addConverterFactory(json.asConverterFactory("application/json".toMediaType()))
        .build()
}

@Provides
@Singleton
fun provideOkHttpClient(
    authInterceptor: AuthInterceptor,
    loggingInterceptor: HttpLoggingInterceptor
): OkHttpClient {
    return OkHttpClient.Builder()
        .addInterceptor(authInterceptor)
        .addInterceptor(loggingInterceptor)
        .connectTimeout(30, TimeUnit.SECONDS)
        .readTimeout(30, TimeUnit.SECONDS)
        .writeTimeout(30, TimeUnit.SECONDS)
        .build()
}

@Provides
@Singleton
fun provideLoggingInterceptor(): HttpLoggingInterceptor {
    return HttpLoggingInterceptor().apply {
        level = if (BuildConfig.DEBUG)
            HttpLoggingInterceptor.Level.BODY
        else
            HttpLoggingInterceptor.Level.NONE
    }
}
```

---

## API Interface

```kotlin
// ✅ Suspend functions, typed responses
interface UserApi {

    @GET("users")
    suspend fun getUsers(): List<UserDto>

    @GET("users/{id}")
    suspend fun getUser(@Path("id") id: String): UserDto

    @POST("users")
    suspend fun createUser(@Body body: CreateUserRequest): UserDto

    @PUT("users/{id}")
    suspend fun updateUser(
        @Path("id") id: String,
        @Body body: UpdateUserRequest
    ): UserDto

    @DELETE("users/{id}")
    suspend fun deleteUser(@Path("id") id: String)

    @GET("users")
    suspend fun searchUsers(
        @Query("q") query: String,
        @Query("page") page: Int = 1,
        @Query("limit") limit: Int = 20
    ): PagedResponse<UserDto>
}
```

---

## Response Handling in Repository

```kotlin
// ✅ Wrap API calls in runCatching — map to domain Result
class UserRepositoryImpl @Inject constructor(
    private val api: UserApi,
    private val mapper: UserMapper
) : UserRepository {

    override suspend fun getUser(id: String): Result<User> = runCatching {
        val dto = api.getUser(id)
        mapper.toDomain(dto)
    }

    override suspend fun getUsers(): Result<List<User>> = runCatching {
        api.getUsers().map { mapper.toDomain(it) }
    }
}
```

---

## Auth Interceptor

```kotlin
// ✅ Attach token via interceptor — not in each API call
class AuthInterceptor @Inject constructor(
    private val tokenProvider: TokenProvider
) : Interceptor {

    override fun intercept(chain: Interceptor.Chain): Response {
        val token = tokenProvider.getToken()
        val request = if (token != null) {
            chain.request().newBuilder()
                .addHeader("Authorization", "Bearer $token")
                .build()
        } else {
            chain.request()
        }
        return chain.proceed(request)
    }
}
```

---

## HTTP Error Handling

```kotlin
// ✅ Custom call adapter or extension for HTTP errors
suspend fun <T> safeApiCall(call: suspend () -> T): Result<T> = runCatching {
    call()
}.recoverCatching { throwable ->
    when (throwable) {
        is HttpException -> throw ApiException(
            code = throwable.code(),
            message = throwable.response()?.errorBody()?.string() ?: "HTTP error"
        )
        is IOException -> throw NetworkException("No internet connection")
        else -> throw throwable
    }
}

// ✅ Usage in repository
override suspend fun getUser(id: String): Result<User> =
    safeApiCall { api.getUser(id) }
        .map { mapper.toDomain(it) }
```

---

## Multipart / File Upload

```kotlin
// ✅ Upload file with multipart
@Multipart
@POST("users/{id}/avatar")
suspend fun uploadAvatar(
    @Path("id") id: String,
    @Part avatar: MultipartBody.Part
): AvatarDto

// ✅ Build MultipartBody.Part from file
fun File.toMultipartPart(partName: String): MultipartBody.Part {
    val requestBody = asRequestBody("image/*".toMediaType())
    return MultipartBody.Part.createFormData(partName, name, requestBody)
}
```

---

## Anti-Patterns

- Creating Retrofit instance per API call — expensive, use singleton
- Returning `Response<T>` from API interface — wrap in `Result` at the repository level
- Using `Call<T>` instead of `suspend` — unnecessary callback complexity
- Adding auth token manually in each API function — use interceptor
- Catching exceptions in ViewModel — handle in repository or use case
- Using `Gson` converter — use Kotlinx Serialization

---

## Related Skills

- `okhttp` — OkHttp client, interceptors, and connection config
- `serialization` — JSON parsing with Kotlinx Serialization
- `authentication` — token management and refresh
- `retry-backoff` — retry logic for failed requests
- `dto-mapping` — mapping API responses to domain models
