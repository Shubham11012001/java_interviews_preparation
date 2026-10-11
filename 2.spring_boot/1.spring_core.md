# Spring Core: IoC, Dependency Injection (DI) & AOP — The Complete Interview Guide

Technical Architect Edition · Beginner-friendly explanations · Senior-level interview preparation

If you are preparing for Senior Java Backend Engineer, Lead Engineer, or Technical Architect interviews, Spring Core is one of the most important foundations to master.

Interviewers don't just ask what IoC, DI, or AOP means. They ask questions such as:

* What happens internally when Spring Boot starts?

* How does Spring create and manage beans?

* What happens when two beans implement the same interface?

* How does Spring resolve circular dependencies?

* Why does `@Transactional` sometimes not work?

* How does Spring AOP intercept a method call?

* What is the difference between `BeanFactory` and `ApplicationContext`?

* How would you design a loosely coupled, testable enterprise application?

This guide starts with simple analogies and builds toward production scenarios and architectural decisions.

# 1. The Big Picture: What Is Spring Core?

Imagine you are building a restaurant.

You need a chef, an oven, ingredients, a cashier, and a delivery person.

Without a manager, every employee tries to find and hire the others. The cashier creates the chef, the chef buys the oven, and the delivery person creates another cashier.

The restaurant becomes difficult to manage.

Now imagine a restaurant manager who creates everything, assigns responsibilities, and coordinates the staff.

Spring Core is like that manager for your Java application.

Spring helps you:

* Create Java objects.

* Connect objects to their dependencies.

* Manage their lifecycles.

* Apply cross-cutting functionality such as transactions, logging, and security.

* Reduce tight coupling and improve testability.

The three foundational concepts are:

1. IoC — Inversion of Control

Spring takes responsibility for creating and managing application objects instead of your application doing everything manually.

2. DI — Dependency Injection

Spring supplies an object with the other objects it needs instead of making it create those objects itself.

3. AOP — Aspect-Oriented Programming

Spring helps apply common behavior, such as transaction management, around selected method executions without repeating the same code everywhere.

Keep this distinction clear:

* IoC is the principle.

* DI is one way to implement that principle.

* AOP is a programming approach for separating cross-cutting concerns.

# 2. IoC — Inversion of Control

## 2.1 What is IoC?

IoC stands for Inversion of Control.

In simple words, you give up some control over object creation and lifecycle management to a framework.

### Without IoC

Suppose you are building an order-processing application.

Java

```
public class EmailService {

    public void sendEmail(String message) {
        System.out.println("Sending email: " + message);
    }
}

public class OrderService {

    private EmailService emailService = new EmailService();

    public void placeOrder() {
        // Save order
        emailService.sendEmail("Order placed");
    }
}
```

What is wrong here?

`OrderService` creates its own `EmailService`.

If you later want to use an SMS notification service, a mock service for testing, or a different email implementation, you may need to modify `OrderService`.

The high-level business class controls the creation of its dependency.

### With IoC

Instead, let Spring create and manage the objects.

Java

```
import org.springframework.stereotype.Service;

@Service
public class EmailService {

    public void sendEmail(String message) {
        System.out.println("Sending email: " + message);
    }
}
```

Java

```
import org.springframework.stereotype.Service;

@Service
public class OrderService {

    private final EmailService emailService;

    public OrderService(EmailService emailService) {
        this.emailService = emailService;
    }

    public void placeOrder() {
        emailService.sendEmail("Order placed");
    }
}
```

Spring creates the `EmailService`, creates the `OrderService`, and supplies the dependency.

Your code focuses on business behavior rather than manually managing the entire object graph.

## 2.2 Why is it called inversion?

Traditionally, your application code decides when and how to create objects.

With IoC, the framework manages those responsibilities according to your configuration and annotations.

The control has been inverted.

Traditional Java

Application code

creates and connects objects

Dependencies

Spring IoC

Spring IoC container

creates, connects and manages beans

Application objects receive dependencies

## 2.3 What is the IoC container?

The IoC container is the Spring component responsible for instantiating, configuring, assembling, and managing Spring beans.

Two important container interfaces are:

| Interface            | Meaning                                                                                                                          |
| -------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `BeanFactory`        | Core container abstraction for managing beans.                                                                                   |
| `ApplicationContext` | Builds on `BeanFactory` and adds features such as events, internationalization, and integration with application infrastructure. |

In a typical Spring Boot application, you work with an `ApplicationContext`, not by manually constructing a `BeanFactory`.

Example:

Java

```
ApplicationContext context =
        SpringApplication.run(MyApplication.class, args);

OrderService orderService =
        context.getBean(OrderService.class);
```

In ordinary business code, prefer constructor injection over repeatedly looking up beans using `context.getBean()`.

## 2.4 What is a Spring bean?

A bean is an object managed by the Spring container.

Not every Java object is automatically a Spring bean.

For example:

Java

```
@Service
public class PaymentService {
}
```

Spring can discover and register this class through component scanning.

You can also register beans explicitly:

Java

```
@Configuration
public class AppConfig {

    @Bean
    public PaymentService paymentService() {
        return new PaymentService();
    }
}
```

Here, `@Bean` tells Spring to manage the object returned by the method.

### Common bean-defining annotations

| Annotation        | Typical purpose                                                                                      |
| ----------------- | ---------------------------------------------------------------------------------------------------- |
| `@Component`      | General-purpose Spring-managed component.                                                            |
| `@Service`        | Service or business-logic component.                                                                 |
| `@Repository`     | Persistence-layer component; also participates in persistence exception translation where supported. |
| `@Controller`     | MVC controller.                                                                                      |
| `@RestController` | REST controller returning response bodies by default.                                                |
| `@Configuration`  | Declares configuration, commonly containing `@Bean` methods.                                         |
| `@Bean`           | Registers a bean returned by a configuration method.                                                 |

Important: `@Service`, `@Repository`, and `@Controller` are specialized forms of `@Component`.

## 2.5 How does Spring discover beans?

Spring commonly discovers beans through two mechanisms.

Mechanism 1: Component scanning

Java

```
@SpringBootApplication
public class MyApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }
}
```

`@SpringBootApplication` includes component scanning, configuration support, and auto-configuration.

By default, component scanning starts from the package containing the application class and covers its subpackages.

Mechanism 2: Explicit configuration

Java

```
@Configuration
public class DatabaseConfig {

    @Bean
    public Clock applicationClock() {
        return Clock.systemUTC();
    }
}
```

This approach is particularly useful for third-party classes you cannot annotate, such as a library-provided client.

