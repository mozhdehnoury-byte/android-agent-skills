---
name: c++
description: >
  C++ patterns and best practices for Android NDK development.
  Load this skill when writing C++ code for Android native libraries,
  using modern C++17 features, managing memory safely, handling
  concurrency in native code, or structuring native C++ modules.
---

# C++

## Overview
Modern C++ (C++17) for Android NDK focuses on safety, expressiveness, and performance. Android's Clang toolchain fully supports C++17. Using RAII, smart pointers, and the standard library eliminates most memory management bugs that plague native Android code.

---

## Core Principles

- **C++17** is the minimum target — use structured bindings, `std::optional`, `std::variant`
- **RAII everywhere** — resources are always tied to object lifetime
- **Smart pointers** instead of raw `new`/`delete` — `unique_ptr`, `shared_ptr`
- **`std::string` not `char*`** — except at JNI boundaries
- **No raw arrays** — use `std::vector`, `std::array`, `std::span`
- **const-correctness** — mark everything `const` that doesn't need to change

---

## Memory Management

```cpp
// ✅ unique_ptr — single ownership
std::unique_ptr<AESContext> ctx = std::make_unique<AESContext>(key);
ctx->encrypt(data);   // auto-deleted when ctx goes out of scope

// ✅ shared_ptr — shared ownership
std::shared_ptr<KeyStore> keyStore = std::make_shared<KeyStore>();
auto cipher = std::make_shared<AESCipher>(keyStore);

// ✅ RAII wrapper for Android resources
class AssetGuard {
public:
    explicit AssetGuard(AAsset* asset) : asset_(asset) {}
    ~AssetGuard() {
        if (asset_) AAsset_close(asset_);
    }
    // Disable copy
    AssetGuard(const AssetGuard&) = delete;
    AssetGuard& operator=(const AssetGuard&) = delete;
    // Enable move
    AssetGuard(AssetGuard&& other) noexcept : asset_(other.asset_) {
        other.asset_ = nullptr;
    }

    AAsset* get() const { return asset_; }
    bool valid() const { return asset_ != nullptr; }

private:
    AAsset* asset_;
};

// Usage
AssetGuard asset(AAssetManager_open(mgr, "config.json", AASSET_MODE_BUFFER));
if (!asset.valid()) return {};
// asset is auto-closed when AssetGuard leaves scope
```

---

## Modern C++17 Features

```cpp
// ✅ Structured bindings
std::map<std::string, int> scores = {{"ali", 90}, {"sara", 85}};
for (const auto& [name, score] : scores) {
    LOGI("%s: %d", name.c_str(), score);
}

// ✅ std::optional — nullable without pointers
std::optional<std::string> findKey(const std::string& id) {
    auto it = keyMap.find(id);
    if (it == keyMap.end()) return std::nullopt;
    return it->second;
}

auto key = findKey("user_key");
if (key.has_value()) {
    encrypt(data, *key);
}

// ✅ std::variant — type-safe union
using Result = std::variant<std::vector<uint8_t>, std::string>;

Result encrypt(const std::vector<uint8_t>& data) {
    if (data.empty()) return std::string("Empty input");
    return performEncryption(data);
}

auto result = encrypt(data);
std::visit([](auto&& val) {
    using T = std::decay_t<decltype(val)>;
    if constexpr (std::is_same_v<T, std::vector<uint8_t>>) {
        // success path
    } else {
        LOGE("Error: %s", val.c_str());
    }
}, result);

// ✅ if constexpr — compile-time branching
template<typename T>
void logValue(T value) {
    if constexpr (std::is_integral_v<T>) {
        LOGI("int: %lld", static_cast<long long>(value));
    } else if constexpr (std::is_floating_point_v<T>) {
        LOGI("float: %f", static_cast<double>(value));
    }
}

// ✅ std::string_view — zero-copy string reference
void processName(std::string_view name) {
    LOGI("Processing: %.*s", static_cast<int>(name.size()), name.data());
}
```

---

## Error Handling

```cpp
// ✅ Return std::expected-like result (C++23) or use std::optional / std::variant
struct Error {
    int code;
    std::string message;
};

// Custom Result type for C++17
template<typename T>
class Result {
public:
    static Result<T> ok(T value) {
        Result r;
        r.value_ = std::move(value);
        return r;
    }
    static Result<T> err(Error error) {
        Result r;
        r.error_ = std::move(error);
        return r;
    }

    bool isOk() const { return value_.has_value(); }
    bool isErr() const { return error_.has_value(); }
    const T& value() const { return *value_; }
    const Error& error() const { return *error_; }

private:
    std::optional<T> value_;
    std::optional<Error> error_;
};

// Usage
Result<std::vector<uint8_t>> encryptData(const std::vector<uint8_t>& data) {
    if (data.empty()) {
        return Result<std::vector<uint8_t>>::err({-1, "Empty input"});
    }
    auto encrypted = performEncryption(data);
    return Result<std::vector<uint8_t>>::ok(std::move(encrypted));
}
```

