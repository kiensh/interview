# Modern Java (Streams, Functional, Java 8/11/17)

## Table of Contents

### Stream API
- [Q1: What is the Stream API and how does it work?](#q1)
- [Q2: What are intermediate and terminal operations?](#q2)
- [Q3: What is the difference between map() and flatMap()?](#q3)
- [Q4: How do parallel streams work?](#q4)

### Functional Programming
- [Q5: What are functional interfaces in Java?](#q5)
- [Q6: What are the built-in functional interfaces?](#q6)
- [Q7: What are lambda expressions and method references?](#q7)
- [Q8: What is Optional and how should you use it?](#q8)

### Java Version Features
- [Q9: What are the key features in Java 8?](#q9)
- [Q10: What are the key features in Java 11?](#q10)
- [Q11: What are the key features in Java 17?](#q11)

---

## Stream API

<a id="q1"></a>
### Q1: What is the Stream API and how does it work?
**Answer:**
The Stream API (Java 8+) provides a functional approach to processing collections.

**Key characteristics:**
- **Not a data structure**: Doesn't store data, operates on a source
- **Lazy evaluation**: Intermediate operations are not executed until terminal operation
- **Consumable**: Can only be traversed once

```java
List<String> names = Arrays.asList("Alice", "Bob", "Charlie", "David");

List<String> result = names.stream()           // Create stream
    .filter(name -> name.length() > 3)         // Intermediate
    .map(String::toUpperCase)                  // Intermediate
    .sorted()                                  // Intermediate
    .collect(Collectors.toList());             // Terminal
// Result: [ALICE, CHARLIE, DAVID]
```

**Creating streams:**
```java
Stream<String> stream1 = list.stream();
Stream<String> stream2 = Arrays.stream(arr);
Stream<String> stream3 = Stream.of("a", "b", "c");
IntStream intStream = IntStream.range(1, 10);  // 1-9
```

<a id="q2"></a>
### Q2: What are intermediate and terminal operations?
**Answer:**

| Intermediate Operations | Terminal Operations |
|------------------------|---------------------|
| Return a Stream | Return non-Stream result |
| Lazy (not executed immediately) | Trigger stream processing |
| filter, map, flatMap, sorted | collect, forEach, reduce, count |

**Intermediate:**
```java
numbers.stream()
    .filter(n -> n % 2 == 0)  // keep even
    .map(n -> n * 2)          // double
    .sorted()                 // sort
    .distinct()               // remove duplicates
    .limit(5)                 // take first 5
    .skip(2);                 // skip first 2
```

**Terminal:**
```java
List<Integer> list = stream.collect(Collectors.toList());
stream.forEach(System.out::println);
int sum = stream.reduce(0, Integer::sum);
long count = stream.count();
boolean hasEven = stream.anyMatch(n -> n % 2 == 0);
Optional<Integer> first = stream.findFirst();
```

<a id="q3"></a>
### Q3: What is the difference between map() and flatMap()?
**Answer:**

| map() | flatMap() |
|-------|-----------|
| One-to-one transformation | One-to-many transformation |
| Returns single value per element | Returns stream per element, flattens result |

```java
List<String> words = Arrays.asList("hello", "world");

// map creates nested structure
List<List<String>> nested = words.stream()
    .map(word -> Arrays.asList(word.split("")))
    .collect(Collectors.toList());
// [[h, e, l, l, o], [w, o, r, l, d]]

// flatMap flattens
List<String> letters = words.stream()
    .flatMap(word -> Arrays.stream(word.split("")))
    .collect(Collectors.toList());
// [h, e, l, l, o, w, o, r, l, d]

// Optional example
Optional<String> city = findPerson(id)
    .flatMap(Person::getAddress)
    .flatMap(Address::getCity);
```

<a id="q4"></a>
### Q4: How do parallel streams work?
**Answer:**
Parallel streams use the ForkJoinPool to execute operations concurrently.

```java
numbers.parallelStream()
    .filter(n -> n % 2 == 0)
    .count();

// Or convert existing stream
numbers.stream().parallel().filter(...);
```

| Use Parallel | Avoid Parallel |
|--------------|----------------|
| Large datasets (10,000+ elements) | Small datasets |
| CPU-intensive operations | I/O operations |
| Independent operations | Stateful operations |
| Splittable data sources (ArrayList) | LinkedList |

**Pitfalls:**
```java
// ❌ DON'T: Shared mutable state
List<Integer> results = new ArrayList<>();  // NOT thread-safe!
numbers.parallelStream().forEach(results::add);  // Race condition!

// ✓ DO: Use collectors
List<Integer> results = numbers.parallelStream().collect(Collectors.toList());
```

---

## Functional Programming

<a id="q5"></a>
### Q5: What are functional interfaces in Java?
**Answer:**
A functional interface has exactly one abstract method and can be used as the target for lambda expressions.

```java
@FunctionalInterface
public interface Calculator {
    int calculate(int a, int b);  // Single abstract method
    
    default int add(int a, int b) { return a + b; }  // Default allowed
    static int multiply(int a, int b) { return a * b; }  // Static allowed
}

Calculator addition = (a, b) -> a + b;
int result = addition.calculate(5, 3);  // 8
```

<a id="q6"></a>
### Q6: What are the built-in functional interfaces?
**Answer:**

| Interface | Method | Description |
|-----------|--------|-------------|
| `Function<T,R>` | `R apply(T t)` | Transform T to R |
| `Predicate<T>` | `boolean test(T t)` | Test condition |
| `Consumer<T>` | `void accept(T t)` | Consume value |
| `Supplier<T>` | `T get()` | Supply value |
| `BiFunction<T,U,R>` | `R apply(T t, U u)` | Two inputs, one output |
| `UnaryOperator<T>` | `T apply(T t)` | Same input/output type |
| `BinaryOperator<T>` | `T apply(T t1, T t2)` | Two same types to one |

```java
Function<String, Integer> length = String::length;
Predicate<String> isLong = s -> s.length() > 5;
Consumer<String> print = System.out::println;
Supplier<LocalDateTime> now = LocalDateTime::now;
```

<a id="q7"></a>
### Q7: What are lambda expressions and method references?
**Answer:**

**Lambda expressions:**
```java
(a, b) -> a + b              // Expression body
x -> x * 2                   // Single parameter
() -> System.out.println()   // No parameters
```

**Method references:**
| Type | Syntax | Lambda Equivalent |
|------|--------|-------------------|
| Static method | `Class::staticMethod` | `x -> Class.staticMethod(x)` |
| Instance method | `object::method` | `x -> object.method(x)` |
| Arbitrary instance | `Class::method` | `(obj, x) -> obj.method(x)` |
| Constructor | `Class::new` | `x -> new Class(x)` |

```java
Function<String, Integer> parseInt = Integer::parseInt;
Function<String, String> toUpper = String::toUpperCase;
Supplier<ArrayList<String>> listSupplier = ArrayList::new;
```

<a id="q8"></a>
### Q8: What is Optional and how should you use it?
**Answer:**
`Optional<T>` is a container that may or may not contain a non-null value.

```java
Optional<String> empty = Optional.empty();
Optional<String> present = Optional.of("value");      // Throws if null
Optional<String> nullable = Optional.ofNullable(str); // Allows null
```

**Using Optional correctly:**
```java
// ✓ DO: Use functional methods
String name = user.map(User::getName).orElse("Unknown");
String name = user.map(User::getName).orElseGet(() -> generateDefault());
String name = user.map(User::getName).orElseThrow(() -> new NotFoundException());

// ✓ DO: Chain operations
String city = user.flatMap(User::getAddress).flatMap(Address::getCity).orElse("Unknown");

// ❌ DON'T: Use isPresent() + get()
if (user.isPresent()) { return user.get(); }  // Anti-pattern!

// ❌ DON'T: Use Optional for fields or parameters
```

---

## Java Version Features

<a id="q9"></a>
### Q9: What are the key features in Java 8?
**Answer:**

| Feature | Description |
|---------|-------------|
| Lambda expressions | Functional programming support |
| Stream API | Functional-style operations on collections |
| Optional | Container for nullable values |
| Default methods | Interface methods with implementation |
| Method references | Shorthand for lambdas |
| New Date/Time API | java.time package |
| CompletableFuture | Async programming |

```java
// Date/Time API
LocalDate today = LocalDate.now();
LocalDateTime dateTime = LocalDateTime.of(2024, 1, 15, 10, 30);
Duration duration = Duration.between(start, end);

// CompletableFuture
CompletableFuture.supplyAsync(() -> fetchData())
    .thenApply(data -> process(data))
    .thenAccept(result -> save(result));
```

<a id="q10"></a>
### Q10: What are the key features in Java 11?
**Answer:**

| Feature | Description |
|---------|-------------|
| `var` for lambda parameters | Type inference in lambdas |
| HTTP Client (standard) | New HTTP client API |
| String methods | isBlank(), lines(), strip(), repeat() |
| Files methods | readString(), writeString() |
| Optional.isEmpty() | Check if Optional is empty |
| Single-file execution | Run .java files directly |

```java
// New String methods
"  ".isBlank();              // true
"hello\nworld".lines();      // Stream of lines
"  hello  ".strip();         // "hello"
"abc".repeat(3);             // "abcabcabc"

// HTTP Client
HttpClient client = HttpClient.newHttpClient();
HttpRequest request = HttpRequest.newBuilder()
    .uri(URI.create("https://api.example.com/data"))
    .GET().build();
HttpResponse<String> response = client.send(request, BodyHandlers.ofString());
```

<a id="q11"></a>
### Q11: What are the key features in Java 17?
**Answer:**

| Feature | Description |
|---------|-------------|
| Sealed classes | Restrict class hierarchy |
| Records | Immutable data classes |
| Pattern matching for instanceof | Enhanced type checking |
| Switch expressions | Switch as expression |
| Text blocks | Multi-line strings |

```java
// Sealed classes
public sealed class Shape permits Circle, Rectangle, Triangle { }
public final class Circle extends Shape { }

// Records
public record Person(String name, int age) {
    public Person {
        if (age < 0) throw new IllegalArgumentException();
    }
}

// Pattern matching for instanceof
if (obj instanceof String s) {
    System.out.println(s.length());
}

// Switch expressions
int numLetters = switch (day) {
    case "MONDAY", "FRIDAY" -> 6;
    case "TUESDAY" -> 7;
    default -> throw new IllegalStateException();
};

// Text blocks
String json = """
    {
        "name": "Alice",
        "age": 30
    }
    """;
```

---

[← Back to Java Index](README.md)
