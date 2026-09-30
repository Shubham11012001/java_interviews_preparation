# Java Backend Interview Questions

---

## 1. Core Java & JVM

1. Explain JVM architecture and how Java code is executed internally.
2. How does Java Garbage Collector work? Explain different GC algorithms (G1, CMS, ZGC).
3. What are strong, soft, weak, and phantom references in Java?
4. How do you analyze and fix a memory leak in a Java application (heap dump analysis)?
5. Explain the difference between Stack memory and Heap memory.
6. What happens internally when you create an object using the `new` keyword?
7. How does String pooling work? What is the memory-level difference between `new String("abc")` and `"abc"`?
8. How does `intern()` work with the String pool?
9. How would you implement immutability in Java? Why is String immutable?
10. How is memory allocated for `static`, `final`, and `static final` variables in the JVM?

---

## 2. OOP & Design Principles

11. Explain SOLID principles with real-world examples.
12. Abstraction vs Encapsulation — real-world example.
13. Describe a scenario where using inheritance instead of composition broke the design.
14. Give a real production example where the Liskov Substitution Principle was violated.
15. How does the diamond problem get resolved in Java through default methods in interfaces?
16. Why aren't multiple abstract methods allowed in functional interfaces?
17. What are the correct use cases for the `Optional` class, and how is it commonly misused?
18. How does the Reflection API work in Java, and what are its performance trade-offs?

---

## 3. Collections & Data Structures

19. How does `HashMap` work internally in Java 8+? What changed after Java 8 (treeify threshold, red-black tree)?
20. What happens when two keys have the same `hashCode()`?
21. What is the performance impact during `HashMap` resize/rehashing, and how can it be avoided?
22. `ConcurrentHashMap` vs `Collections.synchronizedMap()` — how does `ConcurrentHashMap` handle locking internally (segment locking vs CAS)?
23. How does `CopyOnWriteArrayList` work, and what is its memory overhead?
24. When would you use `WeakHashMap` and `IdentityHashMap`?
25. When would you choose `ArrayList`, `LinkedList`, `HashSet`, or `TreeSet`?
26. Explain fail-fast vs fail-safe iterators.
27. Custom sorting using `Comparator` and `Comparable`.

---

## 4. Multithreading & Concurrency

28. Explain the Java Memory Model (JMM) and the happens-before relationship.
29. Why doesn't `volatile` make `i++` thread-safe? What does `volatile` actually guarantee?
30. Can two `volatile` variables still produce an inconsistent state?
31. Difference between `synchronized`, `Lock`, and `ReentrantLock`.
32. Thread lifecycle — `Thread.start()` vs `run()`.
33. When would you choose `AtomicInteger` over `synchronized`?
34. Why does `AtomicInteger` not automatically make a multi-step business operation thread-safe?
35. What is the difference between atomicity of a variable vs atomicity of an operation?
36. How does `ExecutorService` work internally?
37. Difference between `Callable` and `Runnable`.
38. Explain the difference between `ForkJoinPool` and a normal `ExecutorService` (work-stealing).
39. How does `CompletableFuture` execute asynchronous tasks? Explain the practical difference between `thenApply()`, `thenCompose()`, and `thenCombine()`.
40. How does `ThreadLocal` create a memory leak, and how do you avoid it? Can `ThreadLocal` create stale data when threads are reused by a thread pool?
41. What happens when `this` escapes from a constructor? Why is it dangerous?
42. Can a private field still cause a thread-safety issue?
43. Can a class be thread-safe even if it has mutable state?
44. Why can `final` fields improve thread safety, but `final` does not make an object immutable?
45. Can an immutable object contain a mutable `Date`, `List`, or array and still remain truly immutable?
46. Why is double-checked locking broken without `volatile`?
47. How would you detect a deadlock in a production Java application?
48. Your application suddenly throws `OutOfMemoryError`. What would you check first?
49. Your Java application suddenly has thousands of threads. What could be causing it?
50. Explain the use case for `Semaphore` in the context of connection pooling.
51. Can `ConcurrentHashMap` guarantee thread safety for a check-then-act compound operation like `containsKey()` + `put()`? Why is `computeIfAbsent()` safer?
52. What happens if the function passed to `computeIfAbsent()` modifies the same map?
53. Can `Collections.synchronizedList()` prevent all concurrent access problems? Why must you manually synchronize while iterating over it?
54. Why can caching an object in a `static` field introduce a visibility problem?
55. How can a thread-safe singleton still produce incorrect results under concurrent requests?

