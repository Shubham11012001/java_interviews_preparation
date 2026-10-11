# Spring Cloud Microservices: The Interview Cheatsheet

*Service Discovery, Config Server, and the full toolbox, explained so a 10-year-old gets it and an architect can defend it.*

---

## 1. The Big Picture

### The 10-year-old version

Imagine a **school**. A monolith is **one giant classroom** where one teacher teaches everything: math, art, music, and PE. If the teacher is sick, the whole school stops.

Microservices are **many small classrooms**, each with its own specialist teacher. If the art teacher is sick, math still happens.

But now you have new problems:
- How does a student **find** which room the music class is in? → **Service Discovery**
- Who holds the **rulebook** that every classroom follows? → **Config Server**
- Who is the **front gate** that sends visitors to the right room? → **API Gateway**
- What if one classroom is on fire? → **Circuit Breaker**

**Spring Cloud is the toolbox that solves all these "many small classrooms" problems.**

### The architect version

Spring Cloud is a set of **opinionated patterns** (built on Spring Boot) for distributed-system concerns: service registration, centralized configuration, routing, resilience, tracing, and messaging.

> **Key point for interviews:** Spring Cloud does not invent new technology. It provides **abstractions and integrations** over tools like Eureka, Consul, Vault, Resilience4j, and Kubernetes.

---

## 2. Monolith vs Microservices

| Aspect | Monolith | Microservices |
|---|---|---|
| Deployment | One unit | Independent units |
| Scaling | Scale everything | Scale only what's needed |
| Tech stack | One | Polyglot possible |
| Failure | One bug can crash all | Isolated failures (if designed well) |
| Team ownership | Shared codebase | Team per service |
| Complexity | In the code | In the **network and operations** |
| Data | One shared DB | Database per service |

### When to use microservices
- Large teams that need independent release cycles
- Different parts need different scaling (e.g., search vs billing)
- Clear business domain boundaries (DDD bounded contexts)
- You have DevOps maturity (CI/CD, monitoring, containers)

### When NOT to use them
- Small team or early-stage startup
- Domain not well understood yet
- No CI/CD or observability in place
- You'd be splitting just to follow the trend

> **Interview gold:** *"Microservices trade code complexity for operational complexity. I'd start with a modular monolith unless there's a clear scaling or team-autonomy reason to split."*

---

## 3. The Spring Cloud Toolbox (Map)

| Problem | Spring Cloud Solution | Alternatives |
|---|---|---|
| Where is service X? | **Eureka** (Service Discovery) | Consul, Zookeeper, Kubernetes DNS |
| Where do configs live? | **Spring Cloud Config Server** | Consul KV, Vault, K8s ConfigMaps |
| Single entry point | **Spring Cloud Gateway** | Kong, NGINX, AWS API Gateway |
| Calling other services | **OpenFeign / RestClient / WebClient** | gRPC |
| Client-side load balancing | **Spring Cloud LoadBalancer** | Ribbon (deprecated) |
| Failure handling | **Resilience4j** | Hystrix (deprecated) |
| Distributed tracing | **Micrometer Tracing** (+ Zipkin/Tempo) | Sleuth (replaced) |
| Event messaging | **Spring Cloud Stream** | Raw Kafka/RabbitMQ clients |
| Secrets | **Spring Cloud Vault** | AWS Secrets Manager |

> **Warning:** Interviewers love testing whether you know what's **deprecated**. Say this: *"Hystrix, Ribbon, and Zuul 1 are in maintenance or removed. Today it's Resilience4j, Spring Cloud LoadBalancer, and Spring Cloud Gateway. Sleuth was replaced by Micrometer Tracing from Boot 3."*

---

## 4. Service Discovery

### 4.1 The problem

Service A needs to call Service B. In the cloud, B's IP address **changes constantly** (autoscaling, restarts, redeployments). Hardcoding `http://10.0.0.5:8080` is a disaster.

### 4.2 The 10-year-old version

Think of a **phone book**. When a new classroom opens, it writes its name and room number in the book (**registration**). When someone needs the music room, they look it up in the book (**discovery**). If a classroom closes, its entry is removed.

### 4.3 How Eureka works

