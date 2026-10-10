# SOLID Principles in Java: The Complete Interview Guide for Senior Engineers & Technical Architects

A Medium-style tutorial · Beginner-friendly explanations · Java & Spring Boot · Production scenarios · Senior/Architect-level interview preparation

Imagine you are building an online shopping application. Initially, it has only a few features: customers can browse products, place orders, and make payments.

As the business grows, you add discount coupons, multiple payment gateways, notifications, refunds, and new delivery providers.

Suddenly, every small change breaks something else. Adding a new payment method requires modifying five existing classes. Testing an order requires a real database. A developer changes one implementation and accidentally breaks another.

This is the kind of problem SOLID principles help you prevent.

SOLID is a set of five object-oriented design principles that help us write software that is easier to maintain, test, extend, and understand.

For a Technical Architect, SOLID is not about memorizing five definitions. It is about making design decisions that allow a system to evolve without becoming fragile.

## 1. What is SOLID?

SOLID stands for:

# S

Single Responsibility Principle (SRP)

One class should have one well-defined responsibility and one primary reason to change.

# O

Open/Closed Principle (OCP)

Software should be open for extension but closed for modification.

# L

Liskov Substitution Principle (LSP)

A subtype should be usable wherever its parent type is expected without breaking correctness.

# I

Interface Segregation Principle (ISP)

Clients should not be forced to depend on methods they do not need.

# D

Dependency Inversion Principle (DIP)

High-level business logic and low-level implementation details should depend on abstractions, not directly on each other.

### The easiest way to remember SOLID

| Principle | Simple question                                                     |
| --------- | ------------------------------------------------------------------- |
| SRP       | Is this class doing too many unrelated jobs?                        |
| OCP       | Can I add a new feature without repeatedly changing existing logic? |
| LSP       | Can I replace this implementation without breaking the application? |
| ISP       | Am I forcing a class to implement methods it doesn't need?          |
| DIP       | Does my business logic depend directly on a specific technology?    |

### Why does SOLID matter in production?

Consider a payment service supporting only credit cards today.

Tomorrow, the business requests UPI, net banking, wallets, and a new international payment provider.

Without good design, you might modify the same service repeatedly, introduce nested `if-else` statements, and accidentally break existing payment methods.

With appropriate SOLID principles, you can introduce new implementations behind stable interfaces, test them independently, and minimize changes to existing business logic.

However, SOLID is guidance, not a law that every class must follow mechanically. Applying every principle to a tiny, stable program can create unnecessary interfaces, abstractions, and complexity.

# 2. Single Responsibility Principle (SRP)

## 2.1 What is SRP?

The Single Responsibility Principle says:

> A class should have one reason to change.

This principle is commonly associated with Robert C. Martin, also known as Uncle Bob.

Notice that it does not say a class can have only one method.

A class can have ten methods and still follow SRP if those methods serve the same cohesive responsibility.

### Explain it like I'm 10

Imagine a school teacher who teaches students, drives the school bus, cooks lunch, repairs computers, and manages school accounts.

If the bus breaks down, the teacher's work is affected. If the accounting process changes, the teacher must change their routine again.

It would be better to have separate people responsible for teaching, driving, cooking, and accounting.

Software classes work similarly.

## 2.2 Bad example: One class doing everything

Suppose an online shopping application has an `OrderService` that:

* Calculates the order total.

* Saves the order to a database.

* Generates an invoice PDF.

* Sends an email to the customer.

Java

```
public class OrderService {

    public void placeOrder(Order order) {
        double total = calculateTotal(order);

        // Save order to database
        // Generate invoice PDF
        // Send email notification
    }

    private double calculateTotal(Order order) {
        return order.getItems().stream()
                .mapToDouble(item ->
                        item.getPrice() * item.getQuantity())
                .sum();
    }
}
```

At first glance, this looks simple. But think about the reasons this class might change:

1. The business changes its pricing rules.

2. The database schema changes.

3. The invoice format changes.

4. The email provider changes.

Four unrelated reasons can force modifications to one class.

This creates a maintenance problem: a change to invoice generation might accidentally affect order placement.

## 2.3 Better example: Separate responsibilities

Java

```
public class OrderCalculator {

    public double calculateTotal(Order order) {
        return order.getItems().stream()
                .mapToDouble(item ->
                        item.getPrice() * item.getQuantity())
                .sum();
    }
}
```

Java

```
public class OrderRepository {

    public void save(Order order) {
        // Persist order to database
    }
}
```

Java

```
public class InvoiceService {

    public void generateInvoice(Order order) {
        // Generate invoice
    }
}
```

Java

```
public class NotificationService {

    public void sendOrderConfirmation(Order order) {
        // Send email or another notification
    }
}
```

Now each class has a clearer responsibility.

The overall order workflow can still coordinate them through an application service:

Java

```
public class OrderApplicationService {

    private final OrderCalculator calculator;
    private final OrderRepository repository;
    private final InvoiceService invoiceService;
    private final NotificationService notificationService;

    public OrderApplicationService(
            OrderCalculator calculator,
            OrderRepository repository,
            InvoiceService invoiceService,
            NotificationService notificationService) {
        this.calculator = calculator;
        this.repository = repository;
        this.invoiceService = invoiceService;
        this.notificationService = notificationService;
    }

    public void placeOrder(Order order) {
        calculator.calculateTotal(order);
        repository.save(order);
        invoiceService.generateInvoice(order);
        notificationService.sendOrderConfirmation(order);
    }
}
```

The application service coordinates the workflow; it does not need to implement every technical detail itself.

This example illustrates a design direction, not a requirement to create one class for every method.

## 2.4 How to identify an SRP violation

Ask these questions:

* Does the class contain business rules, database access, and presentation formatting all mixed together?

* Do changes from different business departments frequently modify the same class?

* Is the class difficult to unit test without starting external systems?

* Does the class have many unrelated dependencies?

* Is the class name vague, such as `UtilityManager`, `CommonService`, or `HelperService`, because it does too many things?

A common warning sign is a class that changes whenever unrelated parts of the business change.

## 2.5 Production scenarios

| Scenario            | Better responsibility separation                                       |
| ------------------- | ---------------------------------------------------------------------- |
| Payment processing  | Payment orchestration, provider integration, and payment persistence   |
| File import         | File parsing, record validation, data enrichment, and database writing |
| Notification system | Notification orchestration, message formatting, and delivery           |
| Reporting           | Business calculations, report formatting, and file export              |
| Authentication      | Credential verification, token creation, and audit logging             |

