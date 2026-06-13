---
name: baseline-profile
description: >
  Baseline Profiles for ahead-of-time compilation in Android.
  Load this skill when generating baseline profiles, reducing app
  startup time, improving scroll jank on first run, or integrating
  profile generation into the CI pipeline.
---

# Baseline Profile

## Overview
Baseline Profiles tell the Android Runtime (ART) which code paths to compile ahead-of-time (AOT) during app install. Without a profile, ART uses JIT compilation, which is slower on first run. With a baseline profile, critical code paths (startup, scroll, navigation) are pre-compiled, resulting in faster startup and smoother initial interactions.

---

## Core Principles

- Generate profiles for **critical user journeys** — startup, main screen scroll, key flows
- Profile generation requires a **physical device or emulator with API 28+**
- Profiles must be **regenerated** after major code changes
- Combine with **Macrobenchmark** to measure the improvement
- Ship profiles in the release build — they're included in the APK/AAB automatically

---

## Setup

```toml
# libs.versions.toml
[versions]
macrobenchmark = "1.3.0"
profileinstaller = "1.3.1"

[libraries]
androidx-profileinstaller = { module = "androidx.profileinstaller:profileinstaller", version.ref = "profileinstaller" }
androidx-benchmark-macro = { module = "androidx.benchmark:benchmark-macro-junit4", version.ref = "macrobenchmark" }
```

```kotlin
// app/build.gradle.kts
dependencies {
    implementation(libs.androidx.profileinstaller)
}

// macrobenchmark/build.gradle.kts (separate module)
plugins {
    alias(libs.plugins.android.test)
}
android {
    targetProjectPath = ":app"
    experimentalProperties["android.experimental.self-instrumenting"] = true
}
dependencies {
    implementation(libs.androidx.benchmark.macro)
}
```

---

## Profile Generator

```kotlin
// macrobenchmark module
@RunWith(AndroidJUnit4::class)
class BaselineProfileGenerator {

    @get:Rule
    val rule = BaselineProfileRule()

    @Test
    fun generate() = rule.collect(
        packageName = "com.example.app",
        profileBlock = {
            // ✅ Cover critical user journeys

            // 1. App startup
            startActivityAndWait()

            // 2. Navigate to main screen
            device.findObject(By.text("Home")).click()
            device.waitForIdle()

            // 3. Scroll through a list
            val list = device.findObject(By.scrollable(true))
            list.fling(Direction.DOWN)
            list.fling(Direction.UP)
            device.waitForIdle()

            // 4. Open a detail screen
            device.findObject(By.text("First Item")).click()
            device.waitForIdle()
        }
    )
}
```

---

## Generate and Install Profile

```bash
# Generate the profile
./gradlew :macrobenchmark:connectedAndroidTest \
  -Pandroid.testInstrumentationRunnerArguments.androidx.benchmark.enabledRules=BaselineProfile

# Profile is saved to:
# app/src/main/baseline-prof.txt (automatically)
# or manually copy from device:
# adb pull /sdcard/Android/media/com.example.app/baseline-prof.txt app/src/main/

# Verify profile is included in release build
./gradlew :app:assembleRelease
# Check APK contains assets/dexopt/baseline.prof
```

---

## Manual Profile (Simple)

```
# app/src/main/baseline-prof.txt
# List critical class and method patterns

# Startup classes
Lcom/example/app/MainActivity;
Lcom/example/app/di/AppModule;

# Compose runtime
Landroidx/compose/runtime/Composer;->**
Landroidx/compose/ui/platform/AndroidComposeView;->**

# Key repository
Lcom/example/app/data/UserRepository;->getUser(Ljava/lang/String;)**
```

---

## Measuring Impact

```kotlin
// ✅ Measure startup with and without baseline profile
@RunWith(AndroidJUnit4::class)
class StartupBenchmark {
    @get:Rule
    val benchmarkRule = MacrobenchmarkRule()

    @Test
    fun startupWithBaselineProfile() = benchmarkRule.measureRepeated(
        packageName = "com.example.app",
        metrics = listOf(StartupTimingMetric()),
        compilationMode = CompilationMode.Partial(
            baselineProfileMode = BaselineProfileMode.Require  // ✅ must use profile
        ),
        startupMode = StartupMode.COLD,
        iterations = 5
    ) {
        pressHome()
        startActivityAndWait()
    }

    @Test
    fun startupWithoutProfile() = benchmarkRule.measureRepeated(
        packageName = "com.example.app",
        metrics = listOf(StartupTimingMetric()),
        compilationMode = CompilationMode.None,  // no AOT — baseline comparison
        startupMode = StartupMode.COLD,
        iterations = 5
    ) {
        pressHome()
        startActivityAndWait()
    }
}
```

---

## CI Integration

```yaml
# .github/workflows/baseline-profile.yml
- name: Generate Baseline Profile
  run: |
    ./gradlew :macrobenchmark:connectedAndroidTest \
      -Pandroid.testInstrumentationRunnerArguments.androidx.benchmark.enabledRules=BaselineProfile
  # Commit updated baseline-prof.txt to repo
```

---

## Anti-Patterns

- Not regenerating profiles after significant code changes — stale profile misses new code paths
- Generating on emulator only — physical device gives more representative profiles
- Not measuring improvement — can't justify the maintenance cost without data
- Including only startup in profile generation — also cover scroll and navigation
- Shipping without `profileinstaller` dependency — profile won't be installed at app install time

---

## Related Skills
- `macrobenchmark` — measuring startup and runtime with Macrobenchmark
- `startup-optimization` — complementary startup improvements
- `benchmark` — microbenchmark for method-level measurement
- `app-startup` — App Startup library for initialization ordering
