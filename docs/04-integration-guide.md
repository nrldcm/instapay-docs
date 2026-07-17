# 4. Integration Guide

For developers integrating an app with this service. It covers: originating a
payment, how the CI-facing endpoints behave, the ISO 20022 message types involved,
response codes, and using the internal **microservice** mode. See also the concise
[API Reference](05-api-reference.md) and the [Glossary](07-glossary.md).

---

## Two ways to talk to this service

| You are… | Use |
| --- | --- |
| **Your own app** wanting to send a payment | The internal **JSON API** (`POST /payments`) or **microservice** message patterns. |
| **The Central Infrastructure** (or simulator) | The **CI-facing ISO 20022 endpoints** under `/ips-payments` (raw signed XML). You normally don't call these yourself — InstaPay does. |

---

## Originating a payment (JSON API)

Send one HTTP request; the service builds and signs the `pacs.008`, submits it,
waits for the async result, and returns the outcome. The request body is defined by
`OriginatePaymentDto`.

### Request body

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `amount` | string | yes | Decimal string, e.g. `"1500.00"`. Pattern: up to 15 digits, optional up to 5 decimals. |
| `currency` | string | yes | 3-letter ISO 4217 code, e.g. `"PHP"`. |
| `debtor.name` | string | yes | Payer full name (≤140). |
| `debtor.id` | string | yes | Payer identifier (≤35). |
| `debtor.account` | string | yes | Payer account number (≤34). |
| `debtor.accountType` | string | no | Account type proprietary code (defaults to `CASH`). |
| `creditor.name` | string | yes | Payee full name (≤140). |
| `creditor.id` | string | yes | Payee identifier (≤35). |
| `creditor.account` | string | yes | Payee account number (≤34). |
| `creditor.agentBic` | string | yes | Payee bank **BIC** (creditor agent). Must be a valid BIC. |
| `remittanceInfo` | string | no | Unstructured remittance text (≤140). |
| `endToEndId` | string | no | Caller-supplied end-to-end id; generated if omitted (≤35). |

> The debtor's servicing agent is **your** participant BIC (`PARTICIPANT_BICFI`) —
> you don't supply it. Scheme codes (service level, local instrument, category
> purpose, settlement method/clearing system) use InstaPay defaults set in the
> `message builder`.

### Example: JSON body

```json
{
  "amount": "1500.00",
  "currency": "PHP",
  "debtor": {
    "name": "Juan Dela Cruz",
    "id": "DBTR001",
    "account": "00123456789",
    "accountType": "CASH"
  },
  "creditor": {
    "name": "Maria Santos",
    "id": "CDTR001",
    "account": "00987654321",
    "agentBic": "BDOMPHMMXXX"
  },
  "remittanceInfo": "Payment for invoice 123",
  "endToEndId": "E2E-0001"
}
```

### Example: curl

```bash
curl -X POST http://localhost:8443/payments \
  -H "Content-Type: application/json" \
  -d '{
        "amount": "1500.00",
        "currency": "PHP",
        "debtor":   { "name": "Juan Dela Cruz", "id": "DBTR001", "account": "00123456789", "accountType": "CASH" },
        "creditor": { "name": "Maria Santos",   "id": "CDTR001", "account": "00987654321", "agentBic": "BDOMPHMMXXX" },
        "remittanceInfo": "Payment for invoice 123",
        "endToEndId": "E2E-0001"
      }'
```

### Response

Returns `PaymentResultDto` with
HTTP `201`:

```json
{
  "instructionId": "INSTR8f2c...",
  "endToEndId": "E2E-0001",
  "transactionId": "TX3a91...",
  "state": "COMPLETED",
  "status": "ACTC",
  "reasonCode": null,
  "submissions": 1
}
```

**`state`** is the final outcome:

| `state` | Meaning |
| --- | --- |
| `COMPLETED` | The CI returned a `pacs.002` that was not a rejection (e.g. `ACTC`/`RCVD`). |
| `FAILED` | The CI returned a `pacs.002` with `RJCT` — see `reasonCode`. |
| `TIMED_OUT` | No async response within `RESPONSE_TIMEOUT_MS`, even after all `DUPL` resubmissions. |
| `REJECTED_AT_SUBMIT` | The CI did not accept the submission (did not return `202`). |

`status` is the ISO transaction status when resolved; `reasonCode` is the ISO reason
on rejection (e.g. `AC01`); `submissions` counts the original plus any duplicate
resubmissions.

> **Note on the caller experience:** this call is **synchronous from your side** —
> it blocks until the payment resolves or times out (up to ~`RESPONSE_TIMEOUT_MS`),
> even though InstaPay's own reply is asynchronous behind the scenes.

