# Priority of Medium Issues (No Architecture Changes)

Based on the audit file (`ISSUES.md`), here are the top Medium severity issues that are purely internal code improvements. They require **zero architectural changes, no new environment variables, and no `.yml` config changes**.

## 1. ISSUE-008: Misleading Function Name (`DeleteFileIfNotExists`)
- **The Problem:** In `utils/utils.go`, there is a function named `DeleteFileIfNotExists`. However, the code actually deletes the file if it **DOES** exist. The name is the exact opposite of what it does, which is confusing for developers.
- **The Fix:** Rename the function to `DeleteFileIfExists` and update the two places in `main.go` where it is called.
- **Why it fits:** It's just a simple rename refactor. No logic changes, no config changes.

## 2. ISSUE-021: HTTP Response Body Not Drained Before Close
- **The Problem:** In several places (like `pkg/platform/http/agent.go`), the code does `defer resp.Body.Close()` without fully reading the response body first. In Go, if you don't drain the response body, the underlying TCP connection cannot be reused, leading to resource leaks under heavy load.
- **The Fix:** Add `defer io.Copy(io.Discard, resp.Body)` right before `defer resp.Body.Close()`.
- **Why it fits:** It's a standard Go performance fix completely isolated inside existing functions.

## 3. ISSUE-020: Missing Mutex Protection on `failedTCsBySetID`
- **The Problem:** In `pkg/service/replay/replay.go`, the map `r.failedTCsBySetID` is written to without using a mutex lock, which can cause a data race if `--keep-app-alive` is running concurrently.
- **The Fix:** Wrap the map assignment inside `r.completeTestReportMu.Lock()` and `Unlock()` (which is a mutex the struct already has).
- **Why it fits:** Fixes a concurrency bug using an existing lock. No structural changes needed.

## 4. ISSUE-017: `Sentry.Init` with `TracesSampleRate: 1.0`
- **The Problem:** In `utils/utils.go`, Sentry telemetry is initialized with a sample rate of `1.0` (100%). For a tool that intercepts heavy API traffic, this causes unnecessary performance overhead and network usage.
- **The Fix:** Change `1.0` to a lower hardcoded default like `0.1` (10%).
- **Why it fits:** Literally a one-character change.
