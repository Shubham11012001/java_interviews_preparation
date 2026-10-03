# Creational Design Patterns: The Interview Cheatsheet

*A Medium-style guide for Cloud Engineers who get asked "so... tell me about design patterns."*

---

## 🧒 The 10-Year-Old Explanation

Imagine you run a **toy factory**. You have to decide: *how do I make toys?*

- Do I make **just one** toy and let everyone share it? → **Singleton**
- Do I have a **machine** that makes different toys depending on what you ask for? → **Factory Method**
- Do I have a **whole factory** that makes matching sets (a red car + red truck + red train)? → **Abstract Factory**
- Do I build a toy **step by step** (add wheels, add paint, add stickers)? → **Builder**
- Do I just **photocopy** an existing toy instead of building from scratch? → **Prototype**

**Creational patterns = different smart ways of creating objects**, so your code doesn't become a mess of `new` keywords everywhere.

---

## 🎯 What Are Creational Design Patterns? (Interview-Ready Definition)

> *"Creational design patterns deal with **object creation mechanisms**. They abstract the instantiation process so the system is independent of how its objects are created, composed, and represented. They give flexibility in **what** gets created, **who** creates it, **how** it's created, and **when**."*

**Say this in the interview:** *"They decouple the code that uses an object from the code that creates it."*

### The Big 5 (Gang of Four)

| Pattern | One-liner | Kid analogy |
|---|---|---|
| **Singleton** | Only ONE instance, globally accessible | One principal in a school |
| **Factory Method** | Subclass decides which object to create | Pizza shop: you say "veg", they decide the recipe |
| **Abstract Factory** | Create *families* of related objects | Furniture store: a Modern set or a Victorian set |
| **Builder** | Construct complex objects step by step | Building a burger layer by layer |
| **Prototype** | Clone an existing object | Photocopier |

---

## 🔒 1. Singleton

### What
Guarantees a class has **exactly one instance** and provides a **global access point** to it.

### Why
Some things should exist only once: a config manager, a logger, a connection pool. Two of them would cause conflicts or waste resources.

### Java Implementation (Thread-Safe, Interview Favorite)

```java
public class ConfigManager {
    // volatile prevents instruction reordering issues
    private static volatile ConfigManager instance;

    private ConfigManager() {
        // private constructor: nobody can call "new"
    }

    public static ConfigManager getInstance() {
        if (instance == null) {                        // 1st check (no lock, fast)
            synchronized (ConfigManager.class) {
                if (instance == null) {                // 2nd check (with lock)
                    instance = new ConfigManager();
                }
            }
        }
        return instance;
    }
}
```
This is called **Double-Checked Locking**.

### Better Alternatives (Mention These to Impress)

```java
// Option A: Bill Pugh (lazy + thread-safe, no synchronization cost)
public class Logger {
    private Logger() {}
    private static class Holder {
        private static final Logger INSTANCE = new Logger();
    }
    public static Logger getInstance() { return Holder.INSTANCE; }
}

// Option B: Enum Singleton (Joshua Bloch's recommendation)
public enum AppConfig {
    INSTANCE;
    public void doSomething() { }
}
```
**Why enum is the best:** it's inherently thread-safe, and it's **immune to reflection and serialization attacks** that can break the other versions.

### Production Scenarios
- Logger (Log4j/SLF4J loggers)
- Database connection pool manager (HikariCP)
- Application configuration / feature flags cache
- Cache manager

### ✅ Best Case
Shared, stateless (or carefully managed) resource that is expensive to create.

### ❌ Worst Case / Pitfalls
- **Hidden global state** makes unit testing painful (hard to mock)
- **Tight coupling**: everything depends on the global instance
- **Concurrency bugs** if mutable state is shared
- **Multiple instances can sneak in** via: multiple classloaders, reflection, serialization/deserialization, or **multiple JVMs/containers**

### ☁️ Cloud Engineer Angle (Your Superpower)
> *"A Singleton is only singleton **per JVM**. In Kubernetes with 5 pods, I have 5 'singletons'. If I need a truly global one (say, a scheduler that must run once), I use **distributed locks** (Redis/ZooKeeper), leader election, or Cloud Scheduler/Pub/Sub, not the Singleton pattern."*

