# Example: A Real CLAUDE.md

This is a complete, realistic CLAUDE.md for a fictional project — **Taskflow**, a task management
API with a Next.js frontend. Every section is filled in with believable production content. Nothing
is a placeholder.

Notice what makes this useful for an AI agent:

- Commands are copy-pasteable, not described
- Architecture explains naming conventions, not just file trees
- Patterns are explicit enough that an agent can infer the correct approach for a new file
- Credentials, env vars, and test runners are stated once and unambiguous

---

```markdown
# CLAUDE.md

This file provides guidance to Claude Code when working with this repository.

## Language

Reply to the user in English.

## Execution Mode

Operate fully autonomously. Do not ask for confirmation before taking actions — just execute.
Only stop to ask when genuinely blocked (missing credentials, ambiguous requirement with no
safe default).

## Commands

```bash
# Setup
npm install && pnpm --dir web install && npx prisma generate

# Full local stack (API :4100, frontend :4101)
npm run local:up

# Hot-reload for frontend work
npm run local:dev

# Backend only
npm run api:dev        # nodemon on src/server.js, port 4100

# Frontend only
pnpm --dir web dev     # Next.js dev server, port 4101

# Backend tests (Node.js built-in runner — NOT Jest)
npm test                                          # all tests
node --test tests/tasks-service.test.js           # single file
node --test tests/rate-limiter.test.js            # single file

# Frontend tests (Vitest + JSDOM)
npm run web:test                                             # all
pnpm --dir web vitest run tests/task-board.test.tsx         # single file

# Lint and build
npm run web:lint
npm run web:build

# Database
npx prisma migrate dev --name <migration-name>    # create + apply migration
npx prisma migrate deploy                         # apply pending migrations (production)
npx prisma studio                                 # DB GUI, port 5555

# Bootstrap reference data
npm run bootstrap:labels      # seed default label set
npm run bootstrap:roles       # seed default RBAC roles

# Deploy (run from local machine targeting VPS)
ssh deploy@app.example.internal "cd /opt/taskflow && ./deploy/deploy.sh"
```

## Architecture

Domain-driven full-stack task management system. PostgreSQL via Prisma. Pure ESM throughout
(`"type": "module"`). Node.js 22+.

### Backend (`src/`)

Each domain under `src/domain/{name}/` follows a strict layered pattern:

```
src/domain/tasks/
  service.js      — business logic, transaction boundaries
  repository.js   — Prisma data access, no raw SQL unless noted
  access.js       — permission checks, input validation
  contract.js     — DTO normalization, structured error builders
