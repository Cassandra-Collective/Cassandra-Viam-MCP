# Requirements and build plan

**Status:** Proposed for implementation · **Date:** 2026-11-08

The first shared release should let a Collective engineer retrieve a versioned internal result, discover relevant new work, and compare that work with the current project evidence. A user must be able to trace each material assertion to a source revision and see whether the available coverage is current. This plan builds on the [architecture](../ARCHITECTURE.md) and the [reference investigation](research/ai-but-simple.md).

## What the service must support

| ID | Requirement | Acceptance evidence |
| --- | --- | --- |
| R1 | Find decisions, hypotheses, stage results, and project documentation. | An integration fixture links a Cassandra hypothesis to a stage and a result at the same Git snapshot. Tool citations resolve to a commit, path, and locator. A later fixture commit renames a file and retires its former path from current search. |
| R2 | Discover new and revised public work across the initial watch topics. | Scheduled arXiv and GitHub adapters publish deduplicated revisions. A replay, overlapping window, edited release, versioned preprint, and withdrawal exercise the same ingestion path. The relevance suite has queries for every watch topic below. |
| R3 | Make age and source gaps visible. | Every successful data response includes its snapshot and relevant source freshness. Tests cover failed pages, backoff, an expired lease, a valid `304`, a failed publish, a cold start with no snapshot, and a known partial scan. None advances the successful full-run watermark incorrectly. |
| R4 | Keep evidence, interpretation, and decisions distinct. | Every claim in a reviewed note or comparison has at least one accessible revision and locator. A test preserves contradictory evidence, separates author-reported results from reproduced results, and invalidates a reviewed claim after withdrawal or incompatible revision. |
| R5 | Explain tradeoffs in project context. | A comparison fixture separates zero-shot and trained results on the same benchmark. Different datasets, splits, budgets, harness versions, or absent baselines yield a qualified comparison or `insufficient_evidence`, with cited inputs. |
| R6 | Show what changed for an active project without rewriting its history. | A weekly digest lists eligible new or revised works, linked techniques, and project relevance. Each reviewed recommendation has evidence and a named applicability limit. No update automatically changes an ADR or experiment outcome. |
| R7 | Serve authenticated read-only MCP clients. | A client matrix exercises discovery, schema validation, pagination, protocol compatibility, and scoped reads over the selected transport. Unauthorized principals cannot enumerate private results, citations, counts, cache entries, or freshness for sources they cannot access. |
| R8 | Recover and audit. | From a backup, restore published snapshots, source versions, review history, and checkpoints. Rebuild FTS5 and verify reference results, source IDs, revision digests, and access boundaries match before the service is marked ready. |

### Initial watch catalog

Version the catalog in the repository. Each entry declares its topic ID, category and keyword selectors, approved sources, exclusions, polling budget, and fixture queries. Treat the following scopes as starting selectors and tune them against the first relevance run.