In **Spring**, beans are singleton-scoped by default. That's the *container-managed* version of the pattern (better than hand-written).

---

## 🏭 2. Factory Method

### What
Defines an **interface for creating an object**, but lets **subclasses (or a method) decide which class to instantiate**.

### Why
So client code says **"give me a Notification"** instead of **"give me an EmailNotification"**. Adding a new type doesn't break existing code (**Open/Closed Principle**).

### The Problem It Solves (The Ugly Way)

```java
// ❌ BAD: client is tied to concrete classes, has if-else everywhere
if (type.equals("EMAIL")) { n = new EmailNotification(); }
else if (type.equals("SMS")) { n = new SmsNotification(); }
else if (type.equals("PUSH")) { n = new PushNotification(); }
```

### The Clean Way

```java
// Product interface
interface Notification {
    void send(String message);
}

// Concrete products
class EmailNotification implements Notification {
    public void send(String msg) { System.out.println("Email: " + msg); }
}
class SmsNotification implements Notification {
    public void send(String msg) { System.out.println("SMS: " + msg); }
}

// Factory
class NotificationFactory {
    public static Notification create(String type) {
        return switch (type.toUpperCase()) {
            case "EMAIL" -> new EmailNotification();
            case "SMS"   -> new SmsNotification();
            default -> throw new IllegalArgumentException("Unknown type: " + type);
        };
    }
}

// Client
Notification n = NotificationFactory.create("EMAIL");
n.send("Hello!");
```

> 📝 **Interview note:** The above is technically a **Simple Factory** (a very popular idiom, but not an official GoF pattern). The *true* **Factory Method** uses **inheritance**: an abstract creator declares `createProduct()` and subclasses override it.

```java
abstract class Dialog {
    abstract Button createButton();      // <-- the factory method
    void render() { createButton().draw(); }
}
class WindowsDialog extends Dialog { Button createButton() { return new WindowsButton(); } }
class WebDialog     extends Dialog { Button createButton() { return new HtmlButton(); } }
```

### Production Scenarios
- `Calendar.getInstance()`, `NumberFormat.getInstance()`, `List.of()`, `Optional.of()` in Java
- Payment gateway selection (Razorpay / Stripe / PayPal)
- Cloud storage clients (S3 / GCS / Azure Blob behind one interface)
- Parsers (JSON/XML/CSV), loggers, DB drivers (`DriverManager.getConnection`)

### ✅ Best Case
You have a **family of related types** and the exact type is known only at runtime (config, user input, env).

### ❌ Worst Case
- Only **one** implementation exists and likely always will → over-engineering
- Class explosion (one creator subclass per product)

---

## 🏢 3. Abstract Factory

### What
A **factory of factories**. Provides an interface for creating **families of related objects** without specifying their concrete classes.

### Why
When products must be **used together consistently**. Mixing a "Windows button" with a "Mac checkbox" would be a bug.

### Kid Analogy
IKEA has a **Modern collection** and a **Classic collection**. Pick one and you get a matching chair + table + sofa. You never accidentally get a modern chair with a classic table.

### Cloud Example (Great for Your Profile!)

```java
// Abstract products
interface Storage { void upload(String file); }
interface Messaging { void publish(String msg); }

// Abstract factory
interface CloudFactory {
    Storage createStorage();
    Messaging createMessaging();
}

// GCP family
class GcpFactory implements CloudFactory {
    public Storage createStorage()     { return new GcsStorage(); }       // Cloud Storage
    public Messaging createMessaging() { return new PubSubMessaging(); }  // Pub/Sub
}

// AWS family
class AwsFactory implements CloudFactory {
    public Storage createStorage()     { return new S3Storage(); }
    public Messaging createMessaging() { return new SnsMessaging(); }
}

// Client code is cloud-agnostic
CloudFactory cloud = new GcpFactory();   // swap to AwsFactory = zero other changes
cloud.createStorage().upload("report.pdf");
cloud.createMessaging().publish("done");
```
**This is exactly how multi-cloud / cloud-portability abstractions work.**

### Production Scenarios
- Multi-cloud abstraction layers
- UI toolkits (light theme / dark theme component families)
- Database access layers (MySQL family: Connection + Command + Reader vs. PostgreSQL family)
- Cross-platform apps

