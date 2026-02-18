---

## 1) Modules (Why they exist, how to explain in interviews)

### Interview explanation (30–45 sec)

In NestJS, a **module** is the **unit of architecture**. It defines a **feature boundary** and tells Nest:

* what providers (services/repos) exist in that feature,
* what controllers expose APIs,
* what other modules it depends on,
* what it exports for other modules to reuse.
  This solves the “Express spaghetti” problem by enforcing **clear boundaries + DI wiring**.

---

## 2) `@Module()` decorator — Imports / Providers / Exports (mental model)

### The mental model: “Module = wiring container”

```ts
@Module({
  imports: [...],     // what I depend on (other modules)
  providers: [...],   // what I create (DI-managed classes)
  controllers: [...], // what I expose via HTTP
  exports: [...],     // what I allow others to use
})
export class UsersModule {}
```

### What each field means (project-style)

* **imports**: “I need these features/providers from other modules”
* **providers**: “These are my services/repos/helpers; Nest can inject them”
* **controllers**: “These classes define routes/endpoints for this module”
* **exports**: “Other modules may use these providers”

Rule of thumb:
**If another module needs a provider, export it. If you don’t export it, it stays private.**
--------------------------------------------------------------------------------------------

## 3) Feature Modules (real project architecture)

### Project pattern

You break the system into domain features:

* `AuthModule`
* `UsersModule`
* `PaymentsModule`
* `OrdersModule`

Each module contains:

* controller(s) (API layer)
* service(s) (use-case/business layer)
* repository/DAO (data layer)
* DTOs (input contracts)
* guards/interceptors/pipes if feature-specific

Interview line:

> “Feature modules map cleanly to domains and teams, making refactors safer and ownership clearer.”

---

## 4) Shared Modules, Module Re-exporting

### Shared module (common utilities)

A `SharedModule` holds cross-cutting utilities:

* `ConfigService`
* `Logger`
* `CryptoService`
* `HttpClient`

```ts
@Module({
  providers: [LoggerService],
  exports: [LoggerService],
})
export class SharedModule {}
```

### Module re-exporting (clean dependency surface)

Instead of every module importing many modules, you can create an “aggregator module”:

```ts
@Module({
  imports: [UsersModule, AuthModule],
  exports: [UsersModule, AuthModule],
})
export class CoreModule {}
```

Why it’s used:

* avoids repetitive imports everywhere
* provides a clean “public API” of your internal modules

## Use carefully in large systems (avoid turning CoreModule into a dumping ground).

## 5) Dependency Injection (DI) inside Modules — interview + practical

### What DI is (interview)

> “DI means Nest constructs objects for you and injects dependencies, instead of manually `new`-ing them. This reduces coupling and improves testing.”

### Example

```ts
@Injectable()
export class UsersService {
  constructor(private readonly usersRepo: UsersRepo) {}
}
```

DI resolution happens because:

1. `UsersRepo` is listed in `providers`
2. `UsersService` is listed in `providers`
3. `UsersService` constructor asks for `UsersRepo`

Testing advantage:

* you can replace `UsersRepo` with a mock provider easily.

---

## 6) Global Modules (use sparingly)

### What it is

A module marked global can be used **without importing it** everywhere.

```ts
@Global()
@Module({
  providers: [ConfigService],
  exports: [ConfigService],
})
export class ConfigModule {}
```

### Interview guidance

* Good for truly global cross-cutting concerns (config, logger)
* Overuse makes dependencies “invisible” and reduces clarity

Interview line:

> “Global modules reduce boilerplate, but I use them only for true infrastructure services to avoid hidden coupling.”

---

## 7) Dynamic Modules (advanced + real use cases)

### What it is

A **dynamic module** lets you configure a module at import time.

Common examples:

* `ConfigModule.forRoot({ ... })`
* database module with env-based config
* multi-tenant or multi-region setups

```ts
@Module({})
export class DatabaseModule {
  static forRoot(connStr: string): DynamicModule {
    return {
      module: DatabaseModule,
      providers: [{ provide: 'DB_CONN', useValue: connStr }],
      exports: ['DB_CONN'],
    };
  }
}
```

Interview line:

> “Dynamic modules are used when module providers depend on runtime config like environment variables, tenant configuration, or connection strategies.”

---

---

# Controllers (Interview + Project-Oriented)

## 8) `@Controller()` — what it represents

### Interview explanation

A **controller** is the **transport layer** for a feature.

* It receives requests
* maps inputs (`@Body`, `@Param`, `@Query`)
* calls services (business logic)
* returns responses

Key principle:

> Controllers should be thin. Services should hold business logic.

---

## 9) Route handlers (`@Get`, `@Post`, etc.) + Routing

