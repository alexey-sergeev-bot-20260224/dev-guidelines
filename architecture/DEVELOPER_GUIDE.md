# Backend Architecture Developer Guide

> How to structure code, design modules, and make architecture decisions.
> For human developers and AI agents alike.

---

## 1. Why Architecture Matters

- **Maintainability** — well-structured code is easier to change without breaking unrelated parts.
- **Testability** — clean module boundaries make unit testing natural and integration testing focused.
- **Onboarding** — new team members (human or AI) can find things quickly when the structure is consistent.
- **Scalability** — modules with clear boundaries can be extracted into separate services when needed.
- **Longevity** — architecture is what separates a 7-year-old codebase that still moves fast from one that fights you at every change.

Architecture is not a one-time design exercise. It is a daily practice applied in every PR, every class, every package decision.

---

## 2. The Core Principle: Domain-First Organization

### What It Means

Organize code by **business domain**, not by technical layer. Each domain module owns everything it needs — its models, services, controllers, and persistence — grouped together in one package.

### Why It Matters

When code is organized by domain, the package structure answers the question *"what does this system do?"* rather than *"what technologies does this system use?"*. You can navigate to a domain module and see everything about it in one place. Dependencies between modules are explicit through imports.

When code is organized by layer (all controllers together, all services together, all domain classes together), the package structure answers nothing useful. A `PaymentService` sits next to `InvoiceService` sits next to `AlertService` — unrelated things mixed together, with no visible boundaries.

### The Rule

**Every new feature or domain concept must live in its own package.** That package contains all code related to that concept: model, service interface, implementation, controller, events, and exceptions.

```
com.synder.{service}.{domain}/
├── model/                   ← domain objects (POJOs, enums, value objects)
├── service/                 ← public API: service interfaces, search parameters
├── jpa/                     ← implementation: JPA entities, repositories, converters
│   ├── converter/           ← entity ↔ domain model converters
│   └── repository/          ← Spring Data repositories
├── controller/              ← HTTP entry points
├── event/                   ← domain events
└── validation/              ← validation logic and exceptions
```

### Reference Implementation: `transfer` Module (accounting)