## 2.6 What happens when Spring Boot starts?

At a high level:

1. `SpringApplication.run()` starts the application.

2. Spring creates and prepares an application context.

3. It loads configuration, processes component scanning and auto-configuration, and registers bean definitions.

4. Bean factories create required objects and resolve dependencies.

5. Bean post-processors and lifecycle callbacks participate in bean initialization.

6. The application context finishes refreshing and the application becomes ready when startup succeeds.

The actual startup process contains additional steps, and the exact sequence depends on the application and configuration.

A useful interview distinction:

* A bean definition describes how a bean should be created.

* A bean instance is the actual object created from that definition.

### Technical architect perspective

For a production application, ask:

* Are all beans being discovered from the intended packages?

* Are we creating expensive resources at startup unnecessarily?

* Can startup fail clearly when required configuration is missing?

* Are configuration values externalized?

* Are beans scoped correctly?

* Are integrations initialized safely and observably?

When is IoC most useful?

* Applications with many collaborating services.

* Systems requiring unit tests and mocks.

* Applications with interchangeable implementations.

* Enterprise applications needing consistent lifecycle and infrastructure management.

When does it become a problem?

* When developers create hundreds of unnecessary beans.

* When dependencies are hidden through service locators.

* When startup behavior becomes difficult to trace.

* When the application context is used as a global object registry throughout business code.

# 3. Dependency Injection (DI)

## 3.1 What is DI?

Dependency Injection means an object receives the objects it needs from an external source rather than creating those objects itself.

Imagine a phone needs a battery. The phone should not manufacture a battery every time you turn it on. It receives a compatible battery.

Similarly, an `OrderService` needs a payment processor. It should receive one rather than construct it internally.

## 3.2 What is a dependency?

A dependency is another object or service that a class needs to perform its work.

Java

```
public class OrderService {

    private final PaymentService paymentService;

    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

Here:

* `OrderService` is the dependent object.

* `PaymentService` is its dependency.

* The constructor is the injection point.

## 3.3 The three types of DI

### A. Constructor injection — recommended default

Java

```
@Service
public class OrderService {

