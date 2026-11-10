---
name: layout
description: >-
  Layout defaults for web interfaces: grouping by space, shared edges, spacing values, the four view states, breakpoints, direction-aware structure. Use when structuring a page or component. Not for hit areas ([[accessibility]]), surfaces and motion ([[ui-polish]]), text ([[typography]]) or whole-screen audits ([[interface-review]]).
---

# Layout that communicates structure

The project's own instructions win: where its AGENTS.md, CLAUDE.md, design system or established conventions say otherwise, follow them and skip that part of this. Use the project's existing kit and tokens; do not add components, on-screen text or options that the project's design does not call for.

Position, spacing and alignment carry hierarchy before a word is read. Express every change in the project's existing styling system and spacing scale. The numbers below are this skill's defaults for an interface with no established density or spacing system; an existing system that works keeps its own values.

## Defaults

**Group with space before lines.** Negative space is the first grouping tool, a background shape the second, and a separator line the last, kept for dense data where space would cost too much. The gap between groups is at least twice the gap within a group (`8px` inside, `16px` or more between), or the grouping does not read. A separator that is needed is a quiet hairline and is not combined with a large gap.

**One spacing step per level.** Use one project spacing step for each level of nesting (`16px` when no scale exists), and keep everything on a small set of shared edges. A stray edge reads as noise even when nobody can name it.

**Controls look like controls.** An interactive element has a background, a border, an underline or a consistent control zone. Static elements do not borrow control styling, or they collect dead clicks.

**Space between targets.** Start with `12px` between adjacent bordered or filled controls and `24px` around borderless text and icon controls, which have nothing else marking where one target ends. Compact tools may use less when hit areas ([[accessibility]]) do not overlap.

**Order by importance, in leading and trailing terms.** The most important content sits near the top and the leading edge. Use logical properties (`margin-inline-start`, `padding-inline-end`, `inset-inline-start`, `text-align: start`) for anything direction-dependent, and keep physical left and right for geometry that is physical, such as a device notch. Arrangements that encode progression mirror in right-to-left.

**One main action per view.** Detail sits a level deeper, in the way the project's design already provides for.

**Hidden content shows a cue.** Use the project's established cue. Where there is none, let the next item of a horizontal scroller show `16–32px` past the edge, or use a disclosure control that says what is hidden.

**Insets and edges.** Backgrounds and media may run to the viewport edges; text and controls stay inside the layout margins and safe areas (`env(safe-area-inset-*)`). Full-width buttons in content sit inside the margins (about `16px` inline on mobile). Edge-to-edge actions are fine when they follow established platform or application chrome.

**Breakpoints come from the content.** Break where the layout stops fitting, not at device presets, and collapse late. Prefer container queries for components, so a card adapts to its column.

**Plan for growth and clipping.** Strings grow by language, so text containers have no fixed widths or heights and buttons size from their label. Primary actions stay in stable chrome or normal flow where resizing, scrolling or a keyboard cannot clip them; if a modal's content scrolls, its action row does not.

## Every view accounts for four states

A view that shows data has four states: ready, loading, empty and error. Every view accounts for all four, because development runs against populated data and the other three are the ones that ship unconsidered.

It does so with the project's standard loading, empty and error parts. Each of the three is a small family of variants chosen by the shape of what is being shown: for loading, for example, a table-shaped placeholder, a card-shaped one and a small inline indicator; empty and error work the same way. Use the existing variant that matches the content's shape, and add a variant only when no existing one fits that shape. Every variant in a family shares the same look and motion, so they read as one system. Do not design a state for a single screen: that is a new component each time, and the views drift apart.

The loading part keeps the eventual layout where the project's part supports that, and is marked busy for assistive technology. The error part offers the way to retry. The empty part offers the one action that fills the view; its wording follows [[ux-writing]].

State that decides which view a person sees and is worth sharing, such as filters, the page or a selected tab, belongs in the URL where the platform has one, so it survives a refresh.

## Provenance and maintenance

Volatile facts in this skill, last verified 2026-08: platform support claims (`env(safe-area-inset-*)`, container queries preferred over media queries, logical-property behavior). Re-verify against current browser-support data when a finding hinges on one. The numeric values (the 2× gap rule, the 12/16/24px spacing tiers, the 16–32px peek) are this skill's own conventions, not external facts; change them only as a deliberate decision here.
