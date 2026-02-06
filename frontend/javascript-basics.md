# JavaScript Basics Interview Questions

## Table of Contents

### Variables & Types
- [Q1: What are the differences between var, let, and const?](#q1)
- [Q2: What are JavaScript data types?](#q2)
- [Q3: Explain type coercion in JavaScript](#q3)
- [Q4: What is the difference between == and ===?](#q4)

### Functions & Scope
- [Q5: What are the different ways to define functions?](#q5)
- [Q6: Explain closures in JavaScript](#q6)
- [Q7: What is hoisting?](#q7)
- [Q8: Explain the 'this' keyword](#q8)

### Objects & Prototypes
- [Q9: How do you create objects in JavaScript?](#q9)
- [Q10: What is prototypal inheritance?](#q10)
- [Q11: Explain destructuring in JavaScript](#q11)
- [Q12: What is the spread operator?](#q12)

### Asynchronous JavaScript
- [Q13: What is the event loop?](#q13)
- [Q14: Explain callbacks and callback hell](#q14)
- [Q15: How do Promises work?](#q15)
- [Q16: What is async/await?](#q16)

### ES6+ Features
- [Q17: What are template literals?](#q17)
- [Q18: Explain JavaScript modules (import/export)](#q18)
- [Q19: What are Map and Set?](#q19)
- [Q20: What are symbols in JavaScript?](#q20)

---

## Variables & Types

<a id="q1"></a>
### Q1: What are the differences between var, let, and const?
**Answer:**

| Feature | var | let | const |
|---------|-----|-----|-------|
| Scope | Function | Block | Block |
| Hoisting | Yes (undefined) | Yes (TDZ) | Yes (TDZ) |
| Redeclaration | Yes | No | No |
| Reassignment | Yes | Yes | No |
| Global object | Yes | No | No |

```javascript
// var - function scoped, hoisted
function varExample() {
    console.log(x); // undefined (hoisted)
    var x = 10;
    
    if (true) {
        var x = 20; // Same variable
    }
    console.log(x); // 20
}

// let - block scoped, TDZ
function letExample() {
    // console.log(x); // ReferenceError (TDZ)
    let x = 10;
    
    if (true) {
        let x = 20; // Different variable
        console.log(x); // 20
    }
    console.log(x); // 10
}

// const - block scoped, immutable binding
const PI = 3.14159;
// PI = 3.14; // TypeError: Assignment to constant variable

// Objects/arrays can be mutated
const obj = { name: 'John' };
obj.name = 'Jane'; // OK
// obj = {}; // TypeError

const arr = [1, 2, 3];
arr.push(4); // OK
// arr = []; // TypeError

// Global scope difference
var globalVar = 'var';
let globalLet = 'let';
console.log(window.globalVar); // 'var'
console.log(window.globalLet); // undefined

// Best practice: Use const by default, let when needed, avoid var
```

<a id="q2"></a>
### Q2: What are JavaScript data types?
**Answer:**

JavaScript has **8 data types**: 7 primitives and 1 non-primitive.

| Type | Category | Example | typeof |
|------|----------|---------|--------|
| string | Primitive | `"hello"` | "string" |
| number | Primitive | `42`, `3.14` | "number" |
| bigint | Primitive | `9007199254740991n` | "bigint" |
| boolean | Primitive | `true`, `false` | "boolean" |
| undefined | Primitive | `undefined` | "undefined" |
| null | Primitive | `null` | "object" (bug) |
| symbol | Primitive | `Symbol('id')` | "symbol" |
| object | Non-primitive | `{}`, `[]`, `function` | "object" |

```javascript
// Primitives - immutable, passed by value
let str = "hello";
let num = 42;
let big = 9007199254740991n;
let bool = true;
let undef = undefined;
let nul = null;
let sym = Symbol('description');

// Objects - mutable, passed by reference
let obj = { name: 'John' };
let arr = [1, 2, 3];
let func = function() {};
let date = new Date();
let regex = /pattern/;

// Type checking
console.log(typeof "hello");    // "string"
console.log(typeof 42);         // "number"
console.log(typeof true);       // "boolean"
console.log(typeof undefined);  // "undefined"
console.log(typeof null);       // "object" (historical bug)
console.log(typeof Symbol());   // "symbol"
console.log(typeof {});         // "object"
console.log(typeof []);         // "object"
console.log(typeof function(){}); // "function"

// Better type checking
console.log(Array.isArray([]));           // true
console.log(obj instanceof Object);        // true
console.log(Object.prototype.toString.call([])); // "[object Array]"
console.log(Object.prototype.toString.call(null)); // "[object Null]"

// Number edge cases
console.log(typeof NaN);      // "number"
console.log(typeof Infinity); // "number"
console.log(Number.isNaN(NaN)); // true
console.log(Number.isFinite(Infinity)); // false
```

<a id="q3"></a>
### Q3: Explain type coercion in JavaScript
**Answer:**

Type coercion is automatic or implicit conversion between data types.

```javascript
// Implicit coercion (automatic)

// String coercion with +
console.log('5' + 3);      // '53' (number to string)
console.log('5' + true);   // '5true'
console.log('5' + null);   // '5null'
console.log('5' + {});     // '5[object Object]'

// Number coercion with -, *, /, %
console.log('5' - 3);      // 2
console.log('5' * '2');    // 10
console.log('10' / '2');   // 5
console.log('5' - true);   // 4 (true = 1)
console.log('5' - null);   // 5 (null = 0)

// Boolean coercion
// Falsy: false, 0, '', null, undefined, NaN
// Truthy: everything else

if ('hello') console.log('truthy');
if (0) console.log('never');

console.log(Boolean(''));     // false
console.log(Boolean('hello')); // true
console.log(Boolean(0));      // false
console.log(Boolean(1));      // true
console.log(Boolean([]));     // true (empty array is truthy!)
console.log(Boolean({}));     // true (empty object is truthy!)

// Explicit coercion
// To String
String(123);        // '123'
(123).toString();   // '123'
123 + '';           // '123'

// To Number
Number('123');      // 123
parseInt('123px');  // 123
parseFloat('3.14'); // 3.14
+'123';             // 123

// To Boolean
Boolean(1);         // true
!!1;                // true

// Common gotchas
console.log([] + []);       // '' (empty string)
console.log([] + {});       // '[object Object]'
console.log({} + []);       // '[object Object]' or 0 (depends on context)
console.log(true + true);   // 2
console.log(null + 1);      // 1
console.log(undefined + 1); // NaN

// Comparison coercion
console.log('10' > 9);      // true (string to number)
console.log('10' > '9');    // false (string comparison: '1' < '9')
```

<a id="q4"></a>
### Q4: What is the difference between == and ===?
**Answer:**

- `==` (loose equality) - compares with type coercion
- `===` (strict equality) - compares without type coercion

```javascript
// Strict equality (===) - no type coercion
console.log(5 === 5);         // true
console.log(5 === '5');       // false
console.log(null === null);   // true
console.log(undefined === undefined); // true
console.log(null === undefined); // false

// Loose equality (==) - with type coercion
console.log(5 == '5');        // true (string coerced to number)
console.log(0 == false);      // true
console.log(0 == '');         // true
console.log(false == '');     // true
console.log(null == undefined); // true (special case)
console.log(1 == true);       // true

// Tricky cases
console.log([] == false);     // true
console.log([] == 0);         // true
console.log([''] == '');      // true
console.log([0] == 0);        // true
console.log([[]] == 0);       // true

// NaN is never equal to itself
console.log(NaN === NaN);     // false
console.log(NaN == NaN);      // false
console.log(Number.isNaN(NaN)); // true (proper check)

// Objects are compared by reference
const a = { x: 1 };
const b = { x: 1 };
const c = a;

console.log(a == b);   // false (different references)
console.log(a === b);  // false
console.log(a === c);  // true (same reference)

// Best practice: Always use === unless you have a specific reason for ==

// Object.is() - stricter than ===
console.log(Object.is(NaN, NaN)); // true
console.log(Object.is(0, -0));    // false
console.log(0 === -0);            // true
```

---

## Functions & Scope

<a id="q5"></a>
### Q5: What are the different ways to define functions?
**Answer:**

```javascript
// 1. Function Declaration
function greet(name) {
    return `Hello, ${name}!`;
}
// Hoisted - can be called before declaration

// 2. Function Expression
const greet = function(name) {
    return `Hello, ${name}!`;
};
// Not hoisted - can only be called after declaration

// 3. Arrow Function (ES6)
const greet = (name) => {
    return `Hello, ${name}!`;
};

// Shortened arrow function (implicit return)
const greet = name => `Hello, ${name}!`;
const add = (a, b) => a + b;

// 4. Method definition in object
const obj = {
    // Traditional
    greet: function(name) {
        return `Hello, ${name}!`;
    },
    // Shorthand (ES6)
    sayHi(name) {
        return `Hi, ${name}!`;
    }
};

// 5. Constructor Function
function Person(name) {
    this.name = name;
}
const john = new Person('John');

// 6. Class method (ES6)
class Greeter {
    greet(name) {
        return `Hello, ${name}!`;
    }
}

// 7. Generator Function
function* numberGenerator() {
    yield 1;
    yield 2;
    yield 3;
}

// 8. Async Function
async function fetchData() {
    const response = await fetch('/api/data');
    return response.json();
}

// Arrow function differences
const obj = {
    name: 'John',
    regularFunc: function() {
        return this.name; // 'John' - has its own 'this'
    },
    arrowFunc: () => {
        return this.name; // undefined - inherits 'this' from parent scope
    }
};

// Arrow functions cannot:
// - Be used as constructors (new arrow())
// - Access arguments object
// - Be used as methods when you need 'this'
```

<a id="q6"></a>
### Q6: Explain closures in JavaScript
**Answer:**

A closure is a function that has access to variables from its outer (enclosing) function's scope, even after the outer function has returned.

```javascript
// Basic closure
function outer() {
    const outerVar = 'I am from outer';
    
    function inner() {
        console.log(outerVar); // Accesses outerVar
    }
    
    return inner;
}

const closure = outer();
closure(); // 'I am from outer'
// inner() still has access to outerVar even after outer() returned

// Practical examples

// 1. Data privacy / encapsulation
function createCounter() {
    let count = 0; // Private variable
    
    return {
        increment() { return ++count; },
        decrement() { return --count; },
        getCount() { return count; }
    };
}

const counter = createCounter();
console.log(counter.increment()); // 1
console.log(counter.increment()); // 2
console.log(counter.getCount());  // 2
// count is not directly accessible

// 2. Function factory
function multiplier(factor) {
    return function(number) {
        return number * factor;
    };
}

const double = multiplier(2);
const triple = multiplier(3);
console.log(double(5)); // 10
console.log(triple(5)); // 15

// 3. Memoization
function memoize(fn) {
    const cache = {};
    
    return function(...args) {
        const key = JSON.stringify(args);
        if (cache[key]) {
            console.log('From cache');
            return cache[key];
        }
        const result = fn.apply(this, args);
        cache[key] = result;
        return result;
    };
}

const expensiveOperation = memoize((n) => {
    console.log('Computing...');
    return n * 2;
});

expensiveOperation(5); // Computing... 10
expensiveOperation(5); // From cache 10

// 4. Event handlers with data
function setupButtons() {
    for (var i = 0; i < 3; i++) {
        // Problem with var
        document.getElementById(`btn${i}`).onclick = function() {
            console.log(i); // Always 3!
        };
    }
    
    // Solution 1: Use let
    for (let i = 0; i < 3; i++) {
        document.getElementById(`btn${i}`).onclick = function() {
            console.log(i); // 0, 1, 2
        };
    }
    
    // Solution 2: IIFE closure
    for (var i = 0; i < 3; i++) {
        (function(index) {
            document.getElementById(`btn${index}`).onclick = function() {
                console.log(index); // 0, 1, 2
            };
        })(i);
    }
}
```

<a id="q7"></a>
### Q7: What is hoisting?
**Answer:**

Hoisting is JavaScript's behavior of moving declarations to the top of their scope during compilation.

```javascript
// Variable hoisting

// var is hoisted with undefined
console.log(x); // undefined
var x = 5;
console.log(x); // 5

// Interpreted as:
var x;
console.log(x); // undefined
x = 5;
console.log(x); // 5

// let and const - Temporal Dead Zone (TDZ)
console.log(y); // ReferenceError: Cannot access 'y' before initialization
let y = 5;

// TDZ exists from start of block until declaration
{
    // TDZ starts
    console.log(z); // ReferenceError
    let z = 10; // TDZ ends
}

// Function hoisting

// Function declarations are fully hoisted
greet(); // "Hello!" - works!
function greet() {
    console.log("Hello!");
}

// Function expressions are not hoisted
sayHi(); // TypeError: sayHi is not a function
var sayHi = function() {
    console.log("Hi!");
};

// Only declaration is hoisted:
var sayHi;
sayHi(); // TypeError
sayHi = function() {
    console.log("Hi!");
};

// Class hoisting - similar to let (TDZ)
const instance = new MyClass(); // ReferenceError
class MyClass {}

// Hoisting order: function declarations > variable declarations
var foo = 'variable';
function foo() {
    return 'function';
}
console.log(typeof foo); // 'string' (variable assigned after function)

// Best practices:
// 1. Declare variables at the top of their scope
// 2. Use const/let instead of var
// 3. Declare functions before using them
```

<a id="q8"></a>
### Q8: Explain the 'this' keyword
**Answer:**

`this` refers to the object that is executing the current function. Its value depends on how the function is called.

```javascript
// 1. Global context
console.log(this); // Window (browser) or global (Node)

// 2. Function context - depends on how called
function showThis() {
    console.log(this);
}
showThis(); // Window (non-strict) or undefined (strict mode)

// 3. Object method
const obj = {
    name: 'John',
    greet() {
        console.log(this.name);
    }
};
obj.greet(); // 'John' - this = obj

// But if we extract the method:
const greet = obj.greet;
greet(); // undefined - this = Window/undefined

// 4. Constructor function
function Person(name) {
    this.name = name;
}
const john = new Person('John');
console.log(john.name); // 'John' - this = new object

// 5. Arrow functions - inherit 'this' from parent scope
const obj = {
    name: 'John',
    regularFunc: function() {
        console.log(this.name); // 'John'
        
        const arrowFunc = () => {
            console.log(this.name); // 'John' - inherited
        };
        arrowFunc();
    },
    arrowMethod: () => {
        console.log(this.name); // undefined - parent is global
    }
};

// 6. Explicit binding

// call - calls with this and args
function greet(greeting, punctuation) {
    console.log(`${greeting}, ${this.name}${punctuation}`);
}
greet.call({ name: 'John' }, 'Hello', '!'); // 'Hello, John!'

// apply - same but args as array
greet.apply({ name: 'John' }, ['Hello', '!']); // 'Hello, John!'

// bind - returns new function with bound this
const boundGreet = greet.bind({ name: 'John' });
boundGreet('Hello', '!'); // 'Hello, John!'

// 7. Event handlers
button.addEventListener('click', function() {
    console.log(this); // button element
});

button.addEventListener('click', () => {
    console.log(this); // Window (inherited from parent)
});

// 8. Class context
class Person {
    constructor(name) {
        this.name = name;
        // 'this' here refers to the new instance being created
    }
    
    // Regular method - 'this' depends on how it's CALLED
    greet() {
        console.log(this.name);
    }
    
    // Arrow function as class field
    // Arrow functions don't have their own 'this'
    // They CAPTURE 'this' from enclosing scope (lexical 'this')
    // This field is initialized during construction, so it captures the instance
    // Equivalent to writing in constructor: this.greetArrow = () => {...}
    greetArrow = () => {
        console.log(this.name);
    }
}

const person = new Person('John');
const greet = person.greet;
const greetArrow = person.greetArrow;

greet(); // undefined - 'this' is determined by call site (no object.method())
greetArrow(); // 'John' - 'this' was captured when arrow function was created

// Why arrow functions "remember" this:
// - Arrow functions have NO own 'this'
// - They use 'this' from their lexical (surrounding) scope
// - greetArrow was created inside constructor where 'this' = instance
// - So it permanently uses that instance as 'this'
```

---

## Objects & Prototypes

<a id="q9"></a>
### Q9: How do you create objects in JavaScript?
**Answer:**

```javascript
// 1. Object literal
const person = {
    name: 'John',
    age: 30,
    greet() {
        return `Hello, I'm ${this.name}`;
    }
};

// 2. Object constructor
const person = new Object();
person.name = 'John';
person.age = 30;

// 3. Object.create() - with prototype
const personProto = {
    greet() {
        return `Hello, I'm ${this.name}`;
    }
};

const person = Object.create(personProto);
person.name = 'John';
person.age = 30;

// Object.create(null) - no prototype
const pureObject = Object.create(null);
// pureObject has no toString, hasOwnProperty, etc.

// 4. Constructor function
function Person(name, age) {
    this.name = name;
    this.age = age;
}
Person.prototype.greet = function() {
    return `Hello, I'm ${this.name}`;
};

const person = new Person('John', 30);

// 5. Class (ES6)
class Person {
    constructor(name, age) {
        this.name = name;
        this.age = age;
    }
    
    greet() {
        return `Hello, I'm ${this.name}`;
    }
}

const person = new Person('John', 30);

// 6. Factory function
function createPerson(name, age) {
    return {
        name,
        age,
        greet() {
            return `Hello, I'm ${this.name}`;
        }
    };
}

const person = createPerson('John', 30);

// Object methods
const obj = { a: 1, b: 2 };

Object.keys(obj);    // ['a', 'b']
Object.values(obj);  // [1, 2]
Object.entries(obj); // [['a', 1], ['b', 2]]

Object.freeze(obj);  // Immutable
Object.seal(obj);    // Can't add/remove, can modify

Object.assign({}, obj, { c: 3 }); // Merge/clone
const clone = { ...obj }; // Spread clone (shallow)

// Deep clone
const deepClone = JSON.parse(JSON.stringify(obj)); // Simple but limited
const deepClone = structuredClone(obj); // Modern, handles more types
```

<a id="q10"></a>
### Q10: What is prototypal inheritance?
**Answer:**

JavaScript uses prototypes for inheritance. Every object has an internal `[[Prototype]]` link to another object.

```javascript
// Every object has a prototype
const obj = {};
console.log(obj.__proto__ === Object.prototype); // true

// Prototype chain
const arr = [];
arr.__proto__ === Array.prototype;      // true
arr.__proto__.__proto__ === Object.prototype; // true
arr.__proto__.__proto__.__proto__ === null;   // true (end of chain)

// Constructor function prototype
function Animal(name) {
    this.name = name;
}

Animal.prototype.speak = function() {
    console.log(`${this.name} makes a sound`);
};

function Dog(name, breed) {
    Animal.call(this, name); // Call parent constructor
    this.breed = breed;
}

// Set up inheritance
Dog.prototype = Object.create(Animal.prototype);
Dog.prototype.constructor = Dog;

Dog.prototype.bark = function() {
    console.log(`${this.name} barks`);
};

const dog = new Dog('Rex', 'German Shepherd');
dog.speak(); // 'Rex makes a sound' (inherited)
dog.bark();  // 'Rex barks'

// ES6 Class syntax (syntactic sugar over prototypes)
class Animal {
    constructor(name) {
        this.name = name;
    }
    
    speak() {
        console.log(`${this.name} makes a sound`);
    }
}

class Dog extends Animal {
    constructor(name, breed) {
        super(name); // Call parent constructor
        this.breed = breed;
    }
    
    bark() {
        console.log(`${this.name} barks`);
    }
}

// Property lookup in prototype chain
const dog = new Dog('Rex', 'Shepherd');
dog.hasOwnProperty('name');  // true (own property)
dog.hasOwnProperty('speak'); // false (inherited)
'speak' in dog;              // true (includes prototype)

// Check prototype
Dog.prototype.isPrototypeOf(dog); // true
dog instanceof Dog;               // true
dog instanceof Animal;            // true

// Modifying prototypes (not recommended for built-ins)
Array.prototype.first = function() {
    return this[0];
};
[1, 2, 3].first(); // 1
```

<a id="q11"></a>
### Q11: Explain destructuring in JavaScript
**Answer:**

Destructuring is a syntax for extracting values from arrays or properties from objects.

```javascript
// Array destructuring
const numbers = [1, 2, 3, 4, 5];

const [first, second] = numbers;
console.log(first, second); // 1, 2

// Skip elements
const [a, , b] = numbers;
console.log(a, b); // 1, 3

// Rest pattern
const [head, ...tail] = numbers;
console.log(head); // 1
console.log(tail); // [2, 3, 4, 5]

// Default values
const [x, y, z = 0] = [1, 2];
console.log(z); // 0

// Swapping variables
let a = 1, b = 2;
[a, b] = [b, a];
console.log(a, b); // 2, 1

// Object destructuring
const person = { name: 'John', age: 30, city: 'NYC' };

const { name, age } = person;
console.log(name, age); // 'John', 30

// Rename variables
const { name: personName, age: personAge } = person;
console.log(personName); // 'John'

// Default values
const { name, country = 'USA' } = person;
console.log(country); // 'USA'

// Nested destructuring
const user = {
    name: 'John',
    address: {
        city: 'NYC',
        zip: '10001'
    }
};

const { address: { city, zip } } = user;
console.log(city, zip); // 'NYC', '10001'

// Rest pattern in objects
const { name, ...rest } = person;
console.log(rest); // { age: 30, city: 'NYC' }

// Function parameters
function greet({ name, age }) {
    console.log(`${name} is ${age}`);
}
greet(person); // 'John is 30'

// With defaults
function greet({ name = 'Guest', age = 0 } = {}) {
    console.log(`${name} is ${age}`);
}
greet(); // 'Guest is 0'

// Mixed destructuring
const data = {
    title: 'Post',
    comments: ['Great!', 'Nice!']
};

const { title, comments: [firstComment] } = data;
console.log(firstComment); // 'Great!'
```

<a id="q12"></a>
### Q12: What is the spread operator?
**Answer:**

The spread operator (`...`) expands iterables into individual elements.

```javascript
// Array spreading
const arr1 = [1, 2, 3];
const arr2 = [4, 5, 6];

// Combine arrays
const combined = [...arr1, ...arr2];
console.log(combined); // [1, 2, 3, 4, 5, 6]

// Clone array (shallow)
const clone = [...arr1];

// Add elements
const withNew = [0, ...arr1, 4];
console.log(withNew); // [0, 1, 2, 3, 4]

// Function arguments
function sum(a, b, c) {
    return a + b + c;
}
const nums = [1, 2, 3];
console.log(sum(...nums)); // 6

// Math functions
const numbers = [5, 2, 8, 1, 9];
console.log(Math.max(...numbers)); // 9
console.log(Math.min(...numbers)); // 1

// Object spreading
const obj1 = { a: 1, b: 2 };
const obj2 = { c: 3, d: 4 };

// Merge objects
const merged = { ...obj1, ...obj2 };
console.log(merged); // { a: 1, b: 2, c: 3, d: 4 }

// Clone object (shallow)
const clone = { ...obj1 };

// Override properties
const updated = { ...obj1, b: 20 };
console.log(updated); // { a: 1, b: 20 }

// Add properties
const withNew = { ...obj1, e: 5 };

// Order matters - last wins
const result = { a: 1, ...{ a: 2 } };
console.log(result); // { a: 2 }

// String spreading
const str = 'hello';
const chars = [...str];
console.log(chars); // ['h', 'e', 'l', 'l', 'o']

// Rest parameters (collecting)
function sum(...numbers) {
    return numbers.reduce((a, b) => a + b, 0);
}
console.log(sum(1, 2, 3, 4)); // 10

// Rest in destructuring
const [first, ...rest] = [1, 2, 3, 4];
console.log(first); // 1
console.log(rest);  // [2, 3, 4]

const { a, ...others } = { a: 1, b: 2, c: 3 };
console.log(others); // { b: 2, c: 3 }

// Note: Spread creates shallow copies
const nested = { a: { b: 1 } };
const copy = { ...nested };
copy.a.b = 2;
console.log(nested.a.b); // 2 (affected!)
```

---

## Asynchronous JavaScript

<a id="q13"></a>
### Q13: What is the event loop?
**Answer:**

The event loop is how JavaScript handles asynchronous operations in a single-threaded environment.

```javascript
// JavaScript is single-threaded but non-blocking

// Components:
// 1. Call Stack - executes code
// 2. Web APIs - handles async operations (setTimeout, fetch, etc.)
// 3. Callback Queue (Task Queue) - queues callbacks
// 4. Microtask Queue - higher priority (Promises)
// 5. Event Loop - moves tasks from queue to stack when stack is empty

console.log('1'); // Call stack

setTimeout(() => {
    console.log('2'); // Callback queue (macrotask)
}, 0);

Promise.resolve().then(() => {
    console.log('3'); // Microtask queue
});

console.log('4'); // Call stack

// Output: 1, 4, 3, 2
// Explanation:
// 1. '1' - executed immediately
// 2. setTimeout callback -> Web API -> Callback Queue
// 3. Promise.then -> Microtask Queue
// 4. '4' - executed immediately
// 5. Stack empty -> Microtasks first -> '3'
// 6. Stack empty -> Callback Queue -> '2'

// Priority: Microtasks > Macrotasks

// Macrotasks (Callback Queue):
// - setTimeout, setInterval
// - setImmediate (Node.js)
// - I/O operations
// - UI rendering

// Microtasks:
// - Promise callbacks (.then, .catch, .finally)
// - queueMicrotask()
// - MutationObserver

// Example with multiple tasks
console.log('Start');

setTimeout(() => console.log('Timeout 1'), 0);
setTimeout(() => console.log('Timeout 2'), 0);

Promise.resolve()
    .then(() => console.log('Promise 1'))
    .then(() => console.log('Promise 2'));

Promise.resolve().then(() => console.log('Promise 3'));

console.log('End');

// Output:
// Start
// End
// Promise 1
// Promise 3
// Promise 2
// Timeout 1
// Timeout 2

// requestAnimationFrame - before repaint
requestAnimationFrame(() => {
    console.log('Animation frame');
});

// Avoid blocking the event loop
// Bad - blocks for 1 second
function blockingOperation() {
    const start = Date.now();
    while (Date.now() - start < 1000) {}
    console.log('Done blocking');
}

// Good - use async operations or Web Workers
```

<a id="q14"></a>
### Q14: Explain callbacks and callback hell
**Answer:**

```javascript
// Callbacks - functions passed to other functions to be called later

// Simple callback
function greet(name, callback) {
    console.log(`Hello, ${name}`);
    callback();
}

greet('John', () => {
    console.log('Callback executed');
});

// Async callback
function fetchData(callback) {
    setTimeout(() => {
        callback(null, { data: 'result' });
    }, 1000);
}

fetchData((error, data) => {
    if (error) {
        console.error(error);
        return;
    }
    console.log(data);
});

// Error-first callback pattern (Node.js convention)
function readFile(path, callback) {
    // callback(error, result)
    if (!path) {
        callback(new Error('Path required'), null);
        return;
    }
    callback(null, 'file content');
}

readFile('file.txt', (err, content) => {
    if (err) {
        console.error(err);
        return;
    }
    console.log(content);
});

// Callback Hell (Pyramid of Doom)
getUser(userId, (err, user) => {
    if (err) {
        handleError(err);
        return;
    }
    getOrders(user.id, (err, orders) => {
        if (err) {
            handleError(err);
            return;
        }
        getOrderDetails(orders[0].id, (err, details) => {
            if (err) {
                handleError(err);
                return;
            }
            getPaymentInfo(details.paymentId, (err, payment) => {
                if (err) {
                    handleError(err);
                    return;
                }
                // Finally do something with payment
                console.log(payment);
            });
        });
    });
});

// Solutions to callback hell:

// 1. Named functions
function handleUser(err, user) {
    if (err) return handleError(err);
    getOrders(user.id, handleOrders);
}

function handleOrders(err, orders) {
    if (err) return handleError(err);
    getOrderDetails(orders[0].id, handleDetails);
}
// etc.

getUser(userId, handleUser);

// 2. Promises (see next question)

// 3. Async/await (see Q16)
```

<a id="q15"></a>
### Q15: How do Promises work?
**Answer:**

A Promise is an object representing the eventual completion or failure of an async operation.

```javascript
// Promise states:
// - pending: initial state
// - fulfilled: operation completed successfully
// - rejected: operation failed

// Creating a Promise
const promise = new Promise((resolve, reject) => {
    // Async operation
    setTimeout(() => {
        const success = true;
        if (success) {
            resolve('Data loaded');
        } else {
            reject(new Error('Failed to load'));
        }
    }, 1000);
});

// Consuming a Promise
promise
    .then(result => {
        console.log(result); // 'Data loaded'
        return 'Processed';
    })
    .then(result => {
        console.log(result); // 'Processed'
    })
    .catch(error => {
        console.error(error);
    })
    .finally(() => {
        console.log('Cleanup');
    });

// Promise chaining - solving callback hell
getUser(userId)
    .then(user => getOrders(user.id))
    .then(orders => getOrderDetails(orders[0].id))
    .then(details => getPaymentInfo(details.paymentId))
    .then(payment => {
        console.log(payment);
    })
    .catch(error => {
        handleError(error);
    });

// Promise static methods

// Promise.all - wait for all to resolve (fails fast)
const promises = [
    fetch('/api/users'),
    fetch('/api/posts'),
    fetch('/api/comments')
];

Promise.all(promises)
    .then(([users, posts, comments]) => {
        // All resolved
    })
    .catch(error => {
        // Any one rejected
    });

// Promise.allSettled - wait for all to settle
Promise.allSettled(promises)
    .then(results => {
        results.forEach(result => {
            if (result.status === 'fulfilled') {
                console.log(result.value);
            } else {
                console.log(result.reason);
            }
        });
    });

// Promise.race - first to settle wins
Promise.race([
    fetch('/api/data'),
    new Promise((_, reject) => 
        setTimeout(() => reject(new Error('Timeout')), 5000)
    )
]);

// Promise.any - first to resolve wins
Promise.any(promises)
    .then(firstResolved => {
        console.log(firstResolved);
    })
    .catch(errors => {
        // All rejected - AggregateError
    });

// Promise.resolve / Promise.reject
Promise.resolve('value'); // Returns fulfilled promise
Promise.reject(new Error('error')); // Returns rejected promise

// Converting callback to Promise (Promisification)
function readFilePromise(path) {
    return new Promise((resolve, reject) => {
        fs.readFile(path, (err, data) => {
            if (err) reject(err);
            else resolve(data);
        });
    });
}

// Node.js util.promisify
const { promisify } = require('util');
const readFilePromise = promisify(fs.readFile);
```

<a id="q16"></a>
### Q16: What is async/await?
**Answer:**

async/await is syntactic sugar for Promises, making async code look synchronous.

```javascript
// async function always returns a Promise
async function getData() {
    return 'data';
}
getData().then(console.log); // 'data'

// await pauses execution until Promise resolves
async function fetchUser() {
    const response = await fetch('/api/user');
    const user = await response.json();
    return user;
}

// Compared to Promises
function fetchUserPromise() {
    return fetch('/api/user')
        .then(response => response.json());
}

// Error handling
async function fetchData() {
    try {
        const response = await fetch('/api/data');
        if (!response.ok) {
            throw new Error('HTTP error');
        }
        const data = await response.json();
        return data;
    } catch (error) {
        console.error('Error:', error);
        throw error; // Re-throw if needed
    } finally {
        console.log('Cleanup');
    }
}

// Sequential execution
async function sequential() {
    const user = await getUser(); // Wait...
    const orders = await getOrders(user.id); // Wait...
    const details = await getDetails(orders[0]); // Wait...
    return details;
}

// Parallel execution
async function parallel() {
    const [users, posts, comments] = await Promise.all([
        fetch('/api/users').then(r => r.json()),
        fetch('/api/posts').then(r => r.json()),
        fetch('/api/comments').then(r => r.json())
    ]);
    return { users, posts, comments };
}

// Async iteration
async function processItems(items) {
    for (const item of items) {
        await processItem(item); // Sequential
    }
}

// Parallel processing with map
async function processItemsParallel(items) {
    await Promise.all(items.map(item => processItem(item)));
}

// Top-level await (ES2022, modules only)
// In module:
const data = await fetchData();

// IIFE for older environments
(async () => {
    const data = await fetchData();
    console.log(data);
})();

// Async class methods
class DataService {
    async fetchData() {
        const response = await fetch('/api/data');
        return response.json();
    }
}

// Common patterns

// Retry with exponential backoff
async function fetchWithRetry(url, retries = 3) {
    for (let i = 0; i < retries; i++) {
        try {
            return await fetch(url);
        } catch (error) {
            if (i === retries - 1) throw error;
            await new Promise(r => setTimeout(r, 2 ** i * 1000));
        }
    }
}

// Timeout
async function fetchWithTimeout(url, timeout = 5000) {
    const controller = new AbortController();
    const timeoutId = setTimeout(() => controller.abort(), timeout);
    
    try {
        const response = await fetch(url, { signal: controller.signal });
        return response;
    } finally {
        clearTimeout(timeoutId);
    }
}
```

---

## ES6+ Features

<a id="q17"></a>
### Q17: What are template literals?
**Answer:**

Template literals are string literals with embedded expressions, using backticks.

```javascript
// Basic usage
const name = 'John';
const greeting = `Hello, ${name}!`;
console.log(greeting); // 'Hello, John!'

// Multi-line strings
const html = `
    <div>
        <h1>Title</h1>
        <p>Content</p>
    </div>
`;

// Expressions
const a = 5, b = 10;
console.log(`Sum: ${a + b}`);           // 'Sum: 15'
console.log(`Result: ${a > b ? 'a' : 'b'}`); // 'Result: b'

// Function calls
function upper(str) {
    return str.toUpperCase();
}
console.log(`Hello, ${upper('world')}!`); // 'Hello, WORLD!'

// Nested templates
const items = ['apple', 'banana', 'orange'];
const list = `
    <ul>
        ${items.map(item => `<li>${item}</li>`).join('')}
    </ul>
`;

// Escaping
console.log(`\`Backticks\` and \${not interpolated}`);
// `Backticks` and ${not interpolated}

// Tagged templates
function highlight(strings, ...values) {
    return strings.reduce((result, str, i) => {
        const value = values[i] ? `<strong>${values[i]}</strong>` : '';
        return result + str + value;
    }, '');
}

const name = 'John';
const age = 30;
const result = highlight`Name: ${name}, Age: ${age}`;
// 'Name: <strong>John</strong>, Age: <strong>30</strong>'

// Raw strings
console.log(String.raw`Line1\nLine2`); // 'Line1\nLine2' (not Line1<newline>Line2)

// Practical uses

// SQL queries (with proper escaping library)
const query = sql`SELECT * FROM users WHERE id = ${userId}`;

// CSS-in-JS
const styles = css`
    color: ${theme.primary};
    font-size: ${fontSize}px;
`;

// HTML templates
function renderCard(title, content) {
    return `
        <div class="card">
            <h2>${escapeHtml(title)}</h2>
            <p>${escapeHtml(content)}</p>
        </div>
    `;
}

// Internationalization
function i18n(strings, ...values) {
    // Look up translation, format values
    return translate(strings, values);
}
const message = i18n`Hello, ${name}!`;
```

<a id="q18"></a>
### Q18: Explain JavaScript modules (import/export)
**Answer:**

ES Modules provide a way to organize code into separate files.

```javascript
// Named exports
// math.js
export const PI = 3.14159;
export function add(a, b) {
    return a + b;
}
export function subtract(a, b) {
    return a - b;
}

// Or export at end
const PI = 3.14159;
function add(a, b) { return a + b; }
function subtract(a, b) { return a - b; }
export { PI, add, subtract };

// Named imports
// app.js
import { PI, add, subtract } from './math.js';
console.log(add(2, 3)); // 5

// Import with alias
import { add as addition } from './math.js';

// Import all as namespace
import * as math from './math.js';
console.log(math.PI);    // 3.14159
console.log(math.add(2, 3)); // 5

// Default export (one per module)
// greeting.js
export default function greet(name) {
    return `Hello, ${name}!`;
}

// Or
function greet(name) {
    return `Hello, ${name}!`;
}
export default greet;

// Default import (can use any name)
import greet from './greeting.js';
import sayHello from './greeting.js'; // Same thing

// Mixed exports
// utils.js
export const VERSION = '1.0.0';
export default class Utils {
    static format(str) { return str.trim(); }
}

// Mixed imports
import Utils, { VERSION } from './utils.js';

// Re-exports
// index.js
export { add, subtract } from './math.js';
export { default as greet } from './greeting.js';
export * from './utils.js'; // All named exports

// Dynamic imports (lazy loading)
async function loadModule() {
    const module = await import('./heavy-module.js');
    module.doSomething();
}

// Conditional loading
if (condition) {
    const { feature } = await import('./feature.js');
    feature();
}

// CommonJS (Node.js traditional)
// Exporting
module.exports = { add, subtract };
// or
exports.add = add;

// Importing
const { add } = require('./math');
const math = require('./math');

// ES Modules in Node.js
// Use .mjs extension or "type": "module" in package.json

// Import JSON (with assertion)
import data from './data.json' assert { type: 'json' };

// Import meta
console.log(import.meta.url); // File URL of current module
```

<a id="q19"></a>
### Q19: What are Map and Set?
**Answer:**

Map and Set are built-in collection types introduced in ES6.

```javascript
// SET - collection of unique values
const set = new Set();

// Add values
set.add(1);
set.add(2);
set.add(2); // Duplicate ignored
set.add('hello');
set.add({ id: 1 });

console.log(set.size); // 4

// Check existence
console.log(set.has(1)); // true
console.log(set.has(3)); // false

// Delete
set.delete(2);

// Iteration
for (const value of set) {
    console.log(value);
}

set.forEach(value => console.log(value));

// Convert to array
const arr = [...set];
const arr2 = Array.from(set);

// Use case: Remove duplicates
const numbers = [1, 2, 2, 3, 3, 3];
const unique = [...new Set(numbers)]; // [1, 2, 3]

// Use case: Track unique items
const visited = new Set();
visited.add(url);
if (!visited.has(url)) { /* visit */ }

// Clear all
set.clear();

// MAP - key-value pairs with any key type
const map = new Map();

// Set values (any key type)
map.set('name', 'John');
map.set(1, 'one');
map.set(true, 'boolean');
const objKey = { id: 1 };
map.set(objKey, 'object value');

// Get values
console.log(map.get('name')); // 'John'
console.log(map.get(objKey)); // 'object value'

// Check existence
console.log(map.has('name')); // true

// Size
console.log(map.size); // 4

// Delete
map.delete('name');

// Iteration
for (const [key, value] of map) {
    console.log(key, value);
}

map.forEach((value, key) => console.log(key, value));

// Get keys/values/entries
map.keys();    // Iterator
map.values();  // Iterator
map.entries(); // Iterator

// Initialize with array of pairs
const map2 = new Map([
    ['name', 'John'],
    ['age', 30]
]);

// Object vs Map
// Object:
// - Keys are strings/symbols only
// - Has prototype properties
// - No size property
// - Not directly iterable

// Map:
// - Any key type
// - No prototype interference
// - Has size property
// - Directly iterable
// - Better performance for frequent add/delete

// WeakSet / WeakMap - keys are weakly referenced (garbage collected)
const weakMap = new WeakMap();
let obj = { id: 1 };
weakMap.set(obj, 'data');
obj = null; // obj can be garbage collected

// Use case: Private data
const privateData = new WeakMap();

class Person {
    constructor(name) {
        privateData.set(this, { name });
    }
    
    getName() {
        return privateData.get(this).name;
    }
}
```

<a id="q20"></a>
### Q20: What are symbols in JavaScript?
**Answer:**

Symbols are unique, immutable primitive values used as identifiers.

```javascript
// Creating symbols
const sym1 = Symbol();
const sym2 = Symbol('description');
const sym3 = Symbol('description');

console.log(sym2 === sym3); // false - always unique

// Use as object keys
const ID = Symbol('id');
const user = {
    name: 'John',
    [ID]: 123
};

console.log(user[ID]); // 123
console.log(user.ID);  // undefined

// Symbols are not enumerable
console.log(Object.keys(user));           // ['name']
console.log(Object.getOwnPropertyNames(user)); // ['name']
console.log(Object.getOwnPropertySymbols(user)); // [Symbol(id)]

// Use case: Unique property keys (no collision)
const library1Id = Symbol('id');
const library2Id = Symbol('id');

const obj = {
    [library1Id]: 'value1',
    [library2Id]: 'value2'  // No collision!
};

// Well-known symbols (built-in)

// Symbol.iterator - makes object iterable
const iterable = {
    data: [1, 2, 3],
    [Symbol.iterator]() {
        let index = 0;
        return {
            next: () => ({
                value: this.data[index++],
                done: index > this.data.length
            })
        };
    }
};

for (const item of iterable) {
    console.log(item); // 1, 2, 3
}

// Symbol.toStringTag - custom toString
class MyClass {
    get [Symbol.toStringTag]() {
        return 'MyClass';
    }
}
console.log(Object.prototype.toString.call(new MyClass())); 
// '[object MyClass]'

// Symbol.toPrimitive - custom type conversion
const obj = {
    [Symbol.toPrimitive](hint) {
        if (hint === 'number') return 42;
        if (hint === 'string') return 'hello';
        return null;
    }
};

console.log(+obj);     // 42
console.log(`${obj}`); // 'hello'

// Global symbol registry
const globalSym = Symbol.for('app.id');
const same = Symbol.for('app.id');
console.log(globalSym === same); // true

const key = Symbol.keyFor(globalSym);
console.log(key); // 'app.id'

// Use cases:
// 1. Unique keys to avoid collisions
// 2. Implementing protocols (iterators)
// 3. Semi-private properties
// 4. Constants that don't clash
const STATUS = {
    PENDING: Symbol('pending'),
    APPROVED: Symbol('approved'),
    REJECTED: Symbol('rejected')
};
```

---

[← Back to Frontend Index](README.md) | [TypeScript Basics →](typescript-basics.md)
