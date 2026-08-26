---
name: colors
description: >-
  Color systems for web interfaces — OKLCH conversion and palette generation, contrast measurement (APCA/WCAG), gamut and P3 fallbacks, theming, one meaning per color. INVOKE PROACTIVELY when converting color values, building or extending a palette or token set, checking or reporting contrast, or theming light and dark appearances. Which contrast requirement applies: [[accessibility]]; text rendering: [[typography]]; whole-screen audits: [[interface-review]].
---

# Color that carries meaning

OKLCH is a perceptually uniform color space where lightness, chroma, and hue are useful design controls. Use it when the project already uses OKLCH, when creating a new color system, or when the user asks for conversion or palette work. Otherwise preserve the project's established tokens and notation: a consistent hex or RGB token system is better than introducing a second color representation for an isolated fix.

## Core Principles

### 1. Use a Perceptual Color Space

- **Respect the existing system.** Do not convert notation merely because this skill was loaded. Reuse the project's semantic tokens and authoring format unless the task includes a color-system migration.
- **Color carries one meaning.** A hue that means "link" cannot also be decoration, and a semantic token is never borrowed by value for a role it doesn't name. → `references/color-usage.md` — semantic-token roles, one-meaning-per-color, primary-action emphasis, and light/dark appearance variants, read when this change assigns a color to a role or adds a control's color treatment.
- **Perceptual uniformity.** Equal L steps = equal brightness. `oklch(0.5 ...)` is visually mid. HSL's `lightness: 50%` varies wildly by hue.
- **Stable hue.** HSL blue shifts toward purple as lightness changes. OKLCH hue stays constant across the full lightness range.
- **Independent chroma.** Chroma is an absolute measure of colorfulness that doesn't depend on lightness. HSL saturation does.
- **Finite gamut.** Not every oklch value maps to a displayable sRGB color. High-chroma values at certain hues will clip; gamut awareness is required.

### 2. Write and Format OKLCH Consistently

```
oklch(L C H)
oklch(L C H / alpha)
```

| Channel | Range | Description |
| --- | --- | --- |
| L (Lightness) | 0–1 | 0 = black, 1 = white. Perceptually uniform. |
| C (Chroma) | 0–~0.4 | Colorfulness. 0 = gray. Max depends on L and H. |
| H (Hue) | 0–360 | Hue angle in degrees. |
| alpha | 0–1 | Optional transparency. Slash syntax. |

```css
oklch(0.637 0.237 25.331)
oklch(0.8 0.05 200 / 0.5)
```

Use three decimal places for L and C and up to three for H. Drop trailing zeros and format `-0` as `0`. OKLCH is Baseline 2023; when support requirements are unusually broad, check the target project's browser matrix instead of relying on a fixed global-coverage percentage.

→ `references/color-conversion.md` — the conversion procedures from hex, rgb, and hsl into oklch, read when this change converts an existing color value.

→ `references/palette-generation.md` — building scales, multi-hue ramps, and dark-mode palettes, read when this change creates or extends a palette rather than a single value.

### 3. Measure Contrast, Gamut, and Palette Behavior

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

→ `references/accessibility-contrast.md` — how to run APCA and WCAG checks, report a failing pair, and fix one when asked, read when this change touches a foreground/background pair or a contrast complaint.

→ `references/gamut-and-tailwind.md` — P3 fallback patterns, `@theme` scale setup, and gamut clamping, read when this change uses a wide-gamut or high-chroma color, or edits a Tailwind theme.

## Common Mistakes

| Issue | Fix |
| --- | --- |
| Raw color bypasses the project's semantic token system | Reuse or add the correct role token in the project's existing notation |
| Isolated OKLCH value introduced into a hex/RGB codebase | Preserve the established notation unless the task includes a color-system migration |
| HSL palette ramp with hue drift | Rebuild with constant oklch hue |
| Failing contrast (check foreground vs its background using APCA) | Report the pair, its measured Lc and the threshold it misses; change colors only when asked (then adjust L, keep C and H) |
| High chroma without gamut check | Clamp to max chroma for the L/H in sRGB |
| Same absolute C across different hues | Use same C% (percentage of max) for consistent vividness |
| P3 color without sRGB fallback | Add `@media (color-gamut: p3)` pattern |
| Dark mode created by mechanically reversing the light palette | Use the light palette as a starting point, then tune chroma and lightness and recheck every foreground/background pair |
| Hex in Tailwind v4 `@theme` | Convert to oklch values |
| Alpha with comma syntax | Use slash: `oklch(L C H / alpha)` |
| Same hue means two different things (link color reused decoratively) | One color, one meaning; give the second use a neutral |
| Semantic token used outside its role (separator as text) | Add a token for the missing role; never borrow by value |
| Several colored control backgrounds in one view | Fill only the single primary action; secondaries stay neutral |
| Palette verified only in light mode | Recheck every foreground/background pair in both appearances |

## Review Output Format

Use this format only when the user asks for a standalone color review. When [[interface-review]] orchestrates the review, provide domain evidence and findings to that skill and let its output format, severity scale, consolidation rules, cap, and verdict take precedence.

Report all confirmed findings as one markdown table grouped by principle — `| Severity | Location | Before | After | Why |`, never separate "Before:" / "After:" lines. **Location** cites `path/to/file:line` (or the exact screen and component when there are no source files); **Before / After** show the current implementation and an actionable replacement; **Why** names the violated principle and its impact. Consolidate a repeated systemic issue into one row listing every affected location; omit principles with no findings. **Severity**: `HIGH` makes content unreadable or assigns a misleading semantic color; `MEDIUM` creates a noticeable theme, gamut, or consistency failure; `LOW` is isolated polish.

After the findings: **Verification** — list the exact checks run and their observed results (contrast measurements, gamut checks, both light and dark appearances when applicable), and name any check not run. Then **Verdict**: `Block` if any `HIGH` finding remains, `Needs changes` if only `MEDIUM` or `LOW` findings remain, `Approve` only when no actionable findings remain. When there are no findings, omit the table, state "No actionable color findings", report verification, and end with `Approve`.

## Reference files

| File | What it answers |
|---|---|
| `references/color-conversion.md` | Converting hex, rgb, and hsl into oklch (§2) |
| `references/palette-generation.md` | Generating scales, multi-hue ramps, dark-mode palettes (§2) |
| `references/accessibility-contrast.md` | APCA and WCAG checks, reporting failures, fixing on request (§3) |
| `references/gamut-and-tailwind.md` | P3 fallbacks, `@theme` scales, gamut clamping (§1, §3) |
| `references/color-usage.md` | Semantic tokens, one meaning per color, primary-action emphasis, appearance variants (§1) |

## Provenance and maintenance

Last verified 2026-07. The volatile claims here are the platform and tooling ones: OKLCH's Baseline status and browser support (§2), the `@media (color-gamut: p3)` fallback pattern, Tailwind's `@theme` syntax and version-specific behavior (in `references/gamut-and-tailwind.md`), and the APCA thresholds in §3, which track a specification still under revision. Re-verify against current browser-support tables, the project's Tailwind version, and the current APCA and WCAG specifications before treating a specific number or support claim as still true; the perceptual-color-space reasoning and the one-meaning-per-color discipline are stable.
