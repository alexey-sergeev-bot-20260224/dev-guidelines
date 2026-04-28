---
name: pr-design-pressure
description: Pressure-test PRs and implementation decisions by identifying hidden assumptions, unresolved design choices, high-impact questions, edge cases, and operational risks. Use together with code review when a PR changes architecture, API contracts, data models, migrations, async processing, idempotency, security-sensitive behavior, accounting/reconciliation logic, or cross-service integration behavior.
---

Use this skill to pressure-test a PR, plan, or implementation decision.

Focus on:
- hidden assumptions
- unresolved design choices
- missing constraints
- edge cases
- failure modes
- coupling between decisions
- operational risks
- product/accounting behavior ambiguity

When used during PR review:
- Do not replace the primary code review skill.
- Use this skill only as a design-pressure overlay.
- Do not use it for trivial formatting, naming, typo, or obvious small refactoring issues.
- Prefer answering questions from the PR diff, linked Jira task, repository code, or dev-guidelines before asking the user.
- Do not block a cron review by asking the user questions unless the review cannot proceed safely without the answer.
- If unresolved questions remain, collect them under a section named "Open design questions".
- Each open question must include why it matters, what code or behavior triggered it, the recommended default answer, and whether it is blocking or non-blocking.

When used interactively:
- Ask one focused question at a time.
- For each question, provide your recommended answer.
- Continue only while the answers materially change the design or review outcome.
