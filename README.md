# contracts

Repository dedicated to API contracts.

## Rules

- Synchronous integrations must use OpenAPI.
- Asynchronous integrations must use AsyncAPI.
- Each service owns a canonical domain model under `model/schemas`.
- OpenAPI and AsyncAPI contracts must reuse the canonical model via `$ref`.
- Avoid duplicating schemas between sync and async contracts.

## Structure

```text
contracts/
  services/
    service-a/
      model/schemas/
      openapi/openapi.yaml
      asyncapi/asyncapi.yaml
    service-b/
      model/schemas/
      openapi/openapi.yaml
      asyncapi/asyncapi.yaml
    service-c/
      model/schemas/
      openapi/openapi.yaml
      asyncapi/asyncapi.yaml
  shared/
    schemas/
      Error.yaml
      Pagination.yaml
```

## Naming Conventions

- `operationId` in OpenAPI should be explicit and action-oriented.
- Async event names should describe completed facts (for example: `resource.created`).
- Keep `info.version` in contract files aligned with release/versioning strategy.

## How To Add A New Service

1. Create `services/<service-name>/model/schemas`.
2. Create `services/<service-name>/openapi/openapi.yaml`.
3. Create `services/<service-name>/asyncapi/asyncapi.yaml`.
4. Reference schemas from `model/schemas` in both API specs using `$ref`.
5. Reuse common schemas from `shared/schemas` when applicable.
