# Vibe Coding Guide

**Battle-tested best practices for building real products with Claude Code.**

This guide distills 2 months of full-time AI-assisted development into actionable patterns. Every recommendation comes from shipping a production system — not theory. The project involved 34 backend domains, a Next.js frontend, Chrome extension, worker pipelines, and deployment automation — all built primarily through Claude Code conversations.

These practices are **framework-agnostic**. Whether you're building in Python, TypeScript, Rust, Go, or anything else — the workflows, debugging protocols, and quality gates apply universally.

## Who This Is For

- **Beginners**: Start with [Foundation](#01-foundation) and [Workflow](#02-workflow) — you'll immediately write better CLAUDE.md files and stop losing context between sessions
- **Intermediate users**: Jump to [Agents](#03-agents) and [Debugging](#04-debugging) — learn to orchestrate multiple agents and debug systematically instead of randomly
- **Team leads**: Read [Quality](#05-quality) and [Advanced](#06-advanced) — standardize your team's AI-assisted development process

## Philosophy

1. **Structure beats talent.** A mediocre prompt with great workflow beats a great prompt with no workflow.
2. **Evidence over claims.** Never trust "should work" — run the tests, read the output, verify the claim.
3. **Atomic everything.** One task, one commit. One agent, one job. Small, verifiable units compound into reliable systems.
4. **AI is a junior dev with perfect memory.** Give it clear specs, review its work, and enforce process — don't just "let it code."
5. **The CLAUDE.md is your API contract.** It's how you communicate your project's rules to every future Claude session.

## Table of Contents

### 01-Foundation
> Set up correctly from day one. These decisions compound.

- [CLAUDE.md Architecture](01-foundation/claude-md-architecture.md) — The most important file in your project
- [Memory System](01-foundation/memory-system.md) — Persistent learning across sessions
- [Project Structure](01-foundation/project-structure.md) — Organizing your workspace for AI-assisted dev

### 02-Workflow
> The daily combat loop. These are your non-negotiable habits.

- [Planning Before Coding](02-workflow/planning-before-coding.md) — No placeholders, no "TBD"
- [Test-Driven Development](02-workflow/test-driven-development.md) — The Iron Law: no code without a failing test
- [Commit Discipline](02-workflow/commit-discipline.md) — One task = one commit, always
- [Git Worktrees](02-workflow/git-worktrees.md) — Isolated workspaces for parallel features
- [Verification Before Completion](02-workflow/verification-before-completion.md) — Evidence > "should work"

### 03-Agents
> Scale yourself by orchestrating specialized agents.

- [Three-Layer Agent System](03-agents/three-layer-system.md) — Planning + Process + Execution
- [Agent Routing](03-agents/agent-routing.md) — Right agent for the right job
- [Subagent-Driven Development](03-agents/subagent-driven-dev.md) — Fresh context per task, two-stage review
- [Parallel Dispatch](03-agents/parallel-dispatch.md) — Run independent work concurrently

### 04-Debugging
> Stop guessing. Start investigating.

- [Systematic Debugging Protocol](04-debugging/systematic-debugging.md) — 4 phases: Investigate, Analyze, Hypothesize, Implement
- [Common Anti-Patterns](04-debugging/anti-patterns.md) — What NOT to do when things break

### 05-Quality
> Gates that prevent shipping garbage.

- [Code Review Gates](05-quality/code-review-gates.md) — Spec compliance first, then code quality
- [Refactoring Patterns](05-quality/refactoring-patterns.md) — Breaking circular deps, extracting domains
- [Deployment Safety](05-quality/deployment-safety.md) — Health checks, scoped commands, backup discipline

### 06-Advanced
> Level up after you've mastered the basics.

- [Knowledge Graphs](06-advanced/knowledge-graph.md) — Map dependencies before you refactor
- [Multi-Product Workspaces](06-advanced/multi-product-workspace.md) — Multiple projects, one AI workflow
- [Extension & Build Versioning](06-advanced/extension-versioning.md) — Dual builds, snapshot rollback

### Quick Reference

- [Templates](templates/) — Copy-paste starters for CLAUDE.md, plans, stories, agents
- [Examples](examples/) — Sanitized real sessions showing patterns in action
- [Cheatsheet](cheatsheet.md) — Everything on one page

## Quick Start

**Minimum viable setup (5 minutes):**

1. Copy [templates/CLAUDE.md.template](templates/CLAUDE.md.template) to your project root as `CLAUDE.md`
2. Fill in: commands, architecture overview, testing instructions, key conventions
3. Read [Verification Before Completion](02-workflow/verification-before-completion.md) — this alone will 10x your output quality

**Full setup (30 minutes):**

4. Read [Planning Before Coding](02-workflow/planning-before-coding.md) and copy the plan template
5. Set up the [Memory System](01-foundation/memory-system.md)
6. Read [TDD with AI](02-workflow/test-driven-development.md)

**Team adoption:**

7. Read [Three-Layer Agent System](03-agents/three-layer-system.md) for workflow architecture
8. Customize [Agent Routing](03-agents/agent-routing.md) for your tech stack
9. Enforce [Code Review Gates](05-quality/code-review-gates.md) in your process

## Origin

These practices were developed while building [AutoVeoup](https://autoveoup.com) — a video generation platform with:
- 34 backend domain modules (Node.js, Prisma, PostgreSQL)
- Next.js 16 frontend with React 19
- Chrome extension with dual-build system
- BullMQ worker pipelines
- Python video assembly service
- Zero-downtime PM2 deployment

The entire system was built through Claude Code conversations over ~2 months. Every pattern in this guide was born from a real problem, not invented in a vacuum.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

The best contributions come from real experience — if you've discovered a pattern that made your AI-assisted development significantly better, we want to hear about it.

## License

[MIT](LICENSE)
