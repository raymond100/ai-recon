# Extraction Worker Design

Status: Proposed MVP design.

## Responsibility

Consume DocumentUploaded, obtain the tenant-authorized stored PDF, extract text/layout, call the configured structured extraction adapter for the pinned pack, validate normalized output, attach field confidence/evidence, and publish EventNormalized. It never evaluates business rules or authorizes tenants/domains.

## Processing contract

Input: DocumentUploaded with document_id, storage_ref, domain_id, pack_version, correlation_id, and trusted SecurityContext.

Output: EventNormalized with document_id, domain_id, pack_version, schema_version, ruleset_version if selected, required-field confidence, overall_confidence, and typed normalized fields. Each field carries raw_value, normalized_value, confidence in [0,1], evidence {quote, page}, and evidence_id.

Overall confidence is min(confidence[field] for field in pack.required_fields). If any required value is missing or confidence < 0.85, publish a normalized result tagged LOW_CONFIDENCE_EXTRACTION (or a typed exception event as contract review chooses) and do not imply the document is verified. Optional absent/low-confidence fields do not block unless a deterministic rule requires them.

## Processing sequence

1. Validate event version, tenant context presence, document state, and pinned pack artifact availability.
2. Retrieve the raw object through tenant-scoped authorization. Verify content hash and recheck limits if needed.
3. Extract text/layout using a bounded local parser; preserve page references and source spans where feasible.
4. Invoke a provider adapter with only the minimum required context and untrusted document data. Require structured output matching the pinned schema.
5. Validate types, enum/code normalization, Decimal parsing, dates/qualifiers, evidence quote/page, and field confidence. Preserve raw values; never coerce money through float.
6. Persist extraction result and emit EventNormalized idempotently. Ack input after durable output.
7. On retryable errors, retry up to three attempts; terminal errors emit DocumentFailed and safe DLQ metadata.

## LLM boundary

The model may propose field values, classifications, and short source evidence. It cannot set tenant, actor, role, pack version, rules, severity, or final decision. Treat PDF text as hostile, delimit it as data, constrain output to schema, reject invented evidence, and avoid placing tool instructions or credentials in model context. Provider/model/prompt/schema versions and token/time budgets are recorded for reproducibility.

## Data ownership

Owns extraction results, field-level confidence/evidence metadata, provider execution metadata (without secrets), and extraction stage status. It does not own master records, final decisions, raw object lifecycle, or reviewer actions.

## Security and privacy

Raw PDF retrieval is tenant-scoped and role-checked; worker identity is service-authenticated. Evidence quotes may contain sensitive content: return only to authorized Ops users, redact logs, and apply the agreed evidence retention/deletion policy. Raw files expire at 30 days; extracted metadata retention is a separate decision.

## Reliability and observability

Consumer-group idempotency by event_id/document_id + pinned version. Metrics: stage latency, provider latency, token use, schema validation failures, field confidence distribution, retry/DLQ count, and extraction quality on synthetic fixtures. Logs include safe error codes and correlation/document IDs, not prompts, full PDF text, or evidence content.

## Acceptance boundary

Ready when provider interface, prompt/schema versioning, confidence output semantics, evidence persistence/access, and low-confidence event behavior are agreed. Tests/acceptance cover pass, missing required field, optional low confidence, invalid schema, injected malicious PDF text, provider timeout, retry exhaustion, and replay.
