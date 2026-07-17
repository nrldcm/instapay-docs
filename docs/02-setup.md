# 2. Setup & Configuration

This guide gets the service running on your machine and explains **every** setting.
It is written so a non-specialist can follow it. Technical terms are defined in the
[Glossary](07-glossary.md).

---

## Prerequisites

| Tool | Version | Why |
| --- | --- | --- |
| **Node.js** | 20 or newer | Runs the service |
| **npm** | Comes with Node.js | Installs dependencies |
| **OpenSSL** | Any recent version (recommended) | Generates development certificates. If missing, a limited fallback is used — see below. |

Check your versions:

```bash
node --version      # should print v20.x or higher
openssl version     # optional but recommended
```

---

## Step-by-step install

```bash
# 1. Install dependencies
npm install

# 2. Generate throw-away development certificates
#    Creates: certs/server.key, certs/server.crt, certs/ca.crt
npm run gen:certs

# 3. Copy the environment template and edit it
cp .env.example .env        # Windows PowerShell: copy .env.example .env

# 4. Start in watch/hot-reload mode
npm run start:dev

# 5. Open the live API docs in a browser
#    http://localhost:8443/docs
```

That's it for local development. TLS is off by default, so the server listens on
plain HTTP at port `8443`. There is no live network locally — see
[Connecting to a CI or simulator](#connecting-to-a-ci-or-simulator).

### About the development certificates

`npm run gen:certs` (script: `scripts/gen-dev-certs.js`) creates a **self-signed**
key and certificate for local use. **The same key/certificate pair is used for two
things:** encrypting the HTTPS connection (**TLS**) *and* digitally signing every
ISO 20022 message.

- If **OpenSSL** is installed, it creates a proper X.509 certificate with
  `localhost` / `127.0.0.1` names.
- If OpenSSL is **not** found, it writes a private key but cannot mint a
  certificate and exits with guidance — install OpenSSL and re-run, or supply
  `certs/server.crt` and `certs/ca.crt` yourself.

> **These are throw-away certificates for local testing only.** Never commit real
> keys. In production the real certificate comes from an approved CA during BancNet
> onboarding. See [08 — Security & Compliance](08-security-and-compliance.md).

---

## Configuration: the `.env` file

All configuration lives in a `.env` file (copied from
`.env.example`). **Every value has a safe local-dev default**, so
you can start with the template unchanged. The code reads these variables in
`src/config/configuration.ts`.

The tables below explain each variable in plain language, grouped by section.

### Run mode

| Variable | Meaning | Default | When to change |
| --- | --- | --- | --- |
| `APP_MODE` | How the process runs: `api` (HTTP only), `microservice` (internal processor only — CI endpoints OFF), or `hybrid` (both). | `api` | Set `microservice`/`hybrid` if internal apps should call this over TCP/NATS/Redis/RabbitMQ. See [Integration Guide](04-integration-guide.md#microservice-mode). |

### HTTP server

| Variable | Meaning | Default | When to change |
| --- | --- | --- | --- |
| `PORT` | TCP port the HTTP server listens on. | `8443` | Change to avoid a clash or match your infrastructure. |

### Our participant identity

These identify **you** on the network. They are assigned by BancNet during
onboarding; the defaults are placeholders for local testing.

| Variable | Meaning | Default | When to change |
| --- | --- | --- | --- |
| `PARTICIPANT_ID` | Your participant identifier, used in the sign-on / sign-off URL to the CI. | `PARTICIPANT001` | Set to your real ID once onboarded. |
| `PARTICIPANT_BICFI` | Your institution's **BIC** (bank identifier). Appears as the sender in message headers and as the debtor agent. | `NRLDPHM1XXX` | Set to your assigned BIC. |
| `PARTICIPANT_NAME` | Human-readable name, used only in logs. | `NRLDCM EMI` | Cosmetic; set to your name. |

### Central Infrastructure (CI) / simulator

Where **outbound** messages are sent (submitting payments, sign-on, health checks).

| Variable | Meaning | Default | When to change |
| --- | --- | --- | --- |
| `CI_BASE_URL` | Base URL of the CI (or the InstaPay **Simulator** / your mock). | `https://localhost:9443` | Point at your simulator/mock during testing; at the real CI in production. |
| `CI_BICFI` | The BIC that identifies the CI / scheme. Used as the "To" party when addressing the service. | `BNETPHMMXXX` | Set per BancNet's environment details. |

### TLS / mutual TLS

Controls encryption and certificate checking on the **inbound** HTTPS server.

| Variable | Meaning | Default | When to change |
| --- | --- | --- | --- |
| `TLS_ENABLED` | Turn HTTPS + **mutual TLS** on. Off = plain HTTP for local dev. | `false` | Set `true` for any real / networked deployment. |
| `TLS_CERT_PATH` | Path to your server certificate (PEM). Also used as the **signing** certificate. | `certs/server.crt` | Point at your CA-issued cert in production. |
| `TLS_KEY_PATH` | Path to your private key (PEM). Used for the TLS server **and** message signing. | `certs/server.key` | Point at your production key (kept secret). |
| `TLS_CA_PATH` | Path to the CA bundle you trust for the peer's certificate. Also used to **pin** the CI's signing certificate for verification. | `certs/ca.crt` | Point at BancNet's CA bundle in production. |
| `TLS_REQUEST_CLIENT_CERT` | Require and verify the caller's client certificate on inbound calls (true mutual TLS). | `true` | Keep `true` with the real CI; may relax for some local tests. |

### Scheme SLA timers

Timing rules that mirror InstaPay's service-level expectations.

| Variable | Meaning | Default | When to change |
| --- | --- | --- | --- |
| `RESPONSE_TIMEOUT_MS` | How long to wait for the asynchronous `pacs.002` result before timing out an originated payment. | `20000` (20s) | Tune to the scheme SLA / network latency. |
| `MAX_RESUBMISSIONS` | How many times to resubmit the **same** instruction with a duplicate (`DUPL`) flag after a timeout before giving up. | `2` | Adjust per scheme guidance. |
| `HEALTHCHECK_INTERVAL_MS` | Interval for the outbound health-check heartbeat to the CI. `0` disables it. | `0` | Set a positive value (e.g. `30000`) to enable heartbeats once signed on. |

### Participant lifecycle

| Variable | Meaning | Default | When to change |
| --- | --- | --- | --- |
| `AUTO_SIGN_ON` | Automatically sign on to the CI at boot and sign off at shutdown. Needs a reachable CI. | `false` | Set `true` when connected to a real/simulated CI. |

### Internal microservice transport

Used only in `microservice` / `hybrid` mode. **This is not the BancNet contract** —
it is for your own apps calling this service as a processor.

| Variable | Meaning | Default | When to change |
| --- | --- | --- | --- |
| `MS_TRANSPORT` | Transport for internal calls: `TCP`, `NATS`, `REDIS`, or `RMQ` (RabbitMQ). | `TCP` | Match your internal messaging stack. |
| `MS_HOST` | Host/interface to bind (TCP/Redis). | `127.0.0.1` | Change to expose beyond localhost. |
| `MS_PORT` | Port for the internal transport (TCP/Redis). | `8877` | Avoid clashes. |
| `MS_TLS` | Wrap the TCP transport in TLS with mutual auth (reuses the participant certs). | `false` | Set `true` to encrypt/authenticate internal TCP traffic. |
| `MS_URL` | Broker/server URL for NATS/RabbitMQ (e.g. `nats://127.0.0.1:4222`). | *(unset)* | Set when using NATS/RMQ. |

### OpenAPI export

| Variable | Meaning | Default | When to change |
| --- | --- | --- | --- |
| `OPENAPI_EXPORT` | On boot, also write the OpenAPI spec to disk. | `true` | Set `false` to skip the file export. |
| `OPENAPI_DIR` | Folder for the exported `openapi.json`. | `openapi` | Change the output location. |

Regardless of this setting, the live docs are always at `/docs` (UI), `/docs-json`
(JSON), and `/docs-yaml` (YAML).

### Logging (rotating files + console)

See [06 — Logging](06-logging.md) for the full picture.

| Variable | Meaning | Default | When to change |
| --- | --- | --- | --- |
| `LOG_DIR` | Base folder for log files. | `logs` | Point at a mounted volume in production. |
| `LOG_LEVEL` | Minimum severity to record: `error`, `warn`, `info`, `debug`, `verbose`. | `info` | Use `debug` for troubleshooting. |
| `LOG_MAX_SIZE` | Rotate a log file once it reaches this size. | `20m` | Tune to disk/volume. |
| `LOG_MAX_FILES` | How long to keep the combined logs. | `14d` | Increase for longer retention. |
| `LOG_ERROR_MAX_FILES` | How long to keep the **error** logs. | `30d` | Errors are usually kept longer. |

### Hybrid DB logging (optional)

Optionally persist logs to a relational database as well as to files, designed for
very high volumes. See [06 — Logging](06-logging.md#optional-database-logging).

| Variable | Meaning | Default | When to change |
| --- | --- | --- | --- |
| `LOG_DB_ENABLED` | Turn on the database log sink (in addition to files). | `false` | Set `true` to persist logs to a DB. |
| `DB_TYPE` | Database engine: `postgres`, `mysql`, or `mssql`. | `postgres` | Match your database. |
| `DB_HOST` | Database host. | `127.0.0.1` | Point at your DB server. |
| `DB_PORT` | Database port (postgres 5432, mysql 3306, mssql 1433). | `5432` | Match your engine/host. |
| `DB_USERNAME` | Database user. | `instapay` | Set your credentials. |
| `DB_PASSWORD` | Database password. | `change-me` | **Always change** for real use. |
| `DB_DATABASE` | Database name. If unset, the DB sink is disabled even when enabled. | `instapay_logs` | Set to your logs database. |
| `DB_SCHEMA` | Schema name (postgres/mssql; ignored by mysql). | `public` | Match your schema. |
| `DB_SSL` | Use SSL to connect to the database. | `false` | Set `true` for remote/managed DBs. |
| `LOG_DB_TABLE` | Table that receives log rows. | `app_logs` | Match your DDL. |
| `LOG_DB_BATCH_SIZE` | Rows per bulk insert. | `500` | Tune for throughput. |
| `LOG_DB_FLUSH_MS` | Max time before a partial batch is flushed. | `2000` | Lower for fresher data, higher for fewer writes. |
| `LOG_DB_LEVEL` | Minimum severity persisted to the DB. | `info` | Raise to store less. |

### Ledger delivery (money-safe outbox → counterpart ledger)

Every money event (credit/debit/reversal) is written to a durable **outbox** and an
**audit** trail, then delivered to the counterpart ledger with retries, exponential
backoff, and dead-lettering. See [09 — Ledger & Audit](09-ledger-and-audit.md).

| Variable | Meaning | Default | When to change |
| --- | --- | --- | --- |
| `LEDGER_ENABLED` | Actually **deliver** money events to the ledger. `false` = still record them in the outbox + audit, but don't send. | `false` | Set `true` once a ledger endpoint/queue is reachable. |
| `LEDGER_MODE` | How to deliver: `queue` (message queue — durable, recommended) or `api` (HTTP POST). | `queue` | Match how your ledger accepts events. |
| `LEDGER_URL` | For `mode=api`: the ledger HTTP endpoint. Must be **idempotent** (dedupes on the instruction id / Idempotency-Key). | `http://localhost:9100/ledger/events` | Point at your ledger's API. |
| `LEDGER_QUEUE_TRANSPORT` | For `mode=queue`: broker type — `RMQ` (RabbitMQ), `NATS`, or `REDIS`. | `RMQ` | Match your broker. |
| `LEDGER_QUEUE_URL` | For `mode=queue`: broker connection URL. | `amqp://127.0.0.1:5672` | Point at your broker. |
| `LEDGER_QUEUE_NAME` | For `mode=queue`: queue name / event pattern the ledger consumes. | `ledger.events` | Match your consumer. |
| `LEDGER_MAX_ATTEMPTS` | Delivery attempts before an event is moved to **dead-letter** (`DEAD`) for reconciliation. | `5` | Raise for flakier links. |
| `LEDGER_POLL_MS` | How often the dispatcher polls the outbox for due events. | `3000` | Lower for lower latency, higher to reduce load. |
| `LEDGER_BACKOFF_MS` | Base backoff between retries; grows exponentially (`backoff × 2^(attempt-1)`). | `2000` | Tune retry spacing. |

### Ledger DATABASE (separate, transactional — NOT the logs DB)

The outbox + audit are persisted to a **dedicated, transactional** database, entirely
**separate** from the [logs DB](#hybrid-db-logging-optional). When disabled, an
in-memory store is used (dev/test only — **not durable**). Create the tables from
`db/<engine>/ledger-schema.sql` first.

| Variable | Meaning | Default | When to change |
| --- | --- | --- | --- |
| `LEDGER_DB_ENABLED` | Persist the outbox + audit to a database. `false` = in-memory (not durable). | `false` | Set `true` for any real deployment. |
| `LEDGER_DB_TYPE` | Engine: `postgres`, `mysql`, or `mssql`. | `postgres` | Match your database. |
| `LEDGER_DB_HOST` | Database host. | `127.0.0.1` | Point at your DB server. |
| `LEDGER_DB_PORT` | Database port (postgres 5432, mysql 3306, mssql 1433). | `5432` | Match your engine/host. |
| `LEDGER_DB_USERNAME` | Database user. | `instapay` | Set your credentials. |
| `LEDGER_DB_PASSWORD` | Database password. | `change-me` | **Always change** for real use. |
| `LEDGER_DB_DATABASE` | Database name — **separate** from the logs database. | `instapay_ledger` | Set to your ledger database. |
| `LEDGER_DB_SCHEMA` | Schema name (postgres/mssql; ignored by mysql). | `public` | Match your schema. |
| `LEDGER_DB_SSL` | Use SSL to connect to the database. | `false` | Set `true` for remote/managed DBs. |

---

## Connecting to a CI or simulator

Out of the box there is **nothing to talk to**. To exchange real messages you need
one of:

- The **InstaPay ISO 20022 Simulator** (provided by BancNet during onboarding), or
- A **mock CI** you build that exposes the same endpoints (see the CI-facing
  endpoints in the [API Reference](05-api-reference.md)).

Then set `CI_BASE_URL` (and, when the endpoint is HTTPS, `TLS_ENABLED=true` with
the right certs) and, optionally, `AUTO_SIGN_ON=true`.

---

## Running in production (outline)

```bash
npm run build         # compile to dist/
npm run start:prod    # node dist/main
```

For production you must additionally: enable TLS (`TLS_ENABLED=true`) with
**CA-issued** certificates, set your real participant identity and `CI_BASE_URL`,
and complete BancNet onboarding + ITPC certification. See
[08 — Security & Compliance](08-security-and-compliance.md).

---

Next: **[03 — Architecture](03-architecture.md)** or
**[04 — Integration Guide](04-integration-guide.md)**.
