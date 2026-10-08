# Changelog

All notable changes to `compose-microinteractions` are documented here.

Format: `[version] — date — summary`

---

## [1.1.0] — 2026-10-09

Lessons from a second pass over [LegendaryAnimoAndroid](https://github.com/iAmVishal16/LegendaryAnimoAndroid), plus an alignment pass against `swiftui-microinteractions` so the two skills stay a matched pair.

**New sections**

- **Derive, Don't Animate Alongside**: two animations for one event always desync, and that desync is what makes motion feel unglued. Tilt from position, squash from lag, motion blur from measured velocity, rotation from morph progress — with a note on why the tab bar's two springs are not a counter-example. The velocity corollary is scoped: prefer `Animatable.velocity` and `VelocityTracker`, and differentiate position only when several sources move the same thing, with real `dt` and smoothing.
- **Frosted Glass (no backdrop blur)**: Compose has no backdrop blur, so draw the background twice and make the soft copy a small decode drawn large. The two traps that both present as "the blur is broken" — `createScaledBitmap` takes one bilinear step however far it travels, so a single large downscale samples instead of averaging; and magnifying past ~6× shows the tent filter as blocks. Banding and its two fixes with amplitudes. Kotlin for the halving downscale and the softening passes. Scoped scrim rule, and the refracting rim that gives a pane thickness.
- **Generation Defaults**: showcase vs. interaction decides whether an effect loops; idle drift at 6–12 s driven by a clock, never a spring; glass demos get a detailed backdrop, never a soft gradient; opening a screen uses the Screen transition preset and moves first-frame *work* off the main thread, not merely one frame later.

**Changed**

- **Physics Presets** now carry the `swiftui-microinteractions` preset names and values, with the conversion formula (`stiffness = (2π / response)²`, `dampingRatio = dampingFraction`). The previous table was roughly 1.5× stiffer than the iOS presets it claimed to match, so a demo ported preset-for-preset felt wrong. Adds a critically damped **Screen transition** row.
- **Haptics** rungs renamed to the iOS ladder — `light()`, `medium()`, `heavy()`, `selection()` — with `selection()` mapped to `CLOCK_TICK`, so a ported demo asking for a scrub step gets one. Adds the when-to-add-vs-skip lists and a project check before redeclaring the class. Threshold haptics are now **gate-and-rearm with hysteresis** (two thresholds, 0.55/0.45) rather than a frame count, which behaves differently at 60Hz and 120Hz.
- **State & Code Rules**: constant naming (`private const val SCREAMING_SNAKE`, unit in the name). "Composite in one draw scope" is the rule; `requiredSize()` is presented as the fallback it is, with the trap that oversized content is centred on its parent rather than anchored top-left.
- Frontmatter, **When to Use** and **Archetypes** now mention frosted glass and swipe decks, so the new material is reachable.

---

## [1.0.0] — 2026-10-07

First release: the Jetpack Compose counterpart of `swiftui-microinteractions`, built from the [LegendaryAnimoAndroid](https://github.com/iAmVishal16/LegendaryAnimoAndroid) demos.

- **Physics Presets**: `spring(dampingRatio, stiffness)` tuples mapped to their SwiftUI analogs, the `dampingFraction` → `dampingRatio` porting trap, and when to use `animate*AsState` vs `Animatable` (2D motion as one `Animatable<Offset>`).
- **State & Code Rules**: zero-recomposition animation (defer reads into `offset { }`, `graphicsLayer { }`, `drawBehind { }`), `LaunchedEffect` and `derivedStateOf` as timers, and ripple-free taps.
- **Haptics**: a four-rung `rememberHaptics()` ladder on `View.performHapticFeedback`.
- **Draggable Value Control**: direction lock, rubber-band resistance, hold-at-edge auto-repeat, spring-back.
- **Rubber-Band Selection Indicator**: two edges on two springs, squash while stretched, label pop, drawn in `drawBehind` so the stretch never recomposes.
