# AI Reconciliation Platform — Product and Delivery Plan

Status: Draft for planning sign-off  
Owner: Project maintainers  
Last updated: 2026-11-26

## 1. Product summary

An open-source, industry-agnostic platform that turns business documents into tenant-scoped normalized records, checks them against authoritative master data using deterministic rules, and routes breaks to human operators with an auditable path to downstream delivery.

LLMs are limited to unstructured PDF extraction and plain-language exception explanations. Deterministic Python and declarative YAML decide financial, date, and compliance outcomes.

## 2. Problem and users

Organizations receive inconsistent PDFs and must reconcile them against contracts, purchase orders, vendor records, project dates, and budgets. Manual review is slow, rules can be applied inconsistently, and decisions require evidence and auditability.

- API client: submits a document and selects a domain pack.
- Operations analyst: reviews exceptions, evidence, and explanations.
- Authorized reviewer: approves or rejects an exception with a required note.
- Tenant administrator/integrator: manages master data and downstream delivery.
- Developer/pack author: operates the platform or adds a domain pack.

## 3. Outcomes and provisional pilot measures

The MVP demonstrates that a document is submitted once and traced across services; extracted facts retain confidence and evidence; identical inputs and versions produce identical outcomes; tenant data stays isolated; decisions are attributable and auditable; and exceptions can be reviewed and delivered downstream.

Track these pilot measures. The numeric values are initial hypotheses for a small synthetic/golden test corpus, not production guarantees; revise them after a representative pilot baseline:

- Auto-verify rate: report by domain and rule cohort; no universal target until the corpus mix is known.
- Required-field extraction accuracy: at least 95% exact match on a curated golden set per domain, with the set and field-level scoring documented.
- Median operator time to resolve a standard exception: under 5 minutes in the guided demo/pilot workflow.
- Downstream mock webhook delivery success: at least 99% within the configured retry window.
- Local mocked end-to-end latency, upload to terminal decision: p95 under 60 seconds.

Also instrument field confidence distribution, exception rate by rule, stage latency, retry/DLQ rate, and delivery attempts. Segment results; do not hide low-confidence cases inside aggregate accuracy.

## 4. Scope

### MVP
- Python 3.12+, uv, FastAPI, Pydantic v2, PostgreSQL 16, Redis Streams, Docker Compose.
- Ingestion gateway, extraction worker, deterministic reconciliation engine, and triage service.
- Versioned shared events, correlation IDs, immutable security context.
- Freight invoice and vendor compliance packs.
- Tenant-scoped master data and append-only decision/resolution audit.
- Human resolution API and approved-outcome webhook flow; local mock receiver is sufficient.
- Freight-pack MVP is scoped to LTL/TL; tenant-authorized domain selection; deterministic upload limits and raw retention defaults.
- CI, unit/integration tests, synthetic fixtures, and an end-to-end demo.
- Seeded master data plus a minimal tenant-scoped CLI/import or admin API upsert path for pilot corrections; no full CRUD UI.

### Deferred
External identity provider (including Keycloak), cloud hosting/managed infrastructure, multi-region operation, general-purpose rules UI or tenant-supplied executable rules, additional domain packs, broad analytics, autonomous approvals, and a full master-data CRUD UI.

## 5. User journeys

### Submit and reconcile
1. Client submits a PDF and domain_id with X-API-Key.
2. Gateway authenticates, derives immutable SecurityContext, verifies requested domain_id against the tenant's allowed_domains (mismatch fails with 403), validates PDF limits (15 MB and 25 pages), computes SHA-256, stores the raw document with a 30-day TTL, and returns a stable document ID. Domain and pack version are pinned at ingestion.
3. Gateway emits DocumentUploaded with pinned domain/pack version. Extraction parses text and returns schema-validated facts, each with field confidence and evidence.
4. Reconciliation applies the confidence gate: any required field below 0.85 produces LOW_CONFIDENCE_EXTRACTION before business rules. Otherwise it loads tenant-scoped master data and evaluates deterministic rules.
5. Passing records emit EventVerified; violations emit BreakDetected. Both decisions and rule versions are audited.

