# QUICK_WINS.md — Features Achievable in < 1 Day

> These features provide significant value with minimal effort. Each includes **exact implementation steps** and **code locations**. Many are ideal for first-time contributors.

---

## QW-001: `keploy doctor` — Environment Health Check

**Effort:** 4-6 hours | **Impact:** High | **Good First Contribution:** ✅

### What It Does
A single command that validates the entire Keploy environment and reports issues with actionable fixes.

### Implementation Steps

1. **Create `cli/doctor.go`** — New cobra command
```go
// Package cli provides the Keploy CLI commands.
var doctorCmd = &cobra.Command{
    Use:   "doctor",
    Short: "Check system environment for Keploy compatibility",
    RunE: func(cmd *cobra.Command, args []string) error {
        return runDoctor(cmd.Context())
    },
}
```

2. **Create `pkg/service/doctor/doctor.go`** — Health check logic
```go
type Check struct {
    Name    string
    Status  string // "✅ PASS", "⚠️ WARN", "❌ FAIL"
    Message string
    Fix     string
}

func RunChecks(ctx context.Context) []Check {
    checks := []Check{}
    checks = append(checks, checkGoVersion())
    checks = append(checks, checkDockerAvailable())
    checks = append(checks, checkDockerRunning())
    checks = append(checks, checkKernelVersion())    // Linux only
    checks = append(checks, checkEBPFSupport())       // Linux only
    checks = append(checks, checkPortAvailability())
    checks = append(checks, checkConfigFile())
    checks = append(checks, checkKeployDir())
    checks = append(checks, checkDiskSpace())
    return checks
}
```

3. **Register in `cli/root.go`** — Add `doctorCmd` to root

### Files to Create/Modify
- `[NEW] cli/doctor.go`
- `[NEW] pkg/service/doctor/doctor.go`
- `[MODIFY] cli/root.go` — Register command

---

## QW-002: Markdown Report Format

**Effort:** 3-4 hours | **Impact:** High | **Good First Contribution:** ✅

### What It Does
`keploy report --format markdown` outputs a GitHub Flavored Markdown report suitable for PR comments.

### Implementation Steps

1. **Add `markdown` case in `pkg/service/report/report.go`** — In `GenerateReport()`:
```go
case "markdown":
    reports, err := r.collectReports(ctx, latestRunID, testSetIDs)
    if err != nil { return err }
    return r.generateMarkdown(reports)
```

2. **Create `pkg/service/report/markdown.go`**:
```go
func (r *Report) generateMarkdown(reports map[string]*models.TestReport) error {
    fmt.Fprintf(r.out, "## 🧪 Keploy Test Results\n\n")
    fmt.Fprintf(r.out, "| Test Suite | Total | ✅ Passed | ❌ Failed | Duration |\n")
    fmt.Fprintf(r.out, "|-----------|-------|----------|----------|----------|\n")
    for name, rep := range reports {
        fmt.Fprintf(r.out, "| %s | %d | %d | %d | %s |\n",
            name, rep.Total, rep.Success, rep.Failure, rep.TimeTaken)
    }
    // ... add failure details with code blocks
    return r.out.Flush()
}
```

3. **Update `cli/provider/cmd.go`** — Add `"markdown"` to format validation

### Files to Create/Modify
- `[NEW] pkg/service/report/markdown.go`
- `[MODIFY] pkg/service/report/report.go` — Add markdown case (~5 lines)
- `[MODIFY] cli/provider/cmd.go` — Validate `markdown` as a format option

---

## QW-003: Test Retention Policies & `keploy clean`

**Effort:** 3-4 hours | **Impact:** Medium | **Good First Contribution:** ✅

### What It Does
- Config option: `retention.maxTestRuns: 10` and `retention.maxAge: 30d`
- New command: `keploy clean` — removes old test runs based on retention policy

### Implementation Steps

1. **Add to `config/config.go`**:
```go
type Retention struct {
    MaxTestRuns int           `json:"maxTestRuns" yaml:"maxTestRuns" mapstructure:"maxTestRuns"`
    MaxAge      time.Duration `json:"maxAge" yaml:"maxAge" mapstructure:"maxAge"`
}
```

2. **Create `cli/clean.go`** — Cobra command:
```go
var cleanCmd = &cobra.Command{
    Use:   "clean",
    Short: "Remove old test runs based on retention policy",
}
```

3. **Implement cleanup in `pkg/service/tools/clean.go`** — Walk `./keploy/reports/`, sort by `test-run-N` number, remove oldest beyond `maxTestRuns`.

### Files to Create/Modify
- `[MODIFY] config/config.go` — Add `Retention` struct
- `[MODIFY] config/default.go` — Add default retention values
- `[NEW] cli/clean.go`
- `[NEW] pkg/service/tools/clean.go`
- `[MODIFY] cli/root.go` — Register command

---

## QW-004: `keploy validate` — Test/Mock Schema Validation

**Effort:** 4-6 hours | **Impact:** Medium | **Good First Contribution:** ✅

### What It Does
Validates all YAML test/mock files for structural correctness without running a full replay.

