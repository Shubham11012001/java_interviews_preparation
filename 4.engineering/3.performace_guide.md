# Java Performance Engineering: The Complete Interview Guide

A practical, beginner-friendly guide for Senior Software Engineers, Technical Architects, and Java Backend Engineers

Imagine you are the technical architect of a banking application. Thousands of users log in, transfer money, view statements, and receive notifications every second.

Suddenly, the application becomes slow. CPU usage reaches 95%, memory keeps increasing, database queries take longer, and users complain that transactions are timing out.

What do you do?

* Increase the server size?

* Increase JVM heap memory?

* Change the garbage collector?

* Optimize Java collections?

* Introduce Redis caching?

* Add more application instances?

A good technical architect doesn't guess. They measure, identify the bottleneck, fix the root cause, and measure again.

This guide teaches you how to do exactly that, from basic concepts to production troubleshooting and senior-level interview questions.

## What you'll learn

1. Profiling and performance monitoring

JProfiler, VisualVM, Java Flight Recorder (JFR), thread dumps, heap dumps, CPU and memory analysis.

2. JVM tuning and garbage collection

Heap sizing, Xms/Xmx, G1 GC, ZGC, memory leaks, GC pauses, OOM errors and container limits.

3. Collections and algorithms optimization

HashMap, ArrayList, LinkedList, Big-O, streams, sorting, database pagination and memory efficiency.

4. Caching techniques

Caffeine, Guava Cache, Redis, TTL, eviction, cache stampedes, invalidation and distributed caching.

5. Latency, throughput and scalability

p95/p99 latency, load testing, Little's Law, bottleneck analysis, horizontal scaling and capacity planning.

# Part 1: Understand performance before optimizing

## 1. What is performance optimization?

Performance optimization means improving how efficiently a system uses its resources while delivering the required functionality.

Think of a restaurant.

* CPU: Chefs preparing food.

* Memory: Kitchen counters holding ingredients.

* Database: Storage room containing ingredients.

* Network: Waiters carrying orders and food.

* Threads: Workers handling tasks.

* Cache: Frequently used ingredients kept nearby.

* Queue: Orders waiting to be processed.

If customers wait too long, hiring more chefs may not help if the storage room takes 30 seconds to deliver each ingredient.

Similarly, adding CPU or heap memory will not solve a slow SQL query or a downstream API bottleneck.

### The five important performance metrics

| Metric             | Simple meaning                        | Example                      |
| ------------------ | ------------------------------------- | ---------------------------- |
| Latency            | Time taken for one operation          | One API takes 200 ms         |
| Throughput         | Operations completed per unit of time | 2,000 requests/second        |
| CPU utilization    | How busy the processors are           | CPU usage is 85%             |
| Memory utilization | How much memory is occupied           | JVM uses 5 GB of heap        |
| Error rate         | Percentage of failed requests         | 2% of requests return errors |

Two additional metrics matter enormously in production:

* Saturation: How close a resource is to its limit, such as a full thread pool or exhausted database connection pool.

* Availability: Whether the service is operational and able to serve requests.

### The architect's first rule

Never optimize a metric in isolation.

For example, increasing throughput from 1,000 to 5,000 requests per second is not a success if p99 latency increases from 200 ms to 8 seconds and the service starts timing out.

A performance improvement should be evaluated against the system's service-level objectives (SLOs), correctness, cost and reliability.

## 2. How does a Java request travel through a system?

Client / API Gateway

Spring Boot application

HTTP thread → business logic → collections → object allocation

Database

Queries, locks, connections

External APIs

Network waits, timeouts

Response returned

Every layer can introduce a bottleneck.

A request might spend only 10 ms executing Java code but another 400 ms waiting for a database. Optimizing the Java code may make almost no difference to the overall response time.

Architect's habit: Break down total request time into application processing, database time, network time, queue waiting time, lock contention and garbage collection pauses.

# Part 2: Profiling and performance monitoring

## 3. What is profiling?

Profiling is the process of observing a running application to discover where it spends time, how it uses memory, and what resources limit it.

Think of a doctor checking your heartbeat, blood pressure and oxygen levels before deciding on treatment.

You shouldn't prescribe medicine simply because the patient looks tired. Similarly, you shouldn't tune the JVM simply because the application feels slow.

### Monitoring vs. profiling vs. benchmarking

| Activity        | Purpose                                         | Typical example                             |
| --------------- | ----------------------------------------------- | ------------------------------------------- |
| Monitoring      | Observe the system continuously                 | CPU, memory, latency dashboards             |
| Profiling       | Find expensive code or resource usage           | A method consuming 40% of CPU               |
| Benchmarking    | Measure performance under controlled conditions | Comparing two sorting algorithms            |
| Load testing    | Test behavior under expected traffic            | 5,000 concurrent users                      |
| Stress testing  | Discover limits and failure behavior            | Increase traffic until the service degrades |
| Troubleshooting | Diagnose a specific incident                    | Investigate a sudden latency spike          |

Monitoring tells you something is wrong. Profiling helps explain where and why. Benchmarking and load testing help you verify whether a change actually improves performance.

## 4. Tools every Java technical architect should know

