---
name: jni
description: >
  Java Native Interface (JNI) for Android — calling native C/C++ code from Kotlin.
  Load this skill when bridging Kotlin and native code, declaring native methods,
  handling JNI types, managing memory across the JNI boundary,
  or passing complex objects between Kotlin and C/C++.
---

# JNI

## Overview
JNI (Java Native Interface) is the bridge between Kotlin/Java code and native C/C++ code in Android. It allows calling C/C++ functions from Kotlin and vice versa. JNI is used for performance-critical code, hardware access, reusing existing C/C++ libraries, or code that must resist reverse engineering.

---

## Core Principles

- **Minimize JNI boundary crossings** — each call has overhead; batch operations
- **JNI calls are synchronous** — run on the calling thread; use coroutines/threads on the Kotlin side
- **Memory is manual in native code** — no GC; always release resources
- **JNI types map to Java types** — `jstring` not `String`, `jint` not `Int`
- **Avoid storing JNI objects across calls** — use Global References when needed

---

## Basic Setup

```kotlin
// build.gradle.kts
android {
    defaultConfig {
        externalNativeBuild {
            cmake {
                cppFlags("-std=c++17", "-O2")
            }
        }
    }
    externalNativeBuild {
        cmake {
            path("src/main/cpp/CMakeLists.txt")
            version("3.22.1")
        }
    }
}
```

```cmake
# src/main/cpp/CMakeLists.txt
cmake_minimum_required(VERSION 3.22.1)
project("myapp")

add_library(
    myapp-native
    SHARED
    native_lib.cpp
)

find_library(log-lib log)

target_link_libraries(
    myapp-native
    ${log-lib}
)
```

---

## Declaring Native Methods

```kotlin
// ✅ Kotlin side — declare native functions
class NativeCrypto {
    external fun encrypt(data: ByteArray, key: ByteArray): ByteArray
    external fun decrypt(data: ByteArray, key: ByteArray): ByteArray
    external fun generateKey(): ByteArray

    companion object {
        init {
            System.loadLibrary("myapp-native")
        }
    }
}
```

```cpp
// ✅ C++ side — implement with JNI naming convention
// Package: com.example.app.security
// Class: NativeCrypto
// Function name format: Java_<package>_<class>_<method>

#include <jni.h>
#include <string>

extern "C" {

JNIEXPORT jbyteArray JNICALL
Java_com_example_app_security_NativeCrypto_encrypt(
    JNIEnv* env,
    jobject /* this */,
    jbyteArray data,
    jbyteArray key
) {
    // Get data from Java
    jsize dataLen = env->GetArrayLength(data);
    jbyte* dataBytes = env->GetByteArrayElements(data, nullptr);
    jbyte* keyBytes = env->GetByteArrayElements(key, nullptr);

    // Perform encryption (simplified)
    jbyteArray result = env->NewByteArray(dataLen);
    jbyte* resultBytes = env->GetByteArrayElements(result, nullptr);

    for (jsize i = 0; i < dataLen; i++) {
        resultBytes[i] = dataBytes[i] ^ keyBytes[i % env->GetArrayLength(key)];
    }

    // ✅ Always release elements
    env->ReleaseByteArrayElements(data, dataBytes, JNI_ABORT);
    env->ReleaseByteArrayElements(key, keyBytes, JNI_ABORT);
    env->ReleaseByteArrayElements(result, resultBytes, 0);  // 0 = copy back + free

    return result;
}

} // extern "C"
```

---

## JNI Type Mapping

| Kotlin/Java | JNI Type | Array Type |
|---|---|---|
| `Boolean` | `jboolean` | `jbooleanArray` |
| `Int` | `jint` | `jintArray` |
| `Long` | `jlong` | `jlongArray` |
| `Float` | `jfloat` | `jfloatArray` |
| `Double` | `jdouble` | `jdoubleArray` |
| `Byte` | `jbyte` | `jbyteArray` |
| `String` | `jstring` | — |
| `Object` | `jobject` | `jobjectArray` |
| `void` | `void` | — |

---

## Handling Strings

