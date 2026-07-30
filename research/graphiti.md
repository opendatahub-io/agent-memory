# Graphiti

[GitHub](https://github.com/getzep/graphiti) | ~24K stars (combined with Zep) | Apache 2.0 | Zep Software, Inc.

## Overview

Graphiti is a temporal knowledge graph library that processes unstructured text into entities and typed edges with bi-temporal validity windows. It is the open-source core of Zep, a managed agent memory platform. Unlike Zep Cloud (which adds hosted infrastructure and user/session management), Graphiti's algorithmic capabilities are fully available in the OSS library — there is no feature gating between OSS and paid tiers.

## Architecture

Graphiti processes **episodes** (messages, JSON documents, text, or pre-formed fact triples) through an LLM-powered extraction pipeline:

1. **Entity extraction** — LLM identifies entities from episode content, guided by optional custom entity type definitions (Pydantic models)
2. **Node resolution** — extracted entities are deduplicated against the existing graph using MinHash/LSH fuzzy matching plus LLM-based disambiguation
3. **Edge extraction** — LLM identifies relationships (facts) between resolved entities
4. **Edge resolution** — new edges are compared against existing ones; duplicates merge, contradictions trigger temporal invalidation
5. **Community detection** — label propagation clusters related entities; communities get LLM-generated summaries

The core data model has three node types: `EntityNode` (knowledge entities with embeddings and summaries), `EpisodicNode` (raw input records with source metadata), and `CommunityNode` (auto-generated clusters). Edges are typed and carry bi-temporal fields.

Storage is via a pluggable graph driver abstraction (`GraphDriver`) supporting Neo4j, FalkorDB, Kuzu (embedded/in-memory), and Amazon Neptune.

## Assessment

### P1: License — ✅

Apache 2.0. However, a Zep CLA (`Zep-CLA.md`) is required for contributions, which grants Zep Software broad rights over contributed code. Red Hat legal should review the CLA terms before committing to upstream contributions.

### P2: Multi-user / Tenant Isolation — 🔧

Graphiti uses a `group_id` string on every node and edge, validated to ASCII-alphanumeric. All queries filter by `group_id`/`group_ids`. This provides application-level partitioning but not database-enforced isolation — there are no row-level security policies, no per-tenant encryption, and a misconfigured query could leak across groups. Adding tenant isolation is feasible (inject `WHERE n.tenant_id = $tenant_id` alongside existing `group_id` filters) but there is no RBAC or authentication layer.

### P3: Security / PII Scanning — ❌

No PII detection, redaction, or sensitive-data filtering exists anywhere in the codebase. All ingested data is stored verbatim in the graph. Adding a pre-ingestion scanning pipeline is architecturally straightforward (intercept episode content before it enters the extraction pipeline) but would need to be built from scratch.

### P4: Storage Backend Flexibility — ✅

Four graph database drivers: Neo4j (most mature), FalkorDB (Redis-based, low-latency), Kuzu (embedded, no server required), and Amazon Neptune. The `GraphDriver` abstract interface with `GraphOperationsInterface` enables inversion-of-control for custom backends. Kuzu is particularly useful for development and testing (runs in-process like SQLite).

### P5: Retrieval Accuracy — 94.8% DMR, 63.8% LongMemEval

The Zep paper ([arXiv:2501.13956](https://arxiv.org/abs/2501.13956)) reports 94.8% on the DMR benchmark (vs MemGPT 93.4%) and 63.8% on LongMemEval temporal retrieval (vs Mem0's 49.0%). P95 retrieval latency is 300ms with no LLM calls at query time. Results are self-reported with limited third-party replication. Hybrid retrieval combines vector similarity, BM25 keyword search, graph traversal (BFS), and optional cross-encoder reranking.

### P6: Token Overhead / LLM Cost — 5+ LLM calls per episode ingested

Ingestion is LLM-heavy. Entity extraction, edge extraction, deduplication, summarization, and timestamp inference all require LLM round-trips. A single `add_episode()` call can trigger 5+ LLM calls. An SQLite-backed `LLMCache` mitigates repeated calls for identical inputs. Retrieval is LLM-free. For a customer ingesting 1,000 memories/day at ~$0.01/LLM call, ingestion cost is ~$50/day.

### P7: Consolidation / Decay — ✅

This is Graphiti's core differentiator. Every edge carries three temporal fields:
- `valid_at` — when the fact became true in the real world
- `invalid_at` — when it stopped being true
- `expired_at` — when the graph learned this

When new information contradicts an existing edge, the system uses LLM-driven timestamp extraction to invalidate the old edge rather than deleting it. This preserves full history — you can reconstruct past graph state at any point in time. Community nodes are rebuilt via label propagation clustering. Entity deduplication uses a two-pass approach: fast MinHash/LSH narrows candidates, then LLM confirms merges.

There is no confidence decay (temporal invalidation is binary, not gradual) and no batch consolidation into higher-order abstractions (patterns, wisdom).

### P8: Framework Independence — ✅

Standalone Python package (`graphiti_core`) with no framework lock-in. Ships an MCP server with both stdio and HTTP/SSE transports, a FastAPI REST service, and supports OpenAI, Anthropic, Gemini, and Groq LLM clients through an abstract `LLMClient` interface.

### P9: Provenance / Auditability — 🔧

`EpisodicNode` records `source` (enum: message/json/text/fact_triple), `source_description`, and `created_at`. Entity edges link back to episodes via `MENTIONS` relationships, enabling "which episode produced this entity?" queries. OpenTelemetry tracing is integrated. However, there is no immutable audit log, no chain-of-custody tracking, and no built-in mechanism to answer "which LLM call produced this edge."

### P10: Active Acquisition — ✅

`EpisodeType` supports `message`, `json`, `text`, and `fact_triple` formats. The `add_episode()` API accepts structured JSON, plain documents, and pre-formed triples — not just conversation turns. Examples in the repo include product catalog ingestion and podcast transcript parsing. Content chunking handles large documents automatically. `add_episode_bulk()` supports batch imports.

### P11: Deployment Simplicity — 🔧

Two components: Graphiti service and a graph database. `docker-compose.yml` provides Neo4j and FalkorDB configurations with health checks. A combined Dockerfile exists. However, it also requires an external LLM API (OpenAI by default) and an embedding service, making the total dependency count 3-4 services. No Helm chart, Kubernetes manifests, or OpenShift deployment templates exist.

### P12: Reconciliation — ❌

No mechanism to validate graph contents against external ground-truth systems. The temporal invalidation model handles contradictions within the ingestion stream (if a new episode contradicts an existing edge, the old edge is invalidated), but it cannot proactively check whether stored facts have become stale in external systems like JIRA or GitLab.

## Integration Considerations

Graphiti is the most architecturally compatible external system for adding governance features. Of the seven governance capabilities evaluated (tenant isolation, scope model, RBAC/JWT, PII scanning, versioning, graduation/promotion, audit logging), five can be bolted on above Graphiti's existing `group_id` + `SearchFilters` infrastructure. The remaining two (versioning with provenance chains and graduation/promotion) require a PostgreSQL sidecar for transactional workflow state, as graph databases lack ACID transactions across multiple nodes.

The primary concern for RHOAI is LLM cost at scale — 5+ LLM calls per ingested episode makes Graphiti significantly more expensive to operate than systems that rely on heuristics and embeddings (e.g., memory-hub, which makes near-zero LLM calls on the write path).
