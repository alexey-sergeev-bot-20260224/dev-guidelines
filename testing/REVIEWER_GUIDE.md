# Backend Code Reviewer Testing Guide

> How to evaluate unit and integration tests in PRs.
> For human reviewers and AI agents alike.

---

## 1. Reviewer's Mindset

Your job is NOT to verify the code compiles or passes CI. Your job is to evaluate:

1. **Completeness** — are the right things tested?
2. **Quality** — are the tests well-written and maintainable?
3. **Correctness** — do the tests actually verify what they claim to?
4. **Resilience** — will these tests survive refactoring without false failures?

A PR with code changes but no test changes should raise a red flag. Ask: "Should this have tests?"

---

## 2. The Four Pillars Evaluation

Evaluate every test against the four pillars (Khorikov). A test that scores poorly on any pillar should be flagged:

| Pillar | What to check | Red flag |
|--------|--------------|----------|
| **Protection against regressions** | Does the test fail if the production behavior breaks? | Test asserts trivial things (`!= null`) or skips important paths |
| **Resistance to refactoring** | Would a harmless internal refactor break this test? | Test verifies internal call sequences, method names, or execution order |
| **Fast feedback** | Does the test run quickly? | Unit test does real I/O, sleeps, or starts Spring context |
| **Maintainability** | Is the test easy to read and update? | 30+ lines of setup, magic numbers, unclear naming |

### Test Accuracy: False Positives vs False Negatives

| Problem | Definition | Common cause | Impact |
|---------|-----------|--------------|--------|
| **False positive** | Test fails when code is correct | Asserting implementation details, brittle mocking | Erodes trust in tests → developers ignore failures |
| **False negative** | Test passes when code is broken | Weak/missing assertions, mocking away important behavior | False confidence → bugs reach production |

**As a reviewer, watch for both:**
- False positives: "Would a rename/refactor break this test even though behavior is unchanged?"
- False negatives: "If I introduced a bug in this method, would this test catch it?"

---

## 3. First Pass: Completeness Check

### 3.1 Coverage Assessment

For every code change in the PR, ask:

| Changed code | Expected tests |
|-------------|---------------|
| New public method | At least one test per behavior, plus boundary cases |
| Modified method logic | Updated or new tests covering the changed behavior |
| Bug fix | At minimum: a test that reproduces the bug (would fail without the fix) |
| New class | Corresponding `*Spec.groovy` or `*Test.kt` test class |
| Deleted code | Corresponding tests removed or updated |
| New error handling | Tests that trigger the error path |

### 3.2 Code Type vs Test Type

Verify the right kind of test was written for the code being changed:

| Code type | Expected test approach |
|-----------|----------------------|
| **Domain model / algorithms** | Unit tests (output-based or state-based). High test density. |
| **Controllers / orchestrators** | Fewer tests; integration tests or tests with real in-process collaborators. |
| **Trivial code** (getters, setters, delegation) | No tests needed. Flag if someone wrote tests for trivial code. |
| **Overcomplicated code** (complex + many deps) | Should be refactored first. Flag the design, not just the tests. |

### 3.3 Right-BICEP Check

For each tested method, verify coverage across:

- [ ] **Right** — Happy path works?
- [ ] **Boundary** — Edge cases? (null, empty, zero, max, off-by-one)
- [ ] **Inverse** — Can the result be cross-checked?
- [ ] **Error** — Exception cases covered? (invalid input, missing dependencies, timeout)

### 3.4 ZOM — Zero, One, Many

For any method that handles collections or repeated operations:

- [ ] Zero case tested? (empty list, null list, no items)
- [ ] One case tested? (single element)
- [ ] Many case tested? (multiple elements, including edge counts)

### 3.5 Specification-Based Coverage (Aniche)

For critical methods, verify that tests follow systematic test case design:

- [ ] **Equivalence partitions identified?** — Inputs grouped by expected behavior type
- [ ] **Boundary values tested?** — On-point and off-point for each boundary
- [ ] **All partitions covered?** — At least one test per partition
- [ ] **Combinations considered?** — When multiple inputs interact, key combinations are tested
- [ ] **Parameterized tests used?** — When many inputs share the same test skeleton

