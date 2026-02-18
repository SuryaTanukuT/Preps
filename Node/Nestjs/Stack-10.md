---

## 1) Custom Providers

### What

A **custom provider** lets you register dependencies using:

* `useValue` (constant)
* `useClass` (swap implementations)
* `useExisting` (alias)
* `useFactory` (dynamic construction)

### Why (interview)

> “Custom providers let me inject primitives/SDK clients and swap implementations cleanly via injection tokens.”

### Project use-cases

* `PAYMENTS_GATEWAY` token → switch Stripe/Razorpay
* `DB_CONN_READ/WRITE` tokens
* `REDIS_CLIENT`, `KAFKA_PRODUCER` created via factory

### Example

```ts
export const PAYMENTS_GATEWAY = Symbol('PAYMENTS_GATEWAY');

@Module({
  providers: [
    { provide: PAYMENTS_GATEWAY, useClass: StripeGateway },
    { provide: 'REGION', useValue: 'ap-south-1' },
  ],
})
export class PaymentsModule {}
```

Inject:

```ts
constructor(@Inject(PAYMENTS_GATEWAY) private gw: PaymentsGateway) {}
```

---

## 2) Asynchronous Providers

### What

Providers that need **async initialization** (connect to DB/Redis/Kafka) and must be `await`ed before ready.

### Why (interview)

> “Async providers ensure infrastructure clients are initialized once and injected safely across the app.”

### Example (async factory)

```ts
{
  provide: KAFKA_PRODUCER,
  useFactory: async (config: ConfigService) => {
    const producer = createKafkaProducer(config.get('KAFKA_BROKERS').split(','));
    await producer.connect();
    return producer;
  },
  inject: [ConfigService],
}
```

Best practice: clean up on shutdown using lifecycle hooks (`OnApplicationShutdown`).

---

## 3) Dynamic Modules

### What

A module whose providers/imports depend on runtime configuration. Usually exposed as `Module.forRoot()` / `forRootAsync()`.

### Why (interview)

> “Dynamic modules provide configurable infrastructure (DB, cache, external SDKs) without hardcoding environment details.”

### Project use-cases

* multi-tenant DB connections
* enable/disable features (audit, tracing)
* different gateway implementations per env

### Example

```ts
@Module({})
export class DatabaseModule {
  static forRoot(connStr: string): DynamicModule {
    return {
      module: DatabaseModule,
      providers: [{ provide: DB_CONN_WRITE, useValue: createConn(connStr) }],
      exports: [DB_CONN_WRITE],
    };
  }
}
```

---

## 4) Injection Scopes

### What

Provider lifetime:

* **Singleton** (default): one instance for whole app
* **Request**: new instance per request
* **Transient**: new instance every injection

### Why (interview)

> “Singletons are fastest. Request scope is only for request-bound context (tenant, correlation). Transient is rare.”

### How

```ts
@Injectable({ scope: Scope.REQUEST })
export class RequestContextService {}
```

Request scope increases overhead; use only when required.

---

## 5) Circular Dependency

### What

A depends on B, and B depends on A → DI can’t resolve normally.

### Why it happens (project)

* AuthService needs UsersService; UsersService calls AuthService for hashing/tokens (design smell)
* Domain modules call each other directly instead of using an abstraction

### Fix options (best order)

1. **Refactor**: extract shared logic into a third service/module
2. Use `forwardRef()` (if refactor isn’t feasible)

### Example

```ts
@Module({
  imports: [forwardRef(() => UsersModule)],
})
export class AuthModule {}
```

Inject:

```ts
constructor(@Inject(forwardRef(() => UsersService)) private users: UsersService) {}
```

Interview line:

> “I treat circular dependencies as an architecture smell and prefer extracting a shared abstraction over relying on forwardRef.”

---

## 6) Module Reference (`ModuleRef`)

### What

`ModuleRef` is a low-level API to resolve providers dynamically (imperatively) from the DI container.

### When to use (real-world)

* plugin architectures (resolve handler by name)
* strategy pattern (choose implementation at runtime)
* request-scoped resolution in advanced flows

### Example

