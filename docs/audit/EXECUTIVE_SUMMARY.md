# Executive Summary — Keploy Repository Audit

**Date**: 2026-05-30  
**Scope**: Full repository audit of `keploy/keploy` (`go.keploy.io/server/v3`)  
**Branch**: `main`

---

## Repository Health Score: **62/100**

| Dimension | Score | Weight | Weighted |
|---|---|---|---|
| **Security** | 55/100 | 20% | 11.0 |
| **Performance** | 65/100 | 15% | 9.75 |
| **Reliability** | 60/100 | 20% | 12.0 |
| **Scalability** | 55/100 | 10% | 5.5 |
| **Testing** | 45/100 | 15% | 6.75 |
| **Maintainability** | 50/100 | 20% | 10.0 |
| **Total** | | **100%** | **55.0** |

> Adjusted score with qualitative factors (active development, CI coverage): **62/100**

---

## Critical Findings

### 🔴 P0 — Requires Immediate Action

| # | Finding | Impact |
|---|---|---|
| 1 | **Global mutable state (`TemplatizedValues`, `ErrCode`) is unprotected** — concurrent goroutines read/write shared maps, triggering data races under `-race` and causing non-deterministic test results | Test result corruption |
| 2 | **`ToAbsPath` is broken on Windows** — Unix path separator assumptions (`/`) and `path[0] != '/'` check mean all Windows path resolution is incorrect | Windows platform unusable |
| 3 | **Default `disableMapping: true` silently degrades test accuracy** — the superior mapping-based mock filtering is disabled by default, causing timestamp-based fallback with known race-window issues | Flaky tests for all users |

### 🟠 P1 — High Priority

| # | Finding | Impact |
|---|---|---|
| 4 | **pprof server exposes unauthed profiling endpoints** in containerized environments | Information disclosure |
| 5 | **`makeDirectory` creates 0777-permission directories** containing captured API traffic (headers, tokens) | Sensitive data exposure |
| 6 | **`os.Exit(1)` calls bypass deferred cleanup** — log files truncated, telemetry lost, temp files orphaned | Data loss |
| 7 | **`GenerateGithubActions` writes to root `/githubactions/`** instead of `.github/workflows/` | Feature completely broken |
| 8 | **Hardcoded credentials**: GitHub Client ID in source, default MongoDB password `"default@123"` | Credential exposure |

---

## Estimated Technical Debt

### Level: **High**

**Justification**:

1. **Monolithic files**: Two files exceed 100KB (`pkg/util.go` at 132KB, `pkg/models/postgres_v3_cell.go` at 115KB). The core replay logic is a single 3770-line file. These files are change-collision hotspots requiring ~2 sprint-worth of decomposition work.

2. **Global mutable state**: Package-level mutable maps used for template values and secret values across goroutine boundaries. Fixing this requires a multi-file refactor to thread-safe storage.

3. **Cross-platform gaps**: Path handling assumes Unix semantics despite Windows being an officially supported platform. The umask management (`utils.SetUmask()`) is stubbed on Windows but the path handling is actively wrong, not just stubbed.

4. **Missing unit tests**: Core business logic (`record.go` Start flow, `replay.go` RunTestSet, mock correlation, flaky detection) has zero unit test coverage. All coverage comes from e2e tests which are slow, flaky, and can't cover edge cases.

5. **Deprecated flags still wired**: `FallBackOnMiss` is documented as deprecated but still has CLI flag registration, config struct fields, and conditional logic — dead code that increases maintenance burden.

6. **Commented-out code**: Large block of commented-out Sentry logger integration (~40 lines in `utils.go`), commented-out `utgen` command registration.

### Estimated Remediation Effort
- **P0 fixes**: 3-4 developer-days
- **P1 fixes (small)**: 2-3 developer-days
- **P1 fixes (large — file decomposition)**: 2-3 developer-weeks
- **P2 fixes**: 1-2 developer-weeks
- **Total**: ~4-6 developer-weeks for comprehensive remediation

---

## Production Readiness

### Assessment: **Needs Work**

**Reasoning**:

✅ **Strengths**:
- Well-structured CI with backwards/forwards compatibility matrix
- Comprehensive protocol support (HTTP, gRPC, PostgreSQL, MySQL, MongoDB, Redis, DNS)
- Multi-platform support (Linux, Windows, macOS via Docker)
- Good use of `errgroup` for goroutine lifecycle in most places
- Active maintenance with extensive documentation (AGENTS.md)
- Sentry integration for crash reporting
- Clean separation between service interfaces and platform implementations

❌ **Blockers for Enterprise Deployment**:
- **Data race on global state** (ISSUE-001) — will cause sporadic failures under concurrent load
- **Windows path resolution broken** (ISSUE-002) — one of three supported platforms is non-functional for core path logic
- **Security posture** — 0777 permissions, unauthenticated pprof, hardcoded credentials
- **No unit tests for core logic** — regressions can only be caught by 30+ minute e2e runs
- **Single 3770-line file** managing all test-set lifecycle — any bug fix risks introducing new regressions

⚠️ **Concerns**:
- Busy-wait polling patterns that don't scale
- HTTP response bodies not drained (TCP connection leak under load)
- 100% Sentry trace sampling rate in production
- `panic()` in config initialization (before any recovery handler)
- Dockerfile uses Debian testing (trixie) — not suitable for production images

---

## Recommendations

### Immediate (This Week)
1. Fix `ToAbsPath` for Windows — use `filepath.IsAbs()` + `filepath.Join()`
2. Change `disableMapping` default to `false`
3. Wrap `TemplatizedValues`/`SecretValues` in `sync.RWMutex` or move to per-context storage

### Short-Term (This Month)
4. Fix all P1 security issues (pprof auth, permissions, credentials)
5. Replace `os.Exit(1)` with error returns
6. Add unit tests for `GetNextTestSetID`, `shouldAbortTestRun`, `resolveTestSetStatus`, `containsStandalonePhrase`

### Medium-Term (This Quarter)
7. Decompose `replay.go` into 4-5 focused files
8. Split `pkg/util.go` into domain-specific packages
9. Add `go vet -race` to CI for core service tests
10. Replace busy-wait polling with channel-based coordination

### Long-Term (Next Quarter)
11. Replace global state entirely with dependency-injected stores
12. Add structured error types for all user-facing error paths
13. Implement connection reuse for agent HTTP client
14. Migration from Debian trixie to bookworm-slim in Dockerfile
