---
name: layout
description: >-
  Layout structure for web interfaces — grouping, alignment, negative space, reading order, progressive disclosure, breakpoints, direction-aware (RTL) structure. INVOKE PROACTIVELY when structuring a page or component, spacing or aligning controls, deciding what collapses at small sizes, or reviewing frontend structure. Hit areas and focus: [[accessibility]]; radius, shadows, motion: [[ui-polish]]; line length and text spacing: [[typography]]; whole-screen audits: [[interface-review]].
---

# Layout that communicates structure

Layout communicates before a single word is read: position, spacing, and alignment carry hierarchy on their own, and generous space beats decoration. A good layout also survives stress: resize it, translate it, mirror it for RTL, and it should still hold together. Apply these principles when building or reviewing UI code, and express every change in the project's existing styling system (Tailwind, plain CSS, CSS-in-JS); never introduce a second styling approach.

Hit-area sizes and focus behavior are covered by [[accessibility]]; visual polish (radius, shadows, animation) by [[ui-polish]]; line length and text spacing by [[typography]].

Treat the numeric values below as starting points for interfaces without an established density or spacing system. Preserve deliberate platform chrome, compact professional tools, and project tokens when they remain usable under hit-area, zoom, localization, and viewport stress tests.

## Core Principles

### 1. Group with Space, Not Lines

Negative space is the primary grouping tool; background shapes second; separator lines last, only where space alone can't carry the structure. The gap between groups must be at least 2× the gap within a group (`8px` intra-group → `16px`+ inter-group), or the grouping reads as noise.

→ `references/grouping-and-alignment.md` — space-vs-separator decisions, alignment edges, logical properties, and importance ordering behind §1–§4, read when this change groups, aligns, or reorders elements.

### 2. Keep Controls Distinct from Content

Interactive elements must look interactive: a background shape, a border, or a consistent placement zone. Never style a control identically to adjacent static text.

### 3. Align to Shared Edges

Pick alignment edges and stick to them; every stray edge reads as noise. Use one project spacing step for each level of subordination (`16px` is a useful default). Use logical properties (`padding-inline-start`, `margin-inline-end`) for direction-dependent layout; reserve physical left/right for genuinely physical geometry.

### 4. Order by Importance

The most important content sits near the top and the leading edge; reading order flows top-to-bottom, leading-to-trailing. Think in leading/trailing, not left/right.

### 5. Hint at Hidden Content

Progressive disclosure needs a visible affordance. Use the project's established cue; without one, let the next item peek `16–32px` past the scroll edge or show a disclosure control. Content hidden with zero cue may as well not exist.

→ `references/spacing-and-adaptivity.md` — spacing between targets, layout margins, progressive-disclosure cues, full-bleed content, breakpoint selection, and localization growth behind §5–§10, read when this change sets spacing or margins, hides content behind a cue, or adapts across viewport sizes.

### 6. Breathing Room Between Targets

Without an established density system, start with `12px` between adjacent bordered or filled controls and `24px` of clearance around borderless text- and icon-only controls. Compact layouts may use less when [[accessibility]] hit areas do not overlap and the controls remain visually distinct.

### 7. Inset Buttons from the Edges

In content layouts, keep full-width buttons inside the layout margins (start near `16px` inline on mobile) with a visible radius. Edge-to-edge actions are acceptable when they intentionally follow established platform or application chrome, account for safe areas, and remain distinguishable from system UI.

### 8. Content Bleeds, Controls Float

Backgrounds and media extend to the viewport edges; controls and text stay inside the layout margins and safe areas (`env(safe-area-inset-*)`). Sticky chrome floats above the content layer, it doesn't dam it.

### 9. Hold Structure Until It Breaks

Breakpoints come from the content, not device presets. Keep the expanded layout as long as it genuinely fits and collapse late; prefer container queries for component-level adaptation. Test the smallest and largest sizes first.

### 10. Plan for Growth and Clipping

Plan for substantial and language-dependent string growth rather than relying on a universal percentage: no fixed widths or heights on text containers, and let rows wrap. Never park critical actions where resizing or scrolling clips them; keep them reachable in the normal flow or stable chrome appropriate to the product.

## Common Mistakes

| Mistake | Fix |
| --- | --- |
| Separator line where spacing would do | Remove the line, double the gap between groups |
| `margin-left` / `padding-right` in a localizable layout | `margin-inline-start` / `padding-inline-end` |
| Content-layout button accidentally touches the viewport | Inset within the project margins; preserve intentional platform chrome |
| Carousel/scroller that looks complete | Let the next item peek `16–32px` past the edge |
| Adjacent controls merge or expanded hit areas overlap | Increase the gap using the project scale; use `12px`/`24px` as starting points |
| Breakpoints at 768/1024 because they're the defaults | Break where the content actually stops fitting |
| Fixed-width text container sized to one language | `max-width` + wrapping; test pseudo-localization and representative locales |
| Primary action at the clip-prone bottom of a pane | Sticky positioning or stable chrome with safe-area padding |

## Review Output Format

Use this format only when the user asks for a standalone layout review. When [[interface-review]] orchestrates the review, provide domain evidence and findings to that skill and let its output format, severity scale, consolidation rules, cap, and verdict take precedence.

Report all confirmed findings as one markdown table grouped by principle — `| Severity | Location | Before | After | Why |`, never separate "Before:" / "After:" lines. **Location** cites `path/to/file:line` (or the exact screen and component when there are no source files); **Before / After** show the current implementation and an actionable replacement; **Why** names the violated principle and its impact. Consolidate a repeated systemic issue into one row listing every affected location; omit principles with no findings. **Severity**: `HIGH` blocks content or an action at a supported viewport; `MEDIUM` harms hierarchy, reading order, or adaptability; `LOW` is isolated alignment or spacing polish.

After the findings: **Verification** — list the exact checks run and their observed results (the relevant viewport widths, reading order, zoom, RTL state when applicable), and name any check not run. Then **Verdict**: `Block` if any `HIGH` finding remains, `Needs changes` if only `MEDIUM` or `LOW` findings remain, `Approve` only when no actionable findings remain. When there are no findings, omit the table, state "No actionable layout findings", report verification, and end with `Approve`.

## Provenance and maintenance

Volatile facts in this skill and its references, last verified 2026-08: platform support claims (`env(safe-area-inset-*)`, container queries preferred over media queries, logical-property behavior) — re-verify against current browser-support data when a finding hinges on one; the numeric thresholds (12/16/24px spacing tiers, the 16–32px peek) are this skill's own conventions, not external facts — change them only as a deliberate decision here.

## Reference files

| File | What it answers |
|---|---|
| `references/grouping-and-alignment.md` | Space vs. separators, alignment edges, logical properties, importance ordering (§1–§4) |
| `references/spacing-and-adaptivity.md` | Spacing between targets, layout margins, progressive disclosure, full-bleed content, breakpoints, localization growth (§5–§10) |
