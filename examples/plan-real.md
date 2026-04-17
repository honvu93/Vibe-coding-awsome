# Example: A Real Implementation Plan

This is a complete implementation plan for adding **rate limiting** to the fictional Taskflow API.
Every task has real code — no pseudocode, no placeholders. This is what a plan looks like when it
is ready to hand to an agent (or yourself) and execute without clarification.

---

```markdown
# Plan: Rate Limiting Middleware — Taskflow API

## Goal

Add per-IP and per-user rate limiting to all `/api/*` routes, with configuration via environment
variables and observable status via the `/health/rate-limit` endpoint.

## File Map

| File | Action | Description |
|---|---|---|
| `tests/rate-limiter.test.js` | CREATE | Unit tests for middleware logic |
| `src/infrastructure/rate-limiter/sliding-window.js` | CREATE | Redis-backed sliding window counter |
| `src/infrastructure/rate-limiter/middleware.js` | CREATE | Express middleware using the counter |
| `src/infrastructure/rate-limiter/config.js` | CREATE | Configuration loader + defaults |
| `src/server.js` | MODIFY | Wire middleware before route handlers |
| `src/infrastructure/http/health-server.js` | MODIFY | Add `/health/rate-limit` endpoint |
| `src/infrastructure/env/schema.js` | MODIFY | Register new env vars with defaults |

---

## Task 1 — Write failing tests for the rate limiter

**Before writing any implementation**, create the test file. Run it and confirm it fails.

### Test file: `tests/rate-limiter.test.js`

```js
import { describe, it, mock, beforeEach, afterEach } from 'node:test';
import assert from 'node:assert/strict';

// We import the module under test — this does not exist yet, so the import will fail.
// That is the expected "red" state.
import { createSlidingWindowCounter } from '../src/infrastructure/rate-limiter/sliding-window.js';
import { createRateLimitMiddleware } from '../src/infrastructure/rate-limiter/middleware.js';

describe('SlidingWindowCounter', () => {
  let redis;

  beforeEach(() => {
    // Mock Redis client — only the methods we call
    const store = new Map();
    redis = {
      eval: mock.fn(async (script, numkeys, key, ...args) => {
        // Simulate the Lua script: increment count, set expiry if new key
        const now = parseInt(args[0]);
        const window = parseInt(args[1]);
        const limit = parseInt(args[2]);

        const windowStart = now - window;
        const entries = store.get(key) ?? [];
        const fresh = entries.filter((ts) => ts > windowStart);
        fresh.push(now);
        store.set(key, fresh);

        return [fresh.length, fresh.length <= limit ? 1 : 0];
      }),
    };
  });

  afterEach(() => {
    mock.restoreAll();
  });

  it('allows requests within the limit', async () => {
    const counter = createSlidingWindowCounter(redis, { windowMs: 60_000, max: 3 });
    const result1 = await counter.increment('ip:192.0.2.1');
    const result2 = await counter.increment('ip:192.0.2.1');
    const result3 = await counter.increment('ip:192.0.2.1');

    assert.equal(result1.allowed, true);
    assert.equal(result2.allowed, true);
    assert.equal(result3.allowed, true);
    assert.equal(result3.count, 3);
  });

  it('blocks the request that exceeds the limit', async () => {
    const counter = createSlidingWindowCounter(redis, { windowMs: 60_000, max: 2 });
    await counter.increment('ip:192.0.2.2');
    await counter.increment('ip:192.0.2.2');
    const result = await counter.increment('ip:192.0.2.2');

    assert.equal(result.allowed, false);
    assert.equal(result.count, 3);
  });

  it('isolates counters by key', async () => {
    const counter = createSlidingWindowCounter(redis, { windowMs: 60_000, max: 1 });
    const a = await counter.increment('ip:192.0.2.10');
    const b = await counter.increment('ip:192.0.2.11');

    assert.equal(a.allowed, true);
    assert.equal(b.allowed, true);
  });
});

describe('createRateLimitMiddleware', () => {
  it('calls next() when request is allowed', async () => {
    const counter = { increment: mock.fn(async () => ({ allowed: true, count: 1 })) };
    const middleware = createRateLimitMiddleware(counter, { keyFn: (req) => `ip:${req.ip}` });

    const req = { ip: '192.0.2.1', headers: {} };
    const res = { setHeader: mock.fn(), status: mock.fn(), json: mock.fn() };
    res.status.mock.mockImplementation(() => res);
    const next = mock.fn();

    await middleware(req, res, next);

    assert.equal(next.mock.calls.length, 1);
    assert.equal(res.status.mock.calls.length, 0);
  });

  it('returns 429 when request is blocked', async () => {
    const counter = { increment: mock.fn(async () => ({ allowed: false, count: 101 })) };
    const middleware = createRateLimitMiddleware(counter, { keyFn: (req) => `ip:${req.ip}` });

    const req = { ip: '192.0.2.1', headers: {} };
    const res = { setHeader: mock.fn(), status: mock.fn(), json: mock.fn() };
    res.status.mock.mockImplementation(() => res);
    const next = mock.fn();

    await middleware(req, res, next);

    assert.equal(next.mock.calls.length, 0);
    assert.equal(res.status.mock.calls[0].arguments[0], 429);
  });

  it('sets RateLimit-* headers on every response', async () => {
    const counter = { increment: mock.fn(async () => ({ allowed: true, count: 5, max: 100 })) };
    const middleware = createRateLimitMiddleware(counter, { keyFn: (req) => `ip:${req.ip}` });

    const req = { ip: '192.0.2.1', headers: {} };
    const res = { setHeader: mock.fn(), status: mock.fn(), json: mock.fn() };
    res.status.mock.mockImplementation(() => res);

    await middleware(req, res, mock.fn());

    const headerCalls = res.setHeader.mock.calls.map((c) => c.arguments[0]);
    assert.ok(headerCalls.includes('RateLimit-Limit'));
    assert.ok(headerCalls.includes('RateLimit-Remaining'));
  });
});
```

### Verify (expect failure):

```bash
node --test tests/rate-limiter.test.js
# Expected: Error [ERR_MODULE_NOT_FOUND]: Cannot find module
#   '.../src/infrastructure/rate-limiter/sliding-window.js'
```

**Commit**: `test(rate-limiter): add failing tests for sliding window counter and middleware`

---

## Task 2 — Implement the rate limiter

### `src/infrastructure/rate-limiter/config.js`

```js
// Configuration with safe defaults — all overridable via environment variables.

export function loadRateLimitConfig() {
  return {
    // Requests per window per IP
    ipMax: parseInt(process.env.RATE_LIMIT_IP_MAX ?? '100', 10),
    ipWindowMs: parseInt(process.env.RATE_LIMIT_IP_WINDOW_MS ?? String(15 * 60 * 1000), 10),

    // Requests per window per authenticated user (higher than IP to allow multi-tab)
    userMax: parseInt(process.env.RATE_LIMIT_USER_MAX ?? '300', 10),
    userWindowMs: parseInt(process.env.RATE_LIMIT_USER_WINDOW_MS ?? String(15 * 60 * 1000), 10),

    // Routes that bypass rate limiting entirely
    skipPaths: (process.env.RATE_LIMIT_SKIP_PATHS ?? '/health,/ready').split(',').map((s) => s.trim()),
  };
}
```

### `src/infrastructure/rate-limiter/sliding-window.js`

```js
/**
 * Redis-backed sliding window rate limiter.
 *
 * Uses a Lua script for atomic check-and-increment. The Lua script runs inside
 * a single Redis round-trip, preventing the TOCTOU race between reading the
 * count and incrementing it.
 */

