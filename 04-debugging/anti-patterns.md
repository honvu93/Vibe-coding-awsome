# Debugging Anti-Patterns

## The AI Debugging Doom Loop

The most common failure mode when debugging with AI assistance:

```
Error occurs
  → Ask AI to fix it
    → AI suggests a plausible fix
      → Apply fix
        → New error appears
          → Ask AI to fix THAT
            → AI suggests another fix
              → Apply fix
                → Original error returns (or a third error appears)
                  → "Let me just rewrite this whole thing"
```

Each iteration moves you FURTHER from the root cause. The AI is not
investigating — it is pattern-matching to fixes that look reasonable.
Reasonable-looking fixes are not the same as correct fixes.

Break the loop by stopping, documenting what you know, and following
the Systematic Debugging Protocol before touching any more code.

---

## Anti-Pattern Catalog

### 1. Shotgun Debugging

**What it looks like.**
Changing multiple things at once, hoping one of them fixes it. "Let me try
updating the dependency AND changing the config AND refactoring that function."

**Why it fails.**
If it works, you do not know which change fixed it — so you carry dead weight
and cannot explain the fix. If it does not work, you have introduced more
variables, making the next investigation harder.

**The fix.**
One change at a time. One hypothesis at a time. If you have three theories,
test them in sequence — not simultaneously.

---

### 2. Cargo Cult Fixes

**What it looks like.**
Copying a fix from Stack Overflow, AI output, or a blog post without
understanding why it works. "This answer had 847 upvotes, I'll just apply it."

**Why it fails.**
The fix addresses a specific root cause. Your root cause may be completely
different. The fix might appear to work (masking your real problem) or
create a new subtle bug.

**The fix.**
Before applying any external fix, answer: "Why does this fix work? What
specific mechanism does it address? Does that mechanism match my situation?"
If you cannot answer that, keep reading until you can.

---

### 3. Print Statement Archaeology

**What it looks like.**
Adding 30 logging statements across 8 files to "find the bug." Running the
code and then spending 45 minutes trying to read the output.

**Why it fails.**
Too much noise, not targeted enough. You are collecting data without a
hypothesis, which means you do not know what you are looking for.

**The fix.**
Form a hypothesis first. Then add ONE diagnostic at the specific location
your hypothesis predicts the failure. Read the output. If it does not
confirm the hypothesis, MOVE the diagnostic — do not ADD another one.
Target, do not flood.

---

### 4. The "Works on My Machine" Dismissal

**What it looks like.**
Bug does not reproduce locally, conclusion: "must be a deployment issue" or
"not reproducible, closing."

**Why it fails.**
Environment differences are real bugs. The fact that it works locally means
the bug is environment-sensitive — which is important information, not a
reason to dismiss the report.

**The fix.**
Document the environment where it fails: OS, runtime version, environment
variables, database state, network conditions, loaded data. Then either
reproduce in that environment or find the exact environmental difference
that triggers the bug.

---

### 5. Fix the Test, Not the Code

**What it looks like.**
A test fails after your change. You update the test assertion to match
the new (incorrect) output and move on.

**Why it fails.**
The test was correct. It was documenting the expected behavior. You just
deleted your own safety net.

**The fix.**
When a test you did not intend to change fails after your change, your
change broke something. The test is the signal. Investigate the change,
do not silence the signal.

Exception: you intentionally changed the behavior the test covers. In that
case, update the test — but first, make sure the new behavior is correct.

---

### 6. Scope Creep Debugging

**What it looks like.**
Started debugging a login error. Noticed inconsistent variable naming in
the same file. Started renaming variables. Broke something. Now debugging
the renaming. Forgot about the login error.

**Why it fails.**
You have abandoned the active investigation for a side quest. You now have
two problems instead of one, and neither is fully understood.

**The fix.**
When you notice something unrelated worth fixing, write it down as a
separate task. Close the mental tab. Return to the original bug. Fix
improvements separately after the bug is resolved and committed.

---

### 7. The Infinite Retry

**What it looks like.**
Same approach, slightly different each time.
- "Maybe if I change this parameter from 5 to 10..."
- "Maybe if I move this line up two positions..."
- "Maybe if I use a different method name..."

**Why it fails.**
If the approach is wrong, variations of it are also wrong. You are not
making progress; you are sampling the wrong solution space.

**The fix.**
The 3-attempt rule: if you have tried the same strategy three times with
minor variations and it has not worked, the strategy is wrong. Stop
completely. Go back to Phase 1. Your mental model of the problem is
incorrect and needs to be rebuilt, not refined.

---

### 8. Blaming the Framework

