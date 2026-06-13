---
name: incremental-build
description: >
  Gradle incremental build configuration for Android.
  Load this skill when optimizing build times, understanding why builds
  aren't incremental, or configuring tasks for incremental processing.
---

# Incremental Build

## Overview
Gradle's incremental build system skips tasks whose inputs and outputs haven't changed. In a properly configured project, only affected modules and tasks re-run after a change. Incremental builds can reduce build time from minutes to seconds.

---

## Core Principles

- Gradle skips tasks when **inputs and outputs are unchanged** — the UP-TO-DATE check
- Tasks must declare their **inputs and outputs** explicitly for incrementality to work
- **Configuration cache** skips the configuration phase on subsequent builds
- **Build cache** reuses task outputs across machines (local and remote)
- Any task that's not incremental becomes a **build bottleneck**

---

## gradle.properties Configuration

```properties
# ✅ Enable all incremental build features
org.gradle.parallel=true                    # parallel task execution
org.gradle.caching=true                     # local build cache
org.gradle.configuration-cache=true         # cache configuration phase
org.gradle.configuration-cache.problems=warn # warn instead of fail on problems
org.gradle.daemon=true                       # keep Gradle daemon alive
org.gradle.jvmargs=-Xmx4g -XX:+UseParallelGC
```

---

## Understanding Task States

```bash
# Task output states
# UP-TO-DATE    → inputs/outputs unchanged, task skipped ✅
# FROM-CACHE    → output restored from build cache ✅
# NO-SOURCE     → no input sources, task skipped ✅
# SKIPPED       → task condition not met (e.g., disabled) ✅
# EXECUTED      → task ran (not incremental) ⚠️

./gradlew assembleDebug --info | grep "Task :"
# Look for: Task :app:compileDebugKotlin UP-TO-DATE
```

---

## What Breaks Incrementality

```kotlin
// ❌ Reading System.currentTimeMillis() in a task — always changes
tasks.register("generateBuildInfo") {
    doLast {
        val file = File(outputDir, "build_info.txt")
        file.writeText("Built at: ${System.currentTimeMillis()}")  // NOT incremental
    }
}

// ✅ Declare inputs/outputs properly
abstract class GenerateBuildInfoTask : DefaultTask() {
    @get:Input abstract val buildTimestamp: Property<Long>
    @get:OutputFile abstract val outputFile: RegularFileProperty

    @TaskAction
    fun generate() {
        outputFile.get().asFile.writeText("Built at: ${buildTimestamp.get()}")
    }
}

// ✅ Only changes when buildTimestamp input changes
tasks.register<GenerateBuildInfoTask>("generateBuildInfo") {
    buildTimestamp.set(System.currentTimeMillis())
    outputFile.set(layout.buildDirectory.file("generated/build_info.txt"))
}
```

---

## KSP Incremental Processing

```kotlin
// ✅ KSP is incremental by default
// Room and Hilt benefit automatically

// ✅ Explicit Room incremental config
ksp {
    arg("room.incremental", "true")
}
```

---

## Avoiding Configuration Cache Problems

```kotlin
// ❌ Accessing project at execution time — breaks config cache
tasks.register("badTask") {
    doLast {
        println(project.version)  // project accessed at execution — breaks config cache
    }
}

// ✅ Capture at configuration time
tasks.register("goodTask") {
    val version = project.version.toString()  // captured at config time
    doLast {
        println(version)
    }
}

// ❌ Using Task.project inside doLast
tasks.named("assemble") {
    doLast {
        project.copy { ... }  // breaks config cache
    }
}

// ✅ Use injected services instead
abstract class MyTask @Inject constructor(
    private val fs: FileSystemOperations
) : DefaultTask() {
    @TaskAction fun run() { fs.copy { ... } }
}
```

---

## Checking Build Cache Effectiveness

```bash
# ✅ Run build twice — second should be FROM-CACHE
./gradlew assembleDebug
./gradlew clean
./gradlew assembleDebug
# Expected: tasks show FROM-CACHE

# ✅ Build scan — detailed incrementality report
./gradlew assembleDebug --scan
# Shows which tasks ran and why
```

---

## Module-Level Incrementality

```kotlin
// ✅ Modularization is the biggest win for incremental builds
// Only modules with changed files recompile

// ✅ Use implementation() not api() — api() propagates recompilation
// Changing a type in api() triggers recompilation of all consumers
// Changing a type in implementation() only recompiles that module

// ✅ Minimize module dependencies
// Feature modules should depend only on :core:* not on each other
```

---

## Build Performance Profiling

```bash
# ✅ Profile report (HTML)
./gradlew assembleDebug --profile
# Output: build/reports/profile/

# ✅ Build scan (detailed, online)
./gradlew assembleDebug --scan

# ✅ Find slowest tasks
./gradlew assembleDebug --profile 2>&1 | grep "ms" | sort -rn | head -20
```

---

## Anti-Patterns

- Not enabling `org.gradle.caching=true` — cache never used
- Tasks without declared inputs/outputs — always EXECUTED, never UP-TO-DATE
- Reading external state (time, random, env vars) inside task actions without declaring as input
- `api()` dependencies everywhere — breaks incremental compilation between modules
- Disabling the Gradle daemon — cold start every build
- Not enabling configuration cache — configuration phase re-runs every build

---

## Related Skills
- `build-cache` — local and remote build cache
- `gradle` — Gradle fundamentals
- `convention-plugin` — consistent task configuration across modules
- `modularization` — module structure for incremental builds
