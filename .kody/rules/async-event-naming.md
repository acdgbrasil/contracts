---
title: "Async events must describe completed facts"
scope: "file"
path: ["**/asyncapi.yaml"]
severity_min: "medium"
buckets: ["style-conventions"]
enabled: true
---

## Instructions

AsyncAPI event/channel names must describe facts that already happened (past tense), following event sourcing conventions.

Rules:
- Channel names: `<aggregate>.<past-tense-verb>` (e.g., `patient.registered`, `family-member.added`)
- All events must carry metadata: `eventId`, `occurredAt`, `schemaVersion`, `actorId`
- Event payloads must reference canonical schemas via `$ref`

Flag:
- Imperative names (`patient.register`, `create-patient`)
- Present tense (`patient.registering`)
- Missing metadata fields in event payload
- Missing `actorId` in event payload

## Examples

### Bad example
```yaml
channels:
  patient/register:  # Imperative, not past tense
    messages:
      registerPatient:
        payload:
          $ref: '../model/schemas/Patient.yaml'  # Missing metadata
```

### Good example
```yaml
channels:
  patient/registered:
    messages:
      patientRegistered:
        payload:
          type: object
          properties:
            eventId:
              type: string
              format: uuid
            occurredAt:
              type: string
              format: date-time
            schemaVersion:
              type: integer
            actorId:
              type: string
            data:
              $ref: '../model/schemas/PatientRegisteredEvent.yaml'
```
