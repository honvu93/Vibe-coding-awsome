# Project Structure for AI-Assisted Development

Well-organized code is always important. With AI assistants, it becomes **load-bearing infrastructure**. Claude Code reads your directory tree and file contents to build a working model of your project before it writes a single line. The structure you create on day one shapes every suggestion the AI makes for the lifetime of the project.

This guide covers how to organize workspaces and projects so that Claude Code operates with maximum accuracy — regardless of whether you're building in Python, TypeScript, Go, Rust, or any other language.

---

## Why Structure Matters More with AI

When a human developer joins a project, they ask questions. Claude Code cannot. It infers everything from what it can read:

- **Your directory tree** — tells it where things belong
- **Your CLAUDE.md** — tells it your conventions and commands
- **Your file names** — tells it what each file does
- **Your existing code** — tells it the patterns to follow

A well-structured project means Claude navigates confidently, places new files in the right directories, follows your naming conventions, and writes code consistent with your existing patterns.

A poorly structured project means Claude makes wrong assumptions — puts new files in the wrong place, generates code that conflicts with your conventions, and fails to find relevant context when making changes.

**AI amplifies both good and bad organization.** A messy project becomes messier faster. A clean project stays clean longer.

### What Claude Reads When It First Opens Your Project

```
1. CLAUDE.md (at project root, then at workspace root)
2. Directory listing of the root
3. Key config files: package.json, pyproject.toml, go.mod, Cargo.toml
4. Any files you reference in the conversation
```

Everything else is on-demand. This means your root-level signals carry enormous weight.

---

## Workspace vs Project

A **workspace** is a top-level directory that contains multiple related projects. A **project** is an individual codebase within that workspace.

This distinction matters because:

- Your AI workflow tooling (planning artifacts, sprint tracking, agent configs) lives at the workspace level
- Your application code lives at the project level
- The two should not mix

### Workspace-Level CLAUDE.md

The workspace CLAUDE.md answers:
- Which project is currently active?
- What are the shared conventions across all projects?
- Where do planning artifacts live?
- How do projects relate to each other?

### Project-Level CLAUDE.md

The project CLAUDE.md answers:
- How do I run this project?
- What is the architecture?
- What are the testing conventions?
- What are the key files and where are they?
- What patterns should new code follow?

The project CLAUDE.md is the authoritative reference for everything Claude needs to work on that codebase. Treat it as a contract. See [CLAUDE.md Architecture](claude-md-architecture.md) for a complete guide.

---

## Recommended Workspace Layout

```
my-workspace/
├── CLAUDE.md                          # Workspace context: active project, shared conventions
│
├── my-app/                            # Main application
│   ├── CLAUDE.md                      # Deep technical context for this app
│   ├── src/                           # Application source code
│   ├── tests/                         # All test files
│   ├── docs/                          # Technical documentation close to code
│   ├── scripts/                       # Build/deploy/utility scripts
│   └── config/                        # Environment config templates
│
├── planning/                          # All planning artifacts — never in src/
│   ├── planning-artifacts/            # PRDs, architecture docs, epic breakdowns
│   └── implementation-artifacts/     # Story files, sprint tracking
│
├── shared-scripts/                    # Cross-project automation
├── docs/                              # Shared, cross-project documentation
│
└── .claude/                           # AI workflow tooling
    ├── WORKFLOW.md                    # Step-by-step development process
    ├── agents/                        # Agent profile definitions
    └── superpowers-skills/            # Process enforcement rules
```

This layout keeps planning artifacts, application code, and AI tooling in separate top-level directories. Claude can be directed to each layer without contamination from the others.

---

## Separating Concerns

### Planning vs Implementation

Keep planning documents out of your source code directories. This is not just good hygiene — it prevents Claude from treating design documents as code to be edited, and prevents code searches from surfacing irrelevant planning prose.

**Planning artifacts** belong in `planning/planning-artifacts/`:
- Product Requirements Documents (PRDs)
- Architecture decision records
- Epic breakdowns
- Sprint proposals
- UX design specifications

