---
name: build-cache
description: >
  Gradle build cache setup for local and remote caching in Android projects.
  Load this skill when configuring build cache, speeding up CI builds,
  sharing cached outputs across team members, or debugging cache misses.
---

# Build Cache

## Overview
Gradle Build Cache stores task outputs (compiled classes, resources, generated code) and reuses them when the same task runs again with identical inputs. Local cache helps individual developers; remote cache shares outputs across the whole team and CI — dramatically reducing build times.

---

## Core Principles

- Local cache is **always worth enabling** — zero configuration cost, significant speedup
- Remote cache requires a server — Gradle Enterprise, self-hosted, or GitHub Actions cache
- Cache keys are computed from task **inputs** — any input change = cache miss
- **Configuration cache** and **build cache** are complementary but different
- Never cache tasks with **non-deterministic outputs** — timestamps, random values

---

## Local Cache Setup

```properties
# gradle.properties
org.gradle.caching=true       # ✅ enable local build cache
org.gradle.parallel=true      # parallel execution benefits from cache
```

```kotlin
// settings.gradle.kts — explicit local cache config (optional)
buildCache {
    local {
        isEnabled = true
        directory = File(rootDir, ".gradle/build-cache")
        removeUnusedEntriesAfterDays = 30
    }
}
```

---

## Remote Cache Setup

### Gradle Enterprise / Develocity

```kotlin
// settings.gradle.kts
plugins {
    id("com.gradle.develocity") version "3.18"
}

develocity {
    buildScan {
        termsOfUseUrl = "https://gradle.com/terms-of-service"
        termsOfUseAgree = "yes"
    }
}

buildCache {
    local { isEnabled = true }
    remote<HttpBuildCache> {
        url = uri("https://gradle-cache.example.com/cache/")
        credentials {
            username = System.getenv("GRADLE_CACHE_USERNAME")
            password = System.getenv("GRADLE_CACHE_PASSWORD")
        }
        isPush = System.getenv("CI") == "true"  // only CI pushes to remote
        isEnabled = true
    }
}
```

### GitHub Actions Cache (Free Alternative)

```yaml
# .github/workflows/build.yml
- name: Setup Gradle
  uses: gradle/actions/setup-gradle@v3
  with:
    cache-encryption-key: ${{ secrets.GRADLE_ENCRYPTION_KEY }}
    # Automatically caches ~/.gradle and build outputs
```

---

## Verifying Cache Effectiveness

```bash
# ✅ Run build twice — second should show FROM-CACHE
./gradlew assembleDebug
./gradlew clean
./gradlew assembleDebug
# Expected: :app:compileDebugKotlin FROM-CACHE

# ✅ Check cache hit rate
./gradlew assembleDebug --build-cache --info 2>&1 | grep "FROM-CACHE\|UP-TO-DATE\|EXECUTED"

# ✅ Build scan for detailed cache analysis
./gradlew assembleDebug --scan
```

---

## Cache Miss Debugging

```bash
# ✅ Find why a task isn't cached
./gradlew assembleDebug --info 2>&1 | grep -A5 "compileDebugKotlin"
# Look for: "Input property ... has changed"

# ✅ Task input fingerprint
./gradlew assembleDebug --rerun-tasks  # force all tasks to run
# Then compare with cached run inputs

# Common cache miss causes:
# - Absolute paths in task inputs
# - Timestamps in generated files
# - Environment variables not declared as inputs
# - Non-deterministic code generation
```

---

## Making Custom Tasks Cacheable

```kotlin
// ✅ Declare task as cacheable with proper input/output declarations
@CacheableTask
abstract class GenerateApiTask : DefaultTask() {

    @get:InputFiles
    @get:PathSensitive(PathSensitivity.RELATIVE)  // ✅ relative paths = cache-portable
    abstract val inputSpecs: ConfigurableFileCollection

    @get:Input
    abstract val apiVersion: Property<String>

    @get:OutputDirectory
    abstract val outputDir: DirectoryProperty

    @TaskAction
    fun generate() {
        val output = outputDir.get().asFile
        output.mkdirs()
        inputSpecs.forEach { spec ->
            // generate code from spec
        }
    }
}

// ✅ Register
tasks.register<GenerateApiTask>("generateApi") {
    inputSpecs.from(fileTree("src/api/specs"))
    apiVersion.set("v2")
    outputDir.set(layout.buildDirectory.dir("generated/api"))
}
```

---

## Path Sensitivity

```kotlin
// ✅ Use PathSensitivity to make inputs cache-portable
@get:InputFiles
@get:PathSensitive(PathSensitivity.RELATIVE)  // only relative path matters
abstract val sourceFiles: ConfigurableFileCollection

@get:InputFiles
@get:PathSensitive(PathSensitivity.NAME_ONLY)  // only filename matters
abstract val resourceFiles: ConfigurableFileCollection

@get:InputFiles
@get:PathSensitive(PathSensitivity.NONE)  // only content matters, not path
abstract val configFiles: ConfigurableFileCollection

// ❌ Absolute path (default) — cache misses on different machines
@get:InputFiles
abstract val files: ConfigurableFileCollection  // absolute path = machine-specific
```

---

## CI Cache Configuration

```yaml
# .github/workflows/build.yml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: 17

      # ✅ Cache Gradle dependencies and build outputs
      - name: Setup Gradle
        uses: gradle/actions/setup-gradle@v3
        with:
          cache-encryption-key: ${{ secrets.GRADLE_ENCRYPTION_KEY }}

      - name: Build
        run: ./gradlew assembleDebug
        env:
          CI: true
```

---

## Anti-Patterns

- Not enabling local cache — free speedup left on the table
- Absolute paths in task inputs — cache miss on every machine
- Non-deterministic task outputs — cache entries become stale/incorrect
- Pushing to remote cache from developer machines — pollutes cache with unverified outputs
- `@CacheableTask` without `@PathSensitive` on file inputs — over-broad cache invalidation
- Timestamps or build numbers as task inputs — always a cache miss

---

## Related Skills
- `incremental-build` — incremental task execution
- `gradle` — Gradle build configuration
- `build-orchestration` — remote cache in multi-project builds
- `ci-cd` — cache configuration in CI pipeline
