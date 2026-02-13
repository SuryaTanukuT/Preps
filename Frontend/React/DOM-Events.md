# DOM Events::

React uses a wrapper event object called SyntheticEvent:
Normalizes behavior across browsers
Gives a consistent API (event.target, event.currentTarget, preventDefault, etc.)
Handlers run through React’s event system, not direct addEventListener (conceptually)

Key interview line:
“In React, most handlers receive a SyntheticEvent, which behaves like a DOM event but is normalized.”


# preventDefault() vs stopPropagation()

preventDefault() stops the default browser action
(e.g., form submit, link navigation)

stopPropagation() stops the event from bubbling/capturing further


# How does React implement event delegation?

React typically does not attach a listener to every element. Instead it:
Attaches a small set of listeners at a top level (delegation point)
Uses bubbling/capturing to catch events
Figures out which React component handler to run

React 17+ change:
Delegation happens on the root container (the element you render into), not document.
This makes multiple React apps on the same page behave better.

# Bubbling vs capturing in React?

React supports both phases just like native DOM:
Capturing phase (top → down): onClickCapture, onKeyDownCapture, etc.
Bubbling phase (bottom → up): onClick, onKeyDown, etc. (default)

Order (when both exist):
Parent capture
Child capture
Child bubble
Parent bubble