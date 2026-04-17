# Example: A Refactoring Session

This is a realistic refactoring session: extracting the `payments` domain from a growing `orders`
god module. It shows how to plan, execute atomically, and verify that nothing broke — all guided
by tests rather than guesswork.

---

## Setup: The Problem

The `orders` module started as a clean 300-line file. Eight months and 14 features later it is
2,247 lines and handles four distinct concerns:

- Order creation and status management
- Payment processing and refunds
- Shipping calculation and tracking
- Email and push notifications

The problem is not just aesthetics. The module has become a liability:
- A bug fix in payment logic requires understanding order state machine logic to avoid regression
- Payment provider integration tests run as part of the order test suite (slow)
- Two engineers cannot work on orders and payments in parallel without constant merge conflicts

**Goal**: Extract payment processing into its own `payments` domain with a clean interface,
leaving `orders` responsible only for order lifecycle.

---

## Step 1: Discovery — Dependency Mapping

Before moving a single line, understand what the module touches and what touches it.

### 1.1 — Find all callers of payment functions in orders

```bash
# What functions in orders relate to payment?
grep -n "payment\|charge\|refund\|stripe\|invoice" src/domain/orders/service.js
```

```
41:  async createOrder(input) {
78:    const charge = await this._processPayment(input.paymentMethodId, total);  // line 78
119: async _processPayment(paymentMethodId, amount) {
120:   const stripe = this._getStripeClient();
156: async refundOrder(orderId, reason) {
189:   const refund = await this._issueStripeRefund(charge.stripeChargeId, amount);
221: async _issueStripeRefund(chargeId, amount) {
```

```bash
# Who calls these payment methods from outside the domain?
grep -rn "orders.*refund\|orders.*processPayment\|ordersService\.charge" \
  web/src/app/api/ src/domain/
```

```
web/src/app/api/orders/[id]/refund/route.js:14:  await ordersService.refundOrder(params.id, body.reason);
web/src/app/api/orders/route.js:31:  const order = await ordersService.createOrder(body);
src/domain/subscriptions/service.js:67:  await ordersService.createOrder({ ... });
```

### 1.2 — Count coupling before extraction

| Metric | Value |
|---|---|
| `orders/service.js` lines | 2,247 |
| Functions in orders that are payment-related | 6 |
| External callers of payment functions via orders | 3 routes, 1 domain |
| Direct Stripe API calls in orders | 8 |
| Tests for payment logic in orders.test.js | 23 (of 61 total) |

### 1.3 — Map the target file structure

**Current:**
```
src/domain/orders/
  service.js      (2,247 lines — everything)
  repository.js   (340 lines)
  access.js       (89 lines)
  contract.js     (112 lines)
```

**Target:**
```
src/domain/orders/
  service.js      (~1,100 lines — order lifecycle only)
  repository.js   (340 lines — unchanged)
  access.js       (89 lines — unchanged)
  contract.js     (112 lines — unchanged)

src/domain/payments/
  service.js      (~280 lines — charge, refund, retrieve)
  repository.js   (~150 lines — PaymentRecord CRUD)
  access.js       (~60 lines — payment permission checks)
  contract.js     (~80 lines — payment DTOs and error builders)
```

---

## Step 2: Planning — 3 Atomic Commits

The extraction is split into three commits so that at no point does the test suite break.
Each commit leaves the system in a valid, deployable state.

| Commit | Action | Risk |
|---|---|---|
| 1 | Create payments domain (new files, no changes to orders) | None — additive only |
| 2 | Move payment logic into payments, update orders to delegate | Medium — orders behavior changes |
| 3 | Update API routes and external callers to use payments domain | Low — callers change, contract unchanged |

---

## Step 3: Execution

### Commit 1 — Create the payments domain

**Principle**: Add files only. Touch nothing that exists.

**`src/domain/payments/contract.js`**