**Example review check:** If a method has a condition `if (amount >= 100)`, expect tests for:
- Amount = 100 (on-point)
- Amount = 99 (off-point)
- Amount = 500 (clearly in range)
- Amount = 0 (edge)

### 3.6 Missing Test Triggers

Request additional tests when you see:

- `if/else` branches — each branch should be exercised
- `switch` / `when` statements — each case + default/else
- Null checks — test both null and non-null paths
- Exception throwing — test that exceptions are thrown with correct types/messages
- Boundary values in business logic (e.g., `amount > 0`, date ranges, limits)
- External service interactions — verify mocking covers success AND failure

---

## 4. Second Pass: Quality Review

### 4.1 Testing Style Assessment

Check that the right testing style is used:

| Style | When it's correct | When it's wrong |
|-------|------------------|-----------------|
| **Output-based** (assert return value) | Pure functions, calculations, mappers | — (always preferred when applicable) |
| **State-based** (assert state after action) | Domain entities, stateful services | When output-based would suffice |
| **Communication-based** (verify mock interactions) | System boundaries, external integrations | For internal collaborators that could be used as real objects |

**Key rule:** If the test uses communication-based style (mock verification) for something that could be tested output-based, request a change.

### 4.2 Structure and Naming

**Spock (Groovy):**

| Check | ✅ Accept | ❌ Request changes |
|-------|----------|-------------------|
| Test class name | `<ClassUnderTest>Spec` | Random or generic name |
| Test method name | Describes behavior: `'returns null when user not found'` | `'test1'`, `'should work'`, `'testGetById'` |
| Block usage | Proper `given/when/then` or `setup/expect` | Unlabeled code blocks, no structure |
| Single responsibility | One behavior per test | Multiple unrelated assertions |

**JUnit 5 (Kotlin):**

| Check | ✅ Accept | ❌ Request changes |
|-------|----------|-------------------|
| Test class name | `<ClassUnderTest>Test` | Random or generic name |
| Test method name | Backtick with behavior: `` `rejects unsupported provider` `` | `testProcess`, `test1` |
| Structure | Clear Arrange/Act/Assert or `@Nested` groups | Interleaved setup and assertions |
| Single responsibility | One behavior per test | Multiple unrelated assertions |

### 4.3 Test Smells Checklist

Score each test for these smells. Any present = request changes:

| Smell | What to look for | Severity |
|-------|-----------------|----------|
| **Bloated Construction** | Setup section > 15-20 lines | Medium — suggest extracting to a test helper in the same package |
| **Multiple Assertions** | `then:` / assert block tests unrelated behaviors | High — split into separate tests |
| **Irrelevant Details** | Fields set in test data that don't affect the assertion | Low — suggest cleanup |
| **Misleading Organization** | Test name doesn't match what's actually verified | High — rename required |
| **Implicit Meaning** | Magic numbers without explanation: `assert count == 7` | Medium — use named constants or comments |
| **Missing Abstractions** | Same complex setup copy-pasted across multiple tests | Medium — suggest helper in the same package |
| **Generalized Assertions** | `result != null`, `result.size() > 0` instead of exact values | High — assert specific expected values |
| **Brittle Mocking** | Verifying internal method calls that aren't part of the contract | High — test behavior, not implementation |
| **Unnecessary Code** | Commented-out tests, dead assertions, unused variables | Low — request cleanup |
| **Ordering Dependency** | Tests that fail when run individually or in different order | Critical — must fix |
| **Distant Test Helpers** | Test data builders/factories in a package far from the tests | Medium — should be co-located |
| **Sensitive Assertions** | Assertions depend on fragile ordering, formatting, or timestamps | Medium — assert on stable properties |
| **Over-General Fixtures** | Shared `@BeforeEach`/`setup()` sets up data only some tests use | Medium — localize setup per test |

### 4.4 Anti-Patterns (Khorikov)

These are deeper design issues in tests. Flag and explain:

