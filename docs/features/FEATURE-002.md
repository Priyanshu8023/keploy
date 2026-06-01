# FEATURE-002: `keploy doctor` — Environment Diagnostics

## Feature Overview
A single command that validates the entire Keploy environment and reports issues with actionable fixes, modeled after `flutter doctor` and `brew doctor`.

## Why This Feature Matters
- **User Impact**: Instant answer to "is my environment set up correctly?" — the #1 support question
- **Developer Impact**: Reduces issue triage time by 80%+ (users can self-diagnose)
- **Project Impact**: Every mature CLI tool has this. Table stakes for professional tooling.

## Functional Requirements
1. Check OS compatibility and version
2. Check kernel version (Linux: ≥ 5.8 for eBPF)
3. Check Docker availability and running status
4. Check Docker socket permissions
5. Validate port availability (proxy: 16789, DNS: 26789, agent)
6. Validate `keploy.yml` if present (FEATURE-016 overlap)
7. Check for conflicting processes on Keploy ports
8. Check disk space in `./keploy/` directory
9. Check Go version if language is Go
10. Output: checklist with ✅/⚠️/❌ status + fix instructions

## Technical Design

### New Files
- `cli/doctor.go` — Cobra command (~40 lines)
- `pkg/service/doctor/doctor.go` — Check orchestrator (~80 lines)  
- `pkg/service/doctor/checks.go` — Individual check functions (~200 lines)
- `pkg/service/doctor/checks_linux.go` — Linux-specific (eBPF, kernel)
- `pkg/service/doctor/checks_windows.go` — Windows-specific (WinDivert)
- `pkg/service/doctor/checks_darwin.go` — macOS-specific (Docker only)

### Modified Files
- `cli/root.go` — Register `doctorCmd` (~2 lines)

### Example Output
```
Keploy Doctor — Environment Health Check
=========================================

✅ Operating System    — Linux 5.15.0 (compatible)
✅ Kernel Version      — 5.15.0-91-generic (≥ 5.8 required for eBPF)
✅ Docker              — Docker 24.0.7, running
✅ Docker Socket       — /var/run/docker.sock (accessible)
⚠️ Port 16789          — In use by process 'nginx' (PID 1234)
                         Fix: Change proxyPort in keploy.yml or stop nginx
✅ Port 26789          — Available
✅ Config File         — keploy.yml found, valid syntax
❌ Disk Space          — Only 500MB free in ./keploy/ (recommend 2GB+)
                         Fix: Run `keploy clean` or free disk space
✅ Go Version          — go1.22.0 (compatible)

Summary: 7/9 checks passed, 1 warning, 1 failure
```

## Implementation Plan
1. Create check interface: `type Check struct { Name, Status, Message, Fix string }`
2. Implement each check as a standalone function
3. Use build tags for OS-specific checks
4. Wire into Cobra command
5. Add `--json` flag for machine-readable output

## Acceptance Criteria
- [ ] `keploy doctor` runs successfully on Linux, Windows, macOS
- [ ] Each check provides actionable fix instructions on failure
- [ ] `keploy doctor --json` outputs structured JSON
- [ ] Tests cover each check function individually
- [ ] CI passes on all platforms
