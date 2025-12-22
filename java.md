# Java Developer Interview Questions & Answers

---

## Table of Contents

### 1. [Java Core](#java-core)

#### [OOP (Object-Oriented Programming)](#oop-object-oriented-programming)
- [Q1: What are the four pillars of OOP?](#q1)
- [Q2: What is the difference between method overloading and method overriding?](#q2)
- [Q3: What is the difference between == and .equals()?](#q3)

#### [Data Structures (Lists, Maps, Sets, Collections)](#data-structures-lists-maps-sets-collections)
- [Q4: What are the main differences between ArrayList and LinkedList?](#q4)
- [Q5: What is the difference between HashMap and TreeMap?](#q5)
- [Q6: What is the difference between HashSet, LinkedHashSet, and TreeSet?](#q6)
- [Q7: What happens when two keys have the same hashCode in HashMap?](#q7)
- [Q8: Why should you override both hashCode() and equals()?](#q8)

#### [Keywords (static, final, etc.)](#keywords-static-final-etc)
- [Q9: What is the static keyword used for?](#q9)
- [Q10: What are the uses of the final keyword?](#q10)
- [Q11: What is the difference between final, finally, and finalize()?](#q11)
- [Q12: What does the volatile keyword do?](#q12)

#### [Interfaces and Abstract Classes](#interfaces-and-abstract-classes)
- [Q13: What is the difference between Interface and Abstract Class?](#q13)
- [Q14: When would you use an interface vs an abstract class?](#q14)
- [Q15: What are default methods in interfaces (Java 8+)?](#q15)

#### [Garbage Collection (GC)](#garbage-collection-gc)
- [Q16: How does Garbage Collection work in Java?](#q16)
- [Q17: What are the different types of Garbage Collectors?](#q17)
- [Q18: What makes an object eligible for garbage collection?](#q18)
- [Q19: What is the difference between Minor GC and Major GC?](#q19)
- [Q20: What are GC Roots?](#q20)
- [Q21: What is Stop-the-World (STW) pause?](#q21)
- [Q22: What are Weak, Soft, and Phantom References?](#q22)
- [Q23: How do you tune and monitor Garbage Collection?](#q23)

#### [Memory (Heap and Stack)](#memory-heap-and-stack)
- [Q24: What is the difference between Heap and Stack memory?](#q24)
- [Q25: What is a memory leak in Java and how can you prevent it?](#q25)
- [Q26: What is the difference between StackOverflowError and OutOfMemoryError?](#q26)

#### [Multithreading](#multithreading)
- [Q27: What is the difference between Runnable and Callable?](#q27)
- [Q28: What is the difference between synchronized and volatile?](#q28)
- [Q29: What are Thread Pools and why use them?](#q29)
- [Q30: What is a deadlock and how can you prevent it?](#q30)
- [Q31: What is the difference between wait() and sleep()?](#q31)

#### [Immutable Classes](#immutable-classes)
- [Q32: What is an immutable class and how do you create one?](#q32)
- [Q33: Why is String immutable in Java?](#q33)

#### [Atomic Classes](#atomic-classes)
- [Q34: What are Atomic classes and when should you use them?](#q34)
- [Q35: What is Compare-And-Swap (CAS)?](#q35)

---

# Java Core

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

## Data Structures (Lists, Maps, Sets, Collections)

<a id="q4"></a>
### Q4: What are the main differences between ArrayList and LinkedList?
**Answer:**
| ArrayList | LinkedList |
|-----------|------------|
| Backed by dynamic array | Backed by doubly-linked list |
| O(1) random access | O(n) random access |
| O(n) insertion/deletion in middle | O(1) insertion/deletion (if node known) |
| Better for read-heavy operations | Better for write-heavy operations |
| Less memory overhead | More memory (stores next/prev pointers) |

<a id="q5"></a>
### Q5: What is the difference between HashMap and TreeMap?
**Answer:**
| HashMap | TreeMap |
|---------|---------|
| O(1) for get/put operations | O(log n) for get/put operations |
| No ordering guaranteed | Keys are sorted in natural order |
| Allows one null key | Does not allow null keys |
| Uses hashing | Uses Red-Black tree |
| Better for most use cases | Use when sorted order is needed |

<a id="q6"></a>
### Q6: What is the difference between HashSet, LinkedHashSet, and TreeSet?
**Answer:**
- **HashSet**: No ordering, O(1) operations, allows null
- **LinkedHashSet**: Maintains insertion order, O(1) operations
- **TreeSet**: Sorted order, O(log n) operations, no null allowed

<a id="q7"></a>
### Q7: What happens when two keys have the same hashCode in HashMap?
**Answer:**
When two keys have the same hashCode (collision):
1. Both entries are stored in the same bucket
2. Before Java 8: Uses LinkedList to chain entries
3. Java 8+: Uses LinkedList initially, converts to balanced tree (Red-Black) when bucket size exceeds 8 (TREEIFY_THRESHOLD)
4. To find correct value, `equals()` is called on each entry in the bucket

<a id="q8"></a>
### Q8: Why should you override both hashCode() and equals()?
**Answer:**
The contract states:
- If two objects are equal (`equals()` returns true), they **must** have the same hashCode
- If hashCodes are equal, objects may or may not be equal

If you override only `equals()`:
```java
Map<Person, String> map = new HashMap<>();
Person p1 = new Person("John");
Person p2 = new Person("John");
map.put(p1, "Engineer");
map.get(p2); // Returns null! Different hashCode, wrong bucket
```

---

## Keywords (static, final, etc.)

<a id="q9"></a>
### Q9: What is the `static` keyword used for?
**Answer:**
- **Static variable**: Shared across all instances of a class (class-level variable)
- **Static method**: Can be called without creating an instance, cannot access non-static members
- **Static block**: Executes once when class is loaded, used for initialization
- **Static nested class**: Nested class that doesn't need reference to outer class instance

<a id="q10"></a>
### Q10: What are the uses of the `final` keyword?
**Answer:**
- **Final variable**: Value cannot be changed once assigned (constant)
- **Final method**: Cannot be overridden by subclasses
- **Final class**: Cannot be inherited (e.g., String, Integer)
- **Final parameter**: Parameter cannot be modified within the method

<a id="q11"></a>
### Q11: What is the difference between `final`, `finally`, and `finalize()`?
**Answer:**
| final | finally | finalize() |
|-------|---------|------------|
| Keyword for constants | Block in try-catch | Method in Object class |
| Prevents modification | Always executes | Called before GC |
| Used with variable/method/class | Used for cleanup code | Deprecated in Java 9+ |

<a id="q12"></a>
### Q12: What does the `volatile` keyword do?
**Answer:**
- Ensures visibility of changes across threads
- Reads/writes go directly to main memory (not CPU cache)
- Prevents instruction reordering for that variable
- Does NOT provide atomicity (use `synchronized` or `Atomic` classes for that)

---

## Interfaces and Abstract Classes

<a id="q13"></a>
### Q13: What is the difference between Interface and Abstract Class?
**Answer:**
| Interface | Abstract Class |
|-----------|----------------|
| Only abstract methods (before Java 8) | Can have abstract and concrete methods |
| Variables are public static final | Can have any type of variables |
| Supports multiple inheritance | Single inheritance only |
| Implements keyword | Extends keyword |
| No constructor | Can have constructor |
| Java 8+: default and static methods | Always had this capability |

<a id="q14"></a>
### Q14: When would you use an interface vs an abstract class?
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

<a id="q15"></a>
### Q15: What are default methods in interfaces (Java 8+)?
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

## Garbage Collection (GC)

<a id="q16"></a>
### Q16: How does Garbage Collection work in Java?
**Answer:**
1. **Mark Phase**: GC identifies all reachable objects starting from GC roots (stack variables, static variables, JNI references)
2. **Sweep Phase**: Unreachable objects are removed from memory
3. **Compact Phase** (optional): Memory is defragmented

**Generational GC:**
- **Young Generation**: New objects, frequent GC (Minor GC)
  - Eden Space: Where new objects are created
  - Survivor Spaces (S0, S1): Objects that survived Minor GC
- **Old Generation**: Long-lived objects, less frequent GC (Major GC)
- **Metaspace**: Class metadata (replaced PermGen in Java 8)

<a id="q17"></a>
### Q17: What are the different types of Garbage Collectors?
**Answer:**
1. **Serial GC**: Single-threaded, for small applications (`-XX:+UseSerialGC`)
2. **Parallel GC**: Multi-threaded, for medium-large apps (`-XX:+UseParallelGC`)
3. **G1 GC**: Default since Java 9, divides heap into regions (`-XX:+UseG1GC`)
4. **ZGC**: Low latency (<10ms pauses), for large heaps (`-XX:+UseZGC`)
5. **Shenandoah**: Low pause times, concurrent compaction

<a id="q18"></a>
### Q18: What makes an object eligible for garbage collection?
**Answer:**
An object becomes eligible when:
1. Object reference is set to `null`
2. Object reference is reassigned
3. Object is created inside a method (local scope)
4. Island of isolation (circular references with no external references)

<a id="q19"></a>
### Q19: What is the difference between Minor GC and Major GC?
**Answer:**
| Minor GC | Major GC (Full GC) |
|----------|-------------------|
| Cleans Young Generation | Cleans Old Generation (and sometimes Young) |
| Very fast (milliseconds) | Slower (can be seconds) |
| Frequent | Less frequent |
| Uses copying algorithm | Uses mark-sweep-compact |
| Short STW pause | Longer STW pause |

**When they occur:**
- **Minor GC**: When Eden space is full
- **Major GC**: When Old Generation is full, or explicitly triggered via `System.gc()`

```
Young Gen [Eden | S0 | S1] → Minor GC
Old Gen [Tenured]          → Major GC
```

<a id="q20"></a>
### Q20: What are GC Roots?
**Answer:**
GC Roots are the starting points for the garbage collector to determine which objects are reachable (alive).

**Types of GC Roots:**
1. **Local variables** in stack frames of active threads
2. **Active threads** themselves
3. **Static variables** of loaded classes
4. **JNI references** (native code references)
5. **Synchronized monitors** (objects used in synchronized blocks)
6. **Class loaders** and classes they've loaded

```java
public class Example {
    private static Object staticRef;  // GC Root (static)
    
    public void method() {
        Object localRef = new Object();  // GC Root (local variable)
        // localRef is reachable from stack
    }
}
```

<a id="q21"></a>
### Q21: What is Stop-the-World (STW) pause?
**Answer:**
STW pause is when the JVM halts all application threads to perform garbage collection safely.

**Why it happens:**
- GC needs a consistent view of the heap
- Objects can't move while application is modifying them
- Required for marking and compacting phases

**Impact:**
- Application becomes unresponsive during STW
- Critical for latency-sensitive applications

**Minimizing STW:**
| Collector | STW Behavior |
|-----------|--------------|
| Serial GC | Long STW pauses |
| Parallel GC | Shorter but still noticeable |
| G1 GC | Predictable, configurable pause targets |
| ZGC | Sub-millisecond pauses (<10ms) |
| Shenandoah | Concurrent, minimal STW |

```bash
# Set max pause time target for G1
-XX:MaxGCPauseMillis=200
```

<a id="q22"></a>
### Q22: What are Weak, Soft, and Phantom References?
**Answer:**
Java provides reference types for fine-grained control over garbage collection:

| Reference Type | Collected When | Use Case |
|---------------|----------------|----------|
| **Strong** | Never (while reachable) | Normal references |
| **Soft** | When memory is low | Caches |
| **Weak** | Next GC cycle | Canonicalizing mappings |
| **Phantom** | After finalization | Pre-mortem cleanup |

```java
// Strong reference - normal
Object strong = new Object();

// Soft reference - cleared when memory is low
SoftReference<Object> soft = new SoftReference<>(new Object());

// Weak reference - cleared on next GC
WeakReference<Object> weak = new WeakReference<>(new Object());

// Phantom reference - for cleanup actions
PhantomReference<Object> phantom = new PhantomReference<>(obj, referenceQueue);
```

**Practical uses:**
- **WeakHashMap**: Keys are weak references, auto-removed when key is GC'd
- **SoftReference**: Memory-sensitive caches
- **PhantomReference**: Resource cleanup (better than finalize)

<a id="q23"></a>
### Q23: How do you tune and monitor Garbage Collection?
**Answer:**

**Common JVM flags:**
```bash
# Choose GC algorithm
-XX:+UseG1GC              # G1 (default Java 9+)
-XX:+UseZGC               # ZGC (low latency)
-XX:+UseParallelGC        # Parallel (throughput)

# Heap sizing
-Xms4g                    # Initial heap size
-Xmx4g                    # Maximum heap size
-XX:NewRatio=2            # Old/Young ratio

# G1 specific
-XX:MaxGCPauseMillis=200  # Target pause time
-XX:G1HeapRegionSize=16m  # Region size

# GC logging (Java 9+)
-Xlog:gc*:file=gc.log:time,uptime:filecount=5,filesize=10m
```

**Monitoring tools:**
1. **JVisualVM**: Visual monitoring and profiling
2. **JConsole**: JMX-based monitoring
3. **GC logs**: Analyze with GCViewer or GCEasy
4. **jstat**: Command-line GC statistics
5. **Flight Recorder**: Low-overhead profiling

```bash
# Monitor GC with jstat
jstat -gc <pid> 1000  # Every 1 second

# Output: S0C S1C S0U S1U EC EU OC OU MC MU YGC YGCT FGC FGCT GCT
```

**Key metrics to monitor:**
- GC frequency and duration
- Heap utilization before/after GC
- Pause time distribution
- Promotion rate (Young → Old)
- Allocation rate

---

## Memory (Heap and Stack)

<a id="q24"></a>
### Q24: What is the difference between Heap and Stack memory?
**Answer:**
| Stack | Heap |
|-------|------|
| Stores method calls and local variables | Stores objects and instance variables |
| LIFO (Last In First Out) | No specific order |
| Thread-specific (each thread has own stack) | Shared among all threads |
| Faster access | Slower access |
| Fixed size (StackOverflowError if exceeded) | Dynamic size (OutOfMemoryError if exceeded) |
| Automatically managed | Managed by GC |

<a id="q25"></a>
### Q25: What is a memory leak in Java and how can you prevent it?
**Answer:**
Memory leak occurs when objects are no longer needed but cannot be garbage collected.

**Common causes:**
1. Static collections holding references
2. Unclosed resources (streams, connections)
3. Inner class holding reference to outer class
4. ThreadLocal not removed after use
5. Listeners not deregistered

**Prevention:**
- Use try-with-resources for closeable resources
- Remove listeners when no longer needed
- Be careful with static collections
- Use weak references when appropriate
- Profile application with tools like VisualVM

<a id="q26"></a>
### Q26: What is the difference between StackOverflowError and OutOfMemoryError?
**Answer:**
- **StackOverflowError**: Stack memory exhausted, usually due to infinite recursion
- **OutOfMemoryError**: Heap memory exhausted, too many objects created, or memory leak

---

## Multithreading

<a id="q27"></a>
### Q27: What is the difference between Runnable and Callable?
**Answer:**
| Runnable | Callable |
|----------|----------|
| `run()` method | `call()` method |
| Returns void | Returns a value (generic type) |
| Cannot throw checked exceptions | Can throw checked exceptions |
| Used with Thread or ExecutorService | Used with ExecutorService only |

```java
Runnable runnable = () -> System.out.println("Running");

Callable<Integer> callable = () -> {
    return 42;
};
```

<a id="q28"></a>
### Q28: What is the difference between `synchronized` and `volatile`?
**Answer:**
| synchronized | volatile |
|--------------|----------|
| Provides atomicity AND visibility | Provides visibility only |
| Can be used on methods/blocks | Can only be used on variables |
| Causes thread blocking | No blocking |
| Higher performance overhead | Lower performance overhead |

<a id="q29"></a>
### Q29: What are Thread Pools and why use them?
**Answer:**
Thread pools manage a collection of reusable threads.

**Benefits:**
- Reduces overhead of thread creation/destruction
- Controls maximum number of concurrent threads
- Provides task queuing

**Types (via Executors):**
```java
ExecutorService fixed = Executors.newFixedThreadPool(10);
ExecutorService cached = Executors.newCachedThreadPool();
ExecutorService single = Executors.newSingleThreadExecutor();
ScheduledExecutorService scheduled = Executors.newScheduledThreadPool(5);
```

<a id="q30"></a>
### Q30: What is a deadlock and how can you prevent it?
**Answer:**
Deadlock occurs when two or more threads are blocked forever, each waiting for the other.

**Four conditions (all must be present):**
1. Mutual exclusion
2. Hold and wait
3. No preemption
4. Circular wait

**Prevention:**
- Lock ordering (always acquire locks in same order)
- Lock timeout (tryLock with timeout)
- Avoid nested locks
- Use concurrent utilities instead of synchronized

<a id="q31"></a>
### Q31: What is the difference between wait() and sleep()?
**Answer:**
| wait() | sleep() |
|--------|---------|
| Object class method | Thread class method |
| Releases the lock | Does not release lock |
| Must be called in synchronized context | Can be called anywhere |
| Woken by notify()/notifyAll() | Wakes after specified time |

---

## Immutable Classes

<a id="q32"></a>
### Q32: What is an immutable class and how do you create one?
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
    
    public String getName() {
        return name;
    }
    
    public List<String> getHobbies() {
        return Collections.unmodifiableList(hobbies); // Defensive copy
    }
}
```

<a id="q33"></a>
### Q33: Why is String immutable in Java?
**Answer:**
1. **String Pool**: Allows string interning, saving memory
2. **Security**: Parameters in network connections, file paths can't be changed
3. **Thread Safety**: Inherently thread-safe without synchronization
4. **Caching**: hashCode can be cached since it won't change
5. **Class Loading**: Prevents malicious class loading

---

## Atomic Classes

<a id="q34"></a>
### Q34: What are Atomic classes and when should you use them?
**Answer:**
Atomic classes provide lock-free, thread-safe operations on single variables using CAS (Compare-And-Swap).

**Common classes:**
- `AtomicInteger`, `AtomicLong`, `AtomicBoolean`
- `AtomicReference<T>`
- `AtomicIntegerArray`, `AtomicLongArray`

**Use when:**
- Simple counters/flags in multithreaded environment
- You need better performance than synchronized
- Single variable needs thread-safe updates

```java
AtomicInteger counter = new AtomicInteger(0);
counter.incrementAndGet();  // Thread-safe
counter.compareAndSet(1, 2); // CAS operation
```

<a id="q35"></a>
### Q35: What is Compare-And-Swap (CAS)?
**Answer:**
CAS is a CPU instruction that:
1. Reads current value
2. Compares with expected value
3. If equal, updates to new value atomically
4. If not equal, operation fails (retry logic)

This is more efficient than locking as it doesn't block threads.

---

Good luck with your interview! 🚀
