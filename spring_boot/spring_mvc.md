# Spring MVC Explained: Controllers, Views, and Model (The Interview Cheatsheet)

*A Medium-style guide for a Technical Architect preparing for interviews. Simple enough for a 10-year-old, deep enough for a panel round.*

---

## 1. The 10-Year-Old Version (Start Here)

Imagine a **restaurant**:

| Restaurant | Spring MVC |
|---|---|
| **Customer** | Browser / Client |
| **Receptionist at the door** | `DispatcherServlet` (front controller) |
| **Menu lookup (who handles this order?)** | `HandlerMapping` |
| **Waiter who takes the order** | **Controller** |
| **Tray of ingredients the waiter carries** | **Model** |
| **Chef who plates the food nicely** | **View** (JSP, Thymeleaf, or JSON) |
| **Final dish served** | HTTP Response |

> **One line to remember:** *The Controller decides **what** to do, the Model carries **the data**, the View decides **how it looks**.*

---

## 2. What Is Spring MVC?

**Spring MVC** is the web framework inside Spring that follows the **Model-View-Controller** pattern, built on the **Servlet API**.

- **Model**: the data (POJOs, DTOs, `Map`, `Model` object)
- **View**: the presentation (HTML via Thymeleaf/JSP, or JSON/XML)
- **Controller**: the traffic cop that receives a request, calls the service layer, and picks the view

**Why does MVC exist?** Separation of concerns. Developers, UI designers, and testers can work independently, and each layer is replaceable.

---

## 3. The Request Lifecycle (Most Asked Interview Question)

```
Browser
   │  1. HTTP Request  (GET /orders/42)
   ▼
DispatcherServlet  (Front Controller)
   │  2. Ask HandlerMapping: "who handles /orders/42?"
   ▼
HandlerMapping  →  returns Controller method + Interceptors
   │  3. DispatcherServlet asks HandlerAdapter to invoke it
   ▼
HandlerAdapter → Controller method runs
   │     (argument resolvers bind @PathVariable, @RequestBody, etc.)
   │  4. Controller calls Service → Repository → DB
   │  5. Controller returns ModelAndView / view name + Model
   ▼
DispatcherServlet
   │  6. ViewResolver: "which actual view is 'orderDetails'?"
   ▼
View (Thymeleaf/JSP) renders HTML using Model data
   │  7. HTTP Response
   ▼
Browser
```

### The 7 Steps in Plain English

1. Request hits the **DispatcherServlet**, the single entry point.
2. It asks **HandlerMapping** which controller method matches the URL and HTTP method.
3. **HandlerAdapter** actually calls that method (handling argument binding).
4. Controller does its work through the **Service layer**.
5. Controller returns a **view name + model data**.
6. **ViewResolver** converts the logical name (`"orderDetails"`) into a real template (`/templates/orderDetails.html`).
7. **View** renders using the model, and the response goes out.

> **Interview gold:** For `@RestController`, steps 6 and 7 change. The `HttpMessageConverter` (e.g., Jackson) writes the return value straight into the response body. **No ViewResolver is involved.**

---

## 4. Controllers

### 4.1 What Is a Controller?
A class annotated with `@Controller` (or `@RestController`) that maps HTTP requests to Java methods.

```java
@Controller
@RequestMapping("/orders")
public class OrderController {

    private final OrderService orderService;

    public OrderController(OrderService orderService) {   // constructor injection
        this.orderService = orderService;
    }

    @GetMapping("/{id}")
    public String getOrder(@PathVariable Long id, Model model) {
        model.addAttribute("order", orderService.findById(id));
        return "orderDetails";   // logical view name
    }
}
```

### 4.2 `@Controller` vs `@RestController`

| | `@Controller` | `@RestController` |
|---|---|---|
| Returns | View name (HTML page) | Data (JSON/XML) |
| Equivalent to | n/a | `@Controller` + `@ResponseBody` on every method |
| Uses ViewResolver? | Yes | No |
| Typical use | Server-rendered pages | REST APIs, microservices |

### 4.3 Request Mapping Annotations

