# Synder Backend Development Guidelines

> **Note:** This document is being gradually decomposed into focused, topic-specific guides.
> As dedicated guides are created for each area (testing, architecture, naming, etc.),
> the corresponding sections here will be replaced with links.
> See the [`dev-guidelines`](https://github.com/alexey-sergeev-bot-20260224/dev-guidelines) repository for the full collection.

This document covers JVM-based applications and services written in Java, Groovy, or Kotlin, using libraries and frameworks such as Spring, Hibernate, and JDBC.

---

## Domain Model

When creating new business logic from scratch, it must be designed so that other parts of the system can clearly understand:

- what the module does and what data it operates on
- what it depends on (inbound dependencies)
- what depends on it (outbound dependencies)

### Public API vs. Internal Implementation

A domain module exposes two distinct surfaces:

**Public API** — the part the outside world interacts with: service interfaces and domain model classes. This is what other modules import and depend on. It must be stable, well-named, and intentionally designed.

**Internal implementation** — everything behind the interface: database access, file I/O, external HTTP calls, caching, transaction management. This is an implementation detail that no other module should know or care about.

This separation — often called **encapsulation** — is the key to keeping modules independent. When you hide implementation details behind an interface, you can freely change how something works internally without breaking the callers. Breaking changes only happen when you modify the public interface itself.

capsule.jpg

### From Interface to Implementation: Hiding the Details

Here is how the principle flows in practice. The caller depends on an interface and works with domain model objects. It has no idea what is behind that interface — and it should not need to know.

A few examples of the same pattern in different contexts:

**Email sending.** `EmailService` is the interface. In production, `SmtpEmailService` delivers messages via an SMTP server. In local development or tests, `ConsoleEmailService` simply prints the message to the log. The callers — the code that triggers email sending — never change when you switch between the two.

**Data persistence.** `TransferService` is the interface. `JpaTransferService` persists data via Hibernate and a relational database. A future `InMemoryTransferService` could serve as a fast in-memory implementation for tests. Again, the callers are unaffected.

**External integrations.** `PaymentGatewayService` is the interface. One implementation talks to Stripe, another to PayPal, a third is a stub for testing. The business logic that initiates payments never needs to change.

The pattern is the same in every case: **hide everything implementation-specific behind the interface**. Other modules only see the interface and the domain model. If you change how something works internally — switch a database engine, replace an HTTP client, add caching, swap a third-party provider — the callers never change.

### Real Example: `transfer` Module

The [`transfer` module](https://github.com/SynderAccounting/accounting/tree/d02e08cff873761d8807b6f07d1cee42e75f15cd/backend/src/main/java/com/synder/accounting/transfer) in the `accounting` service is a good reference for this approach.

**Package structure:**

```
com.synder.accounting.transfer/
├── model/                   ← domain model: Transfer.java, validation rules
├── service/                 ← public API: TransferService interface, search parameters
│   └── ledger/              ← internal service logic (journal entries)
├── jpa/                     ← hidden implementation: JPA entity mapping, repositories
│   └── repository/
├── controller/              ← HTTP entry point (REST API)
├── event/                   ← domain events (TransferCreatedEvent, TransferDeletedEvent)
└── validation/              ← validation exceptions
```

**Public API — the interface other modules depend on:**

```java
// com.synder.accounting.transfer.service.TransferService
public interface TransferService {

    Page<Transfer> findAll(TransferSearchParameters parameters, Pageable pageable);

    Optional<Transfer> findByTransferId(Long transferId);

    Transfer create(Transfer transfer);

    Transfer update(Long transferId, Transfer transfer);

    void delete(Long transferId);
}
```

**Hidden implementation — callers never see this:**

```java
// com.synder.accounting.transfer.jpa.JpaTransferService
@Service("transferService")
public class JpaTransferService implements TransferService, DataDeleteService, ... {
    // JPA repositories, entity mappings, transaction management — all internal
}
```

Other modules import `TransferService` and `Transfer`. They never import `JpaTransferService`, `TransferMapping`, or `TransferMappingRepository`. The database layer is completely invisible to the outside.

### Package Conventions

Each domain model must live in its **own package**, containing:

- `model/` — domain objects
- `service/` — public service interfaces
- Any implementation sub-packages (`jpa/`, `http/`, etc.) — internal only

This makes the module boundary explicit and enforces encapsulation at the code structure level. When these principles are followed, system modules can be extended and tested independently, without touching the rest of the application.

---

## Application Logic

The application layer sits at the top of the dependency hierarchy. It integrates all domain modules and infrastructure components together to deliver a working product.

Think of the UI as a good mental model: it is the visible tip of the iceberg. The UI knows about many things — users, payments, transfers, reports — but it does not implement any of them. It just wires them together and presents the result.

The same applies to the application layer in the backend: it depends on everything else, but ideally implements very little on its own. Business logic belongs in domain modules. The application layer orchestrates.

---

## Naming Conventions

Consistent naming reduces the cognitive load of reading unfamiliar code. When a class name tells you what it does and how it is implemented, you do not need to open it to understand its role.

### Packages

Packages must be all lowercase, no underscores. Organize by domain, not by technical role — see the Domain Model section.

### Classes — General

Use UpperCamelCase. Treat acronyms as words after the first letter: `XmlParser`, `HttpClient`, `JpaTransferService`. Well-established short initialisms (`Csv`, `Pdf`, `Jwt`, `Sql`) follow camelCase when embedded in a name: `CsvExporter`, `JwtService`.

### Service Interfaces

Name after the domain concept with a `Service` suffix: `TransferService`, `EmailSender`, `ReconciliationService`.

### Service Implementations

If the implementation is tied to a specific technology, use a technology-descriptive prefix:

| Technology | Prefix | Example |
|---|---|---|
| JPA / Hibernate | `Jpa` | `JpaTransferService` |
| JDBC | `Jdbc` | `JdbcReconciliationLockService` |
| HTTP / Feign | `Http` | `HttpAccountingService` |
| SMTP | `Smtp` | `SmtpEmailSender` |
| In-memory (test/stub) | `InMemory` | `InMemoryTransferService` |

If the implementation is not tied to a specific technology, use a semantic qualifier that describes the character of the implementation:

| Qualifier | Meaning | Example |
|---|---|---|
| `Simple` | Minimal, no-frills | `SimpleNotificationService` |
| `Default` | Standard fallback when nothing more specific is configured | `DefaultReconciliationService` |
| `Generic` | General-purpose, not specialized | `GenericReportExporter` |
| `Standard` | Canonical, reference implementation | `StandardInvoiceCalculator` |

Never use the `XxxImpl` suffix — it carries no information about what the implementation does or how.

### JPA Entities

Use the `XxxEntity` suffix for JPA-mapped classes. It distinguishes the persistence object from the domain model object:

```
TransferEntity   ← JPA-mapped class (lives in jpa/ subpackage)
Transfer         ← domain model class (lives in model/ subpackage)
```

### JPA Repositories

Use `XxxRepository`. Do not embed the entity class name: `TransferRepository`, not `TransferEntityRepository`. The entity type is already visible from the generic parameter.

### Constants

When you have a value that needs a name, follow this order:

1. **Use an enum** if the value belongs to a finite, named set: transaction types, statuses, roles, error codes.
2. **Put it on the class that owns it** if only one class uses it: a `MAX_MEMO_LENGTH` constant belongs on `Transfer`, not in a separate file.
3. **Create a focused constants class** with a descriptive name if you have a coherent group of related literals with no single owning class — e.g. `StripeErrorCodes`, `MixpanelEvents`, `ApiPaths`. Every constant in the class must relate to the same concept.
4. **Never create a catch-all `XxxConstants` dump.** A class named `RevenueRecognitionConstants` or `AppConstants` that collects unrelated values from across the application is not a constants class — it is a junk drawer.

**Enums nested inside constants classes** should be top-level classes instead. `StripeConstants.Scope` should be `StripeScope`; `ShopifyConstants.SupportedGateway` should be `ShopifyGateway`.

**Configuration values** (pool sizes, intervals, limits, batch sizes) are not constants — they are configuration. They belong in a `@ConfigurationProperties` class, not hardcoded in source.

**What bad constants look like** — all found in the codebase:

```kotlin
const val INT_100 = 100                    // naming a number after itself — pure noise
const val ONE_HOUR_IN_MILLISECONDS = 3600  // wrong: 3600 is seconds, not milliseconds
const val CORE_POOL_SIZE_FOUR = 3          // name lies about the value
```

*Reference: Effective Java (Bloch), Item 22 — "Use interfaces only to define types". Enums first, focused utility class second, interface never.*

### Tests

> Testing guidelines have been moved to dedicated documents:
> - [Developer Testing Guide](testing/DEVELOPER_GUIDE.md) — how to write good unit and integration tests
> - [Reviewer Testing Guide](testing/REVIEWER_GUIDE.md) — how to evaluate tests in PRs

### Other Suffixes

These suffixes are already used consistently across all services and should be maintained:

| Role | Suffix | Example |
|---|---|---|
| Controller | `Controller` | `TransferController` |
| Object converter (between layers) | `Converter` | `TransferConverter` |
| Search / filter criteria object | `SearchParameters` | `TransferSearchParameters` |
| Application event | `XxxCreatedEvent`, `XxxDeletedEvent`, `XxxUpdatedEvent` | `TransferCreatedEvent` |
| Exception | `Exception` | `TransferValidationException` |
| Spring configuration properties | `Properties` | `MandrillConfigurationProperties` |

---

## Code Reuse and Abstraction

DRY ("Don't Repeat Yourself") is one of the most widely taught principles in software development — and one of the most frequently misapplied. The problem is not the principle itself, but when and how it is applied.

### Premature Abstraction

Premature abstraction is the act of extracting "reusable" code before you have working, tested implementations to base it on. A developer sees that two pieces of logic look similar, creates a shared utility or base class, and makes both depend on it — before either of them is fully built or tested.

This feels productive. In practice, it causes several problems:

- The abstraction is based on superficial similarity, not conceptual identity. The two callers appear to share something today but will diverge tomorrow — and now you cannot change either without breaking the other.
- Shared code without proven callers has no real requirements. It is written speculatively, which means it is usually either too generic (and therefore complex) or too specific (and therefore not actually reusable).
- There are typically no tests, because the abstraction was not built against real, working use cases.
- Once other code depends on it, it is difficult to delete or change — even when it clearly does not fit anymore.

The result is a codebase polluted with `common/`, `shared/`, and `util/` packages full of code that nobody fully owns, that everything partially depends on, and that nobody dares to touch.

### The Rules

**Make it work first.** No abstraction before a working, tested solution exists. Build the concrete implementation, make it correct, cover it with tests — then, and only then, consider whether anything is worth extracting.

**The Rule of Three.** Do not extract shared code until you have three real, independent callers with working implementations. One is a solution. Two might be a coincidence. Three is a pattern worth abstracting.

**Duplication is cheaper than the wrong abstraction.** Tolerate copy-paste until the pattern is proven. Two copies of similar code that can evolve independently are safer than one "reusable" abstraction that ties them together prematurely. Wrong abstractions are hard to undo; duplication is easy to fix once the right pattern becomes clear.

**Challenge every shared class.** Any new class in a `common/`, `shared/`, or `util/` package must answer two questions in code review: *what are its proven callers?* and *what happens if those callers need to diverge?* If there is no good answer, the abstraction is premature.

### What Premature Abstraction Looks Like

Watch for these signs in code review:

- A utility or base class is introduced before its callers are fully implemented or tested.
- A "reusable" method accepts boolean flags or strategy parameters to handle slightly different cases for different callers — this is a sign the abstraction does not actually fit either caller cleanly.
- Two callers share code but one (or both) already has workarounds around the shared logic.
- Shared code lives in `common/` or `util/` with no clear owner and no tests.
- A developer describes code as "we'll need this later" — YAGNI (*You Aren't Gonna Need It*) applies directly here.

---

## What NOT to Do

### The Grails Anti-Pattern

[Grails](https://grails.org/) is a JVM web framework built on top of Spring Boot and Groovy. It was designed for rapid prototyping and MVP development. To make that fast, Grails enforces a fixed folder structure based on **technical layers**, not domain concepts:

```
grails-app/
├── controllers/    ← all HTTP controllers for the entire app
├── services/       ← all services for the entire app
├── domain/         ← all database-mapped domain classes
└── views/          ← all templates
```

This structure works fine for a small app. But as the application grows, it becomes a problem: all `controllers/` belong to every domain at once, all `services/` are mixed together, and there are no natural boundaries between modules.

The result is that domain boundaries erode. A `PaymentService` starts importing from `InvoiceService` which imports from `AccountService` — and soon everything depends on everything. This is how you get a **monolith**: a codebase where you cannot change one part without breaking another, and where testing a single feature requires loading the entire application.

**The rule:** organize code by domain, not by technical layer. Each domain module owns its controllers, services, repositories, and models — grouped together in one package, not spread across separate top-level folders.

> See also: **Premature Abstraction** — applying the same layer-first thinking inside a module (e.g. a shared `common/` package that everything imports) leads to the same coupling problem at a smaller scale.

---

## Revision History

| Date | Authors | Changes |
|------|---------|---------|
| 2026-03-03 | Alexey Sergeev | Initial draft: Domain Model section covering encapsulation, public API vs. internal implementation, transfer module example, and package conventions. Application Logic section. Grails anti-pattern in What NOT to Do. |
| 2026-03-03 | Alexey Sergeev | Grammar, style, and clarity pass. Added "From Interface to Implementation" explanation with multiple examples. Extracted Grails note into dedicated section with framework explanation. Added forward reference to premature abstraction. |
| 2026-03-03 | Alexey Sergeev | Added Code Reuse and Abstraction section covering premature abstraction antipattern, the Rule of Three, YAGNI, and code review smell list. Added Revision History. |
| 2026-03-03 | Alexey Sergeev | Renamed "From Interface to Storage" to "From Interface to Implementation: Hiding the Details". Expanded with multiple examples: email sending, data persistence, external payment gateway. |
| 2026-03-03 | Alexey Sergeev | Added Naming Conventions section: packages, classes, service interfaces/implementations, JPA entities/repositories. |
| 2026-03-03 | Alexey Sergeev | Added Constants sub-section: decision order (enum → owning class → focused class), antipatterns with codebase examples. |
| 2026-03-03 | Alexey Sergeev | Renamed document to "Synder Backend Development Guidelines". |
| 2026-03-03 | Alexey Sergeev | Added Other Suffixes sub-section to Naming Conventions. |
| 2026-03-03 | Alexey Sergeev | Added Tests and Other Suffixes sub-sections to Naming Conventions. |
| 2026-03-13 | Alexey Sergeev | Added decomposition notice. Replaced Tests sub-section with links to dedicated testing guides (Developer Guide, Reviewer Guide). |
