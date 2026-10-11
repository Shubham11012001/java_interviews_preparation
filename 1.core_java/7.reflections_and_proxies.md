# Java Reflection, Annotations & Dynamic Proxy: The Architect's Interview Cheatsheet

> **Who this is for:** an architect-level interview where "what is reflection?" is just the opener. The real questions are *"why does Spring need it?", "what breaks in production?", "how do you do this without it?"*
> **Style:** a 10-year-old should get the idea, and a panel of senior engineers should nod.

---

## Part 0: The Big Picture

Imagine a **toy factory**.

- **Normal Java:** you know every toy at the time you write the code. `new Robot()`. Done.
- **Reflection:** you walk into the factory at runtime and say, *"Show me every toy on the shelf, tell me what buttons it has, and press this one for me."* You didn't know the toy in advance.
- **Annotations:** **sticky notes** on toys. *"FRAGILE"*, *"Needs batteries"*. The notes do nothing by themselves. Someone has to read them.
- **Dynamic Proxy:** a **toy bodyguard**. It looks exactly like the toy. When you press a button, the bodyguard first does something (logs it, checks your ID), then presses the real toy's button.

**How the three work together in real frameworks:**

```
Annotation  →  marks WHAT to do        (@Transactional)
Reflection  →  FINDS the marks         (scans methods for @Transactional)
Proxy       →  ACTS on the marks       (wraps the bean, opens/commits transaction)
```

If you remember one thing: **Spring, Hibernate, JUnit, Jackson and Mockito are these three ideas wearing different hats.**

---

# Part 1: Reflection API

## 1.1 What is it?

**Reflection** = a program **inspecting and modifying its own structure at runtime**: classes, methods, fields, constructors, without knowing them at compile time.

> **ELI10:** Normally you read a toy's instruction manual *before* playing. Reflection is opening the toy *while it's running* to look at its gears and even turn one.

## 1.2 Where does it live?

Package `java.lang.reflect` plus `java.lang.Class`.

| Class | Represents | Think of it as |
|---|---|---|
| `Class<?>` | A type | The blueprint's ID card |
| `Constructor<?>` | A constructor | The "how to build" door |
| `Method` | A method | A button on the toy |
| `Field` | A variable | A drawer inside the toy |
| `Modifier` | public/private/static... | The lock labels |
| `Parameter` | Method param | The slot you put things in |

## 1.3 Step 1: Getting the `Class` object (the entry ticket)

Everything starts here. Three ways:

```java
// 1. From a class name known at compile time
Class<?> c1 = User.class;

// 2. From an object you already have
Class<?> c2 = user.getClass();

// 3. From a String, fully dynamic (this is the "magic" one)
Class<?> c3 = Class.forName("com.shubham.model.User");
```

**Interview gold:**
- `Class.forName()` **loads and initializes** the class (runs static blocks). This is how old JDBC drivers registered themselves.
- `.class` literal does **not** initialize the class.
- Primitives also have class objects: `int.class`, `void.class`.

## 1.4 Our sample class for all examples

```java
public class User {
    private String name = "Shubham";
    private int age = 30;
    public static final String TYPE = "ADMIN";

    public User() {}
    private User(String name, int age) { this.name = name; this.age = age; }

    public String greet(String msg) { return msg + ", " + name; }
    private String secret() { return "Top secret!"; }
}
```

## 1.5 Constructor: creating objects without `new`

```java
Class<?> clazz = Class.forName("com.shubham.model.User");

// Public no-arg constructor
Object u1 = clazz.getDeclaredConstructor().newInstance();

// Private constructor with params
Constructor<?> ctor = clazz.getDeclaredConstructor(String.class, int.class);
ctor.setAccessible(true);                    // break the lock
Object u2 = ctor.newInstance("Ravi", 25);
```

**Why this matters:** this is **exactly how Spring/Hibernate/Jackson create your objects.** They only have a class name and a config file. Hibernate needs a no-arg constructor for this reason.

> `Class.newInstance()` is **deprecated** (it swallows checked exceptions). Always use `getDeclaredConstructor().newInstance()`.

## 1.6 Method: calling methods dynamically

```java
Object user = clazz.getDeclaredConstructor().newInstance();

// Public method
Method greet = clazz.getMethod("greet", String.class);
Object result = greet.invoke(user, "Hello");     // "Hello, Shubham"

// Private method
Method secret = clazz.getDeclaredMethod("secret");
secret.setAccessible(true);
System.out.println(secret.invoke(user));          // "Top secret!"

// Static method: pass null as the target
// method.invoke(null, args);
```

