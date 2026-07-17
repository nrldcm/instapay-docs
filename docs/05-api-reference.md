# 5. API Reference

A concise reference of every HTTP endpoint and microservice message pattern.

> **The live source of truth** is the Swagger UI at **`/docs`**, with the machine
> spec at **`/docs-json`** (OpenAPI JSON) and **`/docs-yaml`** (YAML). On boot the
> spec is also exported to **`openapi/openapi.json`** (configurable via
> `OPENAPI_EXPORT` / `OPENAPI_DIR`). Use those for exact schemas and code generation.

All HTTP endpoints are available in `api` and `hybrid` modes. In `microservice`
mode there is **no HTTP server** — only the message patterns below.

> **Want the journey, not just the signature?**
> [10 — Transaction Flows](10-transaction-flows.md) has the end-to-end cash-in /
> cash-out / reversal diagrams, and
> [code/11 — Request Walkthroughs](code/11-request-walkthroughs.md) traces every
> endpoint step by step through the source files.

---

## CI-facing ISO 20022 endpoints — `/ips-payments`

Called by the Central Infrastructure (or simulator). Bodies are a **signed
`<Message>`** envelope. Source:
`inbound.controller.ts`.

| Method | Path | Purpose | Consumes | Produces | Success | Error |
| --- | --- | --- | --- | --- | --- | --- |
| `POST` | `/ips-payments/service-requests` | Receive an inbound `pacs.008` credit transfer; credit beneficiary; reply with `pacs.002`. | `application/xml` | `application/xml` | **201** (`pacs.002 ACTC`) | 400 `admi.002`/`pacs.002 RJCT`, 500 JSON |
| `PUT` | `/ips-payments/service-responses` | Receive the async `pacs.002` result of a payment we originated; match by `OrgnlInstrId`. | `application/xml` | — | **204** | 400 `admi.002` |
| `PUT` | `/ips-payments/payment-instructions` | `camt.056` cancellation (reverse + reply) **or** `pacs.002` confirmation (ack). | `application/xml` | `application/xml` | **200** (`pacs.002`) for camt.056; **204** for pacs.002 | 400, 500 |
| `POST` | `/ips-payments/system-notifications` | Receive a CI system event (`admi.004`). | `application/xml` | — | **204** | 400 `admi.002` |
| `PUT` | `/ips-payments/health-checks` | Health-check echo (`admn.005` → `admn.006`). | `application/xml` | `application/xml` | **200** (`admn.006`) | 400 `admi.002` |

---

## Internal JSON API — `/payments`

Called by **your** applications to originate payments. Source:
`payments.controller.ts`.

| Method | Path | Purpose | Consumes | Produces | Success |
| --- | --- | --- | --- | --- | --- |
| `POST` | `/payments` | Originate an outbound credit transfer. Body = `OriginatePaymentDto`. Service reconstructs JSON → signed pacs.008 XML internally. | `application/json` | `application/json` | **201** — `PaymentResultDto` |
| `GET` | `/payments` | List the JSON transaction journal (inbound + outbound). Filters: `since`, `status`, `direction`, `limit`, `offset`. The counterpart service's reconciliation cron polls this. | — | `application/json` | **200** — `{ count, transactions[] }` |
| `GET` | `/payments/:instructionId` | Get one transaction (JSON) by Instruction Id. | — | `application/json` | **200** / **404** |
| `GET` | `/payments/in-flight` | Diagnostics: `{ pending, received }` counters. | — | `application/json` | **200** |

> **JSON-only API principle:** your applications and integrators never send or receive
> XML. All integrator endpoints (`/payments`, `/health`, `/`) are JSON. The ISO 20022
> **XML is reconstructed internally** only when talking to the BancNet CI; the
> `/ips-payments/*` endpoints below are the **Scheme adapter** — used solely by BancNet,
> not by your apps.

