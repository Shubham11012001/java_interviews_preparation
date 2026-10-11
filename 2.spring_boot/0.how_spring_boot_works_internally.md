# How Spring Boot Works Internally: The Interview Cheat Sheet

*A plain-English guide with a 10-year-old's analogies, production scenarios for a Cloud Engineer, and rapid-fire interview answers.*

---

## 1. The 30-second answer (memorize this)

> **Spring Boot is an opinionated layer on top of the Spring Framework. It looks at what libraries are on your classpath, automatically configures the beans you'd normally wire up by hand, embeds a web server (Tomcat by default), and packages everything into a single runnable JAR.**

Under the hood, three ideas do all the work:

1. **IoC container:** Spring creates and manages your objects (beans).
2. **Auto-configuration:** Spring Boot guesses what you need using `@Conditional` rules.
3. **Embedded server:** your app carries its own web server inside the JAR.

---

## 2. The 10-year-old analogy 🍕

Imagine a **pizza shop**.

| Real world | Spring Boot world |
|---|---|
| You want pizza, not farming wheat | You want business logic, not wiring objects |
| The shop manager | **ApplicationContext (IoC container)** |
| Ingredients and staff the manager hires | **Beans** |
| The manager sees you ordered cheese, so cheese is prepped | **Auto-configuration**: it sees a DB driver, so it sets up a DataSource |
| Pre-made meal combo boxes | **Starters** |
| The shop includes its own oven | **Embedded Tomcat** |
| The front-desk person who routes your order to the right chef | **DispatcherServlet** |
| Shop settings (opening hours, prices) in a notice board | **application.properties / YAML** |

**Old Spring** = you build the kitchen, buy the oven, hire everyone, and write the rulebook (XML).
**Spring Boot** = you walk into a fully equipped kitchen and say "make me a pizza".

---

## 3. Why Spring Boot exists (the problem)

**Before Spring Boot (plain Spring):**
- Lots of XML or Java config for every tiny thing
- Dependency version conflicts ("JAR hell")
- You had to install Tomcat separately and deploy a `.war` file
- Slow project setup

**Spring Boot fixed this with:**
- **Convention over configuration:** sensible defaults
- **Starters:** curated dependency bundles with compatible versions
- **Auto-configuration:** automatic bean setup
- **Embedded server:** run with `java -jar app.jar`
- **Actuator:** production-ready health and metrics out of the box

---

## 4. The Big Picture

```
        java -jar app.jar
               │
               ▼
   ┌──────────────────────────┐
   │  main() → SpringApplication.run()  │
   └──────────────────────────┘
               │
    ┌──────────┼───────────────┐
    ▼          ▼               ▼
 Load config   Create          Auto-configure
 (properties,  ApplicationContext   (@Conditional)
  env, profiles) (IoC container)
               │
               ▼
     Scan & create Beans
     (@Component, @Service, @Repository, @Controller)
               │
               ▼
     Start embedded Tomcat
               │
               ▼
     App is READY to serve requests
```

---

## 5. `@SpringBootApplication`: the magic annotation

```java
@SpringBootApplication
public class MyApp {
    public static void main(String[] args) {
        SpringApplication.run(MyApp.class, args);
    }
}
```

This single annotation is a **combo of three**:

| Annotation | What it does |
|---|---|
| `@Configuration` | This class can define beans (`@Bean` methods) |
| `@EnableAutoConfiguration` | Turn on Spring Boot's guessing magic |
| `@ComponentScan` | Search this package **and sub-packages** for `@Component` classes |

**⚠️ Classic interview trap:** `@ComponentScan` scans from the main class's package downward. If your `@Service` is in a *sibling or parent* package, it will **not** be found, and you'll get `NoSuchBeanDefinitionException`.

---

## 6. What happens inside `SpringApplication.run()`

This is the **most asked internal question**. Learn the order:

1. **Create `SpringApplication`**: detects the app type (Servlet / Reactive / None) from the classpath and loads initializers and listeners.
2. **Start listeners**: publishes `ApplicationStartingEvent`.
3. **Prepare Environment**: loads `application.properties/yml`, env variables, command-line args, and active profiles.
4. **Print banner** (the ASCII Spring logo 😄).
5. **Create the ApplicationContext**: for a web app it's `AnnotationConfigServletWebServerApplicationContext`.
6. **Prepare the context**: registers your main class as the first bean source.
7. **Refresh the context.** This is the big one:
   - Load bean definitions (component scan plus auto-config imports)
   - Run `BeanFactoryPostProcessor`s
   - Register `BeanPostProcessor`s
   - **Create the embedded web server** (Tomcat)
   - **Instantiate all singleton beans**
   - Start the web server, so the port opens
8. **Call runners**: `CommandLineRunner` and `ApplicationRunner` beans execute.
9. **Publish `ApplicationReadyEvent`**: the app is live.

**Memory trick:** *"Create → Prepare → Refresh → Run."*

---

## 7. IoC and Dependency Injection (the heart of Spring)

**Kid version:** instead of *you* building your toys, a **toy-manager** hands you the toy when you need it.

```java
@Service
public class OrderService {
    private final PaymentService paymentService;

    // Spring injects PaymentService automatically
    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

**Three injection types:**

| Type | Use it? | Why |
|---|---|---|
| **Constructor injection** | ✅ Best | Immutable, testable, fails fast, required deps are obvious |
| Setter injection | ⚠️ Optional deps only | Can leave the object half-initialized |
| Field injection (`@Autowired` on field) | ❌ Avoid | Hard to test, hides dependencies, enables circular deps |

**Stereotype annotations:**
- `@Component`: generic bean
- `@Service`: business logic
- `@Repository`: data access, plus translates DB exceptions into Spring's `DataAccessException`
- `@Controller` / `@RestController`: web layer

---

## 8. Bean Lifecycle (very commonly asked)

```
Instantiate (constructor)
      ↓
Populate dependencies (DI)
      ↓
Aware callbacks (BeanNameAware, ApplicationContextAware)
      ↓
BeanPostProcessor.postProcessBeforeInitialization
      ↓
@PostConstruct
      ↓
InitializingBean.afterPropertiesSet()
      ↓
Custom init-method
      ↓
BeanPostProcessor.postProcessAfterInitialization  ← AOP proxies are created HERE
      ↓
✅ Bean is READY
      ↓
@PreDestroy → DisposableBean.destroy() → custom destroy-method
```

**Bean scopes:**

| Scope | Meaning |
|---|---|
| `singleton` (default) | One instance per container |
| `prototype` | New instance every time it's requested |
| `request` | One per HTTP request |
| `session` | One per HTTP session |

**⚠️ Trap:** singleton beans are shared across threads, so **never keep mutable state in them**. This is a classic production bug.

---

## 9. Auto-Configuration: how the "magic" really works

**Kid version:** the shop manager peeks in your bag. "Oh, you brought a DB driver? I'll set up the database connection for you!"

### Step by step

1. `@EnableAutoConfiguration` triggers an importer.
2. Spring Boot reads a list of candidate config classes from:
   - **Boot 2.7+ / 3.x:** `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`
   - **Older (≤2.6):** `META-INF/spring.factories`
3. Each candidate class has `@Conditional` rules deciding whether it activates.

### The conditions

| Annotation | Activates when… |
|---|---|
| `@ConditionalOnClass` | A class is on the classpath |
| `@ConditionalOnMissingClass` | A class is **not** on the classpath |
| `@ConditionalOnBean` | A bean already exists |
| `@ConditionalOnMissingBean` | **No** bean of that type exists (this lets you override defaults) |
| `@ConditionalOnProperty` | A property has a given value |
| `@ConditionalOnWebApplication` | App is a web app |

### Example (simplified)

```java
@AutoConfiguration
@ConditionalOnClass(DataSource.class)
public class DataSourceAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean   // ← if YOU define a DataSource, this backs off
    public DataSource dataSource(DataSourceProperties props) {
        return props.initializeDataSourceBuilder().build();
    }
}
```

**Golden rule:** *Your config always wins over auto-config.* If you define your own bean, Boot backs off.

### Debugging auto-config
- Run with `--debug` or set `debug=true` to print the **Conditions Evaluation Report**. It shows what matched and what didn't, and why.
- Exclude something: `@SpringBootApplication(exclude = DataSourceAutoConfiguration.class)`

---

## 10. Starters: combo meals

A **starter** is a dependency that pulls in a curated set of libraries with compatible versions.

| Starter | Brings |
|---|---|
| `spring-boot-starter-web` | Spring MVC, Tomcat, Jackson |
| `spring-boot-starter-data-jpa` | Hibernate, Spring Data JPA, HikariCP |
| `spring-boot-starter-security` | Spring Security |
| `spring-boot-starter-actuator` | Health and metrics endpoints |
| `spring-boot-starter-test` | JUnit, Mockito, AssertJ |
| `spring-boot-starter-webflux` | Reactive stack (Netty) |

**How versions stay compatible:** `spring-boot-starter-parent` (or the BOM via `spring-boot-dependencies`) manages versions centrally, so you don't specify versions for most dependencies.

---

## 11. Embedded Server and the Request Flow

Instead of deploying a WAR into an external Tomcat, **Tomcat lives inside your JAR** and starts when the context refreshes.

**Switch servers:** exclude `spring-boot-starter-tomcat`, add `spring-boot-starter-jetty` or `-undertow`.

### What happens when a request arrives

```
Client (browser / Postman)
    ↓
