# Java Thread-Safety & Concurrent Collections: The Interview Cheatsheet

*A Medium-style guide: simple enough for a 10-year-old, deep enough for a senior engineer interview.*

---

## 1. The Big Picture (Explain Like I'm 10)

Imagine a **shared notebook** in a classroom. If one student writes while another is erasing, the page becomes a mess. That mess is a **race condition**.

- **Thread** = a student.
- **Shared collection** = the notebook.
- **Thread-safe** = a rule system so students never ruin each other's work.

> **One-liner for interviews:** *"Thread-safety means a piece of code behaves correctly when multiple threads access it at the same time, without us adding extra synchronization from outside."*

---

## 2. Why Normal Collections Break

`ArrayList`, `HashMap`, `HashSet` are **not thread-safe**.

| Problem | What happens |
|---|---|
| **Race condition** | Two threads `put()` at the same time, so one write is lost |
| **Lost update** | Thread A and B read `count=5`, both write `6` (should be 7) |
| **`ConcurrentModificationException`** | One thread iterates while another modifies |
| **Visibility problem** | Thread A's change isn't seen by Thread B (CPU caching) |
| **Infinite loop (old Java 7 HashMap)** | Concurrent resize could create a circular linked list, causing 100% CPU |
| **Corrupted state** | `ArrayList` size and internal array go out of sync, giving `ArrayIndexOutOfBounds` or `null` entries |

```java
// BROKEN: don't do this in production
Map<String, Integer> map = new HashMap<>();
// 10 threads calling map.put(...) simultaneously = data loss / corruption
```

---

## 3. The Old Way: Synchronized Wrappers (and why they're weak)

```java
List<String> list = Collections.synchronizedList(new ArrayList<>());
Map<String, String> map = Collections.synchronizedMap(new HashMap<>());
Map<String, String> legacy = new Hashtable<>();   // legacy, avoid
Vector<String> vec = new Vector<>();               // legacy, avoid
```

**How they work:** one lock around every method. Only **one thread at a time**, like a single-lane bridge.

**Problems:**
- **Poor scalability:** all threads queue up, even for reads.
- **Compound operations are NOT atomic:**
  ```java
  if (!list.contains(x)) {   // check
      list.add(x);           // act  <- another thread can sneak in between!
  }
  ```
- **Iteration still needs manual locking:**
  ```java
  synchronized (list) {
      for (String s : list) { ... }
  }
  ```

---

## 4. The Modern Way: `java.util.concurrent` Collections

Think of them as **smart notebooks with traffic rules built-in**, with finer-grained locking or lock-free algorithms, so many threads can work simultaneously.

---

## 5. ConcurrentHashMap (The Star of Interviews)

### 5.1 The Analogy
A **big library with many separate shelves (buckets)**. Instead of locking the whole library, you lock only the shelf you're using. Readers don't even need to wait.

### 5.2 How It Works Internally

| Version | Mechanism |
|---|---|
| **Java 7** | **Segment locking**: map split into 16 segments (default), each with its own lock |
| **Java 8+** | **CAS + `synchronized` on individual bucket head node**. No segments. Finer-grained, better scaling |

**Java 8+ details (frequently asked):**
- **Empty bucket** → insert using **CAS** (Compare-And-Swap), lock-free.
- **Non-empty bucket** → `synchronized` on the **first node of that bucket** only.
- **Reads (`get`)** → **no locking at all**, using `volatile` fields for visibility.
- **Collision handling** → linked list, converted to **Red-Black Tree when bucket size ≥ 8** (and table capacity ≥ 64), reverts to a list when ≤ 6.
- **Resizing** → multiple threads can **help** transfer buckets (cooperative resize).

### 5.3 Golden Rules
- **No `null` keys or values.** (`null` is ambiguous: "absent" or "mapped to null?" in a concurrent world.)
- **Iterators are weakly consistent:** they never throw `ConcurrentModificationException`, and may or may not reflect updates made after the iterator was created.
- **`size()` is an estimate** under concurrent updates. Use `mappingCount()` for large maps (returns `long`).

### 5.4 Atomic Methods (Know These Cold!)

```java
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();

map.putIfAbsent("a", 1);                       // insert only if missing
map.computeIfAbsent("b", k -> expensiveLoad(k)); // lazily compute once
map.computeIfPresent("a", (k, v) -> v + 1);
map.compute("a", (k, v) -> v == null ? 1 : v + 1);
map.merge("hits", 1, Integer::sum);            // BEST for counters
map.replace("a", 1, 2);                        // CAS-style replace
map.remove("a", 1);                            // remove only if value matches
```

