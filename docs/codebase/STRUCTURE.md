# Directory Structure

## Core Sections (Required)

### 1) Top-Level Layout

```
agentgateway/
├── crates/                  Rust workspace — the data-plane proxy
│   ├── agentgateway/        Core library: all protocols, routing, policies, config
│   ├── agentgateway-app/    Binary entry point + CLI (`main.rs` → `agentgateway_app::run()`)
│   ├── cel-fork/            Forked CEL (Common Expression Language) engine
│   ├── celx/                CEL extensions/custom functions for agentgateway
│   ├── core/                Shared utilities: drain, metrics, readiness, signals, strng
│   ├── hbone/               HBONE (HTTP/2 tunnel) implementation
│   ├── htpasswd-verify-fork/ Htpasswd credential verification (forked)
│   ├── pool/                Connection pooling
│   ├── protos/              Proto definitions + Rust generated code (resource.proto)
│   ├── xds/                 XDS delta-xDS client (talks to controller)
│   └── xtask/               Cargo xtask: schema generation tasks
│
├── controller/              Go Kubernetes operator (Gateway API controller)
│   ├── api/                 CRD type definitions (v1alpha1, settings, annotations)
│   ├── cmd/
│   │   ├── agentgateway/    Controller binary entry point (main.go)
│   │   └── agctl/           CLI tool for controller
│   ├── hack/                Code-gen, CI helpers, Helm chart tooling
│   ├── install/
│   │   └── helm/            Helm chart for Kubernetes deployment
│   ├── pkg/                 Controller business logic
│   │   ├── controller/      Reconcile loops (GatewayClass, Gateway, routes)
│   │   ├── deployer/        Deploys agentgateway Pods via Kubernetes
│   │   ├── syncer/          Translates K8s resources → XDS protos
│   │   ├── pluginsdk/       Plugin SDK (KRT-based collection utilities)
│   │   ├── agentgateway/    agentgateway-specific controller logic (JWKS, plugins)
│   │   ├── apiclient/       XDS API client (sends config to data plane)
│   │   ├── setup/           Controller startup/wiring
│   │   └── ...              (admin, cli, logging, metrics, reports, etc.)
│   └── test/
│       ├── e2e/             End-to-end tests (Kind cluster)
│       └── conformance/     Gateway API conformance tests
│
├── ui/                      Next.js 15 React web UI (served by proxy at /ui)
│   ├── src/                 Application source
│   │   ├── app/             Next.js App Router pages
│   │   └── components/      React components
│   └── public/              Static assets
│
├── api/                     Shared Go protobuf API (resource.pb.go + JSON helpers)
│                            (imported by both controller and xds)
│
├── architecture/            Developer-facing architecture docs
│   ├── configuration.md     Config model: Static / Local / XDS
│   └── cel.md               CEL usage and ContextBuilder design
│
├── design/                  Enhancement proposals (EP-NNN files)
│
├── examples/                Runnable config examples (per-feature)
│   ├── basic/               Simple MCP gateway example
│   ├── a2a/                 Agent-to-agent example
│   ├── ai-prompt-guard/     Prompt guardrail example
│   ├── authorization/       RBAC + CEL policy example
│   ├── mcp-authentication/  OAuth/OIDC MCP auth example
│   ├── ratelimiting/        Rate limiting example
│   ├── telemetry/           OpenTelemetry example
│   ├── tls/                 TLS configuration example
│   └── ...                  (delegation, http, multiplex, oidc, openapi, etc.)
│
├── schema/                  Generated config JSON schema + Markdown docs
│   ├── config.json          JSON Schema for local config file
│   ├── config.md            Human-readable config reference (generated, 2.5 MB)
│   ├── cel.json             JSON Schema for CEL context variables
│   └── cel.md               CEL variables reference (generated)
│
├── manifests/               Kubernetes/Grafana manifests for local dev
├── img/                     Architecture diagrams + UI screenshots
├── tools/                   Pinned local copies of build tools (buf, helm, kind, etc.)
├── common/scripts/          `get-agentgateway` install script
│
├── Cargo.toml               Rust workspace root
├── go.mod                   Go module root (controller + CLI)
├── Makefile                 Top-level build orchestration
├── Dockerfile               Multi-stage Linux container build
├── Dockerfile.windows       Windows container build
├── rust-toolchain.toml      Pinned Rust toolchain (1.96.0)
├── buf.gen.yaml / buf.yaml  Protobuf generation config
├── deny.toml                Cargo deny (license + advisory checks)
└── Tiltfile                 Tilt live-reload configuration
```

### 2) Entry Points

