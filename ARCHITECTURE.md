# Architecture Deep Dive (OrderQuick)

This document expands on the architectural decisions behind OrderQuick, focusing on **correctness**, **operability**, and **extensibility**. It describes the system in a way that is useful for engineering review without including proprietary code.

---

## System Components

- **React frontend**
  - Role-aware navigation (customer vs staff vs store manager vs admin)
  - Real-time UI updates via Socket.IO
  - API interactions over REST with JWT authentication

- **Node.js/Express backend**
  - Feature-based route modules (orders, payments, wallet, chat, stores/products, settings/campaigns)
  - Strong security middleware ordering (see [Security & Middleware](#security--middleware))
  - Background job scheduler co-located with the API runtime

- **MongoDB**
  - Document storage for orders and marketplace entities
  - Idempotency constraints and targeted indexes for high-volume queries
  - TTL expiration for chat retention policy

- **External payments (Monnify)**
  - Transaction initialization for order payments and wallet funding
  - HMAC signature verification on webhook delivery
  - Disbursement status queries for payout syncing

---

## Domain Model & Invariants

### Order Lifecycle State Machine

Orders are governed by a strict state machine:

- `SEARCHING`: order is in the dispatch pool
- `ACCEPTED`: a staff member has atomically claimed the order
- `PAID`: payment verified (Monnify webhook or wallet-only payment)
- `DELIVERED`: staff marked delivered
- `COMPLETED`: customer completion triggers settlement snapshot + escrow credits
- `CANCELLED`: allowed only in early stages to reduce edge-case complexity

**Why a strict state machine:**

- Prevents inconsistent reporting (“delivered but unpaid”, “completed without settlement”)
- Creates clear guardrails for financial operations (settlement only occurs once)
- Makes failure handling explicit (e.g., webhook may arrive late; only applies in expected states)

### Money Movement Model (Wallet, Escrow, Holds)

The platform uses three key balance concepts:

- **Wallet balance:** cleared funds spendable/withdrawable
- **Pending balance:** uncleared earnings (e.g., escrow cooling period)
- **Hold balance:** funds reserved for a pending withdrawal to prevent double spend

Entities that can hold balances include:

- Customers (`users`)
- Staff (walkers/bikers/store managers in `staff`)
- Stores (`stores`)
- Platform-level bookkeeping (platform balance doc)

**Why this model:**

- It matches real operational risk: you often need “earned but not yet withdrawable”.
- It supports dispute windows while staying transparent and auditable.

### Settlement Snapshot (“Freeze the Split”)

At order completion, the system computes and persists a settlement snapshot with:

- `staffAmount`
- `storeAmount`
- `companyAmount`
- `computedAt`

**Why persist the snapshot:**

- Prevents future business-rule changes from rewriting historical truth
- Makes audit and exports deterministic
- Allows downstream jobs (escrow release) to operate on stable amounts

---

## Real-Time Architecture (Socket.IO)

### Authentication

Socket connections are authenticated with JWT at connection time. This blocks unauthenticated clients from joining rooms or receiving events.

### Room Model

The system uses a hybrid of role-, user-, and order-based rooms:

- **Role rooms** for dispatch:
  - walkers vs bikers
  - store-manager hub for marketplace-related events
- **User rooms** for private updates:
  - payment verification
  - wallet balance updates
  - order status updates
- **Order rooms** for chat:
  - all participants in an order’s conversation are scoped to the order id

**Why rooms matter:**

- They reduce broadcast fanout and let you scale “who hears what” cleanly.
- They prevent data leakage across roles or orders.

---

## Background Processing (Agenda)

Agenda runs recurring reconciliation and “eventual consistency” tasks:

- **Escrow release polling**
  - releases credits when the cooling period is reached
  - uses claim-then-release guards to prevent double release

- **Withdrawal status sync**
  - periodically checks external disbursement status for processing withdrawals
  - settles internal holds when a withdrawal becomes PAID or FAILED

- **Penalty accrual**
  - scans for overdue orders still in active states and accrues penalties once

- **Referral reward backfill**
  - repairs rare partial-failure cases where a referee was marked complete but the idempotent reward record is missing

**Why recurring jobs instead of “only on request”:**

- Payment providers and networks are not reliable; scheduled reconciliation is required for correctness.
- It reduces operational load (fewer manual interventions) and improves user trust.

---

## Security & Middleware

Security hardening is implemented with multiple layers:

- **CORS allowlist with origin normalization**
  - trims trailing slashes / CRLF
  - allows local dev origins on any port in non-production environments

- **HTTP headers / browser hardening**
  - Helmet with a constrained CSP and cross-origin policies tuned for real deployments

- **Request sanitization and parameter pollution defense**
  - body/params sanitization to reduce injection-style attacks
  - `hpp` to prevent HTTP parameter pollution, with an explicit whitelist

- **Rate limiting**
  - global limiters for `/api` plus stricter controls for admin surfaces

- **Webhook integrity**
  - Monnify webhook verification is done via HMAC SHA-512 using the **raw request body** bytes (this is critical to avoid false signature mismatches)

---

## Design Principles Applied

- **Idempotency-first for money flows**
  - store per-order processed transaction references
  - enforce uniqueness on wallet-credit ledgers
  - create escrow credits with unique constraints per order/entity

- **Atomic “claim” operations to prevent race conditions**
  - order acceptance (`SEARCHING → ACCEPTED`)
  - escrow release claiming (only one worker can release an order)

- **“Minimal event payload” real-time updates**
  - sockets notify *that* something changed; clients fetch updated resources as needed
  - reduces data leakage risk and makes events resilient to schema evolution
