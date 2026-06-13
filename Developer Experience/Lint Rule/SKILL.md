---
name: lint-rule
description: >
  Custom Android Lint rules for enforcing project conventions.
  Load this skill when writing custom Lint checks, detecting forbidden
  patterns in code, enforcing naming conventions automatically,
  or integrating custom rules into the build pipeline.
---

# Lint Rule

## Overview
Android Lint is a static analysis tool that inspects source code for bugs, style violations, and custom project conventions. You can write custom Lint rules to enforce patterns specific to your project — preventing anti-patterns before they reach code review.

---

## Core Principles

- Custom rules run at **build time** — catch issues before they reach review
- Rules target **PSI (Program Structure Interface)** — the AST of Kotlin/Java files
- **UAST (Universal AST)** works for both Kotlin and Java — prefer it over PSI
- Each rule has a unique **Issue ID** — used in `@SuppressLint` and baseline files
- Rules are packaged in a separate **lint module** — consumed by the app module

---

## Module Setup

```
project/
├── app/
├── lint/                    # custom lint rules module
│   ├── build.gradle.kts
│   └── src/main/kotlin/
│       └── com/example/lint/
│           ├── MyDetector.kt
│           └── MyIssueRegistry.kt
```

```kotlin
// lint/build.gradle.kts
plugins {
    alias(libs.plugins.kotlin.jvm)
}

dependencies {
    compileOnly(libs.lint.api)
    compileOnly(libs.lint.checks)
    testImplementation(libs.lint.tests)
    testImplementation(libs.junit)
}
```

```kotlin
// app/build.gradle.kts
dependencies {
    lintChecks(project(":lint"))
}
```

---

## Basic Detector Structure

```kotlin
// ✅ Detector that finds a forbidden pattern
class ForbiddenCallDetector : Detector(), Detector.UastScanners {

    override fun getApplicableMethodNames(): List<String> = listOf("printStackTrace")

    override fun visitMethodCall(
        context: JavaContext,
        node: UCallExpression,
        method: PsiMethod
    ) {
        context.report(
            issue = ISSUE,
            scope = node,
            location = context.getLocation(node),
            message = "Use structured logging instead of `printStackTrace()`"
        )
    }

    companion object {
        val ISSUE = Issue.create(
            id = "ForbiddenPrintStackTrace",
            briefDescription = "Use structured logging",
            explanation = """
                `printStackTrace()` logs to stdout which is not captured by crash reporters.
                Use the project's Logger instead.
            """.trimIndent(),
            category = Category.CORRECTNESS,
            priority = 7,
            severity = Severity.ERROR,
            implementation = Implementation(
                ForbiddenCallDetector::class.java,
                Scope.JAVA_FILE_SCOPE
            )
        )
    }
}
```

---

## Issue Registry

```kotlin
// ✅ Register all custom issues
class MyIssueRegistry : IssueRegistry() {
    override val issues: List<Issue> = listOf(
        ForbiddenCallDetector.ISSUE,
        DirectColorUsageDetector.ISSUE,
        MissingTestTagDetector.ISSUE
    )

    override val api: Int = CURRENT_API
    override val minApi: Int = 8
}
```

```
# lint/src/main/resources/META-INF/services/
# File: com.android.tools.lint.client.api.IssueRegistry
com.example.lint.MyIssueRegistry
```

---

## Practical Examples

```kotlin
// ✅ Detect direct color usage (enforce design tokens)
class DirectColorUsageDetector : ResourceXmlDetector() {

    override fun getApplicableAttributes(): Collection<String> =
        listOf("textColor", "background", "backgroundTint", "tint")

    override fun visitAttribute(context: XmlContext, attribute: Attr) {
        val value = attribute.value
        if (value.startsWith("#") || value.startsWith("@android:color/")) {
            context.report(
                issue = ISSUE,
                location = context.getLocation(attribute),
                message = "Use design token colors instead of direct color values (`$value`)"
            )
        }
    }

    companion object {
        val ISSUE = Issue.create(
            id = "DirectColorUsage",
            briefDescription = "Use design token colors",
            explanation = "Direct color values bypass the design system. Use theme attributes.",
            category = Category.CORRECTNESS,
            priority = 6,
            severity = Severity.WARNING,
            implementation = Implementation(
                DirectColorUsageDetector::class.java,
                Scope.RESOURCE_FILE_SCOPE
            )
        )
    }
}

// ✅ Detect GlobalScope usage
class GlobalScopeDetector : Detector(), Detector.UastScanners {

    override fun getApplicableReferenceNames(): List<String> = listOf("GlobalScope")

    override fun visitReference(
        context: JavaContext,
        reference: UReferenceExpression,
        referenced: PsiElement
    ) {
        context.report(
            issue = ISSUE,
            scope = reference,
            location = context.getLocation(reference),
            message = "Avoid `GlobalScope` — use `viewModelScope`, `lifecycleScope`, or an injected scope instead"
        )
    }

    companion object {
        val ISSUE = Issue.create(
            id = "GlobalScopeUsage",
            briefDescription = "Avoid GlobalScope",
            explanation = "GlobalScope creates unstructured coroutines that leak and can't be cancelled.",
            category = Category.CORRECTNESS,
            priority = 8,
            severity = Severity.ERROR,
            implementation = Implementation(
                GlobalScopeDetector::class.java,
                Scope.JAVA_FILE_SCOPE
            )
        )
    }
}
```

---

## Testing Lint Rules

```kotlin
// ✅ Test a custom lint rule
class ForbiddenCallDetectorTest {

    @Test
    fun `detects printStackTrace call`() {
        lint()
            .files(
                kotlin("""
                    fun example() {
                        Exception("test").printStackTrace()
                    }
                """).indented()
            )
            .issues(ForbiddenCallDetector.ISSUE)
            .run()
            .expect("""
                src/test.kt:2: Error: Use structured logging instead of `printStackTrace()` [ForbiddenPrintStackTrace]
                        Exception("test").printStackTrace()
                        ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
                1 errors, 0 warnings
            """.trimIndent())
    }

    @Test
    fun `clean code passes`() {
        lint()
            .files(
                kotlin("""
                    fun example() {
                        Logger.e("Something failed")
                    }
                """).indented()
            )
            .issues(ForbiddenCallDetector.ISSUE)
            .run()
            .expectClean()
    }
}
```

---

## Suppressing Rules

```kotlin
// ✅ Suppress per call site when genuinely needed
@SuppressLint("GlobalScopeUsage")
fun legacyCode() {
    GlobalScope.launch { /* unavoidable */ }
}

// ✅ Suppress in lint.xml (project-wide baseline)
// lint.xml
<lint>
    <issue id="GlobalScopeUsage" severity="ignore">
        <ignore path="src/legacy/**" />
    </issue>
</lint>
```

---

## Anti-Patterns

- Writing rules that are too strict — rules with too many false positives get suppressed everywhere
- Not writing tests for lint rules — untested rules break silently on API changes
- One giant detector class — one detector per concern
- Severity `ERROR` for style issues — use `WARNING`; save `ERROR` for correctness issues
- Forgetting to register the rule in `IssueRegistry` — rule silently never runs

---

## Related Skills
- `detekt` — static analysis for Kotlin code style
- `gradle` — integrating lint into the build pipeline
- `convention-plugin` — sharing lint config across modules