These responsibilities can belong to separate classes within the same service or to separate modules. SRP does not mean every responsibility requires a separate microservice.

For example, a file-import service might have one orchestration service and separate reader, validator, enricher, and writer components. They can all remain in the same deployable application.

## 2.6 Best cases and worst cases

Best case for applying SRP

* The class is expected to evolve in several different directions.

* Different rules have different owners.

* Components need independent unit testing.

* The system has complicated workflows or integrations.

Worst case: Overengineering

Imagine splitting a class containing a single stable calculation into six classes that merely forward calls to one another.

You increase navigation, dependencies, and maintenance overhead without gaining meaningful independence.

The objective is high cohesion, not the maximum number of classes.

## 2.7 Senior-level interview questions

Q1. Does SRP mean one class should have only one method?

No. It means its methods should contribute to one cohesive responsibility. A repository can have `save()`, `findById()`, and `delete()` because they all concern persistence.

Q2. Can a service class coordinate multiple operations and still follow SRP?

Yes. An application service can coordinate a use case, such as placing an order. It becomes problematic when it also owns unrelated implementation details, such as PDF generation, database mechanics, and provider-specific integration logic.

Q3. What is the relationship between SRP and microservices?

They are different levels of design. SRP helps define cohesive classes and modules; service boundaries involve business capabilities, data ownership, deployment, communication, and operational trade-offs.

Q4. How do you detect SRP violations in a large codebase?

Look for unrelated reasons for change, high coupling, low cohesion, excessive dependencies, complicated tests, and classes with several unrelated groups of methods. Metrics such as class size and change history can help, but no single metric proves a violation.

Interview-ready answer:

"SRP means a class should have one cohesive responsibility and one primary reason to change. I apply it by separating business rules, persistence, external integrations, and formatting where those concerns evolve independently. I avoid splitting stable, simple code into unnecessary abstractions."

# 3. Open/Closed Principle (OCP)

## 3.1 What is OCP?

The Open/Closed Principle states:

> Software entities should be open for extension but closed for modification.

In simple words:

* Open for extension: You should be able to add new behavior.

* Closed for modification: You should not need to keep changing stable, existing code every time a new variation is introduced.

This does not mean existing code can never be changed. Bug fixes, security patches, and legitimate changes to existing requirements still require modifications.

The goal is to protect stable, well-tested code from unnecessary changes when adding new features.

### Explain it like I'm 10

Imagine a television with a USB port.

You can plug in a new compatible device without rebuilding the television.

Similarly, good software design lets you add new implementations without rewriting the central business workflow.

## 3.2 Bad example: Payment service with repeated modifications

Java

```
public class PaymentService {

    public void pay(String type, double amount) {

        if ("CARD".equals(type)) {
            System.out.println("Paying by card");
        } else if ("UPI".equals(type)) {
            System.out.println("Paying by UPI");
        } else if ("WALLET".equals(type)) {
            System.out.println("Paying by wallet");
        }
    }
}
```

Today, it supports three payment methods.

Tomorrow, the business asks for net banking. A developer adds another condition.

Next month, it requests an international payment provider. Another condition appears.

Over time, this can become a large conditional statement with provider-specific validation, authentication, retries, error handling, and response mapping.

Every change risks affecting existing methods.

## 3.3 Better example: Use an abstraction

First, define a common interface:

Java

```
public interface PaymentProcessor {
    void pay(double amount);
}
```

Create independent implementations:

Java

```
public class CardPaymentProcessor
        implements PaymentProcessor {

    @Override
    public void pay(double amount) {
        System.out.println("Paying by card: " + amount);
    }
}
```

Java

```
public class UpiPaymentProcessor
        implements PaymentProcessor {

    @Override
    public void pay(double amount) {
        System.out.println("Paying by UPI: " + amount);
    }
}
```

Java

```
public class WalletPaymentProcessor
        implements PaymentProcessor {

    @Override
    public void pay(double amount) {
        System.out.println("Paying by wallet: " + amount);
    }
}
```

The business workflow uses the abstraction:

Java

```
public class CheckoutService {

    public void checkout(
            PaymentProcessor processor,
            double amount) {

        processor.pay(amount);
    }
}
```

Now adding a new payment method generally means creating another implementation:

Java

```
public class NetBankingPaymentProcessor
        implements PaymentProcessor {

    @Override
    public void pay(double amount) {
        System.out.println(
                "Paying through net banking: " + amount);
    }
}
```

The checkout workflow doesn't need a new conditional branch just to support the new processor.

### How OCP works

CheckoutService

Stable business workflow

Depends on the abstraction

PaymentProcessor interface

Card

UPI

New methods

One important detail: this example assumes the caller can supply the correct processor. A real application also needs a reliable selection mechanism.

## 3.4 How do you select an implementation in Spring Boot?

Spring can inject a strategy through an interface.

For example:

Java

```
@Service
public class CheckoutService {

    private final PaymentProcessor paymentProcessor;

    public CheckoutService(
            PaymentProcessor paymentProcessor) {
        this.paymentProcessor = paymentProcessor;
    }
}
```

If Spring discovers several `PaymentProcessor` beans, it cannot choose one automatically without additional configuration. You can use `@Qualifier`, `@Primary`, or a registry of implementations selected using a business key.

For example, a registry can map `"UPI"` to `UpiPaymentProcessor` and `"CARD"` to `CardPaymentProcessor`.

Java

```
Map<String, PaymentProcessor> processors;
```

In a production system, validate the keys, handle unsupported methods explicitly, and ensure each implementation is configured correctly.

## 3.5 Where should you use OCP?

OCP is especially useful when the system has known extension points.

| Business requirement          | Possible extension point |
| ----------------------------- | ------------------------ |
| New payment providers         | `PaymentProcessor`       |
| New discount rules            | `DiscountPolicy`         |
| New notification channels     | `NotificationSender`     |
| New file formats              | `FileParser`             |
| New authentication mechanisms | Authentication strategy  |
| New pricing algorithms        | `PricingStrategy`        |
| New shipping providers        | `ShippingProvider`       |

For example, if your enterprise file-import application receives CSV files today and Excel files tomorrow, a `FileParser` abstraction can let you add an Excel parser without changing the rest of the import workflow.

However, an interface is valuable only when it creates a useful boundary. An abstraction for a feature that will never vary may add unnecessary complexity.

## 3.6 OCP and the Strategy design pattern

The Strategy pattern is a common way to implement OCP.

It defines a common contract and provides multiple interchangeable algorithms or behaviors.

Examples include:

* Different pricing algorithms.

* Different payment providers.

* Different retry policies.

* Different routing strategies.

The pattern moves variation into separate implementations rather than spreading conditional logic throughout the application.

## 3.7 OCP pitfalls

Pitfall 1: Adding an interface for everything

An interface with one implementation is not automatically bad, but adding layers without an anticipated variation or useful testing boundary may create needless complexity.

Pitfall 2: Thinking OCP means no modification

Existing code may still need changes when business requirements genuinely change. OCP is about minimizing unnecessary changes to stable components.

Pitfall 3: Forgetting configuration

Adding a new implementation doesn't help if the application cannot discover or select it.

Pitfall 4: Ignoring security and correctness

A new payment implementation must satisfy the same security, validation, logging, and failure-handling requirements as existing implementations.

Pitfall 5: Creating overly generic frameworks

Do not predict every possible future requirement. Build extension points around actual or reasonably anticipated variation.

## 3.8 Senior-level interview questions

Q1. How does OCP reduce regression risk?

It limits the changes needed in established business workflows when new variants are added. New implementations can be tested independently, while existing tests continue to protect the stable behavior.

Q2. Does using an interface automatically satisfy OCP?

No. If the central service still contains a large `if-else` or `switch` statement that must change for every implementation, the design may still violate OCP.

Q3. Is OCP always worth applying?

No. Apply it when the behavior is likely to vary or when a stable boundary offers a concrete benefit. For simple, stable functionality, direct implementation may be clearer.

Q4. How would you apply OCP to a notification service?

Define a notification-channel contract, such as `NotificationSender`, with implementations for email, SMS, and push notifications. Use a dispatcher or routing registry to select the appropriate implementation. Keep the business workflow independent of channel-specific delivery code.

Interview-ready answer:

"OCP encourages us to isolate frequently changing behavior behind stable abstractions. For example, I would use a payment strategy interface with provider-specific implementations so a new payment method can be introduced without modifying the core checkout workflow. I would introduce that abstraction where variation justifies the extra complexity."

# 4. Liskov Substitution Principle (LSP)

## 4.1 What is LSP?

The Liskov Substitution Principle states that objects of a subtype should be usable wherever objects of the base type are expected, without breaking the correctness of the program.

In simple terms:

If class B claims to be a kind of class A, replacing A with B should not unexpectedly break the code that uses A.

The principle is associated with Barbara Liskov.

### Explain it like I'm 10

Suppose you have a universal remote control designed to turn a television on and off.

You replace one compatible television with another. The same remote commands should still work according to the remote's contract.

If the replacement device claims to be a compatible television but throws an error whenever you turn it on, something is wrong with the substitution.

In software, inheritance creates expectations. LSP tells us not to break those expectations.

## 4.2 Bad example: The classic Rectangle and Square problem

Suppose we have a `Rectangle` class:

Java

```
public class Rectangle {

    protected int width;
    protected int height;

    public void setWidth(int width) {
        this.width = width;
    }

    public void setHeight(int height) {
        this.height = height;
    }

    public int area() {
        return width * height;
    }
}
```

Now create a `Square` class that inherits from `Rectangle`:

Java

```
public class Square extends Rectangle {

    @Override
    public void setWidth(int width) {
        this.width = width;
        this.height = width;
    }

    @Override
    public void setHeight(int height) {
        this.width = height;
        this.height = height;
    }
}
```

Mathematically, a square is a rectangle. So the inheritance looks reasonable.

But look at the behavior expected by a caller:

Java

```
public static void main(String[] args) {

    Rectangle rectangle = new Rectangle();

    rectangle.setWidth(5);
    rectangle.setHeight(4);

    System.out.println(rectangle.area()); // 20
}
```

Now substitute a square:

Java

```
Rectangle rectangle = new Square();

rectangle.setWidth(5);
rectangle.setHeight(4);

System.out.println(rectangle.area()); // 16
```

The caller expects to set the width to 5 and the height to 4 independently. The square overrides that behavior by forcing both dimensions to be equal.

The problem is not the mathematics. It is that `Square` cannot preserve the behavioral expectations of this mutable `Rectangle` contract.

## 4.3 Better design: Model the behavior honestly

If the application only needs shapes that calculate areas, use a common abstraction:

Java

```
public interface Shape {
    int area();
}
```

Java

```
public class Rectangle implements Shape {

    private final int width;
    private final int height;

    public Rectangle(int width, int height) {
        this.width = width;
        this.height = height;
    }

    @Override
    public int area() {
        return width * height;
    }
}
```

Java

```
public class Square implements Shape {

    private final int side;

    public Square(int side) {
        this.side = side;
    }

    @Override
    public int area() {
        return side * side;
    }
}
```

Now both shapes implement the same behavior that clients need: calculating area.

The lesson is not that `Square` can never extend `Rectangle`. It is that the chosen parent contract must support the behavior of its subtypes.

## 4.4 The three important LSP rules

### Rule 1: Do not strengthen preconditions

A precondition is what must be true before a method can execute successfully.

If the parent accepts any positive integer, a child should not suddenly require the number to be greater than 100.

Example:

* Parent: `withdraw(amount)` accepts any positive amount within the account's available balance.

* Child: rejects all withdrawals below ₹1,000.

If callers reasonably rely on the parent's accepted inputs, the child has narrowed the contract and may violate LSP.

### Rule 2: Do not weaken postconditions

A postcondition describes what a method guarantees after successful execution.

Suppose a parent method promises that `save()` persists an entity before returning successfully.

A child implementation that simply stores the entity in memory and loses it after a restart may not satisfy that contract if durable persistence was promised.

The child must preserve the guarantees expected by the caller.

### Rule 3: Preserve invariants

An invariant is a condition that must remain true throughout the object's valid lifetime.

Examples:

* An account balance must never become negative if overdrafts are forbidden.

* An order marked `PAID` must have a successful payment recorded.

* A validated customer record must satisfy the documented validation rules.

A subtype must not introduce behavior that breaks the parent's guarantees or the domain's required invariants.

## 4.5 Other LSP violations in production

| Example                                                                                          | Problem                                         |
| ------------------------------------------------------------------------------------------------ | ----------------------------------------------- |
| Read-only repository extends a fully writable repository                                         | It throws exceptions for `save()` or `delete()` |
| Unsupported cloud storage provider implements a contract promising durable writes                | It silently discards data                       |
| A child payment processor returns success before payment is actually confirmed                   | It breaks the payment contract                  |
| A cache implementation returns stale data despite a contract requiring strongly consistent reads | It violates consistency expectations            |
| A subtype throws unexpected exceptions for otherwise valid inputs                                | It changes the expected failure behavior        |

