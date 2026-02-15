https://www.geeksforgeeks.org/software-testing/software-engineering-integration-testing/
# 🔥 Integration Testing (Senior / FAANG / Backend Perspective)

You asked just “integration testing” — I’ll give you:

1️⃣ Clear definition
2️⃣ Unit vs Integration vs E2E
3️⃣ Node.js + TypeScript integration testing setup
4️⃣ DB + Redis + Kafka style examples
5️⃣ BFSI-grade backend testing strategy
6️⃣ CI/CD best practices
7️⃣ Interview-ready answers

---

# 1️⃣ What is Integration Testing?

## ✅ Definition

Integration testing verifies that **multiple modules work together correctly**.

Instead of testing a single function (unit), we test:

* Controller + Service + DB
* API + Redis
* Service + External API
* Event producer + consumer

---

### 🧠 Example

Unit test:

```ts
expect(add(2,3)).toBe(5)
```

Integration test:

* Call `/api/login`
* Validate DB user created
* Check JWT issued
* Verify Redis session stored

---

# 2️⃣ Testing Pyramid

```
        E2E (Few)
    Integration (More)
Unit Tests (Most)
```

| Type        | Tests What       |
| ----------- | ---------------- |
| Unit        | Single function  |
| Integration | Multiple modules |
| E2E         | Full system      |

Senior answer:

> Integration tests validate business workflows across boundaries.

---

# 3️⃣ Node.js + TypeScript Integration Testing Setup

Most common stack:

* Jest
* Supertest
* Test DB (Docker / in-memory)
* Separate test environment

---

## Example: Express + Jest + Supertest

### app.ts

```ts
import express from 'express';

export const app = express();
app.use(express.json());

app.post('/sum', (req, res) => {
  const { a, b } = req.body;
  res.json({ result: a + b });
});
```

---

### integration.test.ts

```ts
import request from 'supertest';
import { app } from './app';

describe('Integration Test - /sum', () => {
  it('should return correct sum', async () => {
    const res = await request(app)
      .post('/sum')
      .send({ a: 2, b: 3 });

    expect(res.status).toBe(200);
    expect(res.body.result).toBe(5);
  });
});
```

---

# 4️⃣ Integration Test With Database

### 🔹 Strategy

* Use test database
* Run migrations
* Clean DB before each test

---

Example (Postgres):

```ts
beforeEach(async () => {
  await db.query('TRUNCATE TABLE users RESTART IDENTITY');
});
```

---

Example test:

```ts
it('should create user in DB', async () => {
  const res = await request(app)
    .post('/users')
    .send({ name: 'Surya' });

  const user = await db.query('SELECT * FROM users');

  expect(user.rows.length).toBe(1);
});
```

---

# 5️⃣ Redis Integration Testing

```ts
import Redis from 'ioredis';

const redis = new Redis();

it('should cache value', async () => {
  await redis.set('key', 'value');
  const val = await redis.get('key');
  expect(val).toBe('value');
});
```

In CI → use Docker container for Redis.

---

# 6️⃣ Kafka / Event Integration Testing

Test:

* Producer publishes event
* Consumer processes
* DB updated

Use:

* Test Kafka container
* Or mock broker

---

# 7️⃣ BFSI-Grade Integration Testing Strategy

In banking systems we test:

### 🔹 Payment Flow

Test:

1. Create transaction
2. Deduct balance
3. Insert ledger entry
4. Publish event
5. Send notification

All in one test.

---

### 🔹 Fraud Detection

Test:

* 6 transactions in 10 seconds
* 6th should be rejected

---

### 🔹 Idempotency

Test:

* Same idempotency key twice
* Only one DB insert

---

### 🔹 Circuit Breaker

Test:

* External API fails 5 times
* Circuit opens
* Requests rejected fast

---

# 8️⃣ Mock vs Real Dependencies

| Strategy           | When to Use        |
| ------------------ | ------------------ |
| Mock external APIs | Unit testing       |
| Real DB            | Integration        |
| Docker containers  | CI                 |
| In-memory DB       | Fast local testing |

Senior answer:

> Integration tests should hit real infrastructure boundaries but avoid unstable external systems.

---

# 9️⃣ Test Isolation Best Practices

* Separate `.env.test`
* Disable real external calls
* Rollback DB after test
* Use transaction wrapping
* Clean Redis after each test

---

# 🔟 CI/CD Integration

Pipeline:

```
Install deps
Run lint
Run unit tests
Start Docker containers
Run integration tests
Build
Deploy
```

Important:

* Fail fast
* Parallel execution
* Coverage tracking

---

# 1️⃣1️⃣ Common Integration Testing Mistakes

❌ Hitting real payment gateway
❌ Not cleaning DB
❌ Tests dependent on order
❌ Using production config
❌ Slow tests (>1 minute per test suite)

---

# 1️⃣2️⃣ Senior Interview Questions

### Q: Why integration tests are important?

Answer:

> They validate real business workflows and integration boundaries, catching issues unit tests miss — especially in microservices architectures.

---

### Q: How do you test distributed systems?

Answer:

* Contract testing
* Consumer-driven contracts
* Event replay testing
* Chaos testing

---

### Q: How do you test idempotency?

Answer:

* Send duplicate requests
* Validate single DB entry
* Validate response reuse

---

# 🔥 Advanced Integration Patterns

* Testcontainers (Docker per test)
* Pact (contract testing)
* Snapshot testing
* Transaction rollback wrapper
* Seeded test data factory pattern

---

# 🔥 Real Production Example (Banking)

Integration test for fund transfer:

```ts
it('should transfer funds correctly', async () => {
  await request(app)
    .post('/transfer')
    .send({ from: 1, to: 2, amount: 100 });

  const fromAcc = await db.query('SELECT balance FROM accounts WHERE id=1');
  const toAcc = await db.query('SELECT balance FROM accounts WHERE id=2');

  expect(fromAcc.rows[0].balance).toBe(900);
  expect(toAcc.rows[0].balance).toBe(1100);
});
```

---

# 🚀 Senior-Level Mindset

Integration testing ensures:

* Data consistency
* Idempotency
* Event correctness
* Cache invalidation
* Security enforcement
* Auth flow validation

---

---

# 🔥 1️⃣ Complete NestJS Integration Testing Setup

We’ll test:

* Controller
* Service
* DB (Postgres)
* Auth Guard
* Redis (optional)

---

## 🏗 Typical Structure

```
src/
  app.module.ts
  users/
test/
  users.integration.spec.ts
```

---

## Step 1: Install

```bash
npm install --save-dev @nestjs/testing supertest jest
```

---

## Step 2: Test Module Setup

```ts
import { Test } from '@nestjs/testing';
import { INestApplication } from '@nestjs/common';
import request from 'supertest';
import { AppModule } from '../src/app.module';

describe('Users Integration', () => {
  let app: INestApplication;

  beforeAll(async () => {
    const moduleRef = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    app = moduleRef.createNestApplication();
    await app.init();
  });

  afterAll(async () => {
    await app.close();
  });

  it('POST /users should create user', async () => {
    const res = await request(app.getHttpServer())
      .post('/users')
      .send({ name: 'Surya' });

    expect(res.status).toBe(201);
    expect(res.body.name).toBe('Surya');
  });
});
```

---

## 🔹 DB Integration (TypeORM Example)

Use `.env.test`

```ts
TypeOrmModule.forRoot({
  type: 'postgres',
  host: 'localhost',
  database: 'test_db',
})
```

Clean before each test:

```ts
beforeEach(async () => {
  await connection.synchronize(true); // reset DB
});
```

---

## 🔹 Testing Auth Guard

Override guard:

```ts
.overrideGuard(AuthGuard)
.useValue({ canActivate: () => true })
```

---

# 🔥 2️⃣ Microservices Contract Testing (Pact)

## ✅ Problem

Microservice A depends on B.

How to ensure:

* API contract doesn’t break?

---

## 🧠 Solution: Pact

Consumer defines expectations.

Provider must satisfy them.

---

## Consumer Test Example

