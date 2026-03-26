# AutoDemapper Global Instructions

## Core Principles

### 1. Safety First
**Tests must exist and pass before any mapping is migrated.**

```
NO migration without characterization tests.
NO test deletion or modification to make tests pass.
NO assuming behavior - verify with tests.
```

### 2. Incremental Progress
Work in two phases:

**Phase 1 — Analysis** (read-only, no source changes):
1. Discover all mappings
2. Generate characterization tests
3. Audit coverage
4. Establish green baseline

**Phase 2 — Migration** (only after user reviews Phase 1 output):
5. Migrate mappings
6. Verify tests still pass
7. Commit and report

### 3. State Preservation
All inter-agent communication happens via `.github/state/*.json` files.
- Each agent owns specific state files
- Never modify another agent's output files
- Always read from state before acting

### 4. Fail Safely
When encountering issues:
- Mark the item as failed
- Continue with other items
- Report failures clearly
- Never force success

## Prohibited Actions

- **Never** delete existing tests  
- **Never** modify existing source code behavior  
- **Never** skip mappings without documenting why  
- **Never** proceed to migration without green baseline  
- **Never** start Phase 2 (migration) without explicit user approval  
- **Never** assume test framework - detect it  
- **Never** assume AutoMapper exists - verify it  

## Required Behaviors

- **Always** generate characterization tests first  
- **Always** run tests after any code change  
- **Always** record state changes in JSON files  
- **Always** handle null cases in mappings  
- **Always** preserve existing behavior exactly  
- **Always** work in dependency order (base before derived)  

## State File Ownership

| Agent | Creates/Updates |
|-------|-----------------|
| autodemapper-agent | orchestrates only — no state files |
| inventory-agent | inventory.json |
| test-generator-agent | tests.json |
| audit-agent | audit.json |
| baseline-agent | baseline.json |
| migration-agent | migration.json |
| validation-agent | validation.json |
| reporter-agent | report.json |

## Error Messages

Use these standard formats:

```json
{
  "error": "error_code",
  "message": "Human readable description",
  "details": "Technical details",
  "recoverable": true/false
}
```

## Quality Gates

### Before Stage 2 (Test Generation)
- [ ] inventory.json exists
- [ ] At least one mapping found OR explicit "no AutoMapper" flag

### Before Stage 3 (Audit)
- [ ] tests.json exists
- [ ] Test files created

### Before Stage 4 (Baseline)
- [ ] audit.json exists
- [ ] `readyForBaseline: true`

### Before Stage 5 (Migration)
- [ ] baseline.json exists
- [ ] `baselineEstablished: true`
- [ ] Zero test failures
- [ ] **User has explicitly reviewed Phase 1 output and triggered Phase 2**

### Before Stage 6 (Validation)
- [ ] migration.json exists
- [ ] At least one mapping migrated

### Before Stage 7 (Report)
- [ ] validation.json exists

## Naming Conventions

### Files
- State files: `lowercase-with-dashes.json`
- Agent files: `lowercase-agent.agent.md`
- Skill folders: `lowercase-skill-name/`

### Code
- Mapper classes: `{SourceType}Mapper`
- Extension methods: `To{DestType}()`
- Test classes: `{SourceType}MappingTests`
- Test methods: `Maps_{Source}_To_{Dest}_{Scenario}`

## Performance Guidelines

- Batch similar operations
- Minimize grep/glob calls by combining patterns
- Cache type analysis results
- Use efficient regexes (avoid .* where possible)

## Rollback Strategy

If migration must be rolled back:
1. Git revert migration commits
2. Restore from baseline
3. Keep characterization tests (they're valuable)
4. Document failure reasons

## Communication

When reporting to user:
- Be concise
- Use metrics (X of Y completed)
- Highlight blockers clearly
- Suggest next actions
