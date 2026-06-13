---
name: compose-animation
description: >
  Compose animation APIs and patterns for Android UI.
  Load this skill when adding transitions, animated visibility,
  motion between screens, or any time-based UI change in Compose.
---

# Compose Animation

## Overview
Compose provides a rich animation system ranging from simple animated value changes to complex physics-based motion. Animations improve UX by providing visual continuity and feedback. Choose the simplest API that meets the need.

---

## Core Principles

- Choose the **simplest API** for the job — don't use `Animatable` when `animateFloatAsState` works
- Animations should **respect user accessibility settings** — check `reduceMotion`
- Never block the main thread — all Compose animations are coroutine-based
- Animate **state transitions**, not imperative sequences
- Use `AnimatedVisibility` and `AnimatedContent` before reaching for custom animations

---

## Animation API Hierarchy

```
Simple value change      → animateXxxAsState
Visibility show/hide     → AnimatedVisibility
Content swap             → AnimatedContent
Screen transitions       → NavHost transitions
Multi-value / sequenced  → updateTransition
Full control / physics   → Animatable
Infinite                 → rememberInfiniteTransition
```

---

## animateXxxAsState — Simple Value Animation

```kotlin
// ✅ Animate a single value when state changes
@Composable
fun LikeButton(isLiked: Boolean, onToggle: () -> Unit) {
    val scale by animateFloatAsState(
        targetValue = if (isLiked) 1.2f else 1f,
        animationSpec = spring(dampingRatio = Spring.DampingRatioMediumBouncy),
        label = "like_scale"  // ✅ always provide label for tooling
    )
    val color by animateColorAsState(
        targetValue = if (isLiked) Color.Red else Color.Gray,
        label = "like_color"
    )

    Icon(
        imageVector = Icons.Default.Favorite,
        contentDescription = null,
        tint = color,
        modifier = Modifier
            .scale(scale)
            .clickable { onToggle() }
    )
}

// ✅ Animate size
val size by animateDpAsState(
    targetValue = if (isExpanded) 200.dp else 100.dp,
    animationSpec = tween(durationMillis = 300),
    label = "card_size"
)
```

---

## AnimatedVisibility

```kotlin
// ✅ Show/hide with animation
@Composable
fun FilterPanel(isVisible: Boolean, content: @Composable () -> Unit) {
    AnimatedVisibility(
        visible = isVisible,
        enter = slideInVertically() + fadeIn(),
        exit = slideOutVertically() + fadeOut()
    ) {
        content()
    }
}

// ✅ List item appearance
LazyColumn {
    items(users, key = { it.id }) { user ->
        AnimatedVisibility(
            visible = true,
            enter = fadeIn() + expandVertically()
        ) {
            UserCard(user = user)
        }
    }
}
```

---

## AnimatedContent

```kotlin
// ✅ Animate between different content states
@Composable
fun CounterDisplay(count: Int) {
    AnimatedContent(
        targetState = count,
        transitionSpec = {
            if (targetState > initialState) {
                slideInVertically { -it } + fadeIn() togetherWith
                slideOutVertically { it } + fadeOut()
            } else {
                slideInVertically { it } + fadeIn() togetherWith
                slideOutVertically { -it } + fadeOut()
            }
        },
        label = "counter"
    ) { targetCount ->
        Text(text = "$targetCount", style = MaterialTheme.typography.displayLarge)
    }
}

// ✅ Switch between loading/content/error states
@Composable
fun ScreenContent(state: UiState) {
    AnimatedContent(
        targetState = state,
        label = "screen_content"
    ) { currentState ->
        when (currentState) {
            is UiState.Loading -> LoadingView()
            is UiState.Success -> ContentView(currentState.data)
            is UiState.Error   -> ErrorView(currentState.message)
        }
    }
}
```

---

## updateTransition — Coordinated Multi-Value Animation

```kotlin
// ✅ Animate multiple values tied to the same state
@Composable
fun ExpandableCard(isExpanded: Boolean) {
    val transition = updateTransition(targetState = isExpanded, label = "card_transition")

    val cornerRadius by transition.animateDp(label = "corner") {
        if (it) 0.dp else 16.dp
    }
    val elevation by transition.animateDp(label = "elevation") {
        if (it) 8.dp else 2.dp
    }
    val backgroundColor by transition.animateColor(label = "bg") {
        if (it) MaterialTheme.colorScheme.primaryContainer
        else MaterialTheme.colorScheme.surface
    }

    Card(
        shape = RoundedCornerShape(cornerRadius),
        colors = CardDefaults.cardColors(containerColor = backgroundColor)
    ) { ... }
}
```

---

## Screen Transitions (Compose Navigation)

```kotlin
// ✅ Global transitions in NavHost
NavHost(
    navController = navController,
    startDestination = HomeRoute,
    enterTransition = { slideInHorizontally { it } + fadeIn() },
    exitTransition = { slideOutHorizontally { -it } + fadeOut() },
    popEnterTransition = { slideInHorizontally { -it } + fadeIn() },
    popExitTransition = { slideOutHorizontally { it } + fadeOut() }
) {
    composable<HomeRoute> { HomeScreen() }
    composable<DetailRoute> { DetailScreen() }
}

// ✅ Per-route transitions
composable<DetailRoute>(
    enterTransition = { fadeIn(tween(300)) },
    exitTransition = { fadeOut(tween(300)) }
) { DetailScreen() }
```

---

## Animation Specs

```kotlin
// tween — linear/eased time-based
tween(durationMillis = 300, easing = FastOutSlowInEasing)

// spring — physics-based, natural feel
spring(dampingRatio = Spring.DampingRatioMediumBouncy, stiffness = Spring.StiffnessMedium)

// keyframes — precise time-value control
keyframes {
    durationMillis = 400
    0f at 0
    1.2f at 200
    1f at 400
}

// snap — instant, no animation
snap()
```

---

## Accessibility — Reduced Motion

```kotlin
// ✅ Respect user's reduce motion setting
@Composable
fun AccessibleAnimation(isVisible: Boolean) {
    val reduceMotion = LocalReduceMotion.current  // not yet in stable — use workaround

    // Workaround until LocalReduceMotion is stable
    val animationDuration = if (isSystemInDarkTheme()) 0 else 300

    AnimatedVisibility(
        visible = isVisible,
        enter = fadeIn(tween(animationDuration)),
        exit = fadeOut(tween(animationDuration))
    ) { ... }
}
```

---

## Infinite Animation

```kotlin
// ✅ Loading shimmer / pulse
@Composable
fun PulsingDot() {
    val infiniteTransition = rememberInfiniteTransition(label = "pulse")
    val scale by infiniteTransition.animateFloat(
        initialValue = 0.8f,
        targetValue = 1.2f,
        animationSpec = infiniteRepeatable(
            animation = tween(600),
            repeatMode = RepeatMode.Reverse
        ),
        label = "scale"
    )
    Box(modifier = Modifier.scale(scale).size(12.dp).background(Color.Blue, CircleShape))
}
```

---

## Anti-Patterns

- Animating on every recomposition — animation state must be stable
- Missing `label` parameter — breaks Animation Inspector in Android Studio
- Using `Thread.sleep()` or `delay()` for animation timing — use animation specs
- Animating too many properties simultaneously — causes jank
- Not testing with reduced motion — accessibility oversight
- Using `LaunchedEffect` + manual delay for show/hide — use `AnimatedVisibility`

---

## Related Skills
- `compose` — Compose fundamentals and state
- `compose-performance` — avoiding recomposition during animation
- `material3` — Material 3 motion system
- `loading-strategy` — animated loading states