```ts
@Controller('users')
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Get(':id')
  getUser(@Param('id') id: string) {
    return this.usersService.getById(id);
  }

  @Post()
  createUser(@Body() dto: CreateUserDto) {
    return this.usersService.create(dto);
  }
}
```

How routing works:

* `@Controller('users')` → prefix `/users`
* `@Get(':id')` → route `/users/:id`
* Nest builds route metadata during bootstrap

---

## 10) Request object, routing, state sharing (important interview nuance)

### Request object

You *can* access raw request:

* `@Req()` (Express/Fastify request)

But project-best-practice: **prefer decorators** (`@Param`, `@Body`, etc.) for clean contracts.

### State sharing across layers

Typical patterns:

* attach request context in middleware (e.g., `requestId`, user)
* access it in guards/interceptors/services via request scope or AsyncLocalStorage (advanced)
* avoid global mutable state (breaks concurrency)

Interview line:

> “I prefer request-scoped context propagation rather than shared global state to keep concurrency safe.”

---

## 11) Route params (`@Param`, `@Query`, `@Body`) + Query parameters

* `@Param()` → path params
* `@Query()` → querystring
* `@Body()` → JSON body

```ts
@Get()
list(@Query('page') page = '1', @Query('limit') limit = '20') {}
```

Best practice:

* validate query params with DTOs too, not just bodies.

---

## 12) Route wildcards + resources

### Wildcards

```ts
@Get('files/*')
getAnyFile() {}
```

Use cases:

* reverse proxy style routes
* file serving / fallback

### REST resources

Design around nouns:

* `GET /users/:id`
* `POST /users`
* `PATCH /users/:id`

---

## 13) Response headers, redirection, status codes

### Set status codes

```ts
@Post()
@HttpCode(201)
create() {}
```

### Set response headers

```ts
@Header('Cache-Control', 'no-store')
@Get('me')
me() {}
```

### Redirect

```ts
@Redirect('https://example.com', 302)
@Get('go')
go() {}
```

Interview line:

> “I use explicit status codes and headers for API correctness—especially important for caching, idempotency, and security.”

---

## 14) Sub-domain routing (advanced)

Nest can route based on host (Express supports this via router/host patterns).

Use-case:

* `tenant1.api.com` vs `tenant2.api.com` multi-tenant APIs

In practice, many teams implement this at:

* API Gateway / Ingress level, or
* middleware that resolves tenant from host header

Interview framing:

> “Subdomain routing is useful for multi-tenancy; I usually resolve tenant at gateway/middleware and keep controller routes stable.”

---

## 15) Asynchronicity (how Nest handles async)

In Nest you can:

* return `Promise<T>` from handlers (most common)
* use async/await
* use streams/RxJS when needed (advanced)

```ts
@Get(':id')
async getUser(@Param('id') id: string) {
  return await this.usersService.getById(id);
}
```

Interview line:

> “Nest naturally supports async handlers; exceptions thrown in async flows are caught by filters.”

---

## 16) Handle errors (project best practice)

Typical approach:

* throw `HttpException` or specific ones like `NotFoundException`
* use **global exception filter** for consistent error format
* never leak internal errors

```ts
if (!user) throw new NotFoundException('User not found');
```

Interview line:

> “I use typed exceptions + a global filter to enforce consistent error contracts across services.”

---

---

# DTOs (Data Transfer Objects) — why they matter in real projects

## 17) What DTOs are (interview explanation)

A **DTO** is a **contract** for incoming/outgoing data.

It helps with:

* validation
* transformation
* versioning
* documentation (Swagger)
* clarity in APIs

## DTOs are critical for production APIs because they prevent “accept anything” behavior.

## 18) DTOs with validation (typical Nest pattern)

```ts
export class CreateUserDto {
  name: string;
  email: string;
}
```

In real projects you add validation + transformation via global pipe:

* `ValidationPipe` validates DTO rules
* can strip unknown fields
* can auto-transform types

Interview line:

> “DTOs + ValidationPipe give me API safety—rejecting malformed payloads early before business logic runs.”

---

## 19) DTOs for query + params (often missed in interviews)

Best practice:

* Use DTOs not just for bodies, but for query params too:

  * pagination
  * filtering
  * sorting
  * date ranges

Interview line:

> “I treat query params as first-class input and validate them with DTOs to avoid edge-case bugs and injection-style issues.”

---

## 20) Project-style summary (what you’d say confidently)

* **Modules** define feature boundaries and DI wiring (imports/providers/exports).
* **Feature modules** map to domains; **shared modules** hold reusable infrastructure.
* **Re-exporting** helps create clean dependency surfaces.
* **Global modules** reduce boilerplate but should be minimal to avoid hidden coupling.
* **Dynamic modules** enable runtime-configurable infrastructure (DB, config, multi-tenant).
* **Controllers** stay thin; they map HTTP to services.
* **DTOs** enforce API contracts and protect business logic with validation and transformation.