### 5.5 The #1 Trap: Check-Then-Act

```java
// WRONG: not atomic even on ConcurrentHashMap
if (!map.containsKey(k)) {
    map.put(k, v);
}

// RIGHT
map.putIfAbsent(k, v);

// RIGHT for counters
map.merge(word, 1, Integer::sum);
```

> **Interview gold:** *"ConcurrentHashMap makes individual operations thread-safe, but a sequence of operations is not automatically atomic. Use compound atomic methods."*

### 5.6 `compute*` Caveats
- The lambda runs **while holding a bucket lock**. Keep it **short, fast, and non-blocking**.
- **Never modify the same map inside the lambda**, which can cause deadlock or `IllegalStateException`.
- Don't do network or DB calls inside the lambda (blocks other threads on that bucket).

### 5.7 Production Scenarios
- **In-memory cache** (`computeIfAbsent` for lazy loading)
- **Rate limiter**: per-user request counters (`merge`)
- **Session store** / user-token registry
- **Word-frequency / metrics counters**
- **Service registry / config holder**
- **Deduplication** of in-flight requests (`Map<Key, CompletableFuture>`)

---

## 6. CopyOnWriteArrayList (COWAL)

### 6.1 The Analogy
A **school notice board**. Everyone reads the same board. When the teacher wants to change something, she **makes a fresh copy of the entire board**, edits the copy, then swaps in the new board. Readers who were reading the old one are not disturbed.

### 6.2 How It Works
- Holds an internal `volatile Object[] array`.
- **Read** → just reads the array. **No lock. Super fast.**
- **Write (add/set/remove)** → takes a `ReentrantLock`, **copies the whole array**, modifies the copy, then swaps the reference.
- **Iterator** → works on a **snapshot** of the array at creation time. Never throws `ConcurrentModificationException`.

### 6.3 Key Properties

| Property | Detail |
|---|---|
| Reads | Lock-free, very fast |
| Writes | Expensive: O(n) copy every time |
| Iterator | Snapshot-based; **doesn't support `remove()`/`add()`/`set()`** (throws `UnsupportedOperationException`) |
| Memory | Temporarily 2x during writes |
| Consistency | Iterator may show **stale data** |
| Nulls | Allowed |

### 6.4 When to Use (Best Case)
Reads ≫ writes (like 100:1 or more), small list size:
- **Listener / observer lists** (event handlers, callbacks): the classic use case
- **Configuration / feature-flag lists** that rarely change
- **Whitelist / blacklist / allowed-origins** (CORS origins)
- **Routing tables** that are rarely updated

### 6.5 When NOT to Use (Worst Case)
- Frequent writes → each write copies the whole array (CPU + GC pressure)
- Large lists (100K+ elements) with updates
- When you need **real-time freshness** during iteration
- Write-heavy queues/logs

### 6.6 Sibling: `CopyOnWriteArraySet`
Backed by a COWAL, so same trade-offs. `add()` is O(n) because it checks for duplicates. Good for small listener sets.

---

## 7. Other Concurrent Collections You Must Know

| Collection | Analogy | Use case |
|---|---|---|
| **`ConcurrentLinkedQueue`** | Lock-free line at a ticket counter | Non-blocking FIFO queue (CAS-based) |
| **`ConcurrentLinkedDeque`** | Same, but both ends | Work-stealing, double-ended |
| **`LinkedBlockingQueue`** | Queue with a waiting room | Producer-consumer, thread pools (optionally bounded) |
| **`ArrayBlockingQueue`** | Fixed-size queue (bounded) | Back-pressure; fixed capacity |
| **`PriorityBlockingQueue`** | VIP line | Task scheduling by priority (unbounded) |
| **`DelayQueue`** | Item available only after its timer expires | Retry/expiry/scheduled tasks |
| **`SynchronousQueue`** | Direct hand-to-hand pass (no storage) | `Executors.newCachedThreadPool()` |
| **`LinkedTransferQueue`** | Blocking queue with direct handoff | High-performance producer-consumer |
| **`ConcurrentSkipListMap`** | Sorted ConcurrentHashMap | Thread-safe **sorted** map (like `TreeMap`) |
| **`ConcurrentSkipListSet`** | Sorted concurrent set | Thread-safe sorted set |

