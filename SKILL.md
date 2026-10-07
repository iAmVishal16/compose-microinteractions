---
name: compose-microinteractions
description: Generate premium Jetpack Compose micro-interactions in the legendary-Animo style — spring physics, zero-recomposition gestures, rubber-band stretch, haptic ladders, complete compilable Kotlin files.
---

Generate a complete, compilable Jetpack Compose animation file in the legendary-Animo style. No placeholders. No TODO comments.

## When to Use
- Building a Compose button, tab bar, slider, counter, or drag interaction with premium physics
- Porting a SwiftUI micro-interaction to Android / Compose
- Need haptic feedback timed to state changes (the CoreHaptics analog)
- Creating a drag interaction with resistance, spring-back, auto-repeat, or a threshold trigger
- Building a selection indicator that stretches, squashes, or pops instead of sliding flat
- Editing an existing Compose animation file

## Mode Detection
- **Edit mode**: prompt contains "edit", "update", "add X to", "change", or a `.kt` filename → read the file first, change only what's asked, overwrite the file on disk
- **Create mode**: everything else → generate a new complete file and write it to disk

---

## Physics Presets

Compose's `spring()` takes **`dampingRatio` + `stiffness`** — there is no `dampingFraction` or `response`. Porting the SwiftUI names is a compile error. Lower `dampingRatio` = more overshoot; higher `stiffness` = faster.

| Feel | Compose spec | SwiftUI analog |
|---|---|---|
| Snappy lead / UI pop | `spring(dampingRatio = 0.75f, stiffness = 700f)` | `.snappy` |
| Bouncy settle (spring-back) | `spring(dampingRatio = 0.58f, stiffness = 170f)` | `interpolatingSpring` |
| Soft trailing edge | `spring(dampingRatio = 0.60f, stiffness = 190f)` | `.bouncy` with long response |
| Label / icon pop | `spring(dampingRatio = 0.42f, stiffness = 620f)` | `.spring(0.3, 0.45)` |
| Color cross-fade | `spring(stiffness = 300f)` via `animateColorAsState` | `.animation(.smooth)` |

Keep each tuple as a named constant in the file so it can be tuned in one place.

**Which API:**
- `animateFloatAsState` / `animateColorAsState` / `animateDpAsState` — the value follows a target (declarative, like `.animation(value:)`).
- `Animatable` — you drive it imperatively from gestures or effects (`snapTo`, `animateTo`). Use it for drag + spring-back and for "snap then bounce" pops.
- 2D motion uses **one** `Animatable(Offset.Zero, Offset.VectorConverter)`, never two `Float` animatables — two separate axes desync on spring-back.

---

## State & Code Rules

### Zero-recomposition animation (the #1 Compose perf rule)
Never read a per-frame animated value in the composable body. Reading it there recomposes the whole subtree every frame (jank at 120Hz). Defer the read into a lambda that runs in the **layout** or **draw** phase:

| Need | Naive (recomposes every frame) | Correct (deferred) |
|---|---|---|
| Move | `Modifier.offset(x = anim.value.dp)` | `Modifier.offset { IntOffset(anim.value.x.roundToInt(), 0) }` |
| Scale / alpha / rotate | `Modifier.scale(anim.value)` | `Modifier.graphicsLayer { scaleX = anim.value; scaleY = anim.value }` |
| Size / shape that changes every frame | `Modifier.width(w)` driven by animation | draw it: `Modifier.drawBehind { drawRoundRect(...) }` reading the `State` inside |

When using `animate*AsState`, keep the `State` object (`val lead = animateFloatAsState(...)`, no `by`) and read `lead.value` inside the lambda — `by` delegation reads in composition.

### Effects as timers
- A one-shot reaction to a state change = `LaunchedEffect(key)`. It cancels and restarts when the key changes, which is exactly the behaviour you want for retriggerable pops.
- A repeating timer (the `Timer.scheduledTimer` analog) = `derivedStateOf` for the condition + `LaunchedEffect(condition)` with a `while (isActive)` loop: initial `delay(500)`, then `delay(100)` per repeat. It stops automatically when the condition flips false.
- `derivedStateOf` around a boolean read from a fast-changing value (e.g. "is held at the edge") so recomposition only happens when the boolean flips, not on every pixel.

### Taps without ripples
Custom-styled surfaces use `clickable(interactionSource = remember { MutableInteractionSource() }, indication = null)`. The Material ripple fights a custom pop or highlight.

---

## Haptics

`View.performHapticFeedback` via `LocalView.current` works from API 24 and respects the user's system haptic setting. Wrap it as a ladder that mirrors the iOS one:

```kotlin
class Haptics(private val view: View) {
    fun tick()      { view.performHapticFeedback(HapticFeedbackConstants.CLOCK_TICK) }    // step / scrub
    fun selection() { view.performHapticFeedback(HapticFeedbackConstants.KEYBOARD_TAP) }  // light tap
    fun medium()    { view.performHapticFeedback(HapticFeedbackConstants.CONTEXT_CLICK) } // toggle / commit
    fun heavy()     { view.performHapticFeedback(HapticFeedbackConstants.LONG_PRESS) }    // destructive / limit
}

@Composable
fun rememberHaptics(): Haptics {
    val view = LocalView.current
    return remember(view) { Haptics(view) }
}
```

- Fire haptics at **state-change callbacks** (a count changed, a tab changed), never inside the per-frame drag loop.
- Only fire on a real change: `if (i != selected) { haptics.selection(); selected = i }`.

---

## Draggable Value Control (counter / knob)

A knob you drag sideways to step a value, hold at the edge to auto-repeat, pull down to reveal a secondary action, and release to spring back.

- **One `Animatable<Offset>`** for the knob position; on release `animateTo(Offset.Zero, spring(0.58f, 170f))`.
- **Lock the drag direction** after ~2dp of travel. `detectDragGestures` reports both axes, so without a lock a sideways drag also starts the pull-down action.
- **Rubber-band, don't hard-clamp.** Scale the drag delta by a resistance that falls toward the limit: `resistance = (1f - abs(current.x) / limitX).coerceAtLeast(0f)`, then `snapTo(current + delta * resistance)`. The knob feels elastic and still springs back.
- **Hold-at-edge auto-repeat** = `derivedStateOf { abs(offset.value.x) >= edgePx }` + `LaunchedEffect(heldAtEdge)` loop (see Effects as timers).
- Position the knob with `Modifier.offset { }` (lambda), never the `Dp` overload.
- Haptic: `tick()` per step, `medium()` on clear/commit.

---

## Rubber-Band Selection Indicator (tab bar / segmented control)

A flat slide animates **one** offset. A liquid indicator animates its **two edges on two springs**.

1. Both edges target `selectedIndex.toFloat()`, but with different springs: a stiff **lead** (`0.75f / 700f`) and a soft **lag** (`0.60f / 190f`).
2. `left = slot * min(lead, lag)`, `width = slot * (max(lead, lag) - min(lead, lag)) + slot`.
3. While they desync, the pill spans old → new slot (**stretch**); once both settle it collapses back to one slot. It stretches in the right direction whichever way you tap, and the low-damped lag edge adds a small overshoot.
4. **Squash while stretched**: `stretch = abs(lead - lag).coerceIn(0f, 1f)`, `scaleY = 1f - stretch * 0.14f`. Stretch without squash reads as a rubber rectangle; with squash it reads as liquid.
5. **Pop the newly selected label** instead of only recoloring it: `LaunchedEffect(selected) { pop.snapTo(0.82f); pop.animateTo(1f, spring(0.42f, 620f)) }`, applied through `graphicsLayer` to the active label only.
6. Cross-fade label colors with `animateColorAsState` so nothing hard-cuts.

Because the pill's width changes every frame, draw it rather than sizing a `Box`, so the stretch never recomposes:

```kotlin
val lead = animateFloatAsState(selected.toFloat(), spring(dampingRatio = 0.75f, stiffness = 700f), label = "lead")
val lag  = animateFloatAsState(selected.toFloat(), spring(dampingRatio = 0.60f, stiffness = 190f), label = "lag")

Modifier.drawBehind {
    val slot = size.width / tabCount
    val l = min(lead.value, lag.value)
    val r = max(lead.value, lag.value)
    val squash = 1f - abs(lead.value - lag.value).coerceIn(0f, 1f) * 0.14f
    val h = (size.height - inset * 2) * squash
    drawRoundRect(
        brush = Brush.horizontalGradient(listOf(Color(0xFF7AA6FF), Color(0xFF8CF2BF))),
        topLeft = Offset(slot * l + inset, (size.height - h) / 2),
        size = Size(slot * (r - l) + slot - inset * 2, h),
        cornerRadius = CornerRadius(h / 2),
    )
}
```

---

## Archetypes

| Archetype | Core technique |
|---|---|
| Draggable value control | `Animatable<Offset>` + direction lock + rubber-band resistance + `derivedStateOf` auto-repeat |
| Rubber-band tab bar / segmented control | two edges on two springs + squash + label pop |

---

## Output Rules
- One complete `.kt` file per demo: package line, all imports, the composable, and a `@Preview`.
- No `material-icons-extended` dependency — draw small icons inline as `ImageVector` or with `Canvas`.
- Verify the stretch or bounce by feel on a device or emulator. `adb screencap` and `screenrecord` are too slow to catch a sub-second stretch frame.

⚙️  compose-microinteractions v1.0.0