**Useful `Method` APIs:** `getName()`, `getReturnType()`, `getParameterTypes()`, `getParameters()`, `getModifiers()`, `getAnnotation(...)`, `isAnnotationPresent(...)`, `getExceptionTypes()`.

## 1.7 Field: reading and writing variables

```java
Field nameField = clazz.getDeclaredField("name");
nameField.setAccessible(true);

System.out.println(nameField.get(user));         // "Shubham"
nameField.set(user, "Architect Shubham");        // changed a private field!

Field type = clazz.getField("TYPE");             // public static
System.out.println(type.get(null));              // static → null target
```

**This is how dependency injection (`@Autowired` on a private field) works.** Spring just reflects into your private field and sets it.

## 1.8 `getXxx()` vs `getDeclaredXxx()` (asked in nearly every interview)

| | `getMethod / getField / getConstructor` | `getDeclaredMethod / getDeclaredField / getDeclaredConstructor` |
|---|---|---|
| Visibility | **public only** | **all** (private, protected, package) |
| Inheritance | includes **inherited** public members | **only this class**, not inherited |
| Needs `setAccessible(true)`? | No | Yes, for non-public |

> **Memory trick:** *"Declared" = "declared in THIS class, any visibility."* No "Declared" = "public, including inherited."

To walk the full hierarchy for private inherited fields, loop through `clazz.getSuperclass()` yourself.

## 1.9 `setAccessible(true)`: the master key (and why Java 9+ cares)

- Bypasses `private`/`protected`/package checks.
- **Java 9+ modules (JPMS):** if the target class's module doesn't `opens` the package, you get `InaccessibleObjectException`.
- **Java 16+:** strong encapsulation of JDK internals is the default. Reflecting into `java.lang` / `java.util` internals fails unless you pass `--add-opens`.
- Production symptom after upgrading to Java 17: *"Unable to make field private final ... accessible"*. Fix with `--add-opens java.base/java.lang=ALL-UNNAMED` (a short-term patch) or upgrade the library (the real fix).

## 1.10 When / Where / Why

| Question | Answer |
|---|---|
| **When** do you need it? | When types are unknown at compile time: plugins, frameworks, serialization, DI, ORMs, test tools |
| **Where** is it used? | Spring (DI, AOP), Hibernate (entity mapping), Jackson/Gson (JSON), JUnit (finding `@Test`), Mockito, Lombok-free tooling, IDEs, debuggers |
| **Why** does it exist? | To build **generic, configuration-driven, extensible** code without hard-coding classes |

## 1.11 Production scenarios

1. **Dependency Injection:** Spring scans classes, finds `@Component`, creates instances through constructors, and injects `@Autowired` fields.
2. **JSON mapping:** Jackson reads field/getter names and fills your POJO from a JSON string.
3. **ORM:** Hibernate maps table columns to entity fields.
4. **Plugin systems:** load `Class.forName(configuredClassName)` from a properties file.
5. **Unit testing private methods** (generally a smell, but occasionally used for legacy code).
6. **Custom validators:** loop over all fields of a DTO, find `@NotNull`, and validate.

### Mini example: a tiny object-to-map converter (very interview-friendly)

```java
public static Map<String, Object> toMap(Object obj) throws Exception {
    Map<String, Object> map = new HashMap<>();
    for (Field f : obj.getClass().getDeclaredFields()) {
        f.setAccessible(true);
        map.put(f.getName(), f.get(obj));
    }
    return map;
}
```

## 1.12 Best cases vs Worst cases

### Best cases
- Framework/library code where flexibility matters more than nanoseconds
- Startup-time scanning (do it **once**, cache results)
- Tooling, test utilities, serialization

### Worst cases
- **Hot paths** (called millions of times in a loop)
- Business logic that could be a plain interface + polymorphism
- Bypassing encapsulation to "just make it work"

## 1.13 The costs

| Problem | Details |
|---|---|
| **Slower** | No JIT inlining; access checks; boxing of args. The gap has shrunk a lot since Java 18's rewrite on method handles, but is still real |
| **No compile-time safety** | A typo in `"greet"` blows up at **runtime** (`NoSuchMethodException`) |
| **Breaks encapsulation** | Private is no longer private |
| **Security** | `SecurityManager` was the guard (now deprecated for removal); modules are now the main guard |
| **Refactoring hazard** | Rename a method and the string `"oldName"` silently breaks. IDEs can't see it |
| **GraalVM native image / AOT** | Reflection is invisible to static analysis. You must declare it in `reflect-config.json` or hints, which is a **major architect concern with Spring Native** |
| **Obfuscation (ProGuard/R8)** | Renamed members break reflective lookups |

## 1.14 Performance tips (architect-level)

