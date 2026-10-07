# compose-microinteractions

[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

Premium Jetpack Compose animation and interaction skill for AI coding agents. Generate production-ready micro-interactions from plain English prompts.

The Android counterpart of [swiftui-microinteractions](https://github.com/iAmVishal16/swiftui-microinteractions), built from [LegendaryAnimoAndroid](https://github.com/iAmVishal16/LegendaryAnimoAndroid), the Compose port of [legendary-Animo](https://github.com/iAmVishal16/legendary-Animo).

---

## Usage

```
/compose-microinteractions a tab bar where the pill stretches toward the tapped tab
/compose-microinteractions a counter knob you drag sideways, hold at the edge to auto-repeat
/compose-microinteractions edit TabBarDemo.kt — make the squash stronger
```

Each prompt writes a complete `.kt` file to your project. Supports create and edit modes.

## What it knows

- Spring presets as `dampingRatio` / `stiffness`, mapped from SwiftUI
- Zero-recomposition animation: deferred reads in `offset { }`, `graphicsLayer { }`, `drawBehind { }`
- A haptic ladder on `View.performHapticFeedback`
- Draggable value controls: direction lock, rubber-band resistance, auto-repeat, spring-back
- Rubber-band selection indicators: two edges on two springs, squash, label pop

---

## Install

### Claude Code

```bash
mkdir -p ~/.claude/skills/compose-microinteractions
curl -o ~/.claude/skills/compose-microinteractions/SKILL.md \
  https://raw.githubusercontent.com/iAmVishal16/compose-microinteractions/main/SKILL.md
```

Then invoke `/compose-microinteractions`.

### Any other AI agent

Paste this into Cursor, Windsurf, Copilot, Codex or similar:

```
Fetch https://raw.githubusercontent.com/iAmVishal16/compose-microinteractions/main/SKILL.md
and install it as a skill/rule in your own rules or skills location, so you follow it
whenever I ask for Jetpack Compose animations or micro-interactions.
```

---

## Repo layout

```
compose-microinteractions/   ← repo root
  SKILL.md
  README.md
  CHANGELOG.md
  LICENSE
```

---

## Contributing

Contributions welcome. Open a PR to improve the skill, add patterns, or fix physics values.

## About

Built by [Vishal Paliwal](https://twitter.com/iamvishal16_ios), creator of [legendary-Animo](https://github.com/iAmVishal16/legendary-Animo).

Support the work: [Patreon](https://www.patreon.com/c/iamvishal16)

## License

MIT License. See [LICENSE](LICENSE) for details.