**Implementation artifacts** belong in `planning/implementation-artifacts/`:
- Story files with acceptance criteria and task checklists
- Sprint tracking boards
- Retrospective notes

**Source code** belongs in `my-app/src/`.

When you tell Claude "implement story #42", it can read the story file from `planning/implementation-artifacts/` and the codebase from `my-app/src/` independently, without one polluting the other.

### Documentation Layers

Good projects have documentation at multiple levels of abstraction. Each layer serves a different audience:

| Layer | Location | Audience | Purpose |
|---|---|---|---|
| CLAUDE.md | Project root | Claude Code | AI-facing commands, conventions, architecture |
| README.md | Project root | Human developers | Overview, setup, what this does |
| docs/ | `project/docs/` | Human devs + Claude | Feature-level technical notes, gap analyses |
| Planning artifacts | `planning/` | Product/Engineering | Requirements, architecture decisions |
| Inline comments | In source files | Future maintainers | Only where logic is non-obvious |

The key discipline: **do not collapse these layers**. Putting your architecture doc in `src/` confuses Claude. Putting inline comments in your README creates noise. Each layer has a home.

---

## Domain-Driven Directory Structure for Backend Services

For backend applications, a layered domain pattern scales well with AI-assisted development. The pattern is language-agnostic — here it is shown in three ecosystems.

### Node.js (ESM)

```
src/
├── domain/
│   ├── users/
│   │   ├── service.js          # Business logic, transaction boundaries
│   │   ├── repository.js       # Data access layer
│   │   ├── access.js           # Permission checks, input validation
│   │   └── contract.js         # DTOs, error builders, response normalization
│   ├── orders/
│   │   ├── service.js
│   │   ├── repository.js
│   │   ├── access.js
│   │   └── contract.js
│   └── billing/
│       ├── service.js
│       ├── repository.js
│       ├── access.js
│       └── contract.js
│
├── infrastructure/             # Cross-cutting concerns, no business logic
│   ├── db/                     # Database client, connection pooling
│   ├── security/               # Encryption, credential utilities
│   ├── queue/                  # Job queue setup and registry
│   ├── cache/                  # Caching layer
│   └── resilience/             # Circuit breakers, retry logic
│
└── adapters/                   # External service integrations
    ├── stripe/
    ├── sendgrid/
    └── s3/
```

### Python

```
src/
├── domain/
│   ├── users/
│   │   ├── service.py          # Business logic
│   │   ├── repository.py       # SQLAlchemy queries
│   │   ├── access.py           # Permission checks
│   │   └── schemas.py          # Pydantic models, error definitions
│   ├── orders/
│   │   └── ...
│   └── billing/
│       └── ...
│
├── infrastructure/
│   ├── database.py             # SQLAlchemy engine, session factory
│   ├── security/               # JWT, hashing utilities
│   ├── queue/                  # Celery app, task registry
│   └── cache/                  # Redis client
│
└── adapters/
    ├── stripe.py
    └── s3.py
```

### Go

```
internal/
├── domain/
│   ├── users/
│   │   ├── service.go          # Business logic interfaces and implementations
│   │   ├── repository.go       # Database interface and implementations
│   │   ├── handler.go          # HTTP handler
│   │   └── model.go            # Domain types, error definitions
│   ├── orders/
│   │   └── ...
│   └── billing/
│       └── ...
│
├── infrastructure/
│   ├── postgres/               # DB connection, migrations
│   ├── redis/                  # Cache client
│   └── queue/                  # Job queue
│
└── adapters/
    ├── stripe/
    └── s3/
```

### Why This Pattern Works with AI

**Predictable navigation.** When you say "add a feature to the users domain," Claude knows exactly which four files to look at. It does not need to search.

**Clear responsibility.** Claude never confuses which layer to write logic in. Business rules go in `service`, data access goes in `repository`, permission checks go in `access`, and shape definitions go in `contract`/`schemas`/`model`.

**Template effect.** When Claude creates a new domain (say, `invoices`), it copies the structure from an existing domain. Consistency is automatic.

**Infrastructure isolation.** Database clients, encryption utilities, and queue connections live in `infrastructure/`. Claude does not accidentally put database code in a business logic file.

