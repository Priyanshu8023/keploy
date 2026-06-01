# FEATURE-004: Parallel Test Set Execution

## Feature Overview
Execute multiple test sets concurrently during `keploy test`, reducing total test time by 3-5x for typical suites. Uses Go's errgroup for lifecycle management, consistent with the codebase's existing patterns.

## Why This Feature Matters

### User Impact
- A 10-test-set suite taking 60 seconds sequentially completes in ~15 seconds with 4-way parallelism
- CI pipeline feedback loop tightens proportionally
- Developers run full test suites more frequently

### Developer Impact
- Aligns with Go's native testing philosophy (`go test -parallel`)
- Enables future load testing feature (FEATURE-013) which requires concurrency infrastructure

### Project Impact
- **Critical competitive gap**: Every modern test runner supports parallel execution
- Enterprise teams with large test suites won't adopt without this

## Functional Requirements

1. **Default behavior**: Sequential (backward compatible)
2. **`--parallel N`** flag: Run up to N test sets concurrently
3. **`--parallel 0`** or `--parallel auto`: Use `runtime.GOMAXPROCS(0)` as concurrency
4. **Output isolation**: Each test set's output is buffered and printed atomically (no interleaving)
5. **Report aggregation**: Final report aggregates results from all parallel test sets
6. **Error handling**: If one test set fails, others continue (unless `--fail-fast`)
7. **`--fail-fast`** flag: Stop all test sets on first failure
8. **Resource isolation**: Each test set gets its own mock pool instance (already the case)
9. **App lifecycle**: When `--keep-app-alive` is set, single app instance serves all parallel test sets
10. **Exit code**: Non-zero if any test set failed

## Non-Functional Requirements

- **Performance**: Less than 5% overhead per test set from parallelism infrastructure
- **Memory**: Bounded by `N * per-test-set-memory` — document memory implications
- **Reliability**: Must not introduce race conditions in shared state (reportDB, telemetry)
- **Scalability**: Support N up to 32 (above that, port/resource contention is likely)

## Technical Design

### Components Affected
| Component | Change Type | Risk |
|-----------|-------------|------|
| `pkg/service/replay/replay.go` | MODIFY — Parallelize `runAllTestSets` loop | High |
| `pkg/service/replay/replay.go` | MODIFY — Buffer per-set output | Medium |
| `config/config.go` | MODIFY — Add `Parallel` and `FailFast` fields | Low |
| `cli/provider/cmd.go` | MODIFY — Add `--parallel` and `--fail-fast` flags | Low |
| `pkg/service/replay/service.go` | REFERENCE — `ReportDB.InsertReport` must be thread-safe | Medium |

### Architecture Diagram
```
keploy test --parallel 4
       │
       ▼
┌──────────────────────┐
│  Load Test Sets      │  ← Sequential: read test-set-0, test-set-1, ...
└────────┬─────────────┘
         │
         ▼
┌──────────────────────┐
│  errgroup (N=4)      │  ← Parent errgroup with cancel
│  ┌────────────────┐  │
│  │ test-set-0     │  │  ← Each goroutine gets own:
│  │  - load mocks  │  │    - mock pool
│  │  - run tests   │  │    - output buffer
│  │  - write report│  │    - report writer
│  └────────────────┘  │
│  ┌────────────────┐  │
│  │ test-set-1     │  │
│  └────────────────┘  │
│  ┌────────────────┐  │
│  │ test-set-2     │  │
│  └────────────────┘  │
│  ┌────────────────┐  │
│  │ test-set-3     │  │
│  └────────────────┘  │
└────────┬─────────────┘
         │
         ▼
┌──────────────────────┐
│  Aggregate Results   │  ← Merge per-set reports
│  Print Summary       │  ← Ordered output
└──────────────────────┘
```

### Key Design Decisions

1. **Errgroup over WaitGroup**: Consistent with codebase convention. Provides first-error-cancel semantics for `--fail-fast`.

2. **Buffered output**: Each test set captures log/output to a `bytes.Buffer`. After all sets complete, buffers are printed in order. This prevents interleaved output that confuses users.

3. **ReportDB thread safety**: `InsertReport` and `InsertTestCaseResult` currently run sequentially. With parallel test sets, calls to these must be serialized. Options:
   - Add `sync.Mutex` to the YAML reportDB implementation
   - Use a channel-based writer (preferred — matches existing patterns)