1. **Cache** `Method`/`Field`/`Constructor` objects. Lookup is the expensive part, and `invoke` is cheaper.
2. Call `setAccessible(true)` **once**, not on every call.
3. Prefer **`MethodHandle`** (`java.lang.invoke`) or **`VarHandle`** for hot paths. JIT can inline them when held in `static final`.
4. For maximum speed, **generate code at build time** (annotation processors, as in MapStruct or Dagger) instead of reflecting at runtime.

## 1.15 Common exceptions

| Exception | Meaning |
|---|---|
| `ClassNotFoundException` | `Class.forName` couldn't find the name |
| `NoSuchMethodException` / `NoSuchFieldException` | Wrong name or param types |
| `IllegalAccessException` | Forgot `setAccessible(true)` |
| `InvocationTargetException` | The **invoked method itself threw**. Use `e.getCause()` to see the real error |
| `InstantiationException` | Tried to instantiate an abstract class or interface |
| `InaccessibleObjectException` | Java 9+ module blocked you |

> **Interview trap:** *"Method threw NullPointerException, but I'm catching `InvocationTargetException`. Where's my NPE?"* It's wrapped, so call `getCause()` or `getTargetException()`.

## 1.16 Reflection on generics (type erasure)

```java
List<String> list = new ArrayList<>();
list.getClass()   // ArrayList, no String info at runtime!
```

Generic info is **erased**, but it survives in **signatures** of fields/methods/superclasses:

```java
Field f = MyClass.class.getDeclaredField("names");     // List<String> names;
ParameterizedType pt = (ParameterizedType) f.getGenericType();
Type arg = pt.getActualTypeArguments()[0];              // String.class
```

This is how Jackson's `TypeReference<List<User>>` trick works (an anonymous subclass preserves the type argument).

## 1.17 Reflection can break Singleton (classic question)

```java
Constructor<Singleton> c = Singleton.class.getDeclaredConstructor();
c.setAccessible(true);
Singleton second = c.newInstance();   // a second instance!
```

**Defenses:**
- Throw an exception in the constructor if the instance already exists.
- **Use an `enum` Singleton.** Reflection *cannot* instantiate enums (`IllegalArgumentException: Cannot reflectively create enum objects`).

## 1.18 Reflection and `final` fields

- `final` **instance** fields can be modified with `setAccessible(true)` + `set()` (not recommended).
- `static final` fields: modification fails or is unreliable, because the JIT may have inlined the constant.
- **Records and hidden classes** (Java 15+/16+) can't have their final fields changed reflectively.

## 1.19 Quick Reflection cheat table

| Task | Code |
|---|---|
| Get class | `Class.forName("x.Y")` / `Y.class` / `obj.getClass()` |
| New object | `clazz.getDeclaredConstructor().newInstance()` |
| All fields | `clazz.getDeclaredFields()` |
| All methods | `clazz.getDeclaredMethods()` |
| Public methods incl. inherited | `clazz.getMethods()` |
| Call method | `method.invoke(obj, args)` |
| Get/set field | `field.get(obj)` / `field.set(obj, v)` |
| Unlock private | `setAccessible(true)` |
| Check modifiers | `Modifier.isStatic(m.getModifiers())` |
| Superclass / interfaces | `getSuperclass()` / `getInterfaces()` |
| Annotations | `isAnnotationPresent(X.class)` / `getAnnotation(X.class)` |

---

# Part 2: Annotations and Custom Annotations

## 2.1 What is an annotation?

**Metadata attached to code.** It has **zero behavior on its own**. Something else (compiler, framework, your own code) reads it and acts.

> **ELI10:** A sticky note on your lunchbox saying *"CONTAINS NUTS"*. The note doesn't stop anyone from eating. But the school staff **read** it and act accordingly.

## 2.2 Built-in annotations

| Annotation | Purpose |
|---|---|
| `@Override` | Compiler checks you really override something |
| `@Deprecated` | Warns that the API is old (`forRemoval=true` for a harder warning) |
| `@SuppressWarnings` | Silences compiler warnings |
| `@FunctionalInterface` | Compiler checks the interface has exactly one abstract method |
| `@SafeVarargs` | Suppresses heap pollution warnings for generic varargs |

## 2.3 Meta-annotations (annotations that describe annotations)

These four are **the** interview topic:

### `@Retention`: how long does the sticky note survive?

| Policy | Survives till... | Typical use |
|---|---|---|
| `SOURCE` | Only in `.java` and discarded by the compiler | `@Override`, Lombok |
| `CLASS` (**default**) | In `.class` file, **not** available at runtime | Bytecode tools |
| `RUNTIME` | In `.class` **and** readable via reflection | Spring, JPA, custom runtime annotations |