// Lua script: receives (key, nowMs, windowMs, max)
// Returns [currentCount, allowed(1|0)]
const LUA_SCRIPT = `
local key     = KEYS[1]
local now     = tonumber(ARGV[1])
local window  = tonumber(ARGV[2])
local max     = tonumber(ARGV[3])
local expire  = math.ceil(window / 1000)

redis.call('ZREMRANGEBYSCORE', key, '-inf', now - window)
local count = redis.call('ZADD', key, now, now .. '-' .. math.random())
count = redis.call('ZCARD', key)
redis.call('EXPIRE', key, expire)

if count <= max then
  return {count, 1}
else
  return {count, 0}
end
`;

/**
 * @param {import('ioredis').Redis} redis
 * @param {{ windowMs: number, max: number }} options
 */
export function createSlidingWindowCounter(redis, { windowMs, max }) {
  return {
    /**
     * Increment the counter for `key` and return whether this request is allowed.
     * @param {string} key  — e.g. "ip:192.0.2.1" or "user:usr_abc123"
     * @returns {{ allowed: boolean, count: number, max: number }}
     */
    async increment(key) {
      const now = Date.now();
      const [count, allowedInt] = await redis.eval(
        LUA_SCRIPT,
        1,        // numkeys
        key,      // KEYS[1]
        now,      // ARGV[1]
        windowMs, // ARGV[2]
        max,      // ARGV[3]
      );

      return {
        allowed: allowedInt === 1,
        count: Number(count),
        max,
      };
    },
  };
}
```

### `src/infrastructure/rate-limiter/middleware.js`

```js
/**
 * Express middleware factory for rate limiting.
 *
 * Attaches RateLimit-* response headers (draft-ietf-httpapi-ratelimit-headers)
 * and returns 429 when the counter denies the request.
 */

