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

| Term | Meaning |
| --- | --- |
| **ACTC** | *AcceptedTechnicalValidation* — the payment was accepted (InstaPay "success"). |
| **ACWP** | *AcceptedWithoutPosting* — accepted, not yet posted to the account. |
| **RCVD** | *Received* — the message was received and is in process. |
| **RJCT** | *Rejected* — the payment was refused (see the reason code). |

> The InstaPay `pacs.002` schema restricts the transaction status (`TxSts`) to exactly these four values: `ACTC`, `ACWP`, `RCVD`, `RJCT`.
| **Reason code** | An ISO code explaining a rejection, e.g. `AC01` (incorrect/unknown account), `AC06` (blocked), `AM04` (insufficient funds). |
| **DUPL** | The *duplicate* flag (`CpyDplct`) set when resubmitting the same instruction after a timeout. |

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
