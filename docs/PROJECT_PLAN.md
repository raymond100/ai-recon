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
- CI, unit/integration tests, synthetic fixtures, and an end-to-end demo.
- Seeded master data plus a minimal tenant-scoped CLI/import or admin API upsert path for pilot corrections; no full CRUD UI.

### Deferred
External identity provider (including Keycloak), cloud hosting/managed infrastructure, multi-region operation, general-purpose rules UI or tenant-supplied executable rules, additional domain packs, broad analytics, autonomous approvals, and a full master-data CRUD UI.

## 5. User journeys

### Submit and reconcile
1. Client submits a PDF and domain_id with X-API-Key.
2. Gateway authenticates, derives immutable SecurityContext, computes SHA-256, stores the raw document, and returns a stable document ID.
3. Gateway emits DocumentUploaded. Extraction parses text and returns schema-validated facts, each with field confidence and evidence.
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
- FR-3 Extraction: active pack schema controls structured output; every required extracted field carries value, confidence, and source evidence.
- FR-4 Confidence: every required field has confidence in [0,1]. overall_confidence is the minimum confidence among required fields. If any required field or overall_confidence is below 0.85, emit LOW_CONFIDENCE_EXTRACTION and do not run business rules. Optional-field confidence is retained but does not independently block reconciliation unless a rule requires that field.
- FR-5 Evidence: source_quote is required for each extracted field used by reconciliation; a fired rule references the field IDs/evidence IDs it evaluated. Quotes are short excerpts, not free-standing untraceable prose.
- FR-6 Determinism: same normalized payload, master-data snapshot, and rules version yield the same decision.
- FR-7 Isolation: every database lookup and mutation is scoped by trusted tenant_id.
- FR-8 Audit: record correlation, schema/rules versions, rule outcomes, evidence references, actor actions, and timestamps in append-only records.
- FR-9 Triage: exception retrieval is tenant-scoped and supports status/domain filters.
- FR-10 Resolution: enforce role authorization, mandatory note, explicit state transitions, and idempotent request behavior.
- FR-11 Delivery: only approved outcomes are eligible; use idempotency keys and record attempts.
- FR-12 Resilience: bounded retries and DLQ behavior are explicit; unsupported event versions fail safely.
- FR-13 Packs/master data: domain behavior is pack-configured; MVP master data is seeded and can be tenant-safely upserted through a documented script/CLI or minimal API.

## 7. Non-functional requirements

- Security: least privilege, no secrets in source/logs, tenant isolation by construction and tests.
- Determinism: Decimal for money; explicit currency, rounding, and date semantics; ruleset version recorded.
- Reliability: idempotent consumers and side effects; bounded retries; observable DLQ.
- Traceability: propagate correlation_id through API, events, persistence, logs, and callbacks.
- Maintainability: service isolation, strict typing, lint/format, clear interfaces.
- Privacy: synthetic demo documents only; avoid raw document text and credentials in logs; define retention before production use.

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

Header: carrier legal name/ID, invoice number/date, currency, purchase order number, shipment/PRO number, bill of lading number, service level, payment terms, ship date, delivery date, origin, destination, shipper, consignee, and reference numbers.

Charges: line-level charge code/description, quantity/unit, rate, amount, accessorial charges, fuel surcharge basis/rate/amount, tax jurisdiction/rate/amount, subtotal, total, and any credits/discounts. Preserve line-level field confidence and source quote. Treat this as the MVP extraction contract to review against the intended freight invoice examples before freezing the pack; do not claim full EDI-210 compliance without a separate mapping review.

Rules: PO exists for tenant; carrier and shipment references match contracted data where available; surcharge is within contracted cap; line extensions and subtotal/total reconcile using Decimal within $0.05; currency/tax semantics are explicit.

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
- ManuallyApproved: exception_id, decision_id, resolution_id, reviewer identity/role, resolution_note, resolved_at, idempotency key. This is an explicit override and is the only human outcome eligible for downstream delivery.
- ExceptionRejected: exception_id, decision_id, resolution_id, reviewer identity/role, resolution_note, resolved_at. Does not trigger downstream delivery.

Initial document lifecycle: RECEIVED → EXTRACTION_PENDING → EXTRACTING → NORMALIZED → RECONCILING → VERIFIED or EXCEPTION; exhausted processing failures end in PROCESSING_FAILED with a DLQ reference. Exception lifecycle: OPEN → APPROVED or REJECTED (terminal in MVP). Delivery lifecycle is separate: NOT_ELIGIBLE or PENDING → DELIVERING → DELIVERED, RETRYING, or FAILED. Repeated resolution requests with the same idempotency key return the original result; conflicting terminal transitions are rejected and audited.

This section is a proposed contract baseline, not a claim that schemas are implemented. Resolve open contract issues and record ADRs before Phase 1 begins.

## 11. Data, security, and threat model