> **Classic bug:** you wrote a custom annotation, forgot `@Retention(RUNTIME)`, and `isAnnotationPresent()` always returns `false`. The default is `CLASS`.

### `@Target`: where can it be stuck?

`TYPE`, `METHOD`, `FIELD`, `PARAMETER`, `CONSTRUCTOR`, `LOCAL_VARIABLE`, `ANNOTATION_TYPE`, `PACKAGE`, `TYPE_PARAMETER`, `TYPE_USE`, `RECORD_COMPONENT`.

### `@Documented`
Includes the annotation in generated Javadoc.

### `@Inherited`
A class annotation on a **superclass** is visible on **subclasses**. It works **only for class-level annotations**, not on methods, fields, or interfaces.

### `@Repeatable`
Lets you apply the same annotation multiple times (needs a container annotation).

## 2.4 Anatomy of a custom annotation

```java
@Retention(RetentionPolicy.RUNTIME)     // readable at runtime
@Target(ElementType.METHOD)             // only on methods
public @interface LogExecutionTime {
    String value() default "";          // element with default
    boolean enabled() default true;
    int threshold() default 100;        // ms
}
```

**Rules for annotation elements (a favourite question):**
- Allowed types: primitives, `String`, `Class`, enums, other annotations, and **arrays of these**
- **No `null`** values, and **no arbitrary objects**
- Written like methods with **no parameters**
- A single element named `value` can be set without a name: `@Foo("x")`
- Annotations can't `extends` anything. They implicitly extend `java.lang.annotation.Annotation`.

## 2.5 Full working example: a mini validation framework

**Step 1: define the annotation**

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
public @interface NotEmpty {
    String message() default "Field must not be empty";
}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
public @interface MaxLength {
    int value();
}
```

**Step 2: use it**

```java
public class SignupRequest {
    @NotEmpty(message = "Username required")
    @MaxLength(10)
    private String username;

