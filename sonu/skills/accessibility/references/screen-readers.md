# Screen readers

## Choosing how to announce a change

Work down this list and stop at the first match:

1. **Focus moves there anyway** (an opened dialog, the first invalid field on submit): nothing extra. The focus move is the announcement.
2. **Tied to a specific control** (a field error, a character count): `aria-describedby` on the control, so it is announced with the field.
3. **Not urgent and not tied to a control** (a toast, "Saved", a result count, a loading state): a polite live region, `role="status"`.
4. **Urgent and not tied to a control** (a form-level failure, a session expiring): `role="alert"`.

## Live regions

- For repeated polite updates, keep a stable empty region in the DOM and change its text. A polite region inserted together with its content is announced inconsistently.
- Dynamically inserted `role="alert"` content is usually announced, but behavior varies; test the target browser and screen-reader combinations.
- Default to polite. Assertive interrupts whatever the person was reading.
- Do not move focus to a toast.
- For a loading state, set `aria-busy="true"` on the updating region, then announce the outcome ("Loaded, 12 results").

## Timed messages

A message that contains an action, an error or information the person may need stays until dismissed. A message that must time out stays at least 5 seconds, and hovering or focusing it pauses the timer. The only path to an action, such as undo, is never inside a message that disappears.

## Hidden and decorative content

Visually hidden text uses a `1px` box, not a zero-sized one, since some screen readers skip zero-sized elements; `display: none` and `visibility: hidden` remove content from assistive technology. A missing `alt` is worse than an empty one, because the file name gets read. A functional image's `alt` names the action ("Search"), not the picture.
