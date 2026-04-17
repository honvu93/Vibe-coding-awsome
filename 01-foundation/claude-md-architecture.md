# CLAUDE.md Architecture

`CLAUDE.md` is the single most important file you will write in any Claude Code project. Get it right and Claude operates like a senior engineer who has been on the project for months. Get it wrong and you spend every session re-explaining context that Claude promptly forgets.

---

## Why CLAUDE.md Matters

Think of `CLAUDE.md` as the **API contract between you and Claude**. Every session — every new context window — Claude reads this file first. It is the only persistent memory that survives across conversations.

Without it, Claude starts from zero each time:
- It guesses at your build commands and gets them wrong.
- It invents error-handling patterns that contradict your codebase.
- It creates files in the wrong directories.
- It writes code that works in isolation but violates every convention your team has established.

With a well-written `CLAUDE.md`, the compounding effect is significant:

```
Good CLAUDE.md
  → Claude understands context immediately
  → Fewer wrong assumptions in generated code
  → Less correction and back-and-forth
  → Faster iteration
  → More time to improve CLAUDE.md further
  → Repeat
```

The investment in writing `CLAUDE.md` well pays back on the first session and every session after.

---

## Anatomy of a Great CLAUDE.md

A great `CLAUDE.md` answers exactly the questions Claude will ask silently before writing any code: *How do I build this? Where does code go? How are errors handled? What am I not allowed to do?*

Walk through each section in order.

---

### 1. Commands

**Rule: List every command a developer needs. Include the single-file variant, not just the "run everything" variant.**

Claude frequently needs to run a specific test, build a specific target, or start a specific service. If you only provide `npm test`, Claude runs all tests when it only needed to run one. This wastes time and obscures signal.

#### What to include

- Setup/install
- Full stack start (all services)
- Hot-reload / dev mode
- Individual service start
- Backend tests — all, and single-file
- Frontend tests — all, and single-file
- Lint
- Build
- Deploy / reload
- Any bootstrap or seed commands
- Any worker or daemon commands

#### Examples across languages

**Node.js / TypeScript**
```bash
# Install
npm install && pnpm --dir web install

# Start full stack
npm run dev

# Start services individually
npm run backend:dev          # Terminal 1
pnpm --dir web dev           # Terminal 2

# Tests — all
npm test
npm run web:test

# Tests — single file
node --test tests/user-service.test.js
pnpm --dir web vitest run tests/auth-lib.test.ts

# Lint and build
npm run web:lint
npm run web:build
```

**Python**
```bash
# Install
pip install -e ".[dev]"
# or
uv sync

# Start
uvicorn app.main:app --reload --port 8000

# Tests — all
pytest

# Tests — single file
pytest tests/test_user_service.py -v

# Tests — single test
pytest tests/test_user_service.py::test_create_user -v

# Lint
ruff check .
mypy src/
```

**Rust**
```bash
# Build
cargo build
cargo build --release

# Run
cargo run

# Tests — all
cargo test

# Tests — single
cargo test user_service::test_create_user

# Lint
cargo clippy -- -D warnings
cargo fmt --check
```

**Go**
```bash
# Build
go build ./...

# Run
go run cmd/server/main.go

# Tests — all
go test ./...

# Tests — single package
go test ./internal/user/...

# Tests — single test
go test ./internal/user/... -run TestCreateUser -v

# Lint
golangci-lint run
```

**The critical rule**: never write just `npm test` or `pytest` without also showing how to run a single file. Claude runs all tests far more often than necessary when the single-file command is missing.

---

### 2. Architecture Overview

**Rule: Describe the structure of your codebase in concrete terms, not aspirational terms.**

Do not write "we follow clean architecture principles." Write "controllers are in `src/http/`, business logic is in `src/domain/{name}/service.js`, database access is in `src/domain/{name}/repository.js`."

#### Backend structure

Show the directory layout and the role of each layer. One sentence per layer is enough.

```
src/
  domain/
    users/
      service.js      — business logic, transaction boundaries
      repository.js   — database queries
      access.js       — permission checks and input validation
      contract.js     — DTO normalization and error builders
    orders/
      ...
  infrastructure/
    db/               — database client singleton
    queue/            — job queue setup
    security/         — encryption utilities
  adapters/
    payment/          — third-party payment gateway adapter
    email/            — email provider adapter
```

#### Frontend structure

