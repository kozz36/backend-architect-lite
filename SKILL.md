---
name: backend-architect
description: >
  Backend architecture expert for API design, database modeling, auth patterns, caching, and system design.
  Trigger: When designing APIs, choosing databases, planning auth systems, scaling backends, or making architecture decisions.
license: Apache-2.0
metadata:
  author: kozz36
  version: "2.0"
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
| Python — general API | **FastAPI** | Async, OpenAPI auto-gen, Pydantic validation layers |
| Python — high performance | **Litestar** | Faster than FastAPI, stricter typing |
| Python — content/admin heavy | **Django + DRF** | ORM, admin, batteries included |
| Node.js — general API | **Fastify** | Fast, schema-based, great plugin ecosystem |
| Node.js — edge / serverless | **Hono** | Ultra-lightweight, multi-runtime |
| Node.js — Bun runtime | **Elysia** | Native Bun, end-to-end types |
| Infrastructure / CLI / binary | **Go (Echo/Chi)** | Single binary, low memory, extreme concurrency |
| Cost-critical at massive scale | **Rust (Axum)** | Max performance, zero GC pauses, deterministic memory safety |

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
| Embedded / edge / serverless | **SQLite** — with Litestream (replication), rqlite (Raft), or Turso (distributed libSQL) |
| Cache / queue / pub-sub / multi-tier memory | **Redis** |
| Document store (justified) | **MongoDB** — only when schema is genuinely variable |
| Time-series | **TimescaleDB** (PostgreSQL extension) or **InfluxDB** |
| Vector search / semantic retrieval | **pgvector** (PostgreSQL extension) or **Pinecone** / **Milvus** |
| Local-first sync engine | **PowerSync**, **Turso**, **ElectricSQL**, or **Replicache** |

### PostgreSQL as Universal Store

PostgreSQL now serves transactional, relational, analytical, AND semantic workloads:
- **JSONB**: schemaless patterns inside ACID transactions
- **Full-text search**: native tsvector/tsquery
- **pgvector**: first-class vector support (cosine distance, inner product / dot product, HNSW/IVF indexing)
- **TimescaleDB**: hypertables for time-series data

Use **pgvector** as the default vector strategy for teams already operating massive PostgreSQL estates. Eliminates data synchronization overhead with external vector stores.

### SQLite Renaissance (2024–2026)

SQLite is now production-viable for distributed and edge workloads:
- **Litestream**: streams WAL to S3/GCS — free disaster recovery
- **Turso**: libSQL fork, edge-deployed, HTTP API, per-tenant DBs, embedded read replicas at 0ms latency
- **rqlite**: clustered SQLite with Raft — replicated, lightweight
- **PowerSync**: bidirectional delta sync between PostgreSQL central and local SQLite via persistent WebSockets

Use when: single-region, low-concurrency writes, cost-sensitive, per-tenant isolation, or local-first architectures.

### Local-First & Edge Synchronization

| Sync Engine | Architecture | Best For |
|-------------|--------------|----------|
| **PowerSync** | PostgreSQL ↔ SQLite via WebSocket delta sync, automatic conflict resolution | Complex apps requiring strict referential integrity on client and server |
| **Turso** | libSQL edge deployment with embedded read replicas, branching workflows | Portability and 0ms read latency at the edge |
| **ElectricSQL** | Active sync from PostgreSQL to local clients, deterministic conflict resolution | Rapid prototyping and migration of existing Postgres apps to offline-first |
| **Replicache** | In-memory transactional store over IndexedDB, explicit sync control, undo/redo | Document editors, collaboration tools (limit ~100MB data) |

### Vector Databases & RAG Infrastructure

| Solution | Deployment | Best For |
|----------|------------|----------|
| **pgvector** | PostgreSQL native extension | Teams already on Postgres; ACID semantic + relational in one store |
| **Pinecone** | Cloud-native, serverless SaaS | Enterprise speed-to-market; zero infrastructure maintenance |
| **Milvus** | Open-source, Kubernetes-native | Massive-scale similarity search (image search, bioinformatics) |
| **Weaviate** | Open-source, schema-flexible with optional strict mode | Multimodal apps combining semantic + lexical search |
| **Chroma** | Lightweight, embedded or client-server | Rapid LLM prototyping, LangChain integration |

