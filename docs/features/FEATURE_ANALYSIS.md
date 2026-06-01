# FEATURE_ANALYSIS.md — Keploy Repository Analysis

## Phase 1: Repository Understanding

### Project Purpose

**What problem does Keploy solve?**
Keploy is an open-source backend testing tool that **records real API and dependency traffic** from a running application and **replays it as deterministic tests with mocks**. It intercepts traffic at the network layer using eBPF (Linux) and a userspace proxy (macOS/Windows), so applications don't need an SDK or code changes. This eliminates the manual authoring of test cases and mock fixtures — the two highest-friction activities in backend testing.

**Who are the users?**
- **Backend engineers** who want automated regression tests without writing boilerplate
- **Platform/DevOps teams** integrating testing into CI/CD pipelines
- **QA engineers** validating API behavior without deep code knowledge
- **Architects** validating contract compliance between microservices
- **Open-source contributors** building testing infrastructure

**Target audience:**
- Teams running Go, Node.js, Python, or Java backends
- Microservice architectures with HTTP, gRPC, PostgreSQL, MySQL, MongoDB, Redis, Kafka dependencies
- Organizations adopting shift-left testing or seeking higher code coverage with less effort

**Core workflows:**
1. **Record** (`keploy record -c "<app cmd>"`): Run the app, capture dependency traffic into `./keploy/test-set-*`
2. **Replay/Test** (`keploy test -c "<app cmd>"`): Replay recorded calls, mock dependencies, generate reports
3. **Re-record** (`keploy rerecord`): Re-record against new code to pick up accepted changes
4. **Normalize** (`keploy normalize`): Accept newly-observed responses into golden test cases
5. **Contract Testing** (`keploy contract`): OpenAPI contract generation and cross-service validation
6. **Sanitize** (`keploy sanitize`): Scrub secrets from recorded test data
7. **Templatize** (`keploy templatize`): Replace dynamic values with templates
8. **Report** (`keploy report`): Summarize test run results
9. **Diff** (`keploy diff`): Compare two test runs
10. **Export/Import** (`keploy export/import`): Move test sets between repos

---

### Current Capabilities

#### Existing Commands
| Command | Package | Status |
|---------|---------|--------|
| `record` | `pkg/service/record` | ✅ Active |
| `test` | `pkg/service/replay` | ✅ Active |
| `rerecord` | `pkg/service/orchestrator` | ✅ Active |
| `normalize` | `pkg/service/tools` | ✅ Active |
| `sanitize` | `pkg/service/tools` | ✅ Active |
| `templatize` | `pkg/service/tools` | ✅ Active |
| `config --generate` | `cli/config.go` | ✅ Active |
| `contract` | `pkg/service/contract` | ✅ Active |
| `diff` | `pkg/service/diff` | ✅ Active |
| `report` | `pkg/service/report` | ✅ Active |
| `export` / `import` | `cli/export.go`, `cli/import.go` | ✅ Active |
| `update` | `cli/update.go` | ✅ Active |
| `agent` | `cli/agent.go` | ✅ Internal |
| `gen` (utgen) | `pkg/service/utgen` | ❌ Disabled (CLI commented out) |

#### Protocol Support
| Protocol | OSS Binary | Enterprise |
|----------|-----------|------------|
| HTTP/1.1 | ✅ | ✅ |
| MySQL | ✅ | ✅ |
| PostgreSQL | ❌ | ✅ (V3 parser) |
| MongoDB | ❌ | ✅ |
| gRPC | ❌ | ✅ |
| HTTP/2 | ❌ | ✅ |
| Kafka | ❌ | ✅ |
| Redis | ❌ | ✅ |
| DNS | ✅ | ✅ |
| Generic (binary) | ✅ | ✅ |

#### Platform Support
| Platform | Native | Docker |
|----------|--------|--------|
| Linux x86_64 | ✅ (eBPF) | ✅ |
| Linux arm64 | ✅ (eBPF) | ✅ |
| Windows amd64 | ✅ (WinDivert) | ✅ |
| Windows arm64 | ❌ | ✅ |
| macOS | ❌ | ✅ |

#### Existing Integrations
- **CI/CD**: GitHub Actions workflow generation (`--generateGithubActions`)
- **Docker**: Native Docker and Docker Compose support
- **Coverage**: Go, Java (JaCoCo), Python, JavaScript, C# coverage
- **Telemetry**: Sentry error tracking, custom telemetry
- **Profiling**: pprof HTTP server, CPU/Heap profiles
- **Storage**: YAML and JSON on-disk format, gob binary for mocks
- **Reporting**: Text, JSON, and JUnit XML output

