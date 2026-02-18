# NestJS “Techniques” Cheat-Sheet (Production + Interview + Project-Oriented)

This is how you typically wire these concerns in a real NestJS backend (REST + optional microservices).

---

## 1) Configuration

**Goal:** one source of truth for env + typed config.

**Typical approach**

* `ConfigModule.forRoot({ isGlobal: true })`
* `ConfigService.getOrThrow()` for required envs
* validate env at startup (zod/joi) to fail fast

```ts
// app.module.ts
import { ConfigModule } from '@nestjs/config';

@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true }), // process.env -> ConfigService
  ],
})
export class AppModule {}
```

**Interview line:** “I centralize config in ConfigModule, validate env at boot, and inject ConfigService everywhere.”

---

## 2) Database (general patterns)

**Goal:** clean data access + transactions + testability.

Common options in Nest:

* SQL: TypeORM / Prisma / Sequelize (via providers)
* Mongo: Mongoose (most common)

**Project pattern**

* `DatabaseModule` (dynamic module) provides connections
* feature modules inject repositories/services

---

## 3) Mongo (Mongoose)

**Goal:** schema + models + DI.

```ts
import { MongooseModule } from '@nestjs/mongoose';

@Module({
  imports: [
    MongooseModule.forRoot(process.env.MONGO_URI!),
    MongooseModule.forFeature([{ name: 'User', schema: UserSchema }]),
  ],
})
export class UsersModule {}
```

**Tips**

* keep Mongoose models inside feature modules
* use indexes + lean queries for performance
* avoid business logic inside schema hooks; keep it in services

---

## 4) Validation

**Goal:** reject bad input at the boundary.

**Global strict `ValidationPipe`**

```ts
// main.ts
app.useGlobalPipes(new ValidationPipe({
  whitelist: true,
  forbidNonWhitelisted: true,
  transform: true,
  transformOptions: { enableImplicitConversion: true },
}));
```

**Interview line:** “Global validation prevents payload pollution and guarantees services receive clean DTOs.”

---

## 5) Caching

**Goal:** reduce DB load, speed up reads.

**Built-in cache**

* `CacheModule` (often backed by Redis in prod)
* use interceptors or manual cache get/set in services

```ts
import { CacheModule } from '@nestjs/cache-manager';

@Module({
  imports: [CacheModule.register({ isGlobal: true })],
})
export class AppModule {}
```

**Pattern**

* cache-aside for user/profile/catalog
* TTL per endpoint
* invalidate on writes (events/outbox if needed)

---

## 6) Serialization

**Goal:** consistent, safe response shaping (hide secrets, transform types).

* `ClassSerializerInterceptor` + `class-transformer`
* mark fields with `@Exclude()` / `@Expose()`

```ts
app.useGlobalInterceptors(new ClassSerializerInterceptor(app.get(Reflector)));
```

**Also common:** custom `ResponseEnvelopeInterceptor` (adds `requestId`, `data`).

---

## 7) Versioning

**Goal:** evolve APIs without breaking clients.

```ts
// main.ts
app.enableVersioning({
  type: VersioningType.URI, // /v1/users
});
```

Options:

* URI (`/v1`)
* Header (`x-api-version`)
* Media type

**Interview line:** “I use URI versioning for clarity and gateway friendliness.”

---

## 8) Task Scheduling

**Goal:** background jobs (retries, settlements, cleanup).

Use `@nestjs/schedule`:

* `@Cron()`, `@Interval()`, `@Timeout()`

```ts
@Cron('0 0 * * *') // daily midnight
handleDailyReconciliation() {}
```

**BFSI tip:** scheduling should be idempotent + lock-protected (e.g., Redis lock) to avoid double runs.

---

## 9) Queues

**Goal:** async processing, retries, rate smoothing.

Common: Bull/BullMQ + Redis.

* producers in services
* consumers in processors
* retries/backoff/dead-letter patterns

**Interview line:** “Queues decouple user-facing latency from heavy processing and give me retries + DLQ.”

---

## 10) Logging

**Goal:** structured logs + correlation id.

* use JSON logs (pino/winston)
* add requestId via middleware
* log duration via interceptor
* centralize error format via exception filter

**What interviewers like:** “structured logs + requestId + consistent error codes.”

---

## 11) Cookies

**Goal:** cookie-based auth/session, CSRF patterns.

On Express:

* use `cookie-parser`
* set secure flags: `HttpOnly`, `Secure`, `SameSite`

```ts
res.cookie('sid', value, { httpOnly: true, secure: true, sameSite: 'lax' });
```

**Project tip:** if using cookies for auth, add CSRF protection for state-changing routes.

---

## 12) Events

**Goal:** internal decoupling (domain events).

Use EventEmitterModule:

* emit `user.created`
* listeners trigger side effects (email, audit, cache invalidation)

**Best practice:** keep handlers idempotent; for reliability across services use Kafka/RabbitMQ.

---

## 13) Compression

**Goal:** reduce payload size (JSON responses).

* use compression middleware (Express/Fastify equivalent)
* avoid compressing already-compressed content (zip, images)

---

## 14) File Upload

**Goal:** handle multipart uploads (avatars, docs).

Nest uses Multer under the hood:

* `@UseInterceptors(FileInterceptor('file'))`
* validate file size/type
* stream to S3/object storage

**Bank-grade:** scan/validate, store in object storage, never keep on app disk.

---

## 15) Streaming Files

**Goal:** send large files efficiently.

Use `StreamableFile` (no buffering entire file in memory):

* pipe from disk/object storage

---

## 16) HTTP Module (outbound calls)

**Goal:** call other services with timeouts/retries.

Use `HttpModule` from `@nestjs/axios`:

* set baseURL, timeout
* add interceptors for headers (correlation id)
* wrap with resilience (retry/backoff/circuit breaker if required)

---

## 17) Session

**Goal:** server-side session storage (often Redis).

* Express session middleware + Redis store
* set cookie flags properly
* rotate session ids on login

**Interview line:** “For enterprise apps, server-side sessions in Redis reduce token leakage risk and support revocation.”

---

## 18) Model-View-Controller (MVC)

**Goal:** separate concerns:

* Controller = transport (HTTP)
* Service = business logic (use-cases)
* Repository/DAO = data access

**What to say:** “Controllers stay thin; services own domain logic; modules define boundaries.”

---

## 19) Performance (Fastify)

**Goal:** higher throughput, lower overhead.

Use Fastify adapter:

* generally faster than Express for many workloads
* best when you avoid raw `@Req/@Res` coupling

```ts
const app = await NestFactory.create<NestFastifyApplication>(
  AppModule,
  new FastifyAdapter(),
);
```

**Tip:** validate/serialize efficiently; use caching; avoid request-scoped providers unless necessary.

---

## 20) Server-Sent Events (SSE)

**Goal:** one-way real-time updates over HTTP (progress, notifications).

Nest supports `@Sse()`:

* returns an Observable stream
* good for “live status” without full WebSockets

**When to choose SSE**

* streaming updates to browser clients
* simpler than WS for server→client only

---

## Practical “production bundle” most teams adopt

* ConfigModule (global)
* Global ValidationPipe (strict)
* RequestIdMiddleware + LoggingInterceptor
* Global Exception Filter (standard error contract)
* Cache + Queue (Redis-backed)
* Fastify adapter (if performance is a priority)
* Events for internal decoupling, Kafka/RabbitMQ for cross-service