### Review and resolve
1. Analyst lists tenant-scoped exceptions with failed rules, referenced evidence, and a concise explanation.
2. Authorized reviewer approves or rejects with a non-empty resolution note.
3. Resolution is an explicit human override of the automated decision; it does not rewrite the original evaluation. Actor, decision, note, and timestamp are appended to the audit trail.
4. Approval emits ManuallyApproved and makes the outcome eligible for downstream delivery. Rejection emits ExceptionRejected and does not trigger delivery. A future correct-and-rerun workflow is out of MVP scope.

### Recover from failure
Transient worker errors retry with bounded attempts. After three failed attempts, poison messages go to stream:dead_letter with correlation and safe error metadata. Failures never silently become verified outcomes.

## 6. Functional requirements

- FR-1 Authentication: API key maps to tenant, actor, actor type, and role; payload tenant IDs cannot override trusted context.
- FR-2 Idempotency: duplicate file hash uploads within a tenant do not create duplicate processing outcomes.
- FR-3 Extraction: tenant configuration is authoritative for allowed domains; a client-requested domain outside allowed_domains fails fast with 403 and cannot dynamically load a pack. Pin domain and pack version at ingestion. Every required extracted field carries value, confidence, and structured evidence.
- FR-3a Upload policy: PDF <= 15 MB and <= 25 pages; reject oversized input with 413 before extraction. Raw PDF TTL is 30 days; evidence metadata/quotes are retained under tenant-scoped access and configured privacy policy.
- FR-4 Confidence: every required field has confidence in [0,1]. overall_confidence is the minimum confidence among required fields. If any required field or overall_confidence is below 0.85, emit LOW_CONFIDENCE_EXTRACTION and do not run business rules. Optional-field confidence is retained but does not independently block reconciliation unless a rule requires that field.
- FR-5 Evidence: each extracted field has structured evidence {quote, page} plus evidence_id; rule results reference the evidence used. Restrict raw-PDF download to OPS_ANALYST+ through authenticated, tenant-scoped access; never expose storage references as public URLs. Raw PDFs expire after 30 days, with deletion policy and audit records defined.
- FR-6 Determinism: same normalized payload, master-data snapshot, and rules version yield the same decision.
- FR-7 Isolation: every database lookup and mutation is scoped by trusted tenant_id.
- FR-8 Audit: record correlation, schema/pack/rules versions, rule outcomes, evidence references, actor actions, and timestamps in append-only records. Tenant-admin/master-data upserts retain before/after snapshots.
- FR-9 Triage: exception retrieval is tenant-scoped and supports status/domain filters.
- FR-10 Resolution: enforce role authorization, mandatory note, explicit state transitions, and idempotent request behavior.
- FR-11 Delivery: only approved outcomes are eligible; use idempotency keys and record attempts.
- FR-12 Resilience: bounded retries and DLQ behavior are explicit; unsupported event versions fail safely.
- FR-13 Packs/master data: domain behavior is pack-configured; tenant configuration governs allowed domains. MVP master data supports idempotent POST /master-records/{domain_id}/upsert for TENANT_ADMIN/OPS_MANAGER; `make seed` uses the same path/semantics. Each upsert records MASTER_DATA_UPSERTED with before/after JSONB snapshots.

## 7. Non-functional requirements

- Security: least privilege, no secrets in source/logs, tenant isolation by construction and tests.
- Determinism: Decimal for money; explicit currency, rounding, and date semantics; ruleset version recorded.
- Reliability: idempotent consumers and side effects; bounded retries; observable DLQ.
- Traceability: propagate correlation_id through API, events, persistence, logs, and callbacks.
- Maintainability: service isolation, strict typing, lint/format, clear interfaces.
- Privacy: synthetic demo documents only; avoid raw document text and credentials in logs; raw PDFs have a 30-day TTL, authenticated tenant-scoped access, and deletion audit.

## 8. Architecture and boundaries

Client → Ingestion Gateway → Redis Streams → Extraction Worker → Redis Streams → Reconciliation Engine ↔ PostgreSQL → Triage/Ops → approved downstream webhook. Raw PDFs are stored separately. PostgreSQL JSONB supports flexible master records and payloads where appropriate.

- ingestion-gateway: HTTP auth, security context, document validation/storage, hash idempotency, upload event.
- extraction-worker: PDF parsing, pack schema selection, structured LLM call, field confidence/evidence, retry/DLQ.
- reconciliation-engine: deterministic evaluation, master lookup, audit, verified/break events. No LLM imports or dependencies.
- triage-service: exception API, explanation, human resolution, approval/rejection event, downstream delivery boundary.
- shared: versioned contracts, pack definitions, rule loading/validation, reusable types.