**Key algorithms**: HNSW (probabilistic multilayer graph navigation), IVF (inverted file index), Product Quantization (memory compression).

**Dimensionality tradeoff**: 1536-dim (OpenAI text-embedding-3-small) for deep semantic accuracy; 384-dim (all-MiniLM-L6-v2) trades semantic depth for significantly lower latency and memory. Actual speedup depends on index parameters — benchmark with your dataset.

### ORM vs Raw SQL

```
Schema-first, complex queries → Raw SQL (psycopg3, asyncpg) — MUST use parameterized statements: text("... WHERE id = :id").bindparams(id=user_id)
Rapid prototyping, teams      → ORM
Mixed (recommended)           → ORM for CRUD, raw SQL for reports/analytics
```

### ORM Choices

| Ecosystem | ORM | Notes |
|-----------|-----|-------|
| Python | **SQLAlchemy 2.0** | Mature, async support, unit-of-work, Core expressions |
| Python (FastAPI) | **SQLModel** | SQLAlchemy + Pydantic, less boilerplate |
| Node.js (TypeScript) | **Drizzle** | Type-safe, SQL-like syntax, no magic, migration lifecycle |
| Node.js (any) | **Prisma 7** | Schema-first, migrations, great DX |

### Patterns

- **Repository Pattern**: abstracts data access; inject via DI for testability
- **Unit of Work**: wraps multiple repos in one transaction (SQLAlchemy Session is UoW)
- **CQRS Light**: separate read models (optimized queries) from write models (ORM entities); no full event sourcing needed for most apps

```python
# Repository pattern with SQLAlchemy 2.0
# WARNING: async sessions should set expire_on_commit=False and return DTOs
# to avoid DetachedInstanceError when accessing lazy-loaded attributes outside session scope
class UserRepository:
    def __init__(self, session: AsyncSession):
        self.session = session

    async def get_by_id(self, user_id: UUID) -> User | None:
        result = await self.session.get(User, user_id)
        return result

    async def get_by_email(self, email: str) -> User | None:
        stmt = select(User).where(User.email == email)
        result = await self.session.execute(stmt)
        return result.scalar_one_or_none()
```

---

## 3. API Design

### Protocol Decision Matrix

| Scenario | Protocol |
|----------|----------|
| Public API, third-party consumers | **REST + OpenAPI** |
| Frontend-only, TypeScript stack | **tRPC** (end-to-end types, no schema drift) |
| Flexible queries, multiple clients | **GraphQL** (Apollo / Strawberry) |
| Internal microservice communication | **gRPC** (protobuf, streaming, low latency) |
| Real-time / streaming | **WebSocket** or **SSE** |

### REST Best Practices

**Versioning**: URL-based wins in practice (`/api/v1/users`) — header versioning hurts caching and tooling.

**Pagination**:
- Offset (`?page=2&limit=20`): simple, but expensive on large datasets (OFFSET scans)
- Cursor (`?after=eyJpZCI6MTAwfQ`): scalable, consistent under mutations — prefer for feeds and large tables

**Rate Limiting**: token bucket or sliding window. Headers: `RateLimit-Limit`, `RateLimit-Remaining`, `RateLimit-Reset` (IETF standard).

**Webhook Design**:
```python
# Idempotency: include event ID, receiver deduplicates
# HMAC signing: sign payload with shared secret
import hmac, hashlib

def verify_webhook(payload: bytes, signature: str, secret: str) -> bool:
    expected = hmac.new(
        secret.encode(), payload, hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(f"sha256={expected}", signature)

# Retry: exponential backoff (1s, 5s, 30s, 5m, 30m)
# Delivery guarantee: at-least-once — always include event_id for dedup
```

### OpenAPI

FastAPI generates OpenAPI automatically. For other frameworks:
- **Litestar**: built-in OpenAPI
- **Fastify**: `@fastify/swagger` + `@fastify/swagger-ui`
- **Go**: `swaggo/swag` or `oapi-codegen`

Always include: request/response schemas, error responses (400, 401, 403, 404, 422, 500), auth scheme.

### Backend for Frontend (BFF)

The BFF pattern provisions a dedicated backend per client experience (iOS, Web, Android) rather than forcing a single universal API.

