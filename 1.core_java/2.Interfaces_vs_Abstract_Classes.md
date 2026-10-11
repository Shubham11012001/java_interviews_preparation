# Interfaces vs Abstract Classes in Java — The Interview Guide

If you have been writing Java for a few years, you have almost certainly used both **interfaces** and **abstract classes**.

But senior-level interviews rarely stop at:

> “What is an interface?”

They usually move toward:

> “Why would you choose an interface here instead of an abstract class?”
> “Can an interface have state?”
> “Why does Java allow multiple interfaces but not multiple class inheritance?”
> “What happens with default methods?”
> “How would you design this in a production microservice?”

So let's build this from **10-year-old level → Java internals → production design → interview scenarios**.

---

# 1. The 10-Second Definition

### Interface

> An **interface defines a contract** — it tells a class **what it must be able to do**.

### Abstract Class

> An **abstract class provides a partially implemented base class** — it tells subclasses **what they must do and can also provide how some things should be done**.

Think:

```text
Interface
    ↓
"What can you do?"

Abstract Class
    ↓
"What are you, and what common behavior do you share?"
```

---

# 2. The Simplest Possible Example

Imagine we have animals.

A dog can:

```text
eat()
sleep()
bark()
```

A cat can:

```text
eat()
sleep()
meow()
```

Both animals eat and sleep.

So we could create:

```java
abstract class Animal {

    void eat() {
        System.out.println("Eating...");
    }

    void sleep() {
        System.out.println("Sleeping...");
    }

    abstract void makeSound();
}
```

Then:

```java
class Dog extends Animal {

    @Override
    void makeSound() {
        System.out.println("Bark");
    }
}
```

The abstract class provides:

```text
eat()
sleep()
```

while forcing the subclass to implement:

```text
makeSound()
```

---

Now imagine we have:

```java
interface Flyable {

    void fly();
}
```

A bird can fly:

```java
class Bird implements Flyable {

    @Override
    public void fly() {
        System.out.println("Flying");
    }
}
```

An airplane can also fly:

```java
class Airplane implements Flyable {

    @Override
    public void fly() {
        System.out.println("Flying airplane");
    }
}
```

Notice something important.

A bird and an airplane are **not the same type of thing**.

But they share a capability:

```text
Bird --------\
              → Flyable
Airplane ----/
```

That's where interfaces become extremely powerful.

---

# 3. Mental Model

Think about a **USB port**.

Your laptop doesn't care whether you connect:

```text
Keyboard
Mouse
Hard Drive
Camera
```

It only cares:

> "Does this device follow the USB contract?"

That's an interface.

```text
             USB Interface
                  │
        ┌─────────┼─────────┐
        ↓         ↓         ↓
     Keyboard    Mouse    Camera
```

The interface defines the expected behavior.

---

# 4. Why Do We Need Interfaces?

Without interfaces, code can become tightly coupled.

Suppose:

```java
class PaymentService {

    private RazorpayPayment payment = new RazorpayPayment();

}
```

Now your service directly depends on Razorpay.

Tomorrow you want Stripe.

You have to modify:

```java
PaymentService
```

Instead:

```java
interface PaymentGateway {

    void pay(double amount);
}
```

Implementations:

```java
class RazorpayPayment implements PaymentGateway {

    public void pay(double amount) {
        // Razorpay logic
    }
}
```

```java
class StripePayment implements PaymentGateway {

    public void pay(double amount) {
        // Stripe logic
    }
}
```

Service:

```java
class PaymentService {

    private final PaymentGateway paymentGateway;

    PaymentService(PaymentGateway paymentGateway) {
        this.paymentGateway = paymentGateway;
    }

    void makePayment(double amount) {
        paymentGateway.pay(amount);
    }
}
```

Now:

```text
PaymentService
      |
      ↓
PaymentGateway
      |
 ┌────┴─────┐
 ↓          ↓
Stripe    Razorpay
```

The service doesn't care about the implementation.

This is one of the most important reasons interfaces exist:

> **Programming against an abstraction rather than an implementation.**

---

# 5. Interface Syntax

Basic interface:

```java
interface PaymentGateway {

    void pay(double amount);
}
```

Implementation:

```java
class StripePayment implements PaymentGateway {

    @Override
    public void pay(double amount) {
        System.out.println("Stripe payment");
    }
}
```

Keyword:

```java
implements
```

not:

```java
extends
```

---

# 6. Abstract Class Syntax

```java
abstract class Animal {

    abstract void makeSound();

    void sleep() {
        System.out.println("Sleeping");
    }
}
```

Implementation:

```java
class Dog extends Animal {

    @Override
    void makeSound() {
        System.out.println("Bark");
    }
}
```

Keyword:

```java
extends
```

---

# 7. Interface vs Abstract Class — The Core Difference

