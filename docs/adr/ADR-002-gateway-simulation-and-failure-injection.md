# ADR-002: Payment Gateway Simulation and Failure Injection Architecture

- **Status:** Accepted
- **Date:** 2026-11-05 (Day 5–6 Deliverable)
- **Author:** Back-End Architect (Manav Motiramani)
- **Deciders:** Engineering Architecture Team
- **Requirement Reference:** `[PDF-D, p.43]` (Day 5–6 Deliverables: Gateway Adapter Layer), `[PDF-A, p.8]` (Section A1: Payment Gateway Architecture Fundamentals)

---

## 1. Context & Problem Statement

To test and operate a multi-gateway payment orchestrator reliably across four target gateways (**Razorpay, Stripe, PayU, and Direct UPI**), the development and test harness must simulate both nominal and aberrant gateway behaviors.

Relying exclusively on live third-party staging/sandbox environments introduces severe operational friction:
1. External sandboxes cannot reliably simulate controllable socket timeouts (e.g. 30-second connection drops) on demand.
2. Rate limits, HTTP 502/503 outages, and downstream bank declines cannot be triggered deterministically during automated CI/CD pipeline runs.
3. Network jitter and third-party downtime cause non-deterministic test flakiness.
4. Security restrictions prevent simulating webhook replay attacks and signature corruption safely.

The orchestrator requires a mock gateway adapter layer capable of simulating real-world payment networks with fine-grained failure injection.

---

## 2. Options Considered

### Option A: External Sandbox Environments Only
- Connect directly to Razorpay Test Mode, Stripe Test API, and PayU Sandbox.
- **Pros:** Validates real HTTP headers and payload schemas against external vendor infrastructure.
- **Cons:** Extremely slow test execution; rate limits on free sandbox tiers; inability to force socket hangs or gateway-side database crashes on demand.

### Option B: HTTP Mock Server (e.g., WireMock / Prism)
- Run a standalone mock server container proxying gateway requests.
- **Pros:** Isolates the mock environment in a dedicated process; matches network transport layer.
- **Cons:** Additional container dependency; complex state coordination between the test runner and mock server when dynamically configuring failure scenarios mid-test.

### Option C: Uniform In-Process Gateway Adapter with Deterministic Failure Injection — SELECTED
- Implement a shared TypeScript interface `PaymentGateway` with concrete mock adapter implementations:
  - `RazorpayAdapter`
  - `StripeAdapter`
  - `PayUAdapter`
  - `UPIAdapter`
- Provide programmatic fault injection hooks (`failureMode`, `latencyMs`, `declineReason`).

---

## 3. Decision Outcome

**Chosen Approach: Option C (Uniform In-Process Gateway Adapter with Fault Injection Hooks)**

### 3.1 Architectural Structure
Every adapter implements the `PaymentGateway` contract:
```typescript
export interface PaymentGateway {
  readonly name: GatewayName;
  authorize(req: GatewayAuthRequest): Promise<GatewayAuthResponse>;
  capture(req: GatewayCaptureRequest): Promise<GatewayCaptureResponse>;
  void(req: GatewayVoidRequest): Promise<GatewayVoidResponse>;
  refund(req: GatewayRefundRequest): Promise<GatewayRefundResponse>;
  verifyWebhookSignature(rawBody: string, signature: string): boolean;
  setSimulationMode(config: GatewaySimulationConfig): void;
  resetSimulation(): void;
}
```

### 3.2 Five Prescribed Simulated Response Types
Each adapter implements exact simulations of the 5 response types required by Part D (Day 5–6):
1. **`SUCCESS`:** Returns authorization/capture success with generated gateway reference IDs (`pay_rzp_...`, `pi_str_...`, etc.).
2. **`TIMEOUT`:** Emulates upstream network hangs and socket drops (exceeding timeout thresholds) returning `TIMEOUT_UNKNOWN`.
3. **`HTTP_5XX_ERROR`:** Emulates gateway server failures (HTTP 502 Bad Gateway or HTTP 503 Service Unavailable) returning `TRANSIENT_FAILURE` where funds were definitively not held.
4. **`DECLINE`:** Emulates card/bank issuer decline (e.g. `INSUFFICIENT_FUNDS`, `EXPIRED_CARD`, `INVALID_PIN`) returning terminal `HARD_DECLINE`.
5. **`RATE_LIMIT`:** Emulates HTTP 429 Too Many Requests response with retry window.

### 3.3 Configurable Response Latency
Adapters support configurable millisecond delays via `setTimeout` simulation, enabling tests to verify system latency normalizations, circuit breaker timing, and timeout abort thresholds.

### 3.4 UPI Collect Flow Simulation
The `UPIAdapter` specifically simulates two-step asynchronous UPI collect authorization:
- Initial collect dispatch returns a pending status with an expiry window (e.g. 5 minutes).
- Asynchronous customer mandate approval or expiry is triggered via simulated webhook or reconciliation polling.

---

## 4. Consequences & Mitigations

- **Positive:**
  - 100% deterministic, reproducible automated testing across all 15 Part B scenarios and Part C case studies.
  - Zero external network dependencies, enabling completely offline and sandboxed CI execution.
  - Sub-second test execution across 99+ tests.
- **Negative / Trade-offs:**
  - Mock adapters could diverge from real gateway API schemas if external providers update their contracts.
- **Mitigation:**
  - Adapters are strictly modeled on published vendor documentation (Stripe Payment Intents v1, Razorpay Orders API, PayU Payments API, NPCI UPI 2.0 specs) and encapsulated behind a uniform internal domain contract.

