https://www.typescriptlang.org/docs/

# TypeScript (High-Level Interview + Practical Notes)

TypeScript = JavaScript + types (with tooling support).
It compiles to plain JavaScript, so browsers don’t see TypeScript at all.

Because JavaScript at scale is hard to maintain, TypeScript fixes the biggest pain points.

Types enforce type safety, meaning you can’t accidentally assign a number to a variable meant to hold text.

A type is a way to describe the properties and methods a value has.
For example, `"Hello"` is a string type, which comes with methods like `.length` or `.toUpperCase()`.

They make code more predictable, easier to debug, and improve developer productivity with autocomplete and error checking.

---

# Primitive Types

Primitive types represent basic single-value data types.
They are immutable and stored directly in memory.

Main Primitive Types:

* `string`
* `number`
* `boolean`
* `null`
* `undefined`
* `symbol`
* `bigint`

Why Use Primitive Types:

* Ensures data correctness
* Prevents accidental assignment
* Helps IntelliSense & refactoring

TypeScript `number` supports:

* Integer
* Float
* NaN
* Infinity

Unlike some languages, there is no separate `int` or `float`.

```ts
let username: string = "Surya";
let age: number = 30;
let isPremiumUser: boolean = true;
```

Why it matters:

* Angular templates + services are strongly typed
* NestJS DTOs + validation rely on correct typing

---

# Arrays

Arrays store multiple values of the same type in an ordered collection.

## Two Ways to Define Arrays

### Method 1 – Bracket Syntax

```ts
let skills: string[] = ["React", "Node.js", "AWS"];
```

### Method 2 – Generic Syntax

```ts
let skills: Array<string> = ["React", "Node.js"];
```

Both are same internally.

## When To Use Each Syntax

Use `string[]`

* Most common & readable

Use `Array<T>`

* Useful when working with generics

```ts
function printItems<T>(items: Array<T>) {}
```

Arrays enforce homogeneous data.

Wrong:

```ts
let arr: string[] = ["hello", 5];
```

JSON objects or mixed arrays can enforce heterogeneous structures when typed properly.

---

# Tuples

Tuples are fixed-length arrays with fixed types at each index.
Each position has meaning.

## Real Example

Geo Location Example:

```ts
let location: [number, number] = [17.3850, 78.4867];
```

API Response Example:

```ts
let apiResponse: [string, number] = ["Success", 200];
```

Tuple vs Array:

* Tuple = fixed structure
* Array = flexible size

---

# any vs unknown vs never

## any

Disables TypeScript type checking.
You can assign ANY value.

```ts
let data: any = "Hello";
data = 10;
data = true;
```

## unknown

Safer alternative to `any`.
You MUST check type before using it.

## never

Represents value that never occurs.

Used when:

* Function never returns
* Infinite loop
* Always throws error

---

# Type Inference vs Explicit Typing

## Type Inference

TypeScript automatically detects type based on assigned value.

```ts
let name = "Surya";   // inferred as string
let salary = 100000;  // inferred as number
```

## Explicit Typing

Developer manually defines type.

```ts
let name: string = "Surya";
let salary: number = 100000;
```

Type inference improves developer productivity, but explicit typing is preferred for public interfaces and enterprise-scale architecture.

---

# DTO (Data Transfer Object)

DTO is a structured object used to transfer data between different layers of an application.

Client → Server → Service → Database

```ts
interface Payment {
  amount: number;
  currency: string;
  isSuccessful: boolean;
}

interface User {
  name: string;
  roles: string[];
  isActive: boolean;
}

const user: User = {
  name: "Surya",
  roles: ["Admin", "Developer"],
  isActive: true
};
```

TypeScript primitive types ensure fundamental data safety. Arrays allow homogeneous collections while tuples enforce fixed positional contracts. `any` removes type safety whereas `unknown` enforces runtime validation. `never` represents unreachable states and is crucial for exhaustive checking.

---

# Interfaces vs Type Aliases

## Interface

An interface defines the structure (shape) of an object or class.
Think of it as a contract that objects must follow.

```ts
interface User {
  name: string;
  age: number;
}
```

Interfaces support declaration merging.

## Type Alias

A type alias creates a custom name for ANY type (object, union, primitive, tuple, etc.)

```ts
type User = {
  name: string;
  age: number;
};
```

More flexible than interface.

Where you’ll use it:

* Angular: component inputs, service responses, DTO models
* NestJS: request/response types, entities, contract types

Interfaces are preferred when defining object contracts and OOP relationships, while type aliases are preferred for advanced type compositions like unions, tuples, and functional types.

I generally use interfaces for defining object contracts and public APIs because they support extension and merging. I prefer type aliases when working with unions, tuples, and advanced type compositions. In large enterprise applications, both are usually used together based on the use case.

---

# Union Types

A union allows a variable to hold multiple possible types.

Uses `|` operator.

```ts
let userId: string | number;

userId = "EMP123";
userId = 1001;
```

Payment Method Example:

```ts
type PaymentMethod = "card" | "upi" | "netbanking";
```

Common Trap: You must narrow before using.

```ts
function printId(id: string | number) {
  if (typeof id === "string") {
    console.log(id.toUpperCase());
  }
}
```

---

# Intersection Types

Intersection combines multiple types into one.

Uses `&`.

```ts
type Person = { name: string };
type Employee = { employeeId: number };

type Staff = Person & Employee;
```

---

# Literal Types

Literal types restrict values to specific fixed values.

```ts
type Direction = "left" | "right" | "top" | "bottom";
```

React Example:

```ts
type ButtonVariant = "primary" | "secondary";
```

---

# Generics

