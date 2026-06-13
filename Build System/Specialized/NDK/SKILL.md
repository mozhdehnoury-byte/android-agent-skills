---
name: ndk
description: >
  Android NDK build system, toolchain, and project setup.
  Load this skill when configuring the NDK version, setting up CMake
  or ndk-build, managing ABI splits, linking third-party native libraries,
  debugging native crashes, or configuring native build flags.
---

# NDK

## Overview
The Android NDK (Native Development Kit) provides the toolchain for compiling C/C++ code into native libraries (`.so` files) that run on Android. It includes the compiler (Clang), standard libraries, Android-specific headers, and the build system integration with Gradle via CMake or ndk-build.

---

## Core Principles

- **Pin the NDK version** in `build.gradle.kts` — NDK updates can break builds
- **CMake is preferred** over ndk-build for new projects — better IDE support
- **ABI filters** reduce APK size — only include the ABIs your app supports
- **STL selection** matters — use `c++_shared` for shared libraries, `c++_static` for single `.so`
- **Symbols** — strip debug symbols from release builds for size; keep for crash analysis

---

## Gradle Setup

```kotlin
// app/build.gradle.kts
android {
    ndkVersion = "26.1.10909125"    // ✅ pin exact version

    defaultConfig {
        externalNativeBuild {
            cmake {
                cppFlags("-std=c++17", "-fexceptions", "-frtti")
                arguments(
                    "-DANDROID_STL=c++_shared",
                    "-DANDROID_PLATFORM=android-26"
                )
            }
        }

        // ✅ Only build for ABIs you support
        ndk {
            abiFilters += listOf("arm64-v8a", "x86_64")  // drop x86 for production
        }
    }

    externalNativeBuild {
        cmake {
            path("src/main/cpp/CMakeLists.txt")
            version("3.22.1")
        }
    }

    // ✅ Split APKs by ABI (Play Store handles delivery)
    splits {
        abi {
            isEnable = true
            reset()
            include("arm64-v8a", "armeabi-v7a", "x86_64")
            isUniversalApk = false
        }
    }
}
```

---

## CMakeLists.txt — Full Example

```cmake
cmake_minimum_required(VERSION 3.22.1)
project("myapp" VERSION 1.0.0)

# ✅ Set C++ standard
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# ✅ Source files
set(SOURCES
    src/native_lib.cpp
    src/crypto/aes_gcm.cpp
    src/utils/string_utils.cpp
)

# ✅ Create shared library
add_library(myapp-native SHARED ${SOURCES})

# ✅ Find Android system libraries
find_library(log-lib log)
find_library(android-lib android)

# ✅ Include headers
target_include_directories(myapp-native PRIVATE
    ${CMAKE_CURRENT_SOURCE_DIR}/include
)

# ✅ Link libraries
target_link_libraries(myapp-native
    ${log-lib}
    ${android-lib}
)

# ✅ Compiler flags
target_compile_options(myapp-native PRIVATE
    -Wall
    -Wextra
    -O2
    $<$<CONFIG:Debug>:-g>
    $<$<CONFIG:Release>:-Os -fvisibility=hidden>
)

# ✅ Strip symbols in release (reduces .so size)
set_target_properties(myapp-native PROPERTIES
    $<$<CONFIG:Release>:LINK_FLAGS "-Wl,--strip-all">
)
```

---

## Linking Pre-built Libraries

```cmake
# ✅ Link a pre-built .so from a third-party SDK
add_library(third-party-lib SHARED IMPORTED)
set_target_properties(third-party-lib PROPERTIES
    IMPORTED_LOCATION ${CMAKE_SOURCE_DIR}/libs/${ANDROID_ABI}/libthirdparty.so
)

target_link_libraries(myapp-native
    third-party-lib
    ${log-lib}
)

# ✅ Copy the .so to jniLibs so it's packaged in the APK
# Place in: src/main/jniLibs/<abi>/libthirdparty.so
```

---

