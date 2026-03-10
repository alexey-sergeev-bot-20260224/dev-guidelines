# Backend Developer Testing Guide

> How to write good unit and integration tests.
> For human developers and AI agents alike.

---

## 1. Why We Test

- **Verify behavior** — confirm that code does what it's supposed to do.
- **Document intent** — tests are living documentation of how the system behaves.
- **Catch regressions** — prevent breaking existing behavior when changing code.
- **Enable refactoring** — with tests as a safety net, you can restructure code with confidence.
- **Get fast feedback** — find bugs in seconds, not after deployment.

Tests are an investment. Write tests that provide ongoing value, not tests that exist only to satisfy a metric.

---

## 2. The Four Pillars of a Good Test

Every test should be evaluated against these four pillars (Khorikov):

| Pillar | Question to ask |
|--------|----------------|
| **Protection against regressions** | Will this test fail if I introduce a real bug? |
| **Resistance to refactoring** | Will this test survive internal changes that don't alter behavior? |
| **Fast feedback** | Does this test run fast enough to be used in the dev loop? |
| **Maintainability** | Is this test easy to read, change, and keep up to date? |

These pillars conflict — an end-to-end test maximizes regression protection but sacrifices speed. Use them as a decision framework: pick the testing style that best fits the behavior you're protecting.

### Resistance to Refactoring (Key Concept)

The most commonly violated pillar. A test resists refactoring when it continues to pass after internal code changes that preserve external behavior.

**How to achieve it:**
- Assert **externally observable outcomes** (return values, persisted state, emitted events) — not internal call sequences
- Avoid verifying internal method names, call order, or collaborator counts that aren't part of the contract
- Prefer black-box testing: rely on API and observable effects
- If a purely internal refactor would break your test, the test is too tightly coupled to implementation

---

## 3. Test Stack

The backend services use two test ecosystems depending on the language:

### Groovy (Spock)

| What | Tool |
|------|------|
| Test framework | **Spock** (`spock.lang.Specification`) |
| Test file naming | `<ClassUnderTest>Spec.groovy` |
| Mocking | Spock built-in `Mock()`, `Stub()`, `Spy()` |
| Parameterized tests | `where:` block + `@Unroll` |
| GORM test support | `DataTest` trait + `mockDomains()` |
| Integration | `@SpringBootTest`, DbUnit |

### Kotlin (JUnit 5)

| What | Tool |
|------|------|
| Test framework | **JUnit 5** (`org.junit.jupiter.api`) |
| Test file naming | `<ClassUnderTest>Test.kt` |
| Mocking | **Mockito** or **MockK** |
| Parameterized tests | `@ParameterizedTest` + `@ValueSource` / `@MethodSource` / `@CsvSource` |
| Assertions | JUnit `Assertions` or `kotlin.test` |
| Integration | `@SpringBootTest`, `MockMvc`, **Testcontainers** |

### Common Conventions