| Feature                    | Interface                                                                                        | Abstract Class       |
| -------------------------- | ------------------------------------------------------------------------------------------------ | -------------------- |
| Purpose                    | Contract/capability                                                                              | Shared base behavior |
| Keyword                    | `implements`                                                                                     | `extends`            |
| Multiple inheritance       | Multiple interfaces                                                                              | Only one class       |
| Constructors               | No                                                                                               | Yes                  |
| Instance variables         | No normal instance state                                                                         | Yes                  |
| Abstract methods           | Yes                                                                                              | Yes                  |
| Concrete methods           | Yes                                                                                              | Yes                  |
| Static methods             | Yes                                                                                              | Yes                  |
| Default methods            | Yes                                                                                              | No `default` keyword |
| Protected methods          | Modern Java interfaces do not allow protected instance methods                                   | Yes                  |
| Private methods            | Yes, since Java 9                                                                                | Yes                  |
| Final methods              | Interface methods have restrictions depending on kind; abstract contract methods cannot be final | Yes                  |
| Object state               | Generally no instance state                                                                      | Yes                  |
| `super`                    | No interface instance `super` equivalent to class inheritance                                    | Yes                  |
| Constructor initialization | No                                                                                               | Yes                  |

---

# 8. The Most Important Concept: IS-A vs CAN-DO

This is a great interview shortcut.

### Abstract Class

Usually represents:

> **IS-A relationship**

```text
Dog IS-A Animal
Car IS-A Vehicle
SavingsAccount IS-A BankAccount
```

So:

```java
class Dog extends Animal
```

makes sense.

---

### Interface

Usually represents:

> **CAN-DO capability**

```text
Bird CAN fly
Drone CAN fly
PaymentGateway CAN process payment
NotificationSender CAN send notification
```

Example:

```java
interface Flyable {
    void fly();
}
```

Then:

```java
class Bird implements Flyable
class Drone implements Flyable
class Airplane implements Flyable
```

They don't need to share a common parent class.

---

# 9. Interfaces Support Multiple Inheritance of Type

Java doesn't allow:

```java
class C extends A, B
```

This is illegal.

But Java allows:

```java
interface Flyable {
    void fly();
}

interface Swimmable {
    void swim();
}
```

A class can implement both:

```java
class Duck implements Flyable, Swimmable {

    public void fly() {
    }

    public void swim() {
    }
}
```

So:

```text
             Duck
            /    \
           ↓      ↓
       Flyable  Swimmable
```

This is one of the biggest advantages of interfaces.

---

# 10. Why Doesn't Java Allow Multiple Class Inheritance?

Consider:

```java
class A {

    void hello() {
        System.out.println("A");
    }
}
```

```java
class B {

    void hello() {
        System.out.println("B");
    }
}
```

If Java allowed:

```java
class C extends A, B
```

what should this do?

```java
C c = new C();
c.hello();
```

Should it print:

```text
A
```

or:

```text
B
```

This creates ambiguity.

This is commonly known as the **diamond problem**.

Java avoids this by allowing:

```java
class C extends A
```

but:

```java
class C implements AInterface, BInterface
```

for interfaces.

---

# 11. Can an Interface Have Variables?

Yes, but there is an important rule.

Variables declared in an interface are implicitly:

```java
public static final
```

Example:

```java
interface Constants {

    int MAX_RETRY = 3;
}
```

is effectively:

```java
public static final int MAX_RETRY = 3;
```

Therefore:

```java
Constants.MAX_RETRY
```

works.

But:

```java
MAX_RETRY = 5;
```

doesn't.

It's final.

---

# 12. Can an Interface Have Methods?

Yes.

Modern Java interfaces can have:

### Abstract methods

```java
interface Payment {

    void pay();
}
```

### Default methods

```java
interface Payment {

    default void validate() {
        System.out.println("Validating...");
    }
}
```

### Static methods

```java
interface Payment {

    static void log() {
        System.out.println("Logging");
    }
}
```

### Private methods

Since Java 9:

```java
interface Payment {

    default void process() {
        validate();
    }

    private void validate() {
        System.out.println("Validation");
    }
}
```

---

# 13. Why Were Default Methods Added?

This is a **very common interview question**.

Imagine Java originally had:

```java
interface Payment {

    void pay();
}
```

Thousands of classes implement it.

Now Java wants to add:

```java
void refund();
```

Every implementation breaks:

```java
class StripePayment implements Payment
class RazorpayPayment implements Payment
class PaypalPayment implements Payment
...
```

Every class must implement:

```java
refund()
```

This could break backward compatibility.

So Java introduced:

```java
default void refund() {
    // default implementation
}
```

Now existing implementations don't necessarily break.

---

# 14. Default Method Example

```java
interface Vehicle {

    void start();

    default void stop() {
        System.out.println("Vehicle stopped");
    }
}
```

Implementation:

```java
class Car implements Vehicle {

    @Override
    public void start() {
        System.out.println("Car started");
    }
}
```

Car automatically gets:

```java
stop()
```

without implementing it.

---

# 15. What If Two Interfaces Have the Same Default Method?

Example:

```java
interface A {

    default void hello() {
        System.out.println("A");
    }
}
```

```java
interface B {

    default void hello() {
        System.out.println("B");
    }
}
```

Now:

```java
class C implements A, B {
}
```

This causes a compilation error.

Java says:

> "You decide which implementation you want."

So:

```java
class C implements A, B {

    @Override
    public void hello() {
        A.super.hello();
    }
}
```

Now the ambiguity is resolved.

---

