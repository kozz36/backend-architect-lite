---
name: backend-architect
description: >
  Backend architecture expert for API design, database modeling, auth patterns, caching, and system design.
  Trigger: When designing APIs, choosing databases, planning auth systems, scaling backends, or making architecture decisions.
license: Apache-2.0
metadata:
  author: kozz36
  version: "2.0-lite"
---

## When to Use

- Designing a new API (REST, GraphQL, gRPC, tRPC)
- Choosing a database, ORM, or sync engine for a project
- Planning authentication and authorization systems
- Scaling a backend or deciding between monolith and microservices
- Implementing caching, background jobs, or event-driven patterns
- Setting up observability, health checks, or load testing
- Designing AI/vector infrastructure or agentic orchestration
- Planning local-first or edge-synchronized architectures
- Any architectural decision with long-term tradeoffs

---

## 1. Framework Selection

### Decision Matrix

| Scenario | Framework | Reason |
|----------|-----------|--------|
| Python — general API | **FastAPI** | Async, OpenAPI auto-gen, Pydantic validation |
| Python — high performance | **Litestar** | Faster than FastAPI, stricter typing |
| Python — content/admin heavy | **Django + DRF** | ORM, admin, batteries included |
| Node.js — general API | **Fastify** | Fast, schema-based, great plugin ecosystem |
| Node.js — edge / serverless | **Hono** | Ultra-lightweight, multi-runtime |
| Node.js — Bun runtime | **Elysia** | Native Bun, end-to-end types |
| Infrastructure / CLI / binary | **Go (Echo/Chi)** | Single binary, low memory, extreme concurrency |
| Cost-critical at massive scale | **Rust (Axum)** | Max performance, zero GC pauses |

### Team-size rule
- Solo / startup → FastAPI or Fastify (fast iteration)
- Team with TypeScript frontend → Fastify + tRPC (shared types)
- Team preferring Go → Echo or Chi for REST, gRPC for internal services
- Mixed team → FastAPI (Python ubiquity) or Fastify (JS familiarity)

---

## 2. Database Architecture

### Default Choices

| Need | Choice |
|------|--------|
| Primary relational store | **PostgreSQL** — JSONB, arrays, full-text search, extensions |
| Embedded / edge / serverless | **SQLite** — Litestream, rqlite, Turso |
| Cache / queue / pub-sub | **Redis** |
| Document store (justified) | **MongoDB** |
| Time-series | **TimescaleDB** |
| Vector / semantic search | **pgvector** or **Pinecone** / **Milvus** |
| Local-first sync | **PowerSync**, **Turso**, **ElectricSQL**, **Replicache** |

### PostgreSQL as Universal Store

PostgreSQL now serves transactional, relational, analytical, AND semantic workloads via JSONB, full-text search, **pgvector** (HNSW/IVF indexing), and TimescaleDB. Use pgvector as default vector strategy to eliminate sync overhead with external stores.

### SQLite Renaissance

Production-viable for distributed and edge: Litestream (WAL streaming), Turso (libSQL edge, 0ms read replicas), rqlite (Raft consensus), PowerSync (bidirectional delta sync with PostgreSQL).

### Local-First Sync Engines

| Engine | Architecture | Best For |
|--------|--------------|----------|
| **PowerSync** | Postgres ↔ SQLite via WebSocket delta sync | Strict referential integrity on client and server |
| **Turso** | libSQL edge with embedded read replicas | 0ms reads, branching, portability |
| **ElectricSQL** | Active sync from PostgreSQL | Rapid prototyping of offline-first migrations |
| **Replicache** | In-memory over IndexedDB | Document editors, collaboration (~100MB limit) |

### Vector Databases

| Solution | Best For |
|----------|----------|
| **pgvector** | Teams on Postgres; ACID semantic + relational unified |
| **Pinecone** | Enterprise speed-to-market; zero maintenance |
| **Milvus** | Massive-scale similarity search (images, bioinformatics) |
| **Weaviate** | Multimodal: semantic + lexical hybrid |
| **Chroma** | Rapid LLM prototyping, local-first |

