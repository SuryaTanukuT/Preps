https://testing-library.com/docs/react-testing-library/intro/

---

# 🔥 1️⃣ What is React Testing Library (RTL)?

## ✅ Definition

**React Testing Library is a testing utility for React that encourages testing components the way users interact with them.**

Philosophy:

> “The more your tests resemble the way your software is used, the more confidence they can give you.”

RTL:

* Does NOT replace Jest
* Works on top of Jest
* Focuses on DOM behavior, not internals

---

# 🔥 2️⃣ render() Function

## ✅ Definition

`render()` mounts a React component into a virtual DOM (jsdom).

```ts
import { render } from '@testing-library/react';

render(<Button />);
```

Returns utilities like:

* `container`
* `rerender`
* `unmount`
* `debug`

---

# 🔥 3️⃣ Queries (Very Important)

Queries are how you find elements in DOM.

RTL encourages selecting elements like users would.

---

# Query Priority (Best Practice Order)

1. `getByRole` ✅ (Best)
2. `getByLabelText`
3. `getByPlaceholderText`
4. `getByText`
5. `getByTestId` (last resort)

---

## 🔹 Query Variants

| Variant | Behavior            |
| ------- | ------------------- |
| getBy   | Throws if not found |
| queryBy | Returns null        |
| findBy  | Async (waits)       |

---

### Example

```ts
screen.getByRole('button');
screen.queryByText('Hello');
await screen.findByText('Loaded');
```

---

# 🔥 4️⃣ User Interactions

Use `@testing-library/user-event`

```ts
import userEvent from '@testing-library/user-event';

await userEvent.click(button);
await userEvent.type(input, 'Surya');
```

Better than `fireEvent` because it simulates real user behavior.

---

# 🔥 5️⃣ Assertions

Using Jest matchers.

```ts
expect(button).toBeInTheDocument();
expect(input).toHaveValue('Surya');
expect(element).toBeVisible();
```

Install:

```bash
npm install @testing-library/jest-dom
```

---

# 🔥 6️⃣ Testing Async UI

Example:

```ts
await screen.findByText('Data Loaded');
```

OR

```ts
await waitFor(() => {
  expect(screen.getByText('Done')).toBeInTheDocument();
});
```

Used for:

* API loading
* Delayed rendering
* State updates

---

# 🔥 7️⃣ Mocking API Calls

---

## Option 1: Mock fetch

```ts
global.fetch = jest.fn(() =>
  Promise.resolve({
    json: () => Promise.resolve({ name: 'Surya' }),
  })
);
```

---

## Option 2: MSW (Best Practice)

Mock Service Worker intercepts real HTTP requests.

Better for integration-style UI tests.

---

# 🔥 8️⃣ Testing Components with Context

Wrap component with provider.

```ts
render(
  <AuthProvider>
    <Profile />
  </AuthProvider>
);
```

Better pattern: custom render

```ts
const customRender = (ui) =>
  render(<AuthProvider>{ui}</AuthProvider>);
```

---

# 🔥 9️⃣ Testing Components with Router

```ts
import { MemoryRouter } from 'react-router-dom';

render(
  <MemoryRouter>
    <MyComponent />
  </MemoryRouter>
);
```

To test navigation:

```ts
expect(window.location.pathname).toBe('/dashboard');
```

---

# 🔥 🔟 Testing Redux

Wrap with Provider.

```ts
render(
  <Provider store={store}>
    <App />
  </Provider>
);
```

Test:

* Dispatch
* State updates
* UI reflects store changes

---

# 🔥 Testing Zustand

Mock store:

```ts
jest.mock('../store', () => ({
  useStore: () => ({
    count: 5,
    increment: jest.fn(),
  }),
}));
```

---

# 🔥 1️⃣1️⃣ Accessibility Testing

RTL promotes a11y-first queries.

Use:

```ts
getByRole('button', { name: /submit/i })
```

You can also use:

```bash
npm install jest-axe
```

```ts
import { axe } from 'jest-axe';

expect(await axe(container)).toHaveNoViolations();
```

---

# 🔥 1️⃣2️⃣ Avoid Implementation Testing

❌ Don’t test:

* Internal state
* Private methods
* Class names
* useState directly

Good test:

```ts
expect(screen.getByText('Welcome')).toBeInTheDocument();
```

Bad test:

```ts
expect(component.state.loggedIn).toBe(true);
```

---

# 🔥 1️⃣3️⃣ Snapshot Testing

```ts
expect(container).toMatchSnapshot();
```

Good for:

* Static UI components

Bad for:

* Frequently changing UI

---

# 🔥 1️⃣4️⃣ Debugging Tests

```ts
screen.debug();
```

OR

```ts
console.log(container.innerHTML);
```

You can also use:

```bash
--watch
```

---

# 🔥 1️⃣5️⃣ Jest vs RTL

| Jest        | RTL               |
| ----------- | ----------------- |
| Test runner | React utility     |
| Assertions  | DOM utilities     |
| Mocking     | Render components |
| Coverage    | Query elements    |

They work together.

---

# 🔥 1️⃣6️⃣ Playwright vs Cypress

| Feature         | Playwright | Cypress      |
| --------------- | ---------- | ------------ |
| Browser support | All major  | Chrome-based |
| Parallel        | Yes        | Limited      |
| Speed           | Fast       | Slower       |
| Cross-browser   | Strong     | Moderate     |
| CI friendly     | Excellent  | Good         |

Senior answer:

> Playwright is more scalable and cross-browser capable.

---

# 🔥 1️⃣7️⃣ Playwright vs Jest

| Jest         | Playwright   |
| ------------ | ------------ |
| Unit testing | E2E testing  |
| Node/jsdom   | Real browser |
| Fast         | Slower       |
| No real UI   | Real UI      |

Jest tests components.
Playwright tests full browser flow.

---

# 🧠 Senior-Level Testing Strategy

| Layer         | Tool       |
| ------------- | ---------- |
| Unit          | Jest       |
| Component     | RTL        |
| Integration   | RTL + MSW  |
| E2E           | Playwright |
| Load          | k6         |
| Accessibility | jest-axe   |

---

# 🎯 Interview Answer Template

If asked:

**"How do you test React apps?"**

Answer:

1. Unit test logic with Jest
2. Component test with RTL
3. Mock APIs with MSW
4. Test routing & state management
5. Accessibility testing
6. E2E with Playwright
7. Avoid implementation testing

---