# 16. Can an Abstract Class Have Constructors?

Yes.

This is a major difference.

```java
abstract class Vehicle {

    Vehicle() {
        System.out.println("Vehicle constructor");
    }
}
```

Subclass:

```java
class Car extends Vehicle {

    Car() {
        System.out.println("Car constructor");
    }
}
```

When:

```java
new Car();
```

the order is:

```text
Vehicle constructor
        ↓
Car constructor
```

Even though you cannot instantiate:

```java
new Vehicle(); // ❌
```

its constructor still runs when a subclass object is created.

---

# 17. Can an Abstract Class Have Instance Variables?

Yes.

```java
abstract class Employee {

    protected String name;
    protected double salary;

    Employee(String name, double salary) {
        this.name = name;
        this.salary = salary;
    }

    abstract void calculateBonus();
}
```

Subclass:

```java
class Developer extends Employee {

    Developer(String name, double salary) {
        super(name, salary);
    }

    @Override
    void calculateBonus() {
        System.out.println(salary * 0.10);
    }
}
```

This is a strong reason to use an abstract class.

You can share:

```text
State
+
Behavior
+
Initialization
+
Common logic
```

---

# 18. Can an Abstract Class Have Concrete Methods?

Absolutely.

That's actually one of its main purposes.

```java
abstract class Employee {

    void login() {
        System.out.println("Login");
    }

    abstract void work();
}
```

Every employee gets:

```text
login()
```

but each employee must define:

```text
work()
```

---

# 19. Can an Abstract Class Have No Abstract Methods?

Yes.

This surprises many candidates.

This is legal:

```java
abstract class Utility {

    void doSomething() {
    }
}
```

Why make it abstract?

To prevent direct instantiation:

```java
new Utility(); // ❌
```

while still allowing subclasses.

---

# 20. Can an Interface Extend Another Interface?

Yes.

```java
interface Animal {
    void eat();
}
```

```java
interface Pet extends Animal {
    void play();
}
```

Now:

```java
class Dog implements Pet {

    public void eat() {
    }

    public void play() {
    }
}
```

---

# 21. Can an Interface Extend Multiple Interfaces?

Yes.

```java
interface Flyable {
    void fly();
}

interface Swimmable {
    void swim();
}

interface DuckBehavior extends Flyable, Swimmable {
}
```

This is valid.

---

# 22. Can an Abstract Class Implement an Interface?

Yes.

Very common.

```java
interface Payment {

    void pay();
}
```

```java
abstract class BasePayment implements Payment {

    void validate() {
        System.out.println("Validation");
    }
}
```

`BasePayment` doesn't have to implement `pay()` because it's abstract.

A concrete subclass can:

```java
class StripePayment extends BasePayment {

    @Override
    public void pay() {
        System.out.println("Stripe payment");
    }
}
```

This pattern is extremely useful in production.

---

# 23. Can an Abstract Class Extend Another Abstract Class?

Yes.

```java
abstract class Animal {

    abstract void eat();
}
```

```java
abstract class Mammal extends Animal {

    abstract void walk();
}
```

```java
class Dog extends Mammal {

    void eat() {
    }

    void walk() {
    }
}
```

---

# 24. Can an Abstract Class Implement Multiple Interfaces?

Yes.

```java
interface Flyable {
    void fly();
}

interface Swimmable {
    void swim();
}

abstract class Duck
        implements Flyable, Swimmable {

}
```

Because the class is abstract, it doesn't need to implement every method.

---

# 25. Can We Instantiate an Interface?

No.

```java
Payment p = new Payment(); // ❌
```

Same for an abstract class:

```java
Animal a = new Animal(); // ❌
```

But you can create a reference:

```java
Payment p = new StripePayment();
```

This is extremely important.

---

# 26. Interface Reference vs Implementation

Consider:

```java
Payment payment = new StripePayment();
```

There are two sides:

```text
Payment payment
     ↑
reference type


new StripePayment()
     ↑
actual object
```

The reference determines what methods are visible at compile time.

The actual object determines which overridden implementation executes at runtime.

This is **polymorphism**.

---

# 27. Why Is This Useful?

Suppose:

```java
Payment payment;
```

Today:

```java
payment = new StripePayment();
```

Tomorrow:

```java
payment = new RazorpayPayment();
```

The calling code doesn't have to change.

That's one of the foundations of:

* loose coupling
* dependency injection
* polymorphism
* testability
* extensibility

---

# 28. Production Example — Notification Service

Suppose your company sends:

* Email
* SMS
* Push notifications
* WhatsApp notifications

You could create:

```java
interface NotificationSender {

    void send(String message, String recipient);
}
```

Implementations:

```java
class EmailNotificationSender
        implements NotificationSender {

    @Override
    public void send(String message, String recipient) {
        // Send email
    }
}
```

```java
class SmsNotificationSender
        implements NotificationSender {

    @Override
    public void send(String message, String recipient) {
        // Send SMS
    }
}
```

```java
class PushNotificationSender
        implements NotificationSender {

    @Override
    public void send(String message, String recipient) {
        // Send push
    }
}
```

Then:

```java
class NotificationService {

    private final NotificationSender sender;

    NotificationService(NotificationSender sender) {
        this.sender = sender;
    }

    void notifyUser(String message, String recipient) {
        sender.send(message, recipient);
    }
}
```

This is exactly the kind of design you'll encounter in Spring Boot applications.

---

# 29. Why Interfaces Are Everywhere in Spring Boot

You frequently see:

```java
public interface UserRepository
```

```java
public interface PaymentService
```

```java
public interface NotificationService
```

Why?

Because Spring applications heavily use:

```text
Abstraction
     ↓
Implementation
     ↓
Dependency Injection
```

For example:

```java
@Service
class EmailNotificationService
        implements NotificationService {
}
```

Then:

```java
@Service
class OrderService {

    private final NotificationService notificationService;

    OrderService(NotificationService notificationService) {
        this.notificationService = notificationService;
    }
}
```

`OrderService` doesn't need to know the concrete implementation.

---

# 30. Interface + Dependency Injection

This is a very important interview connection.

Without abstraction:

```java
class OrderService {

    private EmailNotificationService service =
        new EmailNotificationService();
}
```

Problems:

```text
tight coupling
harder testing
harder replacement
harder extension
```

With abstraction:

```java
class OrderService {

    private final NotificationService service;

    OrderService(NotificationService service) {
        this.service = service;
    }
}
```

Now Spring can inject an implementation.

```text
OrderService
      |
      ↓
NotificationService
      |
      ↓
EmailNotificationService
```

---

# 31. What About Abstract Classes in Production?

Suppose you have several file processors.

```text
CSV Processor
Excel Processor
JSON Processor
```

All processors perform:

```text
1. Validate file
2. Log start
3. Process records
4. Log completion
```

But processing differs.

An abstract class can capture the common workflow.

```java
abstract class FileProcessor {

    public final void process() {

        validate();

        readFile();

        processRecords();

        cleanup();
    }

    void validate() {
        System.out.println("Common validation");
    }

    abstract void readFile();

    abstract void processRecords();

    void cleanup() {
        System.out.println("Cleanup");
    }
}
```

CSV:

```java
class CsvProcessor extends FileProcessor {

    @Override
    void readFile() {
        System.out.println("Reading CSV");
    }

    @Override
    void processRecords() {
        System.out.println("Processing CSV");
    }
}
```

This is the **Template Method Pattern**.

---

# 32. Interface vs Abstract Class Through a Real Example

Imagine you're building a cloud file-processing system.

You have:

```text
GCS
S3
Azure Blob Storage
```

All can:

```text
upload()
download()
delete()
```

Interface:

```java
interface StorageService {

    void upload(String file);

    void download(String file);

    void delete(String file);
}
```

Implementations:

```java
class GcsStorageService
        implements StorageService {
}
```

```java
class S3StorageService
        implements StorageService {
}
```

```java
class AzureStorageService
        implements StorageService {
}
```

This is a good interface use case because these implementations share a **contract**, not necessarily a common internal implementation.

---

# 33. Now Imagine Common Behavior

Suppose all storage providers need:

```text
authentication
logging
metrics
retry
validation
```

You might introduce:

```java
abstract class BaseStorageService
        implements StorageService {

    protected void logOperation() {
        System.out.println("Logging");
    }

    protected void validate() {
        System.out.println("Validation");
    }
}
```

Then:

```java
class GcsStorageService
        extends BaseStorageService {
}
```

Now you have:

```text
StorageService
       ↑
       |
BaseStorageService
       ↑
       |
GcsStorageService
```

This is a useful combination.

---

# 34. Interface + Abstract Class Together

This is a very powerful design.

```java
interface PaymentProcessor {

    void processPayment();
}
```

Common implementation:

```java
abstract class BasePaymentProcessor
        implements PaymentProcessor {

    protected void validatePayment() {
        System.out.println("Validation");
    }

    protected void logPayment() {
        System.out.println("Logging");
    }
}
```

Concrete implementation:

```java
class StripePaymentProcessor
        extends BasePaymentProcessor {

    @Override
    public void processPayment() {

        validatePayment();

        logPayment();

        System.out.println("Stripe processing");
    }
}
```

Think:

```text
                  Interface
               PaymentProcessor
                     │
                     ↓
             Abstract Class
          BasePaymentProcessor
                     │
              ┌──────┴──────┐
              ↓             ↓
           Stripe         Razorpay
```

---

# 35. When Should I Choose an Interface?

Use an interface when:

### 1. You want to define a contract

```java
interface PaymentGateway {
    void pay();
}
```

### 2. Multiple unrelated classes share a capability

```text
Bird
Airplane
Drone
```

all:

```text
Flyable
```

### 3. You want loose coupling

```text
Service → Interface
```

instead of:

```text
Service → Concrete implementation
```

### 4. You expect multiple implementations

```text
StorageService
   ├── GCS
   ├── AWS S3
   └── Azure
```

### 5. You want easy mocking/testing

```java
PaymentGateway mockGateway;
```

### 6. You want multiple inheritance of behavior contracts

```java
class Robot
    implements Movable, Chargeable, Programmable
```

---

# 36. When Should I Choose an Abstract Class?

Use an abstract class when:

### 1. Classes have a strong parent-child relationship

```text
Dog → Animal
```

### 2. They share state

```java
protected String name;
protected int age;
```

### 3. They share implementation

```java
void log()
void validate()
void authenticate()
```

### 4. You need constructors

```java
protected BaseService(String serviceName) {
}
```

### 5. You want to control inheritance

You can use:

```java
final
protected
private
```

methods and fields.

### 6. You want a template workflow

```text
validate
   ↓
process
   ↓
save
   ↓
cleanup
```

while allowing subclasses to customize individual steps.

---

# 37. When Should You NOT Use an Abstract Class?

Don't create:

```java
abstract class BaseEverything
```

just because several classes have a few common methods.

For example:

```text
BaseUserService
BaseOrderService
BasePaymentService
BaseNotificationService
```

with unrelated responsibilities can become a giant inheritance hierarchy.

This creates:

```text
tight coupling
fragile inheritance
difficult maintenance
```

Prefer composition where appropriate.

---

# 38. Composition vs Inheritance

A very important senior-level concept.

Instead of:

```java
class Car extends Engine
```

which is conceptually wrong:

```text
Car IS-A Engine ❌
```

use:

```java
class Car {

    private Engine engine;
}
```

because:

```text
Car HAS-A Engine ✅
```

This is **composition**.

A common design principle is:

> Prefer composition over inheritance when inheritance does not represent a true "is-a" relationship.

---

# 39. The Golden Interview Rule

Remember:

```text
Interface
    ↓
"What can this object DO?"

Abstract class
    ↓
"What IS this object?"
"What common things does it share?"
```

But don't treat this as an absolute law.

Real production design depends on:

* coupling
* reuse
* state
* extensibility
* ownership
* testing
* API design

---

# 40. Common Interview Question: Interface or Abstract Class?

Interviewer:

> "You are designing a payment system. Would you use interface or abstract class?"

A strong answer:

> "I would generally start with an interface for the payment gateway because the primary requirement is a common contract across potentially different payment providers. If the implementations later share substantial state or common workflow, I could introduce an abstract base class behind that interface to reuse implementation."

That's much stronger than:

> "Interface because interfaces are better."

---

# 41. Another Interview Scenario

### Question

You have:

```text
GCSFileService
S3FileService
AzureFileService
```

What would you use?

Answer:

```java
interface FileStorage {
    upload();
    download();
    delete();
}
```

because these are different implementations of the same capability.

If they share substantial logic:

```java
abstract class BaseFileStorage
        implements FileStorage {
}
```

So you can have both.

---

# 42. Interface Segregation

Interfaces shouldn't become enormous.

Bad:

```java
interface Employee {

    void code();

    void test();

    void deploy();

    void manageTeam();

    void conductInterview();

    void createArchitecture();
}
```

A junior developer may only need:

```text
code()
```

but is forced to implement everything.

Instead:

```java
interface Developer {
    void code();
}
```

```java
interface Tester {
    void test();
}
```

```java
interface Manager {
    void manageTeam();
}
```

This follows the **Interface Segregation Principle**.

---

# 43. SOLID Connection

Interfaces are strongly related to SOLID.

### S — Single Responsibility

Keep interfaces focused.

### O — Open/Closed

Add new implementations without modifying consumers.

```text
PaymentGateway
    ├── Stripe
    ├── Razorpay
    └── PayPal
```

### L — Liskov Substitution

Implementations should behave according to the contract.

### I — Interface Segregation

Prefer small, focused interfaces.

### D — Dependency Inversion

High-level code should depend on abstractions.

This is particularly important in Spring Boot.

---

# 44. Functional Interfaces

An interface with exactly **one abstract method** can be a functional interface.

Example:

```java
@FunctionalInterface
interface Calculator {

    int calculate(int a, int b);
}
```

Then:

```java
Calculator add =
        (a, b) -> a + b;
```

This connects directly to:

* Lambda expressions
* Streams
* Functional programming

Examples from Java:

```java
Runnable
Callable
Comparator
Predicate
Function
Consumer
Supplier
```

---

# 45. Important Trick: `@FunctionalInterface`

This:

```java
@FunctionalInterface
interface Calculator {

    int calculate(int a, int b);
}
```

tells the compiler:

> "This interface must have exactly one abstract method."

If you add another abstract method:

```java
void log();
```

you get a compilation error.

Default/static/private methods don't count as abstract methods.

---

# 46. Interface Method Visibility

An interface's abstract methods are implicitly:

```java
public
```

So:

```java
interface Payment {

    void pay();
}
```

is effectively:

```java
public abstract void pay();
```

Therefore implementation cannot reduce visibility.

Wrong:

```java
class StripePayment implements Payment {

    protected void pay() {
    }
}
```

Correct:

```java
class StripePayment implements Payment {

    public void pay() {
    }
}
```

---

# 47. Can Interface Methods Be `private`?

Yes.

Since Java 9:

```java
interface Payment {

    default void process() {
        validate();
    }

    private void validate() {
        System.out.println("Validating");
    }
}
```

The private method is only available inside the interface.

It cannot be called by implementing classes.

---

# 48. Can Interface Methods Be `final`?

