# `@Module({ imports, providers, controllers, exports })` — Deep, Project-Oriented Explanation

Think of a NestJS module as a **feature container** with **visibility rules**.

A module answers 4 questions:

* **What do I need from others?** → `imports`
* **What do I own/build?** → `providers`
* **What HTTP endpoints do I expose?** → `controllers`
* **What part of my internals can other modules use?** → `exports`

---

## 1) `imports: [...]` — “What I depend on”

### What it does

`imports` tells Nest:

> “This module needs access to providers exported by these other modules.”

If you want to inject a provider from another module, you must:

1. **import the module** that exports it, and
2. that module must **export** the provider.

### Example (realistic)

**AuthModule** needs **UsersService** to validate user credentials.

```ts
@Module({
  imports: [UsersModule],        // I depend on UsersModule
  providers: [AuthService],
  controllers: [AuthController],
})
export class AuthModule {}
```

In `AuthService`:

```ts
constructor(private readonly usersService: UsersService) {}
```

This works only if:

* `UsersModule` exports `UsersService`

### Common mistake

“I declared UsersService in UsersModule but forgot to export it.”
→ Nest can’t resolve dependency error.

---

## 2) `providers: [...]` — “What I create/manage in DI”

### What it does

`providers` are the **DI-managed building blocks** of your module.

Nest will:

* instantiate them (default: singleton)
* resolve their dependencies
* inject them wherever needed

### What belongs in providers (project mapping)

Usually includes:

* **Services** (business/use-cases): `UsersService`
* **Repositories/DAOs**: `UsersRepository`
* **Use-cases / domain services** (if you structure like Clean Architecture)
* **Guards / Interceptors / Pipes / Filters** (feature-scoped)
* **Factories / adapters** (e.g., payment gateway adapter)
* **Tokens/providers** for external clients:

  * DB connection provider
  * Redis client provider
  * HTTP client provider

### Example

```ts
@Module({
  providers: [
    UsersService,
    UsersRepository,
    UsersQueryService,
  ],
})
export class UsersModule {}
```

### Key point

A class must be in `providers` (or globally available) to be injectable.

---

## 3) `controllers: [...]` — “What I expose via HTTP”

### What it does

Controllers are your **transport layer** (REST/HTTP endpoints).

* They define routes (`@Get`, `@Post`, etc.)
* They map request input → DTO/service call
* They return a response

### Project mental model

Controllers should be **thin**:

* validate inputs (DTO + pipes)
* call service
* return result

### Example

```ts
@Module({
  controllers: [UsersController],
  providers: [UsersService],
})
export class UsersModule {}
```

If no controller is listed, the module may be:

* internal-only (e.g., `DatabaseModule`, `CommonModule`)
* used by other modules via exported providers

---

## 4) `exports: [...]` — “What I allow others to use”

### What it does

Exports define the **public API** of your module.

Only exported providers can be injected into other modules that import this module.

### The visibility rule (very important)

* A provider listed in `providers` is **private** to the module by default.
* It becomes **public** only if listed in `exports`.

### Example: exporting a service

```ts
@Module({
  providers: [UsersService, UsersRepository],
  exports: [UsersService], // others can inject UsersService
})
export class UsersModule {}
```

Now `AuthModule` can do:

```ts
imports: [UsersModule]
constructor(private usersService: UsersService) {}
```

### Why you should NOT export everything

If you export everything:

* every module can directly use your internals
* coupling increases
* refactoring becomes risky

Best practice:

> Export only what other modules truly need (usually service interfaces or a small set of providers).

---

# Putting it together: a “real app” example

### UsersModule (owns user domain)

```ts
@Module({
  imports: [],                        // users is core; no deps
  controllers: [UsersController],     // exposes /users APIs
  providers: [UsersService, UsersRepository],
  exports: [UsersService],            // allow other modules to use user operations
})
export class UsersModule {}
```

### AuthModule (depends on Users + Jwt)

```ts
@Module({
  imports: [UsersModule],             // needs UsersService
  controllers: [AuthController],
  providers: [AuthService, JwtStrategy],
  exports: [AuthService],             // optionally export if other modules need auth ops
})
export class AuthModule {}
```

---

# Interview-ready summary (copy/paste)

* **imports**: “Which other modules’ exported providers I need.”
* **providers**: “The DI-managed classes I own: services/repos/guards/etc.”
* **controllers**: “HTTP entry points I expose for this feature.”
* **exports**: “My module’s public API: providers other modules are allowed to inject.”

---
