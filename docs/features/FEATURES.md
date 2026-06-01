# FEATURES.md — Master Feature Catalog

> **25 features** identified through repository analysis, competitive research, and gap analysis. Each feature is grounded in the actual Keploy architecture.

---

## FEATURE-001: Web Dashboard for Test Results
| Field | Value |
|-------|-------|
| **Category** | Product / UX |
| **Problem Solved** | Users must parse YAML files or CLI text output to understand test results. No visual way to browse, compare, or manage test runs. |
| **User Benefit** | Visual test result exploration with diffs, trend charts, and one-click normalization. Reduces time to understand failures from minutes to seconds. |
| **Business Benefit** | Dramatically improves adoption by non-CLI users (QA, PMs). Enables team collaboration around test results. Key differentiator vs. CLI-only competitors. |
| **Similar Implementations** | WireMock Admin UI, Speedscale Dashboard, Pact Broker UI, Allure Reports |
| **Complexity** | 🔴 Large |
| **Impact Score** | 10/10 |
| **Implementation Effort** | 3-4 weeks |

---

## FEATURE-002: `keploy doctor` — Environment Diagnostics
| Field | Value |
|-------|-------|
| **Category** | DX |
| **Problem Solved** | Users encounter cryptic errors when eBPF, Docker, or permissions are misconfigured. No guided troubleshooting. |
| **User Benefit** | Single command validates the entire environment: kernel version, eBPF support, Docker socket, permissions, port availability, config validity. |
| **Business Benefit** | Reduces support burden. Faster onboarding for new users. Common in mature CLI tools (brew doctor, flutter doctor). |
| **Similar Implementations** | `flutter doctor`, `brew doctor`, `docker info`, `npx envinfo` |
| **Complexity** | 🟢 Small |
| **Impact Score** | 8/10 |
| **Implementation Effort** | 2-3 days |

---

## FEATURE-003: `keploy init` — Interactive Setup Wizard
| Field | Value |
|-------|-------|
| **Category** | DX / Onboarding |
| **Problem Solved** | New users must manually create `keploy.yml`, guess the right flags, and understand the project structure. |
| **User Benefit** | Interactive wizard that detects app language, suggests `command`, configures ports, creates `keploy.yml`, and generates a sample test. |
| **Business Benefit** | Reduces time-to-first-test from hours to minutes. Critical for open-source adoption. |
| **Similar Implementations** | `npm init`, `vite create`, `go mod init`, `pnpm create` |
| **Complexity** | 🟡 Medium |
| **Impact Score** | 9/10 |
| **Implementation Effort** | 3-5 days |

---

## FEATURE-004: Parallel Test Set Execution
| Field | Value |
|-------|-------|
| **Category** | Performance |
| **Problem Solved** | Test sets run sequentially. Large suites with 10+ test sets take 10x longer than necessary. |
| **User Benefit** | N-way parallel test set execution with configurable concurrency. 3-5x faster test runs for typical suites. |
| **Business Benefit** | Critical for CI/CD pipeline speed. Reduces feedback loop time. Enables adoption in larger organizations. |
| **Similar Implementations** | `go test -parallel`, `pytest-xdist`, Jest `--maxWorkers` |
| **Complexity** | 🟡 Medium |
| **Impact Score** | 9/10 |
| **Implementation Effort** | 1-2 weeks |

---

## FEATURE-005: Test Tagging & Filtering System
| Field | Value |
|-------|-------|
| **Category** | Product |
| **Problem Solved** | Can't categorize tests as "smoke", "regression", "critical", "slow", "flaky". Can only filter by test-set name. |
| **User Benefit** | Tag tests during recording, filter by tag during replay. `keploy test --tags smoke,critical`. |
| **Business Benefit** | Enables tiered test execution in CI (smoke in PR, full in merge). Reduces CI costs. |
| **Similar Implementations** | Jest `--testPathPattern`, pytest `-m`, Go build tags, JUnit `@Tag` |
| **Complexity** | 🟡 Medium |
| **Impact Score** | 8/10 |
| **Implementation Effort** | 3-5 days |

---

## FEATURE-006: GitHub App for PR Test Comments
| Field | Value |
|-------|-------|
| **Category** | DX / Integration |
| **Problem Solved** | CI test results are buried in workflow logs. No PR-level visibility of which tests passed/failed, what changed, or coverage delta. |
| **User Benefit** | Automatic PR comment with test summary table, failing test diffs, coverage change, and risk assessment. |
| **Business Benefit** | Increases test visibility for code reviewers. Reduces context switching. Competitive parity with Codecov, SonarQube. |
| **Similar Implementations** | Codecov GitHub App, SonarQube PR Decoration, Pact Broker GitHub Checks |
| **Complexity** | 🟡 Medium |
| **Impact Score** | 9/10 |
| **Implementation Effort** | 1-2 weeks |