These examples are violations only when the stated behavior is part of the relevant contract. A read-only repository, for instance, is perfectly valid if its interface promises read operations only.

## 4.6 How do you detect an LSP violation?

Ask:

1. Does the subtype reject inputs accepted by the parent?

2. Does it break promises made by the parent?

3. Does it return results with different meanings?

4. Does it throw unexpected exceptions for supported operations?

5. Does replacing one implementation with another break existing tests or callers?

6. Does the child require callers to check its concrete type before using it?

A frequent warning sign is code such as:

Java

```
if (processor instanceof SpecialPaymentProcessor) {
    // Special handling because the normal contract doesn't work
}
```

Type checks are not automatically wrong. But if callers repeatedly inspect implementation types to avoid broken behavior, the abstraction may be poorly designed.

## 4.7 LSP and polymorphism

Polymorphism lets us write:

Java

```
PaymentProcessor processor = getProcessor();
processor.pay(1000);
```

The caller does not need to know whether the processor is a card processor, a UPI processor, or another implementation.

LSP ensures that each implementation respects the contract the caller relies on.

Without LSP, polymorphism exists syntactically, but replacing implementations becomes dangerous.

## 4.8 Best cases and worst cases

LSP is especially important when:

* You use inheritance.

* Multiple implementations share an interface.

* A framework accepts user-provided implementations.

* You replace providers or adapters.

* You have plug-in architectures or dependency injection.

* Your system relies on polymorphic business rules.

A common mistake: forcing unrelated classes into an inheritance hierarchy just because their names sound similar.

Prefer composition or a smaller interface when two classes do not share the same behavioral contract.

## 4.9 Senior-level interview questions

Q1. Is LSP only about inheritance?

It is most commonly explained through inheritance, but the underlying principle applies to substitutability across implementations of an abstraction. Interface implementations can violate behavioral contracts too.

Q2. Why is the Square–Rectangle example controversial?

Because mathematical subtype relationships do not automatically imply valid behavioral subtyping. A mutable rectangle API that permits independent width and height changes creates expectations a square cannot preserve.

Q3. How do you test LSP?

Write contract tests for the abstraction and run them against every implementation. Verify accepted inputs, output guarantees, failure behavior, state invariants, and relevant concurrency or consistency guarantees.

Q4. What is the difference between LSP and OCP?

OCP focuses on adding behavior without repeatedly modifying stable code. LSP ensures that the new implementations can actually replace existing ones without breaking their contracts. They complement each other.

Interview-ready answer:

"LSP ensures behavioral substitutability. An implementation must honor the preconditions, postconditions, invariants, and failure semantics expected by its abstraction. In practice, I use clear contracts and shared contract tests so that adding or replacing an implementation does not unexpectedly break existing consumers."

# 5. Interface Segregation Principle (ISP)

## 5.1 What is ISP?

The Interface Segregation Principle states:

> Clients should not be forced to depend on methods they do not use.

In simple words, prefer several small, focused interfaces over one large interface that forces every implementation to support unrelated operations.

### Explain it like I'm 10

Imagine a restaurant employee who is hired only to deliver food.

The restaurant gives the employee a job contract requiring them to cook, clean, manage accounts, repair electrical equipment, and deliver food.

That contract is unnecessarily broad. A delivery employee should not have to perform unrelated jobs.

Similarly, a Java class should not be forced to implement methods it does not meaningfully support.

## 5.2 Bad example: One giant interface

Imagine an enterprise device-management application:

Java

```
public interface Machine {

    void print();

    void scan();

    void fax();

    void staple();
}
```

A multifunction printer supports all four operations. But a basic printer supports only printing.

What happens if the basic printer implements this interface?

Java

```
public class BasicPrinter implements Machine {

    @Override
    public void print() {
        System.out.println("Printing...");
    }

    @Override
    public void scan() {
        throw new UnsupportedOperationException();
    }

    @Override
    public void fax() {
        throw new UnsupportedOperationException();
    }

    @Override
    public void staple() {
        throw new UnsupportedOperationException();
    }
}
```

The class is forced to implement operations it cannot perform.

This creates misleading contracts and potentially exposes runtime failures to callers.

## 5.3 Better example: Split the interface

Create smaller interfaces based on capabilities:

Java

```
public interface Printer {
    void print();
}
```

Java

```
public interface Scanner {
    void scan();
}
```

Java

```
public interface FaxMachine {
    void fax();
}
```

Java

```
public interface Stapler {
    void staple();
}
```

A basic printer implements only the capability it supports:

Java

```
public class BasicPrinter implements Printer {

    @Override
    public void print() {
        System.out.println("Printing...");
    }
}
```

A multifunction printer can implement several:

Java

```
public class MultiFunctionPrinter
        implements Printer, Scanner, FaxMachine {

    @Override
    public void print() {
        System.out.println("Printing...");
    }

    @Override
    public void scan() {
        System.out.println("Scanning...");
    }

    @Override
    public void fax() {
        System.out.println("Faxing...");
    }
}
```

Now each implementation depends only on the capabilities relevant to it.

## 5.4 ISP in a Spring Boot application

Suppose your application has a broad user-management interface:

Java

```
public interface UserService {

    User createUser(User user);

    User findUser(Long id);

    void deleteUser(Long id);

    void resetPassword(Long id);

    void exportAllUsers();

    void blockUser(Long id);
}
```

This might become problematic if different clients require different capabilities.

For example:

* A customer-facing profile service needs to read and update profiles.

* An administrative service needs to block accounts and manage users.

* A reporting service needs to read users and export reports.

You could split the contract:

Java

```
public interface UserReader {
    User findUser(Long id);
}
```

Java

```
public interface UserWriter {
    User createUser(User user);
}
```

Java

```
public interface UserAdministrator {
    void deleteUser(Long id);

    void blockUser(Long id);
}
```

Java

```
public interface UserExporter {
    void exportAllUsers();
}
```

Implementations can expose the capabilities they actually support.

One caution: do not split every service into interfaces just because several clients exist. First understand the actual dependency and change boundaries. The extra interfaces should make the system easier to evolve, secure, or test.

## 5.5 ISP vs SRP: What is the difference?

This is a common interview question.

