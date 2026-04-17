# Systematic Debugging Protocol

## The Iron Law

```
NO FIXES WITHOUT ROOT CAUSE INVESTIGATION FIRST
```

The single most common mistake in AI-assisted debugging: asking the AI to "fix this error"
without understanding WHY the error happens. The AI generates a plausible fix, which masks
the real problem, which surfaces later in a harder-to-debug form.

---

## The Four Phases

### Phase 1: Investigate

Before touching ANY code.

**Read the error completely.**

- Full stack trace, not just the first line
- Error code, not just error message
- Context: what was happening when it failed?
- What environment: local dev, CI, staging, production?

**Reproduce consistently.**

- Can you trigger the error reliably?
- What is the minimum reproduction case?
- Does it happen in tests, dev, prod, or all three?
- If it is intermittent, what conditions make it more likely?

**Check recent changes.**

```bash
git log --oneline -10          # what changed recently?
git diff HEAD~3                # did a recent commit introduce this?
git log --oneline -- path/to/affected/file
```

- Was a dependency updated?
- Was a configuration changed?
- Did a schema or API contract change?

**Gather evidence at component boundaries.**

In any multi-component system, the failure is at a boundary:
function call, network request, database query, file read, external API.

- Add one diagnostic log at each suspected boundary
- Which component is producing the bad output?
- Is the INPUT to the failing component correct?
- Is the OUTPUT from the failing component what you expect?

**Trace data flow — work backward from the error.**

```
Error at line 47 in processOrder()
  ← called by checkout() with what arguments?
    ← what did the UI send to checkout()?
      ← where does that data come from?
```

Start at the error site. Work backward. Find where the bad data first appears.
That is your root cause location.

---

### Phase 2: Analyze

**Find a working reference.**

Is there similar code in the codebase that works? A different endpoint, a different
entity, a different flow that does something structurally similar?

Compare the working version with the broken one — completely, line by line.
What is different? Not what looks different at a glance. What IS different.

**Check the obvious culprits first.**

These account for roughly 80% of bugs:

| Category | What to check |
|---|---|
| Types | Is the right type being passed? String vs number, array vs object, null vs empty array |
| Async | Is every Promise being awaited? Is async/await used consistently? |
| Null/undefined | Is something unexpectedly null or missing? |
| Imports | Is the right module being imported? Named vs default export? |
| Scope | Is a variable accessible where it is being used? |
| Mutation | Is data being mutated in place when it should not be? |
| Paths | Are file paths, URLs, and environment variable names correct? |
| Casing | camelCase vs snake_case — especially across API boundaries |

**Map dependencies.**

- What does the broken code depend on?
- Are those dependencies in the state you expect?
- Has an upstream dependency changed its interface or behavior?
- Is a dependency version locked or could it have silently upgraded?

---

### Phase 3: Hypothesize

**Form ONE specific hypothesis.**

Vague: "Something is wrong with authentication."

Specific: "The session token is being JSON-serialized before being stored in the
cookie, which adds extra quote characters, which causes the HMAC signature
verification to fail on the next request."

The hypothesis must be specific enough to test with a single targeted check.
If you cannot describe it that precisely, keep investigating — you do not have
a hypothesis yet, you have a guess.

**Verify the hypothesis BEFORE applying a fix.**

Add a single diagnostic log, assertion, or unit test that will confirm or
disprove your hypothesis without changing behavior:

```javascript
// Hypothesis: the value is undefined at this point
console.log('[DEBUG] value before transform:', JSON.stringify(value));
```

Run it. If the log does not match your hypothesis, your hypothesis is wrong.
Go back to Phase 2. You have NOT wasted time — you have eliminated a wrong theory.

If the log confirms your hypothesis, you now understand the root cause.
Remove the diagnostic, write a test, fix the code.

---

### Phase 4: Implement

**Write a failing test FIRST.**

Before changing production code, write a test that fails because of the bug:

```
1. Write test that exercises the broken behavior
2. Run it — verify it fails
3. Now fix the production code
4. Run the test — it should pass
5. Run the full test suite — nothing else should break
```

This test is your regression guard. It will catch this exact bug forever.

**Apply a single, targeted fix.**

Fix ONLY what the hypothesis identified. Do not:
- "Also clean up" nearby code while you are in there
- Fix multiple bugs in the same commit
- Refactor the code that happens to be near the bug

