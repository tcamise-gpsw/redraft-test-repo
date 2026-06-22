# RFC 001: Standard Pagination Envelope

**Status:** Accepted
**Author:** Platform Team
**Created:** 2026-01-15

## Problem

Different list endpoints return results in different shapes:

- `/users` returns `{ users: [...] }`
- `/posts` returns `[...]` (bare array)
- `/comments` returns `{ items: [...], total: N }`

This forces clients to special-case each endpoint.

## Proposal

All list endpoints MUST return a consistent pagination envelope:

```json
{
  "data": [...],
  "pagination": {
    "page": 1,
    "perPage": 20,
    "total": 142,
    "totalPages": 8,
    "nextCursor": "eyJpZCI6MTIzfQ=="
  }
}
```

### Cursor vs. offset pagination

| Strategy | Use when |
|----------|----------|
| Offset (`page` + `perPage`) | Total count needed for UI, small datasets |
| Cursor (`nextCursor`) | Large / frequently-updated datasets, infinite scroll |

Endpoints MAY support both. `nextCursor` is `null` on the last page.

## Migration

1. Add the new envelope shape to the response under a versioned path (`/v2/...`)
2. Deprecate the old shape in `/v1/` with `X-Deprecation-Date` headers
3. Remove after the deprecation window (minimum 90 days)

## Drawbacks

- Increases response payload size slightly
- Requires client-side migration work

## Alternatives Considered

- **Link header pagination** (RFC 5988) — not JSON-native, harder to auto-generate SDK code
- **GraphQL connections** — too large a change for a REST API
