# Technology Stack

## Core Sections (Required)

### 1) Runtime Summary

| Area | Value | Evidence |
|------|-------|----------|
| Primary language (data plane) | Rust 1.96.0 | `rust-toolchain.toml`, `Cargo.toml` |
| Primary language (controller) | Go 1.26.0 | `go.mod`, `.github/workflows/pull_request.yml` |
| Primary language (UI) | TypeScript / React | `ui/package.json`, `ui/tsconfig.json` |
| Rust package manager | Cargo (workspace resolver v2) | `Cargo.toml` |
| Go module system | `go mod` | `go.mod` |
| Node package manager | npm 10+ | `ui/package.json`, `DEVELOPMENT.md` |
| Build system (Rust) | Cargo + Makefile | `Makefile` |
| Build system (Go) | `go build` / `make -C controller` | `controller/Makefile` |
| Build system (UI) | Next.js + npm | `ui/package.json` |
| Container base image (runtime) | `cgr.dev/chainguard/glibc-dynamic` | `Dockerfile` |
| Container base image (build) | `docker.io/library/rust:1.96.0-trixie` | `Dockerfile` |

### 2) Production Frameworks and Dependencies

#### Rust (data plane — `crates/agentgateway/`)

| Dependency | Version | Role in system | Evidence |
|------------|---------|----------------|----------|
| tokio | ~1.x | Async runtime | `Cargo.toml` workspace deps |
| axum / axum-core / axum-extra | 0.8 / 0.5 / 0.12 | HTTP server framework | `Cargo.toml` |
| hyper / hyper-util | 1.9 / 0.1 | Low-level HTTP/1.1 + HTTP/2 | `Cargo.toml` |
| rustls | latest | TLS implementation | `crates/agentgateway/Cargo.toml` |
| aws-lc-rs | 1.17 | Crypto provider for rustls | `Cargo.toml` |
| prost / prost-build | 0.14 | Protocol Buffers encoding/codegen | `Cargo.toml` |
| opentelemetry + opentelemetry-otlp | 0.32 | Traces + logs via OTLP | `Cargo.toml` |
| prometheus-client | 0.24 | Metrics exposition | `Cargo.toml` |
| async-openai | 0.40 | OpenAI-compatible API types | `Cargo.toml` |
| jsonwebtoken | 10.4 | JWT parsing/verification | `Cargo.toml` |
| serde / serde_json | ~1.x | Serialization | `Cargo.toml` |
| reqwest | 0.13 | Outbound HTTP client | `Cargo.toml` |
| clap | 4.6 | CLI argument parsing | `Cargo.toml` |
| anyhow / thiserror | 1.x | Error handling | `Cargo.toml` |
| tracing | latest | Structured logging | `Cargo.toml` |
| hickory-resolver | 0.26 | DNS resolution | `Cargo.toml` |
| rcgen | 0.14 | Certificate generation | `Cargo.toml` |
| cel (forked) | path | CEL expression engine | `crates/cel-fork/` |
| rmcp | latest | MCP protocol implementation | `crates/agentgateway/Cargo.toml` |
| agent-hbone | path | HBONE/HTTP2 tunnel | `crates/hbone/` |
| agent-xds | path | XDS client | `crates/xds/` |
| aws-config / aws-sigv4 | 1.8 / 1.4 | AWS credentials + Bedrock auth | `Cargo.toml` |
| azure_identity / azure_core | 1.0 | Azure AD auth | `Cargo.toml` |
| google-cloud-auth | git rev | GCP ADC auth | `Cargo.toml` |
| openapiv3 | 2.2 | OpenAPI schema parsing (MCP upstream) | `Cargo.toml` |
| tiktoken-rs | latest | LLM token counting | `crates/agentgateway/Cargo.toml` |
| insta | 1.47 | Snapshot testing | `Cargo.toml` |
| wiremock (fork) | git rev | HTTP mocking in tests | `Cargo.toml` |
| divan | 0.1 | Microbenchmarks | `Cargo.toml` |

#### Go (controller — `controller/` + root `go.mod`)

| Dependency | Version | Role in system | Evidence |
|------------|---------|----------------|----------|
| controller-runtime | latest | Kubernetes reconcile loop | `go.mod` |
| gateway-api (sigs.k8s.io) | latest | Gateway API CRD types | `go.mod`, `controller/pkg/controller/start.go` |
| istio/istio krt | latest | Kubernetes Runtime dependency tracking | `go.mod`, `controller/pkg/controller/start.go` |
| go-control-plane/envoy | 1.37.1 | XDS proto types | `go.mod` |
| cel-go | 0.28.1 | CEL expression evaluation | `go.mod` |
| cobra | 1.10.2 | CLI framework | `go.mod` |
| zap (go.uber.org/zap) | 1.28.0 | Structured logging | `go.mod` |
| modelcontextprotocol/go-sdk | 1.6.0 | MCP Go SDK | `go.mod` |
| prometheus/client_golang | 1.23.2 | Metrics | `go.mod` |
| golang-jwt/jwt | 5.3.1 | JWT handling | `go.mod` |
| grpc-ecosystem/go-grpc-middleware | 1.4.0 | gRPC middleware | `go.mod` |

