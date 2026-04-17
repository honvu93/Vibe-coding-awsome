# Parallel Agent Dispatch

## When to Use

Dispatch agents in parallel when you have two or more tasks that are:

- **Independent** — No shared state, no file conflicts between the tasks
- **Well-specified** — Each task has a clear scope and unambiguous constraints
- **Isolated** — Each agent works on different files, different modules, different layers

Parallelism is a performance optimization. Its value is proportional to task count and task duration. Two independent 10-minute tasks complete in 10 minutes in parallel instead of 20 minutes sequentially.

---

## When NOT to Use

- Tasks that modify the same files — one agent will overwrite the other
- Tasks where task B depends on task A's output — you cannot use B's output until A is done
- Tasks that share a database migration — ordering matters, parallel migrations conflict
- Any situation where you are unsure about independence — sequential is always safer

If you are uncertain whether two tasks are independent, treat them as dependent and run them sequentially. The cost of a conflict is higher than the cost of sequential execution.

---

## The Dispatch Protocol

### Step 1: Identify Independent Tasks

From your plan, group tasks by their dependency relationships:

```
Task A: Add input validation to user domain    → src/domain/users/
Task B: Add request logging middleware          → src/infrastructure/logging/
Task C: Update the profile settings component  → web/src/features/profile/

A, B, and C touch different directories with no shared imports → dispatch in parallel
```

Build a dependency graph, even informally. Tasks with no edges between them can run in parallel. Tasks with an edge (B needs A's output) must run sequentially.

### Step 2: Scope Each Agent

For each parallel agent, specify explicitly:

- **File allowlist** — the exact files or directories the agent may create or modify
- **File denylist** — files the agent must not touch (especially shared infrastructure)
- **Full context** — do not assume agents share knowledge; each prompt is independent
- **Independent acceptance criteria** — each agent verifies its own output

Why explicit file scope? Without it, an agent will make "helpful" changes to adjacent files. These unreviewed changes introduce risk and make the integration step harder.

### Step 3: Dispatch Simultaneously

Send all parallel agent tasks in a single message or batch, not sequentially. This is what enables true concurrent execution. If you send task A, wait for A, then send task B — you have gained nothing.

### Step 4: Collect and Verify Results

After all agents return:

1. Review each agent's output independently (spec compliance first, quality second — see `subagent-driven-dev.md`)
2. Scan for unexpected conflicts: did any agent modify a file outside its allowlist?
3. Run the FULL test suite after merging all outputs — not just the tests for each individual change
4. Resolve integration issues before proceeding to the next batch

Why run the full test suite? Each agent's change was correct in isolation. Integration can introduce failures that individual tests did not catch (e.g., agent A changed an interface that agent B's code calls).

### Step 5: Integration Verification

After merging all agent outputs into the working branch:

```bash
# Verify all tests still pass together
npm test             # or: pytest, cargo test ./..., go test ./...

# Verify no linting regressions
npm run lint         # or: flake8, clippy, golangci-lint

# Verify the project still builds
npm run build        # or: cargo build, go build ./...
```

All three checks must pass. A test suite that passes but a build that fails is not done.

---

## Conflict Resolution

If two agents accidentally modified the same file (despite scoping):

1. Identify which change is primary — the one that lives in that file's owning domain
2. Apply the primary change first
3. Manually integrate the secondary change on top
4. Run tests after integration to confirm both changes coexist correctly

Do not attempt to merge both changes automatically. Read both changes, understand what each does, and combine them deliberately.

---

## Practical Example

Plan has four tasks:

```
Task 1: Add rate limiting middleware             → src/infrastructure/rate-limit/
Task 2: Add user preferences API endpoint        → src/domain/users/
Task 3: Update the settings UI component         → web/src/features/settings/
Task 4: Write database migration for preferences → migrations/
```

Dependency analysis:

```
Task 2 depends on Task 4 — the endpoint reads from a table that does not exist yet
Task 1 is independent of all others
Task 3 is independent of all others
Task 4 is independent of Tasks 1 and 3
```

Dispatch strategy:

```
Parallel batch 1: Task 1 + Task 3 + Task 4
  (all independent, all touch different paths)

Collect + verify batch 1

Sequential: Task 2
  (depends on Task 4 being complete and migrated)

Collect + verify Task 2

Integration verification: run full test suite
```

This completes in the time of max(Task 1, Task 3, Task 4) + Task 2, instead of Task 1 + Task 2 + Task 3 + Task 4.

---

## Anti-Patterns

**Dispatching without verifying independence**
Two agents silently overwrite each other. The second agent's output appears correct in isolation but is missing changes from the first. Both agents declare success. You merge and half the feature is gone.

**Too-broad agent scope ("do everything in src/")**
The agent cannot be constrained to a single task. It touches files in multiple domains, some of which are being modified by another parallel agent. Conflict is guaranteed.

**Missing constraints**
An agent refactors a utility function "while it was in there." The refactor is benign on its own. Another parallel agent built on the original utility signature. Now they conflict.

**No integration verification after parallel work**
Each agent's tests pass. You merge all outputs. An interface changed by agent A breaks a caller modified by agent B. Neither test suite caught it because each tested in isolation.

**Assuming agents coordinate**
Parallel agents have no awareness of each other. They do not communicate, negotiate, or avoid conflicts. You are the coordinator. Scoping, sequencing, and integration verification are your responsibility.
