# JUnit 5, Mockito, TDD & CI: The Architect's Interview Cheatsheet

*A plain-English guide. Every concept gets a kid-friendly picture first, then the technical truth, then the "what would I say in the interview" part.*

---

## 0. The Big Picture

**Kid version:** Imagine you build a LEGO car. Before showing it to anyone, you push it on the table to see if the wheels fall off. **Tests are you pushing the car on the table**, except a computer does it in milliseconds, every time you change anything.

**Architect version:** Automated tests are a **safety net for change**. They let a team refactor, upgrade dependencies, and ship daily without fear. The question in an interview is never "do you write tests?" It's *"what do you test, at which level, and what does it cost to maintain?"*

| Question | One-line answer |
|---|---|
| **Why** test? | Catch bugs early (cheap) instead of in production (expensive). Enable safe refactoring. Act as living documentation. |
| **When** do tests run? | Locally on every save/commit, and in CI on every push/PR. |
| **Where** do they live? | `src/test/java`, mirroring the package structure of `src/main/java`. |
| **Worst case without tests** | Fear-driven development. Nobody touches old code, so the system rots. |

---

## 1. JUnit 5: The Test Runner

**Kid version:** JUnit is the **referee**. You write the rules ("2 + 2 must equal 4"), and the referee runs the game and blows the whistle when a rule breaks.

### 1.1 Architecture (a favorite interview question)

JUnit 5 = **Platform + Jupiter + Vintage**

- **JUnit Platform**: the foundation. Launches test frameworks on the JVM and is what Maven, Gradle, and IDEs talk to.
- **JUnit Jupiter**: the new programming model (`@Test`, etc.) plus the extension model. This is "JUnit 5" in daily life.
- **JUnit Vintage**: lets old JUnit 3/4 tests run on the new platform, which makes it useful for gradual migrations.

> **2026 note:** JUnit 6 has been released (Java 17+ baseline, unified versioning). The Jupiter API you learn here carries over almost unchanged, so say "JUnit 5/Jupiter model" with confidence.

### 1.2 Your first test

```java
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class CalculatorTest {

    private final Calculator calc = new Calculator();

    @Test
    @DisplayName("adds two positive numbers")
    void add_twoPositives_returnsSum() {
        assertEquals(5, calc.add(2, 3));
    }
}
```

**Naming tip:** `method_scenario_expectedResult` or `shouldDoXWhenY`. Test names are documentation.

**Note:** JUnit 5 test classes and methods **don't need to be `public`** (a JUnit 4 difference interviewers love).

### 1.3 Assertions: "Is the world what I expected?"

```java
assertEquals(expected, actual);              // expected FIRST, actual second
assertNotEquals(a, b);
assertTrue(list.isEmpty());
assertFalse(user.isBlocked());
assertNull(repo.findById(99));
assertNotNull(result);
assertSame(a, b);                            // same object reference (==)
assertArrayEquals(new int[]{1,2}, arr);
assertIterableEquals(expectedList, actualList);

// Exceptions
IllegalArgumentException ex = assertThrows(
        IllegalArgumentException.class,
        () -> service.withdraw(-100));
assertEquals("Amount must be positive", ex.getMessage());

assertDoesNotThrow(() -> service.ping());

// Grouped: runs ALL, reports ALL failures (not just the first)
assertAll("user",
        () -> assertEquals("Shubham", user.getName()),
        () -> assertEquals("Indore", user.getCity()),
        () -> assertTrue(user.isActive()));

// Time limits
assertTimeout(Duration.ofMillis(200), () -> service.fastCall());
assertTimeoutPreemptively(Duration.ofMillis(200), () -> service.fastCall()); // aborts the task

// Lazy failure message (only built if the test fails, so it is cheap)
assertTrue(x > 0, () -> "Expected positive but got " + x);
```

**Gotcha:** `assertEquals(actual, expected)` still passes or fails correctly, but the failure message reads backwards and confuses people. Keep the order.

**Pro move:** Use **AssertJ** (`assertThat(list).hasSize(3).contains("a")`) for fluent, readable assertions and much better failure messages. Many architects standardize on it.

### 1.4 Other must-know annotations