| SRP                                                           | ISP                                                         |
| ------------------------------------------------------------- | ----------------------------------------------------------- |
| Focuses on class responsibilities                             | Focuses on interface contracts                              |
| Asks whether a class has unrelated reasons to change          | Asks whether clients are forced to depend on unused methods |
| Often leads to cohesive classes                               | Often leads to focused interfaces                           |
| Example: separating invoice generation from order persistence | Example: separating printing from scanning                  |

They are related but not identical.

A class might have one coherent purpose yet implement a very broad interface that contains operations irrelevant to it. That can violate ISP even when the class is cohesive.

## 5.6 ISP and security: An architect's perspective

Consider an application in which customer-facing code receives a dependency that exposes both reading and administrative operations.

Even if the customer-facing code only uses `findUser()`, the broad interface makes administrative operations available through that dependency.

A smaller interface can reduce accidental coupling and make the intended capability clearer.

However, ISP is not a security boundary by itself. You still need proper authorization, authentication, and enforcement in the service layer.

In a distributed architecture, separate interfaces can also help define API capabilities, but they do not automatically require separate REST endpoints, modules, or microservices.

## 5.7 Best cases and worst cases

Best cases for ISP

* Many implementations support different subsets of operations.

* Different clients use distinct groups of methods.

* Interfaces have become very large and difficult to implement.

* Mocking a dependency requires implementing many irrelevant methods.

* New methods repeatedly break existing implementations.

Worst case: Excessive interface fragmentation

Creating dozens of interfaces with one method each can make a codebase harder to understand.

The goal is to define meaningful capability boundaries, not to make every interface as small as technically possible.

## 5.8 Senior-level interview questions

Q1. Can one class implement multiple interfaces?

Yes. This is one of the most common ways to implement ISP. A class can expose several independent capabilities without forcing every other implementation to support them.

Q2. Is a large interface always a violation?

No. An interface with many methods can still represent one cohesive contract that every implementation genuinely supports. The problem is forcing clients or implementations to depend on irrelevant capabilities.

Q3. How does ISP improve testing?

Focused interfaces make mocks and fakes smaller. Tests need only provide the behavior used by the component under test.

Q4. How does ISP help microservice design?

It encourages consumers to depend on the operations they need instead of broad, unrelated contracts. This can inform API and module boundaries, although ISP alone does not determine service decomposition.

Interview-ready answer:

"ISP means clients should depend only on the capabilities they require. I use it to avoid broad interfaces that force implementations to provide unsupported operations. In enterprise applications, focused interfaces can reduce coupling, simplify testing, and make intended permissions and responsibilities clearer."

# 6. Dependency Inversion Principle (DIP)

## 6.1 What is DIP?

The Dependency Inversion Principle states:

1. High-level modules should not depend directly on low-level modules. Both should depend on abstractions.

2. Abstractions should not depend on implementation details. Implementation details should depend on abstractions.

This is one of the most important principles for a Technical Architect because it influences how business logic connects to databases, cloud services, messaging systems, and external APIs.

### Explain it like I'm 10

Imagine a phone charger that works only with one specific wall socket.

If you move to another country with a different socket, you need a different charger or adapter.

Now imagine a standardized charging interface. Different chargers and power sources can work together through that standard.

In software, abstractions play a similar role: they reduce the dependence of core business logic on specific technologies.

## 6.2 First understand high-level and low-level modules

A high-level module contains important business rules or application workflows.

Examples:

* Order processing.

* Payment orchestration.

* Customer onboarding.

* Loan approval.

* File-import coordination.

A low-level module handles implementation details.

Examples:

* PostgreSQL repository.

* Redis cache adapter.

* Pub/Sub message publisher.

* Email provider client.

* HTTP client calling an external service.

A common design problem is when business logic directly constructs and controls all these concrete implementations.

## 6.3 Bad example: Business logic directly depends on a concrete database

Java

```
public class OrderService {

    private final MySqlOrderRepository repository =
            new MySqlOrderRepository();

    public void placeOrder(Order order) {
        repository.save(order);
    }
}
```

What is wrong?

`OrderService` creates a specific MySQL repository directly.

If you migrate from MySQL to PostgreSQL, switch to a remote persistence service, or want to use a fake repository for testing, you may need to modify the business class.

The problem is not MySQL itself. The problem is that the high-level workflow is tightly coupled to one implementation.

## 6.4 Better example: Depend on an abstraction

Define a repository contract:

Java

```
public interface OrderRepository {
    void save(Order order);
}
```

The business service depends on the contract:

Java

```
public class OrderService {

    private final OrderRepository repository;

    public OrderService(OrderRepository repository) {
        this.repository = repository;
    }

    public void placeOrder(Order order) {
        repository.save(order);
    }
}
```

The database implementation provides the details:

Java

```
public class PostgresOrderRepository
        implements OrderRepository {

    @Override
    public void save(Order order) {
        // Save order using PostgreSQL
    }
}
```

A test implementation can provide another version:

Java

```
public class FakeOrderRepository
        implements OrderRepository {

    @Override
    public void save(Order order) {
        // Keep the order in memory for testing
    }
}
```

Now `OrderService` does not need to know which repository implementation it receives.

### The dependency direction

High-level business logic

OrderService

Depends on

OrderRepository interface

Stable abstraction

Implementation conforms to the contract

PostgreSQL adapter

Fake test adapter

The source-code dependency points toward the abstraction. The implementation conforms to that abstraction.

That is the essence of dependency inversion.

## 6.5 Dependency Inversion vs Dependency Injection

These terms are often confused in interviews.

Dependency Inversion Principle (DIP) is a design principle.

Dependency Injection (DI) is a technique for supplying an object with its dependencies rather than having it construct them directly.

For example:

Java

```
public OrderService(OrderRepository repository) {
    this.repository = repository;
}
```

The constructor allows the dependency to be supplied from outside the class.

Spring's dependency injection container can create the repository implementation and inject it into the service.

| DIP                                                 | DI                                                 |
| --------------------------------------------------- | -------------------------------------------------- |
| A principle about dependency direction              | A technique for supplying dependencies             |
| Encourages business logic to depend on abstractions | Provides objects with the required implementations |
| Can be applied without a framework                  | Can be performed manually or by a framework        |
| Improves architectural flexibility                  | Helps wire and manage object dependencies          |

You can use dependency injection and still violate DIP. For example, injecting a concrete `MySqlOrderRepository` directly into `OrderService` is DI, but the service still depends on a concrete implementation.

## 6.6 DIP in Spring Boot

A common Spring Boot implementation looks like this:

Java

```
public interface NotificationSender {
    void send(String recipient, String message);
}
```

Java

