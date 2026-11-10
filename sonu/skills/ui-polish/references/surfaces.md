# Surfaces

Border radius, optical alignment, shadows and image outlines. These are defaults; an established component token wins.

## Concentric border radius

For nested rounded elements, the outer radius equals the inner radius plus the padding between them:

```
outerRadius = innerRadius + padding
```

A card with `8px` padding around a `12px` inner radius gets `20px`. In Tailwind, `rounded-2xl p-2` around `rounded-lg` (16 = 8 + 8).

The rule applies when the layers are close and share a visible, even inset. If the padding is larger than `24px`, treat the layers as separate surfaces and choose each radius on its own. Keep an established component token when the layers are independent or the padding is deliberately asymmetric.

## Optical alignment

- **Button with text and icon.** When the icon makes symmetric padding look unbalanced, use slightly less padding on the icon side. Starting point: icon-side padding = text-side padding − `2px` (`ps-4 pe-3.5`).
- **Play triangles.** The geometric center is not the visual center; shift the glyph about `2px` toward its point.
- **Asymmetric icons** (stars, arrows, carets). The best fix is in the SVG itself, so component code needs no offset. The fallback is a small offset such as `translate-x-px`.

## Shadows instead of borders

Where a button, card or container uses a border for depth, a subtle `box-shadow` can replace it. A shadow is transparent and adapts to any background; a solid border color only suits the background it was designed for.

This does not apply to dividers (`border-b`, `border-t`, side borders), table cell boundaries, form input outlines or any border whose purpose is separation or state. Those stay borders.

### Light mode

Three layers: a 1px ring, a small lift, and ambient depth.

```css
:root {
  --shadow-border:
    0px 0px 0px 1px oklch(0 0 0 / 0.06),
    0px 1px 2px -1px oklch(0 0 0 / 0.06),
    0px 2px 4px 0px oklch(0 0 0 / 0.04);
  --shadow-border-hover:
    0px 0px 0px 1px oklch(0 0 0 / 0.08),
    0px 1px 2px -1px oklch(0 0 0 / 0.08),
    0px 2px 4px 0px oklch(0 0 0 / 0.06);
}
```

### Dark mode

A single white ring, since layered depth shadows are not visible on dark backgrounds:

```css
/* Dark mode: adapt the selector to the project's setup
   (class, data attribute, or prefers-color-scheme). */
.dark {
  --shadow-border: 0 0 0 1px oklch(1 0 0 / 0.08);
  --shadow-border-hover: 0 0 0 1px oklch(1 0 0 / 0.13);
}
```

Apply with `box-shadow: var(--shadow-border)` and transition only `box-shadow` (`150ms`, `ease-out`) to the hover value.

## Image outlines

A `1px` low-opacity outline gives images a consistent edge, especially where other elements use borders or shadows.

- Light mode: pure black, `oklch(0 0 0 / 0.1)`.
- Dark mode: pure white, `oklch(1 0 0 / 0.1)`.
- Keep the color pure black or white. A near-black or near-white from the palette (slate-900, zinc-900, `#0a0a0a`, `#f5f5f7`) or the accent color is tinted, picks up the surrounding surface color, and reads as dirt on the image edge. Where the project has a token system, express the value through a token.

```css
img {
  outline: 1px solid oklch(0 0 0 / 0.1);
  outline-offset: -1px; /* draw the ring just inside the image edge */
}
```

In Tailwind: `outline outline-1 -outline-offset-1 outline-black/10 dark:outline-white/10`.

`outline` is used instead of `border` because it does not affect layout, and `outline-offset: -1px` draws the ring just inside the edge so it follows the corner radius.