    public SignupRequest(String username) { this.username = username; }
}
```

**Step 3: the processor (this is where reflection brings it alive)**

```java
public class Validator {
    public static List<String> validate(Object obj) throws IllegalAccessException {
        List<String> errors = new ArrayList<>();
        for (Field f : obj.getClass().getDeclaredFields()) {
            f.setAccessible(true);
            Object val = f.get(obj);

            if (f.isAnnotationPresent(NotEmpty.class)) {
                NotEmpty ne = f.getAnnotation(NotEmpty.class);
                if (val == null || val.toString().isBlank())
                    errors.add(f.getName() + ": " + ne.message());
            }
            if (f.isAnnotationPresent(MaxLength.class)) {
                int max = f.getAnnotation(MaxLength.class).value();
                if (val != null && val.toString().length() > max)
                    errors.add(f.getName() + ": max length " + max);
            }
        }
        return errors;
    }
}
```

**That's Hibernate Validator in 20 lines.** The annotation is only a label, and the reflection loop is the engine.

## 2.6 Who reads annotations? Three processing times

| When | Mechanism | Examples | Cost |
|---|---|---|---|
| **Compile time** | Annotation Processors (`javax.annotation.processing`, JSR 269) | Lombok, MapStruct, Dagger, AutoValue | Zero runtime cost, compile-time safe |
| **Class load / build time** | Bytecode tools (ASM, ByteBuddy) | Hibernate bytecode enhancement, AspectJ weaving | Moderate |
| **Runtime** | Reflection | Spring, JPA, JUnit, Jackson | Startup/reflection cost |

> **Architect insight:** the industry trend is **moving from runtime reflection to compile-time processing** (Micronaut, Quarkus, Dagger) for faster startup, lower memory, and GraalVM-native friendliness.

## 2.7 Real annotations you use daily, and how they work

| Annotation | Behind the scenes |
|---|---|
| `@Autowired` | Reflection finds the field/constructor and injects the bean |
| `@Component`, `@Service` | Classpath scanning reads class annotations and registers beans |
| `@Transactional` | **Proxy** wraps the bean; begin/commit/rollback around the method |
| `@Cacheable` | **Proxy** checks the cache before calling the method |
| `@Async` | **Proxy** submits the call to an executor |
| `@Entity`, `@Column` | JPA reads field annotations to map to the table |
| `@JsonProperty` | Jackson reads it to rename JSON keys |
| `@Test` | JUnit scans methods and invokes them reflectively |

## 2.8 Production scenarios for **custom** annotations

1. **`@Audit`**: log who called what (via a proxy or Spring AOP).
2. **`@RateLimit(10)`**: throttle API endpoints.
3. **`@RequiresRole("ADMIN")`**: custom authorization.
4. **`@Retry(times=3)`**: automatic retries on a service method.
5. **`@LogExecutionTime`**: performance metrics.
6. **`@Encrypted`** on fields: auto encrypt/decrypt in a persistence layer.
7. **`@FeatureToggle("new-checkout")`**: turn features on and off.

### Example: `@LogExecutionTime` with Spring AOP (the typical production pattern)

```java
@Aspect
@Component
public class TimingAspect {
    @Around("@annotation(logTime)")
    public Object measure(ProceedingJoinPoint pjp, LogExecutionTime logTime) throws Throwable {
        long start = System.nanoTime();
        try {
            return pjp.proceed();
        } finally {
            long ms = (System.nanoTime() - start) / 1_000_000;
            if (ms > logTime.threshold())
                log.warn("{} took {} ms", pjp.getSignature(), ms);
        }
    }
}
```

## 2.9 Best cases vs Worst cases

### Best
- **Declarative** cross-cutting concerns (logging, security, caching, transactions)
- Config that lives right next to the code it affects
- Clean DSLs for validation, mapping, routing

### Worst
- **Annotation soup:** 8 annotations on one method, and nobody knows the execution order
- **Hidden magic:** new developers can't see *what* happens
- Business logic encoded in annotation attributes
- Annotations that **silently do nothing** (typo'd retention, wrong target, missing processor)

## 2.10 Gotchas that cost real money

1. **Forgot `@Retention(RUNTIME)`**: the annotation is invisible at runtime.
2. **Self-invocation with proxies:** `this.methodB()` inside the same class **bypasses** `@Transactional` / `@Cacheable`, because the proxy isn't in the call path. *(Top Spring interview question. Details in Part 3.)*
3. **`@Transactional` on private/final methods:** ignored with proxy-based AOP.
4. **Annotation inheritance is not automatic:** `@Inherited` only covers class-level annotations on superclasses. Method annotations are **not** inherited by overriding methods.
5. **Order of multiple annotations** isn't guaranteed unless the framework defines it (`@Order`).
6. **Meta-annotations / composed annotations:** Spring supports them (`@RestController` = `@Controller` + `@ResponseBody`), but plain Java reflection doesn't see through them. Spring uses `AnnotatedElementUtils`.

---

# Part 3: Dynamic Proxy

## 3.1 What is it?

A **proxy** is a **stand-in object** that implements the same interface as the real object and controls access to it. A **dynamic** proxy is one **created at runtime**, with no hand-written proxy class.

> **ELI10:** You want to meet the Prime Minister (the real object). You first meet the **secretary** (the proxy). The secretary looks like a valid way to reach the PM, checks your ID, writes your name in a log, then lets you in. And someone can hire a **new secretary on the fly** for any VIP, which is the "dynamic" part.

## 3.2 Static vs Dynamic proxy

| | Static Proxy | Dynamic Proxy |
|---|---|---|
| Created | Hand-written at compile time | Generated at runtime |
| Per-interface effort | One proxy class **per** interface (boilerplate!) | **One** handler serves **any** interface |
| Flexibility | Low | High |
| Use | Simple cases | Frameworks, AOP |

## 3.3 JDK Dynamic Proxy

Lives in `java.lang.reflect.Proxy` plus the `InvocationHandler` interface.

```java
public interface PaymentService {
    void pay(double amount);
}

public class PaymentServiceImpl implements PaymentService {
    public void pay(double amount) { System.out.println("Paid " + amount); }
}
```

```java
public class LoggingHandler implements InvocationHandler {
    private final Object target;
    public LoggingHandler(Object target) { this.target = target; }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("BEFORE: " + method.getName());
        try {
            Object result = method.invoke(target, args);     // call the real thing
            System.out.println("AFTER: " + method.getName());
            return result;
        } catch (InvocationTargetException e) {
            throw e.getCause();                               // unwrap the real exception
        }
    }
}
```

```java
PaymentService real = new PaymentServiceImpl();

PaymentService proxy = (PaymentService) Proxy.newProxyInstance(
        real.getClass().getClassLoader(),          // 1. class loader
        real.getClass().getInterfaces(),           // 2. interfaces to implement
        new LoggingHandler(real));                 // 3. handler