```
┌──────────────┐   1. Register (name + IP + port)   ┌────────────────┐
│ Order Service│ ─────────────────────────────────► │ Eureka Server  │
└──────────────┘   2. Heartbeat every 30s           │ (Registry)     │
                                                    └────────────────┘
┌──────────────┐   3. "Where is PAYMENT?"                  ▲
│ Order Service│ ──────────────────────────────────────────┘
└──────────────┘   4. Gets list: [10.0.0.5, 10.0.0.9]
        │
        └──► 5. Calls one of them (client-side load balancing)
```

**Key concepts:**
- **Eureka Server:** the registry (the phone book)
- **Eureka Client:** any service that registers itself and/or looks up others
- **Heartbeat:** "I'm still alive" signal (default every **30 seconds**)
- **Lease expiration:** if no heartbeat for **90 seconds**, the instance is removed
- **Registry cache:** clients cache the registry locally and refresh periodically (default 30s)

### 4.4 Minimal setup

**Server:**
```java
@SpringBootApplication
@EnableEurekaServer
public class DiscoveryServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(DiscoveryServerApplication.class, args);
    }
}
```
```yaml
server:
  port: 8761
eureka:
  client:
    register-with-eureka: false   # server shouldn't register itself
    fetch-registry: false
```

**Client:**
```yaml
spring:
  application:
    name: order-service      # THIS name is what others use to find it
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
```

> Since recent Spring Cloud versions, `@EnableEurekaClient` / `@EnableDiscoveryClient` is **optional**. Having the dependency on the classpath is enough.

**Calling by name (not IP):**
```java
@FeignClient(name = "payment-service")
public interface PaymentClient {
    @GetMapping("/payments/{id}")
    Payment getPayment(@PathVariable Long id);
}
```

### 4.5 Client-side vs Server-side discovery

| | Client-side (Eureka) | Server-side (Kubernetes, AWS ALB) |
|---|---|---|
| Who picks the instance? | The calling service | A load balancer/proxy |
| Pros | No extra hop, smart routing | Simple clients, language-agnostic |
| Cons | Client needs the library | Extra hop, LB is a component to manage |

> **Architect answer on Kubernetes:** *"If we run on Kubernetes, Eureka is largely redundant. K8s Services plus DNS already give discovery and load balancing. I'd use Eureka mainly for VM-based or hybrid environments, or when migrating gradually."*

### 4.6 Eureka and the CAP theorem

Eureka chooses **AP** (Availability + Partition tolerance). It would rather give you **slightly stale data** than no data. Consul and Zookeeper lean toward **CP**.

**Self-preservation mode:** If Eureka sees too many missed heartbeats at once (e.g., a network blip), it **stops evicting instances**, assuming the network is the problem rather than the services.

- ✅ **Best case:** Protects you from mass de-registration during network glitches.
- ❌ **Worst case:** Dead instances stay in the registry, and clients get routed to them.
- **Mitigation:** Retries and circuit breakers on the client side.

### 4.7 Production scenarios

**Scenario 1: Rolling deployment sends traffic to a dying instance**
- *Cause:* Instance is shutting down, but registry still lists it (up to 90s + cache delay).
- *Fix:* Graceful shutdown, de-register first, then wait before stopping, and add client retries.

**Scenario 2: Eureka server goes down**
- *Impact:* Clients keep working using their **local cached registry**. New instances can't register, and the cache gets stale.
- *Fix:* Run **multiple Eureka nodes** (peer awareness), where each replicates to the others.

**Scenario 3: Zombie instance**
- *Cause:* App is alive (heartbeat OK) but broken (DB connection dead).
- *Fix:* Integrate health checks (`eureka.client.healthcheck.enabled=true`) so Actuator health drives registry status.

### 4.8 Best and worst practices

✅ Run 2 to 3 Eureka nodes across availability zones  
✅ Tune heartbeat and lease settings to your needs  
✅ Use health-check integration  
✅ Always combine with retry plus circuit breaker  

❌ Single Eureka node in production  
❌ Using IPs instead of service names  
❌ Assuming discovery data is instantly accurate  

---

## 5. Config Server

### 5.1 The problem

