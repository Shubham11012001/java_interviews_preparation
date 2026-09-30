
## 1. One-line definition + why it exists

**Garbage Collection (GC) is the JVM's automatic cleaner. It finds objects your program can no longer reach and gives their memory back.**

**What problem does it solve?** In C/C++ you call `malloc` and `free` yourself. People make three classic mistakes:
- They forget to free, so memory slowly fills up (leak).
- They free too early and then use the object (crash or corrupted data).
- They free the same thing twice (crash).

Java removes this whole category of bugs. You only say `new`. The JVM decides when an object is dead.

**What breaks without it?** Every request creates objects: DTOs, strings, lists. A busy service makes millions per minute. Without GC the heap fills up in minutes and you get `OutOfMemoryError`. Or every developer must free memory by hand, and the bugs above come back.

---

## 2. Mental model

**Analogy: a hotel with housekeeping.**
- Every object is a guest in a room.
- A guest is "still in use" if someone holds a valid booking for them (a reference from a running thread, a static field, etc.).
- Housekeeping doesn't ask each guest "are you done?". It starts from the valid bookings (**GC roots**), follows the trail, and marks every room that is reachable. All unmarked rooms are cleaned.
- Most guests leave the same day. So housekeeping checks the "day-stay floor" (**Young Gen**) very often and the "long-stay floor" (**Old Gen**) rarely. This is the **generational idea**.

**Flow in text:**

```
        GC ROOTS (start points)
  thread stacks | static fields | JNI refs | active threads
            |
            v
   follow every reference  ---> MARK reachable objects
            |
            v
   everything NOT marked = garbage
            |
            v
   reclaim: sweep / compact / copy survivors

HEAP LAYOUT (classic generational view)
+-----------------------------------------------+-------------------+
|              YOUNG GEN                        |     OLD GEN       |
|  Eden (new objects) | Survivor S0 | S1        | (long-lived)      |
+-----------------------------------------------+-------------------+
Life of an object:
 new -> Eden -> survives minor GC -> Survivor (age+1 each GC)
     -> age reaches threshold -> promoted to Old -> cleaned by major/mixed/full GC
```

---

## 3. Internals

### 3.1 Reachability, not reference counting
Java uses **reachability from GC roots**. Python uses reference counting, and there a cycle A↔B can leak. In Java, if A and B point to each other but nothing reachable points to them, both are garbage. **Cycles are not a problem in Java.** Interviewers love this.

GC roots include local variables on thread stacks, static fields, active threads, JNI references, and class loaders.

### 3.2 Three basic algorithms
| Algorithm | How | Good | Bad |
|---|---|---|---|
| Mark-Sweep | mark live, free dead | simple | leaves holes (fragmentation) |
| Mark-Compact | mark, then slide live objects together | no holes | moving is slow |
| Copying | copy live objects to a fresh empty area | fast, no holes | needs spare space |

