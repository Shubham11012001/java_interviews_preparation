# Java Collections: Single-Threaded and Concurrent

I've focused on what interviewers actually ask: `ArrayList`, `HashMap`, and `ConcurrentHashMap` get the most depth. Everything else gets just enough. Version-specific points are marked with 🚩.

---

## 1. ONE-LINE DEFINITION + WHY IT EXISTS

**Definition:** The Collections Framework is Java's set of ready-made containers (List, Set, Map, Queue) with standard behavior, so you don't build your own data structures.

**Problem it solves:** Without it, every team writes its own linked list, hash table, and sorted structure. Each is buggy in its own way, none share an interface, and you can't swap one for another.

**Why the concurrent ones exist:** Normal collections are **not safe when two threads touch them at the same time**. You get lost data, corrupted structures, or infinite loops. The concurrent collections (`ConcurrentHashMap`, `CopyOnWriteArrayList`, `BlockingQueue`, etc.) give you safety without one big lock that makes everything slow.

---

## 2. MENTAL MODEL

**ArrayList = a row of numbered lockers.**
Getting locker #5 is instant. Inserting in the middle means everyone after it shifts one locker. When the row is full, you move everything into a bigger row (1.5x).

**LinkedList = a treasure hunt.**
Each clue points to the next. To reach clue #500, you must follow 499 clues.

**HashMap = a hotel with pigeonholes.**
The guest's name goes through a formula (hash) that gives a room number. If two guests get the same room, they stand in a small queue in that room. When the hotel is 75% full, you build one twice as big and reassign rooms.

```
key ──hashCode()──► spread bits ──► index = (n-1) & hash ──► bucket
                                                              │
   empty?  → put node here                                    │
   else    → walk the chain: same key? replace : append at tail
   chain ≥ 8 and table ≥ 64?  → convert chain to red-black tree
   size > 0.75 × capacity?    → resize (double the table)
```

**ConcurrentHashMap = same hotel, but each room has its own tiny lock.**
Two people going to different rooms never wait for each other. Reading never needs a lock at all. For an empty room, a "first come first served" trick (CAS) replaces the lock.

**CopyOnWriteArrayList = a printed notice board.**
To change it, you print a whole new board and swap it in. People reading the old board are never disturbed, but printing is expensive.

**BlockingQueue = a conveyor belt with limited length.**
If the belt is full, the producer waits. If it's empty, the consumer waits.

---

## 3. INTERNALS

### 3.1 ArrayList
- Backed by `Object[] elementData`. `get(i)` is O(1). `add(e)` is amortized O(1). `add(index, e)` and `remove(index)` are O(n) because of shifting.
- **Growth:** new capacity = old + old/2 (1.5x), using `Arrays.copyOf`.
- 🚩 **Java 8+:** `new ArrayList<>()` starts with an empty array and allocates 10 only on the first add. (Java 7 allocated 10 immediately.)
- **Fail-fast:** a `modCount` counter is checked by the iterator. If the list is structurally changed outside the iterator, you get `ConcurrentModificationException` (CME). This is *best effort*, not a guarantee.
- **Traps:** `list.remove(1)` removes by *index*, `list.remove(Integer.valueOf(1))` removes by *value*. `Arrays.asList()` is fixed-size, and `add` throws `UnsupportedOperationException`. 🚩 `List.of()` (Java 9+) is fully immutable and rejects nulls.

### 3.2 LinkedList and ArrayDeque (quick)
- `LinkedList` is a doubly linked list. O(1) at the ends, O(n) by index, and each node has heavy memory overhead plus poor CPU-cache behavior.
- In practice `ArrayList` beats it almost always, and `ArrayDeque` beats it as a stack or queue. Interviewers like hearing this.

