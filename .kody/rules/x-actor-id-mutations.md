---
title: "Mutation endpoints must require X-Actor-Id header"
scope: "file"
path: ["**/openapi.yaml"]
severity_min: "critical"
buckets: ["security"]
enabled: true
---

## Instructions

All mutation endpoints (POST, PUT, PATCH, DELETE) must declare `X-Actor-Id` as a required header parameter for audit trail compliance.

Flag:
- Mutation operations missing `X-Actor-Id` in their `parameters` list
- `X-Actor-Id` declared but not marked as `required: true`

Exceptions:
- Health/readiness endpoints (`/health`, `/ready`)

## Examples

### Bad example
```yaml
/patients:
  post:
    operationId: registerPatient
    # Missing X-Actor-Id parameter
    requestBody: ...
```

### Good example
```yaml
/patients:
  post:
    operationId: registerPatient
    parameters:
      - name: X-Actor-Id
        in: header
        required: true
        schema:
          type: string
          format: uuid
    requestBody: ...
```
