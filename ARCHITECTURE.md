# Mimir MCP Server — Architecture Document

Version: Draft 0.1 · Date: November 1, 2026 · Status: Decisions made; the SRS will be written from this document

## 1. Purpose and scope

This document describes the architecture of Mimir, a Rust-based MCP (Model Context Protocol) server for the Cassandra project.

**Purpose:** Let AI agents such as Claude, Codex and Gemini read Cassandra's decisions, hypotheses and experiment results with sources, instead of guessing. In a second phase, the Cassandra model itself should be able to use tools.

**Out of scope (for now):** Training the Cassandra model, extending its vocabulary, and indexing the Chorus and Ephesus repositories in the first release (the architecture is designed to accommodate them).

## 2. Decision summary

- **Language and SDK:** Rust with the official `rmcp` SDK (v3.x, supports the MCP 2026-07-28 specification).
- **Transport:** Streamable HTTP over HTTPS (stdio was considered at first, then replaced).
- **Session model:** Stateless. State lives in the index, not in the protocol.
- **Data source:** GitHub repositories. The server clones and updates them itself.
- **Hosting:** Cloud, Oracle Cloud Always Free (Arm VM). Not final; depends on capacity and quota.
- **Approach:** Read-only knowledge server; every answer cites its source (`repo@commit`).
- **Phases:** Phase 1 is a knowledge server for agents; Phase 2 lets Cassandra use tools.

## 3. System architecture

```
Claude / Codex / Gemini        Python controller (Phase 2)
          |                              |
          +---------- HTTPS -------------+
                        |
                 Caddy (TLS, Let's Encrypt)
                        |
        +---------------+----------------+
        |   mimir-mcp (Rust, rmcp)       |
        |                                |
        |   Tool handler layer           |
        |      |                 |       |
        |  Knowledge     Model runner    |
        |      |                 |       |
        |  Index (SQLite)  Python child  |
        +------|-----------------|-------+
               |                 |
      Repo clones (GitHub)   Model weights
```

Requests arrive from outside at Caddy, which terminates TLS and forwards them to the Rust server over `127.0.0.1`. The server handles the MCP request in the tool handler layer and delegates the work to the service below.

## 4. Layers

### 4.1 Tool handler layer

The only layer that knows about MCP. It defines tools, resources and prompts with `rmcp` and generates parameter schemas with `schemars`. Errors are returned as tool results carrying `isError`, not as protocol errors, so the agent can see the failure and correct itself. It does not depend directly on the service layers below; it talks to them through traits.

### 4.2 Knowledge service

Pure Rust, independent of MCP, testable on its own. It holds the search, claim-to-evidence matching and project comparison logic. Every result returns with the file and commit it came from.

### 4.3 Index

At startup the repository files are scanned, markdown and frontmatter are parsed and written to SQLite FTS5. When a repository updates, the index is rebuilt. The index can always be regenerated from the repositories, which remain the single source of truth.

### 4.4 Model runner

Runs Cassandra's Python inference code as a subprocess. The rest of the server is independent of this layer. If the model is missing or fails to start, only the `sample_text` tool is disabled.

### 4.5 Repository sync

Repositories are cloned shallowly at startup. A GitHub webhook signals each push; the server runs `git fetch` and refreshes the index. If the webhook fails, a periodic `git fetch` acts as a fallback.

## 5. MCP surface

### 5.1 Tools (Phase 1)

- `list_decisions`, `get_decision`: list and fetch ADRs.
- `list_hypotheses`, `get_hypothesis`: fetch hypotheses together with their verdicts.
- `list_stages`, `get_stage_result`: fetch experiment stages and their result summaries.
- `search_docs`: full-text search across all documents; every result carries `file@commit`.
- `get_claim_status(claim)`: find which ADR, hypothesis or stage evidence supports or refutes a claim.
- `compare_projects(a, b, axis)`: compare two projects along given axes (once Chorus and Ephesus are added).

### 5.2 Model tool (Phase 1.5)

- `sample_text(prompt, max_chars, temperature)`: generate sample text from Cassandra. The tool description states the model's short context window.

### 5.3 Resources

- Model card, results summary and agent guidelines (`MODEL_CARD.md`, `RESULTS.md`, `AGENT.md`).

### 5.4 Prompts

- Ready-made comparison templates (for example, compare two architectures along given axes).

## 6. Transport, security and operations

- **Authentication:** Every request is checked for an `Authorization: Bearer` header; a wrong token returns 401. The token is supplied through an environment variable and never kept in code.
- **Origin check:** The `Origin` header is validated to guard against browser-originated requests.
- **Rate limiting:** Requests are limited, especially for `sample_text`, because it consumes CPU.
- **Webhook security:** The GitHub webhook signature (HMAC secret) is verified.
- **Read-only:** The server writes nothing to the repositories; its only side effect is the model call.
- **Network:** The Rust server listens only on `127.0.0.1`; only Caddy is exposed to the outside.
- **Logging:** Structured logging with `tracing`.

## 7. Deployment (Oracle Cloud Always Free)

1. Create an Arm-based Ubuntu VM and add an SSH key.
2. Open ports 80 and 443 in the security list and in the machine firewall.
3. Point a free domain name at the VM's IP address.
4. Caddy provides automatic HTTPS and proxies to the Rust server.
5. Run the Rust server as a `systemd` service.
6. The architecture is aarch64, so the Rust binary is built on the machine or cross-compiled.

**Watch out for:** Free Arm capacity may be unavailable in some regions. The quota was halved in June 2026 (2 OCPU, 12 GB). Oracle's rule for reclaiming idle machines should be checked in its documentation. Card verification is required.

## 8. Phase 2: Cassandra using tools

The Cassandra model is a base (not instruction-tuned) language model. Its character vocabulary is very narrow, it cannot produce some symbols needed for digits and JSON, and its context window is short. It therefore cannot produce structured tool calls.

**Solution:** A controller written in Python acts as an MCP client. The controller decides on its own which tool a question needs, calls the Phase 1 server, and hands the result to the model as short context. The model only writes the answer.

**Further options (separate research stage):**

- Constrained decoding, where the model picks the tool name only from allowed options.
- Extending the vocabulary and training with the tool format, either from scratch or by continued training.

## 9. Technology stack

- Rust, `tokio`, `axum`, `rmcp`, `schemars`, `serde`, `tracing`.
- `rusqlite` (SQLite FTS5), `notify` or periodic fetch, a `git` client.
- Caddy, `systemd`.
- Python side: the official `mcp` Python SDK and the Cassandra inference code.

## 10. Risks and open questions

- Oracle's free Arm capacity and quota may change; a small VPS or Cloud Run should be kept as a fallback.
- Where the model weights live (repository, LFS or release) is not yet determined.
- Whether the repositories will be public or private must be settled; if private, a GitHub token is needed.
- The model runner's speed on CPU has not been measured.
- Compatibility of the `rmcp` and Python SDK versions with the new specification should be verified with a connection test after installation.
- A shared `Project` data model must be designed so that Chorus and Ephesus can be added.

## 11. Roadmap

1. **Phase 1:** Knowledge tools, index, GitHub sync, HTTPS and bearer token, Oracle deployment.
2. **Phase 1.5:** `sample_text` and the model runner.
3. **Phase 1.6:** Add Chorus and Ephesus, `compare_projects`.
4. **Phase 2:** Python controller (MCP client), Cassandra using tools.
5. **Phase 3 (research):** The model makes its own tool decisions.