---

### Architecture Review

#### Folder Structure
```
keploy/
├── main.go                    # Entry point → cli.Root(...)
├── cli/                       # Cobra command definitions
│   ├── root.go                # Root command
│   ├── record.go, test.go     # User-facing commands
│   └── provider/              # DI wiring, flag binding, validation
│       ├── cmd.go             # CmdConfigurator (1600+ lines)
│       └── core_service.go    # Service provider wiring
├── config/                    # Config structs + defaults
├── pkg/
│   ├── agent/                 # Agent subprocess (eBPF hooks, proxy, routes)
│   │   ├── hooks/             # eBPF + platform-specific hooks
│   │   ├── proxy/             # TLS interception proxy
│   │   ├── routes/            # Agent HTTP API routes
│   │   └── memoryguard/       # Memory pressure management
│   ├── core/proxy/            # Core proxy layer (TLS)
│   ├── models/                # Domain models (44 files, massive)
│   │   ├── mock.go            # Mock struct (835 lines)
│   │   ├── testrun.go         # TestResult, TestReport
│   │   └── postgres_v3_*.go   # PostgresV3 cell/spec types
│   ├── matcher/               # Response comparison engine
│   │   ├── http/, grpc/       # Protocol-specific matchers
│   │   ├── schema/            # Schema comparison
│   │   └── risk.go            # Risk assessment logic
│   ├── service/               # Business logic layer
│   │   ├── record/            # Recording orchestration
│   │   ├── replay/            # Replay orchestration (143K replay.go!)
│   │   ├── report/            # Report generation
│   │   ├── contract/          # Contract testing
│   │   ├── diff/              # Test run diffing
│   │   ├── tools/             # Normalize/sanitize/templatize
│   │   └── export/, import/   # Test set portability
│   ├── platform/              # Infrastructure adapters
│   │   ├── yaml/              # On-disk YAML persistence
│   │   ├── telemetry/         # Analytics
│   │   ├── coverage/          # Multi-language code coverage
│   │   ├── storage/           # Cloud storage (upload/download)
│   │   ├── docker/            # Docker interaction helpers
│   │   └── http/              # HTTP client utilities
│   └── util.go                # Shared utilities (132K — too large)
├── utils/                     # Process-level utilities (signals, permissions)
└── .github/workflows/         # 43 CI workflow files
```

#### Key Extension Points
1. **Protocol parsers**: New protocol support via `pkg/models/mock.go` Kind enum + proxy integration
2. **Coverage drivers**: `pkg/platform/coverage/` — language-specific subdirectories
3. **Storage backends**: `pkg/platform/yaml/` — could be extended with DB backends
4. **Report formats**: `pkg/service/report/` — JUnit, text, JSON — extensible pattern
5. **Matchers**: `pkg/matcher/` — schema, HTTP, gRPC comparison
6. **Test hooks**: `pkg/service/replay/service.go` `TestHooks` interface — powerful extension point
7. **CLI commands**: `cli/` — Cobra-based, straightforward to add new commands

#### Architecture Observations
- **Monolithic service layer**: `replay.go` is 143K (3500+ lines) — a significant code smell
- **God object utility files**: `pkg/util.go` is 132K, `utils/utils.go` is 50K — extremely large
- **Interface-driven DI**: Clean separation via consumer-side interfaces (`TestDB`, `MockDB`, `ReportDB`, etc.)
- **Errgroup lifecycle**: Canonical goroutine management pattern throughout
- **Context-driven cancellation**: Proper `utils.NewCtx()` with SIGINT/SIGTERM handling

---

## Phase 2: Gap Analysis

### User Gaps