```
web/src/
  app/
    (protected)/      — authenticated routes
    (public)/         — unauthenticated pages
    api/              — API route handlers
  features/
    users/
      components/     — React components scoped to this feature
      lib/            — hooks, utilities, query functions
  lib/
    auth/             — auth configuration
    db/               — database client for API routes
```

#### Path aliases

Be explicit. Claude will use the wrong import path if you do not state this.

```
# TypeScript / Next.js
@/*          → web/src/*
@core/*      → src/*        (backend domain code imported by frontend API routes)

# Python
from app.domain.users import UserService    # not from src.domain.users

# Go
import "github.com/yourorg/yourapp/internal/user"
```

#### Key data flows

Two sentences each. Claude needs to understand direction, not implementation.

```
# Example
Job lifecycle: API route creates job record in PostgreSQL →
  worker polls or consumes from queue → worker updates record status →
  frontend queries API route for current status.

Auth: Browser sends session cookie → middleware validates against
  database → request proceeds or returns 401.
```

---

### 3. Key Patterns and Conventions

**Rule: Be explicit about patterns that are non-obvious or that Claude will get wrong by default.**

This section prevents the most common class of subtle bugs: code that compiles and runs but violates your project's conventions in ways that only cause problems later.

#### Error handling

State your error pattern explicitly. Claude defaults to throwing custom error classes; if you use something different, say so.

```
# Pattern: plain Error objects with a code property
throw Object.assign(new Error('User not found'), { code: 'USER_NOT_FOUND' });

# NOT this — we do not use custom error classes
class UserNotFoundError extends Error { ... }  // Do not do this

# Domain contract builders (preferred for domain errors)
import { notFound } from './contract.js';
throw notFound('User not found');
// contract.js builds: Object.assign(new Error(msg), { code: 'USER_NOT_FOUND' })
```

```python
# Pattern: raise ValueError with a structured message dict
raise ValueError({"code": "USER_NOT_FOUND", "message": "User not found"})

# NOT custom exception classes unless they are in exceptions.py
# NOT bare Exception("something went wrong")
```

#### Naming conventions

```
# Files
user-service.js          # kebab-case, always
UserService.ts           # PascalCase only for React components and classes
test_user_service.py     # snake_case for Python
user_service_test.go     # Go test files follow package_test.go

# Variables
userId                   # camelCase in JS/TS
user_id                  # snake_case in Python/Go/Rust

# Constants
MAX_RETRY_COUNT          # SCREAMING_SNAKE in all languages
```

#### File organization rules

State where new files go. This prevents Claude from creating files in wrong locations.

```
# New domain = new directory under src/domain/{name}/
# New React feature = new directory under web/src/features/{name}/
# New test = tests/{name}.test.js (mirrors src/ structure, flat)
# Utilities shared across domains = src/lib/ (not src/domain/)
# Scripts (one-time, ops) = scripts/ (not src/)
```

#### Anti-patterns

This is one of the most valuable things you can put in `CLAUDE.md`. Be direct.

```
# Do NOT do these things in this project:

- Do not use async/await inside Prisma transactions — use the callback form
- Do not import from src/domain/ directly in React components — use API routes
- Do not create new database clients — import the singleton from infrastructure/db/
- Do not write raw SQL — use Prisma query methods
- Do not add new npm dependencies without checking if a built-in solves it first
- Do not use process.env directly — use the env loader from infrastructure/env/
- Do not throw HTTP status codes from domain services — domain is framework-agnostic
```

The "do not" list is as valuable as the "do" list. Claude learns from positive examples, but hard boundaries prevent entire categories of mistakes.

---

### 4. Auth and Environment

**Rule: Document every environment variable your app reads. If Claude does not know what an env var controls, it may duplicate it or ignore it.**

#### Default local credentials

Include dev-only defaults so Claude can reason about the running system.

```
# Development defaults (never use in production)
Admin email: admin@example.local
Admin password: dev-password-only

# Override via environment
ADMIN_EMAIL=your@email.com
ADMIN_PASSWORD=your-password
```

#### Environment variables

Document each variable with its purpose and whether it is required.

```bash
# Required
DATABASE_URL           — PostgreSQL connection string
AUTH_SECRET            — Session encryption key (min 32 chars, random)

# Required in production, optional in development
REDIS_URL              — Redis connection (default: redis://localhost:6379)
QUEUE_BACKEND          — "bullmq" or "polling" (default: "polling" in dev)

# Feature flags
ENABLE_EMAIL_VERIFY    — Require email verification on signup (default: false)
ENABLE_RATE_LIMITING   — Enable API rate limiting (default: true in production)

# External services
STRIPE_SECRET_KEY      — Payment processing
SENDGRID_API_KEY       — Transactional email
S3_BUCKET_NAME         — File upload storage
```

