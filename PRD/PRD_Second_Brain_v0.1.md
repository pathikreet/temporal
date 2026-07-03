# PRD: Second Brain Platform — Draft v0.1

> **Version:** v0.1 (Draft)  
> **Date:** 2026-07-03  
> **Status:** In Progress — Pending Refinement  
> **Owner:** User  

---

## Overview

A personal knowledge graph platform designed to eliminate cognitive overload for a Principal + Staff Engineer + EM operating across three concurrent roles. The system captures, stores, traverses, and surfaces institutional memory with token-optimized responses via a multi-surface harness.

---

## Problem Statement

The user's daily work spans architecture decisions, PR reviews, mentoring, 1:1s, and leadership reporting — most of which is "shadow work" never tracked in JIRA [cite:3]. Thoughts captured on mobile are lost before they are processed [cite:2]. The end goal is a system that recalls, connects, and surfaces this knowledge autonomously.

---

## Core Design Principles

- **BFS by default**: Traverse the knowledge graph broadly first — surface all related nodes before diving deep.
- **DFS on probe**: When a user drills into a specific node/concept, switch to depth-first traversal to exhaust the thread.
- **>90% match = auto-surface**: If retrieval confidence exceeds 90%, push the answer without waiting for the user to ask.
- **TOON responses**: All outputs are Token-Optimized Notation — compressed, structured, no filler prose.
- **Determinism in recording**: Users must explicitly invoke a "record" intent; nothing is stored implicitly without a clear signal.

---

## Must-Have Features

### 1. Graph RAG Engine

- Knowledge stored as a **property graph** (nodes = concepts/events/people/tasks; edges = relationships with metadata).
- Each node carries: `type`, `role_context` (PE/SE/EM), `timestamp`, `source`, `tags`, `raw_content`, `embedding`.
- RAG retrieval uses graph-aware traversal: BFS for discovery queries, DFS for deep-dive queries.
- Similarity threshold gate: auto-respond if cosine similarity > 0.90; else prompt user for clarification.
- Graph backend: **Neo4j** (self-hosted on VM/pod) or **Kuzu** (embedded, lower infra cost) [cite:1].

### 2. Capture Harness (Deterministic Input)

- Every input goes through a **classification harness** before being written to the brain:
  - Intent detection: `RECORD` | `QUERY` | `REFLECT` | `REPORT`
  - Only `RECORD` intent triggers a write operation.
  - User must confirm ambiguous intents before storage (no silent writes).
- Capture sources [cite:5]:
  - Telegram/WhatsApp bot listener (deployed on VM/pod)
  - Slack bot listener
  - Native UI form
  - API endpoint (for programmatic ingestion)
- Role-tagging on capture: every entry auto-tagged to `PE` / `SE` / `EM` context or `GENERAL` [cite:4].

### 3. Exposure Surfaces

| Surface | Mechanism | Use Case |
|---|---|---|
| **Native UI** | Browser-based SPA (React/Next.js) | Deep exploration, graph viz, report generation |
| **Skill/Harness** | REST + MCP-compatible skill endpoint | Embed in any AI harness or agent orchestrator |
| **Slack Listener** | Bot via Socket Mode on VM/pod | Quick capture & recall during standups/meetings |
| **WhatsApp/Telegram** | Webhook listener on VM/pod | Mobile capture on the go [cite:5] |

### 4. TOON Response Format

All API/skill/bot responses follow this schema:

```
[MATCH:92%] [NODE:PR-Review/Auth-Service/2026-06-21]
→ Reviewed JWT refresh logic. Flagged race condition.
→ Linked: JIRA-4421 | Author: @dev_name
→ DFS? Y/N
```

No full sentences unless explicitly requested by the user.

---

## Non-Functional Requirements

### Token Optimization

- Default response mode is TOON (compressed schema above).
- Verbose mode only activated by explicit `--verbose` flag or UI toggle.
- Graph traversal capped at **3 hops** for BFS unless probed (DFS unlocks deeper hops).

### Resilience & SPOF Mitigation

| Component | SPOF Risk | Fallback |
|---|---|---|
| Graph DB (Neo4j/Kuzu) | DB crash | In-memory cache (Redis) + flat markdown fallback vault [cite:1] |
| Embedding service | Model API downtime | Local fallback model (e.g., `nomic-embed-text` via Ollama) |
| Slack/WhatsApp listener | Pod crash | Dead letter queue (Redis Streams / SQS) re-delivers on restart |
| LLM for RAG synthesis | API rate-limit/outage | Cached top-5 nodes returned raw without synthesis |
| Native UI | Frontend unavailability | Skill endpoint always live independently |

### Determinism

- All writes are **idempotent** — same raw input produces the same node (deduplication via content hash).
- Every node write logs `who`, `when`, `surface`, `intent_confidence_score`.
- Audit trail is append-only and immutable.

---

## Data Architecture

```
[Capture Layer]
  └─ Telegram/WA/Slack/UI/API
        │
[Intent Harness]
  └─ Classify → RECORD | QUERY | REFLECT | REPORT
        │
[Embedding + Entity Extraction]
  └─ Chunk → Embed → Extract entities/roles/dates
        │
[Graph Store]        ←→   [Vector Index]
  └─ Neo4j/Kuzu              Qdrant / pgvector
        │
[Retrieval Engine]
  └─ BFS (default) / DFS (on probe)
  └─ Threshold gate (>90% → auto-surface)
        │
[Response Synthesis]
  └─ TOON formatter → Surface
```

---

## Role-Aware Buckets

Carrying forward from prior design [cite:4], all nodes are tagged to one of:

- `PE` — Architecture decisions, design docs, tech spikes
- `SE` — Deep code reviews, debugging trails, performance investigations
- `EM` — 1:1 notes, leadership reports, team health signals, mentoring sessions
- `GENERAL` — Fleeting ideas, cross-cutting captures

Reports (daily/bi-weekly) are generated by filtering + traversing the relevant role bucket [cite:3].

---

## Out of Scope (v1)

- Multi-user / team-shared brain (single-user only in v1)
- Voice-to-text capture (noted for v2)
- Calendar/JIRA auto-ingestion (noted for v2)

---

## Open Items for Refinement

1. Graph DB choice: Neo4j (full-featured, heavier) vs Kuzu (embedded, zero-ops)?
2. Vector store: Qdrant vs pgvector vs Weaviate?
3. LLM backbone: cloud API (GPT-4o / Claude) with local fallback, or local-only (Ollama)?
4. Auth model for the Native UI — SSO or simple API key?
5. Deployment target: single VM, k8s pod, or Fly.io/Railway for simplicity?

---

## Changelog

| Version | Date | Changes |
|---|---|---|
| v0.1 | 2026-07-03 | Initial draft. Incorporated prior discussions on capture friction, role-based buckets, graph RAG, multi-surface exposure, TOON format, resilience requirements. |
