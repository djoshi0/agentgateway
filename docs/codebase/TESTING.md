# Testing

## Core Sections (Required)

### 1) Test Runners and Commands

| Component               | Command                                       | Runner                                                                                 |
| ----------------------- | --------------------------------------------- | -------------------------------------------------------------------------------------- |
| Rust (all tests)        | `make test` → `cargo test --all-targets`      | Cargo test harness                                                                     |
| Rust (release profile)  | `make test-release`                           | Cargo, `quick-release` profile                                                         |
| Rust benchmarks         | `cargo bench` (divan)                         | `crates/agentgateway/benches/`, `crates/celx/benches/`                                 |
| Go controller           | `go test -race ./...`                         | `go test` with race detector                                                           |
| UI                      | `npm test` (from `ui/`)                       | [ASK USER: test framework not confirmed — no test script visible in `ui/package.json`] |
| Controller e2e          | `go test -tags=e2e -v ./controller/test/e2e`  | Kind cluster required                                                                  |
| Gateway API conformance | `make -C controller all-conformance`          | Kind cluster required                                                                  |
| Example validation      | `make validate` → `tools/validate-configs.sh` | Validates all `examples/*/config.yaml`                                                 |

---

### 2) Test File Organization

#### Rust

```
crates/agentgateway/
├── src/
│   ├── proxy/gateway_test.rs          # Sibling test file for gateway.rs
│   ├── proxy/locality_test.rs         # Sibling test file for locality logic
│   ├── http/*_tests.rs                # Sibling test files (jwt_tests, ext_authz_tests, etc.)
│   ├── types/local_tests.rs           # Config normalization tests
│   ├── types/local_tests/             # Snapshot files (*.snap) for local_tests
│   │   ├── basic_config.yaml          # Input config fixture
│   │   ├── basic_normalized.snap      # Expected IR snapshot
│   │   └── ...                        # (aws, llm, mcp, health variants)
│   ├── mcp/mcp_tests.rs               # MCP tests
│   ├── llm/tests.rs                   # LLM tests
│   ├── llm/anthropic_tests.rs         # Provider-specific tests
│   ├── http/route_test.rs             # Route matching tests
│   └── ...
├── tests/
│   ├── integration.rs                 # Integration test entry point
│   ├── tests/                         # Individual integration test modules
│   └── validate_examples.rs           # Validates all examples/ config files
└── benches/                           # divan benchmarks
```

All `_test.rs` / `_tests.rs` files are compiled only in test mode via:

```rust
#[cfg(test)]
#[path = "foo_test.rs"]
mod tests;
```

#### Go (controller)

```
controller/
├── cmd/
│   └── dependencies_test.go
├── pkg/controller/
│   ├── start_test.go
│   └── start_internal_test.go
├── test/
│   ├── e2e/                   # End-to-end (Kind cluster)
│   ├── conformance/           # Gateway API conformance
│   ├── deployer/              # Deployer integration tests
│   ├── helm/                  # Helm chart tests
│   ├── helpers/               # Shared test utilities
│   ├── gomega/                # Custom Gomega matchers
│   ├── setup/                 # Kind cluster setup scripts
│   └── testutils/             # Shared test utilities
```

---

### 3) Testing Approaches by Layer

| Layer                                  | Approach                                     | Tools                                      |
| -------------------------------------- | -------------------------------------------- | ------------------------------------------ |
| Config normalization                   | Snapshot tests                               | `insta` — `cargo insta review` to update   |
| HTTP middleware (jwt, ext_authz, etc.) | Unit tests with `wiremock` for external HTTP | `wiremock` fork, `insta`                   |
| MCP session/routing                    | Unit tests + integration tests               | inline `#[cfg(test)]`                      |
| LLM provider adapters                  | Unit tests with recorded HTTP fixtures       | `wiremock`, inline tests                   |
| Proxy (gateway, httpproxy)             | Sibling test files + integration tests       | `wiremock`, `tokio::test`                  |
| Load balancer                          | Unit tests in `loadbalancer.rs`              | Rust std test                              |
| CEL expressions                        | Unit tests in `celx` + data-driven           | `crates/celx/benches/`                     |
| Controller reconcile                   | Go unit + integration tests                  | `controller-runtime` fake client, `gomega` |
| E2E (K8s)                              | Real cluster tests (Kind)                    | `kind`, Rust binary, `go test -tags=e2e`   |
| Conformance                            | Gateway API conformance suite                | `make -C controller all-conformance`       |
| Auth (Keycloak/OIDC)                   | Only on Blacksmith CI runner                 | `tools/manage-validation-deps.sh start`    |

---

### 4) Key Test Infrastructure

- **`wiremock` (fork):** Mocks outbound HTTP calls (LLM providers, ext_authz servers, webhooks)
- **`insta`:** Snapshot testing with JSON, YAML, and filter/redaction support. Snapshots live in `src/types/local_tests/*.snap`
- **`tokio::test`:** All async Rust tests use `#[tokio::test]`
- **`divan`:** Criterion-style microbenchmarks for hot-path code (CEL evaluation, request parsing)
- **Keycloak:** Spun up by `tools/manage-validation-deps.sh` on Linux Blacksmith CI runners for OIDC testing; skipped on Windows/macOS
- **Kind cluster:** Required for E2E and conformance tests; set up by `controller/test/setup/setup-kind-ci.sh`

---

### 5) CI Test Matrix

| Job                      | Platform(s)                                      | Trigger                     |
| ------------------------ | ------------------------------------------------ | --------------------------- |
| `proxy-test`             | Linux (Blacksmith 4vCPU), Windows 2025, macOS 15 | PR + push to main           |
| `proxy-lint`             | Linux (Blacksmith)                               | PR + push to main           |
| `ui-lint`                | Ubuntu 24.04                                     | PR + push to main           |
| `controller-test`        | Linux (Blacksmith)                               | PR + push to main           |
| `controller-lint`        | Linux (Blacksmith)                               | PR + push to main           |
| `controller-e2e`         | Linux (Blacksmith)                               | PR + push to main           |
| `controller-conformance` | Linux (Blacksmith)                               | PR + push to main           |
| Nightly release          | Linux                                            | Scheduled (02:00 UTC daily) |

CI runs on [Blacksmith](https://blacksmith.sh/) runners for Linux (25 GB cache vs. GitHub's 10 GB).

---

### 6) Coverage

No coverage threshold is configured. `cargo test --all-targets` runs all tests; coverage is not enforced in CI. [ASK USER: is there a coverage target or tool (e.g., cargo-llvm-cov) planned?]

---

### 7) Evidence

- `Makefile` (`test`, `test-release` targets)
- `.github/workflows/pull_request.yml`
- `crates/agentgateway/tests/integration.rs`
- `crates/agentgateway/tests/validate_examples.rs`
- `crates/agentgateway/src/types/local_tests/` (snapshot files)
- `controller/test/` directory
- `Cargo.toml` (`insta`, `wiremock`, `divan` workspace dependencies)