State the **effect** of each variable, not just its name. "Session encryption key" tells Claude more than just `AUTH_SECRET`.

---

### 5. Testing Notes

**Rule: State your test runner explicitly. Never assume.**

Claude knows Jest, pytest, Go testing, Rust test, and many others. But it defaults to the most common choice (Jest for Node.js), and if you use something else, it will write incompatible test syntax until corrected.

```
## Testing Notes

Backend tests use Node.js built-in `node:test` — NOT Jest, NOT Vitest.
Correct imports:
  import { describe, it, mock, before, after } from 'node:test';
  import assert from 'node:assert/strict';

Frontend tests use Vitest + JSDOM + @testing-library/react.
Correct imports:
  import { describe, it, expect, vi } from 'vitest';
  import { render, screen } from '@testing-library/react';

Test file locations:
  Backend: tests/*.test.js  (flat, mirrors src/ names)
  Frontend: web/tests/*.test.ts  or  web/tests/*.test.tsx

Run a single backend test:
  node --test tests/user-service.test.js

Run a single frontend test:
  pnpm --dir web vitest run tests/auth-lib.test.ts
```

```python
## Testing Notes

Tests use pytest with pytest-asyncio for async tests.
NOT unittest, NOT nose.

Async tests require the @pytest.mark.asyncio decorator.
Fixtures go in conftest.py at the relevant directory level.

Run a single file:  pytest tests/test_user_service.py -v
Run a single test:  pytest tests/test_user_service.py::test_create_user -v
Run with coverage:  pytest --cov=src tests/
```

---

### 6. Commit and Documentation Discipline

**Rule: Codify your commit discipline so Claude follows it without being told each time.**

If Claude completes a task and does not commit, or commits everything in one batch, or skips the changelog, it creates noise in your git history that you will have to clean up.

```
## Commit Discipline

Commit after completing each task. Never batch multiple tasks into one commit.

Before every commit:
1. Update CHANGELOG.md with a concise entry (what changed and why)
2. Stage only the relevant files — never `git add .` blindly
3. Write a commit message in imperative mood: "Add user creation endpoint"
   not "Added user creation endpoint" or "Adding user creation endpoint"

Commit message format:
  type(scope): short description

  Types: feat, fix, refactor, test, docs, chore, perf
  Example: feat(users): add email verification on signup

CHANGELOG format (append to top of Unreleased section):
  - feat(users): add email verification flow [story-42]
```

```
## Documentation

- Feature docs → docs/ (close to the code)
- Architecture decisions → docs/adr/ (Architecture Decision Records)
- API reference → generated, do not hand-write
- Planning artifacts → separate planning repo or directory outside src/
- Do NOT create .md files unless explicitly asked — return findings as text
```

---

## Layered CLAUDE.md Strategy

A single `CLAUDE.md` at the project root works for most projects. For larger or multi-project workspaces, a layered strategy gives you more precision.

### The three layers

**Layer 1 — Global: `~/.claude/CLAUDE.md`**

Personal preferences that apply across all your projects. Keep this short.

```markdown
# Global Preferences

Reply to me in English.
Use tabs for indentation, not spaces.
When I ask for a plan, write the plan before writing any code.
Never add emojis to code files or commit messages.
```

**Layer 2 — Workspace root: `/path/to/workspace/CLAUDE.md`**

Multi-repo workspace guidance. Describes what lives where.

```markdown
# Workspace Structure

This workspace contains multiple related projects.

- api/        — Python FastAPI backend
- web/        — Next.js frontend  
- mobile/     — React Native app
- shared/     — Shared TypeScript types

Active development is in api/ and web/.
Run all commands from within the relevant subdirectory.
```

**Layer 3 — Project: `project/CLAUDE.md`**

The full project-specific guide. This is where the detailed content from sections 1-6 above lives.

**Layer 4 — Subdirectory: `project/src/some-module/CLAUDE.md`**

Deep context for a specific subsystem. Use sparingly — only when a module has conventions that differ meaningfully from the project default.

```markdown
# Payment Module

This module integrates with Stripe. Key constraints:

- All Stripe API calls must go through the StripeAdapter class, never call the SDK directly
- Webhook handlers must be idempotent — Stripe delivers events at least once
- Never log card numbers or CVVs, even truncated
- Test with Stripe test mode keys only — production keys are not in .env.local
```