## ABI Reference

| ABI | Architecture | Notes |
|---|---|---|
| `arm64-v8a` | 64-bit ARM | Most modern Android devices |
| `armeabi-v7a` | 32-bit ARM | Older devices; still ~15% of installs |
| `x86_64` | 64-bit x86 | Emulators, some Chrome OS |
| `x86` | 32-bit x86 | Legacy emulators; safe to drop |

```kotlin
// ✅ Production — include arm64-v8a + armeabi-v7a for device coverage
ndk { abiFilters += listOf("arm64-v8a", "armeabi-v7a") }

// ✅ During development — add x86_64 for emulator
ndk { abiFilters += listOf("arm64-v8a", "armeabi-v7a", "x86_64") }
```

---

## Debugging Native Crashes (Tombstones)

```bash
# ✅ Symbolicate a native crash from logcat
# The crash shows: #00 pc 0000abcd  /data/app/.../libmyapp.so

# 1. Find your .so with debug symbols (from build/intermediates/)
# 2. Use ndk-stack:
adb logcat | ndk-stack -sym app/build/intermediates/cmake/debug/obj/arm64-v8a/

# ✅ Or use addr2line to decode a single address
$NDK/toolchains/llvm/prebuilt/linux-x86_64/bin/llvm-addr2line \
  -e libmyapp.so \
  0x0000abcd

# ✅ Use Android Studio's LLDB debugger for live native debugging
# Run/Debug Configurations → Debugger → Debug type: Dual
```

---

## Accessing Android APIs from Native Code

```cpp
// ✅ Android-specific native APIs
#include <android/asset_manager.h>
#include <android/asset_manager_jni.h>
#include <android/log.h>
#include <android/bitmap.h>
#include <GLES3/gl3.h>       // OpenGL ES 3.0
#include <EGL/egl.h>

// ✅ Read asset from native code
JNIEXPORT jbyteArray JNICALL
Java_com_example_NativeAssets_readAsset(
    JNIEnv* env, jobject,
    jobject assetManager, jstring filename
) {
    AAssetManager* mgr = AAssetManager_fromJava(env, assetManager);
    const char* file = env->GetStringUTFChars(filename, nullptr);

    AAsset* asset = AAssetManager_open(mgr, file, AASSET_MODE_BUFFER);
    env->ReleaseStringUTFChars(filename, file);

    if (asset == nullptr) return nullptr;

    off_t size = AAsset_getLength(asset);
    jbyteArray result = env->NewByteArray(size);
    jbyte* buf = env->GetByteArrayElements(result, nullptr);
    AAsset_read(asset, buf, size);
    AAsset_close(asset);

    env->ReleaseByteArrayElements(result, buf, 0);
    return result;
}
```

---

## Build Variants for Native

```kotlin
// ✅ Different native flags per build type
android {
    buildTypes {
        debug {
            externalNativeBuild {
                cmake {
                    cppFlags("-DDEBUG_BUILD", "-g")
                    arguments("-DCMAKE_BUILD_TYPE=Debug")
                }
            }
        }
        release {
            externalNativeBuild {
                cmake {
                    cppFlags("-DNDEBUG", "-O2", "-fvisibility=hidden")
                    arguments("-DCMAKE_BUILD_TYPE=Release")
                }
            }
        }
    }
}
```

---

## Anti-Patterns

- Not pinning `ndkVersion` — CI builds may use different NDK than local builds
- Building for all 4 ABIs — `x86` is dead; `x86_64` only needed for emulators
- `c++_static` STL with multiple `.so` files — causes ODR violations and crashes
- Storing debug `.so` in the release APK — exposes symbols; strip in release
- Not checking `env->ExceptionCheck()` after calling back into Java — silent crashes

---

## Related Skills
- `jni` — JNI bridge between Kotlin and native code
- `c++` — C++ patterns for native Android code
- `gradle` — Gradle build configuration
- `reverse-engineering-resistance` — native code hardening
