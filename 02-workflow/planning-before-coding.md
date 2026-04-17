# Planning Before Coding

## The Problem

AI generates code fast. Dangerously fast. Without a plan, it generates code in the wrong direction — wrong architecture, wrong file locations, wrong abstractions — and you don't discover this until you've accepted six commits.

Three failure modes happen constantly with unplanned AI-assisted coding:

1. **Direction drift** — The AI solves a related problem but not the actual one. You only notice when you try to wire things together.
2. **Scattered changes** — Files get created in random directories because nobody established a file map upfront. Later cleanup is expensive.
3. **Vague plans** — The plan says "add appropriate error handling" and the AI fills in something plausible but wrong. You can't review what you can't read.

## The No-Placeholder Rule

This is the single most important rule for writing plans:

> **Every step must contain ACTUAL code, not descriptions of code.**

A plan is not a plan if it contains:

| Placeholder | Why it fails |
|---|---|
| "Add appropriate error handling" | The AI decides what "appropriate" means — not you |
| "Similar to Task 3" | Copy the full code adapted. Similarity is not specification |
| "TBD" | The plan is not ready. Stop and finish it |
| "Implement the validation logic" | Show the validation. What inputs? What rules? What errors? |
| "Wire up the service" | Which methods? Which arguments? What does the return value look like? |

If you cannot write the actual code in the plan, it means you do not yet understand the task well enough to delegate it. Fix your understanding first.

## Plan Structure

Every plan follows this four-part structure:

### 1. Goal

One sentence. What does "done" look like from the outside?

```
Goal: Users can submit a contact form and receive a confirmation email within 30 seconds.
```

Not "implement the contact form feature." Done means observable, testable behavior.

### 2. File Map

A table of every file you will touch. New files and modified files both appear here.

| File | Action | Responsibility |
|---|---|---|
| `src/domain/notifications/service.js` | CREATE | Send transactional email via provider adapter |
| `src/domain/notifications/repository.js` | CREATE | Persist notification records to DB |
| `src/domain/notifications/contract.js` | CREATE | Error builders, DTO normalizers |
| `src/adapters/email/sendgrid-adapter.js` | CREATE | SendGrid HTTP client wrapper |
| `src/domain/contact/service.js` | MODIFY | Call notification service after form submission |
| `tests/notifications-service.test.js` | CREATE | Unit tests for notification service |
| `web/src/app/api/contact/route.js` | MODIFY | Add POST handler, call contact service |

If a file is not in this table, it should not be touched during execution. If you discover mid-execution that another file needs changing, stop and update the plan.

### 3. Task Breakdown

Ordered steps, each representing 2–5 minutes of actual work. Each task is:

- Small enough to fit in a single commit
- Verified by a test that existed before the implementation
- Described with the actual code that will be written

**Task format:**

```
## Task N: [Name]

Files: src/path/to/file.js, tests/path/to/file.test.js

Test to write first:
  it('sends email when contact form is submitted', async () => {
    const mockAdapter = { send: mock.fn().mockResolvedValue({ id: 'msg-123' }) }
    const svc = new NotificationService({ adapter: mockAdapter })
    await svc.sendContactConfirmation({ email: 'user@example.com', name: 'Alice' })
    assert.strictEqual(mockAdapter.send.mock.calls.length, 1)
    assert.strictEqual(mockAdapter.send.mock.calls[0].arguments[0].to, 'user@example.com')
  })

Implementation:
  export class NotificationService {
    constructor({ adapter }) {
      this.adapter = adapter
    }

    async sendContactConfirmation({ email, name }) {
      const err = validateEmail(email)
      if (err) throw Object.assign(new Error(err.message), { code: err.code })
      return this.adapter.send({
        to: email,
        subject: 'Thanks for reaching out',
        body: `Hi ${name}, we received your message and will reply within 24 hours.`,
      })
    }
  }

Commit: feat(notifications): sendContactConfirmation method with adapter injection
```

If you cannot write the test and the implementation before executing the task, the task is not ready.

### 4. Verification

How do you prove the entire plan succeeded? Specific commands, not descriptions.