### ✅ Best Case
Need **guaranteed compatibility** across a set of related objects, and need to swap the whole family easily.

### ❌ Worst Case
- Adding a **new product type** (e.g., `createDatabase()`) forces changes in **every** factory
- Lots of interfaces/classes → complexity for small apps

### 🆚 Factory Method vs Abstract Factory (Top Interview Question)

| | Factory Method | Abstract Factory |
|---|---|---|
| Creates | **One** product | A **family** of products |
| Mechanism | Inheritance (subclass overrides) | Composition (factory object passed in) |
| Complexity | Low | Higher |
| Adding new product *type* | Easy | Hard |
| Adding new *variant/family* | Easy | Easy |

**One-liner:** *"Abstract Factory is often implemented using multiple Factory Methods."*

---

## 🧱 4. Builder

### What
Separates **construction of a complex object** from its representation, letting you build it **step by step**.

### Why
Solves the **Telescoping Constructor** problem:

```java
// ❌ Nightmare: what does each argument mean? Which are optional?
new Pizza("large", true, false, true, false, null, true);
```

### The Clean Way

```java
public class User {
    private final String name;       // required
    private final String email;      // required
    private final int age;           // optional
    private final String phone;      // optional

    private User(Builder b) {
        this.name = b.name;
        this.email = b.email;
        this.age = b.age;
        this.phone = b.phone;
    }

    public static class Builder {
        private final String name;
        private final String email;
        private int age;
        private String phone;

        public Builder(String name, String email) {   // required fields here
            this.name = name;
            this.email = email;
        }
        public Builder age(int age)       { this.age = age; return this; }
        public Builder phone(String p)    { this.phone = p; return this; }
        public User build()               { return new User(this); }
    }
}

// Usage: reads like English
User u = new User.Builder("Shubham", "s@example.com")
                 .age(28)
                 .phone("99999")
                 .build();
```

### Bonus: Lombok (Production Reality)
```java
@Builder
public class User { String name; String email; int age; }
User u = User.builder().name("Shubham").email("s@x.com").build();
```
As a Java backend dev, mention that you've used `@Builder` in real projects.

### Production Scenarios
- `StringBuilder`, `Stream.Builder`, `HttpRequest.newBuilder()`, `Locale.Builder`
- **Google Cloud client libraries**: `StorageOptions.newBuilder().setProjectId(..).build()`, `PubsubMessage.newBuilder()`
- Spring's `UriComponentsBuilder`, `ResponseEntity.ok().header(...).body(...)`
- Complex DTOs/request objects, SQL query builders, test data builders
- Immutable objects with many fields

### ✅ Best Case
- Many constructor parameters (**4+**), especially optional ones
- Need **immutable** objects
- Want **readable** object construction

### ❌ Worst Case
- Simple object with 2 to 3 fields → pure boilerplate
- Extra code (mitigated by Lombok)
- Object may be built in an **incomplete/invalid state** if validation is missing in `build()`

### 🆚 Builder vs Factory
- **Factory**: *which* object (one-shot, returns immediately)
- **Builder**: *how* to assemble a **complex** object (multi-step)

---

## 🐑 5. Prototype

### What
Create new objects by **copying (cloning) an existing instance** instead of building from scratch.

### Why
When object creation is **expensive** (DB calls, heavy computation, network) and you need many similar objects.

### Kid Analogy
Need 30 copies of a worksheet? **Photocopy** one instead of rewriting it 30 times.

### Implementation

```java
class Report implements Cloneable {
    private String title;
    private List<String> data;

    public Report(String title, List<String> data) {   // imagine this is expensive
        this.title = title;
        this.data = data;
    }

    @Override
    public Report clone() {
        try {
            Report copy = (Report) super.clone();
            copy.data = new ArrayList<>(this.data);    // DEEP copy! Critical!
            return copy;
        } catch (CloneNotSupportedException e) {
            throw new AssertionError();
        }
    }
}
```

### ⚠️ Shallow vs Deep Copy (Guaranteed Interview Question)

| | Shallow Copy | Deep Copy |
|---|---|---|
| What's copied | Object + **references** | Object + **everything it points to** |
| Danger | Original and clone **share** nested objects, so a change in one affects the other 😱 | Fully independent |
| Cost | Cheap | Costlier |

