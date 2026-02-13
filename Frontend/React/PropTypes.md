# PropTypes in React (interview perspective)

PropTypes are a runtime type-checking system for React component props (mainly used in JavaScript projects). They warn in the console if a component receives props of the wrong type.

Key line for interviews:
“PropTypes validate props at runtime and help catch bugs + document component APIs, but TypeScript is preferred for compile-time safety.”

Why PropTypes are used::
Catch wrong prop types early (during development)
Self-document the component’s expected inputs
Helpful in large JS codebases (without TypeScript)

⚠️ PropTypes don’t prevent crashes automatically—they just warn.

# How to use PropTypes
Install: prop-types
Define propTypes on the component

import PropTypes from "prop-types";

function UserCard({ name, age, onSelect }) {
  return (
    <div onClick={onSelect}>
      {name} ({age})
    </div>
  );
}

UserCard.propTypes = {
  name: PropTypes.string.isRequired,
  age: PropTypes.number,
  onSelect: PropTypes.func,
};

# Most common PropTypes validators
PropTypes.string, number, bool
PropTypes.array, object, func
PropTypes.node (anything renderable: string, element, fragment, etc.)
PropTypes.element (a React element)
PropTypes.oneOf(['small','medium','large']) (enum)
PropTypes.oneOfType([PropTypes.string, PropTypes.number]) (union)
PropTypes.arrayOf(PropTypes.string) (array items type)
PropTypes.shape({ id: PropTypes.string }) (object structure)
PropTypes.exact({ ... }) (no extra keys allowed)

Example (realistic):

UserCard.propTypes = {
  user: PropTypes.shape({
    id: PropTypes.string.isRequired,
    name: PropTypes.string.isRequired,
  }).isRequired,
};

“TypeScript for compile-time types; runtime validation needs schema validation, PropTypes only warns.”

#  Where PropTypes still make sense

Legacy React codebases in JS
Shared component libraries published for JS consumers
Quick guardrails when you can’t migrate to TS yet