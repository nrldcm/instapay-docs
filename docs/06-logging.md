# 6. Logging

The service uses **[winston](https://github.com/winstonjs/winston)** for logging.
Out of the box it writes to the **console** and to **rotating files organised by
date and severity**. Optionally, it can **also** persist logs to a relational
database designed for very high volumes. Configuration lives in
`logger.config.ts` and is driven by the
`LOG_*` / `DB_*` environment variables (see [Setup](02-setup.md#logging-rotating-files--console)).

---

## File layout: by date folder, separated by type

Logs are written under `LOG_DIR` (default `logs/`). Each **day gets its own
subfolder**, and within it logs are **split by severity** into separate channels:

```
logs/
└── 2026-07-16/
    ├── combined-2026-07-16.log   ← everything (all levels)
    ├── error-2026-07-16.log      ← errors only
    ├── warn-2026-07-16.log       ← warnings only
    └── info-2026-07-16.log       ← normal / successful operations
```

The `combined` channel captures every record; the `error`, `warn`, and `info`
channels each contain **only** records of exactly that level (they do not inherit
higher severities), which makes it easy to, say, tail just successes or just errors.

File records are written as timestamped **JSON** (one object per line), which is
convenient for log shippers and search tools.

---

## Log levels

| Level | Meaning in this service |
| --- | --- |
| `error` | Something failed (system errors, failed signing, dropped log batches). |
| `warn` | Recoverable problems (rejections, timeouts, unmatched responses, heartbeat failures). |
| `info` | **Normal / successful** operations (accepted transfers, sign-on, reversals). |
| `debug` / `verbose` | Detailed diagnostics — includes lower-level detail; not written to dedicated files. |

`LOG_LEVEL` (default `info`) sets the minimum severity captured.

> **Privacy note (from the code):** never log full ISO 20022 payloads at `info`
> level — they carry account data. Keep payload dumps at `debug`.

---

## Rotation & retention

Each file **rotates** automatically and old files are pruned. Controlled by:

| Env var | Meaning | Default |
| --- | --- | --- |
| `LOG_MAX_SIZE` | Rotate a file once it reaches this size. | `20m` |
| `LOG_MAX_FILES` | Retention for the combined/warn/info logs. | `14d` |
| `LOG_ERROR_MAX_FILES` | Retention for the error logs (usually longer). | `30d` |

Rotated files are **gzip-compressed** (`zippedArchive`). The date pattern
(`YYYY-MM-DD`) drives both the per-day subfolder and the dated filename.

---

## Optional database logging

> **Status:** fully implemented. The DB logging code lives under
> `src/common/logging/db/`
> (`db-log.transport.ts`,
> `logs-datasource.ts`,
> `logs-query.service.ts`), and the
> partitioned table DDL is provided per engine at
> `db/postgres/schema.sql`,
> `db/mysql/schema.sql`, and
> `db/mssql/schema.sql`. Run the matching script once to
> create the `app_logs` table, then set `LOG_DB_ENABLED=true`.

> **This logs DB is SEPARATE from the money-safe ledger DB.** The outbox + audit
> trail live in their own **dedicated, transactional** database with its own
> connection and its own `LEDGER_DB_*` settings — see
> [09 — Ledger & Audit](09-ledger-and-audit.md). The two never share a pool or a
> failure domain: a log-sink outage must never touch money handling, and vice
> versa.

### What it's for

File logs are great for a single node, but for a busy participant you may want a
**central, queryable store** holding **millions to billions** of log rows for audit,
analytics, and cross-node search. When `LOG_DB_ENABLED=true`, every record that also
goes to the files is **additionally** written to a relational database — without ever
slowing down or crashing request handling.

### How to enable it

1. Provision a database (Postgres, MySQL, or SQL Server).
2. Create the log table and its partitions from the DDL at `db/<engine>/schema.sql`.
3. Set the environment variables and restart:

```bash
LOG_DB_ENABLED=true
DB_TYPE=postgres          # postgres | mysql | mssql
DB_HOST=your-db-host
DB_PORT=5432
DB_USERNAME=instapay
DB_PASSWORD=your-secret
DB_DATABASE=instapay_logs
DB_SCHEMA=public          # postgres/mssql only
DB_SSL=true               # for managed/remote DBs
LOG_DB_TABLE=app_logs
LOG_DB_BATCH_SIZE=500
LOG_DB_FLUSH_MS=2000
LOG_DB_LEVEL=info
```

If `DB_DATABASE` is not set, the DB sink disables itself (file + console logging
continues). If the database is unreachable, the app still boots and logs to files.

### The row shape

Each log becomes one row. Columns written by the transport
(`db-log.transport.ts`):

| Column | Contents |
| --- | --- |
| `ts` | Timestamp of the record. |
| `level` | Severity (`error`/`warn`/`info`/…). |
| `context` | The logging context/label (e.g. class name). |
| `message` | The log message. |
| `correlation_id` | Correlation id, if present. |
| `instruction_id` | ISO Instruction Id, if present. |
| `biz_msg_idr` | Business message id, if present. |
| `host` | Hostname of the emitting node. |
| `pid` | Process id. |
| `meta` | Any remaining structured fields, as JSON. |

### Scaling design (high level)

The DB sink is built so it can absorb enormous volumes without becoming a bottleneck:

- **Never blocks the request path.** `log()` only pushes onto an in-memory buffer and
  returns immediately; a logging error is swallowed, never thrown.
- **Batched bulk writes.** Records are flushed as a **single bulk insert**, triggered
  either by a size threshold (`LOG_DB_BATCH_SIZE`) or a timer (`LOG_DB_FLUSH_MS`). On
  a write failure the batch is **dropped** (not retried forever) to avoid unbounded
  memory growth. A best-effort final flush runs on shutdown.
- **Engine-agnostic.** Uses TypeORM's query builder so per-driver
  placeholders/escaping are handled across Postgres/MySQL/SQL Server.
- **Time-range partitioning** (in the DDL). The table is partitioned by time
  (e.g. by day/month) so writes hit the newest partition and queries prune to a
  date range instead of scanning everything.
- **Index strategy.** Time-appropriate indexes — e.g. **BRIN** indexes on the
  timestamp in Postgres (tiny and ideal for naturally time-ordered inserts), plus
  targeted indexes on `instruction_id` / `biz_msg_idr` for lookups.
- **Cheap retention.** Old data is aged out by **dropping whole partitions** (a
  near-instant metadata operation) rather than issuing large, expensive `DELETE`s.
- **Dedicated datasource.** The log store uses a **separate** TypeORM DataSource
  with `synchronize` disabled, so the schema/partitioning stays owned by the DDL and
  the log sink can never interfere with (or depend on) any business database.

### DDL location

Create/maintain the table and partitions here (one file per engine):

```
db/
├── postgres/schema.sql
├── mysql/schema.sql
└── mssql/schema.sql
```

---

Next: **[07 — Glossary](07-glossary.md)** or
**[08 — Security & Compliance](08-security-and-compliance.md)**.
