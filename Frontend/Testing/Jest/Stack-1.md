https://jestjs.io/
---

# 🧠 1️⃣ What is Jest (Proper Definition)

## ✅ Definition

**Jest is a full-featured JavaScript testing framework** that provides:

* Test runner
* Assertion library
* Mocking framework
* Code coverage
* Snapshot testing
* Parallel execution
* Watch mode

It is not just assertions — it is a complete testing ecosystem.

---

# 🧱 2️⃣ Core Testing Concepts (With Clear Definitions)

---

## 🔹 Test Suite

### ✅ Definition

A logical group of related tests.

```ts
describe('User Service', () => {})
```

Used to organize tests.

---

## 🔹 Test Case / Test Block

### ✅ Definition

An individual test verifying one behavior.

```ts
it('should create user', () => {})
```

One test = One behavior.

---

## 🔹 Assertion / Matcher

### ✅ Definition

A function that verifies expected outcome.

```ts
expect(value).toBe(5)
```

---

# 🧪 3️⃣ Fixtures (Very Important in Backend)

## ✅ Definition

A **fixture is predefined test data** used consistently across tests.

Used to:

* Avoid repetitive test setup
* Standardize test data
* Improve readability

---

### Example: User Fixture

```ts
export const mockUser = {
  id: 1,
  name: 'Surya',
  balance: 1000
};
```

Use in test:

```ts
expect(service.createUser(mockUser)).toBeDefined();
```

---

### Why Fixtures Matter in BFSI?

For:

* Payment payload
* Fraud scenarios
* Idempotency tests
* Complex nested JSON

---

# 🧪 4️⃣ Setup & Teardown

## ✅ Definition

Lifecycle hooks that run before/after tests.

| Hook       | Runs              |
| ---------- | ----------------- |
| beforeAll  | Once before suite |
| beforeEach | Before each test  |
| afterEach  | After each test   |
| afterAll   | Once after suite  |

---

### Example

```ts
beforeEach(() => {
  jest.clearAllMocks();
});
```

Used for:

* Reset DB
* Clear mocks
* Initialize test state

---

# 🎭 5️⃣ Mocking (Deep Explanation)

## ✅ Definition

Mocking replaces real dependencies with controlled fake implementations.

---

## 🔹 Why Mock?

* Avoid real DB
* Avoid external APIs
* Test failure scenarios
* Improve speed
* Achieve isolation

---

# 🧩 Types of Mocks

---

## 1️⃣ Function Mock

```ts
const mockFn = jest.fn();
```

Tracks:

* Calls
* Arguments
* Return values

---

## 2️⃣ Module Mock

```ts
jest.mock('./paymentService');
```

Replaces entire module.

---

## 3️⃣ Partial Mock (Spy)

```ts
jest.spyOn(service, 'charge');
```

Monitors real function.

---

## 4️⃣ Manual Mock (**mocks** folder)

Auto-replaced module implementation.

---

## 5️⃣ Factory Mock

```ts
jest.mock('./db', () => ({
  query: jest.fn()
}));
```

---

# 🔍 6️⃣ Spies

## ✅ Definition

A spy tracks calls to real functions.

```ts
const spy = jest.spyOn(obj, 'method');
```

Used for:

* Verify call count
* Verify arguments
* Ensure certain side effects

---

# ⏳ 7️⃣ Testing Async Code

---

## Promise Based

```ts
await expect(service.fetch()).resolves.toEqual(data);
```

---

## Rejection

```ts
await expect(service.fail()).rejects.toThrow();
```

---

## Callback Style

```ts
test('callback test', done => {
  function callback(data) {
    expect(data).toBe('ok');
    done();
  }
});
```

---

# ⏱ 8️⃣ Fake Timers

## ✅ Definition

Allows control of time-based functions.

Used for:

* setTimeout
* setInterval
* Retry logic
* Rate limiting

---

```ts
jest.useFakeTimers();
jest.advanceTimersByTime(1000);
```

---

# 📊 9️⃣ Code Coverage

## ✅ Definition

Measures how much of your code is tested.

Metrics:

* Statements
* Branches
* Functions
* Lines

---

### Run

```bash
npm test -- --coverage
```

---

## 🎯 Good Coverage Strategy

Focus on:

* Business logic
* Edge cases
* Error handling
* Branch conditions

