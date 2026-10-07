# Changelog

All notable changes to `compose-microinteractions` are documented here.

Format: `[version] — date — summary`

---

## [1.0.0] — 2026-10-07

First release: the Jetpack Compose counterpart of `swiftui-microinteractions`, built from the [LegendaryAnimoAndroid](https://github.com/iAmVishal16/LegendaryAnimoAndroid) demos.

- **Physics Presets**: `spring(dampingRatio, stiffness)` tuples mapped to their SwiftUI analogs, the `dampingFraction` → `dampingRatio` porting trap, and when to use `animate*AsState` vs `Animatable` (2D motion as one `Animatable<Offset>`).
- **State & Code Rules**: zero-recomposition animation (defer reads into `offset { }`, `graphicsLayer { }`, `drawBehind { }`), `LaunchedEffect` and `derivedStateOf` as timers, and ripple-free taps.
- **Haptics**: a four-rung `rememberHaptics()` ladder on `View.performHapticFeedback`.
- **Draggable Value Control**: direction lock, rubber-band resistance, hold-at-edge auto-repeat, spring-back.
- **Rubber-Band Selection Indicator**: two edges on two springs, squash while stretched, label pop, drawn in `drawBehind` so the stretch never recomposes.
