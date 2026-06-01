# FEATURE-005: Test Tagging & Filtering System

## Feature Overview
Add metadata tags to test cases during recording and filter by tags during replay. Enables tiered test execution (smoke/regression/full), priority-based CI pipelines, and organizational categorization.

## Why This Feature Matters
- **User Impact**: Run only "smoke" tests in PR CI (30s) and full suite on merge (5min). `keploy test --tags critical,smoke`
- **Business Impact**: Reduces CI costs by 60-80% for PR pipelines. Enables enterprise-scale test management.

## Functional Requirements
1. **Tag during record**: `keploy record -c "..." --tags "smoke,api,v2"` — tags all tests in this session
2. **Tag in config**: `keploy.yml` supports per-test-set tag defaults
3. **Tag in YAML**: Each `test-*.yaml` has a `tags: [smoke, critical]` field
4. **Filter during test**: `keploy test --tags smoke` runs only tagged tests
5. **Exclude tags**: `keploy test --exclude-tags slow,flaky`
6. **Tag expressions**: `keploy test --tags "smoke AND NOT flaky"`
7. **Retro-tag**: `keploy tag --test-set test-set-0 --test test-1 --add critical`

## Technical Design

### Model Changes
```go
// pkg/models/testcase.go
type TestCase struct {
    // ... existing fields
    Tags []string `json:"tags,omitempty" yaml:"tags,omitempty"`
}
```

### Config Changes
```go
// config/config.go
type Record struct {
    // ... existing fields
    Tags []string `json:"tags" yaml:"tags" mapstructure:"tags"`
}
type Test struct {
    // ... existing fields
    Tags        []string `json:"tags" yaml:"tags" mapstructure:"tags"`
    ExcludeTags []string `json:"excludeTags" yaml:"excludeTags" mapstructure:"excludeTags"`
}
```

### New Files
- `cli/tag.go` — `keploy tag` command for retro-tagging
- `pkg/service/tools/tag.go` — Tag manipulation logic

### Modified Files
- `pkg/models/testcase.go` — Add `Tags` field
- `config/config.go` — Add tag fields
- `cli/provider/cmd.go` — Add `--tags`, `--exclude-tags` flags
- `pkg/service/record/record.go` — Propagate tags to recorded test cases
- `pkg/service/replay/replay.go` — Filter tests by tags before execution
- `pkg/platform/yaml/testdb/testdb.go` — Include tags in YAML serialization

### Filter Logic (in replay.go)
```go
func filterByTags(tests []*models.TestCase, include, exclude []string) []*models.TestCase {
    var filtered []*models.TestCase
    for _, tc := range tests {
        if len(include) > 0 && !hasAnyTag(tc.Tags, include) { continue }
        if len(exclude) > 0 && hasAnyTag(tc.Tags, exclude) { continue }
        filtered = append(filtered, tc)
    }
    return filtered
}
```

## Acceptance Criteria
- [ ] Tags persisted in test YAML files
- [ ] `--tags` filters tests during replay
- [ ] `--exclude-tags` excludes tagged tests
- [ ] `keploy tag` command adds/removes tags retroactively
- [ ] Backward compatible — existing tests without tags run normally
- [ ] Tests added, CI passes