Embedded Tomcat (accepts connection, picks a thread)
    ↓
Servlet Filters (security, logging, CORS)
    ↓
DispatcherServlet  ← the "front desk" (Front Controller pattern)
    ↓
HandlerMapping (which controller method handles this URL?)
    ↓
Interceptors → preHandle()
    ↓
HandlerAdapter → calls your @RestController method
    ↓
Argument resolvers (@RequestBody, @PathVariable) convert input
    ↓
Your controller → service → repository → DB
    ↓
HttpMessageConverter (Jackson turns Object → JSON)
    ↓
Interceptors → postHandle / afterCompletion
    ↓
Response goes back to client
```

**Exceptions:** handled by `@ControllerAdvice` + `@ExceptionHandler`.

**Threading model (Servlet stack):** one thread per request. Tomcat defaults are roughly 200 max threads, so slow downstream calls can exhaust the pool.

---

## 12. Externalized Configuration and Profiles

### Property sources (higher wins), simplified order

1. Command-line args (`--server.port=9000`)
2. Java system properties (`-Dserver.port=9000`)
3. **OS environment variables** (`SERVER_PORT=9000`)
4. Profile-specific files (`application-prod.yml`)
5. `application.properties` / `application.yml`
6. Defaults in code (`@Value("${x:default}")`)

### Profiles

```yaml
# application-prod.yml
spring:
  datasource:
    url: jdbc:postgresql://prod-db:5432/app
```

Activate with: `SPRING_PROFILES_ACTIVE=prod`

### Reading config

```java
@Value("${app.timeout}")          // simple, scattered
private int timeout;

@ConfigurationProperties(prefix = "app")   // ✅ type-safe, grouped, validated
public record AppProps(int timeout, String name) {}
```

**Best practice:** use `@ConfigurationProperties` for groups of settings, and **never hardcode secrets** in properties files.

---

## 13. AOP, Proxies and `@Transactional` (internals interviewers love)

Spring implements `@Transactional`, `@Cacheable`, `@Async`, and `@Retryable` using **proxies**, objects that wrap your bean.

```
Caller → [Proxy: begins transaction] → Your real bean method → [Proxy: commit/rollback]
```

- **JDK dynamic proxy:** used when the bean implements an interface.
- **CGLIB proxy:** subclasses your class (the Boot default is CGLIB).
- Proxies are created in `BeanPostProcessor.postProcessAfterInitialization`.

### 🚨 The self-invocation trap

```java
@Service
public class UserService {
    public void outer() {
        this.inner();   // ❌ calls the REAL object, bypasses the proxy → NO transaction!
    }

