---
name: build-flavor-strategy
description: >
  Strategy for designing product flavors and build variant structure.
  Load this skill when deciding how many flavors to create, what each
  flavor dimension should represent, and how to keep variants manageable.
---

# Build Flavor Strategy

## Overview
Product flavors define what variant of the app is built — different environments, tiers, or feature sets. A poorly designed flavor structure leads to exponential variant count, complex CI pipelines, and unmaintainable build files. This skill defines how to design flavor structure deliberately.

---

## Core Principles

- Fewer flavors = simpler CI, faster builds, less maintenance
- Each flavor dimension should represent a **clearly different product concern**
- Prefer **build types** for environment differences — flavors for product differences
- Avoid flavors for things that can be done with **feature flags** or **Remote Config**
- Document the variant matrix — every combination must have a clear purpose

---

## Common Flavor Patterns

### Pattern 1: Environment Only (Most Common)

```kotlin
// ✅ Use when you need different backend environments
flavorDimensions += "environment"

productFlavors {
    create("dev") {
        dimension = "environment"
        applicationIdSuffix = ".dev"
        buildConfigField("String", "API_URL", "\"https://dev.api.example.com/\"")
    }
    create("staging") {
        dimension = "environment"
        applicationIdSuffix = ".staging"
        buildConfigField("String", "API_URL", "\"https://staging.api.example.com/\"")
    }
    create("prod") {
        dimension = "environment"
        buildConfigField("String", "API_URL", "\"https://api.example.com/\"")
    }
}
// Variants: devDebug, devRelease, stagingDebug, stagingRelease, prodDebug, prodRelease
```

### Pattern 2: Tier Only

```kotlin
// ✅ Use when distributing free vs paid version of same app
flavorDimensions += "tier"

productFlavors {
    create("free") {
        dimension = "tier"
        buildConfigField("Boolean", "IS_PREMIUM", "false")
    }
    create("premium") {
        dimension = "tier"
        buildConfigField("Boolean", "IS_PREMIUM", "true")
    }
}
```

### Pattern 3: Two Dimensions (Use Sparingly)

```kotlin
// ✅ Only when both dimensions represent truly independent product axes
flavorDimensions += listOf("environment", "tier")

// Results in: devFreeDebug, devPremiumDebug, prodFreeRelease, prodPremiumRelease...
// 8 variants — manageable if most are excluded

android {
    variantFilter {
        // Only build dev+free and prod+premium — others don't make sense
        val env = flavors.find { it.dimension == "environment" }?.name
        val tier = flavors.find { it.dimension == "tier" }?.name
        if ((env == "dev" && tier == "premium") || (env == "prod" && tier == "free")) {
            ignore = true
        }
    }
}
```

---

## Environment via Build Type vs Flavor

```kotlin
// ✅ Environment differences → Build Types (simpler)
// Use when dev/staging/prod differ only in config (URL, keys)
buildTypes {
    debug   { buildConfigField("String", "API_URL", "\"https://dev.api.com/\"") }
    create("staging") { buildConfigField("String", "API_URL", "\"https://staging.api.com/\"") }
    release { buildConfigField("String", "API_URL", "\"https://api.com/\"") }
}

// ✅ Environment via Flavors
// Use when environments need different:
//   - Firebase projects (different google-services.json)
//   - App IDs (can install dev and prod side by side)
//   - Dependencies (dev has debug tools, prod doesn't)
```

---

## Decision Framework

```
Do you need separate Firebase projects per environment?
    YES → Use flavors (each flavor can have its own google-services.json)
    NO  → Use build types

Do you need to install dev and prod side by side?
    YES → Use flavors (different applicationId per flavor)
    NO  → Use build types + applicationIdSuffix

Do you have multiple product tiers (free/paid)?
    YES → Add a tier flavor dimension
    NO  → Use feature flags or Remote Config

Would Remote Config solve this without a new flavor?
    YES → Use Remote Config — avoid the flavor
    NO  → Proceed with flavor
```

---

## Variant Matrix Documentation

```markdown
<!-- Always document the variant matrix in the project -->

## Build Variant Matrix

| Variant | Purpose | Used By |
|---------|---------|---------|
| devDebug | Local development | Developers |
| devRelease | Dev build for QA | QA team |
| stagingRelease | Pre-prod testing | QA + stakeholders |
| prodRelease | Production release | Play Store |

### Excluded Variants
- stagingDebug — unnecessary (staging = pre-prod, always release config)
- prodDebug — security risk (debuggable prod build)
```

---

## Anti-Patterns

- More than 2 flavor dimensions — variant count explodes (3 dims × 2 values = 8+ variants)
- Using flavors for feature flags — use Remote Config instead, no rebuild needed
- No variant filtering — building all combinations wastes CI time
- Flavors with overlapping purposes with build types — pick one
- Not documenting the variant matrix — developers don't know which variant to use

---

## Related Skills
- `build-variant` — implementing build types and flavors
- `gradle` — Gradle build configuration
- `remote-config` — replacing flavor-based feature switching
- `ci-cd` — building specific variants in CI pipeline