### 3.3 HashMap (the most asked topic)
- **Structure:** array of buckets (`Node<K,V>[] table`). Capacity is always a **power of 2**. Default is 16, load factor 0.75, so it resizes after 12 entries.
- **Hash spreading:** `h ^ (h >>> 16)`. This mixes high bits into low bits, because the index uses only the low bits: `index = (n-1) & hash`.
- **Put:** empty bucket → place node. Same key (hash equal **and** `equals()` true) → replace value. Otherwise append at the **tail** of the chain.
- **Treeify (🚩 Java 8+):** if a chain reaches 8 nodes **and** the table has at least 64 buckets, the chain becomes a red-black tree. Worst-case lookup drops from O(n) to O(log n). If the table is smaller than 64, it resizes instead. On shrink it goes back to a list at 6 or fewer nodes.
- **Resize:** table doubles. Each node either stays at the same index or moves to `oldIndex + oldCapacity`, decided by one bit (`hash & oldCap`). No full rehash is needed.
- **Null:** one null key is allowed (goes to bucket 0). Null values are allowed.
- **Java 7 vs 8:**
  - Java 7 inserted at the **head** of the chain. During resize, this reversed the order. Two threads resizing at once could create a **circular chain**, so `get()` looped forever at 100% CPU. This is the famous production bug.
  - Java 8 inserts at the tail and preserves order, so that specific infinite loop is gone. **But HashMap is still not thread-safe in Java 8.** You can lose writes or see inconsistent data.
- **Presizing:** to store N entries without resizing, use capacity `N / 0.75 + 1`. 🚩 Java 19+ has `HashMap.newHashMap(N)` to do this for you.

### 3.4 HashSet, LinkedHashMap, TreeMap, PriorityQueue (quick)
- `HashSet` is just a `HashMap` where the value is a dummy object.
- `LinkedHashMap` is HashMap plus a doubly linked list through the entries. It keeps insertion order, or access order if you pass `true` as the third constructor argument. Override `removeEldestEntry` and you have an LRU cache.
- `TreeMap` is a red-black tree. Operations are O(log n), keys are sorted, and it has `floorKey`, `ceilingKey`, and `subMap`. It uses `compareTo`/`Comparator` for equality, **not** `equals`. No null keys.
- `PriorityQueue` is a binary min-heap in an array. `offer` and `poll` are O(log n), `peek` is O(1). **Iterating it does not give sorted order.**
- 🚩 **Java 21:** `SequencedCollection`/`SequencedMap` added `getFirst()`, `getLast()`, and `reversed()` to List, Deque, LinkedHashMap, and so on.

### 3.5 ConcurrentHashMap (CHM)
**Java 7:** The map was split into **16 Segments** (default concurrency level). Each segment was its own small hash table with its own `ReentrantLock`. At most 16 writers at once.

**Java 8+:** Segments were removed. One table, finer locking:
- **Bucket empty?** Insert with **CAS** (compare-and-swap). No lock.
- **Bucket has nodes?** `synchronized` on the **head node of that bucket only**.
- **Get:** fully **lock-free**, using volatile reads.
- **Resize:** cooperative. Other threads that arrive during a resize help move buckets.
- Still uses chain → tree at 8, like HashMap.
- **No null keys or null values.** Reason: if `get(k)` returns `null`, you can't tell "absent" from "value is null", and in a concurrent setting you can't safely double-check with `containsKey`.
- **size()** is approximate under concurrency (uses counter cells, like `LongAdder`). Use `mappingCount()` for a `long`. Never make business decisions on it.
- **Iterators are weakly consistent:** they never throw CME and may or may not show updates made after creation.
- **Atomic methods:** `putIfAbsent`, `computeIfAbsent`, `compute`, `merge`, `replace`. These are the correct way to do "check then act".
- **Memory visibility:** anything a thread did before `put` is visible to another thread after it `get`s that key (happens-before guarantee).
- 🚩 `computeIfAbsent`: the function must be **short** and must **not modify the same map**. In Java 8 a recursive call could hang forever. In Java 9+ it often throws `IllegalStateException: Recursive update`. I'm confident about the trap, but the exact behavior varies by case, so never rely on it.

### 3.6 Other concurrent collections
| Class | How it works |
|---|---|
| `CopyOnWriteArrayList` | Every write copies the whole array under a lock. Iterators work on a snapshot (never CME, but `iterator.remove()` is unsupported). 🚩 The lock type (`ReentrantLock` vs `synchronized`) changed across versions. I'm not certain which version, and interviewers rarely ask. |
| `ConcurrentLinkedQueue` | Lock-free, CAS-based linked queue. `size()` is O(n). |
| `ArrayBlockingQueue` | Bounded, array-based, **one lock** for both put and take. |
| `LinkedBlockingQueue` | Linked, **two locks** (one for put, one for take), so producers and consumers don't block each other. 🚩 **Default capacity is `Integer.MAX_VALUE`, which is effectively unbounded.** |
| `PriorityBlockingQueue` | Thread-safe heap, unbounded. |
| `SynchronousQueue` | Zero capacity. A put waits for a matching take (hand-off). |
| `ConcurrentSkipListMap` | Lock-free sorted map (skip list), O(log n). The concurrent version of `TreeMap`. |
| `Collections.synchronizedXxx` / `Vector` / `Hashtable` | One mutex for everything. Safe per call, but slow under contention and **iteration still needs manual locking**. |

