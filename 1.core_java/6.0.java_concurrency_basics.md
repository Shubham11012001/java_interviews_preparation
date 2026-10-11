# Java Multithreading & Concurrency: The Complete Interview Guide

A practical, architect-level tutorial explained in simple language — from your first thread to production-grade concurrent systems.

If you are preparing for Senior Java Backend Engineer, Lead Engineer, or Technical Architect interviews, multithreading is one of those topics where knowing definitions is not enough. Interviewers want to know how you would handle 1,000 requests per second, prevent race conditions, control resource usage, and diagnose a production system that suddenly becomes slow.

This guide builds your understanding step by step, using simple analogies, Java code, production scenarios, comparison tables, common mistakes, and interview questions.

## The roadmap

Part 1 — Thread fundamentals

Thread, Runnable, lifecycle, synchronization, locks, and visibility.

Part 2 — Managing concurrent work

ExecutorService, ThreadPoolExecutor, Callable, Future, ForkJoinPool, and Java 21 virtual threads.

Part 3 — Coordination and thread safety

Latches, barriers, semaphores, concurrent collections, and atomic classes.

Part 4 — Production problems

Deadlocks, starvation, livelocks, producer-consumer, backpressure, and architecture decisions.

# 1. First, understand concurrency

Imagine you own an online shopping application.

At the same time:

* Customer A searches for a product.

* Customer B places an order.

* Customer C checks their payment status.

If your application handles one task at a time, customers may wait unnecessarily. Multithreading allows multiple tasks to make progress concurrently.

## 1.1 Process vs. thread

A process is a running program. A thread is a path of execution inside that program.

Think of a restaurant:

* Process = the restaurant.

* Threads = the workers.

* Shared memory = the kitchen and ingredients.

* Locks = rules for accessing shared equipment.