#### TypeScript / UI (`ui/`)

| Dependency | Version | Role in system | Evidence |
|------------|---------|----------------|----------|
| next | ^15.5.18 | React framework (App Router) | `ui/package.json` |
| react / react-dom | ^19.2.4 | UI rendering | `ui/package.json` |
| @modelcontextprotocol/sdk | ^1.25.3 | MCP client (browser) | `ui/package.json` |
| @a2a-js/sdk | ^0.3.10 | A2A client (browser) | `ui/package.json` |
| @radix-ui/* | various | Headless UI primitives | `ui/package.json` |
| tailwindcss | via postcss | CSS utility framework | `ui/postcss.config.mjs` |
| @monaco-editor/react | ^4.7.0 | In-browser code editor | `ui/package.json` |
| framer-motion | ^12.24.12 | Animations | `ui/package.json` |
| js-yaml | ^4.1.1 | YAML config editing | `ui/package.json` |

### 3) Development Toolchain

| Tool | Purpose | Evidence |
|------|---------|----------|
| `cargo fmt` | Rust formatting | `Makefile`, `CONTRIBUTION.md` |
| `cargo clippy` | Rust linting | `Makefile`, `CONTRIBUTION.md` |
| `cargo test` | Rust unit + integration tests | `Makefile` |
| `divan` | Rust microbenchmarks | `crates/agentgateway/benches/` |
| `insta` | Rust snapshot tests | `crates/agentgateway/src/types/local_tests/` |
| `golangci-lint` | Go linting | `controller/.golangci.yaml`, `controller/.custom-gcl.yml` |
| `go test -race` | Go unit tests with race detector | `.github/workflows/pull_request.yml` |
| `eslint` / `next lint` | UI linting | `ui/package.json` |
| `buf` | Protobuf code generation | `buf.gen.yaml`, `buf.yaml` |
| `cargo xtask schema` | JSON/Markdown schema generation | `crates/xtask/`, `Makefile` |
| `kind` + `tilt` | Local Kubernetes dev cluster | `DEVELOPMENT.md`, `Tiltfile` |
| `cross` | Cross-compilation (Linux ARM/AMD64) | `DEVELOPMENT.md` |
| `ctlptl` | Kind cluster management | `DEVELOPMENT.md` |
| `wiremock` fork | HTTP mocking | `Cargo.toml` |

### 4) Key Commands

```bash
# Build UI
cd ui && npm install && npm run build

# Build Rust binary (release, with embedded UI)
make build          # cargo build --release --features ui

# Run tests (Rust)
make test           # cargo test --all-targets

# Lint (Rust)
make lint           # cargo fmt --check && cargo clippy -D warnings

# Auto-fix lint (Rust)
make fix-lint

# Generate code (protos + schema)
make gen            # buf generate + cargo xtask schema + fmt

# Controller tests (Go)
go test -race ./...

# Controller lint (Go)
make -C controller analyze

# Docker build
make docker

# Local Kubernetes dev (requires kind, tilt, ctlptl)
ctlptl create cluster kind --name kind-kind --registry=ctlptl-registry
tilt up

# Validate example configs
make validate
```

### 5) Environment and Config

- Config sources: static YAML/JSON file, environment variables, XDS remote control plane
- Hot-reload: local YAML/JSON config file (watched with `notify` crate)
- Key env vars (from `crates/agentgateway/src/config.rs`, `agent_core::env::ENV`):
  - `IPV6_ENABLED` — enable IPv6 (default: true)
  - `LOCAL_XDS_PATH` — path to local config file
  - `DNS_LOOKUP_FAMILY` — DNS IP family preference
  - `OTEL_EXPORTER_OTLP_ENDPOINT` — OTLP endpoint for telemetry
  - `OTEL_EXPORTER_OTLP_HEADERS` — OTLP export headers
  - Various `*_KEY`, `*_SECRET`, `*_TOKEN` vars for LLM provider credentials
- Deployment modes: standalone binary, Docker container, Kubernetes via controller
- No `.env.example` exists — env vars must be discovered from source code

### 6) Evidence

- `Cargo.toml` (workspace + all `[workspace.dependencies]`)
- `crates/agentgateway/Cargo.toml` (crate-level features)
- `go.mod` (Go controller + CLI)
- `ui/package.json`
- `rust-toolchain.toml`
- `Dockerfile`
- `Makefile`
- `.github/workflows/pull_request.yml`
- `DEVELOPMENT.md`
