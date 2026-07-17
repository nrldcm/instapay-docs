# 10. Transaction Flows — Cash-In, Cash-Out, Reversal

End-to-end diagrams of every **money movement** through the service, in business
terms. Each flow shows the complete journey: who calls what, which message goes
where, what is recorded, and where the money record finally lands.

> **Companion doc:** for the exact **code path** of each endpoint (file → class →
> method, step by step), see
> [docs/code/11 — Request Walkthroughs](code/11-request-walkthroughs.md).

## Terminology map — business vs. wire

| Business term | What it means here | Wire message | Flow class |
| --- | --- | --- | --- |
| **Cash-out / Send money** | *Our* customer pays someone at another bank/EMI. We are the **originator (debtor side)**. | outbound `pacs.008`, async result `pacs.002` | `originator.flow.ts` |
| **Cash-in / Receive money** | Someone at another bank/EMI pays *our* customer. We are the **receiver (creditor side)**. | inbound `pacs.008`, our reply `pacs.002` | `receiver.flow.ts` |
| **Reversal / Cancellation** | The network asks us to undo a payment we already received. | inbound `camt.056`, our reply `pacs.002` | `cancellation.flow.ts` |
| **Ledger posting** | The durable, retried delivery of each CREDIT / DEBIT / REVERSAL to *your* core ledger. | JSON over queue or HTTP | `outbox.dispatcher.ts` |

Two independent record systems observe every flow:

- **Transaction journal** (protocol view) — one record per payment, status
  `PENDING → COMPLETED / FAILED / TIMED_OUT / RECEIVED / REVERSED`. Read via
  `GET /payments`. Persisted to the ledger DB (`transactions` table) when
  `JOURNAL_DB_ENABLED=true`, so the reconciliation feed survives restarts.
- **Ledger outbox + audit trail** (money view) — one durable event per money
  movement plus an append-only audit line for everything that happens to it.
  Read via `GET /ledger/outbox` and `GET /audit`.

---

## Flow A — Cash-out (your app sends money)

Your application calls one JSON endpoint; everything else — XML, signing,
submission, waiting for the async result, retries, ledger posting — happens
inside the service.

```mermaid
sequenceDiagram
    autonumber
    actor App as Your EMI app
    participant SVC as InstaPay service
    participant CI as BancNet CI
    participant BEN as Beneficiary bank/EMI
    participant LGR as Your core ledger

    App->>SVC: POST /payments (JSON: amount, debtor, creditor)
    Note over SVC: journal: PENDING · in-flight store: waiting
    SVC->>SVC: build + sign pacs.008 (XML)
    SVC->>CI: POST /ips-payments/service-requests
    CI-->>SVC: 202 Accepted (no outcome yet!)
    CI->>BEN: forwards the credit transfer
    BEN-->>CI: pacs.002 (accepted / rejected)
    CI->>SVC: PUT /ips-payments/service-responses (async pacs.002)
    Note over SVC: matched by Instruction Id → unblocks the waiting request
    SVC-->>App: 201 JSON result (COMPLETED / FAILED / ...)
    Note over SVC: journal: COMPLETED · DEBIT event → durable outbox
    SVC-)LGR: deliver DEBIT (retries + Idempotency-Key)
```

### Cash-out outcome states

```mermaid
flowchart TD
    START(["POST /payments"]) --> SUBMIT["submit signed pacs.008 to CI"]
    SUBMIT -->|"not 202"| REJ["REJECTED_AT_SUBMIT<br/>journal: FAILED"]
    SUBMIT -->|202| WAIT{"pacs.002 arrives<br/>within RESPONSE_TIMEOUT_MS?"}
    WAIT -->|"yes, ACTC/ACSC"| OK["COMPLETED<br/>journal: COMPLETED<br/>DEBIT → outbox"]
    WAIT -->|"yes, RJCT"| FAIL["FAILED<br/>journal: FAILED"]
    WAIT -->|no| RETRY{"attempts left?<br/>(MAX_RESUBMISSIONS)"}
    RETRY -->|yes| DUPL["resend SAME InstrId<br/>with DUPL flag"] --> WAIT
    RETRY -->|no| TIMEOUT["TIMED_OUT<br/>journal: TIMED_OUT"]

    style OK fill:#e6ffe6,stroke:#2e7d32,color:#1b5e20
    style FAIL fill:#ffebee,stroke:#c62828,color:#b71c1c
    style REJ fill:#ffebee,stroke:#c62828,color:#b71c1c
    style TIMEOUT fill:#fff8e1,stroke:#f9a825,color:#795500
```

**Duplicate safety:** a resend reuses the *same* Instruction Id with the BAH
`CpyDplct=DUPL` flag, so the network can never double-pay — at-least-once
submission, at-most-once settlement.

---

## Flow B — Cash-in (your customer receives money)

The CI pushes a signed `pacs.008` at us. We validate, record, enqueue the credit
for your ledger, and answer **synchronously** with a signed `pacs.002`.

```mermaid
sequenceDiagram
    autonumber
    participant ORIG as Sending bank/EMI
    participant CI as BancNet CI
    participant SVC as InstaPay service
    participant LGR as Your core ledger

    ORIG->>CI: credit transfer for our customer
    CI->>SVC: POST /ips-payments/service-requests (signed pacs.008)
    SVC->>SVC: verify signature → XSD validate → parse
    Note over SVC: journal: RECEIVED · audit: RECEIVED
    SVC->>SVC: validate beneficiary (your hook)
    alt beneficiary OK
        Note over SVC: CREDIT event → durable outbox · journal: COMPLETED
        SVC-->>CI: 201 + signed pacs.002 (ACTC)
        SVC-)LGR: deliver CREDIT (retries + Idempotency-Key)
    else beneficiary rejected (e.g. AC01 unknown account)
        Note over SVC: journal: FAILED
        SVC-->>CI: 400 + signed pacs.002 (RJCT + reason code)
    end
    CI-->>ORIG: outcome forwarded
```

