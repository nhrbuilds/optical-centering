# optical-centering

An agent skill for fixing vertical optical imbalance in UI rows: text that never looks perfectly centered, and icons that hang low against their labels.

Flexbox centers boxes. Readers judge ink. A text line box reserves empty descender space, so uppercase labels and digits ride high inside their centered box while icons, which have no such asymmetry, center exactly. The result shows up in every chip, menu item, table header strip, and badge row: the icon sits low, the text sits high, and `align-items: center` cannot fix either because it is the thing doing the centering.

This skill teaches the discipline that makes the fix stick: measure the real glyph-ink position in the browser, act only on deltas of half a pixel or more, apply one correction on one side, and verify with a number.

## What it covers

- Why text and icons disagree vertically, and why both directions of the failure come from the same mechanism.
- Three measurement probes, from a cheap DOM rect delta to canvas ink metrics to pixel-level ground truth at 4x device scale.
- The fix ladder: `line-height: 1` for uppercase and numeric labels, label or icon nudges of one pixel, the `translate` property when `transform` would clobber a rotation, cap-band alignment for mixed-case labels, and same-slot absolute positioning for badge-versus-actions crossfades.
- The anti-patterns that cause the churn this skill exists to stop.

## Install

Copy or symlink the folder into your agent's skills directory:

```sh
git clone https://github.com/nhrbuilds/optical-centering
ln -s "$(pwd)/optical-centering" ~/.claude/skills/optical-centering   # Claude Code
ln -s "$(pwd)/optical-centering" ~/.zcode/skills/optical-centering   # ZCode
ln -s "$(pwd)/optical-centering" ~/.agents/skills/optical-centering  # generic
```

Or install from npm and link the shipped folder:

```sh
npm i @nhrbuilds/optical-centering
ln -s node_modules/@nhrbuilds/optical-centering ~/.claude/skills/optical-centering
```

## Usage

Ask your agent to load the skill, or invoke it directly:

```
/optical-centering
```

Typical prompt: "the icons in this menu hang below their labels, fix it with optical-centering." The agent measures the affected rows, reports the before values, applies the smallest fix on the measured side, and re-reports.

## Requirements

Works wherever your agent can render HTML in a browser engine. The probes use standard DOM and Canvas APIs; the pixel probe uses Playwright for screenshots. No dependencies ship with the skill.

## License

MIT
