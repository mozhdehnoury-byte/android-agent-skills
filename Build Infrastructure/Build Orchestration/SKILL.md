---
name: build-orchestration
description: >
  Build orchestration for Android multi-module projects — task ordering,
  parallel builds, and coordinating complex build pipelines.
  Load this skill when optimizing multi-module build order, configuring
  parallel execution, or designing the build pipeline for CI/CD.
---

# Build Orchestration

## Overview
Build orchestration is the coordination of Gradle tasks across modules — ensuring they run in the right order, in parallel where possible, and with the right inputs/outputs declared. In multi-module projects, a poorly orchestrated build is the single biggest source of wasted build time.

---

## Core Principles

- Gradle determines task order from **dependency declarations** — not manual ordering
- Tasks that don't depend on each other run **in parallel** when `org.gradle.parallel=true`
- The **critical path** is the longest chain of dependent tasks — optimize it first
- Declare task **inputs and outputs** correctly — enables parallelism and caching
- Module **dependency graph shape** determines maximum parallelism

---

## Parallel Build Configuration

```properties
# gradle.properties
org.gradle.parallel=true           # ✅ enable parallel task execution
org.gradle.workers.max=8           # max parallel workers (default = CPU cores)
org.gradle.caching=true            # tasks restored from cache run "instantly"
org.gradle.configuration-cache=true
org.gradle.jvmargs=-Xmx6g -XX:+UseParallelGC
```

---

## Module Dependency Graph and Parallelism

```
# Wide graph = more parallelism
:app
├── :feature:home     ← builds in parallel
├── :feature:auth     ← builds in parallel
├── :feature:profile  ← builds in parallel
└── :core:network     ← shared, built once

# Deep/narrow graph = sequential, slower
:app → :feature:home → :core:ui → :core:common
# Each must wait for the previous — no parallelism
```

```kotlin
// ✅ Avoid deep chains — prefer wide, shallow dependency graphs
// feature modules should only depend on :core:*, not on each other
```

---

## Task Dependencies

```kotlin
// ✅ Explicit task ordering when needed
tasks.named("generateBuildConfig") {
    mustRunAfter("clean")
}

// ✅ Finalizer task — runs after another regardless of success/failure
tasks.named("assembleRelease") {
    finalizedBy("uploadToStore")
}

// ✅ Lifecycle task — aggregate multiple tasks
tasks.register("buildAll") {
    group = "build"
    description = "Build all variants"
    dependsOn(
        ":app:assembleDebug",
        ":app:assembleRelease",
        ":app:bundleRelease"
    )
}
```

---

## Build Lifecycle Hooks

```kotlin
// ✅ Run a task before every build
tasks.named("preBuild") {
    dependsOn("validateModuleDependencies")
    dependsOn("assertModuleGraph")
}

// ✅ Run code quality checks as part of build
tasks.named("check") {
    dependsOn("detekt")
    dependsOn("ktlintCheck")
}

// ✅ Hook into assemble
tasks.configureEach {
    if (name == "assembleRelease") {
        doFirst {
            println("Starting release build: ${project.version}")
        }
        doLast {
            println("Release build complete")
        }
    }
}
```

---

## Multi-Module Build Order

```kotlin
// ✅ Gradle determines order automatically from module dependencies
// :core:common compiles first (no deps)
// :core:network compiles after :core:common
// :feature:home compiles after :core:network
// :app compiles last (depends on all features)

// ✅ Verify build order with
// ./gradlew :app:assembleDebug --dry-run
// Shows tasks in order without executing
```

---

## Composite Builds

```kotlin
// ✅ Include build-logic as a composite build
// settings.gradle.kts
pluginManagement {
    includeBuild("build-logic")
}

// ✅ Multiple composite builds
pluginManagement {
    includeBuild("build-logic")
    includeBuild("shared-conventions")
}

// Benefits:
// - build-logic plugins are compiled separately
// - Changes to build-logic don't invalidate app module caches
// - Build-logic can be shared across projects
```

---

## Build Scan Integration

```kotlin
// settings.gradle.kts
plugins {
    id("com.gradle.develocity") version "3.18"
}

develocity {
    buildScan {
        termsOfUseUrl = "https://gradle.com/terms-of-service"
        termsOfUseAgree = "yes"
        publishing.onlyIf { System.getenv("CI") == "true" }

        tag(if (System.getenv("CI") == "true") "CI" else "Local")
        value("Git Branch", System.getenv("GITHUB_REF_NAME") ?: "local")
    }
}
```

---

## CI Build Matrix

```yaml
# .github/workflows/build.yml
jobs:
  build:
    strategy:
      matrix:
        task: [assembleDebug, test, lint, detekt]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: gradle/actions/setup-gradle@v3
      - name: Run ${{ matrix.task }}
        run: ./gradlew ${{ matrix.task }}
```

---

## Build Time Analysis

```bash
# ✅ Profile build
./gradlew assembleDebug --profile
# Open: build/reports/profile/profile-*.html

# ✅ Build scan (detailed — requires Gradle Enterprise or free plan)
./gradlew assembleDebug --scan

# ✅ Find the critical path
./gradlew assembleDebug --scan
# Build scan shows critical path in timeline view

# ✅ Identify slowest tasks
./gradlew assembleDebug --profile 2>&1 | grep "ms :" | sort -t' ' -k1 -rn | head -20
```

---

## Optimizing the Critical Path

```kotlin
// The critical path is typically:
// :app:compileDebugKotlin → :app:mergeDebugResources → :app:packageDebug

// ✅ Strategy 1: Move code out of :app into feature modules
// Less code in :app = faster :app:compileDebugKotlin

// ✅ Strategy 2: Use @JvmField and @JvmStatic where appropriate
// Reduces generated code size

// ✅ Strategy 3: Avoid annotation processing in :app
// Move annotated classes to feature/core modules

// ✅ Strategy 4: Use Baseline Profiles
// Reduces ART compilation burden at install time (not build time)
```

---

## Anti-Patterns

- All code in `:app` — no parallelism possible, everything sequential
- Feature modules depending on each other — forces sequential builds
- Missing `org.gradle.parallel=true` — single-threaded builds
- No build caching — every CI run starts from scratch
- Too many custom task hooks — interference with Gradle's optimization
- Running all checks in one sequential job in CI — parallelize with job matrix

---

## Related Skills
- `gradle` — Gradle build fundamentals
- `build-cache` — caching task outputs
- `incremental-build` — UP-TO-DATE checks
- `modularization` — module structure for build parallelism
- `module-dependency-graph-validation` — keeping the graph valid
- `environment-validator` — validating build environment before running
