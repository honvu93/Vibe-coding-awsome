# Multi-Product Workspaces

## The Pattern

Some teams manage multiple related products from a single workspace. This is different
from a monorepo — it is a collection of independent projects with shared planning
artifacts, documentation, and tooling. Each product has its own dependency tree,
its own test suite, and its own deployment pipeline.

```
workspace/
├── CLAUDE.md                  # Workspace context: active product, shared conventions
├── product-app/               # Main active product
│   ├── CLAUDE.md              # Deep technical context for this product
│   └── src/
├── legacy-app/                # Previous version (reference only, not active)
│   └── src/
├── shared-lib/                # Shared utilities, if any
│   └── src/
├── planning/                  # Cross-product planning artifacts
│   ├── planning-artifacts/    # PRDs, architecture docs, epics
│   └── implementation/        # Stories, sprint tracking
├── scripts/                   # Cross-product automation
└── docs/                      # Shared documentation
```

## Why Multi-Product?

- Related products share domain knowledge and vocabulary
- Planning artifacts reference context that spans multiple products
- Migrating from legacy to new product benefits from physical proximity
- Shared scripts and tooling can live in one place without publishing to a registry

This is **not** the right structure for unrelated projects. The workspace pattern only
earns its overhead when the products genuinely share context.

## Workspace-Level CLAUDE.md

The top-level context file (whether you call it CLAUDE.md, AGENTS.md, or a README) is
critical. It tells any AI assistant — and any new team member — which product is
active, what the relationships are, and where to find detailed context.

It should include:

1. Which product is currently active (be explicit)
2. The relationship between products (migration? parallel? legacy?)
3. A pointer to each product's own context file
4. Shared conventions: language, commit style, branch strategy, testing requirements
5. Quick-reference commands for the active product so nothing requires drilling down

Example structure:

```markdown
# Workspace Context

## Active Product
All development work happens in `product-app/`. See `product-app/CLAUDE.md` for the
full technical reference. The `legacy-app/` directory is read-only reference material.

## Quick Reference
```bash
cd product-app
npm install
npm test
npm run dev
```

## Shared Conventions
- Commit after every completed unit of work, never batch
- All currency in USD
- Tests are required before any production code change
```

## Product-Level Context File

Each product directory gets its own detailed context file covering:

- Full command reference (setup, dev, test, build, deploy)
- Architecture overview (directory structure, key modules, data flow)
- Testing patterns (what runner, where tests live, naming conventions)
- Environment variables (names, defaults, which are required)
- Domain-specific conventions (error patterns, naming, API contract style)

The product-level file is the authoritative reference for that product. The workspace-
level file should not duplicate it — only point to it and summarize what is needed to
orient a new session.

## Cross-Product References

When the active product inherits patterns from a legacy product:

- Document the relationship explicitly in the workspace context file
- Reference the legacy code for orientation, but rewrite for the new product — do not
  copy-paste across products unless you are intentionally extracting a shared library
- Keep the legacy product directory as read-only reference. Do not actively develop in
  both simultaneously — context switching across products in a single session degrades
  quality significantly

## Planning Artifact Location

Planning artifacts belong at the workspace level, not inside any product directory:

- PRDs, architecture documents, and epics live in `planning/planning-artifacts/`
- Stories and sprint tracking live in `planning/implementation/`
- Stories reference specific files and directories inside the target product

This separation keeps the product directory clean and makes it easy to archive or
delete a product without losing its planning history.

## Anti-Patterns

**No workspace-level context file.** An AI assistant opened on the workspace root has
no idea which product is active. The first thing it does is ask, or worse, guess wrong.

**Developing in multiple products simultaneously.** Pick one. Finish it. Switch. Trying
to run parallel development sessions across two products in one context window produces
mediocre results in both.

**Duplicating shared code.** If both products import the same utility, extract it to
`shared-lib/` with its own versioning. Copy-pasting shared code creates a maintenance
debt that compounds every time the logic needs to change.

**Planning artifacts scattered inside product directories.** A story file inside
`product-app/docs/` is invisible to workspace-level planning tools and harder to
archive when the product is retired.

**No clear "active product" designation.** If the workspace context file does not say
explicitly which product is active, every new session starts with ambiguity.

## Migration Workflows

When migrating from a legacy product to a new one:

1. Declare the migration state in the workspace context file. Example: "legacy-app is
   archived. All new development is in product-app."
2. Treat the legacy directory as read-only. Do not fix bugs there — port the fix to the
   new product instead.
3. Reference legacy patterns in planning docs to explain design decisions in the new
   product, but do not reopen legacy files to make changes.
4. Once migration is complete and the new product is stable, archive or delete the
   legacy directory. Keeping it around indefinitely creates confusion.

## Tooling Considerations

If products share tooling (linters, formatters, test runners), consider:

- A root-level config file that each product extends (e.g., `.eslintrc` with
  `extends: "../../.eslintrc.base"`)
- A root-level `Makefile` or `justfile` with targets that delegate to each product
- Cross-product scripts in `scripts/` that accept the target product as an argument

Avoid making any product depend on the workspace root at runtime. Products should be
independently deployable without the workspace structure.
