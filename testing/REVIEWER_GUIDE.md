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

## 2. First Pass: Completeness Check

### 2.1 Coverage Assessment

For every code change in the PR, ask:

| Changed code | Expected tests |
|-------------|---------------|
| New public method | At least one test per behavior, plus boundary cases |
| Modified method logic | Updated or new tests covering the changed behavior |
| Bug fix | At minimum: a test that reproduces the bug (would fail without the fix) |
| New class | Corresponding `*Spec.groovy` or `*Test.kt` test class |
| Deleted code | Corresponding tests removed or updated |
| New error handling | Tests that trigger the error path |

### 2.2 Right-BICEP Check

For each tested method, verify coverage across:

- [ ] **Right** — Happy path works?
- [ ] **Boundary** — Edge cases? (null, empty, zero, max, off-by-one)
- [ ] **Inverse** — Can the result be cross-checked?
- [ ] **Error** — Exception cases covered? (invalid input, missing dependencies, timeout)

### 2.3 ZOM — Zero, One, Many

For any method that handles collections or repeated operations:

- [ ] Zero case tested? (empty list, null list, no items)
- [ ] One case tested? (single element)
- [ ] Many case tested? (multiple elements, including edge counts)

### 2.4 Missing Test Triggers

Request additional tests when you see:

- `if/else` branches — each branch should be exercised
- `switch` / `when` statements — each case + default/else
- Null checks — test both null and non-null paths
- Exception throwing — test that exceptions are thrown with correct types/messages
- Boundary values in business logic (e.g., `amount > 0`, date ranges, limits)
- External service interactions — verify mocking covers success AND failure

---

## 3. Second Pass: Quality Review

### 3.1 Structure and Naming

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

### 3.2 Test Smells Checklist

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

### 3.3 Mock Usage Review

**Spock:**

| Check | ✅ Good | ❌ Problem |
|-------|--------|-----------|
| Mock vs Stub choice | `Mock` when verifying interaction; `Stub` when providing data | `Mock` everywhere "just in case" |
| Verification count | `1 * service.call(expectedArg)` with specific args | `_ * service.call(_)` with wildcards everywhere |
| Spy usage | Rare, for partial mocking of the class under test | Spy as default choice |

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

**Flag for discussion:** If a test requires more than ~8 mocks to set up, the production class likely violates Single Responsibility. Note this in the review.

### 3.4 Test Data Review

| Check | ✅ Good | ❌ Problem |
|-------|--------|-----------|
| Co-location | Test helpers in the same package as the tests | Helpers in a distant shared `utils` package |
| Minimal data | Only fields relevant to the test are set | Every field populated "just in case" |
| Builder pattern | `buildInvoice(amount = BigDecimal.TEN)` with defaults | Inline construction with 15 parameters |
| Data consistency | Test data matches realistic business scenarios | Impossible combinations (e.g., negative dates) |
| Reuse | Shared setup across tests uses `@BeforeEach` / `setup()` or helper | Same 10-line construction copy-pasted in every test |

---

## 4. Third Pass: Correctness Verification

### 4.1 Does the Test Actually Test Anything?

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
// ❌ This just verifies the stub returns what you told it to return
given:
service.findById(1) >> new User(name: 'test')

when:
def result = service.findById(1)

