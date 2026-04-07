---
title: "OpenAPI endpoints must have explicit operationId"
scope: "file"
path: ["**/openapi.yaml"]
severity_min: "high"
buckets: ["style-conventions"]
enabled: true
---

## Instructions

Every operation in the OpenAPI spec must have an explicit, action-oriented `operationId`. This is used for code generation and documentation.

Rules:
- `operationId` must be present on every path operation
- Must be camelCase
- Must be action-oriented (verb + noun): `registerPatient`, `listAppointments`, `updateHousingCondition`
- Must be unique across the entire spec

Flag:
- Missing `operationId`
- Generic names like `get`, `post`, `update`
- Non-camelCase operationIds

## Examples

### Bad example
```yaml
/patients:
  get:
    summary: List patients
    # Missing operationId
```

### Good example
```yaml
/patients:
  get:
    operationId: listPatients
    summary: List patients
```