**Algorithms**: HNSW (graph navigation), IVF, Product Quantization. **Dimensions**: 1536-dim for accuracy (OpenAI ada-002), 384-dim for 4x speed (all-MiniLM-L6-v2).

### ORM Choices

| Ecosystem | ORM | Notes |
|-----------|-----|-------|
| Python | **SQLAlchemy 2.0** | Async, unit-of-work, Core expressions |
| Python (FastAPI) | **SQLModel** | SQLAlchemy + Pydantic |
| Node.js (TypeScript) | **Drizzle** | Type-safe, SQL-like, migration lifecycle |
| Node.js (any) | **Prisma 7** | Schema-first, great DX |

### Patterns

- **Repository Pattern**: abstracts data access, inject via DI
- **Unit of Work**: wraps repos in one transaction
- **CQRS Light**: separate read models from write models

---

## 3. API Design

### Protocol Matrix

| Scenario | Protocol |
|----------|----------|
| Public / third-party | REST + OpenAPI |
| Own TS frontend | tRPC |
| Multiple clients | GraphQL |
| Internal microservices | gRPC |
| Real-time / streaming | WebSocket or SSE |

### REST Best Practices

- **Versioning**: URL-based (`/api/v1/users`)
- **Pagination**: Cursor-based for feeds and large tables; offset only for simple cases
- **Rate Limiting**: token bucket or sliding window
- **Webhooks**: idempotent (event_id), HMAC-signed, exponential backoff retry

### Backend for Frontend (BFF)

Provision dedicated backends per client (iOS, Web, Android). Responsibilities: aggregate microservices, translate protocols, hide internal schemas, optimize payloads.

**Anti-pattern**: absorbing domain logic → accidental integration monolith. Keep BFF spartan.

**Best practice**: Same language as client (TypeScript/Node.js), same monorepo, owned by frontend team. Shift from passive HTTP to persistent WebSockets / GraphQL subscriptions for real-time.

---

## 4. Auth & Authorization

### JWT Patterns

```
Access token:  short-lived (15 min), stateless, signed
Refresh token: long-lived (7–30 days), stored in DB, rotated on use
```

**Signing priority**: EdDSA > ES256 > RS256 > HS256 (avoid HS256 for multi-service).

**Refresh rotation**: invalidate old on use, track family for breach detection.

### Auth Protocols

| Scenario | Choice |
|----------|--------|
| First-party app | JWT or sessions |
| Third-party OAuth | OAuth2 + PKCE |
| Machine-to-machine | API keys (hashed) or client_credentials |
| Enterprise SSO | SAML or OIDC |
| Passwordless | Passkeys (WebAuthn) |

### Sessions vs Tokens

- **Sessions**: instant revocation, stateful, need shared store (Redis)
- **JWTs**: stateless, cross-service, revocation needs denylist or short TTL

Use sessions for monoliths and traditional web apps. Use JWTs for microservices, mobile, SPAs.

### Authorization

- **RBAC**: roles → permissions. Default for most apps.
- **ABAC**: attribute-based policies. For complex multi-tenant or compliance systems.

### Zero-Trust for AI Agents

- Sandboxed execution environments
- Dynamic credential delegation with least-privilege
- Strictly scoped JWTs for AI-to-API interactions
- Structured audit logging of all agent actions

---

## 5. Caching

### Default Stack

**Redis** — cache, sessions, pub/sub, rate limiter, queues, agent state memory.

Connection pooling: always use a pool (`redis.asyncio.ConnectionPool`, `max_connections=20`).

### Patterns

| Pattern | When |
|---------|------|
| Cache-Aside (Lazy) | Read-heavy, tolerate stale data |
| Write-Through | Consistent but higher write latency |
| Write-Behind | High throughput, risk of loss |
| Read-Through | Library handles miss transparently |

### Invalidation