Abstract interface methods cannot be `final`.

Why?

Because:

```text
abstract
```

means:

> "Someone must implement me."

while:

```text
final
```

means:

> "Nobody can override me."

They contradict each other.

---

# 49. Can Abstract Classes Have Final Methods?

Yes.

```java
abstract class PaymentProcessor {

    final void validate() {
        System.out.println("Validation");
    }

    abstract void process();
}
```

Subclass:

```java
class StripeProcessor extends PaymentProcessor {

    @Override
    void process() {
    }
}
```

The subclass cannot override:

```java
validate()
```

This is useful when a base class needs to enforce an invariant.

---

# 50. The Template Method Pattern

This deserves special attention.

Suppose:

```text
Every import must:

1. Validate
2. Read
3. Transform
4. Save
5. Audit
```

But CSV and Excel have different reading logic.

Abstract class:

```java
abstract class ImportProcessor {

    public final void process() {

        validate();

        read();

        transform();

        save();

        audit();
    }

    void validate() {
    }

    abstract void read();

    abstract void transform();

    void save() {
    }

    void audit() {
    }
}
```

The sequence cannot be changed:

```text
validate
   ↓
read
   ↓
transform
   ↓
save
   ↓
audit
```

This is a very good use of an abstract class.

---

# 51. Why `final` on the Template Method?

Notice:

```java
public final void process()
```

Why?

Because we don't want subclasses to change the algorithm.

Otherwise:

```java
class BadProcessor extends ImportProcessor {

    @Override
    public void process() {
        save();
        validate();
    }
}
```

Now the entire workflow is broken.

So:

```text
final template method
+
abstract customizable steps
```

is a classic design.

---

# 52. Interface Default Methods vs Abstract Class Concrete Methods

This is a subtle interview topic.

### Interface

```java
default void log() {
}
```

is primarily useful for:

> Providing behavior alongside a contract, especially for API evolution or small shared capabilities.

### Abstract class

```java
void log() {
}
```

can be used for:

> Shared implementation + state + constructors + controlled inheritance.

So don't say:

> "They are the same."

They solve related but different design problems.

---

# 53. Abstract Class Has State

For example:

```java
abstract class Account {

    protected String accountNumber;
    protected double balance;
}
```

Each object gets its own state.

```text
Account A
balance = 1000

Account B
balance = 5000
```

An interface doesn't provide ordinary per-object instance fields.

---

# 54. Why Interfaces Don't Have Instance State

An interface describes a contract.

Imagine:

```java
interface Animal {

    String name;
}
```

Whose `name` is that?

Every implementation?

What initialization rules?

What constructor?

Java instead makes interface fields:

```text
public static final
```

so they represent constants.

Object-specific state belongs naturally to classes.

---

# 55. Interface Evolution

One thing to remember for interviews:

Interfaces can evolve using:

```java
default methods
```

and:

```java
static methods
```

without forcing every implementation to provide new behavior.

But adding a **new abstract method** to an existing interface can break implementations.

---

# 56. Marker Interfaces

A marker interface has no methods.

Example:

```java
interface MyMarker {
}
```

Historically, Java uses:

```java
Serializable
Cloneable
```

as examples of marker interfaces.

The interface itself communicates metadata/capability.

Conceptually:

```text
Class
  ↓
implements Serializable
  ↓
"This class has this capability/meaning"
```

Modern Java often uses annotations for metadata, but marker interfaces still exist and can be useful where type-based behavior matters.

---

# 57. Interface as a Type

This is important.

An interface isn't only about methods.

It creates a **type**.

```java
List<String> list = new ArrayList<>();
```

Notice:

```java
List
```

is the abstraction.

```java
ArrayList
```

is the implementation.

This is one of the most common examples of programming to an interface.

You can later use:

```java
List<String> list = new LinkedList<>();
```

without changing most consumer code.

---

# 58. Why `List` Instead of `ArrayList`?

Because the consumer usually cares:

> "I need something that behaves like a List."

It doesn't necessarily care:

> "I specifically need ArrayList."

Therefore:

```java
List<String> users
```

is often better than:

```java
ArrayList<String> users
```

when the implementation details aren't important to the caller.

---

# 59. Interface and Testing

Suppose:

```java
class OrderService {

    private final PaymentGateway paymentGateway;

    OrderService(PaymentGateway paymentGateway) {
        this.paymentGateway = paymentGateway;
    }
}
```

In production:

```java
new OrderService(new StripePaymentGateway());
```

In tests:

```java
new OrderService(mockPaymentGateway);
```

This makes unit testing much easier.

You can test:

```text
OrderService
```

without making real payment calls.

---

# 60. Interfaces and Microservices

In a cloud/backend application, interfaces are useful at boundaries.

For example:

```text
OrderService
       |
       ↓
PaymentClient
       |
       ├── RealPaymentClient
       └── MockPaymentClient
```

Or:

```text
NotificationService
       |
       ↓
NotificationSender
       |
 ┌─────┼──────┐
 ↓     ↓      ↓
Email  SMS   Push
```

This makes changing infrastructure implementations easier.

---

# 61. Interface and Cloud Architecture Example

Imagine your application stores files in Google Cloud Storage.

