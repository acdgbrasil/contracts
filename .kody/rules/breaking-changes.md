---
title: "Flag breaking changes in existing schemas"
scope: "pull_request"
severity_min: "critical"
buckets: ["architecture"]
enabled: true
---

## Instructions

Scan `pr_files_diff` for breaking changes in existing schemas that could break consumers (social-care backend, frontend BFF, admin-painel).

Breaking changes to flag:
- Removing or renaming existing properties in schemas
- Changing property types (e.g., `string` → `integer`)
- Making previously optional properties required
- Removing or renaming endpoints (paths)
- Changing HTTP methods on existing endpoints
- Removing or renaming async events/channels
- Changing `operationId` on existing endpoints

Non-breaking (allowed):
- Adding new optional properties to existing schemas
- Adding new endpoints or events
- Adding new schemas
- Deprecating (but not removing) existing fields

When flagged, suggest: "Consider adding a new schema version or deprecating the old field before removing."