- **TTL-based**: simplest, good for reference data
- **Event-driven**: consistent but complex
- **Tag-based**: flush by entity tag, good for related data

---

## 6. AI & Agentic Orchestration

### Vector Search (pgvector)

Enable `CREATE EXTENSION vector;`. Store embeddings as `VECTOR(dims)`. Index with HNSW for approximate nearest neighbor. Query with cosine distance or inner product, filter metadata with JSONB.

### Agent Frameworks

| Framework | Best For |
|-----------|----------|
| **LangGraph** | Auditable, stateful multi-step LLM workflows |
| **Microsoft AutoGen** | Enterprise multi-agent: gRPC, OpenTelemetry, error handling |
| **CrewAI** | Rapid prototyping of hierarchical business teams |
| **Semantic Kernel** | Azure-native, deep semantic memory, deterministic planners |

### Redis as Agent Memory

- Conversation threading: sub-millisecond context retrieval
- State compaction: long-term/short-term hierarchy
- Vector search: sub-millisecond knowledge retrieval
- Volatile state: real-time mutation across agent swarms

---

## 7. Architecture Patterns

### Monolith First

```
< 20 engineers    → Monolith
< 5 services      → Modular monolith
> 5 services, clear contexts → Microservices (justify each)
> 50 engineers, org boundaries → Microservices (Conway's Law)
```

### Modular Monolith

Bounded contexts as modules (users/, billing/, notifications/). Modules communicate via interfaces, not direct cross-layer imports. Enables future extraction to services.

### Clean / Hexagonal

```
Domain → no framework dependencies
Application → use cases + ports (interfaces)
Infrastructure → adapters: DB, HTTP, queue
API → routes call application use cases
```

### Event-Driven

| Tool | Use When |
|------|----------|
| Redis Streams | Simple pub/sub, at-most-once |
| NATS | Lightweight, JetStream persistence |
| RabbitMQ | Complex routing, AMQP ecosystem |
| Kafka | High throughput, replay, audit log |

Start with Redis Streams. Migrate to Kafka when > 100k msg/s or replay needed.

### Background Jobs

| Tool | Ecosystem | Use When |
|------|-----------|----------|
| Dramatiq | Python | Default; simpler than Celery |
| ARQ | Python | Async-native, lightweight |
| BullMQ | Node.js | Redis-backed, delayed jobs |
| Temporal | Any | Long workflows, sagas, crash-proof execution |
| Inngest | Node.js | Serverless-friendly |

**Temporal**: Crash-proof execution — workflows resume at exact line and stack state after interruption, surviving seconds to months of downtime.

---

## 8. Testing

### Test Pyramid

```
Unit (70%)         — domain logic, pure functions
Integration (20%)  — real DB, real Redis, Testcontainers
E2E / Load (10%)   — full stack, production-like
```

**Rule**: mock HTTP calls, but use REAL databases. Mocking Postgres is lying to yourself.

### Testcontainers

Programmatically deploy real PostgreSQL/Redis containers from test code. Resolves flakiness and ensures migrations run on production-identical binaries.

### Load Testing

**k6** (JS/TS scripted): smoke, stress, soak, spike testing. Browser emulation via POM. Kubernetes chaos injection via xk6-disruptor.

---

## 9. Platform Engineering & Observability

### IaC & GitOps

| Layer | Technology |
|-------|------------|
| Provisioning | Terraform / OpenTofu |
| GitOps / CD | Argo CD / GitLab CI |
| Secrets | Infisical / External Secrets Operator |
| Observability | OpenTelemetry + LGTM Stack |

**Golden rule**: ALL topology is defined, reviewed, regression-tested, and deployed as declarative code.

### Observability Stack

| Concern | Tool |
|---------|------|
| Tracing + metrics + logs | OpenTelemetry |
| Structured logs (Python) | structlog |
| Structured logs (Node.js) | pino |
| Metrics | Prometheus + Mimir |
| Visualization | Grafana |
| Log aggregation | Loki |
| Trace backend | Tempo |
| Error tracking | Sentry |

