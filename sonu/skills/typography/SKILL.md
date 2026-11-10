---
name: typography
description: >-
  Typography defaults for web interfaces: type scale, sizes, line-height, measure, wrapping, tabular numbers, underlines, input sizing. Use when setting or reviewing text styles. Not for the words ([[ux-writing]]), heading semantics ([[accessibility]]), direction-aware layout ([[layout]]), contrast ([[colors]]) or whole-screen audits ([[interface-review]]).
---

# Typography

The project's own instructions win: where its AGENTS.md, CLAUDE.md, design system or established conventions say otherwise, follow them and skip that part of this. Use the project's existing kit and tokens; do not add components, on-screen text or options that the project's design does not call for.

Good typography is mostly restraint: a small scale, comfortable spacing and enough contrast. Express every change in the project's styling system and type scale.

**The typeface is the project's choice.** Applying or reviewing typography does not call for a new typeface. Use the product's type system unless the task asks for a type change, and do not introduce a paid or proprietary face. Rendering details such as wrapping and tabular numbers do not override the chosen family.

When reviewing, read the rendered page: check that the hierarchy holds at a glance, read a full paragraph, and resize the viewport to catch bad wrapping and truncation at real content lengths. With no rendered page, verify what the source can prove and name the visual checks as still to be done.

## Defaults

The values are this skill's defaults for a project with no type system of its own.

**Few sizes and weights.** Define a small scale and stay on it; name sizes by use (`text-body-sm`) on a team. Emphasis within a role is one weight step up, not a size change. → `references/spacing-and-sizing.md` — the role-based scale with sizes, line-heights and weights, and `text-box` trimming. Read when a change sets a size or a scale step.

**Sizes.** Long-form body starts near `16px`. Interface text can be smaller: about `14px` for inputs and menus, `13px` for captions, rarely below `12px`. Below `18px` stay at weight `400` or more; weights under `300` are for display sizes of `28px` and up, because thin strokes disappear at text sizes.

**Headings descend with level.** A subordinate heading does not render larger than its parent. Adjacent levels may share a size when weight or spacing keeps them distinct. Choose the heading element by structure ([[accessibility]]) and set its size in CSS.

**Line-height by role.** Headings about `1.1`, body `1.5` to `1.6`, unitless. Tight leading is for short text: anything that wraps to three or more lines gets at least `1.4`, including in height-constrained rows and cards.

**Letter-spacing by size.** Large headings slightly negative (about `-0.02em`), small uppercase labels slightly positive (about `0.05em`), body copy neither.

**Measure.** Cap long-form text at 60–75 characters per line (`65ch`, or roughly `560px`–`680px` at a `16px` body).

**Wrapping.** `text-wrap: balance` on headings and `text-wrap: pretty` on descriptions; neither on long-form text. `overflow-wrap: break-word` where long words, links or ids could escape, and `white-space: nowrap` on labels and badges.

**Changing numbers.** `font-variant-numeric: tabular-nums` on any value that updates (timers, counters, prices), so the layout does not shift.

**Truncation hides content.** Where the missing text matters, the full value stays reachable in whatever way the project already provides.

**Natural case in copy.** Store text in natural case and set presentation with `text-transform`. Use real punctuation: curly quotes in prose, an en dash in ranges, the single ellipsis character, and `&nbsp;` to keep a value and its unit together.

**Properties before raw tags.** `font-weight: 650`, `font-optical-sizing: auto` and `font-variant-numeric` keep working when a non-variable fallback renders; `font-variation-settings` and `font-feature-settings` do not, so keep those for custom axes and features with no property. Load the weights and styles the design uses. `font-synthesis: none` removes bold, italic, small-cap, superscript and subscript synthesis together, so set it only after checking that every emphasis stays distinct across the whole fallback stack.

**Underlines.** Take position and thickness from the font (`text-underline-position: from-font`, `text-decoration-thickness: from-font`). Only the color of a real underline animates reliably, so build an animated underline as its own element.

**Inputs at 16px on mobile.** iOS Safari zooms the page when an input's text is under `16px` (`text-base sm:text-sm`). Do not use `maximum-scale=1` to stop it: Safari ignores it for pinch zoom and other browsers honor it and block zoom.

**Font smoothing once, at the root.** On macOS, `-webkit-font-smoothing: antialiased` and `-moz-osx-font-smoothing: grayscale` (Tailwind's `antialiased`).

**Language and direction.** Set `lang`, and `dir` where direction changes. Digits keep their order in every direction; isolate a mixed-direction value with `<bdi>`. A paragraph of three or more lines aligns by its own script's direction, even inside an interface of the other direction.

**Text stays selectable.** Use `user-select: none` only on a specific drag or gesture surface where selection gets in the way.

When text looks low in contrast, use [[colors]] to measure the rendered pair and [[accessibility]] to decide which requirement applies; do not change colors unless asked.

## Reference files

| File | What it answers |
|---|---|
| `references/spacing-and-sizing.md` | The role-based type scale and `text-box` trimming |

## Provenance and maintenance

Last verified 2026-07. The volatile claims here are browser- and platform-behavior claims, not methodology: font-synthesis behavior, `text-wrap: balance` / `pretty` support and their line limits, `from-font` underline metrics support, the iOS Safari input-zoom threshold and `maximum-scale` handling, the macOS font-smoothing properties, the `text-box` browser-support versions in `references/spacing-and-sizing.md`, and the Tailwind utility names used here. Re-verify against current browser-support tables and the project's Tailwind version before relying on a specific support or utility claim.
