---
name: backend-code-review
description: "Review backend PRs (Java/Groovy/Kotlin, Spring/Hibernate/JPA/Grails) for architecture, naming, coupling, abstraction, and test quality. Use when: (1) reviewing a GitHub PR for a JVM backend project, (2) asked to do a code review, (3) evaluating test coverage or test quality in a PR, (4) checking architecture or naming conventions. Usage: /backend-code-review <PR-URL-or-owner/repo#number> [--focus architecture|tests|naming|all] [--strict]"
metadata: { "openclaw": { "emoji": "🔍", "requires": { "bins": ["gh", "curl"] } } }
---

# Backend Code Review

Review JVM backend PRs using a structured multi-pass approach covering architecture, naming, coupling, abstraction, and test quality.

## Usage Note

Treat this as a **skill guide**, not a shell command. Do **not** run `/backend-code-review` in a terminal. Instead, read and follow the steps in this SKILL.md.

## Arguments

| Flag | Default | Description |
|------|---------|-------------|
| (positional) | required | PR URL or `owner/repo#number` |
| --focus | all | Review scope: `architecture`, `tests`, `naming`, or `all` |
| --strict | false | Blocking-only comments (skip non-blocking suggestions) |

## Review Workflow

### Step 1 — Gather Context

```bash
# Fetch PR diff
gh pr diff <PR-NUMBER> -R <OWNER/REPO>

# Fetch PR description and metadata
gh pr view <PR-NUMBER> -R <OWNER/REPO> --json title,body,files,additions,deletions
```

Read the PR description for linked tickets, business context, and intent.

### Step 2 — Architecture Review (5 passes)

Read `{baseDir}/../../architecture/REVIEWER_GUIDE.md` and follow its 5-pass structure:

1. **Structure** — package placement, domain module organization, Grails anti-pattern detection
2. **Naming** — service impl naming (Jpa/Default, never Impl), entity naming, constants, packages
3. **Variable & Method Hygiene** — apply Fowler's Extract/Inline Variable test, flag long methods
4. **Coupling** — import analysis, dependency direction, cross-module violations, circular deps
5. **Abstraction** — premature extraction, shared/common abuse, interface necessity, Rule of Three

#### Pass 3 — Variable & Method Hygiene

Apply Fowler's Extract Variable vs Inline Variable test to long methods:

**Inline candidates** (flag as non-blocking):
- Variable assigned from a simple getter and the getter name already self-documents
  (`quantityOfProduct = line.getQuantity()` → just use `line.getQuantity()`)
- Variable used only once
- Variable name is essentially the same as the method name

**Keep candidates** (don't flag):
- Variable assigned from a computed/constructed expression (`new Money(...)`, builder patterns)
- Variable that names a concept the raw expression doesn't convey
- Variable used 3+ times AND the expression is non-trivial

**Long method signals:**
- `@SuppressWarnings("MethodLength")` or equivalent → always flag
- Method doing N distinct jobs → propose Extract Method decomposition with concrete sketch
- \>10 local variables in a single method → likely decomposition candidate

**Git history check** (when method is >100 lines):
- Run `git log --oneline --follow` on the file
- Check if the method grew incrementally (accretive pattern) vs was designed long
- Note in review if the pattern was inherited, not designed

### Step 3 — Test Review (3 passes)

Read `{baseDir}/../../testing/REVIEWER_GUIDE.md` and follow its 3-pass structure:

1. **Completeness** — coverage assessment, code type vs test type, Right-BICEP, ZOM, boundary values
2. **Quality** — Four Pillars evaluation, testing style, test smells, mock usage, anti-patterns
3. **Correctness** — empty verification, tautological tests, leaking domain knowledge, exception specificity

### Step 4 — Dev Rules Check

Read `{baseDir}/../../DEV_RULES.md` for the index of all conventions. Follow links to specific sections as needed.

### Step 5 — Compose Review

Organize findings by severity:

**Blocking (request changes):**
- Missing tests for new/changed behavior
- Dependency direction violations
- Cross-module implementation access
- New `XxxImpl` classes
- Brittle or meaningless tests
- Asserting on stubs

**Non-blocking (suggestions):**
- Naming improvements on existing code
- Minor test smell cleanup
- Migration opportunities
- Shared utility naming: methods in `*Util` / shared classes that encode domain rules need
  precise names. Flag generic names (`build*`, `create*`, `process*`) when the method does
  something specific. Propose rename with Javadoc if the method will be used across 3+ services.

**Positive feedback:**
- Good patterns worth acknowledging

### Step 6 — Format Output

Use comment templates from the reference guides. Each comment should:
- Use the appropriate emoji prefix (🧪 📦 📛 🔀 🔗 ⏳ etc.)
- Reference specific file and line numbers
- Explain WHY it's a problem, not just WHAT
- Suggest a concrete fix

## Key Rules

- **Technology stack**: Java, Groovy, Kotlin. Frameworks: Spring, Hibernate, JPA, Grails.
- **Naming**: `JpaXxx` / `JdbcXxx` / `HttpXxx` / `DefaultXxx` — NEVER `XxxImpl`.
- **Entity naming**: `XxxMapping` in accounting repo, `XxxEntity` in others.
- **Domain model**: Plain objects, no JPA annotations. Separate from JPA entities.
- **Package structure**: Domain-first (`{domain}/model/`, `{domain}/service/`, `{domain}/jpa/`), not layer-first.
- **Cross-module**: Only through public API (service interface + domain model). Never import `.jpa.` from another module.
- **Tests**: Output-based > state-based > communication-based. Never assert on stubs. Hardcode expected values.
- **Mocks**: Stub for data, Mock for boundary interactions. >8 mocks = design smell.
- **Spock**: `@Unroll` on parameterized tests, `given/when/then` structure, `*Spec` naming.
- **JUnit 5**: Backtick method names, `@Nested` groups, `*Test` naming.
