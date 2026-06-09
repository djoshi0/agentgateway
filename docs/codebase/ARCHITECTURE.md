# Architecture

## Core Sections (Required)

### 1) Architectural Style

- **Primary style:** Protocol-domain layered proxy, with an XDS-driven control plane / data plane split
- **Why this classification:**
  - The data plane (`crates/`) is organized by protocol domain (`llm/`, `mcp/`, `a2a/`, `http/`) rather than by classic layer (controller → service → repo). Each domain owns its full request pipeline.
  - The control plane (`controller/`) is a Kubernetes operator that translates Gateway API CRDs + custom CRDs into XDS protobuf resources and pushes them to the data plane via a custom delta-XDS protocol.
  - The design explicitly avoids Envoy's fan-out XDS model: resources reference their parent rather than containing children, keeping cardinality 1:1 with user APIs.
- **Primary constraints:**
  1. **Near 1:1 mapping** of user-facing API → XDS proto → IR — minimizes fan-out on changes and simplifies the control plane (see `architecture/configuration.md`).
  2. **Pay-only-for-what-you-use** — the `ContextBuilder` for CEL only retains expensive fields (e.g. request body) when an expression actually depends on them.
  3. **Cross-platform** — runs on Linux (x86-64 + ARM64, glibc + musl), Windows, and macOS; the CI matrix tests all three.

---

### 2) System Flow

#### Standalone / local-config mode

```text
CLI args / config file
        │
        ▼
agentgateway-app::run()                      crates/agentgateway-app/src/lib.rs
        │
        ▼
config::parse_config()                       crates/agentgateway/src/config.rs
        │  ← env vars + YAML/JSON
        ▼
app::run(config)                             crates/agentgateway/src/app.rs
  ├── state_manager::StateManager::new()     watches config file (notify) or XDS
  │     └── local config → NormalizedLocalConfig → IR  (types/local.rs)
  │         or XDS delta push → IR            (types/agent_xds.rs)
  │         feeds Stores { BindStore, DiscoveryStore }
  │
  ├── proxy::gateway::GatewayProxy           crates/agentgateway/src/proxy/gateway.rs
  │     TCP listen → TLS termination
  │           │
  │           ▼
  │     proxy::httpproxy (per-connection)    crates/agentgateway/src/proxy/httpproxy.rs
  │       1. Read Stores → match Bind + Listener + Route
  │       2. Evaluate frontend policies (JWT, API key, CORS, CSRF, ext_authz)
  │       3. CEL authorization                http/authorization.rs
  │       4. Rate limiting                   http/localratelimit.rs, http/remoteratelimit.rs
  │       5. Ext-proc (request mutation)     http/ext_proc.rs
  │       6. Header / body transformations   http/transformation_cel.rs
  │       7. Protocol dispatch ──────────────────────────────────────┐
  │                                                                  │
  │          LLM path:  llm/mod.rs                                   │
  │            → provider selection (openai/anthropic/bedrock/…)     │
  │            → request rewrite + token counting                    │
  │            → prompt policy (guard/PII/moderation/guardrails)     │
  │            → upstream call (reqwest / hyper)                     │
  │            → response rewrite + usage tracking                   │
  │                                                                  │
  │          MCP path:  mcp/handler.rs + mcp/session.rs             │
  │            → session management (SSE / Streamable HTTP)          │
  │            → tool RBAC (mcp/rbac.rs)                             │
  │            → fan-out to upstream MCP servers                     │
  │              (stdio / HTTP / SSE / Streamable / OpenAPI)         │
  │                                                                  │
  │          A2A path:  a2a/mod.rs                                   │
  │            → agent-card URL rewriting                            │
  │            → JSON-RPC proxying                                   │
  │                                                                  │
  │          HTTP path: direct upstream via load balancer ───────────┘
  │       8. Telemetry (metrics, traces, logs)  telemetry/
  │
  ├── management::admin_server               admin API at :15000
  ├── management::metrics_server             Prometheus at :15020
  └── management::readiness_server           readiness probe
```

#### Kubernetes mode (adds control plane)

```text
Kubernetes cluster
  └── Gateway API resources + Custom CRDs (AgentgatewayPolicy, AgentgatewayBackend…)
          │
          ▼
  controller/pkg/controller/   (Go, controller-runtime + Istio KRT)
    reconcile loops → syncer/ → translates to XDS protos
          │
          ▼
  controller/pkg/apiclient/    sends ADP (AgentgatewayDev Protocol) resources
          │  delta-XDS push over gRPC
          ▼
  state_manager::StateManager   crates/agentgateway/src/state_manager.rs
    with_watched_handler::<XdsAddress>  → DiscoveryStore
    with_watched_handler::<ADPResource> → BindStore (routes, policies, backends)
```

---

### 3) Layer / Module Responsibilities

