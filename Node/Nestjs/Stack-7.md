# NestJS Providers, Services & Dependency Injection (Interview + Project-Oriented)

---

## 1) What is a Provider? (and how it differs from “Service”)

### **Provider**

In NestJS, a **provider** is **anything Nest can create and inject** through the DI system.

✅ Providers can be:

* services (business logic)
* repositories/DAOs (data access)
* adapters (3rd-party integrations: payments, SMS, email)
* strategies/guards/interceptors/pipes
* factories (create configured clients: Redis, DB, HTTP)

### **Service**

A **service** is a *type of provider* that usually contains:

* business logic / use-cases
* orchestration across repos/adapters

**Interview line:**

> “In Nest, service is just a provider with a role. Providers are the DI-managed components; services typically implement business use-cases.”

---

## 2) `@Injectable()` — what it actually does

`@Injectable()` marks a class as a provider so Nest can:

* register it in the DI container (if listed in `providers`)
* resolve its dependencies
* instantiate it according to scope

```ts
@Injectable()
export class PaymentsService {
  constructor(private readonly repo: PaymentsRepo) {}
}
```

**Key note:**
`@Injectable()` alone doesn’t make it injectable everywhere — it must be **registered** in a module’s `providers` (or globally provided).

---

## 3) DI Container — mental model (very interview-friendly)

The **DI container** is the “object factory + dependency graph”.

It:

1. scans module metadata
2. registers providers with tokens
3. builds a dependency graph
4. creates instances in the correct order
5. injects them wherever needed

**Interview line:**

> “Nest DI container builds a graph from module provider registrations and resolves constructor dependencies recursively.”

---

## 4) Provider registration (the core rules)

### A) Class provider (most common)

```ts
@Module({
  providers: [UsersService, UsersRepo],
})
export class UsersModule {}
```

### B) Custom providers (tokens + factories + aliases)

Used when:

* you inject non-class values (config, connection strings)
* you want to swap implementations (real vs mock)
* you want async initialization (DB/Redis clients)

#### 1) `useValue` (constant)

```ts
{ provide: 'REGION', useValue: 'ap-south-1' }
```

#### 2) `useClass` (swap implementations)

```ts
{ provide: 'PaymentsGateway', useClass: RazorpayGateway }
```

#### 3) `useExisting` (alias)

```ts
{ provide: 'ILogger', useExisting: AppLogger }
```

#### 4) `useFactory` (dynamic creation)

```ts
{
  provide: 'REDIS_CLIENT',
  useFactory: (config: ConfigService) => createRedisClient(config.get('REDIS_URL')),
  inject: [ConfigService],
}
```

**Interview line:**

> “Custom providers let us inject primitives, configure SDK clients, and switch implementations via tokens.”

---

## 5) Constructor Injection (default + best practice)

### What it is

You declare dependencies in the constructor:

```ts
@Injectable()
export class OrdersService {
  constructor(
    private readonly repo: OrdersRepo,
    private readonly payments: PaymentsService,
  ) {}
}
```

### Why it’s preferred

✅ makes dependencies explicit
✅ easy to mock in unit tests
✅ supports immutability (`readonly`)
✅ aligns with SOLID (DIP)

**Interview line:**

> “Constructor injection is preferred because it’s explicit and makes testing/refactoring safer.”

---

## 6) Provider Scopes (Singleton vs Request vs Transient)

### A) Singleton (default)

* one instance for the whole app
* best for stateless services, repos, clients

✅ Most providers should be singleton.

---

### B) Request-scoped

* new instance per request
* useful when you need **request context** inside providers

Example use-cases:

* per-request tenant resolution
* per-request correlation id stored in provider
* audit trail context

⚠️ Cost:

* more instances created
* slower under high throughput

---

### C) Transient

* new instance each time injected
* rare; use when you need isolated instances per injection

**Interview line:**

> “Default is singleton for performance. Request scope only when per-request context is required; transient is rare.”

---

## 7) Optional providers (when dependency may not exist)