You create:

```java
interface ObjectStorage {

    void upload(String bucket, String key);

    void download(String bucket, String key);

    void delete(String bucket, String key);
}
```

Then:

```java
class GcsObjectStorage implements ObjectStorage {
}
```

Later your organization migrates part of the workload to AWS:

```java
class S3ObjectStorage implements ObjectStorage {
}
```

Your business logic can continue using:

```java
ObjectStorage
```

instead of directly depending on:

```java
GcsObjectStorage
```

This is a very practical cloud-engineering example.

---

# 62. Common Mistake: "Always Use Interfaces"

Don't.

This is not a rule:

```text
Interface > Abstract Class
```

Sometimes an abstract class is the correct design.

Sometimes a concrete class is enough.

For example:

```java
class UserValidator {
}
```

If there is:

* one implementation
* no expected variation
* no need for polymorphism
* no meaningful abstraction

then introducing:

```java
interface UserValidator
```

may simply add unnecessary complexity.

---

# 63. Don't Create Interfaces Just for the Sake of Testing

A common misconception is:

> "Every class must have an interface so it can be mocked."

Not necessarily.

If a class is simple and stable, mocking the class may be unnecessary.

Interfaces are most valuable when they represent a meaningful abstraction or variation point.

---

# 64. Don't Create Giant Abstract Base Classes

Bad architecture:

```text
BaseService
    |
    ├── 30 fields
    ├── 50 methods
    ├── database logic
    ├── logging
    ├── caching
    ├── validation
    ├── authorization
    ├── notification
    └── random utilities
```

This becomes a **God class/base class**.

Every subclass becomes coupled to things it doesn't need.

Prefer:

```text
small abstractions
+
composition
+
focused interfaces
```

---

# 65. Interface vs Abstract Class — Interview Cheat Sheet

```text
                 INTERFACE
                     │
                     ↓
               Defines contract
                     │
          ┌──────────┼──────────┐
          ↓          ↓          ↓
        Email       SMS        Push
```

Versus:

```text
              ABSTRACT CLASS
                     │
                     ↓
              Shared foundation
                     │
          ┌──────────┼──────────┐
          ↓          ↓          ↓
        CSV        Excel       JSON
```

---

# 66. The Decision Tree

When designing something, ask:

### Question 1

> Do I need a common contract/capability?

If yes:

```text
Interface
```

### Question 2

> Do implementations share state and substantial behavior?

If yes:

```text
Abstract class
```

### Question 3

> Are they unrelated objects that simply share a capability?

Usually:

```text
Interface
```

### Question 4

> Do I need multiple types/capabilities?

Use:

```java
implements A, B, C
```

### Question 5

> Do I need a common workflow with customizable steps?

Consider:

```text
Abstract class
+
Template Method
```

### Question 6

> Is there no real variation or abstraction?

Maybe:

```text
Concrete class
```

---

# 67. Interview Question Bank

## Beginner

### Q1. What is an interface?

An interface defines a contract that implementing classes agree to follow.

---

### Q2. What is an abstract class?

A class that cannot be directly instantiated and can contain both abstract and concrete behavior.

---

### Q3. Can we instantiate an interface?

No.

---

### Q4. Can we instantiate an abstract class?

No.

---

### Q5. Can an abstract class have constructors?

Yes.

---

### Q6. Can an abstract class have variables?

Yes.

---

### Q7. Can an interface have variables?

Yes, but they are implicitly:

```java
public static final
```

---

### Q8. Can an interface have implemented methods?

Yes.

Using:

```java
default
static
private
```

methods.

---

### Q9. Can an abstract class have concrete methods?

Yes.

---

### Q10. Can an interface extend another interface?

Yes.

---

# 68. Intermediate Questions

### Q11. Can an interface extend multiple interfaces?

Yes.

```java
interface C extends A, B {
}
```

---

### Q12. Can a class implement multiple interfaces?

Yes.

```java
class C implements A, B {
}
```

---

### Q13. Can an abstract class implement an interface?

Yes.

---

### Q14. Can an abstract class extend another abstract class?

Yes.

---

### Q15. Why doesn't Java support multiple class inheritance?

Primarily to avoid ambiguity and complexity such as the diamond problem.

---

### Q16. Why does Java support multiple interfaces?

Because interfaces primarily define contracts, and Java provides rules to resolve conflicting default methods.

---

### Q17. What happens if two interfaces have the same default method?

The implementing class must resolve the conflict.

---

### Q18. Why were default methods introduced?

Primarily to evolve interfaces while maintaining backward compatibility.

---

### Q19. Can an interface have private methods?

Yes, since Java 9.

---

### Q20. Can an abstract class have static methods?

Yes.

---

# 69. Senior-Level Questions

### Q21. Interface or abstract class for payment providers?

Usually an interface for the common payment contract.

An abstract class may be introduced if implementations genuinely share substantial state or behavior.

---

### Q22. Why are interfaces useful with dependency injection?

They allow high-level components to depend on abstractions instead of concrete implementations.

---

### Q23. Why does Spring encourage programming against interfaces?

It promotes loose coupling and allows implementations to be replaced or injected.

---

### Q24. When would an interface be overengineering?

