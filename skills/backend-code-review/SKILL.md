---
name: backend-code-review
description: "Review backend pull requests for JVM services with priority on correctness, safety, migrations, tests, architecture, and maintainability. Use for GitHub PR review, re-review after author updates, and evaluation of backend changes affecting contracts, persistence, integrations, concurrency, or test quality."
metadata: { "openclaw": { "emoji": "🔍", "requires": { "bins": ["gh", "curl"] } } }
---

# Backend Code Review

Review backend PRs using a structured multi-pass process.

**Priority order:**
1. Intent and correctness
2. Risk and safety
3. Tests
4. Architecture and maintainability
5. Simplification
6. Conventions and naming

Do not start with style or naming if there may be correctness, data integrity, migration, or operational risks.

## Usage Note

Treat this as a **skill guide**, not a shell command. Do **not** run `/backend-code-review` in a terminal. Instead, read and follow the steps in this SKILL.md.

## Arguments

| Flag | Default | Description |
|------|---------|-------------|
| (positional) | required | PR URL or `owner/repo#number` |
| `--focus` | `all` | Review scope: `all`, `correctness`, `risk`, `tests`, `architecture`, `naming`, `simplification`, `docs` |
| `--strict` | `false` | Report only findings that materially affect merge confidence (usually `blocker` and `major`) |
| `--rereview` | `auto` | If prior review discussion exists, treat as re-review |

## Review Principles

- Review the **change**, not the author.
- Prefer **high-signal findings** over exhaustive nitpicking.
- Distinguish: **introduced by this PR** vs **pre-existing** vs **unclear due to missing context**.
- Do not block the PR for unrelated cleanup unless it creates immediate risk.
- Treat project conventions seriously, but do not confuse convention violations with correctness defects.
- When context is insufficient, say so explicitly instead of guessing.
- If the PR is too large for reliable review, state that review confidence is reduced and prioritize the highest-risk areas first.
- Do not demand large refactors inside a small PR unless the current design is genuinely unsafe.

## Severity Model

Every finding must carry one severity:

| Severity | Meaning |
|---|---|
| **blocker** | Must be fixed before merge: correctness, safety, security, data integrity, contract, migration, or critical test issue |
| **major** | Important issue that creates substantial maintainability, operability, or test risk; may block merge when it materially reduces confidence in safe deployment |
| **minor** | Legitimate improvement; should not block merge |
| **nit** | Small polish issue |
| **question** | Reviewer needs clarification before concluding |
| **pre-existing** | Real issue, but not introduced by this PR |

## Final Verdict

Every review must end with one explicit verdict:

| Verdict | When to use |
|---|---|
| **approve** | No unresolved blocker or major findings |
| **approve with nits** | Only `minor`, `nit`, or `pre-existing` findings remain |
| **request changes** | Any unresolved `blocker`, or any `major` that materially reduces confidence in safe merge (e.g., high production risk, correctness ambiguity, operational gaps, or insufficient tests for high-risk behavior) |
| **insufficient context to approve safely** | Cannot review confidently due to missing critical context |

---

# Review Workflow

## Step 1 — Gather Context

### 1.1 PR Data

```bash
# Fetch PR diff
gh pr diff <PR-NUMBER> -R <OWNER/REPO>

# Fetch PR description and metadata
gh pr view <PR-NUMBER> -R <OWNER/REPO> --json title,body,files,additions,deletions
```

Read the PR description, title, branch name, and commit messages for intent.

### 1.2 Jira / Ticket Context

Extract ticket IDs from the PR title, branch name, and body. Look for patterns: `SD-\d+`, `SET-\d+`, `SB-\d+`, or any `[A-Z]+-\d+` Jira key.

If a ticket ID is found, fetch it via the Jira REST API:

```bash
# Fetch ticket with description and recent comments
curl -s -u "$JIRA_EMAIL:$JIRA_TOKEN" \
  "${JIRA_URL}/rest/api/3/issue/<TICKET-ID>?fields=summary,status,description,comment,labels,priority" \
  -H "Accept: application/json"
```

**Required environment:** `JIRA_URL`, `JIRA_EMAIL`, `JIRA_TOKEN` must be available (e.g. from a config file or environment variables).

From the ticket, extract:
- **Acceptance criteria** — what "done" looks like from the business side
- **Scope boundaries** — what's explicitly in/out of scope
- **Business rules** — domain logic the code must implement
- **Edge cases mentioned** — scenarios the author should have handled

