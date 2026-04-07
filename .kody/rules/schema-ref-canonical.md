---
title: "Schemas must use $ref to canonical models, never inline duplicates"
scope: "file"
path: ["**/openapi.yaml", "**/asyncapi.yaml"]
severity_min: "critical"
buckets: ["architecture"]
enabled: true
---

## Instructions

OpenAPI and AsyncAPI specs must reference canonical schemas via `$ref` from `model/schemas/` or `shared/schemas/`. Never inline or duplicate schema definitions.

Flag:
- Inline `properties:` definitions in endpoint request/response bodies that duplicate an existing schema
- Schema definitions in the spec file that should live in `model/schemas/`
- Missing `$ref` where a canonical schema exists
- Duplicate schemas between OpenAPI and AsyncAPI specs (both must reference the same canonical file)

Allowed:
- `$ref: '../model/schemas/SomeSchema.yaml'`
- `$ref: '../../shared/schemas/Error.yaml'`
- Small inline `allOf`/`oneOf` compositions that combine canonical refs

## Examples

### Bad example
```yaml
# Duplicating Patient schema inline instead of referencing
paths:
  /patients:
    post:
      requestBody:
        content:
          application/json:
            schema:
              type: object
              properties:
                name:
                  type: string
                cpf:
                  type: string
```

### Good example
```yaml
paths:
  /patients:
    post:
      requestBody:
        content:
          application/json:
            schema:
              $ref: '../model/schemas/RegisterPatientRequest.yaml'
```
