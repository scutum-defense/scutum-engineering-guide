# TypeScript Style Guide

This guide defines the TypeScript conventions for all Scutum projects. Every contributor is expected to follow these rules. Automated linting enforces as many of these as possible, but code reviewers should verify the intent-level rules that tooling cannot catch.

---

## Strict Mode

All Scutum TypeScript projects must enable strict mode in `tsconfig.json`:

```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true
  }
}
```

Never disable strict checks for convenience. If a library's types are incompatible with strict mode, add a thin wrapper with proper types rather than weakening the compiler.

---

## Type Annotations

### When to annotate explicitly

- **Function signatures**: Always annotate parameter types and return types for exported functions. This serves as documentation and prevents accidental signature changes.
- **Complex inferred types**: When the inferred type is a union, intersection, or deeply nested generic, add an explicit annotation for readability.
- **Public API surfaces**: Every exported type, interface, function, and constant must have explicit types.

### When inference is sufficient

- **Local variables**: Let TypeScript infer the type from the initializer. `const count = 0` is preferred over `const count: number = 0`.
- **Callback parameters**: When the context provides the type (e.g., `.map((item) => ...)`) do not annotate.
- **Trivial returns**: Private helper functions with obvious return types can omit the annotation.

### Prefer interfaces over type aliases for object shapes

```ts
// Preferred
interface Incident {
  id: string;
  severity: string;
}

// Avoid for object shapes
type Incident = {
  id: string;
  severity: string;
};
```

Use `type` for unions, intersections, mapped types, and utility types.

---

## Naming Conventions

| Construct | Convention | Example |
|---|---|---|
| Interfaces and type aliases | PascalCase | `Incident`, `PolicyResult` |
| Classes | PascalCase | `AuditChain` |
| Functions and methods | camelCase | `evaluatePolicy`, `fetchIncident` |
| Variables and parameters | camelCase | `currentIncident`, `policyLabel` |
| Constants (module-level) | UPPER_SNAKE_CASE | `MAX_RETRY_COUNT`, `DEFAULT_TIMEOUT_MS` |
| Enum members | PascalCase | `Severity.Critical` |
| Boolean variables | `is`/`has`/`should` prefix | `isActive`, `hasPermission` |
| Private class members | no prefix (use `private` keyword) | `private connection` |

Never use Hungarian notation. Never prefix interfaces with `I`.

---

## Import Ordering

Organize imports in the following groups, separated by blank lines:

1. **Node built-ins** (e.g., `node:fs`, `node:path`)
2. **External packages** (e.g., `react`, `zod`)
3. **Scutum packages** (e.g., `@scutum/sdk`, `@scutum/policy-engine`)
4. **Relative imports** (e.g., `./utils`, `../hooks/use-incident`)

```ts
import { readFile } from "node:fs/promises";

import { z } from "zod";
import express from "express";

import { ScutumClient } from "@scutum/sdk";
import { PolicyEngine } from "@scutum/policy-engine";

import { validateInput } from "./validation";
import type { Incident } from "../types";
```

Always use `import type` for type-only imports. This ensures they are erased at compile time and do not create runtime dependencies.

---

## Error Handling

### Result types over exceptions

Library code must never throw exceptions for expected failure cases. Use a `Result` pattern:

```ts
type Result<T, E = Error> =
  | { ok: true; value: T }
  | { ok: false; error: E };

function parseConfig(raw: string): Result<Config, ConfigError> {
  // ...
}
```

### When throwing is acceptable

- Application entry points (CLI `main`, server bootstrap)
- Truly exceptional programmer errors (assertion failures)
- Framework boundaries that expect thrown errors (Express error middleware)

### Error messages

- Always include context: what was attempted, what went wrong, and what values were involved (using SafeArg for sensitive data).
- Never swallow errors silently. At minimum, log them.

---

## Async Patterns

### Always handle errors

Every `await` expression must be inside a try/catch or have a `.catch()` handler. Unhandled promise rejections crash the process.

### No floating promises

Never call an async function without awaiting it, returning it, or explicitly voiding it:

```ts
// BAD: floating promise
fetchData();

// GOOD
await fetchData();

// GOOD: intentionally fire-and-forget (must be explicit)
void fetchData();
```

Configure the `@typescript-eslint/no-floating-promises` rule to enforce this.

### Prefer Promise.all for independent work

```ts
// GOOD: parallel fetches
const [incident, audit] = await Promise.all([
  fetchIncident(id),
  fetchAuditTrail(id),
]);

// BAD: sequential when order does not matter
const incident = await fetchIncident(id);
const audit = await fetchAuditTrail(id);
```

### Async iteration

Use `for await...of` for streaming data. Never collect an entire stream into an array unless bounded.

---

## Module Structure

### One concept per file

Each file should export a single primary concept (a function, a class, a type, or a tightly related group). Avoid files that export unrelated utilities.

### Barrel exports

Use `index.ts` barrel files sparingly. They increase bundle size and create circular dependency risks. Prefer direct imports from the source file.

### File naming

- Use kebab-case for file names: `policy-engine.ts`, `use-incident.ts`
- Test files sit next to their source: `policy-engine.test.ts`
- Type-only files use `.types.ts` suffix when they contain only type definitions

### Directory structure

```
src/
  components/     # UI components (React)
  hooks/          # React hooks
  lib/            # Core business logic (framework-agnostic)
  types/          # Shared type definitions
  utils/          # Pure utility functions
  pages/          # Page-level compositions
```

---

## Miscellaneous

- **Prefer `const` over `let`**. Never use `var`.
- **Use template literals** over string concatenation.
- **Avoid `any`**. Use `unknown` and narrow with type guards. If `any` is truly unavoidable, add a `// eslint-disable-next-line` comment explaining why.
- **Prefer readonly**. Use `readonly` on properties and `ReadonlyArray` when mutation is not needed.
- **No magic numbers**. Extract into named constants with UPPER_SNAKE_CASE.
- **Use exhaustive switch**. Every switch on a union type must handle all cases or have a `default: assertNever(x)` guard.