| Anti-pattern | What it looks like | Why it's bad | Suggestion |
|-------------|-------------------|-------------|------------|
| **Testing private methods** | Reflection or package-private access to test internals | Couples test to implementation; misses public behavior | Test through the public API |
| **Asserting on stubs** | `verify(stub).getData()` / `1 * stub.getData()` | Stubs provide data, they're not part of the contract | Remove the verification; stubs are not mocks |
| **Exposing state for testing** | Added `getInternalState()` to production code only for tests | Pollutes production API; test is testing internals | Redesign to assert via observable output |
| **Leaking domain knowledge** | Test recalculates expected value using production logic | Test can't catch bugs — it'll compute the same wrong answer | Use hardcoded expected values |
| **Mocking concrete classes** | `mock(ConcreteService.class)` instead of interface | Fragile to implementation changes in the concrete class | Mock interfaces; only mock types you own |

### 4.5 Mock Usage Review

**Spock (Kapelonis):**

| Check | ✅ Good | ❌ Problem |
|-------|--------|-----------|
| Mock vs Stub choice | `Mock` for boundary interactions; `Stub` for data | `Mock` everywhere "just in case" |
| Verification count | `1 * service.call(expectedArg)` with specific args | `_ * service.call(_)` with wildcards everywhere |
| Spy usage | Rare, last resort for legacy code | Spy as default choice (implies design problem) |
| Stub assertions | Never verified | `1 * stub.getData()` — asserting on a stub |
| Sequential stubs | `>>>` with list for ordered responses | Hard-coded when sequence would be clearer |
| Argument matching | Specific args where possible, `_` only for irrelevant params | All `_` wildcards when args are known |
| Compact init | `Stub(Service) { method() >> value }` for simple setups | 20 separate stub instruction lines |

**Kotlin — Mockito:**

| Check | ✅ Good | ❌ Problem |
|-------|--------|-----------|
| Stubbing | `` `when`(mock.method()).thenReturn(value) `` for specific inputs | `thenReturn` with `any()` when specific args are known |
| Verification | `verify(mock).method(expectedArg)` | `verify(mock, atLeastOnce())` without checking args |
| Argument matching | `eq()`, `argThat()`, or `argumentCaptor` for complex args | All `any()` wildcards |

**Kotlin — MockK:**

| Check | ✅ Good | ❌ Problem |
|-------|--------|-----------|
| Stubbing | `every { mock.method(expectedArg) } returns value` | Relaxed mocks everywhere without justification |
| Verification | `verify(exactly = 1) { mock.method(expectedArg) }` | `verify { mock.method(any()) }` with all wildcards |

**Common checks (all frameworks):**

| Check | ✅ Good | ❌ Problem |
|-------|--------|-----------|
| Mock count | 2-5 mocks per test | 10+ mocks = production code design smell |
| Injection | Constructor injection of mocks | Field injection with reflection hacks |
| Scope | Only mocks direct dependencies of the class under test | Mocking deep internal classes |
| Managed deps | Real instances in integration tests | Mocking your own DB/queue in integration tests |
| Unmanaged deps | Mocked in all tests | Using real 3rd-party APIs in automated tests |

**Flag for discussion:** If a test requires more than ~8 mocks, the production class likely violates Single Responsibility. Note this in the review as a design concern, not just a test concern.

### 4.6 Test Data Review

| Check | ✅ Good | ❌ Problem |
|-------|--------|-----------|
| Co-location | Test helpers in the same package as the tests | Helpers in a distant shared `utils` package |
| Minimal data | Only fields relevant to the test are set | Every field populated "just in case" |
| Builder pattern | `buildInvoice(amount = BigDecimal.TEN)` with defaults | Inline construction with 15 parameters |
| Data consistency | Test data matches realistic business scenarios | Impossible combinations (e.g., negative dates) |
| Hardcoded expected values | `assertEquals(35.50, result)` | Test computes expected value using production logic |

---

## 5. Third Pass: Correctness Verification

### 5.1 Does the Test Actually Test Anything?

Watch for these traps:

**Empty verification:**
```groovy
// ❌ Spock — tests nothing
void 'processes transaction'() {
    when:
    service.process(txn)

    then:
    noExceptionThrown()  // Only acceptable if "no exception" IS the behavior
}
```

