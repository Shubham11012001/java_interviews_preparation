# Non-Thread-Safe Collections in Java: A Complete Guide for Senior Engineers and Technical Architects

Java • Collections Framework • Multithreading • Concurrency • System Design • Interview Preparation

Imagine you have a bank account with ₹10,000. Two people try to withdraw ₹7,000 at exactly the same time. Both read the balance as ₹10,000, both decide the withdrawal is allowed, and both proceed.

Your account could end up in an invalid state.

This is the kind of problem that can happen when multiple threads access a non-thread-safe collection without proper synchronization.

As a Technical Architect, you should understand not only which collections are thread-safe, but also why they are unsafe, how failures occur in production, and how to choose the right solution for a particular workload.

Let's build that understanding step by step.

# 1. What is a non-thread-safe collection?

A non-thread-safe collection is a collection that does not guarantee correct behavior when multiple threads access or modify it concurrently without appropriate coordination.

Examples include:

* `ArrayList`

* `LinkedList`

* `HashMap`

* `HashSet`

* `TreeMap`

* `TreeSet`

* `PriorityQueue`

* `ArrayDeque`

These collections are excellent tools when used in the right conditions. The problem is not that they are inherently bad; it's that they don't automatically coordinate concurrent access.

## A simple analogy

Imagine a classroom with a shared whiteboard.

* Thread A writes a student's name.

* Thread B erases part of the board.

* Thread C reads the board.

If everyone writes and erases without any rules, the information may become inconsistent.

A non-thread-safe collection is like that shared whiteboard: it needs coordination when multiple people modify it simultaneously.

## What does thread-safe mean?

A thread-safe collection provides guarantees that concurrent operations behave according to its documented concurrency rules.

Depending on the collection, these guarantees might include:

* Preventing internal data corruption.

* Making updates visible to other threads.

* Providing atomic operations.

* Allowing safe concurrent reads and writes.

* Offering a consistent view of data during iteration.

Important: Thread safety does not automatically mean that every operation is atomic, that iteration sees a snapshot, or that business logic involving multiple operations is safe.

Those distinctions are crucial in senior-level interviews.

# 2. Why are Java collections often non-thread-safe?

The Java Collections Framework provides general-purpose data structures. Many of them prioritize efficient single-threaded operations without the overhead of automatic synchronization.

Consider this code:

Java

```
List<Integer> numbers = new ArrayList<>();

numbers.add(10);
numbers.add(20);
numbers.add(30);
```

In a single-threaded application, this is straightforward.

Now imagine two threads execute:

Java

```
numbers.add(100);
numbers.add(200);
```

at the same time.

An `ArrayList` internally maintains information such as:

* An array containing the elements.

* A `size` field indicating the number of elements.

* Capacity information indicating how many elements the backing array can hold.

A simplified implementation of adding an element resembles:

Java

```
elementData[size] = element;
size++;
```

This is a conceptual illustration, not the complete implementation.

The assignment and increment are not one atomic operation. Two threads may observe the same size and interfere with each other.

Depending on the timing and implementation details, this can cause lost updates or inconsistent observations.

When the backing array needs to grow, concurrent modifications can introduce additional risks.

## Why doesn't Java make every collection thread-safe by default?

Because synchronization has trade-offs.

| Consideration               | Non-thread-safe collection                      | Thread-safe collection                     |
| --------------------------- | ----------------------------------------------- | ------------------------------------------ |
| Single-threaded performance | Often excellent                                 | May have additional overhead               |
| Concurrent access           | Requires external coordination when necessary   | Provides documented concurrency guarantees |
| Memory overhead             | Often lower                                     | May be higher                              |
| Implementation complexity   | Usually simpler                                 | Depends on concurrency strategy            |
| Best use case               | Thread-confined or externally synchronized data | Data shared across threads                 |

A thread-safe collection is not necessarily slower in every workload. For example, `ConcurrentHashMap` can outperform a globally synchronized `HashMap` under concurrent workloads because it allows substantial concurrency.

The correct choice depends on how the collection is used.

# 3. Which Java collections are non-thread-safe?

Let's categorize the most important ones.

## 3.1 List implementations

| Collection             | Why it needs attention                                                           | Typical use                |
| ---------------------- | -------------------------------------------------------------------------------- | -------------------------- |
| `ArrayList`            | Concurrent structural modifications are not coordinated                          | General-purpose lists      |
| `LinkedList`           | Concurrent modifications can interfere with internal links and size              | Lists and deque operations |
| `Vector`               | Individual methods are synchronized, but compound operations may still be unsafe | Legacy synchronized list   |
| `CopyOnWriteArrayList` | Thread-safe; copies its backing array for mutations                              | Read-heavy shared lists    |

Notice that `Vector` is included for comparison, not as a non-thread-safe collection. Its individual methods are synchronized, but that alone does not make arbitrary multi-operation workflows atomic.