When there is no meaningful abstraction, variation, or polymorphic requirement.

---

### Q25. When would inheritance be problematic?

When subclasses inherit behavior/state they don't conceptually need or when the hierarchy becomes deeply coupled.

---

### Q26. Why prefer composition over inheritance?

Composition generally provides more flexible reuse and reduces coupling between types.

---

### Q27. What is the Template Method pattern?

A base abstract class defines the overall algorithm while subclasses customize selected steps.

---

### Q28. What is Interface Segregation?

Clients shouldn't be forced to depend on methods they don't need.

---

### Q29. What is Dependency Inversion?

High-level modules should depend on abstractions rather than concrete low-level implementations.

---

### Q30. Is an interface always better than an abstract class?

No.

The correct choice depends on whether you need:

```text
contract
vs
shared state/implementation/workflow
```

---

# 70. Rapid-Fire Interview Revision

Before your interview, remember this:

```text
Interface
─────────
Contract
Capability
Loose coupling
Multiple interfaces
No constructors
No instance state
Default methods
Static methods
Private methods
Functional interfaces
Dependency injection
Polymorphism
```

```text
Abstract Class
───────────────
Partial implementation
Common base
Shared state
Constructors
Concrete methods
Abstract methods
Protected members
Final methods
Template Method
Controlled inheritance
```

---

# 71. The Most Important Comparison

If an interviewer asks:

> "Interface vs Abstract Class?"

Don't start listing 20 differences.

Start with the **design reasoning**:

> **"I use an interface when I primarily need to define a contract or capability and potentially support multiple unrelated implementations. I use an abstract class when related classes share common state, implementation, initialization, or a common workflow. In real systems, they can also be used together — an interface defines the public contract while an abstract base class provides reusable implementation."**

Then give an example.

That's a much more senior answer.

---

# 72. One Real Production Design

Imagine we're building:

```text
Cloud File Import Service
```

We support:

```text
CSV
Excel
JSON
```

Our interface:

```java
interface FileImporter {

    void importFile(String file);
}
```

Common processing:

```java
abstract class BaseFileImporter
        implements FileImporter {

    protected void validate(String file) {
        // Common validation
    }

    protected void audit(String file) {
        // Common audit
    }

    protected void save() {
        // Common persistence
    }
}
```

CSV:

```java
class CsvImporter extends BaseFileImporter {

    @Override
    public void importFile(String file) {

        validate(file);

        // CSV-specific parsing

        save();

        audit(file);
    }
}
```

Excel:

```java
class ExcelImporter extends BaseFileImporter {

    @Override
    public void importFile(String file) {

        validate(file);

        // Excel-specific parsing

        save();

        audit(file);
    }
}
```

Architecture:

```text
                    FileImporter
                         │
                         ↓
                 BaseFileImporter
                         │
              ┌──────────┼──────────┐
              ↓          ↓          ↓
             CSV        Excel      JSON
```

This demonstrates:

```text
Interface
   +
Abstract Class
   +
Polymorphism
   +
Inheritance
   +
Code reuse
   +
Loose coupling
```

---

# 73. The "10-Year-Old" Explanation

If you forget everything during an interview, remember this story.

Imagine a school.

There is a rule:

> Every student must be able to **study**.

That's an:

```text
Interface
```

because it defines a requirement:

```text
study()
```

But suppose all students also have:

```text
name
age
school
eat()
sleep()
```

You could have:

```text
Abstract Student
```

which contains the common things.

Then:

```text
EngineeringStudent
MedicalStudent
ArtsStudent
```

can inherit those common things and implement their own specialized behavior.

So:

```text
Interface
=
"You must be able to do this."

Abstract Class
=
"You belong to this family, and here's some stuff you already have."
```

That's the mental model I would keep.

---

# 74. Final Interview Cheat Sheet

```text
                         JAVA ABSTRACTION
                               │
                ┌──────────────┴──────────────┐
                ↓                             ↓
            INTERFACE                  ABSTRACT CLASS
                │                             │
             Contract                    Base class
                │                             │
           "CAN DO"                     "IS A"
                │                             │
       Multiple interfaces            Single class parent
                │                             │
       No constructor                 Constructor possible
                │                             │
       No instance state              Instance state possible
                │                             │
       default methods                Concrete methods
       static methods                 abstract methods
       private methods                final methods
                │                             │
                ↓                             ↓
       Loose coupling               Shared implementation
       Polymorphism                 Shared state
       DI                            Template Method
       Capabilities                 Common workflow
```

### If the interviewer asks "Which one should I use?"

Think:

```text
Do I need a CONTRACT?
        │
       YES
        ↓
    INTERFACE


Do I need SHARED STATE / IMPLEMENTATION / WORKFLOW?
        │
       YES
        ↓
 ABSTRACT CLASS


Do I need BOTH?
        │
       YES
        ↓
INTERFACE + ABSTRACT BASE CLASS
```

And one final rule worth remembering:

> **Don't choose an interface or abstract class because one is "better." Choose based on the abstraction your design actually needs.**

For a ~5-year Java/Spring Boot backend interview, that's the level of reasoning interviewers generally want to hear—not just the syntax differences.
