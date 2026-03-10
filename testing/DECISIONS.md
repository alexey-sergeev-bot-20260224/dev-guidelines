# Testing Guidelines — Decisions & Excluded Approaches

> This document records what was considered, removed, or intentionally excluded from the testing guides, and why.
> Read this before proposing changes to the guides — the topic may have already been discussed.

---

## Decisions Made

### 1. Test Data Co-Location Over Shared Utils Package

**Decision:** Test helpers, builders, and factories must live in the same package (or sub-packages) as the tests that use them.

**What was removed:** References to project-specific shared `TestUtils` classes (e.g., `com.cloudbusiness.utils.TestUtils`, `SocialTokenTestUtils`, `WebhookTestUtils`, etc.) that lived in a single shared package far from the tests.

**Why:**
- A single grab-bag `utils` package for all test data across the entire project violates cohesion.
- When a module changes, its test helpers should change with it — co-location makes this natural.
- Developers can immediately see what helpers are available without hunting through a distant package.
- The existing shared `TestUtils` classes in `synder` are not designed correctly — they mix concerns from many domains into one place.

**What to do instead:**
- Per-domain test helpers in the same package as the tests.
- Only genuinely cross-cutting helpers (used by multiple unrelated domains) go in a shared `testfixtures` source set.

---

### 2. Multi-Language Coverage (Spock + JUnit 5)

**Decision:** Both guides cover Spock/Groovy and JUnit 5/Kotlin side by side.

**What was considered:** Separate guide per language/framework.

**Why single guide:**
- The principles are identical — only syntax differs.
- Developers working across repos need one reference, not two.
- Keeps the review checklist unified.

**Repo-to-stack mapping (as of 2026-03-10):**

| Repo | Test Count | Framework | Mocking | Integration |
|------|-----------|-----------|---------|-------------|
| synder | 880 Groovy Specs | Spock | Spock Mock/Stub/Spy | GORM DataTest |
| accounting | 365 Specs + 81 Tests | Spock + JUnit 5 (Groovy) | Spock + Spring | SpringBootTest + DbUnit |
| daily-summary | 123 Specs + 10 Tests | Spock + JUnit 5 (Groovy) | Spock | — |
| accounting-firm | 11 Specs | Spock | Spock | — |
| listener | 6 Kotlin Tests | JUnit 5 | Mockito | MockMvc, SpringBootTest |
| revenue-recognition | 43 Kotlin Tests | JUnit 5 | Mockito, MockK | Testcontainers |
| transaction-reconciliation | 27 Kotlin Tests | JUnit 5 | Mockito, MockK | Testcontainers |

---

### 3. Source Material

**Decision:** Guides are based on three books:
1. "Pragmatic Unit Testing in Java with JUnit" (3rd edition, Jeff Langr, 2024) — practical patterns, mnemonics, test smells.
2. "Unit Testing: Principles, Practices, and Patterns" (Vladimir Khorikov, Manning, 2020) — four pillars, testing styles, anti-patterns, mocking philosophy.
3. "Effective Software Testing: A Developer's Guide" (Maurício Aniche, Manning, 2022) — systematic test case design, specification-based testing, SQL testing, design for testability, when to stop testing.

**What was considered but not used (yet):**
- "Extreme Programming Explained" (Kent Beck) — Alexey has this in PDF. Good for philosophy but too high-level for practical test instructions. May be used later for TDD workflow guidance.
- "Java Testing with Spock" (Kapelonis) / "Spock: Up and Running" (Fletcher) — Spock-specific deep dives. Not yet incorporated.

- "Growing Object-Oriented Software, Guided by Tests" (Freeman & Pryce) — integration testing focus. Not yet incorporated.