![Restaurant professional cooking kitchen, commercial chef and](https://images.openai.com/static-rsc-4/m4FC0iqWSkIPwRKo1mognf5fm3qzQYHZ6GZQ92XQW4TiN60NZT-01g8Qiik2kDi_jhtZC_EPDK6WNvTRR4WflDrs0xc_K7j-Qao7p_5ODwVtz1p5mldizz3BIrt3puIPG3Fmv1cNFeiNbnDjAIwKe0xVFNXdIPAy8eAoZrXUUdo?purpose=inline)

Add to Favorites

[designbundles](https://designbundles.net/ladadikart/2580736-restaurant-professional-cooking-kitchen-commercial)

| Concept        | Simple meaning                                            | Java example                           |
| -------------- | --------------------------------------------------------- | -------------------------------------- |
| Process        | Running application                                       | Java application/JVM process           |
| Thread         | Independent execution path                                | `Thread`                               |
| Concurrency    | Multiple tasks making progress during overlapping periods | Multiple request handlers              |
| Parallelism    | Multiple tasks executing simultaneously                   | Threads running on different CPU cores |
| Shared state   | Data accessible by multiple threads                       | Shared `HashMap`, account balance      |
| Race condition | Result depends on execution timing                        | Two withdrawals overwrite each other   |

Important interview distinction: concurrency does not guarantee parallel execution. Two threads can take turns on one CPU core and still be concurrent.

## 1.2 Why multithreading exists

1. Improve responsiveness.

2. Perform independent tasks concurrently.

3. Overlap I/O waiting with useful work.

4. Utilize multiple CPU cores for CPU-intensive tasks.

5. Handle multiple application requests efficiently.

However, multithreading is not automatically faster. Threads require memory, scheduling, coordination, and sometimes expensive context switches.

### When should you use it?

| Workload                                 | Typical approach                                           |
| ---------------------------------------- | ---------------------------------------------------------- |
| CPU-intensive image processing           | Bounded pool sized around available CPU cores              |
| Blocking database or REST calls          | Bounded executor or carefully managed virtual threads      |
| Small independent asynchronous tasks     | `ExecutorService`                                          |
| Parallel divide-and-conquer calculations | `ForkJoinPool`                                             |
| Shared mutable state                     | Synchronization, locks, atomics, or concurrent collections |
| Simple sequential operation              | A normal method call                                       |

# 2. Creating threads: `Thread` vs. `Runnable`

Java provides multiple ways to execute work on another thread. The two fundamental concepts are `Thread` and `Runnable`.

## 2.1 Extending the Thread class

Java

```
class PaymentThread extends Thread {

    @Override
    public void run() {
        System.out.println(
            "Processing payment: " +
            Thread.currentThread().getName()
        );
    }
}

public class Main {
    public static void main(String[] args) {
        PaymentThread thread = new PaymentThread();
        thread.start();
    }
}
```

What happens?

1. We create a `PaymentThread` object.

2. We call `start()`.

3. The JVM schedules a new thread of execution.

4. The new thread executes `run()`.

Critical: calling `run()` directly does not start a new thread. It is just a normal method call.

Java

```
thread.start(); // Starts execution on another thread
thread.run();   // Executes in the current thread
```

Also, a Java `Thread` instance can be started only once. Starting it again throws `IllegalThreadStateException`.

## 2.2 Implementing Runnable

Java

```
class PaymentTask implements Runnable {

    @Override
    public void run() {
        System.out.println("Processing payment");
    }
}

public class Main {
    public static void main(String[] args) {
        Thread thread = new Thread(new PaymentTask());
        thread.start();
    }
}
```

Here, `PaymentTask` defines the work, while `Thread` provides the execution mechanism.

You can also use a lambda:

Java

```
Thread thread = new Thread(
    () -> System.out.println("Sending notification")
);

thread.start();
```

## 2.3 Thread vs. Runnable

| Feature                | `Thread`                                                                    | `Runnable`                                                     |
| ---------------------- | --------------------------------------------------------------------------- | -------------------------------------------------------------- |
| Purpose                | Represents a thread of execution                                            | Represents a task                                              |
| Inheritance            | Uses the single class inheritance slot                                      | Can implement multiple interfaces                              |
| Reusability            | Thread instance starts once                                                 | Task object can be submitted repeatedly, if its design permits |
| Separation of concerns | Task and execution are coupled                                              | Task and execution are separated                               |
| Executor compatibility | A `Thread` is not itself a `Runnable` task in the intended executor pattern | Directly accepted by executors                                 |

### Which should you use in production?

Prefer `Runnable` or `Callable` with an executor. Extending `Thread` is rarely necessary unless you specifically need to customize thread behavior.

A task should generally describe what to do, while the executor decides where and when to run it.

## 2.4 Runnable vs. Callable

`Runnable` does not return a result and cannot declare checked exceptions through its `run()` signature.

`Callable` returns a result and can throw checked exceptions.

Java

```
Runnable notificationTask = () -> {
    System.out.println("Notification sent");
};

Callable<Integer> calculationTask = () -> {
    return 10 + 20;
};
```

| Feature            | Runnable                       | Callable                     |
| ------------------ | ------------------------------ | ---------------------------- |
| Method             | `run()`                        | `call()`                     |
| Return value       | None                           | Yes                          |
| Checked exceptions | Cannot declare them in `run()` | Can declare them in `call()` |
| Common use         | Logging, notifications         | Calculations, data retrieval |

Interview question: Can a `Runnable` return a value?

Not through its `run()` method. You can still store a result in another object, but you must manage safe publication and synchronization yourself. If you need a task result, `Callable` with `Future` is usually the cleaner option.

# 3. Thread lifecycle and states

Java defines six official thread states in `Thread.State`.

Think of a worker who is created, waits for a job, works, sometimes waits for equipment, and eventually finishes.

NEW

Thread created, not started

RUNNABLE

Eligible to run or actually running

BLOCKED

Waiting for a monitor lock

WAITING

Waiting indefinitely for a condition

TIMED_WAITING

Waiting with a time limit

TERMINATED

The run method has finished or failed with an uncaught exception

The three middle states are not necessarily a single sequential stage. A thread can move between `RUNNABLE` and waiting states multiple times before terminating.

## 3.1 The six states

| State           | Meaning                                     | Example                                         |
| --------------- | ------------------------------------------- | ----------------------------------------------- |
| `NEW`           | Created but not started                     | `new Thread(task)`                              |
| `RUNNABLE`      | Ready to run or running                     | After `start()`                                 |
| `BLOCKED`       | Waiting to acquire a `synchronized` monitor | Another thread holds the required lock          |
| `WAITING`       | Waiting indefinitely for another action     | `join()`, `wait()`, or `park()`                 |
| `TIMED_WAITING` | Waiting for a specified duration            | `sleep(1000)`, timed `join()`                   |
| `TERMINATED`    | Execution has ended                         | `run()` returns or throws an uncaught exception |

Subtle interview point: Java does not expose a separate `RUNNING` state in `Thread.State`. Running and ready-to-run threads are both reported as `RUNNABLE`.

## 3.2 Important thread methods

Java

```
Thread.sleep(1000);  // Pauses the current thread temporarily
thread.join();       // Waits for another thread to terminate
thread.isAlive();    // Checks whether the thread has started and not terminated
Thread.currentThread(); // Returns the current thread
```

`Thread.sleep()` does not release monitors that the thread owns.

`Object.wait()` does release the monitor associated with the object on which it waits, and must be called while holding that monitor.

## 3.3 Example: join()

Java

```
Thread worker = new Thread(() -> {
    System.out.println("Worker started");
});

worker.start();
worker.join();

System.out.println("Worker completed");
```

The main thread waits for `worker` to terminate before printing the final message.

In real applications, prefer task coordination through executors, futures, and higher-level concurrency utilities where appropriate.

Interview question: How do you find a thread stuck in production?

* Capture thread dumps.

* Inspect thread states and stack traces.

* Look for repeated `BLOCKED` threads or long-running waits.

* Identify monitor ownership, lock cycles, slow external calls, and exhausted thread pools.

* Correlate the findings with CPU, latency, and application metrics.

# 4. The most important multithreading problem: Race conditions

Before learning locks and atomic classes, understand the problem they solve.

Suppose two threads withdraw money from the same account.

Java

```
class BankAccount {
    int balance = 1000;

    void withdraw(int amount) {
        if (balance >= amount) {
            balance = balance - amount;
        }
    }
}
```

Thread A withdraws ₹700. Thread B withdraws ₹500.

Both threads might read the original balance of ₹1,000 before either updates it.

A possible execution:

| Step | Thread A        | Thread B        |
| ---- | --------------- | --------------- |
| 1    | Reads ₹1,000    |                 |
| 2    |                 | Reads ₹1,000    |
| 3    | Calculates ₹300 |                 |
| 4    |                 | Calculates ₹500 |
| 5    | Writes ₹300     |                 |
| 6    |                 | Writes ₹500     |

The final balance becomes ₹500, even though two withdrawals were accepted. The problem is that the check and update were not performed as one protected operation.

This is a race condition.

## 4.1 What is a critical section?

A critical section is the part of a program that accesses shared mutable data and must obey a concurrency rule.

Java

```
if (balance >= amount) {
    balance -= amount;
}
```

Both the check and update belong to the same critical operation. Protecting only the subtraction would not be enough.

## 4.2 Three things every architect must understand

* Atomicity: an operation happens as one indivisible unit from the perspective of other threads.

* Visibility: changes made by one thread become visible to another under the Java Memory Model's guarantees.

* Ordering: operations are observed in an order consistent with the synchronization guarantees.

A solution that fixes only one of these may still be incorrect.

For example, making a field `volatile` improves visibility and constrains ordering, but does not make `balance--` atomic.

# 5. Synchronization: `synchronized` methods and blocks

Synchronization lets a thread acquire an object's monitor before entering a protected section.

Imagine a bathroom with one key. Whoever has the key can enter. Everyone else must wait until the key is released.

## 5.1 Synchronized method

Java

```
class Counter {
    private int count = 0;

    public synchronized void increment() {
        count++;
    }

    public synchronized int getCount() {
        return count;
    }
}
```

For an instance synchronized method, the lock is associated with `this`.

Only one thread at a time can hold that same object's monitor and execute a synchronized section guarded by it.

Important: two different `Counter` objects have different instance monitors.

## 5.2 Synchronized block

Java

```
class Inventory {
    private int stock = 100;

    public void purchase(int quantity) {
        synchronized (this) {
            if (stock >= quantity) {
                stock -= quantity;
            }
        }
    }
}
```

A synchronized block lets you protect only the critical section rather than an entire method.

This can be useful when the method also performs unrelated work that does not require the lock.

However, keep the protected section small for a reason: to reduce contention while preserving correctness. Never split a logically atomic operation across separate locks without proving the design is correct.

## 5.3 Instance lock vs. class lock

Java

```
public synchronized void instanceMethod() {
    // Locks this object
}

public static synchronized void staticMethod() {
    // Locks Counter.class
}
```

A static synchronized method uses the monitor of the corresponding `Class` object.

| Feature             | Instance synchronized                                             | Static synchronized                                      |
| ------------------- | ----------------------------------------------------------------- | -------------------------------------------------------- |
| Lock                | `this`                                                            | `ClassName.class`                                        |
| Protects            | State associated with an instance, if all access follows the rule | Class-level shared state, if all access follows the rule |
| Different instances | Can execute concurrently                                          | Contend on the same class monitor                        |

## 5.4 How synchronized provides visibility

When one thread exits a synchronized block, its actions before releasing the monitor happen-before a subsequent successful acquisition of that same monitor.

That means the second thread gets the required visibility guarantee for the protected data.

This is a Java Memory Model guarantee, not simply a side effect of a thread being paused.

## 5.5 Best and worst cases

Best cases

* Protecting a small shared counter.

* Updating related fields together.

* Guarding a simple in-memory invariant.

* Making compound operations atomic.

Worst cases

* Holding a lock while calling a slow external API.

* Performing database queries inside a shared critical section.

* Acquiring multiple locks without a consistent ordering policy.

* Using one global lock for unrelated operations, creating unnecessary contention.

## 5.6 Common interview questions

Q: Can a synchronized method call another synchronized method on the same object?

Yes. Java's intrinsic monitors are reentrant. A thread that already owns the monitor can acquire it again.

Q: Does `synchronized` make a whole class thread-safe?

No. It protects only operations guarded by the relevant monitor. Every access path that can violate the invariant must follow the synchronization policy.

Q: Can two synchronized methods execute concurrently?

* On the same object, they cannot simultaneously execute if both require that object's monitor.

* On different instances, they can.

* Static synchronized methods use the class monitor, not the instance monitor.

Q: Does synchronized guarantee fairness?

No. Java intrinsic monitors do not promise a fair acquisition order.

# 6. `volatile`: visibility without compound atomicity

`volatile` is one of the most frequently misunderstood Java keywords.

Think of it as a shared noticeboard where reads and writes follow specific visibility and ordering rules. It does not turn a multi-step operation into a single indivisible operation.

## 6.1 Example: stopping a worker

Java

```
class Worker implements Runnable {
    private volatile boolean running = true;

    @Override
    public void run() {
        while (running) {
            // Perform work
        }
    }

    public void stop() {
        running = false;
    }
}
```

The `volatile` field ensures that a thread reading `running` observes writes according to volatile's memory semantics.

Without appropriate synchronization, a thread might not observe another thread's update as intended.

## 6.2 Why volatile does not fix a counter

Java

```
private volatile int count = 0;

public void increment() {
    count++;
}
```

This remains unsafe for concurrent increments.

`count++` involves reading, adding, and writing. Two threads can interleave those steps.

Use an atomic class or a suitable lock instead.

## 6.3 When to use volatile

* A simple status flag.

* A shutdown signal.

* Publishing a new immutable configuration reference.

* A field independently read and written where no compound invariant must be protected.

Avoid using it as a replacement for locks when several fields must change together.

## 6.4 `synchronized` vs. `volatile`

| Feature                       | `volatile`                          | `synchronized`                              |
| ----------------------------- | ----------------------------------- | ------------------------------------------- |
| Visibility                    | Yes                                 | Yes, through monitor synchronization        |
| Atomic compound operations    | No                                  | Yes, if the entire operation is guarded     |
| Mutual exclusion              | No                                  | Yes                                         |
| Blocking to acquire a monitor | No                                  | May block                                   |
| Best use                      | Independent status/reference fields | Multi-step operations and shared invariants |

# 7. ReentrantLock: more control over locking

`ReentrantLock` is an explicit lock from `java.util.concurrent.locks`.

It offers capabilities beyond intrinsic monitors, including optional fairness, interruptible acquisition, timed acquisition, and multiple condition queues.

## 7.1 Basic example

Java

```
import java.util.concurrent.locks.ReentrantLock;

class Account {
    private final ReentrantLock lock = new ReentrantLock();
    private int balance = 1000;

    public void withdraw(int amount) {
        lock.lock();
        try {
            if (balance >= amount) {
                balance -= amount;
            }
        } finally {
            lock.unlock();
        }
    }
}
```

The `finally` block is essential. If an exception occurs, the lock still needs to be released.

For an ordinary exclusive lock, every successful acquisition must be matched by a release by the owning thread.

## 7.2 Why is it called reentrant?

Suppose a thread acquires a lock and calls another method that acquires the same lock. It can acquire that lock again without deadlocking itself.

Java

```
lock.lock();
try {
    updateAccount(); // This method may acquire the same lock
} finally {
    lock.unlock();
}
```

Each acquisition must still be matched by an unlock. Reentrancy does not mean the lock can be released only once.

## 7.3 Important ReentrantLock methods

| Method                | Purpose                                          |
| --------------------- | ------------------------------------------------ |
| `lock()`              | Acquires the lock, waiting if necessary          |
| `unlock()`            | Releases one hold                                |
| `tryLock()`           | Attempts to acquire without waiting indefinitely |
| `tryLock(time, unit)` | Attempts acquisition with a timeout              |
| `lockInterruptibly()` | Allows interruption while waiting to acquire     |
| `isLocked()`          | Diagnostic observation of lock state             |
| `newCondition()`      | Creates a condition associated with the lock     |

Example with timeout:

Java

```
if (lock.tryLock(2, TimeUnit.SECONDS)) {
    try {
        updateAccount();
    } finally {
        lock.unlock();
    }
} else {
    // Handle inability to acquire within the timeout
}
```

The timed version can throw `InterruptedException`, so production code should handle interruption deliberately.

Architect insight: a timeout is not automatically a correctness solution. Decide what should happen if the lock is unavailable: reject, retry, return a fallback, or move the work to a queue.

## 7.4 Fair vs. non-fair locks

Java

```
ReentrantLock fairLock = new ReentrantLock(true);
ReentrantLock nonFairLock = new ReentrantLock(false);
```

* A fair lock generally favors threads that have been waiting longest.

* A non-fair lock may allow a new thread to acquire the lock ahead of waiting threads.

Fairness can reduce some forms of starvation, but it can also reduce throughput. Fairness does not guarantee an absolute end-to-end response-time bound.

When should you use ReentrantLock?

Use it when you specifically need timed or interruptible acquisition, multiple conditions, or explicit locking policies. Prefer `synchronized` for simpler cases where its behavior is sufficient.

# 8. ReadWriteLock: many readers, one writer

Suppose an application stores product configuration in memory.

Thousands of requests read the configuration, but updates happen only occasionally.

A normal exclusive lock allows only one reader at a time. A read-write lock can allow multiple readers concurrently, provided no writer holds the write lock.

## 8.1 Example

Java

```
import java.util.concurrent.locks.*;

class ProductCatalog {
    private final ReadWriteLock lock =
        new ReentrantReadWriteLock();

    private String configuration = "v1";

    public String readConfiguration() {
        lock.readLock().lock();
        try {
            return configuration;
        } finally {
            lock.readLock().unlock();
        }
    }

    public void updateConfiguration(String newValue) {
        lock.writeLock().lock();
        try {
            configuration = newValue;
        } finally {
            lock.writeLock().unlock();
        }
    }
}
```

The read lock is shared; the write lock is exclusive with respect to both readers and writers.

## 8.2 When is it useful?

* Read-heavy in-memory metadata.

* Configuration registries.

* Shared data structures with frequent reads and infrequent writes.

## 8.3 When is it a poor choice?

* When writes are frequent.

* When the critical section is so short that locking overhead dominates.

* When a concurrent collection already provides the required behavior.

* When long-held readers delay writers.

`ReentrantReadWriteLock` does not promise fairness by default. It supports a fairness option, but that may reduce throughput.

### Lock comparison

| Feature                     | `synchronized`                                           | `ReentrantLock`                 | `ReadWriteLock`                        |
| --------------------------- | -------------------------------------------------------- | ------------------------------- | -------------------------------------- |
| Basic mutual exclusion      | Yes                                                      | Yes                             | Write lock                             |
| Multiple concurrent readers | No, for the same monitor                                 | No, for the same exclusive lock | Yes                                    |
| Timed acquisition           | Not directly                                             | Yes                             | Available through its lock objects     |
| Interruptible acquisition   | No direct equivalent while entering a synchronized block | Yes                             | Yes through lock methods               |
| Explicit conditions         | `wait`/`notify` on monitor                               | `Condition`                     | Conditions available on the write lock |
| Typical use                 | Simple critical sections                                 | Advanced lock control           | Read-heavy shared state                |

# 9. Executor Framework: the production standard

Now we reach one of the most important topics for backend interviews.

Imagine your Spring Boot API receives 1,000 requests per second. Each request needs to call an external service that takes approximately 500 milliseconds.

Should you create a new operating-system-backed platform thread for every operation?

Not necessarily. Unbounded thread creation can exhaust memory and operating-system resources, while excessive concurrency can overload the downstream service.

The Executor Framework separates task submission from the mechanism used to execute the task.

## 9.1 The architecture

Spring Boot request handlers

Submit work

ExecutorService

Accepts tasks and manages execution

Thread pool

Workers execute tasks; queued tasks wait for capacity

Database / REST API / background job

## 9.2 Executor vs. ExecutorService

`Executor` is the basic interface for executing a `Runnable`.

`ExecutorService` adds lifecycle management, task submission that can return a `Future`, and bulk operations such as `invokeAll()`.

Java

```
Executor executor = command -> new Thread(command).start();

executor.execute(() -> System.out.println("Task running"));
```

The example illustrates the interface, but creating a new thread for every task is not a sensible general-purpose production policy.

Using an executor service:

Java

```
ExecutorService executor =
    Executors.newFixedThreadPool(4);

executor.submit(() -> {
    System.out.println("Processing order");
});
```

In production, remember to shut down an executor that your component owns. A shared application-managed executor should instead follow the application's lifecycle.

## 9.3 Common executor factory methods

| Factory                     | Typical behavior                                              | Main concern                                                           |
| --------------------------- | ------------------------------------------------------------- | ---------------------------------------------------------------------- |
| `newFixedThreadPool(n)`     | Fixed worker count                                            | Uses an unbounded shared queue                                         |
| `newSingleThreadExecutor()` | One worker                                                    | Tasks can accumulate in an unbounded queue                             |
| `newCachedThreadPool()`     | Creates workers as needed and reuses idle ones                | Can create a very large number of platform threads                     |
| `newScheduledThreadPool(n)` | Schedules delayed and periodic tasks                          | Long-running tasks can delay other scheduled work                      |
| `newWorkStealingPool()`     | Uses work-stealing execution, commonly through `ForkJoinPool` | Not intended as a universal replacement for a bounded request executor |

These methods are convenient, but for workloads with strict resource limits, explicit `ThreadPoolExecutor` configuration is often preferable.

# 10. ThreadPoolExecutor: understand every setting

`ThreadPoolExecutor` gives you direct control over worker counts, queueing, thread creation, and overload behavior.

Java

```
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    10,                          // corePoolSize
    50,                          // maximumPoolSize
    60, TimeUnit.SECONDS,        // keepAliveTime
    new ArrayBlockingQueue<>(100), // bounded queue
    new ThreadPoolExecutor.CallerRunsPolicy()
);
```

This is an illustrative configuration, not a universally safe set of production values. The right numbers depend on workload measurements, downstream capacity, latency objectives, and memory limits.

## 10.1 Understand the five main settings

| Setting                    | Meaning                                                           |
| -------------------------- | ----------------------------------------------------------------- |
| `corePoolSize`             | Normal worker count maintained by the executor                    |
| `maximumPoolSize`          | Maximum worker count allowed                                      |
| `keepAliveTime`            | How long eligible excess idle workers may remain before removal   |
| `workQueue`                | Holds tasks waiting for workers                                   |
| `RejectedExecutionHandler` | Determines what happens when the executor cannot accept more work |

Workers beyond the core size are typically removed after remaining idle for the configured keep-alive duration. Core thread timeout can also be enabled.

## 10.2 The execution order — a favorite interview question

For a typical `ThreadPoolExecutor`, submitting a task follows this logic:

1. If fewer than `corePoolSize` workers exist, try to create a worker for the task.

2. Otherwise, try to enqueue the task.

3. If the queue cannot accept the task, try to create another worker up to `maximumPoolSize`.

4. If neither queuing nor worker creation succeeds, reject the task.

This means the executor does not normally fill all 50 workers before using the queue.

For example, with core size 10, maximum size 50, and queue capacity 100:

* The first 10 tasks may start on core workers.

* The next 100 tasks may wait in the queue.

* If the queue fills, additional tasks can trigger growth toward 50 workers.

* If all workers are busy and the queue is full, the rejection policy applies.

This is one of the most misunderstood executor behaviors.

## 10.3 Why bounded queues matter

An unbounded queue can grow indefinitely when incoming work exceeds processing capacity.

That can cause:

* Increasing memory consumption.

* Long queue delays.

* Requests timing out before they even begin.

* A delayed but massive burst of work hitting the database or external API.

A bounded queue makes overload visible and forces you to define a policy.

## 10.4 Rejection policies

| Policy                | Behavior                                                                 | Trade-off                                                  |
| --------------------- | ------------------------------------------------------------------------ | ---------------------------------------------------------- |
| `AbortPolicy`         | Throws `RejectedExecutionException`                                      | Explicit failure; caller must handle it                    |
| `CallerRunsPolicy`    | Runs the task in the submitting thread, unless the executor is shut down | Slows the producer and may move work onto a request thread |
| `DiscardPolicy`       | Silently discards a task                                                 | Can lose work                                              |
| `DiscardOldestPolicy` | Removes the queue head and retries submission                            | Can discard previously accepted queued work                |

For payments, orders, or other critical work, silently dropping tasks is generally unacceptable unless loss is explicitly part of the design.

`CallerRunsPolicy` can act as a simple form of feedback-based throttling, but it can also cause request threads to block. If the executor has shut down, it does not run the task; the task is discarded.

## 10.5 How do you size a thread pool?

There is no magic number.

For CPU-bound work, a starting point near the available processor count may be reasonable, followed by measurement.

For blocking I/O work, more concurrent tasks may be useful because many workers spend time waiting. But the correct limit is also constrained by:

* Database connection pool size.

* Downstream service concurrency limits.

* Request deadlines.

* Memory per task.

* CPU consumed per task.

* Queueing delay and acceptable latency.

A useful approximation from queueing theory is Little's Law:

L=λWL = \lambda WL=λW

Where:

* LLL = average number of tasks in the system.

* λ\lambdaλ = average arrival rate.

* WWW = average time spent in the system.

For example, if 100 tasks per second arrive and average end-to-end time is 0.5 seconds, the average number of tasks in the system is approximately 50.

That does not automatically mean you need 50 platform threads. The number of concurrently executing tasks depends on the workload, and tasks may spend much of their time queued or waiting for external resources.

## 10.6 Production monitoring checklist

Monitor at least:

* Active worker count.

* Current pool size and maximum pool size.

* Queue size and queue wait time.

* Task completion rate.

* Rejected task count.

* Task execution latency.

* Downstream latency and timeout rate.

* Database connection pool utilization.

* CPU and memory consumption.

Architect-level principle: increasing a thread pool may make a bottleneck worse if the real limit is the database or downstream API.

# 11. Virtual threads in Java 21+

Because modern Java interviews often include virtual threads, do not stop at traditional pools.

A virtual thread is a lightweight Java thread managed by the JVM rather than mapping one-to-one to an operating-system thread throughout its lifetime.

Java

```
try (ExecutorService executor =
         Executors.newVirtualThreadPerTaskExecutor()) {

    Future<String> future = executor.submit(() -> {
        return callExternalService();
    });

    System.out.println(future.get());
}
```

This executor creates a new virtual thread for each submitted task rather than maintaining a fixed-size worker pool.

## 11.1 When are virtual threads useful?

They are particularly helpful for applications with many concurrent tasks that spend substantial time waiting on blocking I/O, such as REST calls or JDBC operations.

Benefits include:

* Lower per-thread overhead than platform threads.

* A straightforward thread-per-task programming model.

* Less need to optimize application code around scarce platform-thread counts.

## 11.2 When are they not a magic solution?

Virtual threads do not:

* Make CPU-bound calculations execute faster by themselves.

* Increase the database's maximum connection capacity.

* Eliminate race conditions.

* Remove the need for timeouts or backpressure.

* Make an unbounded number of downstream operations safe.

They also have runtime and resource constraints of their own. Some blocking or native operations may pin a virtual thread to its carrier platform thread; behavior depends on the JDK version and operation.

### Platform threads vs. virtual threads vs. a fixed pool

| Situation                                  | Typical starting point                  |
| ------------------------------------------ | --------------------------------------- |
| CPU-heavy work                             | Bounded platform-thread pool            |
| Large number of blocking I/O tasks         | Virtual threads, with downstream limits |
| Need explicit bounded worker concurrency   | `ThreadPoolExecutor`                    |
| Small background task set                  | Application-managed executor            |
| CPU-intensive recursive divide-and-conquer | `ForkJoinPool`                          |

For virtual-thread applications, limit expensive resources such as database connections with an appropriate connection pool or semaphore, rather than using a small virtual-thread pool as the primary resource limiter.

# 12. Callable and Future: getting results from background tasks

A `Future` represents the result of a computation that may not have finished yet.

Imagine sending three reports to three workers. You want to collect each result after they finish.

## 12.1 Basic example

Java

```
ExecutorService executor =
    Executors.newFixedThreadPool(3);

try {
    Callable<Integer> task = () -> 100 + 200;

    Future<Integer> future = executor.submit(task);

    Integer result = future.get();

    System.out.println(result); // 300
} finally {
    executor.shutdown();
}
```

`submit()` returns immediately with a `Future` in the usual asynchronous case. Calling `get()` waits until the result is available or the computation fails or the wait is interrupted.

## 12.2 Important Future methods

| Method               | Meaning                                                       |
| -------------------- | ------------------------------------------------------------- |
| `get()`              | Waits for completion and retrieves the result                 |
| `get(timeout, unit)` | Waits up to a limit                                           |
| `isDone()`           | Checks whether the task completed, failed, or was cancelled   |
| `cancel(true)`       | Attempts cancellation and requests interruption if applicable |
| `isCancelled()`      | Checks whether cancellation succeeded                         |

Cancellation is cooperative. Interrupting a thread does not forcibly terminate arbitrary code.

Tasks should respond appropriately to interruption, and blocking operations should be handled with their interruption semantics in mind.

## 12.3 The danger of calling `get()` too early

Java

```
Future<String> future = executor.submit(this::slowCall);

String result = future.get(); // Blocks here
```

The task may be asynchronous, but immediately waiting for it can eliminate the benefit of doing other work concurrently.

A better pattern is to submit independent tasks, perform other useful work, and then collect results.

## 12.4 Exception handling

With `submit()`, task exceptions are captured by the `Future`. Calling `get()` can throw `ExecutionException`, whose cause represents the underlying failure.

You must also consider:

* `InterruptedException` — the waiting thread was interrupted.

* `TimeoutException` — a timed wait expired.

* `CancellationException` — the task was cancelled.

When catching `InterruptedException`, a common practice is to restore the interrupt status if you cannot propagate the exception:

Java

```
try {
    return future.get();
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
    throw new IllegalStateException("Operation interrupted", e);
} catch (ExecutionException e) {
    throw new IllegalStateException("Task failed", e.getCause());
}
```

This is only an example of exception translation; production code should use the application's appropriate error types and cancellation behavior.

## 12.5 Future vs. CompletableFuture

| Feature                   | Future              | CompletableFuture                         |
| ------------------------- | ------------------- | ----------------------------------------- |
| Retrieve result           | `get()`             | `get()` or `join()`                       |
| Compose dependent tasks   | Limited             | `thenApply`, `thenCompose`, and others    |
| Combine independent tasks | Manual coordination | `allOf`, `anyOf`                          |
| Callback-style processing | Not built in        | Supported                                 |
| Exception composition     | Manual handling     | `exceptionally`, `handle`, `whenComplete` |

`CompletableFuture` is particularly useful for composing asynchronous workflows, but its default async methods commonly use the shared common pool. For blocking I/O, choose an appropriate executor rather than assuming the common pool is suitable.

# 13. ForkJoinPool: divide a big task into smaller tasks

Suppose you need to sum one million numbers.

Instead of processing everything in one task, you can divide the work:

1. Split the array into halves.

2. Split each half again.

3. Calculate small chunks.

4. Combine the partial results.

This is the divide-and-conquer approach.

`ForkJoinPool` is designed for tasks that can split into smaller subtasks and combine their results. It uses work stealing: idle workers can take tasks from other workers' queues.

## 13.1 Conceptual example

Java

```
class SumTask extends RecursiveTask<Long> {

    private final long[] numbers;
    private final int start;
    private final int end;

    SumTask(long[] numbers, int start, int end) {
        this.numbers = numbers;
        this.start = start;
        this.end = end;
    }

    @Override
    protected Long compute() {
        if (end - start <= 1_000) {
            long sum = 0;

            for (int i = start; i < end; i++) {
                sum += numbers[i];
            }

            return sum;
        }

        int middle = (start + end) / 2;

        SumTask left =
            new SumTask(numbers, start, middle);
        SumTask right =
            new SumTask(numbers, middle, end);

        left.fork();
        long rightResult = right.compute();
        long leftResult = left.join();

        return leftResult + rightResult;
    }
}
```

The threshold of 1,000 is illustrative. The best chunk size depends on the computation and overhead.

## 13.2 RecursiveTask vs. RecursiveAction

* `RecursiveTask<V>` returns a result.

* `RecursiveAction` performs work without returning a result.

## 13.3 When should you use ForkJoinPool?

Good fits

* Parallel array operations.

* Divide-and-conquer algorithms.

* Recursive computations with reasonably independent subtasks.

* CPU-intensive parallel processing.

Poor fits

* Large numbers of tasks blocked on slow external services.

* Work that spends most of its time waiting on a database.

* Tasks that are too small to justify splitting overhead.

* Tasks that frequently block and prevent pool workers from making progress.

ForkJoinPool has mechanisms intended to help with certain blocking situations, but arbitrary blocking can still harm parallelism. For many blocking I/O workloads, virtual threads or a deliberately configured executor are generally easier to reason about.

Interview question: Why does work stealing improve utilization?

Because an idle worker can take available work rather than remaining idle while another worker has a large backlog of small tasks.

# 14. CountDownLatch: wait until tasks finish

Imagine an application must fetch data from three services before creating a dashboard:

* Customer service.

* Order service.

* Payment service.

The dashboard should be generated only after all three tasks finish.

`CountDownLatch` is designed for this one-time coordination.

## 14.1 Example

Java

```
CountDownLatch latch = new CountDownLatch(3);

ExecutorService executor = Executors.newFixedThreadPool(3);

try {
    for (int i = 0; i < 3; i++) {
        executor.submit(() -> {
            try {
                fetchData();
            } finally {
                latch.countDown();
            }
        });
    }

    latch.await();

    System.out.println("All tasks completed");
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
} finally {
    executor.shutdown();
}
```

The `finally` block ensures that each task decrements the latch even when `fetchData()` throws. In a real application, you should also record task failures; reaching zero does not mean every task succeeded.

## 14.2 Important methods

* `countDown()` — reduces the count by one.

* `await()` — waits until the count reaches zero.

* `await(timeout, unit)` — waits for zero up to a time limit.

* `getCount()` — observes the current count.

Important: a `CountDownLatch` cannot be reset. Once it reaches zero, later calls to `await()` do not block.

### Production use cases

* Waiting for several startup tasks.

* Waiting for parallel test workers.

* Coordinating independent service responses.

* Testing race conditions deterministically.

### Common mistake

If a task never calls `countDown()`, waiters may block forever unless a timeout or other failure-handling mechanism is used.

# 15. CyclicBarrier: wait for everyone at the same point

Imagine four workers each calculate one part of a simulation. Nobody can start the next simulation round until all four finish the current round.

That is the role of `CyclicBarrier`.

## 15.1 Example

Java

```
CyclicBarrier barrier = new CyclicBarrier(3, () -> {
    System.out.println("All workers finished this round");
});

Runnable worker = () -> {
    try {
        for (int round = 1; round <= 2; round++) {
            calculate(round);
            barrier.await();
        }
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    } catch (BrokenBarrierException e) {
        // Handle failed coordination
    }
};
```

Submit three copies of `worker` to an executor with enough available threads.

All three must reach the barrier in each round. When the required parties arrive, they are released, and the barrier can be reused.

## 15.2 CountDownLatch vs. CyclicBarrier

| Feature                    | CountDownLatch                                         | CyclicBarrier                                                  |
| -------------------------- | ------------------------------------------------------ | -------------------------------------------------------------- |
| Primary purpose            | Wait for a count to reach zero                         | Wait for a group of parties to meet                            |
| Reusable                   | No                                                     | Yes                                                            |
| Who decrements or arrives? | Any thread can call `countDown()`                      | Each participating party calls `await()`                       |
| Typical scenario           | Wait for startup tasks                                 | Synchronize simulation rounds                                  |
| Failure behavior           | Waiters can remain blocked if count never reaches zero | Interruption, timeout, or failed arrival can break the barrier |

A barrier can become broken. Code should handle `BrokenBarrierException`, interruption, and timeouts where applicable.

# 16. Semaphore: limit concurrent access

Imagine a service has a database with a limited connection budget. You do not want hundreds of tasks simultaneously calling an expensive operation.

A semaphore is like a set of entry tickets. A task must acquire a permit before entering a protected region and release it afterward.

## 16.1 Example

Java

```
Semaphore semaphore = new Semaphore(10);

public void callExternalService() throws InterruptedException {
    semaphore.acquire();
    try {
        externalServiceCall();
    } finally {
        semaphore.release();
    }
}
```

This allows at most ten callers to hold permits simultaneously through this semaphore instance.

If a caller cannot acquire a permit, it waits. You can also use `tryAcquire()` or timed acquisition to define a different overload behavior.

## 16.2 When should you use Semaphore?

* Limit concurrent calls to a downstream API.

* Limit concurrent resource-intensive jobs.

* Protect a service with a known concurrency budget.

* Implement a simple admission-control mechanism.

## 16.3 Common mistakes

* Forgetting to release a permit after an exception.

* Releasing more permits than were acquired.

* Assuming the semaphore limits all application instances when it exists only in one JVM.

* Using a semaphore without defining what should happen to callers that cannot acquire permits quickly enough.

A semaphore is not a replacement for a database connection pool. If the database connection pool already has a limit, an additional semaphore may be unnecessary unless it serves a separate purpose.

### Fair semaphore

Java

```
Semaphore semaphore = new Semaphore(10, true);
```

The fairness option favors queued waiters in FIFO order for acquisition. Fairness can come at a throughput cost, and it does not guarantee an absolute completion deadline.

# 17. Exchanger: swap data between two threads

`Exchanger<V>` lets two threads meet and exchange values.

Think of two workers who each have a box of intermediate data and exchange boxes when they meet.

## 17.1 Example

Java

```
Exchanger<String> exchanger = new Exchanger<>();

// Worker A
String dataA = "Data from A";
String receivedByA = exchanger.exchange(dataA);

// Worker B
String dataB = "Data from B";
String receivedByB = exchanger.exchange(dataB);
```

Worker A receives `"Data from B"`, and worker B receives `"Data from A"`.

The exchange blocks until another party arrives, unless a timed operation is used. Interruption is also possible.

### When is it useful?

* Two-stage data processing with paired workers.

* Certain buffer-exchange designs.

* Specialized producer-consumer pipelines.

It is much less commonly needed than executors, latches, semaphores, or blocking queues.

# 18. A quick comparison of coordination utilities

This is a useful interview revision table.

| Utility          | Think of it as                    | Main purpose                                       |
| ---------------- | --------------------------------- | -------------------------------------------------- |
| `CountDownLatch` | Countdown clock                   | Wait for a fixed number of events                  |
| `CyclicBarrier`  | Meeting point                     | Wait for a group to reach the same phase           |
| `Semaphore`      | Limited entry tickets             | Limit concurrent access                            |
| `Exchanger`      | Two-person swap                   | Exchange data between two threads                  |
| `BlockingQueue`  | Shared conveyor belt              | Transfer and buffer tasks or data                  |
| `ReentrantLock`  | A lock with extra controls        | Exclusive access with advanced acquisition options |
| `ReadWriteLock`  | Shared reading, exclusive writing | Support concurrent reads with coordinated writes   |
| `AtomicInteger`  | Safe counter                      | Perform atomic operations on a value               |

Architect tip: prefer the highest-level utility that directly expresses your coordination requirement. Lower-level synchronization primitives increase the amount of correctness you must prove.

# 19. Thread safety in Java data structures

This is a major interview topic because many applications share maps, lists, sets, and counters across threads.

A collection is not automatically thread-safe just because it comes from the Java Collections Framework.

## 19.1 HashMap vs. ConcurrentHashMap

Java

```
Map<String, Integer> map = new HashMap<>();
```

`HashMap` is not safe for unsynchronized concurrent modification.

For concurrent access:

Java

```
ConcurrentHashMap<String, Integer> map =
    new ConcurrentHashMap<>();

map.put("orders", 10);

Integer count = map.get("orders");
```

`ConcurrentHashMap` is designed for concurrent access and provides thread-safe individual operations. Its iterators are weakly consistent: they can reflect some concurrent updates without throwing `ConcurrentModificationException` merely because the map is modified concurrently.

But there is a subtle problem.

Java

```
if (!map.containsKey("customer-1")) {
    map.put("customer-1", 100);
}
```

The two calls are individually thread-safe, but the overall check-then-act sequence is not atomic.

Use an atomic map operation when it expresses your intent:

Java

```
map.putIfAbsent("customer-1", 100);
```

Or use `computeIfAbsent()` for suitable atomic per-key initialization.

Be careful: a mapping function should generally be short and should not perform complicated updates to the same map.

## 19.2 Common collection comparison

| Collection                          | Thread safety                                 | Best use                                             |
| ----------------------------------- | --------------------------------------------- | ---------------------------------------------------- |
| `ArrayList`                         | Not inherently thread-safe                    | Single-threaded or externally synchronized access    |
| `LinkedList`                        | Not inherently thread-safe                    | Specialized list/deque use                           |
| `HashMap`                           | Not inherently thread-safe                    | General map in single-threaded or protected use      |
| `ConcurrentHashMap`                 | Designed for concurrent access                | Shared maps                                          |
| `Collections.synchronizedList(...)` | Synchronizes individual wrapper operations    | Simple shared-list scenarios                         |
| `CopyOnWriteArrayList`              | Thread-safe, copy-on-write updates            | Many reads, very few writes                          |
| `ConcurrentLinkedQueue`             | Thread-safe, non-blocking queue               | Concurrent queue without blocking capacity semantics |
| `BlockingQueue` implementations     | Thread-safe with blocking transfer operations | Producer-consumer pipelines                          |

## 19.3 Collections.synchronizedList: a subtle trap

Java

```
List<String> list =
    Collections.synchronizedList(new ArrayList<>());
```

Individual wrapper operations are synchronized. However, iteration generally requires external synchronization on the returned list.

Java

```
synchronized (list) {
    for (String item : list) {
        process(item);
    }
}
```

Without the external synchronization, concurrent modifications can interfere with iteration.

## 19.4 CopyOnWriteArrayList

Imagine a configuration list that is read 100,000 times but updated only a few times a day.

`CopyOnWriteArrayList` creates a new backing array for each mutation, making reads and iteration convenient while updates are expensive.

Use it when:

* Reads vastly outnumber writes.

* Lists are relatively small.

* Snapshot-style iteration is useful.

Avoid it for large, frequently modified lists because every mutation can require copying the array.

## 19.5 ConcurrentHashMap vs. synchronized HashMap

A synchronized map wrapper typically uses one shared monitor for its operations. `ConcurrentHashMap` is designed for greater concurrency and offers atomic compound map methods.

Choose based on your actual access pattern. Neither automatically makes arbitrary multi-step business workflows atomic.

# 20. Atomic classes: AtomicInteger, AtomicBoolean, and friends

Atomic classes are useful when you need thread-safe updates to a single value without using an explicit application-level lock.

They are part of `java.util.concurrent.atomic`.

## 20.1 AtomicInteger

Java

```
AtomicInteger counter = new AtomicInteger(0);

counter.incrementAndGet();

System.out.println(counter.get()); // 1
```

Common methods:

| Method                            | Meaning                                                     |
| --------------------------------- | ----------------------------------------------------------- |
| `get()`                           | Reads the value                                             |
| `set(value)`                      | Sets the value                                              |
| `incrementAndGet()`               | Increments, then returns the new value                      |
| `getAndIncrement()`               | Returns the old value, then increments                      |
| `addAndGet(n)`                    | Adds `n`, then returns the new value                        |
| `compareAndSet(expected, update)` | Updates only if the current value equals the expected value |

## 20.2 What is CAS?

CAS means Compare-And-Set or compare-and-swap, depending on the terminology used.

Conceptually:

1. Read the current value.

2. Check whether it still equals the expected value.

3. If so, update it atomically.

4. If not, report failure or retry.

Java

```
AtomicInteger counter = new AtomicInteger(10);

boolean updated = counter.compareAndSet(10, 20);
```

If the value was 10 at the point of the atomic comparison, it becomes 20 and `updated` is true. Otherwise, the update fails.

CAS-based operations can retry under contention. Atomic does not mean every operation is guaranteed to be lock-free in every implementation or circumstance.

## 20.3 AtomicBoolean

Java

```
AtomicBoolean started = new AtomicBoolean(false);

if (started.compareAndSet(false, true)) {
    initializeOnce();
}
```

This can ensure that only one competing thread wins the transition from false to true.

However, if `initializeOnce()` fails, the flag remains true unless you explicitly design recovery. Atomic state transitions do not automatically solve the whole business workflow.

## 20.4 AtomicLong and LongAdder

`AtomicLong` is useful for a thread-safe long counter.

`LongAdder` is often better for high-contention statistical counters because it spreads updates across internal cells and combines them when `sum()` is called.

| Feature                                  | AtomicLong                                 | LongAdder                                                   |
| ---------------------------------------- | ------------------------------------------ | ----------------------------------------------------------- |
| Atomic increment                         | Yes                                        | Supports concurrent increments                              |
| Read current total                       | Immediate atomic value                     | `sum()` combines accumulated cells                          |
| High contention                          | Updates can contend on one value           | Often scales better for frequent concurrent updates         |
| Exact atomic read-modify-write decisions | Supports CAS and related atomic operations | Not a replacement for atomic conditional business decisions |

Use `LongAdder` for metrics, such as request counts, where high update throughput matters and the sum need not serve as a transactional decision point.

## 20.5 The ABA problem

In CAS-based algorithms, a value may change from A to B and back to A. A thread that only checks whether the current value equals A may not detect the intermediate changes.

This is called the ABA problem.

`AtomicStampedReference` can pair a reference with a stamp to help detect changes, when the algorithm requires it.

This is an advanced topic, but mentioning it in an architect-level interview can demonstrate awareness of lock-free algorithm pitfalls.

## 20.6 Atomic vs. synchronized

| Situation                                     | Better starting point                                                                      |
| --------------------------------------------- | ------------------------------------------------------------------------------------------ |
| Incrementing a shared metric                  | `LongAdder` or `AtomicLong`                                                                |
| Atomic transition of a boolean flag           | `AtomicBoolean`                                                                            |
| Updating multiple related fields consistently | Lock or another appropriate coordinated design                                             |
| Concurrent map initialization                 | `ConcurrentHashMap.computeIfAbsent()` when appropriate                                     |
| Complex inventory reservation                 | A suitable lock, transactional database operation, or other design enforcing the invariant |

Remember: atomics protect atomic operations on particular variables. They do not automatically protect a business transaction involving multiple variables or external systems.

# 21. Deadlock, starvation, and livelock

These are related concurrency problems, but they are not the same.

Deadlock

Threads wait forever for locks held by one another.

Starvation

A thread repeatedly fails to get the resources or execution time it needs.

Livelock

Threads remain active, but their actions prevent meaningful progress.

## 21.1 Deadlock example

Thread A locks resource 1, then requests resource 2.

Thread B locks resource 2, then requests resource 1.

Neither can proceed.

Java

```
Object lock1 = new Object();
Object lock2 = new Object();

// Thread A
synchronized (lock1) {
    synchronized (lock2) {
        // Work
    }
}

// Thread B
synchronized (lock2) {
    synchronized (lock1) {
        // Work
    }
}
```

If the two threads interleave in the wrong way, a deadlock can occur.

### The four Coffman conditions

The classic necessary conditions for a deadlock are:

1. Mutual exclusion: a resource cannot be shared freely.

2. Hold and wait: a thread holds one resource while waiting for another.

3. No preemption: resources cannot simply be taken away from their owners.

4. Circular wait: a cycle of waiting threads exists.

Breaking at least one of these conditions can prevent this classic form of deadlock.

## 21.2 How to prevent deadlocks

* Establish a consistent global order for acquiring multiple locks.

* Avoid holding locks while making network calls or waiting on slow external resources.

* Use timed or interruptible lock acquisition where appropriate.

* Keep critical sections short.

* Reduce nested locking.

* Review code paths that acquire the same resources in different orders.

A timeout alone is not proof that a design is deadlock-free. Retries can themselves create load or correctness problems.

## 21.3 How to diagnose a deadlock

Capture a JVM thread dump and look for:

* Threads marked `BLOCKED`.

* Which monitors each thread owns and wants.

* Circular lock dependencies.

* Repeated stack traces.

* JVM-reported deadlock information.

Useful tools include `jcmd`, `jstack`, Java Flight Recorder, and a profiler such as VisualVM.

## 21.4 Starvation

Suppose a high-priority task repeatedly gets access to a resource while a lower-priority task never gets enough time to proceed.

That is starvation.

Possible causes include:

* Unfair lock acquisition.

* Long-held locks.

* Excessive high-priority work.

* Executor tasks that monopolize workers.

* A bounded resource being continually consumed by other tasks.

Possible remedies include fair locks where appropriate, workload isolation, bounded task execution, and a better scheduling or admission-control policy.

Fairness may help but does not guarantee that every task completes within a deadline.

## 21.5 Livelock

Imagine two people in a narrow corridor. Both repeatedly move left to let the other pass. Then both move right. They remain active but never pass.

In software, two retrying processes may repeatedly detect conflicts and back off in synchronized patterns, never making progress.

Solutions include randomized backoff, avoiding identical retry schedules, limiting retries, and designing conflict resolution so one participant can make progress.

### Deadlock vs. starvation vs. livelock

| Problem    | Are threads active?               | Does meaningful progress happen? |
| ---------- | --------------------------------- | -------------------------------- |
| Deadlock   | Usually blocked                   | No                               |
| Starvation | Some threads may continue working | Not for the starved thread       |
| Livelock   | Yes, repeatedly acting            | No useful progress overall       |

# 22. Producer-consumer problem: a real-world pattern

This is one of the most important concurrency interview problems.

Imagine a notification service:

* Producers receive notification requests.

* A queue stores pending notifications.

* Consumers send notifications to users.

The producer should not need to wait until an email or SMS is actually sent. It can submit work to a queue, and consumers process it at their own pace.

Producers

API requests / incoming events

Bounded BlockingQueue

Buffers work and applies backpressure

Consumers

Send notifications concurrently

## 22.1 Why use a queue?

Without a queue, producers may need to wait for consumers to finish.

With a queue, you can separate the rate at which work arrives from the rate at which it is processed.

A bounded queue is particularly useful because it has a limit. If producers are faster than consumers for long enough, the queue fills, and the system must slow down, reject work, or persist it elsewhere.

This is called backpressure or overload control, depending on the mechanism.

## 22.2 The simplest solution: BlockingQueue

Java provides `BlockingQueue` implementations such as:

* `ArrayBlockingQueue` — bounded array-backed queue.

* `LinkedBlockingQueue` — optionally bounded linked-node queue.

* `PriorityBlockingQueue` — priority ordering, but not bounded capacity.

* `SynchronousQueue` — direct handoff with no stored elements.

For most producer-consumer examples, a bounded `ArrayBlockingQueue` is a good starting point.

## 22.3 Complete example

Java

```
import java.util.concurrent.*;

public class ProducerConsumerExample {

    public static void main(String[] args)
            throws InterruptedException {

        BlockingQueue<String> queue =
            new ArrayBlockingQueue<>(100);

        ExecutorService executor =
            Executors.newFixedThreadPool(3);

        Runnable producer = () -> {
            try {
                for (int i = 1; i <= 20; i++) {
                    queue.put("Notification-" + i);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        };

        Runnable consumer = () -> {
            try {
                while (!Thread.currentThread().isInterrupted()) {
                    String task = queue.take();

                    if ("STOP".equals(task)) {
                        break;
                    }

                    sendNotification(task);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        };

        executor.submit(consumer);
        executor.submit(consumer);
        executor.submit(producer);

        // Simplified demonstration shutdown:
        // real applications need coordinated producer completion.
        executor.shutdown();
    }

    static void sendNotification(String task) {
        System.out.println("Sending " + task);
    }
}
```

This demonstrates the queue mechanics, but it is not a complete lifecycle-managed production implementation: shutting down the executor immediately does not guarantee all queued notifications are delivered. A production version must coordinate producer completion, consumer draining, cancellation, and failure recovery.

## 22.4 `put()` vs. `offer()` vs. `take()` vs. `poll()`

| Method                       | Behavior                                      |
| ---------------------------- | --------------------------------------------- |
| `put(item)`                  | Waits if a bounded queue is full              |
| `offer(item)`                | Returns false if it cannot insert immediately |
| `offer(item, timeout, unit)` | Waits for capacity up to a timeout            |
| `take()`                     | Waits until an item is available              |
| `poll()`                     | Returns immediately, possibly with null       |
| `poll(timeout, unit)`        | Waits for an item up to a timeout             |

Choose based on your overload policy. For example, a REST endpoint may prefer a timed offer followed by a clear rejection response rather than waiting indefinitely.

## 22.5 The traditional wait/notify solution

Interviewers sometimes ask you to implement producer-consumer using `wait()` and `notifyAll()`.

Java

```
class SharedBuffer {
    private final Queue<Integer> queue = new LinkedList<>();
    private final int capacity = 10;

    public synchronized void produce(int value)
            throws InterruptedException {

        while (queue.size() == capacity) {
            wait();
        }

        queue.add(value);
        notifyAll();
    }

    public synchronized int consume()
            throws InterruptedException {

        while (queue.isEmpty()) {
            wait();
        }

        int value = queue.remove();
        notifyAll();

        return value;
    }
}
```

Why use `while`, not `if`?

A thread can wake up without the desired condition being true, and another thread may change the state before the awakened thread reacquires the monitor. Therefore, it must check the condition again.

`wait()` releases the monitor while waiting and reacquires it before returning. `notifyAll()` wakes waiting threads, but they must compete to reacquire the monitor.

For real production code, prefer `BlockingQueue` unless you have a strong reason to implement the coordination yourself.

## 22.6 What if a consumer fails?

This is where the architect-level answer begins.

Ask these questions:

* Is the task lost if the consumer crashes?

* Is the queue in memory or durable storage?

* Can the same task be processed more than once?

* Is the operation idempotent?

* Are retries bounded?

* Is there a dead-letter mechanism?

* How is backlog measured?

* What happens when the downstream service is unavailable?

An in-memory `BlockingQueue` is not a durable message broker. If tasks must survive process restarts, consider a persistent messaging system such as Google Cloud Pub/Sub or Kafka, with suitable delivery, retry, and idempotency design.

## 22.7 How many consumers should you create?

Start from measurements, not guesswork.

If each notification takes 100 milliseconds, one consumer that processes one notification at a time has a theoretical service rate of about 10 notifications per second, before accounting for overhead.

Ten such consumers could theoretically process around 100 per second, if the downstream service can handle that concurrency.

Increasing consumers beyond downstream capacity can worsen failures instead of improving throughput.

# 23. `wait()`, `notify()`, `notifyAll()`, and `sleep()`

These are frequent interview questions.

| Method        | Belongs to | Releases monitor?         | Purpose                  |
| ------------- | ---------- | ------------------------- | ------------------------ |
| `wait()`      | `Object`   | Yes, the object's monitor | Wait for a condition     |
| `notify()`    | `Object`   | No, not immediately       | Wake one waiting thread  |
| `notifyAll()` | `Object`   | No, not immediately       | Wake all waiting threads |
| `sleep()`     | `Thread`   | No                        | Pause the current thread |

Important rules:

1. `wait()`, `notify()`, and `notifyAll()` must be called while owning the relevant object's monitor.

2. After notification, a thread must reacquire the monitor before it can continue.

3. Waking a thread does not guarantee that its condition is now true.

4. Always protect condition checks and updates consistently.

5. Avoid using sleep as a synchronization mechanism.

### Why is `notifyAll()` often safer than `notify()`?

When several kinds of threads share the same monitor, `notify()` might wake a thread that cannot make progress, leaving the appropriate thread asleep.

`notifyAll()` wakes all waiters, which then compete for the monitor and recheck their conditions. It can cost more, but it avoids some missed-progress problems associated with waking the wrong waiter.

# 24. ThreadLocal: data isolated per thread

`ThreadLocal` gives each thread its own independently stored value.

Think of each worker having a separate notebook instead of sharing one notebook.

Java

```
private static final ThreadLocal<String> USER_ID =
    new ThreadLocal<>();

public void processRequest(String userId) {
    USER_ID.set(userId);

    try {
        process();
    } finally {
        USER_ID.remove();
    }
}
```

### Common use cases

* Request-scoped context in legacy designs.

* Per-thread formatting state.

* Correlation IDs in carefully managed execution models.

### The production trap

Thread pools reuse threads. If you set a `ThreadLocal` and forget to remove it, a later task running on the same worker may see stale data.

This can cause request-context leakage or even expose information across requests.

Virtual threads also change the cost and lifecycle assumptions behind some `ThreadLocal` patterns, so use them deliberately.

For modern request tracing and distributed systems, consider explicit context propagation and framework-supported mechanisms rather than assuming thread-local state follows work across asynchronous boundaries.

# 25. Interruption and graceful shutdown

A senior engineer should know how to stop background work without leaving threads running indefinitely.

## 25.1 Interruption is a request, not forced termination

Java

```
Thread worker = new Thread(() -> {
    while (!Thread.currentThread().isInterrupted()) {
        doSmallUnitOfWork();
    }
});

worker.start();
worker.interrupt();
```

This pattern works if `doSmallUnitOfWork()` returns and the loop checks the interrupt status.

If the worker is blocked in an interruptible operation, the operation may throw `InterruptedException`. Catching that exception clears the interrupt status, so code must either propagate it or restore the status when appropriate.

Avoid swallowing interruption:

Java

```
catch (InterruptedException e) {
    Thread.currentThread().interrupt();
    return;
}
```

That is often preferable to ignoring it, though the correct response depends on the method's contract.

## 25.2 `shutdown()` vs. `shutdownNow()`

| Method          | Behavior                                                                                                      |
| --------------- | ------------------------------------------------------------------------------------------------------------- |
| `shutdown()`    | Stops accepting new tasks and allows previously submitted tasks to finish                                     |
| `shutdownNow()` | Attempts to stop active tasks, usually through interruption, and returns tasks that never commenced execution |

Neither method guarantees that arbitrary tasks will terminate. Tasks must cooperate with interruption, and blocked I/O may need appropriate timeouts or cancellation support.

## 25.3 Graceful executor shutdown

Java

```
executor.shutdown();

try {
    if (!executor.awaitTermination(30, TimeUnit.SECONDS)) {
        executor.shutdownNow();

        if (!executor.awaitTermination(10, TimeUnit.SECONDS)) {
            System.err.println("Executor did not terminate");
        }
    }
} catch (InterruptedException e) {
    executor.shutdownNow();
    Thread.currentThread().interrupt();
}
```

In a Spring Boot application, executor lifecycle should generally be managed by the application container so shutdown happens in the right order.

Architect insight: graceful shutdown is part of correctness. It is not enough to create a thread pool; you must know how it drains, how tasks are cancelled, and what happens to unfinished business operations.

# 26. Advanced concurrency concepts worth knowing

These topics often distinguish a developer who can use concurrency utilities from someone who can reason about the Java Memory Model.

## 26.1 Happens-before

The happens-before relationship defines ordering and visibility guarantees between actions.

Examples include:

* Unlocking a monitor happens-before a subsequent lock of that same monitor.

* A volatile write happens-before a subsequent read of that field that observes it.

* Actions before `Thread.start()` happen-before actions in the started thread.

* Actions in a thread happen-before another thread successfully returns from `join()` on it.

* Actions in an asynchronous computation happen-before a corresponding successful result retrieval through specified concurrency APIs, according to their contracts.

Why does this matter?

Because code that looks sequential in one method might be executed concurrently across threads, and the Java Memory Model defines which values and orderings are guaranteed.

## 26.2 Immutable objects

Immutable objects are often the easiest way to share data safely.

For example, an immutable configuration object can be constructed once and safely shared after proper publication, without needing a lock for each read.

Instead of allowing multiple threads to modify a shared object, sometimes the best design is to create a new immutable object and replace the reference.

## 26.3 Safe publication

An object should be made visible to other threads using a mechanism with appropriate memory guarantees.

Common approaches include:

* Storing it in a properly synchronized field.

* Publishing through a volatile reference.

* Publishing through a thread-safe collection.

* Constructing it before starting a thread that will use it.

* Using established framework lifecycle and dependency-injection mechanisms correctly.

Unsafe publication can expose partially initialized state, particularly when an object escapes during construction.

## 26.4 Context switching

When the CPU switches execution between threads, there can be overhead from saving and restoring execution state, scheduling, and cache effects.

Too many runnable threads can reduce throughput rather than improve it.

However, the cost is workload- and platform-dependent; do not make architectural decisions from a fixed universal estimate.

## 26.5 False sharing

Two threads can update different variables that happen to occupy the same CPU cache line. Cache-coherence traffic can cause performance degradation even though the variables are logically independent.

This is called false sharing.

It is mainly a performance issue in highly optimized, high-throughput code. Diagnose it with profiling and measurement before attempting padding or specialized data layouts.

## 26.6 Lock-free vs. wait-free

* Lock-based: threads coordinate using locks.

* Lock-free: the system as a whole makes progress despite some threads being delayed.

* Wait-free: every participating operation is guaranteed to finish in a bounded number of its own steps under the algorithm's assumptions.

These are formal progress guarantees. An algorithm using CAS is not automatically wait-free, and lock-free does not mean contention-free or necessarily faster for every workload.

# 27. Architect-level production scenarios

These scenarios are particularly valuable for Senior Java Backend Engineer, Lead Engineer, and Technical Architect interviews.

## Scenario 1: Your Spring Boot API receives 1,000 requests per second

Each request calls an external REST API that takes 500 ms.

Question: Would you create a new thread for every request?

Strong answer:

1. First determine whether the operation is CPU-bound or I/O-bound.

2. For blocking I/O, consider Java 21+ virtual threads or a suitably bounded executor.

3. Configure connection pools, request timeouts, and concurrency limits for the downstream service.

4. Apply backpressure when the system reaches capacity.

5. Monitor request latency, rejection rate, queue depth, and downstream errors.

6. Load-test the design before choosing concurrency limits.

What not to say: “I will simply increase the thread pool to 1,000.”

That could overload the database, external API, or application itself.

## Scenario 2: Two threads update the same inventory

Question: Two customers purchase the last available product simultaneously. How do you prevent overselling?

Strong answer:

* For in-memory shared state, protect the entire check-and-decrement operation with a lock or another correct atomic design.

* For a database-backed inventory, prefer an appropriate transactional operation, such as a conditional update:

SQL

```
UPDATE inventory
SET stock = stock - 1
WHERE product_id = ?
  AND stock > 0;
```

Check the number of affected rows. If it is zero, the inventory was unavailable or the row did not match.

Use the appropriate transaction and isolation strategy for the wider business operation.

Architect insight: a Java lock protects only threads that coordinate through that lock instance. It does not automatically coordinate separate JVMs or application replicas.

## Scenario 3: A thread pool's queue keeps growing

Question: The API is becoming slower even though CPU utilization is low. What do you investigate?

Strong answer:

* Inspect active threads and queue depth.

* Measure how long tasks wait before execution.

* Check whether threads are blocked on database calls or remote services.

* Review downstream timeouts and connection pool exhaustion.

* Verify whether the queue is bounded.

* Apply admission control or reject work when necessary.

* Scale only after identifying the actual bottleneck.

Low CPU utilization does not mean the application has spare capacity. Threads can be stuck waiting for resources.

## Scenario 4: Your background job processes a million records

Question: How would you process the records concurrently?

Strong answer:

1. Divide work into independent, bounded chunks.

2. Choose an executor appropriate for the workload.

3. Limit concurrent database operations according to connection pool capacity.

4. Avoid sharing mutable state between workers where possible.

5. Make retries bounded and operations idempotent.

6. Track progress and failures.

7. Define cancellation and shutdown behavior.

8. Benchmark throughput and database load.

For CPU-heavy transformations, a bounded CPU-oriented pool may be suitable. For blocking I/O, virtual threads can simplify concurrency, but database and downstream limits still apply.

## Scenario 5: One notification is sent twice

Question: Is thread safety enough to prevent duplicate notifications?

No. Thread safety protects concurrent access to program state, but it does not guarantee exactly-once business processing.

A request may be retried after a timeout even though the original operation succeeded.

Use appropriate idempotency keys, persistent processing state, retry policies, and delivery guarantees. In distributed messaging, consumers should be designed for the actual delivery semantics of the broker.

## Scenario 6: Your application hangs occasionally

Question: How do you diagnose it?

Strong answer:

1. Capture multiple thread dumps during the incident.

2. Identify blocked and waiting threads.

3. Look for deadlocks and circular lock dependencies.

4. Check thread pool saturation and queue growth.

5. Inspect slow database calls and connection acquisition.

6. Correlate thread states with CPU, heap, garbage collection, and request latency.

7. Use Java Flight Recorder or a profiler for deeper analysis.

8. Reproduce the issue under load and verify the fix.

Do not assume every hang is a deadlock. A service may simply have every worker blocked on a slow dependency.

## Scenario 7: Should you use `ConcurrentHashMap` or `AtomicInteger`?

Question: You need a per-customer request counter.

A `ConcurrentHashMap<String, AtomicInteger>` may be suitable when counters are updated independently by customer. But consider lifecycle management, memory growth, counter cleanup, and whether the result is only a metric or a business-critical limit.

For metrics, `LongAdder` can offer better performance under heavy contention. For enforcing a strict account quota across multiple servers, local atomic counters alone are insufficient.

## Scenario 8: You need to wait for five services to initialize

Use `CountDownLatch` if several tasks must signal completion before startup proceeds. Ensure failures are recorded and use a timeout.

If the tasks need to synchronize repeatedly at the end of each processing phase, consider `CyclicBarrier` or another suitable coordination design.

# 28. Interview questions and concise answers

Use this section for rapid revision. Try answering each question aloud before reading the answer.

## Thread fundamentals

1. What is the difference between a process and a thread?

A process is a running program with its own process-level resources. Threads are execution paths within a process and generally share its heap.

2. What happens when you call `start()`?

It starts a thread of execution, which subsequently invokes `run()`. Calling `run()` directly does not start a separate thread.

3. Can we start the same thread twice?

No. A second call to `start()` throws `IllegalThreadStateException`.

4. Why prefer Runnable over extending Thread?

It separates the task from the execution mechanism and avoids consuming Java's single class inheritance slot.

5. What are the six Java thread states?

`NEW`, `RUNNABLE`, `BLOCKED`, `WAITING`, `TIMED_WAITING`, and `TERMINATED`.

6. Does `sleep()` release a lock?

No. It does not release intrinsic monitors held by the sleeping thread.

7. Does `wait()` release a lock?

It releases the monitor of the object on which it is called, and reacquires that monitor before returning.

## Synchronization and locks

8. What is a race condition?

The result depends on how concurrent operations interleave.

9. What does synchronized guarantee?

Mutual exclusion for the same monitor and memory visibility through monitor synchronization.

10. Is a volatile counter thread-safe?

No. `volatile` does not make compound operations such as `count++` atomic.

11. What makes ReentrantLock different from synchronized?

Explicit lock management, timed acquisition, interruptible acquisition, optional fairness, and multiple `Condition` objects.

12. What is reentrancy?

A thread can acquire a lock it already owns again without blocking itself.

13. When should you use ReadWriteLock?

When reads are frequent, writes are less frequent, and concurrent reads can provide a measurable benefit.

14. Can synchronized methods on different objects run concurrently?

Yes, because each instance has its own monitor. Static synchronized methods use the class monitor.

## Executor framework

15. What is the difference between execute() and submit()?

`execute()` accepts a `Runnable` and returns no result. `submit()` accepts a `Runnable` or `Callable` and returns a `Future`.

16. Why use ThreadPoolExecutor?

It provides explicit control over worker counts, queue capacity, thread creation, and task rejection.

17. Why can an unbounded queue be dangerous?

It can accumulate excessive tasks, increase latency, and consume memory until the application becomes unstable.

18. When does ThreadPoolExecutor grow beyond its core size?

Under the normal execution policy, it first creates core workers, then tries to enqueue tasks. If the queue cannot accept a task, it may create additional workers up to the maximum.

19. What is backpressure?

A mechanism that prevents incoming work from overwhelming processing capacity, for example by slowing producers, limiting concurrency, or rejecting excess requests.

20. What is the difference between platform and virtual threads?

Platform threads are typically backed by operating-system threads. Virtual threads are lightweight JVM-managed threads intended to support high concurrency, particularly for blocking tasks.

21. Are virtual threads always faster?

No. They help reduce the cost of managing many concurrent tasks but do not automatically accelerate CPU-bound work or increase downstream capacity.

22. When should you use ForkJoinPool?

For suitable divide-and-conquer tasks that can be split into smaller subtasks and combined, especially CPU-oriented work.

## Futures and coordination

23. What is the difference between Runnable and Callable?

`Runnable` returns no result and cannot declare checked exceptions in `run()`. `Callable` returns a result and can throw checked exceptions.

24. Does Future.get() block?

It can block until completion. The timed overload limits how long the caller waits.

25. CountDownLatch vs. CyclicBarrier?

A latch waits for a count to reach zero and cannot be reset. A barrier coordinates parties meeting at a common point and can be reused.

26. When would you use Semaphore?

To limit concurrent access to a resource or operation.

27. What is Exchanger?

A utility for two threads to exchange values.

28. Why use BlockingQueue?

It safely coordinates producers and consumers and can block producers when full or consumers when empty.

## Thread safety and failures

29. Is ConcurrentHashMap completely atomic?

Its individual operations and documented compound methods provide concurrency guarantees, but arbitrary sequences of calls are not automatically atomic.

30. AtomicInteger vs. LongAdder?

Both support concurrent updates. `LongAdder` often scales better for high-contention statistical counting; `AtomicInteger` is appropriate for atomic operations on one integer.

31. What is a deadlock?

Threads wait on resources held by one another in a cycle, preventing progress.

32. How do you prevent deadlocks?

Use consistent lock ordering, reduce nested locking, avoid long-held locks, and consider timed or interruptible acquisition where appropriate.

33. What is starvation?

A thread is repeatedly denied the resources or scheduling opportunities it needs to progress.

34. What is livelock?

Threads continue reacting to each other but fail to make meaningful progress.

35. Why is `ThreadLocal` risky with thread pools?

Worker threads are reused, so values can leak between tasks if they are not removed or otherwise correctly scoped.

36. Why is `notifyAll()` often safer than `notify()`?

It wakes all waiters, allowing each to recheck its condition. It may cause extra contention but avoids some wrong-waiter problems.

37. Does thread safety guarantee exactly-once processing?

No. Exactly-once business outcomes require additional design, such as idempotency and durable state.

38. How do you gracefully stop a thread pool?

Stop accepting new tasks, allow a bounded drain period, request cancellation if necessary, and ensure tasks cooperate with interruption.

# 29. Common mistakes that interviewers look for

Creating a new thread for every task

Better: select an execution model and define resource limits based on the workload.

Using volatile to protect a counter

Better: use an atomic class or lock for compound updates.

Forgetting to unlock

Better: release explicit locks in a finally block.

Holding a lock during a slow REST call

Better: minimize lock duration and separate external I/O from shared-state updates when correctness permits.

Using unbounded queues everywhere

Better: define capacity, overload behavior, and task deadlines.

Ignoring InterruptedException

Better: propagate interruption or restore the interrupt status when appropriate.

Assuming a thread-safe collection makes the whole workflow safe

Better: identify the complete business invariant and make the required compound operation atomic.

# 30. How to answer a concurrency design question as a Technical Architect

When an interviewer gives you a scenario, use this six-step framework.

1. Clarify the workload. Is it CPU-bound or I/O-bound? How many tasks arrive? What is the expected latency?

2. Identify shared state. Which data is mutable, who owns it, and which operations must be atomic?

3. Choose the execution model. Consider a bounded executor, virtual threads, ForkJoinPool, or a messaging system.

4. Define resource limits. Include thread counts where appropriate, queue capacity, connection pools, timeouts, and concurrency limits.

5. Design failure behavior. Explain rejection, retries, interruption, idempotency, cancellation, and graceful shutdown.

6. Explain observability. Name the metrics, thread dumps, profiling tools, and load tests you would use to validate the design.

For example, if asked to design a high-throughput notification service, do not immediately start writing `synchronized` methods. Discuss the request flow, durable messaging if needed, bounded concurrency, downstream rate limits, duplicate delivery, retry handling, and operational monitoring.

That is the difference between simply knowing Java concurrency APIs and designing a production system.

# 31. Final revision cheat sheet

| Requirement                                          | First API or concept to consider                        |
| ---------------------------------------------------- | ------------------------------------------------------- |
| Execute a simple task on another thread              | `Runnable`                                              |
| Return a task result                                 | `Callable` + `Future`                                   |
| Manage a bounded worker pool                         | `ThreadPoolExecutor`                                    |
| Handle many blocking I/O tasks                       | Virtual threads + resource limits                       |
| Protect a small critical section                     | `synchronized`                                          |
| Need timed or interruptible locking                  | `ReentrantLock`                                         |
| Many reads and occasional writes                     | `ReadWriteLock`                                         |
| Publish a simple state flag                          | `volatile`                                              |
| Atomic integer update                                | `AtomicInteger`                                         |
| High-contention statistics                           | `LongAdder`                                             |
| Concurrent key-value access                          | `ConcurrentHashMap`                                     |
| Read-heavy list                                      | `CopyOnWriteArrayList`                                  |
| Limit concurrent calls                               | `Semaphore`                                             |
| Wait for a set of tasks once                         | `CountDownLatch`                                        |
| Synchronize multiple processing rounds               | `CyclicBarrier`                                         |
| Exchange data between two workers                    | `Exchanger`                                             |
| Safely transfer work between producers and consumers | `BlockingQueue`                                         |
| Split CPU-heavy recursive work                       | `ForkJoinPool`                                          |
| Prevent deadlock                                     | Consistent lock ordering and careful resource ownership |
| Diagnose concurrency incidents                       | Thread dumps, JFR, profiling, metrics                   |

## The 10 principles to remember

1. A thread is an execution path; a task is the work to execute.

2. Prefer executors over manually creating threads for every task.

3. Concurrency and parallelism are different.

4. Thread safety requires reasoning about atomicity, visibility, and ordering.

5. `volatile` gives visibility guarantees, not compound atomicity.

6. A thread-safe collection does not make a multi-step business operation atomic.

7. Bounded queues and explicit overload behavior are essential for predictable systems.

8. Virtual threads simplify high-concurrency blocking I/O but do not remove resource bottlenecks.

9. Every background task needs a clear policy for errors, interruption, cancellation, and shutdown.

10. In production, measure before tuning. More threads do not necessarily mean more throughput.

My recommendation for your interview preparation: master `synchronized`, `volatile`, `ThreadPoolExecutor`, `Callable`/`Future`, atomic classes, and `BlockingQueue` first. Then practice explaining deadlocks, backpressure, virtual threads, and production failure scenarios aloud. Those concepts connect most of the topics in this guide and are especially valuable in senior-level design interviews.
