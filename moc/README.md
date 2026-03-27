# MOC — Medical Device Case Management ERP

> Architecture documentation — source code private per client agreement

## The Problem

Medical device companies in Colombia manage hundreds of simultaneous cases across multiple insurance providers (EPS/ARL). Each provider has different document requirements, SLAs, and workflows. Without a centralized system, cases get lost between stages, SLA deadlines are missed, and traceability is nonexistent.

## The Solution

Multi-tenant B2B SaaS ERP managing the full lifecycle of medical device cases — from initial medical order through authorization, device fabrication, delivery, and billing. Business rules are configurable per insurance provider without code changes.

## Architecture

```
┌──────────────────────────────────────────────┐
│            Frontend (React 19 + TS)          │
│  Kanban Board | Case Detail | Admin Config   │
│  Dashboard | Reports | Work Queue            │
│                                              │
│  TanStack Query (server state)               │
│  Zustand (client state)                      │
│  CVA + Tailwind (design system)              │
└──────────────────┬───────────────────────────┘
                   │ HTTP/JSON (JWT Bearer)
┌──────────────────▼───────────────────────────┐
│          Backend — Modular Monolith           │
│          FastAPI + SQLAlchemy 2.0 async       │
│                                              │
│  ┌─────────────────────────────────────────┐ │
│  │        Router Layer (16 routers)        │ │
│  └──────────────────┬──────────────────────┘ │
│                     │                        │
│  ┌──────────────────▼──────────────────────┐ │
│  │    Dependencies (Auth + RBAC + Tenant)  │ │
│  │  EmpresaContext | require_action        │ │
│  │  validate_caso_access | scope_query     │ │
│  └──────────────────┬──────────────────────┘ │
│                     │                        │
│  ┌──────────────────▼──────────────────────┐ │
│  │       Service Layer (8 services)        │ │
│  │  Workflow | AlertEngine | SLA           │ │
│  │  Bitacora | Consecutivo | WorkQueue     │ │
│  └──────────────────┬──────────────────────┘ │
│                     │                        │
│  ┌──────────────────▼──────────────────────┐ │
│  │    Domain Layer (constants, RBAC,       │ │
│  │    event types, Kanban columns)         │ │
│  └──────────────────┬──────────────────────┘ │
│                     │                        │
│  ┌──────────────────▼──────────────────────┐ │
│  │         Data Layer (30 models)          │ │
│  │    SQLAlchemy ORM + Pydantic schemas    │ │
│  └──────────────────┬──────────────────────┘ │
└──────────────────────┬───────────────────────┘
                       │
              ┌────────▼────────┐
              │  PostgreSQL 16  │
              │  + JSONB        │
              │  + Partial Idx  │
              │  + 20+ migrations│
              └─────────────────┘
```

## Technical Highlights

### Composable Gate Pattern — State Transitions

**Problem:** Each insurance provider has different rules for when a case can advance to the next stage. Rules change monthly.

**Solution:** 5 independent gates that execute in sequence before any state transition:

| Gate | What it validates |
|------|------------------|
| Transition rules | Is this move allowed? (e.g., can't skip stages) |
| Required documents | Are all documents for this stage uploaded? |
| Mandatory fields | Are all required fields filled (including dynamic fields)? |
| Blocking issues | Are there any unresolved blocking issues? |
| Pending tasks | Are all mandatory tasks completed? |

If any gate fails, the user sees exactly what's missing. Gates are configured per insurance provider in the database.

**Why not a state machine?** A traditional state machine hardcodes transitions. In this domain, an insurance provider can change their document requirements any month. With gates, the admin configures that in the UI — no deploy needed.

---

### Multi-Tenant Isolation

**Problem:** Multiple companies share one database. A bug must never expose data from one company to another.

**Solution:** SQL-level scoping — every query is filtered by `empresa_id` before execution, not after fetching results. Enforced through FastAPI dependency injection (`scope_caso_query()`).

Two isolation levels:
- **Company level:** All data filtered by `empresa_id`
- **Aliado level:** Partner companies see only cases assigned to them via `aliado_caso_ids()` subquery

**Security detail:** Access denied returns 404 (not 403) to prevent resource enumeration — an attacker can't probe whether a resource exists.

**Why SQL-level?** Post-fetch filtering is a security risk. If a developer writes a query without the filter, the DB could return data from all tenants. With SQL-level scoping injected via dependency, it's hard to skip.

---

### Alert Engine — Strategy Pattern

**Problem:** Cases slip through cracks — SLA deadlines missed, cases unattended for days, blocking issues unresolved.

**Solution:** 7 alert rules, each implemented as a class with a common interface:

| Alert | Trigger |
|-------|---------|
| SLA at risk | > 75% of SLA time consumed |
| SLA expired | 100% of SLA time consumed |
| Unattended case | No activity for 3+ days |
| Prolonged blocking issue | Blocking issue open 5+ days |
| Overdue tasks | Tasks past their due date |
| Stalled case | Case stuck in same stage 10+ days |
| Pending documentation | Required documents missing 7+ days |

**Dual evaluation:**
- **Reactive:** Runs immediately when a user modifies a case (create, move, complete task). If you complete a task that resolves an alert, it disappears instantly.
- **Batch:** Runs every 4 hours for time-based alerts. You can't detect "no activity for 3 days" reactively — nobody touched the case.

**Deduplication:** PostgreSQL partial unique index: `UNIQUE(caso_id, tipo) WHERE status = 'ACTIVA'`. Only one active alert per case per type. Resolved alerts stay as history.

---

### Optimistic Locking

**Problem:** Two operators editing the same case simultaneously — last write wins, first user's changes lost.

**Solution:** `version` field on each case, incremented on every update. On save, the system checks:
```
UPDATE caso SET ... WHERE id = :id AND version = :expected_version
```
If another user modified the case first (version changed), the update affects 0 rows → 409 Conflict response.

**Why optimistic over pessimistic?** Conflicts are rare — operators usually work on their own assigned cases. Pessimistic locking (`SELECT FOR UPDATE`) blocks the row for all readers, hurting performance in a system with many concurrent viewers.

**Exception:** Consecutive number generation (case numbering) uses an atomic UPSERT — there, conflicts are frequent and correctness is critical.

---

### SLA Engine with Business Days

**Problem:** SLA deadlines must respect Colombian business days (no weekends) and auto-pause when the case is waiting on external parties.

**Solution:**
- **Deadline calculation:** Sum business days from case creation, skipping weekends
- **Auto-pause:** When a case enters a wait state (e.g., "Waiting for EPS authorization"), the SLA clock stops. Pause windows stored in JSONB array
- **Auto-resume:** When the case exits the wait state, the deadline extends by pause duration
- **Progress:** Calculated as percentage of consumed time vs total. Alert at 75% (warning) and 100% (critical)

---

### Append-Only Audit Trail

**Problem:** Healthcare regulations require full traceability of every action on every case.

**Solution:** Bitácora table — append-only log with 75+ event types categorized by domain (case, task, document, SLA, auth, system). Each entry records:
- Who (user ID + role)
- What (event type)
- When (timestamp)
- Changes (old → new values in metadata JSONB)

Records are never updated or deleted.

**Why not full event sourcing?** In full event sourcing, you reconstruct state from events. Here, state lives in normal tables — the bitácora is a log for audit and traceability, not a source of truth. Simpler to implement, simpler to query.

---

## System Metrics

| Metric | Value |
|--------|-------|
| Total codebase | ~35,000 lines |
| Backend models | 30 SQLAlchemy ORM models |
| API endpoints | 16 routers |
| Business services | 8 |
| Automated tests | 235+ |
| Audit event types | 75+ |
| RBAC matrix | 5 roles × 24 permissions |
| DB migrations | 20+ (Alembic) |

## Tech Stack

| Component | Technology |
|-----------|-----------|
| Backend | FastAPI, SQLAlchemy 2.0 async, Pydantic v2 |
| Database | PostgreSQL 16 (JSONB, partial indexes) |
| Auth | JWT (access + refresh token rotation) |
| Frontend | React 19, TypeScript, TanStack Query, Zustand |
| Styling | Tailwind CSS, CVA (class-variance-authority) |
| Migrations | Alembic |
| Infrastructure | Docker, Docker Compose |

## What I Would Improve

- **AWS deployment:** ECS Fargate + RDS + S3 (document storage) + SQS (async alerts) with Terraform
- **Observability:** Structured logging with request correlation, Prometheus metrics, Grafana dashboards
- **Event-driven alerts:** Move alert processing to a message queue instead of synchronous evaluation
- **Load testing:** Concurrent case transitions and multi-tenant isolation under stress
