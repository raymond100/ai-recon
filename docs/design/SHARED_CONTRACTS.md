# Shared Contracts and Event Flow

Status: Proposed; canonical contract for service design review.

## Contract rules

- Events are immutable facts. A new event records a new fact; consumers do not rewrite prior events.
- All tenant identity comes from trusted SecurityContext resolved at ingress. A payload tenant_id is never authoritative.
- Producers assign event_id and correlation_id. Consumers are idempotent by event_id and relevant business key.
- Documents pin domain_id, pack_version, schema_version, and ruleset_version as they become known. Historical replay loads those exact versions or fails explicitly if unavailable.
- Money is Decimal with explicit ISO 4217 currency. Dates retain semantic qualifiers and source precision.
- Every extracted field retains raw value, normalized value, confidence, evidence_id, quote, and page when available.
- Event schema version and pack version are separate: envelope compatibility is independent of domain behavior.

## Base envelope

Required: event_id, event_type, schema_version, occurred_at, correlation_id, retry_count, tenant_id, security_context, payload. Optional: causation_id and producer metadata.

SecurityContext contains tenant_id, actor_id, actor_type, and role(s), established by the ingestion/auth boundary and propagated internally. It must not contain API secrets or raw credentials.

## Event inventory

| Event | Producer | Purpose |
| --- | --- | --- |
| DocumentUploaded | Ingestion Gateway | Raw document accepted; includes document_id, sha256, storage_ref, domain_id, pack_version, submitted_at |
| EventNormalized | Extraction Worker | Validated domain payload; includes schema/pack versions, field confidence/evidence and overall_confidence |
| EventVerified | Reconciliation Engine or Triage & Ops | A decision is eligible for downstream use; includes decision_id, decision_origin=AUTO_PASS or HUMAN_OVERRIDE, rule outcomes and evidence references |
| BreakDetected | Reconciliation Engine | Automated evaluation found rule violations; includes exception_id, decision_id, severity, rule IDs, evidence references |
| BreakOverridden | Triage & Ops | Authorized human override; references original decision and break, resolution note and actor |
| ExceptionRejected | Triage & Ops | Authorized reviewer rejected the exception; terminal, no downstream delivery |
| DocumentFailed | Any processing stage | Terminal processing failure; safe error_code, stage, failed_at, document_id |

Terminal failures also write a safe dead-letter envelope to stream:dead_letter. DocumentFailed is the durable domain event that drives FAILED_PROCESSING state; it is not a substitute for diagnostic DLQ metadata. Do not put document contents, evidence quotes, credentials, or stack traces in DLQ records.

## State model

Document: RECEIVED → EXTRACTION_PENDING → EXTRACTING → NORMALIZED → RECONCILING → VERIFIED or EXCEPTION; terminal processing failure → FAILED_PROCESSING.

Exception: OPEN → APPROVED_OVERRIDE or REJECTED. Resolution is idempotent. Conflicting terminal transitions fail and are audited.

Delivery is a separate state machine: NOT_ELIGIBLE → PENDING → DELIVERING → DELIVERED, RETRYING, or FAILED. A human override preserves the original BreakDetected and automated decision; it creates BreakOverridden and a distinct EventVerified tagged HUMAN_OVERRIDE. Automated passes are AUTO_PASS. Both may be delivered; rejected exceptions may not.

## Redis Streams processing

Each consumer group validates envelope and payload version, claims/retries work according to bounded policy, commits durable state/outbox changes, and acknowledges only after commit. After three unsuccessful attempts, write to the DLQ and emit DocumentFailed. Use transactional/outbox or an equivalent idempotent publication strategy where DB commits and Redis publication must remain consistent.

## Compatibility and replay

Additive optional fields are preferred. Removing/renaming fields or changing semantics requires a schema version bump and rollout plan. Replay must preserve original event identity or record replay causation, pin original pack/rules versions, avoid duplicate decisions/delivery, and record who initiated replay and why.

## Open decisions

- Exact stream names and partition/consumer-group topology.
- Whether a transactional outbox is used for every database-to-stream publication or only externally visible effects.
- Retention/archival duration for events, decisions, and pack artifacts.
- Whether SecurityContext carries a single effective role or a role set.