    @Transactional
    public void inner() { ... }
}
```

**Fix:** move `inner()` to another bean, or inject a self-reference.

### Other `@Transactional` gotchas
- Only works on **public** methods (via proxy).
- By default rolls back on **unchecked** exceptions (`RuntimeException`, `Error`), **not** checked ones. Use `rollbackFor = Exception.class` if needed.
- A long transaction holds a DB connection, which can drain the pool.

---

## 14. Spring Boot Actuator (production-readiness)

Add `spring-boot-starter-actuator`:

| Endpoint | Purpose |
|---|---|
| `/actuator/health` | Is the app healthy? |
| `/actuator/health/liveness` | Is the process alive? (restart if not) |
| `/actuator/health/readiness` | Can it take traffic? |
| `/actuator/metrics` | JVM, HTTP, DB pool metrics (Micrometer) |
| `/actuator/env`, `/beans`, `/conditions` | Debug config, beans, auto-config |
| `/actuator/prometheus` | Metrics in Prometheus format |

**Security:** expose only what's needed, with `management.endpoints.web.exposure.include=health,info,prometheus`. **Never expose `env`, `heapdump`, or `beans` publicly.**

---

## 15. 🌩️ Production Scenarios on Google Cloud (your edge)

As an Associate Cloud Engineer, tie Spring Boot to GCP. This stands out in interviews.

### Where to deploy

| Service | Best for | Spring Boot notes |
|---|---|---|
| **Cloud Run** | Stateless APIs, scale-to-zero, simplest | Listens on `$PORT`. Watch **cold starts**. Set min instances = 1 for latency-sensitive services |
| **GKE** | Complex microservices, full control | Use liveness/readiness probes, HPA, resource limits |
| **App Engine** | Simple managed hosting | Less control, fine for small apps |
| **Compute Engine (VM)** | Legacy or special needs | You manage everything |

### Mapping Spring Boot features to GCP/Kubernetes

| Need | Spring Boot side | GCP side |
|---|---|---|
| Health checks | Actuator probes | GKE liveness/readiness probes, Cloud Run startup probe |
| Config | `application-prod.yml` + env vars | ConfigMaps / Cloud Run env vars |
| Secrets | `SPRING_DATASOURCE_PASSWORD` env var | **Secret Manager** (mounted or injected), never in the JAR |
| Database | HikariCP + JDBC | **Cloud SQL** (via Cloud SQL Auth Proxy or connector), private IP |
| Auth to GCP APIs | Spring Cloud GCP / client libs | **Workload Identity** / service accounts, no key files |
| Logging | JSON logs to stdout | Cloud Logging picks it up automatically |
| Metrics | Micrometer + Prometheus | Cloud Monitoring / Managed Prometheus |
| Tracing | Micrometer Tracing | Cloud Trace |
| Containers | Buildpacks (`mvn spring-boot:build-image`) or Dockerfile | Artifact Registry, then deploy |

### Production-grade config snippet

```yaml
server:
  shutdown: graceful                 # finish in-flight requests on SIGTERM
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
  datasource:
    hikari:
      maximum-pool-size: 10          # tune against Cloud SQL connection limits!
      connection-timeout: 30000
management:
  endpoint:
    health:
      probes:
        enabled: true                # liveness/readiness for K8s
  endpoints:
    web:
      exposure:
        include: health,info,prometheus
