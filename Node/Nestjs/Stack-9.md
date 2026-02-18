# 1) Pipes

## 1.1 What are Pipes? (Interview answer)

**Pipes** run **before the controller handler** and are used to:

* **validate** incoming data
* **transform** incoming data (types, shapes)
* **provide defaults**

**Pipeline position:** Guards → Interceptors(before) → **Pipes** → Controller

Interview line:

> “I use pipes to validate and transform inputs at the boundary so business logic receives clean, typed data.”

---

## 1.2 Built-in Pipes (common ones)

Nest provides built-in pipes like:

* `ParseIntPipe` → `"123"` → `123`
* `ParseBoolPipe`
* `ParseUUIDPipe`
* `ParseArrayPipe`
* `DefaultValuePipe`

Example:

```ts
@Get(':id')
getOne(@Param('id', ParseIntPipe) id: number) {}
```

---

## 1.3 Binding pipes (where to apply)

### A) Parameter-level

```ts
getOne(@Param('id', ParseIntPipe) id: number) {}
```

### B) Route-level

```ts
@UsePipes(new ValidationPipe())
@Post()
create(@Body() dto: CreateUserDto) {}
```

### C) Controller-level

```ts
@UsePipes(new ValidationPipe())
@Controller('users')
export class UsersController {}
```

### D) Global-scoped pipes (best practice)

In `main.ts`:

```ts
app.useGlobalPipes(new ValidationPipe({ whitelist: true, transform: true }));
```

Interview line:

> “I prefer global ValidationPipe for consistent input safety; route-level pipes only for special cases.”

---

## 1.4 Schema-based validation (object schema validation)

Two popular approaches:

* **Class-based**: `class-validator` + `class-transformer` (most Nest projects)
* **Schema-based**: Zod/Joi/Yup (common in modern TS apps)

### Example: Schema-based (Zod-style mental model)

* Validate request body against schema
* Return structured errors

When schema validation shines:

* APIs with shared types between backend + frontend
* extremely strict parsing
* more predictable error shapes

---

## 1.5 The built-in `ValidationPipe` (the real production workhorse)

Typical global config:

* `whitelist: true` → strips unknown fields
* `forbidNonWhitelisted: true` → reject unknown fields (stricter)
* `transform: true` → auto-transform types into DTO classes
* `transformOptions` → implicit conversion if needed

Recommended BFSI-ish config:

```ts
app.useGlobalPipes(
  new ValidationPipe({
    whitelist: true,
    forbidNonWhitelisted: true,
    transform: true,
  }),
);
```

Interview line:

> “ValidationPipe prevents payload pollution, enforces contracts, and keeps services clean.”

---

## 1.6 Transformation use cases (very interview-relevant)

### Example: query params as numbers with defaults

```ts
@Get()
list(
  @Query('page', new DefaultValuePipe(1), ParseIntPipe) page: number,
  @Query('limit', new DefaultValuePipe(20), ParseIntPipe) limit: number,
) {}
```

### Providing defaults (project best practice)

Defaults should happen at boundary (pipe/controller) so service logic is deterministic.

Interview line:

> “Defaults belong at the boundary via pipes, so downstream code doesn’t branch on undefined.”

---

---

# 2) Guards

## 2.1 What are Guards? (Interview answer)

**Guards** decide whether a request is allowed to proceed.
They are used for:

* authentication (is user logged in?)
* authorization (does user have permission?)

They run **before pipes**.

Interview line:

> “Guards are policy gates. They decide access before any business logic executes.”

---

## 2.2 ExecutionContext (key interview topic)

Guards receive `ExecutionContext`, which can switch between:

* HTTP
* WebSockets
* Microservices (RPC)

```ts
const req = context.switchToHttp().getRequest();
```

Interview line:

> “ExecutionContext lets guards work consistently across transports by extracting request/user from the correct context.”

---

## 2.3 Authorization Guard (Role-based example)

### Step 1: Roles decorator sets metadata

```ts
export const Roles = (...roles: string[]) => SetMetadata('roles', roles);
```

### Step 2: Guard reads metadata and checks user roles

* reads roles required by handler
* compares with `req.user.roles`

Practical policy:

* `JwtAuthGuard` authenticates user
* `RolesGuard` authorizes based on roles

---

## 2.4 Binding guards

### Route-level

```ts
@UseGuards(JwtAuthGuard, RolesGuard)
@Roles('admin')
@Get('admin')
getAdmin() {}
```

### Controller-level

```ts
@UseGuards(JwtAuthGuard)
@Controller('users')
export class UsersController {}
```

### Global guards

Used for consistent auth across the app (use carefully for public routes).

Interview line:

> “I bind JwtAuthGuard at controller/global level and use Roles/Permissions per handler.”

---

## 2.5 Setting roles per handler (the clean approach)

Use decorator metadata per route:

```ts
@Roles('maker')
@Post('create')
createTxn() {}
```

```ts
@Roles('checker')
@Post('approve')
approveTxn() {}
```

(BFSI maker-checker flows love this.)

---

---

# 3) Interceptors

## 3.1 What are Interceptors? (Interview answer)

**Interceptors** wrap the request handling like an AOP layer.
They can:

* log timing
* transform responses
* map exceptions
* implement caching
* modify streams (RxJS)

Interview line:

> “Interceptors are cross-cutting wrappers around handlers—perfect for logging, metrics, caching, and response shaping.”

---

## 3.2 Core concepts: ExecutionContext + CallHandler

* `ExecutionContext` → access request/handler metadata
* `CallHandler` → continues execution and returns an Observable-like stream

```ts
intercept(context: ExecutionContext, next: CallHandler) {
  return next.handle().pipe(...operators);
}
```

---

## 3.3 Aspect interception (AOP-style)

Interceptors are used like “aspects”:

* before: start timer, attach correlation id
* after: log duration, map response

---

## 3.4 Binding interceptors

* method
* controller
* global

Production pattern:

* global `LoggingInterceptor`
* feature interceptors for response mapping or caching

---

## 3.5 Response mapping (common)

Example: enforce consistent success envelope:

```json
{ "requestId": "...", "data": ..., "meta": ... }
```

Interview line:

> “I use response mapping interceptors to ensure consistent API contracts and include requestId/meta.”

---

## 3.6 Exception mapping

Interceptors can catch and transform errors in-stream, but **global exception filters** are usually cleaner for HTTP error shaping.

Project advice:

* filters → error contract
* interceptors → success contract + metrics/logging

---

## 3.7 Stream overriding + operators

Interceptors can:

* override response stream
* retry (rare in HTTP)
* timeout
* transform with `map`, handle errors with `catchError`, finalize with `finalize`

Common operators:

* `map` → response shape
* `catchError` → translate errors (sparingly)
* `finalize` → duration logging even on error

---

---

# 4) Custom Route Decorators

## 4.1 Parameter decorators (extract data cleanly)

Example: `@CurrentUser()` decorator

* extracts `req.user` from request
* keeps controller clean

Interview line:

> “Custom param decorators remove repeated boilerplate and keep controllers thin.”

---

## 4.2 Passing data to decorators

Decorators can accept arguments:

* `@CurrentUser('id')` returns `user.id`

This helps keep handlers minimal and consistent.

---

## 4.3 Working with pipes (important)

Custom param decorators can be used with pipes:

* extract value
* then validate/transform

Example pattern:

* `@Param('id', ParseUUIDPipe)`
* `@Body(new ValidationPipe()) dto`

---

## 4.4 Decorator composition (very professional)

You can create a single decorator that applies:

* guards
* roles metadata
* swagger docs (if used)

Example idea:
`@AdminOnly()` internally does:

* `@UseGuards(JwtAuthGuard, RolesGuard)`
* `@Roles('admin')`

Interview line:

> “Decorator composition standardizes security policies and reduces mistakes.”

---

# 5) One-liner “when to use what” (interview gold)

* **Pipes** → validate/transform input (DTO safety)
* **Guards** → allow/deny access (authn/authz)
* **Interceptors** → cross-cutting wrapper (logging/metrics/response shaping)
* **Filters** → consistent error contract
* **Custom decorators** → cleaner controllers + consistent policies

---

---

# 0) Folder suggestion (clean + scalable)

```txt
src/
  common/
    auth/
      roles.decorator.ts
      roles.guard.ts
      current-user.decorator.ts
    interceptors/
      response-envelope.interceptor.ts
  main.ts
```

---

# 1) `@Roles()` Decorator

```ts
// src/common/auth/roles.decorator.ts
import { SetMetadata } from '@nestjs/common';

export const ROLES_KEY = 'roles';
export const Roles = (...roles: string[]) => SetMetadata(ROLES_KEY, roles);
```

What this does:

* attaches metadata `roles: ['admin', ...]` to the handler/class
* `RolesGuard` reads it and enforces authorization

---

# 2) `RolesGuard` (RBAC)