**BFF Responsibilities**:
- Aggregating data from dozens of downstream microservices
- Translating protocols (e.g., internal gRPC → public JSON)
- Hiding internal data schemas from frontend clients (security posture)
- Reducing payload size for mobile (bandwidth/battery optimization)

**Anti-pattern warning**: BFF must remain spartan. Do NOT absorb domain validations or heavy transactional logic — that creates an accidental integration monolith.

**Best practice**: Write BFF in the same stack as the frontend team (typically TypeScript/Node.js with Fastify/Hono for web clients), host in the same monorepo, and assign ownership to the frontend team. Mobile BFFs (iOS/Android) are usually shared or language-agnostic. Reserve deep microservice design for pure backend engineers.

**Protocol shift**: Modern BFF increasingly moves from passive HTTP request/response to persistent WebSockets and GraphQL subscriptions for real-time state propagation (inventory, stocks, AI reasoning streams).

---

## 4. Auth & Authorization

### JWT Patterns

```
Access token:  short-lived (15 min), stateless, signed
Refresh token: long-lived (7–30 days), stored in DB, rotated on use
```

**Signing algorithm priority**: EdDSA (Ed25519) > ES256 (ECDSA P-256) > RS256 (RSA 2048) > HS256 (shared secret — avoid for multi-service)

```python
# FastAPI JWT pattern — HMAC (default, safest for single-service)
from datetime import datetime, timedelta, UTC
import jwt, os, secrets

SECRET_KEY = os.environ["JWT_SECRET_KEY"]  # 32+ bytes for HS256
ALGORITHM = "HS256"

def create_access_token(subject: str) -> str:
    expire = datetime.now(UTC) + timedelta(minutes=15)
    return jwt.encode({"sub": subject, "exp": expire, "aud": "my-api", "iss": "auth-service"}, SECRET_KEY, algorithm=ALGORITHM)

def create_refresh_token(subject: str) -> dict:
    expire = datetime.now(UTC) + timedelta(days=30)
    token = secrets.token_urlsafe(32)
    # Store hash(token) + expiry in DB
    return {"token": token, "expires_at": expire.isoformat()}
```

**EdDSA (Ed25519) for multi-service**: Requires `PyJWT[crypto]` + `cryptography`. Load PEM key or generate `Ed25519PrivateKey` object. Sign with `jwt.encode(payload, private_key, algorithm="EdDSA")`.

**Refresh token rotation**: invalidate old token on use, issue new pair. Track family for breach detection (if rotated token is reused → family was stolen → revoke all).

### Auth Protocols

| Scenario | Choice |
|----------|--------|
| First-party app (SPA/mobile) | OAuth2 + PKCE or JWT (access + refresh) |
| First-party app (web app) | Sessions (Redis store) or JWT |
| Third-party OAuth (Google, GitHub) | OAuth2 + PKCE (Authorization Code flow) |
| Machine-to-machine / S2S | API keys (hashed in DB) or client_credentials OAuth2 |
| Enterprise SSO | SAML or OIDC |
| Passwordless 2025 | **Passkeys** (WebAuthn) — phishing-resistant, UX win |

### Sessions vs Tokens

```
Sessions (server-side):
  + Instant revocation
  + No token size limit
  - Requires shared session store (Redis) for multi-instance
  - Stateful

JWTs (stateless):
  + No shared store needed
  + Works across services
  - Revocation requires denylist (Redis) or short TTL
  - Token size grows with claims
```

Use sessions for: monolith, traditional web apps, admin panels.
Use JWTs for: microservices, mobile apps, SPAs, multi-tenant APIs.

### Authorization

**RBAC** (Role-Based): roles → permissions. Simple, audit-friendly. Use for most apps.

**ABAC** (Attribute-Based): policies evaluate attributes (user, resource, environment). Use for complex multi-tenant or compliance-heavy systems.

```python
# FastAPI RBAC dependency — supports multiple roles per user
from fastapi import Depends, HTTPException, status

def require_any_role(*roles: str):
    def dependency(current_user: User = Depends(get_current_user)):
        if not any(r in current_user.roles for r in roles):
            raise HTTPException(status_code=status.HTTP_403_FORBIDDEN)
        return current_user
    return dependency

# Usage
@router.delete("/users/{id}", dependencies=[Depends(require_any_role("admin", "superuser"))])
async def delete_user(id: UUID): ...
```

