# AI Reconciliation Platform — Product and Delivery Plan

Status: Draft for review  
Owner: Project maintainers  
Last updated: 2026-11-26

## 1. Product summary

An open-source, industry-agnostic platform that turns business documents into tenant-scoped normalized records, checks those records against authoritative master data using deterministic rules, and routes breaks to human operators with an auditable path to downstream approval.

The system uses an event-driven pipeline and pluggable domain packs. Language models handle only unstructured document extraction and plain-language explanations. Deterministic Python and declarative YAML decide financial, date, and compliance outcomes.

## 2. Problem and users

Organizations receive inconsistent PDFs and need to reconcile them against contracts, purchase orders, vendor records, project dates, and budgets. Manual review is slow, rules are applied inconsistently, and decisions need evidence and auditability.

- API client: submits a document and selects a domain pack.
- Operations analyst: reviews exceptions, evidence, and explanations.
- Authorized reviewer: approves or rejects with a required note.
- Tenant administrator/integrator: manages master data and downstream delivery.
- Developer/pack author: operates locally or adds a domain pack.

## 3. Outcomes

The MVP demonstrates that:
1. A document is submitted once and traced across asynchronous services.
2. Extracted facts preserve confidence and source evidence.
3. Identical inputs, master data, and rules produce identical outcomes.
4. Tenant data cannot be accessed across tenant boundaries.
5. Automated and human decisions are attributable and auditable.
6. An exception can be understood, reviewed, resolved, and safely delivered downstream.
7. A contributor can run the demo and quality checks locally.

Instrument processing success, confidence distribution, exception rate by rule, stage latency, retry/DLQ rate, and operator resolution time. Set numeric targets after a baseline.

## 4. Scope

### MVP
- Python 3.12+, uv, FastAPI, Pydantic v2, PostgreSQL 16, Redis Streams, Docker Compose.
- Ingestion gateway, extraction worker, deterministic reconciliation engine, triage service.
- Versioned shared events, correlation IDs, immutable security context.
- Freight invoice and vendor compliance packs.
- Tenant-scoped master data, append-only decision and resolution audit.
- Human resolution API and approved-outcome webhook flow; local mock receiver is sufficient.
- CI, unit/integration tests, synthetic fixtures, end-to-end demo.

### Deferred
- External identity provider, including Keycloak.
- Cloud hosting, managed infrastructure, multi-region operation, production SRE commitments.
- General-purpose rules UI or execution of tenant-supplied code.
- Additional domain packs, broad analytics, and autonomous approvals.

## 5. User journeys

### Submit and reconcile
1. Client submits a PDF and domain_id with X-API-Key.
2. Gateway authenticates, derives immutable SecurityContext, computes SHA-256, stores the raw document, and returns a stable document ID.
3. Gateway emits DocumentUploaded. Extraction parses text and returns schema-validated facts with confidence and source quotes.
4. Reconciliation applies the universal confidence gate (<0.85 produces LOW_CONFIDENCE_EXTRACTION), then reads tenant-scoped master data and runs deterministic rules.
5. Passing records emit EventVerified; violations emit BreakDetected. Both decisions are audited.

### Review and resolve
1. Analyst lists tenant-scoped exceptions with failed rules, evidence, and a concise explanation.
2. Authorized reviewer approves or rejects with a non-empty resolution note.
3. Actor, decision, note, and timestamp are recorded append-only.
4. Only approval emits ManuallyApproved. Downstream delivery is idempotent and retry-safe.

### Recover from failure
Transient worker errors retry with bounded attempts. After three failed attempts, poison messages go to stream:dead_letter with correlation and safe error metadata. Failures never silently become verified outcomes.

## 6. Functional requirements

- FR-1 Authentication: API key maps to tenant, actor, actor type, and role; payload tenant IDs cannot override trusted context.
- FR-2 Idempotency: duplicate file hash uploads within a tenant do not create duplicate processing outcomes.
- FR-3 Extraction: active pack schema controls structured output; extracted fields retain confidence and source quote.
- FR-4 Confidence: confidence below 0.85 creates LOW_CONFIDENCE_EXTRACTION before business rules.
- FR-5 Determinism: same normalized payload, master snapshot, and rules version yield the same decision.
- FR-6 Isolation: every database lookup and mutation is scoped by trusted tenant_id.
- FR-7 Audit: record correlation, rule versions/outcomes, evidence references, actors, and timestamps in append-only records.
- FR-8 Triage: exception retrieval is tenant-scoped and supports status/domain filters.
- FR-9 Resolution: enforce roles, mandatory note, explicit state transitions, and safe duplicate request behavior.
- FR-10 Delivery: only approved outcomes are eligible; use idempotency key and record attempts.
- FR-11 Resilience: bounded retries and DLQ behavior are explicit; unsupported event versions fail safely.
- FR-12 Packs: adding a pack should not require service-specific domain logic edits.

