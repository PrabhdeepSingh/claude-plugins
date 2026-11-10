# Color usage

For picking values, see `references/palette-generation.md`; for checking pairs, see `references/accessibility-contrast.md`.

## One color, one meaning

A color is used for one purpose across the interface (interactive, destructive, featured). If the brand color signals that text is interactive, that hue, anywhere within ±15°, on non-interactive text tells people to click something that is not clickable. Give the second use a neutral.

## Tokens by role

Name colors by role and use them only in that role: `--color-text-secondary` is muted foreground text, and using it as a background breaks every later theme change that assumes the role. If a role has no token, add one in the project's token home (the `@theme` block in a Tailwind project); do not borrow a token that happens to have the right value today.

## One colored action per view

When the product uses filled color for primary emphasis, give that treatment to one primary action in the current decision and leave peer actions neutral. Put the color on the background, not the label: blue label text on a neutral button reads as a link. Keep an established component hierarchy that shows emphasis another way, and do not recolor controls to impose this. Several colored backgrounds are fine when they encode distinct states or categories and are not competing peer actions. A selected state may use the accent on its glyph and label; that is state, not emphasis.

## Color across locales

Color meaning is not universal. Where a color carries meaning (finance, status, alerts), check that it holds in every locale shipped to. The common case is gains and losses: green for gains in English locales, red for gains in Chinese financial interfaces. In a product that localizes into such markets, gain and loss colors are per-locale tokens.

## Light, dark and increased contrast

Every custom color has a light and a dark variant (derivation is in `references/palette-generation.md`). For people who ask for more contrast, supply a `prefers-contrast: more` variant that widens the foreground and background lightness gap by at least `0.15` L over the default, then re-verify the pair against APCA's preferred thresholds (Lc 90 body, Lc 75 non-body).

- Recheck every foreground and background pair in both appearances. The palettes are not mirror images, so a pair that passes in light can fail in dark.
- Account for translucency. A color on a translucent surface shifts with whatever scrolls behind it. Test it over the lightest and darkest content it can sit on, or make the surface opaque enough that the shift cannot break contrast.