Services communicate asynchronously and never import another service implementation. Event changes are additive when possible; breaking changes require a new schema version and rollout plan.

## 9. Domain packs

### Freight invoice pack — proposed MVP extraction fields

See [Freight Invoice Field Research](FREIGHT_FIELD_RESEARCH.md) for source review and the proposed LTL/TL field profile. Core financial identity is invoice number/date, total amount, and ISO 4217 currency; carrier must resolve to a tenant carrier identity, preferring SCAC when present. Use `po_number` as the required primary lookup key and `bol_number` as the fallback; if both are absent, emit `MISSING_SHIPMENT_REFERENCE`. Preserve shipment ID, PRO, load, and other references separately. Keep parties and locations role-based, and dates qualified by business meaning.

Represent charges as an extensible repeated line-item collection containing source code/description, normalized category, amount/currency, and optional quantity, unit, rate, tax, discount, or credit attributes. Preserve unknown codes, raw values, confidence, and structured evidence. The simplified LTL/TL summary may expose `base_freight`, `fuel_surcharge`, `accessorials[]`, `tax_amount`, and `total_amount`, all Decimal. Do not assume every invoice contains each category or force all extracted amounts to two decimal places before reconciliation. These are product-pack requirements, not universal X12 requirements; do not claim full EDI-210 compliance without a separate mapping review.

Rules: require PO match, falling back to BOL match; carrier and available charge categories are compared against tenant master/rate data. Reconcile line extensions and invoice totals using Decimal with explicit currency precision/tolerance and tax, credit, and discount semantics.

### Vendor compliance pack

Fields: vendor identity, policy type/number, effective/expiration dates, GL coverage, project end date, work order, committed/remaining budget, currency, confidence, and field evidence.

Rules: policy valid through project end; GL coverage at least $2M; work order does not exceed tenant-scoped remaining budget.

## 10. Frozen event and state contract proposal — requires maintainer sign-off

Common envelope, schema version 1.0: event_id, event_type, occurred_at, correlation_id, retry_count, tenant_id, immutable security_context, payload, and producer metadata. Consumers validate version/type before processing. Security context is established at ingestion and propagated; downstream services do not accept payload tenant overrides.

Initial events and payload intent:
- DocumentUploaded: document_id, sha256, storage_ref, domain_id, submitted_at.
- EventNormalized: document_id, domain_id, schema_version, ruleset_version requested/active, overall_confidence, normalized fields; each field has value, confidence, source_quote, and evidence_id.
- EventVerified: document_id, decision_id, ruleset_version, evaluated_at, rule outcomes, evidence references.
- BreakDetected: document_id, exception_id, decision_id, violation codes, rule IDs, field/evidence references, detected_at.
- BreakOverridden: exception_id, original_decision_id, resolution_id, reviewer identity/role, severity, resolution_note, resolved_at, idempotency key. This is an immutable human override record; it never deletes or rewrites BreakDetected.
- EventVerified: for automated passes, includes document_id, decision_id, ruleset/pack version and evaluated_at. For an approved override, emit a separate verification outcome referencing BreakOverridden and the original decision; preserve decision_origin=AUTO_PASS|HUMAN_OVERRIDE so downstream consumers distinguish the paths.
- ExceptionRejected: exception_id, decision_id, resolution_id, reviewer identity/role, resolution_note, resolved_at. Does not trigger downstream delivery.
- DocumentFailed: document_id, error_code, stage, safe reason, failed_at. Emit after terminal processing failure and write corresponding message to stream:dead_letter; update document state to FAILED_PROCESSING.

Initial document lifecycle: RECEIVED → EXTRACTION_PENDING → EXTRACTING → NORMALIZED → RECONCILING → VERIFIED or EXCEPTION; exhausted processing failures end in PROCESSING_FAILED with a DLQ reference. Exception lifecycle: OPEN → APPROVED or REJECTED (terminal in MVP). Delivery lifecycle is separate: NOT_ELIGIBLE or PENDING → DELIVERING → DELIVERED, RETRYING, or FAILED. Repeated resolution requests with the same idempotency key return the original result; conflicting terminal transitions are rejected and audited.