Generics allow writing reusable and type-safe components/functions.
Think “dynamic types with safety”.

Basic Example:

```ts
function identity<T>(value: T): T {
  return value;
}

identity<string>("hello");
identity<number>(10);
```

Real Production Example:

```ts
interface ApiResponse<T> {
  data: T;
  success: boolean;
}

let response: ApiResponse<User>;
```

---

# Classes

Classes are blueprints for creating objects with properties and methods.

```ts
class User {
  name: string;

  constructor(name: string) {
    this.name = name;
  }

  greet() {
    return `Hello ${this.name}`;
  }
}
```

---

# Access Modifiers

## public (default)

Accessible everywhere.

```ts
class User {
  public name: string;
}
```

## private

Accessible only inside class.

```ts
class BankAccount {
  private balance = 0;
}
```

## protected

Accessible in class + subclasses.

```ts
class Animal {
  protected age = 5;
}
```

---

# OOP Basics

Encapsulation
Hide internal data (private variables).

Inheritance

```ts
class Admin extends User {}
```

Polymorphism
Same method behaves differently.

Abstraction
Hide complexity via interfaces or abstract classes.

Enterprise Example:

```ts
interface PaymentGateway {
  pay(amount: number): void;
}
```

---

# Narrowing & Type Guards

Type narrowing helps TypeScript determine exact type at runtime.

## typeof Guard

```ts
if (typeof value === "string") {}
```

## instanceof Guard

```ts
if (obj instanceof User) {}
```

## Custom Type Guard

```ts
function isString(value: unknown): value is string {
  return typeof value === "string";
}
```

Real Production Example: Handling external API response safely.

---

# Enums

Enums define named constant values.

```ts
enum Role {
  Admin,
  User,
  Guest
}
```

Real Production Example:

```ts
enum OrderStatus {
  Pending,
  Shipped,
  Delivered
}
```

Interview Insight: Enums are often replaced by literal unions nowadays because:

* Smaller bundle size
* Better tree shaking

---

# Modules (ESM Import/Export)

Modules split code into reusable files.

Named Export:

```ts
export const PI = 3.14;
```

Named Import:

```ts
import { PI } from "./math";
```

Default Export:

```ts
export default function add() {}
```

Default Import:

```ts
import add from "./math";
```

---

# Decorators

Decorators are special annotations that add metadata or behavior to classes, methods, or properties.

Common in:

* NestJS
* Angular
* Validation frameworks
* Dependency injection

Example:

```ts
class CreateUserDto {
  @IsString()
  name: string;

  @IsEmail()
  email: string;
}
```

Enterprise Use Case:

* Validate request payloads
* Handle authorization
* Inject dependencies
* Transform data automatically

---

# DTOs in Backend

DTOs define data shape transferred between layers.

API → Service → Database

```ts
class CreateOrderDto {
  productId: string;
  quantity: number;
}
```

Why Companies Use DTOs:

* Strong API contracts
* Prevents invalid input
* Separates internal models vs public payload
* Security (avoid exposing DB fields)

---

# Class Patterns

## Class Properties

```ts
class User {
  name: string;
  age: number;
}
```

## Constructors

```ts
class User {
  constructor(public name: string) {}
}
```

Enterprise Pattern (Dependency Injection):

```ts
class OrderService {
  constructor(private paymentService: PaymentService) {}
}
```

---

## Optional Properties (?:)

```ts
interface User {
  name: string;
  phone?: string;
}
```

Real API Example:

```ts
interface UpdateUserDto {
  name?: string;
  email?: string;
}
```

---

## Readonly Properties

```ts
class Order {
  readonly orderId: string;

  constructor(id: string) {
    this.orderId = id;
  }
}
```

Real Use Case:

* IDs
* Configuration objects
* Immutable models

---

## implements vs extends

### extends

Used for inheritance between classes.

```ts
class Animal {
  move() {}
}

class Dog extends Animal {}
```

### implements

Used when class follows interface contract.

```ts
interface Logger {
  log(message: string): void;
}

class ConsoleLogger implements Logger {
  log(message: string) {
    console.log(message);
  }
}
```

Interview Insight:

* extends = inherits behavior
* implements = enforces structure

---

# TypeScript in Node.js Projects

TypeScript works at development time.

It adds:

* Static typing
* IDE intelligence
* Safe contracts
* Better refactoring
* Scalable architecture

Node runs JavaScript at runtime.

Flow:

```
src/index.ts
   ↓
TypeScript Compiler (tsc)
   ↓
dist/index.js
   ↓
Node executes
```

---

# Ways to Run TypeScript in Node

## Method 1 — Compile & Run

```
tsc
node dist/app.js
```

## Method 2 — ts-node

```
npx ts-node src/app.ts
```

## Method 3 — Dev Tools

* tsx
* nodemon + ts-node
* vite-node

---

# Where TypeScript Is Used in Node Projects

* API Models / DTOs
* Request & response validation
* Service layer contracts
* Database models (ORM typings)
* Shared libraries
* Microservices contracts (Kafka / GraphQL payload typing)

---

# Why TypeScript Is Extremely Popular in Node Backends

* Prevents runtime bugs (critical for BFSI & enterprise apps)
* Improves team collaboration (clear contracts)
* Enables safer refactoring (important in microservices)
* Strong ecosystem support (modern frameworks depend on it)

---

TypeScript advanced type system enables building scalable and maintainable enterprise applications. Unions and literal types model real-world variability, intersections help compose domain models, generics provide reusable and strongly typed abstractions, classes and access modifiers support OOP-based architecture, narrowing ensures runtime safety, enums manage constant states, and ESM modules enable modular scalable code organization.
