# Backend Developer Testing Guide

> How to write good unit and integration tests at Synder.
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

## 2. Test Stack & Conventions

| What | Tool |
|------|------|
| Test framework | **Spock** (Groovy) |
| Base class | `spock.lang.Specification` |
| Mocking | Spock built-in `Mock()`, `Stub()`, `Spy()` |
| Test file naming | `<ClassUnderTest>Spec.groovy` |
| Test file location | Same package in `src/test/groovy/` |
| Test data factories | `*TestUtils` classes in `com.cloudbusiness.utils` |
| Parameterized tests | `where:` block + `@Unroll` |
| GORM test support | `DataTest` trait + `mockDomains()` |

---

## 3. Test Structure

### 3.1 Use Given-When-Then (Spock BDD Blocks)

Every test should use Spock's labeled blocks to clearly separate phases:

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

### 3.2 One Behavior Per Test

Each test method should verify **one specific behavior**. If you need to assert two different outcomes of two different actions, write two tests.

❌ **Bad:**
```groovy
void 'process transaction'() {
    when:
    service.processTransaction(txn)

    then:
    // testing creation AND notification in one test
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

### 3.3 Test Naming

Use descriptive, readable names that describe behavior. Spock allows string literals:

```groovy
// ✅ Good — describes behavior
void 'returns null when user message belongs to different user'() { ... }
void 'applies tax pseudo code when tax percent is positive'() { ... }
void 'schedules SYNC operations'() { ... }

// ❌ Bad — describes implementation
void 'testGetUserMessageById'() { ... }
void 'test1'() { ... }
void 'should work correctly'() { ... }
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

| Type | When to use | Spock syntax |
|------|-------------|-------------|
| **Stub** | Replace behavior, return canned values | `Stub(ServiceClass)` or `>>` operator |
| **Mock** | Verify interactions (method was called) | `Mock(ServiceClass)` + `then:` verification |
| **Spy** | Partial mock — real object with some overrides | `Spy(ServiceClass)` |

### 5.2 Prefer Stubs Over Mocks

Use stubs when you only need to provide data. Use mocks only when **verifying an interaction is part of the behavior** you're testing.

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

### 5.3 Keep Mock Setup Minimal

If you need 20+ lines of mock setup, that's a design smell — the class under test likely has too many dependencies.

### 5.4 Inject via Constructor

Prefer constructor injection for the class under test, passing mocks directly:

```groovy
void setup() {
    organizationUserService = Mock(OrganizationUserService)
    userDataService = Mock(UserDataService)
    service = new UserMessageService(
            organizationUserService,
            userDataService,
            // ...
    )
}
```

---

## 6. Parameterized Tests (Data-Driven)

Use Spock's `where:` block with `@Unroll` for testing multiple inputs:

```groovy
@Unroll
void 'get line tax code when taxPercent=#taxPercent and applyGeneric=#applyGeneric'() {
    setup:
    def taxParams = buildTaxParams(taxPercent: taxPercent)
    def processor = Spy(USQuickBooksOnlineTaxProcessor) {
        shouldApplyGenericTax(taxParams) >> applyGeneric
    }

    expect:
    processor.getLineTaxCode(taxParams).id == result.id

    where:
    taxPercent | applyGeneric | taxAmount || result
    15.0       | false        | 15.0      || createTaxPseudoTaxCode()
    0.0        | true         | 15.0      || createTaxPseudoTaxCode()
    0.0        | false        | 0.0       || createNonPseudoTaxCode()
}
```

**Rules:**
- Use `||` to visually separate inputs from expected output
- Always `@Unroll` so each case appears as a separate test in reports
- Include distinguishing parameters in the test name using `#paramName`
- Keep the `where:` table readable — if it has more than 5-6 columns, consider a helper method

---

## 7. Test Data

### 7.1 Use TestUtils Factory Classes

Synder has established `*TestUtils` classes for building test entities:

```groovy
def socialToken = SocialTokenTestUtils.buildSocialToken()
def webhook = WebhookTestUtils.buildWebhook(socialToken: socialToken)
def settings = CompanySettingsTestUtils.buildSettings(taxEnabled: true)
```

**Rules:**
- Use existing `TestUtils` classes — don't duplicate factory logic in individual specs
- When you need a new entity factory, add it to (or create) the appropriate `TestUtils` class
- Use `Map params = [:]` pattern for overridable defaults
- TestUtils should produce valid, saveable domain objects with sensible defaults

