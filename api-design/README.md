# API Design Guidelines

This guide defines the conventions for designing HTTP APIs across all Scutum services. Consistent API design reduces integration friction, simplifies documentation, and makes the platform predictable for operators and developers.

---

## RESTful Conventions

### Resource naming

- Use plural nouns for collections: `/incidents`, `/assets`, `/policies`
- Use singular identifiers for specific resources: `/incidents/{incidentId}`
- Use kebab-case for multi-word resources: `/audit-entries`, `/detection-rules`
- Nest resources to express ownership: `/incidents/{incidentId}/recommendations`
- Limit nesting to two levels. Beyond that, use top-level resources with query filters.

### HTTP methods

| Method | Usage | Idempotent |
|---|---|---|
| `GET` | Retrieve a resource or collection | Yes |
| `POST` | Create a new resource | No |
| `PUT` | Replace an entire resource | Yes |
| `PATCH` | Partially update a resource | No |
| `DELETE` | Remove a resource | Yes |

Never use `GET` for operations with side effects. Never use `POST` when `PUT` or `PATCH` is appropriate.

### URL parameters

- Path parameters for resource identification: `/incidents/{incidentId}`
- Query parameters for filtering, sorting, and pagination: `/incidents?severity=critical&sort=-createdAt`
- Never put sensitive data in URL parameters (they appear in logs and browser history)

---

## Response Envelope

All Scutum API responses use a standard envelope format.

### Success response (single resource)

```json
{
  "data": {
    "id": "inc-001",
    "title": "Unauthorized access detected",
    "severity": "critical"
  },
  "meta": {
    "requestId": "req-abc-123",
    "timestamp": "2026-04-08T12:00:00Z"
  }
}
```

### Success response (collection)

```json
{
  "data": [
    { "id": "inc-001", "title": "Unauthorized access detected" },
    { "id": "inc-002", "title": "Anomalous network traffic" }
  ],
  "meta": {
    "requestId": "req-abc-456",
    "timestamp": "2026-04-08T12:00:00Z"
  },
  "pagination": {
    "page": 1,
    "pageSize": 20,
    "totalItems": 142,
    "totalPages": 8,
    "nextCursor": "eyJpZCI6Imlu..."
  }
}
```

### Key rules

- `data` is always present and contains the primary payload (object or array).
- `meta` contains request-level metadata: `requestId`, `timestamp`, and optional `version`.
- `pagination` is present only for collection endpoints that support pagination.
- Never wrap scalar values. If the response is a single string or number, reconsider the API design.

---

## Error Response Format

All errors use a consistent structure with machine-readable error codes.

```json
{
  "error": {
    "code": "INCIDENT_NOT_FOUND",
    "message": "No incident found with ID inc-999",
    "details": [
      {
        "field": "incidentId",
        "reason": "The specified incident does not exist or has been archived"
      }
    ],
    "requestId": "req-abc-789",
    "timestamp": "2026-04-08T12:00:00Z"
  }
}
```

### Error code conventions

- Use UPPER_SNAKE_CASE for error codes: `VALIDATION_FAILED`, `UNAUTHORIZED`, `RATE_LIMIT_EXCEEDED`
- Prefix with the domain when specific: `INCIDENT_NOT_FOUND`, `POLICY_EVALUATION_FAILED`
- Generic codes for common situations: `VALIDATION_FAILED`, `NOT_FOUND`, `INTERNAL_ERROR`
- Never expose stack traces or internal implementation details in production error responses

### HTTP status codes

| Status | Usage |
|---|---|
| `200` | Success (GET, PUT, PATCH) |
| `201` | Created (POST) |
| `204` | No content (DELETE) |
| `400` | Bad request / validation error |
| `401` | Authentication required |
| `403` | Forbidden (authenticated but insufficient permissions) |
| `404` | Resource not found |
| `409` | Conflict (e.g., duplicate creation) |
| `422` | Unprocessable entity (semantic validation failure) |
| `429` | Rate limit exceeded |
| `500` | Internal server error |
| `503` | Service unavailable |

---

## Pagination

Support two pagination strategies. Clients choose which to use.

### Cursor-based pagination (preferred)

```
GET /incidents?cursor=eyJpZCI6Imlu...&pageSize=20
```

The `cursor` is an opaque token returned in the previous response's `pagination.nextCursor`. Cursor pagination is required for datasets that change frequently.

### Offset-based pagination

```
GET /incidents?page=3&pageSize=20
```

Offset pagination is acceptable for small, stable datasets (e.g., static configuration lists). It is not suitable for large, frequently changing collections.

### Default page size

- Default: 20 items
- Maximum: 100 items
- The API must reject `pageSize` values above the maximum with a `400` response

---

## Versioning Strategy

### URL path versioning

All Scutum APIs are versioned in the URL path:

```
/api/v1/incidents
/api/v2/incidents
```

### Version lifecycle

- **v1**: Stable, production-ready. Breaking changes require a new version.
- **v2-beta**: Pre-release. May change without notice. Documented as unstable.
- Deprecated versions are supported for at least 12 months after the successor reaches GA.
- Sunset headers are returned on deprecated endpoints: `Sunset: Sat, 01 Jan 2028 00:00:00 GMT`

### What constitutes a breaking change

- Removing a field from a response
- Changing a field's type
- Renaming a field
- Adding a required field to a request body
- Changing the semantics of an existing field

Non-breaking changes (additive fields, new optional parameters) do not require a version bump.

---

## Authentication

### Bearer token

All authenticated endpoints require a Bearer token in the `Authorization` header:

```
Authorization: Bearer <jwt-token>
```

### API key (service-to-service)

Service-to-service communication uses API keys in the `X-Scutum-Api-Key` header:

```
X-Scutum-Api-Key: <api-key>
```

### Token requirements

- JWTs must be validated for signature, expiration, issuer, and audience
- Tokens must include the `sub` (subject), `iss` (issuer), `aud` (audience), and `exp` (expiration) claims
- Token lifetime must not exceed 1 hour for user tokens and 24 hours for service tokens

---

## Rate Limiting

All APIs must implement rate limiting and communicate limits via standard headers:

```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 847
X-RateLimit-Reset: 1712592000
Retry-After: 30
```

- `X-RateLimit-Limit`: Maximum requests per window
- `X-RateLimit-Remaining`: Requests remaining in the current window
- `X-RateLimit-Reset`: Unix timestamp when the window resets
- `Retry-After`: Seconds to wait before retrying (only on `429` responses)

Rate limits are per-tenant by default. Document per-endpoint limits when they differ from the default.

---

## Health Check Endpoint

Every service must expose a health check at `GET /health`:

```json
{
  "status": "healthy",
  "version": "1.4.2",
  "uptime": 86400,
  "checks": {
    "database": "healthy",
    "cache": "healthy",
    "externalApi": "degraded"
  }
}
```

### Requirements

- Must respond within 5 seconds
- Must not require authentication
- Must return `200` when healthy, `503` when unhealthy
- Must include individual dependency checks in `checks`
- Must include the service version for operational debugging
- Must not expose sensitive configuration or secrets
