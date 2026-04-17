# Memory System

Claude Code sessions are stateless by design. Every time you open a new terminal and run `claude`, the slate is clean. This is a feature — it keeps the AI focused and prevents contamination from unrelated past work. But it also means that hard-won context evaporates: the reason you chose one architecture over another, the deploy gotcha that cost you three hours, the preference for how you like code explained to you.

The memory system is how you bridge sessions without pasting giant context blocks every time.

---

## The Problem

Without a memory system, every session starts with the same ritual:

- "We use PostgreSQL, not MySQL"
- "Don't mock the database in integration tests — we have a test DB"
- "The staging environment uses a different API key format than production"
- "I already told you last week: never restart the whole cluster, only the affected service"

This is wasted time. Worse, Claude might give subtly wrong advice because it lacks the institutional context you've built up. The memory system fixes this by loading relevant facts before you type a single word.

---

## How Memory Works in Claude Code

Claude Code reads memory from files stored at:

```
~/.claude/projects/{url-encoded-project-path}/memory/
```

For a project at `/home/alice/projects/myapp`, the memory directory would be:
```
~/.claude/projects/-home-alice-projects-myapp/memory/
```

The entry point is always `MEMORY.md` — an index file that Claude loads at session start. Each line in the index is a one-line summary with a link to a specific memory file:

```
~/.claude/projects/.../memory/
  MEMORY.md                    ← always loaded, index only
  user_role.md                 ← who you are and how you work
  feedback_no_db_mocks.md      ← a correction you gave Claude
  project_auth_decisions.md    ← why you chose JWT over sessions
  reference_infra_docs.md      ← where to find your runbooks
```

**Important:** Only `MEMORY.md` is guaranteed to be read on every session start. The linked files are referenced when Claude needs to expand on a topic. Keep `MEMORY.md` concise — it occupies prime context window space.

---

## Four Types of Memory

### 1. User Memory

Who you are, what you already know, and how you like to work. This is the most personal category and has the highest leverage — a single well-written user memory prevents dozens of mismatched explanations.

**What to capture:**
- Your technical background ("senior backend engineer, new to React")
- Communication preferences ("terse explanations, no preamble")
- Collaboration style ("autonomous execution preferred — don't ask for confirmation")
- Specific knowledge gaps ("explain CSS layout concepts in terms of flexbox, I don't know Grid yet")

**Example file** (`user_role.md`):
```markdown
---
name: user-role
description: Background and collaboration preferences
type: user
---

Senior backend engineer (10 years Python/Go), learning frontend for the first time.

**Preferences:**
- Frame frontend concepts in backend terms when possible (e.g., "React state is like a request-scoped variable")
- Skip explaining concepts I likely know: async/await, REST, SQL, Docker, CI/CD
- Autonomous execution — make decisions and go, only ask when genuinely blocked
- Terse responses preferred; use bullet points over paragraphs for summaries
```

### 2. Feedback Memory

Corrections you've made to Claude's behavior. This is the most operationally valuable category. Each time you say "no, don't do it that way" — and especially when that correction required effort to discover — it deserves a memory entry.

**What to capture:**
- Explicit corrections: "I told you not to do X but you did it again"
- Hard-learned lessons from incidents: "a restart once took down 3 services"
- Non-obvious project rules that contradict common practice

**Structure:** Rule → **Why:** (the incident that produced the rule) → **How to apply:** (when the rule triggers)

The "Why" is critical. Rules without reasons get ignored or misapplied. Rules with reasons get internalized.

**Example file** (`feedback_no_global_restart.md`):
```markdown
---
name: no-global-restart
description: Never use global process manager commands on shared servers
type: feedback
---

Always scope process manager commands to the specific project config file. Never run commands that affect all processes on a host.

**Why:** During a routine deploy, a global restart command took down two unrelated services on the same server. One of them was a payment webhook receiver — it missed events during the downtime.

**How to apply:** Before any deploy or restart command, check:
1. Does this command target only my project's processes?
2. Are there other projects on this host that could be affected?
If yes to (2), always use a project-scoped config or process name.
```

**Example file** (`feedback_integration_test_db.md`):
```markdown
---
name: integration-test-db
description: Use the real test database in integration tests, not mocks
type: feedback
---

Do not mock the database layer in integration tests. We maintain a dedicated test database for exactly this purpose.

**Why:** Mocked DB tests passed while real data bugs slipped through. The integration test suite exists to catch Prisma query logic, constraint violations, and migration edge cases — none of which mocks can exercise.

**How to apply:** When writing integration tests (files in `tests/integration/`), always connect to `DATABASE_URL_TEST`. Unit tests (files in `tests/unit/`) may use mocks.
```

### 3. Project Memory