| Annotation | Purpose |
|---|---|
| `@DisplayName("...")` | Human-readable name in reports |
| `@Disabled("reason")` | Skip a test (always give a reason, and never let it rot) |
| `@Nested` | Group tests inside inner classes (describe-style structure) |
| `@Tag("slow")` | Label tests, then filter in CI (`-Dgroups=fast`) |
| `@RepeatedTest(5)` | Run the same test N times (useful for hunting flakiness) |
| `@Timeout(2)` | Fail if the test exceeds 2 seconds |
| `@TempDir` | Injects a temporary directory that is auto-cleaned |
| `@TestMethodOrder` | Control order (a code smell if you *need* it) |
| `@EnabledOnOs`, `@EnabledIfEnvironmentVariable` | Conditional execution |
| `assumeTrue(...)` | **Abort** (not fail) a test if a precondition isn't met |
| `@ExtendWith(...)` | Plug in extensions (Mockito, Spring, custom) |

---

## 2. Test Lifecycle: The Order of Events

**Kid version:** Think of a **school play**.
- `@BeforeAll` = the **theater is unlocked once** in the morning.
- `@BeforeEach` = **each actor changes into costume** before their scene.
- The test = the scene.
- `@AfterEach` = **actor takes off costume** after each scene.
- `@AfterAll` = **theater is locked once** at night.

```java
class LifecycleDemoTest {

    @BeforeAll
    static void startDatabase() { System.out.println("1. once, before everything"); }

    @BeforeEach
    void setUp() { System.out.println("2. before EACH test"); }

    @Test void testA() { System.out.println("3. test A"); }
    @Test void testB() { System.out.println("3. test B"); }

    @AfterEach
    void tearDown() { System.out.println("4. after EACH test"); }

    @AfterAll
    static void stopDatabase() { System.out.println("5. once, after everything"); }
}
```

**Output order:** `1 → (2,3,4 for A) → (2,3,4 for B) → 5`

### Key facts (interview gold)