### Zero-Trust for Autonomous AI

- Sandboxed execution environments for agent code
- Dynamic credential delegation with least-privilege access
- Strictly scoped JWTs for AI-to-API interactions
- Audit all agent actions via structured logging

---

## 5. Caching

### Default Stack

**Redis** — primary cache, session store, pub/sub, rate limiter, job queue, and multi-tier memory for agent state.

Connection pooling: always use a pool. FastAPI + Redis: `redis.asyncio.ConnectionPool` with `max_connections=20`.

### Patterns

| Pattern | When |
|---------|------|
| **Cache-Aside** (Lazy) | Read-heavy, tolerate stale data. Check cache → miss → read DB → populate cache |
| **Write-Through** | Write to cache and DB together. Consistent but write latency |
| **Write-Behind** | Write to cache, async flush to DB. High write throughput, risk of loss |
| **Read-Through** | Cache fetches from DB on miss transparently (library handles it) |

### Invalidation Strategies

```
TTL-based:        simplest, stale within window — good for reference data
Event-driven:     publish cache-invalidation event on write — consistent, complex
Tag-based:        tag cache keys by entity, flush by tag — good for related data
```

```python
# Cache-Aside pattern with FastAPI + Redis
# NOTE: UserDTO is a Pydantic model (serialization layer). Never cache SQLAlchemy ORM objects directly.
import json

async def get_user(user_id: UUID, redis: Redis, db: AsyncSession) -> UserDTO:
    cache_key = f"user:{user_id}"
    cached = await redis.get(cache_key)
    if cached:
        return UserDTO.model_validate_json(cached)

    user = await UserRepository(db).get_by_id(user_id)
    if user:
        dto = UserDTO.model_validate(user)
        await redis.setex(cache_key, 300, dto.model_dump_json())  # TTL: 5 min
    return user
```

---

## 6. AI & Agentic Orchestration

### Vector Search Implementation (pgvector)

```sql
-- Enable extension
CREATE EXTENSION IF NOT EXISTS vector;

-- Table with embeddings
CREATE TABLE documents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    content TEXT NOT NULL,
    embedding VECTOR(1536),  -- OpenAI text-embedding-3-small default
    metadata JSONB
);

-- HNSW index for approximate nearest neighbor search
CREATE INDEX idx_documents_embedding ON documents
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);

-- Semantic search query
-- NOTE: <=> returns cosine DISTANCE (0 = identical, 2 = opposite).
-- Cosine similarity of normalized vectors is in [-1, 1].
-- Use 1 - distance only if you know vectors are normalized; otherwise use distance directly.
-- If vectors aren't normalized, use distance directly and ORDER BY ascending.
SELECT id, content, 1 - (embedding <=> query_embedding) AS similarity
FROM documents
WHERE metadata @> '{"category": "technical"}'
ORDER BY embedding <=> query_embedding
LIMIT 10;
```

### Agent Orchestration Frameworks

| Framework | Paradigm | Best For |
|-----------|----------|----------|
| **LangGraph** | Stateful graph workflows in Python/TypeScript | Auditable, predictable multi-step LLM pipelines with recoverable memory |
| **Microsoft AutoGen** | Multi-agent collaborative dialogue (Python/.NET) | Enterprise production: error handling, distributed gRPC, OpenTelemetry |
| **CrewAI** | Role-driven agent teams (Python) | Rapid prototyping of hierarchical business workflows |
| **Semantic Kernel** | Dynamic function binding (.NET/Python) | Azure-native apps with deep semantic memory and deterministic planners |

### Redis as Multi-Tier Agent Memory

Redis has evolved from peripheral cache to the central nervous system of agent architectures:
- **Conversation threading**: sub-millisecond context retrieval across distributed agents
- **State compaction**: long-term/short-term memory hierarchy management
- **Vector search**: sub-millisecond knowledge base retrieval impacting reasoning coherence
- **Volatile state management**: real-time mutation across agent swarms

---

## 7. Architecture Patterns

### Monolith First (consensus 2024–2026)

Teams under 20 engineers: start with a monolith. Microservices add operational complexity that kills small teams.