| Annotation | HTTP Method | Typical Use |
|---|---|---|
| `@GetMapping` | GET | Read data (safe, idempotent) |
| `@PostMapping` | POST | Create |
| `@PutMapping` | PUT | Full update (idempotent) |
| `@PatchMapping` | PATCH | Partial update |
| `@DeleteMapping` | DELETE | Delete (idempotent) |
| `@RequestMapping` | Any | Class-level base path, or custom config |

`@RequestMapping` supports `value`, `method`, `params`, `headers`, `consumes`, `produces`.

```java
@PostMapping(value = "/orders",
             consumes = "application/json",
             produces = "application/json")
```

### 4.4 Getting Data from the Request

| Annotation | Reads From | Example |
|---|---|---|
| `@PathVariable` | URL path | `/orders/42` → `id = 42` |
| `@RequestParam` | Query string / form field | `/orders?status=NEW` |
| `@RequestBody` | JSON body → Java object | POST payload |
| `@RequestHeader` | HTTP header | `Authorization` |
| `@CookieValue` | Cookie | `JSESSIONID` |
| `@ModelAttribute` | Form fields → object | HTML form binding |
| `@RequestPart` | Multipart file upload | File upload |
| `@SessionAttribute` | Session | Logged-in user |

```java
@GetMapping("/search")
public List<Order> search(@RequestParam(defaultValue = "NEW") String status,
                          @RequestParam(required = false) Integer page) { ... }
```

### 4.5 `@RequestParam` vs `@PathVariable` (Very Common Question)

- **`@PathVariable`**: identifies a **resource** (`/users/7`)
- **`@RequestParam`**: **filters / options** (`/users?role=admin&page=2`)

### 4.6 Return Types a Controller Can Use

| Return Type | Meaning |
|---|---|
| `String` | Logical view name |
| `ModelAndView` | View + model together |
| `void` | View name derived from URL (rarely used) |
| `ResponseEntity<T>` | Full control: status, headers, body |
| POJO / `List<T>` + `@ResponseBody` | Serialized to JSON |
| `"redirect:/path"` | HTTP 302 redirect |
| `"forward:/path"` | Server-side forward (same request) |
| `Callable`, `DeferredResult`, `CompletableFuture` | Async processing |

### 4.7 `ResponseEntity`: The Architect's Favorite

```java
@PostMapping("/orders")
public ResponseEntity<OrderDto> create(@Valid @RequestBody OrderRequest req) {
    OrderDto saved = orderService.create(req);
    URI location = URI.create("/orders/" + saved.getId());
    return ResponseEntity.created(location).body(saved);   // 201 + Location header
}
```

**Why?** Explicit status codes (`201`, `204`, `404`), custom headers, and clean semantics.

---

## 5. Model

### 5.1 What Is the Model?
The **data bag** the controller hands to the view. Think of it as a *tray* the waiter carries to the chef.

### 5.2 Ways to Pass Data to the View

| Type | When to Use |
|---|---|
| `Model` | Most common, simple `addAttribute` |
| `ModelMap` | Same idea, Map-like API |
| `Map<String,Object>` | Rare, works but less expressive |
| `ModelAndView` | When you want to return view + data as one object |
| `RedirectAttributes` | Passing data across a redirect (flash attributes) |

```java
// Option 1: Model
model.addAttribute("user", user);

// Option 2: ModelAndView
ModelAndView mv = new ModelAndView("profile");
mv.addObject("user", user);
return mv;
```

### 5.3 `@ModelAttribute`: Two Meanings

1. **On a method parameter**: binds form/query data to an object.
2. **On a method**: runs *before every handler* in that controller and adds shared data (e.g., dropdown lists).

```java
@ModelAttribute("countries")
public List<String> countries() { return List.of("India", "USA"); }
```

### 5.4 `@SessionAttributes`: Use with Care
Keeps model attributes in the HTTP session across requests (e.g., multi-step wizard).
**Production warning:** sessions don't scale horizontally without sticky sessions or Redis (Spring Session). Prefer stateless designs.