---

## FEATURE-007: REST API Server Mode
| Field | Value |
|-------|-------|
| **Category** | DX / Infrastructure |
| **Problem Solved** | Keploy is CLI-only. External tools (IDEs, dashboards, CI systems) can't interact with test data programmatically. |
| **User Benefit** | `keploy serve` starts an HTTP API server. CRUD operations on test sets, test cases, mocks, reports. WebSocket for live test progress. |
| **Business Benefit** | Enables ecosystem growth: IDE plugins, web dashboards, third-party integrations, enterprise workflows. |
| **Similar Implementations** | WireMock Admin API (`/__admin/`), Hoverfly Admin API, Pact Broker API |
| **Complexity** | 🔴 Large |
| **Impact Score** | 10/10 |
| **Implementation Effort** | 2-3 weeks |

---

## FEATURE-008: VS Code Extension
| Field | Value |
|-------|-------|
| **Category** | DX |
| **Problem Solved** | Developers must switch to terminal to run tests, view results, manage test data. No IDE integration. |
| **User Benefit** | Record/replay from VS Code, inline test result diffs, CodeLens for test coverage, test explorer integration. |
| **Business Benefit** | VS Code is the most popular editor. IDE integration is the #1 DX investment for adoption. |
| **Similar Implementations** | Jest Extension, Go Test Explorer, Python Test Adapter |
| **Complexity** | 🔴 Large |
| **Impact Score** | 8/10 |
| **Implementation Effort** | 3-4 weeks (requires FEATURE-007) |

---

## FEATURE-009: Test Analytics & Trend Dashboard
| Field | Value |
|-------|-------|
| **Category** | Analytics |
| **Problem Solved** | No historical visibility into test pass rates, flaky test identification, coverage trends, or failure patterns. |
| **User Benefit** | Track pass/fail rates over time. Auto-identify flaky tests (passes sometimes, fails sometimes). Coverage trend graphs. |
| **Business Benefit** | Data-driven testing decisions. Justifies testing investment to management. Identifies reliability issues before production. |
| **Similar Implementations** | CircleCI Test Insights, GitHub Actions Test Analytics, Datadog CI Visibility |
| **Complexity** | 🟡 Medium |
| **Impact Score** | 8/10 |
| **Implementation Effort** | 1-2 weeks |

---

## FEATURE-010: Fault Injection During Replay
| Field | Value |
|-------|-------|
| **Category** | Product |
| **Problem Solved** | Can't test error handling paths. Replay only returns recorded responses — no way to simulate timeouts, 5xx errors, or connection resets. |
| **User Benefit** | `keploy test --inject-fault="mock-5:timeout=3s"` or `--inject-fault="mock-5:status=500"`. Test resilience without a real outage. |
| **Business Benefit** | Chaos engineering capability at the testing layer. Finds reliability issues early. Differentiator vs. all competitors. |
| **Similar Implementations** | WireMock fault injection, Toxiproxy, Chaos Monkey |
| **Complexity** | 🟡 Medium |
| **Impact Score** | 7/10 |
| **Implementation Effort** | 1 week |

---

## FEATURE-011: Watch Mode (`keploy test --watch`)
| Field | Value |
|-------|-------|
| **Category** | DX |
| **Problem Solved** | Developers must manually re-run `keploy test` after every code change. Breaks the inner development loop. |
| **User Benefit** | Automatic re-test on source file changes. Configurable file patterns. Only re-runs affected test sets. |
| **Business Benefit** | Tighter feedback loop → faster development → more tests written. Common in modern test runners. |
| **Similar Implementations** | Jest `--watch`, `go test -run`, pytest-watch, nodemon |
| **Complexity** | 🟢 Small |
| **Impact Score** | 7/10 |
| **Implementation Effort** | 2-3 days |

---

## FEATURE-012: HTML Test Report Generation
| Field | Value |
|-------|-------|
| **Category** | UX / Reporting |
| **Problem Solved** | Reports are text-only (CLI) or machine-readable (JSON/JUnit). No human-friendly shareable report. |
| **User Benefit** | `keploy report --format html` generates a standalone HTML file with interactive diffs, charts, and filtering. |
| **Business Benefit** | Shareable with non-technical stakeholders. Professional test reports for compliance. Competitive parity with Allure. |
| **Similar Implementations** | Allure Reports, Mochawesome, pytest-html, k6 HTML reports |
| **Complexity** | 🟡 Medium |
| **Impact Score** | 7/10 |
| **Implementation Effort** | 3-5 days |

---

