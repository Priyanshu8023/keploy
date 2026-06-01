# FEATURE-007: REST API Server Mode

## Feature Overview
`keploy serve` starts an HTTP API server that exposes CRUD operations on test sets, test cases, mocks, and reports. Enables external tool integration (IDE plugins, dashboards, CI systems).

## Why This Feature Matters
- **User Impact**: Programmatic access to all test data. Build custom dashboards, IDE plugins, CI integrations.
- **Developer Impact**: Foundation for FEATURE-001 (Web Dashboard), FEATURE-008 (VS Code Extension), FEATURE-009 (Analytics).
- **Project Impact**: Unlocks entire ecosystem of third-party integrations. WireMock's success is largely due to its admin API.

## Functional Requirements

### Endpoints
```
GET    /api/v1/test-sets                    # List all test sets
GET    /api/v1/test-sets/:id/tests          # List tests in a set
GET    /api/v1/test-sets/:id/tests/:testId  # Get single test case
GET    /api/v1/test-sets/:id/mocks          # List mocks for a set
DELETE /api/v1/test-sets/:id                # Delete a test set

GET    /api/v1/test-runs                    # List all test runs
GET    /api/v1/test-runs/:id                # Get test run details
GET    /api/v1/test-runs/:id/reports        # Get reports for a run
GET    /api/v1/test-runs/:id/report/:setId  # Get report for specific set

POST   /api/v1/record/start                # Start recording
POST   /api/v1/record/stop                 # Stop recording
POST   /api/v1/test/start                  # Start test run
GET    /api/v1/test/status                  # Get running test status

GET    /api/v1/config                       # Get current config
PUT    /api/v1/config                       # Update config

WS     /api/v1/ws/test-progress            # WebSocket for live progress
```

## Technical Design

### New Files
```
cli/serve.go                         # Cobra command
pkg/service/api/server.go            # HTTP server setup
pkg/service/api/handlers.go          # Route handlers
pkg/service/api/middleware.go        # CORS, logging, auth middleware
pkg/service/api/routes.go            # Route registration
pkg/service/api/ws.go                # WebSocket handler for live progress
```

### Dependencies
- Use `net/http` standard library (consistent with existing pprof server in main.go)
- JSON responses via `encoding/json`
- WebSocket via `gorilla/websocket` or `nhooyr.io/websocket`

### Architecture
```
keploy serve --port 8787
       │
       ▼
┌──────────────────────┐
│  HTTP Server (:8787) │
│  ┌────────────────┐  │
│  │ /api/v1/...    │──┼──▶ TestDB, MockDB, ReportDB interfaces
│  │ /api/v1/ws/... │──┼──▶ WebSocket event stream
│  └────────────────┘  │
└──────────────────────┘
```

### Reuse Existing Interfaces
The server reuses the same `TestDB`, `MockDB`, `ReportDB` interfaces defined in `pkg/service/replay/service.go`. No new storage layer needed.

## Acceptance Criteria
- [ ] `keploy serve` starts HTTP server on configurable port
- [ ] All CRUD endpoints return correct JSON responses
- [ ] WebSocket endpoint streams test progress events
- [ ] CORS headers allow browser access
- [ ] Graceful shutdown on SIGINT
- [ ] OpenAPI spec generated for the API
- [ ] Tests for all endpoints