### 5.5 Flash Attributes (The Post-Redirect-Get Pattern)

**Problem:** User submits a form, refreshes, and gets a duplicate submission.
**Solution:** **Post → Redirect → Get (PRG)**.

```java
@PostMapping("/orders")
public String create(@Valid OrderForm form, RedirectAttributes ra) {
    orderService.create(form);
    ra.addFlashAttribute("msg", "Order created!");
    return "redirect:/orders";   // browser does a fresh GET
}
```

### 5.6 Validation and Binding

```java
public class OrderRequest {
    @NotBlank private String customerName;
    @Min(1) private int quantity;
}

@PostMapping("/orders")
public String create(@Valid @ModelAttribute OrderRequest req, BindingResult result) {
    if (result.hasErrors()) return "orderForm";   // redisplay form with errors
    ...
}
```

> **Rule:** `BindingResult` must come **immediately after** the `@Valid` parameter, or Spring throws the exception instead of capturing the errors.

For `@RequestBody`, validation failure throws `MethodArgumentNotValidException` (handled globally, see Section 8).

---

## 6. View

### 6.1 What Is a View?
The component that **renders** the response. In server-rendered apps, it's HTML. In REST apps, it's effectively the JSON converter.

### 6.2 View Technologies

| Technology | Notes |
|---|---|
| **Thymeleaf** | Modern default for Spring Boot, natural templates (valid HTML) |
| **JSP + JSTL** | Legacy; needs a WAR/servlet container, not great with executable JARs |
| **FreeMarker / Mustache** | Lightweight alternatives |
| **JSON (Jackson)** | For REST via `@ResponseBody` |
| **PDF / Excel views** | `AbstractPdfView`, `AbstractXlsxView` for reports |

### 6.3 ViewResolver

Translates a **logical name** → **physical view**.

```properties
# Classic JSP config
spring.mvc.view.prefix=/WEB-INF/views/
spring.mvc.view.suffix=.jsp
```
`"orderDetails"` → `/WEB-INF/views/orderDetails.jsp`

Common resolvers: `InternalResourceViewResolver`, `ThymeleafViewResolver`, `ContentNegotiatingViewResolver`.

### 6.4 Thymeleaf Snippet

```html
<h1 th:text="${order.customerName}">Customer</h1>
<tr th:each="item : ${order.items}">
    <td th:text="${item.name}"></td>
</tr>
```

### 6.5 Content Negotiation
Same endpoint, different output based on the `Accept` header (`application/json` vs `application/xml`). Handled by `HttpMessageConverter`s.

---

## 7. Core Components Cheat Table

| Component | Job | One-Liner |
|---|---|---|
| **DispatcherServlet** | Front controller | The receptionist routing everything |
| **HandlerMapping** | URL → controller method | The address book |
| **HandlerAdapter** | Invokes the handler | The one who actually calls the method |
| **ViewResolver** | Name → View | The chef's recipe lookup |
| **HttpMessageConverter** | Object ↔ JSON/XML | The translator |
| **HandlerInterceptor** | Pre/post processing | The security guard at the door |
| **HandlerExceptionResolver** | Exceptions → responses | The complaints desk |
| **LocaleResolver** | Determines the locale | i18n |
| **MultipartResolver** | File uploads | Handles uploaded files |

---

## 8. Exception Handling (Production Must-Know)

### 8.1 Local, per controller
```java
@ExceptionHandler(OrderNotFoundException.class)
public ResponseEntity<String> handle(OrderNotFoundException ex) {
    return ResponseEntity.status(404).body(ex.getMessage());
}
```

### 8.2 Global, for the whole app (preferred)
```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(OrderNotFoundException.class)
    public ResponseEntity<ApiError> notFound(OrderNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
                .body(new ApiError("ORDER_404", ex.getMessage()));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ApiError> invalid(MethodArgumentNotValidException ex) {
        // collect field errors
        return ResponseEntity.badRequest().body(new ApiError("VALIDATION", "Invalid input"));
    }

    @ExceptionHandler(Exception.class)   // safety net
    public ResponseEntity<ApiError> generic(Exception ex) {
        // log it! never leak stack traces to clients
        return ResponseEntity.status(500).body(new ApiError("INTERNAL", "Something went wrong"));
    }
}
```

