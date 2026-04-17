# Git Worktrees for AI Development

## What Are Worktrees?

A Git worktree lets you check out a branch into a **separate directory** while sharing the same `.git` repository. You get full, independent working trees — each with their own files, index, and HEAD — without needing to clone the repository multiple times.

```
my-project/             ← main worktree (e.g., main branch)
.worktrees/
  feature-auth/         ← second worktree (feature/auth branch)
  fix-payment-bug/      ← third worktree (fix/payment-bug branch)
```

All three directories are the same repository. Changes committed in any worktree immediately appear in `git log` across all of them.

---

## Why Use Worktrees with AI?

When working with AI coding agents, worktrees solve several concrete problems:

**Isolation** — Feature work happens in a separate directory. An AI agent cannot accidentally modify files in `main` while working on a feature branch. No stashed changes, no accidental checkouts.

**Parallel agent work** — Multiple agents can work on independent tasks simultaneously, each in their own worktree. No conflicts, no waiting. Agent A builds the authentication service while Agent B builds the notification system.

**Clean baseline** — Each worktree starts from a known-good state: the branch point. You always know what the AI changed relative to a clean foundation.

**Easy rollback** — If a feature goes wrong, `git worktree remove .worktrees/feature-auth` deletes all that work. The main branch is untouched.

**Reviewer-friendly** — Each worktree maps to one PR. The diff is scoped to one feature. No mixed concerns.

---

## Setup

### Creating a Worktree

```bash
# Create a worktree with a new branch based on current HEAD
git worktree add .worktrees/feature-auth -b feature/auth

# Create a worktree from a specific branch or commit
git worktree add .worktrees/hotfix-payment -b hotfix/payment-null-check origin/main

# Create a worktree for an existing branch (no -b flag)
git worktree add .worktrees/existing-branch existing-branch-name
```

### Listing and Removing Worktrees

```bash
# See all active worktrees
git worktree list

# Output:
# /home/user/my-project         abc1234 [main]
# /home/user/my-project/.worktrees/feature-auth  def5678 [feature/auth]

# Remove a worktree (deletes the directory, not the branch)
git worktree remove .worktrees/feature-auth

# Force remove if worktree has untracked changes
git worktree remove --force .worktrees/feature-auth

# Prune stale worktree metadata (if directory was deleted manually)
git worktree prune
```

### Critical: Add to .gitignore First

If you place worktrees inside the project directory (the common pattern), the worktree directory **must** be in `.gitignore` before you create any worktrees.

```bash
# .gitignore
.worktrees/
```

Do this before creating your first worktree. If you accidentally commit a worktree directory, you will push tens of thousands of files to the repository.

To check that it is ignored before proceeding:

```bash
echo ".worktrees/" >> .gitignore
git status   # .worktrees/ should NOT appear in untracked files
```

### Global Worktrees (Alternative Pattern)

If you prefer not to modify `.gitignore`, place worktrees outside the repository entirely:

```bash
# Worktrees in a sibling directory
git worktree add ../myproject-worktrees/feature-auth -b feature/auth

# Worktrees in a dedicated location
git worktree add ~/.worktrees/myproject-feature-auth -b feature/auth
```

This approach requires no `.gitignore` changes but is harder to navigate.

---

## Worktree Workflow

### Step-by-step

```bash
# 1. Add .worktrees/ to .gitignore (first-time setup)
echo ".worktrees/" >> .gitignore
git add .gitignore && git commit -m "chore: ignore worktrees directory"

# 2. Create worktree for your feature
git worktree add .worktrees/feature-auth -b feature/auth

# 3. Enter the worktree and install dependencies
cd .worktrees/feature-auth
npm install           # or pip install, cargo build, etc.

# 4. Run the full test suite — verify clean baseline
npm test

# 5. If tests fail: investigate BEFORE starting work
#    Never build on a broken foundation
#    Fix the baseline issue in main first, then re-create the worktree

# 6. Do your work (TDD: test first, then implementation)

# 7. When done: commit, push, open PR

# 8. Back in main worktree: delete the worktree
cd ../..
git worktree remove .worktrees/feature-auth
# Optionally delete the branch after merging:
git branch -d feature/auth
```

### Dependency Installation

Each worktree is a fresh checkout. Installed dependencies (`node_modules/`, `.venv/`, `target/`) are **not** shared between worktrees.

```bash
# Node.js
cd .worktrees/feature-auth && npm install

# Python
cd .worktrees/feature-auth && python -m venv .venv && source .venv/bin/activate && pip install -r requirements.txt

# Rust
cd .worktrees/feature-auth && cargo build
```

