# InstaPay ISO 20022 Integration Service — Documentation

Public documentation for a **private, proprietary** InstaPay (BancNet / Mastercard
IPS) integration service for the Philippines. An EMI or bank connects it to
InstaPay to **send and receive real-time PHP credit transfers**: integrators use a
simple JSON API, and the service builds, signs, and validates the ISO 20022 XML
for the BancNet Central Infrastructure (CI) internally.

> **Note:** this repository contains **documentation only**. The source code is
> private and proprietary — see [LICENSE](LICENSE). References to source files
> (e.g. `receiver.flow.ts`) describe the private codebase.

## What the service does

- **Originate payments (cash-out)** — one JSON `POST /payments`; the service
  handles pacs.008 build + XMLDSig signing, submission over mutual TLS, async
  pacs.002 matching, timeout + DUPL resubmission.
- **Receive payments (cash-in)** — validates signature/schema, credits via a
  money-safe outbox, replies with a signed pacs.002.
- **Cancellations / reversals** — camt.056 matching and reversal.
- **Money-safe ledger delivery** — durable outbox + append-only audit trail +
  retrying dispatcher with idempotency; delivers every CREDIT / DEBIT / REVERSAL
  to your core ledger over a queue or HTTP.
- **Operations** — health probes, rotating + optional DB logging with query APIs,
  three run modes (api / microservice / hybrid), DB auto-scaffolding
  (Postgres / MySQL / MSSQL).

## Documentation

### Product & operations (`docs/`)

| Doc | Audience | Contents |
| --- | --- | --- |
| [01 — Overview](docs/01-overview.md) | Everyone | Plain-language explanation, diagrams |
| [02 — Setup](docs/02-setup.md) | Operators | Install, certs, every environment variable |
| [03 — Architecture](docs/03-architecture.md) | Developers | Module map, message envelope, flow sequence diagrams |
| [04 — Integration Guide](docs/04-integration-guide.md) | Integrators | Originate payments, message types, microservice mode |
| [05 — API Reference](docs/05-api-reference.md) | Integrators | Every HTTP endpoint & message pattern |
| [06 — Logging](docs/06-logging.md) | Operators / Devs | File layout, levels, optional DB logging |
| [07 — Glossary](docs/07-glossary.md) | Everyone | Plain-language definitions of every acronym |
| [08 — Security & Compliance](docs/08-security-and-compliance.md) | Everyone | TLS, signing, keys, ITPC path to production |
| [09 — Ledger & Audit](docs/09-ledger-and-audit.md) | Operators / Devs | Money-safe outbox → dispatcher → ledger, audit trail |
| [10 — Transaction Flows](docs/10-transaction-flows.md) | Everyone | End-to-end **cash-in / cash-out / reversal** diagrams, state lifecycles |

### Guided tour of the codebase (`docs/code/`)

A page-by-page tour of how the (private) source is organized — start at the
[index](docs/code/00-index.md). Highlights:

- [01 — Big Picture](docs/code/01-big-picture.md) — the journey of money, non-technical.
- [04 — InstaPay Flows](docs/code/04-instapay-flows.md) — receive / originate / cancel, sequence diagrams.
- [05 — Ledger & Money-Safe Delivery](docs/code/05-ledger-money-safe.md) — outbox, audit, dispatcher.
- [10 — Error Handling](docs/code/10-error-handling.md) — signed admi.002 / pacs.002 RJCT / JSON.
- [11 — Request Walkthroughs](docs/code/11-request-walkthroughs.md) — **every endpoint, step by step**: where a request goes next through the code.

### API surface

The full OpenAPI spec (as exported from the running service) is at
[openapi/openapi.json](openapi/openapi.json).

## License

The documentation and the service it describes are proprietary — see
[LICENSE](LICENSE). No permission is granted to use, copy, or implement the
described system without a written agreement.
