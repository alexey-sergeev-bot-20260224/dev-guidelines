# Backend Architecture Reviewer Guide

> How to evaluate code structure, module design, and naming in PRs.
> For human reviewers and AI agents alike.

---

## 1. Reviewer's Mindset

Your job is NOT to verify the code compiles or passes CI. Your job is to evaluate:

1. **Structure** — is the code in the right place? Does it respect module boundaries?
2. **Naming** — do class, package, and method names follow conventions and communicate intent?
3. **Coupling** — does the change introduce unwanted dependencies between modules?
4. **Abstraction** — is the level of abstraction appropriate? Not too early, not too late?

A PR that adds a new feature in a flat `services/` directory, or creates a `shared/` utility with one caller, should raise a red flag — even if it works correctly.

---

## 2. First Pass: Structure Review

### 2.1 Package Placement

For every new class in the PR, verify it lives in the correct package:

| Class type | Expected location | Red flag |
|------------|------------------|----------|
| Domain model (POJO) | `{domain}/model/` | In `jpa/` or in a flat `domain/` directory |
| Service interface | `{domain}/service/` | In `jpa/` or alongside implementation |
| Service implementation | `{domain}/jpa/` or `{domain}/http/` etc. | In `service/` (leaks implementation) |
| JPA entity | `{domain}/jpa/` | In `model/` (domain model must not have JPA annotations) |
| Repository | `{domain}/jpa/repository/` | At module root or in `service/` |
| Controller | `{domain}/controller/` | In `service/` or at module root |
| Event | `{domain}/event/` | Scattered across unrelated packages |
| Converter | `{domain}/jpa/converter/` | In `model/` or `shared/` without reason |

### 2.2 New Domain Module Checklist

When a PR introduces a new domain concept:

- [ ] Has its own top-level package under the service root
- [ ] Contains `model/` sub-package with domain objects
- [ ] Contains `service/` sub-package with service interface
- [ ] Implementation is in a technology-specific sub-package (`jpa/`, `http/`, etc.)
- [ ] Controller (if any) is in `controller/` sub-package
- [ ] Package name is a noun describing the domain concept (not a verb, not a technical term)

### 2.3 Existing Module Modification

When a PR modifies an existing module:

- [ ] New classes follow the module's existing package convention
- [ ] No files placed in the wrong sub-package (e.g., implementation details in `service/`)
- [ ] If the module lacks proper structure (legacy), consider whether this PR is a good time for incremental migration

### 2.4 Grails-Style Detection

Flag if you see:

- New service added to a flat `grails-app/services/` without domain package organization
- New domain class in flat `grails-app/domain/`
- Logic that should be in `src/main/` placed in `grails-app/` without justification
- Multiple unrelated concepts in the same package

---

## 3. Second Pass: Naming Audit

### 3.1 Service Implementation Naming

| Check | ✅ Accept | ❌ Request changes |
|-------|----------|-------------------|
| Technology prefix | `JpaTransferService`, `HttpAccountingService`, `JdbcLockService` | `TransferServiceImpl`, `AccountingServiceImpl` |
| Semantic prefix | `DefaultReconciliationService`, `SimpleNotificationService` | `ReconciliationServiceImpl` |
| Test/stub prefix | `InMemoryTransferService`, `StubPaymentGateway` | `MockTransferService` (reserve "Mock" for test frameworks) |

**Severity:** When a PR introduces a new `XxxImpl` class — request changes (blocking). When modifying an existing `XxxImpl` significantly — suggest renaming (non-blocking, but note it).

### 3.2 JPA Entity Naming

| Check | ✅ Accept | ❌ Request changes |
|-------|----------|-------------------|
| **accounting** repo | `XxxMapping` (established convention) | `XxxEntity` (inconsistent with repo) |
| **Other repos** | `XxxEntity` | No suffix, or `XxxTable`, `XxxRecord` |
| Separation from domain model | Entity and domain model are distinct classes | Domain model annotated with `@Entity` directly |

### 3.3 Repository Naming

| Check | ✅ Accept | ❌ Request changes |
|-------|----------|-------------------|
| Name | `TransferRepository`, `BillRepository` | `TransferEntityRepository`, `TransferMappingRepo` |
| Location | In `jpa/repository/` sub-package | At module root or in `service/` |

### 3.4 Constants Review

