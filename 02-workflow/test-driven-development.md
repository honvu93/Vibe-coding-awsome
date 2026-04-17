# Test-Driven Development with AI

## The Iron Law

```
NO PRODUCTION CODE WITHOUT A FAILING TEST FIRST
```

This is not a suggestion. It is not a guideline for complex cases. It applies to every feature, every function, every bug fix. The moment you compromise on this, the AI will generate fast, plausible, untested code — and you will spend three days debugging it.

## Why TDD Matters MORE with AI

Without AI, writing code without tests is merely undisciplined. With AI, it is actively dangerous. Here is why:

**AI writes tests that confirm what it wrote, not what it should have written.**

If you ask an AI to implement a function and then write tests for it, the tests will pass. Trivially. The AI knows what the function does — it just wrote it — so it writes tests that match that implementation. Those tests are not specifications. They are receipts.

TDD inverts this. The test is written first, before the implementation exists. The test is a precise, executable specification of behavior. The AI must satisfy the test, not invent a behavior and then describe it.

**The test IS the spec.** When you write a test that says "given a negative amount, throw an error with code INVALID_AMOUNT", you have made a decision. When you write the test after the fact, the AI makes that decision and you ratify it. This difference sounds small. It compounds into every design choice across your codebase.

## The Red-Green-Refactor Cycle

### RED — Write a failing test

Write the minimal test that demonstrates the desired behavior. One behavior per test. The test name describes the behavior in plain language.

Rules for writing the test:

- Test observable behavior, not implementation details (what the function does, not how)
- Use concrete values, not "any valid input"
- One assertion per test when possible — multiple when they form a single logical claim
- The test must be runnable immediately after writing (no missing imports, no syntax errors)

### VERIFY RED (mandatory — never skip)

Run the test. Confirm two things:

1. The test fails
2. The failure message is the one you expect

This step catches setup errors before you waste implementation time. Common problems it surfaces:

- The function already exists and already handles this case (your work is done)
- The test is testing the wrong thing (it passes even though the feature doesn't exist)
- A typo in an import path is producing an import error, not a test failure
- The test framework is not configured correctly

If the test fails for a reason you did not expect, fix the test. Do not proceed until the failure is the expected failure.

### GREEN — Write minimal code to pass

Write the simplest implementation that makes the test pass. Not the elegant implementation. Not the complete implementation. The minimum.

Rules for GREEN:

- If hardcoding the return value makes the test pass, hardcode it. The next test will force you to generalize.
- No extra features. No "while I'm here" additions. No speculative code.
- No YAGNI violations. You are not going to need it until a test requires it.
- Stop the moment the test goes green.

### VERIFY GREEN (mandatory)

Run the full test suite, not just the new test. Confirm:

1. The new test passes
2. All previously passing tests still pass
3. No new warnings that indicate a problem

If existing tests break, your implementation has a side effect you didn't intend. Fix it before moving to refactor.

### REFACTOR — Only after GREEN

Now clean up. With tests green, you have a safety net. You can rename, extract, reorganize — and the tests will catch regressions immediately.

Rules for refactor:

- Tests must stay green throughout. Run them after every change.
- Remove duplication — but only duplication that actually exists, not duplication you anticipate
- Improve names if they are genuinely unclear
- Extract helpers only when extraction makes the code easier to reason about, not to hit some arbitrary function-length limit
- No new behavior during refactor. If you want to add behavior, write a new test first.

## Multi-Language Examples

### JavaScript / TypeScript — Node.js built-in test runner

```javascript
// tests/cart-service.test.js
import { describe, it, before } from 'node:test'
import assert from 'node:assert/strict'
import { CartService } from '../src/domain/cart/service.js'

describe('CartService.addItem', () => {
  // RED: run this test before implementing addItem
  it('throws INVALID_QUANTITY when quantity is zero', () => {
    const cart = new CartService()
    assert.throws(
      () => cart.addItem({ productId: 'prod-1', quantity: 0 }),
      (err) => {
        assert.equal(err.code, 'INVALID_QUANTITY')
        return true
      }
    )
  })

  it('adds item to empty cart', () => {
    const cart = new CartService()
    cart.addItem({ productId: 'prod-1', quantity: 2 })
    assert.equal(cart.items().length, 1)
    assert.equal(cart.items()[0].quantity, 2)
  })

  it('increments quantity when same product added twice', () => {
    const cart = new CartService()
    cart.addItem({ productId: 'prod-1', quantity: 2 })
    cart.addItem({ productId: 'prod-1', quantity: 3 })
    assert.equal(cart.items()[0].quantity, 5)
  })
})
```

```bash
# Run tests — fails RED before implementation exists
node --test tests/cart-service.test.js

# Expected failure output:
# ✗ throws INVALID_QUANTITY when quantity is zero
#   ReferenceError: CartService is not defined
```

```javascript
// src/domain/cart/service.js — GREEN implementation
export class CartService {
  #items = new Map()

  addItem({ productId, quantity }) {
    if (quantity <= 0) {
      throw Object.assign(new Error('Quantity must be greater than zero'), {
        code: 'INVALID_QUANTITY',
      })
    }
    const existing = this.#items.get(productId) ?? 0
    this.#items.set(productId, existing + quantity)
  }

  items() {
    return Array.from(this.#items.entries()).map(([productId, quantity]) => ({
      productId,
      quantity,
    }))
  }
}
```

### JavaScript / TypeScript — Vitest

```typescript
// web/tests/price-formatter.test.ts
import { describe, it, expect } from 'vitest'
import { formatPrice } from '@/lib/price-formatter'

describe('formatPrice', () => {
  it('formats integer cents as dollars with two decimal places', () => {
    expect(formatPrice(1000)).toBe('$10.00')
  })

  it('formats zero as $0.00', () => {
    expect(formatPrice(0)).toBe('$0.00')
  })

  it('throws when given a negative value', () => {
    expect(() => formatPrice(-1)).toThrow('NEGATIVE_AMOUNT')
  })

  it('formats large amounts with comma separators', () => {
    expect(formatPrice(1000000)).toBe('$10,000.00')
  })
})
```

```typescript
// src/lib/price-formatter.ts — GREEN implementation
export function formatPrice(cents: number): string {
  if (cents < 0) {
    throw Object.assign(new Error('Amount cannot be negative'), { code: 'NEGATIVE_AMOUNT' })
  }
  return new Intl.NumberFormat('en-US', {
    style: 'currency',
    currency: 'USD',
  }).format(cents / 100)
}
```

### Python — pytest

```python
# tests/test_rate_limiter.py
import pytest
from src.rate_limiter import SlidingWindowLimiter

class TestSlidingWindowLimiter:
    # RED: run before implementing SlidingWindowLimiter

    def test_allows_requests_below_limit(self):
        limiter = SlidingWindowLimiter(limit=5, window_seconds=60)
        for _ in range(5):
            assert limiter.is_allowed("192.168.1.1") is True

    def test_blocks_request_exceeding_limit(self):
        limiter = SlidingWindowLimiter(limit=5, window_seconds=60)
        for _ in range(5):
            limiter.is_allowed("192.168.1.1")
        assert limiter.is_allowed("192.168.1.1") is False

    def test_tracks_keys_independently(self):
        limiter = SlidingWindowLimiter(limit=1, window_seconds=60)
        assert limiter.is_allowed("192.168.1.1") is True
        assert limiter.is_allowed("192.168.1.2") is True  # different key

    def test_resets_after_window_expires(self):
        import time
        limiter = SlidingWindowLimiter(limit=1, window_seconds=0.1)
        limiter.is_allowed("192.168.1.1")
        time.sleep(0.15)
        assert limiter.is_allowed("192.168.1.1") is True
```

```python
# src/rate_limiter.py — GREEN implementation
import time
from dataclasses import dataclass, field
from typing import Dict

@dataclass
class _Bucket:
    count: int = 0
    reset_at: float = 0.0

class SlidingWindowLimiter:
    def __init__(self, limit: int, window_seconds: float):
        self._limit = limit
        self._window = window_seconds
        self._buckets: Dict[str, _Bucket] = {}

    def is_allowed(self, key: str) -> bool:
        now = time.monotonic()
        bucket = self._buckets.get(key, _Bucket(reset_at=now + self._window))
        if now >= bucket.reset_at:
            bucket = _Bucket(reset_at=now + self._window)
        if bucket.count >= self._limit:
            self._buckets[key] = bucket
            return False
        bucket.count += 1
        self._buckets[key] = bucket
        return True
```

```bash
# Run tests
pytest tests/test_rate_limiter.py -v

# RED output (before implementation):
# FAILED tests/test_rate_limiter.py::TestSlidingWindowLimiter::test_allows_requests_below_limit
# ImportError: cannot import name 'SlidingWindowLimiter' from 'src.rate_limiter'
```

### Go — built-in testing package

```go
// domain/invoice/service_test.go
package invoice_test

import (
    "testing"
    "github.com/example/app/domain/invoice"
)

// RED: run before implementing CalculateTax
func TestCalculateTax_StandardRate(t *testing.T) {
    svc := invoice.NewService()
    tax, err := svc.CalculateTax(10000, "US-CA")
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
    // California base rate: 7.25% → 725 cents on $100.00
    if tax != 725 {
        t.Errorf("expected 725, got %d", tax)
    }
}

func TestCalculateTax_UnknownJurisdiction(t *testing.T) {
    svc := invoice.NewService()
    _, err := svc.CalculateTax(10000, "XX-ZZ")
    if err == nil {
        t.Fatal("expected error for unknown jurisdiction, got nil")
    }
}

func TestCalculateTax_ZeroAmount(t *testing.T) {
    svc := invoice.NewService()
    tax, err := svc.CalculateTax(0, "US-CA")
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
    if tax != 0 {
        t.Errorf("expected 0, got %d", tax)
    }
}
```

```go
// domain/invoice/service.go — GREEN implementation
package invoice

import "fmt"

var taxRates = map[string]float64{
    "US-CA": 0.0725,
    "US-NY": 0.08,
    "US-TX": 0.0625,
}

type Service struct{}

func NewService() *Service { return &Service{} }

func (s *Service) CalculateTax(amountCents int, jurisdiction string) (int, error) {
    rate, ok := taxRates[jurisdiction]
    if !ok {
        return 0, fmt.Errorf("unknown jurisdiction: %s", jurisdiction)
    }
    return int(float64(amountCents) * rate), nil
}
```

```bash
# Run tests
go test ./domain/invoice/... -v

# RED output (before implementation):
# ./domain/invoice/service_test.go:8:19: undefined: invoice.NewService
```

### Rust — built-in test attribute

```rust
// src/lib.rs
pub mod rate_limiter;

// src/rate_limiter.rs

// Tests written FIRST — RED phase
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn allows_requests_below_limit() {
        let mut limiter = RateLimiter::new(5, 60);
        for _ in 0..5 {
            assert!(limiter.is_allowed("client-1"));
        }
    }

    #[test]
    fn blocks_request_exceeding_limit() {
        let mut limiter = RateLimiter::new(5, 60);
        for _ in 0..5 {
            limiter.is_allowed("client-1");
        }
        assert!(!limiter.is_allowed("client-1"));
    }

    #[test]
    fn tracks_clients_independently() {
        let mut limiter = RateLimiter::new(1, 60);
        assert!(limiter.is_allowed("client-1"));
        assert!(limiter.is_allowed("client-2")); // different key
    }
}

// GREEN implementation — added after tests fail
use std::collections::HashMap;
use std::time::{Duration, Instant};

struct Bucket {
    count: u32,
    reset_at: Instant,
}

pub struct RateLimiter {
    limit: u32,
    window: Duration,
    buckets: HashMap<String, Bucket>,
}

impl RateLimiter {
    pub fn new(limit: u32, window_seconds: u64) -> Self {
        RateLimiter {
            limit,
            window: Duration::from_secs(window_seconds),
            buckets: HashMap::new(),
        }
    }

    pub fn is_allowed(&mut self, key: &str) -> bool {
        let now = Instant::now();
        let bucket = self.buckets.entry(key.to_string()).or_insert_with(|| Bucket {
            count: 0,
            reset_at: now + self.window,
        });
        if now >= bucket.reset_at {
            bucket.count = 0;
            bucket.reset_at = now + self.window;
        }
        if bucket.count >= self.limit {
            return false;
        }
        bucket.count += 1;
        true
    }
}
```

```bash
# Run tests
cargo test

# RED output (before GREEN implementation added):
# error[E0422]: cannot find struct, variant or union type `RateLimiter` in module `super`
```

## Anti-Patterns

**Writing code first, tests after ("I'll just verify with tests")**
You are not verifying. You are documenting. The AI writes tests that match the implementation it just wrote. Those tests will pass. They prove nothing.

**Test passes immediately without any implementation**
Your test is not testing the thing you think it is testing. Either the behavior already exists (check — maybe your work is done), or the test setup is wrong (the import resolves to something unexpected, the mock is too broad, the condition is always true). A test that has never been red is not a test.

**Keeping a "reference" implementation while writing tests ("I'll adapt it")**
This is the same as writing tests after. You already know what the code does. You are writing tests to confirm it, not to specify it. Delete the reference implementation. Write the test. Then write the implementation from scratch.

**"This is too simple to test"**
The functions that seem too simple to test are the ones that have subtle edge cases you haven't thought of. What happens with an empty string? A null input? An integer where a string is expected? A negative number? Write the tests. You will discover the edge cases.

**Skipping Verify RED**
You write the test, it looks right, you start implementing. Three minutes later, the test passes. You're happy. But the test was always going to pass — the import path was wrong and the test framework silently skipped it. Verify RED catches this in seconds.

**A GREEN step that implements ten things at once**
Your test covers one behavior. If you implement ten behaviors to make it pass, you have untested behaviors. Those untested behaviors will eventually fail in production. Write one test, implement the minimum to pass it, write the next test.

**Refactoring during GREEN**
You are implementing to pass a test. You notice the existing code is messy. You refactor it. Now the test fails for a different reason and you don't know if it's your new code or your refactor. Separate the phases. GREEN first. Then REFACTOR. Never both at once.

## When AI Writes Tests

When you ask an AI to write tests for you, apply these checks before accepting them:

**Do the assertions verify behavior or echo implementation?**
A test that says `assert(result.id === someGeneratedId)` when `someGeneratedId` was also generated by the code under test is verifying nothing. The assertion should be about properties you can reason about independently: `assert(typeof result.id === 'string')`, `assert(result.id.length === 36)`.

**Does the happy path have unhappy path coverage?**
AI reliably covers the happy path and reliably skips error cases, boundary conditions, and invalid inputs. For every test the AI writes, ask: what happens if the input is null? Empty? Too large? The wrong type?

**Can the test actually fail?**
Temporarily break the code. Change a return value, delete a branch, invert a condition. Run the test. If it still passes, the test is not testing the right thing.

**Are the test names descriptive enough to diagnose failures?**
"test_add_item" tells you nothing when it fails at 2am. "addItem throws INVALID_QUANTITY when quantity is negative" tells you exactly where to look.

## When to Skip TDD

TDD applies to production code that will run in your system. It does not apply to:

- **Throwaway prototypes** you have committed in advance to deleting (not "I might throw this away" — "I will definitely delete this")
- **One-off scripts** that run exactly once and are not part of any repeatable process
- **Configuration files** — JSON, YAML, TOML with no logic
- **Pure layout markup** with no conditional logic, no event handlers, no data transformation

Everything else — including "simple" utilities, "obvious" helpers, and "one-liner" functions — follows TDD. The discipline is uniform, not selective. Selective TDD means TDD only when it is convenient, which means never when the code is hard, which is precisely when you need it most.
