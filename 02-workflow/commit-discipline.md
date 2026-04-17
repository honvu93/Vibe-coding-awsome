# Commit Discipline

## The Rule

**One task = one commit. Always.**

Not "one feature" or "one PR" — one task. If your plan has 5 tasks, you make 5 commits. Each commit is atomic, testable, and reversible.

---

## Why This Matters with AI

AI coding assistants can generate massive changesets in seconds. That speed is the superpower — and the danger.

- **Massive commits are unreviable.** When 40 files change in one shot, you cannot meaningfully review the diff.
- **Massive commits are unrevertable in practice.** `git revert` a 40-file commit and you'll likely break other things.
- **Granular commits enable `git bisect`.** When a bug appears two weeks later, you can binary-search the commit history to pinpoint the exact change that caused it.
- **Each commit tells a story.** What changed, why it changed, and what tests prove it works.
- **If an AI-generated change breaks something, you can revert ONE task.** Not everything you've built this week.

The discipline of committing per task forces you to keep tasks small, which forces AI-generated work to stay tractable.

---

## The Commit Protocol

Complete these steps in order. Do not skip steps.

1. **Finish the task** — code written, all tests passing locally
2. **Update CHANGELOG.md** with a concise entry for this task
3. **Stage specific files** — list each file explicitly, not `git add .`
4. **Review staged diff** — `git diff --staged` — confirm only intended changes are staged
5. **Write a clear commit message** — see format below
6. **Commit**
7. Move to the next task

---

## CHANGELOG Discipline

- Update CHANGELOG **before** committing — never batch updates at the end
- Organize by semantic category: `feat`, `fix`, `refactor`, `chore`, `docs`, `test`, `perf`
- Each entry states **what changed and why**, not how (the diff shows how)
- Major changes can include sub-bullets for detail

Example CHANGELOG block:

```markdown
## [Unreleased]

### feat
- Add rate limiting to authentication endpoints — prevents brute-force attacks on login

### fix
- Correct session token expiry calculation — tokens were expiring 1 hour early due to timezone offset

### chore
- Upgrade bcrypt from 5.0 to 5.1 — resolves CVE-2023-XXXXX
```

---

## Commit Message Format

```
type(scope): short imperative description

Optional longer explanation — focus on WHY this change was made.
The diff already shows what changed, so the message body should
answer: why was this necessary? What problem does it solve?
```

**Types:**

| Type | When to use |
|---|---|
| `feat` | New capability added |
| `fix` | Bug corrected |
| `refactor` | Code restructured, no behavior change |
| `test` | Tests added or updated |
| `chore` | Tooling, dependencies, build, config |
| `docs` | Documentation only |
| `perf` | Performance improvement |

**Scope** is the module, domain, or feature area affected — e.g., `auth`, `api`, `db`, `ui`.

---

## Good vs Bad Examples

### Commit message quality

| Bad | Good |
|---|---|
| `update code` | `fix(auth): prevent session token leaking in error responses` |
| `wip` | `feat(payments): add idempotency key support to charge endpoint` |
| `fix bug` | `fix(db): add missing index on user_id in audit_log table` |
| `changes` | `refactor(queue): extract retry logic into shared helper` |

The bad messages tell you nothing when you're reading `git log` six months later. The good messages let you understand what happened without opening the diff.

### Commit size

**Bad: one commit, 15 changed files, mixed concerns**

```
feat: add user authentication system

M src/auth/login.js
M src/auth/logout.js
M src/auth/token.js
M src/db/schema.sql
M src/db/migrations/001.sql
M src/api/users.js
M src/api/sessions.js
M src/middleware/auth.js
M src/config/env.js
M tests/auth.test.js
M tests/api.test.js
M tests/db.test.js
M docs/auth.md
M CHANGELOG.md
M package.json
```

You cannot revert login without also reverting the database schema and unrelated API changes.

**Good: 4 commits, each independently testable**

```
feat(db): add users and sessions tables with indexes
feat(auth): implement JWT token generation and validation
feat(api): add /login and /logout endpoints
feat(middleware): protect routes with session validation
```

Each commit builds on the previous. Each is independently `git revert`-able.

### Staging discipline

**Bad:**
```bash
git add .
# Accidentally stages: .env, node_modules/.cache, debug.log, build/
```

**Good:**
```bash
git add src/auth/login.js src/auth/token.js tests/auth.test.js CHANGELOG.md
git diff --staged   # review before committing
git commit -m "feat(auth): implement JWT token generation and validation"
```

---

## Staging Discipline

Always add files by name. Never use `git add .` or `git add -A`.

**Files that must never be committed:**

- `.env`, `.env.local`, `.env.production` — contain secrets
- Credential files (`credentials.json`, `service-account.json`, `*.pem`, `*.key`)
- `node_modules/`, `__pycache__/`, `.venv/` — dependency directories
- Large binaries — videos, archives, compiled artifacts
- Editor artifacts — `.DS_Store`, `*.swp`, `Thumbs.db`
- Build output — `dist/`, `build/`, `.next/` (unless you explicitly need to commit it)

Ensure these are in `.gitignore` **before** you start working, not after you accidentally stage them.

---

## Anti-Patterns

**"I'll commit everything at the end"**
By the end of a multi-task session, you have an unrevertable mess. If one piece breaks production, you have to revert everything or surgically patch — neither is fast.

**"This is just a small change"**
Small changes still get their own commit. The cost of committing is near zero. The cost of not committing when you need to debug later is high.

**Amending published commits**
Once a commit has been pushed to a shared branch, never amend it. Create a new commit that corrects the issue. Amending rewrites history and causes conflicts for everyone else.

**Skipping CHANGELOG**
Future-you won't remember why you made a change. A brief CHANGELOG entry takes 30 seconds and saves hours of archaeology.

**Batching "related" changes**
"These two tasks are related so they should be one commit." No. Each task is independently verifiable. Independent verification = independent commit.

**Letting the AI batch everything**
AI assistants will, by default, make all requested changes at once. Explicitly instruct the AI to work task-by-task and commit after each before continuing.

---

## When Batching Is Acceptable

Almost never. Reasonable exceptions:

- **Mass rename/refactor** — renaming a type across 30 files is one atomic refactor commit. Use `replace_all` or IDE refactoring, then commit as one `refactor(core): rename UserRecord to UserProfile`.
- **Generated files alongside source** — a database migration file and the model code that requires it can reasonably be one commit, since neither works without the other.
- **Lockfile updates** — `package-lock.json` or `yarn.lock` updates go with the dependency change that caused them.

The test: **can each piece be independently understood and reverted?** If yes, split them. If no, they can be together.

---

## Checklist Before Every Commit

```
[ ] All tests pass locally
[ ] CHANGELOG.md updated
[ ] Only intended files staged (git diff --staged reviewed)
[ ] No secrets, credentials, or large binaries staged
[ ] Commit message follows type(scope): description format
[ ] Message body explains WHY (if non-obvious)
```
