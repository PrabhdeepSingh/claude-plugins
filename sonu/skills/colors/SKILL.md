---
name: colors
description: >-
  Color reference for web interfaces: contrast measurement (APCA and WCAG), OKLCH conversion and palette generation, gamut and P3 fallbacks, one meaning per color. Use when measuring contrast or building or converting a palette. Not for which requirement applies ([[accessibility]]) or whole-screen audits ([[interface-review]]).
---

# Color that carries meaning

The project's own instructions win: where its AGENTS.md, CLAUDE.md, design system or established conventions say otherwise, follow them and skip that part of this. Use the project's existing kit and tokens; do not add components, on-screen text or options that the project's design does not call for.

**Respect the existing system.** Reuse the project's semantic tokens and its notation. Do not convert notation because this skill was loaded: a consistent hex or RGB token system is better than a second color representation introduced for one fix. Use OKLCH when the project already uses it, when the task is a new color system or a migration, or when asked to convert.

**Measure, and report before changing.** Contrast claims rest on a measurement, not an estimate. When a pair fails, report the pair, the measured value and the threshold it misses, and leave the colors alone unless asked to fix them. A project's colors are a design decision.

→ `references/accessibility-contrast.md` — the runnable WCAG check, APCA and WCAG thresholds, how to report a failing pair, and how to fix one on request. Read when a change touches a foreground and background pair.

## OKLCH

`oklch(L C H)` or `oklch(L C H / alpha)`. L runs 0–1 and is perceptually uniform, C runs from 0 to about 0.4 with a maximum that depends on L and H, and H is 0–360. Write three decimal places for L and C and up to three for H, drop trailing zeros, and use the slash for alpha. Equal L steps look equally bright and hue stays constant across lightness, which HSL does not give. Not every OKLCH value is displayable in sRGB, so high chroma needs a gamut check.

When converting, change the color values and nothing else. Leave CSS keywords (`currentColor`, `inherit`, `transparent`), gradient interpolation methods and colors in third-party configs that expect hex. Converting a whole file is a migration and needs to be in scope.

→ `references/palette-generation.md` — the scale algorithm, multi-hue ramps and dark-mode palettes. Read when a change creates or extends a palette.

→ `references/gamut-and-tailwind.md` — maximum chroma by hue, P3 fallbacks and Tailwind `@theme` scales. Read when a change uses a wide-gamut or high-chroma color, or edits a Tailwind theme.

→ `references/color-usage.md` — one meaning per color, role tokens, primary-action emphasis, increased-contrast variants. Read when a change assigns a color to a role.

## Thresholds

| Rule | Value |
| --- | --- |
| Light/dark boundary | L > 0.73 = light background → dark text; below it, light text still scores higher |
| Lightness gap (light bg) | Foreground L < 0.35 when background L > 0.9 |
| Lightness gap (dark bg) | Foreground L > 0.9 when background L < 0.25 |
| Hue drift threshold | > 10° spread across palette steps = visible drift |
| APCA body text | \|Lc\| >= 75 minimum, >= 90 preferred |
| APCA non-body text | \|Lc\| >= 60 minimum |
| WCAG 2 normal text | 4.5:1 AA, 7:1 AAA |
| Contrast fix (only when asked) | Adjust L first; preserve C and H when possible, then remeasure the rendered pair |

## Common mistakes

| Issue | Fix |
| --- | --- |
| Raw color bypasses the project's semantic tokens | Reuse or add the role token, in the project's notation |
| Same absolute C across different hues | Use the same C% (percentage of max) for consistent vividness |
| High chroma without a gamut check | Clamp to the max chroma for that L and H in sRGB |
| P3 color without an sRGB fallback | Add the `@media (color-gamut: p3)` pattern |
| Dark mode made by reversing the light palette | Start from the light palette, tune chroma and lightness, recheck every pair |
| Semantic token used outside its role | Add a token for the missing role; do not borrow by value |
| Palette verified only in light mode | Recheck every foreground and background pair in both appearances |

## Reference files

| File | What it answers |
|---|---|
| `references/accessibility-contrast.md` | APCA and WCAG checks, reporting failures, fixing on request |
| `references/palette-generation.md` | Generating scales, multi-hue ramps, dark-mode palettes |
| `references/gamut-and-tailwind.md` | Max chroma, P3 fallbacks, `@theme` scales, gamut clamping |
| `references/color-usage.md` | Semantic tokens, one meaning per color, primary-action emphasis, appearance variants |

## Provenance and maintenance

Last verified 2026-07. The volatile claims here are the platform and tooling ones: OKLCH's Baseline status and browser support, the `@media (color-gamut: p3)` fallback pattern, Tailwind's `@theme` syntax and version-specific behavior (in `references/gamut-and-tailwind.md`), and the APCA thresholds, which track a specification still under revision. Re-verify against current browser-support tables, the project's Tailwind version, and the current APCA and WCAG specifications before treating a specific number or support claim as still true; the perceptual-color-space reasoning and the one-meaning-per-color discipline are stable.
