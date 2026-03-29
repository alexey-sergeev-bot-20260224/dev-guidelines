---
name: backend-code-review
description: "Review backend PRs (Java/Groovy/Kotlin, Spring/Hibernate/JPA/Grails) for architecture, naming, coupling, abstraction, and test quality. Use when: (1) reviewing a GitHub PR for a JVM backend project, (2) asked to do a code review, (3) evaluating test coverage or test quality in a PR, (4) checking architecture or naming conventions. Usage: /backend-code-review <PR-URL-or-owner/repo#number> [--focus architecture|tests|naming|all] [--strict]"
metadata: { "openclaw": { "emoji": "🔍", "requires": { "bins": ["gh", "curl"] } } }
---

# Backend Code Review

Review JVM backend PRs using a structured multi-pass approach covering architecture, naming, coupling, abstraction, implementation correctness, simplification, and test quality.

## Usage Note

Treat this as a **skill guide**, not a shell command. Do **not** run `/backend-code-review` in a terminal. Instead, read and follow the steps in this SKILL.md.

## Arguments

| Flag | Default | Description |
|------|---------|-------------|
| (positional) | required | PR URL or `owner/repo#number` |
| --focus | all | Review scope: `architecture`, `tests`, `naming`, `simplification`, `implementation`, or `all` |
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

### Step 3 — Implementation Correctness Review

Verify the code actually achieves its stated goal. This is separate from architecture — focus on "does it work?" not "is it structured well?"

#### 3.1 Requirement Coverage

Using the PR description, linked ticket, and commit messages as the stated goal:

- [ ] Does the implementation address **all** aspects of the requirement?
- [ ] Are there scenarios mentioned in the ticket but not handled in the code?
- [ ] Could the code fail to achieve the goal under certain conditions?

#### 3.2 Wiring and Integration

- [ ] Are new components properly registered? (`@Service`, `@Component`, `@Bean` present where needed)
- [ ] Are new endpoints added to routing / controller mappings?
- [ ] Are new config properties documented / have defaults?
- [ ] Are DB migrations present and correct for new entities/columns?
- [ ] Are new dependencies injected (not just declared)?

#### 3.3 Logic Flow

- [ ] Does data flow correctly from input to output?
- [ ] Are transformations / mappings correct? (field-to-field, type conversions, null handling)
- [ ] Is state managed properly? (no stale reads, no lost updates, correct transaction boundaries)
- [ ] Are return values used correctly by callers?

#### 3.4 Completeness Gaps

Watch for code that compiles and passes tests but is silently incomplete:

- Missing `else` branches that should handle a case
- `switch`/`when` without default that will silently skip new enum values
- Try-catch that swallows exceptions without logging or rethrowing
- Event listeners registered but not wired to the event bus
- Async operations without error handling (fire-and-forget that should report failures)
- Feature flags or config toggles referenced but never checked

### Step 4 — Simplification Review

Detect over-engineering — code that works but is more complex than necessary.

#### 4.1 Excessive Abstraction

| Pattern | What to look for | Action |
|---------|-----------------|--------|
| Pass-through wrapper | Method just delegates to another method with the same signature | Flag — inline or justify |
| Factory for single impl | Factory/provider pattern when only one concrete type exists | Flag — use `new` or direct injection |
| Interface on producer side | Interface defined where implemented, not where consumed | Flag — move or remove |
| Layer cake | handler → service → repository where each just passes through | Flag — collapse unnecessary layers |
| DTO/Mapper overkill | Multiple types representing the same data with conversion functions | Flag if fewer than 3 consumers need the abstraction |

#### 4.2 Premature Generalization

| Pattern | What to look for | Action |
|---------|-----------------|--------|
| Generic solution for specific problem | Event bus for one event type, strategy pattern for one strategy | Flag — use direct call |
| Config object for 2-3 options | Options/builder pattern when 2-3 constructor params suffice | Flag — simplify |
| Plugin architecture for fixed functionality | Extension points that nothing extends | Flag — remove hooks |
| Overloaded struct | One class handling all variations via many optional fields | Flag — consider splitting |

#### 4.3 Unnecessary Fallbacks

| Pattern | What to look for | Action |
|---------|-----------------|--------|
| Fallback that never triggers | Default branch with conditions never met | Flag — remove dead code |
| Dual implementations | Old + new code paths when old has no callers | Flag — remove old path |
| Silent fallbacks hiding problems | Catching errors and falling back instead of failing fast | Flag — fail fast or log at minimum |
| Legacy compatibility mode | Old code path always disabled behind a flag | Flag — remove if truly dead |

#### 4.4 Premature Optimization

| Pattern | What to look for | Action |
|---------|-----------------|--------|
| Caching rarely-accessed data | Cache for data read at startup or once per request | Flag — measure first |
| Custom data structures | Complex structures when standard collections work | Flag — justify with benchmarks |
| Connection pooling overkill | Complex pooling for single-connection scenarios | Flag — simplify |
| Manual batching | Hand-rolled batch processing when framework provides it | Flag — use framework support |