| Check | ✅ Accept | ❌ Request changes |
|-------|----------|-------------------|
| Enum for finite set | `TransactionType.SALE` | `public static final String TYPE_SALE = "SALE"` |
| Constant on owning class | `Transfer.MAX_MEMO_LENGTH` | `TransferConstants.MAX_MEMO_LENGTH` |
| Focused constants class | `StripeErrorCodes` (all constants relate to Stripe errors) | `AppConstants` (unrelated values dumped together) |
| Configuration values | In `@ConfigurationProperties` | Hardcoded `POOL_SIZE = 4` |

**Specific red flags:**
- Constants named after their value: `INT_100 = 100`
- Constants with incorrect values: `ONE_HOUR_IN_MILLISECONDS = 3600`
- Constants whose name lies: `CORE_POOL_SIZE_FOUR = 3`
- Nested enums inside constants classes (should be top-level): `StripeConstants.Scope` → `StripeScope`

### 3.5 Package Naming

| Check | ✅ Accept | ❌ Request changes |
|-------|----------|-------------------|
| All lowercase | `com.synder.accounting.transfer` | `com.synder.accounting.Transfer` |
| No underscores | `invoicepayment` | `invoice_payment` |
| Domain-oriented | `reconciliation`, `transfer`, `billing` | `utils`, `helpers`, `misc` |

### 3.6 General Class Naming

| Check | ✅ Accept | ❌ Request changes |
|-------|----------|-------------------|
| Acronym handling | `HttpClient`, `JpaService`, `CsvExporter` | `HTTPClient`, `JPAService`, `CSVExporter` |
| Test naming | `TransferServiceSpec` (Spock), `TransferServiceTest` (JUnit) | `TransferServiceTests`, `TestTransferService` |
| Integration test | `TransferServiceIntegrationSpec`, `TransferServiceIntegrationTest` | `TransferServiceIT`, `TransferServiceISpec` |

---

## 4. Third Pass: Coupling and Dependency Review

### 4.1 Import Analysis

For every new import added in the PR, check:

| Import pattern | ✅ Acceptable | ❌ Flag |
|----------------|--------------|--------|
| `{domain}.service.XxxService` | Cross-module via public API | — |
| `{domain}.model.Xxx` | Cross-module via domain model | — |
| `{domain}.jpa.JpaXxxService` | — | Importing implementation directly |
| `{domain}.jpa.XxxMapping` | — | Importing JPA entity across modules |
| `{domain}.jpa.repository.XxxRepository` | — | Importing repository across modules |
| `{domain}.jpa.converter.XxxConverter` | — | Importing converter across modules |

**Quick check:** Search the diff for imports containing `.jpa.` from a different domain package. These are almost always violations.

### 4.2 Dependency Direction Violations

| Violation | What it looks like | Severity |
|-----------|-------------------|----------|
| Domain model imports JPA | `import javax.persistence.*` in a `model/` class | Blocking |
| Service interface references implementation types | `TransferService` method returns `TransferMapping` | Blocking |
| Controller references implementation | `@Autowired JpaTransferService` instead of `TransferService` | Blocking |
| Cross-module implementation access | Module A imports `moduleB.jpa.SomeEntity` | Blocking |
| Domain model imports Spring | `@Component`, `@Service` on a domain model class | Medium |

### 4.3 New Dependencies Between Modules

When a PR adds a new dependency from module A to module B:

- [ ] Is the dependency going through B's public API (service interface + domain model)?
- [ ] Is the dependency direction correct (not circular)?
- [ ] Is the dependency justified — does A genuinely need B's capability?
- [ ] Could the coupling be reduced with an event instead of a direct call?

### 4.4 Circular Dependency Detection

If module A imports from module B **and** module B imports from module A:

- This is a circular dependency — always flag as blocking
- Resolution options:
  1. Extract the shared concept into a third module
  2. Use domain events instead of direct calls
  3. One module is a sub-module of the other — restructure packages accordingly

---

## 5. Fourth Pass: Abstraction Assessment

### 5.1 Premature Abstraction Detection

| Signal | What to look for | Action |
|--------|-----------------|--------|
| New `shared/` or `common/` class | How many callers exist in this PR? | If < 3, flag as premature |
| New base class or interface | Is it extracted from working code or written speculatively? | Speculative = flag |
| Utility with boolean flags | `processData(data, boolean includeHeaders, boolean skipValidation)` | Suggest splitting into separate methods or classes |
| "We'll need this later" | PR description mentions future use cases | YAGNI — flag |
| Abstract class with one subclass | Base class created for a single implementation | Unnecessary — inline until second subclass exists |