Use this context throughout the review:
- **Step 3 (Correctness)**: verify the code satisfies acceptance criteria and business rules
- **Step 5 (Simplification)**: distinguish intentional complexity (driven by business rules) from over-engineering
- **Step 6 (Tests)**: check that test scenarios cover the ticket's edge cases

If no ticket ID is found (e.g., `NO-TASK` prefix or pure refactor), note it and proceed — not every PR has a ticket, and that's fine. If a ticket exists but cannot be fetched, note the limitation and avoid strong claims about missed requirements unless the gap is obvious from the code itself.

### 1.3 Prior Review Context (re-reviews only)

Skip this step on first review. On re-reviews (2nd+ round), gather the full PR conversation before running any review passes.

```bash
# Fetch PR comments (general + inline review comments)
gh pr view <PR-NUMBER> -R <OWNER/REPO> --json comments,reviews,reviewThreads
```

For each finding from the previous review round, classify the author's response:

| Classification | Meaning | Action |
|---|---|---|
| **Resolved** | Author changed the code to address the finding | Drop — don't re-raise |
| **Explained** | Author provided valid domain reasoning why the current code is correct | Accept — don't re-raise. Note the explanation for your own context |
| **Disputed** | Author disagrees but reasoning is weak or doesn't address the core concern | Re-evaluate with new context. Re-raise with adjusted framing if still valid |
| **Ignored** | No response, code unchanged | Re-raise if the finding still exists in the current diff |

**Guidelines:**
- Authors have domain knowledge the reviewer lacks. Give their explanations genuine consideration — but not blind acceptance. If an explanation contradicts established patterns or hides a real bug, flag it anyway.
- When re-raising a disputed finding, acknowledge the author's point and explain why the concern still stands. Don't just repeat the original comment verbatim.
- Carry these classifications into Step 9 (Verify Findings) — a finding already explained by the author with valid reasoning is a false positive.

**Output:** A list of prior findings with their classifications. Use this as a filter throughout Steps 2–8 to avoid duplicate work.

### 1.4 Repository-Specific Guidance

Load project conventions after understanding the change intent:

- `{baseDir}/../../DEV_RULES.md` — index of all conventions
- `{baseDir}/../../architecture/REVIEWER_GUIDE.md` — architecture and naming rules
- `{baseDir}/../../testing/REVIEWER_GUIDE.md` — test review criteria
- Repo-specific files under `{baseDir}/../../architecture/repo-specific/`, if present

Treat repository-specific guidance as strong constraints where explicitly established, heuristics elsewhere. Do not let naming/package rules overshadow correctness and safety review.

### 1.5 Author Context Expectations