```js
/**
 * Payment domain contracts.
 * DTOs and structured error builders for the payments domain.
 */

export function normalizePaymentRecord(row) {
  return {
    id: row.id,
    orderId: row.orderId,
    amount: row.amount,
    currency: row.currency,
    status: row.status,
    stripeChargeId: row.stripeChargeId ?? null,
    stripeRefundId: row.stripeRefundId ?? null,
    createdAt: row.createdAt.toISOString(),
    updatedAt: row.updatedAt.toISOString(),
  };
}

export const errors = {
  notFound: (id) =>
    Object.assign(new Error(`Payment record ${id} not found`), { code: 'PAYMENT_NOT_FOUND', id }),

  chargeFailed: (reason) =>
    Object.assign(new Error(`Charge failed: ${reason}`), { code: 'PAYMENT_CHARGE_FAILED', reason }),

  refundFailed: (reason) =>
    Object.assign(new Error(`Refund failed: ${reason}`), { code: 'PAYMENT_REFUND_FAILED', reason }),

  alreadyRefunded: (paymentId) =>
    Object.assign(new Error(`Payment ${paymentId} already refunded`), {
      code: 'PAYMENT_ALREADY_REFUNDED',
      paymentId,
    }),
};
```

**`src/domain/payments/repository.js`**

```js
import { normalizePaymentRecord } from './contract.js';

export class PaymentsRepository {
  constructor(prisma) {
    this.prisma = prisma;
  }

  async findById(id) {
    const row = await this.prisma.paymentRecord.findUnique({ where: { id } });
    return row ? normalizePaymentRecord(row) : null;
  }

  async findByOrderId(orderId) {
    const rows = await this.prisma.paymentRecord.findMany({
      where: { orderId },
      orderBy: { createdAt: 'desc' },
    });
    return rows.map(normalizePaymentRecord);
  }

  async create(data) {
    const row = await this.prisma.paymentRecord.create({ data });
    return normalizePaymentRecord(row);
  }

  async update(id, data) {
    const row = await this.prisma.paymentRecord.update({ where: { id }, data });
    return normalizePaymentRecord(row);
  }
}
```

**`src/domain/payments/access.js`**

```js
/**
 * Permission checks for the payments domain.
 * A user may view a payment only if they own the associated order or are an admin.
 */

export function assertCanViewPayment(actor, payment) {
  if (actor.role === 'admin') return;
  if (actor.id === payment.userId) return;
  const err = new Error('Forbidden');
  err.code = 'PAYMENT_FORBIDDEN';
  throw err;
}

export function assertCanRefund(actor) {
  if (actor.role === 'admin') return;
  const err = new Error('Only admins may issue refunds');
  err.code = 'PAYMENT_REFUND_FORBIDDEN';
  throw err;
}
```

**`src/domain/payments/service.js`**

```js
/**
 * Payments domain service.
 *
 * Responsible for: charging, refunding, and retrieving payment records.
 * NOT responsible for: order status transitions, shipping, notifications.
 *
 * Depends on: Stripe client (injected), PaymentsRepository (injected).
 * Zero imports from the orders domain — dependency direction is one-way.
 */

import { errors } from './contract.js';

export class PaymentsService {
  constructor(repository, stripeClient) {
    this.repository = repository;
    this.stripe = stripeClient;
  }

  /**
   * Charge a payment method and persist the record.
   * @returns {Promise<PaymentRecord>}
   */
  async charge({ orderId, userId, paymentMethodId, amount, currency = 'usd' }) {
    let stripeCharge;

    try {
      stripeCharge = await this.stripe.paymentIntents.create({
        amount: Math.round(amount * 100), // Stripe uses cents
        currency,
        payment_method: paymentMethodId,
        confirm: true,
        automatic_payment_methods: { enabled: true, allow_redirects: 'never' },
      });
    } catch (stripeErr) {
      throw errors.chargeFailed(stripeErr.message);
    }

    return this.repository.create({
      orderId,
      userId,
      amount,
      currency,
      status: 'CHARGED',
      stripeChargeId: stripeCharge.id,
    });
  }

  /**
   * Issue a full refund for a payment record.
   * @returns {Promise<PaymentRecord>}
   */
  async refund(paymentId) {
    const record = await this.repository.findById(paymentId);
    if (!record) throw errors.notFound(paymentId);
    if (record.stripeRefundId) throw errors.alreadyRefunded(paymentId);

    let stripeRefund;
    try {
      stripeRefund = await this.stripe.refunds.create({
        charge: record.stripeChargeId,
      });
    } catch (stripeErr) {
      throw errors.refundFailed(stripeErr.message);
    }

    return this.repository.update(paymentId, {
      status: 'REFUNDED',
      stripeRefundId: stripeRefund.id,
    });
  }

  async getPaymentsForOrder(orderId) {
    return this.repository.findByOrderId(orderId);
  }
}
```