```
@Service
public class EmailNotificationSender
        implements NotificationSender {

    @Override
    public void send(String recipient, String message) {
        // Send email through an email provider
    }
}
```

Java

```
@Service
public class OrderNotificationService {

    private final NotificationSender sender;

    public OrderNotificationService(
            NotificationSender sender) {
        this.sender = sender;
    }

    public void notifyOrderPlaced(String email) {
        sender.send(email, "Your order has been placed.");
    }
}
```

Spring can inject the implementation automatically when there is exactly one matching candidate, or when the configuration specifies the intended bean.

If you later add an SMS implementation, you must also decide how to select the channel. Multiple beans of the same type do not automatically imply the correct business choice.

## 6.7 DIP in a cloud-native architecture

Imagine an application that publishes order events to Google Cloud Pub/Sub.

A tightly coupled implementation might create the Pub/Sub client directly inside the order business service.

Java

```
public class OrderService {
    // Direct dependency on cloud-specific publishing code
}
```

A more flexible design introduces an application-level abstraction:

Java

```
public interface OrderEventPublisher {
    void publish(OrderCreatedEvent event);
}
```

A GCP adapter implements the interface using Pub/Sub.

The business workflow depends on `OrderEventPublisher`, not on the details of Pub/Sub.

Potential benefits:

* Easier unit testing.

* Clear separation between business rules and cloud SDK code.

* Simpler migration or integration with another messaging provider.

* Centralized handling of provider-specific errors and configuration.

But abstraction does not make migration free. Message ordering, delivery guarantees, authentication, schema compatibility, retries, and operational behavior may differ across providers.

An architect should preserve the semantics the application actually needs, not merely match method signatures.

## 6.8 DIP and Clean Architecture

DIP is a major influence on Clean Architecture and Hexagonal Architecture (Ports and Adapters).

A simplified architecture might look like this:

Infrastructure

PostgreSQL · Redis · Pub/Sub · External REST APIs

Adapters implement or invoke ports

Application layer

Use cases · Workflow orchestration · Ports

Domain layer

Business rules · Entities · Domain policies

In this architecture, core business rules should not need to import framework-specific database or messaging classes.

The exact layering varies by system, but the central goal is to keep business rules independent of infrastructure details.

## 6.9 Best cases and worst cases

Best cases for DIP

* Business rules must be tested independently.

* You integrate with several external providers.

* Infrastructure technologies may change.

* The system has complex domain logic.

* You need clear boundaries between modules or teams.

Worst case: Abstraction without value

Suppose a class simply calls another class, which calls an interface, which calls one implementation, and every layer exists only to forward a single method.

If there is no meaningful boundary, variation, or testability benefit, the design may be unnecessarily complicated.

A good architect introduces abstractions where they protect important business logic or isolate genuine implementation variation.

## 6.10 Senior-level interview questions

Q1. Does DIP mean every class must use an interface?

No. Use abstractions where they provide value, especially at important architectural boundaries. Simple, stable implementation details do not always need an interface.

Q2. How does DIP improve unit testing?

A business service can receive a fake or mock implementation of a dependency. The test can verify business behavior without starting a real database or calling an external provider.

Q3. Does DIP mean the database must never be called by the application?

No. Applications still need infrastructure to persist data. DIP means the core business logic should depend on a suitable abstraction rather than being tightly coupled to a particular database implementation.

Q4. Can DIP be implemented without Spring?

Yes. Constructor injection can be performed manually. Spring makes object creation, configuration, and dependency wiring easier, but it is not required for the principle.

Q5. Why is DIP important for cloud migrations?

It isolates provider-specific implementation details behind stable application contracts. That can reduce the changes needed in business logic, though differences in provider behavior and operational semantics still require deliberate migration work.

Interview-ready answer:

"DIP ensures high-level business rules don't depend directly on infrastructure implementations. I define stable interfaces at meaningful boundaries, then implement them using adapters such as PostgreSQL repositories or cloud messaging publishers. Dependency injection supplies those implementations, allowing independent testing and reducing infrastructure coupling."

# 7. How all five SOLID principles work together

This is the most important section for an architect-level interview.

Imagine you are designing a notification system for an enterprise application. Whenever a customer submits an application, the system must send a notification.

Initially, only email is supported. Later, the business asks for SMS and push notifications. The service must also handle provider failures and remain easy to test.

Here is how the five principles help.

| Principle | How you apply it                                                                                                           |
| --------- | -------------------------------------------------------------------------------------------------------------------------- |
| SRP       | Separate notification orchestration, message formatting, and channel delivery where they have different reasons to change. |
| OCP       | Add new channel implementations without repeatedly rewriting the core notification workflow.                               |
| LSP       | Ensure every channel implementation respects its contract, including success and failure behavior.                         |
| ISP       | Keep delivery capabilities focused so implementations are not forced to support unrelated operations.                      |
| DIP       | Make the notification workflow depend on an abstraction rather than directly on a specific provider SDK.                   |

## 7.1 Putting it together in Java

Start with a focused abstraction:

Java

```
public interface NotificationSender {
    void send(Notification notification);
}
```

An email implementation:

Java

```
public class EmailNotificationSender
        implements NotificationSender {

    @Override
    public void send(Notification notification) {
        // Deliver notification through email provider
    }
}
```

An SMS implementation:

Java

```
public class SmsNotificationSender
        implements NotificationSender {

    @Override
    public void send(Notification notification) {
        // Deliver notification through SMS provider
    }
}
```

Now create the application service:

Java

```
public class NotificationService {

    private final NotificationSender sender;

    public NotificationService(NotificationSender sender) {
        this.sender = sender;
    }

    public void notifyCustomer(Notification notification) {
        sender.send(notification);
    }
}
```

This design demonstrates DIP because the service depends on an abstraction, OCP because new implementations can be added without rewriting this workflow, and SRP because channel delivery is separate from orchestration.

ISP is reflected in the focused interface, while LSP requires each implementation to honor its behavioral contract.

Architect-level caveat: This is a simplified example with one selected sender. If the application must choose dynamically between email, SMS, and push, you need a routing or strategy-selection mechanism. If it must deliver through multiple channels, you need orchestration for multiple senders.

You should also decide whether `send()` means "accepted for delivery," "handed to the provider," or "successfully delivered." That contract determines what every implementation must guarantee.

# 8. SOLID principles: Quick comparison and common confusion

## 8.1 The five principles at a glance

