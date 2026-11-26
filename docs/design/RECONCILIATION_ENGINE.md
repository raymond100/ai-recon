# Reconciliation Engine Design

Status: Proposed MVP design.

## Responsibility

Consume EventNormalized and make deterministic, tenant-scoped decisions against versioned pack rules and master data. Persist decision/rule outcomes and emit EventVerified or BreakDetected. This service has no LLM dependency.

## Input and output

Input: validated normalized values, field confidence/evidence, tenant security context, domain/pack/schema/rules versions, and stable document/correlation IDs.

Output:
- EventVerified for an automated pass with decision_origin=AUTO_PASS.
- BreakDetected with exception_id, stable violation codes, severity, rule IDs, evaluated fields/evidence, and decision timestamp.
- If extraction is low confidence, preserve LOW_CONFIDENCE_EXTRACTION as a reviewable exception; do not run financial rules that rely on uncertain required data.
- Terminal infrastructure/contract failure emits DocumentFailed and safe DLQ metadata.

## Deterministic evaluation

1. Resolve exact pinned pack/rules version; unavailable artifacts fail safely and never silently use latest.
2. Load tenant-scoped master snapshot and record its version/hash in the decision.
3. Validate required values and confidence. Overall score is minimum across pack-required fields. Optional confidence is nonblocking unless a rule explicitly requires it.
4. Freight MVP: lookup by PO first; if unavailable/unmatched, use BOL fallback per approved rule semantics. Preserve references distinctly. If both missing, return MISSING_SHIPMENT_REFERENCE.
5. Evaluate allowlisted YAML operators only. No eval, imports, arbitrary code, network calls, or model calls.
6. Use Decimal and ISO 4217 currency; compare with explicit currency/rule precision and tolerance. Preserve input precision. Apply tax/discount/credit semantics explicitly.
7. Persist immutable decision and per-rule outcomes with exact input/master/rule versions and evidence IDs. Emit outcome idempotently.

## Data ownership

Owns reconciliation decisions, per-rule outcomes, exception records at creation, master-data read model/snapshot references, and processing status transitions. It does not own extraction result generation, raw PDFs, or human resolution.

## Security and isolation

Tenant context comes only from the trusted event envelope. Every master lookup and persistence operation requires tenant_id. Cross-tenant IDs and payload tenant overrides are rejected. Consider PostgreSQL RLS as defense in depth after explicit tenant scoping is in place. Never log full invoice or evidence content.

## Reliability and observability

Idempotency key combines tenant_id + document_id + pack/rules version for evaluation; store unique constraints to block duplicate decisions. Outbox or reliable publication ensures event persistence and emitted outcome do not diverge. Metrics include rule pass/break counts, reason code rates, processing latency, master lookup misses, confidence-gate rates, replay count, and determinism drift.

## Acceptance boundary

Ready when master snapshot/version semantics, freight lookup fallback behavior, severity levels, money tolerances, low-confidence routing, and event publication strategy are approved. Acceptance covers duplicate events, tenant isolation, missing master/reference, version mismatch, Decimal edge cases, and same-input/same-version deterministic results.
