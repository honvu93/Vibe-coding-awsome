# Example: A Debugging Session Walkthrough

This is a realistic debugging session following the 4-phase investigate-analyze-hypothesize-implement
protocol. The bug is a classic: an API endpoint returns 500 intermittently under load.

The session shows:
- What the developer thought was wrong first (and why they were wrong)
- How following the protocol forced the right answer to surface
- How much time was saved versus shotgun debugging

Estimated time following the protocol: **47 minutes**.
Estimated time with shotgun debugging (restarting server, changing things at random): **4+ hours** —
this is a class of bug that looks different every time you look at it.

---

## Setup: What We Know

**Report from staging**: "The `GET /api/tasks` endpoint occasionally returns 500. It happens
maybe 1 in 20 requests during the afternoon peak. Logs show a database error but it's not
consistent — same request succeeds when retried."

**Initial reaction (wrong)**: "Probably a flaky test environment. Let me restart the database
connection pool."

*This instinct is the enemy. Do not act on it.*

---

## Phase 1: Investigate

### Step 1.1 — Read the actual error log

```
[2024-03-15T14:23:11.042Z] ERROR GET /api/tasks 500 — 312ms
  Error: Cannot read properties of undefined (reading 'id')
    at normalizeTask (src/domain/tasks/contract.js:34:22)
    at tasks.findMany.map (src/domain/tasks/service.js:67:18)
    at Array.map (<anonymous>)
    at TasksService.listTasksForProject (src/domain/tasks/service.js:67:34)
    at handler (web/src/app/api/tasks/route.js:23:28)

[2024-03-15T14:23:11.503Z] INFO  GET /api/tasks 200 — 89ms
[2024-03-15T14:23:11.891Z] INFO  GET /api/tasks 200 — 102ms
[2024-03-15T14:23:12.103Z] ERROR GET /api/tasks 500 — 287ms
  Error: Cannot read properties of undefined (reading 'id')
    at normalizeTask (src/domain/tasks/contract.js:34:22)
```

**Observation**: The error is `undefined.id` in `normalizeTask`. It crashes inside `.map()`.
The same endpoint succeeds milliseconds later. This is not a network issue. This is a data
consistency issue — something returned from the database is unexpectedly `undefined`.

### Step 1.2 — Reproduce locally

```bash
# Hammer the endpoint with concurrent requests to reproduce
for i in $(seq 1 50); do
  curl -s -o /dev/null -w "%{http_code}\n" \
    "http://localhost:4100/api/tasks?projectId=proj_test01" &
done
wait
```

Result: 48 × 200, 2 × 500. Reproduced.

**Key observation**: The 500s only occur with concurrent requests, never with sequential ones.

### Step 1.3 — Check recent git changes

```bash
git log --oneline -10
```

```
a3f9c12 feat(tasks): add soft-delete support — tasks can now be archived
e7b2d44 fix(auth): extend session token expiry to 30 days
b1c8903 refactor(projects): extract member permission checks to access.js
9d2a110 feat(labels): add label color validation
```

**Key observation**: `a3f9c12` added soft-delete to tasks. That is the most recent change touching
the tasks domain. It was merged yesterday — matching when the 500s started appearing.

### Step 1.4 — Add diagnostics without changing behavior

```js
// Temporary diagnostic — NOT a fix, just visibility
// src/domain/tasks/service.js

async listTasksForProject(projectId, filters = {}) {
  const rows = await this.repository.findMany({ projectId, ...filters });

  // DIAGNOSTIC: log any undefined rows before the map crashes
  const undefinedRows = rows.filter((r) => r === undefined || r === null);
  if (undefinedRows.length > 0) {
    console.error('[DIAG] undefined rows in findMany result', {
      count: undefinedRows.length,
      total: rows.length,
      projectId,
    });
  }

  return rows.map((row) => normalizeTask(row));
}
```

Re-run the concurrent hammer test. Diagnostic output:

```
[DIAG] undefined rows in findMany result { count: 1, total: 12, projectId: 'proj_test01' }
[DIAG] undefined rows in findMany result { count: 1, total: 11, projectId: 'proj_test01' }
```

**Finding**: The `rows` array itself contains `undefined` entries. The database query is
returning `undefined` inside the array — not an empty array, not a null result, but a sparse
array with holes.

---

## Phase 2: Analyze

### Step 2.1 — Find a working similar endpoint

The `GET /api/projects` endpoint lists projects and has never returned 500. Read its repository.

