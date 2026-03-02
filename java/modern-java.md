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
Stream API provides a declarative way to process data sequences with pipeline operations.

Key characteristics:
- **Not a data structure**: it operates on a source (`Collection`, array, generator).
- **Lazy intermediate steps**: work is deferred until terminal operation.
- **Single-use**: a stream cannot be reused after terminal execution.
- **Composable pipeline**: build transformations clearly from source to result.

```java
List<String> names = List.of("Alice", "Bob", "Charlie", "David");

List<String> result = names.stream()
    .filter(name -> name.length() > 3)
    .map(String::toUpperCase)
    .sorted()
    .toList();
```

**Performance note:** Stream readability often improves business logic, but avoid over-chaining for very hot loops where plain loops may be faster and easier to profile.

<a id="q2"></a>
### Q2: What are intermediate and terminal operations?
**Answer:**
| Intermediate Operations | Terminal Operations |
|------------------------|---------------------|
| Return another Stream | Produce final result or side effect |
| Usually lazy | Trigger evaluation |
| `filter`, `map`, `flatMap`, `sorted`, `distinct` | `collect`, `reduce`, `forEach`, `count`, `findFirst` |

Important execution behavior:
- Pipelines are fused; elements typically flow through all stages one by one.
- Short-circuit terminals (`findFirst`, `anyMatch`) can stop early.

```java
boolean found = numbers.stream()
    .filter(n -> n > 10)
    .anyMatch(n -> n % 2 == 0); // may stop before scanning full stream
```

**Pitfall:** side effects in intermediate operations (`peek`, mutable shared state) make pipelines brittle and hard to parallelize safely.

<a id="q3"></a>
### Q3: What is the difference between map() and flatMap()?
**Answer:**
| `map()` | `flatMap()` |
|---------|-------------|
| One input -> one output | One input -> many outputs (stream) |
| Keeps nesting | Flattens nested streams |
| Use for simple transformation | Use for nested collections/Optionals |

```java
List<String> words = List.of("hello", "world");

List<List<String>> nested = words.stream()
    .map(w -> Arrays.asList(w.split("")))
    .toList();

List<String> flattened = words.stream()
    .flatMap(w -> Arrays.stream(w.split("")))
    .toList();
```

Optional chaining example:
```java
Optional<String> city = findPerson(id)
    .flatMap(Person::getAddress)
    .flatMap(Address::getCity);
```

<a id="q4"></a>
### Q4: How do parallel streams work?
**Answer:**
Parallel streams split work across threads (typically `ForkJoinPool.commonPool()`).

```java
long count = numbers.parallelStream()
    .filter(n -> n % 2 == 0)
    .count();
```

Use parallel streams when:
- dataset is large
- operations are CPU-bound and stateless
- source splits efficiently (`ArrayList`, arrays)

Avoid when:
- tasks are I/O-bound or blocking
- data is small
- operations depend on shared mutable state or strict ordering

```java
// Unsafe: race condition
List<Integer> out = new ArrayList<>();
numbers.parallelStream().forEach(out::add);

// Safe
List<Integer> safe = numbers.parallelStream().toList();
```

**Operational caveat:** using common pool in app servers can cause contention with unrelated parallel tasks.

---

## Functional Programming

<a id="q5"></a>
### Q5: What are functional interfaces in Java?
**Answer:**
A functional interface has exactly one abstract method (SAM: Single Abstract Method) and is target type for lambdas/method references.

```java
@FunctionalInterface
public interface Calculator {
    int calculate(int a, int b);

    default int add(int a, int b) { return a + b; }
    static int multiply(int a, int b) { return a * b; }
}
```

Rules:
- `default` and `static` methods do not break SAM requirement.
- Methods from `Object` (`toString`, `equals`) are ignored for SAM counting.

**Design tip:** functional interfaces are ideal for behavior injection (callbacks, policies, predicates).

<a id="q6"></a>
### Q6: What are the built-in functional interfaces?
**Answer:**
Core interfaces:

| Interface | Method | Purpose |
|-----------|--------|---------|
| `Function<T,R>` | `R apply(T t)` | transform |
| `Predicate<T>` | `boolean test(T t)` | condition test |
| `Consumer<T>` | `void accept(T t)` | side-effect consume |
| `Supplier<T>` | `T get()` | lazy value supply |
| `BiFunction<T,U,R>` | `R apply(T t, U u)` | two-input transform |
| `UnaryOperator<T>` | `T apply(T t)` | same-type transform |
| `BinaryOperator<T>` | `T apply(T a, T b)` | same-type combine |