1. **`@BeforeAll` / `@AfterAll` must be `static`** by default, because JUnit creates a **new instance of the test class for every test method** (so tests can't leak state through fields).
2. **To make them non-static**, switch the lifecycle:
   ```java
   @TestInstance(TestInstance.Lifecycle.PER_CLASS)
   class SharedStateTest {
       @BeforeAll void init() { ... }   // non-static now: one instance for all tests
   }
   ```
   Use `PER_CLASS` sparingly, since shared mutable state is where flaky tests are born.
3. **JUnit 4 → 5 renames** (asked all the time):
   - `@Before` → `@BeforeEach`
   - `@After` → `@AfterEach`
   - `@BeforeClass` → `@BeforeAll`
   - `@AfterClass` → `@AfterAll`
   - `@Ignore` → `@Disabled`
   - `@RunWith` / `@Rule` → `@ExtendWith`
4. Superclass `@BeforeEach` runs **before** subclass `@BeforeEach`. `@AfterEach` runs in reverse order (subclass first).
5. `@AfterEach` runs **even if the test fails**, so it is the right place for cleanup.

### Production scenarios

| Hook | Real use |
|---|---|
| `@BeforeAll` | Start a Testcontainers DB, load a heavy config once |
| `@BeforeEach` | Reset mocks, build fresh test data, clear caches |
| `@AfterEach` | Delete temp files, clear `ThreadLocal`s, reset static state |
| `@AfterAll` | Stop containers, close connection pools |

**Worst case:** doing expensive setup in `@BeforeEach` (for example, booting a DB for each of 500 tests) makes the suite take 20 minutes, so developers stop running it. Use `@BeforeAll` or a shared container.

---

## 3. Parameterized Tests: One Test, Many Inputs

**Kid version:** Instead of writing 10 identical recipes that differ only in the fruit, write **one recipe and hand it 10 fruits**.

Add the dependency `junit-jupiter-params` (included with `spring-boot-starter-test`).

```java
class PalindromeTest {

    @ParameterizedTest
    @ValueSource(strings = {"madam", "racecar", "level"})
    void isPalindrome_true(String word) {
        assertTrue(Palindrome.check(word));
    }

    @ParameterizedTest(name = "{0} + {1} = {2}")
    @CsvSource({
        "1, 2, 3",
        "0, 0, 0",
        "-5, 5, 0"
    })
    void add(int a, int b, int expected) {
        assertEquals(expected, new Calculator().add(a, b));
    }
}
```

### The sources

| Source | When to use |
|---|---|
| `@ValueSource` | One simple argument (ints, strings, etc.) |
| `@CsvSource` | A few rows of multiple args, inline |
| `@CsvFileSource(resources = "/data.csv")` | Many rows, kept in a file |
| `@EnumSource(Status.class)` | Test every enum value (`mode = EXCLUDE`, `names = ...` to filter) |
| `@MethodSource("provider")` | Complex objects or computed data |
| `@ArgumentsSource` | Reusable custom provider class |
| `@NullSource`, `@EmptySource`, `@NullAndEmptySource` | Classic edge cases for free |

```java
@ParameterizedTest
@MethodSource("invalidOrders")
void rejectsInvalidOrders(Order order, String expectedError) {
    var ex = assertThrows(ValidationException.class, () -> validator.validate(order));
    assertEquals(expectedError, ex.getMessage());
}

static Stream<Arguments> invalidOrders() {
    return Stream.of(
        Arguments.of(new Order(null, 5), "Customer required"),
        Arguments.of(new Order("C1", 0), "Quantity must be > 0")
    );
}
```

**Best case:** boundary-value testing (0, 1, max, max+1), validation rules, tax and discount tables, and input matrices.
**Worst case:** unreadable 80-row `@CsvSource` blocks, or loops of logic inside the test (which makes the test itself need testing). Move big tables to CSV files.
**Alternatives:** `@TestFactory` with `DynamicTest` generates tests at runtime. It is more powerful and rarely needed.

---

## 4. Mocking with Mockito

**Kid version:** You're rehearsing a scene where the hero talks to a dragon. You don't hire a real dragon, you use a **person in a dragon costume** who says exactly the lines you scripted. A **mock** is the costume: a fake object you control, so you can test the hero **alone**.

### Why mock?

Your `OrderService` calls `PaymentGateway` (a remote HTTP service), which is slow, costs money, and fails randomly. In a *unit* test you want to test **only `OrderService`'s logic**, so you replace its collaborators with mocks. This gives you speed, determinism, and isolation, and it lets you simulate failures (timeouts, 500s) that are hard to trigger for real.

### The standard setup

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock PaymentGateway paymentGateway;
    @Mock InventoryClient inventory;
    @InjectMocks OrderService orderService;   // mocks injected into this

    @Test
    void placeOrder_success() {
        // ARRANGE (stub)
        when(inventory.isAvailable("SKU1", 2)).thenReturn(true);
        when(paymentGateway.charge(any(), eq(500.0))).thenReturn(PaymentResult.OK);

        // ACT
        OrderResult result = orderService.placeOrder("SKU1", 2, 500.0);

        // ASSERT
        assertTrue(result.isSuccess());
        verify(paymentGateway, times(1)).charge(any(), eq(500.0));
        verify(inventory).reserve("SKU1", 2);
        verifyNoMoreInteractions(paymentGateway);
    }

    @Test
    void placeOrder_paymentTimeout_rollsBackInventory() {
        when(inventory.isAvailable(any(), anyInt())).thenReturn(true);
        when(paymentGateway.charge(any(), anyDouble()))
            .thenThrow(new PaymentTimeoutException());

        assertThrows(OrderFailedException.class,
            () -> orderService.placeOrder("SKU1", 1, 100.0));

        verify(inventory).release("SKU1", 1);   // compensation happened!
    }
}
```

### The Mockito cheat list

```java
// STUBBING
when(mock.method(arg)).thenReturn(value);
when(mock.method(any())).thenThrow(new RuntimeException());
when(mock.method()).thenReturn(1).thenReturn(2);      // consecutive calls
when(mock.method(anyString())).thenAnswer(inv -> inv.getArgument(0));
doThrow(new X()).when(mock).voidMethod();             // for void methods
doNothing().when(mock).voidMethod();

// VERIFYING
verify(mock).method(arg);                  // exactly once
verify(mock, times(3)).method();
verify(mock, never()).method();
verify(mock, atLeastOnce()).method();
verifyNoInteractions(mock);
verifyNoMoreInteractions(mock);
InOrder order = inOrder(a, b);             // verify call ORDER
order.verify(a).first();
order.verify(b).second();

// ARGUMENT MATCHERS: if one arg uses a matcher, ALL must
verify(mock).save(eq("id"), any(User.class), anyInt());

// CAPTURING what was passed
ArgumentCaptor<Email> captor = ArgumentCaptor.forClass(Email.class);
verify(mailer).send(captor.capture());
assertEquals("hi@x.com", captor.getValue().getTo());
```

### Mocks vs Stubs vs Spies vs Fakes (the "Test Doubles" family)

| Type | Kid explanation | Technical meaning |
|---|---|---|
| **Dummy** | A prop that's never touched | Passed only to satisfy a signature (`null` or `new Object()`) |
| **Stub** | A recording that **always says the same answer** | Returns canned responses and is used for **state** verification ("given X returns Y") |
| **Mock** | A stub with a **notebook** that records who called it, and how | Used for **behavior** verification (`verify(...)`) |
| **Spy** | A **real object with a hidden camera** | Wraps a real object, so real methods run unless you stub specific ones |
| **Fake** | A **toy version** that really works, just simpler | A working lightweight implementation (in-memory repository, H2 DB) |

In Mockito, `@Mock` creates mocks (which you can also stub), and `@Spy` creates spies.

```java
@Spy List<String> list = new ArrayList<>();