Use `@Optional()` when a dependency can be missing (e.g., plugin-based design):

```ts
constructor(@Optional() private readonly audit?: AuditService) {}
```

**Project use-case:**

* audit module enabled only in production
* metrics exporter optional

---

## 8) Property-based injection (possible, but not preferred)

You can inject on properties, but constructor injection is cleaner.

```ts
@Inject('REDIS_CLIENT')
private redis: any;
```

Why not preferred:

* dependency is less visible
* harder to reason/test
* order/initialization clarity reduced

**Interview framing:**

> “Property injection is possible, but I prefer constructor injection for clarity and testability.”

---

## 9) Manual instantiation (what it means and when it’s okay)

### What manual instantiation is

Doing this inside code:

```ts
const service = new UsersService(new UsersRepo());
```

### Why it’s usually bad in Nest

* bypasses DI container
* loses interceptors/hooks/scopes
* harder to swap implementations
* difficult to mock globally

### When it’s acceptable

* pure utility classes (no dependencies)
* small helper functions
* isolated logic where DI adds no value

**Interview line:**

> “In Nest apps, I avoid manual instantiation for DI-managed components; it bypasses lifecycle/scoping and makes testing harder.”

---

## 10) Custom + Optional + Scopes (practical combination example)

A common production pattern:

* `Logger` singleton
* `RequestContext` request-scoped
* `AuditService` optional (enabled via module)

**Conceptually:**

* high-performance core singletons
* request context only where needed
* optional infrastructure features

---

# Interview “killer summary”

* **Providers** are DI-managed building blocks; **services** are providers that implement business use-cases.
* `@Injectable()` marks classes for DI; modules register them in `providers`.
* DI container builds a dependency graph and resolves constructor dependencies.
* **Scopes**: singleton (default), request (per request), transient (per injection).
* **Custom providers** (`useValue/useClass/useExisting/useFactory`) enable tokens, swapping implementations, and SDK client setup.
* Prefer **constructor injection**; use optional/property injection only when justified.
* Avoid **manual instantiation** for DI components.

---

# NestJS Providers & Injection Tokens (Project + Interview Perspective)

In NestJS, every provider is registered in the DI container under a **token**.
That token can be a **class**, a **string**, a **symbol**, or another unique value.

---

## 1) Injection Tokens (the core idea)

### What is an injection token?

An **injection token** is the **key** used by Nest’s DI container to store and retrieve a dependency.

### Why tokens exist

You need tokens when:

* you’re injecting **non-class values** (config, constants)
* you want **multiple implementations** behind one contract (real vs mock)
* you want to inject a **3rd-party client** (Redis, Stripe, DB connection)

✅ Types of tokens

* **Class token**: `UsersService`
* **String token**: `'REDIS_CLIENT'`
* **Symbol token**: `const REDIS = Symbol('REDIS')` (safer than strings)

### How you inject using tokens

* If token is a **class**, Nest can infer it from constructor type.
* If token is **string/symbol**, you use `@Inject(TOKEN)`.

---

## 2) Class Providers (most common)

### What it is

You register a class directly, and the class itself becomes the token.

```ts
@Module({
  providers: [UsersService, UsersRepository],
})
export class UsersModule {}
```

Injection:

```ts
constructor(private readonly usersService: UsersService) {}
```

✅ Interview framing:

> “Class providers are the default. The class type is the token, so injection is clean and strongly typed.”

---

## 3) Value Providers (`useValue`) — constants, configs, mocks

### What it is

You bind a token to a **pre-built value**.

```ts
@Module({
  providers: [
    { provide: 'REGION', useValue: 'ap-south-1' },
  ],
})
export class AppModule {}
```

Inject:

```ts
constructor(@Inject('REGION') private readonly region: string) {}
```

### Project use-cases

* environment/config constants
* feature flags
* test mocks (replace a gateway with stub)

✅ Interview line:

> “Value providers are great for injecting constants or swapping in mocks during testing.”

---

## 4) Factory Providers (`useFactory`) — dynamic + async creation

### What it is

