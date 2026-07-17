# 3. Architecture

Technical design of the service. Audience: developers and integrators. Acronyms are
defined in the [Glossary](07-glossary.md).

The project is a **NestJS + TypeScript** application. It is organised into a small
number of modules with a clean separation between the **transport-agnostic ISO
20022 toolkit** and the **InstaPay business flows**.

---

## Module map

```mermaid
flowchart TD
    subgraph entry["Entry point"]
        MAIN["main.ts\n(boot: api / microservice / hybrid)"]
    end

    subgraph config["config/"]
        CFG["configuration.ts\nconfig.module.ts\n(typed env config)"]
    end

    subgraph iso["iso20022/  (transport-agnostic toolkit)"]
        XML["xml/xml.service.ts\n(build & parse XML)"]
        MSG["messages/\nmessage.builder.ts\nmessage.parser.ts\nmessage.types.ts"]
        SIGN["signing/\nsign.service.ts\nverify.service.ts\nc14n11.ts"]
        VALID["validation/\nxsd.service.ts\n+ bundled XSD schemas"]
        NS["iso-namespaces.ts"]
    end

    subgraph instapay["instapay/  (business integration)"]
        INB["inbound/\ninbound.controller.ts (CI-facing)\npayments.controller.ts (our JSON API)\ninbound-validation.service.ts"]
        OUT["outbound/ips.client.ts\n(calls the CI)"]
        FLOWS["flows/\noriginator / receiver / cancellation"]
        STATE["state/inflight.store.ts\n(in-memory)"]
        ACCT["account.service.ts\n(STUB ledger hook)"]
    end

    subgraph common["common/"]
        FILT["filters/iso-exception.filter.ts"]
        ERR["errors/iso-errors.ts"]
        LIFE["lifecycle/lifecycle.service.ts\n(sign-on/off, heartbeat)"]
        LOG["logging/\nlogger.config.ts\ndb/ (optional DB sink)"]
        ID["id.util.ts"]
    end

    subgraph micro["microservice/"]
        MSOPT["microservice-options.ts"]
        MSCTRL["payments.message.controller.ts"]
    end

    MAIN --> config
    MAIN --> LOG
    MAIN --> micro
    instapay --> iso
    INB --> FLOWS
    FLOWS --> iso
    FLOWS --> STATE
    FLOWS --> ACCT
    OUT --> config
    FILT --> iso
    LIFE --> OUT
    MSCTRL --> FLOWS
```

### What each area is responsible for

| Area | Responsibility |
| --- | --- |
| **`config/`** | Loads and strongly types every environment variable (`configuration.ts`). Nothing else touches `process.env`. |
| **`iso20022/xml/`** | Deterministic XML build/parse (`xml.service.ts`) — order-preserving so output is stable and signable; namespace-prefix-insensitive lookups on parse. |
| **`iso20022/messages/`** | `message.builder.ts` constructs each message type; `message.parser.ts` normalises inbound messages for routing/matching; `message.types.ts` holds the typed parameters and the `TxStatus` enum. |
| **`iso20022/signing/`** | `sign.service.ts` inserts an enveloped XMLDSig signature into the BAH; `verify.service.ts` validates it; `c14n11.ts` registers the C14N 1.1 transform. |
| **`iso20022/validation/`** | `xsd.service.ts` validates messages against the bundled InstaPay XSD schemas using `xmllint-wasm` (WebAssembly libxml2 — no native build). |
| **`instapay/inbound/`** | The **CI-facing** ISO 20022 endpoints (`inbound.controller.ts`), the internal **JSON** origination API (`payments.controller.ts`), and the first-line signature+schema gate (`inbound-validation.service.ts`). |
| **`instapay/outbound/`** | `ips.client.ts` — the HTTP client that calls the CI (submit payment, sign-on/off, health check). Never throws on non-2xx; returns `{ status, body }`. |
| **`instapay/flows/`** | The business logic: **originator** (send), **receiver** (accept), **cancellation** (reverse). |
| **`instapay/state/`** | `inflight.store.ts` — **in-memory** tracking of payments awaiting an async result, keyed by Instruction Id. Behind a small interface so a Redis/Postgres store can be dropped in later. |
| **`instapay/account.service.ts`** | The **stub** ledger hook (`validateBeneficiary`, `creditBeneficiary`, `reversePayment`). Accepts everything by default — replace with your ledger. |
| **`common/filters` + `errors`** | Typed ISO errors and the global filter that renders them as the correct signed response (see below). |
| **`common/lifecycle/`** | Sign-on at boot, sign-off at shutdown, optional health-check heartbeat. |
| **`common/logging/`** | Winston logger config (rotating files by date + level) and the optional DB sink. See [06 — Logging](06-logging.md). |
| **`microservice/`** | The internal transport options and the message-pattern controller mirroring the JSON origination API. |