```
< 20 engineers    → Monolith (deploy as one unit)
< 5 services      → Modular monolith (internal modules, shared DB)
> 5 services, clear bounded contexts → Microservices (justify each split)
> 50 engineers, org boundaries → Microservices (Conway's Law)
```

### Modular Monolith Structure

```
src/
├── users/          # bounded context
│   ├── domain/     # entities, value objects
│   ├── app/        # use cases, services
│   ├── infra/      # repository impl, external calls
│   └── api/        # routes, schemas
├── billing/
├── notifications/
└── shared/         # cross-cutting (auth, db, config)
```

Modules communicate via **interfaces** (not direct imports across layers). This allows future extraction to services.

### Clean / Hexagonal Architecture

```
Domain (entities, value objects) — no framework dependencies
Application (use cases) — depends on domain + ports (interfaces)
Infrastructure (adapters) — implements ports: DB, HTTP, queue
API (delivery) — FastAPI routes call application use cases
```

```python
# Port (interface) in application layer
from abc import ABC, abstractmethod

class UserRepository(ABC):
    @abstractmethod
    async def get_by_id(self, user_id: UUID) -> User | None: ...
    @abstractmethod
    async def save(self, user: User) -> None: ...

# Adapter (implementation) in infrastructure layer
class PostgresUserRepository(UserRepository):
    def __init__(self, session: AsyncSession):
        self.session = session

    async def get_by_id(self, user_id: UUID) -> User | None:
        return await self.session.get(UserModel, user_id)
```

### Event-Driven

| Tool | Use When |
|------|----------|
| **Redis Streams** | Simple pub/sub, same infra as cache, **at-least-once** via consumer groups and XACK |
| **NATS** | Lightweight, JetStream for persistence, fast fan-out |
| **RabbitMQ** | Complex routing, existing AMQP ecosystem |
| **Kafka** | High throughput, event log, replay, analytics pipeline |

Rule: start with Redis Streams. Migrate to Kafka when you need replay, audit log, or > 100k msg/s.

### Background Jobs & Resilient Workflows

| Tool | Ecosystem | Use When |
|------|-----------|----------|
| **Dramatiq** | Python | Preferred over Celery: simpler, reliable, middleware-based |
| **ARQ** | Python | Async-native, Redis-backed, lightweight |
| **BullMQ** | Node.js | Redis-backed, delayed jobs, rate limiting built-in |
| **Temporal** | Any | Long-running workflows, sagas, crash-proof execution |
| **Inngest** | Node.js | Serverless-friendly, event-driven functions |

**Temporal**: Represents a qualitative leap for mission-critical systems. It abstracts state management and retry routing, promising "crash-proof execution" — workflows resume via deterministic replay from event history, not by serializing native stack frames. This survives outages from seconds to months.

```python
# Dramatiq example (Python)
import dramatiq
from dramatiq.brokers.redis import RedisBroker

broker = RedisBroker(host="localhost", port=6379, db=0)
dramatiq.set_broker(broker)

@dramatiq.actor(max_retries=3, min_backoff=1000)
def send_welcome_email(user_id: str) -> None:
    user = get_user(user_id)
    email_service.send(user.email, "Welcome!")
```

---

## 8. Testing

### Test Pyramid

```
Unit tests (70%)      — domain logic, pure functions, no I/O
Integration tests (20%) — real DB, real Redis, Testcontainers
E2E / load tests (10%)  — full stack, production-like
```

**Rule**: mock external HTTP calls (httpx mock, respx), but use REAL databases in integration tests. Mocking Postgres is lying to yourself.

### Testcontainers

```python
# pytest + testcontainers for real PostgreSQL
import pytest, pytest_asyncio
from testcontainers.postgres import PostgresContainer
from sqlalchemy.ext.asyncio import create_async_engine
from sqlalchemy import URL

@pytest.fixture(scope="session")
def postgres():
    with PostgresContainer("postgres:17-alpine") as pg:
        yield pg

@pytest_asyncio.fixture(scope="session")
async def engine(postgres):
    # Build URL safely instead of naive string replacement
    url = URL.create(
        drivername="postgresql+asyncpg",
        username=postgres.username,
        password=postgres.password,
        host=postgres.get_container_host_ip(),
        port=postgres.get_exposed_port(5432),
        database=postgres.dbname,
    )
    engine = create_async_engine(url)
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield engine
    await engine.dispose()
```

