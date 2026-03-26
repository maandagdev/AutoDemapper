---
name: repo-discovery
description: "Detects .NET solution structure, test framework, and build configuration from repository."
---

# Repository Discovery

Analyzes a .NET repository to extract key facts needed for migration planning.

## When to Use
- Initial setup of AutoDemapper in a new repository
- Validating repository structure before migration

## Prerequisites
- .NET repository with *.sln or *.csproj files

## Procedure

### Step 1: Find Solution File
```bash
find . -name "*.sln" -type f | head -1
```

If none found, fall back to project discovery.

### Step 2: List All Projects
```bash
find . -name "*.csproj" -type f
```

### Step 3: Identify Test Projects
Test projects typically:
- Have names containing "Test", "Tests", or "Spec"
- Reference test framework packages
- Have `IsTestProject` in csproj

Check each csproj:
```bash
grep -l "Microsoft.NET.Test.Sdk" *.csproj
```

### Step 4: Detect Test Framework
Scan csproj files for package references:

| Package | Framework |
|---------|-----------|
| `xunit` | xUnit |
| `xunit.runner` | xUnit |
| `NUnit` | NUnit |
| `NUnit3TestAdapter` | NUnit |
| `MSTest.TestFramework` | MSTest |
| `Microsoft.VisualStudio.TestTools.UnitTesting` | MSTest |

### Step 5: Detect .NET Version
From csproj:
```xml
<TargetFramework>net8.0</TargetFramework>
```

### Step 6: Find AutoMapper Version
```bash
grep -r "AutoMapper" --include="*.csproj" | grep "PackageReference"
```

Extract version from:
```xml
<PackageReference Include="AutoMapper" Version="12.0.1" />
```

### Step 7: Determine Test Command
Default: `dotnet test`

With solution: `dotnet test path/to/Solution.sln`

## Output
```json
{
  "solutionPath": "src/MyApp.sln",
  "testFramework": "xunit",
  "testCommand": "dotnet test src/MyApp.sln",
  "testProjects": [
    "tests/MyApp.Tests/MyApp.Tests.csproj",
    "tests/MyApp.IntegrationTests/MyApp.IntegrationTests.csproj"
  ],
  "netVersion": "net8.0",
  "autoMapperVersion": "12.0.1"
}
```

## Edge Cases

### No Solution File
- Use first csproj found as root
- Run tests per-project

### Multiple Test Frameworks
- List all detected
- Use most common for generated tests
- Add note in inventory

### No AutoMapper
- Set `autoMapperVersion: null`
- Allow pipeline to exit cleanly

### SDK-Style vs Legacy
- SDK-style: parse XML directly
- Legacy (packages.config): check packages.config file