```ts
import { Pact } from '@pact-foundation/pact';

const provider = new Pact({
  consumer: 'PaymentService',
  provider: 'UserService',
});

describe('Pact test', () => {
  beforeAll(() => provider.setup());

  it('should get user details', async () => {
    await provider.addInteraction({
      state: 'User exists',
      uponReceiving: 'a request for user',
      withRequest: {
        method: 'GET',
        path: '/users/1',
      },
      willRespondWith: {
        status: 200,
        body: { id: 1, name: 'Surya' },
      },
    });

    const response = await fetch('http://localhost:1234/users/1');
    const data = await response.json();

    expect(data.name).toBe('Surya');
  });

  afterAll(() => provider.finalize());
});
```

---

## Why Pact Matters (BFSI)

* Prevents breaking payment contracts
* Protects AML service APIs
* Safe schema evolution

---

# 🔥 3️⃣ Testcontainers with Docker

## ✅ Why?

Instead of mocking DB/Redis/Kafka,
run real containers in tests.

---

## Install

```bash
npm install --save-dev testcontainers
```

---

## Example: Postgres Container

```ts
import { PostgreSqlContainer } from 'testcontainers';

let container;

beforeAll(async () => {
  container = await new PostgreSqlContainer().start();

  process.env.DB_HOST = container.getHost();
  process.env.DB_PORT = container.getPort().toString();
});
```

---

### Benefits

✔ Real infrastructure
✔ Isolated per test run
✔ CI friendly
✔ No dependency on local setup

---

# 🔥 4️⃣ E2E vs Integration (Deep Comparison)

| Feature       | Integration      | E2E           |
| ------------- | ---------------- | ------------- |
| Scope         | Multiple modules | Full system   |
| DB            | Yes              | Yes           |
| Frontend      | No               | Yes           |
| External APIs | Mocked           | Real/Test env |
| Speed         | Medium           | Slow          |
| Use case      | Business logic   | User journey  |

---

## Integration Example

Test:

* POST /transfer
* Validate DB update
* Validate Redis entry

---

## E2E Example

Test:

* Login
* Transfer money
* Check dashboard updated

---

### Senior Interview Answer

> Integration tests validate service boundaries.
> E2E tests validate complete user flows.

---

# 🔥 5️⃣ BFSI-Grade Payment Test Strategy Blueprint

Now serious architecture.

---

# 🏦 Payment Flow

1. Validate request
2. Check balance
3. Debit
4. Credit
5. Insert ledger entry
6. Publish event
7. Send notification
8. Return response

---

## Test Layers

### 🟢 Unit Tests

* Fee calculation
* Fraud scoring

---

### 🟡 Integration Tests

* Transfer funds
* Ledger insert
* Idempotency

---

### 🔴 E2E Tests

* Login → Transfer → Check balance

---

### 🔵 Non-Functional Tests

* Load testing
* Concurrency testing
* Idempotency retry
* Chaos testing

---

## 🔹 Critical BFSI Scenarios to Test

### 1️⃣ Double Spend

Send same request twice
Expect:

* One DB insert
* Same response returned

---

### 2️⃣ Race Condition

Simultaneous debit
Expect:

* Transaction isolation prevents negative balance

---

### 3️⃣ Circuit Breaker

Payment gateway down
Expect:

* Fast failure
* Retry policy

---

### 4️⃣ Fraud Sliding Window

6 transactions in 10 sec
Expect:

* 6th rejected

---

### 5️⃣ Event Consistency

Transfer success
Kafka event published
Consumer updates analytics DB

---

## 🔹 Performance Targets

| Metric          | Target |
| --------------- | ------ |
| Latency         | <200ms |
| Throughput      | 5k TPS |
| Error rate      | <0.1%  |
| Cache hit ratio | >80%   |

---

# 🔥 Enterprise CI/CD Flow

```
Lint
Unit Tests
Integration Tests
Spin containers
Run contract tests
Build
Deploy staging
Run E2E
Deploy prod
```

---

# 🔥 Senior-Level Talking Points

In interviews, mention:

* Test pyramid
* Idempotency testing
* Isolation level validation
* Event-driven verification
* Contract testing for microservices
* Chaos engineering
* Canary deployments

---

