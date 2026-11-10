# Animations

Every value here is a default. Where the project has an established motion value, keep it. Honoring `prefers-reduced-motion` is covered by [[accessibility]] and applies to everything in this file.

## Transitions or keyframes

A CSS transition interpolates toward the latest state, so reversing mid-way is smooth. A keyframe animation runs on a fixed timeline and restarts. Use transitions for interactive state (hover, toggle, open and close) and keyframes for a one-shot staged sequence.

## Staged entrances

For an infrequent entrance where sequence helps show hierarchy (first load of a view, a success state):

- Split into logical groups (title, description, actions) and stagger by about `100ms`. A title may split into words at about `80ms`.
- Each group enters from `opacity: 0`, `translateY(12px)`, `blur(4px)`, over about `300–400ms`, `ease-out`.

Do not stagger routine interactions such as row hovers, keystrokes or repeated tab changes.

## Exits

Exits are softer than entrances, because attention is moving on.

- A small fixed `translateY` (about `-12px`) with opacity and blur, not the full container height.
- Shorter than the entrance: about `150ms` against `300ms`, `ease-out`.
- Slide fully out (about `200ms`) only when spatial context matters, such as a drawer closing.
- Remove immediately when motion adds no information, the interaction repeats often, or reduced motion is requested.

## Contextual icon animation

When an icon appears or swaps on a state change (hover actions, play and pause, a toolbar that appears in context), animate it instead of toggling visibility. Icons that are always visible, decorative or navigational are not animated.

Default values:

- `scale`: `0.25` → `1`
- `opacity`: `0` → `1`
- `filter`: `blur(4px)` → `blur(0px)`
- `transition`: `{ type: "spring", duration: 0.3, bounce: 0 }`; in CSS, `300ms` with `cubic-bezier(0.2, 0, 0, 1)`

With a motion library: check `package.json`, and import from `"motion/react"` when `motion` is installed or from `"framer-motion"` when that is; if both exist, follow the nearest existing imports. Use `AnimatePresence mode="popLayout"`, key the element by the state it represents so a state change runs the exit, and put the values on `initial`, `animate` and `exit`.

Without one, do not add a dependency for this. Keep both icons in the DOM and cross-fade them with CSS transitions; because neither unmounts, both directions animate. One icon is absolutely positioned over the other, and the one in flow defines the layout size:

```tsx
function IconButton({ isActive, ActiveIcon, InactiveIcon }) {
  const shown = "scale-100 opacity-100 blur-0";
  const hidden = "scale-[0.25] opacity-0 blur-[4px]";
  const fade = "transition-[opacity,filter,scale] duration-300 ease-[cubic-bezier(0.2,0,0,1)]";
  return (
    <button>
      <div className="relative">
        <div className={cn("absolute inset-0 flex items-center justify-center", fade, isActive ? shown : hidden)}>
          <ActiveIcon />
        </div>
        <div className={cn(fade, isActive ? hidden : shown)}>
          <InactiveIcon />
        </div>
      </div>
    </button>
  );
}
```

## Scale on press

Default `scale(0.96)` on press, as a transition so that releasing mid-press returns smoothly (`transition-transform duration-150 ease-out active:scale-[0.96]`; with a motion library, `whileTap={{ scale: 0.96 }}`). Keep an established project value; a value below `0.95` feels exaggerated. Not every button needs it. Where the project's button already has a way to turn it off, use that when the motion would distract.

## No entrance on first render

`initial={false}` on `AnimatePresence` stops enter animations on first render, so an icon swap, toggle, tab or segmented control does not animate into its default state on page load.

It breaks a component that relies on its `initial` prop for a first-time entrance, such as a staggered page header or a loading state: the whole entrance is skipped. Check the component on a full page refresh before applying it.

## Motion restraint

- No custom animation on high-frequency interactions. Something triggered constantly charges its attention cost every time. Expressive motion is for infrequent moments; high-frequency interactions get instant feedback or `opacity` or `background-color` at `150ms` or less.
- Motion is not the only feedback. Every state change an animation communicates is also visible when the animation does not run: a color change, an icon swap, a label.
- If a shorter, smaller animation says the same thing, use it.