4. **Mock pool isolation**: Each test set already gets its own filtered mock pool via `GetFilteredMocks`. No change needed.

5. **App instance**: When `--parallel` is used without `--keep-app-alive`, the app must be shared. Force `--keep-app-alive` when `--parallel > 1` and warn the user.

## Implementation Plan

### Step 1: Add config fields and CLI flags (1 hour)
```go
// config/config.go
type Test struct {
    // ... existing fields
    Parallel int  `json:"parallel" yaml:"parallel" mapstructure:"parallel"`
    FailFast bool `json:"failFast" yaml:"failFast" mapstructure:"failFast"`
}
```

### Step 2: Thread-safe ReportDB (3 hours)
- Add mutex to `pkg/platform/yaml/reportdb/` write operations
- Or use a serial writer goroutine with channels

### Step 3: Refactor runAllTestSets loop (4 hours)
Current code in `replay.go` iterates test sets sequentially:
```go
for _, testSetID := range testSetIDs {
    status, err := r.RunTestSet(ctx, testSetID, testRunID, serveTest)
    // ...
}
```

Parallel version:
```go
sem := make(chan struct{}, cfg.Test.Parallel)
g, gCtx := errgroup.WithContext(ctx)
results := make([]testSetResult, len(testSetIDs))

for i, testSetID := range testSetIDs {
    i, testSetID := i, testSetID
    g.Go(func() error {
        sem <- struct{}{}
        defer func() { <-sem }()
        status, err := r.RunTestSet(gCtx, testSetID, testRunID, serveTest)
        results[i] = testSetResult{status: status, err: err}
        if cfg.Test.FailFast && status == models.TestSetStatusFailed {
            return fmt.Errorf("test set %s failed (fail-fast)", testSetID)
        }
        return nil
    })
}
```

### Step 4: Output buffering (2 hours)
- Wrap logger with a per-goroutine buffer
- Flush buffers in test-set order after errgroup completes

### Step 5: Summary aggregation (1 hour)
- Merge per-set TestReport into aggregate TestReport
- Print combined summary table

## Code Locations

### Modified Files
```
config/config.go                           # Add Parallel, FailFast fields
config/default.go                          # Add default parallel: 1
cli/provider/cmd.go                        # Add --parallel, --fail-fast flags
pkg/service/replay/replay.go              # Parallelize test set loop
pkg/platform/yaml/reportdb/reportdb.go    # Thread-safe write operations
```

### New Files
```
pkg/service/replay/parallel.go            # Parallel execution orchestrator
pkg/service/replay/parallel_test.go       # Concurrency tests
```

## Testing Plan

### Unit Tests
- `parallel_test.go`: Test semaphore-based concurrency with mock test sets
- Verify N goroutines never exceed configured parallelism
- Verify `--fail-fast` cancels remaining test sets

### Integration Tests
- Run `keploy test --parallel 4` on a 4-test-set sample project
- Verify all test sets produce correct reports
- Verify no race conditions with `go test -race`

### Performance Tests
- Benchmark sequential vs. parallel (2, 4, 8) on the same test suite
- Measure memory overhead per parallelism level

### Regression Tests
- `keploy test` (no --parallel) behavior unchanged
- `keploy test --parallel 1` behavior identical to no flag

## Risks

1. **Port contention**: If multiple test sets need the same port → Mitigated by `--keep-app-alive` (single app instance)
2. **Race conditions**: ReportDB writes → Mitigated by mutex/channel
3. **Memory pressure**: N test sets × mock pool size → Document memory implications, add memory guard integration
4. **Log interleaving**: → Mitigated by output buffering

## Rollback Plan
- Remove `--parallel` flag and config field
- Revert `replay.go` to sequential loop
- No data format changes, no migration needed

## Acceptance Criteria
- [ ] `keploy test --parallel 4` runs 4 test sets concurrently
- [ ] Output is ordered (not interleaved) regardless of completion order
- [ ] `--fail-fast` stops remaining test sets on first failure
- [ ] `--parallel 0` uses GOMAXPROCS
- [ ] Default behavior (no flag) is sequential
- [ ] All reports are correctly generated
- [ ] No race conditions (passes `go test -race`)
- [ ] CI passes
- [ ] Documentation updated
