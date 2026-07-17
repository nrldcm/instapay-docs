# 7. Glossary

Plain-language definitions of the acronyms and terms used across this project and
the InstaPay world. Alphabetical within groups.

---

## Organisations & schemes

| Term | Meaning |
| --- | --- |
| **BSP** | *Bangko Sentral ng Pilipinas* — the central bank of the Philippines. It licenses EMIs and banks and oversees the payment system. |
| **EMI** | *Electronic Money Issuer* — a BSP-licensed institution that issues e-money (e.g. e-wallets). Needs a licence to operate. |
| **BancNet** | The operator of InstaPay and the national ATM/switch network in the Philippines. Runs the **Central Infrastructure**. |
| **Mastercard / Vocalink** | Provider of the **IPS** technology platform on which InstaPay runs. |
| **InstaPay** | The Philippines' real-time, 24/7 retail electronic fund-transfer service. One of two clearing houses under the NRPS. |
| **NRPS** | *National Retail Payment System* — the BSP framework InstaPay operates under. |
| **IPS** | *Instant Payment System* — the underlying real-time payments platform (Mastercard/Vocalink) used by InstaPay. |
| **CI** | *Central Infrastructure* — BancNet's central system that receives, routes, and clears messages between participants. This service sends to and receives from the CI. |
| **Participant** | An institution connected to InstaPay that can send/receive payments. Each has an ID, a BIC, and an IIN. |
| **ITPC** | *Industry Testing & Product Certification* — BancNet's mandatory test/certification you must pass before going live. |
| **Simulator** | The InstaPay ISO 20022 Simulator — a test system that behaves like the CI, used before onboarding/certification. |

## Identifiers

| Term | Meaning |
| --- | --- |
| **BIC / BICFI** | *Bank Identifier Code* (a.k.a. SWIFT code) — 8 or 11 characters identifying a financial institution, e.g. `BDOMPHMMXXX`. `BICFI` is the ISO 20022 element name for it. |
| **IIN** | *Issuer Identification Number* — a numeric identifier assigned to a participant institution. |
| **Participant ID** | The identifier assigned to your institution during onboarding, used in the sign-on/off URL. |
| **Instruction Id (InstrId)** | A per-payment identifier. Used as the **matching key** between a sent `pacs.008` and its asynchronous `pacs.002` result. |
| **End-to-End Id (EndToEndId)** | A reference that travels end to end with the payment; can be supplied by the originator. |
| **Transaction Id (TxId)** | Another per-transaction identifier carried in the payment. |
| **BizMsgIdr** | *Business Message Identifier* — a unique id for each message, set in the BAH. |
| **MsgDefIdr** | *Message Definition Identifier* — names which message type a BAH carries, e.g. `pacs.008.001.08`. |

## Standards & message types

| Term | Meaning |
| --- | --- |
| **ISO 20022** | The global standard for financial messaging (XML-based). InstaPay uses it for all messages. |
| **BAH** | *Business Application Header* (`head.001.001.01`) — the "envelope header" carrying from/to, ids, timestamp, and the digital signature. |
| **`<Message>` envelope** | The InstaPay container (namespace `urn:instapay2019`) that wraps the BAH plus one business document. |
| **pacs.008** | FI-to-FI **customer credit transfer** — the actual payment instruction. |
| **pacs.002** | FI-to-FI **payment status report** — the result/acknowledgement of a payment. |
| **camt.056** | **Payment cancellation** request — asks a participant to cancel/reverse a payment. |
| **admi.002** | **Message reject** — sent when a message fails structural checks (parse/signature/schema). |
| **admi.004** | **System event notification** from the CI. |
| **admn.001 / admn.002** | **Sign-on** request / response. |
| **admn.003 / admn.004** | **Sign-off** request / response. |
| **admn.005 / admn.006** | **Health-check echo** request / response. |

## Status & reason codes

Every code you can meet in a message, a reject, or an API response — what it
means and **when this service produces or receives it**.

