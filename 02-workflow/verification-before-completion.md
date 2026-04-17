# Verification Before Completion

## The Iron Law

```
NO COMPLETION CLAIMS WITHOUT FRESH VERIFICATION EVIDENCE
```

This is the most violated rule in AI-assisted development. The AI says "done!" — but did it actually verify? Usually not. This guide teaches you to demand evidence, not assertions.

---

## The Problem

AI assistants are optimized to be helpful, which means they are biased toward claiming success. Common failure modes:

- "The tests should pass now" (did you run them?)
- "I've fixed the bug" (did you verify it's actually fixed?)
- "Everything looks correct" (based on... reading the code?)

Words like **should**, **probably**, **seems to**, **looks like** are RED FLAGS. They signal that the AI is reasoning from code inspection, not from execution output. These are not the same thing.

The difference between "I read the code and it looks right" and "I ran the code and it returned the expected output" is the difference between guessing and knowing.

---

## The Gate Function

Every completion claim must pass through five steps before it is valid.

### Step 1: IDENTIFY

What command or action would **prove** this claim?

- Tests pass → what is the exact test command and expected count?
- Bug is fixed → what is the exact reproduction step?
- Feature works → what is the verification scenario and expected output?

If you cannot identify a concrete verification command, the work is not ready to verify. Go back and define acceptance criteria first.

### Step 2: RUN

Execute the verification command **fresh**:

- Not from cache
- Not from a previous run
- Not partial — run it to completion

If the AI says "I already ran it," ask it to run it again. Output decays. A passing test suite from 30 minutes ago does not confirm the code you just changed is correct.

### Step 3: READ

Read the **full output**:

- Check the exit code (0 = success, anything else = failure)
- Count test results explicitly (34/34 pass, not just "pass")
- Look for warnings, skipped tests, and deprecation notices
- Check that the number of tests matches what you expect — a suite that should have 40 tests but only ran 34 has a problem

### Step 4: VERIFY

Does the output actually confirm the claim?

- "34/34 pass" confirms "all tests pass" ✅
- "33/34 pass, 1 skipped" does NOT confirm "all tests pass" ❌
- Zero errors but a deprecation warning might still indicate a real problem ⚠️
- "Build succeeded" with 200 warnings is different from a clean build ⚠️

Read the output critically. Summaries can lie. Counts don't.

### Step 5: CLAIM

Only now can you make the claim, and the claim must cite the evidence:

- "All tests pass: ran `npm test` → 34/34 pass, exit code 0" ✅
- "Bug fixed: reproduced with input X, applied fix, same input now returns expected Y" ✅
- "Build succeeds: ran `npm run build` → compiled successfully in 4.2s, exit code 0" ✅

A claim without evidence is an assertion. Evidence is the output, not your confidence.

---

## Verification Patterns by Task Type

### Tests

```
GOOD: [Run: npm test] [Output: 34/34 pass, 0 fail, 0 skip, exit code 0] → "All tests pass"
BAD:  "Should pass now" / "I believe the tests will pass" / "Tests look correct"
```

Run the full suite, not just the affected file. A change in one module can break another.

### Bug Fixes

```
GOOD: Reproduce bug → Apply fix → Verify fix → Write regression test
      → Verify test fails WITHOUT fix (red) → Verify test passes WITH fix (green)
BAD:  "I've applied the fix, it should work now"
```

A bug fix without a regression test means the bug is one refactor away from coming back.

### Builds

```
GOOD: [Run: npm run build] [Output: compiled successfully, exit code 0] → "Build succeeds"
BAD:  "The code compiles" (did you actually invoke the compiler?)
```

Syntax checking in an editor is not the same as building. Run the actual build.

### Linting

```
GOOD: [Run: npm run lint] [Output: 0 errors, 0 warnings, exit code 0] → "No lint issues"
BAD:  "The code follows the style guide" (based on visual inspection)
```

### Regression Tests

The correct verification flow for a regression test is:

1. Write the test
2. Run it **without** the fix — it must **fail** (this proves the test actually catches the bug)
3. Apply the fix
4. Run the test again — it must **pass** (this proves the fix resolves the bug)

Skipping step 2 means the test might pass even if the fix is wrong or the fix was already in place. The red-green cycle is not optional.

```
GOOD: Reverted fix → Test fails (RED) → Restored fix → Test passes (GREEN)
BAD:  "I've written a regression test" (without confirming the red phase)
```

---

## Language Red Flags

Train yourself to stop and challenge these phrases whenever you see them in AI output:

| Red Flag | What It Actually Means |
|---|---|
| "should" | Not verified |
| "probably" | Not verified |
| "seems to" | Not verified |
| "I believe" | Not verified |
| "looks correct" | Read the code, did not run it |
| "appears to work" | Did not test |
| "this will fix" | Has not confirmed |
| "you can now" | Has not demonstrated |

When you see these phrases, respond with: "Run it and show me the output."

---

## The Anti-Pattern Cascade

Here is what happens when verification is skipped:

1. AI claims "done" without running tests
2. You commit and push, trusting the claim
3. CI fails on code that "should have worked"
4. Now you are debugging under pressure, context already lost
5. The rushed fix introduces a second bug
6. Repeat

Here is what happens with proper verification:

1. AI runs tests — one fails
2. AI investigates the failure (evidence-based, not guessing)
3. AI applies fix, re-runs tests — all pass
4. You commit with a green suite in hand
5. CI passes
6. Done

The extra 60 seconds of running verification commands saves hours of CI debugging and context reconstruction.

---

## Checklist Template

Before marking **any** work as complete, go through this list:

- [ ] All tests run fresh and pass (count: X/X, exit code: 0)
- [ ] No lint errors (command run, exit code: 0)
- [ ] Build succeeds if applicable (command run, exit code: 0)
- [ ] Every acceptance criterion verified against actual output, not reading the code
- [ ] Regression test written for any bug fix, red-green cycle confirmed
- [ ] No new warnings introduced
- [ ] CHANGELOG or equivalent updated

Do not mark the story or task as done until every checked item has evidence attached to it.

---

## Applying This to AI Sessions

When your AI coding assistant says something is done:

1. Ask: "Run the tests and paste the full output"
2. Check: Did it actually run them, or did it describe what it would run?
3. Read: Full output, not just the assistant's summary of the output
4. Count: Numbers must match (if you added two tests, the count should be higher)
5. Verify: Exit code must be 0

If the assistant gives you a summary instead of output, push back: "Show me the raw output, not a description of it."

---

## When to Be Extra Careful

These are the highest-risk moments — verification is mandatory, no exceptions:

- Before committing
- Before pushing to a shared branch
- Before creating a pull request
- Before marking a story or task as done
- After any refactoring (behavior must be identical)
- After merging branches (integration can break things that worked separately)
- After upgrading dependencies

---

## When Verification Is Lighter

Some work does not require the full gate:

- Exploratory conversations where no code was changed
- Reading code to understand it
- Planning and design discussions (no implementation yet)
- Documentation-only changes (but still check that links render and formatting is valid)

Even for documentation, a quick build or preview check is worth doing. Broken links and rendering errors are real bugs.

---

## The Core Habit

The single most important habit to build in AI-assisted development is this: **never accept a claim without asking for the evidence that proves it**.

This is not about distrusting the AI. It is about acknowledging that reading code and running code are fundamentally different activities, and only one of them tells you what actually happens at runtime.

Demand output. Read it yourself. Count the numbers. Check the exit code.

That is what "done" means.