---

## 5. Java 8+ & Modern Java

56. Explain Stream API internal processing.
57. Difference between `map()`, `flatMap()`, and `filter()`.
58. How does parallel stream work internally?
59. What are functional interfaces and method references?
60. What performance issues can autoboxing/unboxing cause in production?
61. How does the `var` keyword (Java 10) resolve type inference internally at compile time?
62. How does exception chaining (`cause`) help with debugging in production?

---

## 6. Spring Framework & Spring Boot

63. What is Dependency Injection (DI) and Inversion of Control (IoC)?
64. Constructor Injection vs Field Injection — which do you prefer and why?
65. What is a Spring Bean? Explain the complete Spring Bean lifecycle (up to `BeanPostProcessor`).
66. What are different Bean Scopes? (Singleton, Prototype, Request, etc.)
67. Difference between `@Component`, `@Service`, `@Repository`, and `@Controller`.
68. `@RestController` vs `@Controller`.
69. How does `@Autowired` work internally?
70. What happens when multiple beans of the same type are available? `@Primary` vs `@Qualifier`.
71. What are `@Configuration` and `@Bean`?
72. What is Component Scanning?
73. `ApplicationContext` vs `BeanFactory`.
74. How does Spring resolve circular dependencies, and when does it fail?
75. How does Spring Boot Auto-Configuration work? What is `@EnableAutoConfiguration`?
76. What are Spring Boot Starter Dependencies?
77. How do Profiles work? (`@Profile`, `application.properties`, `application.yml`)
78. How do you externalize configuration across different environments?
79. How does Spring perform DI using reflection?
80. `CommandLineRunner` vs `ApplicationRunner`.
81. How does `@Transactional` work internally (Propagation + Isolation)?
82. If `placeOrder()` internally calls `makePayment()` within the same class and both are `@Transactional`, how many transactions are created and why? (self-invocation / proxy bypass)
83. What is transaction propagation?
84. `@Autowired` vs `@Inject`.
85. How do you handle exceptions globally in Spring Boot?
86. Spring Boot Actuator — most used endpoints and how to use it for production monitoring.
87. How to create custom annotations in Spring.

---

## 7. Spring Security

88. What is Spring Security and how does it work?
89. Authentication vs Authorization.
90. Explain the complete Spring Security Filter Chain and how a request is processed.
91. What is `SecurityFilterChain`?
92. How do you configure Spring Security in Spring Boot 3?
93. `UserDetails` vs `UserDetailsService`.
94. How does `PasswordEncoder` work? Why is BCrypt used for password hashing?
95. How does JWT authentication work? Who generates the JWT token? Does Spring Security generate JWT automatically?
96. Access Token vs Refresh Token — why does the Refresh Token have a longer expiry?
97. What happens when the Access Token expires?
98. Filter vs Interceptor — why does Spring Security use Filters instead of Interceptors?
99. What is CSRF? What is CORS?
100. `hasRole()` vs `hasAuthority()`.
101. What is method-level security? How does `@PreAuthorize` work?
102. What are `Authentication` and `SecurityContext`? How does session management work?
103. OAuth2 vs OpenID Connect.
104. How would you secure a microservices architecture using JWT / OAuth2?
105. Key Spring Security components: `AuthenticationManager`, `AuthenticationProvider`, `SecurityContextHolder`, `OncePerRequestFilter`.

---

## 8. JPA, Hibernate & Databases