list.add("real");                            // real method runs
doReturn(100).when(list).size();             // note doReturn style: avoids calling the real method
```

**Spy warning:** needing spies often means the class does too much. Prefer refactoring over spying.

### Mockito rules and gotchas

1. **Don't mock what you don't own.** Wrap third-party APIs in your own adapter and mock the adapter.
2. **Don't mock value objects or simple POJOs.** Just build them.
3. **Strict stubs** (default with `MockitoExtension`) fail on **unused stubs**. This is good, because it exposes dead setup.
4. **Over-mocking** gives tests that verify *implementation* instead of *behavior*, so they break on every refactor and prove nothing. A test that mocks 8 collaborators is shouting "redesign me."
5. **Mockito can't mock** `final` classes or static methods by default in old versions. Since Mockito 5, the inline mock maker is the default, so final classes work, and `mockStatic()` exists but treat it as a last resort.
6. `@Mock` (pure Mockito) vs `@MockBean` / `@MockitoBean` (Spring: replaces a bean in the Spring context). Know the difference. In Spring Boot 3.4+, `@MockitoBean` replaces the deprecated `@MockBean`.

---

## 5. Test Coverage with JaCoCo

**Kid version:** You have a coloring book. After your tests "color" the code, JaCoCo shows **which pages are still blank**. Blank = no test ever ran this code.

### Coverage types

| Metric | Meaning |
|---|---|
| **Line** | Was this line executed? |
| **Branch** | Were **both** sides of each `if`/`switch` taken? (More honest than line coverage) |
| **Instruction** | Bytecode-level coverage |
| **Complexity** | Cyclomatic paths covered |
| **Method / Class** | Were they entered at all? |

### Maven setup

```xml
<plugin>
  <groupId>org.jacoco</groupId>
  <artifactId>jacoco-maven-plugin</artifactId>
  <version>0.8.12</version>
  <executions>
    <execution>
      <goals><goal>prepare-agent</goal></goals>
    </execution>
    <execution>
      <id>report</id>
      <phase>test</phase>
      <goals><goal>report</goal></goals>
    </execution>
    <execution>
      <id>check</id>
      <goals><goal>check</goal></goals>
      <configuration>
        <rules>
          <rule>
            <element>BUNDLE</element>
            <limits>
              <limit>
                <counter>BRANCH</counter>
                <value>COVEREDRATIO</value>
                <minimum>0.70</minimum>
              </limit>
            </limits>
          </rule>
        </rules>
      </configuration>
    </execution>
  </executions>
</plugin>
```

Run `mvn clean verify`, then open `target/site/jacoco/index.html`.

**Colors:** green = covered, yellow = partially covered branches, red = not covered.

### The wisdom (what separates seniors from juniors)

- **Coverage measures what was *executed*, not what was *verified*.** A test with zero assertions can give 100% coverage and catch nothing.
- **100% is a trap.** Chasing it produces junk tests for getters and config. A sensible target for business logic is **70-85% branch coverage**, enforced as a **quality gate** in CI, with exclusions for DTOs, generated code, and `main` classes.
- Use coverage as a **flashlight for untested risk**, not as a scoreboard.
- **Mutation testing (PIT/Pitest)** is the next level. It deliberately breaks your code (flips `>` to `<`) and checks whether tests notice. It answers "are my tests actually good?" Mentioning it in an interview is a strong signal.
- JaCoCo integrates with **SonarQube** for "coverage on new code" gates, a pragmatic approach for legacy systems.

---

## 6. Integration Testing with Spring Boot

**Kid version:** Unit tests check that the **wheel**, the **engine**, and the **steering** each work alone. An integration test **drives the whole car** and checks that the parts work *together*.

### The testing toolbox

| Annotation | What it loads | Speed | Use for |
|---|---|---|---|
| *(none, plain JUnit+Mockito)* | Nothing | ⚡⚡⚡ | Pure business logic |
| `@WebMvcTest(Controller.class)` | **Only the web layer** (controllers, filters, JSON) | ⚡⚡ | Request mapping, validation, status codes, security rules |
| `@DataJpaTest` | **Only JPA** (entities, repos, in-memory or real DB) | ⚡⚡ | Queries, mappings, constraints |
| `@JsonTest` | Only JSON serialization | ⚡⚡ | DTO shapes |
| `@SpringBootTest` | **The full application context** | 🐢 | End-to-end flows across layers |

These are called **test slices**. They load only what's needed, which keeps tests fast.

### Web layer slice (`@WebMvcTest` + MockMvc)

```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {

    @Autowired MockMvc mockMvc;
    @MockitoBean OrderService orderService;     // (@MockBean before Boot 3.4)

    @Test
    void getOrder_returns200() throws Exception {
        when(orderService.find(1L)).thenReturn(new OrderDto(1L, "PAID"));

        mockMvc.perform(get("/orders/1"))
               .andExpect(status().isOk())
               .andExpect(jsonPath("$.status").value("PAID"));
    }

    @Test
    void createOrder_invalidBody_returns400() throws Exception {
        mockMvc.perform(post("/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"quantity\": -1}"))
               .andExpect(status().isBadRequest());
    }
}
```

### Full integration with a real database (Testcontainers, the modern standard)

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
class OrderFlowIT {                       // "IT" suffix → run by Failsafe, not Surefire

    @Container
    @ServiceConnection                    // Boot 3.1+: auto-wires datasource props
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16");

    @Autowired TestRestTemplate rest;
    @Autowired OrderRepository repo;

    @Test
    void createAndFetchOrder() {
        var response = rest.postForEntity("/orders", new CreateOrder("SKU1", 2), OrderDto.class);

        assertEquals(HttpStatus.CREATED, response.getStatusCode());
        assertTrue(repo.findById(response.getBody().id()).isPresent());
    }
}
```