```bash
# Run the unit tests
node --test tests/notifications-service.test.js

# Run the integration test against a staging environment
curl -X POST http://localhost:3000/api/contact \
  -H "Content-Type: application/json" \
  -d '{"name":"Alice","email":"alice@example.com","message":"Hello"}'
# Expected: 200 { "ok": true }
# Expected: email arrives in mailhog at http://localhost:8025
```

## Task Breakdown Rules

Each task must satisfy all four conditions:

1. **Single responsibility** — One behavior changes. If you find yourself writing "and also" in the task name, split it.
2. **Test-first** — The test is written in the plan before execution begins. The test fails before the implementation exists.
3. **Actual code** — No pseudocode, no "something like this", no "adapt as needed."
4. **Atomic commit** — The task produces exactly one commit with one clear message.

## Self-Review Checklist

Before handing a plan to an agent or starting execution yourself:

- [ ] Every acceptance criterion from the story/ticket has a corresponding task
- [ ] No "TBD", "TODO", "implement later", "add appropriate", "similar to" anywhere in the plan
- [ ] Types and interfaces are consistent across tasks (Task 3 uses the same shape Task 1 produces)
- [ ] Import paths reference files that exist OR are created in an earlier task
- [ ] Test file locations match the project's existing conventions
- [ ] Each task has a commit message written in advance
- [ ] The verification section contains runnable commands, not descriptions
- [ ] The file map accounts for every file that will be touched

## Complete Plan Example

```markdown
# Plan: Add rate limiting to the login endpoint

Goal: The POST /api/auth/login endpoint rejects requests exceeding 5 attempts per IP
      per 15-minute window with a 429 response containing a Retry-After header.

## File Map

| File | Action | Responsibility |
|---|---|---|
| src/infrastructure/rate-limiter/store.js | CREATE | In-memory sliding window counter |
| src/infrastructure/rate-limiter/middleware.js | CREATE | Express middleware factory |
| tests/rate-limiter.test.js | CREATE | Unit tests for store and middleware |
| src/server.js | MODIFY | Apply rate limiter middleware to /api/auth/login |

## Tasks

### Task 1: Sliding window counter

Files: src/infrastructure/rate-limiter/store.js, tests/rate-limiter.test.js

Test:
  it('allows requests below the limit', () => {
    const store = new SlidingWindowStore({ limit: 5, windowMs: 15 * 60 * 1000 })
    for (let i = 0; i < 5; i++) {
      assert.strictEqual(store.hit('1.2.3.4'), true)
    }
  })

  it('blocks the 6th request within the window', () => {
    const store = new SlidingWindowStore({ limit: 5, windowMs: 15 * 60 * 1000 })
    for (let i = 0; i < 5; i++) store.hit('1.2.3.4')
    assert.strictEqual(store.hit('1.2.3.4'), false)
  })

  it('resets after the window expires', () => {
    const clock = { now: () => 0 }
    const store = new SlidingWindowStore({ limit: 5, windowMs: 1000, clock })
    for (let i = 0; i < 5; i++) store.hit('1.2.3.4')
    clock.now = () => 1001
    assert.strictEqual(store.hit('1.2.3.4'), true)
  })

Implementation:
  export class SlidingWindowStore {
    #buckets = new Map()
    constructor({ limit, windowMs, clock = Date }) {
      this.limit = limit
      this.windowMs = windowMs
      this.clock = clock
    }
    hit(key) {
      const now = this.clock.now()
      const bucket = this.#buckets.get(key) ?? { count: 0, resetAt: now + this.windowMs }
      if (now >= bucket.resetAt) {
        bucket.count = 0
        bucket.resetAt = now + this.windowMs
      }
      if (bucket.count >= this.limit) return false
      bucket.count++
      this.#buckets.set(key, bucket)
      return true
    }
  }

Commit: feat(rate-limiter): sliding window counter with injectable clock

---

### Task 2: Express middleware factory

Files: src/infrastructure/rate-limiter/middleware.js, tests/rate-limiter.test.js (append)

Test:
  it('returns 429 with Retry-After when limit exceeded', async () => {
    const store = { hit: () => false, retryAfterSeconds: () => 30 }
    const middleware = createRateLimiter({ store })
    const req = { ip: '1.2.3.4' }
    const res = { status: mock.fn().mockReturnThis(), json: mock.fn(), setHeader: mock.fn() }
    const next = mock.fn()
    middleware(req, res, next)
    assert.strictEqual(res.status.mock.calls[0].arguments[0], 429)
    assert.strictEqual(next.mock.calls.length, 0)
  })

Implementation:
  export function createRateLimiter({ store }) {
    return function rateLimiter(req, res, next) {
      const allowed = store.hit(req.ip)
      if (!allowed) {
        res.setHeader('Retry-After', store.retryAfterSeconds(req.ip))
        return res.status(429).json({ error: 'Too many requests', code: 'RATE_LIMIT_EXCEEDED' })
      }
      next()
    }
  }

Commit: feat(rate-limiter): Express middleware with 429 + Retry-After

---

### Task 3: Wire into server

Files: src/server.js

Test: (integration — manual curl verification in verification section)

Implementation change in src/server.js:
  // ADD after existing imports
  import { SlidingWindowStore } from './infrastructure/rate-limiter/store.js'
  import { createRateLimiter } from './infrastructure/rate-limiter/middleware.js'

  // ADD before app.use('/api/auth', authRouter)
  const loginLimiter = createRateLimiter({
    store: new SlidingWindowStore({ limit: 5, windowMs: 15 * 60 * 1000 }),
  })
  app.use('/api/auth/login', loginLimiter)

Commit: feat(auth): apply rate limiting to login endpoint (5 req/15 min per IP)

## Verification

  node --test tests/rate-limiter.test.js
  # Expected: all tests pass

  # Test the endpoint — first 5 succeed
  for i in {1..5}; do
    curl -s -o /dev/null -w "%{http_code}\n" \
      -X POST http://localhost:3000/api/auth/login \
      -H "Content-Type: application/json" \
      -d '{"email":"test@example.com","password":"wrong"}'
  done
  # Expected: 401 401 401 401 401

  # 6th request is rate limited
  curl -i -X POST http://localhost:3000/api/auth/login \
    -H "Content-Type: application/json" \
    -d '{"email":"test@example.com","password":"wrong"}'
  # Expected: HTTP/1.1 429, Retry-After header present
```

