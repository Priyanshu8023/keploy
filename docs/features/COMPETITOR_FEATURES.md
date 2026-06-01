# COMPETITOR_FEATURES.md — Competitive Analysis

## Competitors Overview

| Competitor | Category | License | Key Differentiator |
|-----------|---------|---------|-------------------|
| **Speedscale** | Traffic replay + load testing | Proprietary | K8s-native, dynamic mock generation, load testing |
| **WireMock** | API stubbing/simulation | Apache 2.0 | Programmable DSL, massive ecosystem, WireMock Cloud |
| **Hoverfly** | Lightweight proxy mocking | Apache 2.0 | Spy mode, middleware pipeline, Go native |
| **Pact** | Contract testing | MIT | Consumer-driven contracts, Pact Broker, multi-language |
| **Testcontainers** | Integration testing | Apache 2.0 | Real DB instances, module ecosystem, language SDKs |
| **VCR/Polly** | HTTP recording (Ruby/JS) | MIT | In-process recording, cassette format, language-native |
| **Signadot/Sandbox** | K8s-native test environments | Proprietary | Ephemeral sandboxes, traffic routing |
| **TrafficParrot** | Service virtualization | Proprietary | Enterprise SV, OpenAPI import, stateful virtualization |

---

## Feature-by-Feature Comparison

### 1. Speedscale — Dynamic Mock Intelligence

| Feature | Why It Matters | Difficulty for Keploy |
|---------|---------------|----------------------|
| **Smart Mocks with Transform Chains** | Automatically handles rotating tokens, timestamps, session IDs. Mocks stay valid as production data evolves. | 🔴 Large — Requires AI/ML transform detection |
| **Load Testing from Recorded Traffic** | Replay recorded traffic at 10x, 100x, 1000x volume for performance testing. No separate load test authoring. | 🟡 Medium — Replay infra exists, need concurrency |
| **K8s Traffic Capture (Sidecar)** | Captures traffic via Envoy sidecar without app changes. Native K8s operator. | 🟡 Medium — eBPF already captures; K8s operator needed |
| **Visual Traffic Map** | Interactive graph showing service dependencies discovered from traffic. | 🟡 Medium — Mock data has the info; need UI |
| **SLA Assertions** | Define performance budgets (p95 < 200ms) and assert during replay. | 🟢 Small — Extension to existing test comparison |
| **Cluster-Level Recording** | Record across an entire K8s namespace, not just one service. | 🔴 Large — Architecture change |

### 2. WireMock — Programmable Stubbing

| Feature | Why It Matters | Difficulty for Keploy |
|---------|---------------|----------------------|
| **Standalone Server Mode** | Run as a persistent HTTP server for team-shared mocks. IDE-independent. | 🟡 Medium — Keploy has `agent` cmd; needs HTTP API |
| **Response Templating (Handlebars)** | Mocks return dynamic responses using request data. E.g., `{{request.body.name}}`. | 🟡 Medium — Templatize exists; extend to response |
| **Stateful Behavior** | Mocks can transition through states (e.g., "new → processing → complete") and return different responses. | 🔴 Large — Needs mock state machine |
| **Record/Playback API** | REST API to start/stop recording, manage stubs, verify interactions programmatically. | 🟡 Medium — Critical for IDE/tool integration |
| **WireMock Cloud** | SaaS-hosted mock service. Share mocks across teams without infrastructure. | 🔴 Very Large — SaaS product |
| **Fault Injection** | Simulate timeouts, connection resets, malformed responses. | 🟡 Medium — Proxy has the interception point |
| **Admin UI** | Web UI for browsing/editing stubs, viewing matched/unmatched requests. | 🟡 Medium — Missing in Keploy entirely |
| **Webhook Callbacks** | Automatically fire callbacks when specific requests are matched. | 🟡 Medium |

### 3. Hoverfly — Proxy Intelligence