## 3.2 Map implementations

| Collection                      | Thread-safety behavior                            |
| ------------------------------- | ------------------------------------------------- |
| `HashMap`                       | Not thread-safe                                   |
| `LinkedHashMap`                 | Not thread-safe                                   |
| `TreeMap`                       | Not thread-safe                                   |
| `Hashtable`                     | Legacy class with synchronized individual methods |
| `Collections.synchronizedMap()` | Synchronizes map method calls                     |
| `ConcurrentHashMap`             | Designed for concurrent access                    |

`HashMap` is particularly important in interviews because it is used frequently in application code and can accidentally become shared across threads.

## 3.3 Set implementations

| Collection            | Thread-safety behavior                                 |
| --------------------- | ------------------------------------------------------ |
| `HashSet`             | Not thread-safe                                        |
| `LinkedHashSet`       | Not thread-safe                                        |
| `TreeSet`             | Not thread-safe                                        |
| `CopyOnWriteArraySet` | Thread-safe; suited to read-heavy workloads            |
| Concurrent key sets   | Thread-safe; useful for concurrent membership tracking |

## 3.4 Queue and deque implementations

| Collection                   | Thread-safety behavior                          |
| ---------------------------- | ----------------------------------------------- |
| `PriorityQueue`              | Not thread-safe                                 |
| `ArrayDeque`                 | Not thread-safe                                 |
| `LinkedList` used as a queue | Not thread-safe                                 |
| `ConcurrentLinkedQueue`      | Thread-safe, non-blocking queue                 |
| `ArrayBlockingQueue`         | Thread-safe, bounded blocking queue             |
| `LinkedBlockingQueue`        | Thread-safe, optionally bounded blocking queue  |
| `PriorityBlockingQueue`      | Thread-safe priority queue, logically unbounded |

A blocking queue can be useful when one thread produces work and other threads consume it.

Architect's takeaway: Don't memorize only collection names. Understand whether the collection is shared, which operations happen concurrently, and which consistency guarantees the application requires.

# 4. The four major concurrency problems

When a non-thread-safe collection is shared between threads, four concepts are especially important.

## 4.1 Race condition

A race condition occurs when the correctness of a result depends on the timing or interleaving of concurrent operations.

Example:

Java

```
Map<String, Integer> stock = new HashMap<>();

// Thread A
stock.put("LAPTOP", 10);

// Thread B
stock.put("LAPTOP", 20);
```

Both operations update the same key.

Even if the map remains structurally valid, the final value depends on which write takes effect last.

That may be perfectly acceptable if the requirement is simply "last writer wins." It is incorrect if the business requirement says stock updates must be accumulated or coordinated.

Important distinction: A race condition can exist even when the collection's internal structure is not corrupted.

## 4.2 Lost update

Consider a shared counter stored in a map:

Java

```
Map<String, Integer> counters = new HashMap<>();
counters.put("requests", 0);
```

Two threads perform:

Java

```
int current = counters.get("requests");
counters.put("requests", current + 1);
```

Suppose both threads read `0`.

* Thread A calculates `1`.

* Thread B calculates `1`.

* Thread A writes `1`.

* Thread B writes `1`.

Expected result: `2`.

Actual result: `1`.

The increment was a read-modify-write operation, not one atomic operation.

The solution is not merely to replace `HashMap` with `ConcurrentHashMap`. A concurrent map protects individual map operations, but the two calls together are still not atomic.

One solution is:

Java

```
ConcurrentHashMap<String, Integer> counters =
        new ConcurrentHashMap<>();

counters.put("requests", 0);

counters.compute("requests",
        (key, value) -> value + 1);
```

For high-frequency counters, an atomic counter is often a better fit:

Java

```
ConcurrentHashMap<String, LongAdder> counters =
        new ConcurrentHashMap<>();

counters.computeIfAbsent("requests",
        key -> new LongAdder()).increment();
```

`LongAdder` is useful for highly contended statistical counters. Its `sum()` method is not an atomic snapshot relative to ongoing updates, so it is not suitable when an exact transactional value is required at every instant.

## 4.3 Visibility problem

A thread may update shared state, but another thread is not automatically guaranteed to observe that update merely because the data structure is accessible to both threads.

Java's memory model defines the visibility guarantees that arise from synchronization, volatile access, thread lifecycle operations, and other happens-before relationships.

Thread-safe collection implementations establish appropriate guarantees for their documented operations. For example, concurrent collections provide memory-consistency guarantees for actions performed before placing an object into the collection and subsequent access or removal of that object by another thread.

## 4.4 Structural corruption or inconsistent state

Concurrent structural modifications can interfere with the collection's internal data structures.

For example, an `ArrayList` may need to resize its backing array while another thread modifies it.

