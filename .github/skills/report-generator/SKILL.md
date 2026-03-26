---
name: report-generator
description: "Generates comprehensive migration report with statistics, outcomes, and recommendations."
---

# Report Generator

Creates the final migration report summarizing all stages and outcomes.

## When to Use
- Stage 7: Final reporting
- Generating PR description
- Creating audit trail

## Prerequisites
- All state files from pipeline stages

## Procedure

### Step 1: Aggregate Statistics

**From inventory.json**:
- Total mappings discovered
- Profiles processed
- Complexity distribution

**From tests.json**:
- Tests generated
- Coverage per mapping

**From migration.json**:
- Successfully migrated
- Failed migrations
- Call sites updated

**From validation.json**:
- Test pass rate
- Regressions found

### Step 2: Calculate Summary Metrics

```javascript
summary = {
  totalMappings: inventory.mappings.length,
  successfullyMigrated: migration.statistics.migrated,
  requiresManualWork: migration.statistics.failed + migration.statistics.skipped,
  testsWritten: tests.statistics.testsGenerated,
  callSitesUpdated: migration.statistics.callSitesTotal,
  profilesProcessed: inventory.statistics.profileCount,
  duration: calculateDuration(inventory.timestamp, report.timestamp)
}
```

### Step 3: Determine AutoMapper Removal Readiness

```javascript
automapperRemovalReady = (
  summary.requiresManualWork === 0 &&
  validation.validationPassed === true &&
  validation.regressions.length === 0
)
```

### Step 4: Generate Blockers List

If not ready, list blockers:
```javascript
blockers = [
  ...migration.migrations
    .filter(m => m.status === 'failed')
    .map(m => `${m.mappingId}: ${m.failureReason}`),
  ...validation.regressions
    .map(r => `Regression in ${r.testName}: ${r.investigation}`)
]
```

### Step 5: Generate Recommendations

Based on analysis:

| Condition | Recommendation Type | Description |
|-----------|--------------------| ------------|
| All migrated | cleanup | Remove AutoMapper package |
| Large call site changes | maintainability | Consider facade pattern |
| Complex converters | performance | Profile hot paths |
| Missing tests fixed | testing | Add integration tests |

### Step 6: Format Report

```json
{
  "version": "1.0",
  "timestamp": "2026-03-26T15:00:00Z",
  "summary": { ... },
  "commitStrategy": "per-profile",
  "commits": [ ... ],
  "automapperRemovalReady": true,
  "blockers": [],
  "recommendations": [
    {
      "type": "cleanup",
      "description": "Remove AutoMapper and AutoMapper.Extensions.Microsoft.DependencyInjection packages"
    }
  ]
}
```

### Step 7: Generate PR Description

```markdown
## AutoMapper Migration Report

### Summary
| Metric | Value |
|--------|-------|
| Total Mappings | {totalMappings} |
| Successfully Migrated | {successfullyMigrated} |
| Requires Manual Work | {requiresManualWork} |
| Tests Written | {testsWritten} |
| Call Sites Updated | {callSitesUpdated} |
| Duration | {duration} |

### Status
{automapperRemovalReady ? 'Ready to remove AutoMapper' : 'Manual intervention required'}

### Blockers
{blockers.length > 0 ? blockers.map(b => `- ${b}`).join('\n') : 'None'}

### Commits
{commits.map(c => `- ${c.message}`).join('\n')}

### Recommendations
{recommendations.map(r => `- **${r.type}**: ${r.description}`).join('\n')}
```

## Output Files

1. `.github/state/report.json` - Machine-readable report
2. PR description update - Human-readable summary

## Visualization (Optional)

### Migration Progress
```
Tier Distribution:
  leaf      ████████████████████████ 30 (71%)
  formember ████ 5 (12%)
  converter ███ 3 (7%)
  hierarchy ██ 2 (5%)
  complex   ██ 2 (5%)

Status:
  Migrated: 41
  Failed:   1
  Skipped:  0
```

### Test Results
```
Characterization Tests:
  Total:   126
  Passed:  126
  Failed:  0

  Baseline -> Post-Migration: Match
```
