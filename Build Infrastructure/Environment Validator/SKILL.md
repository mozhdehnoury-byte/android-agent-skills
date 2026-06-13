---
name: environment-validator
description: >
  Validating the build environment before running Android builds.
  Load this skill when ensuring required tools are installed, verifying
  SDK versions, checking environment variables, or creating pre-build
  validation tasks that fail fast with clear error messages.
---

# Environment Validator

## Overview
Environment validation ensures the build environment has all required tools, SDKs, and configuration before the build starts. Without validation, builds fail with cryptic errors mid-way through. A good validator fails fast with clear, actionable messages.

---

## Core Principles

- **Fail fast** — validate before any build work begins
- Provide **clear, actionable error messages** — tell the developer exactly what to fix
- Validate in `settings.gradle.kts` or a `preBuild` task — earliest possible point
- Check **required tools** (Java, Android SDK, NDK if needed) and **environment variables**
- Validation must be **fast** — it runs on every build

---

## Gradle Wrapper Version Validation

```kotlin
// settings.gradle.kts
// ✅ Validate Gradle wrapper version
val minGradleVersion = "8.7"
val currentVersion = gradle.gradleVersion

if (GradleVersion.version(currentVersion) < GradleVersion.version(minGradleVersion)) {
    throw GradleException(
        """
        ❌ Gradle $minGradleVersion or higher is required.
        Current version: $currentVersion
        
        Fix: Run ./gradlew wrapper --gradle-version=$minGradleVersion
        """.trimIndent()
    )
}
```

---

## Java Version Validation

```kotlin
// settings.gradle.kts or root build.gradle.kts
// ✅ Validate Java version
val minJavaVersion = JavaVersion.VERSION_17
val currentJavaVersion = JavaVersion.current()

if (currentJavaVersion < minJavaVersion) {
    throw GradleException(
        """
        ❌ Java $minJavaVersion or higher is required.
        Current version: $currentJavaVersion
        
        Fix: Install JDK 17+ and set JAVA_HOME:
          export JAVA_HOME=${'$'}(/usr/libexec/java_home -v 17)
        """.trimIndent()
    )
}
```

---

## Android SDK Validation

```kotlin
// root build.gradle.kts
// ✅ Validate Android SDK is installed
subprojects {
    afterEvaluate {
        val androidExtension = extensions.findByName("android") ?: return@afterEvaluate
        tasks.register("validateAndroidSdk") {
            group = "verification"
            doLast {
                val sdkDir = System.getenv("ANDROID_HOME")
                    ?: System.getenv("ANDROID_SDK_ROOT")
                    ?: "${System.getProperty("user.home")}/Library/Android/sdk"

                if (!File(sdkDir).exists()) {
                    throw GradleException(
                        """
                        ❌ Android SDK not found at: $sdkDir
                        
                        Fix: Install Android Studio or set ANDROID_HOME:
                          export ANDROID_HOME=~/Library/Android/sdk
                        """.trimIndent()
                    )
                }

                val compileSdkVersion = 35
                val platformDir = File(sdkDir, "platforms/android-$compileSdkVersion")
                if (!platformDir.exists()) {
                    throw GradleException(
                        """
                        ❌ Android SDK platform $compileSdkVersion not installed.
                        
                        Fix: Open Android Studio → SDK Manager → Install Android $compileSdkVersion
                        Or run: sdkmanager "platforms;android-$compileSdkVersion"
                        """.trimIndent()
                    )
                }
            }
        }

        tasks.named("preBuild") {
            dependsOn("validateAndroidSdk")
        }
    }
}
```

---

## Environment Variables Validation

```kotlin
// build.gradle.kts (app)
// ✅ Validate required environment variables for release builds
tasks.register("validateReleaseEnvironment") {
    group = "verification"
    onlyIf { gradle.taskGraph.hasTask(":app:assembleRelease") }

    doLast {
        val required = mapOf(
            "KEYSTORE_PATH"     to "Path to the release keystore file",
            "KEYSTORE_PASSWORD" to "Password for the keystore",
            "KEY_ALIAS"         to "Key alias in the keystore",
            "KEY_PASSWORD"      to "Password for the key"
        )

        val missing = required.filter { (key, _) ->
            System.getenv(key).isNullOrBlank()
        }

        if (missing.isNotEmpty()) {
            throw GradleException(
                buildString {
                    appendLine("❌ Missing required environment variables for release build:")
                    missing.forEach { (key, description) ->
                        appendLine("  - $key: $description")
                    }
                    appendLine()
                    appendLine("Fix: Set these variables in your CI environment or local .env file.")
                }
            )
        }
    }
}

tasks.named("preBuild") {
    dependsOn("validateReleaseEnvironment")
}
```

