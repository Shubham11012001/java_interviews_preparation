# Hibernate & JPA: The Interview Cheatsheet (Explained So a 10-Year-Old Gets It)

> **Who this is for:** a backend engineer with ~5 years of experience who wants to walk into an interview and answer *anything* about Hibernate/JPA, from "what is it?" to "your production API is slow, why?"

---

## 1. The Big Picture

### The toy-box analogy

Your Java objects are **LEGO models** (rich, connected, with behavior). Your database is a **filing cabinet** (flat tables, rows, foreign keys).

Moving a LEGO model into a filing cabinet means taking it apart, labelling every brick, and filing each in the right drawer. Getting it back means finding all the bricks and rebuilding the model. Doing that by hand, thousands of times a day, is what **JDBC** feels like.

**ORM (Object-Relational Mapping)** is a robot assistant that does the taking-apart and rebuilding for you.

### The problem it solves: "Impedance mismatch"

| Object world | Relational world |
|---|---|
| Classes, inheritance | Tables, no inheritance |
| References (`order.getCustomer()`) | Foreign keys + JOINs |
| Identity (`==` / `equals`) | Primary key |
| Collections | Separate tables |
| Navigate the graph | Write joins |

### JPA vs Hibernate vs Spring Data JPA

This is **the** classic opening question.

```
┌───────────────────────────────────────────┐
│ Spring Data JPA   (repositories, less code)│  ← convenience layer
├───────────────────────────────────────────┤
│ JPA  (Jakarta Persistence)                 │  ← the SPECIFICATION (interfaces + annotations)
├───────────────────────────────────────────┤
│ Hibernate (also EclipseLink, OpenJPA)      │  ← the IMPLEMENTATION (the actual engine)
├───────────────────────────────────────────┤
│ JDBC  →  Database                          │
└───────────────────────────────────────────┘
```

- **JPA** = the *rulebook* (like the `List` interface).
- **Hibernate** = a *player* following the rulebook (like `ArrayList`). It also offers extras beyond JPA (`@BatchSize`, `@NaturalId`, `@Formula`, `StatelessSession`, etc.).
- **Spring Data JPA** = a *helper* that writes repository boilerplate on top of JPA (`findByEmail(...)` becomes a query automatically).

> **Interview line:** "JPA is a spec, Hibernate is the most popular implementation, and Spring Data JPA is an abstraction over JPA that removes DAO boilerplate."

**Naming note:** since Jakarta EE 9 / Hibernate 6 / Spring Boot 3, imports are `jakarta.persistence.*`, not `javax.persistence.*`. Interviewers like this detail.

---

## 2. When to Use It (and When NOT To)

### Great fit
- CRUD-heavy business apps (orders, users, accounts)
- Domain models with relationships
- You want portability across databases
- You want less boilerplate and faster development

### Bad fit
- Heavy reporting and analytics with complex SQL (use jOOQ, MyBatis, native SQL, or JdbcTemplate)
- Massive bulk operations (millions of rows)
- Very legacy or odd schemas that don't map to objects
- Teams that don't understand what SQL Hibernate is generating (this is the real danger)

> **Golden rule:** Hibernate doesn't remove the need to know SQL. It hides SQL, and hidden SQL is where performance bugs live.

---

## 3. Core Architecture (Know These Names)

| Hibernate | JPA equivalent | What it is | Analogy |
|---|---|---|---|
| `SessionFactory` | `EntityManagerFactory` | Heavy, thread-safe, created **once** per app/DB | The factory building workers |
| `Session` | `EntityManager` | Lightweight, **not thread-safe**, one per unit of work | One worker with a notepad |
| `Transaction` | `EntityTransaction` | Atomic unit of work | "All or nothing" |
| Persistence Context | same | The Session's **first-level cache** and tracked entities | The worker's notepad |

- `Session extends EntityManager`, and `SessionFactory extends EntityManagerFactory`.
- In Spring, you rarely create these manually. The container injects a **shared EntityManager proxy** that is transaction-bound.