### 8.3 `@ResponseStatus`
Annotate a custom exception with a fixed status code. It's simple, but less flexible than `@ControllerAdvice`.

---

## 9. Interceptors vs Filters (Classic Trap Question)

| | **Servlet Filter** | **HandlerInterceptor** |
|---|---|---|
| Belongs to | Servlet container | Spring MVC |
| Runs | Before `DispatcherServlet` | After `DispatcherServlet`, around the handler |
| Knows the controller method? | No | Yes (`HandlerMethod`) |
| Can access Spring beans easily? | Needs setup | Yes |
| Hooks | `doFilter` | `preHandle`, `postHandle`, `afterCompletion` |
| Use for | Security, CORS, request logging, compression | Auth checks per handler, timing, audit, locale |

**Order:** Filter → DispatcherServlet → Interceptor.preHandle → Controller → Interceptor.postHandle → View render → Interceptor.afterCompletion → Filter (return path).

---

## 10. When, Where, Why (The Decision Section)

### When to use Spring MVC
- Server-rendered web apps (admin portals, dashboards, internal tools)
- REST APIs and microservices (the most common production use)
- Apps that need a mature ecosystem: Security, Validation, Data, Actuator

### Where it sits in the architecture
```
Client → [API Gateway] → Controller → Service → Repository → DB
                         (Web layer)  (Business) (Persistence)
```
**Controllers should be thin.** They handle HTTP concerns only: parsing, validation, status codes, and delegation. **Business logic belongs in the service layer.**

### Why choose it
- Familiar, battle-tested, huge community
- Easy to test (`MockMvc`)
- Seamless with Spring Boot, Security, and Data
- Thread-per-request model is simple to reason about and debug

### When **not** to use it → Consider Spring WebFlux
- Very high concurrency with lots of **I/O-bound, non-blocking** calls
- Streaming / server-sent events / backpressure needs
- Your whole stack (DB drivers included) can be reactive

> **Architect's line:** *"MVC is blocking and thread-per-request. WebFlux is non-blocking and event-loop based. I choose WebFlux only when the entire call chain can be reactive; otherwise, I'd be paying the complexity cost for no gain."* (With Java 21+, virtual threads also let MVC scale well for I/O-heavy loads, which narrows WebFlux's advantage.)

---

## 11. Best Practices ✅ (Production Scenarios)

1. **Thin controllers, fat services.** No business logic or DB calls in controllers.
2. **Use DTOs**, never expose JPA entities directly (avoids leaking fields, lazy-loading issues, and tight coupling).
3. **Constructor injection** (immutable, testable) over field `@Autowired`.
4. **Validate at the boundary** with `@Valid` and Bean Validation.
5. **Global exception handling** via `@RestControllerAdvice` with a consistent error format (consider RFC 7807 `ProblemDetail`, supported in Spring 6+).
6. **Correct HTTP semantics:** `201` + `Location` for create, `204` for delete, `404` vs `400` vs `409` vs `422` used properly.
7. **Version your APIs** (`/api/v1/...` or header-based).
8. **Pagination everywhere** for list endpoints (`Pageable`), never return unbounded lists.
9. **Idempotency** for POST in payments or orders (idempotency key header).
10. **Use PRG** for HTML form submissions.
11. **Externalize configuration**, and secure endpoints with Spring Security.
12. **Add observability:** Actuator, correlation IDs via a filter, structured logs.
13. **Set timeouts** for downstream calls and the server (`server.tomcat.*`).
14. **Document APIs** with OpenAPI (springdoc).
15. **Test pyramid:** `@WebMvcTest` + `MockMvc` for controller slices, Testcontainers for integration.

---

## 12. Worst Cases ❌ (Anti-Patterns & Production Disasters)

