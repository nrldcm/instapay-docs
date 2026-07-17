# 09 — Extending Safely

> **In plain terms.** This service is deliberately a **thin layer** with clearly
> marked **plug points** where you connect *your* systems — your wallet/core, your
> databases, your extra channels — without rewriting the InstaPay plumbing. This
> page points at the **exact seams** and how to fill each one without breaking the
> money-safety guarantees.

Jargon: a **seam** is a designed extension point — usually an interface or a stub
method — you can replace behind a stable contract. **DI token** = the label NestJS
uses to inject one specific implementation.

---

## Seam 1 — Wire your real wallet / core (AccountService)

`account.service.ts` is the money seam.
Today `validateBeneficiary` is a **stub** that accepts everything, and crediting /
reversal go through the durable [ledger](05-ledger-money-safe.md).

| Method | Replace with |
| --- | --- |
| `validateBeneficiary(tx)` | A real account lookup. Return `{ ok:false, reasonCode:'AC01', message }` to reject (unknown account); `AC06` for a blocked account. Returning `{ ok:false, … }` makes the [ReceiverFlow](04-instapay-flows.md#flow-1--receiving-a-credit-transfer-we-are-the-creditor) throw an `IsoBusinessError`, which becomes a signed `pacs.002 RJCT`. |
| `creditBeneficiary(tx)` | Keep the `ledger.recordReceived` + `ledger.postCredit` calls so the credit stays money-safe; the actual posting happens in **your** ledger via [delivery](05-ledger-money-safe.md#delivery-transport--queue-or-http). |
| `reversePayment(evt)` | Keep `ledger.postReversal(evt)`; your ledger performs the reversal idempotently. |

**Rule:** never post money synchronously inside a request handler. Record it to the
outbox (as the code already does) and let the dispatcher deliver — that is what keeps
payments from being lost or double-booked.

---

## Seam 2 — Swap in the durable DB stores

The outbox and audit stores are pluggable behind the
[`OutboxStore` / `AuditStore` interfaces](05-ledger-money-safe.md#the-store-contract-the-important-seam)
and the DI tokens `OUTBOX_STORE` / `AUDIT_STORE`
(`ledger.types.ts`).

- **To turn on durability:** set `LEDGER_DB_ENABLED=true` and configure `LEDGER_DB_*`.
  The module factory in
  `ledger.module.ts` then binds the DB-backed
  `DbOutboxStore` / `DbAuditStore` instead of the in-memory ones.
- **To provide your own store** (e.g. a different engine): implement the two
  interfaces and bind them to the tokens in the module's `useFactory`. Nothing else
  changes — `LedgerService` and the dispatcher only know the interface.

> The concrete `src/ledger/db/*` implementations are being finalized separately; you
> extend against the **interface**, not their internals.

Similarly, the in-memory `InFlightStore`
and `TransactionJournal` sit behind
small classes so a Redis/Postgres version can be dropped in when you run more than one
instance.

---

## Seam 3 — Add a message pattern (microservice)

To expose new functionality over the internal transport, add a `@MessagePattern`
handler to a controller under `src/microservice/` — e.g.
alongside
`payments.message.controller.ts`.
Follow the existing convention `{ cmd: 'area.action' }` and return plain JSON. It
becomes available in `microservice` and `hybrid` modes automatically (the controllers
are already registered — payments/ledger via
`instapay.module.ts`, logs via
`logs.module.ts`). Mirror it with an HTTP route if
you also want it over the web API.

---

## Seam 4 — Add a new ISO 20022 message type

Work through the [toolkit](03-iso20022.md) in this order:

1. **Namespaces** — add the URI, `MSG_DEF` id, and choice-wrapper name in
   `iso-namespaces.ts`.
2. **Types** — add a params interface in
   `message.types.ts`.
3. **Build** — add a `buildXxx()` in
   `message.builder.ts` (emit the
   empty `head:Sgntr` placeholder — signing is automatic).
4. **Parse** — extend `detectKind` and add a `fillXxx()` in
   `message.parser.ts` so inbound
   copies are recognised and normalised into `ParsedMessage`.
5. **Schema** — ensure the message's XSD is bundled so
   `xsd.service.ts` can validate it.
6. **Route & handle** — add the controller endpoint
   (`inbound.controller.ts`) and a
   flow method to act on it.

Signing, verification, and XSD validation then apply to the new type for free.

---

## Seam 5 — Add a new run mode / transport

Transport selection lives in
`microservice-options.ts`; add a
branch there and expose the config in
`configuration.ts`. Run-mode branching is in
`main.ts` — see [07 — Runtime & Modes](07-runtime-and-modes.md).

---

## What NOT to change casually

- **Signing / c14n** — the RSA-SHA256 + c14n11 chain must byte-match what the CI
  expects; changing it breaks signature verification on both ends.
- **The `<Message>` namespace strategy** — the default-namespace redeclaration keeps
  output deterministic and signable.
- **The idempotency key** — delivery dedupe relies on `instructionId`; don't weaken
  the `UNIQUE (instruction_id, type)` constraint.
- **DDL files / DB internals** — schema is DDL-owned; coordinate changes with whoever
  finalizes `db/*` and `src/ledger/db/*`.

---

Next: **[10 — Error Handling](10-error-handling.md)** ·
Back to the **[index](00-index.md)**.