| Feature | Why It Matters | Difficulty for Keploy |
|---------|---------------|----------------------|
| **Spy Mode** | Selectively mock some endpoints while passing others to real service. | 🟢 Small — Bypass rules exist; need per-endpoint granularity |
| **Middleware Pipeline** | Chain Go/Python/JS middleware to transform requests/responses in flight. | 🟡 Medium — Extension point design needed |
| **Diff Mode** | Compare real service responses with mock responses side-by-side. | 🟢 Small — `keploy diff` exists; enhance for live |
| **Journal** | Log all intercepted requests/responses for debugging. | 🟢 Small — Pcap capture exists; need structured log |
| **Import from Swagger/OpenAPI** | Generate mocks automatically from API spec. | 🟡 Medium — Contract module has OpenAPI support |
| **Simulation Schema Versioning** | Version control mock schemas with semantic versioning. | 🟡 Medium |

### 4. Pact — Contract Testing

| Feature | Why It Matters | Difficulty for Keploy |
|---------|---------------|----------------------|
| **Pact Broker** | Centralized contract repository with version management and can-i-deploy checks. | 🔴 Large — Needs server-side infrastructure |
| **Consumer-Driven Contracts** | Consumers define expected interactions; providers verify against them. | 🟡 Medium — Contract module exists; needs maturity |
| **Can-I-Deploy** | Binary check: "Is it safe to deploy service A given the contracts with B, C, D?" | 🔴 Large — Requires dependency graph + broker |
| **Webhooks on Contract Changes** | Notify downstream services when contracts change. | 🟡 Medium |
| **Multi-Language Support** | Pact has client libraries in 12+ languages. | 🔴 Large — Currently Go CLI only |
| **Pending Pacts** | New contracts don't break provider builds until provider explicitly supports them. | 🟡 Medium |

### 5. Testcontainers — Real Environment Testing

| Feature | Why It Matters | Difficulty for Keploy |
|---------|---------------|----------------------|
| **Module Ecosystem** | Pre-built modules for 50+ databases, message queues, and services. | N/A — Different paradigm |
| **Cloud Support** | Testcontainers Cloud runs containers in the cloud, not local Docker. | N/A |
| **Reusable Containers** | Containers persist across test runs for speed. | 🟢 Small — `--keep-app-alive` exists |
| **Wait Strategies** | Rich wait-for-ready strategies (log, HTTP, port, shell). | 🟢 Small — `--health-url` exists; enhance |

### 6. VCR/Polly — In-Process Recording

| Feature | Why It Matters | Difficulty for Keploy |
|---------|---------------|----------------------|
| **Cassette File Format** | Simple, human-readable recording format with request/response pairs. | N/A — Keploy's YAML format is comparable |
| **Request Matching Modes** | Strict, regex, partial, custom matchers. Configurable per cassette. | 🟢 Small — Noise rules + matcher exists |
| **In-Process Hooks** | No proxy needed; intercepts at HTTP client level in application code. | N/A — Different paradigm (Keploy is proxy-based) |
| **Ignore/Filter Headers** | Configurable header exclusion during matching. | ✅ Already exists via noise rules |

### 7. TrafficParrot — Enterprise Service Virtualization

| Feature | Why It Matters | Difficulty for Keploy |
|---------|---------------|----------------------|
| **Stateful Virtual Services** | Multi-step flows with data persistence (e.g., create → read → update). | 🔴 Large |
| **Data-Driven Responses** | CSV/DB-backed response data. Mock returns different data per request. | 🟡 Medium |
| **SOAP/XML Support** | Full SOAP protocol support for legacy enterprise integration. | 🟡 Medium |
| **Performance Testing Integration** | Direct integration with Gatling, JMeter. | 🟡 Medium |
| **Team Collaboration UI** | Shared mock management with RBAC and versioning. | 🔴 Large |

---

## Unique Features Keploy Already Has (Competitive Advantages)

| Feature | Competitors Lacking This |
|---------|-------------------------|
| **Zero-code eBPF traffic capture** | All competitors require app-level changes or proxy config |
| **Deterministic time freezing** | Most replay tools struggle with time-dependent tests |
| **Risk-based failure classification** | No competitor auto-classifies failures as HIGH/MEDIUM/LOW risk |
| **Automatic noise detection** | Manual noise config in all competitors |
| **Test-mock mapping** | Competitors don't trace which mocks a test consumed |
| **Multi-protocol in single tool** | WireMock=HTTP only, Pact=HTTP+message, Hoverfly=HTTP only |
| **Cross-version compatibility CI** | `record_latest_replay_build` matrix is unique to Keploy |
| **Secret sanitization** | Built-in gitleaks-powered secret scrubbing |