```

**Key domains:**

- `tasks` — CRUD, status transitions (OPEN → IN_PROGRESS → DONE → ARCHIVED), assignee changes
- `projects` — project membership, visibility settings, archival
- `comments` — threaded comments on tasks, @mention resolution
- `notifications` — notification fan-out, read/unread, delivery channel selection
- `users` — user CRUD, avatar upload, preference storage
- `labels` — label catalog per workspace, task tagging

**Supporting infrastructure (`src/infrastructure/`):**

- `db/prisma-client.js` — singleton PrismaClient, dev-global cached
- `security/token-box.js` — AES-256-GCM for API tokens (key: `<YOUR_TOKEN_SECRET>`)
- `queue/` — BullMQ job queues for async work (notifications, exports)
- `storage/upload-store.js` — local file staging with SHA-256 checksums
- `resilience/` — circuit breaker, exponential retry

**Adapters (`src/adapters/`):**

- `email/` — Nodemailer-based email delivery (SMTP)
- `push/` — Web Push notification delivery

### Frontend (`web/`)

Next.js 15 App Router, React 19, TailwindCSS 4, React Query (TanStack v5).

Path alias: `@/*` → `web/src/*`, `@core/*` → root `src/*`

```
web/src/app/(app)/             — protected workspace routes
web/src/app/(marketing)/       — public pages
web/src/app/api/               — API route handlers (proxy to domain services)
web/src/features/              — feature modules, each with components/ + lib/
web/src/lib/auth/              — Better Auth config, session cookie: tf_session
web/src/proxy.ts               — middleware protecting (app) routes
```

**Feature modules:** `tasks`, `projects`, `comments`, `notifications`, `user-settings`,
`workspace-admin`, `analytics`, `kanban-board`, `timeline-view`

### Prisma Schema

Key models: `User`, `Workspace`, `Project`, `Task`, `Comment`, `Label`, `Notification`,
`ApiCredential`, `AuditEvent`.

Key indexes: `Task.status + Project.id` (board queries), `Notification.userId + readAt`
(notification bell query).

### Data Flow

1. Frontend sends request to `/api/*` route handler
2. Route handler calls domain service with validated input
3. Domain service applies business rules, calls repository
4. Repository interacts with PostgreSQL via Prisma
5. BullMQ jobs handle async side effects (email, push)

### Key Patterns

**Error handling**: Plain `Error` objects with a `.code` property — never custom error classes.

```js
// Correct
const err = new Error('Task not found');
err.code = 'TASK_NOT_FOUND';
err.taskId = id;
throw err;

// Wrong — do not do this
throw new TaskNotFoundError(id);
```

**Domain errors**: Use `contract.js` builders:

```js
// src/domain/tasks/contract.js
export const errors = {
  notFound: (id) => Object.assign(new Error(`Task ${id} not found`), { code: 'TASK_NOT_FOUND', id }),
  forbidden: () => Object.assign(new Error('Forbidden'), { code: 'TASK_FORBIDDEN' }),
  invalidStatus: (from, to) =>
    Object.assign(new Error(`Cannot transition ${from} → ${to}`), {
      code: 'TASK_INVALID_STATUS_TRANSITION', from, to
    }),
};
```

**Naming conventions:**

- Repository methods: `findById`, `findMany`, `create`, `update`, `delete`, `upsert`
- Service methods: domain verbs — `assignTask`, `archiveProject`, `markNotificationRead`
- API routes: RESTful, plural nouns — `/api/tasks`, `/api/projects/:id/members`

**Response envelope (API routes):**

```json
{ "data": { ... }, "meta": { "timestamp": "2024-03-15T10:30:00Z" } }
```

Errors:
```json
{ "error": "Task not found", "code": "TASK_NOT_FOUND" }
```

## Auth

Better Auth with PostgreSQL adapter. Session cookie: `tf_session`.

**Dev credentials** (auto-seeded on first startup):

```
admin@example.local   /   dev-password   (role: admin)
user@example.local    /   dev-password   (role: member)
```

Override via env: `APP_ADMIN_EMAIL`, `APP_ADMIN_PASSWORD`, `<YOUR_AUTH_SECRET>`.

## Environment Variables

```bash
# Required
DATABASE_URL=postgresql://taskflow:taskflow@localhost:5432/taskflow
<YOUR_AUTH_SECRET>=change-me-in-production-32chars-minimum
REDIS_URL=redis://localhost:6379

# Optional with defaults
APP_PORT=4100
QUEUE_BACKEND=bullmq            # bullmq | polling
WORKER_DRAIN_TIMEOUT_MS=120000
NEXT_PUBLIC_API_URL=http://localhost:4100

# Features
<YOUR_TOKEN_SECRET>=             # falls back to <YOUR_AUTH_SECRET>
SMTP_HOST=localhost
SMTP_PORT=1025
SMTP_FROM=noreply@taskflow.local

# Production only
SITE_URL=https://app.taskflow.io
SENTRY_DSN=
```

## Testing Notes

- Backend tests: Node.js built-in `node:test` — `describe` / `it` / `mock` — NOT Jest
- Frontend tests: Vitest + JSDOM + `@testing-library/react`
- Backend test files: `tests/**/*.test.js`
- Frontend test files: `web/tests/**/*.test.{ts,tsx}`
- Do NOT use `jest`, `expect.assertions()` Jest-style, or `require('jest-mock')`
- Mock Prisma with `mock.fn()` from `node:test`, not `jest.fn()`

## Commit Discipline

After completing any unit of work:

1. Update `CHANGELOG.md` with a concise entry (what + why)
2. Commit immediately

Never batch multiple tasks into one commit.

## Documentation

- `docs/` — feature notes, ADRs, integration guides
- `docs/adr/` — architecture decision records (numbered, immutable)
- Story and planning artifacts: `../_planning/`
```

---

**What makes this CLAUDE.md effective:**

1. **Commands are executable** — copy-paste into terminal, they work
2. **Architecture explains intent** — "Repository methods: findById..." tells the agent what to name a new method without asking
3. **Error pattern is shown with code** — not just "use plain errors", but the exact shape
4. **Auth section has actual credentials** — no "see .env.example" ambiguity
5. **Test runner is explicit** — "NOT Jest" prevents a very common agent mistake
6. **Contract pattern has a full example** — agent knows exactly what to put in a new `contract.js`

The most important principle: **every section answers a question an agent would have to ask**.
If you remove a section and find yourself thinking "the agent would have to guess here" — put it back.
