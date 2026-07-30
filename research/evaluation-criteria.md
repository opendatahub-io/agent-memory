# Agent Memory System Evaluation Criteria

This document defines a prioritized set of requirements for evaluating candidate agent memory systems for RHOAI. The criteria are organized by priority — higher-priority items are harder to retrofit and more likely to block enterprise adoption.

Scoring key:

- **✅** — Present and meets expectations
- **🔧** — Not present, but extensible without intrusive changes
- **❌** — Absent and not easily extensible (would require significant rework)

For quantitative criteria (P5, P6), use descriptive values instead (e.g., benchmark scores, LLM call counts).

## Evaluation Matrix

| Criterion | memory-hub | Mem0 | OpenViking | OGX | [Cognee](cognee.md) |
|---|---|---|---|---|---|
| P1: License | | | | | ✅ Apache 2.0 |
| P2: Multi-user / Tenant Isolation | | | | | 🔧 Dataset-level ACL; database-per-tenant isolation |
| P3: Security / PII Scanning | | | | | ❌ No scanning; content stored verbatim |
| P4: Storage Backend Flexibility | | | | | ✅ 6 graph + 5 vector + 2 relational backends |
| P5: Retrieval Accuracy | | | | | ~90% self-reported; mixed independent results |
| P6: Token Overhead / LLM Cost | | | | | 3-4 LLM calls per chunk; 0-1 at query time |
| P7: Consolidation / Decay | | | | | 🔧 Manual distillation; no automatic decay |
| P8: Framework Independence | | | | | ✅ MCP, REST, SDK, CLI; no lock-in |
| P9: Provenance / Auditability | | | | | 🔧 DataPoint versioning + source chains; no audit log |
| P10: Active Acquisition | | | | | ✅ Files, URLs, databases via DLT |
| P11: Deployment Simplicity | | | | | 🔧 Minimal: 1 container; full: 3-5 containers |
| P12: Reconciliation | | | | | 🔧 Internal truth scoring; no external validation |

## Criteria

### P1: License Compatibility

Can Red Hat package and distribute this system in RHOAI? Apache 2.0 and MIT are preferred. AGPL requires legal review due to network copyleft implications for downstream customers. Evaluate CLA requirements for upstream contribution.

### P2: Multi-User / Tenant Isolation

How does the system isolate data between tenants? Row-level filtering, namespace partitioning, and database-per-tenant are common approaches with different trade-offs. For RHOAI, the system must support multi-tenant deployments where customers share infrastructure without data leakage.

### P3: Security / PII Scanning

Can the system prevent storage of sensitive data (PII, PHI, credentials, API keys)? Enterprise customers in healthcare and finance require compliance with HIPAA, PCI-DSS, and SOX. Evaluate both pre-write scanning (blocking sensitive content before storage) and post-write detection (identifying sensitive content already stored).

### P4: Storage Backend Flexibility

Can customers choose their own storage backend? The working group decided on configurable backends to accommodate enterprise requirements. Evaluate which databases are supported and whether a storage abstraction layer exists.

### P5: Retrieval Accuracy

Does the system retrieve the right information? Evaluate published benchmarks (and whether they are independently replicated), retrieval strategies (vector similarity, BM25, graph traversal, hybrid), and reranking capabilities. Self-reported benchmarks should be noted as such.

### P6: Token Overhead / LLM Cost

How many LLM calls does the system make per memory ingested, per query, and per consolidation cycle? Token overhead directly impacts customer cost. Systems that rely heavily on LLM calls for extraction, deduplication, and summarization can be 10x more expensive than systems that use heuristics and embeddings.

### P7: Consolidation / Temporal Decay

Does the system synthesize accumulated memories into higher-order knowledge? Does it handle staleness — facts that were true but no longer are? Evaluate: batch consolidation pipelines, confidence decay mechanisms, deduplication, contradiction detection, and whether consolidation runs automatically or requires manual triggering.

### P8: Framework Independence

Is the system locked to a specific agent framework (LangChain, CrewAI, etc.), or can it work with any agent? MCP support is effectively table stakes. Evaluate SDK availability, REST API, and CLI access.

### P9: Provenance / Auditability

Can you trace a memory back to its source? Enterprise governance requires knowing who contributed a fact, when, and from what source. Evaluate: source attribution, version history, audit logging, and chain-of-custody tracking.

### P10: Active Acquisition

Can the system learn from non-conversational sources — documents, JIRA tickets, meeting transcripts, code repositories? Or does it only learn from agent conversations? Active acquisition enables the agent to build knowledge proactively rather than passively.

### P11: Deployment Simplicity

How many components does the system require? Is it container-ready? Does it have Helm charts, Kubernetes manifests, or an Operator? For RHOAI, OpenShift-native deployment with UBI base images is strongly preferred.

### P12: Reconciliation Against Ground Truth

Can the system validate its stored memories against authoritative external systems (JIRA, GitLab, HR systems) and self-correct? This is an unsolved problem across the field — no existing system implements it. Evaluate any partial mechanisms (document re-fetching, contradiction detection, truth scoring).

## References

- [memory-hub](https://github.com/redhat-ai-americas/memory-hub)
- [Mem0](https://github.com/mem0ai/mem0)
- [OpenViking](https://github.com/volcengine/OpenViking)
- [OGX](https://github.com/ogx-ai/ogx)
- [Cognee](https://github.com/topoteretes/cognee)