### Better Than `Cloneable` (Say This!)
Java's `Cloneable` is considered **broken/awkward** (Effective Java, Item 13). Prefer:
- **Copy constructor**: `new Report(otherReport)`
- **Static copy factory**: `Report.copyOf(other)`
- **Serialization-based deep copy** (slower)
- Libraries like **MapStruct** or `BeanUtils.copyProperties`

### Production Scenarios
- Game objects (spawning 1000 similar enemies)
- Pre-configured **templates** (email templates, document templates, default configs)
- Caching a baseline object, then cloning and tweaking
- JavaScript's entire inheritance model is prototype-based
- Spring `@Scope("prototype")`: a *new bean instance per request* (a related but different idea)

### ✅ Best Case
Expensive creation + many near-identical objects.

### ❌ Worst Case
- Circular references make deep cloning tricky
- Shallow-copy bugs (silent data corruption)
- `Cloneable` quirks (bypasses constructors, `final` fields trouble)

---

## 🧠 The Master Comparison Table (Memorize This)

| Pattern | Problem it solves | Key idea | Real Java example | Watch out for |
|---|---|---|---|---|
| **Singleton** | Need exactly one instance | Private constructor + static accessor | Spring beans, `Runtime.getRuntime()` | Testing, concurrency, multi-JVM |
| **Factory Method** | Hide *which* class is created | Subclass/method decides | `Calendar.getInstance()` | Over-engineering |
| **Abstract Factory** | Create compatible *families* | Factory of factories | JDBC drivers, multi-cloud SDKs | Hard to add new product types |
| **Builder** | Complex object, many params | Step-by-step + `build()` | `StringBuilder`, GCP client builders | Boilerplate for simple objects |
| **Prototype** | Expensive creation | Clone existing | `Object.clone()`, Spring prototype scope | Shallow vs deep copy |

---

## 🎤 Decision Guide: "Which Pattern Would You Use When...?"

| Scenario | Answer |
|---|---|
| "I need a single shared DB connection pool" | **Singleton** (or Spring singleton bean) |
| "Pick Razorpay vs Stripe based on country" | **Factory Method** |
| "Support both GCP and AWS with the same code" | **Abstract Factory** |
| "Object has 10 fields, 6 optional" | **Builder** |
| "Creating objects is slow; I need 1000 similar ones" | **Prototype** |
| "I want to avoid `if-else` with `new` everywhere" | **Factory** |
| "Need immutable object + readable code" | **Builder** |

---

## 🔥 Frequently Asked Interview Questions (With Crisp Answers)

**Q1. What are creational design patterns?**
Patterns that abstract object creation to make systems independent of how objects are created, composed, and represented.

**Q2. Why not just use `new`?**
`new` tightly couples code to **concrete classes**. Creational patterns add **flexibility, testability, and loose coupling**.

**Q3. How do you make a Singleton thread-safe?**
Double-checked locking with `volatile`, Bill Pugh holder idiom, or enum singleton.

**Q4. How can Singleton be broken? How to prevent it?**
- **Reflection** → throw an exception in the constructor if the instance exists
- **Serialization** → implement `readResolve()`
- **Cloning** → override `clone()` to throw an exception
- **Multiple classloaders**
- **Best fix:** use an **enum singleton**

**Q5. Why is `volatile` needed in double-checked locking?**
Without it, the JVM may reorder instructions so another thread sees a **partially constructed** object.

**Q6. Is Spring's singleton the same as GoF Singleton?**
No. Spring's is **one instance per container** (ApplicationContext), not one per JVM or classloader. It's container-managed, with no private constructor tricks, and is far easier to test.

**Q7. Singleton drawbacks?**
Global state, hard to test/mock, violates Single Responsibility Principle, hidden dependencies. Many call it an **anti-pattern** when overused.

**Q8. Factory Method vs Abstract Factory?**
One product (inheritance) vs a family of products (composition).

**Q9. When would you NOT use Builder?**
For objects with few fields. Use a plain constructor or a Java `record`.

**Q10. Shallow vs deep copy?**
Shallow shares nested references; deep duplicates everything.