Key property: **nothing is ever ACKed to the network without a durable record.**
The `pacs.002 ACTC` is only sent after the CREDIT event is safely in the outbox
and the audit trail — see [docs/09 — Ledger & Audit](09-ledger-and-audit.md).

---

## Flow C — Reversal (the network cancels a received payment)

A `camt.056` asks us to undo a payment we previously received (Flow B).

```mermaid
sequenceDiagram
    autonumber
    participant CI as BancNet CI
    participant SVC as InstaPay service
    participant LGR as Your core ledger

    CI->>SVC: PUT /ips-payments/payment-instructions (signed camt.056)
    SVC->>SVC: verify signature → XSD validate → parse
    SVC->>SVC: was OrgnlInstrId received before?
    alt matched — we credited it earlier
        Note over SVC: REVERSAL event → durable outbox<br/>journal: REVERSED
        SVC-)LGR: deliver REVERSAL (retries + Idempotency-Key)
    else unknown instruction
        Note over SVC: nothing to undo — acknowledge only
    end
    SVC-->>CI: 200 + signed pacs.002 (ACTC)
```

We always reply with a valid signed `pacs.002` — even for an unknown instruction
— so the CI stops re-sending the cancellation.

---

## Flow D — Ledger delivery (how a money event reaches your core)

Every CREDIT (cash-in), DEBIT (cash-out), and REVERSAL from the flows above goes
through the same **money-safe pipeline**: durable outbox → retrying dispatcher →
your ledger. This runs in the background, decoupled from the network reply.

```mermaid
flowchart LR
    subgraph flows["Payment flows"]
        CIN["Cash-in<br/>CREDIT"]
        COUT["Cash-out<br/>DEBIT"]
        REV["Reversal<br/>REVERSAL"]
    end

    subgraph safe["Money-safe pipeline (ledger/)"]
        OUTBOX[("Durable outbox<br/>PENDING")]
        DISP["OutboxDispatcher<br/>poll + exponential backoff"]
        AUDIT[("Audit trail<br/>append-only")]
    end

    subgraph yours["Your systems"]
        Q{{"queue (RMQ/NATS/Redis)<br/>or HTTP POST"}}
        CORE["Your core ledger<br/>(dedupes by Idempotency-Key)"]
    end

    CIN --> OUTBOX
    COUT --> OUTBOX
    REV --> OUTBOX
    OUTBOX --> DISP --> Q --> CORE
    flows -. every step .-> AUDIT
    DISP -. every attempt/result .-> AUDIT
```

### Outbox event lifecycle

```mermaid
stateDiagram-v2
    [*] --> PENDING: flow enqueues event<br/>(audit ENQUEUED)
    PENDING --> DELIVERED: delivery OK<br/>(audit DELIVERY_OK)
    PENDING --> PENDING: delivery failed —<br/>retry with backoff<br/>(audit DELIVERY_FAILED)
    PENDING --> DEAD: max attempts reached<br/>(audit DEAD_LETTER)
    DELIVERED --> [*]
    DEAD --> [*]: reconciliation cron<br/>picks it up via GET /ledger/outbox?status=DEAD
```

- **Idempotent:** the `instructionId` travels as the idempotency key
  (`Idempotency-Key` HTTP header / message field), so retries can never
  double-book.
- **`LEDGER_ENABLED=false`** (default): events still land in the outbox and audit
  trail — recorded and queryable, just not delivered yet.
- Dead-lettered events are never lost: they stay queryable at
  `GET /ledger/outbox?status=DEAD` for your reconciliation cron.

---

## Journal status lifecycle (both directions)

The transaction journal is the reconciliation view your systems poll
(`GET /payments?since=...`). Its statuses:

```mermaid
stateDiagram-v2
    state "OUTBOUND (cash-out)" as OUT {
        [*] --> PENDING: POST /payments
        PENDING --> COMPLETED: pacs.002 ACTC/ACSC matched
        PENDING --> FAILED: pacs.002 RJCT or submit rejected
        PENDING --> TIMED_OUT: no reply after all resubmissions
    }
    state "INBOUND (cash-in)" as IN {
        [*] --> RECEIVED: pacs.008 arrives
        RECEIVED --> COMPLETED_IN: beneficiary OK, credit enqueued
        RECEIVED --> FAILED_IN: beneficiary rejected
        COMPLETED_IN --> REVERSED: camt.056 matched
    }
```

(`COMPLETED_IN` / `FAILED_IN` are the same `COMPLETED` / `FAILED` values, shown
separately per direction.)

---

## Which records to check, when

| Question | Endpoint | Backed by |
| --- | --- | --- |
| "What happened to payment X?" | `GET /payments/{instructionId}` | journal |
| "What changed since my last poll?" (reconciliation) | `GET /payments?since=...` | journal |
| "Did the money event reach our ledger?" | `GET /audit?instructionId=X` | audit trail |
| "Is anything stuck or dead-lettered?" | `GET /ledger/outbox?status=PENDING\|DEAD` | outbox |
| "Show me the service logs for this payment" | `GET /logs?instructionId=X` | logs DB |

Same queries exist as message patterns for queue-based reconciliation — see the
[API Reference](05-api-reference.md#microservice-message-patterns).

---

Next: **[docs/code/11 — Request Walkthroughs](code/11-request-walkthroughs.md)** —
the same flows traced through the actual source files, step by step.
