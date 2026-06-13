---
name: snapshot-testing
description: >
  Snapshot (screenshot) testing for Android Compose UI.
  Load this skill when implementing visual regression testing,
  using Paparazzi for JVM-based screenshot tests, using Shot or
  Roborazzi for device-based snapshots, or managing snapshot artifacts in CI.
---

# Snapshot Testing

## Overview
Snapshot testing captures a screenshot of a composable or screen and compares it against a stored reference image. If the UI changes unexpectedly, the test fails. This catches visual regressions that logic tests miss. Paparazzi runs on the JVM (no device needed); Roborazzi extends Robolectric for JVM-based Compose snapshots.

---

## Core Principles

- Snapshot tests catch **visual regressions** — unexpected UI changes
- **JVM-based** (Paparazzi/Roborazzi) is preferred — fast, no emulator needed
- Test **components in isolation** — not full screens; focus on design system components
- Always test **multiple states** — loading, success, error, empty, dark mode, RTL
- **Commit snapshots** to version control — they are the source of truth
- **Update snapshots intentionally** — never auto-accept all failures

---

## Paparazzi (JVM-based, no device)

```kotlin
// dependency: app.cash.paparazzi:paparazzi

// ✅ Basic Paparazzi test
class UserCardSnapshotTest {

    @get:Rule
    val paparazzi = Paparazzi(
        deviceConfig = DeviceConfig.PIXEL_5,
        theme = "android:Theme.Material.Light.NoActionBar"
    )

    @Test
    fun userCard_success() {
        paparazzi.snapshot {
            AppTheme {
                UserCard(
                    user = UserFactory.create(),
                    onUserClick = {}
                )
            }
        }
    }

    @Test
    fun userCard_longName() {
        paparazzi.snapshot("long_name") {
            AppTheme {
                UserCard(
                    user = UserFactory.create(name = "Very Long Name That Might Overflow"),
                    onUserClick = {}
                )
            }
        }
    }
}

// ✅ Test dark mode
class UserCardDarkModeTest {

    @get:Rule
    val paparazzi = Paparazzi(
        deviceConfig = DeviceConfig.PIXEL_5.copy(
            nightMode = NightMode.NIGHT
        )
    )

    @Test
    fun userCard_darkMode() {
        paparazzi.snapshot {
            AppTheme(darkTheme = true) {
                UserCard(user = UserFactory.create(), onUserClick = {})
            }
        }
    }
}
```

---

## Paparazzi Build Setup

```kotlin
// build.gradle.kts
plugins {
    alias(libs.plugins.paparazzi)
}

// libs.versions.toml
[versions]
paparazzi = "1.3.4"

[plugins]
paparazzi = { id = "app.cash.paparazzi", version.ref = "paparazzi" }
```

---

## Roborazzi (JVM-based with Robolectric)

```kotlin
// dependency: io.github.takahirom.roborazzi:roborazzi
// dependency: io.github.takahirom.roborazzi:roborazzi-compose

// ✅ Roborazzi test
@RunWith(RobolectricTestRunner::class)
@Config(sdk = [34])
class UserCardRoborazziTest {

    @get:Rule
    val composeRule = createComposeRule()

    @Test
    fun userCard_defaultState() {
        composeRule.setContent {
            AppTheme {
                UserCard(user = UserFactory.create(), onUserClick = {})
            }
        }

        composeRule
            .onNodeWithTag("user_card")
            .captureRoboImage("snapshots/user_card_default.png")
    }

    @Test
    fun userCard_allStates() {
        val states = listOf(
            "active" to UserFactory.create(),
            "suspended" to UserFactory.suspended(),
            "admin" to UserFactory.admin()
        )

        states.forEach { (name, user) ->
            composeRule.setContent {
                AppTheme {
                    UserCard(user = user, onUserClick = {})
                }
            }
            composeRule
                .onNodeWithTag("user_card")
                .captureRoboImage("snapshots/user_card_$name.png")
        }
    }
}
```

---

## Parameterized Snapshot Tests

```kotlin
// ✅ Test all states systematically
@RunWith(Parameterized::class)
class UserListSnapshotTest(
    private val name: String,
    private val state: UserListUiState
) {
    companion object {
        @JvmStatic
        @Parameterized.Parameters(name = "{0}")
        fun states() = listOf(
            arrayOf("loading", UserListUiState.Loading),
            arrayOf("success", UserListUiState.Success(UserFactory.list())),
            arrayOf("empty", UserListUiState.Success(emptyList())),
            arrayOf("error", UserListUiState.Error("Something went wrong"))
        )
    }

    @get:Rule
    val paparazzi = Paparazzi(deviceConfig = DeviceConfig.PIXEL_5)

    @Test
    fun userList_state() {
        paparazzi.snapshot(name) {
            AppTheme {
                UserListContent(state = state, onUserClick = {})
            }
        }
    }
}
```

---

## Running Snapshot Tests

```bash
# ✅ Record new snapshots (first run or after intentional UI change)
./gradlew recordPaparazziDebug

# ✅ Verify snapshots (CI)
./gradlew verifyPaparazziDebug

# ✅ Roborazzi record
./gradlew recordRoborazziDebug

# ✅ Roborazzi verify
./gradlew verifyRoborazziDebug

# Failure output: diff image showing what changed
# Location: app/build/outputs/paparazzi/failures/
```

---

## CI Setup

```yaml
# .github/workflows/snapshot.yml
- name: Run snapshot tests
  run: ./gradlew verifyPaparazziDebug

- name: Upload snapshot failures
  if: failure()
  uses: actions/upload-artifact@v3
  with:
    name: snapshot-failures
    path: '**/build/outputs/paparazzi/failures/'
```

---

## What to Snapshot vs What Not To

| Snapshot | Don't Snapshot |
|---|---|
| Design system components (buttons, cards, chips) | Full screens with complex state |
| All states of a component (loading/error/empty) | Components with real network data |
| Dark mode variants | Logic behavior (use unit tests) |
| RTL layout variants | Dynamic content (timestamps, counters) |
| Typography and spacing | Complex animated states |

---

## Anti-Patterns

- Snapshotting with dynamic data (timestamps, random IDs) — causes constant failures
- Never updating snapshots — defeats the purpose; stale snapshots mask real regressions
- Accepting all snapshot failures blindly — review each diff; only accept intentional changes
- Snapshotting full screens with nav — isolate components; full screen tests are fragile
- Not testing dark mode — dark mode regressions are common and invisible in light mode tests

---

## Related Skills
- `compose-testing` — Compose semantic tree tests (complementary to snapshots)
- `ui-testing` — behavior tests alongside visual tests
- `fake-data` — factories for consistent snapshot data
- `design-system` — the components that most benefit from snapshot testing
