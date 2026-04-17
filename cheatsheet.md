# Vibe Coding Cheatsheet
> Quick reference — one page. For full guide see individual chapters.

---

## Setup

| File / Dir | Purpose |
|---|---|
| `CLAUDE.md` | Commands → Architecture → Patterns → Auth → Testing → Commits |
| `.claude/WORKFLOW.md` | End-to-end implementation flow |
| `.claude/agents/` | Specialist agent definitions + routing table |
| `.claude/superpowers-skills/` | Process enforcement skill files |
| `memory/` | `user_*` / `feedback_*` / `project_*` / `reference_*` typed notes |

---

## TDD Cycle

```
RED → Verify RED → GREEN → Verify GREEN → REFACTOR
```

- Write test first. Watch it fail. Then write code.
- Iron Law: **No production code without a failing test.**
- One failing test at a time. Never batch.

---

## Debug Protocol

```
INVESTIGATE → ANALYZE → HYPOTHESIZE → IMPLEMENT
```

- Iron Law: **No fix without confirmed root cause.**
- Read actual error, not assumed error.
- Max 3 attempts on same hypothesis → question architecture.
- Reproduce in isolation before patching.

---

## Verification Gate

```
IDENTIFY → RUN → READ → VERIFY → CLAIM
```

| Red Flag | Correct |
|---|---|
| "should work" | Run it and read output |
| "probably passes" | Execute the test |
| "seems correct" | Diff against spec |

Never claim success without execution evidence.

---

## Commit Protocol

```
1. Task complete + all tests pass
2. Update CHANGELOG
3. Stage specific files  (not git add .)
4. git commit -m "clear description"
5. Start next task
```

- One task = one commit. Never batch.
- Never commit without CHANGELOG entry.

---

## Agent System

| Layer | Role | Scope |
|---|---|---|
| 1 — Planning | Story manager, PM, Architect | Stories, sprints, PRD, epics |
| 2 — Process | Superpowers skills | TDD, verify, review, branch hygiene |
| 3 — Specialists | Backend, Frontend, Security, DB, Reviewer | Domain-specific implementation |

**Routing rule:** Match task domain → dispatch correct specialist. Never route frontend work to backend agent.

---

## Subagent Protocol

- Fresh agent context per task — no shared state.
- Provide: task spec + relevant files + acceptance criteria.
- Two-stage review before merge:

| Stage | Question |
|---|---|
| Spec review | Does it do what was asked? |
| Quality review | Is it well-written and safe? |

**Exit states:** `DONE` / `DONE_WITH_CONCERNS` / `NEEDS_CONTEXT` / `BLOCKED`

---

## Parallel Dispatch

| Safe to parallelize | Do NOT parallelize |
|---|---|
| Independent tasks | Tasks sharing the same file |
| Different modules/domains | Dependent tasks (B needs A's output) |
| Well-specified scope | Unclear or ambiguous scope |

---

## Code Review Severity

| Level | Action |
|---|---|
| **Blocker** | Must fix before merge |
| **Suggestion** | Should fix, discuss if not |
| **Nit** | Minor style, optional |

---

## Refactoring Triggers

Extract / split when:
- File exceeds ~1000 LOC
- Circular dependencies detected
- Same logic duplicated 3+ times
- Single module owns multiple unrelated concerns

**Protocol:** Commit per step. Run tests after every commit. Draw dependency graph before and after.

---

## Deploy Checklist

**Pre-deploy**
- [ ] Tests pass
- [ ] Build succeeds
- [ ] Database backup complete
- [ ] CHANGELOG updated

**Post-deploy**
- [ ] Health-check every service (not just the one you touched)
- [ ] Check logs for errors
- [ ] Verify key user flows

**Never**
- Run global process manager commands on shared infrastructure
- Deploy without a rollback path
- Skip health checks on adjacent services

---

## Pattern Quick-Ref

| Pattern | When |
|---|---|
| Domain errors as plain objects with `.code` | Structured error handling |
| Repository layer for all DB access | Keep business logic out of queries |
| Contract / DTO normalization at boundary | Consistent API shapes |
| Policy snapshot at creation time | Freeze config, avoid live coupling |
| Graceful drain before shutdown | In-flight work safety |