```js
// src/domain/projects/repository.js
async findMany({ workspaceId }) {
  return this.prisma.project.findMany({
    where: { workspaceId, deletedAt: null },
    orderBy: { createdAt: 'desc' },
  });
}
```

Clean. Returns a Prisma result array directly. No custom array construction.

### Step 2.2 — Read the tasks repository after the soft-delete change

```js
// src/domain/tasks/repository.js — AFTER a3f9c12 (the soft-delete commit)

async findMany({ projectId, includeArchived = false }) {
  const tasks = await this.prisma.task.findMany({
    where: {
      projectId,
      ...(includeArchived ? {} : { archivedAt: null }),
    },
    include: { assignee: true, labels: true },
    orderBy: { createdAt: 'desc' },
  });

  // Build result — merge with in-progress cache for real-time feel
  const cached = await this.cache.getProjectTasks(projectId);        // <-- added in soft-delete commit
  const merged = tasks.map((t) => cached.find((c) => c.id === t.id) ?? t);
  return merged;
}
```

### Step 2.3 — Read the cache implementation

```js
// src/infrastructure/cache/project-tasks-cache.js

export class ProjectTasksCache {
  constructor() {
    this._store = new Map(); // projectId → Task[]
  }

  async getProjectTasks(projectId) {
    return this._store.get(projectId) ?? [];
  }

  async setProjectTasks(projectId, tasks) {
    this._store.set(projectId, tasks);
  }

  async invalidate(projectId) {
    this._store.delete(projectId);
  }
}
```

**Key finding**: The cache stores full task objects. The `merged` array maps over database results
and replaces each with its cached version if found. If a task exists in the DB but NOT in the
cache (e.g., the cache was populated before the task was created), `cached.find()` returns
`undefined`, so `cached.find(...) ?? t` falls back to `t` — which is correct.

But what happens when `cache.getProjectTasks` returns a stale array that STILL contains a task
that was soft-deleted (set `archivedAt`) between the time the cache was populated and the time
this request runs?

The DB query filters `archivedAt: null`, so the archived task is excluded from `tasks`.
The `merged = tasks.map(...)` only iterates over DB results. The archived task in the cache is
never merged in. That is fine.

**Wait — re-read the merge logic:**

```js
const merged = tasks.map((t) => cached.find((c) => c.id === t.id) ?? t);
```

This is correct. Let's look at the invalidation path instead.

```bash
git show a3f9c12 --stat
```

```
src/domain/tasks/repository.js      | 28 +++++++++++++++
src/infrastructure/cache/...        | 55 +++++++++++++++
src/domain/tasks/service.js         | 12 +++++-
```

```bash
git show a3f9c12 -- src/domain/tasks/service.js
```

```diff
+  async archiveTask(taskId) {
+    await this.repository.archive(taskId);
+    await this.cache.invalidate(taskId);    // <-- BUG: invalidating by taskId, not projectId
+  }
```

**Root cause found**: When a task is archived, the cache is invalidated using `taskId` as the
key. But the cache is keyed by `projectId`. So `this.cache.invalidate(taskId)` calls
`this._store.delete(taskId)` — which deletes a key that does not exist. The cache for the
project is never cleared.

The stale cache still contains the archived task object. But the archived task IS filtered out
by the DB query. So `cached.find((c) => c.id === t.id)` cannot return the archived task because
the archived task is not in `tasks` anymore. The array stays the right length.

Wait — still no undefined. Let me re-read more carefully.

**Second look at the cache store:**

```js
// What is in the cache store for a projectId?
// It is set here:
async setProjectTasks(projectId, tasks) {
  this._store.set(projectId, tasks);
}
```

```bash
git show a3f9c12 -- src/domain/tasks/service.js | grep -A5 setProjectTasks
```

```diff
+  async listTasksForProject(projectId, filters) {
+    const tasks = await this.repository.findMany({ projectId, ...filters });
+    await this.cache.setProjectTasks(projectId, tasks);    // cache result
+    return tasks;
+  }
```

**The cache stores the result AFTER the repository merge.** So the stored value is already a
merged array. Then on the next call, `getProjectTasks` returns that merged array, and `findMany`
calls `tasks.map((t) => cached.find(...) ?? t)`.

Under concurrent requests:

1. Request A: `findMany` starts. Gets 12 tasks from DB. Cache miss — returns [].
2. Request B: `findMany` starts concurrently. Gets 12 tasks from DB. Cache miss — returns [].
3. Request A: Merges. Result: 12 tasks. Sets cache: 12 items.
4. Request A: `service.js` receives 12 tasks. Sets cache AGAIN with service result.
5. Meanwhile, task #7 is archived (deleted from DB query) between Request B's DB read and its
   merge step.