**Q11. Builder vs Telescoping constructors vs JavaBeans setters?**
Telescoping = unreadable. Setters = object can be in an inconsistent state and is **mutable**. Builder = readable **and** can produce immutable objects.

**Q12. Where does Java itself use these patterns?**
- Singleton → `Runtime.getRuntime()`
- Factory → `Calendar.getInstance()`, `List.of()`
- Builder → `StringBuilder`, `HttpClient.newBuilder()`
- Prototype → `Object.clone()`
- Abstract Factory → `DocumentBuilderFactory`

**Q13. Can Dependency Injection replace these patterns?**
Largely yes, for Singleton and Factory. A DI container (Spring) **is essentially a giant factory** that manages lifecycle and scope. But Builder and Prototype still have their uses.

**Q14. How does the Enum Singleton protect against reflection?**
The JVM forbids reflective instantiation of enums (`Cannot reflectively create enum objects`).

**Q15. Which creational pattern would you use for immutable objects?**
**Builder** (or a `record` for simple cases).

---

## ☁️ Connecting to Your Cloud Engineer Story

Interviewers love when you connect theory to your real experience. Use these hooks:

- **Builder:** *"GCP client libraries use Builder everywhere, like `StorageOptions.newBuilder()` and `PubsubMessage.newBuilder()`, so I use it daily."*
- **Singleton:** *"I reuse a single expensive client (e.g., a `Storage` or `Pub/Sub Publisher`) rather than creating one per request. Clients hold connection pools and are thread-safe, so a singleton (or Spring bean) is ideal. But in GKE with multiple pods, it's one per pod."*
- **Abstract Factory:** *"For cloud portability, I'd define interfaces for storage/messaging and provide GCP/AWS factories behind them."*
- **Factory:** *"Choosing a strategy by config or environment variable at startup, such as picking dev vs prod implementations."*
- **Prototype:** *"Spring prototype scope for stateful, per-request beans; also cloning pre-configured templates (e.g., baseline Terraform/config objects)."*

---

## ⚖️ Quick Pros & Cons Summary

| Pattern | ✅ Pros | ❌ Cons |
|---|---|---|
| Singleton | Saves memory, controlled access | Global state, hard to test |
| Factory Method | Loose coupling, Open/Closed | Extra classes |
| Abstract Factory | Consistent families, easy to swap | Complex, rigid for new product types |
| Builder | Readable, immutable, flexible | Verbose without Lombok |
| Prototype | Fast cloning, avoids costly init | Deep-copy complexity |

---

## 🧩 SOLID Principles Connection (Bonus Points)

- **Factory / Abstract Factory** → **Open/Closed Principle** + **Dependency Inversion** (depend on abstractions, not concretes)
- **Builder** → **Single Responsibility** (construction logic separated from the object)
- **Singleton** → often **violates** SRP (manages its own lifecycle *and* does its job)

---

## 🚀 Last-Minute Memory Tricks

> **"S-F-A-B-P"**: **S**ingle one, **F**actory picks, **A**bstract families, **B**uilder step-by-step, **P**rototype photocopies.

Or the **restaurant story**:
- 🔒 **Singleton** → *one head chef* in the kitchen
- 🏭 **Factory** → *waiter* takes your order; the kitchen decides how to make it
- 🏢 **Abstract Factory** → *combo meal* where everything matches (burger + fries + drink in the same theme)
- 🧱 **Builder** → *Subway sandwich*: choose bread, then filling, then sauce
- 🐑 **Prototype** → *"I'll have what she's having"* (copy an existing order)

---

## ✅ Final 60-Second Revision Checklist

- [ ] Define creational patterns in one sentence
- [ ] Name all 5 and give a one-line analogy for each
- [ ] Write a thread-safe Singleton (double-checked locking + enum)
- [ ] Explain how Singleton can be broken and how to prevent it
- [ ] Explain Factory Method vs Abstract Factory clearly
- [ ] Explain Telescoping Constructor problem → Builder
- [ ] Explain shallow vs deep copy
- [ ] Give a real Java/GCP example for each pattern
- [ ] State one worst case for each
- [ ] Mention Spring's singleton ≠ GoF Singleton
- [ ] Mention Singleton is *per JVM / per pod* in cloud setups