Decisions, constraints, and business context that can't be derived from reading the code. This fills the gap between "what the code does" and "why it was built this way."

**What to capture:**
- Architectural decisions and their rationale
- Technology choices that were actively debated
- Current priorities and deadlines (use absolute dates, never relative ones)
- Constraints from external systems or stakeholders

**What NOT to capture:** Active implementation details. "We are currently migrating from X to Y" ages poorly. Once the migration is done, the memory becomes misleading.

**Example file** (`project_auth_decisions.md`):
```markdown
---
name: auth-decisions
description: Why we use session tokens instead of JWTs for user auth
type: project
---

User authentication uses opaque session tokens stored in the database, not JWTs.

**Why:** JWTs cannot be invalidated server-side without an allowlist (which negates their stateless benefit). After a security incident in 2024 where we needed to force-logout all users, the team decided server-side sessions are worth the extra DB lookup.

**How to apply:** Do not suggest JWT-based auth for user-facing routes. JWT is acceptable for service-to-service calls where token lifetime is short (< 1 hour) and revocation is not required.
```

**Example file** (`project_current_priority.md`):
```markdown
---
name: current-priority
description: Active sprint focus as of 2025-03-01
type: project
---

Sprint running 2025-03-01 through 2025-03-14. Single focus: completing the data export feature.

All other work (refactors, nice-to-haves, new features) deferred until export ships. If a task would block export, escalate immediately. If it would not, defer it.
```

### 4. Reference Memory

Pointers to external systems, dashboards, runbooks, and documentation that Claude might need when helping you. These are breadcrumb files — they tell Claude *where* to look, not what the answer is.

**What to capture:**
- Issue tracker locations and label conventions
- Where production logs live
- Links to external API documentation
- On-call rotation or escalation paths

**Example file** (`reference_infra_docs.md`):
```markdown
---
name: infra-docs
description: Where to find infrastructure runbooks and monitoring
type: reference
---

- **Runbooks**: Internal wiki at wiki.internal/runbooks — covers deploy, rollback, DB backup/restore
- **Production logs**: Centralized logging dashboard at logs.internal — filter by `service=api` or `service=worker`
- **Alerts**: PagerDuty team "backend-oncall" — escalate P1 incidents there
- **Pipeline bugs**: Tracked in Linear project "INGEST" — label `bug` + `pipeline`
- **API rate limits**: Third-party API docs at docs.thirdparty.example/rate-limits — limits change quarterly, always verify before load testing
```

---

## What NOT to Save

Memory files consume context window. Bad memories are worse than no memories — they create false confidence and mislead Claude on stale information.

**Do not save:**
- **Code patterns** — If you have good code, Claude can read it and infer conventions. Saving "we use async/await everywhere" adds noise when the codebase itself shows this clearly.
- **Git history** — "We switched from library X to Y six months ago" is in `git log`. Use that.
- **Debugging session results** — The fix is in the code; the context is in the commit message. Don't save "I fixed a race condition in the job queue" as a memory.
- **Duplicates of CLAUDE.md** — Your project's `CLAUDE.md` is already loaded. Don't repeat its content in memory files.
- **Ephemeral task details** — "Currently working on ticket #42" belongs in a todo list, not a memory file. Memory is for durable facts.
- **Anything verifiable from current state** — "The production database is on version 14.5" should be checked live, not trusted from a memory that may be months old.

---

## Writing Good Memories

### The Two-Step Process

Every memory requires two actions:

1. **Write the memory file** with a YAML frontmatter header, the core rule or fact, and the Why/How context.
2. **Add a one-line pointer to `MEMORY.md`** so Claude can find it.

If you do step 1 without step 2, the memory is effectively invisible. If you do step 2 without step 1, the pointer leads nowhere.

### Memory File Format

```markdown
---
name: descriptive-slug
description: One-sentence summary of what this memory contains
type: feedback | user | project | reference
---

The core rule, fact, or pointer. State it clearly in 1-3 sentences.

**Why:** The incident, decision, or reasoning that produced this memory. Include enough detail that someone reading this cold can understand the stakes.

**How to apply:** The trigger conditions. When should Claude act on this? When does it NOT apply? Edge cases if they exist.
```

**Real example** (feedback type):

```markdown
---
name: deploy-db-backup-first
description: Always take a database backup before running migrations in production
type: feedback
---

Never run `prisma migrate deploy` (or any migration tool) in production without first creating a database snapshot.

**Why:** A failed migration on a table with no backup required 4 hours of manual reconstruction from application logs. The migration itself took 30 seconds. The backup would have taken 2 minutes.

**How to apply:** Before any production deploy that includes a schema migration:
1. Confirm a backup was taken in the last 30 minutes, OR
2. Trigger a manual snapshot and wait for confirmation
Only proceed after backup is confirmed. This applies to all environments except local dev.
```

