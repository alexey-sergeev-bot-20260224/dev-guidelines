# Dev Guidelines

Development guidelines, testing instructions, and coding standards for Synder backend.

For human developers and AI agents alike.

## Structure

```
dev-guidelines/
├── README.md                          # This file
├── DEV_RULES.md                       # Index — links to all topic-specific guides
│
├── architecture/                      # Architecture decisions and patterns
│   ├── DEVELOPER_GUIDE.md             # How to structure code and design modules
│   └── REVIEWER_GUIDE.md              # How to evaluate structure and naming in PRs
│
├── testing/                           # Everything about testing
│   ├── DEVELOPER_GUIDE.md             # How to write tests (for developers)
│   ├── REVIEWER_GUIDE.md              # How to review tests (for reviewers)
│   └── DECISIONS.md                   # Testing decisions and rationale
│
├── coding-standards/                  # Code style and conventions
│   └── (future: language-specific rules)
│
├── onboarding/                        # Getting started guides
│   └── (future: local setup, repo overview, tooling)
│
├── templates/                         # Reusable templates
│   └── (future: PR templates, test class templates, ADR templates)
│
└── checklists/                        # Quick-reference checklists
    └── (future: PR checklist, deployment checklist, review checklist)
```

## Who Is This For

- **Human developers** — reference guides for writing code and tests
- **Human reviewers** — criteria and checklists for code reviews
- **AI agents** — structured instructions for automated development and review tasks

## Stack

- Java / Groovy / Kotlin
- Spock (test framework)
- Grails / Spring Boot
- Gradle / Maven
- PostgreSQL / MongoDB
