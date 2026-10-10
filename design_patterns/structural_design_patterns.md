# Structural Design Patterns in Java — The Complete Interview Guide

A Medium-style tutorial for Senior Java Backend Engineers and Technical Architects | Beginner-friendly explanations, production scenarios, Java examples, trade-offs, and interview questions

Imagine you are building a large application using Java, Spring Boot, microservices, databases, and cloud services.

As your application grows, you start facing problems:

* Two classes need to work together, but their interfaces are incompatible.

* You want to add functionality without modifying an existing class.

* A service depends on a third-party API, and you want to hide its complexity.

* You need to treat individual objects and groups of objects in the same way.

* You want to control access to an expensive or sensitive object.

* You need to combine many small components into a flexible system.

Structural Design Patterns help you solve these problems by organizing classes and objects into useful structures.

This guide covers all seven structural design patterns from the classic Gang of Four (GoF) catalog, with a special focus on the decisions a technical architect should be able to explain in an interview.

## 1. What are Structural Design Patterns?

### The simplest definition

Structural design patterns describe how classes and objects can be combined to build larger, flexible, and maintainable software structures.

Think of building with LEGO blocks.

You already have individual blocks. Structural patterns explain how to connect them so that you can build something bigger without redesigning every block.

![Building the Ultimate Monkie Kid City | Brickset](https://images.openai.com/static-rsc-4/iEX_fF7pCC6Os7p04Ee_xjkb2c8MF1xYEyNzWlE3ZW8io7NUvaWBWhiFZfQApRrfz47oXE_Q4zHKetCC7LJSpHmBnigNZnjWNf9ekOSu7ciuX_dR1LFU06I3R3nubT5N2jfdv0Rty_E3tQ_xrQPtXL-446Cm9OowRPmHNYeszYU?purpose=inline)

Add to Favorites

![Research | Personal Website](https://images.openai.com/static-rsc-4/KaMsF7XU7jgoh6k5s888iFhHF4MyaEQeeVJoRqQfZCB6FnCyh0QHZ8RkuQKAhmBgqQaOhDeApUGhUdyd7fDU4iGpt1Tvg27RM6fISf4HuNQOcaLit5QpLpbnCpRI0h3Fl7EIcngzqhmBuRp2JPWnNAGhS34XrExJhaEcCi_iIA8?purpose=inline)

Add to Favorites

![Re: Stahlträgerfachwerkbrücke Nummer 2 :: LEGO bei 1000steine.de :: Gemeinschaft :: Forum](https://images.openai.com/static-rsc-4/79yk1B3BPYRWmVhrbOG2qO1O6XpUVKuI4dfIzggt4j-38xAtqcNYteM5K3zAZ6zgTktj-aUj-wsTqQ-iwW3QRZO6BqdRFCeS5sUAsuY_py_Jy2TVDGm8P_fOEfltsTWbbsoQ3wKj--0QqQKDfNeApUk-zoQEwpuy200EJQf33LA?purpose=inline)

9

Add to Favorites

For example:

* An Adapter makes two incompatible components work together.

* A Decorator adds functionality to an existing object.

* A Facade provides a simple interface to a complicated subsystem.

* A Proxy controls access to another object.

* A Composite allows individual objects and groups to be treated uniformly.

* A Bridge separates an abstraction from its implementation.

* A Flyweight shares common data to save memory.

### Why do we need them?

Suppose you are building an e-commerce application.

Initially, your checkout service looks simple:

```
CheckoutService
    |
    +--> PaymentService
    +--> InventoryService
    +--> NotificationService
```

As the business grows, you add:

* Multiple payment providers

* Different notification channels

* Fraud detection

* Caching

* Logging and metrics

* Third-party integrations

If you solve every new requirement by adding more dependencies and conditions to existing classes, the code becomes tightly coupled and difficult to change.

Structural patterns help you organize these relationships so that the system remains understandable as it grows.

### Structural vs. Creational vs. Behavioral patterns

| Pattern category | Main question                                             | Examples                    |
| ---------------- | --------------------------------------------------------- | --------------------------- |
| Creational       | How should objects be created?                            | Factory, Builder, Singleton |
| Structural       | How should objects be connected or composed?              | Adapter, Decorator, Facade  |
| Behavioral       | How should objects communicate or share responsibilities? | Strategy, Observer, Command |

Interview tip: If an interviewer asks you to distinguish the three categories, remember:

* Creational = creating objects.

* Structural = arranging objects.

* Behavioral = coordinating object behavior.

Some patterns can influence more than one concern in practice, but this classification describes their primary intent.

## 2. The Seven Structural Design Patterns at a Glance

![YESIDO MC32 Universal Travel Adapter Multi-Nation Compatibility 1380W Global Plug Converter Wholesale & Custom - TVCMALL](https://images.openai.com/static-rsc-4/SIqF41REwXb5o6uMhEfk9JsPdaSU_cmY47ZCvOeZTJ6rq00uIH6EBeLuinusv1kbWtpSAPx3DaShpodbGjX4VZlYcxoTdbqf_kXs-BE8av7_9jlHa_xFH4mcT2NWuRwVMu2reC9BVbEvhfG5x07KnRByjUa-uL7YRMxAEas-yMQ?purpose=inline)

Add to Favorites

1. Adapter

Makes incompatible interfaces work together.

Example: adapting a third-party payment API to your application's payment interface.

![How to use handy wrap](https://images.openai.com/static-rsc-4/LKWoMFZgURDbxH2tCMq4hoNB7IOieRMvloW4tMny9McDZT_GIdjoncKKc7n49PJbScDLEXjpf1WP0oeqmR6iQ72mSFlgIF3Gd-7J-vjBrcu2zOWLcu2jpJRH66vXvYW229mUNWc_ZBp-fSwrP3kXwE4fi2AkHFZcXHpytlpi8IvtnAeIXbtASmkYA2KSEcct?purpose=inline)

Add to Favorites

2. Decorator

Adds responsibilities to an object dynamically.

Example: adding logging, compression, or encryption around a data stream.

![Office Reception Mockup Illustrations - Free Download in SVG, PNG](https://images.openai.com/static-rsc-4/raNAuZeJuQVAVI34UJh5TwTryCrP6tmBU-G5wrg0lOYST6ADNi7yHP7P0oZAmaxrVl6H5aqG9zC9StY5iJZaE6GZKzJmMgKZ7kONJhsER810pwyy9B3rkrt3IKG71zXe0R7fhAj3Aq0sUzwlyWMk7AbmTC05FtM6rtM_U6LNSic?purpose=inline)

Add to Favorites

3. Facade

Provides a simple entry point to a complex subsystem.

Example: a checkout service coordinating inventory, payment, and notifications.

![施設警備の「入出管理」って何をするの？仕事内容や依頼時のポイントを紹介   | 警備員・客室清掃員の手配なら埼玉県草加市のアシスト株式会社へ！](https://images.openai.com/static-rsc-4/0v7ZNuInuH7QdNlU35-hZ_tQYuhvR8EsqH53itxcv3Mw5ZAAnfLRmu714uO4zB4Epv9EM9kmr9Kyv7jiV3DGA1aEBc1GrQKOybtG-mcvFbCm3klrr9NLDlyyr5hC6bwfoSycjViUexbYl_LAYjuAzRKFhMujQrpYiWLERlSTvqQ?purpose=inline)

Add to Favorites

4. Proxy

Controls access to another object.

Example: a lazy-loading proxy or a remote service proxy.

![비선형 자료구조 - 트리(Tree)란 무엇이고 왜 이진 트리를 알아야 하는가?](https://images.openai.com/static-rsc-4/QfYG2owEP4sJJVsitqif3TmAwnrKdaF50769SFM4z2YSRlsx_XDdKDeDGQxHXPhVc_gtPfCXsixs8tjyAJa3SDuLAGFSaOlCnY7PEpBr04cEavVIzpphxd_QP_gi2MXGr0y1gZyQNlHwilmzuTNoa9-GzaPBslHas_Y5-qzrSao?purpose=inline)

Add to Favorites

5. Composite

Treats individual objects and groups uniformly.

Example: a file system with files and folders.

![> Flexible Shaft-Top 10 Benefits](https://images.openai.com/static-rsc-4/xbEpRlhf-EpTk0Gejg-yQvdSrsh0V_T98kpJetyYk0_p9Sl41kw47pohGXDua1EZftq0qr-mZ0_gfsFRa4hms-yAOk3V3YyZ6XHUp_cj8CSKm4PhqWrHieB2p8TBTRtEynUBWxorgTGO9_f2P9dNzjPr3xxN2nZVogiMSZntEEA?purpose=inline)

Add to Favorites

6. Bridge

Separates an abstraction from its implementation.

Example: supporting multiple notification types and delivery providers independently.

![Cube assembled of blocks](https://images.openai.com/static-rsc-4/fDDxw3RccA8tmeV1pGzuM6zhDTJHpV94TTdG44kQKhC-Bxvv1-77LNx1vAjU2U4bMRztN0Wprhk9f7tZSdiqT18Q92Ft-jneGIvq-SpvTk8FJe5-HCT6IP5os7K3Z8hIj34glrwQ6UFJ8IHXt7rJ3UFVkjOuf30tyGd0cTS4EI_GdOd9OOOwdnWzsFW2Rd4R?purpose=inline)

Add to Favorites

7. Flyweight

Shares common data to reduce memory usage.

Example: sharing character glyph data when rendering millions of characters.

Let's understand each pattern from a simple example to a production-level design decision.

## 3. Adapter Pattern

### 3.1 What is the Adapter pattern?

Definition: The Adapter pattern converts one interface into another interface that the client expects.

In simple words, it makes two incompatible things work together without requiring either one to change.

### Real-world analogy

Imagine you travel to another country. Your laptop charger has a plug that doesn't fit the wall socket.

You use a travel adapter.

```
Laptop charger --> Travel Adapter --> Wall socket
```

The adapter doesn't change the charger or the wall socket. It translates between them.

In Java, an Adapter does the same thing with interfaces.

### 3.2 The problem without Adapter

Suppose your application has a standard payment interface:

Java

```
public interface PaymentGateway {
    void pay(double amount);
}
```

Your checkout service expects every payment provider to implement `PaymentGateway`.

Java

```
public class CheckoutService {

    private final PaymentGateway gateway;

    public CheckoutService(PaymentGateway gateway) {
        this.gateway = gateway;
    }

    public void checkout(double amount) {
        gateway.pay(amount);
    }
}
```

Now your company integrates a third-party provider with a completely different API:

Java

```
public class LegacyPaymentProvider {

    public void makePaymentInCents(long cents) {
        System.out.println("Paid: " + cents + " cents");
    }
}
```

The problem is obvious:

* Your application expects `pay(double)`.

* The third-party provider offers `makePaymentInCents(long)`.

* You cannot pass `LegacyPaymentProvider` directly to `CheckoutService`.

You could change your checkout service to understand every provider's API. But that would increase coupling and make future integrations harder.

### 3.3 The solution

Create an adapter that implements your application's expected interface and delegates to the external provider.

Java

```
public class LegacyPaymentAdapter implements PaymentGateway {

    private final LegacyPaymentProvider provider;

    public LegacyPaymentAdapter(LegacyPaymentProvider provider) {
        this.provider = provider;
    }

    @Override
    public void pay(double amount) {
        long cents = Math.round(amount * 100);
        provider.makePaymentInCents(cents);
    }
}
```

Now wire it into the application:

Java

```
LegacyPaymentProvider provider = new LegacyPaymentProvider();

PaymentGateway gateway =
        new LegacyPaymentAdapter(provider);

CheckoutService checkout = new CheckoutService(gateway);

checkout.checkout(100.00);
```

Output:

```
Paid: 10000 cents
```

The checkout service knows nothing about the legacy provider's interface.

Adapter architecture

CheckoutService

LegacyPaymentAdapter

LegacyPaymentProvider

### 3.4 When should you use Adapter?

Use it when:

* Integrating third-party APIs with different interfaces.

* Migrating from a legacy system to a modern interface.

* Supporting multiple vendors behind a common application interface.

* Adapting an old library that you cannot modify.

* Translating data formats or method contracts at an integration boundary.

Production scenario: Your Spring Boot application supports three SMS providers. Each provider has a different API. Create one adapter per provider, all implementing your internal `SmsGateway` interface.

```
NotificationService
        |
     SmsGateway
        |
  +-----+-----+
  |     |     |
Adapter A   Adapter B   Adapter C
  |           |           |
Vendor A    Vendor B    Vendor C
```

The application can switch providers without rewriting notification business logic.

### 3.5 Best and worst cases

Best case: You need to integrate a system with an incompatible interface, and you want to isolate that incompatibility.

Worst case: You build adapters around every class even when their interfaces already fit. This adds unnecessary classes and indirection.

### 3.6 Important interview distinctions

* Adapter vs. Facade: Adapter makes an incompatible interface compatible. Facade simplifies access to a complex subsystem.

* Adapter vs. Decorator: Adapter changes the interface presented to the client. Decorator preserves the component's interface while adding behavior.

* Object Adapter vs. Class Adapter: An object adapter uses composition/delegation; a class adapter uses inheritance. In Java, object adapters are generally easier to use because Java supports only single class inheritance.

Architect-level caution: A payment adapter should translate provider-specific requests and responses, but it should also handle currency units, error mapping, timeouts, idempotency, and provider-specific status semantics. Converting method names alone is not sufficient for a production integration.

## 4. Decorator Pattern

### 4.1 What is the Decorator pattern?

Definition: Decorator adds new behavior to an object dynamically by wrapping it in another object that implements the same interface.

In simple words, you put an additional layer around an existing object without modifying its original class.

### Real-world analogy: ordering coffee

Imagine you order a plain coffee.

Then you add:

* Milk

* Sugar

* Chocolate

Instead of creating a separate class for every combination, you wrap the coffee with additional features.

```
Plain Coffee
     |
     v
Coffee + Milk
     |
     v
Coffee + Milk + Chocolate
```

Each wrapper adds something while preserving the basic coffee interface.

### 4.2 The problem without Decorator

Suppose you have:

Java

```
public interface Coffee {
    double cost();
}
```

Java

```
public class PlainCoffee implements Coffee {

    @Override
    public double cost() {
        return 50;
    }
}
```

Now you want milk and chocolate.

A poor design might create:

```
PlainCoffee
MilkCoffee
ChocolateCoffee
MilkChocolateCoffee
MilkSugarCoffee
MilkSugarChocolateCoffee
...
```

As the number of optional features grows, the number of combinations becomes difficult to maintain.

### 4.3 The solution

Create decorators that implement the same interface and wrap another `Coffee`.

Java

```
public abstract class CoffeeDecorator implements Coffee {

    protected final Coffee coffee;

    protected CoffeeDecorator(Coffee coffee) {
        this.coffee = coffee;
    }
}
```

Milk decorator:

Java

```
public class MilkDecorator extends CoffeeDecorator {

    public MilkDecorator(Coffee coffee) {
        super(coffee);
    }

    @Override
    public double cost() {
        return coffee.cost() + 10;
    }
}
```

Chocolate decorator:

Java

```
public class ChocolateDecorator extends CoffeeDecorator {

    public ChocolateDecorator(Coffee coffee) {
        super(coffee);
    }

    @Override
    public double cost() {
        return coffee.cost() + 20;
    }
}
```

Use them together:

Java

```
Coffee coffee = new PlainCoffee();

coffee = new MilkDecorator(coffee);
coffee = new ChocolateDecorator(coffee);

System.out.println(coffee.cost()); // 80.0
```

Each decorator delegates to the wrapped object and adds its own cost.

### 4.4 How does the call flow work?

```
ChocolateDecorator.cost()
        |
        +--> MilkDecorator.cost()
                    |
                    +--> PlainCoffee.cost() = 50
                    |
                    +--> Add milk = 10
        |
        +--> Add chocolate = 20

Total = 80
```

This is called composition: objects are assembled at runtime to produce the required behavior.

### 4.5 Production examples in Java

Java's own I/O libraries provide classic Decorator examples:

Java

```
InputStream input =
    new BufferedInputStream(
        new FileInputStream("data.txt")
    );
```

Here:

* `FileInputStream` reads bytes from a file.

* `BufferedInputStream` adds buffering.

* Both expose the `InputStream` abstraction.

Other possible applications:

* Adding metrics around a service.

* Adding request/response logging.

* Compressing a stream.

* Encrypting or decrypting stream data.

* Adding retry or caching behavior around a component, when the semantics are carefully designed.

For example, a service decorator could measure execution time while keeping the same service interface:

Java

```
public interface ReportService {
    String generate();
}
```

Java

```
public class TimingReportDecorator implements ReportService {

    private final ReportService delegate;

    public TimingReportDecorator(ReportService delegate) {
        this.delegate = delegate;
    }

    @Override
    public String generate() {
        long start = System.nanoTime();

        try {
            return delegate.generate();
        } finally {
            long elapsed = System.nanoTime() - start;
            System.out.println("Elapsed ns: " + elapsed);
        }
    }
}
```

The decorator adds timing without changing the original report implementation.

### 4.6 When should you use Decorator?

Use it when:

* Features are optional and combinable.

* You need to add behavior without modifying the original class.

* Subclassing would lead to too many combinations.

* The wrapped and decorating objects should expose the same interface.

### 4.7 Best and worst cases

Best case: You need to combine optional behaviors flexibly at runtime.

Worst case: There are too many nested wrappers, and the call chain becomes hard to debug.

For example:

```
LoggingDecorator
  -> MetricsDecorator
      -> RetryDecorator
          -> CacheDecorator
              -> ActualService
```

This is not automatically bad, but the ordering matters. Retrying a cache lookup has different semantics from retrying a remote call. Logging each retry also differs from logging only the final request.

### 4.8 Decorator vs. Proxy

Both wrap objects, so interviewers frequently ask this question.

| Decorator                        | Proxy                                                     |
| -------------------------------- | --------------------------------------------------------- |
| Primary goal is to add behavior  | Primary goal is to control access                         |
| Often composes optional features | Often enforces access, lazy loading, or remote invocation |
| Example: buffered stream         | Example: access-control proxy                             |

Their structures can look almost identical. The intent determines which pattern you are using.

Architect-level caution: In Spring Boot, AOP, transaction management, caching, and security can use proxy-based mechanisms. Do not assume every Spring proxy is a Decorator. A transaction proxy, for example, controls the invocation lifecycle and transaction boundaries rather than merely adding an optional feature.

## 5. Facade Pattern

### 5.1 What is the Facade pattern?

Definition: Facade provides a simplified interface to a complex subsystem.

In simple words, it gives you one convenient entry point instead of forcing you to understand and coordinate many internal components.

### Real-world analogy

Imagine checking into a hotel.

You don't independently contact:

* Housekeeping

* Room allocation

* Payment processing

* Reception

* Key management

You talk to the receptionist, who coordinates the necessary services.

The receptionist acts like a facade.

### 5.2 The problem without Facade

Suppose your checkout flow requires multiple services:

```
Checkout Controller
   |
   +--> InventoryService
   +--> PricingService
   +--> PaymentService
   +--> OrderService
   +--> NotificationService
```

The controller now knows too much about the internal workflow.

Every change in the checkout process can force changes to the controller. Other controllers may duplicate the same orchestration logic.

### 5.3 The solution

Create a `CheckoutFacade` that coordinates the subsystem.

Java

```
public class CheckoutFacade {

    private final InventoryService inventory;
    private final PricingService pricing;
    private final PaymentService payment;
    private final OrderService orders;
    private final NotificationService notifications;

    public CheckoutFacade(
            InventoryService inventory,
            PricingService pricing,
            PaymentService payment,
            OrderService orders,
            NotificationService notifications) {

        this.inventory = inventory;
        this.pricing = pricing;
        this.payment = payment;
        this.orders = orders;
        this.notifications = notifications;
    }

    public String checkout(String productId, int quantity) {
        inventory.reserve(productId, quantity);

        double total = pricing.calculate(productId, quantity);

        payment.charge(total);

        String orderId = orders.create(productId, quantity);

        notifications.sendOrderConfirmation(orderId);

        return orderId;
    }
}
```

The client now calls one method:

Java

```
String orderId = checkoutFacade.checkout("P100", 2);
```

The facade hides the coordination details.

Checkout Controller

Checkout Facade

Simple business entry point

Inventory

Pricing

Payment

Orders

### 5.4 Is Facade just a wrapper?

Not exactly.

A facade intentionally provides a simpler entry point to a subsystem. It may coordinate multiple operations, but its main purpose is to simplify access.

A wrapper can simply delegate one method to another. A facade usually hides a broader set of subsystem details.

### 5.5 Production scenarios

* A checkout facade coordinates several domain services.

* A reporting facade exposes one operation that queries multiple data sources.

* A document-processing facade coordinates file reading, validation, transformation, and storage.

* An application service provides a simplified API over several internal libraries.

### 5.6 When should you use Facade?

Use it when:

* A subsystem has many classes or complex call sequences.

* Client code is tightly coupled to implementation details.

* Multiple clients repeat the same orchestration logic.

* You want a clean boundary between an application layer and an internal subsystem.

### 5.7 Best and worst cases

Best case: A complex subsystem needs a stable, simple entry point.

Worst case: A facade becomes a giant class that contains all business logic, all validation, all transaction management, and all integration logic.

That creates a God Object.

A facade should simplify access, not become the owner of every responsibility.

### 5.8 Facade vs. Adapter vs. Mediator

* Facade: simplifies access to a subsystem.

* Adapter: translates an incompatible interface.

* Mediator: centralizes communication between collaborating objects to reduce direct dependencies.

Architect-level caution: The checkout example is only structural. Real payment and inventory workflows need explicit handling of partial failures. If payment succeeds but order creation fails, the system may need idempotency, compensation, and a durable workflow such as a saga. A facade alone does not provide distributed transaction guarantees.

## 6. Proxy Pattern

### 6.1 What is the Proxy pattern?

Definition: Proxy provides a substitute object that controls access to another object.

In simple words, you interact with a representative instead of directly accessing the real object.

### Real-world analogy

Imagine a security guard at a building entrance.

You don't walk directly into a restricted room. The guard checks whether you have permission, then allows or denies entry.

```
Client --> Security Guard --> Protected Room
```

The guard is a proxy.

### 6.2 Common types of Proxy

| Type             | Purpose                                            | Example                                        |
| ---------------- | -------------------------------------------------- | ---------------------------------------------- |
| Virtual proxy    | Delays creation or loading of an expensive object  | Load a large document only when opened         |
| Protection proxy | Checks permissions                                 | Restrict access to an administrative operation |
| Remote proxy     | Represents an object in another process or machine | Client-side representation of a remote service |
| Caching proxy    | Reuses previously obtained results                 | Cache read-only product information            |
| Logging proxy    | Records access or calls                            | Audit sensitive operations                     |

These are common proxy variations, not separate GoF structural patterns.

### 6.3 Java example: protection proxy

First, define an interface:

Java

```
public interface DocumentService {
    String readDocument();
}
```

The real implementation:

Java

```
public class RealDocumentService implements DocumentService {

    @Override
    public String readDocument() {
        return "Confidential document";
    }
}
```

The proxy checks access before delegating:

Java

```
public class DocumentProxy implements DocumentService {

    private final DocumentService realService;
    private final boolean authorized;

    public DocumentProxy(
            DocumentService realService,
            boolean authorized) {
        this.realService = realService;
        this.authorized = authorized;
    }

    @Override
    public String readDocument() {
        if (!authorized) {
            throw new SecurityException("Access denied");
        }

        return realService.readDocument();
    }
}
```

Usage:

Java

```
DocumentService service =
        new DocumentProxy(new RealDocumentService(), false);

service.readDocument(); // Throws SecurityException
```

The caller interacts with the same interface but does not access the real object directly.

For real security, authorization must be based on a trusted identity and enforced at an appropriate security boundary. A client-controlled boolean is only a teaching example.

### 6.4 How Spring uses proxies

This is especially important for Java backend interviews.

Spring often uses proxies for:

* `@Transactional`

* `@Cacheable`

* Method-security annotations

* Spring AOP advice

* Some asynchronous method execution scenarios

A simplified transaction call looks like this:

```
Client
  |
  v
Spring-managed proxy
  |
  +--> Begin transaction
  |
  +--> Invoke target method
  |
  +--> Commit or roll back
  |
  v
Return result
```

The proxy intercepts the call and applies the configured behavior around the target method.

#### Important Spring gotcha: self-invocation

Consider:

Java

```
@Service
public class PaymentService {

    public void process() {
        savePayment();
    }

    @Transactional
    public void savePayment() {
        // Database operation
    }
}
```

If `process()` calls `savePayment()` directly on `this`, the call normally bypasses the Spring proxy. Therefore, the `@Transactional` advice on `savePayment()` may not be applied as expected.

This is a common interview question.

The exact behavior depends on proxy configuration and invocation path. A typical solution is to put the transactional operation in a separate Spring bean or ensure the call enters through the proxy.

### 6.5 Proxy vs. Decorator

They may share the same structure:

```
Client --> Wrapper --> Real Object
```

But their intent differs.

* Proxy: controls access to the real object.

* Decorator: adds responsibilities to the object.

A caching wrapper may be a proxy if its primary role is controlling access and reusing target results. It can also resemble a decorator structurally. Explain the intent rather than relying only on class names.

### 6.6 When should you use Proxy?

Use it when you need:

* Lazy initialization.

* Authorization or access control.

* Remote invocation.

* Transparent caching.

* Invocation interception or auditing.

### 6.7 Best and worst cases

Best case: You need to control how and when an object is accessed without changing the client-facing interface.

Worst case: The proxy introduces hidden network calls, unexpected latency, confusing object lifecycles, or security assumptions that are not enforced at the correct boundary.

Architect-level caution: A remote proxy does not make a network call behave like a local method call. Networks fail, requests time out, and responses may be lost. Distributed calls still require timeouts, failure handling, observability, and appropriate retry semantics.

## 7. Composite Pattern

### 7.1 What is the Composite pattern?

Definition: Composite lets you treat individual objects and groups of objects through the same interface.

In simple words, it allows you to work with a single item or a group of items in the same way.

### Real-world analogy: files and folders

Think about a computer file system.

A file can contain data. A folder can contain files and other folders.

```
Company
  |
  +-- Engineering
  |      |
  |      +-- Java.java
  |      +-- Spring.java
  |
  +-- Finance
         |
         +-- Budget.xlsx
```

You can ask a file for its size. You can also ask a folder for its total size.

The folder contains other objects, but both files and folders can provide a common operation.

### 7.2 The problem without Composite

Suppose you build a UI component system with individual elements and groups of elements.

Without Composite, the caller may need separate logic:

Java

```
if (component instanceof Button) {
    // Render one button
} else if (component instanceof Panel) {
    // Loop through children and render them
}
```

As you add more component types, every client may need to understand the internal hierarchy.

### 7.3 The solution

Define a common interface:

Java

```
public interface FileSystemComponent {
    long size();
}
```

A leaf represents an individual object:

Java

```
public class FileItem implements FileSystemComponent {

    private final long bytes;

    public FileItem(long bytes) {
        this.bytes = bytes;
    }

    @Override
    public long size() {
        return bytes;
    }
}
```

A composite represents a group:

Java

```
import java.util.ArrayList;
import java.util.List;

public class Folder implements FileSystemComponent {

    private final List<FileSystemComponent> children =
            new ArrayList<>();

    public void add(FileSystemComponent component) {
        children.add(component);
    }

    @Override
    public long size() {
        long total = 0;

        for (FileSystemComponent child : children) {
            total += child.size();
        }

        return total;
    }
}
```

Use both through the same interface:

Java

```
FileSystemComponent file1 = new FileItem(100);
FileSystemComponent file2 = new FileItem(200);

Folder folder = new Folder();
folder.add(file1);
folder.add(file2);

System.out.println(file1.size()); // 100
System.out.println(folder.size()); // 300
```

The folder doesn't need to know whether each child is a file or another folder. It simply calls `size()`.

### 7.4 Understand the structure

Component interface

size()

Leaf

FileItem

Composite

Folder

A Folder can contain FileItems or other Folders.

The two important terms are:

* Leaf: an individual object with no children.

* Composite: an object that contains other components.

### 7.5 Production scenarios

* File and directory hierarchies.

* UI trees containing components and containers.

* Organization structures containing teams and employees.

* Product bundles containing individual products or sub-bundles.

* Menu systems with nested menu items.

* Permission trees and nested configuration structures.

### 7.6 When should you use Composite?

Use it when:

* Your domain naturally forms a tree.

* You need recursive operations.

* Clients should handle individual items and groups uniformly.

* The same operation can meaningfully apply to both leaves and composites.

### 7.7 Best and worst cases

Best case: A hierarchical structure needs operations such as rendering, calculating size, applying permissions, or traversing descendants.

Worst case: Your leaf and composite objects do not genuinely share meaningful behavior. Forcing them into one interface may create confusing methods, such as a file exposing `addChild()` even though files cannot contain other files.

### 7.8 Composite vs. Decorator

This is a common comparison.

* Composite builds a tree of objects, typically containing children.

* Decorator wraps one component to add responsibilities.

Composite focuses on grouping. Decorator focuses on enhancement.

Architect-level caution: If you implement recursive operations over thousands of nested levels, consider stack depth, traversal complexity, cycles, and repeated calculations. A tree should not accidentally become a cyclic graph if your algorithms assume a tree.

## 8. Bridge Pattern

### 8.1 What is the Bridge pattern?

Definition: Bridge separates an abstraction from its implementation so that both can evolve independently.

In simple words, it prevents two independently changing dimensions from becoming tangled together.

This is one of the more difficult structural patterns to understand, so let's take it slowly.

### Real-world analogy: remote controls and devices

Imagine you have:

* Different remote controls: basic remote, advanced remote.

* Different devices: television, projector.

You want every remote to work with different devices.

A poor design creates:

```
BasicTVRemote
BasicProjectorRemote
AdvancedTVRemote
AdvancedProjectorRemote
```

Every new remote type multiplies the number of classes.

Instead, separate the remote abstraction from the device implementation.

```
Remote Control -----> Device Interface
       |                    |
 Basic Remote           TV
 Advanced Remote        Projector
```

Now either side can evolve independently.

### 8.2 The problem without Bridge

Suppose your Java application sends notifications through email or SMS.

You also want several notification types:

* Normal notification.

* Urgent notification.

* Scheduled notification.

A naive inheritance design creates:

```
NormalEmailNotification
UrgentEmailNotification
ScheduledEmailNotification

NormalSmsNotification
UrgentSmsNotification
ScheduledSmsNotification
```

If there are mmm notification types and nnn delivery mechanisms, you could end up maintaining approximately m×nm \times nm×n combinations.

For 4 notification types and 5 providers, that's potentially 20 combinations.

### 8.3 The solution

Separate the two dimensions.

First, define the implementation interface:

Java

```
public interface MessageSender {
    void send(String recipient, String message);
}
```

Implement different delivery mechanisms:

Java

```
public class EmailSender implements MessageSender {

    @Override
    public void send(String recipient, String message) {
        System.out.println("Email to " + recipient + ": " + message);
    }
}
```

Java

```
public class SmsSender implements MessageSender {

    @Override
    public void send(String recipient, String message) {
        System.out.println("SMS to " + recipient + ": " + message);
    }
}
```

Next, create the abstraction:

Java

```
public abstract class Notification {

    protected final MessageSender sender;

    protected Notification(MessageSender sender) {
        this.sender = sender;
    }

    public abstract void notifyUser(
            String recipient, String message);
}
```

Add a specific notification type:

Java

```
public class UrgentNotification extends Notification {

    public UrgentNotification(MessageSender sender) {
        super(sender);
    }

    @Override
    public void notifyUser(String recipient, String message) {
        sender.send(recipient, "[URGENT] " + message);
    }
}
```

Usage:

Java

```
MessageSender email = new EmailSender();
MessageSender sms = new SmsSender();

Notification urgentEmail = new UrgentNotification(email);
Notification urgentSms = new UrgentNotification(sms);

urgentEmail.notifyUser("user@example.com", "Payment failed");
urgentSms.notifyUser("+911234567890", "Payment failed");
```

We can add a new notification type or delivery mechanism without creating a dedicated class for every combination.

### 8.4 Bridge architecture

Abstraction

Notification

Implementation

MessageSender

UrgentNotification

EmailSender / SmsSender

The abstraction contains a reference to the implementation interface and delegates delivery through it.

### 8.5 Production scenarios

* Notification types separated from delivery channels.

* Payment workflows separated from payment processing implementations.

* Reports separated from rendering formats.

* Cross-platform applications separating high-level operations from platform-specific APIs.

* Storage abstractions separated from storage backends, when both dimensions evolve independently.

### 8.6 When should you use Bridge?

Use it when:

* There are two or more independent dimensions of variation.

* Inheritance would produce many combinations.

* You expect the abstraction and implementation to evolve separately.

* You need to swap implementations without changing high-level logic.

### 8.7 Best and worst cases

Best case: Two dimensions change independently, and inheritance would create a combinatorial explosion.

Worst case: The design is simple, has one implementation, and is unlikely to evolve. Bridge may add unnecessary abstractions and indirection.

### 8.8 Bridge vs. Adapter

This is a frequent senior-level interview question.

| Bridge                                                              | Adapter                                                              |
| ------------------------------------------------------------------- | -------------------------------------------------------------------- |
| Designed to separate two dimensions from the beginning              | Commonly used to make existing incompatible interfaces work together |
| Focuses on independent evolution                                    | Focuses on compatibility                                             |
| Both sides are designed to collaborate through a shared abstraction | The adapter translates between different contracts                   |
| Example: notification type × delivery mechanism                     | Example: legacy SMS provider × internal SMS interface                |

Important nuance: Bridge and Adapter can look structurally similar because both use composition. Their intent and the problem they solve distinguish them.

## 9. Flyweight Pattern

### 9.1 What is the Flyweight pattern?

Definition: Flyweight reduces memory consumption by sharing reusable objects that contain common state.

In simple words, instead of creating thousands or millions of identical objects, you reuse shared information wherever possible.

### Real-world analogy: a text document

Imagine a document containing one million occurrences of the letter `A`.

Do you need to store the complete font definition, glyph shape, and other common character data separately for every occurrence?

No. You can share the common character data and store the position, size, or other unique details for each occurrence.

That's the basic idea behind Flyweight.

### 9.2 The problem without Flyweight

Suppose you build a text editor or a map-rendering engine.

You create a separate object for every character or tree on the screen, and every object duplicates large amounts of data.

For example:

```
Character 1: glyph data + position + font + color
Character 2: glyph data + position + font + color
Character 3: glyph data + position + font + color
...
```

If the glyph and font data are identical, storing them repeatedly wastes memory.

### 9.3 Intrinsic vs. extrinsic state

Flyweight works by separating two kinds of state.

* Intrinsic state: shared, relatively stable data stored in the reusable object.

* Extrinsic state: context-specific data supplied by the caller.

For a rendered character:

| Intrinsic state             | Extrinsic state                    |
| --------------------------- | ---------------------------------- |
| Character glyph             | X/Y position                       |
| Font data                   | Current rendering context          |
| Reusable character metadata | Per-occurrence color, if it varies |

The exact split depends on the application. Any data that differs per occurrence generally cannot be shared as intrinsic state.

### 9.4 Java example

First, create a reusable tree type:

Java

```
public class TreeType {

    private final String name;
    private final String color;
    private final String texture;

    public TreeType(String name, String color, String texture) {
        this.name = name;
        this.color = color;
        this.texture = texture;
    }

    public void draw(int x, int y) {
        System.out.println(
                "Drawing " + name + " at " + x + "," + y
        );
    }
}
```

The type contains data shared by all trees of that type.

Create a factory that reuses instances:

Java

```
import java.util.HashMap;
import java.util.Map;

public class TreeTypeFactory {

    private static final Map<String, TreeType> CACHE =
            new HashMap<>();

    public static TreeType getTreeType(
            String name, String color, String texture) {

        String key = name + "|" + color + "|" + texture;

        return CACHE.computeIfAbsent(
                key,
                k -> new TreeType(name, color, texture)
        );
    }
}
```

Each tree occurrence holds its own position but shares its type:

Java

```
public class Tree {

    private final int x;
    private final int y;
    private final TreeType type;

    public Tree(int x, int y, TreeType type) {
        this.x = x;
        this.y = y;
        this.type = type;
    }

    public void draw() {
        type.draw(x, y);
    }
}
```

Usage:

Java

```
TreeType oak =
        TreeTypeFactory.getTreeType("Oak", "Green", "Rough");

Tree tree1 = new Tree(10, 20, oak);
Tree tree2 = new Tree(30, 40, oak);
```

Both trees share the same `TreeType` instance but have different positions.

### 9.5 How much memory can Flyweight save?

Imagine 1,000,000 trees.

If every tree has a separate copy of 1 KB of reusable type data, the duplicated data alone is approximately 1 GB.

If those trees share just 100 tree types, the shared type data would be approximately 100 KB, excluding object overhead, positions, references, maps, and other application data.

The actual savings depend on the object layout, retained state, and JVM memory behavior. The important point is eliminating repeated data, not assuming an exact production memory reduction.

### 9.6 Production scenarios

* Text editors and rendering engines.

* Game engines with repeated sprites or objects.

* Large maps with repeated tile or tree types.

* Reusable immutable metadata.

* Large-scale processing where many records reference a small set of common definitions.

### 9.7 When should you use Flyweight?

Use it when:

* You have a huge number of similar objects.

* Many objects share substantial immutable state.

* Profiling shows memory usage is a meaningful problem.

* Shared state can be separated from per-object state.

### 9.8 Best and worst cases

Best case: Millions of objects share a relatively small number of reusable configurations.

Worst case: Objects have little common data, or creating and managing the cache costs more than the memory saved.

### 9.9 Important production concerns

* Shared intrinsic state should usually be immutable.

* A normal `HashMap` is not safe for concurrent mutation without synchronization.

* An unbounded flyweight cache can itself become a memory leak.

* Cache keys must uniquely identify the shared state.

* Don't assume every repeated object should be cached. Measure memory allocation and retained heap first.

Flyweight vs. Object Pool: Flyweight shares reusable intrinsic data across many logical objects. Object Pool reuses instances of expensive or limited resources, such as connections. They solve different problems.

Flyweight vs. Singleton: Singleton limits a particular object's instantiation; Flyweight shares common state across many logical objects. A Flyweight factory may itself be a singleton, but that is not what defines the pattern.

## 10. One more useful distinction: structural patterns and composition

All seven patterns help structure software, but they don't all use the same mechanism.

| Pattern   | Main structural mechanism                                        |
| --------- | ---------------------------------------------------------------- |
| Adapter   | Translates through a wrapper                                     |
| Decorator | Wraps an object with additional behavior                         |
| Facade    | Presents a simplified subsystem interface                        |
| Proxy     | Represents and controls access to another object                 |
| Composite | Builds a tree of components                                      |
| Bridge    | Connects independently evolving abstractions and implementations |
| Flyweight | Shares common state among many logical objects                   |

Notice how frequently composition appears.

Composition means an object holds references to other objects and delegates work to them.

For example:

Java

```
public class CheckoutService {

    private final PaymentGateway paymentGateway;

    public CheckoutService(PaymentGateway paymentGateway) {
        this.paymentGateway = paymentGateway;
    }
}
```

`CheckoutService` contains a reference to a `PaymentGateway`. It doesn't need to inherit from the gateway.

This gives you flexibility to inject different implementations, use test doubles, and replace dependencies without rewriting the consumer.

For technical architects, the lesson is not to use a design pattern everywhere. It is to identify the pressure that causes a design to become difficult to change, then select the simplest structure that addresses it.

## 11. The most important comparison table

If you remember only one table from this tutorial, remember this one.

| Pattern   | Problem it solves                | Simple example                       | Key question                                                |
| --------- | -------------------------------- | ------------------------------------ | ----------------------------------------------------------- |
| Adapter   | Incompatible interfaces          | Legacy payment API                   | How do I make these interfaces work together?               |
| Decorator | Optional additional behavior     | Add logging around a service         | How can I add behavior without changing the original class? |
| Facade    | Complex subsystem                | Checkout orchestration               | How can I give clients one simple entry point?              |
| Proxy     | Controlled access                | Spring transaction proxy             | How can I control access to the real object?                |
| Composite | Hierarchical objects             | Files and folders                    | How can I treat a group like an individual item?            |
| Bridge    | Independent dimensions of change | Notification type × delivery channel | How can two dimensions evolve independently?                |
| Flyweight | Repeated shared state            | Character glyphs                     | How can I reduce duplicated memory?                         |

### Quick decision guide

Which problem are you trying to solve?

Two interfaces don't match.

I need to add optional behavior.

A subsystem is too complicated to use.

I need to control access to an object.

I need to handle individual objects and groups uniformly.

Two independent dimensions cause too many classes.

Millions of objects duplicate common data.

## Use Adapter

Translate an existing interface into the interface your application expects.

## 12. Frequently asked interview questions and answers

These are the questions you should be comfortable answering in a senior Java backend or technical architect interview.

### Q1. What are structural design patterns?

Structural design patterns describe how classes and objects can be combined to create flexible, reusable, and maintainable software structures. The seven classic GoF structural patterns are Adapter, Bridge, Composite, Decorator, Facade, Flyweight, and Proxy.

### Q2. Why do we need structural patterns if Java already has interfaces and inheritance?

Interfaces and inheritance are language features. Design patterns are reusable solutions to recurring design problems.

Java gives you the tools to build an Adapter, Decorator, or Bridge; it does not automatically decide when or how to use them.

### Q3. Which structural patterns use composition?

Composition is central to Adapter, Bridge, Decorator, and Proxy. Facade also commonly uses composition to coordinate subsystem objects. Composite uses composition to contain children, and Flyweight uses shared references to reuse objects.

The exact implementation varies, but composition is a major theme across this category.

### Q4. Which structural patterns are most useful in Spring Boot?

Common examples include:

* Proxy: Spring AOP, transaction management, caching, and method-security interception.

* Decorator: Adding behavior around a service or stream without changing its interface.

* Adapter: Integrating external systems through internal interfaces.

* Facade: Simplifying access to a complex set of application services.

Not every Spring component is an instance of a GoF pattern. Explain the actual intent and mechanism.

### Q5. Can one class participate in multiple patterns?

Yes.

A third-party payment adapter could also be wrapped by a logging decorator and exposed through a higher-level checkout facade.

```
Checkout Controller
        |
   Checkout Facade
        |
   Logging Decorator
        |
   Payment Adapter
        |
   External Payment API
```

Each layer solves a different problem. However, avoid creating unnecessary layers merely to demonstrate design patterns.

### Q6. What is the difference between inheritance and composition?

Inheritance models an is-a relationship. A `Dog` is an `Animal`.

Composition models a has-a relationship. A `CheckoutService` has a `PaymentGateway`.

Composition is often more flexible when behavior needs to be replaced at runtime or when multiple implementations must be supported. Inheritance remains appropriate when a genuine subtype relationship exists and the substitution contract is respected.

### Q7. Why is Decorator often preferred over subclassing?

If you have five optional features, inheritance can produce many combinations of subclasses. Decorator allows the features to be combined at runtime.

However, Decorator is not always better. Subclassing may be simpler when the variation is stable, the hierarchy is small, and the behavior genuinely belongs in a subtype.

### Q8. Is Facade the same as a service layer?

Not necessarily.

A service layer organizes application or business operations. A facade simplifies access to a subsystem.

A service can act as a facade if it exposes a simple interface over several components, but the terms describe different design concerns.

### Q9. Is a REST client wrapper an Adapter or a Proxy?

It depends on the intent.

* If it translates an external API into an internal interface, it is an Adapter.

* If it represents a remote service and controls invocation, it can be a Remote Proxy.

* If it merely simplifies a group of operations, it might be a Facade.

One class can have characteristics of more than one pattern. In interviews, describe the specific problem it solves.

### Q10. Does using a design pattern automatically make code better?

No.

Patterns introduce structure and often introduce additional classes, interfaces, and indirection.

A good design balances flexibility against complexity. Use a pattern when it solves a real problem or addresses a credible, expected direction of change.

### Q11. Which pattern reduces memory usage?

Flyweight. It shares common intrinsic state across many logical objects while keeping per-object state separate.

### Q12. Which pattern is useful for tree structures?

Composite. It represents individual objects and groups through a common abstraction, enabling recursive operations.

### Q13. How is Bridge different from Strategy?

This is a valuable senior-level distinction.

* Bridge is a structural pattern that separates an abstraction from its implementation, allowing both to evolve independently.

* Strategy is a behavioral pattern that encapsulates interchangeable algorithms or behaviors.

They can look similar in Java because both commonly use composition and interfaces. The design intent is different.

For example, `Notification` combined with `MessageSender` illustrates Bridge when notification types and delivery mechanisms are independent dimensions. Choosing one interchangeable pricing algorithm illustrates Strategy.

### Q14. How do you choose between Proxy and Decorator?

Ask what the wrapper is primarily responsible for.

* Controlling access, lazy loading, or remote invocation points toward Proxy.

* Adding an optional responsibility while preserving the component contract points toward Decorator.

Their structures can be nearly identical, so class diagrams alone may not reveal the intended pattern.

### Q15. Are these patterns only useful in monolithic applications?

No. They are useful in monoliths, modular applications, and microservices.

For example, Adapter can isolate a third-party API, Proxy can control access to a remote object, and Facade can simplify a service's internal subsystem.

However, a design pattern does not replace distributed-system concerns such as timeouts, retries, idempotency, service discovery, or fault tolerance.

## 13. Scenario-based questions for a Technical Architect

These are more important than simply memorizing definitions. In an architect interview, you need to identify the underlying design pressure, justify a pattern, and explain trade-offs.

Scenario 1: Multiple payment providers

Your Spring Boot application integrates with three payment providers. Each exposes a different API. The business wants to switch providers without changing checkout logic.

Think before revealing the answer.

Hide answer

Primary pattern: Adapter.Define an internal payment interface and implement an adapter for each provider. Add provider selection through configuration or an appropriate routing component.Also consider idempotency keys, currency and amount normalization, timeouts, error translation, reconciliation, and provider-specific status handling. Avoid scattering vendor-specific logic throughout the business layer.

Scenario 2: Add logging and metrics without changing business classes

You need to measure service latency and log calls, but you don't want to modify every implementation.

Hide answer

Possible patterns: Decorator or Proxy.Use a Decorator when optional behavior is explicitly composed around a component. Use a Proxy or Spring AOP when invocation interception is the primary goal.In production, consider Spring AOP, metrics instrumentation, tracing, and structured logging before writing custom wrappers. Avoid logging credentials, payment details, or other sensitive data.

Scenario 3: A single API must orchestrate five backend services

The controller contains inventory checks, pricing, payment, order creation, and notifications. Every new checkout feature changes the controller.

Hide answer

Primary pattern: Facade.Provide a clean checkout entry point that coordinates subsystem operations.Keep business rules in appropriate domain services. For distributed operations, design failure recovery, idempotency, and compensation separately; Facade does not guarantee atomicity.

Scenario 4: Millions of trees in a simulation

Each tree has a position and a large texture definition. Most trees share the same few types.

Hide answer

Primary pattern: Flyweight.Share immutable tree-type and texture data. Store position and other per-tree attributes separately.Measure actual allocation and retained heap. Make the cache concurrency-safe, bound its growth when necessary, and ensure cache keys capture all relevant shared state.

Scenario 5: A document contains nested folders

The application must calculate the total size of a folder and all its descendants without writing separate client logic for files and folders.

Hide answer

Primary pattern: Composite.Expose a common component interface. Files implement the operation directly; folders aggregate results from their children.Consider deep recursion, invalid child operations, cycles, and caching derived values for large hierarchies.

Scenario 6: Two dimensions of product variation

Your application has multiple report formats and multiple output destinations. Inheritance is producing dozens of combinations.

Hide answer

Primary pattern: Bridge.Separate the report abstraction from the output implementation. Report types and delivery mechanisms can evolve independently.Check whether the dimensions really are independent. If the problem is simply selecting one interchangeable algorithm, Strategy might be more appropriate.

## 14. Common mistakes candidates make

1. Choosing a pattern before identifying the problem. Start by explaining what is difficult to change, test, or extend.

2. Confusing Adapter with Facade. Adapter addresses interface incompatibility; Facade addresses subsystem complexity.

3. Confusing Proxy with Decorator. Their structure may be identical; their primary intent differs.

4. Thinking Bridge and Adapter are interchangeable. Bridge separates independently varying dimensions; Adapter commonly reconciles interfaces that already exist.

5. Using inheritance for every variation. Consider composition when behaviors need to be combined or implementations swapped.

6. Overengineering small applications. Seven patterns do not mean seven patterns should appear in every service.

7. Assuming patterns solve distributed transactions. They do not automatically provide consistency, retries, idempotency, or compensation.

8. Ignoring concurrency and lifecycle concerns. Shared objects, caches, proxies, and lazy initialization all require careful design in production.

9. Memorizing class diagrams without understanding intent. The interviewer may change the scenario; understanding the underlying problem helps you adapt.

## 15. A practical architect's checklist

Before introducing a structural design pattern, ask yourself:

* What exact design problem am I solving?

* Which part of the system changes frequently?

* Can I reduce coupling with a simple interface and composition?

* Does the pattern isolate a genuine boundary or merely add another layer?

* Are failure handling, security, and concurrency still correct?

* Will the new design be easier to test and maintain?

* Can I explain why a simpler solution is insufficient?

* Have I considered performance and operational complexity?

A useful rule of thumb: introduce abstractions in response to real requirements or credible variation, not simply because a pattern exists.

## 16. Final revision cheat sheet

Adapter

Translate

Make incompatible interfaces work together.

Memory cue: travel plug adapter.

Decorator

Enhance

Add optional behavior through wrapping.

Memory cue: coffee with extra toppings.

Facade

Simplify

Offer one simple entry point to a complex subsystem.

Memory cue: hotel receptionist.

Proxy

Control

Control access to a real object.

Memory cue: security guard.

Composite

Group

Treat an individual and a group uniformly.

Memory cue: files and folders.

Bridge

Separate

Allow an abstraction and its implementation to evolve independently.

Memory cue: remote controls and devices.

Flyweight

Share

Reduce memory by sharing common intrinsic state.

Memory cue: shared character glyphs.

## 17. How to answer any design-pattern question in an architect interview

Use this five-step answer structure:

1. Problem: Explain the design pressure or limitation.

2. Pattern: Name the pattern and define its intent.

3. Design: Explain the relationships between the classes or components.

4. Trade-offs: Discuss complexity, performance, maintainability, and alternatives.

5. Production application: Give a realistic Java/Spring Boot example and mention operational concerns.

For example:

> "I would use the Adapter pattern to isolate the third-party payment provider from our internal payment contract. Each provider would have its own adapter implementing a common interface. This keeps business logic independent of vendor-specific APIs and simplifies provider replacement. I'd also normalize errors and amounts, and handle timeouts, idempotency, and reconciliation. The trade-off is additional abstraction and mapping code, so I would introduce it where the integration boundary genuinely requires it."

That is a much stronger senior-level answer than simply saying, "Adapter converts one interface into another."

The key takeaway: Structural design patterns are not seven definitions to memorize. They are seven ways to reason about the relationships between objects. Once you recognize whether your problem is about compatibility, enhancement, simplification, access, hierarchy, independent variation, or memory sharing, choosing the right pattern becomes much easier.