Primitive specializations reduce boxing cost:
- `IntFunction`, `LongSupplier`, `DoublePredicate`, `IntUnaryOperator`, etc.

```java
IntUnaryOperator square = x -> x * x;
int value = square.applyAsInt(7);
```

<a id="q7"></a>
### Q7: What are lambda expressions and method references?
**Answer:**
**Lambda** is an inline anonymous function with target typing from context.

```java
BiFunction<Integer, Integer, Integer> add = (a, b) -> a + b;
Predicate<String> nonEmpty = s -> !s.isBlank();
```

**Method references** are shorthand lambdas when you only delegate to an existing method:

| Type | Syntax | Equivalent |
|------|--------|------------|
| Static | `Class::staticMethod` | `x -> Class.staticMethod(x)` |
| Bound instance | `obj::method` | `x -> obj.method(x)` |
| Unbound instance | `Class::method` | `(obj, x) -> obj.method(x)` |
| Constructor | `Class::new` | `() -> new Class()` |

```java
Function<String, Integer> parse = Integer::parseInt;
Supplier<ArrayList<String>> createList = ArrayList::new;
```

**Pitfall:** captured local variables must be effectively final.

<a id="q8"></a>
### Q8: What is Optional and how should you use it?
**Answer:**
`Optional<T>` models "value may be absent" explicitly.

```java
Optional<String> maybeName = Optional.ofNullable(rawName);
String name = maybeName.filter(s -> !s.isBlank()).orElse("Unknown");
```

Good usage:
- return type for methods where absence is expected
- fluent transformations with `map`, `flatMap`, `filter`
- fail-fast with `orElseThrow`

Avoid:
- Optional fields in entities/DTOs
- Optional parameters
- `isPresent()` + `get()` imperative style

```java
// Better than isPresent/get:
User user = repo.findById(id).orElseThrow(NotFoundException::new);
```

**Performance note:** Optional improves API clarity; do not overuse inside tight loops or primitive-heavy code paths.

---

## Java Version Features

<a id="q9"></a>
### Q9: What are the key features in Java 8?
**Answer:**
Major Java 8 additions that changed everyday coding style:

| Feature | Why it matters |
|---------|----------------|
| Lambdas + functional interfaces | concise behavior passing |
| Stream API | declarative collection processing |
| Optional | explicit nullability handling |
| Default methods | evolve interfaces safely |
| New Date/Time API (`java.time`) | immutable, timezone-correct date handling |
| CompletableFuture | async composition |

```java
CompletableFuture.supplyAsync(this::fetchData)
    .thenApply(this::transform)
    .thenAccept(this::save);
```

**Migration caveat:** Streams and CompletableFuture improve expressiveness but can hide complexity if error handling and thread execution context are not explicit.

<a id="q10"></a>
### Q10: What are the key features in Java 11?
**Answer:**
Java 11 (LTS) stabilizes many post-Java-8 improvements:

| Feature | Description |
|---------|-------------|
| Standard `HttpClient` | sync + async HTTP with HTTP/2 support |
| `var` in lambda params | cleaner lambda declarations with annotations |
| String APIs | `isBlank`, `lines`, `strip`, `repeat` |
| File APIs | `Files.readString`, `Files.writeString` |
| `Optional.isEmpty()` | more readable optional checks |
| Single-file source execution | `java MyScript.java` |

```java
HttpClient client = HttpClient.newHttpClient();
HttpRequest request = HttpRequest.newBuilder(URI.create("https://example.com")).GET().build();
HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());
```

**Enterprise impact:** Java 11 is a long-term baseline for many production systems due to LTS support and modern runtime improvements.

<a id="q11"></a>
### Q11: What are the key features in Java 17?
**Answer:**
Java 17 (LTS) introduces stronger modeling and language ergonomics:

| Feature | Benefit |
|---------|---------|
| Records | concise immutable data carriers |
| Sealed classes | controlled inheritance |
| Pattern matching for `instanceof` | less casting boilerplate |
| Text blocks | cleaner multiline strings |
| Switch expressions (standardized earlier, widely used in 17-era code) | expression-oriented control flow |

```java
public sealed interface Shape permits Circle, Rectangle {}
public record Circle(double radius) implements Shape {}
public record Rectangle(double width, double height) implements Shape {}

double area(Shape s) {
    return switch (s) {
        case Circle c -> Math.PI * c.radius() * c.radius();
        case Rectangle r -> r.width() * r.height();
    };
}
```

**Design takeaway:** records + sealed classes + pattern matching encourage algebraic-style domain modeling with safer, clearer code.

---

[← Back to Java Index](README.md)
