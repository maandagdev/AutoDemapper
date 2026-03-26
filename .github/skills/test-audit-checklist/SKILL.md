---
name: test-audit-checklist
description: "Verifies characterization test coverage completeness for each mapping before migration."
---

# Test Audit Checklist

Systematically verifies that each AutoMapper mapping has sufficient characterization test coverage.

## When to Use
- After test generation, before establishing baseline
- Reviewing test coverage gaps
- Validating readiness for migration

## Prerequisites
- inventory.json with all mappings
- tests.json with generated tests

## Procedure

### Step 1: Load Mappings and Tests
Cross-reference inventory.json mappings with tests.json entries using `mappingId`.

### Step 2: Check Required Coverage

For **every mapping**, verify:

| Check | Requirement | Severity |
|-------|-------------|----------|
| Basic test exists | At least one test method | Critical |
| Properties covered | Assertions for mapped properties | High |
| Null handling | Test for null source | High |

### Step 3: Check Conditional Coverage

Based on mapping features:

| Feature | Required Test | Severity |
|---------|--------------|----------|
| Collection property | Collection mapping test | High |
| Nested complex type | Nested object test | Medium |
| ForMember with MapFrom | Custom logic assertion | Medium |
| Ignore | Verify property is default/null | Low |
| ConvertUsing | Converter behavior test | High |
| AfterMap/BeforeMap | Hook execution test | High |
| Condition | Conditional logic test | Medium |

### Step 4: Generate Gap Report

For each gap found:
```json
{
  "mappingId": "uuid",
  "issue": "missing_collection_test",
  "recommendation": "Add test for Items collection mapping",
  "severity": "high"
}
```

### Step 5: Calculate Coverage Metrics

```
coverage = mappingsWithTests / totalMappings

criticalGaps = gaps.filter(g => g.severity === 'critical')
highGaps = gaps.filter(g => g.severity === 'high')

readyForBaseline = (
  criticalGaps.length === 0 &&
  coverage >= 1.0
)
```

### Step 6: Determine Readiness

**Ready** if:
- Every mapping has at least one test
- No critical gaps
- Coverage = 100%

**Not Ready** if:
- Any mapping has no tests (critical)
- Any critical severity gap exists

## Checklist Template

For each mapping, check:

```markdown
## {SourceType} → {DestType} ({ProfileClass})

- [ ] Basic mapping test exists
- [ ] All mapped properties have assertions
- [ ] Null source behavior tested
- [ ] Collections tested (if applicable)
- [ ] Nested objects tested (if applicable)
- [ ] Custom ForMember logic tested (if applicable)
- [ ] Ignored properties verified (if applicable)
- [ ] Converter behavior tested (if applicable)
- [ ] AfterMap/BeforeMap hooks tested (if applicable)
```

## Output

```json
{
  "coverage": {
    "mappingsWithTests": 40,
    "mappingsWithoutTests": 2,
    "estimatedCoverage": 0.95
  },
  "gaps": [...],
  "readyForBaseline": false,
  "blockers": ["2 mappings missing required tests"],
  "warnings": ["5 mappings missing optional edge case tests"]
}
```

## Severity Definitions

| Severity | Description | Action |
|----------|-------------|--------|
| Critical | No test exists | Must fix before baseline |
| High | Key scenario untested | Should fix before baseline |
| Medium | Edge case untested | Fix recommended |
| Low | Nice-to-have coverage | Optional |