---

## 4. Entity Basics

```java
@Entity
@Table(name = "employees", indexes = @Index(columnList = "email"))
public class Employee {

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "emp_seq")
    @SequenceGenerator(name = "emp_seq", sequenceName = "emp_seq", allocationSize = 50)
    private Long id;

    @Column(nullable = false, unique = true, length = 100)
    private String email;

    @Enumerated(EnumType.STRING)      // ALWAYS STRING, never ORDINAL
    private Status status;

    @Version                          // optimistic locking
    private Long version;

    @Transient                        // not persisted
    private String tempValue;

    protected Employee() {}           // required by JPA (public or protected)
}
```

### Entity rules (interview favorite)
1. Must be annotated `@Entity`
2. Must have a **no-arg constructor** (public or protected)
3. Must **not be `final`** (Hibernate creates proxy subclasses for lazy loading)
4. Must have an `@Id`
5. Fields shouldn't be `final`

### Useful annotations

| Annotation | Purpose |
|---|---|
| `@Embeddable` / `@Embedded` | Reuse a value object (e.g., `Address`) as columns in the same table |
| `@Enumerated(STRING)` | Store enum name (ORDINAL breaks when you reorder enums) |
| `@Lob` | Big text/binary |
| `@Convert` | Custom attribute converter |
| `@Formula` (Hibernate) | Computed column via SQL |
| `@NaturalId` (Hibernate) | Business key (email, ISBN) |
| `@Immutable` | Never updated; skips dirty checking |
| `@DynamicUpdate` | Only changed columns in `UPDATE` |
| `@CreationTimestamp` / `@UpdateTimestamp` | Auto timestamps |

### ID generation strategies

| Strategy | How | Batch inserts? | Notes |
|---|---|---|---|
| `IDENTITY` | DB auto-increment | **No** (Hibernate must insert immediately to learn the ID) | Simple, but kills JDBC batching |
| `SEQUENCE` | DB sequence, with pooled allocation | **Yes** | **Best for PostgreSQL/Oracle**; `allocationSize=50` means 1 DB call per 50 IDs |
| `TABLE` | Table simulating a sequence | Yes, but slow | Avoid |
| `AUTO` | Provider picks | Depends | Be explicit instead |

> **Prod tip:** MySQL has no real sequences (until recently in MariaDB), so people use IDENTITY and accept no batching, or use UUIDs or a custom generator.

---

## 5. Entity Lifecycle (Super Important)

Think of a **hotel guest**:

```
          new Employee()
               │
          ┌────▼─────┐
          │ TRANSIENT │  ← Not in DB, not tracked. A stranger outside the hotel.
          └────┬─────┘
     persist() │
          ┌────▼─────┐   find()/query
          │ MANAGED   │◄──────────────  ← Checked in. Hibernate watches every move.
          │(PERSISTENT)│
          └─┬──────┬──┘
   detach() │      │ remove()
   close()  │      │
   clear()  │ ┌────▼─────┐
          ┌─▼─┴──┐ │ REMOVED  │  ← Scheduled for DELETE on flush
          │DETACHED│ └──────────┘
          └───┬───┘
              │ merge()  → returns a NEW managed copy
```

| State | In DB? | Tracked? |
|---|---|---|
| Transient | No | No |
| Managed | Yes (or will be) | **Yes** |
| Detached | Yes | No |
| Removed | Being deleted | Yes (until flush) |

### Key operations

| Method | What it does |
|---|---|
| `persist(e)` | Makes a **new** entity managed. Returns void. The same instance becomes managed. |
| `merge(e)` | Copies state of a detached/new entity onto a managed one. **Returns a different managed instance.** |
| `find(Class, id)` | Hits DB (or L1 cache), returns real object or null |
| `getReference(Class, id)` | Returns a **lazy proxy** without hitting DB (great for setting FKs cheaply) |
| `remove(e)` | Schedules delete |
| `refresh(e)` | Reloads state from DB, overwriting in-memory changes |
| `flush()` | Pushes pending SQL to DB (not commit!) |
| `detach(e)` / `clear()` | Stop tracking one / all entities |

