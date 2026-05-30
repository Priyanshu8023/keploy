# Master Issue Tracker — Keploy Repository Audit

> **Total Issues Found**: 25  
> **Critical**: 3 | **High**: 8 | **Medium**: 9 | **Low**: 5

---

## ISSUE-001: Global Mutable State Creates Race Conditions

| Field | Value |
|---|---|
| **Severity** | Critical |
| **Category** | Bug / Concurrency |
| **Location** | [utils/utils.go](file:///c:/Users/priya/OneDrive/Desktop/Coding/keploy/utils/utils.go#L43-L48), [pkg/matcher/utils.go](file:///c:/Users/priya/OneDrive/Desktop/Coding/keploy/pkg/matcher/utils.go#L470-L473) |
| **Code Reference** | `TemplatizedValues`, `SecretValues`, `ErrCode` (package-level vars) |

**Problem**: Package-level mutable maps (`TemplatizedValues`, `SecretValues`) and `ErrCode` (int) are read/written from multiple goroutines without synchronization. `CompareResponses` mutates `TemplatizedValues` during replay while other goroutines may read it. The `-race` detector will flag this in concurrent test-set execution.

**Impact**: Data races leading to corrupted template values, non-deterministic test results, and potential panics on concurrent map access.

**Root Cause**: Global mutable state was introduced before the codebase adopted `errgroup`-based concurrency. Never refactored to thread-safe alternatives.

**Recommended Fix**: Replace global maps with a context-threaded `TemplateStore` struct protected by `sync.RWMutex`, or pass maps explicitly through function parameters. Change `ErrCode` to an `atomic.Int32`.

**Estimated Complexity**: Medium

---

## ISSUE-002: `ToAbsPath` Broken on Windows — Unix Path Assumption

| Field | Value |
|---|---|
| **Severity** | Critical |
| **Category** | Bug |
| **Location** | [utils/utils.go#L747-L765](file:///c:/Users/priya/OneDrive/Desktop/Coding/keploy/utils/utils.go#L747-L765) |
| **Code Reference** | `func ToAbsPath()` |

**Problem**: `ToAbsPath` checks `path[0] != '/'` to determine if a path is relative. On Windows, absolute paths start with a drive letter (e.g., `C:\`), so `C:\foo` is incorrectly treated as relative. Additionally, line 763 hardcodes a Unix separator: `path += "/keploy"`.

**Impact**: On Windows, all paths are incorrectly resolved, potentially creating directories in wrong locations or failing entirely. Since Windows is an officially supported platform, this is a first-class bug.

**Root Cause**: Function was written with Unix assumptions and never updated for cross-platform support.

**Recommended Fix**: Use `filepath.IsAbs(path)` instead of checking `path[0]`, and use `filepath.Join(path, "keploy")` instead of string concatenation.

**Estimated Complexity**: Small

---

## ISSUE-003: Default Config `disableMapping: true` Contradicts CLI Default

| Field | Value |
|---|---|
| **Severity** | Critical |
| **Category** | Bug / Configuration |
| **Location** | [config/default.go#L111](file:///c:/Users/priya/OneDrive/Desktop/Coding/keploy/config/default.go#L111), [cli/provider/cmd.go#L370](file:///c:/Users/priya/OneDrive/Desktop/Coding/keploy/cli/provider/cmd.go#L370) |
| **Code Reference** | `disableMapping: true` in defaults vs `c.cfg.DisableMapping` flag default |

**Problem**: The default config YAML sets `disableMapping: true` (line 111 of default.go), but the CLI `--disable-mapping` flag for the `test` command defaults to `c.cfg.DisableMapping` which is initialized from this same config. The CLI comment at line 361-369 of cmd.go documents this as a known conflict: "Hardcoding the default to true silently disabled mapping-based mock filtering in test mode". The mapping-based path is the more accurate one; silently disabling it causes non-deterministic mock matching.

**Impact**: Users who run `keploy test` without explicit config get timestamp-based mock filtering instead of the superior mapping-based filtering, leading to flaky test results with tightly-spaced mocks.

**Root Cause**: Default was set to `true` for backwards compatibility but should have been changed when mapping support matured.

**Recommended Fix**: Change `disableMapping: true` to `disableMapping: false` in `default.go`.

**Estimated Complexity**: Small

---

## ISSUE-004: Hardcoded GitHub Client ID in Source Code

| Field | Value |
|---|---|
| **Severity** | High |
| **Category** | Security |
| **Location** | [main.go#L33](file:///c:/Users/priya/OneDrive/Desktop/Coding/keploy/main.go#L33) |
| **Code Reference** | `var gitHubClientID = "Iv23liFBvIVhL29i9BAp"` |

**Problem**: OAuth client ID is hardcoded in source. While client IDs are not secrets, they enable OAuth flows and could be used for impersonation if paired with a compromised redirect URI.

**Impact**: Any fork or modified build inherits the same client ID. If used in OAuth PKCE flows, this could allow authorization code interception.

**Recommended Fix**: Inject via `ldflags` like other build-time constants (version, DSN) or move to environment variable.

**Estimated Complexity**: Small

---

## ISSUE-005: Default MongoDB Password Hardcoded in Config

| Field | Value |
|---|---|
| **Severity** | High |
| **Category** | Security |
| **Location** | [config/default.go#L60](file:///c:/Users/priya/OneDrive/Desktop/Coding/keploy/config/default.go#L60) |
| **Code Reference** | `mongoPassword: "default@123"` |

**Problem**: A non-empty default password is provided for MongoDB mocking. Users who don't override this may inadvertently expose a well-known credential in CI/CD logs, YAML files, or error messages.

**Impact**: If used in non-mock contexts (accidental production connection), this password is publicly known from the open-source repo.

**Recommended Fix**: Default to empty string and require explicit configuration. Log a clear warning when MongoDB password is not set.

**Estimated Complexity**: Small

---

## ISSUE-006: `GenerateGithubActions` Writes to Root Path `/githubactions/`

| Field | Value |
|---|---|
| **Severity** | High |
| **Category** | Bug / Security |
| **Location** | [utils/utils.go#L494](file:///c:/Users/priya/OneDrive/Desktop/Coding/keploy/utils/utils.go#L494) |
| **Code Reference** | `filePath := "/githubactions/keploy.yml"` |

**Problem**: The function writes to an absolute root path `/githubactions/keploy.yml` instead of the project's `.github/workflows/` directory. On Linux/Mac this would try to create a directory at `/githubactions/` (requiring root). On Windows it maps to `C:\githubactions\`.

**Impact**: Function never works correctly for its intended purpose (generating GitHub Actions workflow in the project). May create files in unexpected system locations if running with elevated privileges.

**Recommended Fix**: Change to `filepath.Join(".github", "workflows", "keploy.yml")` relative to the project root.

**Estimated Complexity**: Small

---

## ISSUE-007: `makeDirectory` Uses 0777 Permissions

| Field | Value |
|---|---|
| **Severity** | High |
| **Category** | Security |
| **Location** | [utils/utils.go#L769](file:///c:/Users/priya/OneDrive/Desktop/Coding/keploy/utils/utils.go#L769) |
| **Code Reference** | `os.MkdirAll(path, 0777)` |

**Problem**: Directories created with `0777` permissions are world-readable and world-writable. While the umask may reduce this, on some systems (Docker, mounted volumes) the umask is 0, resulting in fully open directories.

**Impact**: Sensitive test data (including captured API traffic, headers with tokens) could be readable by any user on the system.

**Recommended Fix**: Use `0755` (owner full, group/others read+execute) for directories.

**Estimated Complexity**: Small

---

## ISSUE-008: `DeleteFileIfNotExists` — Misleading Name, Deletes File If It EXISTS

| Field | Value |
|---|---|
| **Severity** | Medium |
| **Category** | Code Smell / Bug Risk |
| **Location** | [utils/utils.go#L332-L350](file:///c:/Users/priya/OneDrive/Desktop/Coding/keploy/utils/utils.go#L332-L350) |
| **Code Reference** | `func DeleteFileIfNotExists()` |

**Problem**: Function name says "DeleteFileIfNotExists" but it actually deletes the file if it EXISTS. The `os.IsNotExist` check on line 335 returns early when the file doesn't exist; the actual deletion happens when the file IS present. This is the exact opposite of what the name suggests.

**Impact**: Confusing for contributors. Could lead to incorrect usage if someone trusts the name at face value.

**Recommended Fix**: Rename to `DeleteFileIfExists` or `CleanupFile`.

**Estimated Complexity**: Small

---

## ISSUE-009: `replay.go` is 3770 Lines — God Object Anti-Pattern

| Field | Value |
|---|---|
| **Severity** | High |
| **Category** | Architecture / Maintainability |
| **Location** | [pkg/service/replay/replay.go](file:///c:/Users/priya/OneDrive/Desktop/Coding/keploy/pkg/service/replay/replay.go) |
| **Code Reference** | Entire file |

**Problem**: A single file contains the entire replay orchestration logic: test-set lifecycle, application management, mock filtering, response comparison, report generation, coverage calculation, flaky detection, and keep-alive management. The `Replayer` struct has 25+ fields including multiple mutex-guarded counters.

**Impact**: High merge conflict rate, difficult to test individual components, impossible to reason about concurrency safety, and onboarding barrier for contributors.

**Recommended Fix**: Extract into focused modules: `testset_runner.go`, `mock_filter.go`, `coverage.go`, `flaky_detector.go`, `report_writer.go`.

**Estimated Complexity**: Large

---

## ISSUE-010: `pkg/util.go` is 132KB — Monolithic Utility File

| Field | Value |
|---|---|
| **Severity** | High |
| **Category** | Architecture / Maintainability |
| **Location** | [pkg/util.go](file:///c:/Users/priya/OneDrive/Desktop/Coding/keploy/pkg/util.go) |
| **Code Reference** | Entire file (132KB, ~3500 lines) |

**Problem**: This single file contains HTTP request construction, protocol parsing helpers, mock formatting, URL manipulation, curl generation, test-set ID generation, stream body handling, and dozens of unrelated utilities.

**Impact**: Every change to any utility risks merge conflicts and unintended side effects across unrelated functionality.

**Recommended Fix**: Split into domain-specific packages: `pkg/httputil/`, `pkg/mockutil/`, `pkg/testutil/`.

**Estimated Complexity**: Large

---

## ISSUE-011: Busy-Wait Polling in Mock Correlation

| Field | Value |
|---|---|
| **Severity** | Medium |
| **Category** | Performance |
| **Location** | [pkg/service/record/record.go#L461-L468](file:///c:/Users/priya/OneDrive/Desktop/Coding/keploy/pkg/service/record/record.go#L461-L468) |
| **Code Reference** | Mock correlation retry loop |

**Problem**: The mock-to-test mapping correlation uses a busy-wait loop (`time.Sleep(10ms)` × 50 iterations = 500ms max). This burns CPU cycles while waiting for the mock goroutine to store the correlation entry.

**Impact**: Under high throughput, 50 goroutines each busy-polling every 10ms creates significant CPU overhead. The 500ms timeout is also arbitrary and may be insufficient for slow I/O systems.

**Recommended Fix**: Replace with a `sync.Cond` or channel-based notification pattern. The mock insertion goroutine should signal when a new entry is stored.

**Estimated Complexity**: Medium

---

## ISSUE-012: `os.Exit(1)` Inside Service Code Bypasses Deferred Cleanup

| Field | Value |
|---|---|
| **Severity** | High |
| **Category** | Bug |
| **Location** | [main.go#L220](file:///c:/Users/priya/OneDrive/Desktop/Coding/keploy/main.go#L220), [cli/provider/cmd.go#L1154](file:///c:/Users/priya/OneDrive/Desktop/Coding/keploy/cli/provider/cmd.go#L1154) |
| **Code Reference** | `os.Exit(1)` calls |

**Problem**: `os.Exit(1)` calls in `main.go` (line 220, on installation ID failure) and `cmd.go` (line 1154) bypass all deferred cleanup functions. This means log files aren't flushed, debug file sinks aren't closed, umask isn't restored, and Sentry events aren't flushed.

**Impact**: Data loss on exit paths — log data truncated, telemetry events lost, temporary files left behind.

**Recommended Fix**: Return errors up the call stack and let the single `os.Exit(utils.ErrCode)` in `main()` handle the exit.

**Estimated Complexity**: Medium

---

## ISSUE-013: Bare `go func()` Goroutines Without `errgroup` or Recovery

| Field | Value |
|---|---|
| **Severity** | Medium |
| **Category** | Concurrency / Reliability |
| **Location** | Multiple files: [record.go#L144](file:///c:/Users/priya/OneDrive/Desktop/Coding/keploy/pkg/service/record/record.go#L144), [telemetry.go](file:///c:/Users/priya/OneDrive/Desktop/Coding/keploy/pkg/platform/telemetry/telemetry.go#L81), [agent.go](file:///c:/Users/priya/OneDrive/Desktop/Coding/keploy/pkg/platform/http/agent.go#L114) |
| **Code Reference** | `go func() { ... }()` pattern |

**Problem**: The AGENTS.md explicitly states "use `errgroup.WithContext(ctx)` for any goroutine, not bare `go func()` or `sync.WaitGroup`." However, ~50 instances of bare `go func()` exist in production code (not tests). Many lack `utils.Recover()` panic recovery.

**Impact**: Panics in background goroutines crash the entire process without cleanup. Goroutine lifecycle is untracked — leaks possible on shutdown.

**Recommended Fix**: Audit each bare goroutine. Add to nearest `errgroup` or wrap with `utils.Recover()`.

**Estimated Complexity**: Medium

---

## ISSUE-014: `pprof` HTTP Server Bound Without Authentication

| Field | Value |
|---|---|
| **Severity** | High |
| **Category** | Security |
| **Location** | [main.go#L81-L90](file:///c:/Users/priya/OneDrive/Desktop/Coding/keploy/main.go#L81-L90) |
| **Code Reference** | `http.ListenAndServe(addr, nil)` for pprof |

**Problem**: When `PPROF_PORT` is set, a pprof HTTP server starts on `localhost:<port>` with no authentication. While bound to localhost, in Docker/container environments, "localhost" may be accessible from other containers on the same network.

**Impact**: Pprof endpoints expose goroutine stacks, heap profiles, and CPU profiles which can leak sensitive runtime data (database connection strings, auth tokens in goroutine stacks).

**Recommended Fix**: Add a bearer-token guard via `PPROF_TOKEN` environment variable, or restrict to Unix domain socket.

**Estimated Complexity**: Small

---

## ISSUE-015: Regex Compilation in Hot Path Without Size Limit

| Field | Value |
|---|---|
| **Severity** | Medium |
| **Category** | Performance / Security |
| **Location** | [utils/utils.go#L188-L204](file:///c:/Users/priya/OneDrive/Desktop/Coding/keploy/utils/utils.go#L188-L204), [pkg/matcher/utils.go#L39-L63](file:///c:/Users/priya/OneDrive/Desktop/Coding/keploy/pkg/matcher/utils.go#L39-L63) |
| **Code Reference** | `regexp.Compile(bypass.Host)`, `getCompiled(pattern)` |

**Problem**: `IsPassThrough` compiles user-provided regexes on every call in the bypass rule matching loop. While `pkg/matcher` has a `regexCache`, `IsPassThrough` does not. Additionally, neither path limits regex complexity — a crafted ReDoS pattern in `keploy.yml` can hang the process.

**Impact**: CPU exhaustion on complex regex patterns in bypass rules. The cache in matcher grows unboundedly.

**Recommended Fix**: Pre-compile bypass regexes at config load time. Add a max-length check (e.g., 1024 chars) and a compilation timeout.

**Estimated Complexity**: Medium

---

## ISSUE-016: No Validation of User-Provided Port Numbers

| Field | Value |
|---|---|
| **Severity** | Medium |
| **Category** | Validation |
| **Location** | [config/config.go](file:///c:/Users/priya/OneDrive/Desktop/Coding/keploy/config/config.go) |
| **Code Reference** | Port fields: `Port uint32`, `ProxyPort uint32`, `DNSPort uint32` |

**Problem**: Port configuration values are stored as `uint32` but valid ports are 1-65535. No validation exists to reject port 0 (except where it means "auto") or ports above 65535, or system ports (1-1023) that require root.

**Impact**: Invalid port values cause runtime failures with unhelpful error messages. Port 0 is silently accepted in some contexts.

**Recommended Fix**: Add a `ValidatePorts()` method to `Config` called during `ValidateFlags`.

**Estimated Complexity**: Small

---

## ISSUE-017: `Sentry.Init` with `TracesSampleRate: 1.0` in Production

| Field | Value |
|---|---|
| **Severity** | Medium |
| **Category** | Performance |
| **Location** | [utils/utils.go#L698-L705](file:///c:/Users/priya/OneDrive/Desktop/Coding/keploy/utils/utils.go#L698-L705) |
| **Code Reference** | `sentry.Init(sentry.ClientOptions{TracesSampleRate: 1.0})` |

**Problem**: `TracesSampleRate: 1.0` means 100% of transactions are sent to Sentry. For a CLI tool this may be acceptable, but for long-running record sessions intercepting high-throughput traffic, this generates excessive telemetry overhead.

**Impact**: Network bandwidth consumption and potential Sentry rate-limit errors in high-throughput scenarios.

**Recommended Fix**: Set `TracesSampleRate: 0.1` (10%) for normal mode, or make it configurable.

**Estimated Complexity**: Small

---

## ISSUE-018: `config.New()` Uses `panic()` on Parse Failure

| Field | Value |
|---|---|
| **Severity** | Medium |
| **Category** | Error Handling |
| **Location** | [config/default.go#L141-L158](file:///c:/Users/priya/OneDrive/Desktop/Coding/keploy/config/default.go#L141-L158) |
| **Code Reference** | `panic(err)` in `config.New()` |

**Problem**: `config.New()` panics on YAML merge or unmarshal failure. This happens at process startup before any recovery handler is installed. The AGENTS.md states "Library and service code returns errors. `recover` lives at top-level goroutine entry points and `main`."

**Impact**: An enterprise override to `SetDefaultConfig()` with invalid YAML crashes the process with no user-friendly error message.

**Recommended Fix**: Return `(*Config, error)` and let `main.go` handle the error.

**Estimated Complexity**: Small

---

## ISSUE-019: `golangci-lint` Config Excludes `pkg/service/utgen` — But Path Doesn't Exist

| Field | Value |
|---|---|
| **Severity** | Low |
| **Category** | CI/CD |
| **Location** | [.golangci.yml](file:///c:/Users/priya/OneDrive/Desktop/Coding/keploy/.golangci.yml) |
| **Code Reference** | `paths-except` config |

**Problem**: The AGENTS.md mentions `pkg/service/utgen` is excluded from linting, but the actual `.golangci.yml` only excludes eBPF generated files. The `utgen` command is "commented out" per AGENTS.md but may have residual files that should or shouldn't be linted. The `paths-except` field in `.golangci.yml` uses an inverted exclusion (it means "only run linters on these paths"), which is likely not the intended behavior.

**Impact**: Linter may not be covering the intended scope.

**Recommended Fix**: Verify linter scope; use `paths` (not `paths-except`) for exclusion if the intent is to skip certain directories.

**Estimated Complexity**: Small

---

## ISSUE-020: `Replayer.failedTCsBySetID` Written Without Mutex Protection

| Field | Value |
|---|---|
| **Severity** | Medium |
| **Category** | Concurrency |
| **Location** | [pkg/service/replay/replay.go#L302](file:///c:/Users/priya/OneDrive/Desktop/Coding/keploy/pkg/service/replay/replay.go#L302), [#L633](file:///c:/Users/priya/OneDrive/Desktop/Coding/keploy/pkg/service/replay/replay.go#L633) |
| **Code Reference** | `r.failedTCsBySetID = make(...)` and `r.failedTCsBySetID[testSet] = failedTcIDs` |

**Problem**: `failedTCsBySetID` is initialized (line 302) and written (line 633) without the `completeTestReportMu` mutex that protects sibling fields like `totalTests`. While currently accessed sequentially in the Start loop, the `--keep-app-alive` errgroup goroutine could theoretically access it concurrently.

**Impact**: Potential data race under `--keep-app-alive` mode, detectable by `-race`.

**Recommended Fix**: Protect `failedTCsBySetID` access with `completeTestReportMu`.

**Estimated Complexity**: Small

---

## ISSUE-021: HTTP Response Body Not Drained Before Close

| Field | Value |
|---|---|
| **Severity** | Medium |
| **Category** | Performance / Resource Leak |
| **Location** | Multiple files with `defer resp.Body.Close()` |
| **Code Reference** | [pkg/platform/http/agent.go](file:///c:/Users/priya/OneDrive/Desktop/Coding/keploy/pkg/platform/http/agent.go) (6 occurrences), [utils/utils.go#L535-L539](file:///c:/Users/priya/OneDrive/Desktop/Coding/keploy/utils/utils.go#L535-L539) |

**Problem**: Multiple HTTP client calls use `defer resp.Body.Close()` without first draining the body with `io.Copy(io.Discard, resp.Body)`. Go's HTTP transport reuses TCP connections only when the response body is fully read.

**Impact**: Under load, the agent HTTP client creates new TCP connections for every request instead of reusing them, increasing latency and resource consumption.

**Recommended Fix**: Add `defer io.Copy(io.Discard, resp.Body)` before `defer resp.Body.Close()` (or use a helper).

**Estimated Complexity**: Small

---

## ISSUE-022: Dockerfile Runtime Stage Uses `debian:trixie-slim` (Testing)

| Field | Value |
|---|---|
| **Severity** | Low |
| **Category** | Infrastructure |
| **Location** | [Dockerfile#L51](file:///c:/Users/priya/OneDrive/Desktop/Coding/keploy/Dockerfile#L51) |
| **Code Reference** | `FROM debian:trixie-slim` |

**Problem**: `trixie` is Debian's testing/unstable release (Debian 13). Using a testing distribution in production images means packages may change unexpectedly between builds, and security patches may lag behind stable releases.

**Impact**: Non-reproducible builds; potential for broken dependencies when trixie packages are updated.

**Recommended Fix**: Use `debian:bookworm-slim` (Debian 12 stable) or pin a specific trixie digest.

**Estimated Complexity**: Small

---

## ISSUE-023: Memory Monitor Goroutine Threshold May Cause Premature `FreeOSMemory`

| Field | Value |
|---|---|
| **Severity** | Low |
| **Category** | Performance |
| **Location** | [main.go#L110-L128](file:///c:/Users/priya/OneDrive/Desktop/Coding/keploy/main.go#L110-L128) |
| **Code Reference** | Memory monitor goroutine |

**Problem**: The memory monitor triggers `debug.FreeOSMemory()` when `HeapInuse < prevInUse/2` (50% drop). After a large test-set completes and heap naturally drops, this aggressively forces OS page release. `FreeOSMemory` is expensive and can pause all goroutines.

**Impact**: STW-like pauses during normal heap shrinkage between test-sets.

**Recommended Fix**: Add a minimum threshold (e.g., only trigger when `prevInUse > 100MB`) and increase the check interval to 30s.

**Estimated Complexity**: Small

---

## ISSUE-024: No Unit Tests for Core `record.go` and `replay.go` Logic

| Field | Value |
|---|---|
| **Severity** | Low |
| **Category** | Testing |
| **Location** | [pkg/service/record/](file:///c:/Users/priya/OneDrive/Desktop/Coding/keploy/pkg/service/record/), [pkg/service/replay/](file:///c:/Users/priya/OneDrive/Desktop/Coding/keploy/pkg/service/replay/) |
| **Code Reference** | No `record_test.go` or `replay_test.go` for core business logic |

**Problem**: The record service has 0 test files for its core logic. The replay service has test files only for utilities (`utils_test.go`, `health_poll_test.go`, `hooks_test.go`, `mockinfo_test.go`) but none for the 3770-line `replay.go`. Key logic like `GetNextTestSetID`, `RunTestSet`, flaky detection, and mock correlation are untested at the unit level.

**Impact**: Regressions can only be caught by e2e tests, which are slow and don't cover edge cases.

**Recommended Fix**: Add table-driven tests for pure logic functions: `GetNextTestSetID`, `shouldAbortTestRun`, `mapAppErrorToTestSetStatus`, `resolveTestSetStatus`, `containsStandalonePhrase`.

**Estimated Complexity**: Medium

---

## ISSUE-025: `Replayer` Struct Has 25+ Fields — Complexity Smell

| Field | Value |
|---|---|
| **Severity** | Low |
| **Category** | Maintainability |
| **Location** | [pkg/service/replay/replay.go#L148-L187](file:///c:/Users/priya/OneDrive/Desktop/Coding/keploy/pkg/service/replay/replay.go#L148-L187) |
| **Code Reference** | `type Replayer struct` |

**Problem**: The `Replayer` struct has 25+ fields spanning configuration, state tracking, statistics, concurrency guards, and feature flags. This violates the Single Responsibility Principle — one struct manages the entire replay lifecycle.

**Impact**: Difficult to test, difficult to reason about state transitions, and easy to introduce bugs when adding features.

**Recommended Fix**: Extract statistics into a `RunStatistics` struct, feature flags into `RunConfig`, and move test-set execution to a `TestSetRunner`.

**Estimated Complexity**: Large

---

*GitHub Match: NONE FOUND for all issues (repository-specific codebase issues)*