## Execution Options

Once the plan is complete and passes the self-review checklist:

### Option 1: Subagent-driven (recommended for plans with 4+ tasks)

Spawn a fresh agent per task. Each agent receives:
- The single task description (full code, test, commit message)
- The project's CLAUDE.md or equivalent context file
- No memory of previous tasks (forces clean, atomic work)

Benefits: No context bleed between tasks. Each agent starts fresh. Failures are isolated.

### Option 2: Inline execution (for plans with 1–3 tasks)

Execute sequentially in the current session. After each task:
1. Run the test — confirm it passes
2. Commit with the pre-written message
3. Move to the next task

Never execute more than one task before committing. The discipline of small commits is not optional.

## Anti-Patterns

**"Let me just start and figure it out"**
This produces scattered changes across files you didn't intend to touch, architectural decisions made mid-flight that contradict earlier decisions, and no way to course-correct without a full revert.

**Plans with only descriptions**
"The service should validate the input and call the repository" tells the AI nothing it didn't already know. It will generate something plausible. Plausible is not correct.

**Plans without tests**
You have no way to verify completion. The AI says it's done. You accept the diff. Three features later, something breaks and you have no regression coverage.

**Plans without a file map**
Files appear in unexpected locations. You grep for the function later and find two implementations in different directories. One of them is used. You're not sure which.

**Updating the plan during execution**
If you discover mid-task that the plan is wrong, stop. Update the plan. Re-run the self-review checklist. Then continue. Don't improvise around a broken plan.

## When to Skip Planning

Planning has overhead. Skip it only when all three conditions are true:

1. The change is confined to a single file
2. The total diff will be under 20 lines
3. The behavior is already covered by existing tests

Examples where skipping is safe: fixing a typo in a string literal, adjusting a CSS value, changing a config default. Everything else gets a plan.