This section is a proposed contract baseline, not a claim that schemas are implemented. Resolve open contract issues and record ADRs before Phase 1 begins.

## 11. Data, security, and threat model

- Tenant ID is established at the trust boundary and carried in immutable context; every tenant-owned table and query is tenant-scoped. Tenant configuration is authoritative for allowed_domains; client domain mismatch fails with 403 and never dynamically loads a pack. Evaluate PostgreSQL row-level security as defense in depth after considering pooling/context propagation.
- Audit records are append-only at the application boundary; database roles should prevent routine update/delete. Protect integrity with restricted DB credentials, backups, and a documented tamper-detection/retention approach.
- API keys are stored hashed or managed outside source and never logged.
- Raw document storage, retention, deletion, encryption at rest, and access policy must be decided before production use.
- Source quotes may contain sensitive data; restrict access and avoid placing full excerpts in logs.

Threat scenarios and required mitigations to plan/test:
1. Cross-tenant leakage: derive tenant from auth context; tenant predicates on every query; negative isolation tests; consider RLS.
2. Prompt injection or malicious PDF content: treat document text as untrusted data, isolate it from system instructions, constrain structured outputs, validate all values, and never allow the model to choose rules or authorization.
3. Webhook endpoint takeover/SSRF: tenant admins only configure destinations; validate schemes/hosts and block local/private/link-local metadata ranges; protect secrets; sign requests; use egress controls where available. Revisit before real outbound delivery.
4. Audit tampering: append-only application interface, least-privilege DB role, integrity/backup strategy, and tests for unauthorized mutation paths.
5. Malformed/replayed events: validate envelope/version, enforce idempotency, scope by tenant, and bound retries/DLQ.
6. Sensitive evidence exposure: role-check exception detail access, minimize excerpts, redact logs, and define retention/deletion behavior.

## 12. Proposed ADRs for review before implementation

These are recommended MVP defaults, not accepted decisions. Record each as an individual ADR with context, options, decision, consequences, and status Proposed until the maintainer signs off.

### ADR-001 — Versioned event contracts and delivery
- Proposed: keep a strict shared Pydantic BaseEvent envelope with event_id, correlation_id, schema_version, pack_version, retry_count, timestamp, trusted security_context, typed payload, and optional causation_id.
- Proposed events: DocumentUploaded, EventNormalized, EventVerified, BreakDetected, BreakOverridden, ExceptionRejected, and DocumentFailed. Terminal processing failures update state and emit DocumentFailed as well as entering the DLQ. A human override preserves BreakDetected and emits BreakOverridden plus a verification outcome with decision_origin=HUMAN_OVERRIDE; it is never misrepresented as an automated pass.
- Proposed delivery: Redis Streams consumer groups, acknowledge only after durable processing/outbox or idempotent state commit, bounded retries (three attempts), then DLQ with original event ID/correlation and safe error code. Consumers deduplicate by event ID and business idempotency key.
- Consequence: replay must preserve the original event/rules/schema versions and cannot duplicate decisions or external effects.
- Status: Proposed; confirm acknowledgment/outbox and replay semantics.

### ADR-002 — Extraction confidence and evidence
- Proposed: every extracted field carries value, confidence in [0,1], structured evidence {quote, page}, and evidence_id. overall_confidence is the minimum confidence among schema-required fields.
- Proposed gate: if any required field is below 0.85, create LOW_CONFIDENCE_EXTRACTION and skip business rules; retain optional-field confidence, but gate if a rule requires that optional field.
- Proposed: every rule result references the exact fields/evidence used. Quotes are evidence locators, not proof of correctness. Authenticated, tenant-scoped raw-PDF download is restricted to OPS_ANALYST+; raw objects expire after 30 days.
- Consequence: pack schemas must explicitly identify required fields and golden fixtures must score each required field.
- Status: Proposed; 30-day raw retention and analyst+ access are MVP defaults; confirm evidence metadata retention and deletion behavior.