**Blocking vs Non-blocking**
- **Blocking queues** (`put`/`take`) → thread **waits** if full/empty. Ideal for producer-consumer.
- **Non-blocking queues** (`offer`/`poll`) → returns immediately (`false`/`null`). Lock-free, CAS-based.

---

## 8. Head-to-Head Comparison Table (Memorize This!)

| Feature | `HashMap` | `Hashtable` | `synchronizedMap` | `ConcurrentHashMap` |
|---|---|---|---|---|
| Thread-safe | No | Yes | Yes | Yes |
| Lock granularity | None | Whole map | Whole map | Per bucket (Java 8+) |
| Null key/value | Both allowed | Neither | Allowed | Neither |
| Read concurrency | n/a | Blocked | Blocked | Lock-free |
| Iterator | Fail-fast | Fail-fast | Fail-fast | Weakly consistent |
| Performance (concurrent) | n/a | Poor | Poor | Excellent |
| Recommended? | Single-thread | Legacy | Meh | **Yes** |

| Feature | `ArrayList` | `Vector` | `synchronizedList` | `CopyOnWriteArrayList` |
|---|---|---|---|---|
| Thread-safe | No | Yes | Yes | Yes |
| Read cost | Fast | Locked | Locked | Lock-free, fast |
| Write cost | Fast | Locked | Locked | **O(n) copy** |
| Iterator | Fail-fast | Fail-fast | Fail-fast (manual lock) | **Snapshot** |
| Best for | Single thread | Legacy | Balanced small use | Read-heavy |

---

## 9. Fail-Fast vs Fail-Safe (Weakly Consistent)

| | Fail-Fast | Fail-Safe / Weakly Consistent |
|---|---|---|
| Examples | `ArrayList`, `HashMap` | `ConcurrentHashMap`, `CopyOnWriteArrayList` |
| On concurrent modification | Throws `ConcurrentModificationException` (via `modCount` check) | No exception |
| Reflects live changes? | n/a | CHM: maybe. COWAL: **never** (snapshot) |
| Guarantee | Best-effort detection, **not** a thread-safety guarantee | Safe traversal |

> **Tip:** Even a single thread can trigger CME by removing inside a for-each loop. Use `Iterator.remove()` or `removeIf()`.

---

## 10. Core Concepts Behind the Scenes (Interview Depth)

**Race condition:** outcome depends on thread timing.
**Atomicity:** an operation happens fully or not at all.
**Visibility:** one thread's write is seen by others (`volatile`, locks, happens-before).
**Ordering:** the CPU/compiler can reorder instructions; the Java Memory Model defines guarantees.
**CAS (Compare-And-Swap):** "If the value is still X, change it to Y, else retry." It's a lock-free, optimistic technique used by `Atomic*` classes and `ConcurrentLinkedQueue`.
**`volatile`:** guarantees visibility and ordering, but **not** atomicity for compound actions (`count++`).
**`synchronized` vs `ReentrantLock`:** both give mutual exclusion; `ReentrantLock` adds `tryLock`, timeouts, fairness, and interruptibility.
**Happens-before:** a write to a `ConcurrentHashMap` entry **happens-before** a subsequent read of that entry from another thread.

---

## 11. Production Scenarios (Story Time)

### Scenario 1: Request Counter (rate limiting)
```java
ConcurrentHashMap<String, LongAdder> counts = new ConcurrentHashMap<>();
counts.computeIfAbsent(userId, k -> new LongAdder()).increment();
```
`LongAdder` is better than `AtomicLong` under heavy contention, because it spreads updates across cells.

### Scenario 2: Cache with Lazy Loading
```java
ConcurrentHashMap<String, Config> cache = new ConcurrentHashMap<>();
Config c = cache.computeIfAbsent(key, k -> loadFromDb(k)); // loads ONCE per key
```
Caveat: a slow loader blocks others in that bucket. For heavy loads, cache a `CompletableFuture` or use Caffeine/Guava.

### Scenario 3: Event Listeners
```java
List<Listener> listeners = new CopyOnWriteArrayList<>();
void fire(Event e) { for (Listener l : listeners) l.onEvent(e); } // safe even if a listener unregisters itself
```

### Scenario 4: Producer-Consumer (Log pipeline / Pub-Sub on GCP)
```java
BlockingQueue<Message> q = new ArrayBlockingQueue<>(1000);
// producer: q.put(msg);   // blocks when full = natural back-pressure
// consumer: q.take();     // blocks when empty
```
*Cloud-engineer tie-in:* a Pub/Sub subscriber callback thread pushes messages into a bounded queue, and worker threads drain it. This protects your service from memory blow-up.

