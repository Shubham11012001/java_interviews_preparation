# Behavioral Design Patterns: The Complete Interview Cheatsheet

*A Technical Architect's guide, explained so simply a 10-year-old could follow it.*

---

## 1. What Are Behavioral Design Patterns?

**Simple explanation:** Imagine a classroom. *Creational* patterns decide **how students are born** (objects created). *Structural* patterns decide **how desks are arranged** (objects composed). *Behavioral* patterns decide **how students talk to each other and who does what** (objects communicate and share responsibility).

**Technical definition:** Behavioral patterns deal with **object interaction, communication, and the assignment of responsibilities** between objects. They make the *flow of control* flexible and loosely coupled.

**Core goals:**
- Reduce tight coupling between objects
- Make algorithms swappable at runtime
- Distribute responsibility cleanly
- Make communication flows easy to change

**The 11 GoF Behavioral Patterns:**

| # | Pattern | One-line kid explanation |
|---|---------|--------------------------|
| 1 | Strategy | Choose your way to school: bus, bike, or walk |
| 2 | Observer | YouTube subscribers get notified of new videos |
| 3 | Command | A restaurant order slip: waiter writes it, chef cooks it later |
| 4 | State | A traffic light behaves differently depending on its color |
| 5 | Template Method | A recipe skeleton: steps fixed, ingredients vary |
| 6 | Chain of Responsibility | Passing a complaint up: teacher, principal, board |
| 7 | Mediator | An air traffic controller so planes don't talk to each other directly |
| 8 | Iterator | Flipping through a photo album page by page |
| 9 | Memento | A video game save point |
| 10 | Visitor | A tax inspector visiting different kinds of shops |
| 11 | Interpreter | A translator that understands a small language |

---

## 2. Quick Decision Table (Memorize This!)

| Problem you face | Use this pattern |
|---|---|
| Many `if/else` choosing between algorithms | **Strategy** |
| One change must notify many others | **Observer** |
| Need undo/redo, queues, or logging of actions | **Command** |
| Object's behavior changes with its internal status | **State** |
| Same algorithm skeleton, different steps | **Template Method** |
| Request may be handled by one of many handlers | **Chain of Responsibility** |
| Many objects talk in a spaghetti mesh | **Mediator** |
| Traverse a collection without exposing internals | **Iterator** |
| Save and restore previous state | **Memento** |
| Add operations to a class hierarchy without modifying it | **Visitor** |
| Evaluate sentences of a small language/grammar | **Interpreter** |

---

## 3. Strategy Pattern

**Kid explanation:** You want to go to school. You can walk, cycle, or take a bus. The *goal* is the same (reach school), the *way* is swappable.

**Definition:** Define a family of algorithms, encapsulate each one, and make them interchangeable at runtime.

**When:** You have multiple ways to do one thing and want to switch between them.
**Why:** Eliminates giant `if/else` or `switch` blocks; follows the Open/Closed Principle.

```java
interface PaymentStrategy {
    void pay(double amount);
}

class UpiPayment implements PaymentStrategy {
    public void pay(double amount) { System.out.println("Paid " + amount + " via UPI"); }
}

class CardPayment implements PaymentStrategy {
    public void pay(double amount) { System.out.println("Paid " + amount + " via Card"); }
}

class Checkout {
    private PaymentStrategy strategy;
    Checkout(PaymentStrategy strategy) { this.strategy = strategy; }
    void setStrategy(PaymentStrategy s) { this.strategy = s; }
    void process(double amount) { strategy.pay(amount); }
}

// Usage
new Checkout(new UpiPayment()).process(500);
```

**Production scenarios:**
- Payment gateways (UPI, card, net banking, wallet)
- Credit scoring models (different risk algorithms per customer segment)
- Sorting/compression/pricing/discount engines
- Spring: `Comparator`, `AuthenticationProvider`, `ResourceLoader`

**Best case:** Many interchangeable algorithms, chosen at runtime or via config.
**Worst case:** Only 2 stable algorithms that never change. Then it is over-engineering. Also clients must *know* the strategies to pick one.

---

## 4. Observer Pattern

**Kid explanation:** You subscribe to a YouTuber. When they upload, **everyone subscribed gets a bell notification**. The YouTuber doesn't know or care who exactly is watching.