### 7.2 Minimize Test Data

Only set fields relevant to the test. Use defaults for everything else.

```groovy
// ✅ Good — only sets what matters for this test
def txn = new SalesReceipt(taxList: new TaxList(taxes: [new TaxDetails(amount: 3.33)]))

// ❌ Bad — unnecessary noise
def txn = new SalesReceipt(
    id: 123, name: 'test', date: new Date(), currency: 'USD',
    customer: 'John', status: 'NEW', source: 'STRIPE',
    taxList: new TaxList(taxes: [new TaxDetails(amount: 3.33)])
)
```

---

## 8. Unit Tests vs Integration Tests

| Aspect | Unit Test | Integration Test |
|--------|-----------|-----------------|
| **Scope** | Single class/method | Multiple components working together |
| **Dependencies** | All mocked/stubbed | Some or all real |
| **Database** | GORM `DataTest` (in-memory) | Real DB or embedded |
| **Speed** | Milliseconds | Seconds |
| **When to use** | Business logic, calculations, mappings | DB queries, API integrations, multi-service flows |

### 8.1 Unit Test Rules
- Mock all external dependencies
- No network, no filesystem, no real database
- Use `DataTest` trait + `mockDomains()` for GORM domain classes
- Should run in milliseconds

### 8.2 Integration Test Rules
- Test real interactions between components
- Use embedded databases or Docker containers where needed
- Tag them (`@Tag('integration')`) so they can be run separately
- Clean up test data in `cleanup()` or `cleanupSpec()`

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
| **Bloated Construction** | 20+ lines of setup before the actual test | Extract to TestUtils or helper methods |
| **Multiple Assertions** | Testing multiple behaviors in one test | Split into focused tests |
| **Irrelevant Details** | Test sets up data that doesn't affect the outcome | Remove noise, keep only relevant fields |
| **Misleading Organization** | Test name says one thing, body tests another | Rename or restructure |
| **Implicit Meaning** | Magic numbers, unclear variable names | Use descriptive names and constants |
| **Unnecessary Test Code** | Dead code, commented-out assertions | Remove it |
| **Missing Abstractions** | Complex inline setup that repeats across tests | Extract builder/factory/helper |
| **Generalized Assertions** | `assert result != null` instead of checking actual value | Assert specific expected values |
| **Brittle Tests** | Break when implementation changes but behavior doesn't | Test behavior, not implementation |
| **Slow Tests** | Unnecessary I/O, sleeps, or heavy setup | Mock external deps, use in-memory alternatives |

---

## 11. Checklist Before Submitting

Before pushing tests in a PR, verify:

- [ ] Every new/changed public method has test coverage
- [ ] Tests have descriptive names that explain the behavior
- [ ] Tests use `given/when/then` or `setup/expect` blocks properly
- [ ] Each test verifies one behavior
- [ ] Mock setup is minimal — only what's needed for the test
- [ ] Test data uses existing `TestUtils` classes where available
- [ ] Parameterized tests use `@Unroll` with descriptive names
- [ ] No hard-coded sleeps or timing dependencies
- [ ] Tests pass independently and in any order
- [ ] No commented-out tests or assertions
- [ ] Boundary conditions covered (nulls, empty collections, edge values)
- [ ] Error/exception cases covered
- [ ] Tests run fast (total spec < 2 seconds for unit tests)

---

## 12. Quick Reference: Spock Blocks

```
given:   Setup — create objects, configure mocks
setup:   Alias for given:
when:    Exercise — call the method under test
then:    Verify — assertions and mock verifications
expect:  Combined when+then for simple cases
where:   Data table for parameterized tests
cleanup: Teardown — release resources
and:     Continue any block for readability
```

---

*Based on "Pragmatic Unit Testing in Java with JUnit" (3rd ed, Jeff Langr) adapted for Synder's Spock/Groovy stack.*

---

## Revision History

| Date | Authors | Changes |
|------|---------|---------|
| 2026-03-10 | Alexey Sergeev | Initial draft: test structure, naming, what to test (Right-BICEP, CORRECT, ZOM), mocking guidelines, parameterized tests, test data, unit vs integration boundaries, FIRST principles, test smells, pre-submit checklist. Based on "Pragmatic Unit Testing in Java with JUnit" (3rd ed), adapted for Synder's Spock/Groovy stack. |
