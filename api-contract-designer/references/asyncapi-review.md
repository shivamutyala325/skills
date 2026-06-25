# AsyncAPI Review Checklist

Apply to AsyncAPI 2.x and 3.x contracts.

---

## 1. Structure & Validity

- [ ] `asyncapi` field present and valid (`2.x.x` or `3.x.x`)
- [ ] `info` object has `title`, `version`, and `description`
- [ ] `servers` block defines at least one broker with `url` and `protocol`
- [ ] All `$ref` references resolve to existing components
- [ ] `channels` (2.x) or `channels` + `operations` (3.x) are defined
- [ ] Schema is consistent with declared AsyncAPI version

## 2. Security

- [ ] `securitySchemes` defined in `components`
- [ ] Security applied at server level
- [ ] SASL/SCRAM or mTLS defined for Kafka brokers
- [ ] No credentials or secrets in example payloads
- [ ] Sensitive topics have security requirements explicitly documented

## 3. Channel & Topic Naming (Kafka)

- [ ] Channel names follow `<domain>.<entity>.<event>` convention
- [ ] Channel names are lowercase with hyphens (no underscores, no camelCase)
- [ ] Channel names match Kafka topic names exactly
- [ ] Channel `description` explains the purpose and consumers of the topic
- [ ] No generic names like `events`, `messages`, `data`

## 4. Message Schema Quality

- [ ] Every message has a `name` and `title`
- [ ] Every message has a `payload` schema defined (not `{}`)
- [ ] Message `payload` uses `$ref` to a named schema in `components`
- [ ] All payload properties have `type` defined
- [ ] String fields have `maxLength` for user-generated content
- [ ] Date/time fields use `format: date-time`
- [ ] `required` arrays defined on all payload objects
- [ ] `examples` provided for each message
- [ ] `contentType` set (default `application/json`)

## 5. Operations

### AsyncAPI 2.x
- [ ] Each channel has either `subscribe`, `publish`, or both operations defined
- [ ] Operation `summary` and `description` present
- [ ] `operationId` defined on each operation (camelCase)
- [ ] `tags` used to group related channels

### AsyncAPI 3.x
- [ ] `operations` block defined separately from `channels`
- [ ] Each operation has `action` (`send` or `receive`)
- [ ] Each operation references a valid `channel`
- [ ] `operationId` defined on each operation

## 6. Headers & Bindings

- [ ] Kafka-specific bindings defined where needed (`bindings.kafka`)
- [ ] Kafka message key defined in bindings if used for partitioning
- [ ] Consumer group information documented in channel bindings
- [ ] Message headers schema defined if custom headers are used
- [ ] Retention and partition info documented in `x-` extensions if relevant

## 7. Correlation & Tracing

- [ ] `correlationId` defined on messages that are part of request/reply patterns
- [ ] Trace ID header documented for distributed tracing
- [ ] Event envelope includes `eventId`, `eventType`, `timestamp`, `source` fields

## 8. Versioning & Evolution

- [ ] `info.version` reflects the contract version
- [ ] Breaking schema changes noted with deprecation notices
- [ ] Old message versions kept in `components` with `x-deprecated` extension
- [ ] Topic versioning strategy documented (new topic vs schema evolution)