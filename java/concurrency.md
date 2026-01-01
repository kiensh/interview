# Concurrency & Multithreading

## Table of Contents

### Thread Basics
- [Q1: What is a Thread and what is its lifecycle?](#q1)
- [Q2: What are the different ways to create a thread?](#q2)
- [Q3: What is the difference between Runnable and Callable?](#q3)
- [Q4: What is the difference between wait() and sleep()?](#q4)

### Synchronization
- [Q5: What is the difference between synchronized and volatile?](#q5)
- [Q6: What is thread synchronization and what are the types of locks?](#q6)
- [Q7: What is a deadlock and how can you prevent it?](#q7)
- [Q8: What is the difference between notify() and notifyAll()?](#q8)

### Concurrency Utilities
- [Q9: What are Thread Pools and why use them?](#q9)
- [Q10: What are the concurrency utilities in java.util.concurrent?](#q10)
- [Q11: What is a CountDownLatch, CyclicBarrier, and Semaphore?](#q11)

### Atomic Operations
- [Q12: What are Atomic classes and when should you use them?](#q12)
- [Q13: What is Compare-And-Swap (CAS)?](#q13)

### Memory
- [Q14: What is the difference between Heap and Stack memory?](#q14)
- [Q15: What is a memory leak in Java and how can you prevent it?](#q15)
- [Q16: What is the difference between StackOverflowError and OutOfMemoryError?](#q16)

---

## Thread Basics

<a id="q1"></a>
### Q1: What is a Thread and what is its lifecycle?
**Answer:**
A thread is the smallest unit of execution within a process. Multiple threads share the same memory space.

**Thread Lifecycle States:**
```
NEW → RUNNABLE → RUNNING → BLOCKED/WAITING/TIMED_WAITING → TERMINATED
```

| State | Description |
|-------|-------------|
| **NEW** | Thread created but not started |
| **RUNNABLE** | Ready to run, waiting for CPU |
| **RUNNING** | Currently executing |
| **BLOCKED** | Waiting to acquire a lock |
| **WAITING** | Waiting indefinitely (wait(), join()) |
| **TIMED_WAITING** | Waiting for specified time (sleep(), wait(timeout)) |
| **TERMINATED** | Execution completed |

```java
Thread thread = new Thread(() -> System.out.println("Running"));
System.out.println(thread.getState());  // NEW
thread.start();
System.out.println(thread.getState());  // RUNNABLE
thread.join();
System.out.println(thread.getState());  // TERMINATED
```

<a id="q2"></a>
### Q2: What are the different ways to create a thread?
**Answer:**

```java
// 1. Extend Thread class
class MyThread extends Thread {
    public void run() { System.out.println("Thread running"); }
}
new MyThread().start();

// 2. Implement Runnable interface
Thread t = new Thread(() -> System.out.println("Runnable running"));
t.start();

// 3. Callable with ExecutorService (returns value)
Callable<Integer> callable = () -> 42;
ExecutorService executor = Executors.newSingleThreadExecutor();
Future<Integer> future = executor.submit(callable);
Integer result = future.get();
executor.shutdown();

// 4. ExecutorService with Runnable
ExecutorService executor = Executors.newFixedThreadPool(4);
executor.execute(() -> System.out.println("Task"));
executor.shutdown();
```

| Approach | Pros | Cons |
|----------|------|------|
| Extend Thread | Simple | Can't extend other classes |
| Implement Runnable | Can extend other classes | No return value |
| Callable + Future | Returns value, throws exceptions | More complex |
| ExecutorService | Thread pool management | Requires shutdown |

<a id="q3"></a>
### Q3: What is the difference between Runnable and Callable?
**Answer:**
| Runnable | Callable |
|----------|----------|
| `run()` method | `call()` method |
| Returns void | Returns a value (generic type) |
| Cannot throw checked exceptions | Can throw checked exceptions |
| Used with Thread or ExecutorService | Used with ExecutorService only |

<a id="q4"></a>
### Q4: What is the difference between wait() and sleep()?
**Answer:**
| wait() | sleep() |
|--------|---------|
| Object class method | Thread class method |
| Releases the lock | Does not release lock |
| Must be called in synchronized context | Can be called anywhere |
| Woken by notify()/notifyAll() | Wakes after specified time |

---

## Synchronization

<a id="q5"></a>
### Q5: What is the difference between synchronized and volatile?
**Answer:**
| synchronized | volatile |
|--------------|----------|
| Provides atomicity AND visibility | Provides visibility only |
| Can be used on methods/blocks | Can only be used on variables |
| Causes thread blocking | No blocking |
| Higher performance overhead | Lower performance overhead |

<a id="q6"></a>
### Q6: What is thread synchronization and what are the types of locks?
**Answer:**

| Lock Type | Description | Scope |
|-----------|-------------|-------|
| Object lock | Lock on specific object | Instance methods |
| Class lock | Lock on Class object | Static methods |
| Explicit lock | ReentrantLock, ReadWriteLock | Fine-grained control |

**Object-level lock:**
```java
public class Counter {
    private int count = 0;
    public synchronized void increment() { count++; }
    
    // Or synchronized block
    public void incrementBlock() {
        synchronized(this) { count++; }
    }
}
```

**ReentrantLock:**
```java
private final ReentrantLock lock = new ReentrantLock();

public void increment() {
    lock.lock();
    try {
        count++;
    } finally {
        lock.unlock();  // Always unlock in finally
    }
}
```

**ReadWriteLock:**
```java
private final ReadWriteLock rwLock = new ReentrantReadWriteLock();

public Object get(String key) {
    rwLock.readLock().lock();  // Multiple readers allowed
    try { return map.get(key); }
    finally { rwLock.readLock().unlock(); }
}

public void put(String key, Object value) {
    rwLock.writeLock().lock();  // Exclusive access
    try { map.put(key, value); }
    finally { rwLock.writeLock().unlock(); }
}
```

<a id="q7"></a>
### Q7: What is a deadlock and how can you prevent it?
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

<a id="q8"></a>
### Q8: What is the difference between notify() and notifyAll()?
**Answer:**
| notify() | notifyAll() |
|----------|-------------|
| Wakes up ONE waiting thread | Wakes up ALL waiting threads |
| Which thread is undefined | All compete for the lock |
| Risk of signal loss | Safer but more overhead |

```java
public class ProducerConsumer {
    private final Queue<Integer> queue = new LinkedList<>();
    
    public synchronized void produce(int item) throws InterruptedException {
        while (queue.size() == MAX_SIZE) { wait(); }
        queue.add(item);
        notifyAll();  // Notify all waiting consumers
    }
    
    public synchronized int consume() throws InterruptedException {
        while (queue.isEmpty()) { wait(); }
        int item = queue.remove();
        notifyAll();  // Notify all waiting producers
        return item;
    }
}
```

---

## Concurrency Utilities

<a id="q9"></a>
### Q9: What are Thread Pools and why use them?
**Answer:**
Thread pools manage a collection of reusable threads.

**Benefits:**
- Reduces overhead of thread creation/destruction
- Controls maximum number of concurrent threads
- Provides task queuing

```java
ExecutorService fixed = Executors.newFixedThreadPool(10);
ExecutorService cached = Executors.newCachedThreadPool();
ExecutorService single = Executors.newSingleThreadExecutor();
ScheduledExecutorService scheduled = Executors.newScheduledThreadPool(5);

// Custom ThreadPoolExecutor
ThreadPoolExecutor custom = new ThreadPoolExecutor(
    2,                      // Core pool size
    10,                     // Max pool size
    60L, TimeUnit.SECONDS,  // Keep alive time
    new LinkedBlockingQueue<>(100),
    new ThreadPoolExecutor.CallerRunsPolicy()
);
```

<a id="q10"></a>
### Q10: What are the concurrency utilities in java.util.concurrent?
**Answer:**

| Category | Classes | Purpose |
|----------|---------|---------|
| **Executors** | ExecutorService, ThreadPoolExecutor | Thread pool management |
| **Locks** | ReentrantLock, ReadWriteLock, StampedLock | Fine-grained locking |
| **Synchronizers** | CountDownLatch, CyclicBarrier, Semaphore | Thread coordination |
| **Concurrent Collections** | ConcurrentHashMap, CopyOnWriteArrayList, BlockingQueue | Thread-safe collections |
| **Atomic Variables** | AtomicInteger, AtomicReference, LongAdder | Lock-free operations |
| **Future** | Future, CompletableFuture | Async results |

**BlockingQueue:**
```java
BlockingQueue<String> queue = new LinkedBlockingQueue<>(10);
queue.put("item");        // Blocks if full
String item = queue.take();  // Blocks if empty
```

<a id="q11"></a>
### Q11: What is a CountDownLatch, CyclicBarrier, and Semaphore?
**Answer:**

**CountDownLatch** - One-time barrier, counts down to zero:
```java
CountDownLatch latch = new CountDownLatch(3);
// Workers call: latch.countDown();
latch.await();  // Main thread waits until count = 0
```

**CyclicBarrier** - Reusable barrier, waits for N threads:
```java
CyclicBarrier barrier = new CyclicBarrier(3, () -> System.out.println("All arrived"));
// Each thread calls: barrier.await();
```

**Semaphore** - Controls access to limited resources:
```java
Semaphore semaphore = new Semaphore(3);  // 3 permits
semaphore.acquire();  // Get permit
try { accessResource(); }
finally { semaphore.release(); }
```

| Feature | CountDownLatch | CyclicBarrier | Semaphore |
|---------|----------------|---------------|-----------|
| Reusable | No | Yes | Yes |
| Use case | Startup coordination | Phased computation | Resource limiting |

---

## Atomic Operations

<a id="q12"></a>
### Q12: What are Atomic classes and when should you use them?
**Answer:**
Atomic classes provide lock-free, thread-safe operations on single variables using CAS.

**Common classes:** `AtomicInteger`, `AtomicLong`, `AtomicBoolean`, `AtomicReference<T>`

**Use when:**
- Simple counters/flags in multithreaded environment
- You need better performance than synchronized
- Single variable needs thread-safe updates

```java
AtomicInteger counter = new AtomicInteger(0);
counter.incrementAndGet();  // Thread-safe
counter.compareAndSet(1, 2); // CAS operation
```

<a id="q13"></a>
### Q13: What is Compare-And-Swap (CAS)?
**Answer:**
CAS is a CPU instruction that:
1. Reads current value
2. Compares with expected value
3. If equal, updates to new value atomically
4. If not equal, operation fails (retry logic)

This is more efficient than locking as it doesn't block threads.

---

## Memory

<a id="q14"></a>
### Q14: What is the difference between Heap and Stack memory?
**Answer:**
| Stack | Heap |
|-------|------|
| Stores method calls and local variables | Stores objects and instance variables |
| LIFO (Last In First Out) | No specific order |
| Thread-specific | Shared among all threads |
| Faster access | Slower access |
| Fixed size (StackOverflowError) | Dynamic size (OutOfMemoryError) |
| Automatically managed | Managed by GC |

<a id="q15"></a>
### Q15: What is a memory leak in Java and how can you prevent it?
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

<a id="q16"></a>
### Q16: What is the difference between StackOverflowError and OutOfMemoryError?
**Answer:**
- **StackOverflowError**: Stack memory exhausted, usually due to infinite recursion
- **OutOfMemoryError**: Heap memory exhausted, too many objects created, or memory leak

---

[← Back to Java Index](README.md)
