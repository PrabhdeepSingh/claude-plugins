# Icons

Defaults for a project whose icon set leaves these open. A kit's own icon conventions win.

## Stroke matched to text weight

An icon beside text carries the same optical weight as the text.

| Adjacent text | Icon stroke width (24px grid) |
| --- | --- |
| Regular (400), 14–16px | `1.5px` |
| Medium/Semibold (500–600) | `2px` |
| Bold (700), or emphasized standalone | `2.5px` |

This applies where the icon set supports stroke variants. Otherwise keep the set's native stroke and use size or color for emphasis. Do not mix icon sets with different stroke conventions on one surface. Size an inline icon relative to the text, typically `1em`–`1.25em`, so the pair scales together.

## One SVG, recolored per state

Use one SVG drawn with `currentColor` and let CSS state drive color and opacity. Hardcoded fills inside the SVG (`fill="#666"`) break this; change them to `currentColor` when importing icons.

## Outline default, fill active

Where a set offers outline and filled variants, use them as a state pair: outline for the default state, fill for selected or active (the active tab, a toggled bookmark). Filled icons everywhere leave the active item with no signal. The swap is a contextual icon animation; its default values are in `references/animations.md`.

## Render size

Test each icon at the smallest size it renders, often `16px`. Use the set's native grid sizes (`16`, `20`, `24`); a 16px icon scaled from a 24px grid renders soft.

## Right-to-left

Under `dir="rtl"`, flip icons whose meaning follows reading direction and leave the rest.

| Flip | Don't flip |
| --- | --- |
| Back/forward arrows, chevrons in navigation | Logos and brand marks |
| Text-block glyphs (alignment, lists, indent) | Checkmarks |
| Speaker/volume waves | Physical objects: clocks, cups, pencils |
| "Send" style directional glyphs | Media playback (play and rewind stay left-to-right by convention) |

Mirror with `[dir="rtl"] .icon-directional { scale: -1 1; }` (Tailwind `rtl:-scale-x-100`).