## 7. Non-functional requirements

- Security: least privilege, no secrets in source/logs, tenant isolation by construction and tests.
- Determinism: Decimal for money; explicit timezone/date semantics; ruleset version recorded.
- Reliability: idempotent consumers and side effects; bounded retries; observable DLQ.
- Traceability: propagate correlation_id through API, events, persistence, logs, and callbacks.
- Maintainability: service isolation, strict typing, lint/format, clear interfaces.
- Privacy: synthetic demo documents only; avoid raw document text and credentials in logs.

## 8. Architecture and boundaries

Client → Ingestion Gateway → Redis Streams → Extraction Worker → Redis Streams → Reconciliation Engine ↔ PostgreSQL → Triage/Ops → approved downstream webhook. Raw PDFs are stored separately. PostgreSQL JSONB supports flexible master records and payloads where appropriate.

- ingestion-gateway: HTTP auth, security context, document validation/storage, hash idempotency, upload event.
- extraction-worker: PDF parsing, pack schema selection, structured LLM call, provenance, retry/DLQ.
- reconciliation-engine: deterministic evaluation, master lookup, audit, verified/break events. No LLM imports or dependencies.
- triage-service: exception API, explanation, human resolution, approval event, downstream delivery boundary.
- shared: versioned contracts, pack definitions, rule loading/validation, reusable types.

Services communicate asynchronously and never import another service implementation. Event changes are additive when possible; breaking changes require a new schema version and rollout plan.

## 9. Domain packs

### Freight invoices
Fields: carrier, invoice ID/date, PO ID, line items, surcharge, tax, total, confidence, source quote. Rules: PO exists for tenant; surcharge is within contracted cap; line item plus tax reconciles within $0.05 using decimal arithmetic.

### Vendor compliance
Fields: vendor, policy type/expiry, GL coverage, project end date, work order, remaining budget, confidence, source quote. Rules: policy valid through project end; GL coverage is at least $2M; work order is within remaining budget.

## 10. Event and state model — initial proposal

Event envelope: schema_version, event_id, event_type, occurred_at, correlation_id, retry_count, tenant/security context, typed payload. Events: DocumentUploaded, EventNormalized, EventVerified, BreakDetected, ManuallyApproved, and explicit rejection/resolution representation if needed.

Initial document states: RECEIVED → EXTRACTING → NORMALIZED → VERIFIED or EXCEPTION. Exception lifecycle: OPEN → APPROVED or REJECTED. Delivery lifecycle: PENDING → DELIVERED, RETRYING, or FAILED. State transitions must be guarded and audited. Final names and rejection representation are design decisions to settle before implementation.

## 11. Data and security constraints

- Tenant ID is established at the trust boundary and carried in immutable context.
- Every tenant-owned table has a tenant key; queries include it in predicates. Evaluate PostgreSQL row-level security as defense in depth after considering pooling/context propagation.
- Audit records are append-only at the application boundary; database roles should prevent routine update/delete.
- Decide raw document retention, deletion, encryption at rest, and access policy before production use.
- API keys are stored hashed or managed outside source and never logged.
- Source quotes may contain sensitive data; restrict access and logging.

## 12. Architecture decisions to record before implementation

1. Event delivery/idempotency: stream names, consumer groups, acknowledgment timing, retry/backoff, DLQ envelope, duplicates, and replay.
2. Tenant isolation/security context: API key mapping, event trust, tenant predicates, possible RLS, and authorization.
3. Document/audit retention: file storage abstraction, retention/deletion, evidence references, immutable audit semantics.
4. Pack/rule contract: YAML grammar, supported operators, versioning, and startup validation.
5. Human resolution/downstream: approval/rejection state model, role matrix, webhook idempotency, retry/failure behavior.
6. LLM provider boundary: structured-output guarantees, timeout/retry/cost controls, prompt versioning, mocked tests, and fallback.

## 13. Open questions before coding