Young Gen uses **copying** (few survivors, so it's cheap). Old Gen uses **compacting** or region-based evacuation.

### 3.3 Key mechanics you must know
- **Fast allocation:** each thread gets its own chunk of Eden (**TLAB**). Allocation is just "move a pointer forward", so it's very cheap. Creating short-lived objects in Java is cheap. Cleaning them is also cheap if they die young.
- **Generational hypothesis:** "most objects die young." That is why Young collections are frequent and fast.
- **Minor GC:** cleans Young only. Stop-the-world, but short.
- **Tenuring:** each survival adds 1 to an object's age. At the threshold (max 15) it moves to Old. Big survivors can be promoted early.
- **Card table + write barrier:** Old objects can point to Young objects. To avoid scanning all of Old during a minor GC, the JVM marks "dirty cards" whenever an Old object's reference changes. Minor GC scans only the dirty cards.
- **Stop-the-world (STW):** all app threads pause at a **safepoint** so GC can work safely. Modern collectors try to do most work concurrently and keep STW tiny.
- **Full GC:** cleans the whole heap (and metadata). It's the expensive one, and it's what you fear in production.

### 3.4 The collectors
**Serial:** one thread, STW. For tiny heaps and small containers.

**Parallel (throughput collector):** many threads, STW. Best total throughput, but pauses grow with heap size. It was the default up to Java 8.

**CMS:** mostly concurrent old-gen cleaning, but it fragments memory and eventually falls into a slow Full GC. Deprecated in Java 9, **removed in Java 14**. If an interviewer mentions CMS, answer "legacy, replaced by G1."

**G1 (Garbage-First), default since Java 9:**
- Splits the heap into many equal **regions** (1–32 MB). Each region is Eden, Survivor, Old, or Humongous at a given moment.
- You give a goal (`-XX:MaxGCPauseMillis`, default 200 ms). G1 collects the regions with the most garbage first, within that time budget.
- **Humongous objects** are objects of at least 50% of a region size. They get special contiguous regions and are a common source of trouble.
- Cycle: young-only collections → concurrent marking of Old → **mixed collections** (Young plus some Old regions) → repeat.
- If it can't evacuate fast enough (**"to-space exhausted" / evacuation failure**), it falls back to Full GC. That was single-threaded before Java 10 and parallel after. **[Version]**

**ZGC (low latency):**
- Does almost everything concurrently, including moving objects. Uses **colored pointers and load barriers**.
- Pauses are very short (typically around a millisecond or less) and don't grow with heap size. It handles multi-TB heaps.
- Costs: a bit of throughput, plus it needs spare heap room. If your allocation rate beats its cleaning speed, threads stall.
- **[Version]** Production-ready in Java 15. **Generational ZGC** arrived in Java 21 (opt-in flag `-XX:+ZGenerational`). **[Verify]** In newer JDKs (around 23/24) generational became the default and then the only mode. Check your JDK's release notes.

**Shenandoah:** similar goal to ZGC (concurrent compaction, low pauses). It is in OpenJDK builds, not always in Oracle JDK. **[Verify]** vendor support.

**Epsilon:** a "no-op" collector. It never frees memory. Used for performance tests or very short-lived jobs. Java 11+.

### 3.5 Version differences (interview favourites)
| Java | What changed |
|---|---|
| 7 | String pool moved from PermGen to heap. G1 became production-ready (7u4). |
| 8 | **PermGen removed, Metaspace added** (class metadata in native memory, grows by default). Parallel GC is default. |
| 9 | **G1 becomes default.** CMS deprecated. Unified logging `-Xlog:gc*` replaces `-XX:+PrintGCDetails`. |
| 10 | Container awareness (JVM reads Docker/K8s limits). Parallel Full GC for G1. |
| 11 | ZGC (experimental), Epsilon. |
| 14 | CMS removed. |
| 15 | ZGC and Shenandoah production-ready. |
| 18 | `finalize()` deprecated for removal. Use `Cleaner` or try-with-resources. |
| 21 (LTS) | Generational ZGC (opt-in). |

**Default collector rule:** on Java 9+, G1 is chosen if the machine has at least 2 CPUs and about 1.8 GB memory. Otherwise Serial. So a tiny container can silently get **Serial GC**. This is a common production surprise.

### 3.6 Reference types
- **Strong:** normal reference. Never collected while reachable.
- **Soft:** collected only when memory is tight. Sometimes used for memory-sensitive caches, though real cache libraries (Caffeine) are better.
- **Weak:** collected at the next GC if only weakly reachable. Used in `WeakHashMap` and listener registries.
- **Phantom:** for post-mortem cleanup actions (`Cleaner` uses this idea).

### 3.7 Advanced techniques (what "advanced GC" means in interviews)
1. **Right-size the heap.** Set `-Xms` equal to `-Xmx` to avoid resize pauses. Leave room for Metaspace, thread stacks, and direct buffers, because the total process is bigger than the heap.
2. **In containers, use percentages:** `-XX:MaxRAMPercentage=70` instead of a fixed `-Xmx`. **[Version]** This needs Java 10+ (or 8u191+).
3. **Set a pause goal, not many flags.** With G1, tune `MaxGCPauseMillis` first. Avoid copying 30 flags from blog posts.
4. **Avoid humongous allocations.** Increase `-XX:G1HeapRegionSize` or, better, stop creating giant arrays and lists. Stream or paginate instead.
5. **Stay under ~32 GB heap** if you can. Above that the JVM loses "compressed oops" (smaller pointers), and effective capacity can drop. **[Verify]** The exact limit is about 32 GB and varies slightly.
6. **Reduce allocation, not just GC.** Less garbage means less GC. Reuse builders, avoid boxing in hot loops, avoid useless copies. The JIT helps with **escape analysis**: if an object never escapes a method, it may never hit the heap at all.
7. **Go off-heap for huge, long-lived data** (direct buffers, memory-mapped files, or off-heap caches). Cassandra and Kafka rely on this style.
8. **G1 string deduplication** (`-XX:+UseStringDeduplication`) for workloads with many duplicate strings.
9. **`-XX:+AlwaysPreTouch`** pre-loads heap memory at startup. Startup is slower, but no page-fault pauses later.
10. **Measure first:** GC logs (`-Xlog:gc*`), JFR (Java Flight Recorder), `jstat`, heap dumps. Never tune blind.

---

## 4. Alternatives & comparison

### 4.1 Memory management approaches
| Approach | Where | Wins when | Loses when |
|---|---|---|---|
| Tracing GC (Java) | JVM | you want safety and fast development; handles cycles | you need hard real-time, tiny memory |
| Manual free | C/C++ | you need full control and predictable latency | bugs: leaks, use-after-free |
| Reference counting | Python, Swift (ARC) | immediate cleanup, predictable | cycles leak; counter overhead |
| Ownership (Rust) | Rust | safety without GC pauses | steeper learning curve |

### 4.2 Java collectors
| Collector | Best for | Pauses | Throughput | Watch out |
|---|---|---|---|---|
| Serial | tiny heaps, small containers | long on big heaps | OK | single thread |
| Parallel | batch / ETL jobs where pauses don't matter | longer, grows with heap | **best** | bad for latency SLAs |
| G1 | general services, medium to large heaps (default) | short, tunable goal | good | humongous objects, Full GC fallback |
| ZGC | strict latency (p99), very large heaps | sub-millisecond to few ms | slightly lower | needs spare heap, newer tech |
| Shenandoah | similar to ZGC, OpenJDK builds | very low | slightly lower | vendor availability |

**Simple rule to say aloud:** "Batch → Parallel. Normal web service → G1. Strict latency or huge heap → ZGC."

---

## 5. Trade-offs & limitations

**The triangle:** you can't maximise all three of **throughput, low latency, and low memory footprint**. Parallel picks throughput. ZGC picks latency. Serial picks footprint.

**What GC guarantees:**
- No dangling pointers. A live, reachable object is never freed.
- Memory safety. No manual free bugs.

**What GC does NOT guarantee:**
- **It does not prevent leaks.** A "logical leak" is when you keep a reference you no longer need (a static map that grows forever). GC can't know you don't need it.
- **`System.gc()` is only a hint.** The JVM may ignore it. (`-XX:+DisableExplicitGC` ignores it on purpose.)
- **No exact timing.** You can't know when an object is freed. Never rely on `finalize()`.
- **No cleanup of non-memory resources.** Files, sockets, and DB connections need `close()` or try-with-resources.
- **Pause goals are soft.** `MaxGCPauseMillis=200` is a target, not a promise.

**Costs:**
- Concurrent collectors use extra CPU threads and add barrier overhead to your code.
- G1 keeps bookkeeping structures (remembered sets) that use extra memory.
- Bigger heap means fewer GCs but potentially longer cleanup with some collectors.

**Edge cases:**
- **Promotion failure / evacuation failure:** no room to move survivors, so Full GC.
- **Allocation faster than GC can clean:** the app stalls or Full GCs.
- **`GC overhead limit exceeded`:** the JVM spends about 98% of time in GC and recovers under 2% memory. It's a sign of a leak or an undersized heap.
- **Time-to-safepoint:** GC waits for all threads to reach a safepoint. One thread in a long loop can delay everyone.

---

## 6. Real-world production scenarios

**Scenario 1: Finance analytics SaaS, month-end export (G1)**
- **System:** A credit-risk analytics API (Spring Boot). Analysts export a big portfolio report.
- **Problem:** At month-end, p99 latency jumps from 200 ms to several seconds. GC logs show humongous allocations and mixed GCs falling behind.
- **Choice:** Stay on G1, but stop loading millions of rows into one `List`. Stream or paginate from the DB, and write rows to the response in chunks. If needed, raise `G1HeapRegionSize`.
- **Why:** The root cause is the allocation pattern, not the collector. Fixing it removes the pressure.
- **Caveat:** Streaming keeps DB cursors open longer, so watch connection pool usage and transaction timeouts.

**Scenario 2: Same kind of service on Kubernetes, pod keeps restarting (container memory)**
- **System:** A Java service on GKE with a 2 GB memory limit and `-Xmx2g`.
- **Problem:** Pod is killed with exit code 137 (OOMKilled), yet heap dashboards look fine. No Java `OutOfMemoryError` appears.
- **Choice:** Use `-XX:MaxRAMPercentage=60–70` and leave room for non-heap memory.
- **Why:** The process uses heap plus Metaspace, thread stacks, direct buffers, and JIT code. Heap equal to the limit guarantees the kernel kills you first.
- **Caveat:** The right percentage depends on your thread count and native usage. Measure with JFR or Native Memory Tracking (`-XX:NativeMemoryTracking=summary`).

**Scenario 3: Latency-sensitive pricing/quote service (ZGC)**
- **System:** A service with a strict p99 of 20 ms and a 16–64 GB heap.
- **Problem:** G1 pauses of 100–300 ms cause SLA breaches.
- **Choice:** Move to ZGC (generational if your JDK supports it).
- **Why:** Pause time stays tiny regardless of heap size.
- **Caveat:** Slightly lower throughput, higher CPU, and you need headroom. If the heap is too small, ZGC stalls threads. Always load-test before switching.

**Scenario 4: Well-known open source: Cassandra / Elasticsearch**
- **System:** Distributed databases (Cassandra, Elasticsearch).
- **Problem:** A long GC pause makes a node unresponsive. Other nodes think it's dead, and the cluster rebalances or fails requests.
- **Choice:** Keep heaps modest (Elasticsearch docs advise staying below the compressed-oops threshold, about 30 GB, and giving about half of RAM to the OS file cache). Put big data structures off-heap (Cassandra does this for memtables and caches).
- **Why:** A smaller heap means shorter pauses, and OS cache handles the heavy data.
- **Caveat:** Exact recommendations change between versions, so check the current docs. **[Verify]**

**Scenario 5: Nightly batch scoring job (Parallel GC)**
- **System:** A nightly job that scores millions of records, with no user waiting.
- **Problem:** G1's concurrent overhead wastes some CPU, and the job runs a bit slower than it could.
- **Choice:** `-XX:+UseParallelGC`.
- **Why:** Only total job time matters, not individual pauses.
- **Caveat:** Never use it for an API that shares the same JVM. One long pause would hurt live users.

---

## 7. Common mistakes & failure modes

| Mistake | Symptom | How to detect | Fix |
|---|---|---|---|
| Unbounded cache / static `Map` | Old Gen keeps rising, frequent Full GCs, eventually OOM | GC log: "after Full GC" usage keeps growing; heap dump | Bounded cache with eviction (Caffeine / LRU) |
| `ThreadLocal` not removed in thread pools | Slow leak; worse after redeploys | Heap dump, look at `ThreadLocalMap` | `remove()` in `finally` |
| Fixed `-Xmx` equal to container limit | OOMKilled (exit 137) | `kubectl describe pod`, container memory vs heap | `MaxRAMPercentage`, leave non-heap room |
| `System.gc()` calls (or RMI triggering them) | Periodic full pauses | GC log cause: "System.gc()" | Remove, or `-XX:+DisableExplicitGC` |
| Loading huge lists / arrays | Humongous allocation pressure, long pauses | G1 log: "Humongous Allocation" | Stream, paginate, chunk |
| Copy-pasted old flags (CMS flags on Java 17) | Warnings or ignored options, silent fallback | Startup logs | Remove dead flags, re-tune from zero |
| Classloader leak on redeploy | `OutOfMemoryError: Metaspace` | Metaspace grows per redeploy | Fix static refs to classes, cap with `-XX:MaxMetaspaceSize` |
| Tuning before measuring | Random flags, no improvement | No baseline | Enable GC logs and JFR first |

**Standard investigation routine (say this in interviews):**
1. Turn on GC logging: `-Xlog:gc*:file=gc.log`. Check pause times, frequency, and the heap size after each GC.
2. Decide: is memory returning to a flat baseline after GC (healthy), or is the baseline climbing (leak)?
3. If climbing: take a heap dump (`jcmd <pid> GC.heap_dump` or `-XX:+HeapDumpOnOutOfMemoryError`) and open it in Eclipse MAT. Look at the dominator tree and "leak suspects."
4. If not leaking but pauses are long: check allocation rate, humongous objects, and whether the pause goal is realistic. Then consider changing the collector.

---

## 8. Code

**Minimal: reachability demo (and why `System.gc()` is just a hint)**

```java
import java.lang.ref.WeakReference;

public class GcDemo {
    public static void main(String[] args) {
        Object data = new Object();
        WeakReference<Object> ref = new WeakReference<>(data);

        data = null;      // no strong reference left
        System.gc();      // hint only, not a command

        System.out.println(ref.get()); // usually null, but NOT guaranteed
    }
}
```

**Useful startup flags (Java 17 style):**

```
java -Xms2g -Xmx2g -XX:+UseG1GC -XX:MaxGCPauseMillis=200 \
     -Xlog:gc*:file=gc.log:time,uptime \
     -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/heap.hprof \
     -jar app.jar
```

**Bug #1: unbounded cache**

```java
// BUGGY: grows forever, GC can't free it because the static map keeps a strong reference
public class ReportCache {
    private static final Map<String, byte[]> CACHE = new HashMap<>();

    public static byte[] get(String key) {
        return CACHE.computeIfAbsent(key, k -> loadReport(k)); // never evicted
    }
}
```

```java
// FIXED: bounded LRU (note: not thread-safe, so wrap it or use Caffeine in real code)
public class LruCache<K, V> extends LinkedHashMap<K, V> {
    private final int max;
    public LruCache(int max) { super(16, 0.75f, true); this.max = max; }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > max;   // drop oldest when full
    }
}
// Map<String, byte[]> cache = Collections.synchronizedMap(new LruCache<>(500));
```

**Bug #2: ThreadLocal in a thread pool**

```java
// BUGGY: pool threads live forever, so each thread keeps its big buffer forever
private static final ThreadLocal<byte[]> BUF = ThreadLocal.withInitial(() -> new byte[10_000_000]);

void handle() {
    byte[] b = BUF.get();
    // ... use b
}

// FIXED: clean up after use
void handle() {
    try {
        byte[] b = BUF.get();
        // ... use b
    } finally {
        BUF.remove();
    }
}
```

---

## 9. Interview questions with model answers

I've kept each answer short enough to say aloud in about 30 seconds.

### Basic (5)

**B1. What is garbage collection and why do we need it?**
"In a service that creates millions of objects per minute, manual memory handling would cause leaks and crashes. Java's GC finds objects that no running code can reach and frees them automatically. I'd rely on it because it removes whole bug classes like use-after-free. The caveat is that it frees memory only, not files or connections, and it can't prevent logical leaks."

**B2. How does the JVM decide an object is garbage?**
"When I debug memory, I think in terms of reachability. The JVM starts from GC roots (thread stacks, static fields, JNI refs), follows references, and marks everything reachable. Anything unmarked is garbage. This is better than reference counting because circular references don't leak. The caveat is that 'reachable' doesn't mean 'needed', so a forgotten static reference still keeps objects alive."

**B3. Explain the heap structure.**
"For a typical service, I picture the heap as Young and Old. Young has Eden and two Survivor spaces, where new objects are born and most die quickly. Survivors that live long enough are promoted to Old. I like this design because it makes the frequent cleanup cheap. The caveat is that G1, ZGC, and Shenandoah use regions, so the physical layout differs even though the idea is the same. Class metadata lives separately in Metaspace from Java 8."

**B4. Minor GC vs major vs Full GC?**
"Minor GC cleans only Young and is frequent and short. Major or mixed GC involves Old Gen. In G1, mixed collections clean Young plus some Old regions. Full GC cleans everything and pauses the app the longest. I treat frequent Full GCs as a red flag, meaning a leak or a badly sized heap. The terms are used loosely across collectors, so I'd check the GC log for the exact event names."

**B5. Does `System.gc()` force garbage collection?**
"A developer may call it hoping to free memory before a big task. It is only a hint, and the JVM may ignore it. When honoured, it often triggers an expensive Full GC pause. I'd remove it and fix the real cause, or disable it with `-XX:+DisableExplicitGC`. The caveat is that some libraries (like old RMI) call it internally, so check the GC log cause if you see mystery pauses."

### Intermediate (5)

**I1. How is G1 different from Parallel GC?**
"In a service where latency matters, Parallel's pauses grow with heap size because it cleans big areas in one stop-the-world step. G1 splits the heap into regions and cleans the ones with the most garbage first, within a pause goal. So I pick G1 for web services and Parallel for batch jobs. The reason is predictable pauses versus maximum throughput. The caveat is that the pause goal is soft, and G1 can still fall back to Full GC."

**I2. What is the generational hypothesis, and what does the card table do?**
"The hypothesis says most objects die young, so collecting Young often is cheap and efficient. The problem is that Old objects can point to Young ones, and scanning all of Old each time would be slow. So the JVM uses a write barrier to mark 'dirty cards' when an Old object changes a reference. A minor GC scans only those cards. The caveat is that the barrier adds a small cost to every reference write."

**I3. PermGen vs Metaspace?**
"On old Java 7 apps, redeploying often caused `OutOfMemoryError: PermGen`, because class metadata lived in a fixed-size heap area. Java 8 replaced it with Metaspace in native memory, which grows by default. That removes most fixed-size surprises. The caveat is that it's unbounded by default, so a classloader leak can eat all server memory. I'd set `MaxMetaspaceSize` and monitor it."

**I4. Strong, soft, weak, and phantom references: when to use each?**
"For a cache or listener registry, I'd think about how long the JVM should keep the object. Strong is the default. Weak lets it be collected at the next GC, good for `WeakHashMap` or listener lists. Soft survives until memory is tight, but it makes behavior unpredictable, so I prefer Caffeine for caches. Phantom is for cleanup after collection, and `Cleaner` builds on it. The caveat is that none of these replaces explicit `close()` for real resources."

**I5. How is a Java memory leak different from a C leak, and how do you find one?**
"In C a leak means forgetting `free`. In Java it means keeping a reference you don't need, like a growing static map. I'd confirm it from GC logs: heap after each Full GC keeps climbing. Then I'd take a heap dump and use Eclipse MAT's dominator tree to see who holds the memory. The fix is usually a bounded cache or removing a listener. The caveat is that a dump needs free disk space and briefly pauses the JVM."

### Scenario: "When would you use..." (5)

**S1. Your API has p99 spikes every few minutes. What do you do?**
"My service had latency spikes that correlated with GC. First I'd enable GC logs and check whether pauses match the spikes, and whether they are Young, mixed, or Full. If the heap after GC is flat, I'd check allocation rate and humongous allocations. If the baseline is climbing, I'd hunt a leak with a heap dump. The reason is that tuning without a cause wastes time. The caveat is that if the pause goal is truly strict, I may need ZGC."

**S2. When would you choose ZGC over G1?**
"For a pricing service with a strict p99 of a few tens of milliseconds and a big heap, G1 pauses can break the SLA. I'd move to ZGC because pauses stay tiny no matter the heap size. The reason is concurrent compaction using load barriers. The caveat is slightly lower throughput, more CPU, and the need for spare heap. I'd always load-test with production-like traffic first."

**S3. A pod is OOMKilled but heap usage looks fine. Why?**
"The container limit was 2 GB and `-Xmx` was also 2 GB. The JVM also needs Metaspace, thread stacks, direct buffers, and JIT memory, so total usage crossed the limit and the kernel killed it. I'd switch to `MaxRAMPercentage` around 60–70 and leave non-heap room. The reason is that the process is bigger than the heap. The caveat is that the right number depends on thread count and native usage, so I'd verify with Native Memory Tracking."

**S4. Nightly batch job: which collector?**
"For a scoring job that runs alone at night, nobody cares about individual pauses, only finish time. I'd use Parallel GC. The reason is that it gives the best throughput since it has less concurrent overhead. The caveat is that I'd never share that JVM with an interactive API, because its long pauses would hurt users."

**S5. Would you use object pooling to reduce GC pressure?**
"A developer might pool objects to cut garbage. In modern JVMs, short-lived allocation is very cheap, and pooling can actually hurt because pooled objects live long and get promoted to Old. I'd pool only expensive things like DB connections, threads, or large buffers. The reason is that cheap objects cost less to allocate than to manage. The caveat is that pools add complexity and reset bugs, so I'd profile first."

### Tricky follow-ups (3)

**T1. "G1's goal is 200 ms, but you see 800 ms pauses. Why?"**
"In a busy service, the goal is only a target. Common causes are to-space exhausted (no room to evacuate), too many humongous allocations, a heap that is too small, mixed collections that can't keep up, or heavy reference processing. I'd read the GC log phases to see which part is slow. Then I'd fix the cause, for example chunk big arrays or give more heap. The caveat is that if the requirement is truly strict, raising the goal or tuning flags won't help, and ZGC might."

**T2. "Will a bigger heap like 64 GB solve GC problems?"**
"Not automatically. A bigger heap means fewer collections, but with some collectors each cleanup can take longer, and a Full GC would be painful. Above about 32 GB you also lose compressed oops, which can reduce effective capacity. I'd first fix allocation patterns and leaks. The caveat is that with ZGC large heaps work well because pauses don't scale with size, but you still need to test."

**T3. "Once an object has no references, is it freed immediately? What about `finalize()`?"**
"No. It's only eligible for collection, and memory is reclaimed when a GC cycle runs. `finalize()` makes this worse because it delays cleanup and can even resurrect the object. It is deprecated for removal since Java 18. I'd use try-with-resources for deterministic cleanup, or `Cleaner` as a safety net. The caveat is that neither is a guarantee of timing for memory, only for the cleanup action."

---

## 10. Cheat sheet

**Key points to remember**
- GC frees objects **unreachable from GC roots**. Cycles are not a problem.
- Heap: Young (Eden + 2 Survivors) and Old. Metaspace replaced PermGen in **Java 8**.
- Most objects die young, so minor GCs are frequent and cheap. Full GC is the dangerous one.
- **Defaults:** Parallel up to Java 8, **G1 from Java 9**. CMS removed in Java 14.
- **Rule of thumb:** Batch → Parallel. Web service → G1. Strict latency or huge heap → ZGC.
- Generational ZGC arrived in Java 21. **[Verify]** newer defaults.
- GC doesn't prevent leaks. A static map that grows forever is still a leak.
- `System.gc()` is a hint. `finalize()` is deprecated for removal.
- In containers, use `MaxRAMPercentage`. Heap is not the whole process memory.
- G1 watch-outs: humongous objects, to-space exhausted, Full GC fallback.
- Always: **measure first** (GC logs, JFR, heap dump), tune last.
- Investigation order: GC log → is the baseline climbing? → heap dump + MAT → allocation rate → collector change.

**30-second spoken pitch**
"Garbage collection is the JVM's automatic memory cleaner. It finds objects no running code can reach, starting from GC roots, and frees them. Most objects die young, so the heap is split into Young and Old, and young cleanups are fast. Since Java 9, G1 is the default and works by region with a soft pause goal. For strict latency I'd consider ZGC, and for batch jobs Parallel. GC doesn't stop logical leaks, like a growing static map, and in Kubernetes the heap must leave room for non-heap memory. My approach to problems is: read GC logs first, check whether memory returns to a flat baseline, use a heap dump if it doesn't, and only then change the collector or flags."

---

## 11. Self-test

I'll ask five questions, one at a time. Answer the way you'd speak in an interview (4–6 sentences), and I'll critique it before moving on.

**Question 1 of 5:**
How does the JVM decide that an object is garbage? Then explain why two objects that point to each other can't cause a leak in Java, and what kind of leak *can* still happen.

Take your time and send me your answer.