---

## 4. ALTERNATIVES & COMPARISON

**Maps**

| Type | Order | Thread-safe | Nulls | Speed | Wins when |
|---|---|---|---|---|---|
| `HashMap` | None | No | 1 null key, null values OK | O(1) avg | Default, single thread or confined to one thread |
| `LinkedHashMap` | Insertion or access | No | Yes | O(1) | Predictable iteration, LRU cache |
| `TreeMap` | Sorted | No | No null key | O(log n) | Range queries, sorted keys |
| `ConcurrentHashMap` | None | Yes (fine-grained) | **No nulls** | O(1) avg | Shared cache or counters across threads |
| `synchronizedMap` / `Hashtable` | None | Yes (one big lock) | Hashtable: no nulls | O(1) but contended | Legacy code only |
| `ConcurrentSkipListMap` | Sorted | Yes | No | O(log n) | Sorted map shared by many threads |

**Lists and queues**

| Type | Wins when | Loses when |
|---|---|---|
| `ArrayList` | Default list, reads by index, append-heavy | Many inserts or removes at the front or middle of a big list |
| `LinkedList` | Almost never | Nearly always slower than ArrayList or ArrayDeque |
| `ArrayDeque` | Stack or queue in one thread | Needs thread safety |
| `CopyOnWriteArrayList` | Read-heavy, writes very rare (listeners, config) | Frequent writes or large lists |
| `Vector` / `synchronizedList` | Legacy code | Everything else |
| `ArrayBlockingQueue` / `LinkedBlockingQueue` | Producer-consumer with backpressure | Need non-blocking behavior |

---

## 5. TRADE-OFFS & LIMITATIONS

**What HashMap guarantees:** average O(1) get and put, provided `hashCode` is well-distributed.
**What it does NOT guarantee:** iteration order (it can change after a resize), thread safety, or that mutable keys will still be found.

**What ConcurrentHashMap guarantees:** each single operation is atomic, reads never block, and iterators never throw CME.
**What it does NOT guarantee:**
- Two separate calls together are **not** atomic. `get` then `put` is a race.
- `size()` and `isEmpty()` are estimates under concurrency.
- Iteration is not a snapshot.
- No ordering.

**Performance costs:**
- Resizing is expensive. Presize when you know the count.
- `Map<Integer, Integer>` boxes every entry. Millions of entries means a lot of memory and GC pressure.
- CHM has slightly more overhead than HashMap when there is no contention. Don't use it "just in case".
- `CopyOnWriteArrayList` write cost is O(n) plus garbage.
- `TreeMap` is slower than HashMap, but it gives you order.

**Edge cases that come up:**
- Bad `hashCode` (for example, always returns 1): everything lands in one bucket. In Java 8+ the tree saves you if keys are `Comparable`, otherwise it is still slow.
- `equals` without `hashCode`: two "equal" objects land in different buckets and you never find them.
- Changing a key after insertion: the entry is lost inside the map.

---

## 6. REAL-WORLD PRODUCTION SCENARIOS

**1. Finance analytics / credit-risk service (enterprise SaaS)**
- *System:* a scoring API where many request threads need reference data such as currency rates, industry codes, and scoring rules.
- *Problem:* loading that data on every request is slow. A plain HashMap cache corrupts under concurrent access.
- *Choice:* `ConcurrentHashMap` with `computeIfAbsent`, so one thread loads and others reuse.
- *Caveat:* `computeIfAbsent` blocks other writers to that bucket while loading. For slow loads (DB or remote call), store a `CompletableFuture` as the value, or use Caffeine, which adds expiry and size limits. A plain CHM cache never expires, so it will eventually leak memory.

