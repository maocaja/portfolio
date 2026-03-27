# Inbest Hub — AI Real Estate Investment Platform

> Architecture documentation — source code private per client agreement

## The Problem

Real estate investors in Latin America face fragmented information across multiple portals, no integrated financial analysis, and no intelligent guidance through the decision process.

## The Solution

SaaS platform combining hybrid search (SQL + semantic similarity with embeddings) and a conversational AI agent that acts as a virtual sales advisor — search, compare, analyze ROI, and estimate closing costs through natural language. Part of the founding technical team — designed core architecture and delivered the MVP (Jul 2024 – Mar 2026).

## Architecture

```
┌─────────────────────────────────┐
│     Frontend (React + TS)       │
│  Chat UI (SSE) | Canvas | Admin │
└──────────┬──────────────────────┘
           │ HTTP/JSON
    ┌──────┴──────┐
    │ API Gateway │
    └──┬───────┬──┘
       │       │
┌──────▼──┐ ┌──▼──────────────┐
│  Chat   │ │   Projects      │
│ Service │→│   Service       │
│         │ │                 │
│LangGraph│ │ Hybrid Search   │
│  ReAct  │ │ CRUD + Auth     │
│ 5 tools │ │ Embeddings (E5) │
│         │ │ Index Pipeline  │
│ Redis   │ │                 │
│(session)│ │ Repository +    │
│         │ │ Unit of Work    │
│ChromaDB │ │                 │
│  (RAG)  │ │                 │
└─────────┘ └───────┬─────────┘
                    │
          ┌─────────▼─────────┐
          │   PostgreSQL 14+  │
          │  + pgvector       │
          │  + search_index   │
          │  + index_queue    │
          │                   │
          │   Redis (cache)   │
          │   S3 (images)     │
          └───────────────────┘
```

## Technical Highlights

### Hybrid Search Engine

**Problem:** Users search by intention ("quiet place to retire near parks"), not just structured filters.

**Solution:** Combined scoring in a single SQL query:
- 70% — structured SQL filters (city, price, bedrooms, transaction type)
- 30% — cosine similarity between query embedding and property embeddings (pgvector)
- 5% bonus — price competitiveness per square meter

Embeddings generated with E5 model. Weights configurable from database without deploy.

**Trade-off:** pgvector in same PostgreSQL instance = simpler operations, single query with JOIN. Won't scale beyond ~100K properties. At that point, would evaluate Pinecone or dedicated vector DB.

---

### ReAct Conversational Agent

**Problem:** Users need guided property discovery, not just search results.

**Solution:** LangGraph ReAct agent with 5 tools:

| Tool | What it does |
|------|-------------|
| `search_properties` | Calls hybrid search API with extracted filters |
| `get_property_detail` | Retrieves full property info from session results |
| `compare_properties` | Side-by-side comparison builder |
| `calculate_roi` | ROI, cap rate, cash-on-cash return |
| `estimate_closing_costs` | Taxes, legal fees, registration costs |

The LLM (GPT-4o-mini, temperature 0.4) decides which tool to use based on user intent. Responses stream in real-time via SSE.

**Why SSE over WebSocket?** Streaming is unidirectional (server → client). SSE works over standard HTTP, no protocol upgrade needed, and HTTP/2 multiplexing reduces overhead.

**Why LangGraph over LangChain chains?** Chains are linear (A → B → C). ReAct needs a loop — the agent may call a tool, observe the result, and decide to call another. That's a graph with cycles, not a chain.

---

### Event-Driven Indexing Pipeline

**Problem:** Search index must stay in sync with CRUD operations without blocking them.

**Solution:**
```
Property CRUD → index_queue (table) → Background Job → search_index
                                           │
                                    ┌──────┴──────┐
                                    │ Denormalize  │
                                    │ Generate     │
                                    │  embedding   │
                                    │ (cached by   │
                                    │  content     │
                                    │  hash)       │
                                    │ UPSERT       │
                                    └──────────────┘
```

Embeddings cached by hash of source text — if name, description, and amenities haven't changed, no API call is made.

**Trade-off:** Eventual consistency. There's a delay between CRUD update and search index update. Acceptable for real estate listings, not for financial transactions.

**At scale:** Would replace the queue table with SQS or Kafka, and the background job with dedicated workers processing in parallel.

---

### CQRS-lite Pattern

**Problem:** Normalized tables require expensive JOINs for search queries (properties + projects + amenities + cities).

**Solution:** Two data paths:
- **Write path:** Normalized tables (properties, projects, amenities) — optimized for integrity
- **Read path:** Denormalized `search_index` table with JSONB + embeddings — optimized for search speed, zero JOINs

Sync between paths handled by the indexing pipeline.

---

## Tech Stack

| Component | Technology |
|-----------|-----------|
| Backend (Projects) | FastAPI, SQLAlchemy 2.0 async, Pydantic v2 |
| Backend (Chat) | FastAPI, LangGraph, GPT-4o-mini |
| Database | PostgreSQL 14+ with pgvector extension |
| Search | Hybrid: SQL filters + cosine similarity |
| Embeddings | E5 model (OpenAI) |
| Sessions | Redis (production) / In-memory (development) |
| Frontend | React, TypeScript, TanStack Query, Zustand |
| Infrastructure | Docker, Docker Compose |

## What I Would Improve

- **Evaluation framework:** precision@k for search relevance, hallucination detection for the agent
- **Observability:** Prometheus + Grafana for latency, token usage, and search quality metrics
- **Infrastructure as Code:** Terraform for AWS deployment
- **Scale:** Dedicated vector DB if property count exceeds 100K
