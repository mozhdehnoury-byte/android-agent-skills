---
name: module-dependency-graph-validation
description: >
  Validating and enforcing module dependency rules in Android multi-module projects.
  Load this skill when enforcing architecture boundaries between modules,
  preventing circular dependencies, or visualizing the module dependency graph.
---

# Module Dependency Graph Validation

## Overview
In a multi-module Android project, modules must depend on each other in a controlled, directed way. Without enforcement, modules accumulate arbitrary dependencies — creating cycles, slow builds, and architectural drift. Dependency graph validation catches violations at build time.

---

## Core Principles

- Module dependencies must be **directed and acyclic** — no cycles ever
- Feature modules must **not depend on other feature modules**
- Only `:app` can depend on all feature modules — it's the composition root
- Core modules can be depended upon by everyone but depend on nothing above them
- Validate the graph **at build time** — not in code reviews alone

---

## Module Layer Architecture

```
:app
 ├── :feature:home
 ├── :feature:auth
 ├── :feature:profile
 │
 ├── :core:ui
 ├── :core:network
 ├── :core:database
 ├── :core:domain
 └── :core:common

Rules:
- :feature:* → :core:* ✅
- :feature:* → :feature:* ❌
- :core:domain → :core:common ✅
- :core:network → :core:domain ❌ (domain must be pure)
- :app → everything ✅
```

---

## Dependency Guard Plugin

```toml
# libs.versions.toml
[plugins]
dependency-guard = { id = "com.dropbox.dependency-guard", version = "0.5.0" }
```

```kotlin
// app/build.gradle.kts
plugins {
    alias(libs.plugins.dependency.guard)
}

dependencyGuard {
    configuration("releaseRuntimeClasspath")
}

// ✅ Generate baseline
// ./gradlew dependencyGuard

// ✅ Verify on CI — fails if deps changed without updating baseline
// ./gradlew dependencyGuardVerify
```

---

## Module Graph Plugin

```toml
[plugins]
module-graph = { id = "com.jraska.module.graph.assertion", version = "2.5.0" }
```

```kotlin
// root build.gradle.kts
plugins {
    alias(libs.plugins.module.graph)
}

moduleGraphAssert {
    maxHeight = 4  // max allowed module depth

    // ✅ Assert no feature-to-feature dependencies
    restricted = arrayOf(
        ":feature:.* -> :feature:.*"  // regex: feature modules can't depend on each other
    )

    // ✅ Assert specific allowed paths
    allowed = arrayOf(
        ":feature:.* -> :core:.*",    // features can use core
        ":app -> :.*"                  // app can use everything
    )
}

// ✅ Run validation
// ./gradlew assertModuleGraph
```

---

## Custom Gradle Task Validation

```kotlin
// build-logic/convention/src/main/kotlin/ModuleGraphValidationPlugin.kt
class ModuleGraphValidationPlugin : Plugin<Project> {
    override fun apply(target: Project) {
        target.tasks.register("validateModuleDependencies") {
            group = "verification"
            description = "Validates module dependency rules"

            doLast {
                val violations = mutableListOf<String>()
                val featureModules = target.rootProject.subprojects
                    .filter { it.path.startsWith(":feature:") }

                featureModules.forEach { featureModule ->
                    featureModule.configurations
                        .filter { it.name == "implementation" }
                        .flatMap { it.dependencies }
                        .filterIsInstance<ProjectDependency>()
                        .filter { it.dependencyProject.path.startsWith(":feature:") }
                        .forEach { dep ->
                            violations.add(
                                "${featureModule.path} -> ${dep.dependencyProject.path} " +
                                "(feature-to-feature dependency not allowed)"
                            )
                        }
                }

                if (violations.isNotEmpty()) {
                    throw GradleException(
                        "Module dependency violations:\n${violations.joinToString("\n")}"
                    )
                }
            }
        }

        // ✅ Run before every build
        target.tasks.named("preBuild") {
            dependsOn("validateModuleDependencies")
        }
    }
}
```

---

## Visualizing the Module Graph

```bash
# ✅ Generate dependency graph image (requires graphviz)
./gradlew generateModuleGraph

# ✅ Using project-report plugin
./gradlew projectDependencies > module-deps.txt

# ✅ Using gradle-dependency-graph-generator plugin
plugins {
    id("com.vanniktech.dependency.graph.generator") version "0.8.0"
}
# ./gradlew generateProjectDependencyGraph
# Output: build/reports/dependency-graph/
```

---

## Detecting Cycles

```kotlin
// ✅ Gradle detects cycles automatically — build fails with:
// "Circular dependency between the following tasks: ..."

// For module-level cycle detection — custom task
fun detectCycles(modules: List<Project>): List<List<String>> {
    val graph = buildDependencyGraph(modules)
    return findCycles(graph)
}

// ✅ Or use: ./gradlew dependencies
// Circular module dependencies cause Gradle to fail immediately
```

---

## Enforcing in CI

```yaml
# .github/workflows/build.yml
- name: Validate module graph
  run: ./gradlew assertModuleGraph

- name: Verify dependency guard
  run: ./gradlew dependencyGuardVerify
```

---

## Anti-Patterns

- Feature modules importing other feature modules — breaks independent build and test
- No automated enforcement — violations accumulate silently
- `:core:network` depending on `:core:domain` — domain must be framework-free
- Circular dependencies — Gradle fails but error message can be cryptic
- Skipping graph validation in CI — local dev can bypass it

---

## Related Skills
- `modularization` — multi-module architecture design
- `multi-module-architecture` — module structure patterns
- `gradle` — Gradle build configuration
- `dependency-management` — managing what modules depend on
- `build-orchestration` — building modules in the right order
