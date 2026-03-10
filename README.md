# Dev Guidelines

Development guidelines, testing instructions, and coding standards for Synder backend.

For human developers and AI agents alike.

## Structure

```
dev-guidelines/
├── README.md                          # This file
│
├── testing/                           # Everything about testing
│   ├── DEVELOPER_GUIDE.md             # How to write tests (for developers)
│   ├── REVIEWER_GUIDE.md              # How to review tests (for reviewers)
│   ├── unit/                          # Unit testing specifics
│   │   └── (future: framework-specific guides, patterns, examples)
│   └── integration/                   # Integration testing specifics
│       └── (future: DB testing, API testing, Docker-based tests)
│
├── coding-standards/                  # Code style and conventions
│   └── (future: naming, architecture, language-specific rules)
│
├── architecture/                      # Architecture decisions and patterns
│   └── (future: ADRs, service design, data model guidelines)
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
