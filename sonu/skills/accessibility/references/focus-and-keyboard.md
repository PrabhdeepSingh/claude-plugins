# Focus and keyboard

Use the project's primitives for these where it has them. This file is for the cases they do not cover.

## Focus rings, in order of preference

1. Keep the browser's indicator and add only `outline-offset: 2px`. It adapts to the platform and to forced colors without the author predicting every background.
2. When the design needs a custom ring, use the project's focus token: `outline: 2px solid var(--focus-ring); outline-offset: 2px`.

A custom `outline` with no color renders `currentColor`, which is not automatically visible, because the ring may cross colors other than the text's own background. Check the whole perimeter against every color it crosses, including fills, images, and hover and selected states. In `forced-colors: active`, keep the default color adjustment or use a system color such as `Highlight`.

## Dialogs

Native `<dialog>` with `showModal()` gives the focus trap, the inert background and Escape handling. A custom overlay needs `role="dialog"`, `aria-modal="true"`, an accessible name, and `inert` on everything behind it.

- On open, focus the first focusable element. For a destructive confirmation, focus the least destructive action.
- On close, return focus to the trigger. If the trigger no longer exists, move focus to the nearest logical container.
- `overscroll-behavior: contain` on the dialog keeps scrolling inside it from scrolling the page behind.

## Composite widgets

A composite widget (tabs, menu, toolbar, radio group) is one Tab stop. The active item has `tabindex="0"`, the others `-1`, and arrow keys move both focus and the `0`. Tabs activate on arrow focus when panels render instantly, and on Enter or Space when switching is expensive.

## Client-side route changes

Client-side navigation does not reset focus or announce anything. On a route change, update `document.title`, then move focus to the new view's main heading (given `tabindex="-1"`) or to `<main>`. Restore scroll position on back and forward navigation.
