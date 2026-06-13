---
name: baseline-profile-generator
description: >
  Generating Baseline Profiles for Android app startup and runtime optimization.
  Load this skill when setting up Baseline Profile generation, writing profile
  rules, integrating with CI, or measuring startup improvement.
---

# Baseline Profile Generator

## Overview
Baseline Profiles pre-compile critical code paths (startup, common user journeys) so the Android Runtime (ART) doesn't need to JIT-compile them at runtime. This reduces app startup time and improves runtime performance — typically 20-40% faster startup on first launch after install.

---

## Core Principles

- Baseline Profiles target **critical user journeys** — not the entire app
- Profiles are included in the **release APK/AAB** — they ship to users
- Profile generation requires a **physical device or emulator** running the app
- Regenerate profiles when **critical code paths change** significantly
- Measure startup **before and after** to verify improvement

---

## Setup

```toml
[versions]
androidx-benchmark = "1.3.1"
androidx-profileinstaller = "1.3.1"

[libraries]
androidx-benchmark-macro    = { module = "androidx.benchmark:benchmark-macro-junit4", version.ref = "androidx-benchmark" }
androidx-profileinstaller   = { module = "androidx.profileinstaller:profileinstaller", version.ref = "androidx-profileinstaller" }
androidx-test-uiautomator   = { module = "androidx.test.uiautomator:uiautomator", version = "2.3.0" }

[plugins]
androidx-baselineprofile = { id = "androidx.baselineprofile", version.ref = "androidx-benchmark" }
```

```kotlin
// settings.gradle.kts
include(":baselineprofile")

// baselineprofile/build.gradle.kts
plugins {
    alias(libs.plugins.android.test)
    alias(libs.plugins.androidx.baselineprofile)
}

android {
    namespace = "com.example.baselineprofile"
    targetProjectPath = ":app"

    defaultConfig {
        minSdk = 28  // Baseline Profiles require API 28+
        testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"
    }
}

dependencies {
    implementation(libs.androidx.test.uiautomator)
    implementation(libs.androidx.benchmark.macro)
}

// app/build.gradle.kts
plugins {
    alias(libs.plugins.androidx.baselineprofile)
}

dependencies {
    implementation(libs.androidx.profileinstaller)
    baselineProfile(project(":baselineprofile"))
}
```

---

## Writing Profile Rules

```kotlin
// baselineprofile/src/main/kotlin/com/example/baselineprofile/BaselineProfileGenerator.kt

@RunWith(AndroidJUnit4::class)
class BaselineProfileGenerator {

    @get:Rule
    val rule = BaselineProfileRule()

    @Test
    fun generate() = rule.collect(
        packageName = "com.example.app",
        profileBlock = {
            // ✅ Journey 1: App startup
            startActivityAndWait()

            // ✅ Journey 2: Navigate to main screen
            device.findObject(By.text("Home")).click()
            device.waitForIdle()

            // ✅ Journey 3: Open a list and scroll
            device.findObject(By.text("Users")).click()
            device.waitForIdle()

            device.findObject(By.res("user_list")).also { list ->
                list.setGestureMargin(device.displayWidth / 5)
                list.fling(Direction.DOWN)
                device.waitForIdle()
                list.fling(Direction.UP)
                device.waitForIdle()
            }

            // ✅ Journey 4: Open detail screen
            device.findObject(By.text("Ali Rezaei")).click()
            device.waitForIdle()
        }
    )
}
```

---

## Generating the Profile

```bash
# ✅ Generate profile — runs on connected device/emulator
./gradlew :baselineprofile:generateBaselineProfile

# ✅ Generated file location
# app/src/main/baseline-prof.txt

# ✅ Verify file was generated
cat app/src/main/baseline-prof.txt
# Should contain lines like:
# HSPLcom/example/app/MainActivity;->onCreate(...)V
# Lcom/example/app/ui/UserListScreen;
```

---

## Generated Profile Format

