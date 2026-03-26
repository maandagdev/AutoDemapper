# AutoDemapper

> A multi-agent framework for safely migrating away from AutoMapper in .NET codebases.

## Quick Start

Copy the `.github/` folder into your .NET repository, then type `/autodemapper` in VS Code Copilot chat:

```bash
# Copy to your repo
cp -r .github/ /path/to/your/dotnet/repo/
```

Then in VS Code Copilot chat:
```
/autodemapper
```

That's it. The pipeline runs, pauses for your review, and presents two buttons to proceed.

## What is AutoDemapper?

AutoDemapper is a **template repository** containing GitHub Copilot custom agents, skills, and instructions that migrate .NET codebases away from AutoMapper in two explicit phases:

**Phase 1 — Analysis** (read-only, no source changes):
1. **Discover** all AutoMapper usage in your codebase
2. **Generate** characterization tests capturing current behavior
3. **Verify** test coverage before any changes
4. **Establish** a passing baseline against current AutoMapper

Phase 1 ends with a summary and two handoff buttons — you pick the mapping approach:

| Button | Approach |
|--------|---------|
| ▶ Proceed with Manual Mappers | Static extension methods, zero dependencies, full IDE support |
| ▶ Proceed with Mapperly | Source-generated at compile time, less boilerplate, requires `Riok.Mapperly` NuGet |

**Phase 2 — Migration** (triggered only after you click a handoff button):
5. **Migrate** mappings to manual implementations
6. **Validate** no behavior changes occurred
7. **Report** on migration status and next steps

## Philosophy: Safety First

**No mapping is migrated until:**
- Characterization tests exist
- Tests pass against current AutoMapper behavior
- Tests continue passing after migration

## Architecture

### Agents

Only `autodemapper-agent` appears in the agent picker. All worker agents are internal — they are invoked as subagents by the orchestrator and hidden from the dropdown.

Every agent has the `todo` tool enabled, which drives the step-by-step progress widget visible in the Chat panel. The two high-volume agents (`test-generator-agent` and `migration-agent`) also use `agent/runSubagent` to process each mapping in an isolated context, preventing context overflow on large codebases.

| Agent | Phase | Purpose | Stage |
|-------|-------|---------|-------|
| `autodemapper-agent` | Orchestrator | Drives the full pipeline, enforces quality gates | — |
| `inventory-agent` | Analysis | Discovers solution structure and mappings | 1 |
| `test-generator-agent` | Analysis | Creates characterization tests | 2 |
| `audit-agent` | Analysis | Verifies test coverage | 3 |
| `baseline-agent` | Analysis | Establishes passing test baseline | 4 |
| `migration-agent` | Migration | Implements mappers (manual or Mapperly) | 5 |
| `validation-agent` | Migration | Confirms no regressions | 6 |
| `reporter-agent` | Migration | Generates final report | 7 |

### Skills

Reusable procedures that agents load on demand:

- `repo-discovery` — Detect .NET structure and test framework
- `profile-inventory` — Scan for AutoMapper profiles
- `characterization-test` — Generate test code
- `test-framework-adapter` — Adapt to xUnit/NUnit/MSTest
- `test-audit-checklist` — Verify coverage
- `test-runner` — Execute tests and parse results
- `migration-implementation` — Create mappers (manual or Mapperly)
- `callsite-rewrite` — Update `IMapper.Map` call sites
- `commit-strategy` — Plan commit structure
- `report-generator` — Create final report

### Instructions

File-pattern-scoped conventions that apply automatically when agents work on matching files:

| File | Applies to |
|------|------------|
| `csharp-mappers.instructions.md` | `**/*Mapper.cs` — null guards, collection conventions, private setter patterns |
| `characterization-tests.instructions.md` | `**/*MappingTests.cs` — category markers, required scenarios, factory methods |
| `state-files.instructions.md` | `.github/state/*.json` — schema refs, ownership rules, atomic writes |

### State Files

All inter-agent communication uses JSON files in `.github/state/`:

```
.github/state/
├── schemas/           # JSON Schema definitions
├── inventory.json     # Stage 1 output
├── tests.json         # Stage 2 output
├── audit.json         # Stage 3 output
├── baseline.json      # Stage 4 output
├── migration.json     # Stage 5 output
├── validation.json    # Stage 6 output
└── report.json        # Stage 7 output
```

## Directory Structure

```
.github/
├── agents/                    # Copilot custom agents
│   ├── autodemapper-agent.agent.md   ← only one visible in picker
│   ├── inventory-agent.agent.md
│   ├── test-generator-agent.agent.md
│   ├── audit-agent.agent.md
│   ├── baseline-agent.agent.md
│   ├── migration-agent.agent.md
│   ├── validation-agent.agent.md
│   └── reporter-agent.agent.md
├── skills/                    # Reusable skill procedures
│   ├── profile-inventory/
│   ├── repo-discovery/
│   ├── characterization-test/
│   ├── test-framework-adapter/
│   ├── test-audit-checklist/
│   ├── test-runner/
│   ├── migration-implementation/
│   ├── callsite-rewrite/
│   ├── commit-strategy/
│   └── report-generator/
├── instructions/              # File-scoped coding conventions
│   ├── csharp-mappers.instructions.md
│   ├── characterization-tests.instructions.md
│   └── state-files.instructions.md
├── prompts/                   # Slash commands
│   └── autodemapper.prompt.md        ← /autodemapper entry point
├── state/                     # Pipeline state files
│   └── schemas/               # JSON Schema definitions
└── copilot-instructions.md    # Global pipeline rules
```

## Pipeline

```
┌── Phase 1: Analysis (read-only) ───────────────────────────────────┐
│                                                                    │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐        │
│  │ Stage 1  │──▶│ Stage 2  │──▶│ Stage 3  │──▶│ Stage 4  │        │
│  │Inventory │   │ TestGen  │   │ Audit    │   │ Baseline │        │
│  └──────────┘   └──────────┘   └──────────┘   └──────────┘        │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
         ↓ Summary presented — click a handoff button to continue
┌── Phase 2: Migration (modifies source) ────────────────────────────┐
│                                                                    │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐                        │
│  │ Stage 5  │──▶│ Stage 6  │──▶│ Stage 7  │                        │
│  │ Migrate  │   │ Validate │   │ Report   │                        │
│  └──────────┘   └──────────┘   └──────────┘                        │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

## Usage

### 1. Copy to Your Repository

```bash
git clone https://github.com/maandagdev/AutoDemapper.git
cp -r AutoDemapper/.github /path/to/your/project/
```

### Option A — Slash command (recommended)

```
/autodemapper
```

Runs the full pipeline via `autodemapper-agent`. Phase 1 ends with a summary and two handoff buttons to choose your mapping approach and start Phase 2.

### Option B — Direct agent

```
@autodemapper-agent Start the migration pipeline
```

### Option C — Manual step-by-step

Invoke each specialist agent individually. Note: worker agents are hidden from the picker by default — use `@agent-name` syntax directly.

**Phase 1 — Analysis**
```
@inventory-agent Scan this repository for AutoMapper usage
@test-generator-agent Generate characterization tests for all mappings
@audit-agent Audit characterization test coverage
@baseline-agent Run characterization tests and establish baseline
```

Review `.github/state/inventory.json`, `audit.json`, and `baseline.json`, then continue:

**Phase 2 — Migration**
```
@migration-agent Migrate all mappings to manual implementations
@validation-agent Validate no regressions against baseline
@reporter-agent Generate final migration report
```

## Requirements

- .NET SDK 6.0+
- VS Code with GitHub Copilot
- Repository with AutoMapper usage

## Contributing

1. Fork this repository
2. Make changes to agents/skills/instructions
3. Test with a sample .NET project
4. Submit PR

## License

MIT License — see LICENSE file for details.

---

**Note**: This is a template repository. The actual migration happens in your consumer repository after copying `.github/`.
