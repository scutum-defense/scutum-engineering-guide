# Defense-Specific Engineering Patterns

These patterns address the unique requirements of building software for defense and sovereign infrastructure. They encode the principles that make Scutum software safe, auditable, and trustworthy in high-stakes environments. Every Scutum engineer must understand and apply these patterns.

---

## Human-in-the-Loop Pattern

**Principle:** Every automated action that affects the physical world or changes operational state requires explicit human operator approval before execution.

### Structure

```
Automation detects condition
    |
    v
System generates proposed action
    |
    v
Operator reviews proposed action
    |
    +---> Operator APPROVES ---> Action executes ---> Audit record created
    |
    +---> Operator REJECTS  ---> Action cancelled ---> Audit record created
    |
    +---> Timeout (no response) ---> Action cancelled ---> Audit record created
```

### Rules

- The system must never bypass operator approval, even under time pressure
- Proposed actions must include a clear description of what will happen and what will be affected
- The approval interface must show the operator all relevant context (affected assets, predicted outcomes, confidence levels)
- If the operator does not respond within the configured timeout, the system must fail to the safe state (no action)
- Every approval, rejection, and timeout is recorded in the audit chain

### Implementation

```ts
interface ProposedAction {
  id: string;
  description: string;
  affectedAssets: string[];
  predictedOutcome: string;
  confidence: number;
  expiresAt: string;
}

interface ActionDecision {
  actionId: string;
  decision: "approved" | "rejected" | "expired";
  operator: string;
  timestamp: string;
  rationale?: string;
}
```

---

## Audit Chain Pattern

**Principle:** Every decision, action, and state change must produce a traceable, immutable audit record that can be independently verified.

### Structure

Every audit entry contains:

```ts
interface AuditEntry {
  id: string;                    // Unique entry identifier
  parentId: string | null;       // Previous entry in the chain (null for genesis)
  hash: string;                  // Cryptographic hash of this entry
  timestamp: string;             // ISO 8601 timestamp
  actor: string;                 // Who performed the action (user ID or system ID)
  action: string;                // What was done
  target: string;                // What was affected
  decision: string;              // What was decided
  policyLabel: string;           // Which policy governed the decision
  context: Record<string, unknown>; // Additional context (SafeArg values only)
}
```

### Rules

- Audit entries are append-only. They must never be modified or deleted.
- Each entry includes a cryptographic hash of its contents and the hash of the previous entry, forming a verifiable chain.
- The audit chain must be readable by authorized auditors without requiring access to the operational system.
- Audit storage must be separate from operational storage to prevent tampering.
- The system must detect and alert on chain integrity violations (missing entries, hash mismatches).

### What must be audited

- Operator decisions (approvals, rejections)
- System-generated recommendations
- Policy evaluations and their outcomes
- Configuration changes
- Access control decisions (grants, denials)
- Data exports and transfers

---

## Policy Gate Pattern

**Principle:** A policy engine evaluates every proposed action against the active policy set before the action is allowed to proceed.

### Structure

```
Proposed Action
    |
    v
Policy Engine
    |
    +---> Evaluate against active policies
    |
    +---> Result: ALLOW / DENY / ESCALATE
    |
    v
Action proceeds (if ALLOW) or is blocked (if DENY)
or requires additional approval (if ESCALATE)
```

### Rules

- Every action that modifies state must pass through the policy gate
- Policies are versioned and auditable (changes to policies are themselves audited)
- The policy engine is a separate, independently deployable service
- Policy evaluation must be deterministic: the same input always produces the same output
- If the policy engine is unreachable, the default decision is DENY (fail-closed)

### Policy structure

```ts
interface Policy {
  id: string;
  version: number;
  name: string;
  description: string;
  rules: PolicyRule[];
  effectiveFrom: string;
  effectiveUntil: string | null;
}

interface PolicyRule {
  condition: string;   // Expression evaluated against the action context
  decision: "ALLOW" | "DENY" | "ESCALATE";
  priority: number;    // Higher priority rules are evaluated first
}
```

---