### Scenario 5: Dedup In-Flight Requests
```java
ConcurrentHashMap<String, CompletableFuture<Result>> inFlight = new ConcurrentHashMap<>();
inFlight.computeIfAbsent(key, k -> CompletableFuture.supplyAsync(() -> fetch(k)));
```

---

## 12. Best Case vs Worst Case Summary

| Collection | Best Case | Worst Case |
|---|---|---|
| **ConcurrentHashMap** | Many readers + many writers on different keys | All threads hammering **one hot key** (contention), long-running `compute` lambdas |
| **CopyOnWriteArrayList** | Read-heavy, rarely-written, small list | Frequent writes / large list = O(n) copy storm, GC pressure |
| **synchronizedMap/List** | Low concurrency, simple code | High concurrency = lock contention, bottleneck |
| **BlockingQueue (bounded)** | Back-pressure, steady flow | Poorly sized queue = deadlock/starvation or memory issues |
| **ConcurrentLinkedQueue** | High-throughput non-blocking | Unbounded growth, which can cause OOM if consumers lag |

---

## 13. Common Mistakes (Interviewers Love These)

1. Assuming `ConcurrentHashMap` makes **everything** atomic. (Only single operations.)
2. Using `get` + `put` instead of `putIfAbsent`/`merge`.
3. Putting `null` into a `ConcurrentHashMap` (throws `NullPointerException`).
4. Using `CopyOnWriteArrayList` for a write-heavy workload.
5. Relying on `size()` for exact logic in concurrent code.
6. Doing blocking I/O inside `compute`/`computeIfAbsent`.
7. Modifying the map inside its own `compute` lambda.
8. Wrapping a `ConcurrentHashMap` in another `synchronized` block (defeats its purpose).
9. Thinking `volatile` makes `count++` safe.
10. Using an **unbounded queue** in production (hidden OOM bomb).
11. Using thread-safe collections but **mutable objects inside** them. The collection is safe, but the objects aren't.

---

## 14. Rapid-Fire Interview Q&A

**Q1. Is `ConcurrentHashMap` fully locked like `Hashtable`?**
No. Java 8+ uses CAS for empty buckets and `synchronized` on the bucket's head node only. Reads are lock-free.

**Q2. Why doesn't `ConcurrentHashMap` allow nulls?**
`get(k)` returning `null` would be ambiguous (missing vs. mapped to null), and in a concurrent setting you can't safely double-check with `containsKey`.

**Q3. Does `ConcurrentHashMap` throw `ConcurrentModificationException`?**
No. Its iterators are weakly consistent.

**Q4. When does a bucket become a tree?**
When a bucket's chain length reaches 8 (TREEIFY_THRESHOLD) and table size is at least 64. Otherwise the table resizes. It untreeifies at 6.

**Q5. What is the default concurrency level in Java 8+?**
The `concurrencyLevel` constructor parameter is only a sizing hint now. There are no segments anymore.

**Q6. `HashMap` vs `ConcurrentHashMap` vs `Hashtable`?**
See the comparison table in Section 8.

**Q7. Why is `CopyOnWriteArrayList` iterator "snapshot"?**
The iterator holds a reference to the array that existed when it was created. Writes create new arrays, so the old one is never modified.

**Q8. Can I remove via a `CopyOnWriteArrayList` iterator?**
No: `UnsupportedOperationException`. Call `list.remove()` on the list itself.

**Q9. Is `size()` accurate in `ConcurrentHashMap`?**
Only an estimate under concurrent modification. Use `mappingCount()` for big maps.

**Q10. `synchronizedList` vs `CopyOnWriteArrayList`?**
`synchronizedList` has one lock for all operations and needs manual locking for iteration. COWAL has lock-free reads and copy-on-write, so it's best for read-heavy workloads.

**Q11. How do you make a compound operation atomic?**
Use built-in atomic methods (`putIfAbsent`, `compute`, `merge`), or an explicit lock, or `Atomic*` classes.

**Q12. What is a `ConcurrentSkipListMap`?**
A thread-safe sorted map using a skip list, with O(log n) operations and no global lock.

**Q13. `AtomicLong` vs `LongAdder`?**
`LongAdder` is better under high contention (striped cells), while `AtomicLong` is better when you need the exact value frequently.

**Q14. Does thread-safe collection = thread-safe application?**
No. Compound logic across multiple collections or operations still needs proper design.