proxy.pay(500);
// BEFORE: pay
// Paid 500.0
// AFTER: pay
```

**The three arguments of `newProxyInstance`:**
1. **ClassLoader** to define the proxy class
2. **Interfaces** the proxy will implement
3. **InvocationHandler** that receives *every* method call

**Key facts:**
- Every method call on the proxy goes to **one place**: `invoke()`.
- The proxy class is generated in memory (named like `$Proxy12`).
- **Requirement: the target must implement an interface.** That is the single biggest limitation.
- Proxy is a **sibling** of the real class (both implement the interface). You **cannot** cast it to `PaymentServiceImpl`.

## 3.4 The CGLIB / ByteBuddy approach (class-based proxy)

When there's **no interface**, libraries generate a **subclass** at runtime:

```java
Enhancer e = new Enhancer();
e.setSuperclass(OrderService.class);
e.setCallback((MethodInterceptor) (obj, method, args, methodProxy) -> {
    System.out.println("before");
    Object r = methodProxy.invokeSuper(obj, args);
    System.out.println("after");
    return r;
});
OrderService proxy = (OrderService) e.create();
```

### JDK Proxy vs CGLIB (**top interview comparison**)

| | JDK Dynamic Proxy | CGLIB / ByteBuddy |
|---|---|---|
| Needs interface? | **Yes** | **No** |
| Mechanism | Implements the interface | **Subclasses** the target |
| `final` class/method | N/A | **Cannot** proxy `final` classes or override `final` methods |
| `private` methods | Not on the interface | Not overridden (subclass can't see them) |
| Needs no-arg constructor? | No | Historically yes (Spring now uses Objenesis to avoid it) |
| Where | `java.lang.reflect` (JDK built-in) | Third-party (Spring repackages CGLIB; Hibernate uses ByteBuddy) |
| Spring Boot default | Since Boot 2.0, **CGLIB is the default** (`spring.aop.proxy-target-class=true`) |

## 3.5 How Spring uses all three ideas together

```java
@Service
public class OrderService {
    @Transactional
    public void placeOrder(Order o) { ... }
}
```

**What actually happens:**

```
Caller → [ PROXY ] → real OrderService
            │
            ├─ 1. Reflection found @Transactional on placeOrder
            ├─ 2. Open transaction
            ├─ 3. Call real placeOrder()
            ├─ 4. Commit (or rollback on RuntimeException)
            └─ 5. Return result
```

Spring replaced your bean in the container with a **proxy**. You never see it, but `AopUtils.isAopProxy(bean)` would say `true`.

## 3.6 The famous self-invocation trap

```java
@Service
public class OrderService {

    public void checkout() {
        this.placeOrder();          // calls the REAL object directly. No proxy!
    }

    @Transactional
    public void placeOrder() { ... }   // transaction will NOT start
}
```

**Why:** the transaction logic lives in the **proxy**. `this` is the **real object**, so the call never goes through the proxy.

**Fixes:**
1. Move `placeOrder()` to **another bean** and inject it (cleanest).
2. Inject self (`@Lazy @Autowired OrderService self`) and call `self.placeOrder()`.
3. Use **AspectJ weaving** (compile-time/load-time), which modifies the bytecode itself.
4. Use `TransactionTemplate` programmatically.

## 3.7 When / Where / Why

| Question | Answer |
|---|---|
| **When** | When you want to add behavior **around** method calls **without touching** the target's code |
| **Where** | Spring AOP (`@Transactional`, `@Cacheable`, `@Async`, `@Secured`), Hibernate lazy loading, Mockito mocks, Feign/Retrofit/Spring Data repositories (an interface, no impl!), RMI, MyBatis mappers |
| **Why** | Separates **cross-cutting concerns** (logging, security, transactions, caching, metrics, retries) from business logic: DRY and clean |

## 3.8 Production scenarios

1. **Transaction management:** `@Transactional`.
2. **Declarative HTTP clients:** `@FeignClient interface UserClient { ... }`. You write an interface, and a proxy performs the HTTP calls.
3. **Spring Data JPA:** `interface UserRepository extends JpaRepository<User, Long>`. There is **no implementation class**. Spring creates a proxy that parses method names like `findByEmail`.
4. **Hibernate lazy loading:** `user.getOrders()` returns a **proxy** that hits the database only when first accessed. This is the source of `LazyInitializationException` once the session is closed.
5. **Mockito:** `mock(PaymentService.class)` is a dynamic proxy/subclass recording calls.
6. **Resilience:** circuit breaker/retry/bulkhead wrappers (Resilience4j decorators).
7. **Remote calls:** a client-side proxy makes a network call look like a local call.
8. **Performance monitoring:** auto-time every service method for metrics (Micrometer `@Timed`).

## 3.9 Best cases vs Worst cases

### Best
- Cross-cutting behavior applied **uniformly** across many beans
- Interface-driven designs
- Replacing boilerplate (Feign, Spring Data)
- Lazy initialization and remote-call stubs

### Worst
- **Hot paths:** every call goes through `invoke()` (plus reflection/boxing).
- **Deep proxy stacks:** transaction + security + cache + retry + metrics on every method makes stack traces and debugging painful.
- Proxying `final`/`private`/`static` methods (silently not intercepted).
- Code that relies on **object identity** (`proxy != target`, `equals`, `instanceof` on the concrete class).
- **GraalVM native image:** runtime proxy generation needs build-time hints (Spring AOT handles most of it).

## 3.10 Proxy gotchas checklist

| Gotcha | Result |
|---|---|
| Self-invocation | Advice skipped |
| `private` / `static` / `final` method | Advice skipped |
| `final` class with CGLIB | Startup failure |
| Casting JDK proxy to the concrete class | `ClassCastException` |
| Checked exceptions not declared on the interface | `UndeclaredThrowableException` |
| Forgetting to unwrap `InvocationTargetException` | Callers see the wrong exception |
| Hibernate proxy and `getClass() == Entity.class` | **False!** Use `Hibernate.getClass(obj)` / `instanceof` |
| Lazy proxy used after session closed | `LazyInitializationException` |
| `@Transactional` rolling back only on `RuntimeException` | Checked exceptions **commit** unless `rollbackFor` is set |
| Proxying objects in `@PostConstruct` | Proxy may not be applied yet during init |

> **Interview gem, `UndeclaredThrowableException`:** if the handler throws a **checked exception not declared** by the interface method, the JDK proxy wraps it in this runtime exception.

## 3.11 Java 16+ note: default methods

Calling an interface's **default method** from `InvocationHandler` needs `InvocationHandler.invokeDefault(proxy, method, args)` (Java 16+). Before that it required hacky `MethodHandles.Lookup` tricks.

---

# Part 4: How the Three Fit Together (Mini Spring in 40 lines)

```java
// 1. ANNOTATION: the sticky note
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@interface Timed {}