For large dependency trees, this adds setup time. It is usually worth it for the isolation.

---

## Safety Rules

These rules prevent the most common and most painful mistakes:

1. **ALWAYS add the worktree directory to `.gitignore` before creating any worktrees** — if project-local. Verify with `git status` that the directory does not appear.

2. **ALWAYS run the baseline test suite before starting work in a new worktree.** If tests fail at baseline, you are building on a broken foundation. Stop and fix main first.

3. **NEVER proceed with failing baseline tests without understanding why.** "I'll fix it later" always means "it becomes my problem at 11pm."

4. **NEVER assume what directory you are in.** When working across multiple worktrees, confirm your location with `pwd` and `git branch` before running destructive commands.

5. **NEVER run `git worktree remove --force` on a worktree with uncommitted work you care about.** The changes are gone. Commit or stash first.

---

## Integration with Parallel Agent Dispatch

Worktrees are the foundation for running multiple AI agents in parallel. Each agent gets a dedicated worktree so they cannot interfere with each other.

### Setup for parallel agents

```bash
# Create one worktree per independent task
git worktree add .worktrees/task-01-auth-service -b feat/auth-service
git worktree add .worktrees/task-02-notification-service -b feat/notification-service
git worktree add .worktrees/task-03-audit-logging -b feat/audit-logging
```

Dispatch an agent to each worktree. Give each agent:
- The absolute path to their worktree
- Their specific task specification
- The requirement to run tests before and after

### Merging parallel work

After all agents complete:

```bash
# Review each branch
git log main..feat/auth-service --oneline
git log main..feat/notification-service --oneline

# Merge sequentially or open PRs
git checkout main
git merge feat/auth-service
git merge feat/notification-service
git merge feat/audit-logging

# Run full test suite after integration
npm test

# Clean up worktrees
git worktree remove .worktrees/task-01-auth-service
git worktree remove .worktrees/task-02-notification-service
git worktree remove .worktrees/task-03-audit-logging
```

Conflicts are most common in shared files like configuration, routes index files, and the schema. Plan tasks to minimize overlap on these files.

---

## When to Use Worktrees

| Situation | Use worktrees? |
|---|---|
| Multi-task feature with independent tasks | Yes — one worktree per task |
| Parallel agent dispatch | Yes — mandatory |
| Risky refactoring (easy rollback wanted) | Yes |
| Comparing two implementation approaches | Yes — two worktrees, two strategies |
| Long-running feature (days of work) | Yes — keeps main clean |
| Small single-file change | No — just branch directly |
| Sequential tasks with tight dependencies | No — worktrees don't help here |
| Quick prototype or spike | No — just branch directly |

---

## When NOT to Use Worktrees

**Sequential tasks with dependencies** — If task B cannot start until task A is committed and merged, you do not need separate worktrees. A single feature branch is fine.

**Simple single-file edits** — The setup overhead is not worth it for a two-line fix.

**Prototype exploration** — When you are still figuring out the approach, work on a branch directly. Promote to a worktree workflow once the approach is decided.

**When disk space is constrained** — Each worktree with installed dependencies can be several hundred MB for large projects. Monitor available space.

---

## Troubleshooting

### "fatal: 'feature/auth' is already checked out"

A branch can only be checked out in one worktree at a time.

```bash
# See which worktree has it
git worktree list

# Either work in that worktree, or remove it first
git worktree remove .worktrees/feature-auth
git worktree add .worktrees/feature-auth -b feature/auth-v2
```

### Worktree directory was deleted manually without `git worktree remove`

```bash
# Git still tracks the stale worktree metadata
git worktree prune   # removes stale references
git worktree list    # confirm it is gone
```

### Tests fail in worktree but pass in main

The most likely causes:
1. Missing or outdated dependencies — run `npm install` (or equivalent) in the worktree
2. The worktree branch diverged from an upstream change — `git rebase origin/main`
3. Environment variables not set — the worktree's shell session may not have loaded `.env`

---

## Quick Reference

```bash
# Add worktree directory to .gitignore
echo ".worktrees/" >> .gitignore

# Create worktree with new branch
git worktree add .worktrees/<name> -b <branch>

# Create worktree from specific upstream
git worktree add .worktrees/<name> -b <branch> origin/main

# List all worktrees
git worktree list

# Remove a worktree (directory deleted, branch kept)
git worktree remove .worktrees/<name>

# Clean up stale worktree references
git worktree prune
```