- Does confidence apply per field, whole extraction, or both? What aggregation is used?
- Is source_quote required for each field, each rule input, or once per document?
- How are currency, rounding, tax inclusivity, and tolerance boundaries represented?
- Is domain pack selected by client, tenant configuration, or both? Which wins on mismatch?
- What PDF limits and retention defaults apply?
- What is the role matrix for list, view, and resolve?
- Does approval override a break, or confirm corrected source/master data?
- What constitutes successful webhook delivery, and how are endpoint secrets managed locally?
- How are rule/schema changes handled for in-flight documents and replay?
- Is master data seeded for demo only, or writable through an MVP admin interface?

## 14. Delivery plan

Phase 0 — Planning: review brief/architecture, resolve blocking questions, record ADRs, define event schemas/error taxonomy/threat model/demo acceptance. This is a planning gate, not an implementation ticket.

Phase 1 — Foundation/contracts: Python workspace, Apache 2.0, contributor docs, Compose Postgres/Redis, Make targets, shared Pydantic contracts and compatibility tests.

Phase 2 — Domain packs: pack loader/rule validation, freight schema/rules/seed/tests, vendor compliance schema/rules/seed/tests.

Phase 3 — Reconciliation vertical slice: tenant-scoped persistence and audit, confidence pre-rule, master lookup, deterministic evaluator and decision events; cover monetary/date boundaries, isolation, and duplicate delivery.

Phase 4 — Ingestion/extraction: auth/security context, upload validation/storage/hash idempotency, PDF parsing, structured output adapter, retry/DLQ.

Phase 5 — Triage/resolution/delivery: exception APIs, explanation with evidence and fallback, role-checked approve/reject, mandatory note, audit, idempotent webhook.

Phase 6 — CI/E2E/demo: lint, strict typing, unit/integration CI, synthetic PDFs, mocked LLM, pass/break/low-confidence/manual-approval/downstream E2E, run/reset/demo/troubleshooting docs.

## 15. Story quality and readiness

Each GitHub story states user/value, scope, assumptions, dependencies, testable acceptance criteria, and out-of-scope notes. Keep stories reviewable. Use consistent type/area/priority labels, milestones for delivery phases, and the GitHub Project for status/priority/dependencies.

Ready: value clear; boundaries/dependencies known; acceptance criteria testable; security/tenant effects considered; blocking questions resolved; needed decisions recorded.

Done: acceptance criteria pass; lint/type checks pass; behavior and failure tests exist; tenant scoping/audit addressed; docs/contracts updated; service isolation preserved; reconciliation has no LLM dependency; failure behavior is observable.

## 16. Story map

### Foundation and contracts
- As a developer, I want a Python 3.12+/uv monorepo with Apache-2.0 and strict tooling so changes are reproducible and maintainable.
- As a developer, I want one-command local Postgres/Redis lifecycle and reset/reseed so I can develop without cloud dependencies.
- As a service developer, I want typed versioned envelopes and event payloads so consumers validate compatible asynchronous messages.

### Domain packs
- As an AP operator, I want freight invoices checked against POs/rate cards so billing breaks are consistent and explainable.
- As a compliance operator, I want COIs/work orders checked against vendor/project requirements so expiration, coverage, and budget breaks are caught.
- As a pack author, I want validated schemas, YAML rules, and seed masters so packs plug in without service-specific logic.

### Reconciliation
- As a tenant, I want master data and audit records tenant-scoped so records cannot cross tenant boundaries.
- As an operator, I want normalized records evaluated deterministically so decisions are repeatable, confidence-gated, and evidence-linked.

### Ingestion and extraction
- As an API client, I want authenticated idempotent PDF upload so documents are attributed and processed once.
- As an operator, I want schema-constrained extraction with provenance and bounded retries/DLQ so failures are inspectable and do not block processing.

### Triage and downstream
- As an analyst, I want a filtered exception queue with cited explanations so I can understand and prioritize breaks.
- As an authorized reviewer, I want to approve/reject with a mandatory note and audit so human decisions are accountable.
- As a tenant integrator, I want approved outcomes delivered idempotently to downstream systems so reviewed records can continue into ERP workflows.

### CI and demo
- As a maintainer, I want CI to run lint, strict typing, and tests so regressions are caught before merge.
- As a contributor, I want synthetic end-to-end demo fixtures so I can reproduce pass, break, review, and downstream scenarios locally.