**Full top-10 book list evaluated (2026-03-10):**
1. "Unit Testing Principles, Practices, and Patterns" — Khorikov (C# examples, language-agnostic principles)
2. "Java Testing with Spock" — Kapelonis (Java + Groovy)
3. "Spock: Up and Running" — Fletcher (Java + Groovy)
4. "Pragmatic Unit Testing in Java with JUnit" — Langr ← **USED**
5. "Effective Unit Testing" — Koskela (Java)
6. "Test-Driven Development: By Example" — Beck (Java + Python)
7. "Growing Object-Oriented Software, Guided by Tests" — Freeman & Pryce (Java)
8. "The Art of Unit Testing" — Osherove (C#/JS, language-agnostic)
9. "Testing Java Microservices" — Bueno et al. (Spring Boot, REST-assured)
10. "Effective Software Testing" — Aniche (Java, 2022)

---

### 4. Testing Style Preference Order

**Decision:** Output-based > State-based > Communication-based testing, when you have a choice.

**Why:**
- Output-based tests (assert return values) have the highest resistance to refactoring — they don't couple to internals at all.
- State-based tests (assert state after action) are slightly more coupled but still good.
- Communication-based tests (mock verification) are the most fragile — they lock the interaction pattern and break on harmless refactors.

**When communication-based is correct:** System boundaries (message bus, external API, email sender) where the interaction IS the observable behavior.

**What this means for reviews:** If a reviewer sees mock verification for domain logic that could be tested via return values, they should suggest switching to output-based style.

---

### 5. "Never Assert on Stubs" Rule

**Decision:** Stubs provide data. Mocks verify interactions. Never verify that a stub was called.

**Why (Khorikov):** If you stub `searchService.find() >> results` and then also verify `1 * searchService.find()`, you're testing your own test setup, not production behavior. This creates false confidence and brittle tests.

**Practical impact:** In Spock, if you use `Stub()`, never put it in a `then:` block verification. If you need to verify the call, use `Mock()` instead — but only if the call is part of the observable contract.

---

### 6. Mnemonics Included

**Decision:** Include Right-BICEP, CORRECT, ZOM, and FIRST from the book.

**Why all four:** Each covers a different dimension:
- **Right-BICEP** — what aspects of a method to test
- **CORRECT** — what boundary conditions to check
- **ZOM** — how to handle collections/cardinality
- **FIRST** — test quality properties

These are compact, memorable, and work for both humans and AI agents as checklists.

---

### 7. Systematic Test Case Design Included

**Decision:** Include the 7-step specification-based testing workflow, equivalence partitioning, and boundary value analysis from Aniche.

**Why:** Right-BICEP and CORRECT (from Langr) tell you *what aspects* to test. Aniche's specification-based approach tells you *how to systematically derive test cases* — it's a method, not just a checklist. Particularly valuable for:
- AI agents that need a repeatable algorithm for generating test cases
- Junior developers who don't have the experience to intuit which cases matter
- Complex business logic where ad-hoc testing misses edge cases

**What was considered but excluded (for now):**
- **MC/DC (Modified Condition/Decision Coverage)** — too specialized for most backend code. May be added if safety-critical logic is introduced.
- **Property-based testing** — powerful technique but not currently used in any repo. May be added when adopted.
- **Mutation testing** — valuable for measuring test quality but not currently integrated into CI. Documented in decisions as a future consideration.

---

### 8. No Coverage Targets

**Decision:** No mandatory code coverage percentage in the guides.

**What was considered:** Setting a minimum coverage threshold (e.g., 80%).

**Why excluded:**
- The book explicitly warns against gaming coverage metrics.
- Coverage is useful as a diagnostic (uncovered code = possibly hard to test = possibly needs refactoring) but harmful as a target.
- Teams end up writing low-value tests just to hit the number.
- The review guide focuses on meaningful test completeness (Right-BICEP, ZOM) instead.

---

### 9. No TDD Mandate

**Decision:** Guides don't mandate TDD (test-first) workflow.

**What was considered:** Requiring TDD for all new code.

**Why excluded (for now):**
- TDD is a development workflow, not a test quality standard.
- Both test-first and test-after can produce good tests.
- The guides focus on test quality outcomes, not the process of writing them.
- May be added later as a separate workflow guide if desired.

---

## Excluded Topics (May Be Added Later)

| Topic | Why excluded | When to add |
|-------|-------------|-------------|
| Contract testing (Pact) | Not currently used in any repo | When microservice API contracts become a concern |
| Performance/load testing | Different discipline, different tools | When performance testing guidelines are needed |
| UI/frontend testing | Guides are backend-only | If frontend testing guidelines are requested |
| Mutation testing (PIT) | Concept documented in decisions; not integrated into CI | When test effectiveness measurement is desired |
| Property-based testing | Concept understood (Aniche); not currently used in any repo | When a repo adopts jqwik or similar PBT framework |
| MC/DC coverage | Too specialized for most backend code | When safety-critical or complex boolean logic is introduced |
| Test containers best practices | Briefly covered; could expand | When more repos adopt Testcontainers |
| Flaky test management | Briefly mentioned as anti-pattern | When flaky tests become a systemic problem |
| Design-by-contract tooling | Concept covered; no tooling adopted | When pre/post-condition libraries are introduced |

---

## Revision History

| Date | Authors | Changes |
|------|---------|---------|
| 2026-03-10 | Alexey Sergeev | Initial decisions log: test data co-location rationale, multi-language coverage decision, source material evaluated, mnemonics selection, no coverage targets, no TDD mandate, excluded topics. |
| 2026-03-10 | Alexey Sergeev | Added Khorikov book as second source. Recorded new concepts incorporated: Four Pillars, testing styles preference order, anti-patterns (asserting on stubs, leaking domain knowledge, testing private methods), managed vs unmanaged dependencies, code complexity matrix. |
| 2026-03-10 | Alexey Sergeev | Added Aniche book as third source. Recorded: 7-step specification-based testing, equivalence partitioning, boundary value analysis, SQL testing checklist, design for testability, "when to stop testing" heuristics. Excluded for now: MC/DC, property-based testing, mutation testing, design-by-contract tooling. |