- Test file location: same package as the class under test, under `src/test/`
- One test class per production class (with possible sub-classes via `@Nested` or inner classes)
- Test resources, helpers, and factories live **in the same package or sub-packages** as the tests that use them (see [Section 8: Test Data](#8-test-data))

---

## 4. What to Test: The Code Complexity Matrix

Not all code deserves the same testing effort. Use this matrix (Khorikov) to decide:

```
                    Few collaborators          Many collaborators
                ┌─────────────────────────┬─────────────────────────┐
  High          │  DOMAIN MODEL /         │  OVERCOMPLICATED CODE   │
  complexity    │  ALGORITHMS             │                         │
  / business    │                         │  → Refactor first,      │
  significance  │  → Unit test heavily    │    then test             │
                │    (highest value)      │                         │
                ├─────────────────────────┼─────────────────────────┤
  Low           │  TRIVIAL CODE           │  CONTROLLERS /          │
  complexity    │                         │  ORCHESTRATORS          │
                │  → Don't test           │                         │
                │    (getters, setters)   │  → Integration test     │
                │                         │    (few, focused)       │
                └─────────────────────────┴─────────────────────────┘
```

**In practice:**
- **Domain model & algorithms** — high complexity, few collaborators → unit test heavily (output-based or state-based). This is where most of your tests should live.
- **Controllers / orchestrators** — low complexity, many collaborators → fewer tests, use integration tests that exercise orchestration with real (in-process) collaborators.
- **Trivial code** — don't test simple getters/setters/delegation with no logic.
- **Overcomplicated code** — too complex AND too many dependencies → refactor to separate domain logic from coordination before testing.

### The Humble Object Pattern

When code is hard to test because it mixes business logic with infrastructure concerns (UI, DB, messaging), extract the logic into a testable "domain" object and leave the infrastructure part as a thin "humble" wrapper:

```
Before:  [Controller with complex business logic + DB calls]  ← hard to test

After:   [Controller (humble)]  →  [Domain Service (complex logic)]  ← easy to unit test
              ↓
         [Repository (infrastructure)]  ← integration test
```

---

## 5. Three Styles of Testing

| Style | What you verify | Best for | Resistance to refactoring |
|-------|----------------|----------|--------------------------|
| **Output-based** | Return value of a function | Pure functions, calculations, mappers | Highest — no coupling to internals |
| **State-based** | State of the system after an operation | Domain entities, stateful services | High — if asserting observable state |
| **Communication-based** | Interactions with collaborators (mocks) | System boundaries, external integrations | Lowest — easily couples to implementation |

**Prefer output-based > state-based > communication-based** when you have a choice.

**Output-based (best):**
```groovy
void 'calculates tax for US customer'() {
    expect:
    taxCalculator.calculate(amount: 100.0, country: 'US') == 7.50
}
```

**State-based (good):**
```groovy
void 'adds item to cart'() {
    when:
    cart.addItem(new Item(price: 10.0))

    then:
    cart.items.size() == 1
    cart.items[0].price == 10.0
}
```

**Communication-based (use only when necessary):**
```groovy
void 'sends notification when order placed'() {
    when:
    orderService.placeOrder(order)

    then:
    1 * notificationService.send(order.customerId, _)  // only if this IS the contract
}
```

---

## 6. Test Structure

### 6.1 Spock (Groovy): Given-When-Then

Every Spock test should use labeled blocks to clearly separate phases:

```groovy
void 'calculates total when multiple items exist'() {
    given: 'a cart with two items'
    def cart = new ShoppingCart()
    cart.addItem(new Item(price: 10.00))
    cart.addItem(new Item(price: 25.50))

    when: 'total is calculated'
    def total = cart.calculateTotal()

    then: 'total is sum of item prices'
    total == 35.50
}
```

**Allowed block combinations:**
- `given → when → then` — standard behavior test
- `given → expect` — pure computation (no side effects to verify)
- `setup → when → then` — `setup` is an alias for `given`
- `when → then → where` — parameterized tests

**Quick reference — Spock blocks:**
```
given:   / setup:   Setup — create objects, configure mocks
when:                      Exercise — call the method under test
then:                      Verify — assertions and mock verifications
expect:                    Combined when+then for simple cases
where:                     Data table for parameterized tests
cleanup:                   Teardown — release resources
and:                       Continue any block for readability
```

### 6.2 JUnit 5 (Kotlin): Arrange-Act-Assert

Use clear separation with comments or blank lines:

```kotlin
@Test
fun `calculates total when multiple items exist`() {
    // Arrange
    val cart = ShoppingCart()
    cart.addItem(Item(price = 10.00))
    cart.addItem(Item(price = 25.50))

    // Act
    val total = cart.calculateTotal()

    // Assert
    assertEquals(35.50, total)
}
```

Use `@Nested` inner classes to group related tests:

```kotlin
@Nested
@DisplayName("calculateTotal")
inner class CalculateTotal {

    @Test
    fun `returns zero for empty cart`() { ... }

    @Test
    fun `sums prices of all items`() { ... }
}
```

### 6.3 One Behavior Per Test

Each test method should verify **one specific behavior**. If you need to assert two different outcomes of two different actions, write two tests.

❌ **Bad:**
```groovy
void 'process transaction'() {
    when:
    service.processTransaction(txn)

    then:
    1 * repository.save(_)
    1 * notificationService.notify(_)
    txn.status == 'PROCESSED'
}
```

✅ **Good:**
```groovy
void 'saves transaction when processed'() { ... }
void 'sends notification when transaction processed'() { ... }
void 'marks transaction as processed'() { ... }
```

**Exception:** Integration tests may have multiple act-assert phases when testing a workflow sequence (e.g., create → process → verify result). This is acceptable because integration tests often need to exercise multi-step flows.

### 6.4 Test Naming

Use descriptive, readable names that describe behavior.

**Spock** — string literals:
```groovy
// ✅ Good — describes behavior
void 'returns null when user message belongs to different user'() { ... }
void 'applies tax pseudo code when tax percent is positive'() { ... }

// ❌ Bad — describes implementation
void 'testGetUserMessageById'() { ... }
void 'test1'() { ... }
```

**Kotlin** — backtick functions:
```kotlin
// ✅ Good
@Test
fun `rejects webhook from unsupported provider`() { ... }

@Test
fun `converts amount to smallest currency unit for USD`() { ... }

// ❌ Bad
@Test
fun testProcess() { ... }
```

**Pattern:** `<action/result> when/for <condition/context>`

---

## 7. Systematic Test Case Design

### 7.1 The 7-Step Process (Aniche)

Use this systematic workflow when designing tests for any method/feature:

1. **Understand** — Map inputs, outputs, pre-conditions, and post-conditions
2. **Explore** — Manually run through the behavior with sample inputs
3. **Partition** — Identify equivalence partitions (groups of inputs that should produce the same type of result)
4. **Analyze boundaries** — Find on-point (boundary itself) and off-point (just past boundary) values
5. **Devise test cases** — Create one test per partition + boundary combination
6. **Automate** — Use parameterized tests where many inputs share the same test skeleton
7. **Augment** — Add creative edge cases, unusual combinations, experience-based tests

**Example:** Method `calculateDiscount(amount, customerType)`:
- Partitions: amount (zero, small, large, negative), customerType (REGULAR, PREMIUM, null)
- Boundaries: discount threshold (e.g., amount = 100 is on-point, 99 and 101 are off-points)
- Creative: currency rounding, max BigDecimal value, concurrent calls

### 7.2 Equivalence Partitioning

Group inputs into classes where the system should behave the same way. Test at least one value from each partition.

```
Input: age (integer)
Partitions:
  - Invalid: negative values, null
  - Child: 0-17
  - Adult: 18-64
  - Senior: 65+
  - Boundary: MAX_INT, 0

→ Minimum test cases: one from each partition + boundary values
```

### 7.3 Boundary Value Analysis

Bugs cluster at boundaries. For every boundary, test:
- **On-point** — the boundary value itself
- **Off-point** — the value just past the boundary
- **In-point** — a value clearly inside the partition
- **Out-point** — a value clearly outside the partition

```
Rule: discount applies when amount >= 100

On-point:  100 (boundary — gets discount)
Off-point:  99 (just below — no discount)
In-point:  500 (clearly in discount range)
Out-point:  10 (clearly outside)
```

---

## 8. What to Test: Checklists

### 8.1 Right-BICEP (What to Test)

| Letter | Meaning | Ask yourself |
|--------|---------|--------------|
| **Right** | Are the right results returned? | Does the happy path work? |
| **B** | Boundary conditions | What happens at edges? (zero, one, max, empty, null) |
| **I** | Inverse relationships | Can you verify the result by reversing the operation? |
| **C** | Cross-check | Can you verify using an alternative method? |
| **E** | Error conditions | What should happen when things go wrong? |
| **P** | Performance characteristics | Does it complete within acceptable time? (rare in unit tests) |

### 8.2 CORRECT Boundary Conditions

| Letter | Boundary | Example |
|--------|----------|---------|
| **C** | Conformance | Does input conform to expected format? (email, UUID, date) |
| **O** | Ordering | Does order matter? (sorted results, sequence of operations) |
| **R** | Range | Within valid range? (negative amounts, overflow, min/max) |
| **R** | Reference | External dependencies present? (null references, missing associations) |
| **E** | Existence | Does the thing exist? (empty collection, null, missing record) |
| **C** | Cardinality | Zero, one, many — does each case work? |
| **T** | Time | Timing issues? (timeouts, ordering of events, time zones) |

### 8.3 ZOM — Zero, One, Many

For any collection or repeatable behavior:
1. **Zero** — empty collection, no items, null
2. **One** — single element (simplest case)
3. **Many** — multiple elements (general case + edge cases)

### 8.4 What NOT to Test

- Trivial getters/setters with no logic (trivial code quadrant)
- Framework behavior (don't test that Spring DI works)
- Third-party library internals
- Private methods directly — test them through public behavior
- The same behavior already covered by a higher-level integration test (unless testing specific edge cases)

---

## 9. Test Data

### 9.1 Co-Location Principle

Test resources, helpers, builders, and factories **must be located in the same package (or sub-packages) as the tests that use them**.

```
src/test/
├── groovy/com/synder/accounting/invoice/
│   ├── InvoiceServiceSpec.groovy          # test class
│   ├── InvoiceTestHelper.groovy           # helper — same package
│   └── testdata/                          # sub-package for test data
│       └── InvoiceBuilder.groovy
├── kotlin/com/synder/reconciliation/matching/
│   ├── MatchingServiceImplTest.kt         # test class
│   └── MatchingTestData.kt               # helper — same package
└── resources/com/synder/accounting/invoice/
    └── invoiceCreationTest.xml            # DbUnit data — same package path
```

**Why:** When test helpers sit in the same package as the tests, you can immediately see what's available. No hunting through a distant `utils` package. When a module changes, its test helpers change with it.

**Rules:**
- If a helper is used by only one spec/test → keep it as a private method or inner class in that spec/test
- If a helper is shared across multiple tests in the same domain → create a `*TestHelper` or `*TestBuilder` in the same package
- If a helper is genuinely cross-cutting (used by multiple unrelated domains) → place it in a shared `testfixtures` source set or a `testutils` sub-package of the closest common ancestor
- Never create a single grab-bag utility class for all test data across the entire project

### 9.2 Builder/Factory Pattern

Create test data factories with sensible defaults and overridable parameters:

**Groovy:**
```groovy
class InvoiceTestHelper {

    static Invoice buildInvoice(Map overrides = [:]) {
        def defaults = [
            amount: BigDecimal.TEN,
            currency: 'USD',
            status: 'PENDING',
            creationDate: LocalDate.of(2024, 1, 15)
        ]
        def props = defaults + overrides
        return new Invoice(props)
    }
}
```

**Kotlin:**
```kotlin
object InvoiceTestData {

    fun buildInvoice(
        amount: BigDecimal = BigDecimal.TEN,
        currency: String = "USD",
        status: String = "PENDING",
        creationDate: LocalDate = LocalDate.of(2024, 1, 15)
    ) = Invoice(
        amount = amount,
        currency = currency,
        status = status,
        creationDate = creationDate
    )
}
```

### 9.3 Minimize Test Data

Only set fields relevant to the test. Use defaults for everything else.

```kotlin
// ✅ Good — only sets what matters for this test
val receipt = buildSalesReceipt(taxAmount = BigDecimal("3.33"))

// ❌ Bad — unnecessary noise
val receipt = SalesReceipt(
    id = 123, name = "test", date = Date(), currency = "USD",
    customer = "John", status = "NEW", source = "STRIPE",
    taxAmount = BigDecimal("3.33")
)
```

---

## 10. Test Doubles (Mocking)

### 10.1 Types of Test Doubles

| Type | Purpose | Verify interactions? |
|------|---------|---------------------|
| **Stub** | Provides canned data to the test | No — never assert on stub calls |
| **Mock** | Verifies that interactions happened | Yes — assert calls, args, counts |
| **Spy** | Wraps real object, records interactions | Yes — partial verification |
| **Fake** | Lightweight real implementation (in-memory DB) | No — behaves like production |

**Key rule (Khorikov):** Never assert on stubs. If you verify that a stub was called, you're testing implementation details.

### 10.2 When to Mock vs When Not To

**Mock (communication-based testing) when:**
- Verifying interactions at **system boundaries** (message bus, external API, email sender)
- The interaction is part of the **observable contract** — callers expect this side effect
- The dependency is **out-of-process** and non-deterministic

**Don't mock when:**
- The dependency is an **in-process collaborator** whose real behavior is easy to use
- You're mocking to avoid wiring a small, deterministic collaborator — this adds fragility
- The mock replaces domain logic that should be tested as combined behavior

### 10.3 Managed vs Unmanaged Dependencies

| Type | Examples | In tests |
|------|----------|----------|
| **Managed** (you control) | Your database, your message queue | Use real instances in integration tests |
| **Unmanaged** (external) | 3rd-party APIs, payment gateways | Mock/stub in all tests |

### 10.4 Framework-Specific Syntax

**Spock:**
```groovy
// Stub — providing data
def searchService = Stub(SearchService) {
    find(_) >> [new TaxCode(id: '1')]
}

// Mock — verifying interaction at system boundary
def eventPublisher = Mock(DomainEventPublisher)
// ...
then:
1 * eventPublisher.publish({ it instanceof OrderPlaced })
```

**Kotlin (Mockito):**
```kotlin
// Stub
val searchService = mock<SearchService>()
`when`(searchService.find(any())).thenReturn(listOf(TaxCode("1")))

// Verify boundary interaction
verify(eventPublisher).publish(argThat { it is OrderPlaced })
```

**Kotlin (MockK):**
```kotlin
// Stub
val searchService = mockk<SearchService>()
every { searchService.find(any()) } returns listOf(TaxCode("1"))

// Verify
verify(exactly = 1) { eventPublisher.publish(match { it is OrderPlaced }) }
```

### 10.5 Mock Hygiene

- Keep mock count minimal (2-5 per test). 10+ mocks = production code design smell.
- Inject via constructor, not field reflection.
- Only mock **direct** dependencies of the class under test.
- Prefer stubs over mocks. Prefer output-based assertions over interaction assertions.

---

## 11. Unit Tests vs Integration Tests

| Aspect | Unit Test | Integration Test |
|--------|-----------|-----------------|
| **Scope** | Single class/method | Multiple components working together |
| **Dependencies** | All mocked/stubbed | Some or all real |
| **Database** | GORM `DataTest` (in-memory) or no DB | Real DB, embedded DB, or Testcontainers |
| **Speed** | Milliseconds | Seconds |
| **When to use** | Domain model, algorithms, calculations | DB queries, API integrations, multi-service flows |
| **File naming** | `*Spec.groovy` / `*Test.kt` | `*IntegrationTest.groovy` / `*IntegrationTest.kt` |
| **Multiple acts** | No — one act per test | Acceptable for workflow sequences |

### 11.1 Unit Test Rules
- Mock all external dependencies
- No network, no filesystem, no real database
- For Spock/GORM: use `DataTest` trait + `mockDomains()` for domain classes
- Should run in milliseconds

### 11.2 Integration Test Rules

**Spring Boot (Kotlin/Groovy):**
```kotlin
@SpringBootTest
@AutoConfigureMockMvc
class WebhookControllerIntegrationTest {

    @Autowired
    lateinit var mockMvc: MockMvc

    @Test
    fun `rejects unsupported webhook provider`() {
        mockMvc.perform(post("/webhook/unsupported-provider").content("data"))
            .andExpect(status().isOk)
    }
}
```

**Testcontainers (Kotlin):**
```kotlin
@SpringBootTest
@Testcontainers
class ReconciliationIntegrationTest {

    companion object {
        @Container
        val postgres = PostgreSQLContainer("postgres:15")
    }
}
```

**DbUnit (Groovy):**
```groovy
@SpringBootTest
@TestExecutionListeners([
    TransactionalTestExecutionListener,
    DependencyInjectionTestExecutionListener,
    DbUnitTestExecutionListener
])
class InvoicePaymentIntegrationTest {

    @Test
    @Transactional
    @DatabaseSetup('testData.xml')
    void 'creates payment with correct journal entries'() { ... }
}
```

**General rules:**
- Tag/name integration tests distinctly (`*IntegrationTest`) for CI separation
- Clean up test data in `@AfterEach` / `cleanup()` / use `@Transactional` rollback
- No flaky timing, no race conditions, no port conflicts
- Use real managed dependencies (DB, queues) where feasible; mock only unmanaged (3rd-party) dependencies
- Don't test logging content (except structured log fields when part of the contract)

### 11.3 SQL / Database Testing Checklist (Aniche)

When testing SQL queries or repository methods, verify:

- [ ] **Correctness** — query returns the right results for known test data
- [ ] **Null handling** — behavior when columns contain NULL values
- [ ] **Empty results** — what happens when no rows match
- [ ] **Ordering** — if ORDER BY is expected, verify sort order
- [ ] **Grouping/aggregation** — GROUP BY, COUNT, SUM return correct values
- [ ] **Joins** — left/inner/outer join produces correct result when related data is missing
- [ ] **Boundary values** — date ranges, numeric limits, string length limits
- [ ] **Pagination** — LIMIT/OFFSET work correctly at edges (first page, last page, beyond last)
- [ ] **Concurrent modifications** — if relevant, test optimistic locking / race conditions

**Infrastructure patterns:**
- Use small, deterministic datasets (not production dumps)
- Prefer `@Transactional` rollback or Testcontainers for isolation
- Co-locate test data files (SQL scripts, DbUnit XML) with the test class

---

## 12. Test Quality: FIRST Principles

| Letter | Principle | Meaning |
|--------|-----------|---------|
| **F** | Fast | Tests run in milliseconds. No sleeps, no real I/O in unit tests. |
| **I** | Independent | Tests don't depend on each other. No shared mutable state. No ordering. |
| **R** | Repeatable | Same result every time. No randomness, no external dependencies. |
| **S** | Self-validating | Tests either pass or fail. No manual inspection of output. |
| **T** | Timely | Write tests close to when you write the code. Don't accumulate test debt. |

---

## 13. Test Smells and Anti-Patterns

| Smell | Description | Fix |
|-------|-------------|-----|
| **Bloated Construction** | 20+ lines of setup before the actual test | Extract to test helper/builder in the same package |
| **Multiple Assertions** | Testing multiple unrelated behaviors in one test | Split into focused tests |
| **Irrelevant Details** | Test sets up data that doesn't affect the outcome | Remove noise, keep only relevant fields |
| **Misleading Organization** | Test name says one thing, body tests another | Rename or restructure |
| **Implicit Meaning** | Magic numbers, unclear variable names | Use descriptive names and constants |
| **Unnecessary Test Code** | Dead code, commented-out assertions | Remove it |
| **Missing Abstractions** | Complex inline setup repeated across tests | Extract builder/factory in the same package |
| **Generalized Assertions** | `assert result != null` instead of checking actual value | Assert specific expected values |
| **Brittle Tests** | Break when implementation changes but behavior doesn't | Test behavior, not implementation |
| **Slow Tests** | Unnecessary I/O, sleeps, or heavy setup | Mock external deps, use in-memory alternatives |
| **Distant Test Helpers** | Test data factories in a package far from the tests | Move helpers to the same package |
| **Testing Private Methods** | Direct access to private methods via reflection | Test through public API instead |
| **Asserting on Stubs** | Verifying that a stub was called | Stubs provide data; only mocks verify interactions |
| **Exposing State for Testing** | Adding getters/methods to production code solely for tests | Redesign to assert via observable behavior |
| **Leaking Domain Knowledge** | Test duplicates production algorithm to compute expected value | Use hardcoded expected values |
| **Sensitive Assertions** | Assertions depend on fragile ordering, formatting, or non-deterministic values | Assert on stable, meaningful properties; ignore irrelevant ordering |
| **Over-General Fixtures** | Shared `@BeforeEach` sets up data used by only some tests | Localize setup; each test creates only what it needs |

---

## 14. Design for Testability

### 14.1 Controllability and Observability

Good test design requires two properties in production code:

- **Controllability** — ability to set the system into a desired state for testing (via constructor injection, method parameters, configuration)
- **Observability** — ability to verify the outcome (via return values, state queries, emitted events)

If code is hard to test, it's usually because one of these is missing. Fix the design, don't work around it.

### 14.2 Hexagonal Architecture as Testability Pattern

Separate domain logic from infrastructure through ports and adapters:

```
[Domain Logic]  ←→  [Port (interface)]  ←→  [Adapter (infrastructure)]
     ↑                                              ↑
  Unit test                                    Integration test
  (easy, fast)                                 (real DB, real API)
```

- Domain logic: pure business rules, no framework dependencies → easy to unit test
- Ports: interfaces defining what domain needs → stub/mock in unit tests
- Adapters: implementations (DB, HTTP, messaging) → test with integration tests

### 14.3 Handling Hard-to-Test Constructs

| Problem | Solution |
|---------|----------|
| Static methods | Wrap in an injectable service |
| Singletons | Refactor to injected dependency |
| `new Date()` / `LocalDate.now()` | Inject a `Clock` or time provider |
| Complex conditional logic | Extract into a domain method; test separately |
| Void methods with side effects | Use spies or verify observable state changes |

---

## 15. Parameterized Tests (Data-Driven)

### 15.1 Spock — `where:` Block

```groovy
@Unroll
void 'converts #inputAmount #currency to smallest unit = #result'() {
    expect:
    SmallestUnitCurrencyConvertor.convertToSmallestUnit(inputAmount, currency) == result

    where:
    inputAmount    | currency || result
    null           | null     || null
    BigDecimal.TEN | null     || 10L
    BigDecimal.TEN | "JPY"   || 10L
    BigDecimal.TEN | "USD"   || 1000L
}
```

**Rules:**
- Use `||` to visually separate inputs from expected output
- Always add `@Unroll` so each case appears as a separate test in reports
- Include distinguishing parameters in the test name using `#paramName`
- Keep the table readable — if it has more than 5-6 columns, consider a helper method

### 15.2 Kotlin — `@ParameterizedTest`

```kotlin
@ParameterizedTest
@CsvSource(
    "10.0, USD, 1000",
    "10.0, JPY, 10",
    "10.0, '', 10"
)
fun `converts amount to smallest currency unit`(amount: BigDecimal, currency: String, expected: Long) {
    assertEquals(expected, convertToSmallestUnit(amount, currency))
}
```

For complex test data, use `@MethodSource`:

```kotlin
companion object {
    @JvmStatic
    fun taxCases() = listOf(
        Arguments.of(BigDecimal("15.0"), false, createTaxPseudoTaxCode()),
        Arguments.of(BigDecimal.ZERO, true, createTaxPseudoTaxCode()),
        Arguments.of(BigDecimal.ZERO, false, createNonPseudoTaxCode())
    )
}

@ParameterizedTest
@MethodSource("taxCases")
fun `resolves correct tax code`(percent: BigDecimal, applyGeneric: Boolean, expected: TaxCode) { ... }
```

---

## 16. When to Stop Testing

Testing has diminishing returns. Use these heuristics (Aniche):

- **Specification coverage** — all equivalence partitions and boundaries from the spec are covered
- **Structural coverage** — branch coverage is satisfactory (not 100% necessarily — see Decisions)
- **Mutation score** — if used, most non-equivalent mutants are killed
- **Diminishing returns** — new tests keep finding the same types of issues (saturation)
- **Risk assessment** — critical business logic has more tests; low-risk utility code has fewer

**Practical rule:** When you've covered all partitions, boundaries, error paths, and the code complexity matrix says this code type is tested appropriately — stop.

---

## 17. Checklist Before Submitting

Before pushing tests in a PR, verify:

- [ ] Every new/changed public method has test coverage
- [ ] Tests have descriptive names that explain the behavior
- [ ] Tests use proper structure (`given/when/then` for Spock, `Arrange/Act/Assert` for JUnit)
- [ ] Each test verifies one behavior
- [ ] Assertions target externally observable outcomes (not implementation details)
- [ ] Testing style matches the code type (output-based for domain logic, integration for controllers)
- [ ] Mock setup is minimal — only what's needed for the test
- [ ] Mocks are used only for system boundaries; stubs are never asserted on
- [ ] Test data helpers are co-located with tests (same package or sub-packages)
- [ ] Parameterized tests use `@Unroll` (Spock) or `@ParameterizedTest` (JUnit) with descriptive names
- [ ] No hard-coded sleeps or timing dependencies
- [ ] Tests pass independently and in any order
- [ ] No commented-out tests or assertions
- [ ] Boundary conditions covered (nulls, empty collections, edge values)
- [ ] Error/exception cases covered
- [ ] Tests run fast (total spec/class < 2 seconds for unit tests)
- [ ] Integration tests are clearly named (`*IntegrationTest`) and isolated

---

*Based on "Pragmatic Unit Testing in Java with JUnit" (3rd ed, Jeff Langr), "Unit Testing: Principles, Practices, and Patterns" (Vladimir Khorikov), and "Effective Software Testing: A Developer's Guide" (Maurício Aniche).*

---

## Revision History

| Date | Authors | Changes |
|------|---------|---------|
| 2026-03-10 | Alexey Sergeev | Initial draft: test structure, naming, what to test (Right-BICEP, CORRECT, ZOM), mocking guidelines, parameterized tests, test data, unit vs integration boundaries, FIRST principles, test smells, pre-submit checklist. Based on "Pragmatic Unit Testing in Java with JUnit" (3rd ed). |
| 2026-03-10 | Alexey Sergeev | Added Kotlin/JUnit 5 coverage (MockK, Mockito, Testcontainers, @ParameterizedTest, @Nested). Added test data co-location principle. Covered all repo test stacks: Spock/Groovy + JUnit 5/Kotlin. Removed project-specific TestUtils references. Added integration test patterns (SpringBootTest, DbUnit, Testcontainers). |
| 2026-03-10 | Alexey Sergeev | Added Four Pillars of a Good Test (Khorikov). Added code complexity matrix for deciding what to test. Added Humble Object pattern. Added three testing styles (output-based, state-based, communication-based) with preference order. Added managed vs unmanaged dependencies. Added anti-patterns: testing private methods, asserting on stubs, exposing state for testing, leaking domain knowledge. Extended mocking guidelines with "never assert on stubs" rule. |
| 2026-03-10 | Alexey Sergeev | Added systematic test case design: 7-step specification-based testing, equivalence partitioning, boundary value analysis (Aniche). Added SQL/database testing checklist. Added design for testability section (controllability/observability, hexagonal architecture, hard-to-test constructs). Added "when to stop testing" heuristics. Added test smells: sensitive assertions, over-general fixtures. |
