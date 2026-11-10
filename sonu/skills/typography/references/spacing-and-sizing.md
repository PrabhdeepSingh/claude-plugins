# Spacing and sizing

## A role-based scale

A role pairs a size with its line-height and weight, so a role is one decision and not three. This is the default for a product interface with no scale of its own; a project with a scale keeps it.

| Role | Size | Line-height | Weight |
| --- | --- | --- | --- |
| Display | `2.25rem` (36px) | `1.1` | `600` |
| Title | `1.5rem` (24px) | `1.2` | `600` |
| Heading | `1.125rem` (18px) | `1.3` | `600` |
| Body | `1rem` (16px) | `1.5` | `400` |
| Caption | `0.8125rem` (13px) | `1.4` | `400` |

Emphasis within a role is one weight step up (`400` to `500`), not a size change. A heading is not smaller than body text unless it is deliberately a label-style overline.

## Text trimming with text-box

Fonts reserve space above and below the letters, which is why text sits slightly low in buttons and badges. `text-box` trims it: `text-box: trim-both cap alphabetic` trims the top to the cap height and the bottom to the baseline, and `trim-start` or `trim-end` trims one edge.

Supported in Chromium (133+) and Safari (18.2+), not yet Firefox. Treat it as progressive enhancement; unsupported browsers keep the default leading.