// 2. SERVICE
interface Calculator { @Timed int add(int a, int b); }
class CalculatorImpl implements Calculator {
    public int add(int a, int b) { return a + b; }
}

// 3. PROXY FACTORY: uses REFLECTION to read annotations, PROXY to act
class ProxyFactory {
    @SuppressWarnings("unchecked")
    static <T> T wrap(T target, Class<T> iface) {
        return (T) Proxy.newProxyInstance(
            iface.getClassLoader(), new Class<?>[]{iface},
            (proxy, method, args) -> {
                // reflection: look for the annotation on the REAL method
                Method real = target.getClass()
                        .getMethod(method.getName(), method.getParameterTypes());
                boolean timed = real.isAnnotationPresent(Timed.class)
                             || method.isAnnotationPresent(Timed.class);
                long t0 = System.nanoTime();
                try {
                    return method.invoke(target, args);
                } catch (InvocationTargetException e) {
                    throw e.getCause();
                } finally {
                    if (timed)
                        System.out.println(method.getName() + " took "
                            + (System.nanoTime() - t0) + " ns");
                }
            });
    }
}

// USAGE
Calculator calc = ProxyFactory.wrap(new CalculatorImpl(), Calculator.class);
calc.add(2, 3);   // prints: add took ... ns
```

**This is the entire idea behind Spring AOP.** Everything else is scale, pointcut expressions, and ordering.

---

# Part 5: Alternatives (a common architect follow-up)

| Need | Alternative to reflection/proxy | Trade-off |
|---|---|---|
| Fast dynamic call | `MethodHandle` / `VarHandle` | Faster, more verbose API |
| Avoid runtime scanning | **Annotation processors** (Dagger, MapStruct) | Compile-time safe, zero startup cost, but more build complexity |
| Cross-cutting without proxies | **AspectJ** weaving | Handles self-invocation, final, private. But a complex build |
| Native-image friendly | Micronaut / Quarkus (build-time DI), Spring AOT | Less runtime magic |
| Plugin loading | `ServiceLoader` (SPI) | Standard, no string class names |
| Pure decoration | **Decorator pattern** | Explicit, type-safe, but boilerplate |
| Generate classes | ByteBuddy / ASM | Powerful, low-level |

---

# Part 6: Interview Q&A (rapid-fire)

**Q1. What is reflection and why is it needed?**
Runtime inspection/manipulation of classes, methods, fields, and constructors. It's needed for generic frameworks (DI, ORM, serialization, testing) where types are unknown at compile time.

**Q2. Difference between `getMethod` and `getDeclaredMethod`?**
`getMethod` returns public methods, including inherited ones. `getDeclaredMethod` returns any-visibility methods declared in that class only.

**Q3. What does `setAccessible(true)` do and what are its limits?**
It suppresses Java access checks. On Java 9+, the module system can still block it unless the package is opened (`--add-opens`).

**Q4. What are the downsides of reflection?**
Slower; no compile-time checking; breaks encapsulation; hard to refactor; problems for native image/obfuscation; security risk.

**Q5. How does reflection break the Singleton? How do you prevent it?**
It calls the private constructor with `setAccessible(true)`. Prevent it with a constructor guard or an **enum singleton**.

**Q6. What are the types of `@Retention`? Default?**
`SOURCE`, `CLASS`, `RUNTIME`. The default is `CLASS`. Only `RUNTIME` is visible via reflection.

**Q7. Can annotations have inheritance?**
Annotations can't extend other annotations. `@Inherited` lets subclasses see a **class-level** annotation from the superclass.

**Q8. What can an annotation element's type be?**
Primitives, `String`, `Class`, enum, annotation, or arrays of those. Never `null`.

**Q9. Does an annotation do anything by itself?**
No. It's metadata. A processor (compiler plugin, framework, or your reflection code) gives it meaning.

**Q10. How would you build a custom `@Validate` annotation?**
Define it with `@Retention(RUNTIME)` and `@Target(FIELD)`, then write a processor that loops over fields via reflection, reads the annotation, and applies the rule. (Part 2.5.)

**Q11. JDK proxy vs CGLIB?**
JDK needs an interface and implements it. CGLIB subclasses the target and can't handle `final`. Spring Boot defaults to CGLIB.

**Q12. How does `@Transactional` work internally?**
Spring wraps the bean in a proxy. The proxy's interceptor opens a transaction, calls the real method, then commits or rolls back.

**Q13. Why doesn't `@Transactional` work when I call another method in the same class?**
Self-invocation bypasses the proxy. Fix by moving the method to another bean, self-injecting, or using AspectJ.

**Q14. How does Spring Data JPA work with just an interface?**
At startup it creates a **dynamic proxy** for the repository interface. Method calls are intercepted and translated into queries.

**Q15. What's `InvocationTargetException`?**
A wrapper thrown when the *invoked method itself* throws. Use `getCause()` to get the real exception.

**Q16. How do you make reflection faster?**
Cache `Method`/`Field` objects, call `setAccessible` once, use `MethodHandle`, or move to compile-time code generation.

**Q17. How does reflection affect GraalVM native image?**
The static analysis can't see reflective access, so you must register it in config files or hints. Spring AOT and Quarkus generate these automatically.

**Q18. Can you proxy a `final` class?**
Not with CGLIB (no subclassing). With JDK proxy, only if you proxy an *interface* it implements.

**Q19. Where have you used these in production?** *(tailor this to your project)*
Example: *"Custom `@Audit` annotation plus AOP aspect to log API access in a financial analytics platform; relied on proxy-based advice, and we hit the self-invocation issue once in an `@Async` call and fixed it by extracting a separate bean."*

**Q20. Reflection vs MethodHandle?**
Reflection checks access on every call and is heavier. `MethodHandle` checks access at lookup time, is typed, and can be inlined by the JIT when held in a `static final`.

---

# Part 7: The 60-Second Memory Cheatsheet

```
REFLECTION   = inspect/modify classes at runtime
  Class → Constructor → newInstance()
        → Method      → invoke()
        → Field       → get()/set()
  getX = public+inherited | getDeclaredX = all, this class only
  setAccessible(true) = master key (modules may block it, Java 9+)
  Costs: slow, no compile safety, breaks encapsulation, native-image pain
  Fix:   cache, MethodHandle, compile-time generation