```kotlin
// ❌ Kotlin — tests nothing
@Test
fun `processes transaction`() {
    service.process(txn)  // no assertions at all
}
```

**Assertion on the mock, not the result:**
```groovy
// ❌ Testing your own stub setup
given:
service.findById(1) >> new User(name: 'test')

when:
def result = service.findById(1)

then:
result.name == 'test'  // Meaningless — you're testing your own stub
```

**Tautological test:**
```kotlin
// ❌ Always passes
assertEquals(result, result)
```

**Leaking domain knowledge:**
```kotlin
// ❌ Test recalculates using production logic — can't catch bugs
val expected = amount * taxRate / 100  // same formula as production code
assertEquals(expected, calculator.computeTax(amount, taxRate))

// ✅ Hardcoded expected value
assertEquals(BigDecimal("7.50"), calculator.computeTax(BigDecimal("100"), BigDecimal("7.5")))
```

### 5.2 Does the Test Fail for the Right Reason?

A good test should:
- **Pass** when the production code is correct
- **Fail** when the production code has a bug in the tested behavior
- **NOT fail** when unrelated code changes (resistance to refactoring)

Ask yourself: "If I broke the production method, would this test catch it?"

### 5.3 Are Exception Tests Specific?

**Spock:**
```groovy
// ❌ Too broad
then:
thrown(Exception)

// ✅ Specific
then:
def e = thrown(IllegalArgumentException)
e.message.contains('userId must not be null')
```

**Kotlin (JUnit 5):**
```kotlin
// ❌ Too broad
assertThrows<Exception> { service.process(null) }

// ✅ Specific
val ex = assertThrows<IllegalArgumentException> { service.process(null) }
assertTrue(ex.message!!.contains("userId must not be null"))
```

**Kotlin (kotlin.test):**
```kotlin
// ✅ Also good
assertFailsWith<IllegalArgumentException>("userId must not be null") {
    service.process(null)
}
```

---

## 6. Parameterized Test Review

### 6.1 Spock — `where:` Block

| Check | What to verify |
|-------|---------------|
| `@Unroll` present? | Every parameterized test must have `@Unroll` |
| Name includes params? | `'calculates tax for rate=#rate'` — so failures are identifiable |
| `\|\|` separator used? | Inputs separated from expected output with `\|\|` |
| Edge cases in table? | Table includes boundary values, not just happy paths |
| Table readability | Columns aligned, reasonable number (≤6 columns) |
| Coverage | Does the table cover all branches the code has? |
| Minimum 2 columns | Data tables need at least 2 columns (use `_` filler if single-column) |
| Mock + where combo | Mock verification counts can use `where:` parameters (e.g., `alarm * control.activateAlarm()`) |

### 6.2 JUnit 5 — `@ParameterizedTest`

| Check | What to verify |
|-------|---------------|
| Annotation present? | `@ParameterizedTest` instead of `@Test` |
| Source appropriate? | `@CsvSource` for simple data, `@MethodSource` for complex objects |
| Name template? | `@ParameterizedTest(name = "...")` for clear failure messages |
| Edge cases included? | Boundary values, nulls, empty strings in the data set |
| `@MethodSource` companion? | Source method is `@JvmStatic` in `companion object` |

---

## 7. Integration Test Review

When a PR includes integration tests:

| Check | Requirement |
|-------|-------------|
| Necessity | Is integration test needed, or would a unit test suffice? |
| Naming | Named `*IntegrationTest` for CI separation |
| Isolation | Uses embedded DB / Testcontainers / `@Transactional` rollback, not shared dev env |
| Cleanup | `@AfterEach` / `cleanup()` removes test data, or `@Transactional` handles rollback |
| Speed | Acceptable execution time (< 30s per test ideally) |
| Determinism | No flaky timing, no race conditions, no port conflicts |
| Data setup | DbUnit XML / SQL scripts / programmatic setup — verify data matches realistic scenarios |
| Dependencies | Managed deps (DB, queue) use real instances; unmanaged deps (3rd-party) are mocked |
| Multiple acts | Acceptable in integration tests for workflow sequences — don't flag this |