You create a provider using a function, optionally injecting dependencies into the factory.

```ts
{
  provide: 'REDIS_CLIENT',
  useFactory: (config: ConfigService) => {
    return createRedisClient(config.get('REDIS_URL'));
  },
  inject: [ConfigService],
}
```

Inject:

```ts
constructor(@Inject('REDIS_CLIENT') private readonly redis: any) {}
```

### Why factories are used in real systems

* you need configuration at runtime (`ConfigService`)
* you need to build SDK clients with options
* you need async initialization (DB, Kafka, Redis)

✅ Interview line:

> “Factory providers are used for configured infrastructure clients and for async initialization driven by environment config.”

---

## 5) `useClass` Providers — swapping implementations behind a token

### What it is

You bind a token to a concrete class implementation.

```ts
{
  provide: 'PaymentsGateway',
  useClass: StripeGateway,
}
```

Inject:

```ts
constructor(@Inject('PaymentsGateway') private readonly gateway: any) {}
```

### Project use-cases

* switch between providers per environment:

  * `StripeGateway` (prod)
  * `FakeGateway` (dev/test)
* support multiple payment gateways via config

✅ Interview line:

> “useClass is useful when I want to abstract an interface/contract behind a token and swap implementations.”

---

## 6) Quick comparison table (mental clarity)

| Provider type         | Registration style                | Best for                       | Token type          |
| --------------------- | --------------------------------- | ------------------------------ | ------------------- |
| **Class provider**    | `providers: [MyService]`          | services/repos                 | Class               |
| **Value provider**    | `{ provide, useValue }`           | constants, mocks               | string/symbol/class |
| **Factory provider**  | `{ provide, useFactory, inject }` | configured clients, async init | string/symbol/class |
| **useClass provider** | `{ provide, useClass }`           | swapping implementations       | string/symbol/class |

---

## 7) Best practices for tokens (production-grade)

✅ Prefer **`Symbol()` tokens** over strings to avoid collisions:

```ts
export const REDIS_CLIENT = Symbol('REDIS_CLIENT');
```

Register:

```ts
{ provide: REDIS_CLIENT, useFactory: ... }
```

Inject:

```ts
constructor(@Inject(REDIS_CLIENT) private redis: any) {}
```

✅ Group tokens per domain/module:

* `PAYMENTS_GATEWAY`
* `AUDIT_LOGGER`
* `DB_CONN_READ`
* `DB_CONN_WRITE`

---

## 8) Interview-ready wrap-up (copy/paste)

* “In Nest, every provider is registered under an **injection token**.”
* “Class providers use the class as token; value providers inject constants/mocks; factory providers create configured/async clients; `useClass` lets us swap implementations behind a contract.”
* “In production I prefer symbol tokens to avoid collisions and keep DI predictable.”

---

# PaymentsModule — Tokens + Factory Providers + Test Overrides (Project-Grade)

Below is a clean, copy-paste-friendly structure showing:

* `PAYMENTS_GATEWAY` token
* `DB_CONN_WRITE` / `DB_CONN_READ` tokens
* **factory providers** for Redis + Kafka clients
* **how to override** these providers in Nest tests

---

## 1) Tokens (`tokens.ts`)

Use **Symbols** to avoid collisions.

```ts
// src/payments/tokens.ts
export const PAYMENTS_GATEWAY = Symbol('PAYMENTS_GATEWAY');

export const DB_CONN_WRITE = Symbol('DB_CONN_WRITE');
export const DB_CONN_READ = Symbol('DB_CONN_READ');

export const REDIS_CLIENT = Symbol('REDIS_CLIENT');
export const KAFKA_PRODUCER = Symbol('KAFKA_PRODUCER');
```

---

## 2) Interfaces / Contracts (optional but interview-strong)

```ts
// src/payments/contracts.ts
export interface PaymentsGateway {
  charge(input: { amount: number; currency: string; customerId: string }): Promise<{ id: string; status: string }>;
  refund(input: { chargeId: string; amount?: number }): Promise<{ id: string; status: string }>;
}
```

