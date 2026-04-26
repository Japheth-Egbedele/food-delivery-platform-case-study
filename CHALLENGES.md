# Technical Challenges & Solutions (OrderQuick)

This document highlights the most consequential engineering challenges in OrderQuick and the patterns used to solve them, focusing on the “how” and “why” rather than proprietary code.

---

## 1) Payment Webhook Reliability (Monnify)

### The risk

Payment providers retry webhooks, networks fail mid-request, and “paid” notifications can be late or duplicated. In a delivery platform, the worst failure modes are:

- **double-crediting wallets**
- **double-marking an order as paid**
- **accepting “paid” for the wrong amount**

### What I implemented

- **Authenticity verification**
  - Monnify webhook requests are verified via **HMAC SHA-512** using the provider’s secret key.
  - The signature is computed from the **raw request body bytes**, not a re-stringified JSON object (re-serialization can change byte representation and break signatures).

- **Idempotency & exactly-once effects**
  - **Order payment webhooks:** each order tracks already-applied transaction references, preventing duplicates from applying twice.
  - **Wallet funding webhooks:** wallet credits write through a dedicated ledger collection with a **unique index** on `(paymentReference, source, direction)` to prevent double credits.

- **Correctness checks**
  - Split-pay orders validate the **expected Monnify portion** vs total order value and wallet-applied amount, including a small tolerance for rounding in NGN.

Why this matters: it makes the system safe under the most common real-world provider behaviors (duplicates, retries, partial failures).

---

## 2) Split-Pay (Wallet + Monnify remainder) Without Financial Drift

### The risk

Split-pay is easy to get wrong because it is a “two-system transaction”:

- internal wallet debit must occur
- external payment initialization must succeed
- webhook must later confirm payment

If Monnify init fails after a wallet debit, users can be left with missing funds and “unpaid” orders.

### What I implemented

- **Atomic wallet debits + ledger writes**
  - Wallet debits are performed atomically with writing a ledger entry so the balance change is always explainable.

- **Compensation on external failure**
  - If external payment initialization fails (or network errors occur), the wallet debit is rolled back using a compensating credit + ledger entry.

- **Payment attempt resumability**
  - A time window allows the frontend to reuse an in-flight payment reference (reducing duplicate attempts and “dangling” payment refs).

Why this matters: it prevents a high-severity class of support incidents (“I paid but the system didn’t record it” / “my wallet was debited but payment failed”).

---

## 3) Race Conditions in Order Acceptance (“Two agents clicked accept”)

### The risk

During meal-time spikes, multiple agents can accept the same order within milliseconds.

### What I implemented

- **Atomic claim pattern**
  - Accepting an order is a conditional update that only succeeds if the order is still in `SEARCHING`.
  - The first successful writer wins; others receive a deterministic “already picked up” response.

- **Dispatch partitioning**
  - Orders are fanned out to the correct staff cohort (walkers vs bikers) based on campus type, reducing unnecessary contention.

Why this matters: it produces stable, predictable assignment behavior under high concurrency.

---

## 4) Escrow Cooling Period + Safe Release

### The risk

Paying immediately causes disputes; delaying release introduces background-processing complexity and the possibility of double-release.

### What I implemented

- **Completion-time escrow scheduling**
  - On completion, the system computes a release time (cooling hours) and persists it on the order.

- **Escrow credits as a ledger**
  - A separate escrow ledger stores one line per entity per order (staff/store/platform), enforced by a unique constraint.

- **Idempotent release with claim-then-apply**
  - Orders are “claimed” for release via a guarded update on the escrow-release marker.
  - Release updates move money from `pendingBalance` into `walletBalance` exactly once.

Why this matters: it allows multiple instances/workers to run safely without duplication.

---

## 5) SLA Timers + Late Penalty System

### The risk

If penalties are applied inconsistently (or multiple times), staff trust collapses and finance becomes un-auditable.

### What I implemented

- **Penalty state machine**
  - Penalties move through a strict state machine (`NONE → ACCRUED → APPLIED`) with guarded transitions.

- **Configurable penalty rules**
  - Grace periods and per-minute penalty values are configurable via environment/runtime configuration, avoiding redeploys for policy adjustments.

- **Admin “revert penalty” control**
  - A safe revert path exists for edge cases, implemented as an idempotent action with audit logging.

Why this matters: it enforces operational discipline while still giving administrators controlled escape hatches.

---

## 6) Real-Time UX Under Intermittent Mobile Networks

### The risk

Mobile browsers frequently disconnect; aggressive polling is expensive and stale.

### What I implemented

- **JWT-authenticated Socket.IO**
  - Sockets authenticate at connection time; unauthorized clients can’t join rooms.

- **Room-based fanout**
  - Role rooms for dispatch, user rooms for private events, order rooms for chat.

- **Minimal payload events**
  - Events notify clients that something changed; clients fetch updated resources when needed.

Why this matters: it improves user experience without turning real-time into a data-leak or scalability risk.

---

## 7) Scalable Admin Export Without Taking the API Down

### The risk

Wide exports (orders + users + staff + stores + referrals) can cause slow queries and high memory usage.

### What I implemented

- **Row caps** to prevent unbounded memory growth.
- **Batch fetching** of related entities to avoid N+1 query patterns.
- **CSV escaping** and BOM handling for compatibility with spreadsheet tooling.

Why this matters: it keeps reporting available without degrading the customer experience.