**SQL/Database test checks (Aniche):**

| Check | Requirement |
|-------|-------------|
| Result correctness | Tests verify actual query results, not just "no exception" |
| Null handling | Tests include rows with NULL values in relevant columns |
| Empty results | At least one test for "no rows match" scenario |
| Joins | If query has joins, test with missing related data |
| Boundaries | Date ranges, numeric limits, pagination edges tested |
| Test data | Small, deterministic datasets co-located with tests |

**Framework-specific checks:**

| Framework | What to verify |
|-----------|---------------|
| **SpringBootTest** | Appropriate context slicing? (prefer `@WebMvcTest`, `@DataJpaTest` over full context when possible) |
| **Testcontainers** | Container shared across tests? (`@Container` + `companion object` for reuse) |
| **DbUnit** | XML dataset files co-located with test class? Data realistic? |
| **MockMvc** | Response status AND body verified? Content-type checked? |

---

## 8. Review Comment Templates

Use these patterns for consistent, actionable review comments:

### Missing tests
> 🧪 **Missing test coverage.** The `else` branch on line X isn't covered. Please add a test for the case when `{condition}` is false.

### Test smell
> 🔍 **Test smell: {smell name}.** This test sets up 15 fields but only the `taxPercent` affects the outcome. Consider extracting a builder with sensible defaults in the same test package.

### Weak assertion
> ⚠️ **Weak assertion.** `result != null` doesn't verify the correct result was returned. Please assert on specific expected values.

### Brittle test
> 🔧 **Brittle test (resistance to refactoring).** This verifies internal method call ordering, which couples the test to implementation. Consider asserting on the observable outcome instead.

### Wrong testing style
> 🎯 **Testing style.** This domain calculation is tested via mock interactions (communication-based), but output-based testing would be simpler and more resilient. Consider asserting the return value directly.

### Asserting on stub
> ⚠️ **Asserting on a stub.** Line X verifies that a stub method was called. Stubs provide data — they shouldn't be verified. Remove the interaction check or convert to a proper mock if the call is part of the contract.

### Leaking domain knowledge
> 🧠 **Leaking domain knowledge.** The test computes the expected value using the same formula as production code. Use a hardcoded expected value instead — otherwise the test can't catch formula bugs.

### Test data location
> 📁 **Test helper location.** This test builder is in a shared `utils` package far from the tests. Consider moving it to the same package as the tests that use it.

### Design concern (too many mocks)
> 🏗️ **Design concern.** This test requires 12 mocks to set up. This suggests the production class has too many responsibilities. Consider refactoring to separate domain logic from orchestration (Humble Object pattern).

### Missing boundary tests
> 📐 **Missing boundary tests.** The condition on line X (`amount >= 100`) has tests for values well above and below, but no test for the boundary itself (100) or the off-point (99).

### Testability concern
> 🔧 **Testability.** This method calls `LocalDate.now()` directly, making it hard to test date-dependent logic. Consider injecting a `Clock` to make the behavior controllable.

### Good test (positive feedback)
> ✅ Good parameterized coverage — the `where:` table covers all the edge cases nicely.

---

## 9. Decision Framework: When to Push Back vs Accept

### Request changes (blocking):
- No tests for new/changed behavior
- Tests that don't actually verify anything meaningful (false negatives)
- Tests that will break on any refactoring — brittle (false positives)
- Ordering-dependent tests
- Test names that are misleading
- Missing error/boundary case coverage for critical business logic
- Asserting on stubs
- Leaking domain knowledge in expected value computation
- Testing private methods directly

### Suggest improvements (non-blocking):
- Minor naming improvements
- Extracting shared setup to helpers (in the same package)
- Adding one more edge case to a well-covered method
- Reducing irrelevant test data
- Better `@Unroll` naming or `@ParameterizedTest` name template
- Switching from communication-based to output-based style where possible

### Accept as-is:
- Tests cover the behavior adequately, even if style isn't perfect
- Minor style deviations from convention that don't hurt readability
- Tests for simple/low-risk code that cover the happy path
- Integration tests with multiple act-assert phases for workflow testing

