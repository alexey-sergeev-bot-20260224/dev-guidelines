# Synder Backend Development Guidelines

> **Note:** This document is being gradually decomposed into focused, topic-specific guides.
> As dedicated guides are created for each area (testing, architecture, naming, etc.),
> the corresponding sections here will be replaced with links.
> See the [`dev-guidelines`](https://github.com/alexey-sergeev-bot-20260224/dev-guidelines) repository for the full collection.

This document covers JVM-based applications and services written in Java, Groovy, or Kotlin, using libraries and frameworks such as Spring, Hibernate, and JDBC.

---

## Domain Model

> Architecture guidelines have been moved to dedicated documents:
> - [Architecture Developer Guide](architecture/DEVELOPER_GUIDE.md) — domain-first organization, public API vs implementation, dependency direction, module design, migration patterns
> - [Architecture Reviewer Guide](architecture/REVIEWER_GUIDE.md) — how to evaluate structure, naming, coupling, and abstraction in PRs

---

## Application Logic

> See [Architecture Developer Guide — §10 Application Layer and Orchestration](architecture/DEVELOPER_GUIDE.md#10-application-layer-and-orchestration) — controllers as orchestrators, cross-module coordination.

---

## Naming Conventions

> See [Architecture Developer Guide — §5 Naming Conventions](architecture/DEVELOPER_GUIDE.md#5-naming-conventions) — packages, classes, service interfaces/implementations, JPA entities/repositories, constants, tests, and other suffixes.

### Tests

> Testing guidelines have been moved to dedicated documents:
> - [Developer Testing Guide](testing/DEVELOPER_GUIDE.md) — how to write good unit and integration tests
> - [Reviewer Testing Guide](testing/REVIEWER_GUIDE.md) — how to evaluate tests in PRs

---

## Code Reuse and Abstraction

> See [Architecture Developer Guide — §6 Code Reuse and Abstraction](architecture/DEVELOPER_GUIDE.md#6-code-reuse-and-abstraction) — premature abstraction, Rule of Three, YAGNI, shared/common package hygiene.

---

## What NOT to Do

> See [Architecture Developer Guide — §7 The Grails Anti-Pattern](architecture/DEVELOPER_GUIDE.md#7-the-grails-anti-pattern) — layer-first vs domain-first organization, the synder monolith case study, migration guidance.

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
| 2026-03-13 | Alexey Sergeev | Replaced Domain Model, Application Logic, Naming Conventions, Code Reuse and Abstraction, and What NOT to Do sections with links to architecture guides (DEVELOPER_GUIDE.md, REVIEWER_GUIDE.md). DEV_RULES.md is now an index document. |