## FEATURE-013: Load Testing from Recorded Traffic
| Field | Value |
|-------|-------|
| **Category** | Product / Performance |
| **Problem Solved** | Users must author load tests separately. Recorded traffic represents realistic usage patterns but can only be replayed 1:1. |
| **User Benefit** | `keploy load -c "<app>" --concurrency 50 --duration 60s`. Replay recorded traffic at configurable concurrency. |
| **Business Benefit** | Unique value prop: zero-effort load tests from real traffic. Addresses $2B+ performance testing market. |
| **Similar Implementations** | Speedscale load testing, k6 browser recordings, Gatling HAR import |
| **Complexity** | 🔴 Large |
| **Impact Score** | 8/10 |
| **Implementation Effort** | 2-3 weeks |

---

## FEATURE-014: Webhook/Notification System
| Field | Value |
|-------|-------|
| **Category** | Infrastructure / Enterprise |
| **Problem Solved** | No notification when test runs complete, fail, or when new test sets are recorded. Users must poll. |
| **User Benefit** | Configure Slack, email, or webhook notifications for test events. `keploy config` adds notification settings. |
| **Business Benefit** | Critical for team adoption. Enables async workflows. Table stakes for enterprise. |
| **Similar Implementations** | GitHub Actions notifications, CircleCI Slack Orb, Jenkins Slack Plugin |
| **Complexity** | 🟡 Medium |
| **Impact Score** | 7/10 |
| **Implementation Effort** | 3-5 days |

---

## FEATURE-015: Test Data Masking & Synthetic Data
| Field | Value |
|-------|-------|
| **Category** | Security / Enterprise |
| **Problem Solved** | Recorded test data contains PII (names, emails, SSNs). `sanitize` only removes secrets (tokens, keys), not PII. |
| **User Benefit** | Auto-detect and mask PII fields with synthetic data. GDPR/CCPA compliant test data. |
| **Business Benefit** | Enables enterprise adoption in regulated industries. Competitive parity with data masking tools. |
| **Similar Implementations** | Tonic.ai, Delphix, Faker.js, mimesis |
| **Complexity** | 🟡 Medium |
| **Impact Score** | 8/10 |
| **Implementation Effort** | 1-2 weeks |

---

## FEATURE-016: `keploy validate` — Test/Mock Schema Validation
| Field | Value |
|-------|-------|
| **Category** | DX |
| **Problem Solved** | Corrupted or hand-edited YAML files cause cryptic runtime errors. No pre-flight validation. |
| **User Benefit** | `keploy validate` checks all test/mock YAML files for schema correctness, timestamp consistency, and orphaned mocks. |
| **Business Benefit** | Reduces support tickets. Prevents CI failures from bad test data. Good first contributor task. |
| **Similar Implementations** | `kubectl validate`, `terraform validate`, `docker-compose config` |
| **Complexity** | 🟢 Small |
| **Impact Score** | 6/10 |
| **Implementation Effort** | 2-3 days |

---

## FEATURE-017: Test Retention Policies & Cleanup
| Field | Value |
|-------|-------|
| **Category** | Infrastructure |
| **Problem Solved** | `./keploy/reports/test-run-*` directories accumulate forever. No automatic cleanup. Large repos become unwieldy. |
| **User Benefit** | `keploy config` adds `retention.maxTestRuns: 10` and `retention.maxAge: 30d`. `keploy clean` removes old data. |
| **Business Benefit** | Prevents disk space issues in CI. Enterprise compliance requirement. |
| **Similar Implementations** | Docker `docker system prune`, npm cache clean, gradle cache cleanup |
| **Complexity** | 🟢 Small |
| **Impact Score** | 6/10 |
| **Implementation Effort** | 1-2 days |

---

## FEATURE-018: OpenTelemetry Integration
| Field | Value |
|-------|-------|
| **Category** | Infrastructure / Observability |
| **Problem Solved** | Test failures can't be correlated with distributed traces. No way to see which spans were active during a failing test. |
| **User Benefit** | Keploy emits OTel traces for record/replay operations. Trace IDs in test reports link to Jaeger/Tempo. |
| **Business Benefit** | Positions Keploy in the observability ecosystem. Enables "test-to-trace" debugging workflow. |
| **Similar Implementations** | Cypress OTel plugin, k6 Tracing, pytest-opentelemetry |
| **Complexity** | 🟡 Medium |
| **Impact Score** | 7/10 |
| **Implementation Effort** | 1 week |

---

## FEATURE-019: Selective Re-recording of Individual Tests
| Field | Value |
|-------|-------|
| **Category** | Product |
| **Problem Solved** | Re-recording requires re-recording the entire test set. Can't update just one failing test. |
| **User Benefit** | `keploy rerecord --test-set test-set-0 --test test-1`. Re-record only the specified test case. |
| **Business Benefit** | Saves time for large test sets. Reduces blast radius of re-recording. |
| **Similar Implementations** | Jest `--updateSnapshot` per file, VCR per-cassette re-record |
| **Complexity** | 🟡 Medium |
| **Impact Score** | 7/10 |
| **Implementation Effort** | 3-5 days |

