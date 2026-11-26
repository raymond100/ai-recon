# Ingestion Gateway Design

Status: Proposed MVP design.

## Responsibility

Authenticate API clients, authorize the requested domain for the tenant, validate and store uploaded PDFs, enforce idempotency, and publish DocumentUploaded. The gateway does not parse invoice business fields or decide reconciliation outcomes.

## API

### POST /documents

Request: multipart PDF plus domain_id and idempotency key (optional when tenant-scoped SHA-256 idempotency is used).

Success: 202 Accepted with document_id, correlation_id, lifecycle status, and accepted domain/pack version.

Failure behavior:
- 401 for missing/invalid API key.
- 403 when requested domain is not in tenant.allowed_domains.
- 413 when file exceeds 15 MB or 25 pages.
- 415 for non-PDF or unrecognized/invalid PDF.
- 409 only for an idempotency-key conflict with a different file; same-tenant same-hash resubmission returns the prior document result.

### GET /documents/{document_id}/raw

Authenticated, tenant-scoped retrieval; OPS_ANALYST+ only. Use short-lived authorized download flow or stream via the service. Never return a public or long-lived object URL.

## Processing sequence

1. Resolve API key to trusted tenant, actor, actor type, and role; redact the key immediately.
2. Validate allowed_domains and resolve the active immutable pack artifact/version.
3. Stream the body to enforce byte limit without buffering the whole file. Validate PDF signature and page count in a bounded parser.
4. Compute SHA-256 while streaming. Check tenant-scoped idempotency.
5. Store the raw object encrypted at rest with 30-day TTL; persist metadata and document state atomically where practical.
6. Write DocumentUploaded to the stream/outbox with domain and pack version pinned. Return stable document ID.

## Data ownership

Owns API client credential mapping (hashed keys), tenant allowed-domain configuration read model, document metadata/state at intake, idempotency records, and raw object references. Raw bytes live in object storage or local development storage, not PostgreSQL. Other services access metadata through contracts/authorized APIs, not direct ingestion tables.

## Security and privacy

- Tenant and actor context are server-derived; ignore/reject client-supplied tenant identity.
- Apply request timeout, rate limits, content-length and streaming limits, safe MIME/signature checks, and parser resource limits.
- Store no raw PDF text, API key, or document bytes in logs/events.
- Raw retention is 30 days; deletion process records object deletion result and metadata cleanup. Legal hold is out of MVP; retention is tenant-wide default.
- Every raw download is authorized and auditable; enforce the same tenant scope in object retrieval.

## Reliability and observability

Use idempotent intake keyed by tenant + content hash, durable event publication, bounded retries, and a recoverable outbox if storage and event publication can split. Metrics: upload accepted/rejected counts, size/page distributions, dedupe hits, storage errors, event publication delay, and intake latency. Logs carry document_id/correlation_id and safe error codes only.

## Acceptance boundary

Ready for implementation when API/auth contracts, active-pack resolution, PDF parser limits, storage/outbox transaction strategy, and raw deletion behavior are decided. Acceptance: tenant mismatch is rejected, limits fail before extraction, duplicate uploads converge to one document, and no raw content/secrets escape into logs or events.