| Topic | Include examples | First source targets |
| --- | --- | --- |
| LLM architecture | Attention, state-space models, recurrence, sparse routing, context scaling, inference tradeoffs | arXiv `cs.CL`, `cs.LG`, `cs.AI` and reviewed links to model cards or technical reports |
| Training algorithms | Optimizers, schedules, gradients, stabilization, curriculum, compute efficiency | arXiv `cs.LG`, `stat.ML` and tagged implementations |
| Pretraining | Data mixtures, deduplication, scaling, distributed training, ablations and failure modes | arXiv `cs.CL`, `cs.LG` and [TorchTitan releases](https://github.com/pytorch/torchtitan/releases) |
| Post-training | SFT, preference optimization, RL, distillation, alignment and safety evaluations | arXiv `cs.CL`, `cs.LG`, `cs.AI` and [TRL releases](https://github.com/huggingface/trl/releases) |
| Evaluation and agent harnesses | Benchmarks, eval harnesses, context and tool orchestration, agents, reproducibility and leakage | arXiv `cs.CL`, `cs.AI`, `cs.SE` and [LM Evaluation Harness releases](https://github.com/EleutherAI/lm-evaluation-harness/releases) |
| AI, ML, and data science | Transferable ML methods, data quality, provenance, statistics, data pipelines and experimental methodology | arXiv `cs.AI`, `cs.LG`, `stat.ML`, `cs.DB`, and reviewed data or software releases |

### Source contracts

| Source | First implementation | Failure and rights rules |
| --- | --- | --- |
| Collective GitHub | Start with the approved Cassandra repository and path allowlist. Use its `AGENT.md`, `docs/hypotheses/`, and `experiments/tiny_language_lab/RESULTS.md` as initial mapping candidates. Inventory ADR and stage source paths at the pinned onboarding commit. Keep the configured ref, resolved commit, and parsed locator. | Fetch with read-only credentials where needed. Disable checkout hooks and submodule traversal. Reject out-of-scope paths, symlinks, and oversized files. Check commit reachability and authorization before serving citations. |
| arXiv | Use Atom search with narrow watch predicates and `lastUpdatedDate` sort. Cap pages and use an overlap on repeat discovery. Add OAI-PMH `arXivRaw` reconciliation for metadata changes and version history. Store OAI datestamps separately from publication dates. | Share the legacy API limit across workers and controlled hosts, with one connection and at least three seconds between requests. Retain item metadata according to policy. Review each item license before retaining or exposing full text. |
| GitHub Releases | Poll allowlisted repositories through REST releases with a pinned API revision, persisted conditional request validators, and `Link` pagination. Track release IDs, publish and update dates, and available target commit identity. Label prereleases. | Respect rate-limit and retry headers. A `304` counts as a successful check only when the associated prior response and complete window exist. Treat release notes as maintainer reports. Tags without releases require a separate adapter. |
| Optional enrichment | Add Crossref DOI and retraction relations, or allowlisted Hugging Face model and dataset cards at pinned revisions, once a coverage gap is documented. | Keep metadata rights separate from abstract, card, dataset and full-text rights. A metadata relationship can flag an item for review. It cannot certify a claim or reproduction. |

Use [arXiv API documentation](https://info.arxiv.org/help/api/user-manual.html), [arXiv API terms](https://info.arxiv.org/help/api/tou.html), [OAI-PMH documentation](https://info.arxiv.org/help/oa/index.html), and the [arXiv license guide](https://info.arxiv.org/help/license/index.html) for the first collector. For releases, use the [GitHub endpoint](https://docs.github.com/en/rest/releases/releases), [REST best practices](https://docs.github.com/en/rest/using-the-rest-api/best-practices-for-using-the-rest-api), and [API version policy](https://docs.github.com/en/rest/about-the-rest-api/api-versions). Enrichment contracts come from [Crossref REST guidance](https://www.crossref.org/documentation/retrieve-metadata/rest-api/tips-for-using-the-crossref-rest-api/), [Hugging Face model cards](https://huggingface.co/docs/hub/en/model-cards), and [Hugging Face dataset cards](https://huggingface.co/docs/hub/en/datasets-cards).

Each source config declares allowed domains, visibility, accepted fields, byte limits, retention, and citation rules. Follow redirects only to approved origins, deny private and local address ranges on network collectors, and bound page count, decompression, and parse time. Store raw payloads only where rights and policy permit. Keep a digest and an attributable locator for permitted derivatives. Source text is untrusted data, including embedded instructions. Collectors do not execute code, download model weights, or fetch arbitrary links returned by a paper.

### Freshness, review, and ranking

- Start with a 30-minute fallback poll for Collective Git refs, a four-hour poll for release feeds, and a 12-hour arXiv watch poll. Initial stale thresholds are 2, 12, and 36 hours respectively. Measure them in the pilot and adjust after checking rate limits and actual lag. Apply jitter and bounded backoff.
- A completed run advances the published watermark only after all pages in the configured scan succeed. A cap hit with additional pages sets `partial` and records the inspected range. A valid `304` can renew the health of that validator's scope. It cannot renew unrelated pages or sources.
- Separate successful check age from arrival lag and review delay. Display last attempt, last complete success, observed high-water mark, and coverage at query time. A valid empty result needs a completed in-scope check. Otherwise return the closest prior snapshot and a source status, or `unavailable` where no snapshot exists.
- Keep new works visible as `discovered`. A reviewer can mark evidence `reviewed` and attach source type, a precise locator, experimental context, limitations, and a project applicability note. Reproduction requires a recorded independent run with inputs, versions, and results. A withdrawal or conflicting revision sends dependent claims to `needs_review` and excludes them from current recommendations.
- Publish one immutable weekly digest per configured project and UTC interval. Include new, revised, and withdrawn works, along with a reason for each inclusion. Separate discovery from reviewed recommendations. Allow no-change issues and report stale sources. Store the ranking version and weights or ordering rules with the digest.

## MCP contract

All tools operate on the caller's allowed corpus. Filters use enumerated topics and types, UTC timestamps, bounded query lengths, and a maximum page size. An opaque cursor binds the caller scope, query and immutable publication. When a retained cursor or snapshot is no longer available, return an explicit error so the client can restart pagination. The adapter validates structured results against its advertised output schema and emits a serialized JSON text representation when compatibility requires it. [MCP tools](https://modelcontextprotocol.io/specification/2026-07-28/server/tools)

| Tool | Inputs | Returns |
| --- | --- | --- |
| `search_knowledge` | Query, optional topic/project/type/scope filters, `as_of`, `limit`, cursor | Ranked internal and public evidence candidates with stable item and revision IDs, short excerpts, citations, and review status. |
| `get_item` | Item ID and optional revision ID or `as_of` | Versioned source metadata, allowed locators, relationship links, and revision or withdrawal status. |
| `get_claim_status` | Reviewed claim ID or bounded claim text, optional project and `as_of` | Supporting, contradicting and qualifying references, applicability, and `insufficient_evidence` when no reviewed match is available. Free-text matches remain candidates until a curated match is resolved. |
| `compare_methods` | Two to five method IDs, optional eval/model/budget context and `as_of` | Context-aligned rows, evidence IDs and source locations, unresolved dimensions and comparability. |
| `get_digest` | Project or topic plus UTC period, optional cursor | Published change list, reviewed notes and rationale, with snapshot and stale-source details. |
| `get_source_status` | Optional source ID or project | Authorized source health, bounded coverage, and publication timestamps. |

Expose the original internal decision, hypothesis and stage views through `get_item` and typed search filters. Add specialized convenience tools only where a pilot workflow demonstrates a need. Use separate, versioned MCP resources for referenced model cards, results or agent guidance if clients support them. A result has `schema_version`, `snapshot_id`, `as_of`, `coverage`, `items`, and `next_cursor` where relevant. A citation has `source_id`, `revision_id`, `uri`, `locator`, and `evidence_kind`. Keep these keys machine-readable in both structured and fallback content. Use `isError` for correctable tool failures such as invalid filter combinations and protocol-level errors for malformed requests as specified by MCP.

### Concrete evidence fixtures

- **Internal.** Ask what evidence closed Cassandra's context-utilization hypothesis. Follow Hypothesis 027 to the Stage 62 results and ADR 0019 at the onboarding commit. The result should distinguish the experiment verdict from the follow-on decision. [Hypothesis 027](https://github.com/Cassandra-Collective/CASSANDRA/blob/main/docs/hypotheses/027-context-utilization-and-coherence.md) · [Experiment results](https://github.com/Cassandra-Collective/CASSANDRA/blob/main/experiments/tiny_language_lab/RESULTS.md#stage-62---h027-context-utilization-probe)
- **External.** Use two paper-reported observations from *Context Language Models*. The abstract discusses a zero-shot BrowseComp-Plus experiment and a separate RL-trained Qwen3.5-9B experiment. The ingester should create one versioned source and two evidence records with different settings. `compare_methods` must preserve the difference and cite the versioned abstract. A digest may surface both as author-reported evidence. A reviewer still needs project-specific data and a matched evaluation before recommending an architecture change. [arXiv:2609.37725v1](https://arxiv.org/abs/2609.37725v1)

## Implementation sequence

### Phase 0 · Compatibility and contracts

- Pin the Rust toolchain and compatible `rmcp` release. Record the protocol versions observed in the target clients. Test modern `server/discover` and tool calls plus the older initialization flow for any supported legacy client. Keep CI fixtures for each admitted revision.
- Inventory the first Cassandra repository at an onboarding commit. Approve its allowlist and citation mapping. Commit small synthetic fixtures for internal documents, source changes, versions, and permission boundaries.
- Freeze the v1 response envelope, ID rules, watch catalog, and license policy. Establish expected answers for at least two questions per watch topic and at least one internal-to-external evidence question.

**Exit gate.** One pinned Rust MCP process answers a fixture query with a schema-valid citation in a target client, and every v1 requirement has a named fixture or operational check.

### Phase 1 · Internal vertical slice

- Implement the workspace modules for migrations and immutable revisions, Git snapshots, normalization, evidence links, FTS5, and the read-only MCP adapter.
- Serve `search_knowledge`, `get_item`, and `get_source_status` for one Cassandra repo. Record branch, commit and locator. Implement scheduled sync and keep a separate success watermark and error state.
- Add integration tests for idempotent fetch and publication, replay, deletion, scope, as-of reads, and cursor behavior across refreshes.

**Exit gate.** R1 and R3 pass for Cassandra fixtures. A pilot engineer can retrieve a hypothesis and its result using citations from a pinned commit.

### Phase 2 · Research refresh and evidence workflows

- Add bounded arXiv watch and GitHub Releases adapters. Track duplicate IDs and revised versions, and add the OAI-PMH reconciliation pass for covered arXiv changes. Test provider responses using recorded, rights-permitted metadata fixtures. Run limited live smoke tests under provider limits.
- Add claim and evidence review inputs, re-review on revision, `get_claim_status`, `compare_methods`, and `get_digest`. Seed the public comparison fixture and an internal result with explicit applicability bounds.
- Measure retrieval over the Phase 0 cases. The initial gate is at least 80% expected-source recall in the top five, 100% valid citation locators for reviewed output, and zero unsupported winner claims on the comparison fixtures. Keep the measured corpus and evaluation version with the report.

**Exit gate.** R2–R6 pass on fixtures, a live refresh records one complete configured research window and one release window, and a pilot project can trace one weekly digest entry from discovery through source revision to a reviewed note.

### Phase 3 · Shared pilot

- Deploy on a verified single-host budget with TLS and scoped credentials. Test actual target clients, including denied access and invalid Origin. Verify a signed webhook can enqueue only an approved source. Simulate webhook loss and scheduled recovery.
- Monitor per-source check age, lag and failure count, pending reviews, query p95, and database growth. Seed a provisional p95 below 750 ms for cached MCP search on a named pilot corpus and host. Report the actual corpus and hardware alongside the result.
- Exercise rate limiting, partial refresh, replay, backup restoration, FTS rebuild, and rollback to a known-good service version. Verify snapshot identity and access boundaries following recovery.

**Exit gate.** R7 and R8 pass. Run a seven-day pilot with source health visible through MCP, at least one weekly digest, and a reproducible refresh failure and recovery drill. Collect one workflow review from a downstream project and record any freshness or relevance thresholds that need revision.

### After the pilot

Add further Collective repositories and external sources as their owners approve corpus scope. Prioritize expansion based on missed results in the relevance suite and real project requests. Evaluate a visual briefing interface, richer relationship search, and a Cassandra model controller against separate acceptance criteria. The knowledge service remains independently usable throughout.
