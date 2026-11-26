# Triage and Ops Service Design

Status: Proposed MVP design.

## Responsibility

Provide the tenant-scoped exception queue, evidence/rule explanation, role-checked human resolution, append-only override audit, and secure webhook delivery boundary. It does not change the automated decision record.

## API surface

- GET /exceptions?status=&domain_id=&severity=: tenant-scoped filtered queue.
- GET /exceptions/{exception_id}: rule outcomes, safe explanation, referenced field/evidence records. Raw PDF retrieval is a separate authorized endpoint and OPS_ANALYST+.
- POST /exceptions/{exception_id}/resolve: decision APPROVE or REJECT, resolution_note, idempotency_key. Requires authorization based on severity and role.
- POST /master-records/{domain_id}/upsert is owned by the master-data API boundary and authorized only for TENANT_ADMIN/OPS_MANAGER; see project story/design. It is included here only if the deployment keeps a single Ops API process.

## Role policy

INGEST_CLIENT may upload only. OPS_ANALYST can review and resolve WARNING. OPS_MANAGER/TENANT_ADMIN may override BLOCKING with a non-empty note. Every operation checks tenant scope and effective role server-side. Never infer authorization from caller-provided role fields.

Severity is a deterministic ruleset output. Human resolution may approve/override or reject but does not delete/rewrite BreakDetected or the original decision.

## Resolution and events

Approval appends immutable APPROVED_OVERRIDE audit data and emits BreakOverridden, then emits a distinct EventVerified with decision_origin=HUMAN_OVERRIDE referencing the original decision and override. Automated EventVerified is tagged AUTO_PASS. Rejection emits terminal ExceptionRejected and never enters delivery outbox. Repeated same idempotency key returns the original result; conflicting terminal requests are rejected and audited.

## Webhook delivery

Only eligible EventVerified outcomes enter the durable outbox. Tenant administrator configures HTTPS endpoint and secret. Dispatcher resolves DNS and blocks loopback, private, link-local, and metadata ranges at connection time; redirects are disabled or each destination is fully revalidated. Sign exact request bytes with HMAC-SHA256 in X-Reconcile-Signature: sha256=<digest>. Treat any 2xx within five seconds as success; make at most three attempts at 10s, 60s, and 300s. Persist attempt status, response class, timestamp, and safe error; never store response bodies unnecessarily. Secret rotation/revocation remains an open operational detail.

## Data ownership

Owns exception queue projection, human resolutions, actor/audit records, tenant master-data admin API, endpoint configuration references, delivery outbox and attempts, and reviewer-facing explanation. Reconciliation receives a read-only master-data repository interface; it does not own master-data mutation endpoints. Audit records are append-only; routine DB role cannot update/delete audit rows.

## Security and observability

Evidence and exceptions are tenant-scoped and role-checked. Avoid full quote/response payloads in logs. Webhook secrets are encrypted or externalized, never returned by APIs, and access is restricted. Metrics: queue age/count by severity, resolution latency, override/rejection rate, delivery success/retries/failures, and signature/SSRF rejections.

## Acceptance boundary

Ready when severity taxonomy, role matrix, second-person override policy, exception status model, endpoint configuration/secret rotation, SSRF defense, webhook success/retry contract, and EventVerified compatibility are approved. Acceptance covers cross-tenant denial, role denial, immutable decision audit, idempotent resolution, rejected-not-delivered, human override delivered with origin, DNS/redirect attacks, signature verification, retries, and terminal delivery failure.
