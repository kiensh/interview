# Concurrency & Multithreading

[Introduction to Thread Pools in Java - Baeldung](https://www.baeldung.com/thread-pool-java-and-guava)

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

### Thread Pools
- [Q9: What are Thread Pools and why use them?](#q9)
- [Q10: What are the different types of thread pools and when to use each?](#q10)
- [Q11: What are the parameters of ThreadPoolExecutor?](#q11)
- [Q12: What are the rejection policies in ThreadPoolExecutor?](#q12)
- [Q13: What is the difference between execute() and submit()?](#q13)
- [Q14: What is the difference between Executor and ExecutorService?](#q14)
- [Q15: How do you properly shutdown a thread pool?](#q15)
- [Q16: What is the difference between scheduleAtFixedRate and scheduleWithFixedDelay?](#q16)
- [Q17: What is ForkJoinPool and how does it work?](#q17)

### Concurrency Utilities
- [Q18: What are the concurrency utilities in java.util.concurrent?](#q18)
- [Q19: What is a CountDownLatch, CyclicBarrier, and Semaphore?](#q19)

### Atomic Operations
- [Q20: What are Atomic classes and when should you use them?](#q20)
- [Q21: What is Compare-And-Swap (CAS)?](#q21)

### Memory
- [Q22: What is the difference between Heap and Stack memory?](#q22)
- [Q23: What is a memory leak in Java and how can you prevent it?](#q23)
- [Q24: What is the difference between StackOverflowError and OutOfMemoryError?](#q24)

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

## Thread Pools

<a id="q9"></a>
### Q9: What are Thread Pools and why use them?
**Answer:**
Thread pools manage a collection of reusable threads, avoiding the overhead of creating/destroying threads for each task.

**Benefits:**
- Reduces thread creation/destruction overhead
- Controls maximum concurrent threads
- Provides task queuing
- Better resource management

**Why not create threads directly?**
- Thread creation is expensive (OS resources)
- Too many threads → context switching overhead
- Uncontrolled threads → resource exhaustion

<a id="q10"></a>
### Q10: What are the different types of thread pools and when to use each?
**Answer:**

| Type | Core | Max | KeepAlive | Queue | Use Case |
|------|------|-----|-----------|-------|----------|
| `newFixedThreadPool(n)` | n | n | 0 | `LinkedBlockingQueue` (unbounded) | Stable workload |
| `newCachedThreadPool()` | 0 | Integer.MAX_VALUE | 60s | `SynchronousQueue` | Short-lived tasks |
| `newSingleThreadExecutor()` | 1 | 1 | 0 | `LinkedBlockingQueue` (unbounded) | Sequential execution |
| `newScheduledThreadPool(n)` | n | Integer.MAX_VALUE | 0 | `DelayedWorkQueue` | Delayed/periodic tasks |
| `newWorkStealingPool()` | CPU cores | CPU cores | 60s | Work-stealing queues | Parallel recursive tasks |

**FixedThreadPool** - Known, steady workload:
```java
// Always maintains exactly n threads, excess tasks queue up
ExecutorService executor = Executors.newFixedThreadPool(10);
```

**CachedThreadPool** - Many short-lived tasks:
```java
// Creates threads as needed, reuses idle threads, 60s timeout
ExecutorService executor = Executors.newCachedThreadPool();
```

**SingleThreadExecutor** - Sequential execution:
```java
// Guarantees FIFO order, single thread, wrapped in immutable decorator
ExecutorService executor = Executors.newSingleThreadExecutor();
```

**ScheduledThreadPool** - Delayed/periodic tasks:
```java
ScheduledExecutorService executor = Executors.newScheduledThreadPool(5);
executor.schedule(task, 5, TimeUnit.SECONDS);
executor.scheduleAtFixedRate(task, 0, 10, TimeUnit.SECONDS);
```

**WorkStealingPool** - Parallel computation:
```java
// ForkJoinPool with work-stealing algorithm
ExecutorService executor = Executors.newWorkStealingPool();
```

**⚠️ Production Warning:**
- `FixedThreadPool` / `SingleThreadExecutor`: Unbounded queue → OOM risk
- `CachedThreadPool`: Unlimited threads → resource exhaustion
- **Recommendation:** Use `ThreadPoolExecutor` directly with bounded queue

<a id="q11"></a>
### Q11: What are the parameters of ThreadPoolExecutor?
**Answer:**

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    corePoolSize,      // Threads kept alive even when idle
    maximumPoolSize,   // Maximum threads allowed
    keepAliveTime,     // Idle time before excess threads terminate
    timeUnit,          // Unit for keepAliveTime
    workQueue,         // Queue for waiting tasks
    threadFactory,     // Factory to create new threads
    rejectionHandler   // Policy when queue is full and max threads reached
);
```

**Task handling flow:**
```
New Task Submitted
        │
        ▼
┌───────────────────────────┐
│ threads < corePoolSize?   │──Yes──► Create new core thread
└───────────────────────────┘
        │ No
        ▼
┌───────────────────────────┐
│ workQueue has space?      │──Yes──► Add task to queue
└───────────────────────────┘
        │ No (queue full)
        ▼
┌───────────────────────────┐
│ threads < maxPoolSize?    │──Yes──► Create new non-core thread
└───────────────────────────┘
        │ No
        ▼
    RejectionHandler
```

**Production example:**
```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    10,                              // Core: always keep 10 threads
    50,                              // Max: can grow to 50 under load
    60L, TimeUnit.SECONDS,           // Non-core threads die after 60s idle
    new ArrayBlockingQueue<>(100),   // Bounded queue of 100 tasks
    new ThreadFactoryBuilder()
        .setNameFormat("worker-%d")
        .build(),
    new ThreadPoolExecutor.CallerRunsPolicy()
);

// Optional: allow core threads to timeout too
executor.allowCoreThreadTimeOut(true);
```

<a id="q12"></a>
### Q12: What are the rejection policies in ThreadPoolExecutor?
**Answer:**
When queue is full AND max threads reached, rejection policy is triggered:

| Policy | Behavior |
|--------|----------|
| `AbortPolicy` (default) | Throws `RejectedExecutionException` |
| `CallerRunsPolicy` | Caller thread executes the task (provides backpressure) |
| `DiscardPolicy` | Silently discards the task |
| `DiscardOldestPolicy` | Discards oldest queued task, then retries |

```java
// AbortPolicy - fail fast (default)
new ThreadPoolExecutor(..., new ThreadPoolExecutor.AbortPolicy());

// CallerRunsPolicy - backpressure, slows down producer
new ThreadPoolExecutor(..., new ThreadPoolExecutor.CallerRunsPolicy());

// DiscardPolicy - silent drop (use with caution!)
new ThreadPoolExecutor(..., new ThreadPoolExecutor.DiscardPolicy());

// Custom policy
new ThreadPoolExecutor(..., (runnable, executor) -> {
    log.warn("Task rejected: {}", runnable);
    // Could: save to DB, send to message queue, etc.
});
```

**Best practice:** Use `CallerRunsPolicy` for backpressure or custom handler for logging/recovery.

<a id="q13"></a>
### Q13: What is the difference between execute() and submit()?
**Answer:**

| `execute(Runnable)` | `submit(Runnable/Callable)` |
|---------------------|----------------------------|
| Returns `void` | Returns `Future<T>` |
| `Runnable` only | `Runnable` or `Callable` |
| Exceptions thrown to UncaughtExceptionHandler | Exceptions captured in `Future` |
| Fire-and-forget | Can track, cancel, get result |

```java
// execute() - fire and forget
executor.execute(() -> {
    doWork();  // Exception goes to UncaughtExceptionHandler
});

// submit() - trackable
Future<?> future = executor.submit(() -> {
    doWork();
    return result;
});

// Check completion
boolean done = future.isDone();

// Cancel task
future.cancel(true);  // true = interrupt if running

// Get result (blocks until complete)
try {
    Object result = future.get(5, TimeUnit.SECONDS);
} catch (ExecutionException e) {
    Throwable cause = e.getCause();  // Original exception
} catch (TimeoutException e) {
    future.cancel(true);
}
```

<a id="q14"></a>
### Q14: What is the difference between Executor and ExecutorService?
**Answer:**

| Executor | ExecutorService |
|----------|-----------------|
| Single method: `execute(Runnable)` | Extends Executor with many methods |
| No return value | `submit()` returns `Future` |
| No lifecycle management | `shutdown()`, `shutdownNow()` |
| Cannot track task completion | Can cancel, check status, get result |

```java
// Executor - simple interface
public interface Executor {
    void execute(Runnable command);
}

// ExecutorService - full-featured
public interface ExecutorService extends Executor {
    <T> Future<T> submit(Callable<T> task);
    Future<?> submit(Runnable task);
    void shutdown();
    List<Runnable> shutdownNow();
    boolean isShutdown();
    boolean isTerminated();
    boolean awaitTermination(long timeout, TimeUnit unit);
    // ... more methods
}
```

**Tip:** Program to `ExecutorService` interface for flexibility.

<a id="q15"></a>
### Q15: How do you properly shutdown a thread pool?
**Answer:**

| Method | Behavior |
|--------|----------|
| `shutdown()` | Stop accepting new tasks, complete queued tasks |
| `shutdownNow()` | Stop accepting, interrupt running, return queued |
| `awaitTermination()` | Block until all tasks complete or timeout |

```java
// Graceful shutdown pattern
public void shutdownExecutor(ExecutorService executor) {
    executor.shutdown();  // Disable new tasks
    try {
        // Wait for existing tasks to complete
        if (!executor.awaitTermination(60, TimeUnit.SECONDS)) {
            executor.shutdownNow();  // Force shutdown
            // Wait for tasks to respond to cancellation
            if (!executor.awaitTermination(60, TimeUnit.SECONDS)) {
                log.error("Executor did not terminate");
            }
        }
    } catch (InterruptedException e) {
        executor.shutdownNow();
        Thread.currentThread().interrupt();  // Preserve interrupt status
    }
}
```

**With try-with-resources (Java 19+):**
```java
try (ExecutorService executor = Executors.newFixedThreadPool(10)) {
    executor.submit(() -> doWork());
}  // Automatically calls shutdown() and awaitTermination()
```

<a id="q16"></a>
### Q16: What is the difference between scheduleAtFixedRate and scheduleWithFixedDelay?
**Answer:**

| `scheduleAtFixedRate` | `scheduleWithFixedDelay` |
|-----------------------|--------------------------|
| Period from **start** of each task | Delay from **end** of previous task |
| Fixed execution rate | Fixed gap between executions |
| Tasks may overlap if slow | Guaranteed gap between tasks |

```java
ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);

// scheduleAtFixedRate - every 100ms from START of each task
scheduler.scheduleAtFixedRate(task, 0, 100, TimeUnit.MILLISECONDS);

// scheduleWithFixedDelay - 100ms after END of each task
scheduler.scheduleWithFixedDelay(task, 0, 100, TimeUnit.MILLISECONDS);
```

**Visual comparison (task takes 30ms):**
```
scheduleAtFixedRate (period=100ms):
|--30ms--|      |--30ms--|      |--30ms--|
0        30    100       130   200       230
<--100ms--><--100ms-->

scheduleWithFixedDelay (delay=100ms):
|--30ms--|          |--30ms--|          |--30ms--|
0        30        130       160       260       290
         <--100ms-->         <--100ms-->
```

**If task takes longer than period (150ms task, 100ms period):**
```
scheduleAtFixedRate: Tasks queue up, no gap!
|------150ms------|------150ms------|
0                150               300

scheduleWithFixedDelay: Always 100ms gap
|------150ms------|          |------150ms------|
0                150        250               400
                  <--100ms-->
```

<a id="q17"></a>
### Q17: What is ForkJoinPool and how does it work?
**Answer:**
ForkJoinPool is designed for **divide-and-conquer** algorithms using **work-stealing**.

**Work-stealing algorithm:**
```
Thread 1 Deque: [Task A] [Task B] [Task C] ← pushes here
Thread 2 Deque: [Task D]
Thread 3 Deque: []  ← idle, steals Task C from Thread 1's tail

- Each thread has own double-ended queue (deque)
- Thread pushes/pops from HEAD (LIFO for own tasks)
- Idle threads steal from other threads' TAIL
- Reduces contention vs shared queue
```

**ForkJoinPool vs ThreadPoolExecutor:**

| Feature | ThreadPoolExecutor | ForkJoinPool |
|---------|-------------------|--------------|
| Algorithm | Shared task queue | Work-stealing |
| Best for | Independent tasks | Recursive tasks |
| Task type | Runnable/Callable | ForkJoinTask (RecursiveTask/Action) |
| Used by | General purpose | Parallel Streams, CompletableFuture |

**RecursiveTask (returns value) vs RecursiveAction (void):**
```java
public class SumTask extends RecursiveTask<Long> {
    private final long[] array;
    private final int start, end;
    private static final int THRESHOLD = 1000;
    
    @Override
    protected Long compute() {
        if (end - start <= THRESHOLD) {
            // Base case: compute directly
            long sum = 0;
            for (int i = start; i < end; i++) {
                sum += array[i];
            }
            return sum;
        } else {
            // Recursive case: fork subtasks
            int mid = (start + end) / 2;
            SumTask left = new SumTask(array, start, mid);
            SumTask right = new SumTask(array, mid, end);
            
            left.fork();   // Execute asynchronously
            Long rightResult = right.compute();  // Compute directly
            Long leftResult = left.join();       // Wait for forked task
            
            return leftResult + rightResult;
        }
    }
}

// Usage
ForkJoinPool pool = ForkJoinPool.commonPool();
Long result = pool.invoke(new SumTask(array, 0, array.length));
```

**Common pool:**
```java
// Shared pool used by parallel streams
ForkJoinPool commonPool = ForkJoinPool.commonPool();
int parallelism = commonPool.getParallelism();  // Usually CPU cores - 1

// Parallel stream uses common pool internally
list.parallelStream().map(...).collect(...);
```

---

## Concurrency Utilities

<a id="q18"></a>
### Q18: What are the concurrency utilities in java.util.concurrent?
**Answer:**

| Category | Classes | Purpose |
|----------|---------|---------|
| **Executors** | ExecutorService, ThreadPoolExecutor | Thread pool management |
| **Locks** | ReentrantLock, ReadWriteLock, StampedLock | Fine-grained locking |
| **Synchronizers** | CountDownLatch, CyclicBarrier, Semaphore, Phaser | Thread coordination |
| **Concurrent Collections** | ConcurrentHashMap, CopyOnWriteArrayList, BlockingQueue | Thread-safe collections |
| **Atomic Variables** | AtomicInteger, AtomicReference, LongAdder | Lock-free operations |
| **Future** | Future, CompletableFuture | Async results |

**BlockingQueue:**
```java
BlockingQueue<String> queue = new LinkedBlockingQueue<>(10);
queue.put("item");        // Blocks if full
String item = queue.take();  // Blocks if empty
```

<a id="q19"></a>
### Q19: What is a CountDownLatch, CyclicBarrier, and Semaphore?
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

<a id="q20"></a>
### Q20: What are Atomic classes and when should you use them?
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

<a id="q21"></a>
### Q21: What is Compare-And-Swap (CAS)?
**Answer:**
CAS is a CPU instruction that:
1. Reads current value
2. Compares with expected value
3. If equal, updates to new value atomically
4. If not equal, operation fails (retry logic)

This is more efficient than locking as it doesn't block threads.

---

## Memory

<a id="q22"></a>
### Q22: What is the difference between Heap and Stack memory?
**Answer:**
| Stack | Heap |
|-------|------|
| Stores method calls and local variables | Stores objects and instance variables |
| LIFO (Last In First Out) | No specific order |
| Thread-specific | Shared among all threads |
| Faster access | Slower access |
| Fixed size (StackOverflowError) | Dynamic size (OutOfMemoryError) |
| Automatically managed | Managed by GC |

<a id="q23"></a>
### Q23: What is a memory leak in Java and how can you prevent it?
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

<a id="q24"></a>
### Q24: What is the difference between StackOverflowError and OutOfMemoryError?
**Answer:**
- **StackOverflowError**: Stack memory exhausted, usually due to infinite recursion
- **OutOfMemoryError**: Heap memory exhausted, too many objects created, or memory leak

---

[← Back to Java Index](README.md)