**2. Spring Framework (well-known open source)**
- *System:* Spring's bean registry keeps created singleton beans in a `ConcurrentHashMap`, and many threads ask for beans at once.
- *Problem:* lookups happen constantly, but writes (creating a bean) happen once per bean.
- *Choice:* CHM gives lock-free reads and safe writes.
- *Caveat:* I'm confident about the singleton-cache usage, but I haven't checked the current Spring version's exact field layout. Don't quote class internals in an interview unless sure.

**3. LRU cache in a single service**
- *System:* cache of recently viewed customer profiles, capped at 1,000.
- *Choice:* `LinkedHashMap` with access order `true` and `removeEldestEntry` returning `size() > 1000`.
- *Caveat:* in access-order mode even `get()` **modifies** the structure, so reads need locking too. Wrap in `Collections.synchronizedMap` or use Caffeine for real production.

**4. Backpressure in a report-generation pipeline**
- *System:* web threads submit export jobs, and a small worker pool generates them.
- *Problem:* `Executors.newFixedThreadPool(n)` uses an **unbounded** `LinkedBlockingQueue`. When jobs arrive faster than workers finish, the queue grows until `OutOfMemoryError`.
- *Choice:* `ThreadPoolExecutor` with a bounded `ArrayBlockingQueue` plus a rejection policy (for example, caller-runs or return HTTP 429).
- *Caveat:* a bounded queue forces you to decide what happens when it's full. That decision is the real design work.

**5. Listener list for events (config-change, lifecycle hooks)**
- *System:* an app that notifies listeners on a config reload. Listeners are registered at startup, events fire often.
- *Choice:* `CopyOnWriteArrayList`. Iterating and notifying never needs a lock and never throws CME, even if a listener unregisters itself.
- *Caveat:* if listeners are added and removed constantly, every change copies the array. Also, a listener added during notification won't see the current event.

---

## 7. COMMON MISTAKES & FAILURE MODES

| Mistake | Symptom in production | Detect | Fix |
|---|---|---|---|
| Shared `HashMap` across threads | Lost updates, wrong totals, rare hangs (Java 7: 100% CPU in `HashMap.get`) | Thread dump (`jstack`) showing threads stuck in HashMap code; inconsistent results | `ConcurrentHashMap` or confine the map to one thread |
| Check-then-act on a concurrent map | Duplicate work, overwritten values | Code review: `containsKey` followed by `put` | `putIfAbsent`, `computeIfAbsent`, `merge` |
| Modifying a list inside for-each | `ConcurrentModificationException` | Stack trace | `removeIf()` or `Iterator.remove()` |
| Mutable object as a map key | `get()` returns null for a key you definitely put | Debugging, unit test | Use immutable keys (String, record, Long) |
| Unbounded cache (static map, no eviction) | Heap grows, eventual OOM | Heap dump in MAT, growing old-gen after GC | Caffeine or a bounded LRU with TTL |
| Unbounded queue | OOM under load spike | Queue size metric, heap dump | Bounded queue and a rejection policy |
| Slow or recursive code inside `computeIfAbsent` | Threads blocked, hangs, `IllegalStateException` | Thread dump: many threads BLOCKED on a CHM bin | Compute outside, or store futures |
| Using `size()` of CHM to decide logic | Flaky behavior under load | Code review | Track counts separately (`LongAdder`) |
| Iterating `synchronizedList` without locking | CME or inconsistent view | Intermittent failure | `synchronized (list) { ... }` around iteration, or use COWAL |
| Returning internal list from a getter | Callers modify your state | Code review | `List.copyOf()` or `Collections.unmodifiableList()` |
| `subList()` kept alive | Large parent list can't be freed | Heap dump | Copy with `new ArrayList<>(sub)` |
| Poor `hashCode` | Slow lookups, CPU spike | Profiler shows time in `get` | Use `Objects.hash(...)` on immutable fields |

---

## 8. CODE

### 8.1 Minimal correct example: thread-safe word counter
```java
import java.util.concurrent.ConcurrentHashMap;

public class WordCounter {
    private final ConcurrentHashMap<String, Integer> counts = new ConcurrentHashMap<>();

    public void add(String word) {
        counts.merge(word, 1, Integer::sum);   // atomic: read + add + write
    }
}
```

### 8.2 Buggy vs fixed: check-then-act
```java
// ❌ BUGGY: map is thread-safe, but this sequence is NOT
Integer c = counts.get(word);
if (c == null) counts.put(word, 1);
else counts.put(word, c + 1);
// Two threads both read 5, both write 6. One update is lost.

// ✅ FIXED: one atomic operation
counts.merge(word, 1, Integer::sum);
```