A `HashMap` may also be modified while another thread performs operations that depend on its internal state.

Do not overstate this point in interviews: the exact failure depends on the implementation, Java version, and operation interleaving. Modern `HashMap` implementations differ from older versions in some internal details.

The reliable conclusion is that unsynchronized concurrent modification is unsupported and may lead to lost updates, inconsistent results, or other incorrect behavior.

# 5. Deep dive: Why HashMap is not thread-safe

This is one of the most common Java concurrency interview topics.

A `HashMap` stores key-value pairs using a hash-based data structure. In simplified terms, it uses an array of buckets, with entries stored in those buckets.

### Simplified HashMap structure

Bucket array

Bucket 0

A → 10

Bucket 1

B → 20

C → 30

Bucket 2

D → 40

Illustrative only. Buckets may contain a linked list or a tree, depending on the implementation and conditions.

## What can go wrong?

### Scenario A: Two threads update the map

Two threads add entries concurrently. The map does not provide the synchronization required to coordinate these operations.

Potential outcomes include lost updates or inconsistent observations.

### Scenario B: Concurrent read and write

One thread reads the map while another updates it.

Without appropriate coordination, there is no general thread-safety guarantee for that combination of operations.

### Scenario C: Concurrent resize

The map grows beyond its capacity threshold and needs to resize its internal table. Another thread modifies it at the same time.

Concurrent resizing is one reason unsynchronized concurrent use of `HashMap` is unsafe. Historical Java 7 implementations could exhibit particularly problematic bucket-chain behavior, including infinite loops in some resize races. Do not claim that exact historical failure is the normal behavior of modern Java.

## Is HashMap safe for concurrent reads?

The interview-safe answer is:

A `HashMap` is not a concurrent collection. If it is safely published, never modified after publication, and its contents are otherwise immutable, concurrent reads are generally safe. Do not rely on that to justify unsynchronized concurrent mutation.

For shared mutable maps, use a suitable concurrency mechanism.

# 6. Best ways to make collections thread-safe

There is no single best solution. The right choice depends on whether you need concurrent writes, frequent reads, atomic compound operations, or strict consistency.

## Approach 1: Use `Collections.synchronizedList()`

Java provides synchronized wrappers around many collection types.

Java

```
List<String> names =
        Collections.synchronizedList(new ArrayList<>());

names.add("Alice");
names.add("Bob");
```

The wrapper synchronizes individual method calls on a common lock.

### What does this protect?

Calls such as `add()`, `get()`, and `remove()` through the wrapper are synchronized.

However, a sequence of operations is not automatically one atomic transaction.

For example:

Java

```
if (!names.contains("Alice")) {
    names.add("Alice");
}
```

Another thread may insert `"Alice"` after the check but before the add.

If the entire sequence must be atomic, synchronize around it using the wrapper as the lock:

Java

```
synchronized (names) {
    if (!names.contains("Alice")) {
        names.add("Alice");
    }
}
```

### Important: iteration

Iteration over synchronized wrappers requires external synchronization for the entire iteration.

Java

```
synchronized (names) {
    for (String name : names) {
        System.out.println(name);
    }
}
```

Without this, concurrent modification may cause inconsistent traversal or a `ConcurrentModificationException`.

### When should you use it?

* You have a relatively simple shared collection.

* You want to preserve an existing collection implementation.

* You want synchronized method access.

* Contention is not a major concern.

### When should you avoid it?

* Many threads frequently compete for the same collection lock.

* You need highly concurrent reads and writes.

* You need snapshot iteration without holding a global lock for the entire traversal.

## Approach 2: Use `ConcurrentHashMap`

Java

```
ConcurrentHashMap<String, Integer> cache =
        new ConcurrentHashMap<>();

cache.put("USER_101", 100);
cache.put("USER_102", 200);
```

`ConcurrentHashMap` is designed for concurrent access without synchronizing every operation on one global map lock.

It supports atomic operations such as:

Java

```
cache.putIfAbsent("USER_101", 300);
```

and:

Java

```
cache.compute("USER_101",
        (key, value) -> value == null ? 1 : value + 1);
```

The second example atomically computes and updates the mapping for that key, subject to the method's documented behavior.

### Why is it useful?

Different threads can often operate on different keys concurrently. Its implementation uses concurrency controls designed for scalable access.

The internal implementation has evolved across Java versions, so avoid claiming that modern `ConcurrentHashMap` always uses a particular fixed number of locks or a particular segmentation architecture.

### Limitations

* `null` keys and values are not allowed.

* Iterators are weakly consistent rather than fail-fast or snapshot-based.

* Compound business operations involving multiple keys may require additional coordination.

* It does not make mutable objects stored as values thread-safe.

### Best use cases

* Shared caches.

* User/session metadata.

* Concurrent counters and frequency maps.

