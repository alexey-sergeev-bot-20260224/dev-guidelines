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

**Decision:** Guides are based on "Pragmatic Unit Testing in Java with JUnit" (3rd edition, Jeff Langr, 2024) adapted for our stack.

**What was considered but not used (yet):**
- "Extreme Programming Explained" (Kent Beck) — Alexey has this in PDF. Good for philosophy but too high-level for practical test instructions. May be used later for TDD workflow guidance.
- "Unit Testing Principles, Practices, and Patterns" (Khorikov) — #1 recommended book for reviewer criteria. Not yet incorporated. Would strengthen the reviewer guide significantly.
- "Java Testing with Spock" (Kapelonis) / "Spock: Up and Running" (Fletcher) — Spock-specific deep dives. Not yet incorporated.
- "Effective Software Testing" (Aniche, 2022) — modern Java testing. Not yet incorporated.
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

### 4. Mnemonics Included

**Decision:** Include Right-BICEP, CORRECT, ZOM, and FIRST from the book.

**Why all four:** Each covers a different dimension:
- **Right-BICEP** — what aspects of a method to test
- **CORRECT** — what boundary conditions to check
- **ZOM** — how to handle collections/cardinality
- **FIRST** — test quality properties

These are compact, memorable, and work for both humans and AI agents as checklists.

---

### 5. No Coverage Targets

**Decision:** No mandatory code coverage percentage in the guides.

**What was considered:** Setting a minimum coverage threshold (e.g., 80%).

**Why excluded:**
- The book explicitly warns against gaming coverage metrics.
- Coverage is useful as a diagnostic (uncovered code = possibly hard to test = possibly needs refactoring) but harmful as a target.
- Teams end up writing low-value tests just to hit the number.
- The review guide focuses on meaningful test completeness (Right-BICEP, ZOM) instead.

---

### 6. No TDD Mandate

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
| Mutation testing (PIT) | Not currently used | When test effectiveness measurement is desired |
| Test containers best practices | Briefly covered; could expand | When more repos adopt Testcontainers |
| Flaky test management | Briefly mentioned as anti-pattern | When flaky tests become a systemic problem |

---

## Revision History

| Date | Authors | Changes |
|------|---------|---------|
| 2026-03-10 | Alexey Sergeev | Initial decisions log: test data co-location rationale, multi-language coverage decision, source material evaluated, mnemonics selection, no coverage targets, no TDD mandate, excluded topics. |