Avoid:

* Testing framework internals

---

# 🧩 🔟 Test Isolation

## ✅ Definition

Each test should run independently.

Bad:

```ts
let counter = 0;
```

Good:

Reset state every test.

---

# 🔁 1️⃣1️⃣ Parallel Execution

Jest runs test files in parallel by default.

Benefit:

* Faster execution

Disable:

```bash
--runInBand
```

---

# 👀 1️⃣2️⃣ Watch Mode

```bash
npm test -- --watch
```

Runs only changed tests.

Great for development.

---

# 🧾 1️⃣3️⃣ Snapshot Testing

## ✅ Definition

Captures output and compares future runs.

```ts
expect(result).toMatchSnapshot();
```

Used mostly in React.

Avoid for dynamic data.

---

# ⚙️ 1️⃣4️⃣ Jest Configuration

`jest.config.js`

```js
module.exports = {
  preset: 'ts-jest',
  testEnvironment: 'node',
  collectCoverage: true,
  clearMocks: true,
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80,
    },
  },
};
```

---

# 🧪 1️⃣5️⃣ Testing APIs Properly

---

## Unit Test API Logic

Mock:

* DB
* External APIs

---

## Integration Test

Use:

* Supertest
* Real test DB

---

## Payment API Testing Strategy

Test:

✔ Successful payment
✔ Insufficient balance
✔ Idempotency
✔ Retry
✔ Gateway timeout
✔ Fraud block

---

# 🧱 1️⃣6️⃣ Test Data Factories (Advanced)

Instead of fixtures:

```ts
function createUser(overrides = {}) {
  return {
    id: 1,
    name: 'Test',
    balance: 1000,
    ...overrides
  };
}
```

Better flexibility.

---

# 🧪 1️⃣7️⃣ Parameterized Tests

```ts
test.each([
  [1,2,3],
  [2,3,5]
])('adds %i + %i', (a,b,expected) => {
  expect(a+b).toBe(expected);
});
```

---

# 🧨 1️⃣8️⃣ Testing Error Handling

```ts
expect(() => fn()).toThrow();
```

---

# 🧠 1️⃣9️⃣ Advanced Backend Concepts with Jest

* Testing transaction rollback
* Testing DB isolation level
* Testing race conditions
* Testing rate limiter
* Testing circuit breaker
* Testing idempotency keys
* Testing event consumers

---

# 🏦 BFSI-Level Jest Usage

In banking systems we test:

* Ledger integrity
* Balance consistency
* Double-spend prevention
* Event publishing
* Cache invalidation
* Retry policies

---

# 🔥 Senior-Level Interview Answer

When asked:

**"How do you design testing in Node?"**

Answer structure:

1. Unit tests for logic (mock dependencies)
2. Integration tests for service boundaries
3. Contract tests for microservices
4. E2E for user journey
5. Load testing for performance
6. Chaos testing for resilience

---

# 🧭 Testing Philosophy

* Test behavior, not implementation
* Mock only external dependencies
* Avoid brittle tests
* Keep tests deterministic
* Prefer integration over excessive mocking

---


---

# 🏦 PART 1 — CRITICAL BACKEND CONSISTENCY TESTING

These are high-risk production scenarios.

---

# 🔥 1️⃣ Testing Transaction Rollback

## ✅ Definition

Verifying that **if one operation inside a DB transaction fails, all changes are reverted**.

Critical for:

* Payments
* Ledger updates
* Balance transfers

---

## Example Scenario

Transfer money:

1. Debit A
2. Credit B
3. Insert ledger entry

If ledger insert fails → debit must rollback.

---

## Integration Test Example (TypeORM/NestJS style)

```ts
it('should rollback on failure', async () => {
  await expect(
    service.transfer(1, 2, 100, { forceFail: true })
  ).rejects.toThrow();

  const acc1 = await db.query('SELECT balance FROM accounts WHERE id=1');
  const acc2 = await db.query('SELECT balance FROM accounts WHERE id=2');

  expect(acc1.rows[0].balance).toBe(1000);
  expect(acc2.rows[0].balance).toBe(1000);
});
```

✔ Validates atomicity (ACID).

---

# 🔥 2️⃣ Testing DB Isolation Level

## ✅ Definition

Ensuring concurrent transactions behave correctly based on isolation level.