### MEMORY.md Index Format

Keep each entry to one line: a linked name and a plain-English description of when this memory matters.

```markdown
- [Deploy DB backup first](feedback_deploy_db_backup_first.md) — Take DB snapshot before any production migration
- [No global restarts](feedback_no_global_restart.md) — Never use global process manager commands on shared hosts
- [Integration test DB](feedback_integration_test_db.md) — Real test DB in integration tests, not mocks
- [Auth decisions](project_auth_decisions.md) — Why we use sessions over JWTs
- [User role](user_role.md) — Senior backend eng, new to React — explain frontend in backend terms
- [Infra docs](reference_infra_docs.md) — Where to find runbooks, logs, and alerts
```

The descriptions should answer: "If Claude is about to do something, which of these memories might be relevant?" Write them from that perspective.

---

## Memory Hygiene

### Keep MEMORY.md Under 200 Lines

The index file is always loaded into context. Past 200 lines, it starts crowding out useful working context. If your index is growing too large:

- Merge related memories into a single file
- Delete memories for resolved issues or completed projects
- Ask yourself: "Has this come up in the past month?" If no, archive it

### Convert Relative Dates to Absolute Dates

Never write "the current sprint" or "last week's deploy" in a memory. By the time you read it again, those phrases are meaningless.

Write instead: `Sprint 2025-03-01 through 2025-03-14` or `Production deploy on 2025-02-28 caused the outage`.

### Update, Don't Append

When a situation changes, update the memory — don't add a new one that contradicts the old one. Two conflicting memories are worse than zero memories.

For example, if you change your database from PostgreSQL to SQLite for a local-first product, update the existing project memory rather than adding "we switched to SQLite" alongside the old "we use PostgreSQL" entry.

### Verify Before Acting

Memories describe the state of the world at the time they were written. Before acting on a memory that involves external systems (file locations, service URLs, team structure), verify it's still accurate.

A memory saying "runbooks are at wiki.internal/runbooks" is a starting point for investigation, not a guaranteed truth.

---

## When Claude Accesses Memory

Memory is loaded at session start and referenced when context from prior sessions becomes relevant. Specifically:

- **Architecture or tooling questions** — Claude checks if there are project memories explaining prior decisions
- **Deploy or operations tasks** — Claude checks for feedback memories about operational gotchas
- **Explaining concepts** — Claude checks user memory to calibrate explanation depth
- **Locating external resources** — Claude checks reference memory before asking you where something lives

You can also prompt explicitly: "Check your memories about database testing before writing this test."

---

## Anti-Patterns

### Saving Everything

Memory is not a transcript. If every session ends with five new memory files, most of them will be noise within a week. Save memories for durable, non-obvious facts that have actually caused problems or confusion.

**Signal for "worth saving":** "I've had to explain this more than once" or "this mistake cost real time."

### Never Updating

Stale memories mislead. A memory saying "we target Node.js 16" when you've been on Node.js 22 for a year will confuse Claude on version-sensitive questions. Set a monthly calendar reminder to scan your memory files and prune or update anything that's changed.

### Duplicating CLAUDE.md

Your project's `CLAUDE.md` is the authoritative source for project conventions. If it already says "use pnpm, not npm," don't write a memory file saying the same thing. Duplication creates divergence — the files will eventually contradict each other.

### Saving Implementation Details

"We are currently refactoring the auth module to use middleware" is an implementation detail that will be wrong by next week. Once the refactor is done, this memory actively misleads. Save the *decision* ("we adopted middleware-based auth to simplify route-level permission checks"), not the in-progress work.

### Treating Memory as Source of Truth

Memory files describe a past state. Always verify against the current state of the codebase, infrastructure, or team when it matters. "The API uses basic auth" in a year-old memory might have been superseded by a security audit that added OAuth.

---

## Getting Started

If you're setting up memory for the first time, start small:

1. **Create the memory directory** for your project:
   ```bash
   mkdir -p ~/.claude/projects/$(pwd | sed 's|/|-|g; s|^-||')/memory
   ```

2. **Write one user memory** describing who you are and how you like to collaborate. This single file improves every future session.

3. **Create an empty `MEMORY.md` index:**
   ```bash
   touch ~/.claude/projects/.../memory/MEMORY.md
   ```

4. **After the next session where Claude makes a correctable mistake,** write a feedback memory. That's the right moment — the context is fresh and the stakes are clear.

5. **Review monthly.** Set a recurring reminder. It takes ten minutes and prevents the index from accumulating noise.

The goal is not a comprehensive archive of everything you've ever done. The goal is a small set of high-signal facts that make every future session start from a better position.
