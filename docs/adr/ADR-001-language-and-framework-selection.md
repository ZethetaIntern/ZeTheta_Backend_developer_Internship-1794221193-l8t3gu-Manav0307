# ADR-001: Selection of Language, Framework, and Database Stack

- **Status:** Accepted
- **Date:** 2026-11-01 (Day 1–2 Deliverable)
- **Author:** Back-End Architect (Manav Motiramani)
- **Deciders:** Engineering Architecture Team
- **Requirement Reference:** `[PDF-D, p.42]` (Day 1–2 Deliverables), `[PDF-E, p.47]` (Section E1: Approved Technology Stacks)

---

## 1. Context & Problem Statement

PayFlow Commerce processes over 100,000 transactions daily across multiple downstream payment gateways (Razorpay, Stripe, PayU, Direct UPI). The payment orchestration layer must:
1. Handle highly concurrent, I/O-bound network calls to external payment gateway endpoints without blocking execution threads.
2. Maintain strict financial arithmetic integrity, eliminating floating-point errors and ledger discrepancies.
3. Provide compile-time type safety across complex multi-state transaction lifecycles.
4. Ingest and verify high-throughput asynchronous webhooks with sub-millisecond deduplication.
5. Interface with a relational transactional datastore enforcing strict ACID guarantees, advisory locking, and immutable audit logs.

Section E1 of `Zetheta_Backend_project.pdf` provides three approved technology stacks:
- **Option 1 (Python):** FastAPI + SQLAlchemy / Alembic + PostgreSQL 15+ + pytest
- **Option 2 (Node.js/TypeScript):** Express.js / Fastify + Prisma / Knex.js + PostgreSQL 15+ + Jest / Supertest
- **Option 3 (Java/Kotlin):** Spring Boot + JPA / Hibernate / Flyway + PostgreSQL 15+ + JUnit 5

---

## 2. Options Considered

### Option 1: Python (FastAPI + SQLAlchemy + PostgreSQL)
- **Pros:** Native async support with `asyncio`, concise syntax, automatic OpenAPI generation with Pydantic.
- **Cons:** Python's Global Interpreter Lock (GIL) and single-threaded CPU runtime present scaling bottlenecks under high concurrent hashing/HMAC-SHA256 calculations; dynamic typing requires runtime validation overhead; ORM impedance mismatch with complex partial indexes and custom PostgreSQL triggers.

### Option 2: Node.js / TypeScript (Express.js + Knex.js + PostgreSQL) — SELECTED
- **Pros:**
  - **Asynchronous Event Loop:** Native non-blocking I/O ideally suited for network-heavy payment gateway HTTP orchestration, webhook ingestion, and connection pooling.
  - **Static Type Safety:** TypeScript ensures complete type checking across 15 finite state machine states, gateway payload contracts, and error structures.
  - **Knex.js SQL Query Builder:** Provides direct, transparent control over raw PostgreSQL queries, transaction isolation levels, row-level locks (`SELECT ... FOR UPDATE SKIP LOCKED`), and custom triggers (`trg_state_log_immutable`) without the black-box abstraction penalties of heavy ORMs.
  - **Native BigInt Support:** JavaScript/TypeScript provides native `BigInt` for minor currency units (`amount_minor`), preventing precision loss.
  - **Extensive Ecosystem:** First-class library support for cryptographic primitives (Node.js native `crypto`), Swagger documentation (`swagger-ui-express`), and testing (`vitest` / `supertest`).
- **Cons:** Requires deliberate architectural discipline to avoid callback/promise unhandled rejections; single-threaded event loop requires worker offloading for heavy batch tasks.

### Option 3: Java / Kotlin (Spring Boot + JPA / Hibernate + PostgreSQL)
- **Pros:** Robust enterprise tooling, multi-threading support, comprehensive transaction management (`@Transactional`).
- **Cons:** Significantly higher memory footprint and slower startup times; excessive boilerplate configuration; ORM caching layers (Hibernate Level 1/2 cache) can inadvertently hide concurrent row-level locks and trigger-generated state mutations.

---

## 3. Decision Outcome

**Chosen Stack: Option 2 (Node.js / TypeScript + Express.js + Knex.js + PostgreSQL 15+)**

### Key Justifications:
1. **Asynchronous Non-Blocking I/O:** Payment orchestration is fundamentally an I/O multiplexer (waiting on external gateway HTTP sockets, Postgres transactional locks, and webhook deliveries). Node.js's `libuv` event loop handles thousands of concurrent pending network requests with minimal memory overhead.
2. **Knex.js Query Precision:** Financial orchestration requires fine-grained control over PostgreSQL features:
   - Partial unique indexes (`idx_single_active_attempt_per_tx`).
   - Transactional advisory locks (`pg_advisory_xact_lock`).
   - Worker queues with lock jumping (`FOR UPDATE SKIP LOCKED`).
   Knex gives transparent SQL control while managing connection pools and migration versioning cleanly.
3. **TypeScript Static Contracts:** Eliminates runtime type errors across internal gateway adapters and state machine transitions through discriminated unions and strict compiler options.
4. **PostgreSQL 15+ Integration:** Enforces immutable financial ledgers via database-level triggers and ACID transactional guarantees.

---

## 4. Consequences & Mitigations

- **Positive:**
  - High webhook ingestion throughput ($> 5,000$ RPS per instance).
  - Strict compile-time checks on financial minor units and state transitions.
  - Clean migration and seeding lifecycle via Knex CLI scripts.
  - Fast test suite execution ($< 5$ seconds for unit and integration suites).
- **Negative / Trade-offs:**
  - CPU-intensive tasks (e.g. batch cryptographic hashing or heavy reconciliation runs) can momentarily monopolize the event loop if not decoupled.
- **Mitigation:**
  - Asynchronous background worker processes (`src/workers/webhook.worker.ts` and `src/workers/reconciliation.worker.ts`) run independently from the user-facing REST API server, backed by database row locking.