| Principle | Main focus             | Warning sign                                      | Typical solution                                  |
| --------- | ---------------------- | ------------------------------------------------- | ------------------------------------------------- |
| SRP       | Class responsibility   | Unrelated reasons to change                       | Separate cohesive responsibilities                |
| OCP       | Extensibility          | Every new variant modifies core logic             | Polymorphism, strategies, plug-ins                |
| LSP       | Behavioral correctness | Substitution breaks callers                       | Honor contracts and invariants                    |
| ISP       | Interface design       | Unused or unsupported methods                     | Split into meaningful capabilities                |
| DIP       | Dependency direction   | Business logic constructs concrete infrastructure | Depend on abstractions and inject implementations |

## 8.2 SOLID vs common design patterns

SOLID principles and design patterns are related, but they are not the same thing.

| Design problem                                           | Relevant principle | Possible pattern or technique                 |
| -------------------------------------------------------- | ------------------ | --------------------------------------------- |
| Too many responsibilities in one class                   | SRP                | Composition, separation into services         |
| Adding new behavior requires modifying many conditionals | OCP                | Strategy, Template Method, plug-ins           |
| A subtype breaks expected behavior                       | LSP                | Better contracts, composition, contract tests |
| A giant interface contains unrelated methods             | ISP                | Capability interfaces                         |
| Business logic depends on a concrete provider            | DIP                | Dependency injection, Ports and Adapters      |

A design pattern is a reusable approach to a recurring design problem. SOLID provides principles for evaluating whether a design is likely to remain maintainable.

For example, the Strategy pattern can help implement OCP, but simply using Strategy does not guarantee that all five SOLID principles are satisfied.

# 9. Production scenarios: How an architect applies SOLID

Let's move beyond small code examples and look at realistic enterprise systems.

## Scenario 1: Adding a new payment gateway

Requirement: An application supports one payment provider. The business wants to add another provider.

Without SOLID, the payment service might contain provider-specific conditions, request formats, response handling, and retry rules.

With SOLID:

* SRP: separate payment orchestration from provider integration.

* OCP: add a new provider adapter behind a suitable contract.

* LSP: require each adapter to honor the payment contract.

* ISP: avoid forcing read-only or inquiry integrations to implement unrelated operations.

* DIP: keep payment business rules independent of provider SDKs.

Architect's decision: Keep the payment workflow stable where possible, but don't hide real differences in authentication, idempotency, settlement, or error handling behind an oversimplified interface.

## Scenario 2: Importing millions of customer records

Requirement: A Java application imports large CSV and Excel files into a database, validates records, enriches them through an external REST API, and writes the valid records in batches.

A maintainable design could use:

* `FileReader` for parsing input.

* `RecordValidator` for validation.

* `CustomerEnricher` for external lookups.

* `CustomerWriter` for persistence.

* An import coordinator for workflow and transaction management.

This supports SRP.

If new file formats are introduced, the system can use parser implementations behind a common contract, supporting OCP.

Every parser and writer must honor documented contracts, supporting LSP.

Small, focused contracts avoid forcing parsers to implement database-writing operations, supporting ISP.

The coordinator depends on abstractions instead of directly creating a specific REST client or database adapter, supporting DIP.

Architect's decision: Do not create a separate microservice for every stage by default. Consider throughput, memory, database bottlenecks, retries, data consistency, operational overhead, and deployment boundaries.

## Scenario 3: Replacing a cloud messaging provider

Requirement: The application publishes events through a cloud messaging service. The organization is evaluating a different provider.

DIP can isolate the application from provider SDKs. OCP can allow a new adapter to be introduced. LSP requires the new adapter to honor the same application-level guarantees.

But the architect must compare:

* Delivery guarantees.

* Ordering and partitioning.

* Retry and dead-letter behavior.

* Message-size limits.

* Authentication and authorization.

* Throughput, latency, and cost.

* Schema compatibility and observability.

A common interface alone does not guarantee identical behavior.

## Scenario 4: Growing notification channels

Requirement: The application initially sends email. It later needs SMS, push notifications, and other channels.

A `NotificationSender` abstraction can help isolate channel implementations.

But consider these requirements:

* Some notifications require email and SMS.

* Some are optional and may be sent asynchronously.

* A failed SMS must not cause an already-sent email to be sent twice.

* Delivery attempts need retry limits and audit records.

An architect may separate notification planning, dispatching, provider adapters, and delivery tracking. SRP and DIP help keep these responsibilities independent, while OCP and LSP help new channel implementations fit the system safely.

## Scenario 5: Testing a business service without a database

Requirement: A loan-approval service needs unit tests that run quickly without connecting to production-like infrastructure.

DIP lets the business service receive repository and policy abstractions.

ISP can keep those interfaces focused. SRP keeps database operations separate from loan-approval decisions.

A fake repository can be supplied during unit testing.

Architect's decision: Unit tests validate isolated business behavior; integration tests are still needed to verify SQL, transaction behavior, database constraints, and actual infrastructure integration.

## Scenario 6: A class has become a 2,000-line service

Requirement: A legacy Spring Boot service handles validation, pricing, database queries, REST calls, logging, and notifications.

A poor response is to mechanically split every method into a new class.

A better approach is:

1. Identify business capabilities and reasons for change.

2. Group cohesive operations together.

3. Identify external-system boundaries.

4. Define abstractions only where they provide useful independence.

5. Extract components incrementally.

6. Add tests before changing behavior.

7. Measure coupling, testability, and change impact after each step.

SOLID helps guide the refactoring, but the process should be driven by real responsibilities and change patterns.

# 10. Common mistakes and SOLID anti-patterns

These are particularly useful for senior and architect interviews.

## Mistake 1: "Every class must have an interface"

Incorrect.

An interface is useful when it defines a meaningful contract, supports multiple implementations, isolates a dependency, or provides a useful testing boundary.

A stable value object or a simple internal calculation may not need an interface.

## Mistake 2: "OCP means we never modify existing code"

Incorrect.

You still modify code for bug fixes, security updates, and genuine changes to existing behavior. OCP aims to minimize unnecessary changes when adding new variants.

## Mistake 3: "Inheritance automatically satisfies LSP"

Incorrect.

A child can compile successfully while violating the parent's behavioral contract. LSP is about correct behavior, not merely successful compilation.

## Mistake 4: "Smaller interfaces are always better"

Incorrect.

An interface with many related methods may be a perfectly cohesive contract. Split interfaces when doing so makes dependencies more meaningful and reduces unnecessary coupling.

## Mistake 5: "Using Spring dependency injection means I follow DIP"

Incorrect.

Injecting a concrete infrastructure class can still couple business logic to that implementation. DIP is about the direction of dependencies; DI is a way of supplying them.

