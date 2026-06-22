# RFC 002: Standardized Error Response Shape

**Status:** Draft
**Author:** Platform Team
**Created:** 2026-02-03

## Problem

Error responses are inconsistent across services:

- Some return `{ error: "message" }`
- Some return `{ message: "..." }`
- Some return bare HTTP status with no body
- Validation errors have no standard field-level detail format

## Proposal

All error responses MUST conform to:

```typescript
interface ErrorResponse {
  code: string;       // machine-readable, SCREAMING_SNAKE_CASE
  message: string;    // human-readable, suitable for logs
  details?: FieldError[];
  requestId?: string; // for log correlation
}

interface FieldError {
  field: string;      // dot-path (e.g. "user.email")
  code: string;       // e.g. "REQUIRED", "INVALID_FORMAT"
  message: string;
}
```

### Examples

**Validation failure (422):**
```json
{
  "code": "VALIDATION_FAILED",
  "message": "Request body did not pass validation.",
  "details": [
    { "field": "email", "code": "INVALID_FORMAT", "message": "Must be a valid email address." },
    { "field": "password", "code": "TOO_SHORT", "message": "Must be at least 12 characters." }
  ]
}
```

**Not found (404):**
```json
{
  "code": "RESOURCE_NOT_FOUND",
  "message": "User with ID 42 does not exist.",
  "requestId": "req_8k2nPqLm"
}
```

## HTTP Status Code Mapping

| Situation | Status |
|-----------|--------|
| Validation failed | 422 |
| Authentication required | 401 |
| Insufficient permissions | 403 |
| Resource not found | 404 |
| Business rule violation | 409 |
| Server error | 500 |

## Migration Path

1. Add an error-response middleware that wraps thrown errors in the new shape
2. Update per-endpoint error handling to throw typed errors
3. Ship an updated SDK with typed error classes