    private final PaymentService paymentService;

    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

Spring supplies the dependency when constructing the bean.

When a class has exactly one constructor, Spring can use it without requiring `@Autowired`.

Advantages

* Required dependencies are explicit.

* Fields can be `final`.

* The object can be constructed in a valid state.

* Unit testing is straightforward.

* Circular dependencies are exposed instead of being silently hidden.

Disadvantages

* A constructor with too many dependencies may indicate that the class has too many responsibilities.

* Constructor-based circular dependencies prevent Spring from constructing the involved beans normally.

Best use: Almost all mandatory dependencies in ordinary service classes.

### B. Setter injection

Java

```
@Service
public class ReportService {

    private ExportService exportService;

    @Autowired
    public void setExportService(ExportService exportService) {
        this.exportService = exportService;
    }
}
```

Spring calls the setter to provide the dependency.

Advantages

* Can support optional or replaceable dependencies.

* Can be useful when a dependency genuinely needs to be configured after object construction.

Disadvantages

* The object may temporarily or permanently exist without the dependency.

* The dependency can be replaced after construction.

* The class must handle missing dependencies correctly.

Best use: Optional dependencies or rare cases where post-construction configuration is appropriate.

Do not use setter injection for every dependency merely because it is convenient.

### C. Field injection

Java

```
@Service
public class OrderService {

    @Autowired
    private PaymentService paymentService;
}
```

This is concise, but usually not recommended for application code.

Problems:

* Dependencies are less visible in the constructor.

* The field cannot be `final`.

* Creating a unit-test instance without Spring becomes less convenient.

* It encourages hidden dependencies and can make invalid object states easier to create.

### Comparison table

| Feature                 | Constructor                | Setter         | Field        |
| ----------------------- | -------------------------- | -------------- | ------------ |
| Required dependencies   | Excellent                  | Weaker         | Hidden       |
| Supports `final` fields | Yes                        | No, ordinarily | No           |
| Easy plain unit testing | Excellent                  | Good           | Poorer       |
| Optional dependencies   | Possible, but less natural | Good           | Possible     |
| Recommended default     | Yes                        | Sometimes      | Generally no |

## 3.4 How does Spring resolve a dependency?

Consider this code:

Java

```
public interface NotificationService {
    void send(String message);
}
```

Java

```
@Service
public class EmailNotificationService
        implements NotificationService {

    public void send(String message) {
        System.out.println("Email: " + message);
    }
}
```

Java

```
@Service
public class OrderService {

    private final NotificationService notificationService;

    public OrderService(NotificationService notificationService) {
        this.notificationService = notificationService;
    }
}
```

Spring examines the constructor parameter type, `NotificationService`, and looks for a matching bean.

If exactly one suitable bean exists, Spring injects it.

If no suitable bean exists, startup generally fails with an unsatisfied dependency.

If multiple beans match, Spring needs more information.

## 3.5 What happens when multiple beans implement the same interface?

Suppose email and SMS implementations both exist.

Java

```
@Service
public class EmailNotificationService
        implements NotificationService {
    public void send(String message) {}
}
```

Java

```
@Service
public class SmsNotificationService
        implements NotificationService {
    public void send(String message) {}
}
```

Now Spring cannot determine which implementation should satisfy a single `NotificationService` dependency from type alone.

This is a common interview question.

### Solution 1: `@Primary`

Java

```
@Service
@Primary
public class EmailNotificationService
        implements NotificationService {
    public void send(String message) {}
}
```

`@Primary` makes the email implementation the preferred candidate when a single bean is required and no stronger selection rule overrides it.

Use it when one implementation is the natural default.

### Solution 2: `@Qualifier`

Java

```
@Service("emailNotificationService")
public class EmailNotificationService
        implements NotificationService {
    public void send(String message) {}
}
```

Java

```
@Service("smsNotificationService")
public class SmsNotificationService
        implements NotificationService {
    public void send(String message) {}
}
```

Inject the desired implementation explicitly:

Java

```
@Service
public class OrderService {

    private final NotificationService notificationService;

    public OrderService(
            @Qualifier("smsNotificationService")
            NotificationService notificationService) {
        this.notificationService = notificationService;
    }
}
```

Use `@Qualifier` when a specific implementation is required.

### Solution 3: Inject all implementations

Sometimes an application intentionally supports multiple notification channels.

Java

```
@Service
public class NotificationManager {

    private final List<NotificationService> services;

    public NotificationManager(
            List<NotificationService> services) {
        this.services = services;
    }

    public void notifyAllChannels(String message) {
        services.forEach(service -> service.send(message));
    }
}
```

Spring can inject all matching beans into a collection.

This is useful for plugin-style architectures, multiple payment gateways, or processing strategies.

Architect tip: If implementations are selected dynamically based on business rules, do not depend on incidental bean ordering. Build an explicit strategy registry keyed by a business identifier, or use a deliberate routing component.

## 3.6 Can we inject a dependency without using an interface?

Yes.

Java

```
@Service
public class OrderService {

    private final TaxCalculator taxCalculator;

    public OrderService(TaxCalculator taxCalculator) {
        this.taxCalculator = taxCalculator;
    }
}
```

An interface is not mandatory for DI.

However, interfaces are useful when you need interchangeable implementations, explicit contracts, or isolated testing boundaries.

Do not create an interface for every class automatically. Use one when it represents a meaningful abstraction or architectural boundary.

## 3.7 What is circular dependency?

A circular dependency occurs when two or more beans require each other to be constructed.

For example:

```
OrderService → PaymentService
PaymentService → OrderService
```

The `OrderService` needs `PaymentService` to be constructed, but `PaymentService` needs `OrderService` to be constructed.

Neither can be constructed first using ordinary constructor injection.

Spring normally reports an unresolvable circular reference in this situation.

### How should you fix it?

Preferred solution: redesign the dependency relationship.

For example:

```
OrderService → PaymentService
PaymentService → PaymentRepository
```

Possible approaches:

1. Extract shared functionality into a third service.

2. Reverse a dependency using an interface or event-driven design where appropriate.

3. Publish a domain event when one component needs to notify another.

4. Use a lazy dependency or setter injection only when the relationship is genuinely necessary and the lifecycle implications are understood.

Spring has historically supported some circular references involving singleton beans and setter/field injection, but support is constrained, and Spring Boot generally does not allow circular references by default. Do not rely on circular-reference workarounds as an architectural solution.

Interview answer: Circular dependencies usually indicate excessive coupling or unclear responsibility boundaries. I would first refactor the design rather than enable circular references.

# 4. Bean Lifecycle — How Spring Manages Objects

Understanding bean lifecycle is important for senior-level interviews because it explains initialization, resource cleanup, proxies, and startup failures.

## 4.1 The typical lifecycle

A simplified lifecycle looks like this:

```
Bean definition registered
          ↓
Bean instance created
          ↓
Dependencies injected
          ↓
Aware callbacks, where applicable
          ↓
BeanPostProcessor before initialization
          ↓
@PostConstruct
          ↓
InitializingBean.afterPropertiesSet()
          ↓
Custom init method
          ↓
BeanPostProcessor after initialization
          ↓
Bean ready for use
          ↓
Context shutdown
          ↓
@PreDestroy / destruction callbacks
```

This is a conceptual overview, not an exhaustive ordering contract for every extension point. Spring can also use factory methods, custom scopes, proxies, and other lifecycle mechanisms.

## 4.2 Important lifecycle annotations and interfaces

| Mechanism                               | Purpose                                                                                            |
| --------------------------------------- | -------------------------------------------------------------------------------------------------- |
| `@PostConstruct`                        | Runs a callback after dependency injection and before the bean is fully initialized for use.       |
| `InitializingBean.afterPropertiesSet()` | Initialization callback through a Spring interface.                                                |
| Custom init method                      | Explicitly configured initialization callback.                                                     |
| `@PreDestroy`                           | Cleanup callback before destruction when supported by the bean scope and container shutdown.       |
| `DisposableBean.destroy()`              | Destruction callback through a Spring interface.                                                   |
| `BeanPostProcessor`                     | Allows processing of bean instances before and after initialization; important for proxy creation. |
| `BeanFactoryPostProcessor`              | Modifies bean definitions or container metadata before regular bean instantiation.                 |

Prefer `@PostConstruct` and `@PreDestroy` for straightforward lifecycle callbacks because they avoid coupling the class to Spring-specific lifecycle interfaces.

Do not perform expensive or fragile remote network operations in initialization callbacks unless startup must depend on them. A slow remote service can prevent your entire application from starting.

## 4.3 What are bean scopes?

A bean scope controls how Spring creates and shares bean instances.

| Scope         | Meaning                                                                   | Typical use                                             |
| ------------- | ------------------------------------------------------------------------- | ------------------------------------------------------- |
| `singleton`   | One instance per bean definition per Spring container.                    | Stateless services, repositories, shared configuration. |
| `prototype`   | A new instance each time Spring resolves that bean through the container. | Objects with a deliberate per-use lifecycle.            |
| `request`     | One instance per HTTP request.                                            | Request-specific state in web applications.             |
| `session`     | One instance per HTTP session.                                            | Session-specific state where applicable.                |
| `application` | One instance per `ServletContext`.                                        | Web-application-scoped state.                           |
| `websocket`   | One instance per WebSocket session.                                       | WebSocket session state.                                |

### Important interview question: Is a singleton bean thread-safe?

No. Singleton scope does not make an object thread-safe.

A singleton service is usually shared across concurrent requests.

This can be safe:

Java

```
@Service
public class PriceService {

    public BigDecimal calculatePrice(BigDecimal amount) {
        return amount.multiply(new BigDecimal("1.18"));
    }
}
```

This is dangerous:

Java

```
@Service
public class CounterService {

    private int count = 0;

    public void increment() {
        count++;
    }
}
```

Multiple threads can update `count` concurrently, causing lost updates.

For singleton services:

* Prefer stateless design.

* Keep request-specific data in local variables.

* Protect genuinely shared mutable state appropriately.

* Avoid storing user-specific information in instance fields.

### Prototype scope trap

Suppose a singleton bean injects a prototype bean through a constructor.

The singleton receives one prototype instance at its own creation time. It does not automatically receive a fresh prototype instance on every method call.

If you need a new instance per operation, consider `ObjectProvider<T>`, a provider abstraction, or an appropriate factory.

Also, Spring does not fully manage the destruction lifecycle of prototype instances after handing them to the caller.

# 5. IoC Container Internals: BeanFactory vs ApplicationContext

This is a common interview topic.

## 5.1 BeanFactory

`BeanFactory` is the fundamental Spring container abstraction.

It provides bean creation and retrieval functionality.

Use it when you specifically need a lower-level container abstraction or are working with container infrastructure.

## 5.2 ApplicationContext

`ApplicationContext` extends the basic capabilities of `BeanFactory`.

It adds application-level features such as:

* Event publication and listeners.

* Internationalization through message sources.

* Resource loading.

* Integration with environment and configuration facilities.

* Easier integration with application infrastructure.

Spring Boot applications normally use an application context.

## 5.3 Comparison

| Feature                      | `BeanFactory`                                                        | `ApplicationContext`                                            |
| ---------------------------- | -------------------------------------------------------------------- | --------------------------------------------------------------- |
| Basic bean management        | Yes                                                                  | Yes                                                             |
| Application events           | Not as a core feature                                                | Yes                                                             |
| Message resolution           | Basic container does not provide the full application-level facility | Yes                                                             |
| Typical usage                | Lower-level container scenarios                                      | Standard enterprise applications                                |
| Bean initialization behavior | Depends on access and configuration                                  | Commonly initializes non-lazy singletons during context startup |

A subtle point: people often say "`BeanFactory` is lazy and `ApplicationContext` is eager." This is an oversimplification.

`BeanFactory` does not generally pre-instantiate all singleton beans merely because the factory was created. A standard `ApplicationContext` normally pre-instantiates non-lazy singleton beans during refresh, but lazy initialization, scope, and other configuration can change behavior.

# 6. Configuration: `@Component`, `@Bean`, `@Configuration`, and Profiles

## 6.1 When should you use `@Component` versus `@Bean`?

Use `@Component` and its specialized annotations when you own the class and want Spring to discover it through component scanning.

Use `@Bean` when you want to explicitly construct and register an object, particularly a third-party library object.

Example:

Java

```
@Configuration
public class ClientConfiguration {

    @Bean
    public RestClient restClient() {
        return RestClient.create();
    }
}
```

This example uses Spring's `RestClient` API, available in modern Spring Framework versions.

The method's return value becomes a Spring-managed bean.

## 6.2 What is `@Configuration`?

`@Configuration` marks a class as a source of bean definitions.

Java

```
@Configuration
public class ApplicationConfig {

    @Bean
    public TaxCalculator taxCalculator() {
        return new TaxCalculator();
    }
}
```

A full `@Configuration` class is normally enhanced by Spring so that calls between its non-static `@Bean` methods can respect container-managed singleton semantics.

By contrast, using `@Bean` inside a class that is not a full `@Configuration` class may use so-called lite configuration behavior. Direct Java calls between such methods can create new objects rather than resolve the corresponding managed beans.

Architect tip: Prefer method-parameter injection when one bean method depends on another.

Java

```
@Configuration
public class ApplicationConfig {

    @Bean
    public TaxCalculator taxCalculator() {
        return new TaxCalculator();
    }

    @Bean
    public BillingService billingService(
            TaxCalculator taxCalculator) {
        return new BillingService(taxCalculator);
    }
}
```

This makes the dependency explicit.

## 6.3 What are Spring profiles?

Profiles let you register different beans or configuration for different environments.

Java

```
@Configuration
@Profile("dev")
public class DevConfig {

    @Bean
    public PaymentGateway paymentGateway() {
        return new MockPaymentGateway();
    }
}
```

Java

```
@Configuration
@Profile("prod")
public class ProdConfig {

    @Bean
    public PaymentGateway paymentGateway() {
        return new RealPaymentGateway();
    }
}
```

You can activate a profile using configuration such as:

properties

```
spring.profiles.active=prod
```

In production, the profile is often selected through environment-specific deployment configuration.

Best practice: Use profiles for environment-specific bean registration, not as a replacement for all configuration management. Keep secrets in appropriate secret-management systems rather than committing them to configuration files.

## 6.4 What is `@Value`?

`@Value` injects a configured value into a field, parameter, or method.

Java

```
@Value("${payment.timeout:2000}")
private int paymentTimeout;
```

Here, the default is `2000` if the property is not configured.

For larger sets of related configuration, `@ConfigurationProperties` is usually more maintainable because it supports structured, type-safe configuration.

# 7. AOP — Aspect-Oriented Programming

Now let's move to the third major concept.

## 7.1 What is AOP?

AOP stands for Aspect-Oriented Programming.

It helps separate functionality that cuts across many parts of an application from the main business logic.

Imagine every employee in a company must wear an ID card.

You could manually add ID-card checking code to every employee's task. That would be repetitive.

Instead, you can establish a common rule that applies whenever employees enter a restricted area.

AOP applies the same general idea to method execution and other supported join points.

### Common cross-cutting concerns

* Transaction management.

* Authorization checks.

* Logging.

* Metrics and tracing.

* Auditing.

* Performance measurement.

* Some forms of retry and resilience behavior.

Not every cross-cutting concern should be implemented using AOP. For example, logging infrastructure, servlet filters, message interceptors, and explicit application code may be more appropriate depending on the problem.

## 7.2 Without AOP

Suppose you have three services:

Java

```
public void createOrder() {
    log.info("Starting order creation");
    // Business logic
    log.info("Order creation complete");
}
```

Java

```
public void processPayment() {
    log.info("Starting payment");
    // Business logic
    log.info("Payment complete");
}
```

Java

```
public void cancelOrder() {
    log.info("Starting cancellation");
    // Business logic
    log.info("Cancellation complete");
}
```

Logging code gets repeated across services.

Problems include:

* Duplicated code.

* Inconsistent logging.

* Difficult maintenance.

* Business logic mixed with infrastructure behavior.

AOP can centralize the shared behavior.

## 7.3 Important AOP terminology

| Term          | Simple meaning                                                                                              |
| ------------- | ----------------------------------------------------------------------------------------------------------- |
| Aspect        | A module containing a cross-cutting concern, such as auditing.                                              |
| Join point    | A point in program execution where advice may apply. In Spring's proxy-based AOP, this is method execution. |
| Pointcut      | A rule that selects which join points should be intercepted.                                                |
| Advice        | The action performed at a selected join point.                                                              |
| Weaving       | Connecting aspects with the target program.                                                                 |
| Target object | The underlying object whose methods are advised.                                                            |
| Proxy         | An object that intercepts calls and delegates to the target when appropriate.                               |
| Introduction  | Adding an interface or behavior to a target through AOP mechanisms.                                         |

In Spring's standard proxy-based AOP, method execution is the supported join-point model. AspectJ offers a broader model that can include other join points.

## 7.4 Types of advice

### `@Before`

Runs before the target method.

Java

```
@Before("execution(* com.example.service.*.*(..))")
public void logBefore() {
    System.out.println("Method is about to execute");
}
```

### `@After`

Runs after the method finishes, whether it returns normally or exits exceptionally.

Java

```
@After("execution(* com.example.service.*.*(..))")
public void logAfter() {
    System.out.println("Method execution finished");
}
```

### `@AfterReturning`

Runs after successful normal completion.

Java

```
@AfterReturning(
    pointcut = "execution(* com.example.service.*.*(..))",
    returning = "result"
)
public void logSuccess(Object result) {
    System.out.println("Method returned successfully");
}
```

### `@AfterThrowing`

Runs when the target method throws an exception.

Java

```
@AfterThrowing(
    pointcut = "execution(* com.example.service.*.*(..))",
    throwing = "exception"
)
public void logError(Exception exception) {
    System.out.println("Method failed: " + exception.getMessage());
}
```

### `@Around`

Runs around the target method. It can execute code before and after the call, inspect the result, or prevent the target from being invoked.

Java

```
@Around("execution(* com.example.service.*.*(..))")
public Object measureTime(ProceedingJoinPoint joinPoint)
        throws Throwable {

    long start = System.nanoTime();

    try {
        return joinPoint.proceed();
    } finally {
        long duration = System.nanoTime() - start;
        System.out.println("Duration in ns: " + duration);
    }
}
```

`joinPoint.proceed()` continues the intercepted invocation.

If you omit it, the target method may never execute.

If you invoke it multiple times, the target may execute multiple times. This can be dangerous for payments, order creation, or any non-idempotent operation.

## 7.5 How does Spring AOP work internally?

At a high level:

1. Spring creates the target bean.

2. Bean post-processors identify beans requiring advice.

3. Spring creates a proxy when appropriate.

4. Other beans receive the proxy as the dependency.

5. A caller invokes a method through that proxy.

6. The proxy evaluates the applicable interceptors/advice.

7. The target method executes when the invocation proceeds.

Conceptually:

Client

OrderController

Spring AOP Proxy

Logging · Security · Transaction interceptors

Target bean

OrderService business logic

The proxy is an intermediary. It enables Spring to apply behavior without forcing every target method to contain the same infrastructure code.

## 7.6 JDK dynamic proxy vs CGLIB proxy

This is a classic senior-level question.

Spring AOP can use different proxy mechanisms.

| Feature                                              | JDK dynamic proxy                                      | CGLIB-based proxy                                               |
| ---------------------------------------------------- | ------------------------------------------------------ | --------------------------------------------------------------- |
| Mechanism                                            | Implements interfaces                                  | Creates a subclass of the target class                          |
| Interface required                                   | Yes                                                    | No                                                              |
| Can intercept calls through interfaces               | Yes                                                    | Yes, through the proxy                                          |
| Can proxy final classes                              | Not applicable to the interface proxy mechanism itself | No                                                              |
| Can advise final methods through subclass overriding | Not applicable in the same way                         | No                                                              |
| Typical limitation                                   | Consumers generally need to use a proxied interface    | Cannot override final methods and cannot subclass final classes |

Spring's default proxy selection can depend on configuration and framework version. Spring Boot commonly configures class-based proxies by default, but you should check the actual configuration rather than assume the mechanism.

Neither proxy type makes every possible method call interceptable.

## 7.7 What is a pointcut expression?

A pointcut defines which methods should be advised.

Example:

Java

```
@Pointcut("execution(* com.example.service.*.*(..))")
public void allServiceMethods() {
}
```

Meaning:

* `execution` — match method executions.

* `*` before the package — any return type.

* `com.example.service.*` — classes directly in that package.

* `*` after the class pattern — any method name.

* `(..)` — any number and type of arguments.

To include subpackages, you might use:

Java

```
@Pointcut("execution(* com.example.service..*.*(..))")
public void allServiceMethodsIncludingSubpackages() {
}
```

Other useful pointcut designators include `within`, `args`, `@annotation`, and `@within`.

For example, you can apply advice to methods annotated with a custom annotation rather than relying on a broad package pattern.

Architect recommendation: Prefer narrow, intention-revealing pointcuts. A broad pointcut can accidentally intercept unrelated methods and create surprising behavior.

# 8. The Most Important Spring AOP Trap: Self-Invocation

This question frequently distinguishes someone who knows the annotations from someone who understands Spring internals.

Consider:

Java

```
@Service
public class PaymentService {

    public void processPayment() {
        savePayment();
    }

    @Transactional
    public void savePayment() {
        // Save payment
    }
}
```

You might expect `savePayment()` to start a transaction because it has `@Transactional`.

But there is a problem.

The call from `processPayment()` to `savePayment()` is a direct call on the same target object. It does not pass through the Spring proxy.

Therefore, the transaction advice is not applied by that internal call.

This is called self-invocation.

### How to fix it

The preferred solution is often to move the transactional operation into another Spring bean:

Java

```
@Service
public class PaymentWriter {

    @Transactional
    public void savePayment() {
        // Save payment
    }
}
```

Java

```
@Service
public class PaymentService {

    private final PaymentWriter paymentWriter;

    public PaymentService(PaymentWriter paymentWriter) {
        this.paymentWriter = paymentWriter;
    }

    public void processPayment() {
        paymentWriter.savePayment();
    }
}
```

Now the call goes through the `PaymentWriter` proxy.

Other options exist, such as AspectJ weaving, but splitting responsibilities is usually easier to understand and test.

The same general proxy limitation affects many Spring AOP features, not just transactions.

# 9. Spring Transactions — How AOP and IoC Work Together

Let's connect all three concepts to a real production scenario.

Suppose an e-commerce service needs to:

1. Create an order.

2. Reduce inventory.

3. Record the payment state.

These database operations must be consistent.

Java

```
@Service
public class OrderService {

    private final OrderRepository orderRepository;
    private final InventoryRepository inventoryRepository;

    public OrderService(
            OrderRepository orderRepository,
            InventoryRepository inventoryRepository) {
        this.orderRepository = orderRepository;
        this.inventoryRepository = inventoryRepository;
    }

    @Transactional
    public void placeOrder(Order order) {
        orderRepository.save(order);
        inventoryRepository.reduceStock(order.getProductId());
    }
}
```

When another bean invokes `placeOrder()` through the Spring proxy, transaction advice can:

1. Start or join a transaction according to the transaction configuration.

2. Invoke the target method.

3. Commit when execution succeeds.

4. Roll back when the applicable rollback rules say to do so.

By default, Spring's declarative transaction management rolls back for unchecked exceptions (`RuntimeException` and `Error`), not every checked exception. You can customize rollback rules.

### Common transaction interview questions

Q1. Does `@Transactional` guarantee that all code inside the method is transactional?

No. It applies to work managed by the relevant transaction manager and resources participating in that transaction. A remote HTTP call is not automatically part of a local database transaction.

Q2. What happens if a checked exception occurs?

By default, a checked exception does not trigger rollback in the same way as an unchecked exception. Configure `rollbackFor` if the business requirement calls for rollback on that checked exception.

Q3. Can a private method be advised using ordinary proxy-based transaction management?

Not through the usual proxy interception model. Private methods cannot be overridden by a subclass proxy, and they are not externally invocable through an interface proxy.

Q4. Why might a transaction annotation appear to be ignored?

Possible reasons include:

* Self-invocation.

* The bean is not managed by Spring.

* The method is not invoked through the relevant proxy.

* Incorrect transaction-manager configuration.

* The method or class visibility/proxy setup prevents interception.

* The exception does not satisfy rollback rules.

* The operation uses a resource not participating in the transaction.

Q5. Should you keep remote API calls inside a database transaction?

Usually, avoid long-running remote calls inside database transactions when possible. They extend lock duration, consume connections, and increase contention.

For workflows spanning databases and remote services, consider patterns such as transactional outbox, saga orchestration, idempotency, and compensating actions.

Technical architect lesson: AOP makes declarative transactions possible, but it does not eliminate distributed-systems failure modes.

# 10. Advanced DI and Bean Resolution Questions

## 10.1 What is `@Lazy`?

`@Lazy` delays bean initialization or dependency resolution in supported contexts until the bean is needed.

Java

```
@Service
@Lazy
public class ExpensiveReportService {
}
```

It can help with expensive initialization or optional components.

However, lazy initialization can shift failures from startup to the first request. It can also complicate performance debugging.

Do not use it to hide poor startup design or unresolved dependency problems.

## 10.2 What is `ObjectProvider`?

`ObjectProvider<T>` lets you obtain a bean programmatically from the container when needed.

Java

```
@Service
public class ReportCoordinator {

    private final ObjectProvider<ReportGenerator> provider;

    public ReportCoordinator(
            ObjectProvider<ReportGenerator> provider) {
        this.provider = provider;
    }

    public void generate() {
        ReportGenerator generator = provider.getObject();
        generator.generate();
    }
}
```

This is useful for optional beans, deferred resolution, and some prototype-bean use cases.

However, do not replace ordinary constructor injection with provider lookups everywhere. Normal mandatory dependencies should remain explicit.

## 10.3 What is `@DependsOn`?

`@DependsOn` declares an initialization dependency when the relationship is not represented by an ordinary injected dependency.

Java

```
@Component
@DependsOn("databaseInitializer")
public class ApplicationCache {
}
```

Spring initializes the named bean before the dependent bean.

Use this sparingly. If one bean genuinely needs another to perform its job, constructor injection usually communicates that relationship better.

## 10.4 What are `FactoryBean` and `@Bean`?

They are different concepts.

* `@Bean` is an annotation for declaring a bean-producing method.

* `FactoryBean<T>` is a Spring interface for a factory object that exposes another object as its product.

When a bean is a `FactoryBean`, looking it up by its ordinary bean name generally returns the product. Prefixing the name with `&` accesses the factory itself.

This is an advanced mechanism used in some infrastructure integrations.

## 10.5 What are `BeanFactoryPostProcessor` and `BeanPostProcessor`?

These names sound similar but operate at different levels.

| Feature              | `BeanFactoryPostProcessor`                       | `BeanPostProcessor`                       |
| -------------------- | ------------------------------------------------ | ----------------------------------------- |
| Works primarily with | Bean definitions and factory metadata            | Bean instances                            |
| Typical timing       | Before regular bean instantiation                | Around bean initialization                |
| Common use           | Modify configuration metadata                    | Wrap, inspect, or customize created beans |
| Important example    | Property/configuration processing infrastructure | AOP auto-proxy creator                    |

AOP proxies are commonly created by bean post-processing infrastructure.

This is why understanding bean lifecycle helps explain how annotations such as `@Transactional` become runtime behavior.

# 11. Spring AOP vs AspectJ

These are related but not identical technologies.

| Feature                              | Spring AOP                                               | AspectJ                                               |
| ------------------------------------ | -------------------------------------------------------- | ----------------------------------------------------- |
| Main approach                        | Proxy-based interception                                 | Aspect-oriented weaving                               |
| Supported join points                | Method execution                                         | Broader join-point model                              |
| Can advise self-invocation           | Not through ordinary proxy interception                  | Can, when appropriately woven                         |
| Can advise private method executions | Not through ordinary Spring proxy interception           | Possible with suitable weaving                        |
| Setup                                | Usually straightforward in Spring applications           | May require compile-time or load-time weaving         |
| Typical use                          | Transactions, method-level logging, authorization advice | Cross-cutting behavior requiring broader interception |

Spring can integrate with AspectJ, but simply adding an `@Aspect` does not automatically mean AspectJ weaving is being used.

When should you choose Spring AOP?

For typical enterprise application concerns that can be expressed around Spring-managed method calls.

When might AspectJ be appropriate?

When you need interception that proxy-based AOP cannot provide, and the additional build/runtime complexity is justified.

# 12. Production Scenario: Designing a Payment Processing Service

Let's apply IoC, DI, and AOP from a technical architect's perspective.

## Requirements

An application supports multiple payment providers:

* Provider A.

* Provider B.

* A mock provider for testing.

Every payment also needs authorization, transaction handling, metrics, and auditing.

### Step 1: Define an abstraction

Java

```
public interface PaymentGateway {

    PaymentResult charge(PaymentRequest request);
}
```

The business service depends on the contract, not on a specific vendor.

### Step 2: Implement providers

Java

```
@Service("providerA")
public class ProviderAPaymentGateway implements PaymentGateway {

    @Override
    public PaymentResult charge(PaymentRequest request) {
        // Integrate with provider A
        return new PaymentResult();
    }
}
```

Java

```
@Service("providerB")
public class ProviderBPaymentGateway implements PaymentGateway {

    @Override
    public PaymentResult charge(PaymentRequest request) {
        // Integrate with provider B
        return new PaymentResult();
    }
}
```

### Step 3: Inject a deliberate selection component

Java

```
@Service
public class PaymentRouter {

    private final Map<String, PaymentGateway> gateways;

    public PaymentRouter(
            @Qualifier("providerA") PaymentGateway providerA,
            @Qualifier("providerB") PaymentGateway providerB) {

        this.gateways = Map.of(
                "A", providerA,
                "B", providerB
        );
    }

    public PaymentResult charge(
            String provider,
            PaymentRequest request) {

        PaymentGateway gateway = gateways.get(provider);

        if (gateway == null) {
            throw new IllegalArgumentException(
                    "Unsupported payment provider");
        }

        return gateway.charge(request);
    }
}
```

This example uses explicit routing to make the business decision clear.

In a real system, you would also validate provider configuration and handle provider-specific failures, timeouts, and idempotency.

### Step 4: Add cross-cutting behavior

Use declarative transaction management where appropriate for local database work.

Use suitable authorization controls at the application boundary.

Use metrics and structured logging for observability.

Do not assume that one local `@Transactional` annotation makes the external provider call and the local database update atomic.

### Step 5: Design for failure

A production payment architecture must account for:

* Duplicate requests.

* Timeouts after the provider has processed a payment.

* Retries.

* Idempotency keys.

* Reconciliation.

* Partial failures between the provider and database.

* Secure handling of payment data.

* Monitoring and alerting.

IoC and DI make providers replaceable and testable. AOP helps centralize appropriate infrastructure behavior. Neither replaces domain-level failure handling.

### What should an architect say in an interview?

A strong answer might be:

> I would define a payment-gateway abstraction and inject the appropriate implementation rather than hard-code a vendor into the business service. Spring IoC manages the lifecycle, and constructor injection makes dependencies explicit and testable. I would use proxy-based AOP for suitable cross-cutting concerns such as transaction management and metrics, while keeping payment-specific rules in the business layer. Because the workflow spans an external provider and a database, I would also design for idempotency, reconciliation, retries with bounded policies, and partial failures rather than assuming a local transaction guarantees end-to-end atomicity.

That answer demonstrates both Spring knowledge and architectural judgment.

# 13. Common Mistakes and Worst-Case Scenarios

Knowing where Spring Core fails in practice is often more valuable than memorizing definitions.

| Mistake                                | Why it is a problem                                                | Better approach                                               |
| -------------------------------------- | ------------------------------------------------------------------ | ------------------------------------------------------------- |
| Field injection everywhere             | Dependencies are hidden and unit testing is harder                 | Prefer constructor injection                                  |
| Too many constructor parameters        | Often signals excessive responsibilities                           | Split cohesive responsibilities                               |
| Circular service dependencies          | Tight coupling and difficult lifecycle management                  | Refactor dependency direction                                 |
| Mutable state in singleton beans       | Concurrent requests can corrupt shared state                       | Stateless services or appropriate synchronization             |
| Broad AOP pointcuts                    | Unintended methods may be intercepted                              | Use narrow, intention-revealing pointcuts                     |
| Self-invocation with `@Transactional`  | Call bypasses the proxy                                            | Move the operation to another bean or use appropriate weaving |
| Long HTTP calls inside DB transactions | Locks and connections remain occupied longer                       | Minimize transaction duration                                 |
| Swallowing exceptions in advice        | Failures may be hidden and transactions may commit unexpectedly    | Preserve exception semantics deliberately                     |
| Using AOP for business rules           | Behavior becomes difficult to trace                                | Keep domain decisions explicit                                |
| Excessive lazy initialization          | Failures appear during traffic instead of startup                  | Use laziness selectively                                      |
| Assuming singleton means thread-safe   | Singleton beans are shared across threads                          | Design concurrency-safe beans                                 |
| Assuming all objects are Spring beans  | Objects created with `new` are outside normal container management | Register or inject objects when Spring management is needed   |

# 14. Spring Core Interview Questions and Answers

Use these as rapid revision cards.

## Beginner-level questions

### 1. What is Spring Core?

Spring Core provides foundational features such as IoC, DI, bean management, and the infrastructure that supports Spring applications.

### 2. What is IoC?

IoC is the principle of transferring responsibility for object creation and lifecycle management to a framework or container.

### 3. What is DI?

DI is the technique of supplying an object's dependencies from outside the object.

### 4. Is IoC the same as DI?

No. IoC is the broader principle; DI is one way to implement it.

### 5. What is a bean?

An object managed by the Spring container.

### 6. What is the IoC container?

The component responsible for creating, configuring, assembling, and managing beans.

### 7. What is `ApplicationContext`?

An application-level Spring container interface that extends `BeanFactory` capabilities with features such as events and message resolution.

### 8. What is `@Autowired`?

It tells Spring to resolve and inject a suitable dependency at the annotated injection point. For a class with one constructor, `@Autowired` is generally unnecessary.

### 9. Which DI type is recommended?

Constructor injection for mandatory dependencies.

### 10. Can Spring inject primitive values?

Yes. It can inject configured values through mechanisms such as `@Value` and configuration properties.

### 11. Can a bean exist without `@Component`?

Yes. For example, a bean can be declared through an `@Bean` method.

### 12. Are all Java objects Spring beans?

No. Objects created independently using `new` are not automatically managed by Spring.

## Intermediate-level questions

### 13. What is the difference between `@Component` and `@Bean`?

`@Component` marks a class for component scanning. `@Bean` registers the object returned by a configuration method.

### 14. What is the difference between `@Primary` and `@Qualifier`?

`@Primary` indicates a preferred candidate. `@Qualifier` narrows selection to a specifically qualified candidate.

### 15. What happens when Spring finds multiple matching beans?

Spring applies candidate-resolution rules. If it cannot determine a suitable candidate, dependency resolution fails, unless a supported mechanism such as a collection injection or explicit selection resolves the ambiguity.

### 16. What is a circular dependency?

A dependency cycle in which beans require each other, potentially preventing normal construction.

### 17. Is a singleton bean thread-safe?

No. Singleton defines instance scope, not thread safety.

### 18. What is prototype scope?

A new instance is generally created whenever the container resolves the prototype bean. Spring does not automatically manage the full destruction lifecycle after handing it over.

### 19. What is `@Lazy`?

It delays bean initialization or dependency resolution in supported contexts.

### 20. What is `@Profile`?

It enables conditional registration of beans and configuration based on the active profile.

### 21. What is `@ConfigurationProperties`?

It binds external configuration into structured, typed Java objects.

### 22. What is a `BeanPostProcessor`?

An extension point that can process bean instances before and after initialization, including infrastructure used for proxy creation.

### 23. What is a `BeanFactoryPostProcessor`?

An extension point for modifying bean definitions or factory metadata before regular bean instantiation.

### 24. What is the difference between a bean definition and a bean?

A bean definition describes how a bean should be created; a bean is the actual managed instance.

## AOP questions

### 25. What is AOP?

A programming approach for modularizing cross-cutting concerns such as transactions, auditing, and logging.

### 26. What is an aspect?

A module that encapsulates a cross-cutting concern, usually containing pointcuts and advice.

### 27. What is a pointcut?

A predicate or expression that selects the join points where advice should apply.

### 28. What is advice?

The behavior executed at a selected join point.

### 29. What is a proxy?

An intermediary object that can intercept method calls and apply additional behavior before delegating to a target.

### 30. What is `@Around` advice?

Advice that surrounds a method invocation and can control whether and how the target invocation proceeds.

### 31. What is self-invocation?

A method on a target object calling another method on that same object without going through the Spring proxy.

### 32. Why might `@Transactional` not work?

Possible causes include self-invocation, unmanaged objects, proxy limitations, incorrect transaction configuration, or rollback rules that do not match the exception.

### 33. What is the difference between Spring AOP and AspectJ?

Spring AOP primarily uses proxies and method-execution join points. AspectJ supports broader join points and weaving techniques.

### 34. Can Spring AOP intercept private methods?

Not through ordinary proxy-based interception.

### 35. Can Spring AOP intercept static methods?

Not through ordinary proxy-based interception because static method calls do not dispatch through an instance proxy in the required way.

### 36. What happens if `joinPoint.proceed()` is not called?

The target invocation is not continued by that advice. This can be intentional, such as when short-circuiting, but it can also accidentally suppress business logic.

### 37. What happens if `proceed()` is called twice?

The target invocation may execute twice, which can cause duplicate side effects.

## Senior-level and architect-level questions

### 38. How would you make a Spring service testable?

Use constructor injection, depend on meaningful abstractions where useful, isolate external infrastructure, and test business behavior independently from the container where practical.

### 39. How do you design for multiple payment providers?

Define a stable provider abstraction, register implementations, and use an explicit routing or strategy component.

### 40. Would you use AOP for business rules?

Generally, no. Business decisions should remain explicit in the domain or application layer. AOP is best suited to cross-cutting infrastructure behavior.

### 41. Why can a circular dependency indicate a design problem?

It can indicate that responsibilities are entangled or dependencies point in both directions. Extracting shared behavior or using an appropriate event boundary may restore a clearer architecture.

### 42. How would you investigate a bean creation failure?

Check the deepest nested exception, missing configuration, component-scan boundaries, ambiguous candidates, constructor dependencies, circular references, and initialization callbacks.

### 43. How would you investigate an unexpectedly slow Spring Boot startup?

Inspect startup timings, bean initialization, classpath and auto-configuration costs, database connection establishment, remote calls during initialization, and expensive custom post-processors.

### 44. How would you investigate high memory use from Spring beans?

Look for large retained object graphs, unbounded caches, singleton fields holding request data, excessive eager initialization, and thread or connection pools configured beyond actual needs.

### 45. How would you prevent AOP from making production debugging difficult?

Use narrow pointcuts, consistent tracing and logging, clear naming, tests for interception behavior, and explicit documentation for critical advice.

### 46. Is constructor injection always the answer?

It is the recommended default for mandatory dependencies, but not every object needs every dependency injected at construction time. Use providers, factories, or setter injection when there is a clear lifecycle or optionality reason.

### 47. Should every class have an interface?

No. Use interfaces where they express a meaningful contract, variation point, or architectural boundary. Avoid unnecessary abstraction that adds indirection without flexibility.

### 48. Can `@Transactional` make a distributed workflow atomic?

No. A local transaction does not automatically cover remote HTTP services, unrelated databases, or message brokers. Use appropriate distributed-workflow patterns and idempotency controls.

# 15. Scenario-Based Interview Practice

These questions are designed for someone preparing for senior engineering and technical architect interviews.

Try to answer each question aloud before expanding the answer.

## Architect interview drill

7 of 8 revealed

Try to explain the root cause, the Spring mechanism involved, your preferred fix, and the trade-offs.

1. Two payment implementations implement the same interface. Application startup fails with an ambiguous dependency. How do you resolve it?

Discuss @Qualifier, @Primary, collection injection, and explicit business routing.

Hide hint

2. A method annotated with @Transactional is called from another method in the same class, but rollback behavior is missing. Why?

Explain proxy-based AOP, self-invocation, and the separate-bean solution.

Hide hint

3. A singleton service works in testing but produces inconsistent results under concurrent production traffic. What do you investigate?

Inspect mutable instance fields, shared collections, race conditions, and thread safety.

Hide hint

4. An application starts successfully, but its first request fails while initializing an expensive integration. What Spring configuration might explain this?

Consider lazy initialization, bean scope, initialization callbacks, and deferred dependency resolution.

Hide hint

5. A service has 15 constructor dependencies. Would you use field injection to simplify the constructor?

Discuss single responsibility, cohesive services, dependency boundaries, and why hiding dependencies is not a real fix.

Hide hint

6. A database update and a remote payment-provider call must succeed together. Is @Transactional enough?

Separate local ACID transactions from distributed consistency, and discuss idempotency, outbox/saga, and reconciliation.

Hide hint

7. A logging aspect unexpectedly affects methods outside the service layer. How do you fix it?

Reveal hint

8. You need a fresh report generator for each operation, but the coordinator is a singleton. How do you manage the lifecycle?

Compare prototype scope with ObjectProvider or a dedicated factory.

Hide hint

A strong scenario answer usually follows this sequence:

1. Identify the root cause.

2. Explain the Spring mechanism involved.

3. Recommend the simplest safe fix.

4. Explain alternatives and trade-offs.

5. Mention testing, monitoring, and production failure handling.

# 16. Quick Revision: Memorize These Rules

IoC

Spring manages bean creation and lifecycle.

DI

Prefer constructor injection for mandatory dependencies.

Bean scopes

Singleton means one instance per bean definition per container, not thread safety.

Bean resolution

Use `@Primary` for a default candidate and `@Qualifier` for explicit selection.

Bean lifecycle

Know bean definitions, initialization callbacks, post-processors, and destruction.

AOP

Aspects separate cross-cutting concerns; Spring AOP usually works through proxies.

Transactions

Watch for self-invocation, exception rollback rules, and transaction boundaries.

Architecture

Keep business rules explicit, dependencies clear, and service objects as stateless as practical.

## Final takeaway

If you remember only one thing from this tutorial, remember this:

IoC controls object management. DI connects the objects. AOP adds behavior around selected operations.

Together, they let Spring applications remain modular, testable, and maintainable as they grow.

For a technical architect interview, however, definitions are only the starting point. You should be able to explain how these mechanisms work internally, when they fail, what trade-offs they introduce, and how you would design around those limitations in a production system.
