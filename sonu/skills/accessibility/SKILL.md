---
name: accessibility
description: >-
  Accessibility checks for product interfaces: keyboard and focus, names and roles, page structure and landmarks, forms, announcements, hit areas, motion, zoom and reflow. Use when building or reviewing a component or page structure, or on a keyboard or screen-reader complaint. Not for contrast ([[colors]]), text sizing ([[typography]]) or whole-screen audits ([[interface-review]]).
---

# Accessibility that comes with the craft

The project's own instructions win: where its AGENTS.md, CLAUDE.md, design system or established conventions say otherwise, follow them and skip that part of this. Use the project's existing kit and tokens; do not add components, on-screen text or options that the project's design does not call for.

Most accessibility comes free from the platform and from a good primitive library. Where the project's kit is built on accessible primitives, rely on them for focus trapping, roving focus and keyboard models, and check that they work; do not rebuild them by hand. When unsure, prefer the native element or the kit's part over a custom one, and remove ARIA before adding it.

To review, complete each flow with the keyboard alone, then check that every control exposes a name, a role and its state.

## What to check

**Native elements first.** `<button>` for actions and `<a href>` for navigation, which keeps Cmd/Ctrl/middle-click. A role is a promise of the full keyboard model, so a wrong role is worse than none.

**Visible focus.** Style `:focus-visible`, and prefer the browser's own indicator, which adapts to forced-colors mode. A custom ring uses the project's focus token, is at least a `2px` solid perimeter or equivalent, and is checked against every color it crosses. Do not remove an outline without a verified replacement.

**Keyboard paths.** Every pointer interaction has one. Escape closes the most recent overlay, arrow keys move within a composite widget, Tab moves between widgets. `tabindex` is `0` or `-1` only.

**Dialogs.** Background content is `inert`, focus moves inside on open and returns to the trigger on close. For a destructive confirmation, focus the least destructive action.

**Hit areas.** WCAG 2.5.8 (AA) requires a 24×24 CSS pixel target or one of its exceptions; under the spacing exception, a 24px circle centered on an undersized target must not intersect another target or its circle. Check the exceptions before reporting a small control. This skill's larger default is 44×44px on touch and 40×40px on desktop where density permits. To enlarge a hit area without enlarging the control, put a pseudo-element on the wrapping `<label>` or `<button>`, not on the `<input>`. Hit areas do not overlap.

**Forms.** Every control has a programmatic label, and label and control share one hit target. Use `autocomplete`, and the `type` and `inputmode` that give the right keyboard. Do not block paste. Keep submit enabled until the request starts, then disable it while keeping its label. Validate on submit: `aria-invalid` on the failing field, `aria-describedby` pointing at its inline error, focus on the first invalid field. `aria-disabled="true"` only announces; use it when the control must stay focusable, and then block activation in code and style the state.

**Names.** An icon-only button has an accessible name, and visible label text appears in the accessible name. Decorative elements are `aria-hidden`, which never goes on or above a focusable element.

**Color is not the only cue.** Status has a second cue. Decide which contrast requirement applies, then use [[colors]] to measure the rendered pair. Report a failing pair; do not change the project's colors unless asked.

**Motion.** Make motion opt-in under `prefers-reduced-motion: no-preference`. Under reduced motion, slides and scales become opacity crossfades, and parallax and autoplay stop. A global override uses `0.01ms` durations and not `none`, so `animationend` and `transitionend` still fire. Anything that moves for more than 5 seconds has a visible pause control.

**Announcements.** → `references/screen-readers.md` — how to choose between a focus move, `aria-describedby`, a status region and an alert, and the rules that make live regions reliable. Read when a change announces dynamic content.

**Focus details.** → `references/focus-and-keyboard.md` — focus-ring order of preference, dialog focus handling, and focus on client-side route changes. Read when a change adds an overlay, a custom focusable control or a route transition.

**Structure.** One primary `<main>` landmark, and headings that describe their sections. One page-level `<h1>` and nested levels are the recommended default, not a WCAG failure on their own. Where repeated chrome precedes the content and the project's design has a place for it, a skip link comes first.

**Zoom.** The page works at 200% zoom and reflows at 320px width without horizontal scrolling. Text containers use `min-height`, and zoom is never blocked with `user-scalable=no` or `maximum-scale=1`.

## Reference files

| File | What it answers |
|---|---|
| `references/focus-and-keyboard.md` | Focus-ring preference order, dialog focus, route-change focus |
| `references/screen-readers.md` | Choosing how to announce a change, live-region rules, timed messages |

## Provenance and maintenance

Last verified 2026-07. The volatile claims here are platform-behavior claims, not methodology: `inert` and `overscroll-behavior` support, forced-colors-mode behavior, screen-reader handling of dynamically inserted live regions, and the WCAG success-criterion numbers and thresholds cited above. Re-verify against current browser-support tables and the current WCAG specification before treating any specific number or support claim as still true; the principles themselves hold regardless.
