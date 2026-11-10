# Gamut and Tailwind v4

## Max chroma varies by lightness and hue

Every sRGB color exists in Display P3, which covers about 50% more colors. The gamut boundary is irregular. At L=0.5 in sRGB:

- Highest chroma: purple (H ≈ 285) at C ≈ 0.29
- Red-orange (H ≈ 0-30): C ≈ 0.20
- Lowest chroma: cyan (H ≈ 195) at C ≈ 0.09

The peak hue shifts with lightness: at L=0.7 magenta peaks, at L=0.9 green peaks. Cyan consistently has the lowest max chroma.

## Gamut checking

A color whose chroma exceeds the maximum for its L, H and color space clips. Reduce chroma and keep L and H: `oklch(0.7 0.35 150)` is outside sRGB, and `oklch(0.7 0.22 150)` is the same color clamped.

## Fallback patterns

Give the sRGB value to every browser and the wider one to P3 displays. Add the `@supports` layer only when the project's browser matrix includes browsers without OKLCH.

```css
.accent {
  color: #4ade80;
}

@supports (color: oklch(0 0 0)) {
  .accent {
    color: oklch(0.7 0.2 150);
  }

  @media (color-gamut: p3) {
    .accent {
      color: oklch(0.7 0.3 150);
    }
  }
}
```

## Tailwind v4

Tailwind CSS v4 defines its default palette in OKLCH, and a custom scale in `@theme` follows the same convention (`--color-brand-500: oklch(0.623 0.188 250)` gives `bg-brand-500`, `text-brand-200` and so on). The opacity modifier works with it: `bg-brand-500/50` compiles to `oklch(0.623 0.188 250 / 0.5)`. Generate the scale's values with `references/palette-generation.md`.

When a theme migration is in scope: convert the hex values in `@theme`, replace `theme()` references that used hex, check for hardcoded hex in component code, and look at dark mode again afterwards.