**Tests for the new domain:** `tests/payments-service.test.js`

```js
import { describe, it, mock, beforeEach } from 'node:test';
import assert from 'node:assert/strict';
import { PaymentsService } from '../src/domain/payments/service.js';

describe('PaymentsService', () => {
  let repo, stripe, service;

  beforeEach(() => {
    repo = {
      findById: mock.fn(),
      findByOrderId: mock.fn(),
      create: mock.fn(),
      update: mock.fn(),
    };

    stripe = {
      paymentIntents: {
        create: mock.fn(async () => ({ id: 'pi_test_001', status: 'succeeded' })),
      },
      refunds: {
        create: mock.fn(async () => ({ id: 're_test_001' })),
      },
    };

    service = new PaymentsService(repo, stripe);
  });

  describe('charge()', () => {
    it('creates a payment record on successful charge', async () => {
      repo.create.mock.mockImplementation(async (data) => ({
        id: 'pay_001', ...data, createdAt: new Date().toISOString(), updatedAt: new Date().toISOString(),
      }));

      const result = await service.charge({
        orderId: 'ord_001', userId: 'usr_001',
        paymentMethodId: 'pm_test', amount: 99.99,
      });

      assert.equal(stripe.paymentIntents.create.mock.calls.length, 1);
      assert.equal(repo.create.mock.calls.length, 1);
      assert.equal(result.status, 'CHARGED');
      assert.equal(result.orderId, 'ord_001');
    });

    it('throws PAYMENT_CHARGE_FAILED if Stripe rejects', async () => {
      stripe.paymentIntents.create.mock.mockImplementation(async () => {
        throw new Error('Your card was declined.');
      });

      await assert.rejects(
        () => service.charge({ orderId: 'ord_001', userId: 'usr_001', paymentMethodId: 'pm_bad', amount: 50 }),
        (err) => {
          assert.equal(err.code, 'PAYMENT_CHARGE_FAILED');
          return true;
        },
      );
    });
  });

  describe('refund()', () => {
    it('issues a refund and updates the record', async () => {
      repo.findById.mock.mockImplementation(async () => ({
        id: 'pay_001', stripeChargeId: 'ch_001', stripeRefundId: null, status: 'CHARGED',
      }));
      repo.update.mock.mockImplementation(async (id, data) => ({
        id, ...data, createdAt: new Date().toISOString(), updatedAt: new Date().toISOString(),
      }));

      const result = await service.refund('pay_001');

      assert.equal(result.status, 'REFUNDED');
      assert.equal(result.stripeRefundId, 're_test_001');
    });

    it('throws PAYMENT_ALREADY_REFUNDED if already refunded', async () => {
      repo.findById.mock.mockImplementation(async () => ({
        id: 'pay_001', stripeRefundId: 're_existing', status: 'REFUNDED',
      }));

      await assert.rejects(
        () => service.refund('pay_001'),
        (err) => { assert.equal(err.code, 'PAYMENT_ALREADY_REFUNDED'); return true; },
      );
    });
  });
});
```

```bash
node --test tests/payments-service.test.js
# All tests pass — new domain works in isolation

npm test
# All existing tests still pass — nothing was changed
```