---

## Concurrency

```cpp
#include <mutex>
#include <thread>
#include <atomic>

// ✅ Thread-safe cache with mutex
class KeyCache {
public:
    std::optional<std::vector<uint8_t>> get(const std::string& id) {
        std::lock_guard<std::mutex> lock(mutex_);
        auto it = cache_.find(id);
        if (it == cache_.end()) return std::nullopt;
        return it->second;
    }

    void set(const std::string& id, std::vector<uint8_t> key) {
        std::lock_guard<std::mutex> lock(mutex_);
        cache_[id] = std::move(key);
    }

    void clear() {
        std::lock_guard<std::mutex> lock(mutex_);
        cache_.clear();
    }

private:
    std::mutex mutex_;
    std::unordered_map<std::string, std::vector<uint8_t>> cache_;
};

// ✅ Atomic for simple counters
std::atomic<int> requestCount{0};
requestCount.fetch_add(1, std::memory_order_relaxed);
```

---

## Containers and Algorithms

```cpp
// ✅ Prefer standard containers
std::vector<uint8_t> buffer(1024);        // dynamic array
std::array<uint8_t, 16> iv{};            // fixed-size array
std::unordered_map<std::string, Key> keys; // hash map
std::string_view view = "hello";          // non-owning string ref

// ✅ Range-based algorithms
#include <algorithm>
#include <numeric>

std::vector<int> values = {3, 1, 4, 1, 5, 9};

// Sort
std::sort(values.begin(), values.end());

// Find
auto it = std::find(values.begin(), values.end(), 5);
if (it != values.end()) { /* found */ }

// Transform
std::vector<std::string> names;
std::transform(values.begin(), values.end(),
    std::back_inserter(names),
    [](int v) { return std::to_string(v); });

// Accumulate
int sum = std::accumulate(values.begin(), values.end(), 0);
```

---

## Secure Memory Handling

```cpp
// ✅ Zero sensitive memory before freeing
void secureClear(std::vector<uint8_t>& data) {
    volatile uint8_t* ptr = data.data();
    for (size_t i = 0; i < data.size(); i++) {
        ptr[i] = 0;
    }
    data.clear();
}

// ✅ Secure string for passwords
class SecureString {
public:
    explicit SecureString(std::string s) : data_(std::move(s)) {}
    ~SecureString() {
        volatile char* ptr = &data_[0];
        for (size_t i = 0; i < data_.size(); i++) ptr[i] = '\0';
    }
    const std::string& get() const { return data_; }
    SecureString(const SecureString&) = delete;
    SecureString& operator=(const SecureString&) = delete;
private:
    std::string data_;
};
```

---

## Logging Macro

```cpp
#include <android/log.h>

#define LOG_TAG "NativeLib"
#define LOGD(...) __android_log_print(ANDROID_LOG_DEBUG, LOG_TAG, __VA_ARGS__)
#define LOGI(...) __android_log_print(ANDROID_LOG_INFO,  LOG_TAG, __VA_ARGS__)
#define LOGW(...) __android_log_print(ANDROID_LOG_WARN,  LOG_TAG, __VA_ARGS__)
#define LOGE(...) __android_log_print(ANDROID_LOG_ERROR, LOG_TAG, __VA_ARGS__)

// Usage
LOGI("Initialized with key size: %zu", keySize);
LOGE("Failed to decrypt: error code %d", errorCode);
```

---

## Anti-Patterns

- Raw `new`/`delete` — use `make_unique` / `make_shared`
- `char*` for strings in business logic — use `std::string` or `std::string_view`
- `int` for sizes and indices — use `size_t` or `std::ptrdiff_t`
- Ignoring return values of security functions — always check errors
- `std::endl` in log-heavy code — use `"\n"`; `endl` flushes and is slow
- Capturing `this` in lambdas stored beyond object lifetime — dangling reference

---

## Related Skills
- `jni` — JNI bridge between C++ and Kotlin
- `ndk` — NDK toolchain and build system
- `cryptography` — native crypto implementations
- `reverse-engineering-resistance` — native code as security layer