Isolation levels:

* Read Uncommitted
* Read Committed
* Repeatable Read
* Serializable

---

## Example: Prevent Double Spend

Test:

* Two concurrent debits
* Only one succeeds

```ts
await Promise.allSettled([
  service.debit(1, 900),
  service.debit(1, 900)
]);

const acc = await db.query('SELECT balance FROM accounts WHERE id=1');
expect(acc.rows[0].balance).toBeGreaterThanOrEqual(0);
```

For banking → use `SERIALIZABLE` or row locking.

---

# 🔥 3️⃣ Testing Race Conditions

## ✅ Definition

Verifying system behaves correctly under concurrent access.

---

## Example

```ts
await Promise.all(
  Array(10).fill(null).map(() => service.transfer(1, 2, 10))
);
```

Then verify:

* No negative balance
* No duplicate ledger entries

---

# 🔥 4️⃣ Testing Rate Limiter

## ✅ Definition

Verify API blocks excessive requests.

---

## Example

```ts
for (let i = 0; i < 6; i++) {
  await request(app).get('/api');
}

const res = await request(app).get('/api');
expect(res.status).toBe(429);
```

Use fake timers for time-window testing.

---

# 🔥 5️⃣ Testing Circuit Breaker

## ✅ Definition

Ensure service stops calling failing external dependency.

---

Mock external API:

```ts
jest.spyOn(api, 'charge').mockRejectedValue(new Error());

for (let i = 0; i < 5; i++) {
  await service.processPayment();
}

expect(circuitBreaker.state).toBe('OPEN');
```

---

# 🔥 6️⃣ Testing Idempotency Keys

## ✅ Definition

Same request with same key → processed once.

---

Test:

```ts
await request(app)
  .post('/payment')
  .set('Idempotency-Key', 'abc123')
  .send(payload);

await request(app)
  .post('/payment')
  .set('Idempotency-Key', 'abc123')
  .send(payload);

const payments = await db.query('SELECT * FROM payments');
expect(payments.rows.length).toBe(1);
```

---

# 🔥 7️⃣ Testing Event Consumers

## ✅ Definition

Verify event handler processes events correctly.

Example:

* Kafka message received
* DB updated

---

Test strategy:

* Publish mock event
* Wait for consumer
* Verify DB change

---

# 🧱 PART 2 — TESTING STRATEGY LAYERS

---

# 🟢 Unit Tests (Mock Dependencies)

## Definition

Test single function in isolation.

Mock:

* DB
* External APIs
* Redis

---

# 🟡 Integration Tests (Service Boundaries)

## Definition

Test:

* Controller + Service + DB

Use:

* Real DB
* Supertest

---

# 🔵 Contract Tests (Microservices)

## Definition

Verify consumer-provider contract.

Tools:

* Pact

Ensures:

* API schema stability
* Safe microservice evolution

---

# 🔴 E2E Tests

## Definition

Test full user journey.

Example:
Login → Transfer → View balance.

Uses:

* Real environment
* Test DB

---

# 🧪 PART 3 — NON-FUNCTIONAL TESTING

---

# 🚀 Load Testing

## Definition

Measure performance under high traffic.

Tools:

* k6
* Artillery
* JMeter

Test:

* 5k requests/sec
* P99 latency
* Error rate

---

# 💥 Chaos Testing

## Definition

Simulate failures to test resilience.

Example:

* Kill Redis
* Kill DB
* Timeout external API

Verify:

* Circuit breaker activates
* System degrades gracefully

---

# 🧠 Senior Testing Architecture

```
Unit Tests
Integration Tests
Contract Tests
E2E Tests
Load Tests
Chaos Tests
```

---

# 🔥 Enterprise Payment Testing Blueprint

### Must Cover:

✔ Atomicity
✔ Idempotency
✔ Concurrency
✔ Event consistency
✔ Retry logic
✔ Cache invalidation
✔ Fraud sliding window
✔ Circuit breaker
✔ Performance under load

---

# 🎯 Senior Interview Answer Template

If asked:

**"How would you test a payment system?"**

Answer:

1. Unit test business logic
2. Integration test transactional boundaries
3. Test idempotency & concurrency
4. Contract test external gateway
5. E2E test full flow
6. Load test peak TPS
7. Chaos test dependency failures

---