## Twin Validation Pattern

**Principle:** Before executing any action on a real system, simulate the action on a digital twin and present the predicted outcome to the operator.

### Structure

```
Proposed Action
    |
    v
Digital Twin (simulation environment)
    |
    +---> Simulate action
    +---> Predict outcome
    +---> Compute confidence score
    |
    v
Present predicted outcome to operator
    |
    +---> Operator reviews simulation results
    +---> Operator approves or rejects (Human-in-the-Loop)
    |
    v
Execute on real system (if approved)
```

### Rules

- The digital twin must reflect the current state of the real system as closely as possible
- Simulation results must include a confidence score indicating how reliable the prediction is
- If the twin is stale (last sync exceeds threshold), the system must warn the operator
- Simulations must not have side effects on the real system
- The gap between simulation and execution must be minimized to reduce state drift

---

## Safe vs. Unsafe Path Pattern

**Principle:** When presenting options to an operator, always show both the safe outcome and the risk of the proposed action.

### Structure

Every recommendation presented to an operator includes:

```ts
interface RecommendationView {
  action: string;
  safePath: {
    description: string;     // What happens if we take the safe/conservative option
    tradeoffs: string[];     // What we give up by being conservative
  };
  unsafePath: {
    description: string;     // What happens if we take the aggressive/risky option
    risks: string[];         // What could go wrong
    mitigations: string[];   // How we can reduce the risk
  };
  confidence: number;
  recommendedPath: "safe" | "unsafe";
  rationale: string;
}
```

### Rules

- The system must never hide risks from the operator
- Both paths must be presented with equal prominence (no visual bias toward either option)
- Confidence scores must be calibrated and honest (do not inflate confidence to encourage action)
- The system may recommend a path, but the operator always decides

---

## Sovereign Deployment Pattern

**Principle:** Scutum software is deployed in single-tenant, sovereign environments where data never leaves the customer's jurisdiction.

### Rules

- **Single tenant**: Each customer gets a dedicated deployment. No shared infrastructure, no shared databases, no shared compute.
- **Data residency**: All data is stored within the customer's geographic jurisdiction. No data is transmitted to external services, analytics platforms, or telemetry endpoints.
- **Air-gap capable**: The software must function fully in air-gapped environments with no internet connectivity. External dependencies must be vendored or mirrored.
- **Customer-controlled encryption**: Encryption keys are generated and managed by the customer. Scutum does not hold or escrow keys.
- **Update sovereignty**: The customer controls when updates are applied. There are no forced updates, no auto-updates, and no phone-home behavior.

### Implementation checklist

- No hardcoded external URLs (analytics, CDN, telemetry)
- All dependencies are vendored or available from a private registry
- Container images are self-contained and do not pull layers at runtime
- DNS resolution works in private networks (no reliance on public DNS)
- Time synchronization works with local NTP servers

---

## Detection Rule Authoring Pattern

**Principle:** Detection rules must be versioned, tested, and deployed through the same engineering rigor as application code.

### Structure

```ts
interface DetectionRule {
  id: string;
  version: number;
  name: string;
  description: string;
  severity: "low" | "medium" | "high" | "critical";
  query: string;                    // The detection logic (query, pattern, or expression)
  dataSources: string[];            // Which data sources this rule monitors
  falsePositiveGuidance: string;    // How to distinguish true positives from false positives
  responseGuidance: string;         // What the operator should do when this rule fires
  testCases: DetectionTestCase[];   // Automated test scenarios
  mitreTechniques: string[];        // MITRE ATT&CK technique IDs
}

interface DetectionTestCase {
  description: string;
  input: unknown;
  expectedResult: "match" | "no-match";
}
```

### Rules

- Every detection rule must have at least one test case that triggers a match and one that does not
- Detection rules are version-controlled alongside application code
- Changes to detection rules go through code review
- Detection rules include false positive guidance to reduce operator fatigue
- Detection rules reference MITRE ATT&CK techniques for standardized classification
- Detection rule performance (execution time, resource consumption) must be benchmarked before deployment