* Application-wide lookup tables that change during runtime.

## Approach 3: Use `CopyOnWriteArrayList`

Java

```
List<String> listeners =
        new CopyOnWriteArrayList<>();
```

When a thread modifies the list, the implementation creates a new backing array containing the updated elements.

Readers can continue traversing the array snapshot associated with their iterator.

### Why does this work well for some applications?

Imagine a notification service with 100 registered listeners. The system checks the listener list thousands of times but changes the list only occasionally.

A copy-on-write strategy can be effective because reads and iteration are frequent, while writes are rare.

### Downsides

* Every successful mutation generally requires copying the backing array.

* Large lists can make frequent writes expensive.

* Iterators see a snapshot and do not reflect later changes.

* It is unsuitable for large, frequently modified lists.

Remember: Copy-on-write is a trade-off: expensive writes in exchange for efficient, predictable traversal.

## Approach 4: Use concurrent queues

If your problem is moving work between threads, a concurrent queue or blocking queue may be more appropriate than a shared list.

Java

```
BlockingQueue<String> queue =
        new ArrayBlockingQueue<>(1_000);
```

A producer can submit work:

Java

```
queue.put("TASK_101");
```

A consumer can retrieve it:

Java

```
String task = queue.take();
```

The blocking behavior provides natural backpressure at the queue boundary: producers can wait when a bounded queue is full, and consumers can wait when it is empty.

For production services, bounded queues are often preferable when you need to limit memory usage and prevent unbounded task accumulation.

# 7. Comparison: Which collection should a Technical Architect choose?

| Requirement                                 | Recommended starting point            | Why                                           |
| ------------------------------------------- | ------------------------------------- | --------------------------------------------- |
| Local temporary list inside one method      | `ArrayList`                           | No shared concurrent access                   |
| Shared map with concurrent updates          | `ConcurrentHashMap`                   | Designed for concurrent access                |
| Simple shared list with moderate contention | `Collections.synchronizedList()`      | Straightforward synchronized wrapper          |
| Many reads, rare list modifications         | `CopyOnWriteArrayList`                | Snapshot iteration and efficient reads        |
| Producer-consumer task processing           | `BlockingQueue` implementation        | Coordination and backpressure                 |
| Concurrent queue without blocking           | `ConcurrentLinkedQueue`               | Non-blocking queue operations                 |
| Sorted shared map                           | `ConcurrentSkipListMap`               | Concurrent sorted-map operations              |
| Concurrent set membership                   | `ConcurrentHashMap.newKeySet()`       | Concurrent add, remove, and membership checks |
| Data never changes after initialization     | Safely published immutable collection | Readers need no mutation coordination         |

These are starting points, not universal rules. For example, a `ConcurrentHashMap` may be inappropriate if your business operation requires an atomic update across several different keys.

# 8. Production scenarios every senior engineer should know

Let's connect the theory to real applications.

## Scenario 1: Spring Boot service handling concurrent requests

Suppose your service handles 1,000 requests per second and stores request statistics in a shared map.

Java

```
Map<String, Integer> counts = new HashMap<>();
```

Each request increments a counter.

### What is wrong?

Multiple request-handling threads access the same mutable map. The read-modify-write operation can lose increments, and the map itself is not designed for concurrent mutation.

### Better design

Java

```
ConcurrentHashMap<String, LongAdder> counts =
        new ConcurrentHashMap<>();

void recordRequest(String endpoint) {
    counts.computeIfAbsent(
            endpoint,
            key -> new LongAdder()
    ).increment();
}
```

For a production monitoring system, also consider metric libraries that already provide concurrency-safe counters, lifecycle management, and integration with your monitoring platform.

Architect's decision: Choose a concurrent counter strategy based on contention, cardinality, and whether exact instantaneous totals are required.

## Scenario 2: Duplicate order processing

Imagine two threads process the same order.

Java

```
if (!processedOrders.contains(orderId)) {
    processedOrders.add(orderId);
    processOrder(orderId);
}
```

Even if `processedOrders` is replaced with a concurrent set, the overall check-then-act sequence is not atomic.

Two threads could both observe that the order is absent and both start processing it.

### Better in-memory approach

Use an atomic set operation:

Java

```
Set<String> processedOrders =
        ConcurrentHashMap.newKeySet();

if (processedOrders.add(orderId)) {
    processOrder(orderId);
}
```

Only the thread whose `add()` actually inserts the key proceeds.

### Is this enough for a financial system?

Not necessarily.

This protects only the in-memory set and only within the relevant shared instance. It does not protect against:

* Multiple application instances.

* Application restarts.

* Duplicate messages after a retry.

* Failures between marking the order and completing the transaction.

For critical order processing, consider a durable idempotency key enforced by a database unique constraint or an equivalent distributed coordination mechanism, together with appropriate transaction and retry design.

