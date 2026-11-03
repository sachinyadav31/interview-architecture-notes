# 09: Contracts and DocuSign e-signing

## What it does
Contracts (created manually or from an RFQ/RFP award) go through approval, then get signed electronically by **both buyer and supplier signees** through DocuSign. The signed PDF is stored back in document management.

Services: `ep-contract-service` (contract lifecycle), `ep-e-signing-service` (DocuSign integration, per-tenant DocuSign settings), `ep-document-management-service` (files).

## Flow
```mermaid
sequenceDiagram
  participant C as contract-service
  participant E as e-signing-service
  participant D as DocuSign
  participant U as Signee (buyer/supplier)
  C->>C: contract approved (workflow)
  C->>E: Feign / event: sign request {entityId, signees[], documentId}
  E->>E: load tenant DocusignSettings (account id, integration key, base URL)
  E->>E: get access token (JWT grant with the integration key + RSA key)
  E->>D: EnvelopesApi.createEnvelope(accountId, EnvelopeDefinition)<br/>document (base64 PDF) + recipients (signers with routing order) + status "sent"
  D-->>E: envelopeId
  E->>D: createRecipientView(RecipientViewRequest returnUrl) for embedded signing
  D-->>U: signing URL (or e-mail)
  U->>D: signs
  D->>E: callback (whitelisted /entity-sign/update-contract-from-docusign) or polling
  E->>E: hooks: RecipientSignedHook ("recipient-completed"): mark signee completed
  E->>C: Kafka sign-complete-topic (all signed) | sign-decline-topic
  C->>C: contract status ACTIVE / SIGNED; store signed PDF
```

## Low-level points
- **EnvelopeDefinition** = email subject (entity ID) + `Documents` (base64 PDF, fileExtension, documentId) + `Recipients.signers` (email, name, recipientId, **routingOrder**, so the buyer signs first and the supplier second, or both in parallel) + `status = "sent"` (a `"created"` envelope would be a draft).
- **Embedded signing**: `RecipientViewRequest` (returnUrl, authenticationMethod, clientUserId) → a URL to open inside our portal instead of DocuSign e-mail.
- **Hooks** (`hooks/RecipientSignedHook` etc.): a small **Chain/Strategy** of handlers per DocuSign action (`recipient-completed`, envelope completed, declined).
- `EntitySignCommandController` only acts when the status is `signing_complete`; other events are ignored.
- Kafka topics: `sign-complete-topic`, `sign-decline-topic`, `re-initiate-sign-topic`, `signee-updated-topic`, `client/supplier-signee-assigned-topic`.
- Per-tenant `DocusignSettings` entity (repository + transformer), so each tenant connects its own DocuSign account.

## Weak spots and how I'd improve them
- Callback endpoint is whitelisted (no JWT). It **must verify DocuSign Connect HMAC signatures**; otherwise anyone can mark a contract signed.
- Envelope creation + DB update should be idempotent (store `envelopeId`; on retry, don't create a second envelope).
- Access token: cache until expiry instead of fetching per call.

## Interview Q&A

**Q: How did you integrate e-signature?**
A: Through the DocuSign eSignature REST SDK. We create an envelope with the PDF and signers (routing order decides sequence), send it, optionally create an embedded recipient view so users sign inside our portal, and receive completion through DocuSign Connect callbacks. Completion is published on Kafka so the contract service updates status asynchronously.

**Q: How do you authenticate to DocuSign?**
A: OAuth **JWT Grant** (service integration): the integration key plus an RSA private key sign a JWT asserting a user ID, and we exchange it for an access token. Consent is given once. No user passwords.

**Q: How do you secure the callback?**
A: Enable HMAC on DocuSign Connect and verify the `X-DocuSign-Signature-1` header against the shared secret over the raw body. Reject anything that doesn't match. Even then, re-fetch the envelope status from the DocuSign API before trusting it.

**Q: Why put e-signing in its own service?**
A: Vendor isolation (we could swap DocuSign for another provider behind the same events), separate credentials, and contracts aren't the only thing that gets signed.
