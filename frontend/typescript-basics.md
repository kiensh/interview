# TypeScript Basics Interview Questions

## Table of Contents

### Basic Types
- [Q1: What is TypeScript and why use it?](#q1)
- [Q2: What are the basic types in TypeScript?](#q2)
- [Q3: What is the difference between `any`, `unknown`, and `never`?](#q3)
- [Q4: What are union and intersection types?](#q4)

### Interfaces & Types
- [Q5: What is the difference between interface and type?](#q5)
- [Q6: How do you define optional and readonly properties?](#q6)
- [Q7: What are index signatures?](#q7)

### Generics
- [Q8: What are generics in TypeScript?](#q8)
- [Q9: How do you use generic constraints?](#q9)
- [Q10: What are common generic utility types?](#q10)

### Advanced Types
- [Q11: What are type guards?](#q11)
- [Q12: What are mapped types?](#q12)
- [Q13: What are conditional types?](#q13)
- [Q14: What is declaration merging?](#q14)
- [Q15: How do you handle modules and namespaces?](#q15)

---

## Basic Types

<a id="q1"></a>
### Q1: What is TypeScript and why use it?
**Answer:**

TypeScript is a typed superset of JavaScript that compiles to plain JavaScript.

**Benefits:**
| Benefit | Description |
|---------|-------------|
| Static typing | Catch errors at compile time |
| Better IDE support | Autocomplete, refactoring |
| Self-documenting | Types serve as documentation |
| Safer refactoring | Compiler catches breaking changes |
| Modern features | ES6+ features with backwards compatibility |

```typescript
// JavaScript - no type checking
function add(a, b) {
    return a + b;
}
add(5, "10"); // "510" - no error, unexpected result

// TypeScript - type checking
function add(a: number, b: number): number {
    return a + b;
}
add(5, "10"); // Error: Argument of type 'string' is not assignable to parameter of type 'number'

// Type annotations
let name: string = "John";
let age: number = 30;
let isActive: boolean = true;

// Type inference
let name = "John"; // TypeScript infers string type
name = 42; // Error: Type 'number' is not assignable to type 'string'

// Configuration (tsconfig.json)
{
    "compilerOptions": {
        "target": "ES2020",
        "module": "commonjs",
        "strict": true,
        "outDir": "./dist",
        "rootDir": "./src",
        "esModuleInterop": true
    },
    "include": ["src/**/*"],
    "exclude": ["node_modules"]
}
```

<a id="q2"></a>
### Q2: What are the basic types in TypeScript?
**Answer:**

```typescript
// Primitive types
let str: string = "hello";
let num: number = 42;
let big: bigint = 100n;
let bool: boolean = true;
let sym: symbol = Symbol("id");
let undef: undefined = undefined;
let nul: null = null;

// Arrays
let numbers: number[] = [1, 2, 3];
let strings: Array<string> = ["a", "b", "c"];
let mixed: (string | number)[] = [1, "two", 3];

// Tuple - fixed length, known types
let tuple: [string, number] = ["hello", 42];
let named: [name: string, age: number] = ["John", 30];

// Optional tuple elements
let optionalTuple: [string, number?] = ["hello"];

// Rest elements in tuple
let restTuple: [string, ...number[]] = ["hello", 1, 2, 3];

// Object
let obj: { name: string; age: number } = { name: "John", age: 30 };

// Function
let fn: (a: number, b: number) => number = (a, b) => a + b;

// void - no return value
function log(message: string): void {
    console.log(message);
}

// Object type (non-primitive)
let obj: object = { key: "value" };
let obj2: object = [1, 2, 3]; // Arrays are objects

// Literal types
let direction: "north" | "south" | "east" | "west" = "north";
let count: 1 | 2 | 3 = 2;
let success: true = true;

// Enum
enum Color {
    Red,    // 0
    Green,  // 1
    Blue    // 2
}

enum Status {
    Active = "ACTIVE",
    Inactive = "INACTIVE"
}

const enum Direction {
    Up,
    Down
}
// const enum is inlined at compile time

// Type assertions
let value: unknown = "hello";
let length: number = (value as string).length;
let length2: number = (<string>value).length; // Alternative syntax

// Non-null assertion
let element = document.getElementById("app")!; // Assert not null
```

<a id="q3"></a>
### Q3: What is the difference between `any`, `unknown`, and `never`?
**Answer:**

| Type | Description | Type checking | Use case |
|------|-------------|---------------|----------|
| `any` | Disables type checking | None | Migration, quick fixes |
| `unknown` | Type-safe any | Must narrow before use | External data |
| `never` | No possible value | Impossible states | Exhaustive checks |

```typescript
// any - opt out of type checking (avoid when possible)
let anything: any = "hello";
anything = 42;
anything = { key: "value" };
anything.nonExistent.method(); // No error - dangerous!

// unknown - safe alternative to any
let unknown: unknown = "hello";
unknown = 42;
unknown = { key: "value" };
// unknown.toUpperCase(); // Error: Object is of type 'unknown'

// Must narrow the type first
if (typeof unknown === "string") {
    console.log(unknown.toUpperCase()); // OK
}

// Type guard for unknown
function processValue(value: unknown): string {
    if (typeof value === "string") {
        return value.toUpperCase();
    }
    if (typeof value === "number") {
        return value.toString();
    }
    if (value instanceof Date) {
        return value.toISOString();
    }
    return String(value);
}

// never - represents impossible values
function throwError(message: string): never {
    throw new Error(message);
}

function infiniteLoop(): never {
    while (true) {}
}

// Exhaustive checking with never
type Shape = "circle" | "square" | "triangle";

function getArea(shape: Shape): number {
    switch (shape) {
        case "circle":
            return Math.PI * 1;
        case "square":
            return 1;
        case "triangle":
            return 0.5;
        default:
            // If we add a new shape, TypeScript will error here
            const _exhaustive: never = shape;
            throw new Error(`Unknown shape: ${_exhaustive}`);
    }
}

// never in conditional types
type NonNullable<T> = T extends null | undefined ? never : T;

type A = NonNullable<string | null>; // string

// Difference summary:
// any: I don't care about types (unsafe)
// unknown: I don't know the type yet (safe)
// never: This should never happen (impossible)
```

<a id="q4"></a>
### Q4: What are union and intersection types?
**Answer:**

```typescript
// Union types (OR) - value can be one of several types
type StringOrNumber = string | number;

let value: StringOrNumber = "hello";
value = 42; // OK

function format(input: string | number): string {
    if (typeof input === "string") {
        return input.toUpperCase();
    }
    return input.toFixed(2);
}

// Discriminated unions (tagged unions)
interface Circle {
    kind: "circle";
    radius: number;
}

interface Rectangle {
    kind: "rectangle";
    width: number;
    height: number;
}

type Shape = Circle | Rectangle;

function getArea(shape: Shape): number {
    switch (shape.kind) {
        case "circle":
            return Math.PI * shape.radius ** 2;
        case "rectangle":
            return shape.width * shape.height;
    }
}

// Intersection types (AND) - value must satisfy all types
interface HasName {
    name: string;
}

interface HasAge {
    age: number;
}

type Person = HasName & HasAge;

const person: Person = {
    name: "John",
    age: 30
}; // Must have both name and age

// Combining interfaces
interface Printable {
    print(): void;
}

interface Loggable {
    log(): void;
}

type PrintableAndLoggable = Printable & Loggable;

class Document implements PrintableAndLoggable {
    print() { console.log("Printing..."); }
    log() { console.log("Logging..."); }
}

// Intersection with conflicting properties
interface A {
    value: string;
}

interface B {
    value: number;
}

type AB = A & B; // value is never (string & number = never)

// Practical use: Mixin pattern
type Constructor<T = {}> = new (...args: any[]) => T;

function Timestamped<TBase extends Constructor>(Base: TBase) {
    return class extends Base {
        timestamp = Date.now();
    };
}

function Tagged<TBase extends Constructor>(Base: TBase) {
    return class extends Base {
        tag = "default";
    };
}

class User {
    name = "";
}

const TimestampedUser = Timestamped(User);
const TaggedTimestampedUser = Tagged(Timestamped(User));

const user = new TaggedTimestampedUser();
console.log(user.name, user.timestamp, user.tag);
```

---

## Interfaces & Types

<a id="q5"></a>
### Q5: What is the difference between interface and type?
**Answer:**

| Feature | Interface | Type Alias |
|---------|-----------|------------|
| Declaration | `interface Name {}` | `type Name = {}` |
| Extending | `extends` keyword | `&` intersection |
| Merging | Yes (declaration merging) | No |
| Primitives/Unions | No | Yes |
| Computed properties | No | Yes |
| Use case | Objects, classes | Everything |

```typescript
// Interface - for object shapes
interface User {
    name: string;
    age: number;
}

// Type alias - for any type
type UserId = string | number;
type User = {
    name: string;
    age: number;
};

// Both work for objects
interface Point {
    x: number;
    y: number;
}

type PointType = {
    x: number;
    y: number;
};

// Extending
interface Animal {
    name: string;
}

interface Dog extends Animal {
    breed: string;
}

type AnimalType = {
    name: string;
};

type DogType = AnimalType & {
    breed: string;
};

// Declaration merging (only interface)
interface Window {
    customProperty: string;
}

interface Window {
    anotherProperty: number;
}
// Window now has both properties

// Type cannot be re-declared
type MyType = { a: string };
// type MyType = { b: number }; // Error: Duplicate identifier

// Types can do things interfaces cannot

// Union types
type Status = "pending" | "approved" | "rejected";

// Primitives
type ID = string;

// Tuples
type Pair = [string, number];

// Computed properties
type Keys = "name" | "age";
type Person = {
    [K in Keys]: string;
};
// { name: string; age: string }

// Conditional types
type NonNull<T> = T extends null | undefined ? never : T;

// Interface with callable signature
interface Callable {
    (arg: string): void;
    property: number;
}

// Same with type
type CallableType = {
    (arg: string): void;
    property: number;
};

// Recommendation:
// - Use interface for public API and library definitions
// - Use type for complex types, unions, intersections
// - Be consistent within your codebase
```

<a id="q6"></a>
### Q6: How do you define optional and readonly properties?
**Answer:**

```typescript
// Optional properties (?)
interface User {
    id: number;
    name: string;
    email?: string; // Optional
    phone?: string; // Optional
}

const user1: User = { id: 1, name: "John" }; // OK
const user2: User = { id: 2, name: "Jane", email: "jane@example.com" }; // OK

// Accessing optional properties
function getEmail(user: User): string {
    return user.email ?? "No email"; // Nullish coalescing
    // Or: user.email || "No email"
    // Or: user.email ? user.email : "No email"
}

// Readonly properties
interface Config {
    readonly apiKey: string;
    readonly baseUrl: string;
    timeout: number;
}

const config: Config = {
    apiKey: "secret",
    baseUrl: "https://api.example.com",
    timeout: 5000
};

// config.apiKey = "new-key"; // Error: Cannot assign to 'apiKey' because it is a read-only property
config.timeout = 10000; // OK

// Readonly array
const numbers: readonly number[] = [1, 2, 3];
// numbers.push(4); // Error: Property 'push' does not exist on type 'readonly number[]'

const numbers2: ReadonlyArray<number> = [1, 2, 3];

// Readonly with generics
interface ReadonlyUser {
    readonly name: string;
    readonly age: number;
}

// Or using utility type
type ReadonlyUser2 = Readonly<User>;

// Const assertion (deep readonly)
const user = {
    name: "John",
    address: {
        city: "NYC"
    }
} as const;

// user.name = "Jane"; // Error
// user.address.city = "LA"; // Error

// Optional function parameters
function greet(name: string, greeting?: string): string {
    return `${greeting ?? "Hello"}, ${name}!`;
}

greet("John"); // "Hello, John!"
greet("John", "Hi"); // "Hi, John!"

// Default parameters
function greet(name: string, greeting: string = "Hello"): string {
    return `${greeting}, ${name}!`;
}

// Optional with undefined vs missing
interface Strict {
    required: string;
    optional?: string; // Can be missing or undefined
}

interface ExplicitUndefined {
    required: string;
    maybeUndefined: string | undefined; // Must be present, but can be undefined
}

const strict: Strict = { required: "a" }; // OK
// const explicit: ExplicitUndefined = { required: "a" }; // Error: missing maybeUndefined
const explicit: ExplicitUndefined = { required: "a", maybeUndefined: undefined }; // OK
```

<a id="q7"></a>
### Q7: What are index signatures?
**Answer:**

Index signatures define the types for dynamic property names.

```typescript
// Index signature for string keys
interface StringMap {
    [key: string]: string;
}

const translations: StringMap = {
    hello: "Xin chào",
    goodbye: "Tạm biệt"
};

translations.newKey = "value"; // OK
// translations.number = 42; // Error: Type 'number' is not assignable to type 'string'

// Index signature for number keys
interface NumberArray {
    [index: number]: string;
}

const arr: NumberArray = ["a", "b", "c"];
const first = arr[0]; // string

// Mixed properties with index signature
interface Dictionary {
    [key: string]: number;
    length: number; // OK - compatible with index signature
    // name: string; // Error - not compatible
}

// Using template literal types
interface EventHandlers {
    [key: `on${string}`]: (event: Event) => void;
}

const handlers: EventHandlers = {
    onClick: (e) => console.log(e),
    onHover: (e) => console.log(e)
    // invalid: () => {} // Error: doesn't match 'on${string}'
};

// Record utility type (alternative to index signature)
type StringRecord = Record<string, string>;

// More specific keys
type Status = "pending" | "approved" | "rejected";
type StatusMessages = Record<Status, string>;

const messages: StatusMessages = {
    pending: "Waiting...",
    approved: "Done!",
    rejected: "Failed"
};

// Index signature with optional values
interface Cache {
    [key: string]: string | undefined;
}

const cache: Cache = {};
const value = cache.someKey; // string | undefined

// Readonly index signature
interface ReadonlyDict {
    readonly [key: string]: string;
}

const dict: ReadonlyDict = { a: "1" };
// dict.a = "2"; // Error: Index signature in type 'ReadonlyDict' only permits reading

// Index signature with symbol
interface SymbolMap {
    [key: symbol]: string;
}

const sym = Symbol("key");
const symbolMap: SymbolMap = {
    [sym]: "value"
};

// Combining string and number index signatures
interface Mixed {
    [key: string]: string | number;
    [index: number]: string; // Must be subtype of string index
}
```

---

## Generics

<a id="q8"></a>
### Q8: What are generics in TypeScript?
**Answer:**

Generics allow creating reusable components that work with multiple types.

```typescript
// Without generics - lose type information
function identity(arg: any): any {
    return arg;
}

// With generics - preserve type information
function identity<T>(arg: T): T {
    return arg;
}

const str = identity<string>("hello"); // str is string
const num = identity(42); // Type inference: num is number

// Generic functions
function firstElement<T>(arr: T[]): T | undefined {
    return arr[0];
}

const first = firstElement([1, 2, 3]); // number | undefined
const firstStr = firstElement(["a", "b"]); // string | undefined

// Multiple type parameters
function pair<T, U>(first: T, second: U): [T, U] {
    return [first, second];
}

const p = pair("hello", 42); // [string, number]

// Generic interfaces
interface Box<T> {
    value: T;
}

const stringBox: Box<string> = { value: "hello" };
const numberBox: Box<number> = { value: 42 };

// Generic with default type
interface Container<T = string> {
    value: T;
}

const container: Container = { value: "hello" }; // T is string by default

// Generic classes
class Stack<T> {
    private items: T[] = [];
    
    push(item: T): void {
        this.items.push(item);
    }
    
    pop(): T | undefined {
        return this.items.pop();
    }
    
    peek(): T | undefined {
        return this.items[this.items.length - 1];
    }
}

const numberStack = new Stack<number>();
numberStack.push(1);
numberStack.push(2);
numberStack.pop(); // number | undefined

// Generic type aliases
type Result<T> = {
    success: boolean;
    data: T;
    error?: string;
};

const result: Result<User> = {
    success: true,
    data: { name: "John", age: 30 }
};

// Generic arrow functions
const identity2 = <T>(arg: T): T => arg;
const identity3 = <T,>(arg: T): T => arg; // In JSX files, use trailing comma
```

<a id="q9"></a>
### Q9: How do you use generic constraints?
**Answer:**

Generic constraints limit the types that can be used with generics.

```typescript
// Basic constraint with extends
function getLength<T extends { length: number }>(arg: T): number {
    return arg.length;
}

getLength("hello"); // OK - string has length
getLength([1, 2, 3]); // OK - array has length
getLength({ length: 10 }); // OK - object with length
// getLength(123); // Error - number doesn't have length

// Constraint with interface
interface Printable {
    print(): void;
}

function printItem<T extends Printable>(item: T): void {
    item.print();
}

// Constraint with keyof
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
    return obj[key];
}

const user = { name: "John", age: 30 };
const name = getProperty(user, "name"); // string
const age = getProperty(user, "age"); // number
// getProperty(user, "invalid"); // Error: "invalid" is not assignable to "name" | "age"

// Multiple constraints
interface HasId {
    id: number;
}

interface HasName {
    name: string;
}

function process<T extends HasId & HasName>(item: T): string {
    return `${item.id}: ${item.name}`;
}

// Constraining one type parameter by another
function assign<T extends U, U>(target: T, source: U): T {
    return Object.assign(target, source);
}

// Using class type in constraints
function create<T>(ctor: new () => T): T {
    return new ctor();
}

class User {
    name = "John";
}

const user = create(User); // User

// Constructor with parameters
function createWithParams<T>(
    ctor: new (...args: any[]) => T,
    ...args: any[]
): T {
    return new ctor(...args);
}

// Conditional type constraints
type ExtractString<T> = T extends string ? T : never;

type A = ExtractString<string | number>; // string
type B = ExtractString<boolean>; // never

// Using infer with constraints
type ReturnType<T extends (...args: any) => any> = 
    T extends (...args: any) => infer R ? R : any;

function greet(): string {
    return "hello";
}

type GreetReturn = ReturnType<typeof greet>; // string
```

<a id="q10"></a>
### Q10: What are common generic utility types?
**Answer:**

TypeScript provides built-in utility types for common type transformations.

```typescript
interface User {
    id: number;
    name: string;
    email: string;
    age?: number;
}

// Partial<T> - all properties optional
type PartialUser = Partial<User>;
// { id?: number; name?: string; email?: string; age?: number }

const update: Partial<User> = { name: "Jane" }; // OK

// Required<T> - all properties required
type RequiredUser = Required<User>;
// { id: number; name: string; email: string; age: number }

// Readonly<T> - all properties readonly
type ReadonlyUser = Readonly<User>;
const user: ReadonlyUser = { id: 1, name: "John", email: "j@example.com" };
// user.name = "Jane"; // Error

// Pick<T, K> - select specific properties
type UserPreview = Pick<User, "id" | "name">;
// { id: number; name: string }

// Omit<T, K> - exclude specific properties
type UserWithoutEmail = Omit<User, "email">;
// { id: number; name: string; age?: number }

// Record<K, T> - create object type with keys K and values T
type UserRoles = Record<string, string[]>;
// { [key: string]: string[] }

type StatusMap = Record<"pending" | "approved", boolean>;
// { pending: boolean; approved: boolean }

// Exclude<T, U> - exclude types from union
type Status = "pending" | "approved" | "rejected";
type ActiveStatus = Exclude<Status, "rejected">;
// "pending" | "approved"

// Extract<T, U> - extract matching types from union
type StringTypes = Extract<string | number | boolean, string | boolean>;
// string | boolean

// NonNullable<T> - remove null and undefined
type MaybeString = string | null | undefined;
type DefiniteString = NonNullable<MaybeString>;
// string

// ReturnType<T> - get return type of function
function getUser(): User {
    return { id: 1, name: "John", email: "j@example.com" };
}

type UserReturn = ReturnType<typeof getUser>;
// User

// Parameters<T> - get parameter types as tuple
function createUser(name: string, age: number): User {
    return { id: 1, name, email: "", age };
}

type CreateUserParams = Parameters<typeof createUser>;
// [string, number]

// Awaited<T> - unwrap Promise type
type PromiseUser = Promise<User>;
type ResolvedUser = Awaited<PromiseUser>;
// User

// InstanceType<T> - get instance type of constructor
class UserClass {
    name: string = "";
}

type UserInstance = InstanceType<typeof UserClass>;
// UserClass

// ConstructorParameters<T>
type UserCtorParams = ConstructorParameters<typeof UserClass>;
// []

// Combinations
type UpdateUserDto = Partial<Omit<User, "id">>;
// { name?: string; email?: string; age?: number }
```

---

## Advanced Types

<a id="q11"></a>
### Q11: What are type guards?
**Answer:**

Type guards narrow down types within conditional blocks.

```typescript
// typeof type guard
function process(value: string | number): string {
    if (typeof value === "string") {
        return value.toUpperCase(); // value is string
    }
    return value.toFixed(2); // value is number
}

// instanceof type guard
class Dog {
    bark() { console.log("Woof!"); }
}

class Cat {
    meow() { console.log("Meow!"); }
}

function makeSound(animal: Dog | Cat): void {
    if (animal instanceof Dog) {
        animal.bark(); // animal is Dog
    } else {
        animal.meow(); // animal is Cat
    }
}

// in operator type guard
interface Fish {
    swim: () => void;
}

interface Bird {
    fly: () => void;
}

function move(animal: Fish | Bird): void {
    if ("swim" in animal) {
        animal.swim(); // animal is Fish
    } else {
        animal.fly(); // animal is Bird
    }
}

// Custom type guard (type predicate)
function isString(value: unknown): value is string {
    return typeof value === "string";
}

function process(value: unknown): void {
    if (isString(value)) {
        console.log(value.toUpperCase()); // value is string
    }
}

// Custom type guard for interface
interface User {
    type: "user";
    name: string;
}

interface Admin {
    type: "admin";
    name: string;
    permissions: string[];
}

function isAdmin(person: User | Admin): person is Admin {
    return person.type === "admin";
}

function greet(person: User | Admin): void {
    if (isAdmin(person)) {
        console.log(`Admin ${person.name} with ${person.permissions.length} permissions`);
    } else {
        console.log(`User ${person.name}`);
    }
}

// Discriminated union type guard
type Shape = 
    | { kind: "circle"; radius: number }
    | { kind: "rectangle"; width: number; height: number };

function getArea(shape: Shape): number {
    switch (shape.kind) {
        case "circle":
            return Math.PI * shape.radius ** 2;
        case "rectangle":
            return shape.width * shape.height;
    }
}

// Assertion functions (TypeScript 3.7+)
function assertIsString(value: unknown): asserts value is string {
    if (typeof value !== "string") {
        throw new Error("Value is not a string");
    }
}

function process(value: unknown): void {
    assertIsString(value);
    console.log(value.toUpperCase()); // value is string after assertion
}

// Array type guard
function isStringArray(arr: unknown[]): arr is string[] {
    return arr.every(item => typeof item === "string");
}
```

<a id="q12"></a>
### Q12: What are mapped types?
**Answer:**

Mapped types transform properties of existing types.

```typescript
// Basic mapped type
type Readonly<T> = {
    readonly [P in keyof T]: T[P];
};

type Optional<T> = {
    [P in keyof T]?: T[P];
};

// Using mapped types
interface User {
    id: number;
    name: string;
    email: string;
}

type ReadonlyUser = Readonly<User>;
type OptionalUser = Optional<User>;

// Mapping with key transformation
type Getters<T> = {
    [P in keyof T as `get${Capitalize<string & P>}`]: () => T[P];
};

type UserGetters = Getters<User>;
// {
//     getId: () => number;
//     getName: () => string;
//     getEmail: () => string;
// }

// Filtering keys
type FilterByType<T, U> = {
    [P in keyof T as T[P] extends U ? P : never]: T[P];
};

type StringProps = FilterByType<User, string>;
// { name: string; email: string }

// Remove specific keys
type RemoveId<T> = {
    [P in keyof T as Exclude<P, "id">]: T[P];
};

type UserWithoutId = RemoveId<User>;
// { name: string; email: string }

// Modifying value types
type Nullable<T> = {
    [P in keyof T]: T[P] | null;
};

type NullableUser = Nullable<User>;
// { id: number | null; name: string | null; email: string | null }

// Preserving modifiers
type CreateMutable<T> = {
    -readonly [P in keyof T]: T[P]; // Remove readonly
};

type CreateRequired<T> = {
    [P in keyof T]-?: T[P]; // Remove optional
};

// Practical example: Form fields
type FormFields<T> = {
    [P in keyof T]: {
        value: T[P];
        error: string | null;
        touched: boolean;
    };
};

type UserForm = FormFields<User>;
// {
//     id: { value: number; error: string | null; touched: boolean };
//     name: { value: string; error: string | null; touched: boolean };
//     email: { value: string; error: string | null; touched: boolean };
// }

// Deep mapped types
type DeepReadonly<T> = {
    readonly [P in keyof T]: T[P] extends object ? DeepReadonly<T[P]> : T[P];
};
```

<a id="q13"></a>
### Q13: What are conditional types?
**Answer:**

Conditional types select types based on conditions.

```typescript
// Basic conditional type
type IsString<T> = T extends string ? true : false;

type A = IsString<string>; // true
type B = IsString<number>; // false

// With infer - extract types
type ReturnType<T> = T extends (...args: any[]) => infer R ? R : never;

type FuncReturn = ReturnType<() => string>; // string

// Extract array element type
type ArrayElement<T> = T extends (infer E)[] ? E : never;

type Element = ArrayElement<string[]>; // string

// Unwrap Promise
type Awaited<T> = T extends Promise<infer U> ? U : T;

type Resolved = Awaited<Promise<string>>; // string

// Distributive conditional types
type ToArray<T> = T extends any ? T[] : never;

type Distributed = ToArray<string | number>;
// string[] | number[] (not (string | number)[])

// Prevent distribution with tuple
type ToArrayNonDist<T> = [T] extends [any] ? T[] : never;

type NonDistributed = ToArrayNonDist<string | number>;
// (string | number)[]

// Conditional type with union
type ExtractString<T> = T extends string ? T : never;

type Strings = ExtractString<string | number | boolean>;
// string

// Practical examples

// Function overload helper
type Func = {
    (x: string): string;
    (x: number): number;
};

type OverloadReturnType<T> = T extends {
    (...args: any[]): infer R;
    (...args: any[]): infer R;
} ? R : never;

type FuncReturns = OverloadReturnType<Func>; // string | number

// Extract keys by value type
type KeysOfType<T, V> = {
    [K in keyof T]: T[K] extends V ? K : never;
}[keyof T];

interface User {
    id: number;
    name: string;
    email: string;
    age: number;
}

type StringKeys = KeysOfType<User, string>; // "name" | "email"
type NumberKeys = KeysOfType<User, number>; // "id" | "age"

// Nested conditional
type TypeName<T> = 
    T extends string ? "string" :
    T extends number ? "number" :
    T extends boolean ? "boolean" :
    T extends undefined ? "undefined" :
    T extends Function ? "function" :
    "object";

type T1 = TypeName<string>; // "string"
type T2 = TypeName<() => void>; // "function"
```

<a id="q14"></a>
### Q14: What is declaration merging?
**Answer:**

Declaration merging combines multiple declarations with the same name.

```typescript
// Interface merging
interface User {
    name: string;
}

interface User {
    age: number;
}

// User is now { name: string; age: number }
const user: User = { name: "John", age: 30 };

// Merging with modules (augmentation)
// Extending Express Request
declare global {
    namespace Express {
        interface Request {
            user?: {
                id: string;
                email: string;
            };
        }
    }
}

// Now request.user is available in Express handlers

// Module augmentation
// Extending existing library types
import { AxiosRequestConfig } from "axios";

declare module "axios" {
    interface AxiosRequestConfig {
        retry?: number;
        retryDelay?: number;
    }
}

// Now axios config accepts retry and retryDelay

// Namespace merging
namespace Animals {
    export class Dog {}
}

namespace Animals {
    export class Cat {}
}

// Animals now has both Dog and Cat

// Function and namespace merging
function buildLabel(name: string): string {
    return buildLabel.prefix + name + buildLabel.suffix;
}

namespace buildLabel {
    export let prefix = "Hello, ";
    export let suffix = "!";
}

console.log(buildLabel("World")); // "Hello, World!"

// Class and namespace merging
class Album {
    label: Album.AlbumLabel;
    
    constructor(label: Album.AlbumLabel) {
        this.label = label;
    }
}

namespace Album {
    export interface AlbumLabel {
        name: string;
    }
}

const album = new Album({ name: "Label" });

// Enum and namespace merging
enum Color {
    Red,
    Green,
    Blue
}

namespace Color {
    export function mixColor(c1: Color, c2: Color): Color {
        // Mix logic
        return Color.Red;
    }
}

const mixed = Color.mixColor(Color.Red, Color.Blue);

// Global augmentation
declare global {
    interface Array<T> {
        customMethod(): T | undefined;
    }
}

Array.prototype.customMethod = function() {
    return this[0];
};
```

<a id="q15"></a>
### Q15: How do you handle modules and namespaces?
**Answer:**

```typescript
// ES Modules (recommended)

// Exporting
// math.ts
export const PI = 3.14159;

export function add(a: number, b: number): number {
    return a + b;
}

export default class Calculator {
    add(a: number, b: number): number {
        return a + b;
    }
}

// Importing
// app.ts
import Calculator, { PI, add } from "./math";
import * as math from "./math";
import { add as addition } from "./math";

// Re-exporting
// index.ts
export { PI, add } from "./math";
export { default as Calculator } from "./math";
export * from "./utils";
export * as utils from "./helpers";

// Type-only imports/exports
import type { User } from "./types";
export type { User };

import { type User, createUser } from "./user";

// Dynamic imports
async function loadModule() {
    const module = await import("./heavy-module");
    module.doSomething();
}

// Namespaces (internal modules - legacy)
namespace Validation {
    export interface StringValidator {
        isValid(s: string): boolean;
    }
    
    export class EmailValidator implements StringValidator {
        isValid(s: string): boolean {
            return s.includes("@");
        }
    }
}

const validator = new Validation.EmailValidator();

// Nested namespaces
namespace Shapes {
    export namespace Polygons {
        export class Triangle {}
        export class Square {}
    }
}

const triangle = new Shapes.Polygons.Triangle();

// Alias for nested namespace
import Polygons = Shapes.Polygons;
const square = new Polygons.Square();

// Ambient declarations (for external JS)
// globals.d.ts
declare const API_URL: string;
declare function fetchData(url: string): Promise<any>;

declare namespace MyLib {
    function doSomething(): void;
    let version: string;
}

// Declare module for untyped packages
declare module "untyped-library" {
    export function someFunction(): void;
    export default class SomeClass {}
}

// Wildcard module declarations
declare module "*.css" {
    const styles: { [key: string]: string };
    export default styles;
}

declare module "*.png" {
    const src: string;
    export default src;
}

// CommonJS interop
// tsconfig.json: "esModuleInterop": true

// Allows:
import fs from "fs"; // instead of import * as fs from "fs"
import express from "express";

// Module resolution
// tsconfig.json
{
    "compilerOptions": {
        "moduleResolution": "node", // or "classic"
        "baseUrl": "./src",
        "paths": {
            "@components/*": ["components/*"],
            "@utils/*": ["utils/*"]
        }
    }
}

// Now you can import:
import Button from "@components/Button";
import { format } from "@utils/format";
```

---

[← JavaScript Basics](javascript-basics.md) | [Back to Frontend Index](README.md) | [Node.js Basics →](nodejs-basics.md)