Architect's lesson: Thread safety is not the same as distributed-system correctness.

## Scenario 3: Notification service with multiple workers

Suppose a notification service receives events and has several worker threads.

A common mistake is to store pending notifications in a shared `ArrayList` and let every worker remove items from it.

### Problems

* Concurrent modifications are not coordinated.

* Multiple workers may interfere with shared state.

* There is no built-in producer-consumer blocking behavior.

* Backpressure and capacity limits are not naturally enforced.

### Better approach

Java

```
BlockingQueue<Notification> queue =
        new ArrayBlockingQueue<>(10_000);
```

Workers can take notifications from the queue.

For larger systems, a durable broker such as Google Cloud Pub/Sub or Kafka may be the right architecture, depending on delivery, replay, throughput, and operational requirements.

An in-memory blocking queue does not replace durable messaging when messages must survive process failure.

## Scenario 4: Shared application cache

Suppose a Spring service maintains a map of user profiles.

Java

```
Map<String, UserProfile> profiles = new HashMap<>();
```

Many request threads read and update the cache.

### Better approach

Use `ConcurrentHashMap` if its per-entry concurrency model fits your needs.

But remember that this does not make `UserProfile` itself thread-safe.

For example:

Java

```
UserProfile profile = profiles.get(userId);
profile.setStatus("ACTIVE");
```

The map safely retrieving a reference does not automatically coordinate concurrent mutation of the referenced object.

Possible solutions include:

* Immutable profile objects and replacement-based updates.

* Synchronization around the mutable object's state.

* Atomic update functions that replace the complete value.

* A cache implementation with appropriate loading and eviction semantics.

## Scenario 5: Configuration and feature flags

Suppose an application has configuration that is read by thousands of threads but changes only during occasional refreshes.

A concurrent map might be sufficient, but if readers need a consistent view of the entire configuration, updating individual keys one by one can expose a partially refreshed state.

A better design can be to build an immutable configuration snapshot and atomically publish a reference to it.

This provides a useful architectural pattern:

Build a complete new state privately, then publish it atomically.

Thread-safe individual entries do not automatically provide atomic consistency across a whole collection.

# 9. The subtle interview trap: Thread-safe collection vs thread-safe operation

This is a distinction that separates basic answers from strong senior-level answers.

Consider:

Java

```
Map<String, Integer> map = new ConcurrentHashMap<>();

map.put("balance", 100);
```

The individual `put()` is a thread-safe map operation.

Now consider:

Java

```
if (map.get("balance") >= 50) {
    map.put("balance", map.get("balance") - 50);
}
```

Two threads could both see a balance of `100` and both attempt to withdraw `50`.

The result might remain `50` instead of `0`, depending on the interleaving.

Replacing `HashMap` with `ConcurrentHashMap` does not fix this workflow.

A better solution is to make the read-modify-write operation atomic:

Java

```
map.compute("balance", (key, balance) -> {
    if (balance == null || balance < 50) {
        throw new IllegalStateException(
                "Insufficient balance"
        );
    }

    return balance - 50;
});
```

This makes the update for that key atomic with respect to competing map updates.

However, a real banking application should not use an in-memory map as its source of truth for financial balances. Database transactions, durable constraints, and appropriate isolation or locking are needed for real financial operations.

Interview rule: Whenever you hear "thread-safe," immediately ask, "Thread-safe at what level: a method, a compound operation, an object, or the entire business transaction?"

# 10. Fail-fast iterators and ConcurrentModificationException

Many collection implementations, including `ArrayList` and `HashMap`, have iterators that are fail-fast on a best-effort basis.

Example:

Java

```
List<String> names = new ArrayList<>(
        List.of("Alice", "Bob", "Charlie")
);

for (String name : names) {
    if (name.equals("Bob")) {
        names.remove(name);
    }
}
```

This can throw `ConcurrentModificationException`.

The exception can occur because the collection is structurally modified outside the iterator's supported modification mechanism while iteration is in progress.

## Does this mean another thread must be involved?

No.

A single thread can trigger `ConcurrentModificationException` by structurally modifying a collection during iteration in an unsupported way.

Likewise, not every concurrent modification necessarily produces the exception. Fail-fast behavior is a bug-detection mechanism, not a synchronization guarantee.

## How should you remove elements while iterating?

Use the iterator's `remove()` method when supported:

Java

```
Iterator<String> iterator = names.iterator();

while (iterator.hasNext()) {
    String name = iterator.next();

    if (name.equals("Bob")) {
        iterator.remove();
    }
}
```

Or use an appropriate collection method:

Java

```
names.removeIf(name -> name.equals("Bob"));
```

These approaches address single-threaded iteration and removal semantics. They do not make arbitrary concurrent modifications safe.

## What about ConcurrentHashMap iterators?