**Q15. How would you pick a collection?**
Read-heavy and small → `CopyOnWriteArrayList`. Key-value with high concurrency → `ConcurrentHashMap`. Producer-consumer → `BlockingQueue`. Sorted → `ConcurrentSkipListMap`.

**Q16. `Collections.unmodifiableList` vs `CopyOnWriteArrayList`?**
Unmodifiable is a read-only **view** (the underlying list can still change, and it's not thread-safe). COWAL is mutable and thread-safe.

**Q17. Is `List.of()` thread-safe?**
Yes. Immutable collections are inherently thread-safe, so **prefer immutability whenever possible**.

---

## 15. Decision Flowchart (Mental Model)

```
Need a collection shared across threads?
│
├─ Can it be IMMUTABLE?            → List.of / Map.of / Set.of ✅
│
├─ Key-Value?
│    ├─ Sorted needed?             → ConcurrentSkipListMap
│    └─ Otherwise                  → ConcurrentHashMap
│
├─ List/Set?
│    ├─ Read-heavy, small, rare writes → CopyOnWriteArrayList / Set
│    └─ Write-heavy                → ConcurrentLinkedQueue/Deque, or a queue
│                                     (or synchronize + ArrayList for simple cases)
│
└─ Producer–Consumer?
     ├─ Need back-pressure         → ArrayBlockingQueue / LinkedBlockingQueue (bounded)
     ├─ Non-blocking, high throughput → ConcurrentLinkedQueue
     └─ Priority / delay           → PriorityBlockingQueue / DelayQueue
```

---

## 16. Cloud-Engineer Angle (Your Edge as GCP ACE)

Tie your cloud knowledge in. This impresses interviewers:

- **Stateless services on Cloud Run / GKE:** each instance has its **own JVM**, so a `ConcurrentHashMap` cache is **per-instance**, not shared. For shared state, use **Memorystore (Redis)** or **Firestore/Spanner**.
- **Autoscaling:** in-memory counters (rate limits) reset on scale-in or scale-out, so use a distributed store for global limits.
- **Pub/Sub subscribers:** callbacks run on multiple threads, so shared state must be thread-safe. Use bounded queues for **back-pressure** and **flow control**.
- **Memory limits (container OOMKilled):** unbounded queues or ever-growing `ConcurrentHashMap` caches can cause OOM. Always **bound and evict** (use Caffeine for TTL/size eviction).
- **Graceful shutdown (SIGTERM):** drain your `BlockingQueue` before the pod dies.
- **Thread pools & CPU:** Cloud Run/GKE CPU limits affect `Runtime.availableProcessors()`, so size your pools accordingly.

> **Key line:** *"Thread-safe collections protect memory inside one JVM. They do nothing across pods. For distributed consistency, I'd use Redis, Spanner, or distributed locks."*

---

## 17. 60-Second Cheat Card (Last-Minute Revision)

- **Thread-safe** = correct under concurrent access.
- **HashMap/ArrayList** = NOT thread-safe.
- **synchronizedX / Hashtable / Vector** = one big lock, slow, compound ops still unsafe.
- **ConcurrentHashMap**: CAS plus per-bucket `synchronized` (Java 8+), lock-free reads, no nulls, weakly consistent iterators, treeify at 8, use `merge`/`compute`/`putIfAbsent`.
- **CopyOnWriteArrayList**: copy the array on every write, lock-free reads, snapshot iterator, **read-heavy and small only**.
- **Compound ops** (check-then-act) are the #1 bug, so use atomic methods.
- **Blocking queues** give back-pressure and are used for producer-consumer. Always **bound** them.
- **Immutable > thread-safe > synchronized** (prefer in this order).
- **In-JVM safety ≠ distributed safety.** Use Redis or similar across pods.
- **Keep lambdas in `compute*` short**, with no I/O or recursion into the same map.

---

## 18. How to Answer in an Interview (Template)

> *"Thread-safety means code works correctly under concurrent access. Standard collections like HashMap aren't thread-safe. Older fixes like Hashtable or synchronizedMap use a single lock, which hurts scalability. Modern `java.util.concurrent` collections solve this: `ConcurrentHashMap` uses CAS and per-bucket locking for high-throughput key-value access, while `CopyOnWriteArrayList` copies on write for lock-free reads, ideal for read-heavy data like listeners. I choose based on read/write ratio, and I'm careful that compound operations need atomic methods like `merge` or `computeIfAbsent`. In cloud environments like GKE or Cloud Run, I remember these protect only a single JVM, so for shared state I'd use Redis or Spanner."*

---

