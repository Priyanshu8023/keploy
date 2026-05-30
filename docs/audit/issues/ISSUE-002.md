# ISSUE-002: `ToAbsPath` Broken on Windows — Unix Path Assumption

## Summary
The `ToAbsPath` function in `utils/utils.go` uses `path[0] != '/'` to check if a path is absolute, which fails on Windows where absolute paths start with a drive letter (e.g., `C:\Users`). It also concatenates with Unix separator `/keploy`.

## Technical Explanation

### Current Code (Lines 747-765)
```go
func ToAbsPath(logger *zap.Logger, originalPath string) string {
    path := originalPath
    if len(path) > 0 && path[0] != '/' {       // ← BUG: Windows paths never start with '/'
        absPath, err := filepath.Abs(path)
        if err != nil {
            LogError(logger, err, "failed to get the absolute path from relative path")
        }
        path = absPath
    } else if len(path) == 0 {
        cdirPath, err := os.Getwd()
        if err != nil {
            LogError(logger, err, "failed to get the path of current directory")
        }
        path = cdirPath
    }
    path += "/keploy"                           // ← BUG: Unix separator
    return path
}
```

### Failure Flow on Windows
1. User provides `path = "C:\Users\project"` (absolute)
2. `path[0] = 'C'`, which is `!= '/'` → enters the "relative path" branch
3. `filepath.Abs("C:\Users\project")` returns `"C:\Users\project"` (no-op)
4. `path += "/keploy"` → `"C:\Users\project/keploy"` (mixed separators)

While `filepath.Abs` happens to handle this correctly, the logic is semantically wrong and would break for edge cases. More critically, the `/keploy` suffix creates a mixed-separator path.

## Root Cause
Function was written targeting Linux-first development and never updated for Windows cross-platform support, despite Windows being listed as a supported platform.

## Fix Strategy

### Suggested Patch
```go
func ToAbsPath(logger *zap.Logger, originalPath string) string {
    path := originalPath
    if len(path) > 0 && !filepath.IsAbs(path) {
        absPath, err := filepath.Abs(path)
        if err != nil {
            LogError(logger, err, "failed to get the absolute path from relative path")
        }
        path = absPath
    } else if len(path) == 0 {
        cdirPath, err := os.Getwd()
        if err != nil {
            LogError(logger, err, "failed to get the path of current directory")
        }
        path = cdirPath
    }
    path = filepath.Join(path, "keploy")
    return path
}
```

## Code Changes Required

| File | Change |
|---|---|
| `utils/utils.go#L750` | Replace `path[0] != '/'` with `!filepath.IsAbs(path)` |
| `utils/utils.go#L763` | Replace `path += "/keploy"` with `path = filepath.Join(path, "keploy")` |

## Testing Strategy

### Unit Tests
```go
func TestToAbsPath_Windows(t *testing.T) {
    logger := zap.NewNop()
    tests := []struct {
        input    string
        contains string
    }{
        {".", "keploy"},
        {"", "keploy"},
        {"relative/path", "keploy"},
    }
    for _, tt := range tests {
        result := ToAbsPath(logger, tt.input)
        assert.Contains(t, result, tt.contains)
        assert.True(t, filepath.IsAbs(result))
        // Verify no mixed separators
        if runtime.GOOS == "windows" {
            assert.NotContains(t, result, "/")
        }
    }
}
```

## Risk Assessment
- **Low risk**: `filepath.IsAbs` and `filepath.Join` are standard library functions
- **Rollback**: Simple revert if any path issues arise

## Validation Checklist
- [ ] Bug reproduced on Windows
- [ ] Fix implemented
- [ ] Tests added (cross-platform path tests)
- [ ] CI passed (Windows matrix)
- [ ] Manual verification on Windows
