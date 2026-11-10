---
name: ui-polish
description: >-
  Default values for interface finish: nested radius, optical alignment, shadows, press feedback, icon and motion values, and the generated-interface defaults to avoid. Use when polishing or reviewing components and motion. Not for text ([[typography]]), focus and reduced motion ([[accessibility]]), structure ([[layout]]) or whole-screen audits ([[interface-review]]).
---

# Details that make interfaces feel better

The project's own instructions win: where its AGENTS.md, CLAUDE.md, design system or established conventions say otherwise, follow them and skip that part of this. Use the project's existing kit and tokens; do not add components, on-screen text or options that the project's design does not call for.

Polish is many small details that add up. The values here are defaults: they apply where the project has no value of its own. A project's component library, tokens, density and established motion values win over every number in this skill, and a kit part is not restyled or given new props to match it.

When reviewing motion, slow it down (10% speed in the browser's Animations panel) and walk each state: hover, focus, active, loading, empty. What looks wrong slowed down is what feels slightly off at full speed.

## Surfaces

**Concentric radius.** For closely nested rounded surfaces, outer radius = inner radius + padding. Mismatched nested radii are a common reason an interface looks off.

**Optical alignment.** Where geometric centering looks wrong (a button with an icon, a play triangle, an asymmetric glyph), align by eye.

**Shadows for elevation, borders for structure.** Where a border exists only to give a button, card or container depth, a layered transparent `box-shadow` adapts better to varied backgrounds. Borders that carry structure or state stay: dividers, separators, input outlines, selected and focus states.

**Image outlines.** A `1px` outline at low opacity gives images a consistent edge: pure black at 10% in light mode and pure white at 10% in dark mode. A tinted neutral picks up the surface color and reads as dirt on the edge, so the color stays pure black or white; express it through the project's token where it has one.

→ `references/surfaces.md` — the radius arithmetic and its limit, alignment offsets, the shadow recipes and the image-outline values. Read when a change nests rounded surfaces, adds elevation or places an image.

## Motion

**Interruptible by default.** Use CSS transitions for interactive state changes, since they retarget when interrupted. Keep keyframes for staged sequences that run once.

**Restraint.** No custom animation on high-frequency interactions (keystrokes, row hovers, tab switches in a work tool); they get instant feedback or an opacity or color transition of 150ms or less. Motion is not the only feedback: every animated state change also has a static cue.

**Entrances and exits.** For an infrequent staged entrance, split content into semantic chunks and stagger them by about 100ms. Exits are softer and shorter than entrances: a small fixed `translateY`, `ease-out`.

**Contextual icons.** Defaults for an icon that appears or swaps on a state change: scale `0.25` to `1`, opacity `0` to `1`, blur `4px` to `0`, with a spring of `duration: 0.3, bounce: 0`. Under reduced motion this becomes an opacity crossfade ([[accessibility]]).

**Press feedback.** Default `scale(0.96)` on press. An established project value is kept; below `0.95` feels exaggerated.

**No entrance on first render.** A stateful icon or toggle does not animate into its default state on page load (`initial={false}` on `AnimatePresence`), as long as that does not remove an intended page entrance.

**Name the properties.** Transition the properties that change, not `all`. Tailwind's `transition-transform` covers `transform, translate, scale, rotate`. Add `will-change` only for `transform`, `opacity` or `filter`, and only on observed first-frame stutter, since each layer costs memory.

→ `references/animations.md` — the default values, the cross-fade recipe with and without a motion library, and when `initial={false}` breaks an entrance. Read when a change adds or edits an animation or transition.

## Icons

An icon beside text carries the text's optical weight: `1.5px` stroke beside regular text, `2px` beside semibold, where the icon set supports stroke variants. One icon set per surface. Icons use `currentColor` and take their states from CSS, not from separate assets. Outline is the default variant and fill marks the active state.

→ `references/icons.md` — stroke weights, sizing, the outline and fill pair, and which icons flip in right-to-left. Read when a change adds, swaps or restyles an icon.

## Don't ship the model default

A generated interface has a recognizable house style. Each entry here is a defect when it is used as a default:

| Model default | Why it's a problem |
| --- | --- |
| Purple/indigo everything | The visually "safe" palette makes every generated app look identical — derive color from the product's own brand or content |
| Gradients everywhere | Visual noise that clashes with most real design systems |
| `rounded-2xl` on everything | Maximum rounding ignores the radius hierarchy (the concentric rule) that real designs use to signal nesting |
| Generic hero section | Template-driven layout with no connection to the actual content or user need |
| Lorem-ipsum-style copy | Placeholder text hides the layout problems real content reveals — length, wrapping, overflow |
| Equal oversized padding | Uniform generous spacing destroys visual hierarchy and wastes screen space |
| Stock card grids | A uniform grid is a layout shortcut that ignores information priority and scanning order |
| Shadow-heavy depth | Layered shadows compete with content and are costly to render on low-end devices |

None of these is banned outright. Each is banned as a default: use it only when the product's content and brand call for it.

## Reference files

| File | What it answers |
|---|---|
| `references/surfaces.md` | Border radius, optical alignment, shadows, image outlines |
| `references/animations.md` | Interruptible animations, enter and exit values, icon animation, press feedback, motion restraint |
| `references/icons.md` | Icon stroke weight, states via `currentColor`, outline vs. fill, sizing, RTL flipping |

## Provenance and maintenance

Last verified 2026-07. The volatile claims here are the ecosystem-dependent ones: the `motion` / `framer-motion` package names, import paths, and API surface (`AnimatePresence`, `initial={false}`, the spring `transition` object), and the set of properties Tailwind's `transition-transform` covers. Re-verify against the installed package's own documentation and the project's Tailwind version before relying on a specific API shape. The numeric values are this skill's own defaults, not external facts; change them only as a deliberate decision here.
