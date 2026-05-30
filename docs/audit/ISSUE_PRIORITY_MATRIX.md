# Issue Priority Matrix

> **Priority Levels**: P0 = Immediate | P1 = High | P2 = Medium | P3 = Low

| Issue ID | Title | Severity | Business Impact | Effort | Priority |
|---|---|---|---|---|---|
| ISSUE-001 | Global Mutable State Race Conditions | Critical | Data corruption in concurrent test runs | Medium | **P0** |
| ISSUE-002 | `ToAbsPath` Broken on Windows | Critical | Windows platform completely broken | Small | **P0** |
| ISSUE-003 | `disableMapping: true` Default Contradicts CLI | Critical | Non-deterministic test results for all users | Small | **P0** |
| ISSUE-004 | Hardcoded GitHub Client ID | High | OAuth impersonation risk | Small | **P1** |
| ISSUE-005 | Default MongoDB Password in Config | High | Credential exposure in CI/CD logs | Small | **P1** |
| ISSUE-006 | GitHub Actions Writes to Root Path | High | Function completely broken | Small | **P1** |
| ISSUE-007 | `makeDirectory` Uses 0777 Permissions | High | Sensitive data exposed on shared systems | Small | **P1** |
| ISSUE-009 | replay.go 3770-Line God Object | High | Maintenance bottleneck, merge conflicts | Large | **P1** |
| ISSUE-010 | pkg/util.go 132KB Monolith | High | Change collision, review burden | Large | **P1** |
| ISSUE-012 | `os.Exit(1)` Bypasses Deferred Cleanup | High | Data loss on exit, telemetry lost | Medium | **P1** |
| ISSUE-014 | pprof Server Without Authentication | High | Runtime data leak in containers | Small | **P1** |
| ISSUE-008 | `DeleteFileIfNotExists` — Misleading Name | Medium | Contributor confusion, misuse risk | Small | **P2** |
| ISSUE-011 | Busy-Wait Polling in Mock Correlation | Medium | CPU overhead under high throughput | Medium | **P2** |
| ISSUE-013 | Bare `go func()` Without Recovery | Medium | Silent crashes from background panics | Medium | **P2** |
| ISSUE-015 | Regex Compilation Without Limit | Medium | ReDoS via crafted config | Medium | **P2** |
| ISSUE-016 | No Port Number Validation | Medium | Unhelpful runtime errors | Small | **P2** |
| ISSUE-017 | Sentry 100% Trace Sample Rate | Medium | Network overhead in high-throughput | Small | **P2** |
| ISSUE-018 | `config.New()` Panics on Parse Failure | Medium | Process crash without diagnostics | Small | **P2** |
| ISSUE-020 | `failedTCsBySetID` Race Condition | Medium | Data race under `--keep-app-alive` | Small | **P2** |
| ISSUE-021 | HTTP Response Body Not Drained | Medium | TCP connection leak under load | Small | **P2** |
| ISSUE-019 | Linter Config Exclusion Mismatch | Low | Linter scope may be wrong | Small | **P3** |
| ISSUE-022 | Dockerfile Uses debian:trixie (Testing) | Low | Non-reproducible builds | Small | **P3** |
| ISSUE-023 | Premature FreeOSMemory Calls | Low | Unnecessary GC pauses | Small | **P3** |
| ISSUE-024 | No Unit Tests for Core Logic | Low | Regressions caught only by slow e2e | Medium | **P3** |
| ISSUE-025 | `Replayer` Struct 25+ Fields | Low | Complexity barrier for contributors | Large | **P3** |

---

## Remediation Sequence

### Sprint 1 — Immediate (P0)
1. **ISSUE-002**: Fix `ToAbsPath` — 30-minute fix, unblocks Windows users
2. **ISSUE-003**: Change `disableMapping` default — 5-minute fix, major correctness impact
3. **ISSUE-001**: Thread-safe template store — ~2 day refactor

### Sprint 2 — High Priority (P1, small effort)
4. **ISSUE-004**: Move GitHub Client ID to ldflags
5. **ISSUE-005**: Remove default MongoDB password
6. **ISSUE-006**: Fix GitHub Actions path
7. **ISSUE-007**: Fix directory permissions to 0755
8. **ISSUE-014**: Add pprof authentication
9. **ISSUE-012**: Replace `os.Exit` with error returns

### Sprint 3 — High Priority (P1, large effort)
10. **ISSUE-009**: Split `replay.go` into focused modules
11. **ISSUE-010**: Split `pkg/util.go` into domain packages

### Sprint 4 — Medium Priority (P2)
12. Issues 008, 011, 013, 015–018, 020–021

### Backlog (P3)
13. Issues 019, 022–025