---

## The .claude/ Directory

This directory is the home for your AI workflow tooling. It is not application code — it is the scaffolding that governs how Claude Code operates on your project.

```
.claude/
├── WORKFLOW.md                        # End-to-end development process, step by step
│
├── agents/                            # Agent profile definitions
│   ├── backend-architect.md           # Domain services, APIs, databases
│   ├── frontend-developer.md          # UI components, routing, styles
│   ├── security-engineer.md           # Auth, encryption, access control
│   ├── database-optimizer.md          # Schema design, query tuning
│   ├── code-reviewer.md               # Pre-merge review gate
│   ├── devops-automator.md            # Scripts, workers, CI/CD
│   └── ROUTING.md                     # Which agent handles which task
│
└── superpowers-skills/                # Process enforcement rules
    ├── test-driven-development.md     # The Iron Law: failing test before production code
    ├── systematic-debugging.md        # 4-phase investigation protocol
    ├── writing-plans.md               # How to decompose stories into tasks
    ├── requesting-code-review.md      # What review gates to enforce
    └── verification-before-completion.md  # Evidence requirements before marking done
```

`WORKFLOW.md` is the most important file here. It defines the end-to-end process: how stories become tasks, how tasks become code, how code becomes a merge. Without it, each Claude session reinvents the process. With it, every session follows the same disciplined path.

See [Three-Layer Agent System](../03-agents/three-layer-system.md) for how agents, skills, and workflow fit together.

---

## Anti-Patterns

### Flat Root Directory

```
# Bad
my-app/
├── user.js
├── order.js
├── billing.js
├── database.js
├── auth.js
├── stripe.js
├── email.js
├── ... (40 more files)
```

Claude cannot navigate this. It cannot tell which files are related, which are infrastructure, which are domain logic. Every change risks touching the wrong file.

### Deeply Nested Directories

```
# Bad
src/
└── core/
    └── modules/
        └── business/
            └── domain/
                └── entities/
                    └── user/
                        └── model.js
```

Five levels of nesting to find a single file. Claude loses context navigating through directory structures that have no meaning. Three levels is usually the maximum before directories become noise.

### Mixed Concerns

```
# Bad — planning docs in source directory
src/
├── users/
│   ├── service.js
│   ├── PRD-users.md           # Planning doc doesn't belong here
│   └── architecture-notes.md # Neither does this
```

Planning documents in `src/` get picked up by code searches and confuse Claude about what is a source file. Keep them in `planning/`.

```
# Bad — test utilities in production code
src/
├── users/
│   ├── service.js
│   └── test-helpers.js        # Test utility in production directory
```

Test utilities belong in `tests/`. Production code directories should contain only production code.

### No CLAUDE.md

Without CLAUDE.md, Claude guesses:
- How to run your tests (wrong framework, wrong command)
- Where new files belong (often wrong)
- What conventions to follow (inconsistent with your codebase)
- What the architecture is (inferred from file names, often inaccurate)

A project without CLAUDE.md is a project where the AI is flying blind. Even a minimal CLAUDE.md with just the run commands and the test command is significantly better than nothing.

### Stale Directories

```
# Bad
my-app/
├── src/           # Current code
├── src-v2/        # Old migration attempt, abandoned
├── src-backup/    # "Just in case" backup from 6 months ago
└── old/           # Unknown contents
```

Claude reads all of these. Old code from abandoned migrations or backup copies will contaminate Claude's understanding of your architecture. Use git for history — delete directories that are not current.

### Inconsistent Naming

```
# Bad — three naming conventions in one project
src/
├── UserService.js      # PascalCase
├── order-service.js    # kebab-case
├── billing_service.js  # snake_case
```

Claude will follow whatever convention it sees most recently. Inconsistent naming leads to inconsistent generated code. Pick one convention per language and enforce it in CLAUDE.md.

---

## Phased Workflow Structure

If you use a phased development process (requirements → design → architecture → implementation), map your directory structure to match the phases. This gives Claude the ability to reference the correct artifact at each stage.

### Phase 1: Analysis