**Spring Data's `save()`:** calls `persist` if the entity is *new* (ID null or version null), otherwise `merge`. A `merge` on an entity with an assigned ID triggers an extra `SELECT` first. That's a hidden cost.

---

## 6. Persistence Context: The Magic (and the Traps)

The persistence context is the Session's **notepad**. It gives you three superpowers:

### 6.1 First-Level Cache (L1)
Within one session/transaction, asking for the same ID twice hits the DB **once**.

```java
Employee a = em.find(Employee.class, 1L); // SELECT
Employee b = em.find(Employee.class, 1L); // no SQL, from cache
System.out.println(a == b);               // true: same instance
```

Always on, can't be disabled, and scoped to the session.

### 6.2 Automatic Dirty Checking
Change a managed entity and **don't call save**. Hibernate notices.

```java
@Transactional
public void giveRaise(Long id) {
    Employee e = repo.findById(id).get();
    e.setSalary(e.getSalary() + 1000);   // no save() needed!
}   // on commit → flush → UPDATE employees SET salary=? WHERE id=?
```

*How:* Hibernate keeps a **snapshot** of loaded state and compares at flush time. This is also why loading 100k entities uses lots of memory (entity plus snapshot).

### 6.3 Write-Behind (Transactional Write-Behind)
SQL isn't fired immediately. Hibernate queues up changes and flushes them:
- before a query that might be affected (when `FlushMode.AUTO`),
- on `flush()`,
- on **commit**.

### Flush modes
- `AUTO` (default): flush before queries touching affected tables, and at commit
- `COMMIT`: only at commit
- `MANUAL` (Hibernate): only when you call `flush()`. Spring sets this for `@Transactional(readOnly = true)`, skipping dirty checking and saving CPU and memory.

---

## 7. Relationships (The Heart of Interviews)

### The 4 relationships

| Annotation | Example | Default fetch |
|---|---|---|
| `@ManyToOne` | Many orders → one customer | **EAGER** |
| `@OneToOne` | User ↔ Profile | **EAGER** |
| `@OneToMany` | One customer → many orders | **LAZY** |
| `@ManyToMany` | Students ↔ Courses | **LAZY** |

> **Memorize:** "To-One is EAGER, To-Many is LAZY by default." **Best practice: make everything LAZY explicitly** and fetch what you need per use case.

### Owning side vs inverse side

The **owning side** holds the foreign key and is the only side Hibernate looks at for writing the relationship. The **inverse side** uses `mappedBy` and is read-only from the persistence point of view.

```java
@Entity
public class Customer {
    @Id @GeneratedValue private Long id;

    @OneToMany(mappedBy = "customer", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Order> orders = new ArrayList<>();

    // ALWAYS provide helper methods to keep both sides in sync
    public void addOrder(Order o) { orders.add(o); o.setCustomer(this); }
    public void removeOrder(Order o) { orders.remove(o); o.setCustomer(null); }
}

@Entity
public class Order {
    @Id @GeneratedValue private Long id;

    @ManyToOne(fetch = FetchType.LAZY)       // owning side → has the FK column
    @JoinColumn(name = "customer_id")
    private Customer customer;
}
```

### Best practices for each

- **Prefer bidirectional `@OneToMany` + `@ManyToOne`**, or just unidirectional `@ManyToOne`. The `@ManyToOne` side is the efficient one.
- **Avoid unidirectional `@OneToMany` without `@JoinColumn`.** It creates an extra join table and extra SQL.
- **`@ManyToMany`:** use `Set`, not `List` (a `List` causes delete-all-and-reinsert on changes). For anything with extra columns (e.g., enrollment date), **model the join table as its own entity** with two `@ManyToOne`s.
- **`@OneToOne`:** the inverse (non-owning) side can't be truly lazy without bytecode enhancement, because Hibernate must query to know if it's null. Use `@MapsId` to share the PK and avoid extra column/queries.