### 5.2 Shared Package Review

When a PR adds code to `shared/`, `common/`, or `util/`:

**Acceptable additions:**
- JPA base entities and auditing infrastructure
- Repository base classes used by multiple modules
- Cross-cutting model types genuinely used by 3+ modules (`Money`, `Address`)
- Security infrastructure
- Exception base classes

**Flag for discussion:**
- Domain-specific utilities (should live in their domain package)
- Constants for a specific integration (should live in that integration's package)
- Helpers with 1-2 callers (should live near their callers)
- Anything that could live in a domain package but was placed in `shared/` out of convenience

### 5.3 Interface Necessity

| Situation | Interface needed? | Reasoning |
|-----------|------------------|-----------|
| Service with multiple implementations | ✅ Yes | The whole point of interfaces |
| Service used by other modules | ✅ Yes | Decouples consumer from implementation |
| Service at a system boundary (DB, HTTP, messaging) | ✅ Yes | Enables testing with stubs/mocks |
| Internal helper used within one module only | ❌ No | Overhead without benefit |
| Utility with only static methods | ❌ No | No polymorphism to gain |
| One concrete class, no external consumers | ❌ Probably no | Add interface when a second consumer or implementation appears |

### 5.4 Rule of Three Enforcement

When reviewing extraction of shared code:

- [ ] Are there 3+ independent, working callers?
- [ ] Would callers need to diverge in the future? If yes, keep duplicated.
- [ ] Is the shared code tested independently?
- [ ] Does the shared code have a clear owner (a domain it belongs to)?

If the answer to the first question is "no" — the abstraction is premature. Request keeping the code duplicated until the third caller appears.

---

## 6. Review Comment Templates

Use these patterns for consistent, actionable review comments:

### Package placement
> 📦 **Package placement.** This service implementation is in `service/` alongside the interface. Since it uses JPA, consider moving it to `jpa/` — this keeps the implementation detail hidden from other modules.

### Naming: Impl suffix
> 📛 **Naming: Impl suffix.** `ReconciliationServiceImpl` doesn't communicate what this implementation does. Since it uses JPA/Hibernate, consider `JpaReconciliationService`. If it's not technology-specific, `DefaultReconciliationService` works.

### Naming: Constants dump
> 🗑️ **Constants dump.** `AccountingConstants` contains unrelated values (pool sizes, error messages, magic numbers). Consider: enums for finite sets, `@ConfigurationProperties` for config values, and focused classes per concept (e.g., `StripeErrorCodes`).

### Dependency direction violation
> 🔀 **Dependency violation.** This domain model class imports `javax.persistence.Entity`. Domain models should be plain objects with no infrastructure annotations. Create a separate JPA entity in `jpa/` with a converter.

### Cross-module implementation access
> 🔗 **Cross-module coupling.** This class imports `transfer.jpa.TransferMapping` — a JPA entity from another module. Use `transfer.service.TransferService` and `transfer.model.Transfer` (the public API) instead.

### Premature abstraction
> ⏳ **Premature abstraction.** This utility class in `shared/` has one caller in this PR. Consider keeping the logic in the calling class until you have three independent callers (Rule of Three). Duplication is cheaper than the wrong abstraction.

### Shared package addition
> 📁 **Shared package.** This helper is specific to invoice processing but lives in `shared/`. Consider moving it to the `invoice/` module where its only consumers are.

### Missing interface
> 🔌 **Missing interface.** This service is used by two other modules (`invoice` and `bill`) but has no interface. Extracting an interface in `service/` would decouple the consumers from the JPA implementation.

### Unnecessary interface
> 🪞 **Unnecessary interface.** This interface has exactly one implementation, is used only within this module, and doesn't represent a system boundary. Consider removing the interface and using the concrete class directly until a second implementation or external consumer appears.

### Circular dependency
> 🔄 **Circular dependency.** Module `invoice` now imports from `payment`, and `payment` already imports from `invoice`. This creates a circular dependency. Consider: (1) extracting the shared concept into a third module, (2) using domain events, or (3) restructuring one as a sub-module.

### Good structure (positive feedback)
> ✅ Clean module structure — `model/`, `service/`, `jpa/`, `controller/` separation is exactly right. Nice work keeping the JPA entity separate from the domain model.

### Migration opportunity
> 🔄 **Migration opportunity.** Since you're modifying `ReconciliationServiceImpl` significantly, this might be a good time to rename it to `JpaReconciliationService` to follow the naming convention. Up to you — not blocking.

### Fat controller
> 🎮 **Fat controller.** This controller contains business logic (discount calculation on lines 45-67). Consider extracting this into a domain service — controllers should only translate between HTTP and service calls.

### Flat structure
> 📂 **Flat structure.** This new feature adds classes to the root package without a domain sub-package. Consider creating a `{feature}/` package with the standard `model/`, `service/`, `jpa/` structure.

---

## 7. Decision Framework: When to Push Back vs Accept

### Request changes (blocking):

- New feature placed in flat layer-first directory without domain package
- `XxxImpl` suffix on a newly created class
- Domain model class with JPA annotations (should be separated)
- Cross-module import of `jpa/` package internals (entities, repositories, converters)
- Service interface that exposes implementation types (JPA entities, Spring-specific types)
- Circular dependency between modules
- Controller with embedded business logic that should be in a service
- New `shared/` class with zero or one caller in the codebase
- Constants class that is a catch-all dump of unrelated values

### Suggest improvements (non-blocking):

- Existing `XxxImpl` class significantly modified (suggest rename)
- Missing interface for a service used within one module
- `shared/` class with two callers (suggest keeping but note the Rule of Three)
- Sub-optimal package organization that could be improved but works
- Constants that could be enums but aren't
- Renaming opportunity for better clarity

### Accept as-is:

- Code in a legacy module that follows the module's existing (imperfect) conventions
- Minor naming deviations that don't hurt readability
- `shared/` infrastructure code that is genuinely cross-cutting
- Concrete class without interface when there's only one implementation and no external consumers
- Incremental improvements to legacy structure (don't demand full refactor in every PR)

---

## 8. Quick Reference: Review Checklist

Copy this into your review workflow:

```
## Architecture Review

### Structure
- [ ] New domain concept has its own top-level package
- [ ] Package follows model/ + service/ + jpa/ + controller/ convention
- [ ] No code placed in flat layer-first directories
- [ ] Implementation classes are in jpa/ (or http/, etc.), not in service/

### Naming
- [ ] Service implementations use technology prefix (Jpa, Jdbc, Http) or semantic prefix (Default, Simple)
- [ ] No new XxxImpl classes introduced
- [ ] JPA entities use XxxEntity or XxxMapping (consistent with repo)
- [ ] Repositories named XxxRepository (not XxxEntityRepository)
- [ ] Constants follow decision order: enum → owning class → focused class
- [ ] No catch-all constants dump (AppConstants, XxxConstants)

### Dependencies
- [ ] Cross-module imports go through public API only (service interface + domain model)
- [ ] No imports of another module's jpa/ package (entities, repos, converters)
- [ ] Domain model classes have no infrastructure imports (JPA, Spring)
- [ ] Service interfaces reference only domain model types
- [ ] No circular dependencies between modules
- [ ] Controller depends on interface, not implementation

### Abstraction
- [ ] No new shared/common/util class with fewer than 3 callers
- [ ] No premature interface (single implementation, single consumer, not a boundary)
- [ ] No premature extraction (code extracted before being tested in concrete form)
- [ ] shared/ additions are genuinely cross-cutting, not misplaced domain code

### Application Layer
- [ ] Controllers are thin — no business logic
- [ ] Cross-module coordination in explicit orchestrator services
- [ ] No domain service directly accessing another domain's internals
```

---

*Based on Synder backend development guidelines (Alexey Sergeev), codebase analysis across synder, accounting, accounting-firm, listener, daily-summary, revenue-recognition, and transaction-reconciliation repositories.*

---

## Revision History

| Date | Authors | Changes |
|------|---------|---------|
| 2026-03-12 | Bob (AI agent) | Initial draft: 4-pass review structure (structure, naming, coupling, abstraction), package placement checklist, naming audit tables, import/dependency analysis, premature abstraction detection, Rule of Three enforcement, review comment templates, decision framework (block vs suggest vs accept), copy-paste review checklist. Based on DEV_RULES.md and codebase analysis of all SynderAccounting repositories. |
