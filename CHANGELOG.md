# Changelog

All notable changes and deliverables for the **ZeTheta Payment Orchestration Platform** are documented in this file.

The project structure and daily deliverables strictly track the **15-Day Methodology** prescribed in **Part D** of `Zetheta_Backend_project.pdf`.

---

## [Unreleased] - Final Submission Readiness

### Added
- Created formal Architecture Decision Records:
  - `docs/adr/ADR-001-language-and-framework-selection.md` (Day 1–2 deliverable).
  - `docs/adr/ADR-002-gateway-simulation-and-failure-injection.md` (Day 5–6 deliverable).
- Added standardized specification aliases:
  - `docs/api-specification.yaml` (OpenAPI 3.0 REST contract for Day 11–12).
  - `docs/routing-algorithm.md` (Routing algorithm specification for Day 14).
- Added comprehensive `docs/test-coverage-report.md` proving >80% test coverage across all critical path modules.
- Created consolidated submission artifact `docs/consolidated-deliverables/Consolidated_MainReport.md` ready for compilation to `ProjectCode_YourName_MainReport.pdf`.

---

## [1.3.0] - 2026-11-14: Day 15 (Final Review, Part C Case Studies & Deliberate Errors)

### Added
- **Part C Case Study Suites (C1–C5):**
  - `C1`: High-concurrency double-charge prevention, exponential backoff jitter [750ms–1250ms], ambiguous timeout quarantine (`tests/case-studies/C1-double-charge-prevention.test.ts`).
  - `C2`: Missing settlement reconciliation detection, dispute event handling (`tests/case-studies/C2-settlement-reconciliation.test.ts`).
  - `C3`: Strict order fulfilment gating, blocking dispatch during `AUTH_INITIATED` and mandate expiration (`tests/case-studies/C3-upi-fulfilment-gate.test.ts`).
  - `C4`: Deep webhook semantic validation rejecting amount and currency mismatches despite valid HMAC signatures (`tests/case-studies/C4-webhook-replay-fraud.test.ts`).
  - `C5`: Database connection pool decoupling during external HTTP calls; fail-fast HTTP 503 (`tests/case-studies/C5-connection-pool-stampede.test.ts`).
- **Section C6 Deliberate Errors Bonus (+50 Points):**
  - Comprehensive documentation of all 5 embedded technical errors in `docs/errors-found.md`.
- **Part C Documentation:**
  - `docs/part-c-change-log.md`, `docs/part-c-coverage-matrix.md`, and `docs/part-c-test-report.md`.

---

## [1.2.0] - 2026-11-12: Day 13–14 (Dockerization, Benchmarking & Failure Scenarios)

### Added
- **Part B Failure Scenarios (`FS-01` through `FS-15`):**
  - End-to-end regression suites in `tests/part-b/scenarios/` validating all 15 business failure scenarios.
- **Performance Benchmarks & Load Testing:**
  - k6 load testing script `performance/k6/sustained-load.js` validating p95 latency < 80ms under 200 RPS.
  - Automated benchmark runner `scripts/part-b-benchmarks.ts` and integrity auditor `scripts/part-b-integrity.ts`.
  - Artifacts: `artifacts/part-b/benchmark-results.md` and `artifacts/part-b/integrity-audit.md`.
- **Production Containerization:**
  - Multi-stage `Dockerfile` (builder and non-root runner).
  - High-availability `docker-compose.yml` with PostgreSQL 15, Read Replica, PgBouncer, and asynchronous workers.
  - Comprehensive `README.md` with system overview, architecture diagram, and API quick reference.

---

## [1.1.0] - 2026-11-10: Day 9–12 (API Layer, Reconciliation & Idempotency)

### Added
- **REST API Layer (23 Endpoints):**
  - Full implementation of payment lifecycle endpoints (initiate, capture, void, refund, timeline).
  - Routing configuration, circuit breaker management, gateway metrics, and reconciliation APIs.
  - Interactive Swagger documentation served at `http://localhost:3000/docs` via `docs/openapi.yaml`.
  - Standard error handling format with unique trace IDs and RFC 7807 compliant error codes.
- **Idempotency & Concurrency Control:**
  - PostgreSQL advisory locking (`pg_advisory_xact_lock`) with deterministic SHA-256 payload hashing.
  - Auto-expiring idempotency keys with background cleanup.
- **Webhook Ingestion Pipeline:**
  - HMAC-SHA256 signature verification for Razorpay, Stripe, PayU, and UPI.
  - Asynchronous durable queue ingestion (`webhook_queue`) with worker leasing using `FOR UPDATE SKIP LOCKED`.
  - Dead Letter Queue (`DLQ`) for poison-pill events.
- **Reconciliation Engine:**
  - Scheduled background worker policing `AUTH_TIMEOUT` states and auto-recovering terminal states.
  - Anomaly detection table `reconciliation_anomalies` flagging missing settlements without financial leakage.

---

## [1.0.0] - 2026-11-06: Day 5–8 (Routing, Circuit Breakers & Gateway Adapters)

### Added
- **Intelligent Dynamic Routing Engine:**
  - 5-factor multi-criteria scoring algorithm with normalizations.
  - Automatic exclusion of OPEN circuit breakers.
  - 20% traffic preference bonus for recovering `HALF_OPEN` gateways.
  - Sub-2-second safe failover on verified transient HTTP 503 failures.
- **Gateway Adapter Layer:**
  - Uniform `PaymentGateway` interface.
  - Concrete mock adapters for Razorpay, Stripe, PayU, and Direct UPI.
  - Programmable simulation for 5 failure types: Success, Socket Timeout, 5xx Server Error, Card Decline, Rate Limit.
  - Asynchronous UPI collect flow simulation.
- **Circuit Breakers & Rate Limiters:**
  - Per-gateway circuit breaker tracking failure thresholds and 30-second cooldown transitions.
  - Token-bucket rate limiters preventing gateway throttling.

---

## [0.1.0] - 2026-11-02: Day 1–4 (Architecture, Relational Schema & State Machine)

### Added
- **Relational Database Architecture:**
  - Knex.js migrations defining 11 normalized tables with foreign keys and partial unique indexes.
  - PostgreSQL trigger `trg_state_log_immutable` physically preventing `UPDATE` and `DELETE` on `transaction_state_log`.
  - Strict integer minor currency units (`NUMERIC(16,0)`).
- **Transaction State Machine:**
  - Deterministic 15-state Finite State Machine (`TransactionStateMachine`).
  - Validation guards rejecting illegal state transitions with explicit error codes.
  - Comprehensive unit test suite covering valid and invalid state transitions.
- **Documentation:**
  - `docs/state-machine.md` with full Mermaid state diagrams.
  - `docs/architecture.md` initial draft.
  - `docs/data-model.md` and `docs/database-architecture.md`.