### Cascade types

| Type | Meaning |
|---|---|
| `PERSIST` | Saving parent saves children |
| `MERGE` | Merging parent merges children |
| `REMOVE` | Deleting parent deletes children |
| `REFRESH`, `DETACH` | Same idea |
| `ALL` | All of the above |

- `orphanRemoval = true`: removing a child from the parent's collection **deletes** it from the DB. Perfect for true composition (Order → OrderLines).
- **Never use `CascadeType.REMOVE` / `ALL` on `@ManyToMany`**: you'd delete shared entities.
- **Never cascade from child to a shared parent.**

---

## 8. Fetching: LAZY, EAGER, and the Famous N+1 Problem

### Lazy vs Eager (restaurant analogy)

- **EAGER:** you order a burger and the waiter brings *the entire menu, drinks, and dessert* every time, "just in case."
- **LAZY:** the waiter brings the burger. If you want a drink, they go get it **then**.

Lazy associations are **proxies**. The real data loads on first access, **if the session is still open**.

### `LazyInitializationException`

> "could not initialize proxy - no Session"

**Cause:** you touched a lazy field after the session closed (usually in the controller or JSON serialization).

**Fixes (best to worst):**
1. Fetch what you need in the query: `JOIN FETCH` or `@EntityGraph`
2. Use **DTO projections** and don't return entities from services
3. Touch the data inside the `@Transactional` method
4. ~~Switch to EAGER~~: cures the symptom and creates performance disease
5. ~~Open Session In View (OSIV)~~: see below

### The N+1 Problem

**Story:** you want the names of 100 customers and their orders. Hibernate runs **1** query for customers, then **100** more (one per customer) to load orders. That's **N+1 = 101 queries**.

```java
List<Customer> customers = repo.findAll();            // 1 query
for (Customer c : customers) {
    c.getOrders().size();                              // +1 query per customer!
}
```

**How to detect:**
- `spring.jpa.show-sql=true` or better, log `org.hibernate.SQL=DEBUG`
- Hibernate statistics (`hibernate.generate_statistics=true`)
- Tools like datasource-proxy, p6spy, or APM traces
- Tests that assert query counts

**Fixes:**

| Fix | How | Caveat |
|---|---|---|
| `JOIN FETCH` | `select c from Customer c join fetch c.orders` | Pagination breaks (see below) |
| `@EntityGraph` | `@EntityGraph(attributePaths = "orders")` on repo method | Same as above |
| Batch fetching | `@BatchSize(size=50)` or `hibernate.default_batch_fetch_size=50` | Turns N+1 into N/50 + 1; **great default** |
| DTO projection | `select new Dto(c.name, o.total) ...` | Not managed, read-only |
| Subselect | `@Fetch(FetchMode.SUBSELECT)` | Loads all at once, can be heavy |

### Two famous `JOIN FETCH` gotchas

1. **Pagination + collection fetch:** Hibernate warns `HHH000104: firstResult/maxResults specified with collection fetch; applying in memory`. It loads **everything** into memory and paginates there. **Fix:** paginate IDs first, then fetch by IDs, or use batch fetching.
2. **`MultipleBagFetchException`:** you can't `JOIN FETCH` two `List` (bag) collections in one query (it makes a Cartesian product). **Fix:** use `Set`, or fetch collections in separate queries.

### Open Session In View (OSIV)

Spring Boot enables `spring.jpa.open-in-view=true` **by default**. It keeps the EntityManager open for the entire web request, so lazy loading "just works" in controllers or serializers.

**Why it's dangerous in production:**
- Holds a **DB connection** for the whole request, including slow downstream calls, so you get **connection pool exhaustion**
- Hides N+1 problems
- Lazy loading happens in the view layer