## Mistake 6: "SOLID means more classes, more interfaces, and more microservices"

Incorrect.

SOLID aims to control complexity. Excessive abstraction can create indirection, increase maintenance costs, and make a system harder to understand.

## Mistake 7: "SOLID guarantees high performance"

Incorrect.

SOLID does not automatically improve latency, throughput, or memory use. It primarily guides maintainability and dependency design. Performance must still be measured and optimized.

## Mistake 8: "A SOLID design never breaks"

Incorrect.

It can still have bugs, incorrect business rules, concurrency problems, poor data modeling, security vulnerabilities, and operational failures.

SOLID improves design quality; it is not a substitute for testing, observability, secure coding, or sound system architecture.

# 11. Architect-level design review checklist

When reviewing a Java or Spring Boot application, ask the following questions.

## SOLID design review

0 / 12

### Single Responsibility

Are business rules mixed with persistence, formatting, and infrastructure code unnecessarily?

Do unrelated business changes repeatedly affect the same class?

### Open/Closed

Where do we expect new implementations or business-rule variants?

Can we add them without scattering new conditionals through stable workflows?

### Liskov Substitution

Do implementations honor the same input, output, failure, and state guarantees?

Can we run common contract tests against every implementation?

### Interface Segregation

Are consumers forced to depend on operations they do not use?

Do implementations have to throw unsupported-operation exceptions for normal contract methods?

### Dependency Inversion

Does core business logic directly construct or depend on concrete infrastructure classes?

Can we test business rules without real external services?

### Architecture and operations

Are retries, idempotency, transactions, and failure handling clearly defined?

Are abstractions justified by actual change boundaries rather than speculation?

Reset checklist

This checklist is useful during code reviews, architecture discussions, refactoring, and interview answers. Not every negative answer indicates a violation; use the actual business requirements and cost of complexity to decide.

# 12. Frequently asked SOLID interview questions

Here is a quick revision bank.

Q1. What is SOLID, and why do we use it?

SOLID is a group of five object-oriented design principles that help control coupling, improve cohesion, support testing, and make software easier to extend and maintain.

Q2. Which SOLID principle is most important?

There is no universally most important principle. The relevant principle depends on the problem. DIP is particularly useful for infrastructure independence, SRP for responsibility boundaries, and LSP for safe substitution.

Q3. What is the difference between SRP and OCP?

SRP is about having a cohesive responsibility. OCP is about extending behavior without repeatedly modifying stable code.

Q4. How are OCP and DIP related?

DIP helps isolate implementation details behind abstractions. Those abstractions can create extension points that support OCP. Neither principle automatically guarantees the other.

Q5. How are ISP and LSP related?

ISP encourages focused contracts. LSP ensures implementations honor the contracts they implement. A focused interface makes a contract easier to define, but its implementations must still behave correctly.

Q6. Can a project follow SOLID without using inheritance?

Yes. SOLID can be applied through composition, interfaces, dependency injection, and modular design. Deep inheritance hierarchies are not required.

Q7. Can a Spring Boot monolith follow SOLID?

Absolutely. SOLID is primarily about object-oriented design and dependency boundaries. A monolith can have highly cohesive modules and loosely coupled components without becoming a microservice architecture.

Q8. How do SOLID principles relate to low coupling and high cohesion?

SRP encourages cohesion. DIP reduces certain forms of coupling. OCP, LSP, and ISP help create extension points and contracts that avoid unnecessary dependencies.

Q9. Can a design follow SOLID and still be bad?

Yes. It can be overengineered, slow, insecure, poorly distributed, or inconsistent with business requirements. SOLID addresses particular design concerns, not every aspect of software quality.

Q10. How would you introduce SOLID into a legacy system?

Identify high-change and high-risk areas, add characterization tests, locate mixed responsibilities and concrete dependencies, then refactor incrementally. Avoid large rewrites without evidence that they are necessary.

Q11. How do you know when not to apply SOLID?

If an abstraction adds more complexity than it removes, the behavior is stable, and there is no meaningful testing or change-boundary benefit, keep the design simpler. Revisit it when the requirements change.

Q12. How do you evaluate SOLID in a code review?

Look at actual change scenarios, dependency graphs, contracts, testability, and the cost of introducing new behavior. Review concrete evidence rather than checking whether every class has an interface.

# 13. A practical interview exercise

Imagine you are asked this question:

"We have a Spring Boot `OrderService` that calculates discounts, saves orders to PostgreSQL, publishes events to Pub/Sub, and sends email notifications. Every time a new notification channel is added, we modify the service. How would you redesign it using SOLID?"

Try to answer before revealing the suggested approach.

### Your interview challenge

Explain which responsibilities you would separate, which abstractions you would introduce, how you would handle multiple notification channels, and how you would test the design.

Reveal suggested architect-level answer

# 14. Final revision cheat sheet

Memorize these five statements and the problem each one solves.

## S

Single Responsibility Principle

One cohesive responsibility; one primary reason to change.

Problem solved: classes that change for unrelated reasons.

## O

Open/Closed Principle

Extend behavior without repeatedly modifying stable code.

Problem solved: every new variant requires changes throughout the application.

## L

Liskov Substitution Principle

Implementations must preserve the behavioral contract of their abstraction.

Problem solved: swapping implementations breaks existing callers.

## I

Interface Segregation Principle

Clients should not depend on methods they don't need.

Problem solved: large, unfocused interfaces.

## D

Dependency Inversion Principle

Business logic and implementation details depend on abstractions.

Problem solved: business rules are tightly coupled to infrastructure.

## The one-minute interview answer

If an interviewer asks you to explain SOLID as a whole, you can say:

Writing

SOLID is a set of five object-oriented design principles that help make software maintainable, extensible, and testable.

SRP keeps each class focused on a cohesive responsibility. OCP allows us to extend behavior without repeatedly modifying stable code. LSP ensures implementations can replace their abstractions without breaking expected behavior. ISP prevents clients from depending on unnecessary methods. DIP keeps high-level business logic independent of concrete infrastructure implementations.

In a Spring Boot application, I apply these principles through cohesive services, focused interfaces, composition, dependency injection, and clear architectural boundaries. As a Technical Architect, I balance these principles against complexity, performance, business requirements, and operational constraints rather than applying abstractions mechanically.

The key takeaway: SOLID is not about writing more interfaces or creating more classes. It is about making the next change safer, more predictable, and less expensive than it would have been otherwise. That is the mindset interviewers want to see from a Technical Architect.