`ConcurrentHashMap` iterators are weakly consistent:

* They do not throw `ConcurrentModificationException` merely because the map changes concurrently.

* They can reflect some updates that happen after iteration begins.

* They do not promise a fixed snapshot of the entire map.

If your application needs a stable snapshot, you must design for that requirement explicitly.

# 11. Immutable collections: Another way to avoid concurrency problems

Sometimes the best way to manage shared state is to prevent mutation entirely.

For example:

Java

```
List<String> roles =
        List.of("ADMIN", "USER", "AUDITOR");
```

This list cannot be structurally modified.

Immutable collections are useful for configuration, constants, and reference data shared across threads.

But there is an important detail:

Java

```
List<UserProfile> profiles = List.of(profile1, profile2);
```

The list is unmodifiable, but the `UserProfile` objects may still be mutable.

This is called shallow immutability.

Also, the fact that a collection is immutable does not by itself guarantee safe publication of its reference. Ensure that shared state is published through a mechanism that provides the required Java Memory Model guarantees.

For a stronger design, use immutable elements, construct the collection fully before sharing it, and safely publish the resulting reference.

# 12. Common mistakes developers make

| Mistake                                                                 | Why it is wrong                                          | Better thinking                                             |
| ----------------------------------------------------------------------- | -------------------------------------------------------- | ----------------------------------------------------------- |
| Using `HashMap` as a shared mutable cache                               | Concurrent mutation is unsupported                       | Use a concurrent map or explicit coordination               |
| Assuming `ConcurrentHashMap` makes values thread-safe                   | It protects map operations, not arbitrary value mutation | Use immutable values or coordinate their state              |
| Assuming synchronized methods make a workflow atomic                    | Multiple method calls may interleave                     | Synchronize the entire workflow or use atomic APIs          |
| Assuming `ConcurrentModificationException` means multiple threads       | A single thread can trigger it                           | Understand iterator modification rules                      |
| Using `CopyOnWriteArrayList` for frequent writes                        | Repeated array copying can be expensive                  | Choose based on read/write ratio                            |
| Using an unbounded queue for uncontrolled traffic                       | Backlog can consume memory                               | Use bounded queues and define backpressure                  |
| Using a concurrent collection for distributed coordination              | In-memory guarantees do not span processes               | Use a database, distributed store, or broker as appropriate |
| Assuming `Collections.unmodifiableList()` creates a deep immutable copy | It may wrap a mutable backing list                       | Understand wrapper versus copy semantics                    |

# 13. Best-case and worst-case usage

## When non-thread-safe collections are the best choice

Non-thread-safe collections are often ideal when:

* A collection is created and used inside a single method.

* A collection belongs exclusively to one thread.

* Access is protected by a correctly designed external lock.

* A collection is built privately and then safely published without subsequent mutation.

* You need a lightweight temporary data structure for an algorithm.

Example:

Java

```
public List<String> normalizeNames(List<String> input) {
    List<String> result = new ArrayList<>();

    for (String name : input) {
        result.add(name.trim().toLowerCase());
    }

    return result;
}
```

If `result` remains local to this invocation and is not shared before construction completes, an `ArrayList` is a sensible choice.

## When they become a poor choice

Be cautious when:

* Multiple web request threads modify the same collection.

* Scheduled jobs and request threads share mutable state.

* Multiple executor tasks update the same counters or maps.

* Shared collections are accessed without a clear ownership or synchronization policy.

* The application relies on compound operations being atomic.

* The data must remain correct across multiple application instances.

The worst design is not necessarily using `ArrayList` or `HashMap`. It is sharing mutable state without defining how concurrent operations are coordinated.

# 14. Architect-level decision framework

When reviewing a system design or pull request, ask these questions in order.

## Choose the right concurrency strategy

1. Is the collection shared between threads?

No

Use a normal collection such as `ArrayList` or `HashMap`.

Yes

Continue to question 2.

2. Is the data immutable after safe publication?

Yes

An immutable collection may be sufficient.

No

Continue to question 3.

3. What concurrency pattern do you need?

* Shared map with concurrent updates → `ConcurrentHashMap`.

* Read-heavy shared list → consider `CopyOnWriteArrayList`.

* Simple synchronized access → consider a synchronized wrapper.

* Producer-consumer coordination → `BlockingQueue` or `ConcurrentLinkedQueue`, depending on requirements.

* Sorted concurrent keys → `ConcurrentSkipListMap` or a suitable concurrent sorted set.

4. Is a multi-step operation atomic?

Ensure that the whole business operation is coordinated, not just individual collection methods.

5. Must correctness survive multiple instances or restarts?

If yes, in-memory thread safety alone is insufficient. Evaluate durable storage, transactions, idempotency, and distributed coordination.