ANNOTATION   = sticky note, zero behavior alone
  @Retention: SOURCE < CLASS(default) < RUNTIME   ← need RUNTIME to read
  @Target, @Inherited (class-level only), @Documented, @Repeatable
  Elements: primitives, String, Class, enum, annotation, arrays. No null.
  Processed: compile time (Lombok) | load time (ASM) | runtime (Spring)

DYNAMIC PROXY = runtime stand-in that intercepts calls
  JDK:   Proxy.newProxyInstance(loader, interfaces, handler) → NEEDS interface
  CGLIB: subclass target → no final, no private
  Spring Boot default = CGLIB
  TRAP: self-invocation bypasses proxy → @Transactional ignored
  TRAP: Hibernate proxy → getClass() != Entity.class

COMBO: Annotation (what) + Reflection (find) + Proxy (act) = Spring magic
```

---

## Final Words

When an interviewer asks about these topics, they're **not** checking whether you memorized method names. They're checking whether you understand the **trade-off**: *flexibility vs. speed, magic vs. clarity, runtime vs. compile-time.*

An architect's answer always ends with:

> *"Reflection and proxies give us powerful, declarative code, but I'd cache lookups, avoid them in hot paths, watch out for self-invocation and final/private limits, and prefer compile-time alternatives where startup time or native-image support matters."*

Good luck with the interviews! 🚀