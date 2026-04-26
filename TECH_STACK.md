# Tech Stack (OrderQuick)

This document explains the major technology choices and the engineering trade-offs behind them.

---

## Frontend

- **React (Vite)**
  - Supports a multi-surface product (customer app + staff/admin dashboards) with shared UI patterns.
  - Fast iteration speed via Vite, which mattered under fixed timeline constraints.

- **React Router**
  - Enables clear role-based routing and guarded flows (customer vs staff vs admin surfaces).

- **Socket.IO client**
  - Real-time updates for: order dispatch, payment verification, wallet changes, and chat.
  - Reduces the need for heavy polling under bursty traffic.

- **Tailwind CSS**
  - Utility-first styling helps ship consistent UI quickly while keeping CSS maintenance manageable.

- **Firebase (client)**
  - Used for identity/notification adjacent capabilities in the frontend ecosystem (integrated with the rest of the stack).

---

## Backend

- **Node.js (ES modules) + Express**
  - Single runtime for HTTP API + sockets + background jobs.
  - Express gives explicit middleware ordering (important for webhook raw-body verification).

- **MongoDB + Mongoose**
  - Flexible schema for fast iteration as requirements evolve.
  - Document model fits orders (snapshotted customer info + line items + settlement snapshots).
  - Indexing is used intentionally for hot-path queries (status, customerId, SLA/escrow timestamps).

- **Socket.IO**
  - Room-based fanout for role dispatch and user/order scoped events.
  - JWT authentication at connection time to prevent unauthenticated listeners.

- **Agenda (Mongo-backed scheduler)**
  - Recurring jobs for escrow release, withdrawal status sync, penalty accrual, and referral backfills.
  - Avoids introducing separate infrastructure (e.g., Redis) while still supporting reliable periodic work.

---

## Payments & Fintech Layer

- **Monnify**
  - Transaction initialization for order payments and wallet funding.
  - Webhook delivery verified via HMAC SHA-512 on raw request bytes.
  - Disbursement status querying for “processing → paid/failed” sync.

- **Ledger-style auditability**
  - Wallet credits/debits are recorded in a dedicated collection with unique constraints to prevent double-credit on retries.
  - Escrow credits are recorded as one line per entity per order, with unique constraints to prevent double release.

---

## Security & Reliability Middleware

- **Helmet**: security headers + CSP tuning for a real deployed web app
- **CORS allowlist**: origin normalization and safe dev behavior
- **express-rate-limit**: global + admin-surface rate limiting
- **express-mongo-sanitize**: request sanitization (body/params) for injection risk reduction
- **hpp**: HTTP Parameter Pollution defense
- **JWT (jsonwebtoken)**: stateless auth for both HTTP and sockets

---

## Media & Notifications

- **Cloudinary + Multer**
  - Image uploads for store/product assets with public id tracking to support cascading deletes and cleanup.

- **Resend (email)**
  - Transactional email channel for system notifications where needed.

- **Firebase Admin**
  - Server-side integration capability for Firebase-related backend flows (auth/messaging infrastructure).

---

## Key Trade-offs (and why they were acceptable)

- **MongoDB vs relational**
  - Trade-off: fewer relational guarantees.
  - Mitigation: explicit idempotency fields, unique constraints, and snapshotting of settlement computations.

- **Agenda vs dedicated queue**
  - Trade-off: fewer “at-least-once” semantics knobs than a full queue stack.
  - Mitigation: job logic is idempotent and uses claim-then-apply patterns.

- **Sockets vs polling**
  - Trade-off: connection management and auth complexity.
  - Mitigation: room-based fanout, JWT-gated sockets, and minimal payload event design.