**CHANGELOG entry for Commit 1:**
```markdown
## [feat] payments: create payments domain (service, repository, access, contract)

Introduces a new `payments` domain at `src/domain/payments/` containing `PaymentsService`,
`PaymentsRepository`, access guards, and contract DTOs. Payment logic is not yet wired —
this commit is additive only. Unit tests for `PaymentsService` are included and passing.
```

**Commit**: `feat(payments): create payments domain — service, repository, access, contract`

---

### Commit 2 — Move payment logic, update orders to delegate

**Principle**: Delete from orders, delegate to payments. The external contract of `ordersService`
must not change yet — callers still call `ordersService.refundOrder()` and it still works.

**Modify `src/domain/orders/service.js`**:

Remove these 6 methods entirely:
- `_processPayment`
- `_issueStripeRefund`
- `_getStripeClient`
- And the 3 inline Stripe calls scattered through `createOrder` and `refundOrder`

Replace with delegation:

```js
// src/domain/orders/service.js (excerpt — showing the delegation pattern)

export class OrdersService {
  // paymentsService is now injected
  constructor(repository, paymentsService, notificationsService) {
    this.repository = repository;
    this.payments = paymentsService;       // <-- injected dependency
    this.notifications = notificationsService;
  }

  async createOrder(input) {
    // ... validate, calculate total, etc. (unchanged) ...

    // BEFORE: inline Stripe call (~35 lines)
    // AFTER: delegate to payments domain
    const payment = await this.payments.charge({
      orderId: order.id,
      userId: input.userId,
      paymentMethodId: input.paymentMethodId,
      amount: total,
    });

    await this.repository.update(order.id, { paymentId: payment.id, status: 'CONFIRMED' });
    await this.notifications.sendOrderConfirmation(order.id);

    return this.repository.findById(order.id);
  }

  async refundOrder(orderId, reason) {
    const order = await this.repository.findById(orderId);
    if (!order) throw errors.notFound(orderId);

    // BEFORE: inline Stripe refund call (~40 lines)
    // AFTER: delegate to payments domain
    await this.payments.refund(order.paymentId);

    await this.repository.update(orderId, { status: 'REFUNDED', refundReason: reason });
    await this.notifications.sendRefundConfirmation(orderId);

    return this.repository.findById(orderId);
  }
}
```

**Wire up the dependency in the composition root** `src/server.js`:

```js
// Before:
import { OrdersService } from './domain/orders/service.js';
const ordersService = new OrdersService(ordersRepo, notificationsService);

// After:
import { OrdersService } from './domain/orders/service.js';
import { PaymentsService } from './domain/payments/service.js';
import { PaymentsRepository } from './domain/payments/repository.js';
import { createStripeClient } from './infrastructure/stripe/client.js';

const stripeClient = createStripeClient(process.env.STRIPE_SECRET_KEY);
const paymentsRepo = new PaymentsRepository(prisma);
const paymentsService = new PaymentsService(paymentsRepo, stripeClient);
const ordersService = new OrdersService(ordersRepo, paymentsService, notificationsService);
```

```bash
# Run full test suite
npm test
# Expected: all tests pass (orders tests still pass because external interface unchanged)
```

**Before/after line count:**

| File | Before | After |
|---|---|---|
| `orders/service.js` | 2,247 lines | 1,089 lines |
| `payments/service.js` | 0 lines | 283 lines |
| Net removed from orders | — | 1,158 lines |

**CHANGELOG entry for Commit 2:**
```markdown
## [refactor] orders: delegate payment logic to payments domain

Removes all Stripe API calls and payment record management from `OrdersService`. Payment
processing is now delegated to the injected `PaymentsService`. External API of `OrdersService`
is unchanged — `createOrder()` and `refundOrder()` have the same signatures and return shapes.
`orders/service.js` reduced from 2,247 to 1,089 lines.
```

**Commit**: `refactor(orders): delegate payment logic to payments domain`

---

### Commit 3 — Update API routes to use payments domain directly

Now that the payments domain exists and is wired, update callers that were using orders as a
pass-through for payment operations to call payments directly.