```cpp
// ✅ Convert jstring to std::string
std::string jstringToString(JNIEnv* env, jstring jStr) {
    if (jStr == nullptr) return "";
    const char* chars = env->GetStringUTFChars(jStr, nullptr);
    std::string result(chars);
    env->ReleaseStringUTFChars(jStr, chars);   // ✅ always release
    return result;
}

// ✅ Convert std::string to jstring
jstring stringToJstring(JNIEnv* env, const std::string& str) {
    return env->NewStringUTF(str.c_str());
}

JNIEXPORT jstring JNICALL
Java_com_example_app_NativeUtils_processText(
    JNIEnv* env, jobject, jstring input
) {
    std::string text = jstringToString(env, input);
    std::string result = /* process */ text + "_processed";
    return stringToJstring(env, result);
}
```

---

## Global References (Persistent JNI Objects)

```cpp
// ✅ Use Global Reference to hold a Java object across JNI calls
static jobject gCallback = nullptr;

JNIEXPORT void JNICALL
Java_com_example_app_NativeLib_registerCallback(
    JNIEnv* env, jobject, jobject callback
) {
    // ✅ Delete old reference if exists
    if (gCallback != nullptr) {
        env->DeleteGlobalRef(gCallback);
    }
    // ✅ Create global reference — survives across JNI calls
    gCallback = env->NewGlobalRef(callback);
}

JNIEXPORT void JNICALL
Java_com_example_app_NativeLib_unregisterCallback(
    JNIEnv* env, jobject
) {
    if (gCallback != nullptr) {
        env->DeleteGlobalRef(gCallback);
        gCallback = nullptr;
    }
}
```

---

## Calling Kotlin from C++ (Callbacks)

```cpp
// ✅ Call a Kotlin method from native code
void callKotlinCallback(JNIEnv* env, jobject callback, int progress) {
    jclass callbackClass = env->GetObjectClass(callback);

    // Find the method: void onProgress(int progress)
    jmethodID onProgressMethod = env->GetMethodID(
        callbackClass,
        "onProgress",
        "(I)V"           // method signature: (int) returns void
    );

    if (onProgressMethod != nullptr) {
        env->CallVoidMethod(callback, onProgressMethod, (jint)progress);
    }
    env->DeleteLocalRef(callbackClass);
}
```

---

## JNI Method Signatures

```
Type     Signature
void     V
boolean  Z
byte     B
char     C
short    S
int      I
long     J
float    F
double   D
String   Ljava/lang/String;
int[]    [I
Object   Ljava/lang/Object;

Method: void foo(int, String) → (ILjava/lang/String;)V
Method: int bar(byte[])      → ([B)I
```

---

## Error Handling

```cpp
// ✅ Check for exceptions after calling Java methods
env->CallVoidMethod(obj, method, arg);
if (env->ExceptionCheck()) {
    env->ExceptionDescribe();  // prints to logcat
    env->ExceptionClear();
    // handle error
    return nullptr;
}

// ✅ Throw a Java exception from native code
void throwException(JNIEnv* env, const char* message) {
    jclass exceptionClass = env->FindClass("java/lang/RuntimeException");
    if (exceptionClass != nullptr) {
        env->ThrowNew(exceptionClass, message);
        env->DeleteLocalRef(exceptionClass);
    }
}
```

---

## Logging from Native Code

```cpp
#include <android/log.h>

#define TAG "NativeLib"
#define LOGI(...) __android_log_print(ANDROID_LOG_INFO, TAG, __VA_ARGS__)
#define LOGE(...) __android_log_print(ANDROID_LOG_ERROR, TAG, __VA_ARGS__)

// Usage
LOGI("Processing %d bytes", dataLen);
LOGE("Encryption failed: %s", errorMsg);
```

---

## Anti-Patterns

- Not releasing `GetByteArrayElements` / `GetStringUTFChars` — memory leak
- Storing `JNIEnv*` across threads — `JNIEnv` is thread-local; each thread needs its own
- Using Local References after the JNI call returns — they become invalid
- Crossing the JNI boundary on every frame — batch operations; minimize round-trips
- Forgetting `extern "C"` in C++ files — C++ name mangling breaks JNI lookup

---

## Related Skills
- `ndk` — NDK build system and toolchain setup
- `c++` — C++ patterns for Android native code
- `cryptography` — using JNI for native crypto implementations
- `reverse-engineering-resistance` — native code as a hardening technique