*(Before Boot 3.1, you'd use `@DynamicPropertySource` to inject the container's JDBC URL.)*

### Why Testcontainers over H2?

**H2 is a different database pretending to be yours.** SQL dialect, JSON columns, locking, and indexes behave differently, so you get "passes on H2, fails on Postgres in prod." Testcontainers spins up the **real thing** in Docker for the test run. Costs: Docker is needed on CI, and it is slower. It is worth it.

### Things to know for production

- **`@Transactional` on a test** rolls back after each test (nice cleanup). Caveat: it can **hide** problems like lazy-loading or commit-time constraint failures, and it doesn't span a real HTTP server thread.
- **Spring caches the application context** between tests with identical config. Different `@MockBean`s or `@ActiveProfiles` create *new* contexts, which slows suites down. Keep configurations uniform.
- **Profiles:** use `@ActiveProfiles("test")` plus `application-test.yml`.
- **External HTTP APIs:** use **WireMock** or **MockWebServer** to simulate them, including slow responses and 500s.
- **Messaging (Kafka/RabbitMQ):** Testcontainers modules, or embedded Kafka.
- **Security:** `@WithMockUser`, or `jwt()` request post-processors for OAuth2 resource servers.
- **Async code:** use **Awaitility** (`await().atMost(5, SECONDS).until(...)`), never `Thread.sleep()`.
- **Microservices:** add **consumer-driven contract tests** (Spring Cloud Contract, Pact), so service A and B agree on the API *without* spinning up both.

**Naming convention:** `*Test` = unit tests (Maven Surefire, run in `test` phase); `*IT` = integration tests (Maven Failsafe, run in `verify` phase). This lets CI run fast tests first.

---

## 7. Test-Driven Development (TDD)

**Kid version:** Before you build a bookshelf, you **draw a picture of what "done" looks like**. Then you build until your bookshelf matches the picture. In TDD the test is the picture, written **before** the code.

### The Red → Green → Refactor loop

```
   🔴 RED       Write a small failing test (proves the test can fail)
     ↓
   🟢 GREEN     Write the simplest code that makes it pass (even ugly!)
     ↓
   🔵 REFACTOR  Clean up code AND tests, while tests stay green
     ↓
   (repeat in tiny steps, minutes each)
```

### A mini example

```java
// 1. RED: test first, code doesn't exist yet (won't even compile)
@Test
void fizzBuzz_multipleOf3_returnsFizz() {
    assertEquals("Fizz", FizzBuzz.of(3));
}

// 2. GREEN: the dumbest thing that passes
static String of(int n) { return "Fizz"; }

// 3. Next test forces generalization:
@Test void fizzBuzz_multipleOf5_returnsBuzz() { assertEquals("Buzz", FizzBuzz.of(5)); }
// → now you must write real logic. The tests drive the design.
```

### Why bother? (benefits you can claim)

1. **Design pressure:** hard-to-test code is usually badly designed (tight coupling, hidden dependencies). TDD forces small, injectable, single-purpose classes.
2. **Living specification:** tests describe behavior.
3. **Fewer defects and fearless refactoring.**
4. **Focus:** you only write code that a test demands, which prevents over-engineering.

