# Integrations

## Core Sections (Required)

### 1) LLM Providers

Agentgateway provides a unified OpenAI-compatible API that proxies to all of the following:

| Provider               | Auth method                            | Module             | Notes                            |
| ---------------------- | -------------------------------------- | ------------------ | -------------------------------- |
| OpenAI                 | Bearer token (API key)                 | `llm/openai.rs`    | Reference implementation         |
| Anthropic              | Bearer token (API key)                 | `llm/anthropic.rs` |                                  |
| Google Gemini          | GCP ADC / service account              | `llm/gemini.rs`    |                                  |
| Google Vertex AI       | GCP ADC via `google-cloud-auth`        | `llm/vertex.rs`    |                                  |
| AWS Bedrock            | AWS SigV4 (`aws-sigv4`) + `aws-config` | `llm/bedrock.rs`   | Supports event stream            |
| Azure OpenAI           | Azure identity (`azure_identity`)      | `llm/azure.rs`     |                                  |
| GitHub Copilot         | Bearer token                           | `llm/copilot.rs`   |                                  |
| Custom / InferencePool | Configurable (see EP-288)              | `llm/custom.rs`    | Targets Service or InferencePool |

Credentials are injected from the config file or environment variables; never committed to the repo.

---

### 2) MCP Upstream Transports

The MCP gateway can fan out to multiple upstream MCP servers over any of:

| Transport                | Module                           | Notes                              |
| ------------------------ | -------------------------------- | ---------------------------------- |
| HTTP (Streamable HTTP)   | `mcp/upstream/streamablehttp.rs` | Preferred MCP transport            |
| Server-Sent Events (SSE) | `mcp/upstream/sse.rs`            | Legacy MCP transport               |
| stdio                    | `mcp/upstream/stdio.rs`          | Spawns subprocess (local tools)    |
| OpenAPI                  | `mcp/upstream/openapi/`          | Converts OpenAPI spec to MCP tools |
| HTTP client (generic)    | `mcp/upstream/client.rs`         | Shared HTTP upstream logic         |

---

### 3) A2A Protocol