### Diagnostics

```bash
curl http://localhost:8443/payments/in-flight
# → { "pending": 0, "received": 0 }
```

Returns the count of payments currently awaiting a result (`pending`) and the number
of received inbound instructions remembered for cancellation matching (`received`).

---

## Sample payloads by destination

All of these go to the **same** endpoint — `POST /payments` with an
`OriginatePaymentDto` body.
**Only the `creditor` block changes** depending on where the money is going. The
service always builds the same signed `pacs.008`; InstaPay routes it by the
`creditor.agentBic` (the beneficiary institution's BIC).

> ⚠️ **The BICs below are ILLUSTRATIVE EXAMPLES.** Always confirm the exact
> `BICFI` for each participant against the **official InstaPay participant
> directory** issued by BancNet during onboarding before using them in any
> environment. Participant BICs and their InstaPay membership can change.

> 💡 **E-wallet accounts are mobile numbers.** For GCash, Maya, and most
> e-wallets, `creditor.account` is the beneficiary's **enrolled mobile number**
> (as registered with the wallet), not a bank account number. For banks,
> `creditor.account` is the **deposit account number**.

### GCash (e-wallet) — account = mobile number

Example agent BIC: **`GXCHPHM2XXX`** *(confirm against the participant directory).*

```json
{
  "amount": "500.00",
  "currency": "PHP",
  "debtor":   { "name": "Juan Dela Cruz", "id": "DBTR001", "account": "00123456789", "accountType": "CASH" },
  "creditor": { "name": "Maria Santos",   "id": "CDTR001", "account": "09171234567", "agentBic": "GXCHPHM2XXX" },
  "remittanceInfo": "Send to GCash",
  "endToEndId": "E2E-GCASH-0001"
}
```

### Maya (e-wallet) — account = mobile number

Example agent BIC: **`PYMYPHM1XXX`** *(confirm against the participant directory).*

```json
{
  "amount": "750.00",
  "currency": "PHP",
  "debtor":   { "name": "Juan Dela Cruz", "id": "DBTR001", "account": "00123456789", "accountType": "CASH" },
  "creditor": { "name": "Ana Reyes",      "id": "CDTR002", "account": "09209876543", "agentBic": "PYMYPHM1XXX" },
  "remittanceInfo": "Send to Maya",
  "endToEndId": "E2E-MAYA-0001"
}
```

### Bank (BDO) — account = deposit account number

Example agent BIC: **`BNORPHMMXXX`** *(confirm against the participant directory).*

```json
{
  "amount": "1500.00",
  "currency": "PHP",
  "debtor":   { "name": "Juan Dela Cruz", "id": "DBTR001", "account": "00123456789", "accountType": "CASH" },
  "creditor": { "name": "Maria Santos",   "id": "CDTR001", "account": "00987654321", "agentBic": "BNORPHMMXXX" },
  "remittanceInfo": "Payment for invoice 123",
  "endToEndId": "E2E-BANK-0001"
}
```

Other Philippine banks (swap `creditor.agentBic` — **examples, confirm each one**):

| Bank | Example `agentBic` |
| --- | --- |
| BDO | `BNORPHMMXXX` |
| BPI | `BOPIPHMMXXX` |
| Metrobank | `MBTCPHMMXXX` |
| Landbank | `TLBPPHMMXXX` |
| UnionBank | `UBPHPHMMXXX` |

### Credit / Debit CARD — NOT InstaPay (different rail)

> 🚫 **Pushing money to a bare card number is NOT an InstaPay transaction and is
> out of scope for this service.** Card "push" transfers ride a **different rail**
> — **Visa Direct**, **Mastercard Send**, or the **BancNet card switch** — with
> their own message formats, network, and settlement. This service speaks only the
> InstaPay ISO 20022 credit-transfer contract, which is **account-based** (bank
> account number or e-wallet mobile number).
>
> If a card is **linked to an e-wallet or bank account**, the card is merely an
> *access instrument* for that account. The InstaPay leg is still addressed to the
> underlying **account** (mobile number for a wallet, account number for a bank) via
> that institution's BIC — exactly like the examples above. There is no card
> number anywhere in the InstaPay payload.

---

## How the CI-facing endpoints behave

These live under `/ips-payments` in
`inbound.controller.ts`. They
consume/produce **`application/xml`** (a signed `<Message>`). Every request is
**signature- and schema-validated first**; failures become a signed `admi.002`
(structural) or `pacs.002 RJCT` (business) — see
[Architecture → error mapping](03-architecture.md#how-an-inbound-request-flows).

| Endpoint | Receives | We do | Reply |
| --- | --- | --- | --- |
| `POST /ips-payments/service-requests` | `pacs.008` (inbound credit transfer) | validate, credit beneficiary (stub), remember InstrId | **201** + signed `pacs.002` (`ACTC`) |
| `PUT /ips-payments/service-responses` | `pacs.002` (async result of a payment **we** sent) | match by `OrgnlInstrId`, complete the originator workflow | **204** |
| `PUT /ips-payments/payment-instructions` | `camt.056` (cancel) **or** `pacs.002` (confirm) | reverse a matched payment (camt.056) or acknowledge (pacs.002) | **200** + signed `pacs.002` (camt.056) / **204** (pacs.002) |
| `POST /ips-payments/system-notifications` | `admi.004` (system event) | log it | **204** |
| `PUT /ips-payments/health-checks` | `admn.005` (echo request) | build echo reply | **200** + signed `admn.006` |

---

## The ISO 20022 message types (one line each)

| Message | Direction | What it is |
| --- | --- | --- |
| **`pacs.008`** | both | FI-to-FI **customer credit transfer** — the actual payment instruction. |
| **`pacs.002`** | both | FI-to-FI **payment status report** — the result/acknowledgement of a `pacs.008` (status `ACTC`/`ACWP`/`RCVD`/`RJCT`). |
| **`camt.056`** | inbound | **Payment cancellation** request — asks us to cancel/reverse a received payment. |
| **`admi.002`** | outbound | **Message reject** — structural failure (bad parse / signature / schema). |
| **`admi.004`** | inbound | **System event notification** from the CI. |
| **`admn.001` / `admn.002`** | out / in | **Sign-on** request / response. |
| **`admn.003` / `admn.004`** | out / in | **Sign-off** request / response. |
| **`admn.005` / `admn.006`** | both | **Health-check echo** request / response. |

Exact message-definition identifiers and namespaces are in
`iso-namespaces.ts`.

---

## Response / HTTP codes at a glance

| Situation | HTTP | Body |
| --- | --- | --- |
| Inbound credit transfer accepted | 201 | signed `pacs.002` (`ACTC`) |
| Async result / confirmation / notification accepted | 204 | *(empty)* |
| Cancellation processed | 200 | signed `pacs.002` |
| Health-check echo | 200 | signed `admn.006` |
| Structural failure (parse/signature/schema) | 400 | signed `admi.002` |
| Business rejection | 400 | signed `pacs.002` (`RJCT`) |
| System / unexpected error | 500 | Error JSON |
| Originate accepted (JSON API) | 201 | `PaymentResultDto` |

---

## Microservice mode

Set `APP_MODE=microservice` (internal processor only, **CI endpoints OFF**) or
`APP_MODE=hybrid` (both). This exposes the **same origination capability** over an
internal transport (TCP by default; also NATS, Redis, RabbitMQ) so your own apps —
or partner EMIs fronting this as a shared processor — can call it without HTTP. It is
**not** the BancNet contract. Transport is configured in
`microservice-options.ts`; handlers in
`payments.message.controller.ts`.

### Message patterns

| Pattern | Payload | Returns |
| --- | --- | --- |
| `{ cmd: 'payments.originate' }` | `OriginatePaymentDto` (same shape as the JSON API) | `PaymentResultDto` |
| `{ cmd: 'payments.in-flight' }` | *(none)* | `{ pending, received }` |
| `{ cmd: 'health.ping' }` | *(none)* | `{ status: 'ok', ts }` |

### Client example (NestJS `ClientProxy`, TCP)

```ts
import { ClientProxyFactory, Transport } from '@nestjs/microservices';
import { firstValueFrom } from 'rxjs';

const client = ClientProxyFactory.create({
  transport: Transport.TCP,
  options: { host: '127.0.0.1', port: 8877 }, // matches MS_HOST / MS_PORT
});

const result = await firstValueFrom(
  client.send(
    { cmd: 'payments.originate' },
    {
      amount: '1500.00',
      currency: 'PHP',
      debtor:   { name: 'Juan Dela Cruz', id: 'DBTR001', account: '00123456789', accountType: 'CASH' },
      creditor: { name: 'Maria Santos',   id: 'CDTR001', account: '00987654321', agentBic: 'BDOMPHMMXXX' },
      remittanceInfo: 'Payment for invoice 123',
    },
  ),
);
console.log(result); // PaymentResultDto
```

For NATS/Redis/RabbitMQ, create the `ClientProxy` with the matching transport and
`MS_URL`. To secure the TCP transport, set `MS_TLS=true` (mutual TLS reusing the
participant certificate) and configure your client's `tlsOptions` accordingly.

---

Next: **[05 — API Reference](05-api-reference.md)** or, for message security,
**[08 — Security & Compliance](08-security-and-compliance.md)**.