### Factory Pattern

Use factories (not fixtures with hardcoded data) to build test objects:

```python
# factory-boy example
from uuid import uuid4
import factory
from factory.alchemy import SQLAlchemyModelFactory

class UserFactory(SQLAlchemyModelFactory):
    class Meta:
        model = User

    id = factory.LazyFunction(uuid4)
    email = factory.Sequence(lambda n: f"user{n}@example.com")
    name = factory.Faker("name")
    role = "user"
```

### Load Testing & Chaos Engineering

**k6** is the default. Scripted in JS/TS, integrates with CI, outputs InfluxDB/Prometheus metrics.

```javascript
// k6 smoke test
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = { vus: 10, duration: '30s' };

export default function () {
  const res = http.get('http://localhost:8000/api/v1/users');
  check(res, { 'status is 200': (r) => r.status === 200 });
  sleep(1);
}
```

**k6 disciplines**: smoke testing, stress testing (breakpoint detection), soak testing (memory leak detection), spike testing, browser emulation (POM), and Kubernetes chaos injection (xk6-disruptor).

---

## 9. Platform Engineering & Observability

### Infrastructure as Code (IaC)

| Layer | Technology | Purpose |
|-------|------------|---------|
| **Provisioning** | **Terraform / OpenTofu** | Immutable cloud topology, GPU instance management |
| **GitOps / CD** | **Argo CD / GitLab CI** | Git as single source of truth; automatic drift reconciliation |
| **Secrets Governance** | **Infisical / ESO** | Dynamic injection, rotation, audit; zero `.env` files in runtime |
| **Observability** | **OpenTelemetry + LGTM** | Distributed tracing, metrics, logs, visualization |

**Golden rule**: ALL topology is defined, reviewed, regression-tested, and deployed as declarative code.

### Observability Stack

| Concern | Tool |
|---------|------|
| Tracing + metrics + logs | **OpenTelemetry** (vendor-neutral, OTEL SDK) |
| Structured logging Python | **structlog** |
| Structured logging Node.js | **pino** |
| Metrics scraping | **Prometheus** |
| Error tracking | **Sentry** |
| Visualization | **Grafana** |
| Log aggregation | **Loki** |
| Trace backend | **Tempo** |
| Metrics backend | **Mimir** |

### OpenTelemetry Setup (FastAPI)

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.sqlalchemy import SQLAlchemyInstrumentor

provider = TracerProvider()
provider.add_span_processor(BatchSpanProcessor(OTLPSpanExporter()))
trace.set_tracer_provider(provider)

FastAPIInstrumentor.instrument_app(app)
SQLAlchemyInstrumentor().instrument(engine=engine)
```

### Structured Logging

```python
import structlog

log = structlog.get_logger()

# Context is carried through the request
log.info("user.created", user_id=str(user.id), email=user.email, role=user.role)
# Output: {"event": "user.created", "user_id": "...", "email": "...", "role": "user", "timestamp": "..."}
```

### Health Check Endpoints

```python
@app.get("/health/live")
async def liveness():
    """Kubernetes liveness probe — is the process alive?"""
    return {"status": "ok"}

@app.get("/health/ready")
async def readiness(db: AsyncSession = Depends(get_db), redis: Redis = Depends(get_redis)):
    """Kubernetes readiness probe — can it serve traffic?"""
    status = {"status": "ok"}
    try:
        await db.execute(text("SELECT 1"))
        status["db"] = "ok"
    except Exception:
        raise HTTPException(status_code=503, detail="database unavailable")
    try:
        await redis.ping()
        status["cache"] = "ok"
    except Exception:
        # Redis is a soft dependency — log but don't fail readiness
        status["cache"] = "degraded"
    return status
```

---

## 10. Decision Framework

Full architecture flowchart:

```
1. LANGUAGE
   ├── Python team?        → FastAPI (default) / Litestar (perf) / Django (content)
   ├── JS/TS team?         → Fastify (default) / Hono (edge) / Elysia (Bun)
   ├── Infra / binary?     → Go
   └── Max perf / cost?    → Rust (Axum)

2. TEAM SIZE
   ├── < 20 engineers      → Monolith or Modular Monolith
   └── > 20, org boundaries → Microservices (justify each split)