---

## The message envelope

Every message on the wire is an InstaPay **`<Message>`** envelope
(`targetNamespace urn:instapay2019`) that wraps **two** things:

1. a **Business Application Header** (BAH, `head.001.001.01`) — who it's from/to,
   the message id, the message-definition id, timestamp, a duplicate flag, and the
   **digital signature**; and
2. exactly **one business document** (e.g. `pacs.008`, `pacs.002`), placed under a
   named choice element (`CreditTransfer`, `MessageStatusReport`, `EchoRequest`, …).

```mermaid
flowchart TD
    M["&lt;Message&gt;  (urn:instapay2019)"]
    M --> H["&lt;AppHdr&gt; — BAH (head.001)"]
    M --> C["&lt;CreditTransfer&gt; / &lt;MessageStatusReport&gt; / … (named choice)"]
    H --> FR["Fr / To  (BICs)"]
    H --> IDR["BizMsgIdr, MsgDefIdr, CreDt"]
    H --> DUP["CpyDplct = DUPL  (resubmissions only)"]
    H --> SIG["Sgntr → &lt;Signature&gt;  (XMLDSig, filled by SignService)"]
    C --> DOC["business document root\n(in its own ISO namespace)"]
```

**Namespace strategy** (see `message.builder.ts`
and `iso-namespaces.ts`): the envelope, header,
and choice wrapper are in the default `instapay2019` namespace; BAH children use the
`head:` prefix; each business document root **redeclares** the default `xmlns` to its
own ISO namespace so its children inherit it prefix-free. This keeps the output
deterministic and reliably signable.