```

### Real production issues

**1. Connection pool exhaustion**
- *Symptom:* requests hang, "Connection is not available, request timed out".
- *Cause:* slow queries, long `@Transactional` blocks, or too many pods × pool size exceeding the Cloud SQL `max_connections`.
- *Math:* 20 pods × pool 10 = **200 connections**. Does your DB allow it?
- *Fix:* right-size the pool, keep transactions short, use a connection pooler.

**2. Cold starts on Cloud Run**
- *Cause:* JVM startup plus Spring context init (can be several seconds).
- *Fixes:* min instances ≥ 1, lazy init (`spring.main.lazy-initialization=true`, with trade-offs), **GraalVM native image**, CDS/AOT, trim unused starters, enable startup CPU boost.

**3. OOMKilled in containers**
- *Cause:* JVM heap + metaspace + threads exceed the container memory limit.
- *Fix:* set `-XX:MaxRAMPercentage=75`, leave headroom for non-heap memory, and use a container-aware JDK (Java 17+ is).

**4. Dropped requests during deployments**
- *Cause:* the pod is killed while serving requests.
- *Fix:* `server.shutdown=graceful`, readiness probe, and a `preStop` hook or proper termination grace period.

**5. Pod restarts in a loop (CrashLoopBackOff)**
- *Cause:* the liveness probe fires before the app finishes starting.
- *Fix:* use a **startup probe** or a longer `initialDelaySeconds`. Don't make the liveness probe depend on external systems like the DB (that causes cascading restarts). Put those checks in readiness.

**6. Secrets leaked**
- *Cause:* passwords in `application.properties` committed to Git, or `/actuator/env` exposed.
- *Fix:* Secret Manager, env injection, and lock down Actuator.

**7. Thread exhaustion on a slow downstream**
- *Cause:* Tomcat's thread-per-request model blocks on a slow API.
- *Fix:* timeouts on `RestClient`/`WebClient`, circuit breakers (Resilience4j), bulkheads, or virtual threads (`spring.threads.virtual.enabled=true`, Boot 3.2+ on Java 21).

---

## 16. Best Cases vs Worst Cases

### ✅ When Spring Boot shines
- REST APIs and microservices
- Enterprise CRUD apps with relational DBs
- Teams that want fast setup and a huge ecosystem (Security, Data, Cloud, Batch)
- Apps needing strong observability and production tooling

### ❌ When it's not ideal
- **Tiny serverless functions** where cold-start time and memory dominate (consider native images, or lighter frameworks like Micronaut or Quarkus)
- **Very memory-constrained** environments
- **Ultra-low-latency, startup-sensitive** workloads, unless you use native/AOT
- Projects where "magic" hurts: teams that don't understand auto-config end up debugging blind

### Good vs bad practices

| ✅ Do | ❌ Don't |
|---|---|
| Constructor injection | Field injection |
| `@ConfigurationProperties` | Scattered `@Value` everywhere |
| Externalize config and secrets | Hardcode credentials |
| DTOs at the API boundary | Expose JPA entities directly |
| Pagination and indexes | `findAll()` on huge tables |
| Short transactions | `@Transactional` around remote calls |
| Graceful shutdown + probes | Kill pods abruptly |
| Pin and upgrade Boot versions deliberately | Ignore security patches |

---

## 17. Common Pitfalls and Gotchas

1. **Circular dependencies** (A needs B, B needs A). Boot 2.6+ **forbids them by default**. Fix the design instead of enabling `spring.main.allow-circular-references`.
2. **N+1 query problem** in JPA: 1 query for the parent plus N for the children. Use `JOIN FETCH`, `@EntityGraph`, or batch fetching.
3. **`LazyInitializationException`**: accessing a lazy relation after the session closes. Fetch what you need inside the transaction, or use DTO projections.
4. **Open-in-view** (`spring.jpa.open-in-view`) is on by default and holds a DB connection for the whole request. Consider turning it off.
5. **Component scan missing your class**: wrong package structure.
6. **Multiple beans of the same type**: `NoUniqueBeanDefinitionException`. Use `@Primary` or `@Qualifier`.
7. **`@Transactional` self-invocation** (see section 13).
8. **Wrong profile active**: the app connects to the wrong DB. Always log the active profile at startup.
9. **Fat JAR size and slow startup** from unnecessary starters.
10. **Boot 3 migration**: needs **Java 17+**, and `javax.*` became **`jakarta.*`**.

---

## 18. Packaging: the Fat JAR

`mvn package` produces an **executable "fat" JAR** with your classes, all dependencies in `BOOT-INF/lib`, and a special launcher (`JarLauncher`) that sets up a classloader to run it.

Run it with `java -jar app.jar`.

For containers, use **layered JARs** (`BOOT-INF` layers) so Docker caches dependencies separately from your code, which makes rebuilds faster.

---

## 19. Rapid-Fire Interview Q&A

**Q: What is Spring Boot?**
An opinionated framework on top of Spring that provides auto-configuration, starters, and an embedded server to build production-ready apps quickly.

**Q: Spring vs Spring Boot?**
Spring is the core framework (IoC, AOP, MVC). Spring Boot removes the boilerplate: auto-config, starters, embedded server, Actuator. Boot doesn't replace Spring, it builds on it.

**Q: How does auto-configuration work?**
`@EnableAutoConfiguration` loads candidate config classes from `AutoConfiguration.imports`. Each is guarded by `@Conditional` annotations (class on classpath, missing bean, property set) and activates only when its conditions match. User-defined beans override them via `@ConditionalOnMissingBean`.

**Q: What does `@SpringBootApplication` do?**
`@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan`.

**Q: What happens when you call `SpringApplication.run()`?**
It prepares the environment, creates the ApplicationContext, refreshes it (loads bean definitions, instantiates singletons, starts the embedded server), then calls runners and publishes ready events.

**Q: What is the ApplicationContext vs BeanFactory?**
BeanFactory is the basic container with lazy bean creation. ApplicationContext extends it with events, i18n, AOP integration, and eager singleton creation. Boot always uses an ApplicationContext.

**Q: How do you override a default config?**
Define your own bean (auto-config backs off), set properties, or exclude the auto-configuration class.

**Q: How do you handle different environments?**
Profiles (`application-{profile}.yml`), activated via `SPRING_PROFILES_ACTIVE`, plus env vars for secrets.

**Q: `@Component` vs `@Bean`?**
`@Component` marks a class for scanning. `@Bean` is a method in a `@Configuration` class that returns an object, useful for third-party classes you can't annotate.

**Q: `@Controller` vs `@RestController`?**
`@RestController` = `@Controller` + `@ResponseBody`, so return values are written as JSON/XML instead of resolved as view names.

**Q: Singleton scope and thread safety?**
One shared instance, so beans must be stateless or thread-safe.

**Q: How does `@Transactional` work?**
Via an AOP proxy that opens the transaction before the method and commits or rolls back after. It fails on self-invocation and non-public methods.

**Q: How do you secure Actuator?**
Expose only needed endpoints, put them behind Spring Security, and use a separate management port or network rules.

**Q: How do you reduce startup time?**
Remove unused starters, lazy init, AOT/GraalVM native image, CDS, and avoid heavy work in `@PostConstruct`.

**Q: How do you deploy to GCP?**
Containerize (buildpacks or Dockerfile), push to Artifact Registry, deploy to Cloud Run or GKE. Use Secret Manager for secrets, Cloud SQL for the DB, Workload Identity for auth, and Actuator probes for health.

**Q: Embedded vs external server?**
Embedded means one self-contained artifact, simple deploys, and consistent environments. The trade-off is that the server version is tied to your app, so patching means redeploying.

**Q: Blocking (MVC) vs reactive (WebFlux)?**
MVC uses one thread per request, which is simple and fine for most apps. WebFlux uses an event loop (Netty) and suits high-concurrency, I/O-heavy workloads, but has a steeper learning curve. With virtual threads, MVC scales much better than before.

---

## 20. One-Page Cheat Sheet (last-minute revision)

```
START      java -jar → main() → SpringApplication.run()
RUN STEPS  Create app → Environment → Create context → Refresh → Runners → Ready
CONTEXT    IoC container; creates, wires, and manages beans
ANNOTATION @SpringBootApplication = @Configuration + @EnableAutoConfiguration + @ComponentScan
AUTO-CONF  imports file → @Conditional checks → beans registered → user beans win
STARTERS   curated dependency bundles, versions managed by parent/BOM
SERVER     Embedded Tomcat; DispatcherServlet routes requests
REQUEST    Filter → DispatcherServlet → HandlerMapping → Controller → Converter → Response
CONFIG     CLI > sysprops > env vars > profile file > application.yml > defaults
BEAN LIFE  construct → DI → Aware → @PostConstruct → proxy → ready → @PreDestroy
PROXIES    @Transactional/@Async/@Cacheable work via proxies (beware self-invocation)
ACTUATOR   health, metrics, probes; lock it down
GCP        Cloud Run/GKE + Secret Manager + Cloud SQL + Workload Identity + probes
PROD RULES graceful shutdown, tuned pool, container-aware JVM, no secrets in code
```

---

### 🎯 Final interview tip

When asked "how does X work?", answer in **three layers**:
1. **What** it is (one sentence)
2. **How** it works internally (the mechanism)
3. **Production impact**, with an example from GCP or your real project experience

That structure shows you understand it beyond the textbook. Good luck, you've got this! 🚀