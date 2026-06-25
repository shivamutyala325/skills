# Team API Conventions

These conventions must be enforced in every generated or reviewed contract.

---

## 1. Security Schemes

### OpenAPI — Always include one of:

**JWT Bearer (default)**
```json
"securitySchemes": {
  "BearerAuth": {
    "type": "http",
    "scheme": "bearer",
    "bearerFormat": "JWT"
  }
}
```
Apply globally:
```json
"security": [{ "BearerAuth": [] }]
```

**API Key**
```json
"securitySchemes": {
  "ApiKeyAuth": {
    "type": "apiKey",
    "in": "header",
    "name": "X-API-Key"
  }
}
```

**OAuth2 (when user specifies OAuth)**
```json
"securitySchemes": {
  "OAuth2": {
    "type": "oauth2",
    "flows": {
      "clientCredentials": {
        "tokenUrl": "https://auth.example.com/oauth/token",
        "scopes": {
          "read:resources": "Read access",
          "write:resources": "Write access"
        }
      }
    }
  }
}
```

### AsyncAPI — Always include:
```json
"securitySchemes": {
  "saslScram": {
    "type": "scramSha256",
    "description": "SASL SCRAM-SHA-256 authentication for Kafka"
  }
}
```

---

## 2. Error Response Standard — RFC 7807 Problem Details

All error responses (4xx, 5xx) MUST use this schema:

```json
"ProblemDetails": {
  "type": "object",
  "required": ["type", "title", "status"],
  "properties": {
    "type": {
      "type": "string",
      "format": "uri",
      "description": "URI reference identifying the problem type",
      "example": "https://api.example.com/errors/validation-error"
    },
    "title": {
      "type": "string",
      "description": "Short, human-readable summary of the problem",
      "example": "Validation Error"
    },
    "status": {
      "type": "integer",
      "description": "HTTP status code",
      "example": 400
    },
    "detail": {
      "type": "string",
      "description": "Human-readable explanation of this specific occurrence",
      "example": "The 'email' field must be a valid email address"
    },
    "instance": {
      "type": "string",
      "format": "uri",
      "description": "URI reference identifying this specific occurrence",
      "example": "https://api.example.com/logs/request-12345"
    }
  }
}
```

Standard error responses to include on every endpoint:
- `400` — Validation / bad request
- `401` — Unauthorized (missing/invalid auth)
- `403` — Forbidden (insufficient permissions)
- `404` — Resource not found
- `422` — Unprocessable entity
- `500` — Internal server error

---

## 3. Pagination — Cursor-Based

All list/collection endpoints MUST use cursor-based pagination.

**Query parameters:**
```json
{
  "cursor": {
    "name": "cursor",
    "in": "query",
    "required": false,
    "schema": { "type": "string" },
    "description": "Opaque cursor for the next page, obtained from previous response"
  },
  "limit": {
    "name": "limit",
    "in": "query",
    "required": false,
    "schema": { "type": "integer", "minimum": 1, "maximum": 100, "default": 20 },
    "description": "Number of items to return per page"
  }
}
```

**Response envelope for paginated lists:**
```json
"PaginatedResponse": {
  "type": "object",
  "required": ["data", "pagination"],
  "properties": {
    "data": {
      "type": "array",
      "items": { "$ref": "#/components/schemas/YourResource" }
    },
    "pagination": {
      "type": "object",
      "required": ["hasMore"],
      "properties": {
        "hasMore": { "type": "boolean" },
        "nextCursor": { "type": "string", "nullable": true },
        "total": { "type": "integer", "description": "Total count if available" }
      }
    }
  }
}
```

---

## 4. Versioning Strategy — URL Path

All REST APIs MUST include the version in the URL path:

```
/v1/resources
/v2/resources
```

- `openapi.info.version` reflects the **API version** (e.g., `"1.0.0"`)
- `servers[].url` includes the version prefix: `"https://api.example.com/v1"`
- Breaking changes require a new major version (`/v2/`)
- Deprecation: mark old version endpoints with `deprecated: true` and add `x-sunset` header info

**Never use:**
- Header-based versioning (`Accept: application/vnd.api+json;version=1`)
- Query param versioning (`?version=1`)

---

## 5. General Naming Conventions

- Resource names: **plural nouns**, lowercase, hyphen-separated (`/user-profiles`, `/api-keys`)
- Operation IDs: **camelCase** verb+noun (`listUsers`, `createApiKey`, `deleteUserProfile`)
- Schema names: **PascalCase** (`UserProfile`, `ApiKeyRequest`)
- Property names: **camelCase** (`firstName`, `createdAt`, `isActive`)
- Kafka topic names: `<domain>.<entity>.<event>` (e.g., `infra.node.health-changed`)
- AsyncAPI channel names match Kafka topic names exactly