### Honest downsides (say these in interviews, it shows maturity)

- Slower at first, with a learning curve.
- Poor fit for exploratory or spike work, UI-heavy work, and throwaway prototypes. *Spike first, then delete it and TDD the real thing.*
- Bad tests (over-mocked, implementation-coupled) make TDD feel like pain.
- It doesn't replace integration, performance, or exploratory testing.

### TDD styles

- **Classicist / Chicago / Detroit:** use real objects where possible, mocks sparingly, and verify state.
- **Mockist / London:** mock collaborators heavily and verify interactions (outside-in design).
- **BDD (Given-When-Then):** the same idea, phrased in business language (Cucumber, or `@Nested` plus clear naming).
- **ATDD:** start from an *acceptance* test that stakeholders understand.

### Testing legacy code (very common in real life)

You can't TDD what already exists. Instead: write **characterization tests** (tests that lock in current behavior), then refactor safely, and use TDD for **new** changes. Mention Michael Feathers' *Working Effectively with Legacy Code* for bonus points.

---

## 8. The Test Pyramid

**Kid version:** Build a **pyramid of snacks**: a big base of cheap, quick crackers (lots), a middle of sandwiches (some), and one expensive cake on top (just a few).

```
          /\
         /  \       E2E / UI tests       → FEW, slow, brittle, costly
        /----\
       /      \     Integration / API    → SOME, medium speed
      /--------\
     /          \   UNIT tests           → MANY, milliseconds, cheap
    /____________\
```

| Level | Scope | Speed | Flakiness | Cost to maintain | Tools |
|---|---|---|---|---|---|
| **Unit** | One class or method, collaborators mocked | ms | Very low | Low | JUnit, Mockito |
| **Integration** | Several components plus real infrastructure | seconds | Medium | Medium | Spring Boot Test, Testcontainers, WireMock |
| **E2E / UI** | Whole system through the UI or public API | minutes | High | High | Selenium, Playwright, Cypress, REST Assured |

### Anti-patterns

- **Ice-cream cone 🍦:** inverted pyramid. Few unit tests, tons of manual and UI tests. Result: slow feedback, flaky pipelines, release fear.
- **Hourglass:** many unit and E2E tests but nothing in the middle, so integration bugs slip through.

### The modern nuance (architect-level)

In **microservices**, many teams prefer the **"testing honeycomb/trophy"**: fewer pure unit tests on trivial logic, **more integration tests per service** (real DB via Testcontainers, stubbed neighbors), plus contract tests between services and a *very thin* layer of E2E. The underlying principle stays the same: **push each check to the cheapest level that can catch the bug.**

---

## 9. Continuous Integration (CI)

**Kid version:** A **robot teacher** who checks every homework page the moment you hand it in. If anything is wrong, the whole class hears about it immediately, so nobody builds on a broken page.

### What CI does on every push or PR

```
Commit → Checkout → Compile → Unit tests → Static analysis (Sonar/Checkstyle)
       → Package → Integration tests → Coverage gate → Publish artifact
       → (CD) Deploy to staging → Smoke tests → Deploy to prod
```

**CI** = integrate and verify often. **CD** = Continuous *Delivery* (deployable at the push of a button) or *Deployment* (auto-releases to production).

### Tools compared

| Tool | Style | Strengths | Watch out for |
|---|---|---|---|
| **Jenkins** | Self-hosted, plugin-driven | Hugely flexible, huge ecosystem, enterprise favorite | You maintain servers, plugins, and security patches |
| **GitLab CI** | Built into GitLab, YAML | Pipeline lives with repo, built-in registry, runners, and security scans | Tied to GitLab |
| **GitHub Actions** | Built into GitHub, YAML | Giant marketplace of actions, easy start | Minutes cost on private repos |
| **Azure DevOps / CircleCI / Bamboo / TeamCity** | Various | Enterprise integrations | Varies |

### Jenkinsfile (Declarative Pipeline)

```groovy
pipeline {
  agent any
  tools { maven 'Maven3'; jdk 'JDK21' }
  stages {
    stage('Build & Unit Test') {
      steps { sh 'mvn clean test' }
      post { always { junit 'target/surefire-reports/*.xml' } }
    }
    stage('Integration Test') {
      steps { sh 'mvn verify -DskipUnitTests' }
    }
    stage('Quality Gate') {
      steps {
        withSonarQubeEnv('sonar') { sh 'mvn sonar:sonar' }
        timeout(time: 5, unit: 'MINUTES') { waitForQualityGate abortPipeline: true }
      }
    }
    stage('Package') { steps { sh 'mvn package -DskipTests' } }
  }
  post { failure { mail to: 'team@company.com', subject: 'Build failed', body: "${env.BUILD_URL}" } }
}
```

