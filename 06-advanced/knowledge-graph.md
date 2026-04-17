# Knowledge Graphs for Codebase Understanding

## The Problem

Before refactoring or adding a major feature, you need to understand the dependency
landscape. But reading code file-by-file misses the big picture:

- Which modules are tightly coupled?
- Where are the "god modules" that everything depends on?
- Which changes will cascade across many files?
- Are there hidden circular dependencies?

## What a Knowledge Graph Shows

A knowledge graph maps your codebase as nodes (modules/files) and edges (imports/calls):

- **High edge-count nodes** — coupling hotspots (change one, break many)
- **Clusters** — naturally cohesive modules that belong together
- **Cross-cluster edges** — integration points that need careful handling
- **Cycles** — circular dependencies that need breaking

## Tools for Generation

Several approaches work depending on your language and toolchain:

| Tool | Language | Command |
|---|---|---|
| **dependency-cruiser** | JS/TS | `npx depcruise --output-type dot src/` |
| **pydeps** | Python | `pydeps mypackage` |
| **go-callvis** | Go | `go-callvis ./...` |
| **graphviz** | Any | Consume DOT output from above tools |
| **Custom scripts** | Any | Parse imports with regex, output DOT format |
| **AI-assisted** | Any | Feed code or docs to an LLM, ask for a structured graph |

All of these produce either a DOT file or JSON that can be rendered visually with
Graphviz (`dot -Tsvg graph.dot -o graph.svg`) or an interactive tool like D3.

## When to Generate

- **Before refactoring** — Map what you will touch and what cascades
- **Before adding a major feature** — Understand where it connects
- **When a module feels "too big"** — Visualize to find extraction boundaries
- **After a merge** — Verify no unintended new dependencies were introduced
- **Monthly health check** — Track coupling trends over time

## How to Read the Graph

1. **Find the biggest nodes.** These are your coupling hotspots. If everything imports
   `utils.js`, that is a risk — a bug there breaks everything.
2. **Find clusters.** Groups of tightly connected nodes are natural domain boundaries.
   If you do not have explicit domain directories yet, the graph will show you where
   they should be.
3. **Find cross-cluster edges.** These are your integration points. They should be
   narrow and well-defined. Too many cross-cluster edges means the "boundary" is
   not a real boundary.
4. **Find cycles.** Any cycle (A imports B, B imports C, C imports A) is a circular
   dependency. Circular dependencies cause load-order bugs, prevent tree-shaking, and
   make testing harder.

## Using Graphs with AI

When asking an AI assistant to refactor a large codebase:

1. Generate the graph first.
2. Share the graph output (DOT or JSON) with the AI alongside your code.
3. Ask the AI to identify extraction boundaries and coupling hotspots.
4. Use the graph to validate the AI's refactoring plan — a good refactor should reduce
   cross-cluster edges and eliminate cycles.
5. Regenerate the graph after the refactor to confirm improvement.

## Incremental Updates

Regenerating the full graph on every file change is expensive. Instead:

- Regenerate only after significant structural changes (new domain, large refactor).
- Use `--include-only` or similar flags to scope regeneration to changed modules.
- Store graph snapshots in version control (`graphs/before-refactor.svg`,
  `graphs/after-refactor.svg`) to make the before/after comparison reviewable in PRs.

## Practical Workflow

```
1. Generate graph before starting refactor
2. Identify coupling hotspots and cycles
3. Plan refactor to break cycles and reduce hotspot edge counts
4. Execute refactor (atomic commits per extracted module)
5. Regenerate graph after refactor
6. Compare: coupling should be reduced, no new cycles introduced
```

## Interpreting Coupling Metrics

When using dependency-cruiser or similar tools that produce metrics alongside the graph:

- **Fan-in** (how many modules import this one) — high fan-in means high blast radius
  on change. These modules need the most stable contracts.
- **Fan-out** (how many modules this one imports) — high fan-out means high surface
  area. These modules are likely doing too much.
- **Instability** = fan-out / (fan-in + fan-out). High instability means a module
  changes often and is depended on by few. Low instability means stable and widely
  depended on. Core utilities should have low instability.

## When This Is Overkill

- Projects with fewer than 20 files (you can hold the dependency graph in your head)
- Pure UI component libraries with no shared state (the component tree is the graph)
- Single-purpose scripts with no complex import chains
- Projects you are about to delete or archive

## Further Reading

- Dependency Cruiser docs: https://github.com/sverweij/dependency-cruiser
- DOT language reference: https://graphviz.org/doc/info/lang.html
- Robert Martin's Stable Dependencies Principle (a formal basis for the metrics above)