---

## google-services.json Validation

```kotlin
// app/build.gradle.kts
// ✅ Validate google-services.json exists before build
tasks.register("validateGoogleServices") {
    group = "verification"
    doLast {
        val googleServicesFile = file("google-services.json")
        if (!googleServicesFile.exists()) {
            throw GradleException(
                """
                ❌ google-services.json not found at: ${googleServicesFile.absolutePath}
                
                Fix:
                1. Go to Firebase Console → Project Settings → Your App
                2. Download google-services.json
                3. Place it in the app/ directory
                
                Note: Never commit this file to public repositories.
                """.trimIndent()
            )
        }
    }
}

tasks.named("preBuild") {
    dependsOn("validateGoogleServices")
}
```

---

## Convention Plugin for Environment Validation

```kotlin
// build-logic/convention/src/main/kotlin/EnvironmentValidationPlugin.kt
class EnvironmentValidationPlugin : Plugin<Project> {
    override fun apply(target: Project) {
        target.tasks.register("validateEnvironment") {
            group = "verification"
            description = "Validates build environment prerequisites"

            doLast {
                val errors = mutableListOf<String>()

                // Java version
                if (JavaVersion.current() < JavaVersion.VERSION_17) {
                    errors += "Java 17+ required (current: ${JavaVersion.current()})"
                }

                // Gradle version
                if (GradleVersion.version(target.gradle.gradleVersion) <
                    GradleVersion.version("8.7")) {
                    errors += "Gradle 8.7+ required (current: ${target.gradle.gradleVersion})"
                }

                // ANDROID_HOME
                val androidHome = System.getenv("ANDROID_HOME")
                    ?: System.getenv("ANDROID_SDK_ROOT")
                if (androidHome.isNullOrBlank()) {
                    errors += "ANDROID_HOME environment variable not set"
                }

                if (errors.isNotEmpty()) {
                    throw GradleException(
                        buildString {
                            appendLine("❌ Build environment validation failed:")
                            errors.forEach { appendLine("  • $it") }
                            appendLine()
                            appendLine("Fix each issue above and try again.")
                        }
                    )
                }

                println("✅ Build environment validation passed")
            }
        }

        target.tasks.configureEach {
            if (name == "preBuild") {
                dependsOn("validateEnvironment")
            }
        }
    }
}
```

---

## local.properties Validation

```kotlin
// root build.gradle.kts
// ✅ Validate local.properties exists and has required keys
tasks.register("validateLocalProperties") {
    group = "verification"
    doLast {
        val localProps = file("local.properties")
        if (!localProps.exists()) {
            throw GradleException(
                """
                ❌ local.properties not found.
                
                Fix: Create local.properties in the project root with:
                  sdk.dir=/path/to/your/android/sdk
                  
                Android Studio creates this automatically — open the project in AS.
                """.trimIndent()
            )
        }

        val props = java.util.Properties().apply {
            load(localProps.inputStream())
        }

        if (props.getProperty("sdk.dir").isNullOrBlank()) {
            throw GradleException(
                """
                ❌ sdk.dir not set in local.properties.
                
                Fix: Add to local.properties:
                  sdk.dir=/Users/yourname/Library/Android/sdk
                """.trimIndent()
            )
        }
    }
}
```

---

## Anti-Patterns

- Validation that runs too late — after expensive tasks have already started
- Cryptic error messages — "SDK not found" without telling where it looked
- Validating everything on every build — slow, check only what's needed
- Not validating in CI — environment assumption mismatches cause mysterious failures
- Hardcoding paths in validation — use environment variables and standard locations

---

## Related Skills
- `build-orchestration` — task ordering and build pipeline
- `gradle` — Gradle task configuration
- `build-variant` — validating per-variant requirements
- `ci-cd` — environment setup in CI pipeline
