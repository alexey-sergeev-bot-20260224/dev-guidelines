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

## 2. Test Stack

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
- Test resources, helpers, and factories live **in the same package or sub-packages** as the tests that use them (see [Section 7: Test Data](#7-test-data))

---

## 3. Test Structure

### 3.1 Spock (Groovy): Given-When-Then

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

### 3.2 JUnit 5 (Kotlin): Arrange-Act-Assert

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

### 3.3 One Behavior Per Test

Each test method should verify **one specific behavior**. If you need to assert two different outcomes of two different actions, write two tests.

❌ **Bad:**
```groovy
void 'process transaction'() {
    when:
    service.processTransaction(txn)

    then:
    1 * repository.save(_)         // testing save
    1 * notificationService.notify(_)  // AND notification
    txn.status == 'PROCESSED'          // AND status change
}
```

✅ **Good:**
```groovy
void 'saves transaction when processed'() { ... }
void 'sends notification when transaction processed'() { ... }
void 'marks transaction as processed'() { ... }
```

### 3.4 Test Naming

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

## 4. What to Test

### 4.1 Right-BICEP (What to Test)

| Letter | Meaning | Ask yourself |
|--------|---------|--------------|
| **Right** | Are the right results returned? | Does the happy path work? |
| **B** | Boundary conditions | What happens at edges? (zero, one, max, empty, null) |
| **I** | Inverse relationships | Can you verify the result by reversing the operation? |
| **C** | Cross-check | Can you verify using an alternative method? |
| **E** | Error conditions | What should happen when things go wrong? |
| **P** | Performance characteristics | Does it complete within acceptable time? (rare in unit tests) |

### 4.2 CORRECT Boundary Conditions

| Letter | Boundary | Example |
|--------|----------|---------|
| **C** | Conformance | Does input conform to expected format? (email, UUID, date) |
| **O** | Ordering | Does order matter? (sorted results, sequence of operations) |
| **R** | Range | Within valid range? (negative amounts, overflow, min/max) |
| **R** | Reference | External dependencies present? (null references, missing associations) |
| **E** | Existence | Does the thing exist? (empty collection, null, missing record) |
| **C** | Cardinality | Zero, one, many — does each case work? |
| **T** | Time | Timing issues? (timeouts, ordering of events, time zones) |

### 4.3 ZOM — Zero, One, Many

For any collection or repeatable behavior:
1. **Zero** — empty collection, no items, null
2. **One** — single element (simplest case)
3. **Many** — multiple elements (general case + edge cases)

### 4.4 What NOT to Test

- Trivial getters/setters with no logic
- Framework behavior (don't test that Spring DI works)
- Third-party library internals
- Private methods directly — test them through public behavior
- The same behavior already covered by a higher-level integration test (unless testing specific edge cases)

---

## 5. Test Doubles (Mocking)

### 5.1 Choosing the Right Double

**Spock:**

| Type | When to use | Syntax |
|------|-------------|--------|
| **Stub** | Replace behavior, return canned values | `Stub(ServiceClass)` or `>>` operator |
| **Mock** | Verify interactions (method was called) | `Mock(ServiceClass)` + `then:` verification |
| **Spy** | Partial mock — real object with some overrides | `Spy(ServiceClass)` |

**Kotlin — Mockito:**

| Type | When to use | Syntax |
|------|-------------|--------|
| **Stub** | Return canned values | `mock<Service>()` + `` `when`(mock.method()).thenReturn(value) `` |
| **Mock** | Verify interactions | `mock<Service>()` + `verify(mock).method()` |
| **Argument capture** | Inspect call arguments | `argumentCaptor<Type>()` + `verify(mock).method(captor.capture())` |

**Kotlin — MockK:**

| Type | When to use | Syntax |
|------|-------------|--------|
| **Stub** | Return canned values | `mockk<Service>()` + `every { mock.method() } returns value` |
| **Mock** | Verify interactions | `verify { mock.method() }` |
| **Relaxed mock** | Auto-return defaults | `mockk<Service>(relaxed = true)` |

### 5.2 Prefer Stubs Over Mocks

Use stubs when you only need to provide data. Use mocks only when **verifying an interaction is part of the behavior** you're testing.

**Spock:**
```groovy
// Stub — just providing data
def searchService = Stub(SearchService) {
    find(_) >> [new TaxCode(id: '1')]
}

// Mock — we NEED to verify this was called
def notificationService = Mock(NotificationService)
// ...
then:
1 * notificationService.sendAlert(userId, _)
```

**Kotlin (Mockito):**
```kotlin
// Stub
val searchService = mock<SearchService>()
`when`(searchService.find(any())).thenReturn(listOf(TaxCode("1")))

// Verify
verify(notificationService).sendAlert(eq(userId), any())
```

### 5.3 Keep Mock Setup Minimal

If you need 20+ lines of mock setup, that's a design smell — the class under test likely has too many dependencies.

### 5.4 Inject via Constructor

Prefer constructor injection for the class under test, passing mocks directly:

**Spock:**
```groovy
void setup() {
    repository = Mock(OrderRepository)
    notifier = Mock(NotificationService)
    service = new OrderService(repository, notifier)
}
```

**Kotlin:**
```kotlin
@BeforeEach
fun setup() {
    repository = mock<OrderRepository>()
    notifier = mock<NotificationService>()
    service = OrderService(repository, notifier)
}
```

---

## 6. Parameterized Tests (Data-Driven)

### 6.1 Spock — `where:` Block

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

### 6.2 Kotlin — `@ParameterizedTest`

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

## 7. Test Data

### 7.1 Co-Location Principle

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

### 7.2 Builder/Factory Pattern

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

### 7.3 Minimize Test Data

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

## 8. Unit Tests vs Integration Tests

| Aspect | Unit Test | Integration Test |
|--------|-----------|-----------------|
| **Scope** | Single class/method | Multiple components working together |
| **Dependencies** | All mocked/stubbed | Some or all real |
| **Database** | GORM `DataTest` (in-memory) or no DB | Real DB, embedded DB, or Testcontainers |
| **Speed** | Milliseconds | Seconds |
| **When to use** | Business logic, calculations, mappings | DB queries, API integrations, multi-service flows |
| **File naming** | `*Spec.groovy` / `*Test.kt` | `*IntegrationTest.groovy` / `*IntegrationTest.kt` |

### 8.1 Unit Test Rules
- Mock all external dependencies
- No network, no filesystem, no real database
- For Spock/GORM: use `DataTest` trait + `mockDomains()` for domain classes
- Should run in milliseconds

### 8.2 Integration Test Rules

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

    // ...
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
- Keep integration tests focused — don't re-test business logic already covered by unit tests

---

## 9. Test Quality: FIRST Principles

| Letter | Principle | Meaning |
|--------|-----------|---------|
| **F** | Fast | Tests run in milliseconds. No sleeps, no real I/O in unit tests. |
| **I** | Independent | Tests don't depend on each other. No shared mutable state. No ordering. |
| **R** | Repeatable | Same result every time. No randomness, no external dependencies. |
| **S** | Self-validating | Tests either pass or fail. No manual inspection of output. |
| **T** | Timely | Write tests close to when you write the code. Don't accumulate test debt. |

---

## 10. Test Smells to Avoid

| Smell | Description | Fix |
|-------|-------------|-----|
| **Bloated Construction** | 20+ lines of setup before the actual test | Extract to test helper/builder in the same package |
| **Multiple Assertions** | Testing multiple behaviors in one test | Split into focused tests |
| **Irrelevant Details** | Test sets up data that doesn't affect the outcome | Remove noise, keep only relevant fields |
| **Misleading Organization** | Test name says one thing, body tests another | Rename or restructure |
| **Implicit Meaning** | Magic numbers, unclear variable names | Use descriptive names and constants |
| **Unnecessary Test Code** | Dead code, commented-out assertions | Remove it |
| **Missing Abstractions** | Complex inline setup that repeats across tests | Extract builder/factory in the same package |
| **Generalized Assertions** | `assert result != null` instead of checking actual value | Assert specific expected values |
| **Brittle Tests** | Break when implementation changes but behavior doesn't | Test behavior, not implementation |
| **Slow Tests** | Unnecessary I/O, sleeps, or heavy setup | Mock external deps, use in-memory alternatives |
| **Distant Test Helpers** | Test data factories in a package far from the tests | Move helpers to the same package as the tests |

---

## 11. Checklist Before Submitting

Before pushing tests in a PR, verify:

- [ ] Every new/changed public method has test coverage
- [ ] Tests have descriptive names that explain the behavior
- [ ] Tests use proper structure (`given/when/then` for Spock, `Arrange/Act/Assert` for JUnit)
- [ ] Each test verifies one behavior
- [ ] Mock setup is minimal — only what's needed for the test
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

*Based on "Pragmatic Unit Testing in Java with JUnit" (3rd ed, Jeff Langr).*

---

## Revision History

| Date | Authors | Changes |
|------|---------|---------|
| 2026-03-10 | Alexey Sergeev | Initial draft: test structure, naming, what to test (Right-BICEP, CORRECT, ZOM), mocking guidelines, parameterized tests, test data, unit vs integration boundaries, FIRST principles, test smells, pre-submit checklist. Based on "Pragmatic Unit Testing in Java with JUnit" (3rd ed). |
| 2026-03-10 | Alexey Sergeev | Added Kotlin/JUnit 5 coverage (MockK, Mockito, Testcontainers, @ParameterizedTest, @Nested). Added test data co-location principle. Covered all repo test stacks: Spock/Groovy + JUnit 5/Kotlin. Removed project-specific TestUtils references. Added integration test patterns (SpringBootTest, DbUnit, Testcontainers). |
