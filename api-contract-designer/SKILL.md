---
name: api-contract-designer
description: >
  Use this skill whenever a user wants to design, generate, review, or lint an API contract.
  Triggers include: "design an API", "write an OpenAPI spec", "create a Swagger contract",
  "generate an AsyncAPI schema", "review this API spec", "lint my OpenAPI", "check my AsyncAPI",
  "write a contract for this endpoint", "define an event schema", "create an API definition",
  "help me document this API", or when a user describes an API or event-driven system and wants
  a formal contract for it. Covers OpenAPI 3.0/3.1 (REST) and AsyncAPI 2.x/3.x (Kafka/MQTT/event-driven).
  Always outputs valid JSON contracts. Apply this skill even if the user just describes an API
  informally — if it sounds like they need a contract or spec, use it.
---

# API Contract Designer Skill

Generates and reviews **OpenAPI 3.0/3.1** (REST) and **AsyncAPI 2.x/3.x** (event-driven/Kafka/MQTT)
contracts in **JSON format**, enforcing team conventions for security, error handling, pagination,
and versioning.

---

## Mode Detection

Determine the mode from the user's request:

| User says... | Mode |
|---|---|
| Describes an API, endpoints, events in plain language | **GENERATE** |
| Pastes an existing spec / contract | **REVIEW** |
| Asks for both | **GENERATE then REVIEW** |

---

## Mode 1: GENERATE

### Step 1 — Identify Contract Type
- Mentions REST, HTTP, endpoints, CRUD → **OpenAPI**
- Mentions Kafka, events, pub/sub, topics, MQTT, message broker → **AsyncAPI**
- Mixed system → generate both, clearly separated

### Step 2 — Detect Version
- User specifies version → use it
- Not specified → default to **OpenAPI 3.1** / **AsyncAPI 3.x**
- Existing codebase context suggests older version → match it, note the choice

### Step 3 — Extract Contract Details from Description

Ask clarifying questions if critical info is missing:
- **OpenAPI**: base path, auth method, key resources, CRUD operations needed
- **AsyncAPI**: broker type (Kafka/MQTT), topics/channels, message payload structure, publish vs subscribe

Do NOT ask for info that can be reasonably inferred. Make sensible defaults and note them.

### Step 4 — Apply Team Conventions (always enforce)

Load `references/conventions.md` for full details. Summary:
- **Security**: include appropriate security scheme (OAuth2, API key, or JWT Bearer)
- **Errors**: all error responses use RFC 7807 Problem Details format
- **Pagination**: cursor-based pagination on all list endpoints
- **Versioning**: URL path versioning (`/v1/`, `/v2/`) for OpenAPI

### Step 5 — Output the Contract

Output a complete, valid JSON contract. Structure:
1. Brief plain-language summary (2–3 sentences) of what the contract covers
2. The full JSON contract in a code block
3. A **Decisions & Defaults** section noting any assumptions made

---

## Mode 2: REVIEW

### Step 1 — Detect Contract Type & Version
Parse the input to identify: OpenAPI 3.0 / 3.1 / AsyncAPI 2.x / 3.x.
State what was detected at the top of the review.

### Step 2 — Run the Review Checklist

Load the relevant checklist:
- OpenAPI → `references/openapi-review.md`
- AsyncAPI → `references/asyncapi-review.md`

### Step 3 — Output Review Comments

Use this format for each issue found:

```
**[SEVERITY] path.to.field**
**Category:** <category>
**Issue:** <what's wrong and why it matters>
**Fix:**
```json
<corrected JSON snippet>
```
```

Severity levels:
- 🔴 **CRITICAL** — Invalid spec, missing required fields, security exposure
- 🟠 **WARNING** — Convention violation, missing best practice, interoperability risk
- 🟡 **SUGGESTION** — Improvement to clarity, completeness, or consistency

End with a **Summary Table**:

| Severity | Count |
|----------|-------|
| 🔴 Critical | N |
| 🟠 Warning | N |
| 🟡 Suggestion | N |
| **Total** | **N** |

And a **Contract Health**: ✅ Valid & complete / ⚠️ Valid with gaps / 🚫 Invalid — fix before use

---

## Output Rules

- Always output JSON (not YAML)
- Contracts must be complete and valid — not skeletons or pseudocode
- Include realistic example values in all `example` fields
- Use `$ref` for reused schemas (don't repeat inline)
- All string fields should have `maxLength` where appropriate
- Required fields must be explicitly listed in `required` arrays

---

## Edge Cases

- **Vague description**: Generate a minimal but valid contract, note what's assumed, ask for clarification at the end
- **Mixed REST + events**: Generate OpenAPI and AsyncAPI separately, note shared schemas
- **Breaking changes detected in review**: Flag with 🔴 CRITICAL and explain the backward-compatibility impact
- **Unknown broker type**: Default to Kafka for AsyncAPI unless specified
- **No auth mentioned**: Default to Bearer JWT, note the assumption