### ADR-003 — Tenant isolation, authorization, and audit
- Proposed: API key resolves to immutable tenant/actor/type/role context at ingestion; never trust payload tenant IDs. Every tenant-owned query includes tenant_id and every audit/event carries the trusted context.
- Proposed: start with explicit tenant predicates and negative cross-tenant tests; evaluate PostgreSQL RLS before pilot exposure as defense in depth, accounting for connection-pool context reset.
- Proposed: application audit interface is append-only; service DB roles cannot update/delete audit rows. Record original automated decision separately from human resolution.
- Consequence: tenant context must be explicit in repository/service APIs; migration and pool tests become security acceptance criteria.
- Status: Proposed; finalize role matrix, RLS decision, and tamper-evidence/retention requirements.

### ADR-004 — Domain packs, rules, and master data
- Proposed: a pack consists of a Pydantic extraction schema, declarative YAML rule set, pack metadata/version, seed records, and golden examples. YAML supports only an allowlisted deterministic operator set; no executable expressions or arbitrary imports.
- Proposed: ruleset and schema versions are pinned on each document at normalization/reconciliation so replay is reproducible.
- Proposed: freight MVP covers LTL/TL only; use po_number as primary lookup key and bol_number as fallback, emitting MISSING_SHIPMENT_REFERENCE when both are absent. Preserve distinct typed references, role-based parties/locations, dates, service, and extensible charge lines. Use Decimal and ISO 4217 currency with explicit tax/credit/discount semantics.
- Proposed: expose idempotent POST /master-records/{domain_id}/upsert for TENANT_ADMIN/OPS_MANAGER; make seed calls the same operation. Store a MASTER_DATA_UPSERTED append-only audit event with before/after JSONB snapshots. No full CRUD UI.
- Consequence: pack authors need contract validation and versioned golden fixtures; changing a rule does not silently alter an in-flight result.
- Status: Proposed; review LTL/TL golden examples, PO/BOL lookup availability, rule grammar, and currency-aware precision/tolerance.

### ADR-005 — Human override and downstream webhook
- Proposed roles: INGEST_CLIENT uploads only; OPS_ANALYST views queue and resolves WARNING breaks; OPS_MANAGER/TENANT_ADMIN may override BLOCKING breaks with a non-empty resolution_note. Enforce least privilege and tenant scope on every action.
- Proposed override: preserve BreakDetected; append APPROVED_OVERRIDE audit entry; emit BreakOverridden and a distinct EventVerified outcome with decision_origin=HUMAN_OVERRIDE. Automated passes use decision_origin=AUTO_PASS. This preserves downstream compatibility while making human approval explicit and auditable. Rejection emits ExceptionRejected and never triggers delivery.
- Proposed delivery: only eligible EventVerified outcomes (AUTO_PASS or HUMAN_OVERRIDE) enter the outbox. Configure destinations through trusted tenant administration; require HTTPS; resolve and validate every address, block loopback/private/link-local/metadata IPs, revalidate redirects, sign payload with HMAC-SHA256, and protect secrets. Any 2xx within a 5-second timeout succeeds; make three attempts with exponential backoff (10s, 60s, 300s).
- Consequence: downstream delivery is at-least-once transport with idempotent receiver semantics; failure remains visible and does not undo approval.
- Status: Proposed; confirm severity model, role matrix, callback payload, and destination-management boundary.

### ADR-006 — LLM boundary and untrusted documents
- Proposed: isolate an extraction/explanation provider interface outside reconciliation. Treat document text as untrusted data, constrain outputs to the active schema, validate after generation, and never let the model choose rules, tenant, role, or final decision.
- Proposed: version prompts/schema/provider settings; enforce timeout and bounded retry/cost limits; use deterministic mocks in tests and a safe explanation fallback.
- Consequence: provider-specific code stays in extraction/triage adapters and is replaceable without changing the deterministic engine.
- Status: Proposed; select provider configuration and decide timeout/cost defaults before the extraction story is Ready.

## 13. Remaining decisions before contract freeze

The following details remain open for maintainer review; the defaults above are proposed and sufficient to draft implementation stories:
- Confirm evidence metadata retention/deletion audit details beyond the 30-day raw PDF TTL and analyst+ access.
- Validate PO-primary/BOL-fallback against representative LTL/TL invoices; confirm currency-aware comparison tolerance.
- Confirm severity taxonomy and OPS_ANALYST / OPS_MANAGER / TENANT_ADMIN permissions, including any second-person review for overrides.
- Confirm exact event compatibility behavior for HUMAN_OVERRIDE EventVerified and DocumentFailed with DLQ.
- Define webhook DNS rebinding/redirect protections and endpoint-secret rotation.
- Confirm pack-version availability/retention for historical replay.
- Confirm master-data upsert uniqueness key and before/after audit retention.

