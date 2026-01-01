# Java Core Fundamentals

## Table of Contents

### OOP (Object-Oriented Programming)
- [Q1: What are the four pillars of OOP?](#q1)
- [Q2: What is the difference between method overloading and method overriding?](#q2)
- [Q3: What is the difference between == and .equals()?](#q3)

### Interfaces and Abstract Classes
- [Q4: What is the difference between Interface and Abstract Class?](#q4)
- [Q5: When would you use an interface vs an abstract class?](#q5)
- [Q6: What are default methods in interfaces (Java 8+)?](#q6)

### Keywords
- [Q7: What is the static keyword used for?](#q7)
- [Q8: What are the uses of the final keyword?](#q8)
- [Q9: What is the difference between final, finally, and finalize()?](#q9)
- [Q10: What does the volatile keyword do?](#q10)

### Generics
- [Q11: What are generics and why use them?](#q11)
- [Q12: What is type erasure?](#q12)
- [Q13: What are bounded type parameters and wildcards?](#q13)
- [Q14: What is the difference between <? extends T> and <? super T>?](#q14)
- [Q15: What is the difference between bounded type parameter and bounded wildcard?](#q15)
- [Q16: When should we use bounded type parameter vs bounded wildcard?](#q16)

### Exception Handling
- [Q17: What is the difference between checked and unchecked exceptions?](#q17)
- [Q18: What is the difference between throw and throws?](#q18)
- [Q19: What is exception chaining?](#q19)
- [Q20: How do you handle multiple exceptions?](#q20)
- [Q21: What is exception propagation?](#q21)

### Serialization
- [Q22: What is serialization and deserialization?](#q22)
- [Q23: What is the transient keyword?](#q23)
- [Q24: What is serialVersionUID?](#q24)
- [Q25: What is the difference between Serializable and Externalizable?](#q25)

### Immutability
- [Q26: What is an immutable class and how do you create one?](#q26)
- [Q27: Why is String immutable in Java?](#q27)

### OOP Principles
- [Q28: What are the SOLID principles?](#q28)
- [Q29: What are the GRASP principles?](#q29)

---

## OOP (Object-Oriented Programming)

<a id="q1"></a>
### Q1: What are the four pillars of OOP?
**Answer:**
1. **Encapsulation** - Bundling data and methods that operate on that data within a single unit (class), and restricting direct access to some components using access modifiers (private, protected, public).
2. **Inheritance** - A mechanism where a new class inherits properties and behaviors from an existing class, promoting code reuse.
3. **Polymorphism** - The ability of objects to take many forms. Includes:
   - *Compile-time (Method Overloading)*: Same method name, different parameters
   - *Runtime (Method Overriding)*: Subclass provides specific implementation of parent method
4. **Abstraction** - Hiding complex implementation details and showing only necessary features of an object.

<a id="q2"></a>
### Q2: What is the difference between method overloading and method overriding?
**Answer:**
| Overloading | Overriding |
|-------------|------------|
| Same method name, different parameters | Same method name and parameters |
| Compile-time polymorphism | Runtime polymorphism |
| Can occur within same class | Occurs between parent and child class |
| Return type can be different | Return type must be same or covariant |
| Access modifier can be any | Cannot be more restrictive |

<a id="q3"></a>
### Q3: What is the difference between `==` and `.equals()`?
**Answer:**
- `==` compares **reference equality** (whether two references point to the same object in memory)
- `.equals()` compares **value equality** (whether two objects are logically equivalent)

```java
String s1 = new String("hello");
String s2 = new String("hello");
System.out.println(s1 == s2);      // false (different objects)
System.out.println(s1.equals(s2)); // true (same content)
```

---

## Interfaces and Abstract Classes

<a id="q4"></a>
### Q4: What is the difference between Interface and Abstract Class?
**Answer:**
| Interface | Abstract Class |
|-----------|----------------|
| Only abstract methods (before Java 8) | Can have abstract and concrete methods |
| Variables are public static final | Can have any type of variables |
| Supports multiple inheritance | Single inheritance only |
| Implements keyword | Extends keyword |
| No constructor | Can have constructor |
| Java 8+: default and static methods | Always had this capability |

<a id="q5"></a>
### Q5: When would you use an interface vs an abstract class?
**Answer:**
**Use Interface when:**
- You need multiple inheritance
- You want to define a contract without implementation
- Unrelated classes need to implement the same behavior

**Use Abstract Class when:**
- You want to share code among related classes
- You need non-public members
- You need non-static, non-final fields
- You want to provide a common base implementation

<a id="q6"></a>
### Q6: What are default methods in interfaces (Java 8+)?
**Answer:**
Default methods allow interfaces to have method implementations:
```java
public interface Vehicle {
    void start();
    
    default void honk() {
        System.out.println("Beep beep!");
    }
}
```
- Allows adding new methods to interfaces without breaking existing implementations
- Can be overridden by implementing classes
- Solves the "diamond problem" through explicit resolution

---

## Keywords

<a id="q7"></a>
### Q7: What is the `static` keyword used for?
**Answer:**
- **Static variable**: Shared across all instances of a class (class-level variable)
- **Static method**: Can be called without creating an instance, cannot access non-static members
- **Static block**: Executes once when class is loaded, used for initialization
- **Static nested class**: Nested class that doesn't need reference to outer class instance

<a id="q8"></a>
### Q8: What are the uses of the `final` keyword?
**Answer:**
- **Final variable**: Value cannot be changed once assigned (constant)
- **Final method**: Cannot be overridden by subclasses
- **Final class**: Cannot be inherited (e.g., String, Integer)
- **Final parameter**: Parameter cannot be modified within the method

<a id="q9"></a>
### Q9: What is the difference between `final`, `finally`, and `finalize()`?
**Answer:**
| final | finally | finalize() |
|-------|---------|------------|
| Keyword for constants | Block in try-catch | Method in Object class |
| Prevents modification | Always executes | Called before GC |
| Used with variable/method/class | Used for cleanup code | Deprecated in Java 9+ |

<a id="q10"></a>
### Q10: What does the `volatile` keyword do?
**Answer:**
- Ensures visibility of changes across threads
- Reads/writes go directly to main memory (not CPU cache)
- Prevents instruction reordering for that variable
- Does NOT provide atomicity (use `synchronized` or `Atomic` classes for that)

---

## Generics

<a id="q11"></a>
### Q11: What are generics and why use them?
**Answer:**
Generics enable type-safe collections and methods, catching type errors at compile time.

**Benefits:**
1. **Type safety** - Compile-time type checking
2. **Elimination of casts** - No explicit casting needed
3. **Code reuse** - Write generic algorithms once

```java
// Without generics (pre-Java 5)
List list = new ArrayList();
list.add("Hello");
list.add(123);  // No compile error!
String str = (String) list.get(1);  // Runtime ClassCastException!

// With generics
List<String> list = new ArrayList<>();
list.add("Hello");
list.add(123);  // COMPILE ERROR!
String str = list.get(0);  // No cast needed
```

**Generic class:**
```java
public class Box<T> {
    private T content;
    public void set(T content) { this.content = content; }
    public T get() { return content; }
}

Box<String> stringBox = new Box<>();
stringBox.set("Hello");
```

**Generic method:**
```java
public static <T> T getFirst(List<T> list) {
    return list.isEmpty() ? null : list.get(0);
}
```

<a id="q12"></a>
### Q12: What is type erasure?
**Answer:**
Type erasure removes generic type information at compile time, replacing it with bounds or Object.

```java
// Source code
public class Box<T> {
    private T value;
    public T get() { return value; }
}

// After type erasure (bytecode)
public class Box {
    private Object value;
    public Object get() { return value; }
}

// Source code with bounds
public class Box<T extends Number> {
    private T value;
}

// After erasure - erased to bound
public class Box {
    private Number value;
}
```

**Implications:**
```java
// Cannot do these due to erasure:
T[] array = new T[10];  // COMPILE ERROR - cannot create generic arrays
if (obj instanceof List<String>) { }  // COMPILE ERROR
T instance = new T();  // COMPILE ERROR - cannot create instances
```

<a id="q13"></a>
### Q13: What are bounded type parameters and wildcards?
**Answer:**

**Bounded type parameters:**
```java
// Upper bound - T must be Number or subclass
public class Calculator<T extends Number> {
    public double sum(T a, T b) {
        return a.doubleValue() + b.doubleValue();
    }
}

// Multiple bounds
public class Demo<T extends Comparable<T> & Serializable> { }
```

**Wildcards:**
```java
// Unbounded wildcard - unknown type
public void printList(List<?> list) {
    for (Object item : list) {
        System.out.println(item);
    }
}

// Upper bounded - ? extends Type (read-only)
public double sumNumbers(List<? extends Number> list) {
    double sum = 0;
    for (Number n : list) {
        sum += n.doubleValue();
    }
    return sum;
}

// Lower bounded - ? super Type (write-only)
public void addIntegers(List<? super Integer> list) {
    list.add(1);
    list.add(2);
}
```

<a id="q14"></a>
### Q14: What is the difference between <? extends T> and <? super T>?
**Answer:**

| `<? extends T>` (Upper Bound) | `<? super T>` (Lower Bound) |
|-------------------------------|----------------------------|
| T or any subtype | T or any supertype |
| Producer (read from) | Consumer (write to) |
| Can get as T | Can add T |
| Cannot add (except null) | Cannot get as T |

**PECS: Producer Extends, Consumer Super**

```java
// Producer - use extends (read from)
public void processAnimals(List<? extends Animal> animals) {
    for (Animal a : animals) {  // Can read as Animal
        a.makeSound();
    }
    animals.add(new Dog());  // COMPILE ERROR - can't add
}

// Consumer - use super (write to)
public void addDogs(List<? super Dog> list) {
    list.add(new Dog());  // Can add Dog
    Dog d = list.get(0);  // COMPILE ERROR - can only get as Object
}

// Practical example: Collections.copy()
public static <T> void copy(List<? super T> dest, List<? extends T> src) {
    for (T item : src) {  // Read from src (extends)
        dest.add(item);   // Write to dest (super)
    }
}
```

<a id="q15"></a>
### Q15: What is the difference between bounded type parameter and bounded wildcard?
**Answer:**

| Bounded Type Parameter `<T extends X>` | Bounded Wildcard `<? extends X>` |
|----------------------------------------|----------------------------------|
| Declares a new type variable T | Anonymous placeholder, no new type |
| Can be used throughout the class/method | Only used at specific usage site |
| Establishes relationship between parameters | No relationship, each `?` is independent |
| Can add constraints (multiple bounds) | Single bound only |
| Type is known and reusable | Type is unknown |

```java
// Bounded type: T establishes relationship
public <T extends Number> void copyNumbers(List<T> src, List<T> dest) {
    for (T item : src) {
        dest.add(item);  // Works! Both are List<T>
    }
}

// Wildcard: No relationship, each ? is independent
public void copyWildcard(List<? extends Number> src, List<? extends Number> dest) {
    for (Number item : src) {
        dest.add(item);  // ERROR! dest might be List<Integer> while item is Double
    }
}
```

<a id="q16"></a>
### Q16: When should we use bounded type parameter vs bounded wildcard?
**Answer:**

**Use bounded type parameter `<T extends X>` when:**
- You need to use the type in multiple places (return type, multiple parameters)
- You need to establish relationships between parameters
- You need multiple bounds (`<T extends A & B>`)

```java
public <T extends Comparable<T>> T findMax(List<T> list) {
    return Collections.max(list);
}
```

**Use bounded wildcard when:**
- You only need to read from a collection (`extends`)
- You only need to write to a collection (`super`)
- The type is only used once

```java
public double sumAll(List<? extends Number> numbers) {
    return numbers.stream().mapToDouble(Number::doubleValue).sum();
}
```

---

## Exception Handling

<a id="q17"></a>
### Q17: What is the difference between checked and unchecked exceptions?
**Answer:**

| Checked Exceptions | Unchecked Exceptions |
|--------------------|---------------------|
| Compile-time checking | Runtime checking |
| Must be caught or declared | Optional handling |
| Extends Exception | Extends RuntimeException |
| Recoverable situations | Programming errors |
| IOException, SQLException | NullPointerException, IllegalArgumentException |

```
Throwable
├── Error (unchecked) - OutOfMemoryError, StackOverflowError
└── Exception
    ├── RuntimeException (unchecked) - NullPointerException, etc.
    └── Checked Exceptions - IOException, SQLException, etc.
```

<a id="q18"></a>
### Q18: What is the difference between throw and throws?
**Answer:**

| throw | throws |
|-------|--------|
| Throws an exception object | Declares possible exceptions |
| Used inside method body | Used in method signature |
| Can throw one exception | Can declare multiple exceptions |

```java
// throws - declaration in method signature
public void readFile(String path) throws IOException, FileNotFoundException {
    // Method might throw these exceptions
}

// throw - actually throwing an exception
public void validateAge(int age) {
    if (age < 0) {
        throw new IllegalArgumentException("Age cannot be negative");
    }
}
```

<a id="q19"></a>
### Q19: What is exception chaining?
**Answer:**
Exception chaining wraps one exception inside another, preserving the original cause.

```java
public void processFile(String path) throws ServiceException {
    try {
        readFile(path);
    } catch (IOException e) {
        // Chain: wrap low-level exception in higher-level one
        throw new ServiceException("Failed to process file: " + path, e);
    }
}

// Accessing the chain
try {
    processFile("data.txt");
} catch (ServiceException e) {
    Throwable cause = e.getCause();  // Get original IOException
}
```

<a id="q20"></a>
### Q20: How do you handle multiple exceptions?
**Answer:**

**Multi-catch (Java 7+):**
```java
try {
    processFile(path);
} catch (IOException | SQLException e) {
    logger.error("Error processing: " + e.getMessage());
}
```

**Try-with-resources (Java 7+):**
```java
try (FileReader reader = new FileReader(path);
     BufferedReader br = new BufferedReader(reader)) {
    return br.readLine();
} catch (IOException e) {
    throw new ServiceException("Failed to read", e);
}
// reader and br automatically closed
```

<a id="q21"></a>
### Q21: What is exception propagation?
**Answer:**
Exception propagation is how exceptions travel up the call stack until caught.

```java
public void method1() {
    method2();  // Exception propagates here if not caught
}

public void method2() {
    method3();  // Exception propagates here if not caught
}

public void method3() {
    throw new RuntimeException("Error!");  // Exception thrown
}

// Call stack: method3() → method2() → method1() → main()
// Exception travels up until caught
```

---

## Serialization

<a id="q22"></a>
### Q22: What is serialization and deserialization?
**Answer:**
- **Serialization**: Converting an object into a byte stream
- **Deserialization**: Reconstructing an object from a byte stream

```java
public class Person implements Serializable {
    private String name;
    private int age;
}

// Serialization
try (ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("person.ser"))) {
    oos.writeObject(person);
}

// Deserialization
try (ObjectInputStream ois = new ObjectInputStream(new FileInputStream("person.ser"))) {
    Person restored = (Person) ois.readObject();
}
```

<a id="q23"></a>
### Q23: What is the transient keyword?
**Answer:**
`transient` marks fields that should NOT be serialized.

```java
public class User implements Serializable {
    private String username;
    private transient String password;  // Won't be serialized
    private transient Connection dbConnection;  // Non-serializable anyway
}
```

<a id="q24"></a>
### Q24: What is serialVersionUID?
**Answer:**
`serialVersionUID` is a version identifier for serialized classes, used to verify compatibility.

```java
public class Person implements Serializable {
    private static final long serialVersionUID = 1L;  // Explicit version
    private String name;
}
```

Without explicit serialVersionUID, JVM generates one based on class structure - any change generates different UID.

<a id="q25"></a>
### Q25: What is the difference between Serializable and Externalizable?
**Answer:**

| Serializable | Externalizable |
|--------------|----------------|
| Marker interface (no methods) | Has writeExternal/readExternal |
| JVM handles serialization | You control serialization |
| Slower (uses reflection) | Faster (no reflection) |
| Serializes all non-transient fields | You choose what to serialize |
| Default constructor not required | Public no-arg constructor required |

---

## Immutability

<a id="q26"></a>
### Q26: What is an immutable class and how do you create one?
**Answer:**
An immutable class is one whose state cannot be modified after creation.

**Rules to create:**
1. Declare class as `final`
2. Make all fields `private` and `final`
3. No setter methods
4. Initialize all fields via constructor
5. Return defensive copies for mutable objects

```java
public final class ImmutablePerson {
    private final String name;
    private final List<String> hobbies;
    
    public ImmutablePerson(String name, List<String> hobbies) {
        this.name = name;
        this.hobbies = new ArrayList<>(hobbies); // Defensive copy
    }
    
    public List<String> getHobbies() {
        return Collections.unmodifiableList(hobbies); // Defensive copy
    }
}
```

<a id="q27"></a>
### Q27: Why is String immutable in Java?
**Answer:**
1. **String Pool**: Allows string interning, saving memory
2. **Security**: Parameters in network connections, file paths can't be changed
3. **Thread Safety**: Inherently thread-safe without synchronization
4. **Caching**: hashCode can be cached since it won't change
5. **Class Loading**: Prevents malicious class loading

---

## OOP Principles

<a id="q28"></a>
### Q28: What are the SOLID principles?
**Answer:**

| Principle | Description |
|-----------|-------------|
| **S**ingle Responsibility | A class should have only one reason to change |
| **O**pen/Closed | Open for extension, closed for modification |
| **L**iskov Substitution | Subtypes must be substitutable for base types |
| **I**nterface Segregation | Many specific interfaces > one general interface |
| **D**ependency Inversion | Depend on abstractions, not concretions |

```java
// Single Responsibility - bad
class Employee {
    void calculatePay() { }
    void saveToDatabase() { }
    void generateReport() { }
}

// Single Responsibility - good
class Employee { }
class PayrollCalculator { void calculate(Employee e) { } }
class EmployeeRepository { void save(Employee e) { } }

// Dependency Inversion - good
interface Database { void save(Order order); }
class OrderService {
    private Database db;
    OrderService(Database db) { this.db = db; }
}
```

<a id="q29"></a>
### Q29: What are the GRASP principles?
**Answer:**
GRASP (General Responsibility Assignment Software Patterns) guide object design.

| Principle | Description |
|-----------|-------------|
| **Information Expert** | Assign responsibility to the class with the information |
| **Creator** | Assign creation to class that contains/aggregates/uses |
| **Controller** | Assign system event handling to a controller class |
| **Low Coupling** | Minimize dependencies between classes |
| **High Cohesion** | Keep related functionality together |
| **Polymorphism** | Use polymorphism for varying behavior |
| **Pure Fabrication** | Create classes that don't represent domain concepts |
| **Indirection** | Introduce intermediate objects to reduce coupling |
| **Protected Variations** | Shield from variations using interfaces |

---

[← Back to Java Index](README.md)