### Transaction status codes (`TxSts` in `pacs.002`)

The InstaPay `pacs.002` schema restricts the transaction status to exactly these
four values:

| Code | Name | Meaning | When you see it |
| --- | --- | --- | --- |
| **ACTC** | *AcceptedTechnicalValidation* | The payment passed validation and was accepted — InstaPay's "success". | We reply `ACTC` for an accepted inbound payment and an acknowledged cancellation; a `pacs.002 ACTC` for a payment we sent means it **completed**. |
| **ACWP** | *AcceptedWithoutPosting* | Accepted, but not yet posted to the beneficiary account. | May arrive as the async result of a payment we originated. |
| **RCVD** | *Received* | The message was received and is still in process. | Interim state; the in-flight store records it if no final status is given. |
| **RJCT** | *Rejected* | The payment was refused — the **reason code** says why. | We send `RJCT` when beneficiary validation fails; receiving `RJCT` marks our originated payment `FAILED`. |

### Structural reject reason codes (in `admi.002`)

Sent (HTTP 400) when an inbound message fails the validation gate **before** any
business logic — see the [inbound gate](code/11-request-walkthroughs.md#cash-in--post-ips-paymentsservice-requests-the-network-delivers-money).

| Code | Meaning | Raised by |
| --- | --- | --- |
| **DU01** | Message is **unparseable** — not well-formed XML / not a recognized `<Message>` envelope. | the message parser |
| **DS02** | **Signature validation failed** — the XMLDSig in `<head:Sgntr>` is missing, altered, or signed by the wrong key. | the signature verifier |
| **DU02** | **Schema validation failed** — parses fine but violates the InstaPay XSDs (missing/invalid element). | the XSD validator |

### Business reject reason codes (in `pacs.002 RJCT`)

ISO *ExternalStatusReason* codes explaining **why** a syntactically valid payment
was refused. The ones this service uses / expects most:

| Code | Meaning | When |
| --- | --- | --- |
| **AC01** | *IncorrectAccountNumber* — the account is unknown/incorrect. | Default reject when your `validateBeneficiary` hook says the account doesn't exist. |
| **AC06** | *BlockedAccount* — the account exists but is blocked. | Return it from `validateBeneficiary` for frozen/blocked wallets. |
| **AM04** | *InsufficientFunds* — not enough balance to settle. | Typically arrives on a `RJCT` for a payment we originated. |

> The full ISO list is much longer (AB, AC, AG, AM, BE, RC, TM, … families); any
> code your core returns in the `BeneficiaryCheck.reasonCode` is passed through
> verbatim into the signed `pacs.002`.

### System error codes (JSON `Errors.Error[].ReasonCode`)

Unexpected faults return **HTTP 500 + JSON** (never XML), shaped per Appendix B:

| Code | Meaning |
| --- | --- |
| **SYS01** | Generic/unclassified system error (the default for an unexpected exception). |
| **SYS…** | Any `IsoSystemError` thrown with its own code passes through (e.g. a DB outage); `Recoverable: true/false` tells the caller whether a retry can help. |

### Journal statuses (`GET /payments` → `status`)

The readable lifecycle of a transaction in the reconciliation feed:

| Status | Direction | Meaning |
| --- | --- | --- |
| **PENDING** | outbound | Originated, submitted, still awaiting the async result. |
| **RECEIVED** | inbound | An inbound payment arrived and is being credited. |
| **COMPLETED** | both | Final success — outbound got `ACTC/ACWP/RCVD`; inbound was credited & ACKed. |
| **FAILED** | both | Final failure — outbound got `RJCT` (or was rejected at submit); inbound failed beneficiary validation. `reasonCode` says why. |
| **TIMED_OUT** | outbound | No `pacs.002` arrived within the SLA after all DUPL resubmissions. |
| **REVERSED** | inbound | A matched `camt.056` cancellation reversed the credit. |

### Ledger outbox & audit codes (`GET /ledger/outbox`, `GET /audit`)

| Code | Where | Meaning |
| --- | --- | --- |
| **CREDIT / DEBIT / REVERSAL** | outbox `type` | The money-event type: credit the beneficiary (cash-in), debit for a sent payment (cash-out), or undo a credit (cancellation). |
| **PENDING** | outbox `status` | Awaiting delivery to your core ledger (or between retries). |
| **DELIVERED** | outbox `status` | Successfully delivered — terminal. |
| **DEAD** | outbox `status` | Gave up after `LEDGER_MAX_ATTEMPTS` — kept for reconciliation, never deleted. |
| **RECEIVED** | audit `action` | Proof an inbound payment arrived (recorded before we ACK). |
| **ENQUEUED** | audit `action` | A money event was written to the durable outbox. |
| **DELIVERY_ATTEMPT / DELIVERY_OK / DELIVERY_FAILED** | audit `action` | One delivery try and its outcome. |
| **DEAD_LETTER** | audit `action` | The event was moved to DEAD after exhausting retries. |

### Other codes & flags

| Term | Meaning |
| --- | --- |
| **DUPL** | The *duplicate* flag (`CpyDplct` in the BAH) set when resubmitting the **same** instruction after a timeout — tells the network "same payment, don't double-process". |
| **731** | The echo function code (`FnctnCd`) carried in `admn.005`/`admn.006` health-check messages. |
| **Id prefixes** (`INSTR…`, `E2E…`, `TX…`, `MSG…`, `GRP…`, `ECHO…`, `SON…`, `SOF…`) | Generated identifiers: **INSTR** instruction id, **E2E** end-to-end id, **TX** transaction id, **MSG** business message id (BAH), **GRP** group header id, **ECHO** health-check, **SON/SOF** sign-on/sign-off. |
| **SDVA** | The service-level code (`SvcLvl`) used on InstaPay credit transfers (same-day value). |
| **SLEV** | Charge bearer *FollowingServiceLevel* — charges applied per the scheme's service level. |

## Roles in a payment

| Term | Meaning |
| --- | --- |
| **Originator / Debtor** | The party (and its bank) **sending** the money. When your app calls `POST /payments`, you are the originator. |
| **Receiver / Creditor** | The party (and its bank) **receiving** the money. When a `pacs.008` arrives on `service-requests`, you are the receiver. |
| **Debtor Agent / Creditor Agent** | The banks (BICs) servicing the debtor and creditor accounts. |
| **Beneficiary** | The end recipient of the funds (the creditor's customer). |

## Security & technical terms

| Term | Meaning |
| --- | --- |
| **TLS** | *Transport Layer Security* — encryption for data in transit (the "S" in HTTPS). |
| **Mutual TLS (mTLS)** | TLS where **both** sides present and verify certificates, so each end proves its identity. |
| **XMLDSig** | *XML Digital Signature* — the W3C standard used to digitally sign each message so the recipient can verify it wasn't altered and came from you. |
| **RSA-SHA256** | The cryptographic algorithms used for the signature (RSA) and the digest (SHA-256). |
| **Enveloped signature** | A signature that sits **inside** the document it signs (here, in the BAH `<Sgntr>` element). |
| **Canonicalization (C14N / C14N11)** | A step that puts XML into a single, standard byte-for-byte form before signing/verifying, so trivial formatting differences don't break the signature. |
| **XSD schema** | A formal definition of a valid message's structure. Every message is validated against the bundled InstaPay XSDs. |
| **CA** | *Certificate Authority* — the trusted issuer of digital certificates. Production certs come from a BancNet-approved CA. |
| **IPSEC VPN** | An encrypted private network tunnel used to reach BancNet's infrastructure in production. |
| **NDA** | *Non-Disclosure Agreement* — signed during onboarding; the InstaPay specifications are confidential. |
| **Stub** | A placeholder implementation (here, beneficiary crediting) that you replace with real logic later. |
| **In-flight** | A payment that has been sent and is awaiting its asynchronous result. |