**Severity:** Simplification findings are typically **non-blocking suggestions** unless the over-engineering introduces a maintenance hazard or obscures a bug.

### Step 5 — Test Review (3 passes)

Read `{baseDir}/../../testing/REVIEWER_GUIDE.md` and follow its 3-pass structure:

1. **Completeness** — coverage assessment, code type vs test type, Right-BICEP, ZOM, boundary values
2. **Quality** — Four Pillars evaluation, testing style, test smells, mock usage, anti-patterns
3. **Correctness** — empty verification, tautological tests, leaking domain knowledge, exception specificity

### Step 6 — Documentation Check (lightweight)

Quick scan for documentation gaps. Skip if the PR is a pure internal refactor with no user-visible changes.

- [ ] **New API endpoints or config properties** — are they documented anywhere? (README, Swagger annotations, config examples)
- [ ] **Changed behavior** — does any existing documentation (Javadoc, README, wiki) now describe wrong behavior?
- [ ] **New environment variables or feature flags** — documented with defaults and purpose?
- [ ] **Migration steps** — if the change requires deployment steps beyond "deploy," are they noted?

**Severity:** Documentation findings are **non-blocking** unless the missing docs would cause deployment failures or user confusion.

### Step 7 — Dev Rules Check

Read `{baseDir}/../../DEV_RULES.md` for the index of all conventions. Follow links to specific sections as needed.

### Step 8 — Verify Findings (false-positive filter)

Before composing the review, **verify every finding** against actual code context. For each finding from Steps 2-7:

1. **Re-read the actual code** at the exact file and line — not just the diff hunk, but 20-30 lines of surrounding context
2. **Check for existing mitigations** — is the issue already handled elsewhere? (parent class, AOP, framework convention, config)
3. **Check if it's pre-existing** — did this pattern exist before the PR? If yes, still note it but mark as `[pre-existing]` and make non-blocking
4. **Classify each finding:**

| Classification | Meaning | Action |
|---|---|---|
| **CONFIRMED** | Issue is real, exists in context, no mitigation | Include in review |
| **FALSE POSITIVE** | Doesn't exist, already mitigated, or misread the code | Discard silently |
| **PRE-EXISTING** | Real issue but not introduced by this PR | Include as non-blocking observation |

**Why this step matters:** AI-generated reviews are prone to flagging things that look like issues in a diff snippet but are handled by framework conventions, parent class behavior, or adjacent code not visible in the hunk. This step prevents noise that erodes reviewer credibility.

### Step 9 — Compose Review

Organize **confirmed** findings by severity:

**Blocking (request changes):**
- Missing tests for new/changed behavior
- Dependency direction violations
- Cross-module implementation access
- New `XxxImpl` classes
- Brittle or meaningless tests
- Asserting on stubs
- Implementation correctness gaps (code doesn't achieve stated goal)
- Missing wiring (component not registered, endpoint not mapped)

**Non-blocking (suggestions):**
- Naming improvements on existing code
- Minor test smell cleanup
- Migration opportunities
- Simplification opportunities (over-engineering, unnecessary abstraction)
- Documentation gaps
- Pre-existing issues (mark with `[pre-existing]`)
- Shared utility naming: methods in `*Util` / shared classes that encode domain rules need
  precise names. Flag generic names (`build*`, `create*`, `process*`) when the method does
  something specific. Propose rename with Javadoc if the method will be used across 3+ services.

**Positive feedback:**
- Good patterns worth acknowledging

### Step 10 — Format Output

Use the structured finding template for every comment. Each finding MUST include all four fields:

```
<emoji> **<Category>.**
📍 Location: `<file>#L<line>` (or line range)
🔍 Issue: <clear description of what's wrong>
💥 Impact: <how this affects the code — bugs, maintenance, performance, correctness>
💡 Fix: <specific, actionable suggestion>
```

**Emoji prefixes by category:**
- 🧪 Test issue
- 📦 Package placement
- 📛 Naming violation
- 🔀 Dependency direction
- 🔗 Cross-module coupling
- ⏳ Premature abstraction
- 🧩 Implementation correctness
- 🔧 Simplification opportunity
- 📄 Documentation gap
- ⚠️ Weak assertion / silent failure
- ✅ Positive feedback (no Impact/Fix needed)

**Example:**
```
🧩 **Implementation correctness.**
📍 Location: `TransferService.java#L145`
🔍 Issue: New `processRefund()` method is registered as `@Transactional(readOnly = true)` but performs a write operation (updates refund status).
💥 Impact: Write will silently fail or throw on some JPA providers; data corruption risk in production.
💡 Fix: Change to `@Transactional` (remove `readOnly = true`), or split the read and write into separate transactional methods.
```

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