![JMX 와 VisualVM 그리고 Heap dump 분석하기](https://images.openai.com/static-rsc-4/8s23AOYO6xF1Qb2tNvKuY7O8tGHRZZcS9IXrM8-kOrigjoHAi6Li_38ZrR5TMia_190v026fov1xcmlCuttamntcEW5cr-2yjiWpcguKEO7AH2kqOao3l-U8NQzqRmBi0htFCPUzG_0i_ZS6_rrIWhJttwUpnqNygn2XOVMfh0s?purpose=inline)

Add to Favorites

VisualVM

A graphical tool for inspecting Java processes, CPU activity, heap usage, threads, heap dumps and garbage collection.

Best for: Learning JVM behavior, local debugging and basic production investigations where access is permitted.

Limitation: Deep profiling can introduce overhead. Available features depend on the VisualVM version, plugins and target JVM.

![ej-technologies - What's new in JProfiler 11.1](https://images.openai.com/static-rsc-4/SdX_O8PvTdJNFl8-GsSM5JEVEfGDqju_HoVaNBKpIpTMoLYpHVrpV8NHEKL_q4mbnS650NNIT6BJ_xbVe-0WAMJ_593bWw0EwbVPBj9kXA2N2ts-RKRYIi3sjUpyPGtxM7RCtX4lTRXKbf7mtXRT9LlXPjUqO2_jGFSCGFib_Is?purpose=inline)

Add to Favorites

JProfiler

A commercial Java profiler with CPU profiling, allocation analysis, heap inspection, thread analysis and database-related profiling capabilities.

Best for: Deep investigations into expensive methods, excessive object creation, memory retention and database interactions.

Limitation: Requires licensing, and instrumentation or profiling can affect the workload.

![Profiling Java applications with Flight Recorder | by Jawad Zaarour | Medium](https://images.openai.com/static-rsc-4/smhI0-bGUYJNBsIcrsrnXnq-h_YJ4S8F_2CyQsi2J8cnPnfRJ2nQl_zu3gp-7Ky_YVmyJPgawxIAHsR82D31cge3oXgRl3enU2qFDYH8hQUe7zJwq_lNwSpRKVbsc0WRj7oDbprvB-DWJxlLBYQSFq1ueCDjwPGuUCT-LvmRAb0?purpose=inline)

Add to Favorites

Java Flight Recorder (JFR) and Java Mission Control (JMC)

JFR records JVM events such as allocations, garbage collection, thread activity and execution samples. JMC helps analyze those recordings.

Best for: Low-overhead production diagnostics and understanding intermittent performance problems.

Limitation: Sampling and recorded events are evidence, not necessarily a complete record of every method call.

![Spring Boot, Micrometer, Prometheus and Grafana - how to add custom metrics to your application | by Aleksander Kołata | Medium](https://images.openai.com/static-rsc-4/eA1kI_aTTrwHPkazA5RcnyOpMTmevVxAW9ZMLHFyr4TBxjsiA6w8RLCmMdphAWfVupH-gidobJvE-uq-wbwu9pb5HNivaoYmFi5otyhoPlz27cPeTUfrVgZoc0AX3eQis1pkfqug-9owmq97w2Dp_juPE3SoSKiCRU4Kuw16xi0?purpose=inline)

Add to Favorites

Micrometer, Prometheus and Grafana

Micrometer instruments Spring applications; Prometheus can collect and store metrics; Grafana visualizes trends and alerts.

Best for: Continuous monitoring across multiple Spring Boot instances.

Limitation: Metrics alone usually cannot tell you exactly which line of code is slow.

Other useful tools:

* `jcmd` — diagnostic commands for a running JVM, including JFR and GC-related information.

* `jstack` — thread dumps for diagnosing blocked or waiting threads.

* `jmap` — heap-related diagnostics; usage and availability depend on the JDK.

* async-profiler — sampling profiler useful for CPU, allocation and other supported profiling modes.

* APM platforms — distributed tracing, request breakdowns, database timings and cross-service visibility.

For a modern Java service, a strong combination is continuous metrics + distributed tracing + JFR recordings + heap/thread dumps when needed.

## 5. CPU profiling: Find expensive methods

Imagine this code runs on every request:

Java

```
public List<Order> getOrders(List<Order> orders) {
    List<Order> result = new ArrayList<>();

    for (Order order : orders) {
        if (isEligible(order)) {
            result.add(order);
        }
    }

    return result;
}
```

Suppose `isEligible()` contains an expensive regular expression or repeatedly queries a database.

A CPU profile might show:

```
getOrders()           100% inclusive time
  isEligible()         70% inclusive time
    validatePattern()  55% self time
```

Two important terms:

* Self time: Time spent directly in a method, excluding its child methods.

* Inclusive time: Time spent in a method plus its child methods.

If `getOrders()` spends most of its time in `isEligible()`, optimizing the outer loop alone may not help much.

### Sampling vs. instrumentation

Sampling profiling

* Periodically samples what threads are executing.

* Usually has lower overhead.

* Good for finding hotspots in real workloads.

* May miss very short-lived operations or rare events.

Instrumentation profiling

* Adds measurement hooks to methods or code paths.

* Can provide detailed call counts and timings.

* May have greater overhead and can change application behavior.

An architect chooses the least intrusive tool that can answer the question.

## 6. Memory profiling: Find excessive allocation and memory leaks

Consider:

Java

```
public List<CustomerDto> getCustomers(List<Customer> customers) {
    return customers.stream()
            .map(customer -> new CustomerDto(
                    customer.getId(),
                    customer.getName()))
            .toList();
}
```

If this executes for 100,000 customers per request, the application creates many DTO objects. Depending on request volume and object lifetime, this may increase garbage collection work.

Memory profiling answers two different questions:

1. Allocation problem: Which code creates too many objects?

2. Retention problem: Which objects remain reachable longer than necessary?

These are not the same problem.

### What is a memory leak in Java?

Java has garbage collection, but memory leaks can still occur.

A memory leak happens when objects that are no longer useful remain reachable through references, preventing garbage collection from reclaiming them.

Example:

Java

```
private static final List<byte[]> CACHE = new ArrayList<>();

public void process() {
    CACHE.add(new byte[1024 * 1024]);
}
```

Each call retains another roughly 1 MB array. If entries are never removed, the list grows until the application becomes unhealthy or runs out of heap.

Common causes:

* Unbounded static collections.

* Caches without size or expiration limits.

* Listeners and callbacks that are never removed.

* `ThreadLocal` values retained by long-lived pooled threads.

* Unbounded queues.

* Objects retained by sessions or large object graphs.

### How do you confirm a leak?

1. Observe heap usage over time.

2. Trigger or wait for representative garbage collection cycles.

3. Check whether post-GC heap usage continues to rise under a stable workload.

4. Capture heap dumps at appropriate times.

5. Compare object counts and retained heap across dumps.

6. Inspect GC roots and reference paths retaining suspicious objects.

7. Fix the retention source and repeat the same workload.

A rising heap graph alone does not prove a leak. A healthy application can retain a large working set, and the JVM can legitimately use available heap space.

## 7. Thread dumps: Why is the application hanging?

A thread dump shows the states and stack traces of JVM threads at a particular moment.

Useful states include:

| State           | Meaning                                                                 |
| --------------- | ----------------------------------------------------------------------- |
| `RUNNABLE`      | Executing or ready to execute; may also be in certain native operations |
| `BLOCKED`       | Waiting to acquire a Java monitor lock                                  |
| `WAITING`       | Waiting indefinitely for another action                                 |
| `TIMED_WAITING` | Waiting for a specified period                                          |

Example command:

Bash

```
jcmd <pid> Thread.print
```

Or, where available:

Bash

```
jstack <pid>
```

### Production scenario: API latency suddenly increases

Suppose 150 request threads are waiting on a slow external service.

You inspect thread dumps and find many threads blocked in socket reads or waiting on futures.

The problem may not be CPU or garbage collection at all. It may be an overloaded downstream service combined with too many requests waiting for responses.

What you investigate next:

* External API latency and timeout settings.

* Thread pool utilization and queue length.

* Connection pool exhaustion.

* Retry volume and retry storms.

* Whether circuit breakers and bulkheads are needed.

* Whether asynchronous execution or virtual threads would help.

Remember: virtual threads can make blocking I/O easier to scale, but they do not create unlimited downstream capacity.

## 8. A practical JFR workflow

On supported JDKs, you can record JVM behavior using JFR.

Bash

```
jcmd <pid> JFR.start \
  name=performance \
  settings=profile \
  duration=120s \
  filename=/tmp/performance.jfr
```

This example requests a two-minute recording. Check the options supported by your particular JDK and account for disk space and operational policies.

Then open the recording in Java Mission Control and examine:

* Hot methods and execution samples.

* Allocation hotspots.

* Garbage collection events.

* Thread parking, contention and blocked time.

* Socket and file I/O events where recorded.

* Long pauses and correlations with latency spikes.

For production, use an approved diagnostic process, especially when heap dumps or recordings may contain sensitive application data.

# Part 3: JVM tuning and garbage collection

## 9. What is the JVM heap?

The JVM heap is the area of memory where Java objects are generally allocated.

Java

```
Customer customer = new Customer();
```

The `Customer` object normally lives on the heap, while the local variable `customer` holds a reference to it. The reference itself may live on a thread's stack when it is a local variable, but actual object placement is ultimately an implementation detail of the JVM.

Think of the heap as a warehouse:

* New objects arrive.

* Objects still needed remain in the warehouse.

* Objects no longer reachable become eligible for cleanup.

* The garbage collector (GC) reclaims memory occupied by eligible objects.

A JVM also uses memory outside the Java heap, including thread stacks, metaspace, JIT code cache and native allocations.

Important interview point: Increasing heap size does not fix every memory problem. Your process can run out of native memory even when Java heap usage is relatively low.

## 10. JVM memory areas you should understand

| Memory area          | Stores                                          | Typical problem                                  |
| -------------------- | ----------------------------------------------- | ------------------------------------------------ |
| Heap                 | Most Java objects and arrays                    | `OutOfMemoryError: Java heap space`              |
| Stack                | Method frames, local variables, execution state | `StackOverflowError`                             |
| Metaspace            | Class metadata                                  | `OutOfMemoryError: Metaspace`                    |
| Code cache           | JIT-compiled native code                        | Code-cache exhaustion or reduced compilation     |
| Direct/native memory | Off-heap buffers and native allocations         | Native memory exhaustion                         |
| Thread stacks        | Per-thread stack memory                         | Excessive thread count or native memory pressure |

Each Java platform thread typically needs stack memory. If you create thousands of platform threads, their combined stack reservations and other thread resources can become significant.

### Heap vs. stack — simple example

Java

```
public void process() {
    int count = 10;
    Customer customer = new Customer();
}
```

Conceptually:

* `count` is a local primitive variable.

* `customer` is a local reference.

* The `Customer` object is generally allocated on the heap.

The JVM may optimize allocations through escape analysis and other techniques, so this is a useful mental model, not a guarantee of physical placement in every optimized execution.

## 11. Understanding `-Xms` and `-Xmx`

These are two of the most frequently asked JVM tuning options.

Bash

```
java -Xms512m -Xmx2g -jar application.jar
```

* `-Xms512m`: Sets the initial/minimum configured heap size.

* `-Xmx2g`: Sets the maximum heap size.

Imagine the application has a warehouse that can grow from 512 MB to 2 GB.

If the application needs more heap than the maximum permits, and garbage collection cannot reclaim enough space, it may throw `OutOfMemoryError`.

### Should you always set `-Xms` equal to `-Xmx`?

No. It depends on the workload and environment.

| Configuration           | Potential advantage                               | Potential drawback                             |
| ----------------------- | ------------------------------------------------- | ---------------------------------------------- |
| Smaller Xms, larger Xmx | Can allow a smaller initial heap footprint        | Heap growth may introduce overhead             |
| Xms equal to Xmx        | More predictable heap sizing in some environments | Reserves/configures a larger heap from startup |
| Very large Xmx          | Can accommodate a large live object set           | May increase resource pressure and GC costs    |

Modern JVMs and garbage collectors have their own ergonomics. The exact effect of these flags depends on the JVM version, collector and runtime environment.

### How should an architect choose heap size?

1. Measure the live heap after garbage collection under representative traffic.

2. Measure allocation rate and peak heap occupancy.

3. Estimate headroom for bursts, traffic growth and temporary allocations.

4. Leave enough memory for native allocations, thread stacks and the operating system.

5. Test with the actual container memory limit.

6. Validate latency, throughput, GC behavior and OOM risk under load.

For example, if a container has a 4 GB memory limit, setting `-Xmx4g` is risky because the JVM process needs memory beyond the Java heap.

The right heap size is the one that meets performance and reliability targets without exceeding the total memory budget.

## 12. What is garbage collection?

Garbage collection is the JVM's process of reclaiming memory from objects that are no longer reachable.

Java

```
public void processRequest() {
    Customer customer = new Customer();
    customer.setName("Amit");
}
```

Once the method finishes, the `customer` reference may no longer be reachable. If nothing else references that object, it becomes eligible for garbage collection.

It is eligible, not guaranteed to be collected immediately.

You generally should not call `System.gc()` to solve routine production memory problems. It is only a request/hint to the JVM, and its effects depend on JVM configuration.

### Why do we need GC?

Without automatic memory reclamation, developers would have to manually manage the lifetime of heap objects. This is error-prone and can lead to use-after-free and double-free bugs.

GC reduces that burden, but it consumes CPU and can affect application latency.

## 13. Young generation, old generation and object lifetime

Traditional generational GC designs take advantage of a common observation: many objects die young.

Imagine a shopping application processing an order.

* Temporary validation objects are created and quickly discarded.

* Request DTOs may live only for milliseconds.

* Shared configuration and caches may remain alive for hours.

The JVM's generational collectors optimize around these different lifetimes.

In traditional generational collectors:

* Young generation: New objects are typically allocated here.

* Minor/young GC: Reclaims dead young objects and may promote survivors.

* Old generation: Holds objects that survive longer.

* Major/full GC: These terms vary by collector and JVM; they are not universally interchangeable.

Not all collectors use exactly the same memory layout or collection model. For example, ZGC has different internal mechanics from a traditional generational layout, and modern ZGC versions support generational collection.

### What happens when an object survives GC?

It may remain in the young generation or be promoted to an older region/generation, depending on the collector, age thresholds, survivor space and memory pressure.

A long-lived object is not automatically a leak. The key question is whether it is still needed.

## 14. Important Java garbage collectors

The right collector depends on latency targets, heap size, allocation patterns, CPU budget, JDK version and operational requirements.

| Collector   | Main strength                               | Trade-off / suitable use                                        |
| ----------- | ------------------------------------------- | --------------------------------------------------------------- |
| Serial GC   | Simple, low overhead for small workloads    | Stop-the-world collection can hurt larger workloads             |
| Parallel GC | High throughput using parallel GC work      | Can have longer pauses                                          |
| G1 GC       | Balanced throughput and pause-time goals    | Pause targets are goals, not guarantees                         |
| ZGC         | Very low pause times, including large heaps | Requires evaluation of CPU/memory overhead and workload fit     |
| Shenandoah  | Low-pause collection                        | Availability and support depend on JVM distribution and version |

### G1 GC

G1 is a common choice for server applications and is the default collector in many modern HotSpot configurations.

It divides the heap into regions and prioritizes collection work based on factors such as reclaimable space and estimated effort.

Example:

Bash

```
java -Xms2g -Xmx2g \
  -XX:+UseG1GC \
  -jar application.jar
```

You can set a pause-time goal:

Bash

```
-XX:MaxGCPauseMillis=200
```

This is a goal used by the collector, not a promise that every pause will remain below 200 ms.

Good fit: General-purpose backend services that need a balance of throughput and latency.

### ZGC

ZGC is designed to keep garbage collection pauses very short, including with large heaps.

Bash

```
java -Xms4g -Xmx4g \
  -XX:+UseZGC \
  -jar application.jar
```

Check your JDK version and distribution for the supported options. ZGC's generational mode and defaults have evolved across JDK releases.

Good fit: Services where GC pause latency is a major concern and testing shows ZGC meets the workload's needs.

Trade-off: Low pauses do not mean zero GC cost. Allocation rate, CPU consumption, memory overhead and application-level bottlenecks still matter.

### Parallel GC

Parallel GC emphasizes throughput by performing GC work in parallel.

Good fit: Batch processing, ETL or background workloads where total completion time matters more than very low request latency.

Trade-off: Stop-the-world pauses may be less suitable for latency-sensitive APIs.

### How do you choose a GC in an interview?

Start with requirements, not a favorite collector.

* For general-purpose APIs: evaluate G1 first.

* For strict tail-latency goals: benchmark ZGC or another suitable low-pause collector.

* For throughput-oriented batch jobs: evaluate Parallel GC.

* For small heaps or constrained workloads: consider simpler collectors where appropriate.

Then compare the same representative workload using throughput, p95/p99 latency, pause duration, CPU usage, memory consumption and failure behavior.

## 15. How do you troubleshoot GC problems?

Imagine your API is usually fast but occasionally takes two seconds to respond.

You see spikes in GC activity at exactly those times.

Investigate:

1. GC frequency: Are collections occurring more frequently than expected?

2. Pause duration: How much application time is lost to stop-the-world pauses?

3. Allocation rate: Is the application creating excessive temporary objects?

4. Live set: How much memory remains occupied after collection?

5. Promotion pressure: Are many objects surviving young collections?

6. Full GC: Why is it happening, and is it reclaiming enough memory?

7. CPU contention: Is GC competing with application threads?

8. Latency correlation: Do the pauses actually line up with user-facing latency spikes?

Enable GC logs where supported:

Bash

```
-Xlog:gc*:file=/var/log/app-gc.log:time,uptime,level,tags
```

This unified logging syntax is used by modern HotSpot JDKs. Logging options differ in older Java releases.

### How to interpret heap graphs

Pattern A: Healthy allocation and collection

![](data\:image/svg+xml;charset=utf-8,%3Csvg%20font-family%3D%22-apple-system-body%2C%20ui-sans-serif%2C%20-apple-system%2C%20system-ui%2C%20%26quot%3BSegoe%20UI%26quot%3B%2C%20Helvetica%2C%20%26quot%3BApple%20Color%20Emoji%26quot%3B%2C%20Arial%2C%20sans-serif%2C%20%26quot%3BSegoe%20UI%20Emoji%26quot%3B%2C%20%26quot%3BSegoe%20UI%20Symbol%26quot%3B%22%20font-weight%3D%22400%22%20data-d-component%3D%22svg%22%20fill%3D%22currentColor%22%20style%3D%22color%3Argb\(255%2C%20255%2C%20255\)%22%20viewBox%3D%220%200%20320%20100%22%20width%3D%22100%25%22%20xmlns%3D%22http%3A%2F%2Fwww.w3.org%2F2000%2Fsvg%22%3E%3Cpath%20d%3D%22M24%2010%20V80%20H305%22%20fill%3D%22none%22%20stroke%3D%22currentColor%22%20stroke-opacity%3D%22.5%22%20stroke-width%3D%221%22%2F%3E%3Cpath%20d%3D%22M25%2072%20L45%2036%20L65%2056%20L85%2024%20L105%2050%20L125%2020%20L145%2049%20L165%2023%20L185%2048%20L205%2019%20L225%2050%20L245%2021%20L265%2045%20L285%2020%22%20fill%3D%22none%22%20stroke%3D%22%2316a34a%22%20stroke-width%3D%222.5%22%20stroke-linejoin%3D%22round%22%2F%3E%3Ctext%20x%3D%2228%22%20y%3D%2296%22%20fill%3D%22currentColor%22%20font-size%3D%2210%22%3ETime%20%E2%86%92%3C%2Ftext%3E%3Ctext%20x%3D%2232%22%20y%3D%2213%22%20fill%3D%22currentColor%22%20font-size%3D%229%22%3EHeap%20used%3C%2Ftext%3E%3C%2Fsvg%3E)

Heap usage rises and falls after collections, while the post-GC baseline remains broadly stable.

Pattern B: Possible memory leak

![](data\:image/svg+xml;charset=utf-8,%3Csvg%20font-family%3D%22-apple-system-body%2C%20ui-sans-serif%2C%20-apple-system%2C%20system-ui%2C%20%26quot%3BSegoe%20UI%26quot%3B%2C%20Helvetica%2C%20%26quot%3BApple%20Color%20Emoji%26quot%3B%2C%20Arial%2C%20sans-serif%2C%20%26quot%3BSegoe%20UI%20Emoji%26quot%3B%2C%20%26quot%3BSegoe%20UI%20Symbol%26quot%3B%22%20font-weight%3D%22400%22%20data-d-component%3D%22svg%22%20fill%3D%22currentColor%22%20style%3D%22color%3Argb\(255%2C%20255%2C%20255\)%22%20viewBox%3D%220%200%20320%20100%22%20width%3D%22100%25%22%20xmlns%3D%22http%3A%2F%2Fwww.w3.org%2F2000%2Fsvg%22%3E%3Cpath%20d%3D%22M24%2010%20V80%20H305%22%20fill%3D%22none%22%20stroke%3D%22currentColor%22%20stroke-opacity%3D%22.5%22%20stroke-width%3D%221%22%2F%3E%3Cpath%20d%3D%22M25%2072%20L45%2062%20L65%2068%20L85%2050%20L105%2058%20L125%2040%20L145%2050%20L165%2032%20L185%2043%20L205%2025%20L225%2035%20L245%2017%20L265%2025%20L285%2012%22%20fill%3D%22none%22%20stroke%3D%22%23dc2626%22%20stroke-width%3D%222.5%22%20stroke-linejoin%3D%22round%22%2F%3E%3Ctext%20x%3D%2228%22%20y%3D%2296%22%20fill%3D%22currentColor%22%20font-size%3D%2210%22%3ETime%20%E2%86%92%3C%2Ftext%3E%3Ctext%20x%3D%2232%22%20y%3D%2213%22%20fill%3D%22currentColor%22%20font-size%3D%229%22%3EHeap%20used%3C%2Ftext%3E%3C%2Fsvg%3E)

The baseline appears to rise. Confirm under a comparable workload and inspect retained objects before concluding there is a leak.

These sketches illustrate patterns, not actual measurements.

## 16. Common JVM errors and what they mean

| Error                        | Likely meaning                                                           | First investigations                                  |
| ---------------------------- | ------------------------------------------------------------------------ | ----------------------------------------------------- |
| `Java heap space`            | Heap cannot satisfy an allocation                                        | Heap dump, live set, allocation rate, Xmx             |
| `GC overhead limit exceeded` | Too much time in GC with little progress under applicable JVM conditions | Retained objects, allocation pressure, GC logs        |
| `Metaspace` OOM              | Class metadata memory exhausted                                          | Class loading, classloader leaks, metaspace usage     |
| `StackOverflowError`         | Excessive call-stack depth, often recursion                              | Recursion, cyclic calls, stack traces                 |
| `Direct buffer memory`       | Direct buffer allocation cannot be satisfied                             | Native/direct memory usage, buffer lifecycle          |
| Container OOM kill           | Process exceeded container or system memory constraints                  | RSS, heap, native memory, thread count, cgroup limits |

An OOM error does not always mean “increase Xmx.” First determine which memory area or resource is exhausted.

## 17. Production scenario: High CPU, low throughput

Your Spring Boot service has:

* CPU usage: 95%.

* API latency: p99 = 3 seconds.

* Throughput: lower than expected.

* GC pauses: modest.

* Database latency: normal.

A reasonable investigation:

1. Check CPU profiles for hot methods.

2. Look for expensive loops, repeated serialization, regex processing, cryptography or compression.

3. Check lock contention and thread scheduling.

4. Examine algorithmic complexity and unnecessary repeated work.

5. Optimize the confirmed hotspot.

6. Load-test the change and compare the same metrics.

If the CPU profile shows an expensive nested loop, replacing it with an indexed lookup might produce a much larger improvement than changing GC settings.

# Part 4: Optimizing collections and algorithms

## 18. Why do collections matter?

Collections store and organize data. Choosing the wrong collection can turn a fast application into a slow one.

Imagine you have one million customer IDs and need to check whether each ID exists.

Using a list and searching from the beginning each time may be expensive. A hash-based set is generally better for membership checks when its hashing and equality behavior are appropriate.

Architect's rule: Understand the data size, access pattern, ordering requirements, concurrency requirements and memory budget before selecting a collection.

## 19. Big-O notation explained simply

Big-O describes how an algorithm's work tends to grow as input size increases.

Suppose nnn is the number of records.

| Complexity                   | Name         | Example                                    |
| ---------------------------- | ------------ | ------------------------------------------ |
| O(1)O(1)O(1)                 | Constant     | Typical hash lookup                        |
| O(log⁡n)O(\log n)O(logn)     | Logarithmic  | Binary search on sorted random-access data |
| O(n)O(n)O(n)                 | Linear       | One pass through a list                    |
| O(nlog⁡n)O(n \log n)O(nlogn) | Linearithmic | Typical comparison-based sorting           |
| O(n2)O(n^2)O(n2)             | Quadratic    | Comparing every pair in a nested loop      |
| O(2n)O(2^n)O(2n)             | Exponential  | Some brute-force combinatorial searches    |

Big-O usually describes asymptotic growth, not exact execution time. Constant factors, allocation, cache locality, data distribution and I/O can dominate real workloads.

### Example: Find duplicates

A slow approach:

Java

```
for (int i = 0; i < ids.size(); i++) {
    for (int j = i + 1; j < ids.size(); j++) {
        if (ids.get(i).equals(ids.get(j))) {
            // Duplicate found
        }
    }
}
```

The number of comparisons can grow quadratically.

A hash-based approach:

Java

```
Set<String> seen = new HashSet<>();
Set<String> duplicates = new HashSet<>();

for (String id : ids) {
    if (!seen.add(id)) {
        duplicates.add(id);
    }
}
```

Expected time complexity is O(n)O(n)O(n) under typical hash-distribution assumptions, with additional memory proportional to the number of distinct IDs.

When is the second approach best? When the dataset fits in memory and fast membership checks matter.

When might it be unsuitable? When the dataset is too large for memory, exact ordering is needed, or a database can perform the operation more efficiently.

## 20. Choosing the right Java collection

| Collection      | Typical lookup                                                          | Best use                                        | Main caution                                 |
| --------------- | ----------------------------------------------------------------------- | ----------------------------------------------- | -------------------------------------------- |
| `ArrayList`     | Index access: O(1)O(1)O(1); search: O(n)O(n)O(n)                        | Ordered data, iteration, append-heavy workloads | Middle insertions/removals shift elements    |
| `LinkedList`    | Search: O(n)O(n)O(n)                                                    | Specialized deque/list use cases                | High object overhead and poor cache locality |
| `HashMap`       | Expected O(1)O(1)O(1)                                                   | Key-value lookup                                | No guaranteed iteration order                |
| `LinkedHashMap` | Expected O(1)O(1)O(1)                                                   | Lookup with predictable iteration order         | Additional linked-entry overhead             |
| `TreeMap`       | O(log⁡n)O(\log n)O(logn)                                                | Sorted keys and range queries                   | More overhead than typical hash lookup       |
| `HashSet`       | Expected O(1)O(1)O(1)                                                   | Uniqueness and membership checks                | Requires correct `equals()` and `hashCode()` |
| `TreeSet`       | O(log⁡n)O(\log n)O(logn)                                                | Sorted unique elements                          | Slower membership than typical hash sets     |
| `ArrayDeque`    | Amortized O(1)O(1)O(1) at ends                                          | Queue and stack operations                      | Does not allow `null` elements               |
| `PriorityQueue` | Peek: O(1)O(1)O(1); insertion/removal of head: O(log⁡n)O(\log n)O(logn) | Scheduling and top-priority tasks               | Full iteration is not sorted                 |

These are typical complexity characteristics, not hard timing guarantees for every workload or JVM implementation.

### ArrayList vs. LinkedList

A common interview trap is assuming `LinkedList` is always better when inserting elements.

Consider:

Java

```
List<Integer> list = new ArrayList<>();
```

`ArrayList` stores references in a contiguous backing array. This generally gives good locality and fast indexed access.

`LinkedList` stores separate linked nodes. It can insert or remove a node in constant time once you already have the position, but finding that position may take O(n)O(n)O(n), and node allocation and pointer chasing have costs.

For most ordinary application lists, `ArrayList` is the better default.

Use `LinkedList` only when its specific behavior has a demonstrated advantage over alternatives such as `ArrayDeque`.

## 21. HashMap internals — an architect's view

A `HashMap` stores entries in a hash table.

Simplified process:

1. Compute the key's hash.

2. Use the hash to identify a bucket.

3. Search the bucket for a matching key.

4. Use `equals()` to distinguish keys that map to the same bucket.

5. Insert or update the entry.

If multiple keys land in the same bucket, collisions occur. Modern OpenJDK implementations can treeify sufficiently large collision-heavy buckets under specific capacity and threshold conditions, improving certain worst-case searches.

### Why are `equals()` and `hashCode()` important?

The contract requires equal objects to have equal hash codes.

Java

```
class Employee {
    private final int id;

    Employee(int id) {
        this.id = id;
    }

    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (!(obj instanceof Employee other)) return false;
        return id == other.id;
    }

    @Override
    public int hashCode() {
        return Integer.hashCode(id);
    }
}
```

Here, two employees with the same ID are treated as equal keys.

Important: Do not mutate fields used by `equals()` or `hashCode()` while an object is being used as a map key. Doing so can make the entry difficult to find through ordinary lookups.

### Why does HashMap capacity matter?

When a hash table becomes too full, it may resize and redistribute entries. Resizing costs time and can create allocation pressure.

If you know you need to store a large number of entries, choosing a suitable initial capacity can reduce repeated resizing.

Avoid blindly assigning an enormous capacity, though: over-allocation wastes memory.

## 22. Optimizing loops and streams

Consider:

Java

```
List<String> names = employees.stream()
        .filter(Employee::isActive)
        .map(Employee::getName)
        .toList();
```

This is clear and often perfectly adequate.

A common mistake is assuming a traditional loop must always be faster than a stream, or that parallel streams must always be faster than sequential streams.

The actual result depends on:

* Input size.

* Work per element.

* Allocation behavior.

* Data source and splitting characteristics.

* Parallel execution overhead.

* Shared-state contention.

* CPU availability.

### When should you optimize a stream?

Optimize when profiling or benchmarking shows that it matters.

For example, if a stream performs repeated expensive transformations, precomputing reusable results may help more than replacing the stream syntax.

### Avoid repeated work inside loops

Bad:

Java

```
for (Order order : orders) {
    if (configuration.getRules().contains(order.getType())) {
        // Process order
    }
}
```

If `getRules()` constructs a new list each time, unnecessary work may occur on every iteration.

Better, if the rules are stable for the operation:

Java

```
Set<String> rules = configuration.getRules();

for (Order order : orders) {
    if (rules.contains(order.getType())) {
        // Process order
    }
}
```

If the rules list is large and membership checks are frequent, a `HashSet` may be more appropriate. If the collection is already a set, avoid unnecessary conversion.

## 23. Database optimization is part of application performance

A Java algorithm can be fast while the whole API remains slow because the application makes too many database calls.

Consider the N+1 query problem:

Java

```
List<Order> orders = orderRepository.findAll();

for (Order order : orders) {
    Customer customer = customerRepository
            .findById(order.getCustomerId())
            .orElseThrow();
}
```

For 1,000 orders, this can mean one query for orders plus up to 1,000 customer queries.

Potential improvements:

* Fetch the required data with an appropriate join or batch query.

* Use projections to return only required columns.

* Add indexes aligned with query predicates and sorting.

* Avoid loading huge result sets when only a page is needed.

* Use JDBC or ORM batching for bulk operations where appropriate.

* Inspect query plans and database wait events before guessing.

### Offset vs. keyset pagination

Offset pagination:

SQL

```
SELECT id, created_at
FROM orders
ORDER BY created_at, id
LIMIT 100 OFFSET 100000;
```

Deep offsets can become expensive because the database may still need to traverse many rows before returning the requested page.

Keyset pagination:

SQL

```
SELECT id, created_at
FROM orders
WHERE (created_at, id) > (:lastCreatedAt, :lastId)
ORDER BY created_at, id
LIMIT 100;
```

With an appropriate index and matching query plan, keyset pagination can be more efficient for deep sequential traversal.

It also needs a stable, deterministic ordering and careful handling of changing data. It is not a drop-in replacement for every arbitrary page-jump use case.

## 24. Common collection optimization mistakes

* Using `List.contains()` repeatedly for large datasets when a set is appropriate.

* Using `LinkedList` for ordinary indexed access.

* Keeping unbounded collections in memory.

* Returning thousands of entities when a small projection is sufficient.

* Using parallel streams on a small dataset without measuring.

* Using `ConcurrentHashMap` as if it makes every multi-step operation atomic.

* Ignoring object allocation and memory footprint.

* Sorting repeatedly when the data has not changed.

* Performing remote calls inside loops.

* Optimizing algorithmic complexity while ignoring database and network costs.

# Part 5: Caching techniques

## 25. What is caching?

Caching means storing the result of expensive work so that a later request can reuse it instead of repeating the work.

Imagine a library.

If a student asks for the same popular book every minute, the librarian keeps it on the front desk instead of walking to the basement each time.

The front desk is the cache.

In a Java backend, you might cache:

* Product details.

* User preferences.

* Reference data.

* Feature flags.

* Frequently accessed configuration.

* Expensive, reusable computations.

* Session-related data where the design and security requirements allow it.

Caching can reduce latency, database load and network traffic. But it also introduces freshness, consistency, memory and invalidation challenges.

## 26. Cache types and when to use them

![Distributed Cache 도입기 (3): RDB 조회 90% 감소시키기](https://images.openai.com/static-rsc-4/Njfo3HKnv1-IfA8jtauiVHAPgAbJHgQD0VNy01IttG8dAwSDE3nfZlAZRFpGX1j9a9bngBcbWgFX3SZyvqb4PTyW1uI2aCipvRaXvHFWZ8ROyq1Y3357IMHuXtp_CjoXkaxXBviZwEv8-JACxqURJFquYxQFMQvN9eM8FnkLLEU?purpose=inline)

Add to Favorites

Local in-memory cache

Examples: Caffeine and Guava Cache.

Very fast because data is stored in the application process. Useful for frequently accessed data that can be independently cached by each application instance.

Trade-off: Each instance has its own cache, so values may differ across instances.

![In-Memory Cache vs Distributed Cache — Which One Powers Your Food delivery App Better? | by DotNet Full Stack Dev | Medium](https://images.openai.com/static-rsc-4/mL35yIqndxl8NMpuFXdTf36C1_ORRfxnJPeD_76QMDFk7UpnqulJyE-YUZIZGuwjrL4HsmP9_D2l5L9Vg9mfMMe06yewqahVhUoIEsWVn8u5yoYlzAPUo2vIgiJrtrCI-OvFFke2rqxaFzIxDlLwWiqVTJuZRfzNAzqOkx35Q1w?purpose=inline)

Add to Favorites

Distributed cache

Example: Redis.

Multiple application instances can use the same cache service. Useful for shared data, cross-instance reuse and centralized cache management.

Trade-off: Adds network latency, infrastructure, serialization and operational failure modes.

![Behind Cache-Control: Unveiling the Secrets of Advanced Browser Caching Strategies | by Prayugo Dwi Setiyoko | Medium](https://images.openai.com/static-rsc-4/0qSQF2JbRfj7WMb_IOKP9n0-bnnmA3X6nhbxw5ylu4x35S7wny4Zx0lOJnEFp8SjghZc3c75BakNFPLvat-JrIMe4icDEM6F9mZspO4G3TEykl46CAiT35GBab-dI31v5GFhwMq11kSbW6JIuK9LAdwI1ySx4IStHrG7kt6LrHU?purpose=inline)

Add to Favorites

HTTP/CDN cache

Stores eligible responses near clients or at the edge.

Useful for public assets, images and appropriately cacheable API responses.

Trade-off: Cache keys, authentication, invalidation and HTTP caching directives must be correct.

## 27. Caffeine vs. Guava Cache vs. Redis

| Feature                 | Caffeine          | Guava Cache                             | Redis                              |
| ----------------------- | ----------------- | --------------------------------------- | ---------------------------------- |
| Storage                 | In-process memory | In-process memory                       | Separate cache service             |
| Network hop             | No                | No                                      | Yes                                |
| Shared across instances | No, not by itself | No, not by itself                       | Yes                                |
| Eviction and expiration | Yes               | Yes                                     | Yes, configurable                  |
| Best fit                | Fast local cache  | Local cache in Guava-based applications | Shared, distributed cache          |
| Main limitation         | Per-instance data | Per-instance data                       | Network and operational dependency |

For many modern Java applications, Caffeine is a strong default for local caching. Guava Cache remains useful in applications already using Guava or with existing integration needs. Redis is appropriate when shared access or centralized cache behavior is required.

These technologies are not mutually exclusive. An application can use a local cache in front of Redis and the database, provided its consistency rules are well defined.

## 28. Implementing a local cache with Caffeine

Example Maven dependency:

XML

```
<dependency>
    <groupId>com.github.ben-manes.caffeine</groupId>
    <artifactId>caffeine</artifactId>
    <version>${caffeine.version}</version>
</dependency>
```

Use a current, approved version managed through your dependency policy.

Example:

Java

```
import com.github.benmanes.caffeine.cache.Cache;
import com.github.benmanes.caffeine.cache.Caffeine;

import java.time.Duration;

public class ProductService {

    private final Cache<String, Product> cache =
            Caffeine.newBuilder()
                    .maximumSize(10_000)
                    .expireAfterWrite(Duration.ofMinutes(5))
                    .recordStats()
                    .build();

    public Product getProduct(String id) {
        return cache.get(id, this::loadProduct);
    }

    private Product loadProduct(String id) {
        // Load from repository or another source.
        return repositoryFindProduct(id);
    }

    private Product repositoryFindProduct(String id) {
        // Illustrative placeholder for the repository call.
        throw new UnsupportedOperationException("Implement repository lookup");
    }
}
```

This is illustrative code; replace the repository placeholder with your actual data access implementation.

### What does this configuration do?

* `maximumSize(10_000)`: Limits the number of cached entries.

* `expireAfterWrite(Duration.ofMinutes(5))`: Makes an entry eligible for expiration five minutes after it is written or updated.

* `recordStats()`: Enables cache statistics.

* `cache.get(id, loader)`: Loads the value on a cache miss and coordinates loading for the same key within the cache.

Expiration is not necessarily a precise timer that removes every entry at the exact instant its TTL ends. Also, a size bound limits entry count, not exact bytes. If values vary significantly in size, consider a weight-based bound.

### When is Caffeine a good choice?

Suppose product category names change only occasionally and are requested thousands of times per second.

A local cache can avoid repeated database lookups with very low access overhead.

But if the application runs on 20 instances, each instance may have a different cached version. You need an expiration or invalidation strategy that matches the business requirements.

## 29. Redis caching in a Spring Boot application

A common architecture is:

Spring Boot service

Redis cache lookup

Hit → return cached value

Database on cache miss

Store result in cache, then return response

A simplified cache-aside implementation looks like this:

Java

```
public Product getProduct(String id) {
    Product cached = redisCache.get(id);

    if (cached != null) {
        return cached;
    }

    Product product = repository.findById(id)
            .orElseThrow();

    redisCache.put(id, product, Duration.ofMinutes(5));

    return product;
}
```

The `redisCache` methods here are illustrative wrappers; a production implementation should use an appropriate Redis client or Spring Data Redis integration, with serialization, expiration and error handling configured explicitly.

Why use cache-aside?

The application controls when it reads from the database and when it populates the cache. It is simple and widely used.

What can go wrong?

* Multiple concurrent misses trigger repeated database queries.

* Data changes but stale cache entries remain.

* Redis becomes unavailable.

* Keys grow without appropriate limits.

* Serialization changes break compatibility.

* Cache data is exposed to users who should not see it.

## 30. Cache hit ratio and performance

The cache hit ratio measures the fraction of cache lookups that find a usable cached entry.

Hit ratio=Cache hitsCache hits+Cache misses×100%\text{Hit ratio} = \frac{\text{Cache hits}} {\text{Cache hits}+\text{Cache misses}} \times 100\%Hit ratio=Cache hits+Cache missesCache hits×100%

Example:

* Hits: 90,000

* Misses: 10,000

* Total lookups: 100,000

* Hit ratio: 90%

A high hit ratio is often useful, but it does not automatically prove that the cache improves overall performance.

If cached reads are expensive, cached values are tiny, or the underlying database is already very fast, the improvement may be smaller than expected.

Measure end-to-end latency and resource usage, not just hit ratio.

## 31. Cache invalidation: The difficult part

Suppose a product costs ₹1,000, and the value is cached.

The price changes to ₹1,200 in the database, but the cache still returns ₹1,000.

Now the application serves stale data.

Common approaches:

| Strategy                 | How it works                                                      | Best fit                                                      |
| ------------------------ | ----------------------------------------------------------------- | ------------------------------------------------------------- |
| TTL expiration           | Entry expires after a configured period                           | Data that can tolerate bounded staleness                      |
| Explicit invalidation    | Remove or update cache after a write                              | Data that changes through known application paths             |
| Write-through            | Update the cache as part of the write workflow                    | Workloads needing cache updates with writes                   |
| Write-behind             | Cache accepts writes and updates the backing store asynchronously | Specialized workloads where delayed persistence is acceptable |
| Event-based invalidation | Publish change events that trigger invalidation or updates        | Distributed applications with multiple writers                |

Each strategy has failure modes. For example, updating the database and cache separately is not automatically atomic.

A robust design defines what happens if one update succeeds and the other fails.

### Spring Cache annotations

With Spring's caching abstraction, you may use annotations such as:

Java

```
@Cacheable(cacheNames = "products", key = "#id")
public Product getProduct(String id) {
    return productRepository.findById(id).orElseThrow();
}
```

Java

```
@CacheEvict(cacheNames = "products", key = "#id")
public void updateProduct(String id, ProductUpdate update) {
    // Update the database.
}
```

`@Cacheable` checks the configured cache before invoking the method when caching is active. `@CacheEvict` removes the specified entry when its configured eviction behavior executes.

For correctness, verify the actual cache provider, TTL configuration, proxy behavior, transaction boundaries and exception handling. Spring annotations do not by themselves guarantee consistency across a database and a distributed cache.

## 32. Advanced caching problems architects must know

### Cache stampede

A popular key expires, and thousands of requests attempt to load it simultaneously.

Possible solutions:

* Coordinate loading for the same key.

* Refresh popular values before expiry.

* Use jittered TTLs to avoid synchronized expiration.

* Apply request coalescing or carefully designed locks.

* Use stale-while-revalidate when serving slightly stale data is acceptable.

### Cache penetration

Requests repeatedly ask for nonexistent keys, bypassing the cache and hitting the database.

Possible solutions:

* Short-lived negative caching.

* Input validation.

* Rate limiting.

* Carefully designed Bloom filters for suitable large key sets.

Negative caching must distinguish genuine absence from transient errors, and authorization-sensitive misses must not leak information.

### Cache avalanche

Many entries expire around the same time, causing a sudden surge in backing-store requests.

Possible solutions:

* Add randomized expiration jitter.

* Stagger cache refreshes.

* Use bounded concurrency and request coalescing.

* Protect the database with rate limits and bulkheads.

### Hot keys

One Redis key receives a disproportionate amount of traffic.

Potential solutions include local caching, key sharding where semantically possible, replication/read scaling and request coalescing. The appropriate approach depends on whether the key is mutable and what consistency guarantees are required.

## 33. When should you NOT cache?

Do not introduce caching automatically.

Be cautious when data is:

* Frequently updated.

* Required to be immediately consistent.

* Cheap to calculate or fetch.

* Highly sensitive or user-specific.

* Too large for the available memory.

* Difficult to invalidate correctly.

* Dependent on permissions that vary by user or tenant.

For banking balances, a stale display may be acceptable in some read-only contexts if clearly handled, but transaction authorization and balance validation must follow the authoritative data and consistency rules. A cache hit must never become a substitute for required correctness checks.

# Part 6: Latency, throughput and scalability

## 34. Latency vs. throughput

These concepts are related, but they answer different questions.

Latency: How long does one request take?

Throughput: How many requests can the system complete per unit of time?

Imagine a checkout counter.

* Latency is how long one customer waits before finishing checkout.

* Throughput is how many customers the cashier serves each minute.

A system can have high throughput and still provide poor latency if requests spend a long time waiting in queues.

### Example

| System | Throughput         | p99 latency |
| ------ | ------------------ | ----------- |
| A      | 1,000 requests/sec | 100 ms      |
| B      | 2,000 requests/sec | 5 sec       |

System B processes more requests per second, but one in a hundred requests may take five seconds or longer.

Which is better? It depends on the business SLO. For an interactive payment API, System B may be unacceptable.

## 35. What are p50, p95 and p99 latency?

Latency percentiles describe how requests are distributed.

* p50: Median latency; half of requests are at or below this value.

* p95: 95% of requests complete at or below this value.

* p99: 99% complete at or below this value.

Example:

| Metric | Value    |
| ------ | -------- |
| p50    | 50 ms    |
| p95    | 200 ms   |
| p99    | 1,200 ms |

Most requests are fast, but a small fraction are much slower.

That tail may be caused by GC pauses, database locks, slow downstream APIs, connection-pool waits, queueing, retries or noisy neighbors.

Averages alone can hide these problems. Always examine latency distributions and relevant error rates.

## 36. What is scalability?

Scalability is a system's ability to handle increased workload while meeting its performance, reliability and cost requirements.

### Vertical scaling

Increase the resources of one machine or instance.

```
Before: 4 vCPU, 8 GB RAM
After:  16 vCPU, 32 GB RAM
```

Advantages

* Often simpler to operate.

* May require fewer distributed-system changes.

* Can help when the bottleneck is a resource that can be enlarged.

Disadvantages

* Hardware and platform limits.

* Potentially higher cost.

* Does not automatically solve architectural bottlenecks.

* Can leave a single point of failure.

### Horizontal scaling

Add more instances.

```
                  Load Balancer
                 /      |      \
            App 1     App 2     App 3
                 \      |      /
                    Database
```

Advantages

* More capacity when work can be distributed.

* Can improve resilience when designed appropriately.

* Supports incremental capacity increases.

Disadvantages

* Requires attention to shared state, connection limits, distributed caching, coordination and load balancing.

* Does not automatically scale the database or downstream services.

* More instances can increase total pressure on shared dependencies.

### When does horizontal scaling work well?

It works best when requests can be handled independently, application state is externalized appropriately, and dependencies can support the added load.

If ten application instances all send five times as many queries to an already saturated database, adding instances may make the system slower.

## 37. What is a bottleneck?

A bottleneck is the resource or stage that limits overall system performance.

Common examples:

* CPU-bound serialization or computation.

* Slow database queries.

* A small connection pool.

* A saturated thread pool.

* Network latency.

* Lock contention.

* High garbage collection overhead.

* An overloaded external API.

Think of a highway with ten lanes that narrows to one lane. Adding more lanes before the narrowing point will not eliminate the queue at the one-lane section.

Architect's principle: Find the limiting resource before increasing capacity elsewhere.

## 38. Little's Law — a must-know interview concept

Little's Law relates the average number of items in a stable system to throughput and average time spent in the system.

L=λWL = \lambda WL=λW

Where:

* LLL = average number of requests in the system.

* λ\lambdaλ = average throughput, such as requests per second.

* WWW = average time spent in the system, in seconds.

### Example

Your API handles 500 requests per second, and the average time in the system is 0.2 seconds.

L=500×0.2=100L = 500 \times 0.2 = 100L=500×0.2=100

On average, approximately 100 requests are in the system.

If latency increases to 2 seconds while throughput remains 500 requests per second:

L=500×2=1000L = 500 \times 2 = 1000L=500×2=1000

Now approximately 1,000 requests are in the system on average.

This can mean more in-flight requests, more memory, more queued work and greater pressure on connection pools.

Interview insight: Little's Law describes a stable system's averages; it does not say that simply increasing concurrent requests will increase throughput.

## 39. Why does performance collapse under heavy load?

Consider a service whose downstream dependency can process 500 requests per second.

The application receives 800 requests per second.

If requests cannot be shed, queued or otherwise managed, backlog grows by roughly 300 requests per second while the overload persists.

Consequences include:

1. Queues grow.

2. Request latency increases.

3. More requests time out.

4. Retries generate additional work.

5. Threads and connections remain occupied longer.

6. Memory usage increases.

7. Throughput may eventually fall.

8. The system can enter a cascading failure.

This is why backpressure, bounded queues, rate limits, timeouts, circuit breakers and bulkheads matter.

### What is backpressure?

Backpressure means preventing a downstream consumer from being overwhelmed by more work than it can handle.

Depending on the architecture, this may involve:

* Bounded queues.

* Reactive demand signaling.

* Rate limiting.

* Rejecting excess requests.

* Slowing producers.

* Pausing message consumption.

An unbounded queue often turns an immediate, visible rejection into a delayed failure involving memory exhaustion and timeouts.

## 40. Thread pools and connection pools affect throughput

Suppose your Spring Boot application has a fixed number of database connections.

If all connections are occupied by long-running queries, new requests may wait even when CPU usage is low.

The same principle applies to thread pools: a pool can become saturated by blocking operations.

Useful metrics include:

* Active threads.

* Pool size.

* Queue length.

* Rejected tasks.

* Request wait time.

* Active and idle database connections.

* Connection acquisition timeout count.

* Downstream latency.

Do not automatically increase pool sizes. More concurrency can improve utilization when resources are underused, but can make an overloaded dependency worse.

### Java 21 virtual threads: Do they solve scalability?

Virtual threads make it cheaper to represent many concurrent blocking tasks than creating the same number of platform threads.

They can be useful for I/O-heavy applications, especially when code naturally uses blocking APIs.

However, virtual threads do not increase the capacity of:

* Your database.

* Your external API.

* A rate-limited service.

* A CPU-bound algorithm.

* A finite memory budget.

Use semaphores, connection pools, rate limits and bounded resource access where required.

## 41. Load testing: Prove that your optimization works

A local benchmark is useful, but production traffic includes concurrency, network behavior, database contention and unpredictable workloads.

A sound load test should define:

* Expected request mix.

* Arrival rate or concurrency model.

* Payload sizes.

* Data distribution.

* Cache warm-up state.

* Downstream dependencies.

* Test duration.

* Error-rate limits.

* Latency SLOs.

* Resource limits.

Common tools include JMeter, Gatling, k6 and wrk, depending on the protocol and test objective.

### A useful test sequence

1. Establish a baseline at normal load.

2. Increase traffic gradually.

3. Observe throughput, p95/p99, CPU, memory, GC, queues and connection pools.

4. Test expected peak traffic.

5. Test bursts and sustained overload.

6. Identify where latency or errors start rising sharply.

7. Test recovery after the load drops.

8. Repeat after each meaningful optimization.

A test that only reports maximum requests per second is incomplete. You need to know the latency and error rate at that throughput.

# Part 7: Real-world production scenarios

This is where technical architect interviews become interesting. The interviewer is not simply checking whether you know what G1 GC is. They want to know whether you can make good decisions under pressure.

## 42. Scenario 1: API latency increases from 200 ms to 3 seconds

Situation: A Spring Boot application was performing well yesterday. Today, p99 latency has increased to three seconds.

Do not immediately increase heap size or add more servers.

Investigate in this order:

| Step | What to inspect             | What you are trying to discover                         |
| ---- | --------------------------- | ------------------------------------------------------- |
| 1    | Request rate and error rate | Did traffic change?                                     |
| 2    | Distributed traces          | Which part of the request is slow?                      |
| 3    | Database metrics            | Slow queries, locks or pool exhaustion?                 |
| 4    | Thread pools                | Are requests waiting for workers?                       |
| 5    | JVM metrics                 | CPU, heap, allocation and GC pauses                     |
| 6    | Downstream services         | Are external calls slow or failing?                     |
| 7    | Recent deployments          | Did a code or configuration change introduce the issue? |

Suppose traces show that 2.5 seconds of the request are spent waiting for an external API.

The best next step is to investigate that dependency, its timeouts, concurrency and failure behavior—not to tune the garbage collector without evidence.

## 43. Scenario 2: Heap usage increases until the application crashes

Situation: A service starts with 1 GB of heap, grows to 3 GB and eventually crashes with an OOM error.

Possible causes:

* A true memory leak.

* A legitimate increase in live data.

* A cache with no effective bound.

* Large temporary allocations.

* A sudden increase in traffic.

* A large batch loaded entirely into memory.

* Unexpected retention of request or session objects.

### Architect's solution

1. Identify the exact OOM error and inspect container memory.

2. Review heap trends and GC logs.

3. Capture and analyze a heap dump if appropriate.

4. Find classes with high retained heap and their GC-root paths.

5. Fix the root cause.

6. Repeat the same workload and verify stable post-GC memory usage.

7. Adjust heap size only if measurements show that the legitimate live set requires it and the memory budget permits it.

If a cache is retaining ten million obsolete entries, increasing Xmx may simply postpone the next crash.

## 44. Scenario 3: Database is slow, but CPU is only 20%

Situation: Your API is slow, but the application CPU is not heavily utilized.

This suggests that the application may be waiting rather than doing computational work.

Investigate:

* Query execution plans.

* Missing or ineffective indexes.

* Lock contention.

* Database connection pool saturation.

* Network latency.

* N+1 query patterns.

* Excessive transactions or large result sets.

* Database CPU, I/O and connection limits.

Do not increase the Java thread pool blindly. That may create more concurrent database requests and worsen contention.

## 45. Scenario 4: High CPU and frequent garbage collection

Situation: CPU is 95%, allocation rate is high, and the service spends considerable time in GC.

Potential root causes:

* Creating temporary objects in hot loops.

* Excessive JSON serialization/deserialization.

* Repeated string concatenation in inefficient patterns.

* Large intermediate collections.

* Unnecessary copying of arrays or buffers.

* High traffic with no allocation-efficiency improvements.

Use allocation profiling to find the code creating the objects.

Possible optimizations:

* Avoid repeated conversions.

* Reuse expensive immutable computed results where safe.

* Process large data in chunks.

* Stream records instead of materializing everything.

* Remove unnecessary intermediate collections.

* Reduce redundant serialization.

Do not introduce object pooling for ordinary short-lived Java objects without strong evidence. Modern JVM allocation is often very efficient, while object pools add complexity and can retain memory longer.

## 46. Scenario 5: Redis cache hit ratio is 98%, but latency is still high

A high hit ratio is not enough.

Investigate:

* Redis response latency.

* Network latency between application and Redis.

* Serialization/deserialization costs.

* Large values or expensive object reconstruction.

* Hot-key contention.

* Connection pool exhaustion.

* Cache lookups performed repeatedly during a single request.

* Whether the remaining misses are disproportionately expensive.

A distributed cache still requires a network hop. For some small, frequently accessed, mostly immutable values, a local Caffeine cache in front of Redis may reduce latency.

But this introduces another cache layer and more invalidation complexity. Measure first.

## 47. Scenario 6: Application becomes slow after adding more instances

Situation: You scale from four application instances to twelve, but latency increases.

Possible explanation:

* Each instance creates its own connection pool.

* Total database connections exceed the database's useful capacity.

* Cache misses increase across new instances.

* Downstream APIs become overloaded.

* Background jobs run independently on every instance.

* Shared locks or hot keys become more contentious.

For example, if each instance has a 40-connection database pool:

* 4 instances permit up to 160 connections.

* 12 instances permit up to 480 connections.

That may be far more than the database can handle efficiently.

Architect's lesson: Capacity planning must consider the whole dependency chain, not just application instances.

## 48. Scenario 7: A bulk-import job processes ten million records

Consider an enterprise import service built with Java 21, Spring Batch and PostgreSQL.

Loading ten million records into a single in-memory list is risky because of object overhead, heap pressure and GC costs.

A more scalable design would consider:

* Chunk-based reading and writing.

* Bounded in-flight work.

* JDBC batch inserts.

* Streaming CSV parsing.

* Database-side bulk loading where appropriate.

* Controlled concurrency.

* Idempotent processing and checkpoints.

* Separate handling for invalid records.

* Measuring parsing, enrichment and database time independently.

If database insertion is the slowest stage, adding more parser threads may not improve total job time.

If a remote enrichment API is the bottleneck, bounded parallelism and request batching may help, provided the API's rate limits are respected.

The objective is not to maximize every component's parallelism. It is to balance the pipeline around the slowest constrained stage.

## 49. Scenario 8: p50 is excellent, but p99 is terrible

Situation: Median latency is 40 ms, but p99 latency is 2.5 seconds.

Investigate tail-latency causes:

* GC pauses.

* Occasional slow database queries.

* Lock contention.

* Connection-pool waits.

* DNS or network stalls.

* Retry delays.

* Cold cache misses.

* Long-running requests competing with short requests.

* Uneven load balancing.

Look at latency by endpoint, dependency, instance and request type. Averages across the entire application can hide one badly performing route.

A useful architecture improvement may be workload isolation: prevent expensive report generation from starving latency-sensitive transaction requests.

# Part 8: An architect's performance troubleshooting playbook

## 50. The step-by-step framework to remember

Use this framework for almost any performance incident.

## 1

Define the symptom

Is the problem latency, throughput, errors, CPU, memory or cost? Identify the affected endpoints and time window.

## 2

Establish a baseline

Capture traffic, p95/p99 latency, throughput, error rate, CPU, memory, GC, thread pools and dependency metrics.

## 3

Locate the bottleneck

Use traces, query plans, JFR, CPU profiles, heap analysis and thread dumps as appropriate.

## 4

Form a testable hypothesis

Example: repeated linear searches are causing CPU overhead, or a database connection pool is saturated.

## 5

Make one targeted change

Optimize the confirmed cause before making unrelated JVM or infrastructure changes.

## 6

Measure and validate

Repeat the workload. Compare latency percentiles, throughput, errors, resource usage and cost.

## 7

Protect the improvement

Add dashboards, alerts, regression tests and capacity limits. Roll out safely and retain a rollback path.

### Which tool should you use first?

| Symptom                  | First useful investigation                          |
| ------------------------ | --------------------------------------------------- |
| High CPU                 | CPU profile, JFR, hot methods                       |
| Rising heap after GC     | Heap dump, retained objects, GC logs                |
| Threads stuck            | Thread dump, pool metrics, traces                   |
| Slow endpoint            | Distributed tracing, endpoint-level latency         |
| High database time       | Query plans, database metrics, connection pool      |
| Slow cache reads         | Redis latency, network, serialization, key patterns |
| High p99 only            | Tail-latency breakdown, GC, locks, retries          |
| OOM kill in a container  | Container memory, RSS, heap and native memory       |
| Throughput stops growing | Saturated dependency, queues, pools and load test   |

# Part 9: Interview questions and answers

The following questions are phrased the way a senior Java engineer or technical architect might be asked in an interview.

## 51. Profiling and monitoring questions

### Q1. What is the difference between monitoring and profiling?

Answer: Monitoring tracks system health and behavior over time through metrics, logs and traces. Profiling examines runtime behavior to identify expensive code, allocations, contention or other hotspots.

Monitoring tells me that latency increased. Profiling helps me discover which methods or resource patterns contributed to the increase.

### Q2. How would you investigate high CPU usage in a Java application?

Answer:

1. Confirm CPU usage and affected instances.

2. Check traffic volume and recent deployments.

3. Capture a CPU profile or JFR recording.

4. Identify hot methods and excessive computation.

5. Inspect loops, serialization, regex processing and contention.

6. Optimize the confirmed hotspot.

7. Run a representative load test and compare results.

I would not increase CPU capacity until I understood whether the workload could be made more efficient.

### Q3. What is the difference between a heap dump and a thread dump?

A heap dump captures heap objects and their reference relationships at a point in time, helping diagnose memory retention and leaks.

A thread dump captures thread states and stack traces, helping diagnose blocked threads, deadlocks, contention and waiting.

### Q4. What is JFR, and why is it useful in production?

JFR records JVM runtime events with configurable settings and overhead. It can provide evidence about execution, allocation, GC and thread behavior.

It is useful for investigating intermittent issues without relying exclusively on intrusive profiling. I would still validate its overhead and recording configuration for the specific production workload.

### Q5. What is the difference between self time and inclusive time?

Self time is time attributed directly to a method. Inclusive time includes the method's child calls.

If an endpoint spends 90% of its inclusive time in a database client, optimizing a few arithmetic operations in the endpoint may have little effect.

### Q6. How do you detect a memory leak in Java?

Compare heap usage and object retention under a stable workload. Examine post-GC heap trends, heap dumps, retained heap and GC-root paths. Confirm that objects remain reachable unnecessarily and fix the retention source.

### Q7. What is the difference between a memory leak and high memory usage?

High memory usage may be legitimate because the application has a large working set or cache. A memory leak is unnecessary retention of objects that should no longer be needed.

A large heap alone is not proof of a leak.

### Q8. Why might a profiler change the performance of an application?

Profilers can introduce sampling, instrumentation, event recording or stack-walking overhead. They may also alter timing and thread scheduling.

I would choose a suitable profiling mode, measure its overhead and avoid interpreting a heavily instrumented run as identical to uninstrumented production.

## 52. JVM and GC questions

### Q9. What is the difference between `-Xms` and `-Xmx`?

`-Xms` configures the initial/minimum heap size, and `-Xmx` sets the maximum heap size. The right values depend on the live set, allocation behavior, workload and total process memory budget.

### Q10. Does increasing the heap always improve performance?

No. It can reduce allocation pressure or collection frequency if the heap was too small, but it can also increase resource usage and allow more live objects to accumulate. It does not fix leaks or slow external dependencies.

### Q11. Which garbage collector would you choose for a Spring Boot microservice?

I would start with the collector supported and appropriate for the JDK and workload—often G1 as a general-purpose baseline. If strict tail-latency requirements justify it, I would benchmark ZGC or another suitable low-pause collector.

The decision would be based on p95/p99 latency, throughput, GC pauses, CPU, memory and reliability under representative load.

### Q12. What is a stop-the-world pause?

A stop-the-world pause is a period when application threads are paused for a GC or another JVM operation requiring them to stop. Not every GC phase necessarily stops all application threads.

Long pauses can directly increase request latency when they overlap with request processing.

### Q13. What is the difference between heap and metaspace?

Heap holds most Java objects. Metaspace holds class metadata and uses native memory rather than the Java object heap.

Increasing Xmx does not directly solve metaspace exhaustion.

### Q14. Can Java have memory leaks despite garbage collection?

Yes. GC only reclaims objects that are no longer reachable. If a static collection, cache, listener or other reachable object retains unused objects, GC cannot reclaim them.

### Q15. Why can an application be killed even if heap usage is below Xmx?

Because the process also consumes native memory, thread stacks, direct buffers, code cache and other resources. The container or operating system can terminate the process when its total memory usage exceeds the permitted limit.

### Q16. Is `System.gc()` a good production fix?

Generally no. It is not a reliable way to solve memory leaks or allocation problems. It can introduce unnecessary collection work, and its behavior depends on JVM settings and implementation.

### Q17. What is allocation rate?

Allocation rate measures how quickly an application creates objects, commonly expressed in bytes per second.

A high allocation rate can increase GC work even when most objects die quickly. Allocation profiling can identify which code paths create the most objects.

### Q18. Why can increasing thread count make a service slower?

More threads can increase scheduling overhead, memory consumption, lock contention and pressure on downstream systems. If the bottleneck is a database with a fixed capacity, more concurrent callers may worsen latency.

## 53. Collections and algorithm questions

### Q19. Why is HashMap generally faster than TreeMap for lookup?

HashMap typically offers expected O(1)O(1)O(1) lookup under suitable hashing conditions. TreeMap provides O(log⁡n)O(\log n)O(logn) lookup while maintaining sorted key order.

HashMap is a common choice for general key-value lookup; TreeMap is useful when sorted order or range operations are needed.

### Q20. Why is ArrayList often preferred over LinkedList?

ArrayList offers fast indexed access, compact reference storage and generally good cache locality. LinkedList requires node objects and pointer traversal, and finding an arbitrary position is linear.

LinkedList can have specialized uses, but it is not automatically faster for insertion or deletion.

### Q21. How do you optimize duplicate detection?

Use a HashSet to track values already seen when the dataset fits in memory and typical hash behavior is suitable. This generally reduces quadratic pairwise comparisons to expected linear-time processing, at the cost of extra memory.

### Q22. What causes HashMap collisions?

Different keys can map to the same bucket. HashMap uses hash values to locate candidate entries and `equals()` to determine whether a key matches.

A well-behaved `hashCode()` helps distribute entries and reduce collisions.

### Q23. When would you use a PriorityQueue?

For scheduling and retrieving the highest- or lowest-priority item efficiently. For example, it can help implement top-K selection or a task scheduler.

Insertion and removal of the head are generally O(log⁡n)O(\log n)O(logn), while reading the head is O(1)O(1)O(1).

### Q24. Are Java streams slower than loops?

Not necessarily. Performance depends on the workload, data size, allocation, implementation and JVM optimizations. Use streams for readability where appropriate, and benchmark representative workloads if performance is critical.

### Q25. When is parallelStream a bad choice?

It can be a poor choice for small tasks, blocking I/O, workloads with shared-state contention, or applications that cannot afford competition for shared execution resources. Measure before using parallel execution in latency-sensitive services.

### Q26. How can an application process millions of records without exhausting heap memory?

Use streaming or chunk-based processing, bounded concurrency, batched database writes and controlled object lifetimes. Avoid materializing the entire dataset unless memory requirements and capacity have been validated.

## 54. Caching questions

### Q27. When would you choose Caffeine over Redis?

I would choose Caffeine for very fast local access when per-instance caching is acceptable and freshness can be managed locally. I would choose Redis when data must be shared across instances or a distributed cache is required.

They can also be combined if the additional consistency and invalidation complexity is justified.

### Q28. What is cache-aside?

The application checks the cache first. On a miss, it loads the data from the authoritative source, stores it in the cache and returns it.

It is simple and flexible, but needs a strategy for stale values, concurrent misses and cache failures.

### Q29. What is a cache stampede?

A large number of requests attempt to load the same missing or expired key at once, potentially overwhelming the database.

Request coalescing, early refresh, jittered TTLs and bounded concurrency are common mitigations.

### Q30. How do you maintain cache consistency?

Define the allowed staleness and use an appropriate combination of TTLs, explicit invalidation, write-through behavior or change events. Handle failures between the database and cache updates, and never rely on caching to bypass required correctness checks.

### Q31. Is a 99% cache hit ratio always good?

No. A cache might add more overhead than it saves, or cache incorrect or stale values. Evaluate end-to-end latency, backing-store load, correctness and resource cost.

### Q32. What happens if Redis goes down?

The application needs a deliberate failure policy. Depending on the data and service, it might use a local fallback, read directly from the database with strict concurrency limits, return a controlled error, or degrade nonessential features.

Allowing every request to bypass Redis simultaneously can overload the database. A cache outage should not automatically become a database outage.

## 55. Latency, throughput and scalability questions

### Q33. What is the difference between latency and throughput?

Latency measures the time taken by an operation. Throughput measures how many operations complete per unit of time. A system can have high throughput but poor tail latency.

### Q34. Why do you monitor p99 latency?

p99 highlights slow requests that an average can hide. It helps identify tail-latency problems that may affect user experience, especially in systems with strict response-time objectives.

### Q35. What is Little's Law?

It is L=λWL = \lambda WL=λW: average work in a stable system equals average arrival/completion rate multiplied by average time in the system.

It helps reason about in-flight requests and queue growth.

### Q36. Vertical scaling vs. horizontal scaling?

Vertical scaling increases resources on an instance. Horizontal scaling adds instances.

Vertical scaling is often simpler, but limited by instance capacity. Horizontal scaling can improve capacity and resilience, but requires handling shared state, load balancing and downstream limits.

### Q37. Why can adding servers reduce performance?

More servers may generate more database connections, cache misses, background work or downstream calls. If a shared dependency is already saturated, additional instances can worsen contention and latency.

### Q38. How do you determine the maximum capacity of a service?

Load-test realistic traffic while measuring throughput, latency percentiles, errors and resource saturation. Identify the point at which SLOs fail or a limiting resource becomes saturated, then establish a safe operating capacity with headroom.

### Q39. What is backpressure?

Backpressure prevents producers from overwhelming consumers. Depending on the design, it may involve bounded queues, demand signaling, rate limiting or rejecting excess work.

### Q40. What is the difference between load testing and stress testing?

Load testing checks behavior under expected and peak workloads. Stress testing pushes beyond normal capacity to discover failure modes, limits and recovery behavior.

Both are valuable, but they answer different questions.

# Part 10: Best practices, worst practices and architect-level decisions

## 56. Best practices vs. worst practices

| Area               | Best practice                               | Worst practice                                  |
| ------------------ | ------------------------------------------- | ----------------------------------------------- |
| Profiling          | Measure before optimizing                   | Guess based on symptoms                         |
| Monitoring         | Track latency percentiles and errors        | Monitor only average latency                    |
| JVM tuning         | Tune using representative workloads         | Copy JVM flags from another project             |
| Heap sizing        | Account for total container memory          | Set Xmx equal to the container limit            |
| Garbage collection | Compare collectors against requirements     | Assume one collector is always best             |
| Collections        | Choose based on access patterns             | Choose based on familiarity alone               |
| Algorithms         | Reduce unnecessary work and complexity      | Optimize syntax without measuring               |
| Caching            | Define TTL, bounds and invalidation         | Cache everything indefinitely                   |
| Concurrency        | Bound work and protect dependencies         | Maximize thread count blindly                   |
| Scalability        | Find the bottleneck across the system       | Add instances without checking the database     |
| Load testing       | Test realistic traffic and failure behavior | Measure only maximum requests per second        |
| Production changes | Roll out safely and validate                | Change heap, GC and thread pools simultaneously |

## 57. Performance optimization decision guide

When you encounter a performance problem, use this table to narrow your investigation.

| Observation                      | Likely area to investigate                              |
| -------------------------------- | ------------------------------------------------------- |
| High CPU, low GC overhead        | Algorithm, hot methods, serialization, contention       |
| High allocation rate             | Object creation, temporary collections, conversions     |
| Rising post-GC heap              | Memory retention, growing live set, unbounded cache     |
| Long GC pauses                   | Collector behavior, heap pressure, allocation patterns  |
| Low CPU, high latency            | I/O waits, locks, connection pools, queueing            |
| Database latency high            | Query plans, indexes, locks, connection limits          |
| Redis hit ratio low              | TTL, cache keys, workload distribution, invalidation    |
| Throughput plateaus              | Saturated dependency, CPU, connection pool or queue     |
| p99 much worse than p50          | Tail latency, retries, pauses, locks, slow dependencies |
| More instances make things worse | Shared-resource saturation or coordination overhead     |

## 58. Important production anti-patterns

These are particularly useful to recognize during architecture interviews.

Anti-pattern 1: Increasing heap to fix a memory leak.

Why it is bad: It can delay the failure without fixing the underlying retention.

Anti-pattern 2: Increasing threads to fix slow APIs.

Why it is bad: If the database or downstream API is saturated, more concurrent work can worsen the queue.

Anti-pattern 3: Caching data without an invalidation strategy.

Why it is bad: The application may return stale or incorrect information.

Anti-pattern 4: Using an unbounded queue.

Why it is bad: Excess work accumulates until latency and memory consumption become unacceptable.

Anti-pattern 5: Optimizing code before measuring.

Why it is bad: You may spend time optimizing code that contributes very little to the total request duration.

Anti-pattern 6: Treating a benchmark as production truth.

Why it is bad: Local tests often omit realistic concurrency, network delays, database contention and deployment constraints.

Anti-pattern 7: Maximizing cache hit ratio at any cost.

Why it is bad: A huge cache can consume excessive memory, and stale values can violate business requirements.

Anti-pattern 8: Assuming horizontal scaling solves everything.

Why it is bad: The database, message broker, external API or another shared dependency may be the actual capacity limit.

# Part 11: Your interview revision sheet

## 59. The 30 concepts you should be able to explain confidently

## Interview readiness checklist

0/30

Latency vs. throughput vs. scalability

p50, p95 and p99 latency

Monitoring vs. profiling vs. benchmarking

VisualVM vs. JProfiler vs. JFR

CPU profiling and hot methods

Heap dumps vs. thread dumps

Memory leaks and GC roots

Heap vs. stack vs. metaspace vs. native memory

-Xms vs. -Xmx

Young generation and object lifetime

G1 GC vs. ZGC vs. Parallel GC

GC logs and pause-time analysis

OOM errors and container memory limits

Big-O time and space complexity

ArrayList vs. LinkedList

HashMap internals and collisions

equals() and hashCode() contracts

HashSet vs. TreeSet

HashMap vs. TreeMap

Streams vs. loops vs. parallel streams

N+1 queries and database batching

Caffeine vs. Guava Cache vs. Redis

Cache-aside and TTL

Cache invalidation and consistency

Cache stampede, penetration and avalanche

Vertical vs. horizontal scaling

Little's Law

Thread pools and connection pools

Backpressure and bounded queues

Load testing, stress testing and capacity planning

Reset checklist

## 60. How to answer a performance scenario in a senior interview

When an interviewer gives you a production problem, avoid jumping straight to a solution.

Use this response structure:

1. Clarify the requirement. What is the latency SLO, expected traffic, error rate and affected functionality?

2. Establish the baseline. What changed, when did it change, and which metrics are abnormal?

3. Localize the bottleneck. Is the time spent in Java code, GC, the database, the network or a queue?

4. State a hypothesis. Explain the evidence that supports it.

5. Propose the smallest effective change. Avoid unnecessary architectural complexity.

6. Validate the change. Compare p95/p99, throughput, errors, resource usage and correctness.

7. Prevent recurrence. Add alerts, capacity limits, regression tests and a safe rollout plan.

### Example: A polished architect-level answer

Writing

If a Spring Boot API becomes slow in production, I would first establish whether the issue is increased latency, reduced throughput, elevated errors or resource saturation.

I would examine request-level traces and application metrics to identify where time is being spent. If CPU is high, I would use JFR or a CPU profiler to locate hot methods. If heap usage is growing, I would analyze GC logs and heap dumps. If threads are waiting, I would inspect thread dumps, connection pools and downstream dependencies.

Once I identify the bottleneck, I would form a hypothesis and make a targeted change—for example, optimizing an expensive algorithm, fixing a database query, introducing a bounded cache or applying backpressure.

Finally, I would load-test the change against a representative workload, compare p95/p99 latency, throughput, error rates and resource consumption, and roll it out safely with monitoring and a rollback plan.

My goal would not simply be to make the code faster, but to meet the service's performance, reliability and cost requirements without creating bottlenecks elsewhere.

## Final takeaway

Remember these five principles:

* Measure before you optimize. Profiling tells you where the work is happening.

* Understand memory before tuning GC. Heap size, allocation rate and object retention are different problems.

* Choose data structures based on complexity and workload. A better algorithm can outperform infrastructure upgrades.

* Cache deliberately. Fast reads are valuable, but freshness, memory bounds and failure behavior matter just as much.

* Optimize the entire system. Latency, throughput, concurrency and scalability depend on the application and all its dependencies.

For a technical architect, the strongest answer is rarely “I would use Redis” or “I would increase the heap.” It is: “I would identify the limiting resource, validate the root cause with evidence, choose the simplest appropriate solution, and prove that it meets the system's SLOs.”
