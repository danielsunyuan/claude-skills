---
name: api-design
description: REST API design patterns, HTTP semantics, versioning strategies, and response formats. Use when designing APIs, reviewing endpoint structure, or establishing API conventions.
allowed-tools: Read, Grep, Glob
---

# API Design Patterns

Quick reference for REST API design conventions and best practices.

## HTTP Verbs

| Verb | Usage | Idempotent | Safe |
|------|-------|------------|------|
| GET | Retrieve resource(s) | Yes | Yes |
| POST | Create resource | No | No |
| PUT | Replace resource | Yes | No |
| PATCH | Partial update | No | No |
| DELETE | Remove resource | Yes | No |

## URI Patterns

```
# Resources (nouns, plural)
GET    /users              # List users
POST   /users              # Create user
GET    /users/{id}         # Get user
PUT    /users/{id}         # Replace user
PATCH  /users/{id}         # Update user
DELETE /users/{id}         # Delete user

# Nested resources
GET    /users/{id}/orders  # User's orders
POST   /users/{id}/orders  # Create order for user

# Actions (when CRUD doesn't fit)
POST   /users/{id}/activate
POST   /orders/{id}/cancel
```

## Naming Conventions

- Use **lowercase** with **hyphens**: `/user-profiles` not `/userProfiles`
- Use **plural nouns**: `/users` not `/user`
- Avoid verbs in URIs: `/users` not `/getUsers`
- Max nesting depth: 2 levels (`/users/{id}/orders`)
- Use query params for filtering: `/users?status=active&role=admin`

## Status Codes

### Success (2xx)
| Code | When to Use |
|------|-------------|
| 200 | Success with body |
| 201 | Resource created |
| 204 | Success, no body (DELETE) |

### Client Error (4xx)
| Code | When to Use |
|------|-------------|
| 400 | Bad request / validation error |
| 401 | Not authenticated |
| 403 | Not authorized |
| 404 | Resource not found |
| 409 | Conflict (duplicate) |
| 422 | Unprocessable entity |
| 429 | Rate limit exceeded |

### Server Error (5xx)
| Code | When to Use |
|------|-------------|
| 500 | Internal server error |
| 502 | Bad gateway |
| 503 | Service unavailable |

## Response Envelope

```json
{
  "data": { ... },
  "meta": {
    "page": 1,
    "per_page": 20,
    "total": 100,
    "total_pages": 5
  }
}
```

## Error Response

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed",
    "details": [
      {
        "field": "email",
        "message": "Invalid email format"
      }
    ],
    "request_id": "req_abc123"
  }
}
```

## Pagination

### Offset-based
```
GET /users?page=2&per_page=20

Response:
{
  "data": [...],
  "meta": {
    "page": 2,
    "per_page": 20,
    "total": 150,
    "total_pages": 8
  }
}
```

### Cursor-based (better for large datasets)
```
GET /users?cursor=eyJpZCI6MTAwfQ&limit=20

Response:
{
  "data": [...],
  "meta": {
    "next_cursor": "eyJpZCI6MTIwfQ",
    "has_more": true
  }
}
```

## Filtering & Sorting

```
# Filtering
GET /users?status=active&role=admin
GET /users?created_after=2024-01-01

# Sorting
GET /users?sort=created_at:desc
GET /users?sort=-created_at  # prefix minus for desc

# Field selection (sparse fieldsets)
GET /users?fields=id,name,email
```

## Versioning Strategies

### URI-based (recommended)
```
GET /api/v1/users
GET /api/v2/users
```

### Header-based
```
GET /api/users
Accept: application/vnd.api+json; version=1
```

## API Versioning Rules

- **Major version** (v1 → v2): Breaking changes only
- **Additive changes** are safe: new fields, new endpoints
- **Breaking changes**: removing fields, changing types, renaming
- **Deprecation**: 6-12 month notice with `Sunset` header

## Request/Response Headers

```
# Request
Content-Type: application/json
Accept: application/json
Authorization: Bearer <token>

# Response
Content-Type: application/json
X-Request-Id: req_abc123
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1640995200
```

## Bulk Operations

```json
POST /users/bulk
{
  "operations": [
    { "method": "create", "data": { "name": "Alice" } },
    { "method": "create", "data": { "name": "Bob" } }
  ]
}

Response:
{
  "results": [
    { "status": 201, "data": { "id": 1, "name": "Alice" } },
    { "status": 201, "data": { "id": 2, "name": "Bob" } }
  ],
  "summary": { "succeeded": 2, "failed": 0 }
}
```

## Async Operations

```
POST /reports/generate
→ 202 Accepted
{
  "job_id": "job_123",
  "status": "pending",
  "status_url": "/jobs/job_123"
}

GET /jobs/job_123
→ 200 OK
{
  "job_id": "job_123",
  "status": "completed",
  "result_url": "/reports/rpt_456"
}
```