| Gap | Description | Severity |
|-----|------------|----------|
| **No Web UI/Dashboard** | No visual interface for browsing test results, comparing runs, or managing test sets. Users must parse YAML files or CLI output. | 🔴 Critical |
| **No Real-Time Test Progress** | During `keploy test`, users see a wall of text. No progress bar, no live status, no ETA. | 🟡 High |
| **No Test Search/Filter** | Can't search across test cases by endpoint, status code, or response content. Must manually browse YAML files. | 🟡 High |
| **No Notifications** | No Slack/email/webhook notification when test runs complete in CI. | 🟡 High |
| **No Interactive Mode** | No TUI for selecting which test sets to run, which tests to skip, or which failures to normalize. | 🟠 Medium |
| **No Test Analytics/Trends** | No ability to track test pass rates over time, identify flaky tests, or measure coverage trends. | 🟡 High |
| **Limited Report Formats** | Only text, JSON, and JUnit. No HTML report, no Markdown for PR comments, no SARIF. | 🟠 Medium |
| **No Watch Mode** | Can't auto-re-run tests when source code or test files change. | 🟠 Medium |
| **No Selective Re-recording** | Can't re-record just one specific test case; must re-record the entire test set. | 🟠 Medium |
| **No Test Case Tagging** | Can't tag tests as "smoke", "regression", "critical" and filter by tag during replay. | 🟡 High |

### Developer Gaps

| Gap | Description | Severity |
|-----|------------|----------|
| **No Language SDKs** | No Go/Python/Node/Java SDK for programmatic test management. Everything is CLI-only. | 🟡 High |
| **No REST/gRPC API** | No server mode with an API for external tool integration (IDE plugins, CI systems, dashboards). | 🔴 Critical |
| **No IDE Integration** | No VS Code / JetBrains plugin for viewing tests, running replay from the editor, or inline diff. | 🟡 High |
| **No OpenTelemetry Integration** | Can't correlate test failures with trace/span data from OTel instrumentation. | 🟠 Medium |
| **No Plugin System** | Can't extend Keploy with custom matchers, custom report formats, or custom storage backends without forking. | 🟠 Medium |
| **Missing `keploy validate` Command** | No way to validate test/mock YAML files for schema correctness without running a full replay. | 🟠 Medium |
| **No `keploy doctor` Command** | No diagnostic tool to check if the environment is correctly configured (eBPF, Docker, permissions). | 🟡 High |
| **Sparse Unit Tests** | Coverage is primarily e2e; many core packages lack unit test coverage. | 🟡 High |
| **Undocumented Internal APIs** | Agent HTTP routes, mock DB interfaces, and hook lifecycle are not documented for contributors. | 🟠 Medium |
| **No Changelog Generation** | No automated changelog from conventional commits despite using `commitizen`. | 🟠 Medium |

### Enterprise Gaps

| Gap | Description | Severity |
|-----|------------|----------|
| **No RBAC** | No role-based access control for multi-user test management. | 🟠 Medium |
| **No Audit Logging** | No record of who ran what tests, who normalized failures, or who edited test data. | 🟡 High |
| **No SSO/OIDC** | No authentication/authorization integration. | 🟠 Medium |
| **No Multi-Tenancy** | No support for multiple teams/projects sharing a Keploy instance. | 🟠 Medium |
| **No Usage Analytics** | No metrics on recording frequency, test execution patterns, or team adoption. | 🟡 High |
| **No Compliance Reports** | No PCI/SOC2/HIPAA-style compliance report generation for test coverage. | 🟠 Medium |
| **No Test Retention Policies** | Old test runs accumulate forever. No TTL or automatic cleanup. | 🟡 High |
| **No Parallel Test Execution** | Test sets run sequentially. No parallelism for large test suites. | 🔴 Critical |
| **No Secrets Vault Integration** | Sanitize uses regex patterns. No integration with HashiCorp Vault, AWS Secrets Manager, etc. | 🟠 Medium |

### Open Source Gaps

| Gap | Description | Severity |
|-----|------------|----------|
| **No Interactive Getting Started** | No `keploy init` wizard that detects app type, suggests config, and generates a minimal keploy.yml. | 🟡 High |
| **No Sample App Templates** | Samples live in separate repos. No `keploy create --template go-gin-postgres` command. | 🟠 Medium |
| **No Contribution Guide for Protocols** | Adding a new protocol requires deep tribal knowledge. No step-by-step guide. | 🟡 High |
| **No GitHub App** | No GitHub App for automatic PR comments with test results, coverage diffs, or failing test summaries. | 🟡 High |
| **No Ecosystem Badges** | No `keploy test` badge for README files. | 🟢 Low |
| **No Schema Migration Guide** | No documented process for upgrading test/mock YAML schemas between Keploy versions. | 🟠 Medium |
| **Outdated Multilingual READMEs** | Spanish, French, Japanese READMEs exist but are likely stale. | 🟢 Low |
