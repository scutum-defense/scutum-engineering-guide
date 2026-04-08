# scutum-engineering-guide

```
 ____            _
/ ___|  ___ _   _| |_ _   _ _ __ ___
\___ \ / __| | | | __| | | | '_ ` _ \
 ___) | (__| |_| | |_| |_| | | | | | |
|____/ \___|\__,_|\__|\__,_|_| |_| |_|
  E N G I N E E R I N G   G U I D E
```

[![License: Apache-2.0](https://img.shields.io/badge/License-Apache%202.0-blue)](./LICENSE)
[![Guides](https://img.shields.io/badge/Guides-5-b99a66)](.)
[![Defense Grade](https://img.shields.io/badge/Defense-Grade-critical)](.)

**Engineering style guides and best practices for building defense-grade sovereign infrastructure software.**

This repository contains the engineering standards that govern all Scutum codebases. These guides encode the patterns, conventions, and safety requirements that make Scutum software trustworthy in high-stakes defense environments.

---

## Table of Contents

| Guide | Path | Description |
|---|---|---|
| [TypeScript Style Guide](./typescript/) | `typescript/` | Language conventions, type annotation rules, naming, imports, error handling, and module structure |
| [API Design Guidelines](./api-design/) | `api-design/` | RESTful conventions, response envelopes, error formats, pagination, versioning, and authentication |
| [Security Engineering](./security/) | `security/` | Logging safety, input validation, injection prevention, authentication, secret management, and OT safety |
| [Testing Guidelines](./testing/) | `testing/` | Test structure, what to test, mocking strategy, coverage expectations, and performance requirements |
| [Defense Patterns](./defense-patterns/) | `defense-patterns/` | Human-in-the-loop, audit chains, policy gates, twin validation, safe/unsafe paths, and sovereign deployment |

---

## Who This Is For

These guides are for every engineer who contributes to the Scutum platform:

- **New engineers** joining Scutum should read all five guides during their first week. They serve as the onboarding curriculum for engineering standards.

- **Experienced engineers** use these guides as reference material during code reviews, architecture discussions, and design decisions.

- **Architecture council members** maintain these guides and ensure they evolve with the platform. Changes to guides go through the same review process as code changes.

- **Partner engineers** building applications on the Scutum platform should read the TypeScript, API Design, and Security guides to ensure their integrations meet Scutum standards.

- **Auditors and compliance reviewers** reference the Security and Defense Patterns guides to understand the safety mechanisms built into Scutum software.

---

## How to Use These Guides

### During onboarding

Read all five guides in order: TypeScript, API Design, Security, Testing, Defense Patterns. Each guide builds on concepts from the previous ones.

### During code review

Reference specific sections in PR comments. For example: "This violates the SafeArg logging rule -- see security/README.md." Link directly to the relevant section so the author can read the full context.

### During design

When proposing a new feature or system, verify that the design follows the patterns in these guides. The defense patterns guide is especially important for features that interact with operator workflows or OT systems.

### In documentation

Link to these guides from project-level documentation rather than duplicating the content. This ensures a single source of truth.

---

## Guide Summaries

### TypeScript Style Guide

Defines the TypeScript conventions that apply to every Scutum project. Covers strict mode configuration, when to use explicit type annotations versus inference, naming conventions for all language constructs, import ordering rules, error handling with Result types instead of exceptions, async patterns that prevent floating promises, and module structure conventions. The goal is readable, maintainable, type-safe code that any team member can understand.

### API Design Guidelines

Establishes the contract between Scutum services and their consumers. Covers RESTful resource naming, HTTP method usage, the standard response envelope format with `data`, `meta`, and `pagination` fields, error response format with machine-readable error codes, cursor-based and offset-based pagination strategies, URL path versioning with lifecycle management, authentication via Bearer tokens and API keys, rate limiting headers, and health check endpoint requirements. Consistent API design reduces integration friction across the platform.

### Security Engineering Guidelines

Defines the security baseline for all Scutum software. Covers the SafeArg/UnsafeArg pattern for safe logging, input validation at every trust boundary using schema validation, SQL injection prevention through parameterized queries, JWT validation requirements, RBAC enforcement, multi-tenancy isolation, secret management through vault services, dependency security with automated auditing, and OT safety rules that prohibit autonomous writes to operational technology systems. Security is not optional in defense software.

### Testing Guidelines

Specifies how to write effective automated tests. Covers test file naming and co-location, the describe/it/expect structure, what to test (public API, edge cases, error paths, state transitions), what not to test (implementation details, framework behavior, trivial code), mocking guidelines that prefer real dependencies over mocks, the test doubles hierarchy, minimum coverage thresholds (80% line, 75% branch, 85% function), elevated coverage requirements for security-critical code (95%+), and test performance budgets.

### Defense-Specific Patterns

Codifies the engineering patterns unique to defense software. The human-in-the-loop pattern ensures every automated action requires operator approval. The audit chain pattern creates immutable, cryptographically verifiable records of every decision. The policy gate pattern enforces policy evaluation before any state-changing action. The twin validation pattern simulates actions on a digital twin before executing on real systems. The safe vs. unsafe path pattern ensures operators always see both the conservative and aggressive options with their tradeoffs. The sovereign deployment pattern mandates single-tenant, air-gap-capable, customer-controlled infrastructure. The detection rule authoring pattern applies software engineering rigor to threat detection logic.

---

<details>
<summary><strong>Contributing</strong></summary>

### Process

1. **Propose**: Open an issue describing the change you want to make and why
2. **Draft**: Create a branch and write the guide content in Markdown
3. **Review**: Submit a PR. All guide changes require approval from at least two members of `@ScutumDefense/architecture-council`
4. **Merge**: Once approved, the change is merged to `main` and takes effect immediately

### Writing standards

- Use clear, direct language. Avoid jargon unless it is a defined term.
- Include code examples for every rule that can be demonstrated with code.
- State the "why" alongside the "what". Engineers follow rules they understand.
- Use tables for reference material (naming conventions, coverage thresholds).
- Use the imperative mood for rules: "Always validate input" not "Input should be validated."

### What belongs in these guides

- Language and framework conventions that apply to all Scutum projects
- API design patterns that affect cross-service compatibility
- Security requirements that are non-negotiable
- Testing standards that maintain quality across the codebase
- Defense-specific patterns that encode operational safety requirements

### What does NOT belong here

- Project-specific configuration (belongs in the project's own docs)
- Tutorials or how-to guides (belong in a separate knowledge base)
- Temporary workarounds or exceptions (track in issues, not in guides)
- Tool-specific setup instructions (belong in project READMEs)

</details>

<details>
<summary><strong>Versioning</strong></summary>

These guides are versioned implicitly through Git history. To reference a specific version of a guide, use the Git commit hash.

Major changes (new guides, removed sections, changed requirements) are announced in the `#engineering` channel and tracked in the changelog below.

### Changelog

**2026-04-08** -- Initial release

- TypeScript Style Guide v1.0
- API Design Guidelines v1.0
- Security Engineering Guidelines v1.0
- Testing Guidelines v1.0
- Defense-Specific Engineering Patterns v1.0

</details>

<details>
<summary><strong>Frequently Asked Questions</strong></summary>

**Q: Do these guides apply to prototype/spike code?**

A: The Security and Defense Patterns guides apply to all code, including prototypes. The TypeScript and Testing guides can be relaxed for throwaway spikes that will never be merged, but any code that reaches a PR must comply.

**Q: What if a guide conflicts with a project's existing conventions?**

A: These guides take precedence. If a project has a legitimate reason to deviate, document the exception in the project's README and get approval from the architecture council.

**Q: How often are guides updated?**

A: As needed. There is no fixed schedule. Changes are driven by new patterns, lessons learned from incidents, or evolving platform requirements.

**Q: Can I suggest a new guide topic?**

A: Yes. Open an issue in this repository with the proposed topic, scope, and motivation. The architecture council will evaluate it.

**Q: Are these guides enforced automatically?**

A: As much as possible. Linting rules, TypeScript compiler settings, and CI checks enforce many of these conventions automatically. Rules that require human judgment are enforced during code review.

</details>

<details>
<summary><strong>Related Resources</strong></summary>

- [scutum-starter-app](https://github.com/ScutumDefense/scutum-starter-app) -- React starter template that follows these guides
- [@scutum/sdk](https://github.com/ScutumDefense/scutum-sdk) -- TypeScript SDK for the Scutum Command Platform
- [@scutum/policy-engine](https://github.com/ScutumDefense/scutum-policy-engine) -- Sovereign policy evaluation engine
- [@scutum/audit-core](https://github.com/ScutumDefense/scutum-audit-core) -- Immutable audit chain library
- [MITRE ATT&CK Framework](https://attack.mitre.org/) -- Referenced in the Detection Rule Authoring pattern

</details>

---

## License

Apache 2.0 -- see [LICENSE](./LICENSE) for details.

Copyright 2026 Scutum Defense.