This is the level of reasoning expected from a Technical Architect: choose based on semantics, workload, failure handling, and consistency requirements rather than just replacing every `HashMap` with `ConcurrentHashMap`.

# 15. Interview questions and answers

## Basic questions

### Q1. What is a non-thread-safe collection?

A collection that does not provide the necessary guarantees for concurrent access and modification without external coordination.

### Q2. Name five non-thread-safe collections.

`ArrayList`, `LinkedList`, `HashMap`, `HashSet`, and `TreeMap`.

### Q3. Why is `ArrayList` not thread-safe?

It does not coordinate concurrent access to its backing array, size, or structural modifications.

### Q4. Is `HashMap` thread-safe?

No. Concurrent reads and writes or concurrent writes require appropriate synchronization or a concurrent alternative.

### Q5. Is `Vector` thread-safe?

Its individual methods are synchronized. However, compound operations and iteration still require careful coordination.

### Q6. Is `Hashtable` thread-safe?

Its individual methods are synchronized, but compound operations and iteration are not automatically atomic as a whole. `ConcurrentHashMap` is usually a better modern choice for concurrent map access.

### Q7. What is the difference between `ArrayList` and `CopyOnWriteArrayList`?

`ArrayList` does not support concurrent mutation safely. `CopyOnWriteArrayList` creates a new backing array for mutations and supports snapshot-style iteration, making it useful for read-heavy workloads.

### Q8. Can we make an `ArrayList` thread-safe?

Yes. One option is:

Java

```
List<String> list =
        Collections.synchronizedList(new ArrayList<>());
```

You can also use `CopyOnWriteArrayList` when its read/write trade-offs fit the workload.

### Q9. What is `ConcurrentModificationException`?

A runtime exception commonly raised by fail-fast iterators when structural modification occurs outside the supported iteration mechanism. It can happen in a single thread and is not a reliable concurrency detector.

### Q10. What is the difference between fail-fast and weakly consistent iterators?

* Fail-fast iterators attempt to detect certain structural modifications and may throw `ConcurrentModificationException`.

* Weakly consistent iterators, such as those of `ConcurrentHashMap`, tolerate concurrent updates without that fail-fast behavior and do not promise a fixed snapshot.

## Intermediate questions

### Q11. Does `ConcurrentHashMap` allow null keys or values?

No. It rejects both null keys and null values.

### Q12. Does `ConcurrentHashMap` make an entire sequence of operations atomic?

No. Use atomic map methods such as `compute`, `merge`, or `putIfAbsent` when their semantics match the operation. More complex workflows may require additional coordination.

### Q13. What is the difference between `Collections.synchronizedMap()` and `ConcurrentHashMap`?

A synchronized map wrapper synchronizes method calls through a common mutex. `ConcurrentHashMap` is designed for scalable concurrent access and provides atomic per-key operations, but not a universal transaction across multiple keys.

### Q14. Can multiple threads safely read an `ArrayList`?

Concurrent read-only access can be safe if the list is properly initialized, safely published, and never modified afterward. An immutable collection is often clearer when immutability is the intended design.

### Q15. Is `CopyOnWriteArrayList` suitable for 100,000 writes per second?

Usually not without careful workload-specific benchmarking. Every mutation can require copying a large array, making frequent writes costly.

### Q16. What is the difference between a concurrent collection and a synchronized collection?

A synchronized wrapper generally serializes method calls through a shared lock. Concurrent collections use concurrency strategies suited to their operations and may allow greater parallelism. The actual guarantees vary by implementation.

### Q17. Can a thread-safe map contain a non-thread-safe object?

Yes. For example, a `ConcurrentHashMap<String, ArrayList<String>>` does not make the contained lists safe for concurrent modification.

### Q18. Is a concurrent collection enough for a distributed application?

No. Its thread-safety guarantees generally apply within the relevant JVM and collection instance. Distributed consistency requires additional mechanisms.

## Advanced and scenario-based questions

### Q19. Two threads increment a counter in a `ConcurrentHashMap`. Can updates be lost?

Yes, if they use separate `get()` and `put()` calls for a read-modify-write operation. Use `compute`, `merge`, or a suitable atomic counter such as `LongAdder`.

### Q20. Two threads execute `if (!map.containsKey(key)) map.put(key, value)`. Is this safe with `ConcurrentHashMap`?

The individual methods are safe, but the overall check-then-act operation is not atomic. Use `putIfAbsent()` or another appropriate atomic method.

### Q21. A notification system reads a list of listeners on every request, but listeners are rarely added or removed. Which collection might you choose?

`CopyOnWriteArrayList` is a strong candidate because iteration is frequent and mutation is rare.

### Q22. A system requires strict consistency while iterating over all entries in a map. Is `ConcurrentHashMap` iteration sufficient?

