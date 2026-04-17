# Agent Routing

## The Problem

When you have multiple specialist agents, you need clear rules for which agent handles what. Without routing rules, you waste time deciding, or worse, the wrong agent touches the wrong code.

The most common failure mode: a backend agent "helpfully" updates a UI component, or a frontend agent reaches into a security module to "fix" a validation. Both agents act in good faith. Both cause problems.

---

## Routing by File Path

The simplest and most effective routing strategy. File paths are unambiguous, mechanical, and easy to enforce.

General pattern for a typical full-stack project:

```
src/domain/               → Backend Architect
src/infrastructure/
  security/               → Security Engineer
  db/ (or database/)      → Database Optimizer
  *                       → Backend Architect
src/adapters/             → Backend Architect
web/src/features/         → Frontend Developer
web/src/components/       → Frontend Developer
web/src/app/api/          → Backend Architect
web/src/lib/auth/         → Security Engineer
prisma/ (or migrations/)  → Database Optimizer
tests/                    → API Tester
scripts/                  → DevOps Automator
deploy/                   → DevOps Automator
```

Adapt this to your project's structure. The principle is: define ownership before work begins, not during a conflict.

---

## Routing by Task Type

When file path alone is not enough:

| Task Type | Primary Agent | When to Escalate |
|-----------|--------------|-----------------|
| New domain service | Backend Architect | If touches auth, escalate to Security |
| UI component | Frontend Developer | If needs a new API, loop in Backend |
| Schema change | Database Optimizer | If performance-sensitive, request review |
| Bug fix | Depends on file location | If 3+ attempts fail, escalate up |
| Security audit | Security Engineer | Always, no exceptions |
| Pre-merge review | Code Reviewer | Always, no exceptions |

---

## Decision Tree

```
New task arrives
  |
  What files does it touch?
    |
    Single domain → route to file path owner
    |
    Multiple domains → identify primary (where most code lives)
                        secondary agent reviews their portion only
  |
  Is it a security concern?
    |
    Yes → always Security Engineer, regardless of file path
  |
  Is it a review task?
    |
    Yes → always Code Reviewer
```

The security and review exceptions are absolute. They override file path routing.

---

## Rules for Agent Invocation

These rules prevent the most common dispatch mistakes:

1. **Never pass full conversation context.** Construct a minimal, focused prompt. Long context increases hallucination risk and token cost.
2. **Always include the task description, relevant file paths, and acceptance criteria.** Do not assume the agent remembers prior context — it does not.
3. **Always include project conventions.** Link to or paste the relevant sections of your CLAUDE.md or equivalent.
4. **Always specify the expected output format.** "Return the modified file" vs "return a diff" vs "return only the function" — be explicit.
5. **Never let agents modify files outside their domain without an explicit written reason.** If an agent proposes an out-of-domain change, it must justify why and you must approve it explicitly.

---

## Cross-Domain Tasks

When a task genuinely touches multiple domains (e.g., a new feature that needs a schema change, a new API endpoint, and a new UI component):

1. Break it into per-domain sub-tasks
2. Identify dependencies between sub-tasks (schema before endpoint, endpoint before UI)
3. Route each sub-task to its domain agent
4. Run each sequentially if dependent, or in parallel if independent
5. Never let two agents modify the same file simultaneously — this always produces a merge conflict or one agent undoing the other's work

For a worked example, see `parallel-dispatch.md`.

---

## Creating Your Routing Table

1. List your project's top-level directory structure
2. Assign each directory to an agent
3. Mark security and auth paths as Security Engineer overrides
4. Document the table in `.claude/agents/ROUTING.md` (or equivalent)
5. Reference the routing file from your WORKFLOW.md so it is consulted at the start of every task

The routing table should be a file in version control, not a convention held in memory. When a new team member (human or AI) joins, they read the routing table and immediately know the rules.

---

## Why Explicit Routing Matters

Agent routing is boring until it fails. The failure modes are:
- Two agents overwrite each other's changes
- A security-naive agent simplifies an intentionally complex auth check
- A backend agent changes a frontend contract, breaking the UI silently
- No one owns a cross-cutting file, so it accumulates changes from everyone

Explicit routing is cheap to set up and prevents all of these. Set it up before you need it.