**Definition:** One-to-many dependency. When one object (Subject) changes state, all dependents (Observers) are notified automatically.

**When:** A change in one object must trigger updates in others, and you don't know how many.
**Why:** Loose coupling between publisher and subscribers.

```java
interface Observer { void update(String news); }

class Subject {
    private List<Observer> observers = new ArrayList<>();
    void subscribe(Observer o) { observers.add(o); }
    void unsubscribe(Observer o) { observers.remove(o); }
    void publish(String news) { observers.forEach(o -> o.update(news)); }
}

class EmailService implements Observer {
    public void update(String news) { System.out.println("Email sent: " + news); }
}
class SmsService implements Observer {
    public void update(String news) { System.out.println("SMS sent: " + news); }
}
```

**Production scenarios:**
- Event-driven systems: Kafka/RabbitMQ (distributed version of Observer)
- Spring `ApplicationEvent` / `@EventListener`
- UI listeners, stock price tickers, notification systems
- Order placed → send email, update inventory, trigger invoice

**Best case:** Decoupled event broadcasting with dynamic subscribers.
**Worst case:**
- **Memory leaks** (the "lapsed listener" problem: forgot to unsubscribe)
- Unpredictable notification order
- Cascading updates that are hard to debug
- Performance hit with thousands of observers if notified synchronously

**Architect tip:** Observer is in-process; Pub/Sub (Kafka) is the distributed cousin. Know the difference.

---

## 5. Command Pattern

**Kid explanation:** At a restaurant, you don't walk into the kitchen. The waiter writes your order on a **slip of paper (the command)**. The chef reads it later. The slip can be saved, canceled, or repeated.

**Definition:** Encapsulate a request as an object, so you can parameterize, queue, log, and undo operations.

**When:** You need undo/redo, job queues, macros, or to decouple the *sender* from the *executor*.

```java
interface Command { void execute(); void undo(); }

class Light { void on() { System.out.println("ON"); } void off() { System.out.println("OFF"); } }

class LightOnCommand implements Command {
    private Light light;
    LightOnCommand(Light l) { this.light = l; }
    public void execute() { light.on(); }
    public void undo() { light.off(); }
}

class RemoteControl {
    private Deque<Command> history = new ArrayDeque<>();
    void press(Command c) { c.execute(); history.push(c); }
    void undoLast() { if (!history.isEmpty()) history.pop().undo(); }
}
```

**Production scenarios:**
- Undo/redo in editors
- Task queues and job schedulers (`Runnable` is a Command!)
- Transaction logging, event sourcing, CQRS commands
- Retry mechanisms and macro recording

**Best case:** Need to queue, schedule, log, retry, or undo operations.
**Worst case:** Simple direct calls. Creating a class per action causes **class explosion**.

---

## 6. State Pattern

**Kid explanation:** A traffic light acts differently depending on whether it's red, yellow, or green. Same light, different behavior based on its *current mood*.

**Definition:** Allow an object to change its behavior when its internal state changes. It looks like the object changed its class.

**When:** An object has many states and behavior depends on the state (replaces giant `switch(state)`).

```java
interface OrderState { void next(Order o); }

class Placed implements OrderState {
    public void next(Order o) { System.out.println("Shipping..."); o.setState(new Shipped()); }
}
class Shipped implements OrderState {
    public void next(Order o) { System.out.println("Delivered!"); o.setState(new Delivered()); }
}
class Delivered implements OrderState {
    public void next(Order o) { System.out.println("Already delivered."); }
}

class Order {
    private OrderState state = new Placed();
    void setState(OrderState s) { this.state = s; }
    void next() { state.next(this); }
}
```

**Production scenarios:**
- Order lifecycle (Placed → Paid → Shipped → Delivered → Returned)
- Loan/credit application workflow (Submitted → Under Review → Approved/Rejected)
- Payment status, TCP connection states, vending machines, document approval
- Spring State Machine

**State vs Strategy (very common interview question):**

| | Strategy | State |
|---|---|---|
| Who decides the change? | **Client** picks it | **Object itself** transitions |
| Awareness | Strategies don't know each other | States often know the next state |
| Purpose | Swap *algorithms* | Model *lifecycle* |

**Best case:** Complex lifecycle with many transitions.
**Worst case:** Only 2 to 3 simple states, where an enum plus switch is simpler.