| Entry point | Language | Path | Description |
|-------------|----------|------|-------------|
| Proxy binary | Rust | `crates/agentgateway-app/src/main.rs` | Calls `agentgateway_app::run()` |
| App wiring | Rust | `crates/agentgateway-app/src/lib.rs` | CLI args, allocator selection, subcommands |
| Core run loop | Rust | `crates/agentgateway/src/app.rs` | Spawns proxy, XDS, metrics, readiness |
| Config parsing | Rust | `crates/agentgateway/src/config.rs` | Builds `Config` from env + YAML |
| State manager | Rust | `crates/agentgateway/src/state_manager.rs` | Watches config file or XDS, feeds `Stores` |
| Gateway proxy | Rust | `crates/agentgateway/src/proxy/gateway.rs` | Accepts TCP, dispatches HTTP |
| HTTP proxy | Rust | `crates/agentgateway/src/proxy/httpproxy.rs` | Per-request routing + policy pipeline |
| Controller binary | Go | `controller/cmd/agentgateway/main.go` | Starts Kubernetes controller |
| Controller setup | Go | `controller/pkg/setup/` | Wires manager, controllers, deployer |
| Controller reconcile | Go | `controller/pkg/controller/start.go` | Registers reconcilers |

### 3) Key Source Modules (`crates/agentgateway/src/`)

| Module | Purpose |
|--------|---------|
| `app.rs` | Top-level async startup, thread pool, drain, readiness |
| `config.rs` | Static config parsing (env vars + YAML) |
| `state_manager.rs` | Dynamic config: file-watch hot-reload + XDS subscription |
| `store/` | In-memory IR stores: `BindStore` (routes/policies) + `DiscoveryStore` (workloads) |
| `types/agent.rs` | Core IR types: `Bind`, `Listener`, `Route`, `Backend`, policies (3 327 lines) |
| `types/local.rs` | Local config → IR translation (3 315 lines) |
| `types/agent_xds.rs` | XDS proto → IR translation |
| `types/loadbalancer.rs` | Load balancer (P2C, round-robin, weighted, locality-aware) |
| `proxy/gateway.rs` | TCP listener, TLS termination, connection dispatch |
| `proxy/httpproxy.rs` | HTTP request routing, auth, policy, backend calls (~3 200 lines) |
| `proxy/tcpproxy.rs` | TCP passthrough proxy |
| `llm/mod.rs` | LLM gateway: provider dispatch, token counting, request/response rewrite |
| `llm/policy/` | Guardrail policies: prompt guard, PII, moderation, Bedrock/Azure/Google guardrails |
| `llm/{openai,anthropic,bedrock,gemini,azure,vertex,copilot}.rs` | Per-provider adapters |
| `mcp/mod.rs` | MCP protocol core: session lifecycle, error types, failure modes |
| `mcp/handler.rs` | MCP request handling + tool routing |
| `mcp/session.rs` | MCP session multiplexing (SSE / Streamable HTTP) |
| `mcp/upstream/` | Upstream transports: HTTP, SSE, stdio, Streamable HTTP, OpenAPI |
| `a2a/mod.rs` | A2A protocol: agent-card rewriting, RPC classification |
| `http/` | HTTP middleware: JWT, API key, OAuth, OIDC, CORS, CSRF, rate limiting, ext_authz, ext_proc, retries, timeouts, transformations |
| `cel/` | CEL execution engine integration + `ContextBuilder` |
| `telemetry/` | Metrics, distributed tracing (OTLP), structured logging |
| `transport/` | TLS, HBONE, stream utilities |
| `control/` | CA client for mTLS cert provisioning |
| `management/` | Admin API, metrics server, readiness endpoint |
| `client.rs` | Outbound HTTP client used for backend calls |
| `ui.rs` | Serve embedded Next.js build at `/ui` |

### 4) Non-Obvious Directories

| Directory | What it is |
|-----------|-----------|
| `tools/` | Pinned tool binaries (`buf`, `helm`, `kind`, `ginkgo`, etc.) — NOT source code |
| `crates/cel-fork/` | Forked fork of the cel-rust crate; patched via `[patch.crates-io]` |
| `crates/htpasswd-verify-fork/` | Forked htpasswd verifier |
| `common/scripts/` | Shell install script for the binary release |
| `controller/hack/` | Code generation, CI utility scripts — not production code |
| `controller/install/` | Helm chart templates (including large CRD YAML templates) |
| `api/` | Shared Go package for the protobuf-generated resource types |

### 5) Evidence

- `Cargo.toml` (workspace members)
- `go.mod` (module root)
- `crates/agentgateway/src/lib.rs` (all `pub mod` declarations)
- `crates/agentgateway/src/app.rs`
- `crates/agentgateway/src/state_manager.rs`
- `controller/cmd/agentgateway/main.go`
- `controller/pkg/controller/start.go`