| Anti-pattern | What Goes Wrong |
|---|---|
| **Business logic in controllers** | Untestable, duplicated, a maintenance nightmare |
| **Returning JPA entities directly** | Infinite JSON recursion, `LazyInitializationException`, leaks sensitive fields |
| **Open Session In View (OSIV) left on** (Spring Boot default `true`) | DB connections held during view rendering, connection pool starvation under load. Consider `spring.jpa.open-in-view=false` |
| **No global exception handler** | Stack traces leak to clients, inconsistent errors |
| **Unbounded list endpoints** | OOM, slow responses, DB hammering |
| **Blocking calls with no timeouts** | Thread pool exhaustion, cascading failure |
| **Storing large objects in HttpSession** | Memory bloat, can't scale horizontally |
| **Mass assignment** via `@ModelAttribute` binding to entities | Attackers set fields like `isAdmin=true`. Use DTOs or `@InitBinder` allow-lists |
| **Ignoring CSRF** on form-based apps | Cross-site request forgery |
| **`@RequestParam` without handling missing/invalid values** | `400` errors with ugly messages |
| **Singleton controller with mutable instance state** | Race conditions (controllers are **singletons**, so they must be stateless) |
| **Using `@Controller` but forgetting `@ResponseBody`** | Spring tries to resolve the JSON return as a view name, giving a 404 or template error |
| **Using `GET` for state changes** | Prefetchers, crawlers, and caches trigger side effects |

---

## 13. Rapid-Fire Interview Q&A

**Q1. What is the DispatcherServlet?**
The front controller, a single servlet that receives all requests and delegates to the right handler, then to the view. Spring Boot auto-registers it, mapped to `/`.

**Q2. Is a Spring controller a singleton? Is it thread-safe?**
Yes, singleton by default. It's thread-safe **only if stateless**. Never keep per-request state in fields.

**Q3. Difference between `@Controller` and `@RestController`?**
`@RestController` = `@Controller` + `@ResponseBody`. It returns data instead of view names.

**Q4. `Model` vs `ModelMap` vs `ModelAndView`?**
`Model` is an interface for adding attributes, `ModelMap` is a `LinkedHashMap`-based implementation, and `ModelAndView` bundles the view name and model into one return object.

**Q5. `forward:` vs `redirect:`?**
`forward` happens server-side, with the same request and the URL unchanged. `redirect` sends a 302, the browser makes a new request, and the URL changes. Use redirect for PRG.

**Q6. What does `@RequestBody` do internally?**
`HttpMessageConverter` (Jackson) deserializes the JSON body into a Java object.

**Q7. How do you handle file uploads?**
`MultipartFile` with `@RequestParam` or `@RequestPart`. Configure max size via `spring.servlet.multipart.max-file-size`.

**Q8. How do you handle CORS?**
`@CrossOrigin` on controllers, or globally via `WebMvcConfigurer.addCorsMappings`, or through Spring Security's CORS config. For production, use specific origins, **not** `*` with credentials.

**Q9. How do you map a request to different methods based on content type?**
Use `consumes` and `produces` in `@RequestMapping`.

**Q10. What is `@InitBinder`?**
Customizes data binding per controller, such as registering custom editors, date formats, or restricting allowed fields.

**Q11. How do you make a controller async?**
Return `Callable`, `DeferredResult`, or `CompletableFuture`. The servlet thread is freed while work runs elsewhere.

**Q12. Why constructor injection over field injection?**
Immutability, easier unit tests with no Spring context, fail-fast on missing dependencies, and no hidden dependencies.

**Q13. What happens if two methods map to the same URL and HTTP method?**
`IllegalStateException: Ambiguous mapping` at **startup**, which is good because it fails fast.

**Q14. How is a view name resolved if the controller returns `"home"`?**
The ViewResolver chain: Thymeleaf resolves `classpath:/templates/home.html` by default in Spring Boot.

**Q15. How would you version a REST API?**
URI (`/v1/`), header (`X-API-Version`), or media type (`application/vnd.company.v1+json`). URI is the simplest and most common.

**Q16. How do you test a controller?**
`@WebMvcTest(OrderController.class)` loads only the web layer. Mock the service with `@MockBean` (or `@MockitoBean` in newer Spring Boot versions) and use `MockMvc` to perform requests and assert.