You have 30 microservices across dev, QA, and prod. Each has `application.yml` with DB URLs, feature flags, and timeouts. Changing one value means **rebuilding and redeploying** 30 services. And secrets sit in your Git repo.

### 5.2 The 10-year-old version

Instead of every classroom having its own copy of the rulebook (which gets outdated), there is **one central rulebook office**. Classrooms go there at the start of the day and ask, *"What are today's rules for me?"*

### 5.3 How it works

```
┌─────────────┐   1. "I'm order-service, profile=prod"   ┌───────────────┐   ┌──────────┐
│ Order Svc   │ ───────────────────────────────────────► │ Config Server │──►│ Git Repo │
└─────────────┘   2. Gets merged properties              └───────────────┘   └──────────┘
```

The Config Server reads from a backend, most commonly **Git** (also supports filesystem, Vault, JDBC, S3).

**File naming convention in the Git repo:**
```
application.yml                 # shared by ALL services
order-service.yml               # for order-service, all profiles
order-service-prod.yml          # for order-service in prod
```

**Precedence (highest wins):** `order-service-prod.yml` > `order-service.yml` > `application-prod.yml` > `application.yml`

### 5.4 Minimal setup

**Config Server:**
```java
@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication { ... }
```
```yaml
server:
  port: 8888
spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/your-org/config-repo
          default-label: main
          clone-on-start: true
```

**Config Client (bootstrap approach was replaced):**
```yaml
spring:
  application:
    name: order-service
  profiles:
    active: prod
  config:
    import: "optional:configserver:http://localhost:8888"
```

> **Interview trap:** Older tutorials use `bootstrap.yml`. In Spring Boot 2.4+, the recommended way is **`spring.config.import`**. Mention this and you look current.

### 5.5 Refreshing config without restart

```java
@RestController
@RefreshScope
public class FeatureController {
    @Value("${feature.new-checkout:false}")
    private boolean newCheckout;
}
```

Then trigger: `POST /actuator/refresh` on that instance.

**Problem:** With 50 instances, you'd call 50 endpoints.  
**Solution:** **Spring Cloud Bus** (Kafka/RabbitMQ). Call `POST /actuator/busrefresh` **once** and it broadcasts to all instances. Combine it with a **Git webhook** for full automation.

**Caveats of `@RefreshScope`:**
- Only refreshes beans annotated with it (and `@ConfigurationProperties` beans)
- Does **not** re-create DB connection pools or already-started thread pools smoothly
- Some changes still need a restart

### 5.6 Secrets and encryption

Never store plain-text passwords in Git. Options:
1. **Config Server encryption:** store `{cipher}AQB...` values, and the server decrypts (symmetric key or RSA keystore).
2. **HashiCorp Vault** backend (best for serious setups).
3. Cloud secret managers (AWS Secrets Manager, GCP Secret Manager).

> **Architect answer:** *"Git for non-sensitive config with versioning and audit trail. Secrets go to Vault or a cloud secret manager, never plain in Git."*

### 5.7 Production scenarios

**Scenario 1: Config Server is down at startup**
- *Impact:* Services configured as mandatory **fail to start**.
- *Fixes:* `fail-fast: true` with retry, run multiple Config Server instances behind a load balancer, or use `optional:` with sensible local defaults.

**Scenario 2: Bad config pushed to prod**
- *Impact:* A typo in a timeout value affects every service on refresh.
- *Fixes:* Pull-request reviews on the config repo, separate branches or repos per environment, and Git revert is your instant rollback (a big benefit of Git as backend).

**Scenario 3: Git is unreachable**
- *Fix:* Config Server clones to local disk, so it serves the **last known good copy**. Set `clone-on-start: true`.

### 5.8 Config Server vs alternatives

| Option | Best for |
|---|---|
| Spring Cloud Config | Spring-heavy shops, Git-based audit trail |
| Kubernetes ConfigMap/Secret | K8s-native deployments |
| Consul KV | Already using Consul |
| AWS Parameter Store / AppConfig | AWS-centric |

> **Honest take:** On Kubernetes, ConfigMaps plus Secrets often replace Config Server. Config Server still shines for **multi-environment, Git-versioned, Spring-centric** setups.

### 5.9 Best and worst practices

