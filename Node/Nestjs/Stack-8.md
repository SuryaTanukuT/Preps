# NestJS Middleware & Exception Filters (Interview + Project-Oriented, Markdown)

---

# 1) Middleware (NestJS)

## What middleware is (interview-ready)

**Middleware** runs **before Nest hits guards/pipes/interceptors/controllers**.
It’s best for **request-level pre-processing** at the HTTP adapter layer (Express/Fastify):

Typical uses:

* request/correlation id
* basic logging
* CORS tweaks
* parsing/normalizing headers
* simple allow/deny (rare; usually guards are better)

**Key point:**

> Middleware is **framework-level** (Express/Fastify style). For auth/authorization, prefer **Guards**.

---

## Where middleware sits in the pipeline

**Request order (simplified):**
**Middleware → Guards → Interceptors → Pipes → Controller → Service → (Filters on error)**

---

## Dependency Injection in Middleware

### 1) Functional middleware (no DI)

Simple function form (fast, but no DI injection):

```ts
export function requestId(req, res, next) {
  req.requestId = req.headers['x-request-id'] ?? crypto.randomUUID();
  next();
}
```

### 2) Class-based middleware (supports DI)

If you need DI (ConfigService, Logger, etc.), use class middleware:

```ts
import { Injectable, NestMiddleware } from '@nestjs/common';

@Injectable()
export class RequestIdMiddleware implements NestMiddleware {
  use(req: any, _res: any, next: () => void) {
    req.requestId = req.headers['x-request-id'] ?? crypto.randomUUID();
    next();
  }
}
```

Interview line:

> “If middleware needs injected services (logger/config), I use class-based middleware; otherwise functional middleware is fine.”

---

## Applying middleware (Module-level with MiddlewareConsumer)

You register middleware in a module implementing `NestModule`.

```ts
import { MiddlewareConsumer, Module, NestModule, RequestMethod } from '@nestjs/common';

@Module({})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(RequestIdMiddleware)
      .forRoutes({ path: '*', method: RequestMethod.ALL });
  }
}
```

### Route wildcards

* Apply to all routes: `path: '*'`
* Apply to a prefix: `{ path: 'payments/*', method: ALL }`

Practical use:

* apply request-id middleware to all routes
* apply body-size limit only to uploads
* apply raw-body parser only to webhook endpoints

---

## Excluding routes (common in projects)

Example: apply middleware to all routes **except** health checks:

```ts
consumer
  .apply(RequestIdMiddleware)
  .exclude(
    { path: 'health', method: RequestMethod.GET },
    { path: 'metrics', method: RequestMethod.GET },
  )
  .forRoutes({ path: '*', method: RequestMethod.ALL });
```

Interview line:

> “I exclude health/metrics routes to reduce noise and keep probes lightweight.”

---

## Multiple middleware (ordering matters)

Middleware runs in the order you apply them:

```ts
consumer
  .apply(RequestIdMiddleware, AccessLogMiddleware)
  .forRoutes({ path: '*', method: RequestMethod.ALL });
```

**Ordering guidance:**

1. request-id / correlation id
2. logging
3. request normalization

---

## Global middleware

### Option A: `app.use()` (common)

In `main.ts`:

```ts
app.use(requestId); // functional middleware
```

### Option B: global via module consumer (preferred in modular projects)

Use `consumer.apply(...).forRoutes('*')` so it’s visible in module config.

Interview framing:

> “For global middleware I prefer module-based registration because it’s explicit and consistent with Nest module structure.”

---

## Middleware Consumer (what it really is)

`MiddlewareConsumer` is Nest’s DSL to:

* apply middleware
* choose routes
* exclude routes
* control ordering
* keep config close to modules

---

# 2) Exception Filters (NestJS)

## What exception filters are (interview-ready)

**Exception filters** handle errors **after** something throws (controller/service/pipe/guard/interceptor).
They convert exceptions into **consistent HTTP responses** and can log/trace them.

Why it matters:

* consistent error shape
* no internal stack traces leaked
* centralized logging + correlation id
* maps domain errors → HTTP codes cleanly

**Interview line:**

> “I use a global exception filter to enforce a consistent error contract and to centralize logging and sanitization.”