| Layer or module | Owns | Must not own | Evidence |
|-----------------|------|--------------|----------|
| `crates/agentgateway-app` | CLI arg parsing, allocator selection, top-level command dispatch | Business logic, protocol handling | `crates/agentgateway-app/src/lib.rs` |
| `config.rs` | Parse static configuration (env + YAML) into `Config` struct | Dynamic state, routing decisions | `crates/agentgateway/src/config.rs` |
| `state_manager.rs` | Watch config file / XDS, drive `Stores` updates | Per-request decisions | `crates/agentgateway/src/state_manager.rs` |
| `store/` (`Stores`) | Live in-memory IR (Binds, Listeners, Routes, Policies, Workloads) | Parsing, protocol logic | `crates/agentgateway/src/store/` |
| `types/agent.rs` | Canonical IR type definitions | Config translation, protocol I/O | `crates/agentgateway/src/types/agent.rs` |
| `types/local.rs` | Local config → IR translation | XDS translation | `crates/agentgateway/src/types/local.rs` |
| `types/agent_xds.rs` | XDS proto → IR translation | Local config | `crates/agentgateway/src/types/agent_xds.rs` |
| `proxy/gateway.rs` | TCP listen, TLS termination, connection lifecycle, drain | Routing, policy evaluation | `crates/agentgateway/src/proxy/gateway.rs` |
| `proxy/httpproxy.rs` | Per-request routing, policy chain execution, backend dispatch | Protocol-specific logic | `crates/agentgateway/src/proxy/httpproxy.rs` |
| `llm/` | LLM provider protocol adaptation, token counting, guardrail policies | MCP/A2A handling | `crates/agentgateway/src/llm/` |
| `mcp/` | MCP session lifecycle, tool RBAC, upstream multiplexing | LLM/A2A logic | `crates/agentgateway/src/mcp/` |
| `a2a/` | A2A agent-card rewriting, JSON-RPC classification | MCP/LLM logic | `crates/agentgateway/src/a2a/` |
| `http/` | Reusable HTTP middleware (auth, rate-limit, transform, ext_authz) | Protocol-specific parsing | `crates/agentgateway/src/http/` |
| `cel/` | CEL execution context + `ContextBuilder` | Request routing decisions | `crates/agentgateway/src/cel/` |
| `telemetry/` | Metrics, OTLP traces, structured logs | Business decisions | `crates/agentgateway/src/telemetry/` |
| `transport/` | TLS, HBONE tunnel, stream buffering | Application-layer protocols | `crates/agentgateway/src/transport/` |
| `crates/xds/` | XDS client: subscribe to ADP type URLs, feed handlers | IR translation | `crates/xds/src/` |
| `controller/pkg/controller/` | K8s reconcile loops (GatewayClass, Gateway, routes) | XDS encoding | `controller/pkg/controller/` |
| `controller/pkg/syncer/` | Translate K8s Gateway API types → XDS proto resources | K8s watch, deployer | `controller/pkg/syncer/` |
| `controller/pkg/deployer/` | Manage agentgateway Pod lifecycle in Kubernetes | Config translation | `controller/pkg/deployer/` |

---

### 4) Reused Patterns

| Pattern | Where found | Why it exists |
|---------|-------------|---------------|
| **IR (Internal Representation)** | `types/agent.rs`, `store/` | Decouples config source (local file vs. XDS) from runtime decisions |
| **Drain / graceful shutdown** | `crates/core/src/drain.rs`, `app.rs` | Every long-lived component takes a `DrainWatcher`; shutdown is cooperative with a hard timeout |
| **CEL ContextBuilder** | `crates/agentgateway/src/cel/` | Lazily captures expensive request data only if a policy expression references it |
| **Forked crates with `[patch.crates-io]`** | `Cargo.toml` | Allows patching upstream (`schemars`, `http-serde`, `wiremock`) without waiting for upstreams |
| **`_test.rs` sibling files** | `src/proxy/gateway_test.rs`, `src/http/*_tests.rs` | Keeps tests near implementation; avoids polluting `lib.rs` with test-only imports |
| **Snapshot testing** | `types/local_tests/*.snap`, `insta` crate | Deterministic regression tests for config normalization; easy to update with `cargo insta review` |
| **Separate `StoreUpdater` / `Store`** | `store/binds.rs`, `store/discovery.rs` | Write access is only held by state_manager; read access is cheaply cloned by proxy workers |
| **`Strng` type** | `crates/core/src/strng.rs` | Immutable reference-counted string (cheap clone, pooled literals); hot path string overhead reduction |
| **XDS resource-per-user-object** | `crates/protos/proto/resource.proto`, `architecture/configuration.md` | Avoids Envoy-style fan-out: 1 K8s route = 1 XDS proto = 1 IR update |
| **Plugin SDK (KRT)** | `controller/pkg/pluginsdk/` | Istio KRT-based reactive collection wiring for controller reconcilers |

---

### 5) Known Architectural Risks

- **`httpproxy.rs` size and coupling (~3 200 lines):** This single file owns routing, policy evaluation, backend dispatch, load balancing, and partial protocol handling. Changes to any one concern risk regressions in the others.
- **XDS is custom, not Envoy-compatible:** The `ADP` (AgentgatewayDev Protocol) types are purpose-built. Any tooling, mesh integration, or debugging tool expecting standard Envoy xDS types will not work.
- **`types/agent.rs` and `types/local.rs` (~3 300 lines each):** The IR type system is centralized in very large files. Extending the type system requires navigating large files and risks merge conflicts.
- **No `.env.example`:** Required environment variables are only discoverable by reading `config.rs` and `agent_core::env::ENV`. Onboarding friction for new contributors.
- **Version is `0.0.0`:** The workspace version is intentionally unset (`version = "0.0.0"`). Actual versioning is driven by git tags. This can confuse dependency tooling.

---

### 6) Evidence

- `crates/agentgateway-app/src/main.rs`
- `crates/agentgateway/src/app.rs`
- `crates/agentgateway/src/config.rs`
- `crates/agentgateway/src/state_manager.rs`
- `crates/agentgateway/src/proxy/gateway.rs`
- `crates/agentgateway/src/proxy/httpproxy.rs`
- `crates/agentgateway/src/store/mod.rs`
- `crates/agentgateway/src/types/agent.rs`
- `architecture/configuration.md`
- `architecture/cel.md`
- `controller/pkg/controller/start.go`
- `controller/cmd/agentgateway/main.go`
