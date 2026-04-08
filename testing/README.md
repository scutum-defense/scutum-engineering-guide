# Testing Guidelines

Automated tests are the primary mechanism for maintaining confidence in Scutum software. Every change must be tested. These guidelines define what to test, how to structure tests, and what standards to meet.

---

## Test File Naming

Test files sit next to the source file they test and use the `.test.ts` (or `.test.tsx`) suffix:

```
src/
  lib/
    policy-engine.ts
    policy-engine.test.ts
  hooks/
    use-incident.ts
    use-incident.test.ts
```

Integration tests that span multiple modules live in a `__tests__/` directory at the project root or within the relevant `src/` subdirectory.

---

## Test Structure

Use the `describe` / `it` / `expect` pattern:

```ts
describe("PolicyEngine", () => {
  describe("evaluate", () => {
    it("returns ALLOW when the policy permits the action", () => {
      const engine = new PolicyEngine(allowAllPolicy);
      const result = engine.evaluate(action);
      expect(result.decision).toBe("ALLOW");
    });

    it("returns DENY when the policy forbids the action", () => {
      const engine = new PolicyEngine(denyAllPolicy);
      const result = engine.evaluate(action);
      expect(result.decision).toBe("DENY");
    });

    it("returns DENY when the policy is missing", () => {
      const engine = new PolicyEngine(undefined);
      const result = engine.evaluate(action);
      expect(result.decision).toBe("DENY");
    });
  });
});
```

### Naming conventions

- `describe` blocks name the unit under test (class, function, or component)
- Nested `describe` blocks name the method or scenario
- `it` blocks describe the expected behavior in plain language
- Use the pattern: "returns X when Y", "throws when Z", "renders A given B"

### Setup and teardown

- Use `beforeEach` for per-test setup, not `beforeAll` (isolation matters)
- Clean up side effects in `afterEach`
- Avoid shared mutable state between tests

---

## What to Test

### Public API surface

Test every exported function, class method, and component's rendered output. If it is part of the public contract, it must have a test.

### Edge cases

- Empty inputs (empty strings, empty arrays, null, undefined)
- Boundary values (0, -1, MAX_SAFE_INTEGER, very long strings)
- Invalid inputs (wrong types, malformed data)

### Error paths

- Network failures (timeouts, connection refused)
- Invalid responses (malformed JSON, unexpected status codes)
- Permission denied scenarios
- Resource not found scenarios

### State transitions

For stateful components or state machines, test every valid transition and verify that invalid transitions are rejected.

---

## What NOT to Test

### Implementation details

Do not test private methods, internal state, or how a result is computed. Test the observable behavior. If you refactor internals, tests should not break.

### Framework behavior

Do not test that React renders a `<div>` or that Express routes requests. Trust the framework. Test your code.

### Trivial code

Do not write tests for simple getters, setters, or pass-through functions that contain no logic.

### Third-party libraries

Do not re-test behavior of external libraries. If you depend on specific behavior, write a thin wrapper and test the wrapper.

---

## Mocking Guidelines

### Prefer real dependencies

Use real implementations whenever practical. Real dependencies catch integration issues that mocks hide.

### Mock at boundaries

When mocking is necessary, mock at system boundaries:

- HTTP clients (mock the network, not the client library)
- Database connections (use test databases or in-memory stores)
- External service integrations
- Time-dependent code (mock `Date.now()` or use fake timers)

### Mock rules

- Never mock the unit under test
- Verify that mocks are called with expected arguments (interaction testing)
- Reset mocks between tests to prevent cross-contamination
- Prefer dependency injection over module-level mocking

### Test doubles hierarchy

Prefer the simplest double that meets the need:

1. **Stub**: Returns canned responses (simplest)
2. **Spy**: Records calls for verification
3. **Mock**: Stubs with built-in verification
4. **Fake**: Lightweight working implementation (e.g., in-memory database)

---

## Coverage Expectations

### Minimum thresholds

| Metric | Minimum |
|---|---|
| Line coverage | 80% |
| Branch coverage | 75% |
| Function coverage | 85% |

### Critical paths

Security-critical code (authentication, authorization, policy evaluation, audit logging) must have 95%+ line coverage. There are no exceptions.

### Coverage is necessary but not sufficient

High coverage does not mean high quality. A test that asserts nothing has 100% coverage and 0% value. Every test must make meaningful assertions about behavior.

### CI enforcement

Coverage thresholds are enforced in CI. PRs that decrease coverage below the threshold are blocked until additional tests are added.

---

## Test Performance

- Unit tests must complete in under 10 seconds total
- Individual test cases must complete in under 100ms
- Integration tests have a 60-second timeout per suite
- Use parallel test execution where possible
- Avoid `setTimeout` and real delays in tests; use fake timers