---

## Throwing standard exceptions (built-in HTTP exceptions)

Nest provides built-in exceptions:

* `BadRequestException` (400)
* `UnauthorizedException` (401)
* `ForbiddenException` (403)
* `NotFoundException` (404)
* `ConflictException` (409)
* `UnprocessableEntityException` (422)
* `InternalServerErrorException` (500)

Example:

```ts
if (!user) throw new NotFoundException('User not found');
```

Interview tip:

> “I throw explicit HTTP exceptions at the boundary; deep domain errors get mapped in a global filter.”

---

## Custom exceptions (domain-level)

Create domain errors that are not tied to HTTP:

```ts
export class InsufficientBalanceError extends Error {
  constructor() {
    super('Insufficient balance');
    this.name = 'InsufficientBalanceError';
  }
}
```

Then map them to HTTP in a filter (better separation).

---

## Exception logging (production best practice)

When an exception occurs, log:

* requestId / correlationId
* route/method
* userId (if available)
* status code
* error name/message
* stack trace (server-side only)

Never return stack traces to clients in prod.

---

## Exception filters: core mechanics

A filter looks like:

```ts
import { ArgumentsHost, Catch, ExceptionFilter, HttpException, HttpStatus } from '@nestjs/common';

@Catch() // catch everything
export class AllExceptionsFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const req = ctx.getRequest();
    const res = ctx.getResponse();

    const isHttp = exception instanceof HttpException;
    const status = isHttp ? exception.getStatus() : HttpStatus.INTERNAL_SERVER_ERROR;

    const responseBody = isHttp
      ? exception.getResponse()
      : { message: 'Internal server error' };

    // log here (use your logger)
    // logger.error({ requestId: req.requestId, exception });

    res.status(status).json({
      requestId: req.requestId,
      timestamp: new Date().toISOString(),
      path: req.url,
      ...(typeof responseBody === 'string' ? { message: responseBody } : responseBody),
    });
  }
}
```

### ArgumentsHost (what interviewers want)

`ArgumentsHost` gives access to the current execution context:

* HTTP: `switchToHttp()`
* RPC/microservices: `switchToRpc()`
* WebSockets: `switchToWs()`

Interview line:

> “ArgumentsHost lets the same filter work across HTTP/RPC/WS by switching context appropriately.”

---

## Binding filters (where you apply them)

### 1) Global filter (recommended)

In `main.ts`:

```ts
app.useGlobalFilters(new AllExceptionsFilter());
```

### 2) Controller-level

```ts
@UseFilters(AllExceptionsFilter)
@Controller('payments')
export class PaymentsController {}
```

### 3) Route-level

```ts
@UseFilters(AllExceptionsFilter)
@Get(':id')
getOne() {}
```

Interview guidance:

> “I typically apply a global filter for consistent API error shape; feature filters only when needed for special mapping.”

---

## Catch everything vs specific filters

### Catch everything

`@Catch()` without args catches all exceptions (good global safety net).

### Specific exception filter

Catch only one type:

```ts
@Catch(InsufficientBalanceError)
export class BalanceFilter implements ExceptionFilter { ... }
```

**Best practice**:

* use one global catch-all filter
* add narrow filters for special domain mapping if needed

---

## Inheritance (clean pattern for big systems)

You can build a base filter for logging & formatting, then extend:

* `BaseExceptionFilter` → common response shape + logging
* `PaymentsExceptionFilter` → maps payment domain errors
* `AuthExceptionFilter` → maps auth-specific cases

Interview line:

> “I use inheritance/composition to keep a consistent error contract while allowing domain-specific mappings.”

---

# Middleware vs Exception Filters (quick comparison)

* **Middleware**: runs *before* route handling; good for request setup (requestId, basic logs)
* **Exception Filters**: run *after* an error is thrown; good for response formatting + logging

---

## BFSI-grade best practice (what to say)

* Add middleware to generate **correlation/request id**
* Global exception filter returns consistent error format:

  * `requestId`, `timestamp`, `path`, `errorCode`, `message`
* Log enriched error context server-side (no stack trace to clients)
* Map domain errors (like insufficient balance) to correct HTTP status (409/422)