```ts
@Injectable()
export class HandlerFactory {
  constructor(private readonly moduleRef: ModuleRef) {}

  get(name: string) {
    return this.moduleRef.get(name, { strict: false });
  }
}
```

Overuse turns DI into a service locator. Use sparingly.

---

## 7) Lazy-loading Modules

### What

Load a module only when needed (not at startup). Useful when:

* heavy module, rarely used
* plugins/optional features
* multi-tenant “feature packs”

### Common Nest pattern

Use `LazyModuleLoader` to load module at runtime.

```ts
@Injectable()
export class ReportsService {
  constructor(private readonly lazyLoader: LazyModuleLoader) {}

  async loadReports() {
    const moduleRef = await this.lazyLoader.load(() => ReportsModule);
    return moduleRef.get(ReportsRunner);
  }
}
```

Interview line:

> “Lazy-loading reduces startup cost and enables optional features, but I ensure dependencies and caching are managed.”

---

## 8) Execution Context

### What

`ExecutionContext` describes the current runtime context:

* HTTP
* WebSockets
* Microservices/RPC

Used in **guards/interceptors/filters/decorators**.

### Why (interview)

> “ExecutionContext makes cross-transport code possible—same guard can work for HTTP or RPC by switching context.”

Example:

```ts
const req = context.switchToHttp().getRequest();
```

---

## 9) Lifecycle Events (Lifecycle Hooks)

### What

Hooks that run at startup/shutdown/module init.

Common hooks:

* `OnModuleInit`
* `OnApplicationBootstrap`
* `OnModuleDestroy`
* `BeforeApplicationShutdown`
* `OnApplicationShutdown`

### Project use-cases

* connect/disconnect Kafka producer
* warm caches
* verify DB migrations
* flush telemetry buffers on shutdown

Example:

```ts
@Injectable()
export class KafkaLifecycle implements OnApplicationShutdown {
  constructor(@Inject(KAFKA_PRODUCER) private producer: any) {}

  async onApplicationShutdown() {
    await this.producer.disconnect();
  }
}
```

---

## 10) Discovery Service

### What

A utility to **discover providers/controllers** at runtime using metadata—useful for frameworks/plugins.

Commonly used via `@nestjs/core` discovery utilities.

### Project use-cases

* auto-register event handlers annotated with `@EventHandler()`
* build a job registry (`@CronJob()`)
* scan for policy classes (`@Policy()`)

Interview line:

> “Discovery is for advanced plugin-like patterns where the system auto-wires handlers based on metadata.”

---

## 11) Platform Agnosticism

### What

Nest separates application logic from the underlying platform adapter:

* HTTP adapters: Express / Fastify
* microservices transport adapters

### Why (interview)

> “Nest is platform-agnostic because controllers/providers are independent of the HTTP engine; swapping adapters doesn’t rewrite business logic.”

### Project guidance

* Don’t overuse `@Req()`/`@Res()` (ties you to the platform)
* Prefer DTOs, interceptors, and framework abstractions

---

## 12) Testing (unit, integration, e2e)

### What interviewers want to hear

> “I test at DI boundaries: unit tests mock tokens, integration tests wire real modules with some overrides, e2e tests hit HTTP endpoints.”

### A) Unit tests (mock tokens)

* override gateway/DB/Redis/Kafka using `useValue`

```ts
.overrideProvider(PAYMENTS_GATEWAY)
.useValue(gatewayMock)
```

### B) Integration tests

* import the module, override only external boundaries
* optionally use test DB/containers

### C) E2E tests

* boot the full app, hit endpoints via supertest
* assert HTTP contract + error contract + requestId header

Tooling (typical):

* Jest + Nest testing utilities

Interview line:

> “Tokens make testing easy—I can override infrastructure clients cleanly without changing business code.”

---

# Super-compact “interview recap”

* **Custom/async providers + dynamic modules** → configurable infrastructure and clean DI
* **Scopes** → singleton by default; request scope only for request context
* **Circular deps** → refactor first; `forwardRef` only when unavoidable
* **ModuleRef/Lazy loader/Discovery** → advanced plugin + runtime resolution patterns
* **ExecutionContext** → transport-agnostic guards/interceptors/filters
* **Testing** → override tokens; unit/integration/e2e pyramid
