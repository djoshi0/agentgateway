# Conventions

## Core Sections (Required)

### 1) File and Module Naming

#### Rust

| Convention                                           | Example                            | Evidence                                               |
| ---------------------------------------------------- | ---------------------------------- | ------------------------------------------------------ |
| Files: `snake_case.rs`                               | `state_manager.rs`, `httpproxy.rs` | All `src/` files                                       |
| Modules: `snake_case`                                | `pub mod state_manager;`           | `crates/agentgateway/src/lib.rs`                       |
| Test siblings: `{name}_test.rs` or `{name}_tests.rs` | `gateway_test.rs`, `jwt_tests.rs`  | `proxy/`, `http/`                                      |
| Snapshot test data: `{name}_tests/` subdirectory     | `local_tests/`                     | `src/types/local_tests/`                               |
| Benchmark files: in `benches/`                       | `benches/`                         | `crates/agentgateway/benches/`, `crates/celx/benches/` |
| Proto files: `snake_case.proto`                      | `resource.proto`                   | `crates/protos/proto/`                                 |

#### Go (controller)

| Convention                            | Example                                         | Evidence                                  |
| ------------------------------------- | ----------------------------------------------- | ----------------------------------------- |
| Files: `snake_case.go`                | `controller.go`, `gw_controller.go`, `start.go` | `controller/pkg/controller/`              |
| Package names: lowercase, single word | `package controller`                            | `controller/pkg/controller/controller.go` |
| Test files: `{name}_test.go`          | `start_test.go`, `start_internal_test.go`       | `controller/pkg/controller/`              |

#### TypeScript / UI

| Convention                         | Example                                | Evidence             |
| ---------------------------------- | -------------------------------------- | -------------------- |
| Component files: PascalCase `.tsx` | [ASK USER — not inspected in depth]    | `ui/src/components/` |
| Config files: kebab-case           | `next.config.ts`, `postcss.config.mjs` | `ui/` root           |

---

### 2) Naming Conventions (Rust)

- **Types (structs, enums, traits):** `PascalCase` — `Bind`, `ListenerSet`, `McpAuthorization`, `ContextBuilder`
- **Functions and methods:** `snake_case` — `parse_config()`, `apply_to_request()`, `with_watched_handler()`
- **Constants / statics:** `SCREAMING_SNAKE_CASE` — `ADDRESS_TYPE`, `DEFAULT_SESSION_IDLE_TTL`
- **Private fields:** No prefix; Rust visibility (`pub`, `pub(crate)`, no qualifier) handles encapsulation
- **Feature flags:** `snake_case` in `Cargo.toml` — `ui`, `schema`, `jemalloc`, `tls-aws-lc`
- **Serde rename conventions:** `#[serde(rename_all = "camelCase")]` used broadly for JSON/YAML public types (config, XDS API)

---

### 3) Error Handling

- **Data plane (Rust):**
  - `anyhow::Result<T>` throughout for propagated errors
  - `thiserror::Error` derive for domain-specific error enums (e.g., `mcp::Error`, `proxy::ProxyError`)
  - Errors are propagated with `.context("message")` to build informative chains
  - Panics are not used for control flow; `unwrap()` is avoided outside test code
- **Controller (Go):**
  - Standard Go error wrapping: `fmt.Errorf("message: %w", err)`
  - Controller-runtime reconcile errors are returned as `error`; retries are managed by the framework
- **User-facing error messages:**
  - MCP errors hide internal tool names for RBAC (e.g., `"Unknown {1}: {2}"` intentionally omits "not authorized") — `mcp/mod.rs`

---

### 4) Formatting

#### Rust

```bash
cargo fmt -- \
  --config imports_granularity=Module,group_imports=StdExternalCrate,normalize_comments=true
```

Import grouping order: `std` → external crates → local (`use crate::…`)

#### Go

Standard `gofmt` (enforced by `golangci-lint`); see `controller/.golangci.yaml`

#### UI

ESLint via `next lint`; `eslintrc` config at project root of `ui/`

---

### 5) Logging

- **Data plane (Rust):** `tracing` crate — structured key=value log events
  - Logger initialization: `pretty_env_logger` for human output, OTLP for export
  - Log levels: standard Rust tracing levels (`trace`, `debug`, `info`, `warn`, `error`)
  - CEL-configurable attributes: users can define additional log fields via CEL expressions (see `architecture/cel.md`)
- **Controller (Go):** `go.uber.org/zap` with `logr` adapter; `istio/pkg/log` wrapper

---

### 6) Import Organization (Rust)

Imports are grouped into three sections by `cargo fmt`:

1. `std::` items
2. External crates
3. `crate::` / `super::` local items

Workspace-internal crates are imported as `use agent_core::prelude::*` (re-exports common types) or individually.

---

### 7) Configuration Schema

- Config types are annotated with `#[cfg_attr(feature = "schema", derive(JsonSchema))]`
- Schema generation is a `cargo xtask schema` step that produces `schema/config.json` and `schema/cel.json`
- Markdown docs (`schema/config.md`, `schema/cel.md`) are auto-generated from the JSON schema via `tools/schema-to-md.sh`
- CEL function docs (`schema/cel-functions.md`) are hand-maintained

---

### 8) Test Conventions

- **Unit tests:** `#[cfg(test)] mod tests { ... }` inline, or `#[path = "foo_test.rs"] mod tests;` sibling file
- **Snapshot tests:** `insta` crate; snapshots stored in `local_tests/*.snap`; update with `cargo insta review`
- **Integration tests:** `crates/agentgateway/tests/` (separate binary); `crates/agentgateway/tests/tests/` for individual test modules
- **Example validation:** `crates/agentgateway/tests/validate_examples.rs` — loads all `examples/*/config.yaml` files
- **Windows exclusion:** `#[cfg(all(test, target_family = "unix"))]` on snapshot tests due to line-ending differences
- **Benchmarks:** `divan` crate; `crates/agentgateway/benches/` and `crates/celx/benches/`

---

### 9) Commit Messages

Follows [Conventional Commits](https://www.conventionalcommits.org/) (`feat`, `fix`, `docs`, `style`, `refactor`, `perf`, `test`, `chore`) — see `CONTRIBUTION.md`

---

### 10) Evidence

- `CONTRIBUTION.md`
- `Makefile` (lint targets)
- `crates/agentgateway/src/lib.rs` (module declarations)
- `crates/agentgateway/src/mcp/mod.rs` (error enum + `thiserror`)
- `crates/agentgateway/src/state_manager.rs` (`anyhow::Result`)
- `architecture/cel.md` (CEL logging fields)
- `controller/.golangci.yaml`
- `ui/package.json` (`"lint": "next lint"`)