**Best practice:** `spring.jpa.open-in-view=false` and design your fetches deliberately.

---

## 9. Querying

### 9.1 Spring Data JPA derived queries
```java
List<Employee> findByDepartmentAndSalaryGreaterThan(String dept, BigDecimal min);
Optional<Employee> findByEmail(String email);
boolean existsByEmail(String email);
long countByStatus(Status s);
```

### 9.2 JPQL (queries entities, not tables)
```java
@Query("select e from Employee e where e.department.name = :dept")
List<Employee> findByDept(@Param("dept") String dept);
```

### 9.3 Native SQL
```java
@Query(value = "select * from employees where email = ?1", nativeQuery = true)
```
Use for DB-specific features, window functions, complex reports. Loses portability.

### 9.4 Criteria API / Specifications
Type-safe, dynamic queries (great for search filters with optional fields). Verbose, but Spring Data's `Specification` makes it nicer.

### 9.5 Projections (don't fetch what you don't need)
```java
interface EmployeeSummary { String getName(); String getEmail(); }   // interface projection
record EmployeeDto(String name, String email) {}                      // DTO projection
@Query("select new com.x.EmployeeDto(e.name, e.email) from Employee e")
```

### 9.6 Pagination
```java
Page<Employee> findByStatus(Status s, Pageable p);   // runs data query + count query
Slice<Employee> findByStatus(Status s, Pageable p);  // no count query, cheaper
```
For big tables, prefer **keyset (seek) pagination** over deep `OFFSET`.

### 9.7 Modifying queries
```java
@Modifying(clearAutomatically = true, flushAutomatically = true)
@Query("update Employee e set e.status = :s where e.lastLogin < :d")
int deactivate(...);
```
**Bulk JPQL updates/deletes bypass the persistence context.** Managed entities become stale, hence `clearAutomatically`. They also skip `@Version` and cascades.

---

## 10. Transactions (`@Transactional`)

```java
@Service
public class TransferService {
    @Transactional
    public void transfer(Long from, Long to, BigDecimal amt) { ... }
}
```

### Rules that bite in production

1. **Rollback only on unchecked exceptions by default** (`RuntimeException`, `Error`). Checked exceptions **commit** unless you set `rollbackFor = Exception.class`.
2. **Self-invocation doesn't work.** Calling a `@Transactional` method from another method in the **same class** bypasses the Spring proxy, so no transaction.
3. `@Transactional` on **private** methods is ignored (proxy-based AOP). Use public methods.
4. **Keep transactions short.** Never make remote HTTP calls or do heavy computation inside one. It holds the DB connection and locks.
5. Prefer `@Transactional` on the **service layer**, not the repository or controller.

### Propagation (how transactions nest)

| Type | Behavior |
|---|---|
| `REQUIRED` (default) | Join existing, or create new |
| `REQUIRES_NEW` | Suspend current, start a **new independent** one (e.g., audit logs that must persist even if the main one rolls back) |
| `SUPPORTS` | Join if exists, else run without |
| `MANDATORY` | Must have one, else exception |
| `NOT_SUPPORTED` | Run without a transaction |
| `NEVER` | Fail if one exists |
| `NESTED` | Savepoint inside current (JDBC-savepoint-based) |

### `readOnly = true`
A hint that tells Hibernate to skip dirty checking (flush mode MANUAL), and can route to read replicas. **Use it for all pure reads.**

### Isolation levels (DB-level concept)
`READ_UNCOMMITTED`, `READ_COMMITTED` (default in PostgreSQL/Oracle), `REPEATABLE_READ` (default in MySQL InnoDB), `SERIALIZABLE`. They control dirty reads, non-repeatable reads, and phantom reads.

---

## 11. Concurrency & Locking

### Optimistic locking (`@Version`): "hope for the best, check at the end"
```java
@Version private Long version;
```
Hibernate runs: `UPDATE ... SET ..., version = version+1 WHERE id=? AND version=?`
If 0 rows are updated, someone else changed it, so you get **`OptimisticLockException`** (`ObjectOptimisticLockingFailureException` in Spring).