### 8.3 Buggy vs fixed: removing while iterating
```java
// ❌ BUGGY: throws ConcurrentModificationException
for (String s : list) {
    if (s.startsWith("tmp")) list.remove(s);
}

// ✅ FIXED (Java 8+)
list.removeIf(s -> s.startsWith("tmp"));
```

### 8.4 Buggy vs fixed: mutable key
```java
// ❌ BUGGY
class Key { int id; /* equals/hashCode use id */ }
Key k = new Key(); k.id = 1;
map.put(k, "A");
k.id = 2;                 // hashCode changed after insert
map.get(k);               // null: looks in the wrong bucket

// ✅ FIXED: immutable key
record Key(int id) {}     // Java 16+
```

### 8.5 Buggy vs fixed: slow loader in computeIfAbsent
```java
// ❌ RISKY: slow DB call while holding the bucket lock
cache.computeIfAbsent(id, k -> loadFromDb(k));

// ✅ BETTER: store a future, so the lock is held only briefly
ConcurrentHashMap<Long, CompletableFuture<Profile>> cache = new ConcurrentHashMap<>();
CompletableFuture<Profile> f = cache.computeIfAbsent(id,
        k -> CompletableFuture.supplyAsync(() -> loadFromDb(k)));
Profile p = f.join();
// Caveat: a failed future stays cached. Remove it on exception.
```

### 8.6 Tiny LRU cache
```java
class LruCache<K, V> extends LinkedHashMap<K, V> {
    private final int max;
    LruCache(int max) { super(16, 0.75f, true); this.max = max; }
    @Override protected boolean removeEldestEntry(Map.Entry<K, V> e) {
        return size() > max;
    }
}
// Not thread-safe. Wrap with Collections.synchronizedMap(...) if shared.
```

---

## 9. INTERVIEW QUESTIONS

Each answer follows **Scenario → Problem → Choice → Reason → Caveat**. Say them aloud in your own words.

### Basic (5)

**B1. ArrayList vs LinkedList?**
In a service that loads 10,000 rows and reads them by position, I need a list. The problem is that LinkedList must walk node by node to reach index i, which is O(n). So I choose ArrayList, because it is an array underneath and index access is O(1) and memory-compact. For a queue or stack I'd pick ArrayDeque, not LinkedList. The caveat is that middle inserts in ArrayList shift elements, but even then it usually wins because of CPU cache behavior.

**B2. How does HashMap work internally?**
Say I'm storing customer IDs to profiles. The problem is finding a key fast without scanning everything. HashMap computes the key's hash, spreads the bits, and uses `(n-1) & hash` as a bucket index. Inside the bucket it compares hash and `equals()` to find or append the node, and since Java 8 a long chain becomes a red-black tree. The caveat is that this only stays O(1) if `hashCode` is well spread, and the map resizes at 75% load, which costs time.

**B3. HashMap vs Hashtable vs ConcurrentHashMap?**
Imagine a cache shared by 50 request threads. HashMap isn't thread-safe, and Hashtable is safe but locks the whole map for every call. So I choose ConcurrentHashMap, which gives lock-free reads and locks only one bucket for writes. That keeps throughput high under contention. The caveat is that it rejects nulls and its `size()` is only approximate while others are writing.

**B4. What if you override `equals()` but not `hashCode()`?**
Suppose two `Customer` objects with the same ID are "equal", and I use them as map keys. The problem is that the two objects have different default hash codes, so they land in different buckets. I'd override both together using the same fields. The reason is the contract: equal objects must have equal hash codes. The caveat is that the bug is silent, since `get()` returns null with no error.

**B5. Fail-fast vs fail-safe iterators?**
Picture a thread iterating a list while another removes an item. ArrayList and HashMap iterators are fail-fast: they check `modCount` and throw CME. Concurrent collections have weakly consistent iterators that never throw CME, and CopyOnWriteArrayList iterates over a snapshot. The reason is that they are designed for concurrent change. The caveat is that fail-fast is best-effort only, so never build logic that depends on catching CME. (Also, "fail-safe" is informal, and the official term is weakly consistent.)

### Intermediate (5)