then:
result.name == 'test'  // Meaningless — you're testing your own stub
```

**Tautological test:**
```kotlin
// ❌ Always passes regardless of implementation
assertEquals(result, result)
```

### 4.2 Does the Test Fail for the Right Reason?

A good test should:
- **Pass** when the production code is correct
- **Fail** when the production code has a bug in the tested behavior
- **NOT fail** when unrelated code changes

Ask yourself: "If I broke the production method, would this test catch it?"

### 4.3 Are Exception Tests Specific?

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

## 5. Parameterized Test Review

### 5.1 Spock — `where:` Block

| Check | What to verify |
|-------|---------------|
| `@Unroll` present? | Every parameterized test must have `@Unroll` |
| Name includes params? | `'calculates tax for rate=#rate'` — so failures are identifiable |
| `\|\|` separator used? | Inputs separated from expected output with `\|\|` |
| Edge cases in table? | Table includes boundary values, not just happy paths |
| Table readability | Columns aligned, reasonable number (≤6 columns) |
| Coverage | Does the table cover all branches the code has? |

### 5.2 JUnit 5 — `@ParameterizedTest`

| Check | What to verify |
|-------|---------------|
| Annotation present? | `@ParameterizedTest` instead of `@Test` |
| Source appropriate? | `@CsvSource` for simple data, `@MethodSource` for complex objects |
| Name template? | `@ParameterizedTest(name = "...")` for clear failure messages |
| Edge cases included? | Boundary values, nulls, empty strings in the data set |
| `@MethodSource` companion? | Source method is `@JvmStatic` in `companion object` |

---

## 6. Integration Test Review

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

**Framework-specific checks:**

| Framework | What to verify |
|-----------|---------------|
| **SpringBootTest** | Appropriate context slicing? (prefer `@WebMvcTest`, `@DataJpaTest` over full context when possible) |
| **Testcontainers** | Container shared across tests? (`@Container` + `companion object` for reuse) |
| **DbUnit** | XML dataset files co-located with test class? Data realistic? |
| **MockMvc** | Response status AND body verified? Content-type checked? |

---

## 7. Review Comment Templates

Use these patterns for consistent, actionable review comments:

### Missing tests
> 🧪 **Missing test coverage.** The `else` branch on line X isn't covered. Please add a test for the case when `{condition}` is false.

### Test smell
> 🔍 **Test smell: {smell name}.** This test sets up 15 fields but only the `taxPercent` affects the outcome. Consider extracting a builder with sensible defaults in the same test package.

### Weak assertion
> ⚠️ **Weak assertion.** `result != null` doesn't verify the correct result was returned. Please assert on specific expected values.

### Brittle test
> 🔧 **Brittle test.** This verifies internal method call ordering, which couples the test to implementation. Consider asserting on the observable outcome instead.

### Test data location
> 📁 **Test helper location.** This test builder is in a shared `utils` package far from the tests. Consider moving it to the same package as the tests that use it.

### Good test (positive feedback)
> ✅ Good parameterized coverage — the `where:` table covers all the edge cases nicely.

---

## 8. Decision Framework: When to Push Back vs Accept

### Request changes (blocking):
- No tests for new/changed behavior
- Tests that don't actually verify anything meaningful
- Tests that will break on any refactoring (brittle)
- Ordering-dependent tests
- Test names that are misleading
- Missing error/boundary case coverage for critical business logic

### Suggest improvements (non-blocking):
- Minor naming improvements
- Extracting shared setup to helpers (in the same package)
- Adding one more edge case to a well-covered method
- Reducing irrelevant test data
- Better `@Unroll` naming or `@ParameterizedTest` name template

### Accept as-is:
- Tests cover the behavior adequately, even if style isn't perfect
- Minor style deviations from convention that don't hurt readability
- Tests for simple/low-risk code that cover the happy path

---

## 9. Quick Reference: Review Checklist

Copy this into your review workflow:

```
## Test Review
### Completeness
- [ ] All new/changed public methods have tests
- [ ] Happy path covered
- [ ] Boundary conditions covered (null, empty, zero, max)
- [ ] Error/exception paths covered
- [ ] Bug fixes include regression test

### Quality
- [ ] Tests have descriptive behavior-based names
- [ ] Proper structure (given/when/then for Spock, Arrange/Act/Assert for JUnit)
- [ ] One behavior per test
- [ ] Minimal mock setup (Stub vs Mock used correctly)
- [ ] Test data helpers co-located with tests (same package)
- [ ] No test smells (bloated, brittle, irrelevant details, distant helpers)

### Correctness
- [ ] Assertions verify meaningful expected values
- [ ] Tests would fail if the production code were broken
- [ ] Exception tests use specific types (not generic Exception)
- [ ] Parameterized tests have @Unroll (Spock) or @ParameterizedTest (JUnit)
- [ ] Parameterized tests cover edge cases, not just happy paths

### Resilience
- [ ] Tests are independent (no ordering dependency)
- [ ] Tests don't mock implementation details
- [ ] Tests won't break on harmless refactoring

### Integration Tests (if present)
- [ ] Named *IntegrationTest for CI separation
- [ ] Proper cleanup / @Transactional rollback
- [ ] No flaky timing or external dependencies
- [ ] Context slicing used where possible (avoid full @SpringBootTest if unnecessary)
```

---

*Based on "Pragmatic Unit Testing in Java with JUnit" (3rd ed, Jeff Langr).*

---

## Revision History

| Date | Authors | Changes |
|------|---------|---------|
| 2026-03-10 | Alexey Sergeev | Initial draft: 3-pass review structure (completeness, quality, correctness), test smell checklist with severities, mock usage review criteria, ZOM/Right-BICEP coverage checks, decision framework, review comment templates, copy-paste review checklist. Based on "Pragmatic Unit Testing in Java with JUnit" (3rd ed). |
| 2026-03-10 | Alexey Sergeev | Added Kotlin/JUnit 5 review criteria (MockK, Mockito, @ParameterizedTest, @Nested, Testcontainers, SpringBootTest context slicing). Added test data co-location principle. Removed project-specific TestUtils references. Extended integration test review section with framework-specific checks. |