### Health Checks

```
/health/live  → liveness probe (process alive)
/health/ready → readiness probe (DB, cache, dependencies accessible)
```

---

## 10. Decision Framework

```
1. LANGUAGE
   ├── Python?        → FastAPI / Litestar / Django
   ├── JS/TS?         → Fastify / Hono / Elysia
   ├── Infra / CLI?   → Go
   └── Max perf?      → Rust (Axum)

2. TEAM SIZE
   ├── < 20  → Monolith / Modular Monolith
   └── > 20  → Microservices (justify each)

3. API
   ├── Public        → REST + OpenAPI
   ├── Own TS client → tRPC
   ├── Multi-client  → GraphQL
   └── Internal      → gRPC

4. DATABASE
   ├── Default        → PostgreSQL
   ├── Edge           → SQLite + Litestream / Turso
   ├── Vector / AI    → pgvector or Pinecone
   ├── Local-first    → PowerSync / Turso
   ├── Variable schema → MongoDB
   └── Time-series    → TimescaleDB

5. ORM
   ├── Python      → SQLAlchemy 2.0 / SQLModel
   ├── Node.js     → Drizzle / Prisma
   └── Complex     → Raw SQL

6. AUTH
   ├── SPA/mobile  → JWT (EdDSA)
   ├── Web app     → Sessions (Redis)
   ├── OAuth       → OAuth2 + PKCE
   ├── S2S         → API keys
   └── Enterprise  → OIDC / SAML

7. JOBS
   ├── Python     → Dramatiq / ARQ
   ├── Node.js    → BullMQ / Inngest
   └── Sagas      → Temporal

8. CACHING
   └── Default    → Redis (Cache-Aside)

9. AI / AGENTS
   ├── Vectors     → pgvector / Pinecone
   ├── Workflows   → LangGraph
   ├── Multi-agent → AutoGen
   └── Memory      → Redis multi-tier

10. PLATFORM
    ├── IaC        → Terraform / OpenTofu
    ├── GitOps     → Argo CD
    ├── Secrets    → Infisical
    └── Observability → OpenTelemetry + LGTM
```

---

## Commands

```bash
# FastAPI bootstrap
uv init my-api && cd my-api
uv add fastapi uvicorn[standard] sqlalchemy[asyncio] asyncpg pydantic-settings redis structlog

# Fastify bootstrap
npm create fastify@latest my-api -- --lang=ts
npm install @fastify/swagger @fastify/swagger-ui @fastify/rate-limit

# Drizzle setup
npm install drizzle-orm pg && npm install -D drizzle-kit @types/pg
npx drizzle-kit generate && npx drizzle-kit migrate

# Testcontainers (Python)
uv add --dev testcontainers pytest-asyncio factory-boy

# k6 load test
k6 run scripts/load-test.js

# OpenTelemetry (Python)
uv add opentelemetry-sdk opentelemetry-instrumentation-fastapi opentelemetry-instrumentation-sqlalchemy opentelemetry-exporter-otlp

# pgvector via Docker
docker pull ankane/pgvector

# Temporal CLI
brew install temporal
```

## Resources

- FastAPI: https://fastapi.tiangolo.com
- SQLAlchemy 2.0: https://docs.sqlalchemy.org/en/20/
- Drizzle ORM: https://orm.drizzle.team
- Testcontainers: https://testcontainers-python.readthedocs.io
- OpenTelemetry: https://opentelemetry-python.readthedocs.io
- k6: https://grafana.com/docs/k6/latest/
- Temporal: https://docs.temporal.io
- pgvector: https://github.com/pgvector/pgvector
- PowerSync: https://www.powersync.com
- LangGraph: https://langchain-ai.github.io/langgraph/
- AutoGen: https://microsoft.github.io/autogen/
- Infisical: https://infisical.com
- OpenTofu: https://opentofu.org
- Argo CD: https://argo-cd.readthedocs.io
