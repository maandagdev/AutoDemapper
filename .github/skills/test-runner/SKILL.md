---
name: test-runner
description: "Executes .NET tests and parses results into structured format for pipeline consumption."
---

# Test Runner

Executes characterization tests and captures results in a structured format.

## When to Use
- Establishing baseline (Stage 4)
- Validating migration (Stage 6)
- Any test execution during pipeline

## Prerequisites
- .NET SDK installed
- Test projects configured
- Test command known from inventory.json

## Procedure

### Step 1: Determine Test Command

From `inventory.json`:
```json
"repoFacts": {
  "testCommand": "dotnet test",
  "testFramework": "xunit"
}
```

### Step 2: Build Filter

For characterization tests only:

| Framework | Filter |
|-----------|--------|
| xUnit | `--filter "Category=Characterization"` |
| NUnit | `--filter "TestCategory=Characterization"` |
| MSTest | `--filter "TestCategory=Characterization"` |

### Step 3: Execute Tests

```bash
dotnet test {solution_path} \
  --filter "Category=Characterization" \
  --logger "trx;LogFileName=results.trx" \
  --results-directory ./TestResults \
  --no-build
```

Options:
- `--logger trx` - Generate TRX file for parsing
- `--results-directory` - Known output location
- `--no-build` - Skip build if already compiled

### Step 4: Parse Results

**From Console Output**:
```
Passed!  - Failed:     0, Passed:   126, Skipped:     0, Total:   126
```

Extract:
- Total tests
- Passed count
- Failed count
- Skipped count

**From TRX File** (more detailed):
```xml
<TestRun>
  <ResultSummary outcome="Completed">
    <Counters total="126" passed="126" failed="0" />
  </ResultSummary>
  <Results>
    <UnitTestResult testName="..." outcome="Passed|Failed" />
  </Results>
</TestRun>
```

### Step 5: Handle Failures

For each failed test, extract:
- Test name
- Error message
- Stack trace (if available)

```json
{
  "testName": "Maps_Order_To_OrderDto_Discount",
  "error": "Assert.Equal() Failure",
  "stackTrace": "at OrderMappingTests.cs:45"
}
```

### Step 6: Calculate Duration

Parse from TRX or measure execution time:
```
duration = endTime - startTime
format as "HH:MM:SS"
```

## Output

```json
{
  "command": "dotnet test --filter Category=Characterization",
  "totalTests": 126,
  "passed": 126,
  "failed": 0,
  "skipped": 0,
  "duration": "00:02:34"
}
```

## Error Handling

### Build Failure
```json
{
  "error": "build_failed",
  "message": "Solution failed to build",
  "details": "..."
}
```

### No Tests Found
```json
{
  "warning": "no_tests_found",
  "message": "No tests matched filter"
}
```

### Test Timeout
Default timeout: 10 minutes per test run
```json
{
  "error": "timeout",
  "message": "Test run exceeded timeout"
}
```

## Console Output Patterns

**Success**:
```
Test Run Successful.
Total tests: 126
     Passed: 126
```

**Failure**:
```
Test Run Failed.
Total tests: 126
     Failed: 1
```

Parse using regex:
```regex
Total tests:\s*(\d+)
\s*Passed:\s*(\d+)
\s*Failed:\s*(\d+)
```
