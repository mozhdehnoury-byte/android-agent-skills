---
name: detekt
description: >
  Detekt static analysis for Kotlin Android projects.
  Load this skill when setting up Detekt, configuring rules,
  writing custom Detekt rules, suppressing false positives,
  integrating Detekt into CI, or generating reports.
---

# Detekt

## Overview
Detekt is a static code analysis tool for Kotlin. It detects code smells, complexity issues, style violations, and potential bugs. Unlike Android Lint (which targets Android-specific patterns), Detekt focuses on general Kotlin code quality. Both should be used together.

---

## Core Principles

- Detekt runs on **Kotlin source** — not bytecode; catches issues before compilation
- Configure via **`detekt.yml`** — enable/disable rules, set thresholds
- **Baseline** for existing codebases — suppress existing violations, only check new code
- Run in **CI** — fail the build on new violations
- **Type resolution** mode gives more accurate results but is slower

---

## Setup

```kotlin
// root build.gradle.kts
plugins {
    alias(libs.plugins.detekt) apply false
}

// app/build.gradle.kts (or in convention plugin)
plugins {
    alias(libs.plugins.detekt)
}

detekt {
    config.setFrom(files("$rootDir/config/detekt/detekt.yml"))
    baseline = file("$rootDir/config/detekt/baseline.xml")
    buildUponDefaultConfig = true    // start from default, override in yml
    allRules = false                 // don't enable all rules by default
    autoCorrect = false              // set true to auto-fix style issues
}

dependencies {
    detektPlugins(libs.detekt.formatting)   // ktlint-based formatting rules
}
```

```toml
# libs.versions.toml
[versions]
detekt = "1.23.6"

[libraries]
detekt-formatting = { module = "io.gitlab.arturbosch.detekt:detekt-formatting", version.ref = "detekt" }

[plugins]
detekt = { id = "io.gitlab.arturbosch.detekt", version.ref = "detekt" }
```

---

## detekt.yml Configuration

```yaml
# config/detekt/detekt.yml

build:
  maxIssues: 0           # fail on any new issue

complexity:
  LongMethod:
    threshold: 40        # max lines per function
  LongParameterList:
    functionThreshold: 6
    constructorThreshold: 7
  TooManyFunctions:
    thresholdInFiles: 20
    thresholdInClasses: 15
  CyclomaticComplexMethod:
    threshold: 15

naming:
  FunctionNaming:
    functionPattern: '[a-z][a-zA-Z0-9]*'
    excludes: ['**/test/**', '**/androidTest/**']
  ClassNaming:
    classPattern: '[A-Z][a-zA-Z0-9]*'
  VariableNaming:
    variablePattern: '[a-z][A-Za-z0-9]*'

style:
  MagicNumber:
    active: true
    ignoreNumbers: ['-1', '0', '1', '2']
    ignoreHashCodeFunction: true
    ignorePropertyDeclaration: true
  WildcardImport:
    active: true
  UnusedPrivateMember:
    active: true
  ForbiddenComment:
    active: true
    values: ['FIXME:', 'STOPSHIP:']

exceptions:
  TooGenericExceptionCaught:
    active: true
    exceptionNames: ['Exception', 'Throwable']
    allowedExceptionNameRegex: '_|(ignore|expected).*'
  SwallowedException:
    active: true

coroutines:
  GlobalCoroutineUsage:
    active: true
  RedundantSuspendModifier:
    active: true
  SuspendFunWithFlowReturnType:
    active: true

formatting:
  MaximumLineLength:
    maxLineLength: 120
  NoUnusedImports:
    active: true
  TrailingCommaOnCallSite:
    active: false
```

---

## Running Detekt

```bash
# ✅ Run all checks
./gradlew detekt

# ✅ Run with type resolution (more accurate, slower)
./gradlew detektMain

# ✅ Auto-fix formatting issues
./gradlew detekt --auto-correct

# ✅ Generate HTML report
./gradlew detekt
# Report at: app/build/reports/detekt/detekt.html
```

---

## Baseline (for existing projects)

```bash
# ✅ Generate baseline from current violations
./gradlew detektBaseline

# This creates baseline.xml with all current issues
# Future runs only fail on NEW violations
```

---

## Suppressing Violations

```kotlin
// ✅ Suppress a specific rule inline
@Suppress("MagicNumber")
val timeout = 30_000L

// ✅ Suppress multiple rules
@Suppress("LongMethod", "ComplexMethod")
fun complexLegacyFunction() { ... }

// ✅ Suppress in detekt.yml for entire paths
style:
  MagicNumber:
    excludes: ['**/test/**', '**/androidTest/**', '**/Preview*.kt']
```

---

## Custom Detekt Rule

```kotlin
// ✅ Custom rule — detect hardcoded strings in Compose
class HardcodedStringInComposeRule(config: Config) : Rule(config) {

    override val issue = Issue(
        id = "HardcodedStringInCompose",
        severity = Severity.Warning,
        description = "Hardcoded strings in Compose should use string resources",
        debt = Debt.FIVE_MINS
    )

    override fun visitCallExpression(expression: KtCallExpression) {
        super.visitCallExpression(expression)

        // Check if it's a Text() composable with a hardcoded string
        if (expression.calleeExpression?.text == "Text") {
            val firstArg = expression.valueArguments.firstOrNull()
            val argText = firstArg?.getArgumentExpression()?.text
            if (argText != null && argText.startsWith("\"")) {
                report(CodeSmell(
                    issue = issue,
                    entity = Entity.from(expression),
                    message = "Use stringResource() instead of hardcoded string: $argText"
                ))
            }
        }
    }
}

// Register in custom rule set provider
class CustomRuleSetProvider : RuleSetProvider {
    override val ruleSetId: String = "custom-rules"
    override fun instance(config: Config) = RuleSet(
        ruleSetId,
        listOf(HardcodedStringInComposeRule(config))
    )
}
```

---

## CI Integration

```yaml
# .github/workflows/quality.yml
- name: Run Detekt
  run: ./gradlew detekt

- name: Upload Detekt report
  if: failure()
  uses: actions/upload-artifact@v3
  with:
    name: detekt-report
    path: '**/build/reports/detekt/'
```

---

## Detekt vs Lint

| | Detekt | Android Lint |
|---|---|---|
| Target | Kotlin code quality | Android-specific issues |
| Runs on | JVM, CI, pre-commit | Build, CI |
| Custom rules | Kotlin DSL | PSI/UAST |
| Auto-fix | ✅ (formatting) | Limited |
| Best for | Code smells, complexity | Android anti-patterns |

---

## Anti-Patterns

- Ignoring all violations with a baseline — baseline is for migration, not ongoing suppression
- `allRules = true` — too noisy; causes over-suppression
- Not running Detekt in CI — violations accumulate silently
- Per-file `@Suppress` without a comment explaining why — future maintainers won't understand
- Skipping `detekt-formatting` plugin — misses import order and whitespace issues

---

## Related Skills
- `lint-rule` — Android Lint for Android-specific checks
- `gradle` — build integration
- `convention-plugin` — sharing Detekt config across modules
