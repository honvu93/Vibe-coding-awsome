# Subagent-Driven Development

## The Concept

Instead of one long conversation implementing everything, dispatch a fresh subagent for each task. Each agent gets clean context, focused instructions, and produces a specific deliverable.

This is a different mental model from "chat with the AI until the feature is done." Each subagent is more like a contractor: briefed once, works independently, delivers a discrete output, then is reviewed before the next task begins.

---

## Why Fresh Agents Per Task?

- **Context isolation** — Agent #5 does not carry the confusion from Agent #1's failed attempt. Stale context is one of the main sources of AI-generated bugs.
- **Focused work** — Each agent has one job, not "also fix that other thing I mentioned earlier."
- **Parallel potential** — Independent tasks can run simultaneously across agents. See `parallel-dispatch.md`.
- **Easier review** — Each agent's output is a discrete, reviewable unit. You can reject task 3 without affecting task 2.
- **Predictable cost** — Each task has a bounded token budget. A single long conversation grows unboundedly.

---

## The Workflow

### Step 1: Prepare the Task

From your plan, extract for each task:
- Exact files to create or modify (list them explicitly)
- The implementation to produce (from the plan — do not summarize, copy the relevant section)
- The test to write first (TDD: test before production code)
- Acceptance criteria (what "done" means, mechanically)
- Constraints (what the agent must NOT touch)

Why copy from the plan instead of referencing it? The subagent cannot see your prior conversation or your plan file. It only knows what you put in the prompt. Any reference to "the plan we discussed" will produce guesswork.

### Step 2: Dispatch the Agent

Provide in the prompt:
- Full task description (self-contained, no references to other conversations)
- Relevant file contents (paste them — do not say "look at the file")
- Project conventions (from CLAUDE.md or equivalent)
- Expected deliverable format (modified file, diff, function only, etc.)

### Step 3: Two-Stage Review

After the agent returns output, review in two stages:

**Stage 1: Spec Compliance**
- Does the output match the acceptance criteria?
- Are all files created or modified as specified?
- Do the tests exist and test the correct behavior?
- Does the implementation actually do what the spec required?

**Stage 2: Code Quality** (only after Stage 1 passes)
- Are there security vulnerabilities?
- Does it follow project conventions (naming, error handling, patterns)?
- Is there unnecessary complexity or dead code?
- Are there missing edge cases not covered by the tests?

Why two stages? If Stage 1 fails, the code will change anyway. Running a thorough code quality review on code you are about to rewrite is wasted effort. Fix spec issues first, then evaluate quality.

### Step 4: Handle Agent Status

Agents should signal their completion status. Common statuses:

- **DONE** — Output is complete, proceed to review
- **DONE_WITH_CONCERNS** — Output is complete but the agent flagged something — read the concern before proceeding, it may affect correctness
- **NEEDS_CONTEXT** — Agent could not complete without missing information — stop, provide the context, re-dispatch
- **BLOCKED** — Agent hit a fundamental issue — assess root cause, adjust the plan before retrying

Never push past a NEEDS_CONTEXT or BLOCKED status by asking the agent to "do its best." The agent will hallucinate a solution that compiles but is wrong.

---

## Model Tier Selection

Not every task needs the most capable (most expensive) model. Matching model tier to task complexity is a meaningful cost and quality optimization.

| Task Complexity | Model Tier | Examples |
|----------------|------------|---------|
| Mechanical — isolated, clear spec, no design decisions | Fast/cheap | Rename function, add field validation, write simple unit test |
| Integration — multi-file changes, requires coordination | Standard | New API endpoint with tests, refactor a module's interface |
| Architecture — design decisions, ambiguous requirements, security review | Most capable | New domain design, complex refactoring, vulnerability analysis |

Use the most capable model for tasks that require judgment. Use cheaper models for tasks that require execution of a clear spec.

---

## Red Flags (Never Do These)

- Skip reviews (spec OR quality) — review gates exist because agents are confidently wrong at a non-trivial rate
- Proceed with unfixed spec issues — if the spec is wrong, quality review is irrelevant
- Dispatch multiple agents to modify the same file — one will overwrite the other
- Reference a plan file in the prompt instead of pasting the content — the agent cannot read it
- Start code quality review before spec compliance passes — the code will change, making the review moot
- Ignore agent escalations — NEEDS_CONTEXT means the task is blocked, not that the agent should guess

---

## Anti-Patterns

**"Just do the whole feature"**
Too broad. The agent loses focus around the third file, starts making assumptions, and produces something that technically compiles but fails half the acceptance criteria. Break it into tasks.

**"Fix the race condition"** (without context)
The agent guesses which race condition, in which file, under which conditions. It will produce a fix for the wrong problem. Always provide the specific file, the specific call site, and the observed failure behavior.

**No constraints specified**
The agent may helpfully refactor unrelated code "while I'm in here." This introduces unreviewed changes outside the task scope. Always specify what the agent must not touch.

**Trusting agent success claims**
An agent that says "tests pass" or "this is complete" is not always correct. Always run the tests yourself. Always open the files and verify the output matches the spec. Trust but verify — but actually verify.

---

## The Single Most Important Rule

Write the test first. Every task that produces production code must start with a failing test. If the agent skips this, reject the output and ask for the test first. The test is not optional, not "I'll add it later," not "it's obvious what it does." The test comes first. Always.

This rule is enforced at the process layer (Layer 2 in the three-layer system), not left to agent discretion. See `three-layer-system.md`.