### `.gitlab-ci.yml`

```yaml
stages: [test, verify, package]

unit-tests:
  stage: test
  image: maven:3.9-eclipse-temurin-21
  script: mvn clean test
  artifacts:
    when: always
    reports:
      junit: target/surefire-reports/TEST-*.xml
  cache:
    paths: [.m2/repository]

integration-tests:
  stage: verify
  image: maven:3.9-eclipse-temurin-21
  services:
    - docker:dind          # for Testcontainers
  script: mvn verify
  coverage: '/Total.*?([0-9]{1,3})%/'
```

### CI best practices (the cheat list)

1. **Keep the main branch always green.** A broken build is the team's top priority.
2. **Fast feedback:** unit tests in under ~10 minutes. Fail fast (cheap checks first), cache dependencies, and parallelize.
3. **Run in a clean, reproducible environment** (containers), so there is no "works on my machine."
4. **Quality gates:** minimum coverage on *new code*, zero critical Sonar issues, and dependency vulnerability scans (OWASP Dependency-Check, Snyk, Trivy).
5. **Publish test reports** (JUnit XML) and coverage so failures are diagnosable.
6. **Build once, deploy the same artifact everywhere.** Don't rebuild per environment.
7. **Pipeline as code:** Jenkinsfile or YAML lives in the repo, reviewed like code.
8. **Branch protection:** PRs require a green pipeline plus review.

### Flaky tests: the silent killer

A flaky test passes and fails randomly with no code change. Common causes: `Thread.sleep()` timing, shared state or test order dependence, real network and clocks, random data, and timezone or locale issues. Fixes: use **Awaitility**, inject a `Clock`, isolate data, give each test its own data, and use `@RepeatedTest` to reproduce. **Policy:** quarantine and fix flakies fast, because they train the team to ignore red builds, which is how real bugs ship.

---

## 10. Putting It Together: A Production Scenario

**Feature:** "Place an order" in an e-commerce microservice.

| Layer | Test | Tools |
|---|---|---|
| Pricing and discount rules | Unit tests plus **parameterized** boundary cases | JUnit 5, `@CsvSource` |
| `OrderService` orchestration | Unit tests with mocked `PaymentGateway` and `InventoryClient`, including failure and **compensation** paths | Mockito |
| `OrderController` | Slice test: JSON, validation, 400/404/500 mapping, security | `@WebMvcTest`, MockMvc |
| `OrderRepository` queries | Real Postgres container | `@DataJpaTest` + Testcontainers |
| Whole flow | Boot the app, POST an order, check DB and outbound Kafka event | `@SpringBootTest` + Testcontainers + WireMock |
| Service contracts | Provider/consumer contract verification | Pact / Spring Cloud Contract |
| Pipeline | All of the above on every PR, with a coverage gate | Jenkins/GitLab CI, JaCoCo, Sonar |

---

## 11. Best Cases, Worst Cases & Anti-Patterns

### ✅ Do

- **AAA structure:** Arrange, Act, Assert (or Given-When-Then), one clear behavior per test.
- Tests must be **F.I.R.S.T.:** **F**ast, **I**ndependent, **R**epeatable, **S**elf-validating, **T**imely.
- Test **behavior**, not implementation details.
- Cover **edge cases:** null, empty, boundaries, concurrency, time zones, large inputs, failure paths.
- Treat test code as **production-quality code** (readable, refactored, reviewed).
- Use test data builders or Object Mothers to reduce setup noise.

### ❌ Don't

| Anti-pattern | Why it hurts |
|---|---|
| Tests with **no assertions** | Fake coverage |
| **Over-mocking** | Tests break on refactor, prove nothing |
| **Testing private methods** directly | Couples tests to internals. Test via the public API |
| **Order-dependent tests** | Fail randomly, can't parallelize |
| **`Thread.sleep`** in tests | Slow *and* flaky |
| **Shared mutable static state** | Cross-test pollution |
| **Giant "God" tests** (50 asserts) | A failure tells you nothing |
| **Chasing 100% coverage** | Wasteful tests for getters |
| **Testing the framework** (does Spring autowire?) | Not your job |
| **Ignoring or `@Disabled` forever** | Silent rot |
| **Real network or clock calls** in unit tests | Non-deterministic |
| **Copy-paste tests** instead of parameterized | Maintenance nightmare |

---

