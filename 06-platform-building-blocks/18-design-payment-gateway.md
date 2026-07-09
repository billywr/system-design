# Design a Payment Gateway

> **Hello Interview Framework** — A Big Tech–level system design guide for building a production payment processing platform like Stripe, Adyen, PayPal, or Square's payment infrastructure.

---

## Table of Contents

1. [Problem Statement](#1-problem-statement)
2. [Requirements Clarification](#2-requirements-clarification)
3. [Capacity Estimation](#3-capacity-estimation)
4. [API Design](#4-api-design)
5. [Data Model](#5-data-model)
6. [High-Level Architecture](#6-high-level-architecture)
7. [Deep Dive: Idempotency](#7-deep-dive-idempotency)
8. [Deep Dive: Double-Spend Prevention](#8-deep-dive-double-spend-prevention)
9. [Deep Dive: PCI Compliance & Security](#9-deep-dive-pci-compliance--security)
10. [Deep Dive: Reconciliation](#10-deep-dive-reconciliation)
11. [Scaling & Reliability](#11-scaling--reliability)
12. [Failure Modes & Edge Cases](#12-failure-modes--edge-cases)
13. [Trade-offs Summary](#13-trade-offs-summary)
14. [Interview Walkthrough Script](#14-interview-walkthrough-script)
15. [Follow-Up Questions](#15-follow-up-questions)
16. [Real-World References](#16-real-world-references)

---

## 1. Problem Statement

Design a payment gateway that processes credit card and bank payments for merchants. The system must authorize, capture, and settle transactions; handle refunds and chargebacks; guarantee idempotent processing; prevent double-spending; maintain PCI compliance; and reconcile financial records with external payment processors and banks.

**What the interviewer is really testing:**

- Idempotency and exactly-once payment semantics
- Distributed transaction handling without 2PC
- PCI scope minimization (tokenization)
- Ledger-based accounting and reconciliation
- Failure handling (timeouts, partial failures, async webhooks)

---

## 2. Requirements Clarification

### Clarifying Questions to Ask

| Question | Why It Matters |
|----------|----------------|
| Auth-only or auth + capture? | Two-phase payment flow |
| Supported methods? Cards, ACH, wallets? | Integration complexity |
| Multi-currency? | FX and settlement rails |
| Merchant model? Marketplace with splits? | Ledger complexity |
| Refunds and partial refunds? | State machine expansion |
| Real-time or batch settlement? | Reconciliation timing |

### Functional Requirements

**Must Have (P0):**

- Process payment: authorize + capture funds from customer
- Idempotent payment API (retry-safe)
- Prevent double-charging same order
- Refund full or partial amount
- Webhook notifications to merchant on status change
- Transaction status query

**Should Have (P1):**

- Support multiple payment methods (Visa, MC, ACH)
- 3D Secure / SCA for EU compliance
- Merchant dashboard for transaction history
- Dispute / chargeback handling workflow
- Payment method tokenization (store card for reuse)

**Nice to Have (P2):**

- Marketplace split payments (Stripe Connect model)
- Subscription / recurring billing
- Fraud scoring integration
- Multi-currency with FX

### Non-Functional Requirements

| Dimension | Target | Rationale |
|-----------|--------|-----------|
| **Authorization latency (p99)** | < 3 sec | User waiting at checkout |
| **Availability** | 99.99% | Payment downtime = revenue loss |
| **Durability** | Zero lost transactions | Financial correctness |
| **Consistency** | Strong for ledger | Money must balance |
| **Scale** | 10K TPS peak | Large e-commerce events |
| **Compliance** | PCI DSS Level 1 | Legal requirement |

```mermaid
graph TB
    subgraph Core Guarantees
        I[Idempotency]
        D[Double-Spend Prevention]
        P[PCI Compliance]
        R[Reconciliation]
    end
    Payment[Payment Gateway] --> I
    Payment --> D
    Payment --> P
    Payment --> R
```

---

## 3. Capacity Estimation

Assume **50M transactions/day**, average **$45/transaction**, peak **10K TPS** (Black Friday).

### Transaction Volume

```
Daily TPS average: 50M / 86400 ≈ 580 TPS
Peak TPS:          10,000 (17× average)
Daily GMV:         50M × $45 = $2.25B/day
Annual GMV:        ~$820B (large platform scale)
```

### Storage

```
Transaction record: ~2 KB (metadata, audit trail, PCI-safe tokens)
Daily storage: 50M × 2 KB = 100 GB/day
7-year retention (regulatory): 100 GB × 365 × 7 ≈ 255 TB
Ledger entries (3 per txn): 150M entries/day × 500 B ≈ 75 GB/day
```

### External Processor Calls

```
1 authorization + 1 capture per txn = 2 processor calls
Peak: 10K × 2 = 20K external API calls/sec
Processor latency: 500ms–2s → need 20K × 1s = 20K concurrent connections
Connection pool + async processing essential
```

```mermaid
pie title Transaction Types
    "Card Capture" : 70
    "Wallet (Apple Pay)" : 15
    "ACH/Bank" : 10
    "Refunds" : 5
```

---

## 4. API Design

### Create Payment (Idempotent)

```http
POST /v1/payments
Authorization: Bearer sk_live_merchant_key
Idempotency-Key: order-abc123-payment
Content-Type: application/json

{
  "amount": 4999,
  "currency": "usd",
  "payment_method": "pm_tokenized_card_xyz",
  "merchant_id": "merch_123",
  "order_id": "order_abc123",
  "capture": true,
  "metadata": { "customer_id": "cust_456" }
}

Response 201:
{
  "payment_id": "pay_789",
  "status": "succeeded",
  "amount": 4999,
  "currency": "usd",
  "processor_ref": "ch_stripe_xyz",
  "created_at": "2026-07-08T12:00:00Z"
}
```

### Payment Status Polling

```http
GET /v1/payments/pay_789

Response 200:
{
  "payment_id": "pay_789",
  "status": "succeeded",
  "amount": 4999,
  "refunds": [],
  "timeline": [
    {"status": "pending", "at": "2026-07-08T12:00:00.000Z"},
    {"status": "authorized", "at": "2026-07-08T12:00:00.500Z"},
    {"status": "captured", "at": "2026-07-08T12:00:01.200Z"},
    {"status": "succeeded", "at": "2026-07-08T12:00:01.200Z"}
  ]
}
```

### Refund

```http
POST /v1/payments/pay_789/refunds
Idempotency-Key: refund-order-abc123

{
  "amount": 2500,
  "reason": "partial_return"
}
```

### Merchant Webhook

```http
POST https://merchant.com/webhooks/payments
Stripe-Signature: t=1720430400,v1=abc123...

{
  "event_type": "payment.succeeded",
  "payment_id": "pay_789",
  "amount": 4999,
  "order_id": "order_abc123"
}
```

---

## 5. Data Model

### Payment State Machine

```mermaid
stateDiagram-v2
    [*] --> Pending: POST /payments
    Pending --> Authorized: processor auth OK
    Pending --> Failed: processor auth declined
    Authorized --> Captured: capture call
    Authorized --> Voided: void before capture
    Captured --> Succeeded: settlement queued
    Captured --> Refunded: full refund
    Captured --> PartiallyRefunded: partial refund
    Succeeded --> Disputed: chargeback filed
    Failed --> [*]
    Voided --> [*]
    Refunded --> [*]
    Succeeded --> [*]
```

### Entity Relationship

```mermaid
erDiagram
    MERCHANT ||--o{ PAYMENT : receives
    PAYMENT ||--o{ LEDGER_ENTRY : generates
    PAYMENT ||--o{ REFUND : may_have
    PAYMENT ||--|| IDEMPOTENCY_RECORD : deduped_by
    CUSTOMER ||--o{ PAYMENT_METHOD : owns
    PAYMENT }o--|| PAYMENT_METHOD : uses

    PAYMENT {
        uuid payment_id PK
        uuid merchant_id FK
        uuid idempotency_key UK
        int amount_cents
        string currency
        string status
        string processor_ref
        timestamp created_at
    }
    LEDGER_ENTRY {
        uuid entry_id PK
        uuid payment_id FK
        string account
        int debit_cents
        int credit_cents
        timestamp posted_at
    }
    IDEMPOTENCY_RECORD {
        string idempotency_key PK
        uuid merchant_id
        uuid payment_id
        json response_body
        timestamp expires_at
    }
```

### Double-Entry Ledger

Every payment creates balanced ledger entries:

```
Payment of $49.99 captured:

  DEBIT   customer_funds      $49.99
  CREDIT  merchant_pending    $47.99  (after 2% platform fee)
  CREDIT  platform_revenue     $2.00
```

```mermaid
flowchart LR
    subgraph Ledger Accounts
        CF[Customer Funds]
        MP[Merchant Pending]
        PR[Platform Revenue]
        PP[Processor Clearing]
    end
    CF -->|debit 4999| PP
    PP -->|credit 4799| MP
    PP -->|credit 200| PR
```

---

## 6. High-Level Architecture

```mermaid
flowchart TB
    subgraph Merchant
        MWeb[Merchant Checkout]
        MBackend[Merchant Backend]
    end

    subgraph Payment Gateway
        API[Payment API]
        Idem[Idempotency Service]
        Router[Payment Router]
        Orchestrator[Payment Orchestrator]
        Ledger[Ledger Service]
        Webhook[Webhook Dispatcher]
        Recon[Reconciliation Engine]
    end

    subgraph Security
        Vault[Token Vault / HSM]
        Fraud[Fraud Scoring]
    end

    subgraph Processors
        Visa[Visa Network]
        StripeP[Stripe / Adyen]
        ACH[ACH Network]
    end

    subgraph Storage
        PG[(PostgreSQL - Ledger)]
        Redis[(Redis - Idempotency)]
        S3[(S3 - Audit Logs)]
    end

    MWeb --> MBackend
    MBackend --> API
    API --> Idem
    API --> Fraud
    API --> Orchestrator
    Orchestrator --> Router
    Router --> StripeP
    Router --> Visa
    Router --> ACH
    Orchestrator --> Ledger
    Ledger --> PG
    Idem --> Redis
    Orchestrator --> Webhook
    Recon --> PG
    Recon --> StripeP
    API --> Vault
```

### Payment Flow

```mermaid
sequenceDiagram
    participant M as Merchant
    participant API as Payment API
    participant Idem as Idempotency
    participant Orch as Orchestrator
    participant Fraud as Fraud Service
    participant Proc as Processor (Stripe)
    participant Ledger as Ledger Service
    participant WH as Webhook

    M->>API: POST /payments (Idempotency-Key)
    API->>Idem: Check key
    Idem-->>API: New request
    API->>Orch: Process payment
    Orch->>Fraud: Score transaction
    Fraud-->>Orch: risk=low
    Orch->>Proc: Authorize + Capture
    Proc-->>Orch: approved, ch_xyz
    Orch->>Ledger: Post double-entry
    Ledger-->>Orch: OK
    Orch->>Idem: Store result
    Orch->>WH: Enqueue webhook
    API-->>M: 201 payment.succeeded
    WH->>M: POST webhook (async)
```

---

## 7. Deep Dive: Idempotency

### Why Idempotency Is Non-Negotiable

```
Scenario: Merchant POSTs payment → network timeout → retries
Without idempotency: customer charged twice
With idempotency: second request returns same payment_id
```

### Idempotency Key Design

```
Key scope: (merchant_id, idempotency_key) → unique
Key source: merchant-generated (order_id + attempt) or UUID
TTL: 24 hours minimum (Stripe uses 24h)
Storage: Redis (fast) + PostgreSQL (durable audit)
```

```mermaid
sequenceDiagram
    participant M as Merchant
    participant API as Payment API
    participant Redis as Redis
    participant PG as PostgreSQL

    M->>API: POST Idempotency-Key: order-123
    API->>Redis: SETNX idem:merch:order-123
    alt Key acquired (first request)
        Redis-->>API: OK
        API->>API: Process payment
        API->>PG: INSERT payment + idempotency record
        API->>Redis: SET idem:merch:order-123 → response (TTL 24h)
        API-->>M: 201 pay_789
    else Key exists (duplicate)
        Redis-->>API: existing response
        API-->>M: 201 pay_789 (same response, no re-processing)
    end
```

### Idempotency State Machine

```mermaid
stateDiagram-v2
    [*] --> Locked: SETNX acquired
    Locked --> Processing: payment in flight
    Processing --> Completed: store response
    Processing --> Failed: store error response
    Completed --> [*]: return cached response
    Failed --> [*]: return cached error
    Locked --> [*]: concurrent duplicate waits
```

### Concurrent Duplicate Handling

```
Request A: SETNX → success → processing
Request B: SETNX → fail → poll/wait for A's result (max 30s)
Request B: GET idem:merch:order-123 → return A's response
```

**Never re-process** if idempotency key exists, even if first attempt appeared to fail (may have succeeded at processor).

### Idempotency + Processor

```
Always pass idempotency key to processor API:
  Stripe: Idempotency-Key header → same charge object
  Adyen: Idempotency-Key → same pspReference

Two layers: gateway idempotency + processor idempotency
```

---

## 8. Deep Dive: Double-Spend Prevention

### Threat Model

| Attack / Bug | Scenario | Prevention |
|--------------|----------|------------|
| Network retry | Duplicate POST | Idempotency key |
| Race condition | Two tabs checkout simultaneously | Order-level lock |
| Partial failure | Auth succeeds, capture fails, retry captures twice | State machine + idempotency |
| Refund abuse | Refund more than captured | Balance check in ledger |
| Replay attack | Resubmit old payment request | Timestamp + nonce validation |

### Order-Level Payment Lock

```mermaid
sequenceDiagram
    participant M as Merchant
    participant API as Payment API
    participant Lock as Distributed Lock
    participant PG as Database

    M->>API: Pay order_abc123
    API->>Lock: ACQUIRE lock:order:abc123
    alt lock acquired
        API->>PG: SELECT status FROM orders WHERE id=abc123
        PG-->>API: status=pending_payment
        API->>API: Process payment
        API->>PG: UPDATE status=paid
        API->>Lock: RELEASE
    else lock held
        API-->>M: 409 Payment already in progress
    end
```

### Database Constraints

```sql
-- One successful payment per order
CREATE UNIQUE INDEX idx_one_payment_per_order
  ON payments (merchant_id, order_id)
  WHERE status IN ('succeeded', 'captured');

-- Ledger must balance (application-enforced + audit)
-- Sum of debits = Sum of credits for each payment_id
```

### Authorization vs Capture (Two-Phase)

```mermaid
sequenceDiagram
    participant O as Orchestrator
    participant P as Processor
    participant L as Ledger

    O->>P: Authorize $49.99
    P-->>O: auth_ref (hold placed)
    O->>L: Record auth hold
    Note over O: Order fulfilled...
    O->>P: Capture auth_ref $49.99
    P-->>O: captured
    O->>L: Post final ledger entries
```

**Why two-phase:**

- Hotel holds $200, captures $150 at checkout
- E-commerce: auth at order, capture at ship
- Prevents charging before goods confirmed

### Timeout Handling

```
Auth placed → capture attempt → timeout (unknown result)
NEVER blindly retry capture
Instead:
  1. Query processor for auth status
  2. If captured → update local state
  3. If not captured → safe to retry capture with same idempotency key
```

---

## 9. Deep Dive: PCI Compliance & Security

### PCI DSS Scope Reduction

**Goal:** Never touch raw card numbers (PAN) in payment gateway application layer.

```mermaid
flowchart TB
    subgraph PCI Scope ["PCI Scope (Minimal)"]
        Vault[Token Vault]
        HSM[HSM]
    end

    subgraph Out of Scope
        API[Payment API]
        Merchant[Merchant App]
        DB[(Application DB)]
    end

    Browser[Customer Browser] -->|card data| Vault
    Vault -->|token: pm_xyz| API
    API --> DB
    Merchant --> API
```

### Tokenization Flow

```mermaid
sequenceDiagram
    participant C as Customer Browser
    participant IF as Hosted iFrame (Vault)
    participant V as Token Vault
    participant M as Merchant Backend
    participant PG as Payment Gateway

    C->>IF: Enter card (never touches merchant server)
    IF->>V: Encrypt PAN via HSM
    V-->>IF: pm_token_abc
    IF-->>C: token
    C->>M: pm_token_abc (not PAN)
    M->>PG: POST /payments { payment_method: pm_token_abc }
    PG->>V: Detokenize for processor call (inside PCI boundary)
```

### PCI Compliance Levels

| Level | Criteria | Requirements |
|-------|----------|--------------|
| Level 1 | > 6M txn/year | Full audit, QSA assessment |
| Level 2 | 1M–6M txn/year | SAQ-D, quarterly scan |
| Level 3 | 20K–1M e-commerce | SAQ-A (if fully outsourced) |
| Level 4 | < 20K e-commerce | SAQ-A |

**Architecture target:** SAQ-A for merchants (hosted fields); Level 1 for gateway itself.

### Security Controls

```mermaid
flowchart TD
    subgraph Security Layers
        TLS[TLS 1.3 Everywhere]
        Token[Tokenization]
        HSM[HSM Key Storage]
        Audit[Immutable Audit Log]
        RBAC[Role-Based Access]
        Fraud[Fraud Detection]
        Encrypt[Encryption at Rest AES-256]
    end
```

| Control | Implementation |
|---------|----------------|
| PAN storage | Never store; token only |
| CVV | Never store (PCI rule) |
| Key management | HSM (AWS CloudHSM, Thales) |
| Audit trail | Append-only log to S3 + WORM |
| Access control | Least privilege; break-glass for prod |
| Vulnerability scan | Quarterly ASV scan |
| 3D Secure / SCA | Required for EU PSD2 |

### Webhook Security

```
Verify webhook signature:
  expected = HMAC-SHA256(webhook_secret, timestamp + body)
  Reject if timestamp > 5 min old (replay protection)
  Idempotent webhook processing (event_id dedup)
```

---

## 10. Deep Dive: Reconciliation

### Why Reconciliation Matters

```
Internal ledger says: $2.25B processed today
Processor settlement file says: $2.2498B
Difference: $200K → investigate (failed captures, FX, chargebacks)
```

### Reconciliation Pipeline

```mermaid
flowchart TB
    subgraph Daily Batch
        SF[Processor Settlement File]
        Ingest[File Ingestion S3]
        Parser[CSV/ISO8583 Parser]
        Match[Matching Engine]
        Internal[(Internal Ledger DB)]
        Report[Discrepancy Report]
    end

    SF --> Ingest --> Parser --> Match
    Internal --> Match
    Match -->|matched| Settled[Mark Settled]
    Match -->|mismatch| Report
    Report --> Ops[Ops Dashboard]
```

### Three-Way Match

```mermaid
flowchart LR
    A[Payment Record] --> Match{Three-Way Match}
    B[Processor Confirmation] --> Match
    C[Bank Settlement] --> Match
    Match -->|all agree| OK[Reconciled ✓]
    Match -->|disagree| Investigate[Exception Queue]
```

| Record | Source | Timing |
|--------|--------|--------|
| Authorization | Real-time API | T+0 |
| Capture | Real-time API | T+0 to T+7 days |
| Settlement | Processor batch file | T+1 to T+3 |
| Bank deposit | Bank statement | T+2 to T+5 |

### Matching Logic

```python
def reconcile(processor_txn, internal_txn):
    if processor_txn.ref == internal_txn.processor_ref:
        if processor_txn.amount == internal_txn.amount:
            return MATCH
        return AMOUNT_MISMATCH
    return MISSING_MATCH
```

### Exception Handling

| Exception | Action |
|-----------|--------|
| Missing in internal | Processor-only txn → investigate fraud |
| Missing in processor | Internal-only → retry processor query |
| Amount mismatch | Halt merchant payout; manual review |
| Duplicate settlement | Idempotent settlement posting |
| Chargeback | Reverse ledger entries; notify merchant |

### Settlement & Payout

```mermaid
sequenceDiagram
    participant PG as Payment Gateway
    participant Ledger as Ledger
    participant Bank as Bank / Processor
    participant M as Merchant Bank

    Note over PG,M: T+0: Payment captured
    PG->>Ledger: Credit merchant_pending $47.99

    Note over PG,M: T+2: Processor settlement file
    PG->>Ledger: Move merchant_pending → merchant_available
    PG->>Bank: Initiate payout batch
    Bank->>M: ACH transfer $47.99
    PG->>Ledger: Debit merchant_available
```

---

## 11. Scaling & Reliability

### Database Strategy for Ledger

```
PostgreSQL with:
  - SERIALIZABLE isolation for ledger writes
  - Partition by date (daily partitions)
  - Read replicas for reporting/reconciliation
  - Never delete — append-only ledger entries
```

```mermaid
flowchart TB
    subgraph Write Path
        Primary[(PG Primary)]
    end
    subgraph Read Path
        R1[(Replica - Reporting)]
        R2[(Replica - API reads)]
    end
    Primary --> R1
    Primary --> R2
```

### Async Processing

```
Synchronous: Auth decision (< 3 sec user-facing)
Asynchronous: Settlement, reconciliation, webhooks, fraud ML
```

```mermaid
flowchart LR
    API[Payment API] -->|sync| Auth[Authorization]
    Auth --> Kafka[Kafka Events]
    Kafka --> WH[Webhook Workers]
    Kafka --> Recon[Reconciliation]
    Kafka --> Analytics[Analytics]
```

### Multi-Processor Routing

```mermaid
flowchart TD
    Payment --> Router{Payment Router}
    Router -->|US Visa| ProcA[Processor A]
    Router -->|EU MC| ProcB[Processor B]
    Router -->|failover| ProcC[Processor C backup]
```

**Smart routing:** Cost optimization, approval rate optimization, geographic proximity.

### Disaster Recovery

| Scenario | RPO | RTO | Strategy |
|----------|-----|-----|----------|
| DB failure | 0 (sync repl) | < 1 min | Automatic failover |
| Processor outage | N/A | < 5 min | Route to backup processor |
| Region failure | 0 | < 15 min | Active-passive DR region |

---

## 12. Failure Modes & Edge Cases

| Failure | Impact | Mitigation |
|---------|--------|------------|
| Auth timeout | Unknown if charged | Query processor; never blind retry |
| Double capture | Customer overcharged | Idempotency + state machine |
| Webhook delivery fail | Merchant unaware | Retry 3 days exponential backoff |
| Ledger write fail after capture | Accounting mismatch | Saga: compensating refund |
| Processor settlement delay | Merchant payout delay | Hold in pending; communicate SLA |
| Chargeback | Merchant balance negative | Reserve fund; clawback |
| Currency FX rounding | Penny discrepancies | Integer cents everywhere; banker's rounding |

### Saga Pattern for Distributed Payment

```mermaid
sequenceDiagram
    participant O as Orchestrator
    participant P as Processor
    participant L as Ledger
    participant M as Merchant

    O->>P: Capture
    P-->>O: Success
    O->>L: Post ledger
    L-->>O: FAIL
    O->>P: Refund (compensating transaction)
    O->>M: Notify payment.failed
```

---

## 13. Trade-offs Summary

| Decision | Option A | Option B | Recommendation |
|----------|----------|----------|----------------|
| Idempotency store | Redis only | Redis + DB | **Both** (speed + durability) |
| Auth model | Single-phase | Auth + capture | **Two-phase** for e-commerce |
| PCI scope | Store PAN encrypted | Tokenize | **Tokenize** (hosted fields) |
| Consistency | 2PC | Saga + ledger | **Saga** with compensating transactions |
| Reconciliation | Real-time | Daily batch | **Daily batch** + real-time alerts |

---

## 14. Interview Walkthrough Script

### Minutes 0–5: Requirements

> "Payment gateway for merchants: authorize + capture, idempotent API, no double-charging, PCI compliant via tokenization, daily reconciliation with processor."

### Minutes 5–10: Estimation

> "50M txn/day, 580 TPS average, 10K peak. 100 GB/day transaction storage. Integer cents everywhere — never floats for money."

### Minutes 10–20: Architecture

Draw merchant → API → idempotency → orchestrator → processor + ledger. Emphasize async webhooks and reconciliation.

### Minutes 20–35: Deep Dives

- Idempotency: SETNX + cached response + processor-level key
- Double-spend: order lock + unique constraint + two-phase auth/capture
- PCI: tokenization, HSM, scope diagram
- Reconciliation: three-way match, exception queue

### Minutes 35–45: Wrap-Up

> "Financial correctness over availability — fail-closed on ambiguous processor state. Immutable ledger, never delete. Monitor reconciliation discrepancy rate daily."

---

## 15. Follow-Up Questions

1. **Design Stripe Connect (marketplace payments).** — Separate charges and transfers; connected accounts; split ledger.
2. **Handle subscription billing.** — Scheduler + dunning + payment method updater webhooks.
3. **Design fraud detection.** — Real-time scoring (< 100ms); rules engine + ML; block before processor call.
4. **Multi-currency payments.** — Presentment currency vs settlement currency; FX rate locking.
5. **Handle processor outage mid-capture.** — Circuit breaker; queue for retry; customer sees "processing" state.

---

## 16. Real-World References

| Company | Notable Design |
|---------|----------------|
| **Stripe** | Idempotency keys, Connect, webhook signatures |
| **Adyen** | Unified commerce platform, in-person + online |
| **PayPal** | Two-phase auth/capture, buyer/seller protection |
| **Square** | POS + online unified ledger |
| **Visa/Mastercard** | ISO 8583 messaging, settlement networks |

**Standards:**

- PCI DSS v4.0
- PSD2 / SCA (EU Strong Customer Authentication)
- ISO 8583 (card network messaging)
- Double-entry bookkeeping (centuries-old, still the standard)

---

> **Interview Tip:** Say **"integer cents, never floats"** early — interviewers notice. Draw the idempotency flow before the payment flow; it frames everything else.
