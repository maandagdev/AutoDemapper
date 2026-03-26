---
name: commit-strategy
description: "Determines optimal commit strategy based on migration size and complexity."
---

# Commit Strategy

Analyzes migration scope and recommends how to structure commits for review and rollback safety.

## When to Use
- Stage 7: Finalizing migration
- Planning PR structure

## Prerequisites
- migration.json with all migrations
- Statistics on mappings and complexity

## Procedure

### Step 1: Analyze Migration Scope

From migration.json:
```json
{
  "statistics": {
    "migrated": 42,
    "failed": 1,
    "skipped": 1,
    "callSitesTotal": 340
  }
}
```

### Step 2: Apply Decision Matrix

| Migrated | Call Sites | Recommended Strategy |
|----------|------------|---------------------|
| < 5 | < 20 | atomic |
| 5-15 | 20-100 | per-profile |
| 16-50 | 100-300 | per-profile |
| > 50 | > 300 | per-tier |
| Mixed success | Any | per-mapping |

### Step 3: Consider Risk Factors

**Prefer smaller commits if**:
- Multiple profiles with dependencies
- Complex mappings (hierarchy, converters)
- Large call site changes per mapping
- High-traffic code areas

**Prefer larger commits if**:
- All mappings are simple (leaf tier)
- Few call sites
- Isolated module

### Step 4: Generate Commit Plan

**Atomic** (single commit):
```json
{
  "commits": [
    {
      "message": "feat: migrate all AutoMapper mappings to manual implementations",
      "files": ["all changed files"],
      "mappingIds": ["all"]
    }
  ]
}
```

**Per-Profile**:
```json
{
  "commits": [
    {
      "message": "feat: migrate UserProfile mappings",
      "files": ["UserMapper.cs", "UserService.cs"],
      "mappingIds": ["uuid-1", "uuid-2"]
    },
    {
      "message": "feat: migrate OrderProfile mappings", 
      "files": ["OrderMapper.cs", "OrderService.cs"],
      "mappingIds": ["uuid-3", "uuid-4"]
    }
  ]
}
```

**Per-Tier**:
```json
{
  "commits": [
    {
      "message": "feat: migrate leaf-tier mappings (30 mappings)",
      "files": ["..."],
      "mappingIds": ["..."]
    },
    {
      "message": "feat: migrate converter-tier mappings (5 mappings)",
      "files": ["..."],
      "mappingIds": ["..."]
    }
  ]
}
```

**Per-Mapping** (most granular):
```json
{
  "commits": [
    {
      "message": "feat: migrate User to UserDto mapping",
      "files": ["UserMapper.cs"],
      "mappingIds": ["uuid-1"]
    }
  ]
}
```

### Step 5: Generate Commit Messages

**Template**:
```
{type}: {scope} - {description}

{body}

Mapping ID(s): {ids}
Tests: Characterization tests passing
```

**Examples**:
```
feat: migrate UserProfile to manual mapping

- Created UserMapper with ToDto() extension method
- Updated 15 call sites in UserService and UserController
- All characterization tests passing

Mapping IDs: uuid-1, uuid-2, uuid-3
```

## Strategy Definitions

| Strategy | Description | Rollback Scope |
|----------|-------------|----------------|
| atomic | All changes in one commit | All or nothing |
| per-profile | Group by Profile class | Per profile |
| per-tier | Group by complexity | Per tier |
| per-mapping | One mapping per commit | Individual |

## Output

```json
{
  "commitStrategy": "per-profile",
  "rationale": "15 mappings across 3 profiles with clear boundaries",
  "commits": [
    {
      "message": "...",
      "files": ["..."],
      "mappingIds": ["..."]
    }
  ]
}
```

## Special Cases

### Partial Migration
If some mappings failed:
- Use per-mapping strategy
- Successful mappings get their own commits
- Failed mappings remain with AutoMapper

### Dependencies Between Profiles
If ProfileA depends on ProfileB:
- Commit ProfileB first
- Then ProfileA
- Note dependency in commit message