Not necessarily. Its iterators are weakly consistent, not whole-map snapshots. Use a suitable snapshot or coordination strategy if the business requirement demands a consistent view.

### Q23. A Spring Boot application uses a static `HashMap` cache. Is it thread-safe?

Not if multiple request threads mutate it without coordination. A static field is shared by code using that class in the same class-loader context; it does not imply thread safety.

### Q24. Why might you choose `ArrayList` over a thread-safe collection in a multithreaded application?

If the list is local to a thread or method and not shared while being modified, a normal `ArrayList` is simpler and avoids unnecessary synchronization overhead.

### Q25. What should you investigate when a production application has intermittent collection-related failures?

Check for shared mutable state, unsynchronized access, check-then-act sequences, unsafe iteration, mutation of stored values, object publication, and assumptions about atomicity. Reproduce the issue with a focused concurrency test and inspect thread dumps and relevant logs.

# 16. Quick revision cheat sheet

## Remember these ten rules

1. `ArrayList` and `HashMap` are not thread-safe. They are good general-purpose collections when used correctly.

2. Thread-safe is not the same as atomic workflow. Two safe calls can still form an unsafe sequence.

3. `ConcurrentHashMap` is the standard starting point for shared concurrent maps. It does not make stored values thread-safe.

4. `Collections.synchronizedList()` synchronizes individual method calls. External synchronization is needed for safe iteration and compound workflows.

5. `CopyOnWriteArrayList` favors reads over writes. It is particularly useful for listener registries.

6. `BlockingQueue` is designed for producer-consumer coordination. A bounded queue can provide backpressure.

7. `ConcurrentModificationException` does not prove that multiple threads are involved. One thread can trigger it.

8. Concurrent collections do not solve distributed consistency. Multiple JVMs and process failures need additional design.

9. Immutability reduces shared-state risks. But mutable elements and unsafe publication still matter.

10. Choose based on the workload and required guarantees. Reads, writes, iteration, atomicity, contention, and durability all matter.

# 17. Final interview answer: Explain non-thread-safe collections in 60 seconds

If an interviewer asks, "Explain non-thread-safe collections in Java and how you handle them in production," you can answer:

> Non-thread-safe collections such as `ArrayList` and `HashMap` do not provide the necessary concurrency guarantees for unsynchronized concurrent modification. When multiple threads access shared mutable collections, we can encounter race conditions, lost updates, inconsistent observations, and iteration-related issues.
>
> My solution depends on the use case. For a shared map with concurrent updates, I generally consider `ConcurrentHashMap`. For a read-heavy listener registry, `CopyOnWriteArrayList` may be appropriate. For simple shared collections, a synchronized wrapper can work, provided compound operations and iteration are handled correctly. For producer-consumer workloads, I would consider a bounded `BlockingQueue`.
>
> At the architectural level, I also distinguish thread safety from atomic business operations. A concurrent collection does not automatically make a multi-step workflow atomic, protect mutable values stored inside it, or guarantee consistency across multiple application instances. For critical operations, I would evaluate atomic APIs, immutable state, database transactions, idempotency, and distributed coordination based on the requirements.

That answer demonstrates collection knowledge, concurrency fundamentals, and architectural judgment.

## Test yourself: Can you answer these without looking above?

## 5-question interview quiz

5/5 answered

1. You have a shared cache modified by multiple request threads. What is your first collection candidate?

ArrayList

ConcurrentHashMap

TreeMap without synchronization

Correct

ConcurrentHashMap is designed for shared maps with concurrent access and updates.

2. Does ConcurrentHashMap make get-then-put atomic?

Yes, all calls are atomic together

No, separate calls can interleave

Only if the map has fewer than 1,000 entries

Review this answer

Use an atomic operation such as putIfAbsent, compute, or merge when appropriate.

3. Which collection is a strong fit for frequent reads and rare list modifications?

CopyOnWriteArrayList

LinkedList

ArrayDeque

Correct

CopyOnWriteArrayList trades more expensive writes for efficient snapshot-style iteration.

4. Can ConcurrentModificationException happen in a single thread?

No, it requires multiple threads

Yes, when iteration rules are violated

Only with HashMap

Correct

Fail-fast behavior can be triggered by a single thread modifying a collection during iteration.

5. You need a durable guarantee that an order is processed only once across several application instances. Is a concurrent set enough?

Yes, it is always enough

No, use durable idempotency and appropriate coordination

Yes, if the set is static

Correct

An in-memory concurrent set does not coordinate independent JVMs or survive process restarts.

Review answers again

### Score: 4 / 5

Try again

My recommendation for your interview preparation: Master three things especially well: `HashMap` vs `ConcurrentHashMap`, atomic operations vs thread-safe methods, and thread safety vs distributed consistency. These concepts help you move beyond memorized definitions and answer the production scenarios expected of a senior engineer or Technical Architect.