106. JPA vs Hibernate.
107. JPQL vs Native Query — when and why would you use Native Queries?
108. How do you perform joins in JPA?
109. What is the Hibernate N+1 Query Problem, and how do you detect and fix it in production?
110. What is Auditing in JPA?
111. Why is an index not being used even when it exists?
112. How can an index make writes slower?
113. What causes database deadlocks, and how would you handle concurrent updates?
114. Optimistic vs Pessimistic Locking — when would you use each?
115. Offset vs Cursor/Keyset Pagination.
116. How would you handle millions of records in a database?
117. How would you perform a large database migration with minimal downtime?

---

## 9. Microservices Architecture

118. What is service discovery? (Eureka / Consul)
119. REST vs Kafka vs gRPC — which to use when?
120. Common microservices challenges and solutions.
121. What is distributed tracing? (Zipkin, Sleuth, OpenTelemetry)
122. Circuit Breaker, Retry, and Rate Limiting.
123. Inter-service transactions — Saga Pattern (Choreography vs Orchestration) vs 2PC.
124. Role of an API Gateway.
125. Spring Cloud Config Server.

---

## 10. Kafka

126. What is a Consumer Group? How are partitions assigned to consumers?
127. What happens if there are more consumers than partitions?
128. What happens when a consumer in a Consumer Group fails? What is Kafka Rebalancing?
129. What happens if a Kafka consumer crashes after processing a message but before committing the offset?
130. How do you handle duplicate Kafka messages (idempotency)?
131. What causes Consumer Lag?
132. How do you preserve message ordering?
133. How do you handle failed events? Retry Topic vs Dead Letter Queue (DLQ).
134. How do you safely replay Kafka messages?

---

## 11. Caching & Redis

135. How do you implement caching using Redis?
136. Cache Stampede / Thundering Herd Problem.

---

## 12. System Design & Distributed Systems

137. How would you design a high-volume payment processing system?
138. How do you handle millions of database requests?
139. What is eventual consistency?
140. CAP Theorem.
141. Distributed Locks and Transactions.
142. Idempotency in Distributed Systems.
143. Horizontal vs Vertical Scaling.
144. Load Balancing.
145. Stateless vs Stateful Services.
146. Database Sharding & Partitioning.
147. Connection Pooling.
148. Synchronous vs Asynchronous Processing.
149. How do you design a scalable REST API?
150. How do you design an idempotent API?
151. How would you design pagination for millions of records?
152. How do you handle API versioning?
153. How do you handle timeouts and retries, and how do you avoid retry storms?
154. How would your system handle 10x traffic tomorrow?

---

## 13. Production Debugging & Performance

155. How would you debug a slow API in production?
156. CPU usage suddenly reaches 90% — how would you investigate?
157. How would you analyze Thread Dumps and Heap Dumps?
158. How would you identify slow database queries?
159. What happens when the DB connection pool is exhausted?
160. How do you troubleshoot frequent long GC pauses?
161. How do you monitor and debug production issues?
162. What happens to in-flight requests when Kubernetes sends `SIGTERM`? How do you handle graceful shutdown?

---

## 14. Coding & Problem Solving

163. Find the second highest number in an array.
164. Reverse a Linked List (iterative + recursive).
165. Find the first non-repeated character in a string.
166. Implement an LRU Cache.
167. Find a pair in an array whose sum equals a target.
168. Implement Producer-Consumer using `wait()` and `notify()`.
169. REST API — return the top 5 highest salaries by department (using Streams).
170. Sort a list of `Employee` objects by salary, then by name.
171. Calculate totals grouped by user ID using Java Streams.
172. Implement API rate limiting in Java.

---

## 15. AI & Developer Productivity

173. How does GitHub Copilot work?
174. What is an LLM (Large Language Model)?
175. How would you design an AI-based code generation application?
176. Do you think AI will change software development? Will AI replace software developers?

---

## 16. Behavioural & Situational

177. Tell me about a production issue you solved.
178. What was the most challenging bug you have debugged?
179. How did you improve API performance in your project?
180. How did you handle concurrent requests?
181. Where did you use Kafka, RabbitMQ, or Redis, and why?
182. What technical decision would you change if you redesigned your project today?
183. Why do you perform code reviews? What do you check during a code review?
184. What aspects of Spring have you worked on other than REST APIs?
185. Do you have any questions for us?