## 12. Interview Rapid-Fire Q&A

**Q: Difference between JUnit 4 and 5?**
Modular (Platform/Jupiter/Vintage), Java 8+ lambdas, `@ExtendWith` replaces `@RunWith`/`@Rule`, new lifecycle annotations, `assertAll`/`assertThrows`, `@Nested`, `@ParameterizedTest`, `@DisplayName`, and non-public tests.

**Q: Why are `@BeforeAll` methods static?**
A new test-class instance is created per test method, so a once-only hook can't belong to an instance. Use `@TestInstance(PER_CLASS)` to change that.

**Q: Mock vs Stub vs Spy?**
Stub returns canned answers. Mock additionally verifies interactions. Spy wraps a real object and lets you override parts of it.

**Q: `@Mock` vs `@MockBean`?**
`@Mock` is pure Mockito (no Spring). `@MockBean` / `@MockitoBean` replaces a bean in the Spring context.

**Q: `@WebMvcTest` vs `@SpringBootTest`?**
Slice (only the web layer, fast) vs the full application context (slow, realistic).

**Q: What's wrong with 100% coverage?**
It measures execution, not verification. Aim for meaningful branch coverage on business logic, backed by mutation testing for quality.

**Q: How do you test a `void` method?**
Verify side effects: `verify(mock).doSomething()`, capture arguments, or check state changes.

**Q: How do you test private methods?**
You don't directly. Test through public behavior. If a private method is complex enough to need its own test, it probably deserves its own class.

**Q: How do you test time-dependent code?**
Inject `java.time.Clock`, and use `Clock.fixed(...)` in tests.

**Q: How do you test legacy code with no tests?**
Characterization tests first, then refactor behind them, then TDD new changes.

**Q: Unit vs Integration vs E2E?**
The pyramid. Many fast unit tests, fewer integration tests, and very few E2E tests.

**Q: How do you handle flaky tests?**
Find the root cause (timing, shared state, external dependency), fix or isolate it, and quarantine it temporarily rather than ignoring red builds.

**Q: Why Testcontainers over H2?**
It runs the real database engine, so there is no dialect-mismatch surprise in production.

**Q: Surefire vs Failsafe?**
Surefire runs unit tests (`test` phase), Failsafe runs integration tests (`integration-test`/`verify` phases) and lets the build clean up before failing.

**Q: How do you test microservices in CI?**
Unit tests plus slice tests plus Testcontainers integration tests per service, contract tests between services, and a minimal E2E smoke suite after deploy.

**Q: Where does TDD fail?**
Exploratory spikes, UI-heavy work, and teams that over-mock. Spike first, then rewrite test-first.

---

## 13. One-Page Memory Cheat

```
LIFECYCLE   @BeforeAll(static) → [@BeforeEach → @Test → @AfterEach]×N → @AfterAll(static)

ASSERT      assertEquals(expected, actual) · assertThrows · assertAll · assertTimeout
PARAM       @ValueSource · @CsvSource · @CsvFileSource · @MethodSource · @EnumSource

MOCKITO     @Mock · @InjectMocks · @Spy · when().thenReturn() · doThrow().when()
            verify(times/never/inOrder) · ArgumentCaptor · "don't mock what you don't own"

COVERAGE    JaCoCo: Line · Branch(best) · prepare-agent → report → check(gate)
            Coverage ≠ quality → mutation testing (PIT)

SPRING      @WebMvcTest(web) · @DataJpaTest(JPA) · @SpringBootTest(full)
            @MockitoBean · MockMvc · Testcontainers(+@ServiceConnection) · WireMock
            *Test → Surefire · *IT → Failsafe

TDD         🔴 Red → 🟢 Green → 🔵 Refactor · tests drive design

PYRAMID     Many Unit · Some Integration · Few E2E  (avoid the ice-cream cone)
DOUBLES     Dummy · Stub · Mock · Spy · Fake

CI          Commit→Build→Unit→Sonar→Package→IT→Gate→Deploy
            Jenkins (Jenkinsfile) · GitLab CI (.gitlab-ci.yml) · GitHub Actions
            Green main · fast feedback · fix flaky tests · quality gates
```

---

### 🎯 Architect-level closing line for interviews

> *"I treat tests as an investment portfolio: cheap, fast unit tests guard business logic, Testcontainers-backed integration tests guard the seams, contract tests guard service boundaries, and a thin E2E layer guards the critical user journeys, all enforced by CI quality gates. Coverage is a flashlight, not a goal. TDD is my tool for design and for legacy refactoring safety."*

Good luck with the interviews, Shubham. You've got this. 🚀