```ts
// src/common/auth/roles.guard.ts
import { CanActivate, ExecutionContext, Injectable } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { ROLES_KEY } from './roles.decorator';

type AppUser = {
  id: string;
  roles?: string[];
  permissions?: string[];
};

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private readonly reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    // roles required by handler/class
    const requiredRoles =
      this.reflector.getAllAndOverride<string[]>(ROLES_KEY, [
        context.getHandler(),
        context.getClass(),
      ]) ?? [];

    // If no roles required, allow
    if (requiredRoles.length === 0) return true;

    const req = context.switchToHttp().getRequest();
    const user: AppUser | undefined = req.user;

    // If user missing, deny (assumes AuthGuard should have populated user)
    if (!user) return false;

    const userRoles = user.roles ?? [];
    return requiredRoles.some((r) => userRoles.includes(r));
  }
}
```

### Usage

```ts
// example usage in controller
@Roles('admin')
@UseGuards(JwtAuthGuard, RolesGuard)
@Get('admin-area')
getAdminArea() {
  return { ok: true };
}
```

> Note: `JwtAuthGuard` is whatever auth guard you use to authenticate and set `req.user`.

---

# 3) `@CurrentUser()` Decorator (param decorator)

### A) Basic: return full user

```ts
// src/common/auth/current-user.decorator.ts
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const CurrentUser = createParamDecorator(
  (data: keyof any | undefined, ctx: ExecutionContext) => {
    const req = ctx.switchToHttp().getRequest();
    const user = req.user;

    // If data is provided, return a specific field (e.g., 'id')
    return data ? user?.[data] : user;
  },
);
```

### Usage

```ts
@Get('me')
@UseGuards(JwtAuthGuard)
getMe(@CurrentUser() user: any) {
  return user;
}

@Get('me/id')
@UseGuards(JwtAuthGuard)
getMyId(@CurrentUser('id') userId: string) {
  return { userId };
}
```

Interview line:

> “Custom param decorators reduce boilerplate and keep controllers thin and consistent.”

---

# 4) `ResponseEnvelopeInterceptor` → `{ requestId, data }`

This standardizes **success responses** and includes `requestId` for traceability.

```ts
// src/common/interceptors/response-envelope.interceptor.ts
import {
  CallHandler,
  ExecutionContext,
  Injectable,
  NestInterceptor,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

@Injectable()
export class ResponseEnvelopeInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const http = context.switchToHttp();
    const req = http.getRequest();

    const requestId = req.requestId ?? req.headers['x-request-id'] ?? 'unknown';

    return next.handle().pipe(
      map((data) => ({
        requestId,
        data,
      })),
    );
  }
}
```

### Apply globally in `main.ts`

```ts
app.useGlobalInterceptors(new ResponseEnvelopeInterceptor());
```

Interview framing:

> “Interceptors are ideal for response shaping and for injecting correlation/request IDs into every response.”

---

# 5) Global ValidationPipe (strict options)

This is the standard “bank-grade” configuration.

```ts
// main.ts (snippet)
import { ValidationPipe } from '@nestjs/common';

app.useGlobalPipes(
  new ValidationPipe({
    whitelist: true,               // strip unknown properties
    forbidNonWhitelisted: true,    // throw if unknown properties exist
    transform: true,               // transform payloads into DTO instances
    transformOptions: {
      enableImplicitConversion: true, // converts "1" -> 1 for number fields (use carefully)
    },
    // stopAtFirstError: true,      // optional: reduce noise in validation responses
    // forbidUnknownValues: true,   // optional: stricter object handling
  }),
);
```

### DTO Example (for context)

```ts
export class CreatePaymentDto {
  amount: number;
  currency: string;
  customerId: string;
}
```

Interview line:

> “Global ValidationPipe ensures API contract safety everywhere—unknown fields rejected, DTO transformation enabled, and inputs are sanitized before reaching services.”

---

# 6) Wire everything in `main.ts` (complete minimal)

```ts
// src/main.ts
import { NestFactory } from '@nestjs/core';
import { ValidationPipe } from '@nestjs/common';
import { AppModule } from './app.module';
import { ResponseEnvelopeInterceptor } from './common/interceptors/response-envelope.interceptor';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Strict validation
  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,
      forbidNonWhitelisted: true,
      transform: true,
      transformOptions: { enableImplicitConversion: true },
    }),
  );

  // Success envelope
  app.useGlobalInterceptors(new ResponseEnvelopeInterceptor());

  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```

---

# 7) Quick “how to explain this in interviews”

* **Roles decorator** sets required roles as metadata.
* **RolesGuard** reads metadata via `Reflector` and checks `req.user.roles`.
* **CurrentUser decorator** extracts user or `user.id` cleanly, avoiding repetitive `@Req()` usage.
* **ResponseEnvelopeInterceptor** enforces consistent success responses with `requestId`.
* **Global ValidationPipe** enforces strict request contracts, prevents payload pollution, and transforms input types.