---

## 7. Template Method Pattern

**Kid explanation:** A recipe for making sandwiches: *1. Take bread, 2. Add filling, 3. Add sauce, 4. Serve.* The steps never change, but the *filling* differs (veg or chicken).

**Definition:** Define the skeleton of an algorithm in a base class, letting subclasses override specific steps without changing the structure.

```java
abstract class DataProcessor {
    // Template method: final so subclasses can't break the flow
    public final void process() {
        readData();
        transform();
        saveData();
    }
    abstract void readData();
    abstract void transform();
    void saveData() { System.out.println("Saved to DB"); } // default step
}

class CsvProcessor extends DataProcessor {
    void readData() { System.out.println("Reading CSV"); }
    void transform() { System.out.println("Transforming CSV"); }
}
```

**Production scenarios:**
- Spring `JdbcTemplate`, `RestTemplate`, `HttpServlet` (`doGet`/`doPost`)
- ETL pipelines, report generation, test frameworks (`setUp → test → tearDown`)
- Batch jobs (read → process → write)

**Best case:** Many classes share the same workflow but differ in details.
**Worst case:** Inheritance-based, so it creates **tight coupling** to the parent class. Prefer Strategy (composition) if flexibility matters more.

---

## 8. Chain of Responsibility

**Kid explanation:** You complain about a broken toy. The teacher says "not me, ask the principal." The principal says "not me, ask the manager." The request **travels along a chain** until someone handles it.

**Definition:** Pass a request along a chain of handlers. Each decides to process it or pass it on.

```java
abstract class Handler {
    protected Handler next;
    Handler setNext(Handler n) { this.next = n; return n; }
    abstract void handle(int amount);
}

class Manager extends Handler {
    void handle(int amt) {
        if (amt <= 10000) System.out.println("Manager approved");
        else if (next != null) next.handle(amt);
    }
}
class Director extends Handler {
    void handle(int amt) {
        if (amt <= 100000) System.out.println("Director approved");
        else if (next != null) next.handle(amt);
    }
}
```

**Production scenarios:**
- Servlet Filters, Spring Security filter chain, Spring Interceptors
- Middleware (Express.js, API gateways: auth → rate limit → logging → routing)
- Approval workflows, logging levels, validation pipelines, exception handlers

**Best case:** Multiple potential handlers, and the handler isn't known upfront.
**Worst case:**
- Request may go **unhandled** if the chain has no default
- Hard to debug long chains
- Performance cost with very long chains

---

## 9. Mediator Pattern

**Kid explanation:** Planes don't talk to each other directly. They all talk to the **air traffic controller**, who coordinates everyone. Without it, it would be chaos.

**Definition:** Centralize complex communication between objects in a mediator object, so they don't reference each other directly.

```java
interface ChatMediator { void send(String msg, User sender); void addUser(User u); }

class ChatRoom implements ChatMediator {
    private List<User> users = new ArrayList<>();
    public void addUser(User u) { users.add(u); }
    public void send(String msg, User sender) {
        for (User u : users) if (u != sender) u.receive(msg);
    }
}

class User {
    private String name; private ChatMediator room;
    User(String n, ChatMediator r) { name = n; room = r; }
    void send(String m) { room.send(m, this); }
    void receive(String m) { System.out.println(name + " got: " + m); }
}
```

**Production scenarios:**
- Chat rooms, UI dialogs (form fields interacting), air traffic systems
- Orchestrators in microservices (Saga orchestrator)
- Message brokers, Spring MVC `DispatcherServlet`

**Mediator vs Observer:**
- Observer: one-to-many broadcast, subject unaware of the observers
- Mediator: many-to-many coordination, the mediator knows everyone and controls the flow

**Best case:** Many objects with tangled interdependencies.
**Worst case:** The mediator becomes a **"God Object"**, with all logic crammed in, hard to maintain.

---

## 10. Iterator Pattern

**Kid explanation:** Flip through a photo album page by page. You don't need to know how the album is stored, you just say "next".

**Definition:** Provide a way to access elements of a collection sequentially without exposing its underlying structure.

```java
List<String> list = List.of("A", "B", "C");
Iterator<String> it = list.iterator();
while (it.hasNext()) System.out.println(it.next());
```