Each of those is a separate task. Do them separately. You cannot isolate what
fixed the problem if you changed five things at once.

**Verify end-to-end.**

- The failing test now passes
- The full test suite passes
- The original reproduction case is resolved
- No new errors appear in logs

**Count your attempts.**

If you are on your third fix attempt and still failing, STOP.
You are fixing the wrong thing. Go back to Phase 1 with fresh eyes.

---

## The Escalation Rule

After 3 failed fix attempts on the same problem:

1. Document what you have tried and what happened (not what you expected)
2. Question your core assumption — is the bug where you think it is?
3. Consider: is this a design flaw being expressed as a bug?
4. Re-read the original error message from scratch, as if you have never seen it
5. Walk away for 10 minutes — you are too close to it
6. If still stuck: ask for help with your documented investigation attached

---

## Worked Example

**The scenario.** A function `getUserById(id)` is returning `null` for a user
that definitely exists in the database.

**Phase 1: Investigate.**

Full error: no exception — the function just returns null. Recent changes: the
user repository was refactored yesterday to use a new query builder. The bug
only appears in production, not locally.

Reproduction: call `getUserById('usr_12345')` — returns null in production,
returns the correct user locally.

**Phase 2: Analyze.**

Find a working reference: `getOrderById()` was refactored at the same time and
works correctly. Compare the two functions. Difference found: `getUserById` uses
`WHERE id = $1` but the new query builder automatically wraps string IDs with
quotes, producing `WHERE id = '"usr_12345"'` — the quotes are part of the value.

**First hypothesis (wrong):** The database index on the id column is corrupt.
Disprove: run the query manually in psql — it returns the user. Index is fine.

**Revised hypothesis (correct):** The query builder double-encodes the id
parameter when the value is already a quoted string. The WHERE clause is
matching against `'"usr_12345"'` (with embedded quotes) instead of `'usr_12345'`.

**Verify:** Add diagnostic log showing the final SQL before execution.
Confirmed: the query contains the double-encoded value.

**Phase 4: Fix.**

Write a test: call `getUserById` with a known ID, assert it returns the user
(not null). Run it — it fails.

Fix: strip surrounding quotes from the id parameter before passing to the query
builder. Run the test — it passes. Run the full suite — passes. Deploy to
staging — confirmed working.

---

## Debugging with AI — Special Considerations

The AI will always suggest something. That is not the same as the AI being right.

**What to ask instead of "fix this error":**

| Instead of | Ask |
|---|---|
| "Fix this error" | "Help me investigate why this error occurs" |
| "Why isn't this working?" | "Here is the full stack trace, the relevant code, and my hypothesis — is my hypothesis correct?" |
| "Try another approach" | "My last 2 attempts failed. Help me step back and re-examine the root cause." |

**What context to provide:**

- Full error message and stack trace (not a screenshot — paste the text)
- The last 3-5 git commits
- The relevant code section AND the code that calls it
- What you have already tried and what happened

**Verify AI hypotheses independently before applying fixes.**

If the AI says "the problem is X," add a diagnostic to confirm X is actually
happening before changing any code. If the diagnostic does not confirm it,
tell the AI — do not just try the fix anyway.

If an AI fix does not work, do not ask it to "try again." Go back to Phase 1.
The AI's mental model of your problem is also wrong, and it will keep generating
variations of the same wrong fix.

---

## Quick Reference Card

```
PHASE 1 — INVESTIGATE
  [ ] Read the full error (stack trace + error code + context)
  [ ] Reproduce it consistently
  [ ] Check recent git changes
  [ ] Add diagnostics at component boundaries
  [ ] Trace data flow backward from the error

PHASE 2 — ANALYZE
  [ ] Find a working reference to compare against
  [ ] Check: types, async, null, imports, paths, casing
  [ ] Map dependencies — has anything upstream changed?

PHASE 3 — HYPOTHESIZE
  [ ] Form ONE specific hypothesis
  [ ] Verify it with a diagnostic BEFORE fixing anything
  [ ] If disproved, go back to Phase 2

PHASE 4 — IMPLEMENT
  [ ] Write a failing test first
  [ ] Apply a single targeted fix
  [ ] Run all tests
  [ ] Verify the original reproduction case is resolved

ESCALATE after 3 failed attempts — your model of the problem is wrong
```