The [`transfer` module](https://github.com/SynderAccounting/accounting/tree/production/backend/src/main/java/com/synder/accounting/transfer) demonstrates this pattern:

```
com.synder.accounting.transfer/
├── model/
│   ├── Transfer.java                    ← domain object (public API)
│   └── validation/                      ← validation annotations
├── service/
│   ├── TransferService.java             ← interface (public API)
│   ├── TransferSearchParameters.java    ← query parameters (public API)
│   └── ledger/
│       └── TransferJournalEntryService.java
├── jpa/
│   ├── JpaTransferService.java          ← implementation (hidden)
│   ├── TransferMapping.java             ← JPA entity (hidden)
│   ├── TransferConverter.java           ← entity ↔ model converter (hidden)
│   └── repository/
│       └── TransferMappingRepository.java
├── controller/
│   └── TransferController.java          ← HTTP endpoint
├── event/
│   ├── TransferCreatedEvent.java
│   └── TransferDeletedEvent.java
└── validation/
    └── TransferValidationException.java
```

Other modules import `TransferService` and `Transfer`. They never import `JpaTransferService`, `TransferMapping`, or `TransferMappingRepository`. The database layer is invisible to the outside.

### Good Examples Across Repos

| Repo | Module | Structure |
|------|--------|-----------|
| **accounting** | `bill`, `invoice`, `salesreceipt`, `transfer` | `model/` + `service/` + `jpa/` + `controller/` |
| **accounting-firm** | `firm`, `firmclient`, `firmcolleague`, `license` | `model/` + `service/` + `jpa/` + `controller/` |
| **daily-summary** | `summary`, `channelconfiguration`, `organization` | `model/` + `service/` + `jpa/` + `controller/` |
| **listener** | `shopify`, `stripe`, `amazon` | Domain-by-integration: each provider owns its extractor, verification, and configuration |

---

## 3. Public API vs Internal Implementation

### The Separation

A domain module has two distinct surfaces:

**Public API** — what other modules depend on:
- Service interfaces (`TransferService`)
- Domain model classes (`Transfer`, `BillStatus`)
- Search/filter parameter objects (`TransferSearchParameters`)
- Domain events (`TransferCreatedEvent`)
- Exceptions (`TransferValidationException`)

**Internal implementation** — what no other module should know about:
- JPA entities (`TransferMapping`, `TransferMetaPropertyMapping`)
- Repository interfaces (`TransferMappingRepository`)
- Entity-to-model converters (`TransferConverter`)
- Database-specific logic, caching, file I/O, HTTP clients
- Transaction management details

### Why This Matters

When implementation details are hidden behind an interface, you can change how something works internally without breaking callers. You can:
- Switch from JPA to JDBC without touching a single caller
- Add caching without changing the service contract
- Replace an SMTP email sender with a queue-based sender
- Swap a real implementation for an in-memory stub in tests

Breaking changes only happen when you modify the **public interface** itself.

### Examples of the Pattern

**Data persistence:**
```java
// Public API — other modules depend on this
public interface TransferService {
    Transfer create(Transfer transfer);
    Optional<Transfer> findById(Long id);
}

// Hidden implementation — callers never see this
@Service("transferService")
public class JpaTransferService implements TransferService {
    // JPA repositories, entity mappings, transactions — all internal
}
```

**Email sending:**
```java
// Public API
public interface EmailSender {
    void send(String to, String subject, String body);
}

// Production
public class SmtpEmailSender implements EmailSender { ... }

// Local development / tests
public class ConsoleEmailSender implements EmailSender { ... }
```

**External integrations:**
```java
// Public API
public interface PaymentGatewayService {
    PaymentResult charge(PaymentRequest request);
}

// Implementations: StripePaymentGatewayService, PayPalPaymentGatewayService, StubPaymentGatewayService
```

### When to Create an Interface

**Create an interface when:**
- The service has (or may have) multiple implementations
- Other modules depend on it — an interface makes the dependency explicit and stable
- You need testability — mocking an interface is cleaner than mocking a concrete class
- The service represents a **boundary** (database, external API, messaging, email)

**Skip the interface when:**
- The class is a pure internal helper used only within its own module
- There is genuinely only one possible implementation and no external consumers
- The class is a utility with only static methods

**Practical default:** If the service is in `service/` and other modules import it, give it an interface. If it's a private helper inside `jpa/`, a concrete class is fine.

---

## 4. Dependency Direction

### The Rule

Dependencies flow **inward** — from infrastructure toward the domain, never the other way:

```
Controller  →  Service Interface  ←  Implementation (JPA, HTTP, etc.)
                     ↓
                Domain Model
```

- Controllers depend on service interfaces and domain models
- Implementations depend on service interfaces, domain models, and infrastructure (JPA, HTTP clients)
- Domain models depend on **nothing** (no framework annotations, no infrastructure imports)
- Service interfaces depend only on domain models

### What This Means in Practice

1. **Domain model classes must not import JPA/Hibernate annotations.** The domain model (`Transfer.java`) is a plain Java object. The JPA entity (`TransferMapping.java`) is a separate class in the `jpa/` package with a converter between them.

2. **Service interfaces must not reference implementation types.** `TransferService` returns `Transfer`, not `TransferMapping`. It accepts `TransferSearchParameters`, not a JPA `Specification`.

3. **Controllers depend on service interfaces, not implementations.** Spring injects the implementation; the controller never mentions `JpaTransferService`.

4. **Cross-module imports go through public API only.** If `invoice` needs to look up a transfer, it imports `TransferService` and `Transfer` — never `JpaTransferService` or `TransferMapping`.

### Forbidden Dependencies

| From | To | Why it's wrong |
|------|----|----------------|
| Domain model | JPA entity | Domain model becomes tied to persistence strategy |
| Service interface | JPA repository | Interface leaks implementation details |
| Module A's `jpa/` | Module B's `jpa/` | Bypasses the public API, creates hidden coupling |
| Any module | Another module's `converter` | Converters are implementation details |
| Domain model | Controller DTOs | Domain should not know about presentation |

---

## 5. Naming Conventions

Consistent naming reduces cognitive load. When a class name tells you what it does and how, you don't need to open it.

### Packages

All lowercase, no underscores. Organize by domain, not by technical role:

```
✅  com.synder.accounting.transfer
✅  com.synder.accounting.bill.service
❌  com.synder.accounting.services.transfer
❌  com.synder.accounting.transfer_service
```

### Classes — General

UpperCamelCase. Treat acronyms as words after the first letter: `XmlParser`, `HttpClient`, `JpaTransferService`. Well-established short initialisms follow camelCase when embedded: `CsvExporter`, `JwtService`.

### Service Interfaces

Name after the domain concept with a `Service` suffix:

```
TransferService, EmailSender, ReconciliationService, BillWorkflowService
```

### Service Implementations

If the implementation is tied to a **specific technology**, use a technology-descriptive **prefix**:

| Technology | Prefix | Example |
|---|---|---|
| JPA / Hibernate | `Jpa` | `JpaTransferService` |
| JDBC | `Jdbc` | `JdbcReconciliationLockService` |
| HTTP / Feign | `Http` | `HttpAccountingService` |
| SMTP | `Smtp` | `SmtpEmailSender` |
| In-memory (test/stub) | `InMemory` | `InMemoryTransferService` |
| Spring-specific | `Spring` | `SpringSecuritySupportService` |

If the implementation is **not tied** to a specific technology, use a semantic qualifier:

| Qualifier | Meaning | Example |
|---|---|---|
| `Simple` | Minimal, no-frills | `SimpleNotificationService` |
| `Default` | Standard fallback | `DefaultReconciliationService` |
| `Standard` | Canonical reference implementation | `StandardInvoiceCalculator` |

**Never use `XxxImpl`.** It carries no information about what the implementation does or how. A class named `ReconciliationServiceImpl` tells you nothing — `JpaReconciliationService` or `DefaultReconciliationService` immediately communicates its nature.

**Current codebase state:** The `Impl` suffix is widespread in older code (104 in synder, 94 in revenue-recognition, 46 in transaction-reconciliation). New code must use descriptive prefixes. When touching existing `Impl` classes significantly, consider renaming as part of the PR.

### JPA Entities

Use the `XxxEntity` suffix (or `XxxMapping` in accounting, where the convention is established). The suffix distinguishes the persistence object from the domain model:

```
TransferMapping / TransferEntity   ← JPA-mapped class (in jpa/ package)
Transfer                           ← domain model class (in model/ package)
```

### JPA Repositories

Use `XxxRepository`. Don't embed the entity class name:

```
✅  TransferRepository (entity type visible from generic parameter)
❌  TransferEntityRepository
❌  TransferMappingRepository (acceptable in accounting where XxxMapping is the entity convention)
```

### Controllers

`XxxController` suffix. One controller per domain module:

```
TransferController, BillController, InvoiceController
```

### Events

Use descriptive verb form: `XxxCreatedEvent`, `XxxDeletedEvent`, `XxxUpdatedEvent`.

### Exceptions

Domain-specific with `Exception` suffix: `TransferValidationException`, `ObjectNotFoundException`.

### Search/Filter Objects

`XxxSearchParameters`:

```
TransferSearchParameters, BillSearchParameters, ReconciliationSearchParameters
```

### Converters

`XxxConverter` — converts between domain models and JPA entities (or DTOs):

```
TransferConverter, BillConverter, BillLineConverter
```

### Constants

Follow this decision order:

1. **Use an enum** if the value belongs to a finite, named set: transaction types, statuses, roles, error codes.
2. **Put it on the owning class** if only one class uses it: `MAX_MEMO_LENGTH` belongs on `Transfer`, not in a separate file.
3. **Create a focused constants class** with a descriptive name if you have a coherent group of related literals: `StripeErrorCodes`, `MixpanelEvents`, `ApiPaths`. Every constant must relate to the same concept.
4. **Never create a catch-all dump.** A class named `AppConstants` or `AccountingConstants` that collects unrelated values is a junk drawer, not a constants class.

**Enums nested inside constants classes** should be top-level classes: `StripeConstants.Scope` → `StripeScope`.

**Configuration values** (pool sizes, intervals, batch sizes) belong in `@ConfigurationProperties`, not hardcoded in source.

**What bad constants look like** (from the codebase):
```kotlin
const val INT_100 = 100                    // naming a number after itself
const val ONE_HOUR_IN_MILLISECONDS = 3600  // wrong: 3600 is seconds, not milliseconds
const val CORE_POOL_SIZE_FOUR = 3          // name lies about the value
```

### Tests

| Test type | Naming | Example |
|-----------|--------|---------|
| Unit test (Spock) | `XxxSpec` | `TransferServiceSpec` |
| Unit test (JUnit 5) | `XxxTest` | `TransferServiceTest` |
| Integration test (Spock) | `XxxIntegrationSpec` | `TransferServiceIntegrationSpec` |
| Integration test (JUnit 5) | `XxxIntegrationTest` | `TransferServiceIntegrationTest` |

The suffix follows the framework, not the language. A Groovy class using JUnit is `XxxTest`.

---

## 6. Code Reuse and Abstraction

### The Problem: Premature Abstraction

Premature abstraction is extracting "reusable" code before you have working, tested implementations to base it on. A developer sees that two pieces of logic look similar, creates a shared utility or base class, and makes both depend on it — before either is fully built or tested.

This feels productive. In practice:

- The abstraction is based on superficial similarity, not conceptual identity. The two callers appear to share something today but will diverge tomorrow.
- Shared code without proven callers has no real requirements. It is written speculatively — either too generic (complex) or too specific (not actually reusable).
- Once other code depends on it, it is difficult to delete or change.

### The Rules

**Make it work first.** No abstraction before a working, tested solution exists. Build the concrete implementation, make it correct, cover it with tests — then consider extraction.

**The Rule of Three.** Do not extract shared code until you have **three** real, independent callers with working implementations. One is a solution. Two might be a coincidence. Three is a pattern worth abstracting.

**Duplication is cheaper than the wrong abstraction.** Two copies of similar code that can evolve independently are safer than one "reusable" abstraction that ties them together prematurely. Wrong abstractions are hard to undo; duplication is easy to fix once the right pattern emerges.

**Challenge every shared class.** Any new class in `common/`, `shared/`, or `util/` must answer: *what are its proven callers?* and *what happens if those callers need to diverge?*

### What Premature Abstraction Looks Like

- A utility or base class introduced before its callers are fully implemented or tested.
- A "reusable" method that accepts boolean flags or strategy parameters to handle different cases for different callers.
- Two callers share code, but one already has workarounds around the shared logic.
- Shared code in `common/` or `util/` with no clear owner and no tests.
- "We'll need this later" — YAGNI applies directly.

### The `shared/` and `common/` Problem

Most services have a `shared/` or `common/` package. Some are well-organized (legitimate cross-cutting concerns like tenant awareness, JPA base classes, security). Others have become dumping grounds.

**Acceptable in `shared/`:**
- JPA base entities and auditing (`TenantAwareMapping`, `AuditableTenantIdAwareMapping`)
- Repository base classes (`ExtendedRepository`, `TenantAwareRepository`)
- Cross-cutting model types genuinely used everywhere (`Money`, `Address`, `PaymentProcessor`)
- Security infrastructure (`SecuritySupportService`)
- Exception base classes

**Should NOT be in `shared/`:**
- Domain-specific utilities (`XeroProductSearchPriorityService` in `common/` — belongs in the Xero module)
- Constants for a specific domain (`ProductMappingConstants` — belongs in the product mapping module)
- Helpers used by only 1-2 modules (move to those modules)

**Before adding to `shared/`:** Ask whether the class is genuinely cross-cutting or just doesn't have a clear home. If it's the latter, give it a home in the right domain package.

---

## 7. The Grails Anti-Pattern

### What It Is

[Grails](https://grails.org/) enforces a fixed folder structure by technical layer:

```
grails-app/
├── controllers/    ← ALL controllers for the entire app
├── services/       ← ALL services for the entire app
├── domain/         ← ALL domain classes for the entire app
└── views/          ← ALL templates
```

This works for MVPs and small applications. As the application grows, it becomes a problem: all controllers belong to every domain at once, all services are mixed together, and there are no natural boundaries.

### The Synder Monolith

The `synder` repository illustrates this exactly:
- **632 service classes** in one flat `grails-app/services/` directory
- **198 domain classes** in one flat `grails-app/domain/` directory
- **153 controllers** in one flat `grails-app/controllers/` directory
- A `common/` package containing unrelated utilities: `AppConstants`, `DateUtil`, `StringUtil`, `PhoneUtils`, `CommonService`
- 104 `XxxImpl` service implementations with no naming convention to indicate technology

The result: no visible module boundaries, everything can depend on everything, and changing one feature risks breaking others.

### Why It Happens

- Grails auto-discovers classes by directory convention, discouraging custom package structures.
- Rapid development pressure ("ship the feature, worry about structure later").
- No enforced rules about which service can call which.
- Over time, `PaymentService` imports `InvoiceService` imports `AccountService` → everything depends on everything.

### The Lesson

New services (accounting, accounting-firm, listener, daily-summary) use domain-first organization. When writing new code, even in the synder monolith, use domain-first packages under `src/main/` when possible. When touching existing Grails-style code, consider incremental migration.

---

## 8. Module Design Decisions

### When to Create a New Module (Package)

Create a new top-level domain package when:

- You're implementing a new **business concept** (e.g., "transfers", "invoices", "reconciliation")
- The concept has its own **lifecycle** (create, update, delete, query)
- The concept has its own **data** (entities, models) that other concepts reference but don't own
- The concept could potentially be described to a non-technical person as "a thing the system manages"

### When to Add to an Existing Module

Add to an existing module when:

- The new code is a sub-feature of an existing concept (e.g., invoice email → part of invoice module)
- The new code operates exclusively on the existing module's data
- Splitting would create two modules that always change together (false boundary)

### When to Extract from an Existing Module

Extract into a new module when:

- A sub-package has grown to have its own model, service, and persistence
- Two consumers need different implementations of the same concept
- The sub-feature is referenced by multiple other modules independently

### Sub-Module Organization

Within a domain module, use sub-packages for logical grouping:

```
invoice/
├── model/
├── service/
│   └── ledger/                    ← sub-concern: journal entries for invoices
├── jpa/
├── controller/
├── email/                         ← sub-module: invoice-specific email logic
│   ├── model/
│   ├── service/
│   └── controller/
└── event/
```

The `email/` sub-module within `invoice/` has its own model-service-controller structure because it represents a distinct sub-concern. But it's not a top-level module because it only makes sense in the context of invoices.

---

## 9. Migration Patterns: Moving Toward Clean Architecture

### Incremental Migration from Grails-Style

When touching legacy flat-structure code:

1. **Don't rewrite.** Migrate incrementally as you touch code for feature work.
2. **Create the target package structure** in `src/main/` for the domain you're modifying.
3. **Move the domain model first** — extract the POJO from the GORM domain class.
4. **Create the service interface** in the new package.
5. **Move/create the implementation** — the existing Grails service becomes the implementation.
6. **Update callers** one at a time to use the new interface.
7. **Move the controller** last (most visible change, most imports to update).

### Splitting Interface from Implementation

For services that are concrete classes without interfaces:

1. **Extract the interface** from the existing class (IDE: "Extract Interface").
2. **Name the interface** after the domain concept: `ReconciliationService`.
3. **Rename the implementation** with a technology prefix: `JpaReconciliationService`.
4. **Register the implementation** with `@Service("reconciliationService")` so existing injection points work.

### Renaming `Impl` to Descriptive Prefix

When renaming an `XxxImpl` class:

1. **Determine the right prefix** — look at what the class actually depends on:
   - Uses JPA/Hibernate → `Jpa`
   - Uses JDBC → `Jdbc`
   - Uses HTTP client → `Http`
   - No specific technology → `Default`
2. **Rename** the class and update all references.
3. **Include in the same PR** as the functional change if touching the file anyway. Don't create rename-only PRs unless the scope is small.

---

## 10. Application Layer and Orchestration

The application layer sits at the top of the dependency hierarchy. It integrates domain modules together to deliver features.

### The Rule

The application layer **orchestrates** but does not **implement** business logic. Business logic belongs in domain modules. The application layer wires things together.

Think of the UI: it knows about users, payments, invoices, reports — but it doesn't implement any of them. The backend application layer works the same way.

### Controllers as Orchestrators

Controllers should be thin — they translate HTTP requests into service calls and service responses into HTTP responses:

```java
@RestController
@RequestMapping("/api/v1/transfers")
public class TransferController {

    private final TransferService transferService;

    @PostMapping
    public Transfer create(@RequestBody Transfer transfer) {
        return transferService.create(transfer);
    }
}
```

If a controller contains business logic (calculations, validation beyond input format, conditional workflows), that logic should be extracted to a service.

### Cross-Module Coordination

When a feature spans multiple domain modules, create a dedicated orchestration service:

```java
// Good: explicit orchestrator for a workflow that spans invoices and payments
public class InvoicePaymentWorkflowService {
    private final InvoiceService invoiceService;
    private final PaymentService paymentService;
    private final NotificationService notificationService;

    public void processPayment(Long invoiceId, PaymentRequest request) {
        Invoice invoice = invoiceService.getRequired(invoiceId);
        PaymentResult result = paymentService.charge(request);
        invoiceService.markPaid(invoiceId, result);
        notificationService.sendPaymentConfirmation(invoice, result);
    }
}
```

Don't put cross-module logic inside one of the domain services — it creates a hidden dependency direction violation.

---

## 11. Service Ecosystem: Where Each Repo Fits

Understanding the role of each service helps you make the right architecture decisions:

| Service | Language | Role | Architecture Maturity |
|---------|----------|------|----------------------|
| **synder** | Groovy (Grails) | Main monolith — integrations, sync, user management, billing | Legacy. Flat Grails structure. Treat as migration source. |
| **accounting** | Java (Spring Boot) | Books: invoices, bills, transfers, journal entries, reconciliation | Reference. Domain-first, clean module boundaries. |
| **accounting-firm** | Groovy (Spring Boot) | Multi-org management for accounting firms | Good. Domain-first, clean structure. |
| **listener** | Kotlin (Spring Boot) | Webhook ingestion: Stripe, Shopify, Amazon events | Good. Domain-by-integration, focused. |
| **revenue-recognition** | Kotlin (Spring Boot) | Revenue recognition calculations and scheduling | Mixed. Large `common/` package, heavy `Impl` usage. |
| **transaction-reconciliation** | Kotlin (Spring Boot) | Transaction matching and reconciliation workflows | Mixed. `Impl` naming, some `shared/` concerns. |
| **daily-summary** | Java (Spring Boot) | Daily email summaries and reporting | Decent. `api/impl` pattern, some `shared/`. |

**When building something new**, follow the accounting repo patterns. When modifying existing code, apply the architecture guidelines incrementally.

---

## 12. Checklist Before Submitting

Before submitting code in a PR, verify:

### Module Structure
- [ ] New domain concept lives in its own top-level package
- [ ] Package follows the `model/` + `service/` + `jpa/` + `controller/` convention
- [ ] No code lives in a layer-first flat directory (e.g., all services together)

### Public API vs Implementation
- [ ] Service interface lives in `service/` sub-package
- [ ] Implementation lives in `jpa/` (or `http/`, etc.) sub-package
- [ ] Other modules only import the interface and domain model — never the implementation
- [ ] JPA entities, converters, and repositories are not imported by other modules

### Dependencies
- [ ] Domain model classes have no infrastructure imports (no JPA, no Spring, no HTTP)
- [ ] Service interfaces reference only domain model types
- [ ] No cross-module imports bypass the public API (no importing another module's `jpa/` package)
- [ ] Controller depends on service interface, not implementation class

### Naming
- [ ] Service interface: `XxxService` (domain concept + Service suffix)
- [ ] Service implementation: technology prefix (`Jpa`, `Jdbc`, `Http`, `Default`) — **not** `XxxImpl`
- [ ] JPA entity: `XxxEntity` or `XxxMapping` (consistent with the repo's convention)
- [ ] Repository: `XxxRepository` (not `XxxEntityRepository`)
- [ ] Constants follow the decision order: enum → owning class → focused class → **never** a catch-all dump

### Abstraction
- [ ] No new `shared/`, `common/`, or `util/` class added without proven callers
- [ ] No premature abstraction — concrete implementation exists and is tested before extraction
- [ ] If extracting shared code: three or more independent callers exist
- [ ] Duplication is tolerated when callers may diverge

### Application Layer
- [ ] Controller is thin — no business logic, only HTTP translation
- [ ] Cross-module coordination uses explicit orchestration service
- [ ] No domain service directly imports another domain's implementation details

---

*Based on Synder backend development guidelines (Alexey Sergeev), codebase analysis across synder, accounting, accounting-firm, listener, daily-summary, revenue-recognition, and transaction-reconciliation repositories.*

---

## Revision History

| Date | Authors | Changes |
|------|---------|---------|
| 2026-03-12 | Alexey Sergeev | Initial draft: domain-first organization, public API vs implementation, dependency direction, naming conventions, code reuse and abstraction, Grails anti-pattern, module design decisions, migration patterns, application layer, service ecosystem overview, pre-submit checklist. Based on DEV_RULES.md and codebase analysis of all SynderAccounting repositories. |
