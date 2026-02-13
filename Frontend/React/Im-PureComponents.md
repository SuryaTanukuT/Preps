# Impure Components::

Impure components in React = components whose render logic has side effects or depends on mutable external state,
so calling them multiple times can produce different results or break the UI.

React render must be pure: given the same props/state, it should return the same UI and not cause side effects.

This matters even more in React 18 because rendering can be interrupted, restarted, or run twice in dev StrictMode.