### Checks
- YAML parse validity
- Required fields present (Version, Kind, Name)
- Timestamp consistency (reqTimestamp < resTimestamp)
- Orphaned mocks (referenced in mappings but missing from mocks file)
- Mock name uniqueness within a test set

### Implementation Steps

1. **Create `cli/validate.go`** — Cobra command
2. **Create `pkg/service/tools/validate.go`**:
```go
func Validate(ctx context.Context, logger *zap.Logger, path string) []ValidationError {
    // Walk ./keploy/test-set-*/
    // For each: parse tests/*.yaml, parse mocks.yaml
    // Run validation checks
    // Return errors with file path, line number, message
}
```

### Files to Create/Modify
- `[NEW] cli/validate.go`
- `[NEW] pkg/service/tools/validate.go`
- `[MODIFY] cli/root.go`

---

## QW-005: `--progress` Flag for Test Execution

**Effort:** 3-4 hours | **Impact:** High | **Good First Contribution:** ✅

### What It Does
Shows a live progress indicator during `keploy test`:
```
Test Set: test-set-0 [████████░░░░░░░░] 8/16 tests (50%) | 3 passed, 1 failed | 12s elapsed
```

### Implementation Steps

1. **Add `--progress` flag** in `cli/provider/cmd.go` for `test` command
2. **Create `pkg/service/replay/progress.go`**:
```go
type ProgressReporter struct {
    total, current, passed, failed int
    startTime time.Time
    mu sync.Mutex
}

func (p *ProgressReporter) Render() string {
    // Build progress bar string
}
```

3. **Integrate in `replay.go`** — Call `progressReporter.Update()` after each test case completes

### Files to Create/Modify
- `[NEW] pkg/service/replay/progress.go`
- `[MODIFY] pkg/service/replay/replay.go` — Add progress hooks (~10 lines)
- `[MODIFY] cli/provider/cmd.go` — Add `--progress` flag

---

## QW-006: Test Run Summary in JSON (Machine-Readable Exit)

**Effort:** 2-3 hours | **Impact:** Medium | **Good First Contribution:** ✅

### What It Does
When `keploy test --json` is used, output a structured JSON summary at the end of the run:
```json
{
  "status": "FAILED",
  "total": 24,
  "passed": 21,
  "failed": 3,
  "testSets": [...],
  "duration": "12.5s",
  "failedTests": [
    {"testSet": "test-set-0", "testCase": "test-3", "risk": "HIGH"}
  ]
}
```

### Implementation Steps
1. **Create `pkg/service/replay/json_summary.go`** — Build summary struct
2. **Emit at end of `RunTestSet`** — After all test sets complete, write JSON to stdout
3. **Already uses `utils.NewJSONWriter`** — Reuse existing pattern from `report.go`

### Files to Create/Modify
- `[NEW] pkg/service/replay/json_summary.go`
- `[MODIFY] pkg/service/replay/replay.go` — Emit summary (~15 lines)

---

## QW-007: `--dry-run` Flag for Test Command

**Effort:** 2-3 hours | **Impact:** Medium | **Good First Contribution:** ✅

### What It Does
`keploy test --dry-run` lists all test cases that would be executed without actually running them:
```
Would execute 24 tests across 3 test sets:
  test-set-0: 8 tests (test-1, test-2, ..., test-8)
  test-set-1: 10 tests (test-1, ..., test-10)
  test-set-2: 6 tests (test-1, ..., test-6)
```

### Implementation Steps
1. **Add `--dry-run` flag** in `cli/provider/cmd.go`
2. **Short-circuit in replay.go** — After loading test cases, print summary and return
3. **Useful with `--test-sets` and `--tags`** — Shows filter effects without execution

### Files to Create/Modify
- `[MODIFY] cli/provider/cmd.go` — Add `--dry-run` flag
- `[MODIFY] config/config.go` — Add `DryRun bool` to `Test` struct
- `[MODIFY] pkg/service/replay/replay.go` — Add early-return branch

---

## QW-008: Keploy Test Badge Generator

**Effort:** 1-2 hours | **Impact:** Low | **Good First Contribution:** ✅

### What It Does
`keploy report --badge` generates a shields.io badge URL:
```
![Keploy Tests](https://img.shields.io/badge/keploy-21%20passed%20%7C%203%20failed-red)
```

### Implementation Steps
1. **Add to `pkg/service/report/report.go`**:
```go
func (r *Report) generateBadge(reports map[string]*models.TestReport) error {
    total, passed, failed := 0, 0, 0
    for _, rep := range reports {
        total += rep.Total
        passed += rep.Success
        failed += rep.Failure
    }
    color := "brightgreen"
    if failed > 0 { color = "red" }
    url := fmt.Sprintf("https://img.shields.io/badge/keploy-%d%%20passed%%20%%7C%%20%d%%20failed-%s",
        passed, failed, color)
    fmt.Fprintln(r.out, url)
    return r.out.Flush()
}
```

### Files to Create/Modify
- `[MODIFY] pkg/service/report/report.go` — Add badge generator (~15 lines)
- `[MODIFY] cli/provider/cmd.go` — Add `--badge` flag