---

## 3) Infrastructure creators (factory helpers)

Keep creation logic outside module files.

```ts
// src/infra/redis/create-redis.ts
export function createRedisClient(url: string) {
  // return new Redis(url) or ioredis client etc.
  return { url, get: async () => null, set: async () => 'OK', quit: async () => undefined };
}
```

```ts
// src/infra/kafka/create-kafka.ts
export function createKafkaProducer(brokers: string[]) {
  // return new Kafka({ brokers }).producer() etc.
  return { brokers, connect: async () => undefined, send: async () => undefined, disconnect: async () => undefined };
}
```

---

## 4) Providers (DB + Redis + Kafka + Gateway)

### DB provider example (read/write separated)

```ts
// src/infra/db/db.providers.ts
import { DB_CONN_READ, DB_CONN_WRITE } from '../payments/tokens';

export const dbProviders = [
  {
    provide: DB_CONN_WRITE,
    useFactory: () => {
      // create / return your write-optimized connection or ORM dataSource
      return { role: 'write', query: async (_sql: string) => [] };
    },
  },
  {
    provide: DB_CONN_READ,
    useFactory: () => {
      // create / return your read-replica connection
      return { role: 'read', query: async (_sql: string) => [] };
    },
  },
];
```

### Redis + Kafka factory providers

```ts
// src/infra/clients/client.providers.ts
import { REDIS_CLIENT, KAFKA_PRODUCER } from '../payments/tokens';
import { createRedisClient } from './redis/create-redis';
import { createKafkaProducer } from './kafka/create-kafka';

export const clientProviders = [
  {
    provide: REDIS_CLIENT,
    useFactory: () => createRedisClient(process.env.REDIS_URL ?? 'redis://localhost:6379'),
  },
  {
    provide: KAFKA_PRODUCER,
    useFactory: async () => {
      const producer = createKafkaProducer((process.env.KAFKA_BROKERS ?? 'localhost:9092').split(','));
      await producer.connect();
      return producer;
    },
  },
];
```

### Gateway provider (`PAYMENTS_GATEWAY`)

```ts
// src/payments/gateways/stripe.gateway.ts
import { PaymentsGateway } from '../contracts';

export class StripeGateway implements PaymentsGateway {
  async charge() {
    return { id: 'ch_123', status: 'succeeded' };
  }
  async refund() {
    return { id: 're_123', status: 'succeeded' };
  }
}
```

```ts
// src/payments/payments.providers.ts
import { PAYMENTS_GATEWAY } from './tokens';
import { StripeGateway } from './gateways/stripe.gateway';

export const paymentsProviders = [
  { provide: PAYMENTS_GATEWAY, useClass: StripeGateway },
];
```

---

## 5) Payments Service consumes tokens (constructor injection)

```ts
// src/payments/payments.service.ts
import { Inject, Injectable } from '@nestjs/common';
import { PAYMENTS_GATEWAY, DB_CONN_WRITE, DB_CONN_READ, REDIS_CLIENT, KAFKA_PRODUCER } from './tokens';
import type { PaymentsGateway } from './contracts';

@Injectable()
export class PaymentsService {
  constructor(
    @Inject(PAYMENTS_GATEWAY) private readonly gateway: PaymentsGateway,
    @Inject(DB_CONN_WRITE) private readonly dbWrite: { query: (sql: string) => Promise<any[]> },
    @Inject(DB_CONN_READ) private readonly dbRead: { query: (sql: string) => Promise<any[]> },
    @Inject(REDIS_CLIENT) private readonly redis: { get: (k: string) => Promise<any>; set: (k: string, v: any) => Promise<any> },
    @Inject(KAFKA_PRODUCER) private readonly kafka: { send: (args: any) => Promise<void> },
  ) {}

  async chargeCustomer(input: { amount: number; currency: string; customerId: string }) {
    // read-side lookup
    await this.dbRead.query('select * from customers where id = ...');

    const charge = await this.gateway.charge(input);

    // write-side persistence
    await this.dbWrite.query('insert into charges ...');

    // cache / event
    await this.redis.set(`charge:${charge.id}`, charge);
    await this.kafka.send({ topic: 'payments.charged', messages: [{ value: JSON.stringify(charge) }] });

    return charge;
  }
}
```