## 14. Delivery plan

Phase 0 — Planning/design gate: review this plan, resolve blocking decisions, write concise ADRs, freeze event/confidence/evidence/domain schemas and rule grammar, define threat model and golden corpus. No product implementation starts until these contracts are reviewed.

Phase 1 — Foundation/contracts: Python workspace, Apache 2.0, contributor docs, Compose Postgres/Redis, Make targets, shared Pydantic contracts and compatibility tests.

Phase 2 — Domain packs: pack loader/rule validation, freight schema/rules/seed/golden examples, vendor compliance schema/rules/seed/golden examples, tenant master upsert path.

Phase 3 — Reconciliation vertical slice: tenant-scoped persistence and audit, confidence pre-rule, master lookup, deterministic evaluator and decision events; cover money/date boundaries, isolation, and duplicate processing.

Phase 4 — Ingestion/extraction: auth/security context, upload validation/storage/hash idempotency, PDF parsing, structured output adapter with per-field provenance, retry/DLQ.

Phase 5 — Triage/resolution/delivery: exception APIs, explanation with rule/evidence references and fallback, role-checked approve/reject override, mandatory note, audit, signed/idempotent webhook.

Phase 6 — CI/E2E/demo: lint, strict typing, unit/integration CI, synthetic PDFs, mocked LLM, pass/break/low-confidence/manual override/rejection/downstream E2E, run/reset/demo/troubleshooting docs.

## 15. Story quality, readiness, and done

Each GitHub story states user/value, scope, assumptions, dependencies, observable acceptance criteria, and out-of-scope notes. Keep stories small and reviewable. Use consistent type/area/priority labels, milestones for delivery phases, and the GitHub Project for status/priority/dependencies.

Ready: value clear; boundaries/dependencies known; acceptance testable; security/tenant effects considered; blocking questions resolved; needed decisions recorded.

Done: acceptance criteria pass; lint/type checks pass; behavior and failure tests exist; tenant scoping/audit addressed; docs/contracts updated; service isolation preserved; reconciliation has no LLM dependency; failure behavior is observable.

## 16. Story map

### Foundation and contracts
- As a developer, I want a Python 3.12+/uv monorepo with Apache-2.0 and strict tooling so changes are reproducible and maintainable.
- As a developer, I want one-command local Postgres/Redis lifecycle and reset/reseed so I can develop without cloud dependencies.
- As a service developer, I want typed versioned envelopes and event payloads so consumers validate compatible asynchronous messages.

### Domain packs
- As a pack author, I want validated schemas, YAML rules, seed masters, and golden fixtures so packs plug in without service-specific domain code.
- As an AP operator, I want freight invoices checked against POs/rate cards so billing breaks are consistent and explainable.
- As a compliance operator, I want COIs/work orders checked against vendor/project requirements so expiration, coverage, and budget breaks are caught.
- As a tenant administrator, I want to upsert master records through a minimal documented path so pilot corrections can be reconciled.

### Reconciliation
- As a tenant, I want master data and audit records tenant-scoped so records cannot cross tenant boundaries.
- As an operator, I want normalized records evaluated deterministically so decisions are repeatable, confidence-gated, and evidence-linked.

### Ingestion and extraction
- As an API client, I want authenticated idempotent PDF upload so documents are attributed and processed once.
- As an operator, I want field-level confidence/evidence and bounded retries/DLQ so low-confidence data and failures are inspectable.

### Triage and downstream
- As an analyst, I want a filtered exception queue with cited explanations so I can understand and prioritize breaks.
- As an authorized reviewer, I want to approve/reject as an explicit override with a mandatory note and audit so human decisions are accountable.
- As a tenant integrator, I want only approved outcomes delivered idempotently so reviewed records can continue into ERP workflows.

### CI and demo
- As a maintainer, I want CI to run lint, strict typing, and tests so regressions are caught before merge.
- As a contributor, I want synthetic end-to-end fixtures so I can reproduce pass, break, low-confidence, review, and downstream scenarios locally.
