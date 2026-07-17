# 08 — Data Model

> **In plain terms.** This is the **shape of what the service remembers**. Three
> durable tables live in databases: a **to-do list** of money movements
> (`outbox_events`), a permanent **logbook** of every step (`audit_log`), and an
> optional searchable **diary** of application logs (`app_logs`). Plus one thing
> kept only in memory: a lightweight **transaction journal** for reconciliation.
> The money tables and the log table live in **two separate databases** on purpose —
> a logging problem must never touch money handling.

> **The DDL/SQL files are being finalized by another agent.** This page describes
> the **columns and their meaning** at the model level and *why* they are
> partitioned/indexed — not the exact `CREATE TABLE` text. Definitions live in
> `ledger.types.ts` and
> `logs-query.service.ts`; the
> DDL lives under `db/<engine>/` (do not treat as final here).

Jargon: **partitioning** = physically splitting one huge table into time-sliced
chunks so queries scan only the relevant slice and old data can be dropped instantly.
**BRIN** = a tiny Postgres index ideal for naturally time-ordered data. **Idempotency
constraint** = a uniqueness rule that stops the same thing being recorded twice.

---

## Two separate databases

| | **Ledger DB** | **Logs DB** |
| --- | --- | --- |
| Tables | `outbox_events`, `audit_log` | `app_logs` |
| Config | `LEDGER_DB_*` | `DB_*` / `LOG_DB_*` |
| Nature | Transactional, must-not-lose | Best-effort observability |
| DataSource | `ledger-datasource.ts` | `logs-datasource.ts` |

Both DataSources have `synchronize` **disabled** — schemas are owned by DDL (and
optionally auto-scaffolded, see [07](07-runtime-and-modes.md#auto-scaffolding-databases)),
never by the app.

---

## `outbox_events` — the durable to-do list

One row per money movement to deliver. Fields from `OutboxEvent`
(`ledger.types.ts`):

| Column | Meaning |
| --- | --- |
| `id` | Unique row id (UUID). |
| `instruction_id` | The payment's ISO Instruction Id — the **idempotency key**. |
| `type` | `CREDIT` / `DEBIT` / `REVERSAL`. |
| `direction` | `INBOUND` / `OUTBOUND`. |
| `amount`, `currency` | The money moved. |
| `counterparty_bic`, `counterparty_name` | The other institution / party. |
| `end_to_end_id`, `transaction_id` | ISO correlation ids. |
| `status` | `PENDING` → `DELIVERED` \| `DEAD`. |
| `attempts` | Delivery attempts so far. |
| `next_attempt_at` | Epoch ms of the next due attempt (drives backoff + lease). |
| `last_error` | Last delivery error (for DEAD triage). |
| `created_at`, `updated_at`, `delivered_at` | Timestamps. |

**Why it's built this way (high level):**

- **UNIQUE `(instruction_id, type)`** — the idempotency constraint. `enqueue` inserts
  "do nothing on conflict", so a duplicate returns the existing row instead of
  booking a second movement.
- **Filtered index on `(status, next_attempt_at) WHERE status='PENDING'`** — keeps
  the dispatcher's poll `O(due rows)` even with millions of `DELIVERED` rows.
- **Index on `(instruction_id)`** — point lookups.
- **Concurrency:** `claimDue` uses locked/leased claims (`FOR UPDATE SKIP LOCKED` /
  `READPAST`) so many dispatcher instances never grab the same row — see
  [05 — Ledger](05-ledger-money-safe.md#the-separate-transactional-ledger-db-design-intent).
- **Retention:** archive/prune `DELIVERED`; keep `DEAD` for reconciliation.

---

## `audit_log` — the permanent logbook

Append-only; one row per step, grows to billions. Fields from `AuditEntry`:

| Column | Meaning |
| --- | --- |
| `id` | Unique row id. |
| `ts` | When the step happened. |
| `instruction_id` | Which payment. |
| `action` | `RECEIVED` · `ENQUEUED` · `DELIVERY_ATTEMPT` · `DELIVERY_OK` · `DELIVERY_FAILED` · `DEAD_LETTER`. |
| `type`, `direction` | Movement kind (when applicable). |
| `amount`, `currency` | Money (when applicable). |
| `attempt` | Delivery attempt number (for delivery actions). |
| `detail` | Free-text context (e.g. an error). |

**Why it's built this way:**

- **Monthly RANGE partitioning on `ts`** — time-bounded reads prune to the right
  month(s); retention is an instant DROP/SWITCH of an old partition, not a giant
  `DELETE`.
- **Postgres BRIN index on `ts`** — audit rows insert in ~time order, so a tiny
  per-block min/max index covers time-range scans (MySQL uses clustered PK +
  partition pruning; SQL Server a partition-aligned clustered index).
- **Indexes on `(instruction_id)` and `(action, ts)`** — the common audit filters.

---

## `app_logs` — the searchable diary (optional)

Written only when `LOG_DB_ENABLED=true`, in the **separate** logs DB. Columns
(`db-log.transport.ts` /
`LogRecord`):

| Column | Meaning |
| --- | --- |
| `ts` | Record timestamp. |
| `level` | `error`/`warn`/`info`/… |
| `context` | Logging context (e.g. class name). |
| `message` | The log message. |
| `correlation_id` | Correlation id, if present. |
| `instruction_id` | ISO Instruction Id, if present. |
| `biz_msg_idr` | Business message id, if present. |
| `host`, `pid` | Emitting node hostname / process id. |
| `meta` | Remaining structured fields, as JSON. |

**Why:** time-range partitioning + BRIN-on-timestamp for cheap time scans, plus
targeted indexes on `instruction_id` / `biz_msg_idr` / `correlation_id` for the
`GET /logs` lookups ([06](06-logging-and-query.md)). Retention drops whole partitions.

---

## The in-memory transaction journal

Not a database — an in-memory map in
`transaction.journal.ts` that
backs the `GET /payments` reconciliation feed. It survives only while the process
runs. A `TxRecord`:

| Field | Meaning |
| --- | --- |
| `instructionId` | Key. |
| `endToEndId`, `transactionId` | ISO correlation ids. |
| `direction` | `OUTBOUND` / `INBOUND`. |
| `amount`, `currency` | The payment. |
| `counterpartyName`, `counterpartyBic` | The other party. |
| `status` | `PENDING` / `RECEIVED` / `COMPLETED` / `FAILED` / `TIMED_OUT` / `REVERSED`. |
| `reasonCode` | ISO reason on failure. |
| `createdAt`, `updatedAt` | ISO timestamps. |

> The **in-flight store** (`inflight.store.ts`)
> is also in-memory but transient (only tracks payments *awaiting a reply* and
> received ids for cancellation matching) — see
> [04](04-instapay-flows.md#in-flight-matching-timeout--duplicates--inflightstore).

---

Next: **[09 — Extending Safely](09-extending.md)** ·
Back to the **[index](00-index.md)**. Schema rationale in depth:
[09 — Ledger & Audit](../09-ledger-and-audit.md#billions-scale-schema).