- Great for read-heavy, low-contention systems (most web apps)
- Handle with retry (`@Retryable`) or user-facing "data changed, reload"

### Pessimistic locking: "lock the door first"
```java
@Lock(LockModeType.PESSIMISTIC_WRITE)   // SELECT ... FOR UPDATE
@Query("select a from Account a where a.id = :id")
Account lockById(Long id);
```
- Use for high contention (inventory, seat booking, balance transfers)
- Risks: **deadlocks**, blocked threads. Always set a lock timeout, and lock rows in a consistent order.
- `SKIP LOCKED` (via hint) is excellent for job-queue-style processing.

---

## 12. Caching

| Cache | Scope | Default | Notes |
|---|---|---|---|
| **L1** | Session / transaction | Always ON | Can't turn off |
| **L2** | SessionFactory (shared across sessions) | **OFF** | Needs provider (Ehcache, Caffeine/JCache, Infinispan, Hazelcast). Annotate entities with `@Cacheable` / `@Cache`. Good for **read-mostly, rarely-changing** reference data (countries, config) |
| **Query cache** | Query results (IDs) | OFF | Invalidated on **any** change to involved tables; often not worth it |

**L2 pitfalls:** stale data in clustered setups, memory use, and cache invalidation complexity. **Don't cache frequently-written entities.**

---

## 13. Inheritance Mapping

| Strategy | Tables | Pros | Cons |
|---|---|---|---|
| `SINGLE_TABLE` (default) | One table + discriminator column | Fastest, no joins | Nullable columns, weaker constraints |
| `JOINED` | Table per class, joined by PK | Normalized, strict constraints | Joins on every query |
| `TABLE_PER_CLASS` | Table per concrete class | No joins for concrete queries | Polymorphic queries use `UNION`; **avoid** |
| `@MappedSuperclass` | Not an entity, just shared fields (id, audit columns) | Simple code reuse | Not queryable polymorphically |

**Rule of thumb:** `SINGLE_TABLE` for performance, `JOINED` for data integrity, and `@MappedSuperclass` for shared base fields.

---

## 14. `equals()` / `hashCode()` / Lombok Traps

- **Don't use Lombok `@Data` on entities.** It generates `equals/hashCode/toString` using *all* fields, including lazy collections, which triggers lazy loading, infinite recursion in bidirectional relationships (`StackOverflowError`), and broken `hashCode` when the ID changes.
- **Safe approach:** base `equals/hashCode` on a **business/natural key**, or use the ID with null-handling, and a constant `hashCode` (e.g., `getClass().hashCode()`) so it's stable across persist.
- **`toString`:** exclude lazy associations.
- Compare using `Hibernate.getClass(o)` or `instanceof`, not `getClass()`, because proxies are subclasses.

---

## 15. Batch Processing & Performance

### Enable JDBC batching
```properties
spring.jpa.properties.hibernate.jdbc.batch_size=50
spring.jpa.properties.hibernate.order_inserts=true
spring.jpa.properties.hibernate.order_updates=true
```
Needs **SEQUENCE** ID generation (IDENTITY disables insert batching).

### Batch loop pattern
```java
for (int i = 0; i < items.size(); i++) {
    em.persist(items.get(i));
    if (i % 50 == 0) {
        em.flush();
        em.clear();     // free memory, avoid OOM
    }
}
```