**`web/src/app/api/orders/[id]/refund/route.js`** — unchanged. The `refundOrder` call on
`ordersService` is semantically correct (refunding an order) and now correctly delegates
internally. No change needed here.

**`web/src/app/api/orders/[id]/payments/route.js`** — NEW file to expose payment history:

```js
// This route did not exist before. It's only possible now because payments is its own domain.

import { NextResponse } from 'next/server';
import { getServerSession } from '@/lib/auth/session';
import { paymentsService } from '@/lib/services';

export async function GET(request, { params }) {
  const session = await getServerSession();
  if (!session) return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });

  const payments = await paymentsService.getPaymentsForOrder(params.id);

  return NextResponse.json({ data: payments });
}
```

**`src/domain/subscriptions/service.js`** — update subscription renewal to use the payments
domain directly instead of going through orders:

```js
// Before:
await ordersService.createOrder({ userId, items, paymentMethodId, ... });

// After (direct charge for subscription renewal — no order object needed):
await paymentsService.charge({
  orderId: renewal.orderId,
  userId,
  paymentMethodId: subscription.defaultPaymentMethodId,
  amount: plan.price,
});
```

```bash
npm test
npm run web:build
# Expected: all pass
```

**Before/after coupling metrics:**

| Metric | Before | After |
|---|---|---|
| `orders/service.js` lines | 2,247 | 1,089 |
| Stripe API calls in orders | 8 | 0 |
| External callers routing payment ops through orders | 3 | 0 |
| Payment-specific tests in orders test suite | 23 | 0 |
| Dedicated payment tests in payments test suite | 0 | 31 |
| Domains that import Stripe client | 1 (orders) | 1 (payments) |

**CHANGELOG entry for Commit 3:**
```markdown
## [refactor] payments: update callers to use payments domain directly

Updates `subscriptions/service.js` to call `PaymentsService.charge()` directly for subscription
renewals rather than routing through `OrdersService`. Adds `GET /api/orders/[id]/payments` route
for payment history (previously not possible without cross-domain queries). All existing tests
pass. Payment logic is now fully contained in `src/domain/payments/`.
```

**Commit**: `refactor(payments): update callers to use payments domain directly`

---

## Step 4: Verification

```bash
# Full test suite — zero regressions
npm test

# Frontend build — no broken imports
npm run web:build

# Check for any remaining Stripe imports outside the payments domain
grep -rn "stripe" src/domain/ | grep -v "src/domain/payments/"
# Expected: no output (all Stripe usage is in payments domain)

# Check for any remaining payment logic in orders
grep -n "stripe\|paymentIntent\|_processPayment\|_issueStripe" \
  src/domain/orders/service.js
# Expected: no output

# Verify import graph — orders must not import payments
grep -n "from.*payments" src/domain/orders/service.js
# Expected: no output (orders receives paymentsService via injection, not import)
```

---

## What Made This Work

**Tests as a safety net, not an afterthought.** The payments service tests (Commit 1) were written
before any code moved. They verified the new domain worked in isolation before orders was touched.
The existing orders tests verified nothing broke during the delegation swap (Commit 2). Without
this coverage, Commit 2 would have required manual testing of every order flow.

**Atomic commits with a deployable state at each step.** If a production incident occurred after
Commit 1, the deployment could be reverted with no harm — Commit 1 was purely additive. If a bug
appeared after Commit 2, the diff is 1,158 lines of deletion from a single file — easy to audit.
Compare this to a single "big bang" refactor commit that touches 14 files: auditing that under
pressure is nearly impossible.

**Dependency injection over direct imports.** `OrdersService` never imports `PaymentsService`.
It receives it as a constructor argument. This is why the coupling metric "domains that import
Stripe client" went from 1 to 1 — it did not increase. The orders domain genuinely no longer
knows Stripe exists.

**The CHANGELOG entries tell a story.** Reading the three CHANGELOG entries in sequence, a
developer can understand the shape of the refactor without reading the code. This matters at
3am when something is broken and you need to understand what changed and why.
