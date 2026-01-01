# JVM, Garbage Collection & Security

[Viblo Garbage Collector](https://viblo.asia/p/garbage-collector-1Je5EJ61KnL)

[Understanding how Java Virtual Machine (JVM) works](https://hasithas.medium.com/understanding-how-java-virtual-machine-jvm-works-a1b07c0c399a)

## Table of Contents

### JVM Architecture
- [Q1: What is JVM and how is it different from JDK and JRE?](#q1)
- [Q2: Is JVM platform independent?](#q2)
- [Q3: What are the main components of JVM Architecture?](#q3)
- [Q4: How are JVM instances created and destroyed?](#q4)

### ClassLoader
- [Q5: What is ClassLoader and how does it work?](#q5)
- [Q6: What are the types of ClassLoaders in JVM?](#q6)
- [Q7: What are the ClassLoader principles (Delegation, Visibility, Uniqueness)?](#q7)
- [Q8: What are the phases of class loading (Loading, Linking, Initialization)?](#q8)

### JVM Memory Areas
- [Q9: What are the JVM Memory Areas (Runtime Data Areas)?](#q9)
- [Q10: What is the Method Area?](#q10)
- [Q11: What is the Heap Area?](#q11)
- [Q12: What is the Stack Area?](#q12)
- [Q13: What are PC Registers and Native Method Stack?](#q13)
- [Q14: What is the Java Memory Model (JMM)?](#q14)

### Execution Engine
- [Q15: What is the Execution Engine and its components?](#q15)
- [Q16: What is the difference between Interpreter and JIT Compiler?](#q16)
- [Q17: How does JIT Compiler optimize code?](#q17)

### Garbage Collection
- [Q18: How does Garbage Collection work in Java?](#q18)
- [Q19: What are the different types of Garbage Collectors?](#q19)
- [Q20: What makes an object eligible for garbage collection?](#q20)
- [Q21: What is the difference between Minor GC and Major GC?](#q21)
- [Q22: What are GC Roots?](#q22)
- [Q23: What is Stop-the-World (STW) pause?](#q23)
- [Q24: What are Weak, Soft, and Phantom References?](#q24)
- [Q25: How do you tune and monitor Garbage Collection?](#q25)

### Security
- [Q26: What is the difference between Truststore and Keystore?](#q26)
- [Q27: How do you create and manage Truststore and Keystore?](#q27)

---

## JVM Architecture

<a id="q1"></a>
### Q1: What is JVM and how is it different from JDK and JRE?
**Answer:**

**JVM (Java Virtual Machine):**
- A specification that describes how JVM should work
- An abstract computing machine that enables Java bytecode execution
- Provides runtime environment for Java applications
- NOT platform independent (different implementations for different OS)

**JRE (Java Runtime Environment):**
- Implementation of JVM + core libraries
- Used to RUN Java applications
- Contains: JVM + Java class libraries + supporting files

**JDK (Java Development Kit):**
- JRE + development tools
- Used to DEVELOP and run Java applications
- Contains: JRE + compiler (javac) + debugger + other dev tools

```
┌─────────────────────────────────────────────┐
│                    JDK                      │
│  ┌───────────────────────────────────────┐  │
│  │                 JRE                   │  │
│  │  ┌─────────────────────────────────┐  │  │
│  │  │              JVM                │  │  │
│  │  └─────────────────────────────────┘  │  │
│  │  + Class Libraries (rt.jar)           │  │
│  └───────────────────────────────────────┘  │
│  + javac, javadoc, jar, jdb, etc.           │
└─────────────────────────────────────────────┘
```

**JVM Implementations:**
- HotSpot VM (Oracle, most common)
- OpenJ9 (IBM/Eclipse)
- GraalVM (Oracle, polyglot)
- Azul Zing (commercial, low latency)

<a id="q2"></a>
### Q2: Is JVM platform independent?
**Answer:**
**No, JVM is platform dependent.** 

When you install JRE on Windows, it deploys JVM code for Windows. When installed on Linux, it deploys JVM code for Linux.

However, **Java bytecode is platform independent** - the same `.class` file can run on any JVM regardless of OS.

```
                    ┌─────────────────┐
                    │  HelloWorld.java │
                    └────────┬────────┘
                             │ javac (compile)
                             ▼
                    ┌─────────────────┐
                    │ HelloWorld.class │ ← Platform Independent
                    │   (bytecode)     │
                    └────────┬────────┘
           ┌─────────────────┼─────────────────┐
           ▼                 ▼                 ▼
    ┌────────────┐   ┌────────────┐   ┌────────────┐
    │ JVM Windows│   │ JVM Linux  │   │ JVM macOS  │
    └────────────┘   └────────────┘   └────────────┘
           │                 │                 │
           ▼                 ▼                 ▼
      Windows OS         Linux OS         macOS
```

This is why Java's motto is **"Write Once, Run Anywhere" (WORA)**.

<a id="q3"></a>
### Q3: What are the main components of JVM Architecture?
**Answer:**
JVM consists of 3 main components:

```
┌──────────────────────────────────────────────────────────┐
│                         JVM                              │
│  ┌────────────────────────────────────────────────────┐  │
│  │               1. CLASS LOADER                      │  │
│  │  Loading → Linking → Initialization                │  │
│  └────────────────────────────────────────────────────┘  │
│                          ↓                               │
│  ┌────────────────────────────────────────────────────┐  │
│  │          2. RUNTIME DATA AREAS (Memory)            │  │
│  │  ┌─────────┐ ┌──────┐ ┌───────┐ ┌────┐ ┌────────┐  │  │
│  │  │ Method  │ │ Heap │ │ Stack │ │ PC │ │ Native │  │  │
│  │  │  Area   │ │      │ │       │ │Reg │ │ Stack  │  │  │
│  │  └─────────┘ └──────┘ └───────┘ └────┘ └────────┘  │  │
│  └────────────────────────────────────────────────────┘  │
│                          ↓                               │
│  ┌────────────────────────────────────────────────────┐  │
│  │            3. EXECUTION ENGINE                     │  │
│  │  Interpreter + JIT Compiler + Garbage Collector    │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │        Native Method Interface (JNI)               │  │
│  └────────────────────────────────────────────────────┘  │
│  ┌────────────────────────────────────────────────────┐  │
│  │           Native Method Libraries                  │  │
│  └────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

| Component | Responsibility |
|-----------|----------------|
| **ClassLoader** | Load, link, and initialize class files |
| **Runtime Data Areas** | Memory management during execution |
| **Execution Engine** | Execute bytecode (interpret/compile to native) |
| **JNI** | Interface to native libraries (C/C++) |

<a id="q4"></a>
### Q4: How are JVM instances created and destroyed?
**Answer:**

**Creation:**
- When you run `java ClassName`, a new JVM instance is created
- Each Java application runs in its own JVM instance
- 3 different programs = 3 different JVM instances
- If no Java applications are running, there are no JVM instances

```bash
java HelloWorld  # Creates JVM instance 1
java MyApp       # Creates JVM instance 2 (separate)
```

**Destruction:**
JVM instance is destroyed when:

1. **All non-daemon threads complete** - JVM exits when all user threads finish (daemon threads are automatically terminated)
2. **System.exit() is called** - Explicit termination
3. **Runtime.halt() is called** - Forceful termination (no shutdown hooks)
4. **Fatal error occurs** - Native error, OutOfMemoryError

```java
// Daemon vs Non-daemon threads
Thread daemon = new Thread(() -> { /* ... */ });
daemon.setDaemon(true);  // JVM won't wait for this thread
daemon.start();

// JVM waits for this thread to complete
Thread userThread = new Thread(() -> { /* ... */ });
userThread.start();
```

---

## ClassLoader

<a id="q5"></a>
### Q5: What is ClassLoader and how does it work?
**Answer:**
ClassLoader is responsible for loading Java classes into JVM memory at runtime.

**Key responsibilities:**
1. Find the `.class` file using fully qualified class name (FQCN)
2. Load the bytecode into memory
3. Create a `Class` object representing the loaded class

**Important behaviors:**
- First class loaded is the one containing `main()` method
- Classes are loaded **on demand** (lazy loading)
- Each class is loaded only **once** per ClassLoader
- If class not found: `ClassNotFoundException` or `NoClassDefFoundError`

```java
// Get ClassLoader of a class
ClassLoader loader = MyClass.class.getClassLoader();

// Load class dynamically
Class<?> clazz = Class.forName("com.example.MyClass");

// Different ClassLoaders
String.class.getClassLoader();  // null (Bootstrap)
MyClass.class.getClassLoader(); // AppClassLoader
```

<a id="q6"></a>
### Q6: What are the types of ClassLoaders in JVM?
**Answer:**

```
┌─────────────────────────────────────────┐
│     Bootstrap ClassLoader (Native)      │  ← Loads core Java classes
│     ($JAVA_HOME/jre/lib/rt.jar)         │     java.lang, java.util, etc.
└─────────────────────┬───────────────────┘
                      │ parent
┌─────────────────────▼───────────────────┐
│   Extension/Platform ClassLoader        │  ← Loads extension classes
│   ($JAVA_HOME/jre/lib/ext)              │     javax.*, security providers
└─────────────────────┬───────────────────┘
                      │ parent
┌─────────────────────▼───────────────────┐
│   Application/System ClassLoader        │  ← Loads application classes
│   (classpath, -cp)                      │     Your code, dependencies
└─────────────────────┬───────────────────┘
                      │ parent
┌─────────────────────▼───────────────────┐
│      Custom ClassLoaders (optional)     │  ← Plugin systems, app servers
└─────────────────────────────────────────┘
```

| ClassLoader | Location | Loads |
|-------------|----------|-------|
| **Bootstrap** | `$JAVA_HOME/jre/lib` | Core Java API (rt.jar) |
| **Extension** | `$JAVA_HOME/jre/lib/ext` | Extension libraries |
| **Application** | Classpath (`-cp`) | Application classes |
| **Custom** | Defined by developer | Dynamic loading, plugins |

<a id="q7"></a>
### Q7: What are the ClassLoader principles (Delegation, Visibility, Uniqueness)?
**Answer:**

**1. Delegation Principle (Parent-First)**
```
Request to load com.example.MyClass
          │
          ▼
    App ClassLoader ──────► "Can parent load it?"
          │                        │
          │                        ▼
          │               Extension ClassLoader ──► "Can parent load it?"
          │                        │                       │
          │                        │                       ▼
          │                        │              Bootstrap ClassLoader
          │                        │                       │
          │                        │          "No, not in rt.jar"
          │                        │                       │
          │               "No, not in ext"  ◄──────────────┘
          │                        │
    "Found in classpath" ◄─────────┘
```

- Child delegates to parent first
- Only if parent can't load, child tries
- **Security**: Prevents loading fake `java.lang.String`

**2. Visibility Principle**
- Child ClassLoader can see classes loaded by parent
- Parent cannot see classes loaded by child

```java
// Classes loaded by Bootstrap are visible to all
// Classes loaded by App ClassLoader not visible to Extension ClassLoader
```

**3. Uniqueness Principle**
- A class is loaded only once by a ClassLoader
- Same class can be loaded by different ClassLoaders (different Class objects!)

```java
// Two classes with same name from different ClassLoaders are different
Class<?> c1 = loader1.loadClass("MyClass");
Class<?> c2 = loader2.loadClass("MyClass");
c1 == c2;  // false if different ClassLoaders
```

<a id="q8"></a>
### Q8: What are the phases of class loading (Loading, Linking, Initialization)?
**Answer:**

```
.class file ──► LOADING ──► LINKING ──► INITIALIZATION ──► Ready to use
                              │
                   ┌──────────┼──────────┐
                   ▼          ▼          ▼
              Verification  Prepare   Resolution
```

**1. Loading**
- Read `.class` file bytecode
- Create `Class` object in Heap
- Store class metadata:
  - Fully Qualified Class Name (FQCN)
  - Parent class information
  - Methods, fields, constructors
  - Whether class, interface, or enum

**2. Linking**

| Sub-phase | Description |
|-----------|-------------|
| **Verification** | Verify bytecode is valid, comes from valid compiler, correct structure |
| **Preparation** | Allocate memory for static variables, assign DEFAULT values (not initial!) |
| **Resolution** | Replace symbolic references with direct memory references |

```java
// During Preparation:
static int year = 2024;  // year is set to 0 (default), NOT 2024

// During Resolution:
// "Employee" string reference → actual memory address
```

**3. Initialization**
- Assign actual values to static variables
- Execute static blocks in order of appearance

```java
public class Example {
    static int x = 10;           // Assigned here (not in Preparation)
    
    static {
        System.out.println("Static block executed");
    }
}
```

**Important:** Every class must be initialized before its **active use**:
- Creating instance (`new`)
- Accessing static field/method
- Reflection
- Initializing subclass

---

## JVM Memory Areas

<a id="q9"></a>
### Q9: What are the JVM Memory Areas (Runtime Data Areas)?
**Answer:**
JVM divides memory into 5 areas:

```
┌─────────────────────────────────────────────────────────┐
│                   PER JVM (Shared)                       │
│  ┌─────────────────────┐  ┌──────────────────────────┐  │
│  │     METHOD AREA     │  │          HEAP            │  │
│  │  (Class metadata,   │  │  (Objects, instance      │  │
│  │   static variables, │  │   variables, arrays)     │  │
│  │   constant pool)    │  │                          │  │
│  └─────────────────────┘  └──────────────────────────┘  │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│               PER THREAD (Not shared)                    │
│  ┌──────────┐  ┌──────────────┐  ┌───────────────────┐  │
│  │  STACK   │  │  PC REGISTER │  │  NATIVE METHOD    │  │
│  │ (Frames, │  │ (Next instr  │  │      STACK        │  │
│  │  locals) │  │   address)   │  │  (Native methods) │  │
│  └──────────┘  └──────────────┘  └───────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

| Area | Scope | Contents |
|------|-------|----------|
| Method Area | Per JVM | Class info, static vars, constant pool |
| Heap | Per JVM | Objects, arrays |
| Stack | Per Thread | Method frames, local variables |
| PC Register | Per Thread | Current instruction address |
| Native Stack | Per Thread | Native method data |

<a id="q10"></a>
### Q10: What is the Method Area?
**Answer:**
Method Area (also called Metaspace in Java 8+) stores class-level data.

**Contains:**
- Class structure (fields, methods, constructors)
- Runtime constant pool
- Static variables
- Method bytecode
- Field and method data

```java
public class Employee {
    // Stored in Method Area:
    private static int count = 0;     // Static variable
    private static final String COMPANY = "XYZ";  // Constant
    
    public void work() { }  // Method bytecode
}
```

**Metaspace (Java 8+):**
- Replaced PermGen
- Uses native memory (not heap)
- Auto-grows (configurable with `-XX:MaxMetaspaceSize`)

<a id="q11"></a>
### Q11: What is the Heap Area?
**Answer:**
Heap is shared memory where all objects and arrays are stored.

```
┌─────────────────────────────────────────────────────┐
│                        HEAP                          │
│  ┌─────────────────────────────────────────────┐    │
│  │            YOUNG GENERATION                  │    │
│  │  ┌────────────┬─────────┬─────────┐         │    │
│  │  │   EDEN     │   S0    │   S1    │         │    │
│  │  │ (new objs) │(survivor)│(survivor)│        │    │
│  │  └────────────┴─────────┴─────────┘         │    │
│  └─────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────┐    │
│  │           OLD GENERATION (Tenured)           │    │
│  │        (Long-lived objects)                  │    │
│  └─────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────┘
```

**Characteristics:**
- Shared among all threads
- Created when JVM starts
- Objects are garbage collected here
- Divided into Young and Old generations

**Heap sizing:**
```bash
-Xms512m    # Initial heap size
-Xmx2g      # Maximum heap size
-Xmn256m    # Young generation size
```

<a id="q12"></a>
### Q12: What is the Stack Area?
**Answer:**
Each thread has its own stack that stores method invocation data.

**Structure:**
```
Thread Stack
┌─────────────────────────┐
│  Frame: method3()       │  ← Top (current method)
│  ┌───────────────────┐  │
│  │ Local Variables   │  │
│  │ Operand Stack     │  │
│  │ Frame Data        │  │
│  └───────────────────┘  │
├─────────────────────────┤
│  Frame: method2()       │
├─────────────────────────┤
│  Frame: method1()       │
├─────────────────────────┤
│  Frame: main()          │  ← Bottom
└─────────────────────────┘
```

**Stack Frame contains:**
1. **Local Variables Array** - method parameters, local variables
2. **Operand Stack** - intermediate calculation results
3. **Frame Data** - constant pool reference, exception table

**Behavior:**
- New frame pushed when method called
- Frame popped when method returns or throws uncaught exception
- LIFO (Last In, First Out)
- `StackOverflowError` if stack is full (deep recursion)

```bash
-Xss512k    # Stack size per thread
```

<a id="q13"></a>
### Q13: What are PC Registers and Native Method Stack?
**Answer:**

**PC (Program Counter) Register:**
- Each thread has its own PC Register
- Holds address of current JVM instruction being executed
- If executing native method, PC Register is undefined

```
Thread 1: PC = 0x1234 (bytecode instruction address)
Thread 2: PC = 0x5678
Thread 3: PC = undefined (executing native method)
```

**Native Method Stack:**
- Similar to JVM Stack but for native methods
- Used when Java calls C/C++ code via JNI
- Each thread has its own native method stack

```java
// Example: System.currentTimeMillis() calls native code
public static native long currentTimeMillis();
// This executes in Native Method Stack
```

<a id="q14"></a>
### Q14: What is the Java Memory Model (JMM)?
**Answer:**
The Java Memory Model (JMM) defines how threads interact with memory and ensures consistency across platforms.

**Memory structure:**
```
Thread 1         Thread 2         Thread 3
┌───────┐       ┌───────┐       ┌───────┐
│ Stack │       │ Stack │       │ Stack │
│ Cache │       │ Cache │       │ Cache │
└───┬───┘       └───┬───┘       └───┬───┘
    └───────────────┼───────────────┘
                    ▼
            ┌─────────────┐
            │ Main Memory │ (Heap)
            └─────────────┘
```

**Key concepts:**
- **Visibility**: When changes by one thread become visible to others
- **Atomicity**: Operations that execute completely or not at all
- **Happens-before**: Guarantees ordering between operations

**Visibility solutions:**
```java
volatile boolean running = true;  // Visibility guaranteed
// Or synchronized, or Atomic classes
```

**Happens-before rules:**
1. Program order within a thread
2. Monitor lock/unlock
3. Volatile write → volatile read
4. Thread.start() → first action in started thread
5. Last action in thread → Thread.join() returns

---

## Execution Engine

<a id="q15"></a>
### Q15: What is the Execution Engine and its components?
**Answer:**
Execution Engine converts bytecode to machine code and executes it.

```
┌─────────────────────────────────────────────────────┐
│                 EXECUTION ENGINE                     │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────┐  │
│  │ INTERPRETER │  │ JIT COMPILER │  │  GARBAGE   │  │
│  │             │  │              │  │ COLLECTOR  │  │
│  │ Line by line│  │ Compiles hot │  │            │  │
│  │ execution   │  │ code to      │  │ Frees      │  │
│  │             │  │ native code  │  │ memory     │  │
│  └─────────────┘  └──────────────┘  └────────────┘  │
└─────────────────────────────────────────────────────┘
```

| Component | Role |
|-----------|------|
| **Interpreter** | Executes bytecode line by line |
| **JIT Compiler** | Compiles frequently used code to native |
| **Garbage Collector** | Reclaims unused memory |

<a id="q16"></a>
### Q16: What is the difference between Interpreter and JIT Compiler?
**Answer:**

| Interpreter | JIT Compiler |
|-------------|--------------|
| Executes bytecode line by line | Compiles entire method to native code |
| Fast startup | Slower startup, faster execution |
| Re-interprets same code each call | Caches compiled code |
| Lower memory usage | Higher memory for compiled code |
| Default execution mode | Kicks in for "hot" code |

**How they work together:**
```
Bytecode
    │
    ▼
Interpreter ──────────────────► Execute line by line
    │
    │ (Method called many times - detected by Profiler)
    ▼
JIT Compiler ──► Compile to native ──► Cache ──► Execute native
```

**JIT Compilation Trigger:**
- Method invocation threshold (default ~10,000 calls)
- Loop back-edge threshold

```bash
# JIT tuning
-XX:CompileThreshold=10000    # Invocations before JIT
-XX:+PrintCompilation         # Log JIT compilations
```

<a id="q17"></a>
### Q17: How does JIT Compiler optimize code?
**Answer:**
JIT Compiler includes several optimization components:

**Components:**
1. **Profiler** - Identifies "hot spots" (frequently executed code)
2. **Intermediate Code Generator** - Creates intermediate representation
3. **Code Optimizer** - Applies optimizations
4. **Target Code Generator** - Generates native machine code

**Optimizations performed:**

| Optimization | Description |
|--------------|-------------|
| **Inlining** | Replace method call with method body |
| **Dead Code Elimination** | Remove unreachable code |
| **Loop Unrolling** | Reduce loop overhead |
| **Escape Analysis** | Allocate on stack if object doesn't escape |
| **Constant Folding** | Compute constants at compile time |
| **Null Check Elimination** | Remove redundant null checks |

```java
// Before optimization
for (int i = 0; i < 100; i++) {
    result += getValue();  // Method call overhead
}

// After inlining
for (int i = 0; i < 100; i++) {
    result += 42;  // Method body inlined
}
```

**Tiered Compilation (Java 7+):**
- Level 0: Interpreted
- Level 1-3: C1 compiler (quick compilation)
- Level 4: C2 compiler (aggressive optimizations)

```bash
-XX:+TieredCompilation    # Enable tiered compilation (default)
```

---

## Garbage Collection

<a id="q18"></a>
### Q18: How does Garbage Collection work in Java?
**Answer:**
GC automatically frees memory occupied by unreachable objects.

**Phases:**
1. **Mark Phase**: GC identifies all reachable objects starting from GC roots
2. **Sweep Phase**: Unreachable objects are removed from memory
3. **Compact Phase** (optional): Memory is defragmented

**Generational GC:**
```
┌─────────────────────────────────────────────────────┐
│  YOUNG GENERATION (Minor GC - frequent, fast)       │
│  ┌────────────┬──────────┬──────────┐               │
│  │   EDEN     │    S0    │    S1    │               │
│  │ (new objs) │(survivor)│(survivor)│               │
│  └────────────┴──────────┴──────────┘               │
│                     ↓ (survives multiple GCs)       │
│  OLD GENERATION (Major GC - infrequent, slow)       │
│  ┌─────────────────────────────────────┐            │
│  │            TENURED                   │            │
│  └─────────────────────────────────────┘            │
└─────────────────────────────────────────────────────┘
```

**Object lifecycle:**
1. New object created in **Eden**
2. Survives Minor GC → moved to **Survivor** (S0 or S1)
3. Survives multiple Minor GCs → promoted to **Old Generation**

<a id="q19"></a>
### Q19: What are the different types of Garbage Collectors?
**Answer:**

| Collector | Description | Use Case | Flag |
|-----------|-------------|----------|------|
| **Serial GC** | Single-threaded, simple | Small apps, single CPU | `-XX:+UseSerialGC` |
| **Parallel GC** | Multi-threaded STW | Throughput-focused | `-XX:+UseParallelGC` |
| **G1 GC** | Region-based, concurrent | General purpose (default Java 9+) | `-XX:+UseG1GC` |
| **ZGC** | Concurrent, low latency | Large heaps, <10ms pauses | `-XX:+UseZGC` |
| **Shenandoah** | Concurrent compaction | Low pause times | `-XX:+UseShenandoahGC` |

**G1 GC (Garbage First):**
```
┌────┬────┬────┬────┬────┬────┬────┬────┐
│ E  │ S  │ O  │ E  │ O  │ H  │ E  │ O  │  (Regions)
└────┴────┴────┴────┴────┴────┴────┴────┘
E=Eden, S=Survivor, O=Old, H=Humongous (large objects)
```

<a id="q20"></a>
### Q20: What makes an object eligible for garbage collection?
**Answer:**
An object becomes eligible when it's **unreachable** from any GC Root.

**Ways to become unreachable:**
1. **Reference set to null**
   ```java
   Person p = new Person();
   p = null;  // Object eligible for GC
   ```

2. **Reference reassigned**
   ```java
   Person p = new Person("John");
   p = new Person("Jane");  // "John" object eligible
   ```

3. **Local scope ends**
   ```java
   void method() {
       Person p = new Person();  // Created
   }  // p goes out of scope, object eligible
   ```

4. **Island of isolation** (circular references with no external refs)
   ```java
   class Node { Node next; }
   Node a = new Node();
   Node b = new Node();
   a.next = b;
   b.next = a;
   a = null;
   b = null;  // Both nodes eligible (circular but unreachable)
   ```

<a id="q21"></a>
### Q21: What is the difference between Minor GC and Major GC?
**Answer:**

| Minor GC | Major GC (Full GC) |
|----------|-------------------|
| Cleans Young Generation | Cleans Old Generation (and sometimes Young) |
| Very fast (milliseconds) | Slower (can be seconds) |
| Frequent | Less frequent |
| Uses copying algorithm | Uses mark-sweep-compact |
| Short STW pause | Longer STW pause |

```
Young Gen [Eden | S0 | S1] → Minor GC
Old Gen [Tenured]          → Major GC
```

**When Major GC occurs:**
- Old Generation is full
- Metaspace is full
- `System.gc()` called (not guaranteed)
- Promotion failure (not enough space in Old Gen)

<a id="q22"></a>
### Q22: What are GC Roots?
**Answer:**
GC Roots are the starting points for the garbage collector to determine which objects are reachable.

**Types of GC Roots:**
1. **Local variables** in stack frames of active threads
2. **Active threads** themselves
3. **Static variables** of loaded classes
4. **JNI references** (native code references)
5. **Synchronized monitors** (objects used in synchronized blocks)
6. **Class loaders** and classes they've loaded

```
GC Roots
    │
    ├── Thread stacks (local vars)
    │       └── obj1 ──► obj2 ──► obj3  ✓ Reachable
    │
    ├── Static variables
    │       └── obj4 ──► obj5  ✓ Reachable
    │
    └── Active threads
            └── obj6  ✓ Reachable

    obj7 ◄──► obj8  (circular but no root) ✗ Eligible for GC
```

<a id="q23"></a>
### Q23: What is Stop-the-World (STW) pause?
**Answer:**
STW pause is when the JVM halts all application threads to perform garbage collection safely.

**Why needed:**
- Ensures object graph doesn't change during GC
- Prevents moving objects while references exist

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

<a id="q24"></a>
### Q24: What are Weak, Soft, and Phantom References?
**Answer:**

| Reference Type | Collected When | Use Case |
|---------------|----------------|----------|
| **Strong** | Never (while reachable) | Normal references |
| **Soft** | When memory is low | Caches |
| **Weak** | Next GC cycle | Canonicalizing mappings |
| **Phantom** | After finalization | Pre-mortem cleanup |

```java
// Strong - normal reference
Object obj = new Object();

// Soft - collected when memory needed
SoftReference<Object> soft = new SoftReference<>(new Object());

// Weak - collected on next GC
WeakReference<Object> weak = new WeakReference<>(new Object());

// Phantom - for cleanup notifications
ReferenceQueue<Object> queue = new ReferenceQueue<>();
PhantomReference<Object> phantom = new PhantomReference<>(obj, queue);
```

**Practical uses:**
- `WeakHashMap` - cache that allows GC of keys
- `SoftReference` - memory-sensitive caches
- `PhantomReference` - cleanup before/after finalization

<a id="q25"></a>
### Q25: How do you tune and monitor Garbage Collection?
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
-XX:NewRatio=2            # Old/Young ratio (Old = 2x Young)
-XX:SurvivorRatio=8       # Eden/Survivor ratio

# G1 specific
-XX:MaxGCPauseMillis=200  # Target pause time
-XX:G1HeapRegionSize=16m  # Region size

# GC logging (Java 9+)
-Xlog:gc*:file=gc.log:time,uptime:filecount=5,filesize=10m

# GC logging (Java 8)
-XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:gc.log
```

**Monitoring tools:**
1. **JVisualVM**: Visual monitoring and profiling
2. **JConsole**: JMX-based monitoring
3. **GC logs**: Analyze with GCViewer or GCEasy
4. **jstat**: Command-line GC statistics
5. **Flight Recorder**: Low-overhead profiling
6. **jcmd**: Diagnostic commands

```bash
# jstat example
jstat -gc <pid> 1000  # GC stats every 1 second

# jcmd example
jcmd <pid> GC.heap_info
```

---

## Security

<a id="q26"></a>
### Q26: What is the difference between Truststore and Keystore?
**Answer:**

| Keystore | Truststore |
|----------|------------|
| Stores YOUR private keys and certificates | Stores TRUSTED certificates (CAs) |
| Used to prove YOUR identity | Used to verify OTHERS' identity |
| Server uses to prove identity to clients | Client uses to verify server identity |
| `-Djavax.net.ssl.keyStore` | `-Djavax.net.ssl.trustStore` |

```
TLS Handshake:
Server                              Client
┌─────────────┐                    ┌─────────────┐
│  KEYSTORE   │──Certificate──────►│ TRUSTSTORE  │
│ (identity)  │                    │ (validation)│
└─────────────┘                    └─────────────┘
```

<a id="q27"></a>
### Q27: How do you create and manage Truststore and Keystore?
**Answer:**

**Using keytool:**
```bash
# Create keystore with new key pair
keytool -genkeypair -alias mykey -keyalg RSA -keysize 2048 \
    -keystore keystore.jks -validity 365

# Export certificate
keytool -exportcert -alias mykey -keystore keystore.jks -file cert.cer

# Import into truststore
keytool -importcert -alias serverkey -file cert.cer -keystore truststore.jks

# List contents
keytool -list -keystore keystore.jks
```

**Spring Boot configuration:**
```yaml
server:
  ssl:
    key-store: classpath:keystore.p12
    key-store-password: ${KEY_STORE_PASSWORD}
    key-store-type: PKCS12
    trust-store: classpath:truststore.jks
    trust-store-password: ${TRUST_STORE_PASSWORD}
```

---

[← Back to Java Index](README.md)