- Implements [Agent-to-Agent (A2A) protocol](https://developers.googleblog.com/en/a2a-a-new-era-of-agent-interoperability/)
- Supports `/.well-known/agent.json` and `/.well-known/agent-card.json` discovery
- Rewrites agent-card URLs to route through the gateway
- Uses `@a2a-js/sdk` in the UI and `a2a/mod.rs` in the data plane

---

### 4) Authentication & Authorization

| Mechanism                          | Module / crate                              | Notes                                          |
| ---------------------------------- | ------------------------------------------- | ---------------------------------------------- |
| JWT verification                   | `http/jwt.rs`, `jsonwebtoken` crate         | RSA, EC, HMAC; JWKS URL fetch                  |
| API key                            | `http/apikey.rs`                            | Header or query param                          |
| Basic auth / htpasswd              | `http/basicauth.rs`, `htpasswd-verify-fork` |                                                |
| OAuth 2.0 / OIDC                   | `http/oauth.rs`, `http/oidc/`               | Token exchange, PKCE                           |
| External authorization (ext_authz) | `http/ext_authz.rs`                         | Envoy-compatible gRPC ext_authz                |
| External processing (ext_proc)     | `http/ext_proc.rs`                          | Envoy-compatible gRPC ext_proc                 |
| mTLS / CA client                   | `control/caclient.rs`                       | Certificate provisioning for workload identity |
| CEL RBAC                           | `http/authorization.rs`, `mcp/rbac.rs`      | Policy evaluated per-request using CEL         |

---

### 5) Guardrails / Content Safety

| Provider               | Module                               | Trigger                      |
| ---------------------- | ------------------------------------ | ---------------------------- |
| OpenAI Moderation API  | `llm/policy/moderation.rs`           | LLM request/response         |
| AWS Bedrock Guardrails | `llm/policy/bedrock_guardrails.rs`   | LLM request/response         |
| Google Model Armor     | `llm/policy/google_model_armor.rs`   | LLM request/response         |
| Azure Content Safety   | `llm/policy/azure_content_safety.rs` | LLM request/response         |
| Custom webhook         | `llm/policy/webhook.rs`              | LLM request/response         |
| Regex / PII            | `llm/policy/pii/`                    | In-process, no external call |

---

### 6) Observability

| Tool                        | Integration                             | Module                                                     |
| --------------------------- | --------------------------------------- | ---------------------------------------------------------- |
| OpenTelemetry OTLP (traces) | `opentelemetry-otlp`, gRPC + HTTP proto | `telemetry/trc.rs`                                         |
| OpenTelemetry OTLP (logs)   | `opentelemetry-otlp`                    | `telemetry/log.rs`                                         |
| Prometheus metrics          | `prometheus-client`                     | `telemetry/metrics.rs`, `management/metrics_server.rs`     |
| Grafana dashboard           | Pre-built dashboard JSON                | `manifests/grafana.json`, `controller/install/dashboards/` |
| Tokio runtime metrics       | `agent_core::metrics::TokioCollector`   | `crates/core/src/metrics.rs`                               |

Telemetry endpoint and headers are configurable via env vars (`OTEL_EXPORTER_OTLP_ENDPOINT`, etc.) or the static config file.

---

### 7) Kubernetes / Control Plane

| Component                    | Technology                       | Notes                                                          |
| ---------------------------- | -------------------------------- | -------------------------------------------------------------- |
| Gateway API CRDs             | `sigs.k8s.io/gateway-api`        | Standard GatewayClass, Gateway, HTTPRoute, etc.                |
| Custom CRDs                  | `controller/api/v1alpha1/`       | `AgentgatewayPolicy`, `AgentgatewayBackend`, etc.              |
| Controller runtime           | `sigs.k8s.io/controller-runtime` | Reconcile loops                                                |
| Istio KRT                    | `istio.io/istio/pkg/kube/krt`    | Reactive dependency tracking                                   |
| Deployer                     | `controller/pkg/deployer/`       | Creates/manages agentgateway Pods                              |
| XDS transport                | Custom delta-xDS over gRPC       | `crates/xds/`, `controller/pkg/apiclient/`                     |
| Helm chart                   | `controller/install/helm/`       | Packages controller + CRDs                                     |
| Kubernetes Inference Gateway | EPP (Endpoint Picker Protocol)   | `http/ext_proc.rs` — selects endpoints based on GPU/KV metrics |

---

### 8) DNS

- DNS resolution: `hickory-resolver` crate (Rust async DNS)
- Configurable lookup family (`DNS_LOOKUP_FAMILY` env var)
- Used for backend hostname resolution and service discovery

---

### 9) Credential Storage

- **No secrets in the repo.** Credentials flow through:
  - Static config file (e.g., `apiKey`, `token` fields) — file permissions are the security boundary
  - Environment variables
  - Cloud identity (GCP ADC, AWS credential chain)
  - Kubernetes Secrets (in K8s mode, injected by the controller)
- No secret manager integration is implemented in the OSS data plane (as of scan date). [ASK USER: is there a planned vault/ASM integration?]

---

### 10) AWS AgentCore

- Example at `examples/aws-agentcore/config.yaml`
- Uses the `agentcore.rs` module in the data plane
- [ASK USER: what is the full scope of the AWS AgentCore integration beyond the example?]

---

### 11) Evidence

- `crates/agentgateway/src/llm/` (all provider modules)
- `crates/agentgateway/src/mcp/upstream/`
- `crates/agentgateway/src/http/` (auth, ext_authz, ext_proc, jwt, oauth, oidc)
- `crates/agentgateway/src/telemetry/`
- `Cargo.toml` (aws-config, azure_identity, google-cloud-auth, opentelemetry-otlp, etc.)
- `go.mod` (controller-runtime, gateway-api, istio krt)
- `controller/pkg/controller/start.go`
- `manifests/grafana.json`
