---
name: annotation-processing
description: >
  Annotation processing setup and usage for Android development.
  Load this skill when working with KSP or KAPT, creating custom annotations,
  configuring code generation tools (Hilt, Room, Moshi, etc.), or
  understanding how annotation processors fit into the build pipeline.
---

# Annotation Processing

## Overview

Annotation processing generates code at compile time based on annotations in the source code. In Android, this powers libraries like Room, Hilt, Moshi, and Glide. KSP (Kotlin Symbol Processing) is the modern replacement for KAPT and should be preferred in all new projects.

---

## Core Principles

- **Always prefer KSP over KAPT** — KSP is faster, Kotlin-first, and KMP-compatible
- Use KAPT only when a library hasn't migrated to KSP yet
- Never mix KSP and KAPT for the same library
- Annotation processors run at **compile time** — errors are caught early
- Generated code lives in `build/generated/` — never edit it manually

---

## KSP vs KAPT

|                | KSP                  | KAPT                    |
| -------------- | -------------------- | ----------------------- |
| Speed          | Fast (Kotlin-native) | Slow (stubs generation) |
| KMP Support    | ✅ Yes                | ❌ No                    |
| Incremental    | ✅ Yes                | Partial                 |
| Kotlin-first   | ✅ Yes                | ❌ Java-based            |
| Recommendation | **Use this**         | Only if no KSP support  |

---

## Setup

### KSP

```toml
# libs.versions.toml
[versions]
ksp = "2.0.0-1.0.21"

[plugins]
ksp = { id = "com.google.devtools.ksp", version.ref = "ksp" }
```

```kotlin
// build.gradle.kts
plugins {
    alias(libs.plugins.ksp)
}

dependencies {
    // KSP-based libraries
    ksp(libs.hilt.compiler)
    ksp(libs.room.compiler)
}
```

### KAPT (legacy — only when KSP unavailable)

```kotlin
plugins {
    kotlin("kapt")
}

dependencies {
    kapt(libs.some.legacy.compiler)
}
```

---

## Common Libraries and Their Processor

| Library     | Processor Type | Dependency                       |
| ----------- | -------------- | -------------------------------- |
| Hilt        | KSP            | `ksp(libs.hilt.compiler)`        |
| Room        | KSP            | `ksp(libs.room.compiler)`        |
| Moshi       | KSP            | `ksp(libs.moshi.kotlin.codegen)` |
| Glide       | KAPT           | `kapt(libs.glide.compiler)`      |
| DataBinding | Built-in       | No extra setup                   |

---

## Custom Annotations

```kotlin
// ✅ Define annotation
@Target(AnnotationTarget.CLASS)
@Retention(AnnotationRetention.SOURCE)
annotation class AutoFactory

// ✅ Define annotation with parameters
@Target(AnnotationTarget.FUNCTION)
@Retention(AnnotationRetention.RUNTIME)
annotation class RequiresPermission(val value: String)
```

---

## Custom KSP Processor

```kotlin
// ✅ Implement SymbolProcessor
class AutoFactoryProcessor(
    private val codeGenerator: CodeGenerator,
    private val logger: KSPLogger
) : SymbolProcessor {

    override fun process(resolver: Resolver): List<KSAnnotated> {
        val symbols = resolver
            .getSymbolsWithAnnotation(AutoFactory::class.qualifiedName!!)
            .filterIsInstance<KSClassDeclaration>()

        symbols.forEach { classDecl ->
            generateFactory(classDecl)
        }

        return emptyList()
    }

    private fun generateFactory(classDecl: KSClassDeclaration) {
        val packageName = classDecl.packageName.asString()
        val className = classDecl.simpleName.asString()

        codeGenerator.createNewFile(
            dependencies = Dependencies(false, classDecl.containingFile!!),
            packageName = packageName,
            fileName = "${className}Factory"
        ).use { stream ->
            stream.write(generateFactoryCode(packageName, className).toByteArray())
        }
    }
}

// ✅ Register processor
class AutoFactoryProcessorProvider : SymbolProcessorProvider {
    override fun create(environment: SymbolProcessorEnvironment): SymbolProcessor =
        AutoFactoryProcessor(environment.codeGenerator, environment.logger)
}
```

```
# Register in resources/META-INF/services/
# com.google.devtools.ksp.processing.SymbolProcessorProvider
com.example.processor.AutoFactoryProcessorProvider
```

---

## KSP in Multi-Module Projects

```kotlin
// ✅ Apply KSP only in modules that use it
// feature/build.gradle.kts
plugins {
    alias(libs.plugins.ksp)
}

// ✅ Room — schema export location per module
ksp {
    arg("room.schemaLocation", "$projectDir/schemas")
    arg("room.incremental", "true")
}

// ✅ Hilt — no extra config needed per module
```

---

## Incremental Processing

```kotlin
// ✅ Enable incremental annotation processing for Room
ksp {
    arg("room.incremental", "true")
}

// ✅ KSP is incremental by default — no extra config needed
// ❌ KAPT incremental is unreliable — another reason to prefer KSP
```

---

## Debugging Annotation Processors

```bash
# View generated code
# build/generated/ksp/<variant>/kotlin/

# Enable KSP verbose logging
ksp {
    arg("verbose", "true")
}

# Clean and rebuild when processor changes
./gradlew clean assembleDebug
```

---

## Anti-Patterns

- Using KAPT when KSP is available for the same library
- Mixing KAPT and KSP for the same library in the same module
- Editing generated files in `build/generated/` — they are overwritten on rebuild
- Applying KSP plugin to modules that don't need it — adds unnecessary compile overhead
- Using `@Retention(RUNTIME)` when `SOURCE` is sufficient — bloats the binary

---

## Related Skills

- `hilt` — Hilt DI with KSP setup
- `room` — Room database with KSP compiler
- `gradle` — build configuration and plugin setup
- `kmp` — KSP in multiplatform modules
