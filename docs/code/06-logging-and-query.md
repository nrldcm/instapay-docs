# 06 — Logging & Query

> **In plain terms.** The service keeps a **diary** of everything it does. By
> default it writes to the screen (console) and to **files sorted into a folder per
> day and a separate file per severity** — so you can open "today's errors" or
> "today's successes" on their own. For a busy operation you can *also* mirror every
> line into a database so you can **search** across all machines ("show me every log
> for payment INSTR123"). The database mirror is best-effort: if it hiccups, the
> diary still writes to files and the app never slows down or crashes.

**Code:** `src/common/logging/` +
`src/logs/`.

Jargon: a **transport** (winston term) is one destination for logs — the console,
a file, or a database. **Rotation** = closing a file when it gets big/old and
starting a new one, deleting very old ones.

---

## Console + rotating files

`logger.config.ts` `buildLoggerOptions()`
builds these transports (winston):

| Transport | Contents |
| --- | --- |
| **Console** | Human-friendly, colorized (NestLike format). |
| `combined-%DATE%.log` | Everything (all levels). |
| `error-%DATE%.log` | **Only** `error` (retained longer). |
| `warn-%DATE%.log` | **Only** `warn`. |
| `info-%DATE%.log` | **Only** `info` (normal successes). |

Files live under a **per-day subfolder** and are timestamped **JSON**, one object
per line:

```
logs/
└── 2026-07-16/
    ├── combined-2026-07-16.log
    ├── error-2026-07-16.log
    ├── warn-2026-07-16.log
    └── info-2026-07-16.log
```

The dedicated `error`/`warn`/`info` files hold **only** that exact level (they do
not inherit higher severities), so tailing successes or errors alone is trivial. All
rotating files gzip old segments (`zippedArchive`).

### Controlling files & levels

| Env var | Meaning | Default |
| --- | --- | --- |
| `LOG_DIR` | Base folder | `logs` |
| `LOG_LEVEL` | Minimum severity captured | `info` |
| `LOG_MAX_SIZE` | Rotate a file at this size | `20m` |
| `LOG_MAX_FILES` | Retention for combined/warn/info | `14d` |
| `LOG_ERROR_MAX_FILES` | Retention for error logs | `30d` |

Levels: `error` (something failed) · `warn` (recoverable — rejections, timeouts,
unmatched responses) · `info` (normal successes) · `debug`/`verbose` (diagnostics,
not written to the dedicated files).

> **Privacy (from the code):** never log full ISO 20022 payloads at `info` — they
> carry account data. Keep payload dumps at `debug`.

---

## Optional hybrid database logging

When **`LOG_DB_ENABLED=true`**, `buildLoggerOptions()` lazily loads the DB transport
(`db/db-log.transport.ts` via
`createDbLogTransport()`) and adds it **alongside** the files. It is wrapped in
try/catch so it can never crash boot.

**Design (documented at a high level — DB internals are being finalized
separately):**

- **Never blocks the request path.** `log()` pushes to an in-memory buffer and
  returns; errors are swallowed, never thrown.
- **Batched bulk writes.** Flushed as one bulk insert on a size threshold
  (`LOG_DB_BATCH_SIZE`) or a timer (`LOG_DB_FLUSH_MS`). On write failure the batch is
  dropped (bounded memory), with a best-effort final flush on shutdown.
- **Dedicated DataSource.** `db/logs-datasource.ts`
  is a **separate**, memoized TypeORM DataSource with `synchronize` disabled — the
  partitioned `app_logs` table is owned by the DDL.
- **Engine-agnostic** across Postgres / MySQL / SQL Server.

### Enabling it

| Env var | Meaning |
| --- | --- |
| `LOG_DB_ENABLED` | Turn the DB sink on. |
| `DB_TYPE` / `DB_HOST` / `DB_PORT` / `DB_USERNAME` / `DB_PASSWORD` / `DB_DATABASE` / `DB_SCHEMA` / `DB_SSL` | Logs DB connection. |
| `LOG_DB_TABLE` | Table name (default `app_logs`). |
| `LOG_DB_BATCH_SIZE` / `LOG_DB_FLUSH_MS` / `LOG_DB_LEVEL` | Batch size / flush timer / minimum level. |

If `DB_DATABASE` is unset the sink disables itself; if the DB is unreachable the app
still boots and logs to files.

> **This logs DB is SEPARATE from the money-safe ledger DB** (`LEDGER_DB_*`). They
> never share a pool or a failure domain — see [05 — Ledger](05-ledger-money-safe.md#the-separate-transactional-ledger-db-design-intent).
> The row shape is in [08 — Data Model](08-data-model.md#app_logs).

---

## Querying logs

When the DB sink is on, two query interfaces let you search the stored logs.

### HTTP — `GET /logs`

`logs.controller.ts` →
`logs-query.service.ts`.

| Query param | Filters on |
| --- | --- |
| `from`, `to` | Timestamp range (ISO). |
| `level` | `error`/`warn`/`info`/`debug`/`verbose`. |
| `instructionId` | ISO Instruction Id. |
| `correlationId` | Correlation id. |
| `bizMsgIdr` | Business message id. |
| `limit`, `offset` | Paging (limit clamped 1–1000, default 100). |

Returns `{ count, logs }`. If `LOG_DB_ENABLED !== 'true'`, it returns
`{ count: 0, logs: [], note: 'DB logging disabled …' }` rather than erroring; query
errors likewise return an empty result with a `note`, so a caller is never broken by
the log store.

### Message pattern — `logs.query`

`logs.message.controller.ts`
exposes `@MessagePattern({ cmd: 'logs.query' })` accepting the same filter and
returning `{ count, logs, note? }` — the transport-agnostic equivalent of
`GET /logs` for microservice / hybrid mode ([07](07-runtime-and-modes.md)).

---

Next: **[07 — Runtime & Modes](07-runtime-and-modes.md)** ·
Back to the **[index](00-index.md)**. See also the top-level
[06 — Logging](../06-logging.md).