---

## 10. Quick Reference: Review Checklist

Copy this into your review workflow:

```
## Test Review

### Four Pillars
- [ ] Protection against regressions: tests fail when behavior breaks
- [ ] Resistance to refactoring: tests survive harmless internal changes
- [ ] Fast feedback: unit tests run in milliseconds
- [ ] Maintainability: tests are readable and have minimal setup

### Completeness
- [ ] All new/changed public methods have tests
- [ ] Right test type for code type (unit for domain, integration for controllers)
- [ ] Happy path covered
- [ ] Boundary conditions covered (null, empty, zero, max)
- [ ] Error/exception paths covered
- [ ] Bug fixes include regression test

### Quality
- [ ] Tests have descriptive behavior-based names
- [ ] Proper structure (given/when/then for Spock, Arrange/Act/Assert for JUnit)
- [ ] One behavior per test
- [ ] Correct testing style (output-based preferred over communication-based)
- [ ] Minimal mock setup (Stub vs Mock used correctly)
- [ ] Never asserting on stubs
- [ ] Test data helpers co-located with tests (same package)
- [ ] No test smells (bloated, brittle, irrelevant details, distant helpers)

### Correctness
- [ ] Assertions verify meaningful expected values (not `!= null`)
- [ ] Expected values are hardcoded (no domain knowledge leaking)
- [ ] Tests would fail if the production code were broken
- [ ] Exception tests use specific types (not generic Exception)
- [ ] Parameterized tests have @Unroll (Spock) or @ParameterizedTest (JUnit)
- [ ] Parameterized tests cover edge cases, not just happy paths

### Resilience
- [ ] Tests are independent (no ordering dependency)
- [ ] Tests don't verify implementation details (internal calls, method order)
- [ ] Tests won't break on harmless refactoring

### Integration Tests (if present)
- [ ] Named *IntegrationTest for CI separation
- [ ] Managed deps use real instances; unmanaged deps are mocked
- [ ] Proper cleanup / @Transactional rollback
- [ ] No flaky timing or external dependencies
- [ ] Context slicing used where possible
```

---

*Based on "Pragmatic Unit Testing in Java with JUnit" (3rd ed, Jeff Langr), "Unit Testing: Principles, Practices, and Patterns" (Vladimir Khorikov), "Effective Software Testing: A Developer's Guide" (Maurício Aniche), and "Java Testing with Spock" (Konstantinos Kapelonis).*

---

## Revision History

| Date | Authors | Changes |
|------|---------|---------|
| 2026-03-10 | Alexey Sergeev | Initial draft: 3-pass review structure (completeness, quality, correctness), test smell checklist with severities, mock usage review criteria, ZOM/Right-BICEP coverage checks, decision framework, review comment templates, copy-paste review checklist. Based on "Pragmatic Unit Testing in Java with JUnit" (3rd ed). |
| 2026-03-10 | Alexey Sergeev | Added Kotlin/JUnit 5 review criteria (MockK, Mockito, @ParameterizedTest, @Nested, Testcontainers, SpringBootTest context slicing). Added test data co-location principle. Removed project-specific TestUtils references. Extended integration test review section with framework-specific checks. |
| 2026-03-10 | Alexey Sergeev | Added Four Pillars evaluation framework (Khorikov). Added false positives vs false negatives analysis. Added code type vs test type matrix. Added anti-patterns: testing private methods, asserting on stubs, exposing state for testing, leaking domain knowledge, mocking concrete classes. Added testing style assessment (output-based > state-based > communication-based). Added review comment templates for new patterns. Extended review checklist with Four Pillars section. |
| 2026-03-10 | Alexey Sergeev | Added specification-based coverage review (equivalence partitions, boundary values). Added SQL/database testing review checklist (Aniche). Added test smells: sensitive assertions, over-general fixtures. Added review comment templates: missing boundary tests, testability concern. |
| 2026-03-10 | Alexey Sergeev | Added Spock-specific review criteria (Kapelonis): advanced stubbing patterns (sequential >>>, argument matching, compact init), Spy as last resort, parameterized test data table rules (2-column min, mock+where combo). |
