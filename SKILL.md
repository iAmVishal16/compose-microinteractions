---
name: compose-microinteractions
description: Generate premium Jetpack Compose micro-interactions in the legendary-Animo style — spring physics, zero-recomposition gestures, rubber-band stretch, frosted glass and backdrop blur, swipe-card decks, haptic ladders, complete compilable Kotlin files.
---

Generate a complete, compilable Jetpack Compose animation file in the legendary-Animo style. No placeholders. No TODO comments.

## When to Use
- Building a Compose button, tab bar, slider, counter, or drag interaction with premium physics
- Porting a SwiftUI micro-interaction to Android / Compose
- Need haptic feedback timed to state changes (the CoreHaptics analog)
- Creating a drag interaction with resistance, spring-back, auto-repeat, or a threshold trigger
- Building a selection indicator that stretches, squashes, or pops instead of sliding flat
- Building a frosted-glass or blurred-backdrop surface (Compose has no backdrop blur — see *Frosted Glass*)
- Building a swipe-card deck, or anything where a second property should follow a drag
- Editing an existing Compose animation file

## Mode Detection
- **Edit mode**: prompt contains "edit", "update", "add X to", "change", or a `.kt` filename → read the file first, change only what's asked, overwrite the file on disk
- **Create mode**: everything else → generate a new complete file and write it to disk

---

## Physics Presets

Compose's `spring()` takes **`dampingRatio` + `stiffness`** — there is no `dampingFraction` or `response`. Porting the SwiftUI names is a compile error. Lower `dampingRatio` = more overshoot; higher `stiffness` = faster.

**Converting from the iOS skill.** For mass 1:

```
stiffness    = (2 * PI / response)^2
dampingRatio = dampingFraction
```

An `interpolatingSpring(stiffness:damping:)` converts as `dampingRatio = damping / (2 * sqrt(stiffness))`.

The rows below are the `swiftui-microinteractions` presets, by the same names, run through that formula. Use these names in prompts and comments so a demo ported either way keeps its feel — earlier versions of this table were roughly 1.5× stiffer than the iOS values they claimed to match.

| Feel | Compose spec | iOS preset |
|---|---|---|
| Snap / bounce | `spring(dampingRatio = 0.50f, stiffness = 322f)` | `.spring(response: 0.35, dampingFraction: 0.5)` |
| UI pop | `spring(dampingRatio = 0.60f, stiffness = 322f)` | `.spring(response: 0.35, dampingFraction: 0.6)` |
| Standard settle | `spring(dampingRatio = 0.65f, stiffness = 247f)` | `.spring(response: 0.4, dampingFraction: 0.65)` |
| Physics settle | `spring(dampingRatio = 0.70f, stiffness = 195f)` | `.spring(response: 0.45, dampingFraction: 0.7)` |
| Slow morph | `spring(dampingRatio = 0.80f, stiffness = 110f)` | `.spring(response: 0.6, dampingFraction: 0.8)` |
| Precision stiff | `spring(dampingRatio = 0.74f, stiffness = 220f)` | `.interpolatingSpring(stiffness: 220, damping: 22)` |
| Dial / scrub | `spring(dampingRatio = 0.70f, stiffness = 439f)` | `.interactiveSpring(response: 0.3, dampingFraction: 0.7)` |
| Screen transition | `spring(dampingRatio = 1f, stiffness = 247f)` | `.spring(response: 0.4, dampingFraction: 1.0)` |

`Screen transition` is critically damped on purpose — see *Judging It*. Where a value is close to a built-in (`Spring.StiffnessLow` = 200f, `Spring.StiffnessMediumLow` = 400f, `Spring.DampingRatioNoBouncy` = 1f) either is fine; the explicit float is what matches iOS.

Never write a bare `spring()`. If the user supplies values, use them verbatim. Map feel words: "stretchy" → Snap/bounce, "snappy" → UI pop, "melts" → Slow morph.

Keep each tuple as a named constant in the file so it can be tuned in one place.

**Which API:**
- `animateFloatAsState` / `animateColorAsState` / `animateDpAsState` — the value follows a target (declarative, like `.animation(value:)`).
- `Animatable` — you drive it imperatively from gestures or effects (`snapTo`, `animateTo`). Use it for drag + spring-back and for "snap then bounce" pops.
- 2D motion uses **one** `Animatable(Offset.Zero, Offset.VectorConverter)`, never two `Float` animatables — two separate axes desync on spring-back.