---

## 6) Module wiring

```ts
// src/payments/payments.module.ts
import { Module } from '@nestjs/common';
import { PaymentsService } from './payments.service';
import { paymentsProviders } from './payments.providers';
import { dbProviders } from '../infra/db/db.providers';
import { clientProviders } from '../infra/clients/client.providers';

@Module({
  providers: [
    PaymentsService,
    ...paymentsProviders,
    ...dbProviders,
    ...clientProviders,
  ],
  exports: [PaymentsService], // export service if other modules need it
})
export class PaymentsModule {}
```

---

# 7) How to override tokens in tests (critical interview part)

You override providers using Nest TestingModule.

## A) Unit-test style: override everything external (gateway/db/redis/kafka)

```ts
// src/payments/payments.service.spec.ts
import { Test } from '@nestjs/testing';
import { PaymentsService } from './payments.service';
import { PAYMENTS_GATEWAY, DB_CONN_READ, DB_CONN_WRITE, REDIS_CLIENT, KAFKA_PRODUCER } from './tokens';

describe('PaymentsService', () => {
  let service: PaymentsService;

  const gatewayMock = {
    charge: jest.fn().mockResolvedValue({ id: 'ch_test', status: 'succeeded' }),
    refund: jest.fn(),
  };

  const dbReadMock = { query: jest.fn().mockResolvedValue([]) };
  const dbWriteMock = { query: jest.fn().mockResolvedValue([]) };

  const redisMock = {
    get: jest.fn(),
    set: jest.fn().mockResolvedValue('OK'),
  };

  const kafkaMock = {
    send: jest.fn().mockResolvedValue(undefined),
  };

  beforeEach(async () => {
    const moduleRef = await Test.createTestingModule({
      providers: [
        PaymentsService,

        // original registrations can be replaced right here
        { provide: PAYMENTS_GATEWAY, useValue: gatewayMock },
        { provide: DB_CONN_READ, useValue: dbReadMock },
        { provide: DB_CONN_WRITE, useValue: dbWriteMock },
        { provide: REDIS_CLIENT, useValue: redisMock },
        { provide: KAFKA_PRODUCER, useValue: kafkaMock },
      ],
    }).compile();

    service = moduleRef.get(PaymentsService);
  });

  it('charges customer and emits event', async () => {
    const res = await service.chargeCustomer({ amount: 100, currency: 'INR', customerId: 'c1' });

    expect(gatewayMock.charge).toHaveBeenCalled();
    expect(dbWriteMock.query).toHaveBeenCalled();
    expect(redisMock.set).toHaveBeenCalledWith('charge:ch_test', expect.anything());
    expect(kafkaMock.send).toHaveBeenCalledWith(
      expect.objectContaining({ topic: 'payments.charged' }),
    );
    expect(res.id).toBe('ch_test');
  });
});
```

✅ Interview line:

> “In unit tests, I override infrastructure tokens with `useValue` mocks so I test business logic deterministically.”

---

## B) Integration-test style: import module, override only some providers

You can import `PaymentsModule` and override specific dependencies:

```ts
const moduleRef = await Test.createTestingModule({
  imports: [PaymentsModule],
})
  .overrideProvider(PAYMENTS_GATEWAY)
  .useValue(gatewayMock)
  .overrideProvider(KAFKA_PRODUCER)
  .useValue(kafkaMock)
  .compile();
```

Use this when you want to keep most real wiring but avoid external calls.

---

## C) Notes that interviewers like

* Prefer **symbol tokens** for safety (`Symbol('TOKEN')`)
* Use `useFactory` for **configured infra clients** (Redis/Kafka/DB)
* For testing:

  * **unit tests**: override all external infra
  * **integration tests**: override only boundaries (gateway, kafka), keep repo real (or use test DB)

---