**What it looks like.**
"This is a React/Django/Rails/[framework] bug, not my code."
"The ORM is generating the wrong query."
"The library has a race condition."

**Why it fails.**
In practice, it is almost never the framework. Frameworks are tested by
thousands of developers. It is almost always your usage of the framework —
a misunderstood API, an unexpected behavior you did not read the docs for,
or a version incompatibility in your configuration.

**The fix.**
Find the framework's own documentation or test suite for the feature you
are using. Build the minimal reproduction case. If the minimal case works
and your code does not, the bug is in how your code uses the framework.
Only escalate to "framework bug" after you have a minimal reproduction
that does not involve any of your application logic.

---

### 9. The Time Pressure Fix

**What it looks like.**
"We need to ship this today. Just make it stop throwing errors."
`try { ... } catch (e) { /* TODO: investigate */ }`

**Why it fails.**
The suppressed error is still happening. The root cause is still present.
The "temporary" fix becomes permanent within two weeks. The real bug
surfaces during a demo, a customer escalation, or a production incident —
at the worst possible time.

**The fix.**
A correct fix that takes 2 hours costs less total time than:
a wrong fix (30 min) + debugging the wrong fix (2 hours) + the correct
fix anyway (2 hours) + any damage the wrong fix caused.

If the timeline genuinely cannot accommodate a correct fix, make a
conscious, documented decision to ship a known workaround — with a
tracked issue, not a TODO comment.

---

### 10. Not Reading the Error Message

**What it looks like.**
Error appears. Developer glances at the first line, pastes it into AI chat,
asks "how do I fix this?"

**Why it fails.**
The error message usually tells you what is wrong. The stack trace usually
tells you where. The error code in the documentation tells you why. Most
bugs are solved by actually reading the complete output.

**The fix.**

Read the full error. Completely. Then:

1. Google the exact error message (in quotes)
2. Read the official documentation for that error code
3. Read the full stack trace line by line — find YOUR code in the trace
4. THEN investigate

If you cannot understand the error message, that is what you ask AI to
explain — not "fix this."

---

## How to Break the Doom Loop

When you recognize that you are in the loop (you will feel it —
the same error appearing in new forms, mounting frustration, "one more try"):

1. **Stop.** Do not apply one more fix.

2. **Revert** all changes you made since the last known-working state.
   (`git stash` or `git checkout -- .`)

3. **Document** what you have tried and what happened.
   Not what you expected to happen. What actually happened.

4. **Re-read** the original error message from scratch.
   Pretend you have never seen it before.

5. **Find the boundary** — which specific function call, query, or request
   is producing the wrong output?

6. **Form a hypothesis** and verify it WITHOUT changing production code.

7. Only then: apply a single, targeted fix.

---

## The 3-Attempt Rule

If you have tried 3 different fixes and none have worked, these things are true:

- Your mental model of the problem is wrong
- The bug is not located where you think it is
- Additional fix attempts will not solve it

What to do instead:

| Action | How |
|---|---|
| Change your information source | Read the documentation, not AI output |
| Change your vantage point | Add diagnostics at a completely different layer |
| Change the question | Instead of "how do I fix X" ask "where is the actual failure?" |
| Get fresh eyes | Walk away for 10 minutes, then re-read the error |
| Ask for help | Share your documented investigation — not just the error |

The 3-attempt rule is not a sign of failure. It is a forcing function that
prevents you from spending 4 hours on a problem that needs a 20-minute
rethink.

---

## Quick Reference: Warning Signs You Are In a Bad Pattern

| You are thinking... | The pattern | What to do |
|---|---|---|
| "Let me try changing a few things and see what happens" | Shotgun Debugging | Pick one thing, form a hypothesis first |
| "This fix worked for someone on Stack Overflow" | Cargo Cult Fix | Understand why before applying |
| "Let me add more logging everywhere" | Print Statement Archaeology | Form hypothesis, add ONE targeted log |
| "It works locally, must be their environment" | "Works on My Machine" | Find the exact environmental difference |
| "I'll update the test to match what it does now" | Fix the Test | The test is right; your code is wrong |
| "While I'm here, let me also fix this naming" | Scope Creep | Note it, finish the bug, do it separately |
| "Maybe if I just tweak this value..." | Infinite Retry | 3 attempts = wrong strategy, start over |
| "This must be a framework bug" | Blaming the Framework | Minimal reproduction case first |
| "Ship it, we'll investigate later" | Time Pressure Fix | Document it as a known issue with a real ticket |
| "I'll just paste this error into chat" | Not Reading the Error | Read the full error message yourself first |
