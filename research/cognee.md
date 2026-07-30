# Cognee

[GitHub](https://github.com/topoteretes/cognee) | ~17.6K stars | Apache 2.0 | Topoteretes UG

## Overview

Cognee is a knowledge graph + vector hybrid memory system with composable task-based pipelines. It processes documents through an ECL flow (Extract, Cognify, Load) to build a knowledge graph with entity/relationship extraction, then provides multiple retrieval strategies across the graph and vector stores. It has the most flexible storage backend support of any system evaluated, with adapters for 6 graph databases, 5 vector databases, and 2 relational backends.

Cognee reports 70+ enterprise customers (including Bayer) and 1M+ pipelines/month. It is fully self-hostable under Apache 2.0.

## Architecture

The core pipeline:

1. **`add()`** — ingest documents (text, files, URLs, database connections via DLT). Supports PDF, DOCX, CSV, images, audio, and code files.
2. **`cognify()`** — process ingested data through composable task pipelines: classify documents → chunk → LLM entity/relationship extraction → ontology validation → graph storage + vector indexing → summarization.
3. **`search()`** / **`memify()`** — retrieve via 14 `SearchType` variants including graph completion, RAG, temporal, lexical, Cypher, chain-of-thought, and natural language.

Storage uses adapter interfaces for all backends:
- **Graph:** Ladybug (default, embedded), Neo4j, Amazon Neptune, PostgreSQL, Kuzu
- **Vector:** LanceDB (default, embedded), PGVector, ChromaDB, Qdrant, Weaviate, Milvus
- **Relational:** SQLite (default), PostgreSQL
- **File:** Local filesystem, S3
- **Cache:** SQLite, PostgreSQL, Redis, filesystem

Minimal deployment requires zero external services (SQLite + LanceDB + Ladybug defaults). Full-featured deployment adds Neo4j, PostgreSQL, and Redis.

## Assessment

### P1: License — ✅

Apache 2.0. Copyright 2024 Topoteretes UG. No CLA requirement found. Fully compatible with Red Hat productization.

### P2: Multi-user / Tenant Isolation — 🔧

Cognee has a multi-tenant architecture enabled via `ENABLE_BACKEND_ACCESS_CONTROL=True`. When enabled, each user+dataset combination gets isolated graph and vector database instances (separate Ladybug/LanceDB files or separate Neo4j databases). ACL-based permissions provide read/write/delete/share per dataset. A `parent_user_id` mechanism supports hierarchical user relationships.

However, isolation is **dataset-scoped**, not **team-scoped** — a single user sees all their datasets. There is no concept of shared team memory where multiple engineers contribute to and query a common knowledge graph while tracking individual contributions. Adding team-level shared knowledge with per-user attribution would require work on top of the existing permission model. Open issue [#2228](https://github.com/topoteretes/cognee/issues/2228) discusses challenges with concurrent multi-tenant LLM configuration.

### P3: Security / PII Scanning — ❌

No PII detection, redaction, or scanning exists in the codebase. All ingested content is stored and graph-extracted as-is. Adding a scanning task is architecturally straightforward — Cognee's task-based pipeline supports custom tasks cleanly, so a pre-write scanning task could slot in before the graph extraction step without modifying existing code.

### P4: Storage Backend Flexibility — ✅

The most flexible of any system evaluated. All backends use clean adapter interfaces (`GraphDBInterface`, `VectorDBInterface`). The default stack (SQLite + LanceDB + Ladybug) requires zero external services. Customers can swap to enterprise backends (PostgreSQL + PGVector, Neo4j, Qdrant, etc.) via configuration.

### P5: Retrieval Accuracy — ~90% self-reported; mixed independent results

Cognee's own benchmarks claim ~90% contextual accuracy on HotPotQA/MuSiQue/2WikiMultihop, compared to ~60% for plain RAG. Their evals framework computes Exact Match, F1, and DeepEval correctness scores. However, an independent evaluation ([arXiv:2507.05257](https://arxiv.org/pdf/2507.05257)) found Cognee underperforms GPT-4o-mini on some long-range tasks. The discrepancy between self-reported and independent results warrants caution.

14 retrieval strategies are available, including graph completion, chain-of-thought, triplet, temporal, and hybrid approaches.

### P6: Token Overhead / LLM Cost — 3-4 LLM calls per chunk

The `cognify` pipeline calls the LLM at multiple stages: entity/relationship extraction (via Instructor), summarization, chunk association, entity completion. At minimum 3-4 LLM calls per document chunk during cognification. Some retrieval strategies (graph completion) also make LLM calls at query time. Rate limiting is supported (`LLM_RATE_LIMIT_ENABLED`) but there is no cost tracking or token budget mechanism.

### P7: Consolidation / Decay — 🔧

Several consolidation mechanisms exist but none run automatically:
- `memify()` enriches the graph with additional context and rules
- `improve()` endpoint enriches the knowledge graph
- `distill_session()` / `propose_lessons()` / `publish_distilled_lessons()` for session-level consolidation
- Global context index builds hierarchical summaries with bucket-based grouping
- `SummaryNode` aggregation validates and rebuilds summary hierarchies

Temporal awareness exists via optional Graphiti integration (`build_graph_with_temporal_awareness.py`), but this requires Neo4j and `graphiti-core` as a dependency.

There is no TTL, automatic confidence decay, or forgetting mechanism. Consolidation is triggered manually via `memify()`/`improve()` calls, not scheduled.

### P8: Framework Independence — ✅

Zero framework lock-in. LangChain and LlamaIndex are optional extras. Ships a first-party MCP server (`cognee-mcp/`) with FastMCP, a standalone Python SDK, REST API (FastAPI), and CLI.

### P9: Provenance / Auditability — 🔧

`DataPoint` base model carries `id`, `created_at`, `updated_at`, `version`, and `ontology_valid` fields. Graph edges link entities back to `DocumentChunk` nodes which reference source data locations (`raw_data_location`). The `truth_subspace` module tracks `truth_epoch` per node. Session QA entries track `used_graph_element_ids` and `used_session_context_ids`.

Missing: no immutable audit log, no change history beyond a version counter, no per-user attribution on individual facts.

### P10: Active Acquisition — ✅

`add()` accepts text, file paths, URLs (with BeautifulSoup/Tavily/Playwright crawlers), and database connections via DLT integration. Supports PDF, DOCX, CSV, images, audio, and code files. The DLT integration (`create_dlt_source_from_connection_string`) enables structured data ingestion from relational databases with automatic foreign-key edge extraction. The `agent_memory` decorator enables automatic memory persistence from agent traces.

### P11: Deployment Simplicity — 🔧

`docker-compose.yml` with optional profiles. Minimal deployment is a single container with SQLite/LanceDB/Ladybug defaults — zero external services required. Full-featured deployment (Neo4j + PostgreSQL + Redis) requires 3-5 containers. A frontend is included. No Helm charts, Kubernetes manifests, or OpenShift deployment templates exist.

### P12: Reconciliation — 🔧

The `truth_subspace` module computes cosine alignment of chunks against "session learnings" centroids, producing a `truth_factor` score per node. This provides internal consistency scoring — nodes that drift from learned patterns score lower. However, there is no mechanism for external ground-truth validation (querying JIRA, GitLab, or other authoritative systems to verify stored facts remain current).

## Integration Considerations

Adding memory-hub's governance features to Cognee would require significant rework. The core challenge is that Cognee's tenant isolation model (database-per-tenant) is architecturally opposite to memory-hub's (row-per-tenant with SQL predicates). Converting Cognee to row-level isolation would touch the `DataPoint` model, all graph DB adapters (4 implementations), all vector DB adapters (4+ implementations), all search retrievers, and the cognify pipeline — an estimated 30-40 files across the infrastructure layer.

PII scanning and audit logging are clean bolt-ons (Cognee's pipeline and middleware architecture accommodates them). JWT/RBAC middleware is a moderate lift. But tenant isolation, scope model, and versioning with provenance are invasive changes.

The most practical integration pattern would be using Cognee's ECL pipeline as the knowledge extraction engine feeding into a governed storage layer (e.g., memory-hub), rather than trying to make Cognee's storage layer enterprise-grade.