See the [Integration Guide](04-integration-guide.md#originating-a-payment-json-api)
for request/response examples.

## Ledger & audit (JSON) — `/audit`, `/ledger/outbox`

Read-only JSON views over the **money-safe** subsystem: the audit trail (what came
in / what was posted) and the durable outbox (what is pending or dead-lettered).
A reconciliation cron on the counterpart service polls these. Source:
`ledger.controller.ts`. See
[09 — Ledger & Audit](09-ledger-and-audit.md) for the full design.

| Method | Path | Purpose | Query filters | Produces | Success |
| --- | --- | --- | --- | --- | --- |
| `GET` | `/audit` | Audit trail: every `RECEIVED`, `ENQUEUED`, `DELIVERY_ATTEMPT`, `DELIVERY_OK`, `DELIVERY_FAILED`, `DEAD_LETTER` event. Verify exactly what arrived and what was posted. | `instructionId`, `action`, `since` (ISO timestamp), `limit` | `application/json` | **200** — `{ count, entries[] }` |
| `GET` | `/ledger/outbox` | Durable outbox of money events awaiting or failed delivery to the ledger. | `status` (`PENDING` \| `DELIVERED` \| `DEAD`), `limit` | `application/json` | **200** — `{ count, events[] }` |

> Backed by the in-memory stores by default, or the **dedicated, transactional
> ledger DB** when `LEDGER_DB_ENABLED=true` (separate from the logs DB).

## Logs query (JSON) — `/logs`

Read the persisted logs (hybrid DB log sink). Requires `LOG_DB_ENABLED=true`; otherwise
returns `{ count: 0, logs: [], note }`. Source:
`logs.controller.ts`.

| Method | Path | Purpose | Query filters | Produces | Success |
| --- | --- | --- | --- | --- | --- |
| `GET` | `/logs` | Fast, indexed query over the partitioned log table. | `from`, `to` (ISO ts), `level`, `instructionId`, `correlationId`, `bizMsgIdr`, `limit`, `offset` | `application/json` | **200** — `{ count, logs[] }` |

## Health probes — `/health`

For load balancers, Docker healthchecks, and Kubernetes probes. Source:
`health.controller.ts`.

| Method | Path | Purpose | Produces | Success |
| --- | --- | --- | --- | --- |
| `GET` | `/` | Service info (name, mode, participant, doc links) as JSON. | `application/json` | **200** |
| `GET` | `/health` | Liveness — process is up. Returns `{ status, mode, uptimeSec }`. | `application/json` | **200** |
| `GET` | `/health/ready` | Readiness — ready to serve. Returns `{ status, inFlight }`. | `application/json` | **200** |

> **Response format:** the integrator-facing routes above (and `/payments`) always
> respond in **JSON** — including errors (a bad body → `400` JSON, an unknown path →
> `404` JSON). Only the BancNet-facing `/ips-payments/*` endpoints exchange ISO 20022
> **XML**, as required by the Scheme.

### Validation errors

The JSON API uses a global validation pipe with `whitelist` + `forbidNonWhitelisted`
+ `transform`. A malformed body (missing required field, bad BIC, non-decimal amount,
unknown property) returns **HTTP 400** with a standard NestJS validation error body.

---

## Documentation endpoints

| Method | Path | Purpose |
| --- | --- | --- |
| `GET` | `/docs` | Swagger UI (interactive). |
| `GET` | `/docs-json` | OpenAPI spec as JSON. |
| `GET` | `/docs-yaml` | OpenAPI spec as YAML. |

---

## Microservice message patterns

Available in `microservice` / `hybrid` mode. Source:
`payments.message.controller.ts`.

| Pattern | Payload | Returns |
| --- | --- | --- |
| `{ cmd: 'payments.originate' }` | `OriginatePaymentDto` | `PaymentResultDto` |
| `{ cmd: 'payments.in-flight' }` | — | `{ pending, received }` |
| `{ cmd: 'health.ping' }` | — | `{ status: 'ok', ts }` |
| `{ cmd: 'ledger.transactions' }` | `{ since?, status?, direction?, limit?, offset? }` | `{ count, transactions[] }` |
| `{ cmd: 'ledger.transaction.get' }` | `{ instructionId }` | transaction record or `null` |
| `{ cmd: 'ledger.audit' }` | `{ instructionId?, action?, since?, limit? }` | `{ count, entries[] }` |
| `{ cmd: 'ledger.outbox' }` | `{ status?, limit? }` | `{ count, events[] }` |
| `{ cmd: 'ledger.in-flight' }` | — | `{ pending, received }` |
| `{ cmd: 'logs.query' }` | `{ from?, to?, level?, instructionId?, correlationId?, bizMsgIdr?, limit?, offset? }` | `{ count, logs[] }` |

Ledger/audit/logs query patterns are the message-queue equivalent of the HTTP
`GET /audit`, `GET /ledger/outbox`, `GET /payments`, and `GET /logs` reads — a
counterpart service can reconcile over the queue instead of HTTP. Source:
`ledger.message.controller.ts`,
`logs.message.controller.ts`.

Transport (TCP/NATS/Redis/RMQ), host/port/URL, and TLS are set by the `MS_*`
environment variables — see [Setup](02-setup.md#internal-microservice-transport) and
the [Integration Guide](04-integration-guide.md#microservice-mode).

---

## Outbound calls we make to the CI

Not endpoints *we* expose — for completeness, these are the CI endpoints this
service **consumes** via `ips.client.ts`
(base URL from `CI_BASE_URL`):

| Method | Path (on the CI) | Purpose | Expected |
| --- | --- | --- | --- |
| `POST` | `/ips-payments/service-requests` | Submit an originated `pacs.008`. | `202` accepted |
| `PUT` | `/ips-payments/participants/{participantId}` | Sign-on (`admn.001`) / sign-off (`admn.003`). | `200` |
| `PUT` | `/ips-payments/health-checks` | Health-check echo (`admn.005`). | `200` (`admn.006`) |