**Production scenarios:**
- Java Collections (`Iterator`, enhanced `for` loop)
- Database cursors, paginated API results, file readers
- Streams, tree/graph traversals (in-order, BFS, DFS)

**Best case:** Uniform traversal across different collection types.
**Worst case:**
- Overkill for simple arrays
- **`ConcurrentModificationException`** if the collection is modified during iteration
- Slightly slower than direct index access

---

## 11. Memento Pattern

**Kid explanation:** A **save point in a video game**. If you die, load the save and continue from there.

**Definition:** Capture and externalize an object's internal state so it can be restored later, without violating encapsulation.

```java
class Editor {
    private String text = "";
    void type(String t) { text += t; }
    Memento save() { return new Memento(text); }
    void restore(Memento m) { this.text = m.getText(); }

    static class Memento {
        private final String text;
        private Memento(String t) { text = t; }
        private String getText() { return text; }
    }
}
```

**Production scenarios:**
- Undo in editors, game checkpoints
- Database transaction rollback, draft auto-save
- Snapshotting in event sourcing, configuration rollback

**Best case:** Need undo/rollback with a clean encapsulated state.
**Worst case:** **Memory-heavy** if snapshots are large or frequent. Mitigate with incremental or diff snapshots.

---

## 12. Visitor Pattern

**Kid explanation:** A tax inspector visits a bakery, a toy shop, and a bank. Each shop is different, so the inspector does a *different check at each*. The shops don't change, but you can send a new kind of inspector (a fire inspector) without changing the shops.

**Definition:** Add new operations to an object hierarchy without modifying the classes, using **double dispatch**.

```java
interface Shape { void accept(ShapeVisitor v); }
class Circle implements Shape { public void accept(ShapeVisitor v) { v.visit(this); } }
class Square implements Shape { public void accept(ShapeVisitor v) { v.visit(this); } }

interface ShapeVisitor { void visit(Circle c); void visit(Square s); }

class AreaCalculator implements ShapeVisitor {
    public void visit(Circle c) { System.out.println("Circle area"); }
    public void visit(Square s) { System.out.println("Square area"); }
}
```

**Production scenarios:**
- Compilers/AST processing, tax calculators over different product types
- Document exporters (PDF, XML, JSON) over a fixed element structure
- Java's `FileVisitor`, ASM bytecode library

**Best case:** Stable class hierarchy but frequently *new operations*.
**Worst case:** Adding a **new element class** forces changing *every visitor*. It's also complex and breaks encapsulation. (Java 21 sealed classes and pattern matching reduce the need for Visitor.)

---

## 13. Interpreter Pattern

**Kid explanation:** A translator who understands a tiny secret language. Give them a sentence, they work out its meaning piece by piece.

**Definition:** Define a grammar for a language and an interpreter that evaluates sentences in that grammar.

**Production scenarios:** SQL parsers, regex engines, Spring Expression Language (SpEL), rule engines, math expression evaluators.

**Best case:** Small, simple grammar.
**Worst case:** Complex grammars lead to a **class explosion**. Use parser generators (ANTLR) instead. This is the least-used GoF pattern.

---

## 14. Pattern Comparisons (Interview Favorites)

| Compare | Key difference |
|---|---|
| **Strategy vs State** | Strategy: client swaps algorithm. State: object self-transitions through a lifecycle |
| **Strategy vs Template Method** | Strategy uses composition (runtime swap). Template uses inheritance (compile-time) |
| **Observer vs Mediator** | Observer: broadcast one-to-many. Mediator: centralized many-to-many coordination |
| **Command vs Strategy** | Command: encapsulates a *request/action* (with undo). Strategy: encapsulates an *algorithm* |
| **Chain vs Decorator** | Chain: one handler may stop the request. Decorator: all wrappers always run and add behavior |
| **Memento vs Command (undo)** | Memento stores *state snapshots*. Command stores *operations to reverse* |
| **Observer vs Pub/Sub** | Observer is in-process and direct. Pub/Sub uses a broker, so it is decoupled and distributed |

---

## 15. Real-World Framework Map (Spring and Java)

