---
name: State File Conventions
description: Rules for reading and writing AutoDemapper pipeline state files. Each file has a schema and a single owning agent.
applyTo: '.github/state/*.json'
---

# State File Rules

## Schemas

Every state file has a JSON Schema. Validate output against it before writing:

| File | Schema |
|------|--------|
| `inventory.json` | `.github/state/schemas/inventory.schema.json` |
| `tests.json` | `.github/state/schemas/tests.schema.json` |
| `audit.json` | `.github/state/schemas/audit.schema.json` |
| `baseline.json` | `.github/state/schemas/baseline.schema.json` |
| `migration.json` | `.github/state/schemas/migration.schema.json` |
| `validation.json` | `.github/state/schemas/validation.schema.json` |
| `report.json` | `.github/state/schemas/report.schema.json` |

## Ownership — Do Not Cross

Each state file is owned by exactly one agent. An agent must **never** write to another agent's file:

| File | Owner |
|------|-------|
| `inventory.json` | `inventory-agent` |
| `tests.json` | `test-generator-agent` |
| `audit.json` | `audit-agent` |
| `baseline.json` | `baseline-agent` |
| `migration.json` | `migration-agent` |
| `validation.json` | `validation-agent` |
| `report.json` | `reporter-agent` |

## Required Fields

Every state file MUST include:

```json
{
  "version": "1.0",
  "timestamp": "2026-03-26T10:00:00Z"
}
```

## Write Atomically

Write the complete file in a single operation. Never write partial state (e.g., appending results one-by-one).

## Error Format

When recording a failure, use the standard error object:

```json
{
  "error": "error_code",
  "message": "Human readable description",
  "details": "Technical details",
  "recoverable": true
}
```