### How they compose

Claude reads all `CLAUDE.md` files it can find in the path from the working directory up to the filesystem root, plus the global one. Closer files take precedence over farther files. A subdirectory `CLAUDE.md` can override rules from the project root.

Use this deliberately: the workspace root sets broad rules, the project sets project-specific rules, a subdirectory overrides for its narrow scope.

---

## Common Anti-Patterns

These are the mistakes that make `CLAUDE.md` less effective. Each one has a cost.

### Too long

A `CLAUDE.md` over 500 lines gets skimmed, not read. Critical rules buried on line 400 get missed. If yours is long, it means you have not decided what is truly important.

**Fix**: Ruthlessly cut anything Claude can infer from the code itself. Document decisions, not descriptions.

### Too vague

```markdown
# Bad
Follow best practices for error handling.
Write clean, readable code.
Use appropriate design patterns.
```

These instructions are noise. Claude already tries to do these things; it just disagrees with you about what they mean.

```markdown
# Good
Error handling: throw plain Error objects with a .code string property.
Never throw HTTP status codes from service functions.
Never use custom error classes.
```

Concrete beats abstract every time.

### Outdated references

A `CLAUDE.md` that references `src/utils/old-auth.js` (a file deleted 3 months ago) actively misleads Claude. It will try to import from a non-existent path, get confused, and waste your time.

**Fix**: Treat `CLAUDE.md` as code. Update it in the same PR as the change it describes. Stale documentation is worse than no documentation.

### Missing commands

If `npm run bootstrap` is not in `CLAUDE.md`, Claude will not know it exists. The first time it needs to seed the database, it will write a migration script from scratch.

**Fix**: Every command a developer needs on day one goes in the Commands section. Every command.

### No architecture section

Without architecture context, Claude makes structural decisions based on patterns it has seen in other projects. These may be fine patterns in general but wrong for yours.

```markdown
# What Claude assumes without guidance:
- Feature code goes in src/features/ (maybe yours is src/modules/)
- API routes go in src/routes/ (maybe yours uses a file-based router)
- Tests go in __tests__/ (maybe yours uses tests/)
- Database access is in the service layer (maybe yours separates repository)
```

**Fix**: Write three to five sentences describing where things live and why. That is enough to align Claude with your structure.

### Treating it as documentation for humans

`CLAUDE.md` is not a README. It is not for onboarding humans. It is a precise instruction set for an AI that will read it thousands of times. Write it with that audience in mind: dense, specific, unambiguous.

---

## Evolution Over Time

`CLAUDE.md` is a living document, not a one-time artifact.

### Start minimal

On day one of a new project, write only what you know for certain:

```markdown
# MyApp

## Commands
npm install
npm run dev
npm test

## Architecture
Node.js backend, React frontend. Backend in src/, frontend in web/.
```

That is enough. Add more as you establish patterns.

### Update after every architectural change

When you add a new layer, rename a directory, change your error pattern, or adopt a new test runner: update `CLAUDE.md` in the same commit. The diff in your git history documents when and why the convention changed.

```bash
# Good commit discipline
git add src/domain/payments/ tests/payment-service.test.js CLAUDE.md CHANGELOG.md
git commit -m "feat(payments): add payment domain with Stripe adapter"
```

### Remove stale information aggressively

A wrong instruction in `CLAUDE.md` is worse than a missing one. Missing information causes Claude to ask or guess; wrong information causes Claude to confidently do the wrong thing.

When you delete a pattern, delete its documentation. When you rename a directory, update every reference. Run a search for any file paths mentioned in `CLAUDE.md` before committing.

### Let the git diff tell the story

Your `CLAUDE.md` history is an architectural log. Future you — and future team members — can read the git log of `CLAUDE.md` to understand how the project's conventions evolved and why decisions were made.

```bash
git log --follow -p CLAUDE.md
```

This is a feature. Write commit messages for `CLAUDE.md` changes that explain the decision, not just the text change.

---

## Template

A starter template is available at `../templates/CLAUDE.md.template`.

It provides the full skeleton with placeholder text for all six sections. Copy it, fill in the real values for your project, delete any sections that do not apply, and you have a working `CLAUDE.md` in under ten minutes.

The template is intentionally minimal. Resist the urge to fill every section on day one. A minimal, accurate `CLAUDE.md` is more useful than a comprehensive, partially-wrong one.
