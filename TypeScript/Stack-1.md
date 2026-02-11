TypeScript = JavaScript + types (with tooling support).It compiles to plain JavaScript, so browsers don’t see TypeScript at all.Because JavaScript at scale is hard to maintain, and TypeScript fixes the biggest pain points.
Types enforce type safety, meaning you can’t accidentally assign a number to a variable meant to hold text.- A type is a way to describe the properties and methods a value has. For example, "Hello" is a string type, which comes with methods like .length or .toUpperCase()
They make code more predictable, easier to debug, and improve developer productivity with autocomplete and error checking.
Primitive types represent basic single-value data types. They are immutable and stored directly in memory.Main Primitive Types:stringnumberbooleannullundefinedsymbolbigint
Why Use Primitive TypesEnsures data correctnessPrevents accidental assignmentHelps IntelliSense & refactoring
TypeScript number supports:IntegerFloatNaNInfinityUnlike some languages, there is no separate int or float
let username: string = "Surya";let age: number = 30;let isPremiumUser: boolean = true;
Why it matters:Angular templates + services are strongly typedNestJS DTOs + validation rely on correct typing--Arrays::Arrays store multiple values of the same type in an ordered collection.
 Two Ways to Define ArraysMethod 1 – Bracket Syntaxlet skills: string[] = ["React", "Node.js", "AWS"];
Method 2 – Generic Syntaxlet skills: Array<string> = ["React", "Node.js"];
 Both are same internally When To Use Each SyntaxUse string[]
 Most common & readableUse Array<T>
 Useful when working with genericsfunction printItems<T>(items: Array<T>) {}

Arrays enforce homogeneous data. Wrong:let arr: string[] = ["hello", 5];
JSON objects, mixed arrays enforces hetrogenious
---Tuples are fixed-length arrays with fixed types at each index.
 Each position has meaning
 Real Example Geo Location Examplelet location: [number, number] = [17.3850, 78.4867];
 API Response Examplelet apiResponse: [string, number] = ["Success", 200];
Tuple vs Array?
 Tuple = fixed structure Array = flexible size---
any disables TypeScript type checking.
 You can assign ANY value
 Real Examplelet data: any = "Hello";data = 10;data = true;
unknown is safer alternative to any.
 You MUST check type before using it.
Never:Represents value that never occurs.Used when:Function never returnsInfinite loopAlways throws error
---
Type InferenceTypeScript automatically detects type based on assigned value.let name = "Surya";  // inferred as stringlet salary = 100000; // inferred as number

Explicit TypingDeveloper manually defines type.
 Examplelet name: string = "Surya";let salary: number = 100000;

DTO (Data Transfer Object) is a structured object used to transfer data between different layers of an application, such as from client to server or between services.
interface Payment {   amount: number;   currency: string;   isSuccessful: boolean;}...interface User {   name: string;   roles: string[];   isActive: boolean;}
const user: User = {   name: "Surya",   roles: ["Admin", "Developer"],   isActive: true};
TypeScript primitive types ensure fundamental data safety. Arrays allow homogeneous collections while tuples enforce fixed positional contracts. any removes type safety whereas unknown enforces runtime validation. never represents unreachable states and is crucial for exhaustive checking. Type inference improves developer productivity, but explicit typing is preferred for public interfaces and enterprise-scale architecture.
--B) Interfaces vs Type AliasesInterface merges but type alieases notAn interface defines the structure (shape) of an object or class.
 Think of it as a contract that objects must follow.
interface User {  name: string;  age: number;}

Type AliasA type alias creates a custom name for ANY type (object, union, primitive, tuple, etc.)
 More flexible than interface.
 Basic Exampletype User = {  name: string;  age: number;};
Where you’ll use it
Angular: component inputs, service responses, DTO modelsNestJS: request/response types, entities, contract types
Interfaces are preferred when defining object contracts and OOP relationships, while type aliases are preferred for advanced type compositions like unions, tuples, and functional types.
I generally use interfaces for defining object contracts and public APIs because they support extension and merging. I prefer type aliases when working with unions, tuples, and advanced type compositions. In large enterprise applications, both are usually used together based on the use case.
---
A union allows a variable to hold multiple possible types.
 Uses | operator
let userId: string | number;
userId = "EMP123";userId = 1001;
Payment Methodtype PaymentMethod = "card" | "upi" | "netbanking";
 Common TrapYou must narrow before using.
function printId(id: string | number) {   if (typeof id === "string") {      console.log(id.toUpperCase());   }}
--
Intersection combines multiple types into one.
 Uses &
 Exampletype Person = {  name: string;};
type Employee = {  employeeId: number;};
type Staff = Person & Employee;
--
Literal types restrict values to specific fixed values.
 Exampletype Direction = "left" | "right" | "top" | "bottom";
 Real React Exampletype ButtonVariant = "primary" | "secondary";
--Generics allow writing reusable and type-safe components/functions.
 Think “dynamic types with safety”.
 Basic Examplefunction identity<T>(value: T): T {   return value;}
Usageidentity<string>("hello");identity<number>(10);
Real Production ExampleAPI Response Wrapperinterface ApiResponse<T> {   data: T;   success: boolean;}
Usagelet response: ApiResponse<User>;
--Classes are blueprints for creating objects with properties and methods.
 Exampleclass User {   name: string;
   constructor(name: string) {      this.name = name;   }
   greet() {      return `Hello ${this.name}`;   }}---