3. API CONSUMERS
   ├── Public / third-party → REST + OpenAPI
   ├── Own TS frontend only → tRPC
   ├── Multiple varied clients → GraphQL
   └── Internal services → gRPC

4. DATABASE
   ├── Default             → PostgreSQL
   ├── Edge / embedded     → SQLite + Litestream or Turso
   ├── Vector / AI search  → pgvector (existing PG) or Pinecone (new)
   ├── Local-first sync    → PowerSync or Turso
   ├── Variable schema     → MongoDB (justify)
   └── Time-series         → TimescaleDB

5. ORM
   ├── Python + FastAPI    → SQLModel (simple) / SQLAlchemy 2.0 (complex)
   ├── Node.js + TS        → Drizzle (SQL-like) / Prisma (schema-first)
   └── Complex queries     → Raw SQL for those queries

6. AUTH
   ├── Own users, SPA/mobile → JWT (access + refresh, EdDSA)
   ├── Own users, web app  → Sessions (Redis store)
   ├── Third-party login   → OAuth2 + PKCE
   ├── S2S / M2M           → API keys (hashed) or client_credentials
   └── Enterprise          → OIDC / SAML

7. BACKGROUND JOBS
   ├── Python              → Dramatiq (default) / ARQ (async)
   ├── Node.js             → BullMQ
   └── Long workflows / sagas → Temporal

8. CACHING
   ├── Default             → Redis (Cache-Aside pattern)
   ├── Invalidation        → TTL (simple) → event-driven (consistent)
   └── Connection pool     → Always; max_connections tuned to DB pool size

9. AI / AGENTS
   ├── Vector store (PG native) → pgvector
   ├── Vector store (managed)   → Pinecone
   ├── Agent workflows            → LangGraph
   ├── Multi-agent collaboration  → AutoGen
   └── Agent memory hierarchy     → Redis multi-tier

10. PLATFORM / OBSERVABILITY
    ├── IaC                 → Terraform / OpenTofu
    ├── GitOps              → Argo CD
    ├── Secrets             → Infisical / ESO
    └── Observability       → OpenTelemetry + structlog/pino + Sentry + /health/*
```

---

## Commands

```bash
# FastAPI project bootstrap
uv init my-api && cd my-api
uv add fastapi uvicorn[standard] sqlalchemy[asyncio] asyncpg pydantic-settings redis structlog

# Run with hot reload
uvicorn app.main:app --reload --port 8000

# Fastify project bootstrap
npm create fastify@latest my-api -- --lang=ts
npm install @fastify/swagger @fastify/swagger-ui @fastify/rate-limit

# Drizzle ORM setup
npm install drizzle-orm pg && npm install -D drizzle-kit @types/pg
npx drizzle-kit generate && npx drizzle-kit migrate

# Testcontainers (Python)
uv add --dev testcontainers pytest-asyncio factory-boy

# k6 load test
k6 run scripts/load-test.js

# OpenTelemetry auto-instrumentation (Python)
uv add opentelemetry-sdk opentelemetry-instrumentation-fastapi opentelemetry-instrumentation-sqlalchemy opentelemetry-exporter-otlp

# pgvector (PostgreSQL)
# In psql: CREATE EXTENSION vector;
# Via Docker: docker pull ankane/pgvector

# Temporal CLI
brew install temporal  # macOS
# or visit https://docs.temporal.io

# Terraform / OpenTofu
brew install terraform  # or opentofu
```

## Resources

- **FastAPI docs**: https://fastapi.tiangolo.com
- **SQLAlchemy 2.0**: https://docs.sqlalchemy.org/en/20/
- **Drizzle ORM**: https://orm.drizzle.team
- **Testcontainers Python**: https://testcontainers-python.readthedocs.io
- **OpenTelemetry Python**: https://opentelemetry-python.readthedocs.io
- **k6 docs**: https://grafana.com/docs/k6/latest/
- **Temporal**: https://docs.temporal.io
- **pgvector**: https://github.com/pgvector/pgvector
- **PowerSync**: https://www.powersync.com
- **LangGraph**: https://langchain-ai.github.io/langgraph/
- **AutoGen**: https://microsoft.github.io/autogen/
- **Infisical**: https://infisical.com
- **OpenTofu**: https://opentofu.org
- **Argo CD**: https://argo-cd.readthedocs.io
