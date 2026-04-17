# Refactoring Patterns for AI-Assisted Development

## When to Refactor

Refactor when:

- A module has grown too large (over 1000 lines is a smell)
- Circular dependencies exist between modules
- The same code is duplicated in 3 or more places
- A "god module" touches everything in the system
- New features are increasingly hard to add to a specific area

Do not refactor when:

- "The code could be cleaner" — but it works fine and is not changing
- You are in the middle of a feature — finish first, refactor in a separate commit
- The improvement is speculative — "we might need this later"

## Pattern 1: Breaking Circular Dependencies

### The Problem

Module A imports from Module B, and Module B imports from Module A. This creates unpredictable initialization order, tight coupling, and import errors in some environments.

### Solutions (in order of preference)

**Extract shared code to a new module:**

```
Before: A <-> B (circular)
After:  A  ->  C  <- B  (shared dependency)
```

Move the code both modules need into Module C. Neither A nor B knows about each other.

**Lazy dynamic imports** (when extraction is impractical):

```javascript
// Instead of a top-level import that creates the cycle:
// import { something } from './moduleB.js'

// Use a dynamic import where the dependency is actually needed:
async function doWork() {
  const { something } = await import('./moduleB.js')
  return something()
}
```

**Dependency injection** (when you control the call site):

```javascript
// Module A does not import Module B directly.
// Instead, B is passed in as an argument.
function createServiceA(serviceB) {
  return {
    doWork() {
      return serviceB.getData()
    }
  }
}
```

**Event-based decoupling** (for fire-and-forget communication):

```javascript
// Module A emits an event instead of calling Module B directly.
eventBus.emit('order-completed', orderData)

// Module B listens without A knowing B exists.
eventBus.on('order-completed', (data) => { /* ... */ })
```

## Pattern 2: Extracting Domains

### When a Module Is Too Big

Signs: over 1000 lines of code, more than 5 unrelated responsibilities, or the file is modified by nearly every feature.

### The Extraction Process

1. **Identify the bounded context** — What subset of functionality is cohesive and self-contained?
2. **Map the dependencies** — What does the subset import? What imports it?
3. **Create the new module** following the project's standard file pattern.
4. **Move the code** — functions, tests, and types.
5. **Update all imports** across the codebase that pointed at the old location.
6. **Re-wire API routes or entry points** to reference the new module.
7. **Verify** — All tests pass, no broken imports, no regressions.

### Rule: One Extraction Per Commit

Each extraction step should be a standalone commit. This makes rollback trivial and keeps the diff readable. Never combine an extraction with a feature change.

## Pattern 3: Deduplicating Cross-Cutting Code

### The 3-File Rule

If the same pattern appears in 3 or more files, extract it:

| Repeated pattern | Where to extract |
|---|---|
| Error response builders | Shared utility module |
| Auth/session checks | Middleware |
| File type validation | Shared helper |
| API response formatting | Shared helper |
| Database connection setup | Infrastructure layer |

### Do Not Extract Prematurely

Two occurrences is not duplication — it is coincidence. Wait for the third before extracting.

## Pattern 4: Infrastructure Extraction

### What Belongs in Infrastructure

Code that is used by multiple domains, handles cross-cutting concerns (auth, logging, caching), manages external connections (database, queue, HTTP), or provides security primitives (encryption, hashing) belongs in an infrastructure layer, not in a domain module.

### Infrastructure Organization

```
src/infrastructure/
├── db/          # Database client, connection management
├── security/    # Encryption, auth utilities, credential management
├── queue/       # Job queue setup, worker factories
├── cache/       # Caching layer
├── http/        # Health servers, shared HTTP utilities
├── events/      # Event bus, lifecycle event emission
└── resilience/  # Circuit breakers, retry logic
```

Domain modules import from infrastructure. Infrastructure modules do not import from domain modules. This one-way dependency is what prevents circular imports at scale.

## Refactoring with AI — Best Practices

- Map dependencies before refactoring. An AI that does not understand the dependency graph will create new circular imports while fixing old ones.
- Create a plan listing exact file moves and import changes. Execute the plan one step at a time.
- Commit after each extraction step. Atomic, reversible commits.
- Run the full test suite after every commit. Do not batch multiple extractions and test at the end.
- Never combine refactoring with feature work in the same commit. The diff becomes unreadable and the blast radius of any error expands.

## The Refactoring Mindset

Refactoring is not improvement for its own sake. It is a precondition for adding the next feature without accumulating debt. If a refactor does not make the next task easier, ask whether it is worth doing at all.

The best refactors are invisible to users and obvious to the next developer who reads the code.