| Pattern | Where you've already used it |
|---|---|
| Strategy | `Comparator`, `AuthenticationProvider`, `PasswordEncoder` |
| Observer | `ApplicationEventPublisher`, `@EventListener`, JMS/Kafka |
| Command | `Runnable`, `Callable`, `ExecutorService` tasks |
| Template Method | `JdbcTemplate`, `RestTemplate`, `HttpServlet` |
| Chain of Responsibility | Servlet `Filter`, Spring Security chain, `HandlerInterceptor` |
| Mediator | `DispatcherServlet`, `ExecutorService`, Saga orchestrators |
| Iterator | `Iterable`, `Iterator`, `Stream`, JDBC `ResultSet` |
| State | Spring State Machine, order/loan workflows |
| Interpreter | SpEL, `java.util.regex` |
| Visitor | `FileVisitor`, `SimpleFileVisitor` |
| Memento | `Serializable` snapshots, editor undo |

---

## 16. Architect-Level Talking Points

**Why do behavioral patterns matter at scale?**
- They embody **SOLID**: Strategy/State/Visitor support Open/Closed, Observer/Mediator support Dependency Inversion.
- They map directly to **distributed patterns**: Observer → Pub/Sub, Command → CQRS, Mediator → Saga Orchestrator, Chain → API Gateway filters, State → workflow engines.

**Fintech and credit-risk examples (relevant to your domain):**
- **Strategy:** different risk scoring models per customer segment
- **State:** loan application lifecycle
- **Chain of Responsibility:** validation → fraud check → credit check → approval pipeline
- **Observer:** score change → notify alerts, dashboards, and downstream services
- **Command:** auditable, retryable financial transactions
- **Template Method:** standard report pipeline with custom data sources

**Anti-patterns to mention:**
- Pattern overuse ("golden hammer"). Don't use a pattern unless it solves a real problem.
- Mediator turning into a God Object
- Observer causing memory leaks or cascade storms
- Deep inheritance with Template Method (prefer composition)

---

## 17. Rapid-Fire Interview Q&A

**Q: Which behavioral pattern removes big if-else chains?**
Strategy (for algorithms) or State (for lifecycle behavior).

**Q: Which pattern would you use for undo/redo?**
Command (operation-based) or Memento (snapshot-based).

**Q: How is the Observer pattern different from event-driven architecture?**
Observer is an in-process, synchronous pattern. Event-driven architecture uses brokers, async delivery, durability, and replay.

**Q: Why is the Template Method `final`?**
To stop subclasses from altering the algorithm's skeleton.

**Q: What's double dispatch?**
The behavior depends on the runtime types of *two* objects. Visitor achieves this via `accept()` calling `visit()`.

**Q: Which pattern is Spring Security's filter chain?**
Chain of Responsibility.

**Q: How do you avoid Observer memory leaks?**
Always unsubscribe, or use weak references.

**Q: Is `Runnable` a pattern?**
Yes, it is the Command pattern: an action encapsulated as an object.

**Q: When would you NOT use a pattern?**
When the problem is simple and stable. Patterns add indirection and complexity.

**Q: Thread safety concerns?**
Observer lists need concurrent-safe collections (`CopyOnWriteArrayList`). States and strategies should be stateless or immutable so they can be shared across threads.

---

## 18. One-Page Memory Tricks

> **S**ome **O**bservant **C**ats **S**tay **T**idy, **C**alm, **M**ellow, **I**nquisitive, **M**ischievous, **V**ocal, **I**ntelligent

*(Strategy, Observer, Command, State, Template, Chain, Mediator, Iterator, Memento, Visitor, Interpreter)*

**Kid-level recall:**
- 🚌 **Strategy**: choose the route
- 🔔 **Observer**: subscribe and get notified
- 🧾 **Command**: order slip
- 🚦 **State**: mood-based behavior
- 🍞 **Template**: recipe skeleton
- 🔗 **Chain**: pass the complaint up
- 🗼 **Mediator**: air traffic control
- 📖 **Iterator**: turn the page
- 💾 **Memento**: save point
- 🕵️ **Visitor**: the inspector
- 🗣️ **Interpreter**: the translator

---

## 19. Final Interview Strategy

1. **Never just name a pattern.** Say: *problem → pattern → trade-off → real example.*
2. **Always mention a downside.** It shows seniority.
3. **Connect to production:** Kafka, Spring, microservices, Saga, CQRS.
4. **Know the comparisons** (Section 14). They are asked in almost every interview.
5. **Be ready to code** Strategy, Observer, and Chain on a whiteboard. These three come up most.