```
# baseline-prof.txt — auto-generated, commit to version control
HSPLcom/example/app/MainActivity;->onCreate(Landroid/os/Bundle;)V
HSPLcom/example/app/MainActivity;->onResume()V
Lcom/example/app/ui/UserListScreen;
Lcom/example/app/data/UserRepository;
PLcom/example/app/ui/UserCard;->invoke(Landroidx/compose/runtime/Composer;I)V

# H = Hot (compiled with full optimizations)
# S = Startup (compiled for startup path)
# P = Post-startup (compiled after startup)
# L = class loaded at startup
```

---

## Measuring Startup Improvement

```kotlin
// ✅ Macrobenchmark test to measure startup time
@RunWith(AndroidJUnit4::class)
class StartupBenchmark {

    @get:Rule
    val benchmarkRule = MacrobenchmarkRule()

    @Test
    fun startupCompilationNone() = benchmarkRule.measureRepeated(
        packageName = "com.example.app",
        metrics = listOf(StartupTimingMetric()),
        compilationMode = CompilationMode.None(),  // no pre-compilation
        startupMode = StartupMode.COLD,
        iterations = 5,
        measureBlock = { startActivityAndWait() }
    )

    @Test
    fun startupWithBaselineProfile() = benchmarkRule.measureRepeated(
        packageName = "com.example.app",
        metrics = listOf(StartupTimingMetric()),
        compilationMode = CompilationMode.Partial(
            baselineProfileMode = BaselineProfileMode.Require
        ),
        startupMode = StartupMode.COLD,
        iterations = 5,
        measureBlock = { startActivityAndWait() }
    )
}
```

---

## ProfileInstaller

```kotlin
// ✅ ProfileInstaller installs the profile at app install time
// No code needed — adding the dependency is sufficient
dependencies {
    implementation(libs.androidx.profileinstaller)
}

// ✅ Force install profile for testing (debug builds)
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        if (BuildConfig.DEBUG) {
            ProfileVerifier.getCompilationStatusAsync()
                .addListener({
                    val status = ProfileVerifier.getCompilationStatusAsync().get()
                    Log.d("Profile", "Status: ${status.profileInstallResultCode}")
                }, ContextCompat.getMainExecutor(this))
        }
    }
}
```

---

## CI Integration

```yaml
# .github/workflows/generate-baseline-profile.yml
name: Generate Baseline Profile

on:
  workflow_dispatch:  # manual trigger
  schedule:
    - cron: '0 0 * * 1'  # weekly on Monday

jobs:
  generate-profile:
    runs-on: macos-latest  # macOS for Android emulator acceleration
    steps:
      - uses: actions/checkout@v4

      - name: Enable KVM
        run: |
          echo 'KERNEL=="kvm", GROUP="kvm", MODE="0666", OPTIONS+="static_node=kvm"' | sudo tee /etc/udev/rules.d/99-kvm4all.rules
          sudo udevadm control --reload-rules
          sudo udevadm trigger --name-match=kvm

      - uses: reactivecircus/android-emulator-runner@v2
        with:
          api-level: 34
          target: google_apis
          arch: x86_64
          script: ./gradlew :baselineprofile:generateBaselineProfile

      - name: Commit generated profile
        run: |
          git config user.email "ci@example.com"
          git config user.name "CI Bot"
          git add app/src/main/baseline-prof.txt
          git commit -m "Update Baseline Profile" || echo "No changes to commit"
          git push
```

---

## Anti-Patterns

- Not committing `baseline-prof.txt` to version control — profile lost between builds
- Covering only app startup — also profile common user journeys (lists, navigation)
- Never regenerating the profile — drifts from actual hot paths over time
- Using emulator without Google Play — profile generation requires Play services
- Not measuring startup improvement — can't verify benefit

---

## Related Skills
- `baseline-profile` — applying and using Baseline Profiles
- `benchmark` — measuring performance with Macrobenchmark
- `startup-optimization` — app startup performance
- `compose-performance` — Compose-specific optimization