- Tenant ID is established at the trust boundary and carried in immutable context; every tenant-owned table and query is tenant-scoped. Evaluate PostgreSQL row-level security as defense in depth after considering pooling/context propagation.
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
- Proposed: keep a shared Pydantic envelope with schema version, event ID/type, occurrence time, correlation ID, retry count, trusted tenant/security context, and typed payload.
- Proposed events: DocumentUploaded, EventNormalized, EventVerified, BreakDetected, ManuallyApproved, and ExceptionRejected. Approval and rejection are distinct terminal events; only ManuallyApproved is deliverable downstream.
- Proposed delivery: Redis Streams consumer groups, acknowledge only after durable processing/outbox or idempotent state commit, bounded retries, then DLQ with original event ID/correlation and safe error code. Consumers deduplicate by event ID and business idempotency key.
- Consequence: replay must preserve the original event/rules/schema versions and cannot duplicate decisions or external effects.
- Status: Proposed; confirm acknowledgment/outbox and replay semantics.

### ADR-002 — Extraction confidence and evidence
- Proposed: every extracted field carries value, confidence in [0,1], short source_quote, and evidence_id. overall_confidence is the minimum confidence among schema-required fields.
- Proposed gate: if any required field is below 0.85, create LOW_CONFIDENCE_EXTRACTION and skip business rules; retain optional-field confidence, but gate if a rule requires that optional field.
- Proposed: every rule result references the exact fields/evidence used. Quotes are evidence locators, not proof of correctness.
- Consequence: pack schemas must explicitly identify required fields and golden fixtures must score each required field.
- Status: Proposed; confirm required-field policy and evidence retention/access.

### ADR-003 — Tenant isolation, authorization, and audit
- Proposed: API key resolves to immutable tenant/actor/type/role context at ingestion; never trust payload tenant IDs. Every tenant-owned query includes tenant_id and every audit/event carries the trusted context.
- Proposed: start with explicit tenant predicates and negative cross-tenant tests; evaluate PostgreSQL RLS before pilot exposure as defense in depth, accounting for connection-pool context reset.
- Proposed: application audit interface is append-only; service DB roles cannot update/delete audit rows. Record original automated decision separately from human resolution.
- Consequence: tenant context must be explicit in repository/service APIs; migration and pool tests become security acceptance criteria.
- Status: Proposed; finalize role matrix, RLS decision, and tamper-evidence/retention requirements.

### ADR-004 — Domain packs, rules, and master data
- Proposed: a pack consists of a Pydantic extraction schema, declarative YAML rule set, pack metadata/version, seed records, and golden examples. YAML supports only an allowlisted deterministic operator set; no executable expressions or arbitrary imports.
- Proposed: ruleset and schema versions are pinned on each document at normalization/reconciliation so replay is reproducible.
- Proposed: use Decimal and explicit currency/tax semantics. Freight MVP includes shipment/PRO and BOL identifiers, route parties/locations, service/dates, detailed charge lines, fuel surcharge, tax, subtotal, total, and credits.
- Proposed: ship seed data plus a documented tenant-scoped CLI/import upsert for master corrections; no full CRUD UI.
- Consequence: pack authors need contract validation and versioned golden fixtures; changing a rule does not silently alter an in-flight result.
- Status: Proposed; review freight fields against representative PDFs and confirm rule grammar/money semantics.

### ADR-005 — Human override and downstream webhook
- Proposed: reviewer approval is an explicit override of the recorded automated break, with mandatory note and actor/role/time; it does not mutate the original rule outcome. Rejection is terminal and emits ExceptionRejected without downstream delivery. Correct-and-rerun is deferred.
- Proposed: only ManuallyApproved can trigger delivery. Persist a tenant-scoped idempotency key/outbox record before sending; retries reuse the same key and signature.
- Proposed: configure destinations through trusted tenant administration, require HTTPS, block loopback/private/link-local/metadata targets and redirects to them, sign payloads, protect secrets, and set a bounded retry policy.
- Consequence: downstream delivery is at-least-once transport with idempotent receiver semantics; failure remains visible and does not undo approval.
- Status: Proposed; confirm role matrix, webhook success/retry policy, and destination-management boundary.

### ADR-006 — LLM boundary and untrusted documents
- Proposed: isolate an extraction/explanation provider interface outside reconciliation. Treat document text as untrusted data, constrain outputs to the active schema, validate after generation, and never let the model choose rules, tenant, role, or final decision.
- Proposed: version prompts/schema/provider settings; enforce timeout and bounded retry/cost limits; use deterministic mocks in tests and a safe explanation fallback.
- Consequence: provider-specific code stays in extraction/triage adapters and is replaceable without changing the deterministic engine.
- Status: Proposed; select provider configuration and decide timeout/cost defaults before the extraction story is Ready.

## 13. Remaining decisions before contract freeze

- Confirm field-level confidence plus overall minimum-of-required-fields rule and whether any exceptions are needed.
- Confirm per-field source quote/evidence IDs and retention/access rules.
- Review freight field list against representative invoice examples; decide exact charge/tax/currency semantics.
- Select domain pack authority when client request and tenant configuration differ.
- Set PDF size/page and raw retention defaults.
- Confirm reviewer role matrix and approval override semantics.
- Confirm event payload required/optional fields, terminal failure representation, and rejection event.
- Define webhook destination trust, signing, SSRF protections, success response, and retry window.
- Decide how in-flight documents pin schema/rules versions during replay.
- Confirm master data upsert CLI/API boundary and audit behavior.

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