**I1. What changed in HashMap in Java 8?**
Consider a hot map where many keys collide. In Java 7 a long chain made lookup O(n), and concurrent resize could create a circular chain and hang. Java 8 converts chains of 8 or more into red-black trees (if the table has at least 64 buckets), so worst case is O(log n). It also inserts at the tail, which preserves order during resize. The caveat is that it is still not thread-safe, because you can still lose updates.

**I2. How does ConcurrentHashMap stay thread-safe? Java 7 vs 8?**
Take a map with heavy concurrent writes. A single lock makes writers queue up. Java 7 split the map into 16 segments, each with its own lock. Java 8 dropped segments: an empty bucket is filled with a CAS, and a non-empty bucket uses `synchronized` on its head node, while reads use no lock at all. This gives much finer concurrency. The caveat is that compound operations still need the atomic methods like `merge` or `computeIfAbsent`.

**I3. Why doesn't ConcurrentHashMap allow null keys or values?**
Say a thread calls `get(k)` and gets null. The problem is that it can't tell "key absent" from "value is null". In a normal HashMap you'd follow up with `containsKey`, but in a concurrent map another thread may change things between those two calls. So the designers banned nulls to keep `get` unambiguous. The caveat is that if you need "no value", use `Optional` or a sentinel object.

**I4. How do you avoid HashMap resize cost?**
Suppose I'm loading 1 million records into a map at startup. The default map starts at 16 and doubles repeatedly, copying entries each time. I presize with `expected / 0.75 + 1` (or `HashMap.newHashMap(n)` in Java 19+). That way it never resizes during the load. The caveat is that overestimating wastes memory, since capacity rounds up to the next power of 2.

**I5. How does CopyOnWriteArrayList work and when do you use it?**
Consider a listener list that is read on every event but changed only at startup. A normal list needs locking around iteration. CopyOnWriteArrayList copies the whole array on each write, so readers iterate a stable snapshot with no lock and no CME. I use it for small, read-heavy, rarely-changed lists. The caveat is that every write is O(n) with garbage, and iterators don't see later changes.

### Scenario: "When would you use...?" (5)

**S1. Shared in-memory cache for request threads.**
In a scoring API, many threads need the same reference data, and loading is slow. A HashMap would corrupt, and locking everything would serialize requests. I'd use ConcurrentHashMap with `computeIfAbsent` so one thread loads and the others reuse. The reason is lock-free reads and atomic load-once. The caveat is that a slow loader blocks that bucket, so I'd store futures or use Caffeine, and I'd add expiry so it doesn't leak.

**S2. You need an LRU cache.**
Say I need to keep the last 1,000 customer profiles. Hand-rolling eviction logic is error-prone. I'd extend LinkedHashMap with access order `true` and override `removeEldestEntry`. The reason is that it keeps entries ordered by recent use and evicts the oldest automatically. The caveat is that it isn't thread-safe and even `get()` mutates it, so I'd wrap it in a synchronized map, or use Caffeine for production.

**S3. Producers are faster than the consumer.**
In an export service, web threads submit jobs faster than workers can process. An unbounded queue hides the problem until memory runs out. I'd use a bounded `ArrayBlockingQueue` with a rejection policy. The reason is that a full queue pushes back on producers, which protects the JVM. The caveat is that I must decide what "full" means for users, whether that is waiting, failing with HTTP 429, or running on the caller thread.

**S4. Find the 10 riskiest accounts from a stream of 50 million.**
Sorting everything costs O(n log n) and holds all data in memory. I'd keep a min-heap `PriorityQueue` of size 10. For each score, if it beats the smallest in the heap, I remove the smallest and add the new one. That costs O(n log 10) with tiny memory. The caveat is that `PriorityQueue` isn't thread-safe and its iteration order isn't sorted, so I poll into a list at the end.

**S5. A live leaderboard read by many threads, with range queries.**
Suppose many threads update scores and others ask "who is ranked 100 to 200?". ConcurrentHashMap has no order, and a synchronized TreeMap has one lock. I'd use ConcurrentSkipListMap, a lock-free sorted map with O(log n) operations. The reason is that it gives sorted navigation (`headMap`, `ceilingKey`) under concurrency. The caveat is that it is slower than CHM for plain lookups and its `size()` is O(n).

### Tricky follow-ups (3)