Types public (default)
Accessible everywhere.
class User {   public name: string;}
 private
Accessible only inside class.
class BankAccount {   private balance = 0;}
 protected
Accessible in class + subclasses.
class Animal {   protected age = 5;}
---
OOP Basics Encapsulation
Hide internal data.
 Example → private variables
 Inheritanceclass Admin extends User {}
 Polymorphism
Same method behaves differently.
 Abstraction
Hide complexity via interfaces or abstract classes.
 Enterprise Example
Payment gateway abstraction:
interface PaymentGateway {   pay(amount: number): void;}
--Narrowing & Type Guards Definition
Type narrowing helps TypeScript determine exact type at runtime.
 typeof Guardif(typeof value === "string") {}
 instanceOf Guardif(obj instanceof User) {}
 Custom Type Guardfunction isString(value: unknown): value is string {   return typeof value === "string";}
 Real Production Example
Handling external API response safely.--
Enums Definition
Enums define named constant values.
 Exampleenum Role {   Admin,   User,   Guest}
 Real Production Exampleenum OrderStatus {   Pending,   Shipped,   Delivered}
 Interview Insight
Enums are often replaced by literal unions nowadays because:
 Smaller bundle size Better tree shaking

---
Modules (ESM Import/Export) Definition
Modules split code into reusable files.
 Named Exportexport const PI = 3.14;
 Named Importimport { PI } from "./math";
 Default Exportexport default function add() {}
 Default Importimport add from "./math";--
TypeScript advanced type system enables building scalable and maintainable enterprise applications. Unions and literal types model real-world variability, intersections help in composing domain models, generics provide reusable and strongly typed abstractions, while classes and access modifiers support OOP-based architecture. Narrowing and type guards ensure runtime safety, enums help manage constant states, and ESM modules enable modular scalable code organization.

--Decorators are special annotations that add metadata or behavior to classes, methods, or properties.
 Common in:NestJSAngularValidation frameworksDependency injection
 Example (Validation Decorator)class CreateUserDto {   @IsString()   name: string;
   @IsEmail()   email: string;}

 Here decorators validate incoming data. Real Backend Use CaseIn enterprise APIs, decorators:Validate request payloadsHandle authorizationInject dependenciesTransform data automatically
--DTOs (Data Transfer Objects)DTOs define data shape transferred between layers.
 API → Service → Database
 Real Production Exampleclass CreateOrderDto {   productId: string;   quantity: number;}
 Why Companies Use DTOsStrong API contractsPrevents invalid inputSeparates internal models vs public payloadSecurity (avoid exposing DB fields)

---
Class-Based Validation PatternUsually used with libraries like validation decorators.
Exampleclass RegisterUserDto {   name!: string;   email!: string;   password!: string;}
Production FlowRequest → DTO → Validation → Service → Database--
Class Properties::Properties define data stored inside a class.
Exampleclass User {   name: string;   age: number;}
 3. ConstructorsConstructor initializes object when class is created.
Exampleclass User {   constructor(public name: string) {}}
 Enterprise PatternDependency Injectionclass OrderService {   constructor(private paymentService: PaymentService) {}}
 4. Optional Properties (?:)Property may or may not exist.
Exampleinterface User {   name: string;   phone?: string;}
Real API ExampleOptional update payload:interface UpdateUserDto {   name?: string;   email?: string;}
 5. Readonly PropertiesProperty can only be assigned once.
Exampleclass Order {   readonly orderId: string;
   constructor(id: string) {      this.orderId = id;   }}
 Real Use Case
IDsConfiguration objectsImmutable models
 6. implements vs extends extends
Used for inheritance between classes
class Animal {   move() {}}
class Dog extends Animal {}
 implements
Used when class follows interface contract.
interface Logger {   log(message: string): void;}
class ConsoleLogger implements Logger {   log(message: string) {      console.log(message);   }}
 Interview Insight
 extends = inherits behavior implements = enforces structure

--
What TypeScript Does in Node.js Projects TypeScript Works at Development Time
TypeScript adds:
 Static typing IDE intelligence Safe contracts Better refactoring Scalable architecture
 Node Runs JavaScript at RuntimeFlowsrc/index.ts       ↓TypeScript Compiler (tsc)       ↓dist/index.js       ↓Node executes
--
Common Ways to Run TypeScript in Node Method 1 — Compile & Runtscnode dist/app.js
 Method 2 — ts-node
Runs TS directly.
npx ts-node src/app.ts
 Method 3 — Dev Tools (Popular)tsxnodemon + ts-nodevite-node---Where TypeScript Is Used in Node.js Projects API Models / DTOs
Request & response validation
 Service Layer
Business logic contracts
 Database Models
ORM typings
 Shared Libraries
Reusable utilities
 Microservices Contracts
Kafka / GraphQL payload typing---
Why TypeScript Is Extremely Popular in Node Backends Prevents Runtime Bugs
Huge for BFSI & enterprise apps.
 Improves Team Collaboration
Clear contracts reduce misunderstandings.
 Enables Safer Refactoring
Critical in large microservices.
 Strong Ecosystem Support
Most modern frameworks depend on it.