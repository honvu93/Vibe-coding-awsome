# Code Review Gates

## The Two-Stage Review

Every piece of code goes through TWO reviews, in order:

**Stage 1: Spec Compliance** — Does the code do what was asked?
**Stage 2: Code Quality** — Is the code well-written?

Never reverse this order. If the code does not meet the spec, quality review is wasted effort — the code will change.

### Stage 1: Spec Compliance

Check against acceptance criteria:

- [ ] Every AC is implemented (not just most of them)
- [ ] No extra features added (scope creep)
- [ ] Tests exist for each AC
- [ ] Tests actually verify the behavior (not just exist)
- [ ] Edge cases from the spec are handled

### Stage 2: Code Quality

Only after Stage 1 passes:

- [ ] No security vulnerabilities (OWASP Top 10)
- [ ] Error handling follows project conventions
- [ ] No race conditions or data loss scenarios
- [ ] Test coverage adequate (especially edge cases)
- [ ] No unnecessary complexity
- [ ] Follows project patterns (not reinventing existing utilities)
- [ ] No dead code, unused imports, or commented-out code

## Severity Levels

Classify every finding with one of three levels.

### Blockers (Must Fix Before Merge)

- Security vulnerabilities
- Data loss scenarios
- Race conditions
- Broken contracts (API returns wrong shape)
- Missing error handling for likely failures

### Suggestions (Should Fix)

- Missing input validation
- Unclear variable/function names
- Missing tests for edge cases
- Performance issues (N+1 queries, unnecessary loops)
- Code duplication

### Nits (Nice to Have)

- Style inconsistencies (only if not auto-formatted)
- Minor naming improvements
- Documentation gaps
- Alternative approaches that might be cleaner

## Communication Patterns

Good review feedback:

- Start with a summary (overall impression, key concerns, what is good)
- Pair every problem with a suggested solution
- Ask clarifying questions instead of assuming the code is wrong
- Use consistent severity markers

Bad review feedback:

- Just listing problems with no solutions
- "This is wrong" without explaining why
- Nitpicking style in a file with real bugs
- Ignoring the good parts entirely

## When to Push Back

The reviewer is not always right. Push back when:

- The suggestion adds unnecessary complexity
- The "improvement" would break the existing project pattern
- The reviewer misunderstands the requirements
- The fix is outside the scope of the current task

Always push back with reasoning, not just disagreement.

## Review Frequency

| Context | Frequency |
|---|---|
| Subagent-driven dev | After EACH task |
| Normal development | Every 3 tasks or before merge |
| Hot fixes | Before AND after deployment |
| Security changes | Always — no exceptions |

## Anti-Patterns

- **Skipping review "because it's simple"** — Simple code has bugs too.
- **Batch review at the end** — Reviewing everything at once produces shallow reviews.
- **Ignoring blocker findings** — One unresolved blocker invalidates the merge.
- **Self-review** — You cannot objectively review code you just wrote.
- **Reviewing after deploy** — The review gate exists before merge, not after.

## The Mindset

A code review is not an audit. It is a collaboration. The goal is not to find fault — it is to ship correct, maintainable code. A reviewer who finds nothing is not doing the job. A reviewer who finds only faults is not doing it well either.

The best reviews leave the author with a clear understanding of what to fix and confidence that the rest is solid.
