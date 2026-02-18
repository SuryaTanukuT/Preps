# Decorators in NestJS (clear, practical)

## 1) What are decorators?

**Decorators** are TypeScript features that let you attach **metadata** to:

* classes
* methods
* parameters
* properties

NestJS reads this metadata at runtime to **wire your app automatically**:

* build routes
* inject dependencies (DI)
* run validation
* apply guards/interceptors
* generate OpenAPI docs (if you use Swagger)

Think of decorators as **“configuration next to the code”**.

---

## 2) Why NestJS uses decorators so heavily

Because NestJS is **reflection + metadata driven**.

Decorators let Nest:

* discover controllers/routes without manual routing tables
* understand what dependencies to inject
* know what to validate/transform
* know what auth policies to run

---

## 3) Main decorator types in NestJS (with examples)

## A) Class decorators

### `@Module()`

Defines a module boundary and wiring.

```ts
@Module({
  imports: [UsersModule],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

### `@Controller('users')`

Marks a class as a controller and sets a route prefix.

```ts
@Controller('users')
export class UsersController {}
```

### `@Injectable()`

Marks a class as a provider so it can be injected via DI.

```ts
@Injectable()
export class UsersService {}
```

---

## B) Method decorators (routes + pipeline)

### Route decorators: `@Get`, `@Post`, `@Put`, `@Delete`, `@Patch`

```ts
@Get(':id')
findOne() {}
```

### `@UseGuards()`, `@UseInterceptors()`, `@UsePipes()`

Apply cross-cutting behavior at method (or class) level.

```ts
@UseGuards(JwtAuthGuard)
@UseInterceptors(LoggingInterceptor)
@Post()
create() {}
```

---

## C) Parameter decorators (extract request data)

These pull data from the incoming request.

```ts
@Get(':id')
findOne(
  @Param('id') id: string,
  @Query('expand') expand?: string,
  @Headers('x-request-id') requestId?: string,
) {}
```

Common parameter decorators:

* `@Body()` → request body
* `@Param()` → URL params
* `@Query()` → query string
* `@Headers()` → headers
* `@Req()` / `@Res()` → raw request/response (use sparingly)

---

## D) Custom decorators (your own metadata / request helpers)

### 1) “Extract current user” decorator

```ts
export const CurrentUser = createParamDecorator(
  (_data, ctx: ExecutionContext) => {
    const req = ctx.switchToHttp().getRequest();
    return req.user;
  },
);
```

Usage:

```ts
@Get('me')
getMe(@CurrentUser() user: any) {
  return user;
}
```

### 2) “Roles” decorator (metadata for guards)

```ts
export const Roles = (...roles: string[]) => SetMetadata('roles', roles);
```

Usage:

```ts
@Roles('admin')
@UseGuards(JwtAuthGuard, RolesGuard)
@Get('admin-area')
getAdminData() {}
```

---

## 4) Decorators + DI: how injection works

When you write:

```ts
constructor(private readonly usersService: UsersService) {}
```

Nest uses decorator metadata (`@Injectable`) + module wiring to:

* know `UsersService` is a provider
* know which module provides it
* create it (and its dependencies)
* inject it into the controller

If Nest can’t resolve it, you’ll see the classic:

> “Nest can’t resolve dependencies of X…”

---

## 5) Quick mental model

* **Decorators = metadata**
* **Nest runtime = reads metadata**
* **DI container = builds objects**
* **Request pipeline = executes guards/pipes/interceptors based on metadata**

---

## 6) Common NestJS decorators cheat-sheet

* Structure: `@Module`, `@Controller`, `@Injectable`
* Routes: `@Get`, `@Post`, `@Put`, `@Delete`, `@Patch`
* Request data: `@Body`, `@Param`, `@Query`, `@Headers`, `@Req`
* Pipeline: `@UseGuards`, `@UsePipes`, `@UseInterceptors`
* Custom metadata: `SetMetadata`, `createParamDecorator`

