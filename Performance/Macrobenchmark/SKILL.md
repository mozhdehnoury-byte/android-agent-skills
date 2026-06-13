---
name: macrobenchmark
description: >
  Macrobenchmark for end-to-end performance measurement in Android.
  Load this skill when measuring app startup time, scroll jank,
  navigation transitions, or any user journey performance metric
  using the Macrobenchmark library.
---

# Macrobenchmark

## Overview
Macrobenchmark measures complete user journeys — startup, scroll performance, navigation — from outside the app using UI automation. Unlike microbenchmark, it measures the full system including rendering, I/O, and process startup. It is the standard tool for startup time regression testing and scroll jank measurement.

---

## Core Principles

- Macrobenchmark runs in a **separate test process** — it instruments the app from outside
- Always test in **release** or **benchmark** build type — not debug
- Use `StartupMode.COLD` for startup measurement — ensures process is killed first
- Test on **physical devices** — emulator performance is not representative
- Run multiple iterations — Macrobenchmark reports min, median, and max automatically

---

## Setup

```toml
# libs.versions.toml
[libraries]
androidx-benchmark-macro = { module = "androidx.benchmark:benchmark-macro-junit4", version = "1.3.0" }
androidx-test-uiautomator = { module = "androidx.test.uiautomator:uiautomator", version = "2.3.0" }
```

```kotlin
// macrobenchmark/build.gradle.kts
plugins {
    alias(libs.plugins.android.test)
}

android {
    targetProjectPath = ":app"
    experimentalProperties["android.experimental.self-instrumenting"] = true

    defaultConfig {
        testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"
        testInstrumentationRunnerArguments["androidx.benchmark.suppressErrors"] = "EMULATOR"
    }

    buildTypes {
        create("benchmark") {
            isDebuggable = true
            signingConfig = signingConfigs.getByName("debug")
            matchingFallbacks += listOf("release")
        }
    }
}

dependencies {
    implementation(libs.androidx.benchmark.macro)
    implementation(libs.androidx.test.uiautomator)
}
```

---

## Startup Benchmark

```kotlin
@RunWith(AndroidJUnit4::class)
class StartupBenchmark {

    @get:Rule
    val benchmarkRule = MacrobenchmarkRule()

    @Test
    fun coldStart() = benchmarkRule.measureRepeated(
        packageName = "com.example.app",
        metrics = listOf(StartupTimingMetric()),
        compilationMode = CompilationMode.Partial(
            baselineProfileMode = BaselineProfileMode.Require
        ),
        startupMode = StartupMode.COLD,   // kill process before each iteration
        iterations = 5
    ) {
        pressHome()
        startActivityAndWait()            // wait for first frame
    }

    @Test
    fun warmStart() = benchmarkRule.measureRepeated(
        packageName = "com.example.app",
        metrics = listOf(StartupTimingMetric()),
        compilationMode = CompilationMode.Partial(),
        startupMode = StartupMode.WARM,   // process alive, Activity recreated
        iterations = 5
    ) {
        startActivityAndWait()
    }
}
```

---

## Scroll Jank Benchmark

```kotlin
@RunWith(AndroidJUnit4::class)
class ScrollBenchmark {

    @get:Rule
    val benchmarkRule = MacrobenchmarkRule()

    @Test
    fun scrollUserList() = benchmarkRule.measureRepeated(
        packageName = "com.example.app",
        metrics = listOf(FrameTimingMetric()),   // measures frame timing and jank
        compilationMode = CompilationMode.Partial(),
        startupMode = StartupMode.WARM,
        iterations = 5
    ) {
        startActivityAndWait()

        // Navigate to list screen
        device.findObject(By.text("Users")).click()
        device.waitForIdle()

        // ✅ Scroll the list — FrameTimingMetric captures frame data
        val list = device.findObject(By.scrollable(true))
        repeat(3) {
            list.fling(Direction.DOWN)
            device.waitForIdle()
        }
    }
}
```

---

## Custom Metrics

```kotlin
// ✅ Measure time to specific state (e.g. data loaded)
@Test
fun timeToDataLoaded() = benchmarkRule.measureRepeated(
    packageName = "com.example.app",
    metrics = listOf(
        StartupTimingMetric(),
        TraceSectionMetric("data_loaded")   // custom trace section
    ),
    startupMode = StartupMode.COLD,
    iterations = 5
) {
    pressHome()
    startActivityAndWait()
    // Wait for custom trace section to appear
    device.wait(Until.hasObject(By.text("Users loaded")), 5_000)
}

// In app code — mark the trace section
class UserListViewModel : ViewModel() {
    fun loadUsers() {
        viewModelScope.launch {
            val users = repository.getUsers()
            trace("data_loaded") {   // ✅ marks trace section for benchmark
                _state.value = UiState.Success(users)
            }
        }
    }
}
```

---

## Compilation Modes

```kotlin
// CompilationMode.None — no AOT, pure JIT (worst case baseline)
// CompilationMode.Partial() — use existing baseline profile if present
// CompilationMode.Partial(baselineProfileMode = BaselineProfileMode.Require) — must have profile
// CompilationMode.Full — fully AOT compiled (best case)
```

---

## Running and Reading Results

```bash
# Run benchmarks
./gradlew :macrobenchmark:connectedBenchmarkAndroidTest

# Results saved to:
# build/outputs/connected_android_test_additional_output/benchmark/
# Open in Android Studio via Profiler → Import benchmark results
```

---

## Anti-Patterns

- Running on emulator — performance not representative
- Testing in debug build — JIT + debug overhead inflates results
- Not using `startActivityAndWait()` — benchmark ends before first frame
- Single iteration — statistical noise makes results unreliable
- Not using `CompilationMode.None` as baseline — can't see profile improvement

---

## Related Skills
- `baseline-profile` — generating profiles measured by Macrobenchmark
- `benchmark` — microbenchmark for method-level measurement
- `startup-optimization` — techniques to improve startup metrics
- `rendering-performance` — understanding frame timing results
