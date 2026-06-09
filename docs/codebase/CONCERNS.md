# Concerns

## Core Sections (Required)

### 1) Technical Debt — Production Code TODOs

**~55 TODOs / TODOMERGE items in production source** (from scan, test dirs excluded). High-density files:

| File                            | Count | Themes                                                                                                                                         |
| ------------------------------- | ----- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| `proxy/httpproxy.rs`            | 8     | Session persistence unimplemented, body mirroring missing, ImmediateResponse not supported in inference router, DNS resolution type assumption |
| `mcp/session.rs`                | 4     | Error handling too broad, ping forwarding unclear, notification fan-out incomplete, issue #404 linked                                          |
| `types/agent_xds.rs`            | 3     | HTTPS/TLS distinction, unmapped proto fields                                                                                                   |
| `types/local.rs`                | 2     | Vertex URL extraction, prefix rank auto-computation                                                                                            |
| `types/loadbalancer.rs`         | 3     | Multi-bucket selection unsupported, first-request overload, hash collision risk                                                                |
| `hbone/pool.rs`                 | 3     | TODOMERGE (in-progress merge), timeout/nodelay missing, stream tracking                                                                        |
| `hbone/lib.rs`                  | 1     | Hyper ping data-awareness                                                                                                                      |
| `llm/mod.rs`                    | 1     | Stream usage impact on users                                                                                                                   |
| `llm/types/bedrock.rs`          | 2     | Prompt variable values unimplemented, performance config unimplemented                                                                         |
| `llm/conversion/completions.rs` | 2     | Unmapped fields                                                                                                                                |
| `llm/policy/webhook.rs`         | 2     | Configurable timeout, missing action types                                                                                                     |
| `http/ext_authz.rs`             | 2     | Always-insert mode (should support append), response-side evaluation                                                                           |
| `http/localratelimit.rs`        | 1     | Missing success-path response headers                                                                                                          |
| `http/route.rs`                 | 2     | Query string re-parsed per call, port extraction                                                                                               |
| `http/mod.rs`                   | 2     | Absolute HTTP/1.1 form handling                                                                                                                |
| `http/backendtls.rs`            | 1     | Global TLS config de-duplication                                                                                                               |
| `http/jwt.rs`                   | 1     | Useful error content                                                                                                                           |

Selected high-impact TODOs:

```
httpproxy.rs:275  // TODO: implement session persistence
httpproxy.rs:853  // TODO: mirror the body. For now, we just ignore the body
httpproxy.rs:2300 // TODO: we currently do not support ImmediateResponse from inference router
mcp/session.rs:547 // TODO: the notification needs to be fanned out in some cases
hbone/pool.rs:76  // TODOMERGE: timeout, nodelay
types/loadbalancer.rs:867 // TODO: this will overload them on the first request
```

---

### 2) Large / Complex Files

Files over ~1 000 lines that mix multiple concerns:

| File                             | Lines     | Concern                                                                            |
| -------------------------------- | --------- | ---------------------------------------------------------------------------------- |
| `types/agent.rs`                 | ~3 327    | All canonical IR types in one file; adding a new resource requires editing this    |
| `types/local.rs`                 | ~3 315    | Entire local-config→IR translation; complex async `from()` chain                   |
| `proxy/httpproxy.rs`             | ~3 200+   | Routing, policy evaluation, backend dispatch, LB, protocol dispatch — all combined |
| `llm/mod.rs`                     | ~1 901    | Full LLM gateway: provider dispatch, response rewriting, token counting, streaming |
| `llm/policy/mod.rs`              | ~1 653    | All guardrail policy dispatch                                                      |
| `crates/agentgateway/Cargo.toml` | 231       | Dense dependency list                                                              |
| `schema/config.md`               | ~2 563 KB | Generated, but reflects the breadth of config surface area                         |

---

### 3) Security Risks

| Risk                                                    | Location                                    | Severity                                                                      |
| ------------------------------------------------------- | ------------------------------------------- | ----------------------------------------------------------------------------- |
| No `.env.example`                                       | repo root                                   | Low — new developers may not discover required env vars                       |
| Credentials in config file                              | `types/local.rs`, static config             | Medium — file-permission-based secret protection; no vault integration in OSS |
| `ext_authz` runs on response side (issue noted in TODO) | `http/ext_authz.rs:991`                     | Medium — ext_authz currently cannot block responses                           |
| HTTP/1.1 absolute-form requests not handled             | `http/mod.rs:752-759`                       | Low — edge case but bypass risk if a client sends absolute-form URIs          |
| `key_log` TLS debugging (warned if enabled)             | `transport/tls.rs` (referenced in `app.rs`) | Low — warn_if_key_log_enabled() guards this                                   |
| MCP error messages deliberately hide tool names         | `mcp/mod.rs`                                | ✅ Intentional security design — hides RBAC-filtered tool existence           |
| OpenVEX file present                                    | `.github/vex/openvex.json`                  | ✅ — indicates active CVE tracking/dismissal                                  |
| `deny.toml` (cargo deny)                                | repo root                                   | ✅ — license and advisory checks configured                                   |
| Dependabot                                              | `.github/dependabot.yml`                    | ✅ — automatic dependency security updates                                    |

