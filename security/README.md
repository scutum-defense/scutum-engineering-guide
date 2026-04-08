# Security Engineering Guidelines

Scutum builds defense-grade software for sovereign operators. Security is not a feature -- it is a foundational requirement. Every engineer is responsible for the security posture of the code they write. These guidelines define the minimum security standards for all Scutum projects.

---

## Logging and Sensitive Data

### Never log sensitive data

Sensitive data includes credentials, tokens, API keys, PII (names, emails, phone numbers, national IDs), and classified operational data. These must never appear in log output, error messages, or exception stack traces.

### SafeArg and UnsafeArg

Use the SafeArg/UnsafeArg pattern to distinguish between safe-to-log and unsafe-to-log values:

```ts
import { SafeArg, UnsafeArg } from "@scutum/logging";

logger.info("Processing incident", {
  incidentId: SafeArg.of("incidentId", incident.id),       // OK to log
  operatorName: UnsafeArg.of("operatorName", operator.name), // Redacted in logs
});
```

- `SafeArg`: Values that are safe to log in plaintext (IDs, timestamps, status codes, counts)
- `UnsafeArg`: Values that must be redacted or omitted in production logs (PII, credentials, request bodies containing user data)

### Log aggregation

- All logs must use structured JSON format
- Include `requestId`, `traceId`, and `spanId` for correlation
- Never log at `DEBUG` level in production
- Rotate and expire logs according to data retention policy

---

## Input Validation

### Validate at every system boundary

Every input crossing a trust boundary must be validated before use. Trust boundaries include:

- HTTP request handlers (body, query params, headers, path params)
- Message queue consumers
- File uploads
- Database query results from untrusted sources
- External API responses

### Validation strategy

Use schema validation with runtime checking:

```ts
import { z } from "zod";

const IncidentSchema = z.object({
  title: z.string().min(1).max(500),
  severity: z.enum(["low", "medium", "high", "critical"]),
  affectedAssetIds: z.array(z.string().uuid()).min(1).max(100),
});

// In the handler:
const parsed = IncidentSchema.safeParse(req.body);
if (!parsed.success) {
  return res.status(400).json({ error: formatValidationError(parsed.error) });
}
```

### Rules

- Validate type, length, format, and range
- Reject unexpected fields (use strict schemas)
- Sanitize strings that will be rendered in HTML (XSS prevention)
- Validate content types for file uploads (do not trust file extensions)
- Set maximum request body sizes at the framework level

---

## SQL Injection Prevention

### Parameterized queries only

Never construct SQL queries by concatenating user input:

```ts
// NEVER DO THIS
const query = `SELECT * FROM incidents WHERE id = '${incidentId}'`;

// ALWAYS DO THIS
const result = await db.query(
  "SELECT * FROM incidents WHERE id = $1",
  [incidentId]
);
```

### ORM safety

When using an ORM (Prisma, Drizzle, TypeORM), use the query builder's parameterized methods. Never use `raw()` or `$queryRawUnsafe()` with user input.

### Stored procedures

Prefer stored procedures for complex queries. They provide an additional layer of parameterization and can be independently audited.

---

## Authentication and Authorization

### JWT validation

Every service that accepts JWTs must validate:

1. **Signature**: Verify against the expected signing key (RS256 or EdDSA)
2. **Expiration**: Reject expired tokens (`exp` claim)
3. **Issuer**: Match against the expected issuer (`iss` claim)
4. **Audience**: Match against the service's audience identifier (`aud` claim)
5. **Not-before**: Respect the `nbf` claim if present

### Role-Based Access Control (RBAC)

- Define roles at the platform level: `operator`, `analyst`, `admin`, `auditor`, `system`
- Enforce role checks at the handler level, not in middleware alone
- Follow the principle of least privilege: grant the minimum permissions required
- Log all access denials with the requesting identity and attempted action

### Multi-tenancy

- Tenant isolation is mandatory: queries must always include a tenant scope
- Never trust tenant identifiers from the client; derive them from the authenticated token
- Test for horizontal privilege escalation: can user A access user B's data?

---

## Secret Management

### Never in code

Secrets (API keys, database passwords, signing keys, certificates) must never appear in source code, configuration files committed to version control, or environment variable defaults in code.

### Vault integration

All production secrets are managed through a vault service (HashiCorp Vault, AWS Secrets Manager, or equivalent):

- Secrets are injected at runtime, not build time
- Applications read secrets from environment variables or mounted files
- Secret rotation is automated with zero-downtime rollover
- Access to secrets is audited

### Development secrets

- Use `.env` files locally (never committed; listed in `.gitignore`)
- Use separate secrets for each environment (dev, staging, production)
- Never copy production secrets to development environments

---

## Dependency Security

### Automated updates

All projects use Renovate for automated dependency updates. Renovate PRs must be reviewed by a code owner before merging.

### Audit

Run `pnpm audit` (or equivalent) in CI. Builds must fail on known critical or high-severity vulnerabilities.

### Review

- Review new dependencies before adding them: check maintenance status, download counts, known vulnerabilities, and license compatibility
- Prefer well-maintained, widely-used libraries over obscure alternatives
- Pin exact versions for production dependencies (use lockfiles)
- Remove unused dependencies promptly

### Supply chain

- Verify package integrity with lockfile checksums
- Use a private registry or proxy for production deployments
- Monitor for dependency hijacking and typosquatting

---

## OT Safety Rules

Scutum software may interact with operational technology (OT) systems in defense environments. The following rules are non-negotiable:

### No autonomous writes

Software must never write to OT systems without explicit human operator approval. Every write operation to an OT system must pass through the human-in-the-loop pattern (see Defense Patterns guide).

### Read-only by default

All OT integrations default to read-only mode. Write access requires explicit configuration, operator authentication, and audit logging.

### Fail safe

If communication with an OT system is lost, the software must fail to a safe state. It must never assume the last known state is still valid. It must never retry writes automatically.

### Simulation first

Before any write to an OT system, the twin validation pattern must be used to simulate the effect and present the predicted outcome to the operator for approval.

---

## Additional Requirements

- **HTTPS everywhere**: All network communication uses TLS 1.3. No exceptions.
- **CORS**: Configure CORS allowlists explicitly. Never use `*` in production.
- **CSP**: Set Content-Security-Policy headers on all web responses.
- **HSTS**: Enable HTTP Strict Transport Security with a minimum one-year max-age.
- **Rate limiting**: All public endpoints must be rate-limited (see API Design guide).
- **Security headers**: Include `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, and `Referrer-Policy: strict-origin-when-cross-origin`.