For medium/high-risk PRs, the reviewer should expect the PR to provide:
- change intent and linked ticket/business context
- migration notes (if DB changes are involved)
- rollout/deployment notes (if non-trivial)
- risky areas the author is aware of
- test notes (what was tested, what wasn't)
- known limitations or planned follow-ups

If a risky PR lacks essential review context, request clarification early instead of pretending to review with high confidence.

### 1.6 Oversized PR Behavior

If the PR is very large (guideline: >1000 changed lines across many files, or >15 files with non-trivial logic):

- State that exhaustive review confidence is reduced.
- Review highest-risk files first: data, money, auth, migrations, integrations.
- State explicitly in the review summary which areas were reviewed deeply and which were reviewed lightly or skipped.
- Recommend splitting or staged follow-up review if the PR covers multiple independent concerns.
- Do not pretend the review was exhaustive when it was not.

---

## Step 2 — Summarize Intent and Risk

Before producing any finding, write a short internal summary:

- What is this PR trying to change?
- Which parts are highest risk?
- Which parts are low risk?
- What must be true for this PR to be safe?

Build an initial risk map across these domains:
- domain logic risk
- data / migration risk
- API / contract risk
- concurrency risk
- external integration risk
- testing risk
- architecture / convention risk

If the PR affects any of these, treat it as high risk:
- money or accounting correctness
- persistence or data migration
- retries / async processing
- external integrations
- public APIs
- auth / permissions
- multi-tenant isolation

Use this summary to drive the review order — review the highest-risk areas first.

---

## Step 3 — Correctness and Design Review

Focus on: **does the change actually do the right thing?**

Do this before naming or structural polish.

### 3.1 Requirement Coverage

Using the PR description, linked ticket, and commit messages as the stated goal:

- [ ] Does the implementation address **all** aspects of the requirement?
- [ ] Are there scenarios mentioned in the ticket but not handled in the code?
- [ ] Could the code fail to achieve the goal under certain conditions?
- [ ] Is any part only partially implemented?
- [ ] Is any behavior silently skipped?

### 3.2 Wiring and Integration

- [ ] Are new components properly registered? (`@Service`, `@Component`, `@Bean` present where needed)
- [ ] Are new endpoints added to routing / controller mappings?
- [ ] Are new config properties documented / have defaults?
- [ ] Are DB migrations present and correct for new entities/columns?
- [ ] Are new dependencies injected (not just declared)?
- [ ] Are new paths reachable from the actual application flow?

### 3.3 Logic Flow

- [ ] Does data flow correctly from input to output?
- [ ] Are transformations / mappings correct? (field-to-field, type conversions, null handling)
- [ ] Is state managed properly? (no stale reads, no lost updates, correct transaction boundaries)
- [ ] Are return values used correctly by callers?

### 3.4 Completeness Gaps

Watch for code that compiles and passes tests but is silently incomplete:

- Missing `else` branches that should handle a case
- `switch`/`when` without default that will silently skip new enum values
- Try-catch that swallows exceptions without logging or rethrowing
- Event listeners registered but not wired to the event bus
- Async operations without error handling (fire-and-forget that should report failures)
- Feature flags or config toggles referenced but never checked
- Config added but not read
- Result of a new method ignored by caller

### 3.5 Design Quality

Check:
- Is the design appropriate for the current problem size?
- Is complexity justified?
- Are responsibilities reasonably separated?
- Does the design make future changes easier or harder?

Do not force redesign for taste alone. Flag design only when it materially affects correctness, clarity, testability, coupling, or operational safety.

---

## Step 4 — Backend Risk and Safety Review

Mandatory pass for backend PRs. Scan each subsection; skip subsections not relevant to the current PR.

### 4.1 API and Contract Safety

- [ ] Public API changes are backward-compatible (or justified breaking change)
- [ ] Request/response field renames/removals are safe
- [ ] Validation changes don't break existing clients
- [ ] Error contract is preserved
- [ ] Pagination/sorting/filter behavior changes are documented

### 4.2 Data and Migration Safety

- [ ] New tables / columns / indexes / constraints are correct
- [ ] Nullable vs non-nullable changes are deploy-safe
- [ ] Default values are present where needed
- [ ] Backfill is handled if required
- [ ] Old and new code can coexist during rolling deploy
- [ ] Rollback is safe (expand-contract pattern where needed)
- [ ] No deployment-order dependency between migration and code

### 4.3 Concurrency and Idempotency

- [ ] No duplicate processing risk
- [ ] Race conditions checked (especially for check-then-act patterns)
- [ ] Lock boundaries are correct
- [ ] Retry is safe (idempotent handlers/jobs/webhooks)
- [ ] Out-of-order events are handled
- [ ] Transaction + async interaction is correct
- [ ] No catch-and-recover on constraint violations (in JPA/Hibernate contexts, catching constraint exceptions corrupts the session — use check-before-insert instead)

### 4.4 External Integration Safety

- [ ] Timeouts are configured
- [ ] Retries are bounded and safe
- [ ] Partial failure is handled
- [ ] Response validation is present
- [ ] Idempotency keys are used where needed
- [ ] Remote failures are logged

### 4.5 Operational Safety

- [ ] Metrics exist for critical new behavior
- [ ] Logs are useful but not noisy
- [ ] Feature flags have safe defaults
- [ ] Config is discoverable
- [ ] Deployment notes exist if non-trivial

### 4.6 Security and Privacy

- [ ] Auth/authz effects are correct
- [ ] Tenant isolation is preserved
- [ ] Input validation is present
- [ ] No unsafe deserialization
- [ ] No secrets or PII in logs
- [ ] No overexposure in APIs
- [ ] Permission checks are present where needed

If the change is security-sensitive and confidence is low, escalate per the Escalation Rules below.

---

## Step 5 — Simplification Review

Detect over-engineering — code that works but is more complex than necessary.

### 5.1 Excessive Abstraction

| Pattern | What to look for | Action |
|---------|-----------------|--------|
| Pass-through wrapper | Method just delegates to another method with the same signature | Flag — inline or justify |
| Factory for single impl | Factory/provider pattern when only one concrete type exists | Flag — use `new` or direct injection |
| Interface on producer side | Interface defined where implemented, not where consumed | Flag — move or remove |
| Layer cake | handler → service → repository where each just passes through | Flag — collapse unnecessary layers |
| DTO/Mapper overkill | Multiple types representing the same data with conversion functions | Flag if fewer than 3 consumers need the abstraction |

### 5.2 Premature Generalization

| Pattern | What to look for | Action |
|---------|-----------------|--------|
| Generic solution for specific problem | Event bus for one event type, strategy pattern for one strategy | Flag — use direct call |
| Config object for 2-3 options | Options/builder pattern when 2-3 constructor params suffice | Flag — simplify |
| Plugin architecture for fixed functionality | Extension points that nothing extends | Flag — remove hooks |
| Overloaded struct | One class handling all variations via many optional fields | Flag — consider splitting |

### 5.3 Unnecessary Fallbacks

| Pattern | What to look for | Action |
|---------|-----------------|--------|
| Fallback that never triggers | Default branch with conditions never met | Flag — remove dead code |
| Dual implementations | Old + new code paths when old has no callers | Flag — remove old path |
| Silent fallbacks hiding problems | Catching errors and falling back instead of failing fast | Flag — fail fast or log at minimum |
| Legacy compatibility mode | Old code path always disabled behind a flag | Flag — remove if truly dead |

### 5.4 Premature Optimization

| Pattern | What to look for | Action |
|---------|-----------------|--------|
| Caching rarely-accessed data | Cache for data read at startup or once per request | Flag — measure first |
| Custom data structures | Complex structures when standard collections work | Flag — justify with benchmarks |
| Connection pooling overkill | Complex pooling for single-connection scenarios | Flag — simplify |
| Manual batching | Hand-rolled batch processing when framework provides it | Flag — use framework support |

**Severity:** Simplification findings are usually **minor** or **nit**, unless the abstraction actively obscures correctness, safe operation, or testability.

---

## Step 6 — Test Review (3 passes)

Read `{baseDir}/../../testing/REVIEWER_GUIDE.md` and follow its 3-pass structure:

1. **Completeness** — coverage assessment, code type vs test type, Right-BICEP, ZOM, boundary values
2. **Quality** — Four Pillars evaluation, testing style, test smells, mock usage, anti-patterns
3. **Correctness** — empty verification, tautological tests, leaking domain knowledge, exception specificity

Always ask:
- Should this change have tests?
- Are the right kinds of tests present?
- Would these tests catch a real bug?
- Would harmless refactoring break them?

Do not demand tests for trivial delegation/getters/setters.

---

## Step 7 — Architecture Review (5 passes)

Read `{baseDir}/../../architecture/REVIEWER_GUIDE.md` and follow its 5-pass structure:

1. **Structure** — package placement, domain module organization, Grails anti-pattern detection
2. **Naming** — service impl naming, entity naming, constants, packages
3. **Variable & Method Hygiene** — apply Fowler's Extract/Inline Variable test, flag long methods
4. **Coupling** — import analysis, dependency direction, cross-module violations, circular deps
5. **Abstraction** — premature extraction, shared/common abuse, interface necessity, Rule of Three

**Prioritization rule:** If blocker/major correctness, safety, migration, or test findings already exist from earlier steps, keep architecture/naming/hygiene comments minimal unless they materially contribute to the same risk. Deep hygiene feedback is most valuable when it improves correctness, clarity, testability, or maintenance of code introduced in this PR.

### Pass 3 — Variable & Method Hygiene

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

---

## Step 8 — Documentation Check (lightweight)

Quick scan for documentation gaps. Skip if the PR is a pure internal refactor with no user-visible changes.

- [ ] **New API endpoints or config properties** — are they documented anywhere? (README, Swagger annotations, config examples)
- [ ] **Changed behavior** — does any existing documentation (Javadoc, README, wiki) now describe wrong behavior?
- [ ] **New environment variables or feature flags** — documented with defaults and purpose?
- [ ] **Migration steps** — if the change requires deployment steps beyond "deploy," are they noted?
- [ ] **PR description** — is it sufficient for future readers to understand the change?

**Severity:** Documentation findings are **minor** unless missing docs would cause deployment failure or operator confusion.

---

## Step 9 — Verify Findings (false-positive filter)

Before composing the review, **verify every finding** against actual code context. For each finding from Steps 3-8:

1. **Re-read the actual code** at the exact file and line — not just the diff hunk, but 20-30 lines of surrounding context
2. **Check for existing mitigations** — is the issue already handled elsewhere? (parent class, AOP, framework convention, config)
3. **Check if it's pre-existing** — did this pattern exist before the PR? If yes, still note it but mark as `pre-existing`
4. **Check re-review classifications** — was this finding already explained by the author with valid reasoning? If so, treat as false positive
5. **Classify each finding:**

| Classification | Meaning | Action |
|---|---|---|
| **CONFIRMED** | Issue is real, exists in context, no mitigation | Include in review |
| **FALSE POSITIVE** | Doesn't exist, already mitigated, or misread the code | Discard silently |
| **PRE-EXISTING** | Real issue but not introduced by this PR | Include as `pre-existing` severity |
| **UNCLEAR** | Not enough context to conclude | Include as `question` severity |

**Why this step matters:** AI-generated reviews are prone to flagging things that look like issues in a diff snippet but are handled by framework conventions, parent class behavior, or adjacent code not visible in the hunk. This step prevents noise that erodes reviewer credibility.

---

## Step 10 — Compose Review

### 10.1 Review Summary

Start every review with:

```text
Verdict: <approve | approve with nits | request changes | insufficient context to approve safely>

Summary:
- Intent: <1-2 sentences>
- Main risks reviewed: <comma-separated>
- Confidence: <high | medium | low>
- Coverage: <what was reviewed deeply vs lightly, if applicable>
- Missing context: <none or brief note>
```

### 10.2 Findings

Organize **confirmed** findings by severity (blockers first, then major, then minor/nit).

Use the structured finding template for every comment. Each finding MUST include all fields:

```text
<emoji> **<Severity> — <Category>.**
📍 Location: `<file>#L<line>` (or line range)
🔍 Issue: <clear description of what's wrong>
💥 Impact: <how this affects the code — bugs, maintenance, performance, correctness>
🧾 Evidence: <brief concrete reasoning tied to code context>
💡 Suggested change: <specific actionable fix or clarifying question>
```

For `positive feedback`, use:

```text
✅ **Positive feedback.**
📍 Location: `<file>#L<line>` (optional)
🟢 Note: <what was done well and why it is good>
```

**Emoji prefixes by category:**
- 🧩 Correctness
- 🔐 Security / privacy
- 💾 Data / migration safety
- 🔄 Concurrency / idempotency
- 🌐 API / contract
- 🔌 External integration
- ⚙️ Operational safety
- 🧪 Test issue
- 📦 Package placement
- 📛 Naming violation
- 🔀 Dependency direction
- 🔗 Cross-module coupling
- ⏳ Premature abstraction
- 🔧 Simplification opportunity
- 📄 Documentation gap
- ⚠️ Weak assertion / silent failure
- ✅ Positive feedback

**Example:**
```text
🧩 **blocker — Correctness.**
📍 Location: `TransferService.java#L145`
🔍 Issue: New `processRefund()` method is registered as `@Transactional(readOnly = true)` but performs a write operation (updates refund status).
💥 Impact: Write will silently fail or throw on some JPA providers; data corruption risk in production.
🧾 Evidence: Line 148 calls `refundRepository.save(refund)` inside the read-only transaction.
💡 Suggested change: Change to `@Transactional` (remove `readOnly = true`), or split the read and write into separate transactional methods.
```

### 10.3 Strict Mode

If `--strict` is enabled:
- Report only findings that materially affect merge confidence (usually `blocker` and `major`; include a critical `question` if it blocks confident review)
- Still include final verdict and summary
- Optionally include at most 1-2 strong positive notes

---

## Comment Quality Rules

When writing review comments:
- Be specific and evidence-based
- Explain impact over asserting taste
- When uncertain, ask a focused question instead of making a claim
- If re-raising, acknowledge the author's earlier explanation

Good:
> "This migration makes the new column non-null immediately, but I don't see a backfill step. That can fail during rolling deploy if old rows still exist."

Bad:
> "This looks wrong."
> "I don't like this abstraction."
> "Please rewrite this."

---

## Do Not Do These

- Do not start with naming if correctness or data safety risk exists.
- Do not report speculative findings without verifying surrounding context.
- Do not block merge for unrelated legacy cleanup.
- Do not restate linter-level style comments unless they materially affect readability.
- Do not re-raise already explained findings without new reasoning.
- Do not demand large refactors inside a small PR unless the current design is genuinely unsafe.
- Do not confuse local conventions with universal correctness rules.
- Do not produce dozens of low-value comments — prefer fewer strong findings.

### Do Not Comment on These Unless Materially Impactful

- Formatting / linter-only issues
- Personal naming taste (as opposed to enforced repo conventions)
- Speculative future abstractions ("what if someday...")
- Unrelated legacy cleanup
- Broad refactor wishes outside the PR scope
- Comments already handled by automated tooling
- Style issues that do not affect readability or policy compliance

---

## Escalation Rules

Recommend specialist or additional review when confidence is low in any of these areas:

- Auth/authz or security-sensitive changes
- Complex data migrations (especially irreversible or large-scale)
- Performance-sensitive SQL / query paths
- Accounting invariants or money movement logic
- Distributed concurrency / retries / deduplication behavior
- Public API changes with multiple consumers

When escalating, state what specific aspect needs specialist attention and why.

---

## Quick Review Checklist

```text
## Backend PR Review

### Intent and Context
- [ ] I understand what the PR is trying to change
- [ ] I identified the highest-risk parts
- [ ] I checked linked ticket/business context if available

### Correctness
- [ ] The implementation appears complete for the stated goal
- [ ] Wiring/integration paths are actually reachable
- [ ] No obvious silent gaps or skipped branches
- [ ] State/transaction semantics make sense

### Backend Safety
- [ ] API/contract changes are safe
- [ ] DB/migration changes are deploy-safe
- [ ] Retry/idempotency/concurrency risks were checked
- [ ] External failure handling was checked
- [ ] Security/privacy implications were checked

### Tests
- [ ] Right tests exist for changed behavior
- [ ] Tests would catch a real regression
- [ ] Tests are not brittle or meaningless
- [ ] Boundary/error cases were considered

### Architecture and Maintainability
- [ ] Module/package placement is reasonable
- [ ] Dependency direction is sound
- [ ] No serious cross-module internal access
- [ ] Abstractions are not premature or harmful

### Documentation
- [ ] Config/API/behavior changes are documented where needed
- [ ] Deployment or migration notes exist if needed

### Review Quality
- [ ] Every reported finding was verified in context
- [ ] Pre-existing issues are marked as such
- [ ] Low-signal nits were filtered out
- [ ] Final verdict is explicit
```

---

## Synder-Specific Conventions

The following conventions apply to Synder repositories. They are enforced where the repo's `DEV_RULES.md` or `REVIEWER_GUIDE.md` explicitly establishes them.

**How to apply these rules:**
- Treat as **blocking** when: the repo enforces the convention, the violation is introduced by this PR, and fixing it is proportionate to the PR scope.
- Treat as **major/minor** when: the convention is established but the violation is pre-existing or the fix would be disproportionate to the PR scope.
- Do not treat convention violations as correctness defects unless they actually cause incorrect behavior.

### Naming

- Prefer `JpaXxx` / `JdbcXxx` / `HttpXxx` / `DefaultXxx` over `XxxImpl`. The prefix communicates implementation semantics; `Impl` does not. In repos that enforce this, introducing a new `XxxImpl` is normally a convention violation.
- **Entity naming**: `XxxMapping` in accounting repos, `XxxEntity` in others.
- **Shared utility naming**: Methods in `*Util` / shared classes that encode domain rules should have precise names. Flag generic names (`build*`, `create*`, `process*`) when the method does something domain-specific. Propose rename with Javadoc if the method will be used across 3+ services.

### Architecture

- **Domain model**: Plain objects, no JPA annotations. Separate from JPA entities.
- **Package structure**: Domain-first (`{domain}/model/`, `{domain}/service/`, `{domain}/jpa/`), not layer-first.
- **Cross-module**: Only through public API (service interface + domain model). Never import `.jpa.` from another module.

### Tests

- **Style preference**: Output-based > state-based > communication-based. Never assert on stubs. Hardcode expected values.
- **Mocks**: Stub for data, Mock for boundary interactions. >8 mocks = design smell.
- **Spock**: `@Unroll` on parameterized tests, `given/when/then` structure, `*Spec` naming.
- **JUnit 5**: Backtick method names, `@Nested` groups, `*Test` naming.

---

## Optional Supporting Files

Use these when available:

- `{baseDir}/../../testing/REVIEWER_GUIDE.md` — detailed test review criteria
- `{baseDir}/../../architecture/REVIEWER_GUIDE.md` — detailed architecture/convention review criteria
- `{baseDir}/../../architecture/repo-specific/*.md` — repository overlays

If such files do not exist, continue with the core workflow in this file.