**T1. "ConcurrentHashMap is thread-safe, so is `if (!map.containsKey(k)) map.put(k, v)` safe?"**
No. In a de-duplication job, two threads can both see the key missing, and both put, so one write is overwritten and side-effects run twice. CHM only makes each *single* call atomic, not the gap between calls. The fix is `putIfAbsent` or `computeIfAbsent`, where check and write happen as one atomic step. The caveat is that `putIfAbsent` still evaluates the value eagerly, so use `computeIfAbsent` when creating the value is expensive.

**T2. "You put a key in a HashMap, then modify the key's field. What happens?"**
Say a mutable `Account` object is a key and its `status` field is part of `hashCode`. After the change, the hash differs, so `get()` searches the wrong bucket and returns null, while the entry is still stuck in the old bucket. You've effectively leaked it. The choice is to use immutable keys such as String, Long, or a record. The reason is that the bucket position is decided at insertion time and never recomputed. The caveat is that even a `final` field holding a mutable list can cause this.

**T3. "What goes wrong with a slow or recursive `computeIfAbsent`?"**
Consider a cache where the loader calls the DB, or calls `computeIfAbsent` again on the same map, as in memoized Fibonacci. The mapping function runs while holding a lock on that bucket, so other writers to the bucket wait. In Java 8 recursion could hang forever, and in Java 9+ it may throw `IllegalStateException`. So I keep the function short and never touch the map inside it, and I store futures for slow loads. The caveat is that the exact failure mode differs by version and bucket collisions, so I treat any recursion as a bug.

---

## 10. CHEAT SHEET

**Key points**
- **ArrayList** for almost everything. **ArrayDeque** for stack or queue. LinkedList rarely.
- **HashMap:** power-of-2 table, index = `(n-1) & spreadHash`, load factor 0.75, resize doubles.
- 🚩 **Java 8:** chain ≥ 8 and table ≥ 64 → tree; tail insertion. Java 7 had the concurrent-resize infinite loop.
- **Presize** with `n / 0.75 + 1`.
- **equals and hashCode go together. Keys must be immutable.**
- **Fail-fast** (`modCount`, CME) is best effort. Concurrent iterators are **weakly consistent**.
- **CHM Java 8:** CAS for empty bucket, `synchronized` on the bucket head, lock-free reads, no nulls, `size()` approximate.
- **Check-then-act is never atomic.** Use `merge`, `putIfAbsent`, `computeIfAbsent`.
- Keep `computeIfAbsent` functions short and never recursive.
- **COWAL:** read-heavy, write-rare, snapshot iterators.
- **Queues:** bounded `ArrayBlockingQueue` for backpressure. `LinkedBlockingQueue` defaults to unbounded. `newFixedThreadPool` uses an unbounded queue.
- **TreeMap** is sorted (O(log n)). **ConcurrentSkipListMap** is its concurrent twin.
- **LinkedHashMap** with access order is an LRU cache (not thread-safe).
- Unbounded caches and queues are the most common production leak.

**30-second spoken pitch**
"Java collections give me ready-made structures, and my job is to pick the right one. By default I use ArrayList and HashMap. HashMap hashes the key to a bucket, handles collisions with a chain that becomes a tree after 8 nodes in Java 8, and resizes at 75% load, so I presize it when I know the size. These are not thread-safe, so when threads share data I use ConcurrentHashMap, which uses CAS and per-bucket locks with lock-free reads. But I remember that each call is atomic, not a sequence of calls, so I use `merge` or `computeIfAbsent`. For read-heavy lists I use CopyOnWriteArrayList, and for producer-consumer I use a bounded blocking queue. The biggest production risks are unbounded caches, unbounded queues, and mutable keys."

---

## 11. SELF-TEST

I'll ask one question at a time. Answer in your own words using **Scenario → Problem → Choice → Reason → Caveat**, and I'll critique it before moving on.

**Question 1 of 5 (scenario):**

Your team has a scoring service. A developer stores computed risk scores in a plain `HashMap<String, RiskScore>` that is shared across request threads. It passed all tests. In production you see two things: some scores are occasionally missing or wrong, and once in a while a server hits 100% CPU on one thread.

1. What is going wrong, and which Java version could explain the 100% CPU case?
2. What would you change, and what new mistake could you still make after the fix?

Take your time and answer aloud first, then type it.