Artifacts: market research, domain analysis, technical feasibility assessments.

```
planning/planning-artifacts/analysis/
├── market-analysis.md
├── domain-glossary.md
└── technical-feasibility.md
```

### Phase 2: Requirements and Design

Artifacts: PRDs, UX specifications, validated requirements.

```
planning/planning-artifacts/
├── prd.md                          # Product Requirements Document
├── ux-design.md                    # UX flows and wireframe descriptions
└── architecture.md                 # High-level architecture decisions
```

### Phase 3: Architecture

Artifacts: epic breakdowns, module specifications, API contracts.

```
planning/planning-artifacts/
├── epics/
│   ├── epic-01-user-management.md
│   ├── epic-02-billing.md
│   └── epic-03-notifications.md
└── api-contracts/
    ├── users-api.md
    └── billing-api.md
```

### Phase 4: Implementation

Artifacts: story files with acceptance criteria and task checklists, sprint tracking.

```
planning/implementation-artifacts/
├── stories/
│   ├── story-001-user-registration.md
│   ├── story-002-password-reset.md
│   └── story-003-billing-setup.md
└── sprints/
    ├── sprint-01-tracking.md
    └── sprint-02-tracking.md
```

When Claude implements story-003, it can read:
- The story file for acceptance criteria and task list
- `planning-artifacts/api-contracts/billing-api.md` for interface specs
- The existing `src/domain/billing/` code for implementation context

No file contamination, no navigation confusion.

---

## Migration: Retrofitting an Existing Project

You do not need to reorganize everything at once. Add structure incrementally:

**Day 1 (30 minutes): Create CLAUDE.md**

Write down:
- How to run the project
- How to run the tests
- The top-level architecture in plain language
- One or two key conventions (naming, error handling, etc.)

This single file gives Claude enough context to be useful immediately.

**Week 1: Organize tests**

If your test files are scattered throughout the codebase, move them into a top-level `tests/` directory. This is almost always a safe mechanical operation and it immediately clarifies to Claude what is production code and what is test code.

**Week 1: Add .claude/ directory**

Create `.claude/WORKFLOW.md` documenting your development process. Even a simple list of "1. Write failing test, 2. Write implementation, 3. Verify, 4. Commit" is better than nothing.

**Week 2: Separate planning from code**

If you have design documents or planning notes mixed into your source directories, move them to a top-level `planning/` or `docs/` directory.

**Ongoing: Add memory system**

As you discover things Claude keeps getting wrong (or right), document them. See [Memory System](memory-system.md) for how to set up persistent learning across sessions.

**Never: Reorganize everything at once**

A big-bang restructure on a live codebase is risky and disruptive. Incremental migration keeps the project functional throughout and lets you evaluate each change before making the next one.

---

## When This Doesn't Apply

**Single-file scripts and tiny utilities.** If your project is one file or five files, a CLAUDE.md and a flat directory is fine. Add structure only when navigation becomes a problem.

**Early-stage prototypes.** Prototypes change too fast for structure to pay off. Keep them flat and simple. When a prototype graduates to a real product, add structure then.

**Existing large projects.** If you are working with a large existing codebase that has an established structure, do not reorganize it to match this guide. The guide describes what to aim for when you have a choice. In an existing project, your highest-leverage move is adding a CLAUDE.md that describes the current structure accurately.

---

## Checklist: Is Your Structure AI-Ready?

Before your first serious Claude Code session, verify:

- [ ] CLAUDE.md exists at project root with commands, architecture, and key conventions
- [ ] Test files are in a separate `tests/` directory (not mixed with source)
- [ ] Infrastructure code (DB clients, queue setup, security utilities) is in its own directory
- [ ] Planning documents are outside `src/`
- [ ] No stale backup directories or abandoned migration attempts
- [ ] Naming conventions are consistent within each language/module
- [ ] `.claude/WORKFLOW.md` exists with your development process
- [ ] Directory depth is three levels or fewer for common operations

This structure is not about being tidy for its own sake. It is about giving your AI assistant an accurate, navigable model of your project — so that every suggestion it makes is grounded in reality instead of guesswork.
