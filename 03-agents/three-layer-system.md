# Three-Layer Agent System

## The Concept

AI-assisted development works best with three distinct layers working together, not one monolithic AI doing everything:

**Layer 1: Planning Backbone** — Manages what to build and in what order
**Layer 2: Process Enforcement** — Ensures HOW work is done follows consistent rules
**Layer 3: Specialist Execution** — Domain experts who write the actual code

## Why Three Layers, Not One?

- One agent doing everything loses context, drifts from spec, skips process
- Planning and execution are different thinking modes — mixing them degrades both
- Process enforcement catches when execution drifts (TDD skipped, tests not run)
- Specialists produce better code than generalists in their domain

A single agent given "build the entire feature" will write code, skip tests, forget the spec halfway through, and declare success. Separating planning, enforcement, and execution creates natural checkpoints where drift is caught early.

---

## Layer 1: Planning Backbone

Role: Sprint planning, story management, progress tracking.

What it manages:
- PRD (Product Requirements Document) — what to build and why
- Architecture decisions — how the system is designed
- Epic/story breakdown — work divided into implementable units
- Sprint status — what is done, in progress, blocked

Key artifacts:
- Planning documents (PRD, architecture, UX spec)
- Epic and story files with acceptance criteria
- Sprint status tracking

Named agent personas (optional but effective):
- Product Manager — requirements and prioritization
- Architect — technical design decisions
- Scrum Master — sprint planning and status tracking

Why named personas? Each persona has a different mandate. When the Architect writes a story, she is constrained to design decisions. When the Scrum Master reviews status, he is constrained to delivery tracking. Named personas prevent scope creep within the planning layer.

---

## Layer 2: Process Enforcement (Superpowers)

Role: Ensure every task follows consistent quality process.

Skills loaded as needed:
- Writing plans — enforced before implementation starts
- TDD — enforced during implementation
- Systematic debugging — enforced when bugs appear
- Verification before completion — enforced before any "done" claim
- Code review — enforced before merge
- Git worktrees — enforced for feature isolation

How it works:
- Skills are markdown files describing a repeatable process
- Each skill has trigger conditions (when to activate)
- Skills are referenced in a central WORKFLOW.md
- The AI reads the skill file and follows the protocol step by step

Why markdown files instead of agent memory:
- Skills are versioned in git — every change is auditable
- Skills can be shared across projects without copying configuration
- Skills are readable by humans — any team member can inspect and improve them
- Skills can be evolved through normal pull request review

Example skill trigger table:

| Situation | Skill file |
|---|---|
| Break story into tasks | writing-plans |
| Start implementation | test-driven-development |
| Bug investigation | systematic-debugging |
| Pre-merge | requesting-code-review, verification-before-completion |
| Branch done | finishing-a-development-branch |

**The TDD Iron Law: Never write production code without a failing test first.** This is enforced at the process layer, not left to individual agent judgment.

---

## Layer 3: Specialist Agents

Role: Domain experts who execute specific types of work.

Common agent profiles:

| Agent | Domain | Typical Tasks |
|-------|--------|--------------|
| Backend Architect | Server-side logic | Domain services, APIs, data models |
| Frontend Developer | UI/UX | Components, routes, styling |
| Database Optimizer | Data layer | Schema design, queries, migrations |
| Security Engineer | Security | Auth, encryption, vulnerability review |
| Code Reviewer | Quality | Pre-merge review, standards enforcement |
| DevOps Automator | Infrastructure | Deploy scripts, CI/CD, monitoring |
| API Tester | Testing | Unit tests, integration tests, E2E |

Agent profiles include:
- Role description and areas of expertise
- Key principles and non-negotiable rules
- Performance standards (response time targets, test coverage thresholds)
- File path ownership — which directories they can modify

Why file path ownership? It prevents two agents from modifying the same file simultaneously, and prevents a frontend agent from "helpfully" refactoring a security-critical backend module.

---

## How the Layers Interact

```
Layer 1 (Planning)
  produces stories with acceptance criteria
    |
Layer 2 (Process)
  enforces: planning before implementation, TDD during implementation
    |
Layer 3 (Specialists)
  executes tasks within their domain
    |
Layer 2 (Process)
  enforces: code review, verification before "done"
    |
Layer 1 (Planning)
  updates sprint status, marks story complete
```

The flow is not strictly linear. A specialist agent that discovers a design ambiguity escalates back to Layer 1. A verification step that finds a test gap routes back to Layer 3. The layers are checkpoints, not a waterfall.

---

## Getting Started (Incremental Adoption)

Do not try to set up all three layers at once. Adopt incrementally:

- **Week 1**: Add CLAUDE.md + WORKFLOW.md — basic project context and process pointers
- **Week 2**: Add TDD and verification skills — enforce quality discipline
- **Week 3**: Add agent profiles for your top 2-3 domains — route work by expertise
- **Month 2**: Add planning backbone if you are managing sprints or multiple epics

Each layer delivers value independently. A team that only uses Layer 2 (skills) will still produce more consistent code than a team with no process enforcement.

---

## When This Is Overkill

- Solo projects with fewer than 5 files
- One-off scripts
- Learning and tutorial projects
- Prototypes that will be thrown away

The three-layer system pays off when the codebase grows past the point where one person (or one agent context window) can hold the whole thing in mind at once. Below that threshold, the overhead is not worth it.
