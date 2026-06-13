---
name: ksp
description: >
  Kotlin Symbol Processing (KSP) setup and usage for Android.
  Load this skill when configuring KSP for code generation libraries
  (Room, Hilt, Moshi), writing custom KSP processors, or migrating
  from KAPT to KSP.
---

# KSP (Kotlin Symbol Processing)

## Overview
KSP is Kotlin's modern annotation processing framework. It replaces KAPT, is up to 2x faster, supports incremental processing, and works with Kotlin Multiplatform. Most major Android libraries (Room, Hilt, Moshi) have migrated to KSP.

---

## Core Principles

- **Always prefer KSP over KAPT** for new code and when migrating
- Never mix KSP and KAPT for the **same library** in the same module
- KSP generates code in `build/generated/ksp/` — never edit generated files
- KSP configuration goes in the `ksp {}` block — not in `android {}`
- Enable incremental processing for large codebases

---

## Setup

```toml
# libs.versions.toml
[versions]
ksp = "2.0.0-1.0.21"

[plugins]
ksp = { id = "com.google.devtools.ksp", version.ref = "ksp" }
```

```kotlin
// build.gradle.kts (module)
plugins {
    alias(libs.plugins.ksp)
}

dependencies {
    // ✅ KSP-based processors
    ksp(libs.hilt.compiler)
    ksp(libs.androidx.room.compiler)
    ksp(libs.moshi.kotlin.codegen)
}
```

---

## KSP Configuration per Library

```kotlin
// ✅ Room — schema export and incremental processing
ksp {
    arg("room.schemaLocation", "$projectDir/schemas")
    arg("room.incremental", "true")
    arg("room.expandProjection", "true")
}

// ✅ Moshi — codegen options
ksp {
    arg("moshi.generateProguardRules", "true")
}

// ✅ Auto Factory — if using Dagger's factory generation
ksp {
    arg("dagger.fastInit", "enabled")
}
```

---

## KSP in Multi-Module Projects

```kotlin
// ✅ Apply KSP plugin only in modules that use it
// feature/home/build.gradle.kts
plugins {
    id("myapp.android.library")
    id("myapp.android.hilt")  // convention plugin applies ksp + hilt.compiler
}

// ✅ Convention plugin handling KSP setup
class AndroidHiltConventionPlugin : Plugin<Project> {
    override fun apply(target: Project) {
        with(target) {
            pluginManager.apply("com.google.devtools.ksp")
            dependencies {
                add("ksp", libs.findLibrary("hilt-compiler").get())
            }
        }
    }
}
```

---

## KSP in KMP Projects

```kotlin
// ✅ KSP works in KMP — specify target
kotlin {
    sourceSets {
        commonMain.dependencies {
            implementation(libs.room.runtime)
        }
    }
}

dependencies {
    // ✅ KSP for specific KMP targets
    add("kspAndroid", libs.androidx.room.compiler)
    add("kspJvm", libs.androidx.room.compiler)
    // add("kspIos...", libs.androidx.room.compiler)  // if supporting iOS
}
```

---

## KAPT to KSP Migration

```kotlin
// ✅ Migration steps per library

// Room — was KAPT
kapt(libs.androidx.room.compiler)
// → now KSP
ksp(libs.androidx.room.compiler)

// Hilt — was KAPT
kapt(libs.hilt.compiler)
// → now KSP
ksp(libs.hilt.compiler)

// Moshi Codegen — was KAPT
kapt(libs.moshi.kotlin.codegen)
// → now KSP
ksp(libs.moshi.kotlin.codegen)

// ✅ Remove KAPT plugin after migration
// plugins { kotlin("kapt") }  // remove this
```

---

## Writing a Custom KSP Processor

```kotlin
// ✅ Implement SymbolProcessor
class MyProcessor(
    private val codeGenerator: CodeGenerator,
    private val logger: KSPLogger,
    private val options: Map<String, String>
) : SymbolProcessor {

    override fun process(resolver: Resolver): List<KSAnnotated> {
        val symbols = resolver
            .getSymbolsWithAnnotation("com.example.MyAnnotation")
            .filterIsInstance<KSClassDeclaration>()

        val unprocessed = mutableListOf<KSAnnotated>()

        symbols.forEach { classDecl ->
            if (!classDecl.validate()) {
                unprocessed.add(classDecl)
                return@forEach
            }
            generateCode(classDecl)
        }

        return unprocessed
    }

    private fun generateCode(classDecl: KSClassDeclaration) {
        val packageName = classDecl.packageName.asString()
        val className = "${classDecl.simpleName.asString()}Generated"

        codeGenerator.createNewFile(
            dependencies = Dependencies(false, classDecl.containingFile!!),
            packageName = packageName,
            fileName = className
        ).use { stream ->
            stream.write(
                """
                package $packageName
                
                class $className {
                    fun hello() = "Generated for ${classDecl.simpleName.asString()}"
                }
                """.trimIndent().toByteArray()
            )
        }
    }
}

// ✅ Provider — registers the processor
class MyProcessorProvider : SymbolProcessorProvider {
    override fun create(environment: SymbolProcessorEnvironment): SymbolProcessor =
        MyProcessor(
            codeGenerator = environment.codeGenerator,
            logger = environment.logger,
            options = environment.options
        )
}
```

```
# Register in resources/META-INF/services/
# com.google.devtools.ksp.processing.SymbolProcessorProvider
com.example.processor.MyProcessorProvider
```

---

## Viewing Generated Code

```bash
# Generated code location
build/generated/ksp/<variant>/kotlin/

# Example for Room
build/generated/ksp/debug/kotlin/com/example/UserDao_Impl.kt

# ✅ Never edit — regenerated on every build
```

---

## Incremental Processing

```kotlin
// ✅ KSP is incremental by default
// Only reprocesses files that changed

// For custom processors — declare incremental support
class MyProcessor(...) : SymbolProcessor {
    // Returning unprocessed symbols from process() is the incremental contract
    // Symbols that can't be processed yet are returned and retried
}
```

---

## Anti-Patterns

- Using KAPT when KSP is available — 2x slower builds
- Mixing KAPT and KSP for the same library — one will be ignored or conflict
- Applying KSP plugin to modules that don't use it — unnecessary overhead
- Editing files in `build/generated/ksp/` — overwritten on next build
- Not configuring `room.schemaLocation` — migrations can't be written or tested

---

## Related Skills
- `annotation-processing` — broader annotation processing patterns
- `room` — Room database with KSP compiler
- `hilt` — Hilt DI with KSP setup
- `gradle` — Gradle plugin configuration
- `kmp` — KSP in multiplatform projects
