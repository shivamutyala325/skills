# OpenAPI Review Checklist

Apply to OpenAPI 3.0 and 3.1 contracts.

---

## 1. Structure & Validity

- [ ] `openapi` field present and valid (`3.0.x` or `3.1.x`)
- [ ] `info` object has `title`, `version`, and `description`
- [ ] At least one `servers` entry with a real URL
- [ ] All `$ref` references resolve to existing components
- [ ] No duplicate operation IDs
- [ ] All schema `required` fields reference properties that exist

## 2. Security

- [ ] `securitySchemes` defined in `components`
- [ ] Global `security` applied at root level
- [ ] Endpoints that should be public explicitly override with `security: []`
- [ ] No credentials, tokens, or secrets in example values
- [ ] OAuth2 scopes are defined and applied to operations
- [ ] Sensitive endpoints (write, delete, admin) have stricter security than read endpoints

## 3. Error Responses (RFC 7807)

- [ ] All operations define `400`, `401`, `403`, `404`, `500` responses
- [ ] All error responses reference the `ProblemDetails` schema (or equivalent)
- [ ] No bare `{}` or empty error response schemas
- [ ] `422` present on endpoints that accept request bodies

## 4. Pagination

- [ ] All list/collection endpoints (`GET /resources`) have `cursor` and `limit` query params
- [ ] Response schema for lists uses the `PaginatedResponse` envelope
- [ ] `hasMore` and `nextCursor` fields present in pagination object
- [ ] No offset-based pagination (`page`, `offset`, `skip` params)

## 5. Versioning

- [ ] `servers[].url` includes version prefix (`/v1/`)
- [ ] `info.version` reflects the API version
- [ ] Deprecated endpoints marked with `deprecated: true`
- [ ] No version information in individual path definitions (should be in server URL)

## 6. Schema Quality

- [ ] All request body schemas have `required` arrays defined
- [ ] All properties have `type` defined (no bare `{}` schemas)
- [ ] String fields have `maxLength` where user input is accepted
- [ ] Numeric fields have `minimum`/`maximum` where applicable
- [ ] Date/time fields use `format: date-time` (ISO 8601)
- [ ] Enum values are documented with descriptions
- [ ] `nullable` / `oneOf: [type, null]` used correctly for optional fields
- [ ] Reused schemas use `$ref`, not inline repetition

## 7. Operations

- [ ] Every operation has an `operationId` (camelCase verb+noun)
- [ ] Every operation has a `summary` and `description`
- [ ] `tags` used to group related operations
- [ ] Path parameters defined both in path string and `parameters` array
- [ ] `GET` operations do not have request bodies
- [ ] `DELETE` operations return `204 No Content` or `200` with confirmation
- [ ] `POST` for create returns `201 Created` with the created resource
- [ ] `PUT`/`PATCH` distinction is correct (full replace vs partial update)

## 8. Naming Conventions

- [ ] Paths use plural nouns, lowercase, hyphen-separated
- [ ] Operation IDs are camelCase
- [ ] Schema names are PascalCase
- [ ] Property names are camelCase
- [ ] No snake_case in paths or property names