**Signing** (see `sign.service.ts`): the
BAH is built with an **empty `<Sgntr/>`** placeholder. `SignService` computes a W3C
XMLDSig **enveloped** signature over the whole document (`Reference URI=""`) with an
`enveloped-signature` + `xml-c14n11` transform chain, `rsa-sha256` signature and
`sha256` digest, and appends the `<Signature>` inside `<Sgntr>`. Algorithm URIs live
in `SIG_ALGO` in `iso-namespaces.ts`. Details:
[08 — Security & Compliance](08-security-and-compliance.md#message-signing-xmldsig).

---

## How an inbound request flows

Every inbound message passes through the same first-line gate before any business
logic — signature verification, then XSD validation, then parse
(`inbound-validation.service.ts`).
Failures become signed ISO error responses via the global exception filter
(`iso-exception.filter.ts`):

| Error type | Cause | Response |
| --- | --- | --- |
| `IsoStructuralError` | Parse (`DU01`), signature (`DS02`), or schema (`DU02`) failure | HTTP **400** + signed **`admi.002`** |
| `IsoBusinessError` | Valid message, business rejection | HTTP **400** + signed **`pacs.002`** with `RJCT` |
| `IsoSystemError` / other | Unexpected/system fault | HTTP **500** + Error JSON |

### (a) Receiving a credit transfer (we are the creditor)

```mermaid
sequenceDiagram
    participant CI as Central Infrastructure
    participant IC as InboundController
    participant V as InboundValidationService
    participant R as ReceiverFlow
    participant A as AccountService (stub)
    participant B as MessageBuilder + SignService

    CI->>IC: POST /ips-payments/service-requests (signed pacs.008)
    IC->>V: accept(xml)
    V->>V: verify signature → XSD validate → parse
    alt structural failure
        V-->>IC: throws IsoStructuralError
        IC-->>CI: 400 + signed admi.002
    else valid
        V-->>IC: ParsedMessage
        IC->>R: handleCreditTransfer(parsed)
        R->>A: validateBeneficiary()
        alt business reject
            A-->>R: not ok
            R-->>CI: 400 + signed pacs.002 (RJCT)
        else accepted
            R->>A: creditBeneficiary() (stub)
            R->>R: recordReceived(InstrId)
            R->>B: build + sign pacs.002 (ACTC)
            R-->>IC: { httpStatus 201, xml }
            IC-->>CI: 201 + signed pacs.002 (ACTC)
        end
    end
```

### (b) Originating a payment (we are the debtor) with the async `pacs.002` match

This is the most important flow. The reply from the CI to our submission is
**asynchronous**: submitting returns `202`, and the real outcome arrives **later** on
our `service-responses` endpoint, matched by **Instruction Id**.

```mermaid
sequenceDiagram
    participant App as Your app
    participant PC as PaymentsController
    participant O as OriginatorFlow
    participant S as InFlightStore
    participant CL as IpsClient
    participant CI as Central Infrastructure
    participant IC as InboundController

    App->>PC: POST /payments (JSON)
    PC->>O: originate(dto)
    O->>S: add(tx) — state PENDING (keyed by InstrId)
    O->>CL: submit signed pacs.008
    CL->>CI: POST /ips-payments/service-requests
    CI-->>CL: 202 Accepted
    Note over O: wait up to RESPONSE_TIMEOUT_MS
    CI->>IC: PUT /ips-payments/service-responses (async signed pacs.002)
    IC->>O: handleServiceResponse(parsed)
    O->>S: resolve(OrgnlInstrId, status)
    S-->>O: unblocks the waiter
    O-->>PC: PaymentResultDto (COMPLETED / FAILED)
    PC-->>App: 201 + result

    Note over O,CI: On timeout: resubmit SAME InstrId with CpyDplct=DUPL,<br/>up to MAX_RESUBMISSIONS, then return TIMED_OUT
```

Outcome states returned to the caller: `COMPLETED`, `FAILED` (`pacs.002` = `RJCT`),
`TIMED_OUT` (no response within the SLA after all resubmissions), or
`REJECTED_AT_SUBMIT` (the CI did not return `202`).

### (c) Cancellation (we are the receiver being asked to reverse)

```mermaid
sequenceDiagram
    participant CI as Central Infrastructure
    participant IC as InboundController
    participant V as InboundValidationService
    participant CF as CancellationFlow
    participant S as InFlightStore
    participant A as AccountService (stub)

    CI->>IC: PUT /ips-payments/payment-instructions (signed camt.056)
    IC->>V: accept(xml) → verify/validate/parse
    V-->>IC: ParsedMessage (kind = PaymentCancellation)
    IC->>CF: handleCancellation(parsed)
    CF->>S: wasReceived(OrgnlInstrId)?
    alt matched
        CF->>A: reversePayment(InstrId) (stub)
    else unknown
        Note over CF: acknowledge with no action
    end
    CF-->>IC: { httpStatus 200, signed pacs.002 (ACTC) }
    IC-->>CI: 200 + signed pacs.002
```

> Note: a `pacs.002` **confirmation** arriving on `payment-instructions` (not a
> `camt.056`) is a fire-and-forget acknowledgement returning **204**.

---

## In-flight state & idempotency notes

- Originated payments live in `InFlightStore`
  keyed by **Instruction Id**; the async `pacs.002` is matched on its
  `OrgnlInstrId`. Received instruction ids are remembered separately so a later
  `camt.056` cancellation can be matched.
- State is **in memory**: it is not shared across processes and does not survive a
  restart. This matches the integration-only scope; swap in a shared store when you
  add a ledger.

---

Next: **[04 — Integration Guide](04-integration-guide.md)** or
**[05 — API Reference](05-api-reference.md)**.
