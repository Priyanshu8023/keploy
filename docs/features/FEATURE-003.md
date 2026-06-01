# FEATURE-003: `keploy init` — Interactive Setup Wizard

## Feature Overview
An interactive CLI wizard that detects the user's application type, suggests configuration, and generates a ready-to-use `keploy.yml` with a single command. Replaces the current "read the docs → manually create config → trial and error" onboarding flow.

## Why This Feature Matters

### User Impact
- Time-to-first-test drops from ~1 hour to ~5 minutes
- Eliminates the most common support questions ("how do I configure Keploy for my app?")
- Provides immediate validation that the setup is correct

### Developer Impact
- Reduces issue/bug reports related to misconfiguration
- Creates a natural entry point for docs and tutorials

### Project Impact
- Critical for open-source adoption metrics (stars, first-time contributors, retention)
- Every competing tool has an init/create command — table stakes

## Functional Requirements

1. **Language Detection**: Auto-detect Go (go.mod), Node.js (package.json), Python (requirements.txt/pyproject.toml), Java (pom.xml/build.gradle)
2. **Command Suggestion**: Based on language, suggest the `command` field:
   - Go: `./app` or detected binary name
   - Node: `npm start` or detected start script
   - Python: `python app.py` or detected entry point
   - Java: `java -jar target/*.jar`
3. **Port Detection**: Scan common port patterns in source code (`:8080`, `:3000`, `PORT=`)
4. **Docker Detection**: If Dockerfile or docker-compose.yml exists, suggest Docker mode with container/network names
5. **Config Generation**: Write `keploy.yml` with detected values
6. **Validation**: Run `keploy doctor` checks after generation to confirm readiness
7. **Non-Interactive Mode**: `keploy init --yes` accepts all defaults for CI environments

## Non-Functional Requirements

- **Performance**: Complete in < 5 seconds on any project
- **Security**: Never read file contents beyond filenames; no network calls
- **Reliability**: Gracefully handle undetectable projects with sensible defaults
- **Scalability**: Plugin-based language detectors for community contributions

## Technical Design

### Components Affected
| Component | Change Type |
|-----------|-------------|
| `cli/init.go` | NEW — Cobra command definition |
| `pkg/service/init/` | NEW — Detection + generation logic |
| `pkg/service/init/detect.go` | NEW — Language/framework detection |
| `pkg/service/init/generate.go` | NEW — Config file generation |
| `cli/root.go` | MODIFY — Register `initCmd` |
| `config/default.go` | REFERENCE — Use existing default template |

### Architecture Diagram
```
User runs `keploy init`
        │
        ▼
┌──────────────────┐
│ DetectLanguage()  │ ← Scan for go.mod, package.json, etc.
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ DetectCommand()   │ ← Infer app start command from Makefile, scripts, etc.
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ DetectPorts()     │ ← Grep source for port patterns
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ DetectDocker()    │ ← Check for Dockerfile, docker-compose.yml
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ PromptUser()      │ ← Confirm/override detected values (skip with --yes)
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ GenerateConfig()  │ ← Write keploy.yml using config/default.go template
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ RunDoctor()       │ ← Validate environment (FEATURE-002)
└──────────────────┘
```

## Implementation Plan

### Step 1: Create detection module (2 hours)
- `pkg/service/init/detect.go` — File-based language/framework detection
- Functions: `DetectLanguage()`, `DetectCommand()`, `DetectPorts()`, `DetectDocker()`

### Step 2: Create config generator (2 hours)
- `pkg/service/init/generate.go` — Template-based config generation
- Use `config/default.go` as the base template
- Override detected fields
- Write to `keploy.yml` in current directory

### Step 3: Create CLI command (1 hour)
- `cli/init.go` — Cobra command with `--yes` flag
- Register in `cli/root.go`

### Step 4: Add interactive prompts (2 hours)
- Use `github.com/AlecAivazis/survey/v2` or simple stdin prompts
- Confirm language, command, port, Docker mode
- Skip with `--yes`

### Step 5: Integration with doctor (1 hour)
- After config generation, run environment checks
- Report any issues with fixes

## Code Locations

### New Files
```
cli/init.go                          # Cobra command (50 lines)
pkg/service/init/init.go             # Orchestrator (100 lines)
pkg/service/init/detect.go           # Detection logic (200 lines)
pkg/service/init/detect_test.go      # Unit tests (150 lines)
pkg/service/init/generate.go         # Config generator (80 lines)
```

### Modified Files
```
cli/root.go                          # Add initCmd registration (2 lines)
```

## Suggested Code Structure

```go
// pkg/service/init/detect.go
package init

type ProjectInfo struct {
    Language    string   // "go", "node", "python", "java", "unknown"
    Framework   string   // "gin", "express", "django", "spring", ""
    Command     string   // Suggested start command
    Port        uint32   // Detected port or 0
    HasDocker   bool
    DockerFile  string   // Path to Dockerfile
    ComposeFile string   // Path to docker-compose.yml
    ContainerName string
    NetworkName string
}

func Detect(dir string) (*ProjectInfo, error) {
    info := &ProjectInfo{}
    info.Language = detectLanguage(dir)
    info.Framework = detectFramework(dir, info.Language)
    info.Command = suggestCommand(dir, info.Language, info.Framework)
    info.Port = detectPort(dir, info.Language)
    info.HasDocker, info.DockerFile, info.ComposeFile = detectDocker(dir)
    return info, nil
}
```

## Testing Plan

### Unit Tests
- `detect_test.go`: Test each detector with fixture directories containing go.mod, package.json, etc.
- `generate_test.go`: Test config generation produces valid YAML

### Integration Tests
- Run `keploy init --yes` in a Go sample project directory
- Verify generated `keploy.yml` has correct language, command, and port
- Run `keploy init --yes` in an empty directory — verify graceful fallback

### Regression Tests
- Ensure existing `keploy.yml` is not overwritten without confirmation

## Risks

1. **False detection**: May detect wrong language in polyglot repos → Mitigated by user confirmation step
2. **Port detection**: Regex-based port scanning is heuristic → Accept imperfection; user can override

## Rollback Plan
- Command is additive (new files only). Remove `cli/init.go` and `pkg/service/init/` to revert.
- No changes to existing functionality.

## Acceptance Criteria
- [ ] `keploy init` generates a valid `keploy.yml` for Go projects
- [ ] `keploy init` generates a valid `keploy.yml` for Node.js projects
- [ ] `keploy init --yes` runs non-interactively
- [ ] Docker projects detected and configured correctly
- [ ] Existing `keploy.yml` prompts for confirmation before overwriting
- [ ] Unit tests for all detectors
- [ ] CI passes
- [ ] Documentation updated (README command table)
