---
name: benchmark
description: >
  Microbenchmarking with Jetpack Benchmark in Android.
  Load this skill when measuring method-level performance, comparing
  algorithm implementations, benchmarking serialization or data processing,
  or setting up the benchmark module.
---

# Benchmark

## Overview
Jetpack Benchmark (microbenchmark) measures the performance of individual code units — methods, algorithms, serialization — with statistical accuracy. It handles warm-up, clock stability, and GC pauses automatically. Use it to compare implementations and catch regressions.

---

## Core Principles

- Benchmark measures **one thing at a time** — isolate the code under test
- Run on a **physical device** — emulator results are unreliable
- Use `BenchmarkRule` — it handles warm-up and measurement automatically
- Benchmark in **release mode** — debug builds are significantly slower
- Compare relative results — absolute numbers vary by device

---

## Setup

```toml
# libs.versions.toml
[libraries]
androidx-benchmark-junit4 = { module = "androidx.benchmark:benchmark-junit4", version = "1.2.4" }
```

```kotlin
// benchmark/build.gradle.kts (separate module)
plugins {
    alias(libs.plugins.android.library)
}

android {
    defaultConfig {
        testInstrumentationRunner = "androidx.benchmark.junit4.AndroidBenchmarkRunner"
        testInstrumentationRunnerArguments["androidx.benchmark.suppressErrors"] = "EMULATOR,LOW-BATTERY"
    }

    buildTypes {
        release {
            isMinifyEnabled = true
            proguardFiles(getDefaultProguardFile("proguard-android-optimize.txt"))
        }
        create("benchmark") {
            initWith(buildTypes.getByName("release"))
            signingConfig = signingConfigs.getByName("debug")
            matchingFallbacks += listOf("release")
        }
    }
}

dependencies {
    androidTestImplementation(libs.androidx.benchmark.junit4)
}
```

---

## Writing a Benchmark

```kotlin
// ✅ Basic benchmark
@RunWith(AndroidJUnit4::class)
class SerializationBenchmark {

    @get:Rule
    val benchmarkRule = BenchmarkRule()

    private val json = Json { ignoreUnknownKeys = true }
    private val sampleJson = """{"id":"1","name":"John","email":"john@example.com"}"""

    @Test
    fun deserializeUser() = benchmarkRule.measureRepeated {
        // ✅ Only the code under test inside measureRepeated
        json.decodeFromString<UserDto>(sampleJson)
    }

    @Test
    fun serializeUser() = benchmarkRule.measureRepeated {
        val user = UserDto("1", "John", "john@example.com")
        json.encodeToString(user)
    }
}
```

---

## Comparing Implementations

```kotlin
// ✅ Compare two sorting algorithms
@RunWith(AndroidJUnit4::class)
class SortingBenchmark {

    @get:Rule
    val benchmarkRule = BenchmarkRule()

    private val data = (1..10_000).shuffled()

    @Test
    fun sortWithSortedBy() = benchmarkRule.measureRepeated {
        data.sortedBy { it }
    }

    @Test
    fun sortWithSort() = benchmarkRule.measureRepeated {
        val copy = data.toMutableList()
        copy.sort()
        copy
    }
}
```

---

## Setup and Teardown

```kotlin
// ✅ runWithTimingDisabled for setup outside measurement
@Test
fun processLargeList() = benchmarkRule.measureRepeated {
    val data = runWithTimingDisabled {
        // Setup not counted in benchmark
        generateLargeDataSet(1000)
    }
    // Only this is measured
    processData(data)
}
```

---

## Running Benchmarks

```bash
# Run all benchmarks
./gradlew :benchmark:connectedAndroidTest \
  -Pandroid.testInstrumentationRunnerArguments.class=com.example.SerializationBenchmark

# Output in logcat:
# SerializationBenchmark.deserializeUser: min=45us, median=47us, max=62us
```

---

## Allocation Benchmark

```kotlin
// ✅ Measure allocations alongside time
@Test
fun checkAllocations() = benchmarkRule.measureRepeated {
    // BenchmarkRule automatically tracks allocations
    // Check "allocationCount" in results
    val result = mapper.toDomain(userDto)
}
```

---

## Anti-Patterns

- Running benchmarks on emulator — results are not representative
- Putting setup code inside `measureRepeated` — inflates measurement
- Benchmarking in debug build — JIT and debug overhead skew results
- Benchmarking with the screen on and other apps running — interference from system
- Comparing results across different devices — only compare on same device

---

## Related Skills
- `macrobenchmark` — end-to-end performance measurement
- `baseline-profile` — using benchmark results to generate profiles
- `allocation-optimization` — reducing allocations flagged by benchmark
- `rendering-performance` — frame-level performance measurement