/**
 * @param {ReturnType<import('./sliding-window.js').createSlidingWindowCounter>} counter
 * @param {{ keyFn: (req: import('express').Request) => string }} options
 */
export function createRateLimitMiddleware(counter, { keyFn }) {
  return async function rateLimitMiddleware(req, res, next) {
    const key = keyFn(req);
    const result = await counter.increment(key);

    res.setHeader('RateLimit-Limit', result.max);
    res.setHeader('RateLimit-Remaining', Math.max(0, result.max - result.count));
    res.setHeader('RateLimit-Policy', `${result.max};w=${counter.windowMs ?? 'unknown'}`);

    if (!result.allowed) {
      res.setHeader('Retry-After', '60');
      return res.status(429).json({
        error: 'Too many requests. Please slow down.',
        code: 'RATE_LIMIT_EXCEEDED',
      });
    }

    next();
  };
}
```

### Verify (expect green):

```bash
node --test tests/rate-limiter.test.js
# Expected: all 6 tests pass
```

**Commit**: `feat(rate-limiter): implement Redis sliding window counter and Express middleware`

---

## Task 3 — Wire into API routes

### Modify `src/server.js`

Find the section where routes are registered and insert rate limiting before it.

**Before:**
```js
// src/server.js (excerpt)
import express from 'express';
import { router as apiRouter } from './routes/api.js';
import { createRedisConnection } from './infrastructure/queue/redis-connection.js';

const app = express();

app.use(express.json());
app.use('/api', apiRouter);
```

**After:**
```js
// src/server.js (excerpt)
import express from 'express';
import { router as apiRouter } from './routes/api.js';
import { createRedisConnection } from './infrastructure/queue/redis-connection.js';
import { createSlidingWindowCounter } from './infrastructure/rate-limiter/sliding-window.js';
import { createRateLimitMiddleware } from './infrastructure/rate-limiter/middleware.js';
import { loadRateLimitConfig } from './infrastructure/rate-limiter/config.js';

const app = express();
const redis = createRedisConnection();
const rateLimitConfig = loadRateLimitConfig();

// IP-based rate limiting — applied to all /api routes except health paths
const ipCounter = createSlidingWindowCounter(redis, {
  windowMs: rateLimitConfig.ipWindowMs,
  max: rateLimitConfig.ipMax,
});

const ipRateLimit = createRateLimitMiddleware(ipCounter, {
  keyFn: (req) => `ip:${req.ip}`,
});

app.use(express.json());

// Skip rate limiting for health check paths
app.use('/api', (req, res, next) => {
  if (rateLimitConfig.skipPaths.some((p) => req.path.startsWith(p))) {
    return next();
  }
  return ipRateLimit(req, res, next);
});

app.use('/api', apiRouter);
```

### Verify:

```bash
# Start server in test mode
NODE_ENV=test RATE_LIMIT_IP_MAX=5 node src/server.js &

# Send 6 requests — 5th should succeed, 6th should return 429
for i in $(seq 1 6); do
  curl -s -o /dev/null -w "%{http_code}\n" http://localhost:4100/api/tasks
done
# Expected: 200 200 200 200 200 429

# Kill test server
kill %1
```

**Commit**: `feat(server): wire IP rate limiting middleware into /api routes`

---

## Task 4 — Configuration and health endpoint

### Register env vars in `src/infrastructure/env/schema.js`

```js
// Add to the existing env schema exports:

export const RATE_LIMIT_IP_MAX = {
  key: 'RATE_LIMIT_IP_MAX',
  default: '100',
  description: 'Max requests per IP per window (default 15 min)',
};

export const RATE_LIMIT_IP_WINDOW_MS = {
  key: 'RATE_LIMIT_IP_WINDOW_MS',
  default: String(15 * 60 * 1000),
  description: 'Rate limit window in milliseconds for IP limiting',
};

export const RATE_LIMIT_USER_MAX = {
  key: 'RATE_LIMIT_USER_MAX',
  default: '300',
  description: 'Max requests per authenticated user per window',
};

export const RATE_LIMIT_USER_WINDOW_MS = {
  key: 'RATE_LIMIT_USER_WINDOW_MS',
  default: String(15 * 60 * 1000),
  description: 'Rate limit window in milliseconds for user limiting',
};

export const RATE_LIMIT_SKIP_PATHS = {
  key: 'RATE_LIMIT_SKIP_PATHS',
  default: '/health,/ready',
  description: 'Comma-separated path prefixes that bypass rate limiting',
};
```

### Add `/health/rate-limit` to `src/infrastructure/http/health-server.js`

```js
// Add this handler alongside existing /health and /ready handlers:

healthApp.get('/health/rate-limit', async (req, res) => {
  const config = loadRateLimitConfig();

  // Smoke-test: can we reach Redis?
  let redisReachable = false;
  try {
    await healthRedis.ping();
    redisReachable = true;
  } catch {
    redisReachable = false;
  }

  const status = redisReachable ? 'ok' : 'degraded';

  res.status(redisReachable ? 200 : 503).json({
    status,
    config: {
      ipMax: config.ipMax,
      ipWindowMs: config.ipWindowMs,
      userMax: config.userMax,
      userWindowMs: config.userWindowMs,
      skipPaths: config.skipPaths,
    },
    redis: redisReachable ? 'connected' : 'unreachable',
    timestamp: new Date().toISOString(),
  });
});
```

### Verify:

```bash
# Start local stack
npm run local:up

# Health check should return config
curl http://localhost:4100/health/rate-limit
# Expected:
# {
#   "status": "ok",
#   "config": { "ipMax": 100, "ipWindowMs": 900000, ... },
#   "redis": "connected",
#   "timestamp": "..."
# }

# Confirm full test suite still passes
npm test
```

**Commit**: `feat(rate-limiter): add env var schema and /health/rate-limit endpoint`

---

## Self-Review Checklist

- [x] Tests were written BEFORE implementation (Task 1 before Task 2)
- [x] Tests cover happy path (allowed), error path (blocked), edge case (isolated keys)
- [x] No placeholder comments — all code is complete
- [x] Error responses use the project's envelope format (`{ error, code }`)
- [x] New env vars have defaults — feature works without setting them
- [x] Health endpoint degrades gracefully if Redis is unreachable (503, not crash)
- [x] Middleware is opt-out (skip list) rather than opt-in — new routes are protected by default
- [x] RateLimit headers follow the IETF draft standard (observable by clients)
- [x] Lua script is atomic — no TOCTOU race between read and increment
- [x] Four commits, one per task — no batching

---

## Verification

After all four tasks are complete:

```bash
# All backend tests pass
npm test

# Health endpoint reports correct config
curl http://localhost:4100/health/rate-limit

# Rate limiting is enforced end-to-end
RATE_LIMIT_IP_MAX=3 npm run api:dev &
for i in $(seq 1 4); do
  status=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:4100/api/tasks)
  echo "Request $i: $status"
done
# Expected: 200 200 200 429

# Frontend build is unaffected
npm run web:build
```

No regressions in existing tests. Rate limiting is active in development and configurable in
production without a code change.
```

---

**What makes this plan work:**

1. **Real code, not pseudocode** — you can copy the test file into `tests/` and run it immediately
2. **Verify commands follow every task** — you know when a task is done without ambiguity
3. **Before/after diffs for modifications** — agent knows exactly what lines to change, not just "modify server.js"
4. **Commit message per task** — commit discipline is built into the plan, not an afterthought
5. **Self-review checklist is filled in** — shows the reasoning, not just checkboxes
6. **Task 1 explicitly fails on first run** — TDD discipline is enforced structurally, not by trust