---

## Derive, Don't Animate Alongside

**Two animations for one event always desync**, and that desync is what makes motion feel **unglued** — a card caught tilted at an angle that doesn't match where it is, a squash that outlasts the motion that caused it. (Not "rubbery": rubber-band is a compliment in this skill, see *Rubber-Band Selection Indicator*.)

If a second property follows from the first, **compute it — never animate it**:

| Thing | Animated separately (wrong) | Derived (right) |
|---|---|---|
| Swipe-card tilt | `animateFloatAsState(if (dragging) 15f else 0f)` | `rotationZ = drag.value.x / size.width * MAX_TILT_DEGREES` |
| Squash of a dragged body | its own spring | the lag vector between finger and body |
| Motion blur | a blur that fades in on drag start | taps sampled along the measured velocity |
| Rotation during a shape morph | a second `animateFloatAsState` | `morphProgress * ROTATION_PER_STEP_DEGREES` |
| Cards behind a swipe deck | an animation triggered on commit | the same displacement the front card reads |

The rubber-band tab bar below is the one case where two animations are right — and note *why*: the springs are not animating one property each, they are the **two ends of the same property**, and the gap between them is the effect.

**Corollary — only when position has several sources.** Prefer the velocity you are already given: `Animatable.velocity` is exact for a spring, and a drag already has `VelocityTracker`. Differentiate position yourself only when a finger, a spring, a fling *and* a clamp all move the same thing and none of them knows about the others. Then do it properly, or it is noise:

```kotlin
// In the frame loop — real dt from the frame clock, and smoothed.
val dt = (now - previousFrame) / 1_000_000_000f
val instant = (position - lastPosition) / dt
velocity += (instant - velocity) * 0.35f   // one jittery frame must not become a throw
```

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

