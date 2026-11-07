# Cassandra-Viam-MCP

Cassandra-Viam-MCP is Cassandra Collective's planned Rust knowledge service for AI research and engineering decisions. It will connect versioned public research to the Collective's decisions, hypotheses, and experiment results. Projects and their MCP clients will be able to see what changed, trace a result to its source, and assess whether it applies to their work.

The first release is designed around LLM architecture, training algorithms, pretraining, post-training, evaluation and agent harnesses, and relevant AI, ML, and data science. It starts with one internal Cassandra repository, arXiv watches, and selected software release feeds. Each response will identify the evidence and source freshness it used.

The repository currently contains design documentation. Mimir is the working name for the MCP server described in the original architecture draft.

## Build documents

| Document | What it covers |
| --- | --- |
| [Architecture](ARCHITECTURE.md) | Rust services, ingestion and publication flow, versioned evidence, retrieval, and deployment boundaries. |
| [Requirements and build plan](docs/specification.md) | Watch catalog, source contracts, MCP tool surface, and implementation phases with acceptance gates. |
| [AI But Simple investigation](docs/research/ai-but-simple.md) | Public-site observations, REA evidence, unresolved questions, and design implications for Viam. |

Phase 0 starts with a pinned Rust and MCP compatibility spike, the first Cassandra source inventory, and evidence fixtures for the initial watch topics.