---

## FEATURE-020: Markdown Report for PR Comments
| Field | Value |
|-------|-------|
| **Category** | DX / Reporting |
| **Problem Solved** | CI must parse JSON/text output to create PR comments. No Markdown-formatted report output. |
| **User Benefit** | `keploy report --format markdown` outputs a GitHub-flavored Markdown report ready for PR comments. |
| **Business Benefit** | Enables simple GitHub Actions integration without a full GitHub App. Quick win for CI adoption. |
| **Similar Implementations** | pytest-md, go test2json, coverage-badge-creator |
| **Complexity** | 🟢 Small |
| **Impact Score** | 7/10 |
| **Implementation Effort** | 1-2 days |

---

## FEATURE-021: Spy Mode (Selective Mocking)
| Field | Value |
|-------|-------|
| **Category** | Product |
| **Problem Solved** | During replay, ALL dependency calls are mocked. Can't selectively let some calls through to real services. |
| **User Benefit** | `keploy test --spy-ports 5432` or `--spy-hosts "auth-service"`. Only mock selected dependencies; others hit real services. |
| **Business Benefit** | Enables partial integration testing. Critical for gradual migration from integration to mock-based testing. |
| **Similar Implementations** | Hoverfly Spy Mode, WireMock partial mocking |
| **Complexity** | 🟡 Medium |
| **Impact Score** | 7/10 |
| **Implementation Effort** | 3-5 days |

---

## FEATURE-022: Flaky Test Detection & Quarantine
| Field | Value |
|-------|-------|
| **Category** | Product / Analytics |
| **Problem Solved** | Flaky tests erode trust in the test suite. No automatic detection or isolation of flaky tests. |
| **User Benefit** | Auto-detect tests that flip between pass/fail across runs. Quarantine them from blocking CI. `keploy test --skip-flaky`. |
| **Business Benefit** | Maintains CI reliability. Prevents developer frustration. Identifies tests needing noise rules. |
| **Similar Implementations** | CircleCI Test Insights, GitHub Actions Flaky Test Detection, pytest-flaky |
| **Complexity** | 🟡 Medium |
| **Impact Score** | 8/10 |
| **Implementation Effort** | 3-5 days |

---

## FEATURE-023: `keploy compare` — Live Traffic vs Recorded Comparison
| Field | Value |
|-------|-------|
| **Category** | Product |
| **Problem Solved** | No way to compare live production responses with recorded test expectations without running full replay. |
| **User Benefit** | `keploy compare --live-url https://prod.api.com --test-set test-set-0`. Sends recorded requests to live API and compares responses. |
| **Business Benefit** | Canary testing, migration validation, shadow traffic comparison. Used by Speedscale, Diffy. |
| **Similar Implementations** | Twitter Diffy, Speedscale Traffic Compare, AWS Application Signals |
| **Complexity** | 🟡 Medium |
| **Impact Score** | 7/10 |
| **Implementation Effort** | 1 week |

---

## FEATURE-024: SARIF Report Output for Code Scanning
| Field | Value |
|-------|-------|
| **Category** | Enterprise / Security |
| **Problem Solved** | GitHub Code Scanning and enterprise SAST/DAST pipelines expect SARIF format. Keploy can't integrate. |
| **User Benefit** | `keploy report --format sarif`. Upload test failures as code scanning alerts. |
| **Business Benefit** | Enterprise compliance integration. GitHub Advanced Security compatibility. |
| **Similar Implementations** | CodeQL SARIF, ESLint SARIF formatter, Snyk SARIF |
| **Complexity** | 🟢 Small |
| **Impact Score** | 5/10 |
| **Implementation Effort** | 2-3 days |

---

## FEATURE-025: Replay.go Modular Decomposition
| Field | Value |
|-------|-------|
| **Category** | Infrastructure / DX |
| **Problem Solved** | `pkg/service/replay/replay.go` is 143K / 3500+ lines. Extremely difficult to understand, test, or modify. Same issue with `pkg/util.go` (132K). |
| **User Benefit** | Faster PR reviews, easier contribution, fewer merge conflicts, better testability. |
| **Business Benefit** | Reduces contributor onboarding time. Enables parallel development. Prevents architectural rot. |
| **Similar Implementations** | Standard Go project layout best practices |
| **Complexity** | 🔴 Large |
| **Impact Score** | 8/10 |
| **Implementation Effort** | 2-3 weeks |
