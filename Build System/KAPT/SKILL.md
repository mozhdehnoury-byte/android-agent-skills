---
name: kapt
description: >
  Kotlin Annotation Processing Tool (KAPT) setup and usage for Android.
  Load this skill when working with libraries that haven't migrated to KSP yet,
  configuring KAPT options, or understanding why KAPT is slower than KSP.
---

# KAPT (Kotlin Annotation Processing Tool)

## Overview
KAPT is the legacy annotation processing framework for Kotlin. It works by generating Java stubs from Kotlin code and then running Java annotation processors on them. This two-step process makes KAPT significantly slower than KSP. Use KAPT only when a library has no KSP support yet.

---

## Core Principles

- **Prefer KSP over KAPT** whenever possible — KSP is 2x faster
- Use KAPT only for libraries that **haven't migrated to KSP**
- Never use both KAPT and KSP for the **same library** in the same module
- KAPT is not compatible with **Kotlin Multiplatform** — use KSP for KMP
- Track library KSP migration status — switch when available

---

## When to Use KAPT

| Library | KAPT | KSP |
|---------|------|-----|
| Hilt | Legacy | ✅ Use KSP |
| Room | Legacy | ✅ Use KSP |
| Moshi Codegen | Legacy | ✅ Use KSP |
| Glide | ✅ Still KAPT | ❌ No KSP yet |
| Dagger (without Hilt) | ✅ Still KAPT | Partial |
| Some third-party libs | ✅ Check | Check |

---

## Setup

```kotlin
// build.gradle.kts
plugins {
    kotlin("kapt")  // apply KAPT plugin
}

dependencies {
    // ✅ Use kapt() configuration for annotation processors
    kapt(libs.glide.compiler)
    kapt(libs.some.legacy.compiler)
}
```

---

## KAPT Configuration Options

```kotlin
// build.gradle.kts
kapt {
    // ✅ Enable incremental processing (limited — less effective than KSP)
    useBuildCache = true

    // ✅ Correct error types — helps KAPT work with complex generics
    correctErrorTypes = true

    // ✅ Pass arguments to processors
    arguments {
        arg("dagger.fastInit", "enabled")
        arg("dagger.experimentalDaggerErrorMessages", "enabled")
    }

    // ✅ Show processor warnings
    showProcessorStats = true
}
```

---

## KAPT vs KSP Performance

```
KAPT Build Flow:
1. Kotlin → Java stubs (slow)
2. Java annotation processors run on stubs (slow)
3. Generated Java code compiled
4. Kotlin code compiled

KSP Build Flow:
1. KSP processors run directly on Kotlin symbols (fast)
2. Generated Kotlin code compiled together

Result: KSP is typically 2x faster than KAPT
```

---

## Incremental KAPT (Limited)

```kotlin
// ✅ Enable incremental annotation processing in KAPT
// Note: much less effective than KSP's incremental processing
kapt {
    useBuildCache = true
}

// gradle.properties
kapt.incremental.apt=true
kapt.use.worker.api=true
```

---

## Migrating from KAPT to KSP

```kotlin
// Step 1: Remove KAPT plugin (if no more KAPT processors remain)
// plugins { kotlin("kapt") }  // remove

// Step 2: Add KSP plugin
plugins {
    alias(libs.plugins.ksp)
}

// Step 3: Replace kapt() with ksp() for each processor
// Before:
kapt(libs.hilt.compiler)
kapt(libs.androidx.room.compiler)

// After:
ksp(libs.hilt.compiler)
ksp(libs.androidx.room.compiler)

// Step 4: Update KSP-specific config if needed
ksp {
    arg("room.schemaLocation", "$projectDir/schemas")
    arg("room.incremental", "true")
}
```

---

## Mixing KAPT and KSP (When Unavoidable)

```kotlin
// ✅ OK — different libraries, different processors
plugins {
    alias(libs.plugins.ksp)   // for Room, Hilt
    kotlin("kapt")            // for Glide (no KSP yet)
}

dependencies {
    ksp(libs.hilt.compiler)           // KSP
    ksp(libs.androidx.room.compiler)  // KSP
    kapt(libs.glide.compiler)         // KAPT — no KSP alternative yet
}

// ❌ NOT OK — same library using both
ksp(libs.hilt.compiler)   // ❌
kapt(libs.hilt.compiler)  // ❌ — pick one
```

---

## Debugging KAPT Issues

```bash
# ✅ Enable verbose KAPT output
./gradlew assembleDebug --info 2>&1 | grep -i "kapt"

# ✅ See processor output
./gradlew assembleDebug -Dkapt.verbose=true

# ✅ Check generated stubs
# build/tmp/kapt3/stubs/debug/

# ✅ Check generated sources
# build/generated/source/kapt/debug/
```

---

## Anti-Patterns

- Using KAPT for libraries that have KSP support — unnecessary slowdown
- Mixing KAPT and KSP for the same library — one will be ignored or conflict
- Not setting `correctErrorTypes = true` — obscure compilation errors with complex types
- Keeping KAPT plugin when all processors have migrated to KSP — dead weight
- Using KAPT in KMP modules — it's not supported, use KSP

---

## Related Skills
- `ksp` — modern alternative to KAPT (preferred)
- `annotation-processing` — annotation processing patterns
- `gradle` — plugin configuration
- `hilt` — Hilt now uses KSP
- `room` — Room now uses KSP