### Performance checklist
- Make all associations `LAZY`
- Use `readOnly = true` for reads
- Use projections/DTOs for read APIs
- Set `default_batch_fetch_size`
- Use `Slice` instead of `Page` when you don't need totals
- Add the right DB indexes (Hibernate won't think for you)
- Never load huge result sets into memory. Use pagination or streaming (`Stream<T>` in a transaction, or `ScrollableResults`)
- For massive bulk work, use `StatelessSession`, JDBC batch, or native SQL
- Use HikariCP and size the pool sensibly

---

## 16. Production Scenarios (The Stories Interviewers Love)

### Scenario 1: "The API got slow after we added a field."
**Cause:** N+1 from a new lazy collection access (or a serializer touching it).
**Fix:** detect via SQL logs, add `@EntityGraph`/`JOIN FETCH`/batch size, or switch to a DTO projection.

### Scenario 2: "Random `LazyInitializationException` in prod."
**Cause:** accessing lazy data outside the transaction (often via Jackson serializing an entity).
**Fix:** return DTOs, fetch needed data in the service, and don't expose entities.

### Scenario 3: "App freezes under load; DB shows few queries."
**Cause:** connection pool exhaustion. OSIV is holding connections, or a transaction is wrapping a slow external call.
**Fix:** `open-in-view=false`, shorten transactions, and move HTTP calls outside `@Transactional`.

### Scenario 4: "Two users edited the same record; one's changes were silently lost."
**Cause:** no locking (last write wins).
**Fix:** add `@Version` (optimistic), handle the exception with retry or a user message.

### Scenario 5: "Job inserting 1M rows takes hours / OOMs."
**Cause:** persistence context grows unbounded, no batching, IDENTITY IDs.
**Fix:** `batch_size`, SEQUENCE ids, `flush()` + `clear()` every N rows, or `StatelessSession`/JDBC batch.

### Scenario 6: "We updated rows with a bulk query, but the app still shows old data."
**Cause:** bulk JPQL bypasses the persistence context, so L1 is stale.
**Fix:** `@Modifying(clearAutomatically = true)` or `em.clear()`.

### Scenario 7: "Deadlocks in production."
**Cause:** transactions locking rows in different orders (pessimistic locks or concurrent updates).
**Fix:** consistent lock ordering, shorter transactions, lock timeouts, retries on deadlock.

### Scenario 8: "Someone deployed with `ddl-auto=update`/`create` and wiped/altered prod."
**Fix:** In prod use `ddl-auto=validate` or `none`, and manage schema with **Flyway/Liquibase**.

### Scenario 9: "Pagination endpoint uses tons of memory."
**Cause:** `JOIN FETCH` on a collection + `Pageable` → in-memory pagination (HHH000104).
**Fix:** two-step query (page IDs, then fetch), or batch fetching.

### Scenario 10: "`@Transactional` seems to do nothing."
**Cause:** self-invocation, private method, wrong `@Transactional` import, or checked exception not triggering rollback.

---

## 17. Best Cases vs Worst Cases

| ✅ Best practice | ❌ Worst practice |
|---|---|
| All associations `LAZY`, fetch on demand | `EAGER` everywhere |
| DTO projections for reads | Returning entities from REST controllers |
| `@Version` for optimistic locking | No concurrency control |
| `SEQUENCE` + batching for bulk writes | `IDENTITY` + giant loops |
| Short, service-level transactions | Remote calls inside `@Transactional` |
| `open-in-view=false` | OSIV on, hiding lazy-load queries |
| Flyway/Liquibase for schema | `ddl-auto=update` in prod |
| `Set` for `@ManyToMany`, helper methods for sync | `List` + one-sided updates |
| `@Enumerated(STRING)` | `ORDINAL` enums |
| `equals/hashCode` on business key | Lombok `@Data` on entities |
| Log SQL and monitor query counts | "Hibernate will handle it" |
| Testcontainers with real DB for tests | Only H2 (hides dialect differences) |

---

## 18. Hibernate 6 / Spring Boot 3 Quick Notes

- Package moved from `javax.persistence` to **`jakarta.persistence`**
- Improved SQL generation, a new query engine (SQM), and stricter HQL/JPQL validation
- Default sequence naming and allocation behavior changed in some cases (watch when upgrading schemas)
- Java 17+ baseline with Spring Boot 3
- Better support for records as DTO projections

---

## 19. Rapid-Fire Q&A (Say These Out Loud)

**Q: JPA vs Hibernate?**
JPA is the specification, Hibernate is an implementation with extra features.

**Q: `get()`/`find()` vs `getReference()`?**
`find` hits the DB immediately and returns null if missing. `getReference` returns a lazy proxy and throws `EntityNotFoundException` on access if missing.

**Q: `persist` vs `merge` vs `save`?**
`persist` makes a new entity managed (same instance). `merge` copies state onto a managed instance and returns it. Spring Data `save` chooses between them based on whether the entity is new.

**Q: What is the persistence context?**
The set of managed entities in a session. It provides L1 caching, dirty checking, and write-behind.

**Q: What is dirty checking?**
At flush, Hibernate compares each managed entity against its loaded snapshot and issues `UPDATE`s for changes automatically.

**Q: What does `flush()` do vs commit?**
Flush sends SQL to the DB within the transaction. Commit makes it permanent.

**Q: LAZY vs EAGER defaults?**
To-one is EAGER, to-many is LAZY. Prefer LAZY everywhere.

**Q: What is the N+1 problem and fixes?**
One query for the parent plus N for the children. Fix with `JOIN FETCH`, `@EntityGraph`, batch fetching, or DTO projections.

**Q: What does `mappedBy` mean?**
It marks the inverse side. The other side owns the FK.

**Q: Why use helper methods in bidirectional relationships?**
To keep both sides of the in-memory graph consistent, since only the owning side writes the FK.

**Q: How do you handle concurrent updates?**
Optimistic locking with `@Version` for most cases, and pessimistic locking (`FOR UPDATE`) for high contention.

**Q: What are the cache levels?**
L1 (session, always on), L2 (shared, opt-in, with a provider), and the query cache (opt-in, rarely worth it).

**Q: Why avoid `CascadeType.ALL` blindly?**
It can delete or merge shared entities unexpectedly. Use it only for true parent-owns-child relationships.

**Q: `orphanRemoval` vs `CascadeType.REMOVE`?**
`REMOVE` deletes children when the parent is deleted. `orphanRemoval` also deletes children removed from the parent's collection.

**Q: What causes `LazyInitializationException`?**
Accessing an uninitialized lazy association after the session has closed.

**Q: Why is `IDENTITY` bad for batching?**
Hibernate needs the generated ID immediately after each insert, so it can't defer and batch them.

**Q: How do you do pagination efficiently?**
`Pageable` with `Slice` or `Page`. Avoid collection `JOIN FETCH` with pagination, and use keyset pagination for deep pages.

**Q: What is `@Transactional(readOnly = true)` doing?**
Marks the transaction as read-only, so Hibernate skips dirty checking (flush mode MANUAL) and the DB or driver may optimize.

**Q: Why might `@Transactional` not work?**
Self-invocation, private/final methods, wrong proxy, or a checked exception without `rollbackFor`.

**Q: How would you debug slow JPA?**
Enable SQL logging and statistics, find N+1 and slow queries with `EXPLAIN`, check indexes, check fetch strategies, and check transaction and connection-pool metrics.

**Q: When would you skip JPA?**
For complex reporting and analytics, extreme bulk operations, or heavily SQL-centric logic. Use jOOQ, JdbcTemplate, or native queries instead.

**Q: Entity vs DTO?**
Entities are persistence models (managed, with lifecycle). DTOs are API/read models (plain and detached). Don't leak entities through your API.

---

## 20. The One-Page Memory Trick 🧠

> **"L.E.A.R.N. Hibernate"**

- **L**azy by default (fix N+1 with fetch joins, graphs, batch size)
- **E**ntities ≠ DTOs (don't expose entities)
- **A**ttach/detach lifecycle (transient → managed → detached → removed)
- **R**ead-only transactions + short transactions (keep connections free)
- **N**o `ddl-auto` in prod (use Flyway/Liquibase) and **n**ever trust magic (log your SQL)

---