---

### 4) Performance Concerns

| Concern                                | Location                                    | Notes                                                                                                                                   |
| -------------------------------------- | ------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| Query string re-parsed per request     | `http/route.rs:80`                          | TODO noted inline; perf risk on high-RPS paths                                                                                          |
| Body buffering for CEL                 | `architecture/cel.md`, `http/bufferbody.rs` | Buffering request/response bodies is expensive; `ContextBuilder` minimizes this but it still occurs when any expression references body |
| First-request endpoint overload        | `types/loadbalancer.rs:867`                 | On the first request to a new endpoint, all traffic may be sent there                                                                   |
| Connection pool hash collision         | `hbone/pool.rs:406`                         | Acknowledged as "probably resistant enough" but not collision-proof                                                                     |
| `httpproxy.rs` sequential policy chain | `proxy/httpproxy.rs`                        | Some middleware stages could be parallelized (e.g., independent auth + transform), but current implementation is sequential             |
| Large CRD YAML templates               | `controller/install/helm/.../templates/`    | CRD YAML files are ~900 KB and ~647 KB, which can slow Helm operations                                                                  |

---

### 5) In-Progress / Fragile Areas

| Area                               | Signal                                    | Notes                                                               |
| ---------------------------------- | ----------------------------------------- | ------------------------------------------------------------------- |
| HBONE pool merge                   | `TODOMERGE` comments in `hbone/pool.rs`   | Active merge in progress                                            |
| Inference router + LLM AI policies | `design/288-inferencepool-ai-policies.md` | Active design proposal (status: proposed, May 2026)                 |
| MCP notification fan-out           | `mcp/session.rs:547`                      | Incomplete — current fan-out strategy is unclear                    |
| XDS for `control_client`           | `app.rs:56` — `// TODO: use for XDS`      | XDS client created but not yet wired to the existing control client |
| `SelfWorkload` non-updateable      | `store/mod.rs`                            | Set-once; comment suggests ArcSwap upgrade needed for live updates  |
| `version = "0.0.0"`                | `Cargo.toml`                              | Intentional but can confuse tooling (cargo-release, etc.)           |

---

### 6) Test Gaps

| Gap                                                         | Location                                                       |
| ----------------------------------------------------------- | -------------------------------------------------------------- |
| No enforced coverage threshold                              | CI pipeline                                                    |
| UI testing framework not confirmed                          | `ui/package.json` lacks `test` script                          |
| Windows snapshot tests skipped                              | `types/local.rs` — `#[cfg(all(test, target_family = "unix"))]` |
| Auth validation (Keycloak) only runs on Blacksmith Linux CI | `.github/workflows/pull_request.yml`                           |

---

### 7) Dependency / Supply Chain

- **Forked crates:** `schemars`, `http-serde`, `wiremock` — patched via `[patch.crates-io]` in `Cargo.toml`. These diverge from upstream and require manual maintenance.
- **Git rev dependencies:** `google-cloud-auth` (googleapis/google-cloud-rust) is pinned to a git commit, not a released crate version — can break silently on upstream changes.
- **`deny.toml`:** cargo-deny configured for license and advisory checking. [TODO: verify which advisories are currently suppressed in `deny.toml`]

---

### 8) High-Churn Files

Git history not available in this workspace (no commits found by scan). [TODO: run `git log --follow --name-only` after cloning with full history to identify high-churn files]

---

### 9) Evidence

- `docs/codebase/.codebase-scan.txt` (TODO/FIXME list)
- `crates/agentgateway/src/proxy/httpproxy.rs`
- `crates/agentgateway/src/types/agent.rs`
- `crates/agentgateway/src/types/local.rs`
- `crates/agentgateway/src/hbone/pool.rs`
- `crates/agentgateway/src/mcp/session.rs`
- `Cargo.toml` (`[patch.crates-io]`)
- `.github/dependabot.yml`
- `.github/vex/openvex.json`
- `deny.toml`
- `design/288-inferencepool-ai-policies.md`