### Constant naming
Kotlin's idiom, and this skill's: `private const val SCREAMING_SNAKE` at file top (the iOS skill says camelCase — that is correct *there*, don't port it). **Put the unit in the name.** `TILT` reads like a 0–1 factor; `MAX_TILT_DEGREES` cannot be misread. Same for `_SECONDS`, `_PX`, `_DP`.

### Composite in one draw scope
Anything built from the same content twice — a frosted panel, a lens, a reflection — is tempting to build as a second composable nested inside the first and offset back into register. **Don't.** Two layout trees then have to agree about a position, and they will not.

Do it in one `DrawScope` — draw the content, then `clipPath` the panel's shape and draw it again differently. A clip costs nothing, there is no size to negotiate, and register stops being something to get right: it falls out of using one function on one clock.

**`size()` vs `requiredSize()`** — needed only when a nested copy is genuinely unavoidable (a `graphicsLayer` whose `RenderEffect` must filter real composables, say). A child is measured with its **parent's** constraints, so `Modifier.size()` is coerced into them and fails *silently*: a screen-sized copy inside a 230dp panel comes out 230dp with the whole image squeezed into it, which reads as a mystery rectangle rather than a sizing bug. `requiredSize()` ignores incoming constraints.

But it brings its own trap: **content larger than its parent is centred on it, not anchored top-left**, so the offset that was supposed to put the copy back in register is now measured from the wrong origin. Prefer the single draw scope above; reach for `requiredSize` only when you cannot.

### Taps without ripples
Custom-styled surfaces use `clickable(interactionSource = remember { MutableInteractionSource() }, indication = null)`. The Material ripple fights a custom pop or highlight.

---

## Haptics

`View.performHapticFeedback` via `LocalView.current` works from API 24 and respects the user's system haptic setting. Wrap it as a ladder that mirrors the iOS one:

**Check the project first.** Search for an existing `rememberHaptics()` / `Haptics` before declaring one — every demo redeclaring it is a build error, not a duplicate.

The rungs are named after the iOS ladder, *not* after what the Android constant is called. Someone porting a demo calls `selection()` for a scrub step and must get a scrub step:

```kotlin
class Haptics(private val view: View) {
    fun light()     { view.performHapticFeedback(HapticFeedbackConstants.KEYBOARD_TAP) }  // drag start / touch down
    fun medium()    { view.performHapticFeedback(HapticFeedbackConstants.CONTEXT_CLICK) } // threshold crossed / toggle
    fun heavy()     { view.performHapticFeedback(HapticFeedbackConstants.LONG_PRESS) }    // commit / destructive
    fun selection() { view.performHapticFeedback(HapticFeedbackConstants.CLOCK_TICK) }    // discrete scrub step
}

@Composable
fun rememberHaptics(): Haptics {
    val view = LocalView.current
    return remember(view) { Haptics(view) }
}
```

`View.performHapticFeedback` via `LocalView.current` works from API 24 and respects the user's system haptic setting.

### When to add haptics vs. skip

**Add** — a threshold that triggers something irreversible (`heavy()` at the crossing); a toggle between two meaningful states (`medium()` on commit); a destructive action confirmed (`heavy()`); a scrub or dial across discrete points (`selection()` per step); a morph or metaball fuse/separate completing (`medium()`).

**Skip** — showcase loops with no user action; loading indicators and ambient background motion; any prompt that describes no interaction at all.

### Firing rules

- Fire at **state-change callbacks** (a count changed, a tab changed), never inside the per-frame drag loop.
- Only on a real change: `if (i != selected) { haptics.selection(); selected = i }`.
- **Gate-and-rearm with hysteresis.** A trigger derived from a continuous value crossing a threshold must fire once per *crossing* — not once ever, and not once per frame while it sits on the line. Use **two thresholds**, not a frame count: a frame count means something different at 60Hz and 120Hz.

```kotlin
// Gate at 0.55, rearm at 0.45. The dead band between them is what stops the chatter
// when a value hovers exactly on the boundary.
if (!fused && strength > 0.55f) { fused = true;  haptics.medium() }
else if (fused && strength < 0.45f) { fused = false }
```

  If a dead band genuinely doesn't fit the quantity, debounce instead — but measure it in **milliseconds**, never frames.

---

## Frosted Glass (no backdrop blur)

**Compose has no backdrop blur.** `Modifier.blur` blurs a composable's *own* content; there is no "blur what is behind me". When you own the background, draw it twice — once sharp, once soft, the soft copy clipped to the panel (see *Composite in one draw scope*).

Make the soft copy **a small decode drawn large**. Magnifying is a tent filter, and tenting a tiny image is a blur the GPU performs as part of drawing it: one `drawImage`, no `RenderEffect`, no extra layer — and it works below API 31, where `Modifier.blur` is a no-op.

Two traps, both of which present as "the blur is broken":

- **Blur comes from the box filter on the way *down*, not from ending up small.** `createScaledBitmap` takes **one** bilinear step however far it is asked to travel, so 1200px → 96px *samples* the picture rather than averaging it: a handful of pixels read, hundreds ignored. On anything grainy that is a speckle generator. Halve repeatedly so every step is a true 2×2 average.
- **Never magnify more than about 6×.** Past that the tent filter shows as blocks with diamond seams. Blur harder by averaging more — down-and-back-up passes at the same size — not by starting smaller.

```kotlin
private const val SOFT_WIDTH_PX = 150   // small enough to blur, large enough to upscale cleanly
private const val FROST_PASSES = 3      // the knob for "more frost" — not SOFT_WIDTH_PX

/** Halving repeatedly: every step a true 2x2 average, which a single big jump is not. */
private fun downscaleByHalves(source: Bitmap, targetWidth: Int): Bitmap {
    var current = source
    while (current.width / 2 >= targetWidth) {
        val next = Bitmap.createScaledBitmap(
            current, current.width / 2, (current.height / 2).coerceAtLeast(1), true,
        )
        if (current !== source) current.recycle()
        current = next
    }
    return current
}

/** Widens the blur without losing resolution: halve, scale straight back, repeat. */
private fun soften(source: Bitmap, passes: Int): Bitmap {
    var current = source
    repeat(passes) {
        val down = Bitmap.createScaledBitmap(
            current, (current.width / 2).coerceAtLeast(1), (current.height / 2).coerceAtLeast(1), true,
        )
        val up = Bitmap.createScaledBitmap(down, source.width, source.height, true)
        down.recycle()
        if (current !== source) current.recycle()
        current = up
    }
    return current
}
```

**A heavy blur will band.** Blurred means nearly flat, and a nearly flat gradient written at eight bits a channel rounds whole regions to one level; the boundaries appear as contour lines tracing the image underneath. **Blurring harder makes it worse.** Two fixes, either or both:

- Dither with noise smaller than one output level — **±0.5/255**, i.e. `(hash(coord) - 0.5) * (1.0 / 255.0)` in a shader, or a tiled noise layer at ~3.5% alpha over the panel.
- Let a little of the sharp image through — **8–12%**. Real frosted glass is not a perfect diffuser either, and the direct component carries the fine detail that rides over the steps.

**Keep scrims off a frosted panel.** A gradient dark enough to make white type readable over a bright backdrop is dark enough to *see*, and then the panel is no longer glass — it is a tinted rectangle with frost at the top. Put the contrast on the glyphs (`TextStyle(shadow = Shadow(...))`).

This is scoped to the panel itself. A scrim **over a full-bleed blurred backdrop** is correct and the iOS skill prescribes it (black 0.55 → clear → 0.70). The rule is: scrim the backdrop, never the glass.

**Give the pane thickness.** Frosted glass does not blur to its edge — the last few millimetres are a curved shoulder that refracts, gathering what lies just *outside* the panel into a bright lip. A blur can never produce that (a blur mixes pixels where they are; refraction fetches them from elsewhere), and it is most of what makes a pane read as a physical object. A signed-distance function for a rounded box gives both the distance to the rim and, through its gradient, the direction to bend in.

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
| Frosted panel | background drawn twice, soft copy clipped in one `DrawScope` + refracting rim + dither |
| Swipe-card deck | one `Animatable<Offset>`, tilt derived from it, one threshold choosing spring-back vs. tween-out |

---

## Generation Defaults

Rules for what to emit when the prompt doesn't say, and for checking the result.

**Showcase vs. interaction decides whether it loops.** If the prompt describes no user action, it is *showcase*: add an idle loop. If it describes a gesture, it is *interaction*: **no auto-loop** — the user's finger is the clock.

**Every showcase effect ships with an idle drift**, because anything that moves only while touched looks dead between touches. One cycle every **6–12 s** — fast enough to prove it is live, slow enough not to become the subject. Drive it from `rememberInfiniteTransition` or a `withFrameNanos` clock with `LinearEasing` or a sine, **never a spring**: springs look jittery when looped. Skip haptics entirely for ambient motion.

**When generating a glass or frost demo, use a detailed backdrop** — a photograph, stripes, or a grid — **never a soft gradient.** A frosted panel over a soft dark gradient is invisible *while working perfectly*, because a blur of something already soft looks identical to the thing itself. Detail at every scale, hard edges, and straight lines; a bend is only visible against a line that was straight.

**Opening a screen is the worst possible moment to be slow.** During a navigation both screens are composed and drawing, a shared-element transition adds an overlay on top, and the new screen is doing its first-frame work underneath. Two defaults:

- Use the **Screen transition** preset (`spring(dampingRatio = 1f, stiffness = 247f)`). A soft spring chosen because it looks good in isolation spends its whole budget here.
- Get first-frame work off the opening frame. **`LaunchedEffect` alone is not enough** — it runs on the main thread, one frame later, so hundreds of text measurements or an image decode still stutter the transition. Move the *work*, not just the timing:

```kotlin
val bitmap by produceState<ImageBitmap?>(null, res) {
    value = withContext(Dispatchers.Default) { decodeScaled(res) }  // off the main thread
}
// ...and render something until it arrives, rather than nothing.
bitmap?.let { /* draw it */ } ?: PlaceholderSurface()
```

  Shader compilation is the exception: it must happen on the GL/UI thread, so it cannot be moved. Build the shader once in `remember`, keep the `shader == null` path you already need for older API levels, and accept one frame of fallback.

---

## Output Rules
- One complete `.kt` file per demo: package line, all imports, the composable, and a `@Preview`.
- No `material-icons-extended` dependency — draw small icons inline as `ImageVector` or with `Canvas`.
- Verify the stretch or bounce by feel on a device or emulator. `adb screencap` and `screenrecord` are too slow to catch a sub-second stretch frame.

⚙️  compose-microinteractions v1.1.0