✅ Separate config repo, with PR reviews  
✅ Encrypt or externalize secrets  
✅ Secure the Config Server itself (Spring Security, mTLS)  
✅ HA: at least 2 instances  
✅ Keep sensible defaults inside the app  

❌ Secrets in plain text in Git  
❌ Open, unauthenticated Config Server endpoint  
❌ Putting every tiny property there (keep it for what changes per environment)  
❌ One giant shared `application.yml` that everybody edits blindly  

---

## 6. API Gateway (Spring Cloud Gateway)

### 6.1 The 10-year-old version

The **front gate of the school**. Visitors don't wander the halls; the guard checks their ID, tells them which classroom to go to, and stops troublemakers.

### 6.2 What it does

- **Routing:** `/orders/**` goes to order-service
- **Authentication and authorization** (JWT validation at the edge)
- **Rate limiting** (e.g., Redis-based)
- **Load balancing** (via `lb://service-name`)
- **Cross-cutting concerns:** logging, CORS, header manipulation, retries, circuit breaking

### 6.3 Core concepts

- **Route:** ID + destination URI + predicates + filters
- **Predicate:** "when" to match (path, header, method, time)
- **Filter:** "what" to do (add header, rewrite path, rate limit)

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: order-route
          uri: lb://order-service        # discovered via Eureka
          predicates:
            - Path=/api/orders/**
          filters:
            - StripPrefix=1
            - name: CircuitBreaker
              args:
                name: orderCB
                fallbackUri: forward:/fallback/orders
```

> Spring Cloud Gateway is built on **Spring WebFlux (reactive, non-blocking, Netty)**. Don't put blocking calls in custom filters.

### 6.4 Pitfalls

- ❌ Putting **business logic** in the gateway (it becomes a bottleneck and a mini-monolith)
- ❌ Single gateway instance (single point of failure)
- ✅ Keep it thin: security, routing, throttling
- ✅ Consider **BFF (Backend For Frontend)** pattern: separate gateways for web and mobile

---

## 7. Inter-Service Communication

| Style | Tool | Use when |
|---|---|---|
| Sync REST | OpenFeign, RestClient, WebClient | Need an immediate response |
| Async messaging | Spring Cloud Stream (Kafka/RabbitMQ) | Decoupling, events, high volume |
| gRPC | Separate library | Low latency, internal service-to-service |

**OpenFeign** = declare an interface and Spring writes the HTTP client:

```java
@FeignClient(name = "inventory-service", fallback = InventoryFallback.class)
public interface InventoryClient {
    @GetMapping("/stock/{sku}")
    StockResponse check(@PathVariable String sku);
}
```

> **Architect principle:** *"Prefer async events for cross-domain flows to reduce temporal coupling. Use sync calls only when the caller truly needs the answer now."*

**Worst case of too much sync calling:** A chain A → B → C → D. If D is slow, **everything** is slow. That's the **distributed monolith** anti-pattern.

---

## 8. Resilience (Circuit Breaker, Retry, and Friends)

### 8.1 The 10-year-old version

A **circuit breaker** is like the fuse in your home. If a toaster keeps short-circuiting, the fuse **trips** and stops the electricity so the whole house doesn't burn. After a while, you test it again.

### 8.2 States

```
   CLOSED ──(failures > threshold)──► OPEN ──(wait time)──► HALF-OPEN
      ▲                                                          │
      └──────────(test calls succeed)────────────────────────────┘
                 (test calls fail → back to OPEN)
```

- **CLOSED:** normal, calls pass through
- **OPEN:** calls **fail immediately** (or go to fallback), and the struggling service gets breathing room
- **HALF-OPEN:** lets a few trial calls through to check recovery

### 8.3 Resilience4j patterns

| Pattern | Purpose |
|---|---|
| **Circuit Breaker** | Stop calling a failing service |
| **Retry** | Retry transient failures (with backoff and jitter) |
| **Rate Limiter** | Limit calls per period |
| **Bulkhead** | Isolate resources so one slow dependency can't eat all threads |
| **Time Limiter** | Timeout on async calls |

```java
@CircuitBreaker(name = "inventory", fallbackMethod = "fallbackStock")
@Retry(name = "inventory")
public StockResponse check(String sku) { ... }

public StockResponse fallbackStock(String sku, Throwable t) {
    return StockResponse.unknown();   // graceful degradation
}
```

### 8.4 Production scenarios

**Retry storm:** Service B is struggling, and 10 callers each retry 3 times, which means **30x load** on B and it dies completely.
- *Fix:* Exponential backoff plus jitter, a retry budget, and a circuit breaker **in front of** retries.

**Cascading failure:** Slow payment service makes order threads pile up and the whole order service dies.
- *Fix:* Timeouts, bulkheads, and a circuit breaker.

> **Rule of thumb:** **Always set timeouts.** A missing timeout is the most common root cause of outages in microservices.

### 8.5 Idempotency warning

Retrying a **non-idempotent** call (like "charge card") can charge twice. Only retry idempotent operations, or use **idempotency keys**.

---

## 9. Distributed Tracing and Observability

**The problem:** One user request touches 8 services. Where did it slow down?

- **Trace ID:** one ID for the whole request journey
- **Span ID:** one ID per hop
- Propagated via HTTP headers (e.g., W3C `traceparent`)

**Modern stack:** Micrometer Tracing + OpenTelemetry → Zipkin / Jaeger / Tempo.  
**Sleuth** was replaced by Micrometer Tracing in Spring Boot 3.

**Three pillars of observability:**
1. **Logs** (include trace ID in every log line)
2. **Metrics** (Micrometer → Prometheus → Grafana)
3. **Traces**

---

## 10. Data Management in Microservices

- **Database per service** (no shared tables, because shared DB equals hidden coupling)
- No distributed ACID transactions across services

### Saga pattern (for cross-service business transactions)

Example: place order → reserve stock → charge payment. If payment fails, **compensate** (release stock).

| Type | How | Pros | Cons |
|---|---|---|---|
| **Choreography** | Services react to each other's events | Simple, decoupled | Hard to track the flow |
| **Orchestration** | A central coordinator tells each step | Clear flow, easy to monitor | Orchestrator is an extra component |

### Other key patterns
- **Outbox pattern:** write the event to a DB table in the same transaction as the data, then publish reliably (avoids "DB saved but event lost")
- **CQRS and Event Sourcing:** for complex read/write separation (use only if truly needed)

---

## 11. Security Basics

- **Edge:** Gateway validates JWT (OAuth2 / OIDC with Keycloak, Okta, and so on)
- **Service-to-service:** propagate the token, or use mTLS / service mesh
- **Never** trust the network just because it's "internal" (zero-trust)
- Secure Actuator endpoints, Eureka dashboard, and Config Server

---

## 12. Deployment and Spring Cloud on Kubernetes

| Concern | Spring Cloud way | Kubernetes-native way |
|---|---|---|
| Discovery | Eureka | K8s Service + DNS |
| Config | Config Server | ConfigMap / Secret |
| Load balancing | Spring Cloud LoadBalancer | kube-proxy / Ingress / mesh |
| Gateway | Spring Cloud Gateway | Ingress / Gateway API / Istio |
| Resilience | Resilience4j | Istio retries/timeouts |

> **Great interview answer:** *"On Kubernetes I use the platform for discovery and config, and keep Spring Cloud for what it does best: Gateway, Resilience4j, and tracing integration. There's also Spring Cloud Kubernetes for bridging."*

---

## 13. Putting It All Together

```
                    ┌─────────┐
   Client ────────► │ Gateway │ ◄──── Config Server ◄── Git/Vault
                    └────┬────┘
                         │ (lb://)
                  ┌──────▼───────┐
                  │   Eureka     │ (Registry)
                  └──┬────────┬──┘
            ┌────────▼──┐  ┌──▼────────┐
            │Order Svc  │  │Payment Svc│
            │+Resilience4j│ │           │
            └─────┬─────┘  └─────┬─────┘
                  │              │
              [Order DB]    [Payment DB]
                  └────► Kafka (events) ◄────┘
        Tracing: Micrometer → Zipkin | Metrics: Prometheus → Grafana
```

---

## 14. Rapid-Fire Interview Q&A

**Q: What is Spring Cloud?**  
A set of tools on top of Spring Boot for building distributed systems: discovery, config, gateway, resilience, tracing, messaging.

**Q: Why Service Discovery?**  
Instances have dynamic IPs. Discovery lets services find each other by **name** instead of address.

**Q: Eureka heartbeat and eviction defaults?**  
30s heartbeat, 90s lease expiry, 30s registry cache refresh.

**Q: What is Eureka self-preservation?**  
Safety mode that stops evicting instances during suspected network issues. It favors availability over accuracy.

**Q: Eureka is AP or CP?**  
AP.

**Q: What if Eureka goes down?**  
Clients use their cached registry. Existing calls continue, but new registrations and updates stop.

**Q: How does Config Server get its data?**  
From a backend: Git (most common), filesystem, Vault, JDBC, S3.

**Q: How do you refresh config without restart?**  
`@RefreshScope` plus `/actuator/refresh`, or Spring Cloud Bus (`/actuator/busrefresh`) for broadcasting to all instances.

**Q: How do you protect secrets in config?**  
`{cipher}` encryption, Vault, or cloud secret managers. Never plain text in Git.

**Q: Config Server vs Kubernetes ConfigMap?**  
ConfigMap is simpler on K8s. Config Server gives Git versioning, per-app/profile layering, and dynamic refresh for Spring apps.

**Q: What replaced Ribbon, Hystrix, Zuul, Sleuth?**  
Spring Cloud LoadBalancer, Resilience4j, Spring Cloud Gateway, Micrometer Tracing.

**Q: Circuit breaker vs retry?**  
Retry handles **transient** blips. Circuit breaker handles **sustained** failure by stopping calls. Use both, with the breaker wrapping the retries.

**Q: Gateway vs Load Balancer?**  
An LB just distributes traffic. A gateway understands **application-level** concerns (auth, routing rules, rate limits, transformations).

**Q: How do you handle transactions across services?**  
Saga pattern (choreography or orchestration) plus the Outbox pattern. No 2PC in most modern designs.

**Q: How do you version APIs?**  
URI (`/v1/`), header, or content negotiation, with backward-compatible changes and consumer-driven contract tests (Pact / Spring Cloud Contract).

**Q: How do you debug a slow request across services?**  
Take the trace ID from the logs, open it in Zipkin/Jaeger, find the slow span, and check that service's metrics and logs.

**Q: Biggest microservices mistakes?**  
Splitting too early, shared databases, chatty synchronous calls, missing timeouts, no observability, and ignoring data consistency.

---

## 15. Common Pitfalls Checklist (Say these in interviews)

- ❌ Distributed monolith (services too tightly coupled)
- ❌ No timeouts, retries, or fallbacks
- ❌ Shared database across services
- ❌ Config Server or Eureka as a single point of failure
- ❌ Secrets in Git
- ❌ Ignoring idempotency with retries
- ❌ No correlation IDs in logs
- ❌ Gateway with business logic
- ❌ Adopting microservices without CI/CD and monitoring

---

## 16. Architect-Level Talking Points

1. **"Start with the problem, not the pattern."** Justify microservices by team and scaling needs.
2. **"Design for failure."** Everything on the network will eventually fail, so plan timeouts, fallbacks, and graceful degradation.
3. **"Eventual consistency is a business conversation, not just a technical one."**
4. **"Observability is not optional."** Logs, metrics, and traces from day one.
5. **"Platform choice shapes tool choice."** On K8s, lean on native features; on VMs, Spring Cloud shines.
6. **"Automate config and deployment."** GitOps, pipelines, and environment parity.
7. **"Own the migration path."** Strangler Fig pattern for moving from a monolith step by step.

---

## 17. Last-Minute Memory Hooks 🧠

| Concept | One-liner |
|---|---|
| Eureka | The **phone book** |
| Config Server | The **central rulebook** |
| Gateway | The **front gate guard** |
| Circuit Breaker | The **fuse** |
| Bulkhead | **Watertight ship compartments** |
| Saga | **Undo button** across services |
| Tracing | **Package tracking number** |
| Feign | **Auto-written phone dialer** |

---

**Good luck with your interviews. You've got this.** 🚀

*Tip: If you want, I can follow up with a mock interview round, a scenario-based system design drill, or a Kubernetes-vs-Spring-Cloud deep dive.*