6. Request B: Merges tasks against cache (now populated by Request A with task #7 still present).
   `tasks` has 11 items (task #7 filtered out by `archivedAt`). Cache has 12 items.
   `tasks.map(t => cached.find(c => c.id === t.id) ?? t)` — correct, 11 items.
7. Request B's service.js caches these 11 items.
8. NEW request C arrives. `findMany` runs. DB returns 11 items. Cache has 11 items.
   Merge: 11 items. Fine.

Still no undefined. Let me look at what `archive` does to the DB record:

```bash
git show a3f9c12 -- src/domain/tasks/repository.js | grep -A10 "async archive"
```

```diff
+  async archive(taskId) {
+    const task = await this.prisma.task.update({
+      where: { id: taskId },
+      data: { archivedAt: new Date() },
+    });
+    // Invalidate the task slot in the cache store
+    const cached = await this.cache.getProjectTasks(task.projectId);
+    const idx = cached.findIndex((c) => c.id === taskId);
+    if (idx !== -1) {
+      delete cached[idx];    // <-- BUG HERE: creates a sparse array hole → undefined
+    }
+  }
```

**There it is.** `delete cached[idx]` on a JavaScript array does NOT remove the element. It
sets `cached[idx] = undefined` and keeps the array length the same — creating a sparse array.

The next time `findMany` runs and calls `tasks.map(t => cached.find(c => c.id === t.id) ?? t)`:
- `cached` contains `[task0, task1, undefined, task3, ...]`
- `cached.find(...)` skips `undefined` entries correctly
- But the result of the map is `[task0_merged, task1_merged, task3_merged, ...]` with 11 items

Still no undefined in the merge output... Let me re-read the full merge:

```js
const merged = tasks.map((t) => cached.find((c) => c.id === t.id) ?? t);
```

`cached.find((c) => c.id === ...)` — if `c` is `undefined`, this will throw
`Cannot read properties of undefined (reading 'id')`.

**Not in the map. In the find callback.** When iterating over the sparse array, `.find()` visits
`undefined` entries. `undefined.id` throws. This is the exact error in the stack trace.

---

## Phase 3: Hypothesize

**Hypothesis**: `delete cached[idx]` in `repository.js:archive()` creates a sparse array by
setting the index to `undefined` rather than splicing the element out. On the next call to
`findMany`, the merge logic calls `cached.find((c) => c.id === ...)`, which visits the `undefined`
slot and throws `Cannot read properties of undefined (reading 'id')`.

**Verify WITHOUT fixing — confirm the hypothesis is correct:**

```js
// Quick Node.js REPL verification — does not touch production code
const arr = ['a', 'b', 'c', 'd'];
delete arr[1];
console.log(arr);         // [ 'a', <1 empty item>, 'c', 'd' ]
console.log(arr.length);  // 4 — length unchanged

arr.find((x) => x.id);    // TypeError: Cannot read properties of undefined (reading 'id')
```

Hypothesis confirmed. The fix is one character: `splice` instead of `delete`.

**Why it is intermittent**: The `archive` repository method is only called when a task is
archived. The corrupt cache entry persists until the next write resets it. If the `findMany`
call happens between the `delete` and the next `setProjectTasks` that overwrites the corrupt
cache, it crashes. Under concurrent load this window is wide enough to hit 1-in-20 requests.

---

## Phase 4: Implement

### Step 4.1 — Write regression test first

```js
// Add to tests/tasks-repository.test.js

describe('TasksRepository.archive — cache behavior', () => {
  it('does not leave undefined holes in the cache after archiving a task', async () => {
    // Arrange: populate cache with 3 tasks
    const tasks = [
      { id: 'task_001', projectId: 'proj_001', title: 'First', archivedAt: null },
      { id: 'task_002', projectId: 'proj_001', title: 'Second', archivedAt: null },
      { id: 'task_003', projectId: 'proj_001', title: 'Third', archivedAt: null },
    ];

    const cache = new ProjectTasksCache();
    await cache.setProjectTasks('proj_001', [...tasks]);

    const prisma = {
      task: {
        update: mock.fn(async () => ({ id: 'task_002', projectId: 'proj_001', archivedAt: new Date() })),
      },
    };

    const repo = new TasksRepository(prisma, cache);

    // Act: archive task_002
    await repo.archive('task_002');

    // Assert: cache must not contain undefined
    const cached = await cache.getProjectTasks('proj_001');
    assert.equal(cached.length, 2, 'cache should have 2 items after archive');
    assert.ok(cached.every((t) => t !== undefined), 'no undefined slots in cache');
    assert.ok(!cached.find((t) => t.id === 'task_002'), 'archived task is removed');
  });

  it('find() on cache result does not throw when accessing .id', async () => {
    const tasks = [
      { id: 'task_001', projectId: 'proj_001', title: 'First', archivedAt: null },
      { id: 'task_002', projectId: 'proj_001', title: 'Second', archivedAt: null },
    ];

    const cache = new ProjectTasksCache();
    await cache.setProjectTasks('proj_001', [...tasks]);

    const prisma = {
      task: {
        update: mock.fn(async () => ({ id: 'task_001', projectId: 'proj_001', archivedAt: new Date() })),
      },
    };

    const repo = new TasksRepository(prisma, cache);
    await repo.archive('task_001');

    const cached = await cache.getProjectTasks('proj_001');

    // This must not throw
    assert.doesNotThrow(() => {
      cached.find((c) => c.id === 'task_002');
    });
  });
});
```

Run the test — it fails:

```bash
node --test tests/tasks-repository.test.js
# FAIL: does not leave undefined holes in the cache after archiving a task
#   AssertionError: 2 == 2 but cache contains undefined slot
```

### Step 4.2 — Apply the fix

```js
// src/domain/tasks/repository.js — in the archive() method

// Before:
const idx = cached.findIndex((c) => c.id === taskId);
if (idx !== -1) {
  delete cached[idx];   // WRONG: leaves undefined hole
}

// After:
const idx = cached.findIndex((c) => c.id === taskId);
if (idx !== -1) {
  cached.splice(idx, 1);  // Correct: removes element, compacts array
}
```

Run the test — it passes:

```bash
node --test tests/tasks-repository.test.js
# PASS: does not leave undefined holes in the cache after archiving a task
# PASS: find() on cache result does not throw when accessing .id
```

### Step 4.3 — Remove diagnostic logging, run full suite

```bash
# Remove the temporary diagnostic added in Phase 1
# Then run all tests
npm test
# Expected: all tests pass
```

### Step 4.4 — Verify the concurrent scenario is resolved

```bash
# Start server
npm run api:dev &

# Archive a task while hammering the list endpoint
curl -X POST http://localhost:4100/api/tasks/task_001/archive &
for i in $(seq 1 50); do
  curl -s -o /dev/null -w "%{http_code}\n" \
    "http://localhost:4100/api/tasks?projectId=proj_001" &
done
wait
# Expected: all 200, no 500s
```

**Commit**: `fix(tasks): use splice instead of delete when removing archived task from cache`

**CHANGELOG entry**:

```markdown
## [fix] tasks: prevent sparse array crash when archiving tasks

`repository.archive()` was using `delete arr[idx]` to remove an archived task from the
in-memory cache array. In JavaScript, `delete` on an array index sets the slot to `undefined`
rather than compacting the array. A subsequent `Array.find()` call in `findMany()` would visit
the `undefined` slot and throw `Cannot read properties of undefined (reading 'id')`, producing
intermittent 500 errors under concurrent load. Fixed by using `splice(idx, 1)` instead.
Regression test added.
```

---

## Retrospective: What the Protocol Saved

| Action | Time without protocol | Time with protocol |
|---|---|---|
| Staring at logs, guessing | 45 min | 0 min |
| Restarting DB pool (wrong fix) | 20 min + re-investigation | 0 min |
| Random code changes to "see what happens" | 90 min | 0 min |
| Reading actual error location | 3 min | 3 min |
| Reproducing with concurrent load | 5 min | 5 min |
| Checking recent git changes | 2 min | 2 min |
| Finding working similar endpoint | 5 min | 5 min |
| Comparative read → finding the diff | 20 min | 20 min |
| REPL verification (no code change) | 0 min | 3 min |
| Writing regression test + fix | 15 min | 12 min |
| **Total** | **~4 hours** | **~47 minutes** |

**The most important phase was Phase 1, Step 1.3**: checking recent git changes. The bug was
introduced 24 hours ago in a specific commit touching a specific file. Without that step, the
investigation would have started from scratch on 34 domain modules.

**The most important discipline was Phase 3**: forming a hypothesis and verifying it with a
two-line REPL test BEFORE touching production code. The REPL showed `delete` creates a sparse
array in 30 seconds. Without that step, there is a real risk of applying a wrong fix — perhaps
changing the cache invalidation key (the first hypothesis about `taskId` vs `projectId`) — which
would have changed behavior without fixing the crash.
