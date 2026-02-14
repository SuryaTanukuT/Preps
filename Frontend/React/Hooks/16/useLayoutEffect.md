# `useLayoutEffect` in React (interview-ready)

`useLayoutEffect` is like `useEffect`, but it runs synchronously after DOM mutations and before the browser paints the screen.

`useLayoutEffect` runs synchronously after React commits DOM changes but before the browser paints, so it’s used for reading layout measurements and making immediate DOM adjustments without flicker.

It has the same dependency and cleanup semantics as `useEffect`, but because it blocks painting, I use it sparingly—defaulting to `useEffect` for most side effects.

So it’s used when you must:

* Read layout from the DOM
* Apply DOM changes
* Before the user sees anything (avoid flicker)

Interview one-liner:
“useLayoutEffect runs after DOM updates but before paint, so it’s for layout measurements and synchronous DOM adjustments.”

---

# Timing difference: `useEffect` vs `useLayoutEffect`

## `useEffect`

* Runs after paint
* Non-blocking (better for performance)
* Used for: data fetching, subscriptions, logging

## `useLayoutEffect`

* Runs before paint
* Blocks painting until it finishes
* Used for: measuring layout + applying immediate changes

Interview line:
“Prefer useEffect; use useLayoutEffect only when you need DOM measurements to avoid visual glitches.”

---

# When to use `useLayoutEffect` (real use cases)

✅ Measure element size/position (`getBoundingClientRect`) and set state immediately
✅ Positioning tooltips/popovers based on actual DOM size
✅ Sync scroll position (restore scroll without flicker)
✅ Integrate with libraries that need DOM measurements before paint