**Q17. Spring MVC vs Spring Boot?**
MVC is the web framework. Boot is an opinionated setup layer (auto-config, embedded server, starters) that makes using MVC easy.

**Q18. How do you secure endpoints?**
Spring Security's filter chain (authentication/authorization), method security (`@PreAuthorize`), CSRF protection for forms, and JWT/OAuth2 for APIs.

**Q19. What is `HandlerMethodArgumentResolver`?**
Lets you create custom controller arguments (e.g., `@CurrentUser User user`) by plugging into argument binding.

**Q20. How do you handle slow requests in production?**
Timeouts, async processing, caching (`Cache-Control`, Spring Cache), pagination, circuit breakers (Resilience4j), and profiling via Actuator and tracing.

---

## 14. Architect-Level Talking Points 🏗️

- **Layering:** Controller (web/HTTP) → Service (business, transactions) → Repository (data). Keep boundaries strict.
- **API contract first:** Define OpenAPI, then generate or validate against it. DTOs decouple your API from your domain model.
- **Cross-cutting concerns** belong in filters, interceptors, or AOP, not copy-pasted across controllers: logging, correlation IDs, auth, rate limiting, auditing.
- **Statelessness** enables horizontal scaling. Externalize sessions (Redis) only when truly needed.
- **Failure handling:** timeouts, retries with backoff, circuit breakers, bulkheads, and a consistent error contract.
- **Observability:** metrics (Micrometer), distributed tracing (OpenTelemetry), and structured logs with trace IDs.
- **Backward compatibility:** additive changes, deprecation policy, and consumer-driven contract tests.
- **Security by default:** validate input, use DTOs, enforce HTTPS, use least privilege, and protect against OWASP Top 10 (injection, mass assignment, CSRF, broken auth).
- **Performance:** thread pool tuning (Tomcat `max-threads`), connection pool sizing (HikariCP), avoiding N+1 queries, and response compression and caching.

---

## 15. Quick-Revision Mind Map

```
Spring MVC
├── Flow: Request → DispatcherServlet → HandlerMapping → HandlerAdapter
│         → Controller → (Model) → ViewResolver → View → Response
├── Controller
│   ├── @Controller / @RestController
│   ├── @Get/Post/Put/Patch/DeleteMapping
│   ├── @PathVariable / @RequestParam / @RequestBody / @RequestHeader
│   └── ResponseEntity, redirect:, forward:
├── Model
│   ├── Model / ModelMap / ModelAndView
│   ├── @ModelAttribute, @SessionAttributes
│   └── RedirectAttributes (flash) → PRG pattern
├── View
│   ├── Thymeleaf / JSP / JSON
│   └── ViewResolver, content negotiation
├── Cross-cutting
│   ├── Filters vs Interceptors
│   ├── @ControllerAdvice / @ExceptionHandler
│   └── Validation (@Valid, BindingResult)
└── Production
    ├── Thin controllers, DTOs, pagination, timeouts
    ├── Disable OSIV, stateless design
    └── Security, observability, versioning
```

---

## 16. Final 60-Second Pitch (Say This When Asked "Explain Spring MVC")

> *"Spring MVC is a request-driven web framework built on the Servlet API. Every request first hits the **DispatcherServlet**, which uses **HandlerMapping** to find the right **controller** method. The controller handles HTTP concerns, delegates to the service layer, and returns either a **view name plus model** for server-rendered pages or a **response body** for REST APIs, in which case an **HttpMessageConverter** serializes it to JSON. For HTML, the **ViewResolver** maps the logical name to a template like Thymeleaf, which renders using the model data. In production I keep controllers thin, use DTOs, validate input, centralize error handling with **@RestControllerAdvice**, apply cross-cutting concerns through filters and interceptors, and design stateless services so they scale horizontally."*

---

**Good luck with the interviews, Shubham! 🚀** If you'd like, I can follow up with a companion cheatsheet on **Spring Boot internals (auto-configuration)**, **Spring Security**, or a **mock interview round** on this topic.