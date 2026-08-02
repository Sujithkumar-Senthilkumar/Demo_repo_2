---
workflow: create-spec
run_id: e0d47ece-3ddd-4106-93e2-20a658b0c4a2
sections: 7
units: 20
generated_by: build_doc_fanout_graph
---

## Feature Goal


A centralized Customer Feedback Portal that replaces ad hoc intake (email threads, spreadsheets, disparate tickets) with a single, auditable system of record and a public roadmap. The portal must be implementation‑ready: it shall specify user-visible behavior, machine-readable data contracts, governance controls, and measurable acceptance conditions so engineering and operations can build, test, and operate the solution with minimal assumptions.

Target end state (implementation-ready characteristics)

- System scope and primary behavioral difference
 - All external customer and internal team feature requests and their lifecycle events are ingested, stored, surfaced for voting, and published to a public roadmap. Existing channels remain readable for historical data only; new primary intake is disabled at source for legacy channels. Historical items may be imported via a controlled import API (see Import rules below) and are flagged imported=true; imports are not accepted through the regular public submission endpoint.

- Lifecycle state machine (explicit states and transitions)
 - Request states: Submitted -> Triaged -> UnderReview -> Planned -> Committed -> InProgress -> Released OR Rejected OR Duplicate OR Archived. A request in Duplicate points to a canonical_request_id.
 - Allowed transitions:
 - Submitted -> Triaged (automatic or manual)
 - Triaged -> UnderReview or Rejected or Duplicate
 - UnderReview -> Planned or Rejected
 - Planned -> Committed -> InProgress -> Released
 - Any state -> Archived (administrative)
 - Events that create timeline entries: submit, edit, vote_cast, vote_removed, comment, triage_action (assign, label, set_priority), decision (approved/rejected/accepted), roadmap_publish, import, attachment_add, attachment_remove, audit_correction. Each event is represented as an immutable audit entry (see Audit log format).

- Deduplication rules
 - Canonicalization steps (server-side, deterministic):
 1. Normalize title and tags using Unicode NFKC, casefold, strip surrounding punctuation, and collapse internal whitespace (locale: en-US for casefolding and punctuation rules).
 2. Exact-match check: if normalized_title and product_area match an active request, candidate is considered a probable duplicate.
 3. Similarity check: compute normalized cosine similarity on token sets; if similarity ≥ 0.85 and either (customer_account_id matches or summary similarity ≥ 0.9) then return candidate duplicate.
 4. If a probable duplicate is detected at create time and the client did not provide createIfDuplicate=true, respond 409 Conflict with suggested canonical_request_id and duplicate_reason details. Creation may still proceed if createIfDuplicate=true; server will then link the new request to the canonical id and mark as duplicate only after human triage.
 - Human triage must explicitly set canonical_request_id to finalize duplicate status; duplicates do not become primary roadmap candidates unless canonical_request is promoted.

- Submission schema and validation (server-enforced)
 - Exact field constraints (summary):
 - title: string, 5–140 UTF-8 characters; normalized_title unique among active requests per (normalized_title, product_area). Uniqueness enforced by a transactional DB unique index; concurrency conflicts return 409 Conflict.
 - summary: markdown, 20–2000 chars.
 - product_area: enum {billing, onboarding, reporting, integrations, mobile, platform, other}. Invalid enum -> 400 with field-level errors.
 - customer_account_id: nullable UUID v4 when supplied; enterprise accounts recommended to supply.
 - submitter_contact: email; optional for anonymous posts but required when anonymous=false.
 - use_case_description: plain text, 50–5000 chars.
 - proposed_benefit: plain text, 20–2000 chars.
 - tags: array of strings, max 10, each 1–50 chars; normalized using NFKC + casefold + punctuation strip (en-US); duplicates removed server-side.
 - attachments: array of files, max 5, max 10 MB each; allowed types {pdf, png, jpeg, txt, csv, md}. Each file is virus-scanned before acceptance; disallowed types -> 415 Unsupported Media Type; oversize -> 413 Payload Too Large.
 - privacy_consent: boolean; must be true for anonymous public posts; otherwise 400.
 - Validation responses:
 - 400 Bad Request with structured field-level error objects for validation errors.
 - 409 Conflict for uniqueness/deduplication conflicts (see Deduplication rules).
 - 401 Unauthorized and 403 Forbidden per authentication/authorization rules.

- API surface (minimal deterministic endpoints and behavior)
 - Public submission and retrieval:
 - POST /api/requests
 - Headers: Authorization: Bearer <token>; Idempotency-Key optional for clients to avoid duplicate submission.
 - Body: JSON per submission schema.
 - Responses: 201 Created with Location header and body { request_id, status }; 409 Conflict with canonical candidates; 400/401/403 per validation/auth.
 - GET /api/requests/{request_id}
 - Returns request record with lifecycle_state, vote_count, canonical_request_id, timeline (paginated), and ETag header for concurrency control.
 - PATCH /api/requests/{request_id}
 - Requires If-Match: <ETag>; supports partial updates to summary, tags, attachments (uploads via presigned URL flow described in attachments).
 - Conflicting ETag -> 409 Conflict.
 - POST /api/requests/{request_id}/vote
 - Authenticated only. Body optional {vote_weight_override_token} for enterprise-weighted votes.
 - Response: 200 OK with new vote_count.
 - DELETE /api/requests/{request_id}/vote
 - Removes the caller’s vote. 200 OK.
 - GET /api/requests?filters...
 - Supports filtering and sorting by product_area, lifecycle_state, tag, vote_count, date ranges. Pagination via cursor.
 - Admin/import:
 - POST /api/requests/import
 - Admin-only. Requires Idempotency-Key header. Body includes legacy_id and original_channel. If legacy_id already imported, returns 200 with existing mapping; otherwise creates new imported record and returns 201.
 - Imported items are marked imported=true and original_channel preserved.
 - Roadmap publishing:
 - POST /api/roadmap/items
 - Role-limited to users with roadmap_publisher permission. Roadmap item links to request_id(s) and records publish_event (approver_id, decision_note).
 - Changes to public roadmap are recorded as timeline events and audit entries.

- Vote semantics, verification, and anti-abuse
 - One active upvote per verified account per request; voters may remove or change vote but cannot exceed one vote at any time.
 - Anonymous votes are not permitted. All vote actions require an authenticated OAuth2 bearer token; token must contain sub and account_id claims; for SSO-authenticated users, email_verified must be true.
 - Enterprise customers may submit weighted votes via a signed entitlement token in the vote request; server verifies signature and maps to a numeric weight. Weighting requires explicit product configuration and is not enabled by default.
 - Abuse mitigation: server enforces rate limits (per-account and per-IP) and anomaly detection (sudden vote spikes) that can flag a request for manual review and temporarily suspend vote tallying until reviewed.

- Audit log format, timestamping, actor identifiers, and retention
 - Each event creates an immutable JSON audit entry with schema:
 - event_id: UUID v4
 - request_id: UUID v4 (nullable for system-level events)
 - event_type: one of {submit, edit, vote_cast, vote_removed, comment, triage_action, decision, roadmap_publish, import, attachment_added, attachment_removed, archival}
 - actor_id: string formatted as "user:{account_id}" or "system:{service_name}"
 - actor_display: optional human name/email
 - timestamp: ISO 8601 string in UTC with millisecond precision (e.g., 2026-08-02T15:04:05.123Z)
 - payload: JSON object containing event-specific details
 - previous_etag: optional for audit of edits
 - Audit entries are stored append-only and exported to immutable storage; each entry is cryptographically signed (HMAC-SHA256) and stored alongside signature to enable tamper detection.
 - Retention policy: audit entries retained for 7 years; soft-deleted requests retained 90 days before permanent deletion unless legal hold applies. Retention durations configurable but must be enforced by the operations team; deletion events themselves are audit-logged.

- Attachments and virus-scan failure behavior
 - Upload flow: client requests presigned URL, uploads blob, server triggers virus scan. On successful scan -> attachment accepted and associated event written. On detected malware -> upload rejected with 422 Unprocessable Entity; server quarantines blob (quarantine TTL 30 days) and notifies uploader and security ops. For transient scan failures (timeout or scanner error) server retries up to 3 times; persistent scanner failures return 503 Service Unavailable with guidance to retry later.
 - Attachment storage must support quarantined vs. accepted buckets and record scan_result in attachment metadata.

- Concurrency and uniqueness handling
 - Uniqueness of normalized_title among active requests is enforced by a transactional unique constraint on (normalized_title, product_area, active_flag). Implementations must use a retry-with-backoff strategy upon constraint violation:
 - On constraint violation, server responds 409 Conflict with current conflicting resource reference and a recommended retry-after header; clients may attempt createIfDuplicate=true or present conflict resolution to the user.
 - PATCH updates must require If-Match ETag to avoid lost updates. Server returns 412 Precondition Failed if ETag does not match.

- Roadmap publication and governance controls
 - Roadmap items are separate entities that reference request_id(s). Only users with role roadmap_publisher may move a request into Planned/Committed on the public roadmap.
 - Decisions that place a request on the roadmap require a decision_event recorded in the audit log including approver_id, decision_timestamp, decision_deadline (if any), and rationale. The owner of the decision is a named role (product_owner) that must be mapped in organizational configuration (see Assumptions & Open Questions in BRD).
 - The system must expose an approvals endpoint for manual sign-off and maintain an SLA dashboard measuring time from Submitted -> decision_event to support the median-under-30-days success metric.

- Import, idempotency, and legacy integration rules
 - Legacy import is allowed only via admin import endpoint and requires Idempotency-Key. The server records legacy_id->request_id mappings; repeated import with same Idempotency-Key or legacy_id returns the original resource and 200 OK.
 - Imported records preserve original timestamps (imported_created_at) but must be re-validated against current submission schema; non-conforming legacy items are flagged for manual triage and not published to the public roadmap until corrected.

- Observability, metrics, and acceptance traces
 - The system must emit metrics for: submission rate, vote rate, distinct active users (monthly), median time from submission to decision, duplicate detection rate, import success rate, attachment malware incidents, and rate of concurrency conflicts.
 - For each acceptance test, a reproducible trace must exist: API call sequence, event_id(s) recorded, and audit entries verifying expected state transitions.

- Operational and edge-case rules
 - Timezone and timestamps: all stored timestamps are UTC; UIs may render local time but must store UTC.
 - Tag normalization: uses Unicode NFKC + casefold + punctuation strip with en-US rules; server will return normalized tags in responses.
 - Error semantics: validation 400 includes field path, code, and human_message; conflicts 409 include conflicting_resource_id and conflict_reason.

This unit defines the end state and prescriptive behavioral rules needed to implement the portal without unstated assumptions: explicit lifecycle states and transitions, deterministic deduplication rules, vote semantics and verification, precise audit log schema/timestamping/actor format, retention policy, virus-scan failure handling, concurrency and uniqueness enforcement, tag normalization locale, minimal API surface for submission, voting, import and roadmap publishing, and governance controls required to achieve the stated reduction in decision time.

## Business Justification

- Business value and user impact
 The portal centralizes feature-request intake into a single, auditable system of record and publicly visible roadmap, converting distributed signals into actionable, measurable inputs for prioritization. Expected business value:
 - Faster, evidence-based roadmap decisions that reduce opportunity cost and backlog churn by surfacing consolidated demand signals (votes, comments, request volume) tied to request records.
 - Improved customer retention and satisfaction through transparent engagement, visible decision outcomes, and a public roadmap that communicates status to external audiences.
 - Reduced internal cost from duplicated intake and context-switching by replacing ad hoc channels (email, spreadsheets) with a discoverable, searchable repository.
 Measurable targets and clarifying rules:
 - Decision metric: a “decision” is a recorded state transition on a request record to one of {Accepted, Rejected, Deferred, Scheduled}. A decision is valid for KPIs only when the decision record is persisted in the portal audit log, contains the required decision fields, and is exposed via the portal API.
 - Required decision fields: decision_date (UTC ISO‑8601 timestamp), decision_owner (account_id or role_id), and decision_rationale. decision_rationale must include either (a) one or more portal datum references (request_id and a snapshot of vote_count and comment_summary at decision_date) or (b) an external URL to a resolvable artifact plus a one-line metadata summary. This satisfies the “demonstrably informed” requirement.
 - Timeline clarification: the product-level success target is to reduce median decision time to under 30 calendar days measured in UTC. The program will evaluate progress over a rolling 90‑day window; the delivery target is to achieve the under‑30‑day median within nine (9) months of GA, with monitoring and intermediate checkpoints beginning at month six and concluding no later than month twelve following GA.
 - KPI inclusion rules: a request counts toward KPIs only if it is a Valid Request as defined below and the decision record meets persistence and exposure requirements.

- Integration with existing features
 The portal integrates with existing intake and product workflows as follows:
 - Submission sources: supports submissions from the web UI and the REST API (see FUNC-001 for intake interfaces). Bulk imports and migrations are supported via the import API and must include source metadata fields (source_system, original_created_at, source_author_id, source_migrated=true).
 - Authentication & API expectations: REST API access requires OAuth2 bearer tokens or scoped API keys. API responses use standard HTTP status codes: 200/201 on success, 400 for validation errors with a structured field-error payload, 401/403 for auth/authorization failures, 404 for not-found, 429 for rate limit exceeded, and 500 for server errors. Default rate limit is 200 requests/minute per account; contracts may specify higher tiers.
 - Data handling and compliance: request records and audit logs are retained for a default seven-year period subject to legal hold exceptions. Personally identifiable information (PII) shall be minimized in request fields; the system must support GDPR-compliant data subject requests: export (machine-readable), rectification, and verified deletion. Deletion will either remove or pseudonymize personal data while preserving non‑PII audit trails as required by law.
 - Integration points: the portal exposes a documented REST API and webhook subscriptions for state changes (decisions, status updates, merges). External systems may subscribe to decision events and request updates; payloads include request_id, current_status, decision snapshot, and correlation_id when present.

- Problems this solves and for whom
 Problem: Fragmented intake of feature requests
 - Who benefits: External customers, Customer Success, Sales, Product Management, and Engineering.
 - How solved: The portal replaces ad hoc channels with a centralized, searchable system of record that captures submission metadata, vote and comment signals, and audit trails. It enforces minimum submission quality (see Valid Request) so demand signals are comparable and actionable.

 Problem: Slow prioritization and decision cycles
 - Who benefits: Product Management, Engineering, Support, and customer communities awaiting decisions.
 - How solved: By surfacing vote_count and comment_summary alongside request records and providing a required decision_rationale tied to portal data, the portal creates evidence trails that streamline triage and support accountability for decisions.

 Operational definitions and edge-case rules (to remove ambiguity for implementers):
 - Valid Request (for KPI eligibility): a request eligible for KPIs MUST meet all of:
 - Created by an authenticated submitter (external customer or internal staff) via the portal UI or REST API, or created by a permitted migration/import with source_migrated=true and original_created_at present.
 - Contains required fields: title (non-empty), description (>= 20 characters), submitter_id, created_at (UTC ISO‑8601).
 - Not flagged as spam by the automated filter at creation time.
 - Not explicitly marked excluded_from_kpis=true (applies to historic migrations or legal/contractual exclusions).
 - vote_count computation:
 - vote_count is the count of unique authenticated accounts whose last vote action on the request is an active vote and whose last vote timestamp <= decision_date. An active vote is an upvote not subsequently retracted. Repeated vote actions by the same authenticated account update the vote state and timestamp but do not increment unique counts. Anonymous or unauthenticated interactions do not contribute.
 - comment_summary:
 - comment_summary is a structured aggregation {total_comments:int, top_comment_id:string}.
 - total_comments counts non-spam comments authored by authenticated accounts with created_at <= decision_date.
 - top_comment_id is chosen by deterministic ranking: highest reaction_count, then earliest created_at, then lowest comment_id as final tie-breaker.
 - Spam is determined by the automated comment filter at creation time and persists as a flag; only comments without the spam flag count.
 - Duplicate-detection algorithm and reconciliation:
 - Similarity computation: normalized similarity score s in [0.0,1.0] is computed as a weighted combination of normalized Levenshtein distance on title (weight 0.4) and TF‑IDF cosine similarity on title+description (weight 0.6). All text comparisons are performed on normalized (case-folded, punctuation-stripped, Unicode-normalized) text.
 - Thresholds and behavior:
 - s >= 0.85: auto-flag as probable duplicate; candidate duplicate links are displayed in the request view. The system may auto-link candidate duplicates (create non-destructive cross-reference links) but MUST NOT auto-merge records without explicit internal user confirmation.
 - 0.60 <= s < 0.85: present as potential duplicate suggestions to reviewers; no auto-linking.
 - s < 0.60: no duplicate suggestion shown.
 - Reconciliation: system must provide a review UI for merging or designating canonical requests; merges require an internal user with merge permission and create a merge record in the audit log. Auto-merge is prohibited.
 - “Demonstrably informed” reference:
 - A decision_rationale is demonstrably informed when it contains either:
 - At least one portal datum reference: request_id(s) and a snapshot of vote_count and comment_summary at decision_date (these fields are stored with the decision record), or
 - An external resolvable URL to an artifact plus a one-line metadata summary (artifact type, author, date).
 - Timezones and SLA measurement:
 - All timestamps and SLA windows are measured in UTC and expressed in calendar days unless explicitly annotated as business days. Business day definitions (Mon–Fri UTC excluding company-observed holidays) and holiday lists are maintained by the Head of Product and applied where SLA requires business-day semantics.
 - Edge cases:
 - Bulk imports/migrations: imports must supply source metadata and may set excluded_from_kpis=true for historical archives. To include migrated records in KPIs, imports must include include_in_kpis=true and attest that source consent and data quality meet portal rules; Product Management review is required.
 - Correlated decisions: when a decision applies to multiple requests, the system must create a separate decision record per request and also create a correlation_id tying them together; each decision record must include its own decision_rationale and persisted snapshot.
 - Error handling: API validation errors return structured field-level messages; rate-limited clients receive 429 with Retry-After header. Clients must be able to distinguish transient server errors (5xx) from client errors (4xx).
 - Governance requirements:
 - Decision owner and approval workflow: Product Management must nominate named decision owner roles and define the internal approval workflow and SLAs before launch. The portal enforces that a decision record cannot be persisted without a populated decision_owner and decision_rationale; audit logs record approver account_id and timestamps.

This unit complements the implementation-level requirements in FUNC-001 (Submission & Intake) and the Scope/Success Criteria in FEAT-002; it defines the operational definitions and integration expectations necessary to make KPIs, auditability, and decision traceability unambiguous and testable.

## Feature Scope

### Scope Description
- What the end-user will experience (user interactions, UI flows)
 - Submission flow: Authenticated external customers and internal users may create feature-request records via the web UI and the REST API (see FUNC-001 for interfaces). The submission form and API require explicit values for: title, product area, short description, detailed use case, and customer impact; attachments are optional. A consent checkbox controls public display; when unchecked the request is marked Private and is excluded from all public listings and the public roadmap. Anonymous web submissions are accepted but remain in Pending state until email verification completes via a one-time verification link valid for 24 hours; until verification completes the request is not included in public vote tallies and the submitter cannot cast votes. The UI presents immediate client-side validation for required fields and displays server-side structured error responses for API clients using documented error codes (4xx for client errors, 5xx for server errors). When a user is rate-limited or suspended, vote and comment controls are disabled and show a concise tooltip explaining the state and the next allowable action time.
 - Duplicate-suggestion UX: During submission the UI displays up to five candidate duplicates ranked by a combined score derived from deterministic signals and a heuristic similarity score. Deterministic signals are: exact normalized-title match (case- and whitespace-normalized), exact product-area match, and matching external IDs. Heuristic signals use a named semantic-similarity model (Model v1.0) plus keyword-overlap. The client-visible reason tags (e.g., "title match", "high semantic similarity") and a numeric similarity score (0.00–1.00) are shown. A candidate is flagged "Likely duplicate" when any deterministic signal matches OR when the heuristic similarity score is >= 0.85. The submitter may select merge, link-to, or ignore; any user acknowledgement, override, or merge action is recorded to the request audit trail with actor, timestamp, and action reason.
 - Discovery and list/detail views: Search supports keyword full-text search and faceted filters for status, tags, product area, public/private, and date range. Sort options include vote count, recent activity, and decision date. Detail pages display vote count, comments, submission metadata (submitter identity when public, submission timestamps, attachments), public/private status, and an append-only audit trail of status changes, merges, and acknowledgements. Vote and comment controls are disabled when the actor lacks permission, is rate-limited, or suspended; the UI surfaces the disabling condition and a suggested next step (e.g., "verify email", "wait X minutes").
 - Voting and comment interactions: Vote and comment actions are available in UI and API (see FUNC-002 for endpoints). Default vote allocations are: authenticated external customers = 3 votes per rolling 30-day sliding window; internal users = 10 votes per rolling 30-day sliding window. Administrators may configure per-role vote limits (0–100 votes per 30-day window). Rolling-window semantics are calculated by counting timestamped vote records whose timestamps fall within the preceding 30*24 hours from the current evaluation time. Users may reallocate a vote from one request to another within 14 calendar days of the original vote timestamp; reallocation is implemented as a withdrawal of the original vote record plus creation of a new vote record, both recorded with timestamps and an audit entry. Vote actions and resulting counts are applied atomically and are reflected in the UI within 5 seconds under normal operating conditions.
- Key technical capabilities being delivered
 - Authentication and identity: Integration with the platform's SSO/OIDC identity provider for authenticated users; support for anonymous submission with email-verification flow using one-time tokens.
 - Duplicate-detection pipeline: Deterministic matching module and a heuristic semantic-similarity module (Model v1.0) producing normalized scores; ranking and thresholding logic executed synchronously during submission with documented model versioning and configurable thresholds.
 - Audit and immutability: All state transitions (create, update, merge, acknowledge, vote, reallocate) are persisted to an append-only audit store with immutable timestamps and actor IDs. Audit entries are immutable except for administrative redaction metadata (redaction marks preserved as audit entries). Audit data retention is at least seven years or per organizational compliance policy.
 - Concurrency and data integrity: API endpoints implement optimistic concurrency control via resource versioning and idempotency tokens for create/update operations; vote operations are applied with atomic increment/decrement semantics to avoid double-counting under concurrent requests.
 - Notifications and email verification SLA: The system sends verification and notification emails via an external email service; the system must accept delivery confirmation from the email service and retry deliveries on transient failures up to three times over a 24-hour period. Verification emails are queued and sent within 60 seconds for 95% of requests; failed deliveries are logged to a dead-letter queue and surfaced to administrators.
 - Observability and analytics: Submission, vote, merge, and decision events are emitted to the analytics pipeline and to operational logs with standardized event schemas for downstream reporting and the Success Criteria metrics.
- Integration points with existing systems
 - Identity provider (SSO/OIDC) for authenticated users and role resolution.
 - Email delivery service (e.g., SES, SendGrid) for verification and notification emails, with delivery confirmation and retry handling.
 - Analytics/event pipeline for adoption and decision-metric reporting.
 - Platform audit/log storage for append-only audit trail retention and administrative reporting.
 - Administrative configuration UI/API for role-based vote limit configuration and duplicate-detection threshold tuning.
- What is explicitly OUT of scope for this feature
 - Other feedback workflows such as general support tickets, bug-reporting pipelines, or CSAT surveys are excluded from initial delivery.
 - Full ticketing backend synchronization (bi-directional ticket sync), enterprise SLA management, or complete customer service workflow automation are excluded; lightweight one-way links to external systems (external IDs) are allowed but full integration is out of scope.
 - Advanced fraud-detection beyond configured vote-rate safeguards (e.g., third-party identity fraud services) is excluded; basic safeguards (vote limits, rate-limits, account verification) are included.
 - Long-term archival policies beyond the stated minimum retention and preservation of audit immutability (organization may define extended archival separately).

### Success Criteria
- [ ] Median time from feature request submission to a recorded product decision (accepted or rejected) is under 30 days, measured over a trailing 90-day window.
- [ ] Monthly active external customers performing at least one submit, vote, or comment reaches adoption target defined by Product (baseline and target set in roadmap) and shows sustained or increasing trend month-over-month for three consecutive months.
- [ ] Monthly active internal team users performing at least one review or roadmap update reaches target defined by Product and maintains sustained engagement.
- [ ] At least 90% of email verification requests are delivered to the email service for sending within 60 seconds; transient delivery retries do not exceed three attempts over 24 hours and failed deliveries are logged to the dead-letter queue.
- [ ] Audit trail retention and immutability validated: 100% of state-change events (create, merge, vote, reallocate, decision) are present in the append-only audit store and immutable for the configured retention period.
- [ ] Duplicate-suggestion accuracy and behavior: deterministic matches always surface when criteria met; heuristic similarity model (Model v1.0) flags Likely duplicates at similarity >= 0.85; system logs model version and threshold used for each suggestion for traceability.
- [ ] Voting semantics validated: rolling 30-day vote windows and 14-day reallocation behavior operate as specified (timestamped vote records counted and reallocation implemented as withdraw+create) with atomicity under concurrent operations.
- [ ] Product analytics show an increasing proportion of roadmap decisions that reference portal request records or vote counts (measured monthly), as defined by Product targets for "decisions informed by portal activity."

## Functional Requirements

### FUNC-001: Submission & Intake

*[Unit FUNC-001 — content pending. Intent: Define all FRs for intake and submission (FR-001 through FR-002): collecting feature requests, ensuring discoverability, recording metadata and auditable history, supporting customers and internal teams as submitters, and enabling comments on requests.]*

### FUNC-002: Voting & Ranking

- FR-003: [DETERMINISTIC] Vote submission, update, and withdrawal APIs and UI controls MUST be provided and behave deterministically:
 - Endpoints and methods: POST /api/requests/{requestId}/votes for create/update and DELETE /api/requests/{requestId}/votes/{voteId} for withdrawal. Web UI controls MUST present equivalent capabilities.
 - Authentication and principal resolution: the server SHALL derive the authenticated principal's identity and role solely from the session/principal presented by the authentication layer. Any actorId or actorRole supplied in the request body MUST be ignored; if the request body attempts to act on a different principal than the authenticated principal the server SHALL return 403 Forbidden.
 - Payload contract (canonicalized JSON): { voteValue: integer, clientMetadata?: { clientId?: string }, idempotencyKey?: string }. The server SHALL apply a canonicalization rule for idempotency: sort JSON object keys, trim strings, and treat omitted optional fields as null for normalization.
 - Allowed vote values: by default the deployment configuration (config key "vote.allowedValues") SHALL be {1} (upvote-only). The allowedValues set is configurable per deployment to any finite set of integer values (for example {-1, 0, 1}). The API SHALL validate voteValue against the configured set and return 400 Bad Request for invalid values.
 - Idempotency semantics: the server SHALL accept an Idempotency-Key header or idempotencyKey field. Identical tuples of (authenticatedPrincipalId, endpoint, idempotencyKey, normalizedRequestBody) SHALL map to a single logical operation for a configurable retention window (config key "vote.idempotencyWindowHours", default 24). For subsequent identical idempotencyKey uses within the retention window the server SHALL return the original HTTP status, response body, and the same operationRequestId.
 - Operation identifier and audit: each accepted vote operation (create/update/withdraw) SHALL generate and persist a UUID operationRequestId and return it in the response body.
 - Uniqueness and update semantics: the system SHALL enforce exactly one active (non-withdrawn) vote per authenticated principal per request. A POST from the same authenticated principal for the same request SHALL update the existing active vote (operationType = update) rather than create a duplicate voteId. A distinct voteId may be created only when the authenticated principal is different.
 - Withdrawals: DELETE /api/requests/{requestId}/votes/{voteId} SHALL be permitted only if the authenticated principal is the vote owner or has admin privileges. Successful withdrawal SHALL mark the stored vote record as withdrawn (soft-delete) with withdrawal timestamp and SHALL be recorded as an operation in the append-only audit store.
 - Concurrency and transactional behavior: vote create/update/withdraw operations that concurrently target the same (requestId, authenticatedPrincipalId) SHALL be serialized using database transactional semantics and/or optimistic concurrency control (for example version number or unique active-vote constraint). If a conflicting concurrent modification is detected the API SHALL return 409 Conflict.
 - Response codes and synchronous/deferred recalculation hinting: APIs SHALL use standard HTTP status codes. Typical returns: 201 Created for a new vote operation that completed and synchronous recalculation succeeded within configured timeout, 200 OK for successful updates and withdrawals that completed synchronously, 202 Accepted when the vote operation completed but ranking recalculation was deferred due to synchronous timeout, 409 Conflict for uniqueness/state conflicts, 400 Bad Request for invalid input, 401/403 for authentication/authorization failures, and 429 for rate-limit enforcement. When returning 202 Accepted the response body SHALL include operationRequestId and { "recalcStatus": "deferred" }.

- FR-004: [DETERMINISTIC] Append-only audit and persisted vote record requirements:
 - Every vote operation (create/update/withdraw) SHALL be recorded in an append-only audit store containing at minimum these immutable fields: voteId, requestId, operationRequestId, idempotencyKey (if provided), authenticatedPrincipalId, actorRoleAtTimeOfOperation, operationType (create | update | withdraw), voteValue (nullable for withdraw), clientMetadata (as supplied, advisory), serverObservedClientIp, serverObservedUserAgent, timestampUTC, source (API | UI), and storageVersion.
 - IP and User-Agent capture behind proxies: extraction of client IP SHALL follow deployed proxy-trust configuration: when config "trustProxyHeaders" = true and trusted proxy CIDR list "trustedProxyCidrs" is configured, server SHALL parse X-Forwarded-For and select the first IP from the left that is not contained within any trusted proxy CIDR; if no such header or trust disabled, server SHALL use the connection remote address. The server SHALL always persist the raw headers (e.g., X-Forwarded-For) and the resolved serverObservedClientIp.
 - Role snapshot semantics: the actorRoleAtTimeOfOperation field SHALL capture the authenticated principal's role at the time the operation is recorded. Historical operations SHALL NOT be modified retroactively if a principal's role later changes; role changes SHALL be recorded as separate operations elsewhere.
 - Retention and immutability: the audit store SHALL be append-only; soft-deleted/withdrawn vote records SHALL remain in the audit with a withdrawn flag and timestamp. Administrative or legal retention policy exceptions SHALL be implemented as separate processes that mark records as sealed, not by modifying operation records themselves.

- FR-005: [DETERMINISTIC] Ranking calculation, normalization, exposure of vote metrics, and consistency guarantees:
 - Ranking inputs and primary outputs: the ranking engine SHALL compute and persist for each request: unique_voter_count (count of distinct authenticatedPrincipalId with an active vote), total_votes_sum (arithmetic sum of voteValue across active votes), weighted_vote_sum (sum of voteValue * roleWeight where roleWeight is taken from actorRoleAtTimeOfOperation), normalized_score (computed as weighted_vote_sum / normalizationDenominator), and last_recalc_timestamp.
 - NormalizationDenominator and maxWeightedVoteSumObserved windowing: normalizationDenominator SHALL be computed as max(1, maxWeightedVoteSumObservedWindow) where maxWeightedVoteSumObservedWindow is the maximum absolute weighted_vote_sum observed across all requests within a configurable sliding window (config key "vote.normalizationWindowDays", default 30). Using max(1, ...) prevents divide-by-zero. This definition of maxWeightedVoteSumObserved is global to the deployment and windowed by time as specified; it is not per-request.
 - Normalized score range: normalized_score SHALL be a floating value in [0,1] representing request popularity relative to the observed window maximum. If weighted_vote_sum is negative allowed by configuration, normalized_score SHALL represent scale relative to absolute max (still using max(1,...)).
 - Recalculation performance and consistency: recalculation SHALL be attempted synchronously after vote operations within a configured synchronous timeout (config key "vote.syncRecalcTimeoutMs", default 2000 ms). If recalculation cannot complete in time, the API SHALL accept the vote operation and return 202 Accepted and enqueue the request for background recalculation. Background recalculation queue SHALL guarantee eventual consistency; updates from background recalculation SHALL be applied within a configurable background window (config key "vote.backgroundRecalcWindowSeconds", default 300 seconds). Read APIs SHALL document this eventual consistency window.
 - Tie-breaking deterministic ordering: when ranking items with identical normalized_score, tie-breakers SHALL be applied in this deterministic order: 1) higher weighted_vote_sum, 2) higher unique_voter_count, 3) most recent last_vote_timestamp, 4) older request.creationTimestamp (older first), and finally 5) lexicographic requestId as final deterministic tie-breaker. The tie-breaker sequence SHALL be configurable.
 - Read API exposure: read APIs SHALL expose the fields named above and last_recalc_timestamp. When a vote operation returned 202 Accepted the read APIs MAY temporarily reflect pre-recalc values until background recalculation completes; the API SHALL surface last_recalc_timestamp so callers can detect staleness.

- FR-006: [AI-CANDIDATE] Safeguards against manipulation, monitoring signals, and configurable mitigation actions:
 - Rate-limits and throttling: the system SHALL enforce rate-limits configurable per principal (config key "rateLimits.principal") and per resolved client IP (config key "rateLimits.ip"). Exceeding limits SHALL return 429 Too Many Requests.
 - Uniqueness and rapid change protections: the system SHALL apply a configurable minimum interval between vote changes from the same principal on the same request (config key "vote.minChangeIntervalSeconds", default 10 seconds) to reduce rapid automated flips; attempts inside the interval SHALL return 409 Conflict.
 - Fraud and anomaly signals (AI-candidate): the system SHALL emit signals suitable for automated anomaly detection: unusually high vote creation rate for a single request, high proportion of votes from a single IP/subnet, repeated idempotencyKey reuse from different principals, and concentration of activity from newly created accounts. The pipeline that consumes these signals MAY use ML-based detectors to surface suspicious patterns; detection and remediation policies (soft-block, quarantining vote counts, human review) SHALL be configurable and require human approval for final removal of votes.
 - Administrative controls: the system SHALL support staff actions to mark vote batches as quarantined, exclude quarantined votes from public counts, and publish audit reasons. Administrative overrides SHALL be recorded in the audit store as operations.
 - Account hygiene and eligibility checks: voting SHALL require an authenticated account that meets eligibility checks configurable at deployment (for example minimum account age, email verification). These checks SHALL be enforced before accepting a vote operation.
 - Monitoring and alerts: the system SHALL produce operational metrics and alerts for anomalous voting patterns (spike in votes per minute, spike in new accounts voting) for product/security teams to investigate.

- FR-007: [HYBRID] Role-aware voting policies, weights, and public visibility:
 - Role permission model: the system SHALL support at minimum these actor roles: external_customer, internal_employee, partner, admin. By default external_customer, internal_employee, and partner SHALL be permitted to vote; admin MAY vote but actions by admin shall be flagged.
 - Role weights and policy: the system SHALL apply a roleWeight to each vote for weighted_vote_sum computation. The mapping from actorRole to roleWeight SHALL be configurable (config key "vote.roleWeights"), and default roleWeights SHALL be deployment-defined (no hard-coded numeric defaults mandated by this requirement). The system SHALL use actorRoleAtTimeOfOperation (snapshot) to compute the weight for that vote so that subsequent role changes do not retroactively change historical vote weights.
 - Visibility controls: the system SHALL support configuration to include or exclude internal votes from public-facing vote totals and roadmap displays (config key "vote.excludeInternalFromPublic", default false). When internal votes are excluded from public totals, internal interfaces SHALL still surface the full weighted and unweighted metrics for internal teams.
 - Policy decision points requiring human configuration or judgment: decisions such as default numeric role weights, whether to expose internal votes publicly, and exact thresholds for anomaly-detection remediation are subject to product/ops policy and SHALL be treated as configurable parameters. These policy settings and recommended defaults SHALL be reviewed and approved by Product Management prior to production rollout.

### FUNC-003: Public Roadmap Publishing

- FR-008: [DETERMINISTIC] System MUST provide a public-facing roadmap page and a REST read API that list published roadmap items and expose a staff-controlled publish workflow. Data contract and fields: the public API and page JSON for each published roadmap item MUST include these required fields with types and validation:
 - item_id: UUID (canonical, stable)
 - title: string, max length 200 characters, must be UTF-8, no control characters
 - short_description: string, max length 1000 characters
 - status: enumeration with exact allowed values {Proposed, Under Review, Planned, In Progress, Complete, Archived}
 - published_order: integer, >= 1, persisted index used for ordered listing
 - last_updated: UTC ISO‑8601 timestamp (RFC 3339)
 - public_url: string, canonical path /roadmap/{item_id}
 - api_resource: string, canonical path /api/v1/roadmap/{item_id}
 - linked_request_ids: array of request_id strings (may be empty)
 - aggregated_metrics: object ( for schema and semantics)
 - owner_id: internal user id (UUID) of staff owner
 - visibility: enum {public, internal}
 - publish_version: integer (incremented on each successful publish)
 - diagnostics: object (may include link_state and error_code per linked request)
 - etag: string (opaque, returned in headers and JSON for client caching)
 - acceptance: published boolean (true for items currently visible on public page)
 - correlation_id: UUID assigned at publish operation
 - audit_reference: audit_id of the publish operation
 - aggregated_metrics.metrics_timestamp: UTC ISO‑8601 timestamp for metric freshness
 - Validation: title and short_description MUST be trimmed and validated before publish; any invalid field MUST prevent publish and return 400 with field-specific error codes.
 UI and API behavior:
 - Pagination: public UI and API list endpoints MUST paginate with default page_size=50 and support page_size up to 200. The UI MUST return page 1 (50 items) within 2 seconds at the 95th percentile under nominal load.
 - Read SLA: /api/v1/roadmap/{item_id} single-item GETs MUST respond within 500ms at the 95th percentile under nominal load.
 - Snapshot consistency: reads MUST present a consistent snapshot as of the last successful publish (publish_version) and MUST return the publish_version and an ETag header; clients MAY use ETag/If-None-Match for caching.
 - Access control: public read access MUST conform to the system auth model defined in FUNC-005; edit/management functions MUST be inaccessible to unauthenticated users. Editing and publish operations MUST require authenticated accounts with account.role in {InternalTeamMember, Administrator} and explicit assigned permissions roadmap.editor for edits and roadmap.approver for publish actions (permission assignment managed via the system's RBAC store referenced by FUNC-005).
 - Workflow and allowed status transitions (authoritative): Proposed -> Under Review; Under Review -> Proposed or Planned; Planned -> In Progress; In Progress -> Complete or Archived; any status may be set to Archived by roadmap.approver. Only users with roadmap.approver may transition an item into Planned, In Progress, Complete, or Archived for public visibility; roadmap.editor may create and edit draft content and set Under Review.
 - Publish semantics: a publish operation is an atomic process that (a) validates fields and linked_request_ids, (b) materializes aggregated_metrics, (c) writes an audit entry, (d) increments publish_version, and (e) flips published.acceptance to true for public visibility. Publish MUST complete within 5 seconds at the 95th percentile under nominal load; if publish fails, the system MUST record publish_result="failed", error_code, and leave prior published_version unchanged.
 - Error handling for unresolved links: if a linked_request_id cannot be resolved at render time, the roadmap item MUST include diagnostics.link_state per linked_request_id with one of {resolved, unresolved, deleted, anonymized}; for unresolved entries the diagnostics object MUST include error_code="REQUEST_NOT_FOUND" and a human-readable diagnostics.message; the UI and API MUST still render the item and MUST record exactly one audit entry per render-time failure including audit.action="render_link_failure" ( audit schema).
 - Performance & scale: the read API and page MUST be architected to support at least 100,000 published items while preserving pagination and SLAs by persisting published_order as an indexed column and storing materialized aggregated_metrics per published item.

- FR-009: [DETERMINISTIC] System MUST expose precise, rule-based prioritization signals on each public roadmap item and define exact aggregation rules and freshness SLAs. Aggregated metrics schema (stored as materialized fields on published items and returned in aggregated_metrics):
 - vote_count: non-negative integer; defined as the sum of active votes for all requests linked to the roadmap item where an active vote is the most recent non-revoked vote record for a unique voter_id for a request (see FUNC-002 FR-003 for vote mechanics). Votes whose record state is deleted or revoked MUST be excluded.
 - unique_voter_count: non-negative integer; defined as the count of distinct voter_id values that have at least one active vote on any request linked to the roadmap item.
 - request_count: non-negative integer; total count of linked_request_ids currently existing (exclude deleted requests).
 - request_volume_30d: non-negative integer; count of requests linked to the roadmap item with created_at within the trailing 30-day window.
 - metrics_timestamp: UTC ISO‑8601 timestamp indicating when aggregated_metrics were last computed.
 Calculation, update cadence and freshness:
 - Base rule: aggregated_metrics MUST be computed deterministically from request and vote records using the definitions above.
 - Update model: metrics MUST be updated in near real-time using event-driven propagation such that a vote or request change is reflected in aggregated_metrics within 30 seconds at the 95th percentile under nominal load. If event-driven update is unavailable, a periodic reconciliation job MUST run at least once every 5 minutes and update metrics.
 - Aggregation storage: aggregated_metrics MUST be materialized per published item to guarantee read SLAs; source of truth remains request and vote records.
 - Edge cases:
 - Deleted requests: if a linked request is deleted, implementation MUST remove it from request_count and related vote contributions and MUST write an audit entry noting removal and exclusion from metrics.
 - Anonymized requests: if a request is anonymized (author identity removed) the request_id remains resolvable as link_state="anonymized"; votes on anonymized requests MUST continue to contribute to vote_count and unique_voter_count unless specific vote records have been revoked or deleted.
 - Partial data: if some linked requests are unresolved at metric computation time, the aggregated_metrics MUST be computed over resolved requests only and metrics_timestamp MUST reflect the computation time; diagnostics SHOULD indicate incomplete aggregation if unresolved requests exist.
 - API exposure: aggregated_metrics MUST be included in both the single-item GET (/api/v1/roadmap/{item_id}) and list endpoints. The read API MUST document metric calculation rules and metrics_timestamp in the OpenAPI specification.

- FR-010: [DETERMINISTIC] System MUST enforce controlled roadmap editing and provide a tamper-evident audit trail with defined format, retention, and revert capabilities.
 - RBAC and permissions: edits and publishes MUST be performed only by authenticated accounts with account.role in {InternalTeamMember, Administrator} and with explicit permissions: roadmap.editor for save/edit/draft operations and roadmap.approver for publish/unpublish/revert operations; permission checks MUST reference the central RBAC implementation per FUNC-005.
 - Draft → review → publish workflow:
 - Create/edit: roadmap.editor may create items and save drafts with visibility=internal; drafts do not increment publish_version nor appear on public roadmap.
 - Review: roadmap.editor may mark a draft as Under Review; marking Under Review must record reviewer_id and review_timestamp fields.
 - Approve & Publish: only roadmap.approver may set status to Planned/In Progress/Complete/Archived and execute the publish operation which atomically validates, materializes metrics, writes audit entries, and increments publish_version.
 - Revert: roadmap.approver may revert to any prior publish_version; revert operation MUST create a new publish_version and audit entry documenting the previous and new publish_version.
 - Audit schema and storage (audit_log entry written for each edit, publish, unpublish, render failure, revert, and failed publish attempts):
 - audit_id: UUID
 - timestamp: UTC ISO‑8601
 - actor_id: UUID
 - actor_role: enum {InternalTeamMember, Administrator}
 - action: enum {create_draft, edit_draft, mark_under_review, approve_for_publish, publish, unpublish, revert_publish, render_link_failure, publish_failed, delete_linked_request, remove_link}
 - target_item_id: UUID
 - change_type: enum {minor_edit, major_edit, publish, unpublish, revert}
 - patch_diff: JSON Patch (RFC 6902) describing the change from previous saved state (may be null for create)
 - full_snapshot: full JSON snapshot of the item at time of audit (only stored on publish actions and revert operations; stored for drafts when requested)
 - rationale: string, optional, max length 2000 characters
 - correlation_id: UUID to link related audit entries
 - previous_publish_version: integer or null
 - publish_result: enum {success, failure} for publish actions
 - error_code: string, optional (useable values include PUBLISH_VALIDATION_ERROR, REQUEST_NOT_FOUND, METRICS_COMPUTATION_ERROR)
 - Diffs vs full snapshots: implementations MUST store JSON Patch diffs for every edit and MUST store a full_snapshot for every successful publish and every revert. Full_snapshot for publishes is the canonical content used for public rendering.
 - Retention: audit metadata and patch_diff entries MUST be retained for at least 3,650 days (10 years). Full_snapshot entries MUST be retained for at least 365 days. Storage and retention policies MUST be configurable by administrators.
 - Concurrency and atomicity: publish operations MUST acquire a publish lock per item to prevent concurrent conflicting publishes; reads MUST present the last committed publish_version snapshot. Concurrent draft edits by multiple editors MUST be managed via optimistic locking (e.g., edit etag) and produce merge-conflict errors (409) when conflicts are detected.
 - Acceptance and testability: audit entries MUST be queryable by audit_id, item_id, correlation_id, actor_id, and action. Acceptance tests MUST validate that every successful publish writes exactly one audit entry with publish_result="success" and that failed publishes write publish_result="failure" with error_code.

- FR-011: [DETERMINISTIC] System MUST provide traceability between roadmap items and originating request records, expose vote counts for traceability, and define deterministic behaviors for orphaned or privacy-modified requests.
 - Linked_request_ids semantics: linked_request_ids on a roadmap item are authoritative pointers to request records collected via submission interfaces defined in FUNC-001. Each linked_request_id MUST resolve to a request record or explicitly present diagnostics.link_state in {resolved, unresolved, deleted, anonymized} when returned by the roadmap API.
 - Traceability metadata and API:
 - For each linked_request_id the roadmap API MUST optionally expose a trace object (traced_by default=false; admin API may expose true) containing {request_id, trace_status, request_title, request_created_at, last_activity_at, contributing_vote_count, contributing_unique_voter_count}. Public API MUST not expose personally identifiable information beyond request_id and request_title unless the request author has consented.
 - The public roadmap MUST include aggregated vote counts and MUST include per-request contributing_vote_count when visibility and privacy policy permit; if privacy prevents exposure, the API MUST return trace_status="anonymized" and omit PII fields.
 - Handling orphaned/deleted/anonymized requests:
 - Deleted requests: if a linked_request_id is deleted, the roadmap item MUST remove its contribution to aggregated_metrics and set trace_status="deleted"; an audit entry MUST be written.
 - Anonymized requests: anonymization removes PII but preserves request existence; trace_status="anonymized"; votes remain included in aggregated_metrics unless vote records themselves are revoked or deleted.
 - Unresolvable references: for unresolvable ids the API MUST indicate trace_status="unresolved" and include diagnostics.error_code="REQUEST_NOT_FOUND".
 - Integration and index considerations: linkage between roadmap items and requests MUST be implemented using referential integrity where possible (foreign keys or deterministic indexed mapping) to support queries that trace roadmap decisions back to request records and vote history. Implementations MUST support queries that list all roadmap items referencing a request_id and all request_ids referenced by a roadmap item.
 - Retention of traceability metadata: the traceability mapping and minimal per-linked-request metrics MUST be retained for the same period as audit metadata (3,650 days) to enable historical reconciliation and success-metric reporting.

### FUNC-004: Prioritization Signals & Decision Tracking

- FR-012: [DETERMINISTIC] System MUST surface prioritization signals to internal product and decision-making teams and define exact semantics for each surfaced metric. Required signals, semantics, aggregation, freshness, presentation channels, error handling, and acceptance tests:
 - Definitions:
 - unique_voter_count: count of distinct, active authenticated account identifiers (account_id) that currently hold an active vote on the request. "Active vote" is a vote record with state = Active and not superseded by a later vote withdrawal. Each account_id may contribute at most one active vote per request at any time. Accounts flagged soft-deleted or suspended in the account store (refer to FUNC-005 auth/role model) MUST be excluded from unique_voter_count.
 - vote_count: total number of active vote records for the request (includes internal and external actors). vote_count may be >= unique_voter_count when multi-vote types are supported; current implementation constrains one active vote per account (see unique_voter_count).
 - submission_volume: count of new request records created within a configurable rolling window. Default window = 7 days; allowed range = 1–90 days. Window boundaries are computed in UTC and inclusive of aggregation_window_start_utc and exclusive of aggregation_window_end_utc.
 - vote_trends_over_time: time-series where each point is an aggregation of vote_count (and unique_voter_count) deltas and percent-change versus the immediately preceding configured window. Default granularity = 1 day; allowed granularities = 1 hour, 1 day, 7 days, 30 days. Percent-change = (current_window_value − prior_window_value) / max(prior_window_value, 1) to avoid division by zero; trend deltas are integer differences.
 - Freshness and measurement:
 - All signals MUST include freshness metadata: last_event_timestamp_utc (newest source event timestamp included), data_as_of_utc (wall-clock timestamp when aggregated data was emitted), and processing_latency_ms = data_as_of_utc − last_event_timestamp_utc measured in milliseconds.
 - Default data-freshness SLA: maximum allowed processing_latency_ms = 300000 ms (5 minutes); SLA is configurable per-tenant. The system consistency model is eventual consistency bounded by the configured SLA; this model MUST be documented in the dashboard and read API.
 - Completeness_boolean: boolean indicating whether source event ingestion up to data_as_of_utc is complete; true means all source events with timestamp <= last_event_timestamp_utc are included.
 - Presentation channels and contracts:
 - Internal triage dashboard: supports sorting and filtering by product, component, timeframe, requester segment, status (see FUNC-006 and FUNC-003), and configured granularity. Dashboard MUST surface the defined metrics and freshness metadata.
 - Exports: CSV and JSON formats conforming to stable schema with fields: request_id, metric_name, metric_value, aggregation_window_start_utc, aggregation_window_end_utc, granularity, as_of_timestamp_utc (alias for data_as_of_utc), completeness_boolean, last_event_timestamp_utc, processing_latency_ms.
 - Read API: endpoint (e.g., GET /api/internal/requests/{requestId}/signals) MUST return the same fields and schema as exports. API MUST use standard HTTP status codes; error responses MUST include payload fields: error_code (machine-readable), message (human readable), details (optional diagnostics), and timestamp_utc.
 - Edge cases and semantics:
 - Withdrawn requests: requests with status = Withdrawn ( state machine) MUST be excluded from submission_volume and from active-vote counts unless a configuration flag include_withdrawn_signals = true is set and documented.
 - Deferred and Reopened states: votes cast while request is Deferred or Reopened are included in active counts; trend calculations MUST preserve continuity across state transitions.
 - Timezone and boundary: all time computations in UTC; rolling windows align to UTC boundaries.
 - Acceptance tests:
 - Verify unique_voter_count semantics including exclusion of suspended/soft-deleted accounts.
 - Verify vote_trends_over_time percent-change and delta computations for all supported granularities and zero-prior-window handling.
 - Verify rolling-window boundaries computed in UTC and configurable window lengths.
 - Verify exports and API responses include required metadata fields and conform to schema.
 - Verify processing_latency_ms and completeness_boolean semantics and that data-freshness SLA enforcement is testable (simulate lagging ingestion and observe completeness_boolean=false and latency > SLA).
 - Verify API error payloads for invalid parameters, partial data scenarios, authorization failures, and concurrent read-after-write scenarios.

- FR-013: [DETERMINISTIC] System MUST implement, enforce, and audit a canonical submission→decision lifecycle state machine with exact transition rules, role-to-transition mappings, concurrency controls, immutable-rationale semantics, and acceptance tests.
 - Canonical states (explicit): Submitted, Under Review, Deferred, Accepted, Rejected, Reopened, Withdrawn. (Terminal states: Accepted, Rejected, Withdrawn.)
 - Allowed transitions and required actor roles:
 - Submitted -> Under Review: allowed actors = triage_reviewer, internal_employee with triage permission (see FUNC-005). Transition may be automatic on first assignment.
 - Under Review -> Accepted | Rejected | Deferred: allowed actors = decision_owner OR Administrator. Transition to Accepted or Rejected MUST record decision_owner_id if performed by a reviewer acting on behalf of a decision_owner.
 - Deferred -> Under Review | Withdrawn: allowed actors = decision_owner, triage_reviewer, Administrator.
 - Accepted | Rejected -> Reopened: allowed actors = decision_owner, Administrator; Reopened may only occur if new substantive information is recorded and MUST record reopen_reason.
 - Any state -> Withdrawn: allowed actors = original_requester OR Administrator.
 - Reopened -> Under Review: allowed actors = triage_reviewer, decision_owner.
 - Immutable rationale and required fields:
 - Transitions into terminal states (Accepted or Rejected) MUST include an immutable_rationale boolean flag and rationale_text (UTF-8 string) that becomes permanent in the audit log and cannot be modified except by an Administrator who MUST append a superseding rationale; original rationale_text remains part of the audit trail.
 - Rationale for Deferred and Withdrawn transitions SHOULD be recorded; when present it is appended to the audit trail but is not required to be immutable.
 - Concurrency and conflict handling:
 - All state updates MUST use optimistic concurrency control via integer version (request_record_version). If a write is based on a stale version, the API MUST return HTTP 409 Conflict with payload containing current_state, request_record_version, and recommended next steps.
 - The system MUST surface concurrent-edit detection to UI actors and provide the latest state and pending transitions to avoid lost updates.
 - Enforcement and authorization:
 - Role-to-transition mappings MUST be enforced by the authorization layer (see FUNC-005). If an actor lacks permission, API returns 403 Forbidden.
 - Bulk transitions (multiple requests) allowed only for Administrator role and MUST be atomic per-request (not cross-request).
 - Auditing:
 - Every transition MUST create an immutable audit record (see FUNC-006) with fields: audit_id, request_id, previous_state, new_state, actor_id, actor_role, timestamp_utc, rationale_text (if provided), immutable_rationale boolean, request_record_version_after.
 - Acceptance tests:
 - Verify each allowed transition prevents disallowed roles and returns correct HTTP codes for unauthorized attempts.
 - Verify terminal-state transitions require immutable_rationale and that rationale is immutable (attempts to modify result in rejection and audit record preserved).
 - Verify optimistic concurrency behavior: concurrent update attempts produce 409 Conflict with current state and version.
 - Verify audit records created for every transition contain required fields and are immutable (see FUNC-006 for storage expectations).
 - Verify behavior for withdrawn/deferred/reopened flows per role mappings and that reopened_count is incremented and exposed in the audit export.

- FR-014: [DETERMINISTIC] System MUST measure, report, and export decision-time metrics (including median time-to-decision) with precise inclusion criteria and handling of reopened/withdrawn records; provide measurement configuration and acceptance tests to support the success metric (median < 30 days).
 - Decision-time definition:
 - decision_time for a request = timestamp_of_first_terminal_decision_utc − submission_timestamp_utc where first terminal decision is the first transition of the request into Accepted or Rejected after initial submission.
 - If a request is Withdrawn before any terminal decision, it is excluded from decision-time calculations unless include_withdrawn_in_metrics configuration = true (documented and default = false).
 - If a request is Reopened after a terminal decision, subsequent terminal decisions are recorded as separate decision events; decision_time metric for the success criterion uses the first terminal decision only. A separate metric "time_to_final_decision" MAY be collected to represent time to final terminal state after reopens.
 - Aggregation windows and filters:
 - Default reporting window = 30 days rolling; allowed ranges configurable. Measurements computed in UTC. Metrics MUST support filtering by product, component, requester_segment, decision_owner_id, and time-of-submission.
 - Only requests that reached a terminal decision within the measurement window are included in median calculations for that window.
 - Export and API schema:
 - Decision-time metric exports MUST include fields: request_id, submission_timestamp_utc, first_terminal_decision_timestamp_utc, decision_time_ms, decision_owner_id, decision_type (Accepted|Rejected), excluded_reason (if excluded), as_of_timestamp_utc.
 - SLA alignment to success criteria:
 - Product-level success criteria is median(decision_time) < 30 days; the system MUST compute median(decision_time_ms) over the configured measurement window and expose it on dashboards and via API.
 - Acceptance tests:
 - Verify decision_time calculation using test cases including normal accepted/rejected flows, requests withdrawn prior to decision, requests reopened after initial decision.
 - Verify median computation and filtering by dimensions; confirm exclusions per configuration.
 - Verify exported schema fields and accuracy of timestamps and decision_owner_id.
 - Verify that metric generation is resilient to late-arriving events and that completeness_boolean and processing_latency_ms semantics (FR-012) indicate metric reliability.

- FR-015: [HYBRID] System MUST link decisions to roadmap items with explicit cardinality, link semantics, event schemas, visibility rules, and configurable automatic-sync behavior; provide audit trail and acceptance tests.
 - Cardinality and referential constraints:
 - A decision record MAY link to zero or more roadmap_item identifiers. A roadmap_item MAY be linked from zero or more decision records. Links MUST enforce referential integrity: link creation fails with 422 Unprocessable Entity if roadmap_item_id does not exist or belongs to a different product without explicit cross-product-link permission.
 - Link types and propagation semantics:
 - Supported link_type enum: {implements, informs, duplicates, related}. Link creation MUST record propagation_action enum: {none, create_or_update_roadmap_item, annotate_roadmap_item}.
 - Default behavior: when a decision transitions to Accepted and propagation_action = create_or_update_roadmap_item AND auto_sync_enabled for the product, the system MUST create a new roadmap_item draft (if none exists) or append a decision reference to an existing roadmap_item, and add an audit annotation. Automatic creation/update is configurable per-product and requires Administrator or decision_owner permission to enable.
 - Propagation MUST NOT automatically publish a roadmap_item to the public roadmap (publication is a separate action controlled by FUNC-003).
 - Visibility, privacy, and public APIs:
 - Link visibility MUST respect roadmap_item visibility: if roadmap_item is unpublished/internal, public read APIs and public exports MUST omit internal roadmap identifiers and provide an anonymized reference or public_visibility = false. Internal APIs and staff dashboards MUST show full linkage.
 - Event schema and emissions:
 - Each link action MUST emit a decision_link_event with fields: event_id, decision_id, roadmap_item_id, link_type, propagation_action, actor_id, actor_role, timestamp_utc, previous_link_state (if updating), new_link_state.
 - Event delivery: events MUST be written to the canonical audit store (see FUNC-006) and emitted to the internal event bus for downstream subscribers; consumers MUST be able to reconcile idempotently using event_id.
 - Authorization and controls:
 - Only decision_owner, Administrator, or users with explicit roadmap-link permission (see FUNC-005) may create or remove links. Attempts by unauthorized actors return 403 Forbidden.
 - Acceptance tests:
 - Verify cardinality rules: create multiple links from one decision to many roadmap_items and from many decisions to one roadmap_item.
 - Verify referential integrity rejection on invalid roadmap_item_id and cross-product constraints.
 - Verify propagation_action outcomes for create_or_update_roadmap_item and annotate_roadmap_item with auto_sync_enabled true/false, and verify that publication remains a separate, explicit action.
 - Verify event schema fields present and events are persisted to audit store and emitted to event bus; verify idempotent replay using event_id.
 - Verify visibility enforcement: internal links hidden from public APIs/exports and shown in internal dashboards.
 - Verify that link creation/removal is audited and that audit records reference FUNC-006 auditing storage requirements.

### FUNC-005: User Roles, Access & Permissions

*[Unit FUNC-005 — content pending. Intent: Define all FRs for user roles and access control (FR-016 through FR-017): support customers and internal teams as roles, role-based permissions for submit/vote/edit, public read access to roadmap, and authentication/identity integration considerations.]*

### FUNC-006: Auditing, Change History & Data Export

- FR-018: [DETERMINISTIC] System MUST maintain a single, auditable source-of-truth for every feature request; implementation details and acceptance criteria:
 - Canonical record: a single authoritative record per request_id (UUID v4) MUST be stored in an ACID-compliant primary datastore. The canonical record schema MUST include the following typed fields and semantics:
 - request_id: UUID v4 (primary key)
 - stable_url: persistent, human-readable permalink (string) that resolves to the canonical record for the configured retention period
 - creator_id: string (internal account id or external account id)
 - created_at, updated_at: UTC timestamps (ISO 8601)
 - status: enumerated type aligned with lifecycle in FUNC-004
 - title: string (max 1024 chars)
 - summary: string (max 8192 chars)
 - metadata: JSON object with a defined schema:
 - category: string (nullable)
 - priority_hint: string (nullable)
 - client_tags: array[string]
 - custom_properties: map[string->string] (allowed free-form key/value pairs)
 - vote_summary: object with fields that mirror the signal semantics defined in FUNC-004 (for example unique_voter_count:int, votes_total:int, weighted_score:decimal) — exact vote field semantics are authoritative in FUNC-004 and MUST be referenced at implementation time
 - attachment_refs: array of objects { attachment_id: UUID, filename: string, content_type: string, size_bytes: integer, storage_path: string }
 - comment_count: integer
 - current_decision_id: nullable reference to decision record
 - revision_seq: integer (monotonic, starts at 1)
 - Append-only revision ledger: every committed change to a canonical record MUST be recorded in an append-only ledger. Each ledger entry MUST include:
 - ledger_entry_id (UUID), request_id, revision_seq, author_id, change_type (CREATE/UPDATE/DELETE/REDACT/DECISION), timestamp (UTC ISO 8601), changed_fields (list of field names or null if full snapshot), snapshot (complete canonical record JSON OR null if diff-only), and change_summary (human-readable string).
 - Ledger MUST support queries filtered by request_id, revision_seq range, author_id, change_type, and timestamp range, and MUST support pagination and deterministic sort (ascending by revision_seq).
 - Atomicity and write semantics: updates to the canonical record and the corresponding ledger entry MUST be applied in a single transactional unit; implementations MUST use a single transactional commit or an equivalent atomic two-phase write with compensating rollback so that either both the canonical record and ledger entry are committed, or neither is.
 - Concurrency control: the system MUST enforce optimistic concurrency using revision_seq (or equivalent etag). Update requests MUST include expected_revision_seq; mismatches MUST return 409 CONFLICT. The system MUST NOT silently perform last-write-wins; concurrent write collisions MUST be surfaced and require client retry or explicit merge. Where automated merge is offered it MUST be governed by documented, deterministic merge rules; otherwise merges MUST be manual.
 - Tamper-evidence and verification: every ledger entry MUST include entry_hash (SHA‑256 of the canonical JSON representation of the ledger entry) and chain_hash computed as SHA‑256(concat(previous_chain_hash, entry_hash)) to form a verifiable chain per request_id. The system MUST persist periodic chain_hash manifests to immutable storage with WORM semantics (object-lock or equivalent) and expose a verification API that, given request_id and revision_seq, returns:
 - the persisted chain_hash for the sequence,
 - the sequence of entry_hash values up to that revision_seq (or a cryptographic proof/merkle root), and
 - a boolean verification result computed by recomputing the chain.
 - The verification API MUST return a deterministic error if verification fails.
 - Use of externally anchored signatures (for example, timestamping or third-party ledger anchoring) is allowed but optional; chain_hash manifest persistence is mandatory.
 - High-availability, replication and backups: primary datastore and ledger MUST be deployed with synchronous or strongly-consistent replication within a region and asynchronous cross-region replication for disaster recovery. Backups MUST be taken at least daily, encrypted at rest, accompanied by a verifiable checksum manifest (SHA‑256), and stored under the same immutable manifest policy. Documented default RPO ≤ 24 hours and RTO ≤ 4 hours for critical metadata MUST be provided; operational teams MAY set stricter SLAs per deployment.
 - Retention, redaction and GDPR: the system MUST support configurable retention and data-erasure workflows:
 - Default configurable retention window for ledger and canonical records: 7 years (product default). This default is configurable by tenants; specific regulatory retention requirements override defaults.
 - Full deletion of ledger entries that would break auditability is not permitted. Personal data subject to erasure requests MUST be redacted or pseudonymized in canonical records and ledger entries while preserving an audit marker entry that records the redaction action (change_type = REDACT) with author_id and timestamp.
 - Implementations MUST provide a documented erasure flow and proof that personal data was redacted (e.g., redaction entry and post-redaction chain_hash).
 - Stability guarantees: stable_url and attachment identifiers referenced by attachment_refs MUST remain resolvable for the configured retention window. When attachments cannot be retained (e.g., per retention or user erasure), attachment_refs MUST be replaced by a redaction marker that indicates reason and timestamp.
 - Acceptance tests: canonical record read after write, ledger entry present and queryable by revision_seq, verification API returns true for chain_hash over known sequence, and concurrent write attempts produce 409 and require client-driven resolution.

- FR-019: [DETERMINISTIC] System MUST record changelogs and decision-related history with explicit decision records and provide query and access controls:
 - Decision records: every decision that affects a request (for example accept/reject/triage/roadmap_move) MUST be stored as a first-class decision record with fields:
 - decision_id: UUID
 - request_id: UUID
 - decision_owner_id: string (internal user account)
 - decision_status: enumerated value (Accepted/Rejected/Deferred/Triage/Other)
 - decision_timestamp: UTC ISO 8601
 - decision_rationale: string (max 32,768 chars)
 - decision_documents: array of document refs (document_id, filename, content_type, storage_path)
 - linked_roadmap_item_id: nullable reference
 - Linkage to canonical record: canonical.current_decision_id MUST reference the latest decision_id; the append-only ledger MUST record decision changes as change_type=DECISION and include decision_id details.
 - Queryable change history: APIs MUST support filtering ledger and decision records by request_id, decision_id, author_id, decision_owner_id, change_type, status transition (from/to), and timestamp ranges. Sorting by revision_seq or decision_timestamp MUST be supported. Pagination and deterministic ordering are required.
 - Access control and authorization: access to full changelog and decision rationale is governed by role-based permissions defined in FUNC-005:
 - InternalTeamMember and Administrator roles MAY view full changelog and decisions for requests within their scope.
 - External customers MAY view canonical record and changelog entries limited to non-sensitive public fields for public requests; request owners MAY view their own full changelog if permitted by tenant policy.
 - Exporting decision rationale or changelog entries that contain PII MUST be restricted to Administrator or explicitly authorized roles and MUST generate an audit log entry recording the export actor, filters used, and timestamp.
 - Notifications and linkage: when a decision record is created or updated, the system MUST trigger the notification workflow described in FUNC-008 (for submitter and affected stakeholders) and include a reference to the decision_id in notification payloads.
 - Acceptance tests: changes to status or decision produce a ledger entry of type DECISION, decision record is queryable and linked from canonical record, and permissioned users can retrieve full decision_rationale while unauthorized users are denied.

- FR-020: [HYBRID][DETERMINISTIC] System MUST provide export and bundle capabilities for canonical records, changelogs, decisions, and attachments via both UI and API with deterministic formats, access controls, and performance constraints:
 - Export interfaces:
 - Synchronous exports: REST API endpoints and UI flows that produce small exports (configurable default threshold: <= 5 MB and <= 1,000 ledger entries) synchronously with response-time SLA: first byte <= 3 seconds and complete response <= 30 seconds under normal load.
 - Asynchronous exports: for exports exceeding synchronous thresholds or large result sets (default thresholds: >5 MB, >1,000 ledger entries, or >100 attachments), provide an asynchronous export job API that accepts export parameters and returns a job_id. The system MUST process large exports in background, provide job status (pending/processing/failed/complete), progress percent, and on completion provide a signed download URL (pre-signed) valid for a configurable window (default 7 days).
 - Export formats and content:
 - JSON (canonical): complete snapshot of selected canonical records plus full ledger entries; newline-delimited JSON allowed for streaming large exports.
 - CSV: flattened record list suitable for tabular analysis (limited to fields that can be safely flattened; nested data such as comments/attachments summarized by counts or IDs).
 - PDF bundle: single human-readable document that must include:
 - a cover page with request_id, title, creator_id (anonymized on request), created_at, stable_url and export_checksum (SHA‑256 of bundle)
 - table of contents
 - canonical snapshot formatted for readability
 - chronological changelog entries with author_id, timestamp, change_summary, and link or inline excerpt of decision_rationale
 - Appendix listing attachment filenames, sizes and checksums; for text and image attachments below the embed threshold (configurable default 2 MB) the content MUST be embedded inline; larger binaries MUST be included in a ZIP alongside the PDF or referenced by the signed download URL.
 - ZIP bundle: optional for binary attachments; when requested, produce a single ZIP that contains the PDF bundle plus embedded attachments.
 - Export parameters and filters: exports MUST accept filters: request_id(s), date range on revisions, change_type(s), decision_id(s), status, author_id, include_attachments boolean, attachment_size_limit, and pagination parameters. APIs MUST validate filters and return 400 for invalid combinations.
 - Attachment handling: attachments referenced in attachment_refs MUST be stored in immutable object storage keyed by attachment_id; exports MUST include either embedded content (when size and content-type permit) or a captured snapshot file within the export bundle. Exports MUST include checksums (SHA‑256) for each attachment included.
 - Stability and URLs: stable_url for canonical records and attachment identifiers MUST remain resolvable for the configured retention window; signed download URLs issued for async exports or attachments MUST expire after their configured TTL.
 - Performance, throttling and concurrency:
 - Limit concurrent asynchronous export jobs per tenant (default 10 concurrent); rate-limit export creation (default 60/minute per tenant). Implement fair-queuing and backpressure.
 - Ledger query first-page latency target: <= 200ms for queries that return <= 1,000 entries under normal load.
 - Error handling and retries:
 - Define explicit HTTP error semantics: 400 invalid parameters, 401 unauthorized, 403 forbidden, 404 not found, 409 conflict (concurrency), 413 payload too large for synchronous export, 429 rate limit exceeded, 500 internal server error.
 - Asynchronous export jobs MUST implement retry with exponential backoff up to 3 attempts for transient failures and MUST record failure reason in job status.
 - Auditing of export operations:
 - Every export request (sync or async) MUST generate an audit entry recording actor_id, tenant_id, filters used, export_format, requested_at, completed_at, resulting_bundle_checksum (SHA‑256), and whether PII was included or redacted.
 - Audit entries for exports MUST themselves be written to the append-only ledger or a separate audit ledger and be queryable by Administrator role.
 - Privacy and access constraints:
 - Exports containing PII or full changelog contents MUST be restricted to authorized roles per FUNC-005. If a tenant policy requires anonymization, exports MUST provide an anonymize option which replaces user-identifying fields with stable pseudonyms and records an anonymization entry in the ledger.
 - Acceptance tests:
 - Export with small dataset completes synchronously within SLA and matches canonical and ledger data.
 - Large export completes asynchronously and the bundle checksum validates against exported chain_hash/entry_hash proofs.
 - Unauthorized export attempts are denied and produce audit log entries.
 - Redaction requests produce a REDACT ledger entry and exported bundle reflects redaction per policy.

### FUNC-007: Analytics, Metrics & Reporting

- FR-021: [DETERMIN

### FUNC-008: Notifications & Communications

- FR-022: [HYBRID][DETERMINISTIC] System MUST notify submitters and affected stakeholders when a submission's status or decision changes. Affected stakeholders are: the original submitter(s)

### FUNC-009: Integration Points & Future Links

- FR-023: [DETERMINISTIC] System MUST publish and maintain a versioned, machine-readable integration contract exposing a documented API surface for optional future integrations. The contract MUST be published as OpenAPI v3 JSON at a stable URL and include: endpoint list, request/response schemas, example calls, error model, authentication/scopes, pagination/sorting/filtering rules, webhook event catalog and payload schemas, rate limits, and a deprecation/change-log policy with minimum 90 calendar days for breaking changes.

- FR-024: [DETERMINISTIC] System MUST implement the primary REST/JSON surface under a versioned path (/api/v

## Use Case Analysis

### UC-001: Submit Feature Request

*[Unit UC-001 — content pending. Intent: Describe the 'Submit Feature Request' use case and flows, and include a PlantUML use case diagram with @startuml/@enduml tags showing actors (Customer, Internal User, System) and the system boundary.]*

### UC-002: Vote on Feature Request

*[Unit UC-002 — content pending. Intent: Describe the 'Vote on Feature Request' use case and flows, and include a PlantUML use case diagram with @startuml/@enduml tags showing actors (Customer, Internal User, System) and the system boundary.]*

### UC-003: Review and Triage Feature Requests (Internal)

*[Unit UC-003 — content pending. Intent: Describe the internal 'Review/Triage' use case and flows, and include a PlantUML use case diagram with @startuml/@enduml tags showing actors (Product Manager, Decision Owner, System) and the system boundary.]*

### UC-004: Publish and View Public Roadmap

*[Unit UC-004 — content pending. Intent: Describe the 'Publish/View Public Roadmap' use case and flows, and include a PlantUML use case diagram with @startuml/@enduml tags showing actors (External Customer, Internal User, System) and the system boundary.]*

### UC-005: Decision Tracking and Notifications

*[Unit UC-005 — content pending. Intent: Describe the 'Decision Tracking & Notifications' use case and flows, and include a PlantUML use case diagram with @startuml/@enduml tags showing actors (Submitter, Product Team, System) and the system boundary.]*

### UC-006: Analyze Prioritization Signals & Reporting

*[Unit UC-006 — content pending. Intent: Describe the 'Analyze Signals & Reporting' use case and flows, and include a PlantUML use case diagram with @startuml/@enduml tags showing actors (Product Analyst, Product Manager, System) and the system boundary.]*

## Risks & Mitigations

*[Unit RISK-001 — content pending. Intent: List the key risks from the BRD (low adoption, manipulable votes, missing decision owner/SLA) and propose mitigations aligned with owners named in the BRD.]*

## Constraints & Assumptions


1. Adoption and signal-volume assumption
- Assumption: both external customers and internal teams will actively adopt and use the portal for submitting requests and casting votes.
- Rationale & impact: voting and discovery signals require a minimum volume of active users to produce reliable prioritization signals and to meet the BRD success metric (median decision time < 30 days). Low adoption is a primary risk; see RISK-001 for mitigation actions that must be triggered when thresholds are not met.
- Measurable thresholds and operational targets (initial release / pilot):
 - Adoption: >= 250 unique external customer users and >= 50 unique internal staff users in any rolling 30-day window, where each counted user performs at least one submit or vote action in that window.
 - Concurrency & capacity: sustain 200 concurrent active sessions (authenticated reads/writes) and support spikes to 1,000 concurrent sessions for up to 5 minutes without user-facing errors.
 - Rate limits: per-account write limit = 60 write actions/minute; per-account read limit = 600 reads/minute. Exceeding rate limits returns HTTP 429 with machine-readable error code RATE_LIMIT_EXCEEDED and a Retry-After header.
- Acceptance criteria (testable):
 - Adoption: telemetry query for unique_voter_count across a rolling 30-day window returns >=250 external and >=50 internal unique users; otherwise raise the adoption flag and execute mitigations in RISK-001.
 - Capacity: load test simulating 200 sustained concurrent sessions with realistic read/write mix completes with <=1% error rate and response-time targets met; spike test to 1,000 sessions for 5 minutes completes without user-facing errors.
 - Rate limiting: automated tests that issue 2× rate-limit volume receive HTTP 429 and Retry-After; system recovers to normal throughput after limit window.

2. Vote integrity, duplicate-detection and update semantics
- Assumption: vote records are a reliable prioritization signal only if technical safeguards and process rules enforce uniqueness, traceability, and detectable tampering.
- Required behaviours, definitions and limits:
 - Uniqueness: one active vote per authenticated account per request. The system must enforce a unique constraint on (request_id, account_id).
 - Duplicate attempts: casting an additional vote for the same request returns HTTP 409 Conflict with machine-readable error code VOTE_DUPLICATE and an explanatory message.
 - Vote update/delete semantics:
 - A user may update or remove their vote. Update operations MUST be explicit (PUT/PATCH/DELETE).
 - Updates must be versioned and guarded by optimistic concurrency: client-supplied If-Match header with resource version; concurrent write conflicts return HTTP 412 Precondition Failed with code VOTE_CONFLICT.
 - Deleted votes are recorded in the audit trail as a deletion event; the public aggregate may be recalculated but audit records retain the original event (pseudonymized per privacy rules).
- Acceptance criteria (testable):
 - Duplicate enforcement: integration test attempts duplicate vote and receives 409 + VOTE_DUPLICATE.
 - Concurrency: concurrent update test with stale If-Match receives 412 + VOTE_CONFLICT; successful update returns 200 with new version.

3. Tamper-evidence, auditability and export behaviour
- Assumption: audit logs and exports must provide tamper-evidence and reproducible provenance for all vote- and decision-related events.
- Required behaviours and technical capabilities:
 - Audit log schema: each event records timestamp (UTC), event_id, request_id, account_id (pseudonymized for external public exports), staff_actor_id (when applicable), action (create/update/delete/flag), source_ip, user_agent, client_request_id, and resource_version.
 - Tamper-evidence: audit store must be append-only and produce periodic signed manifests (SHA-256) for each daily bucket. Exported manifests must include the manifest signature.
 - Export semantics & partial failures: exports operate at object granularity and return 200 on fully successful export. If any object export fails, the API returns HTTP 207 Multi-Status with machine-readable codes per object and an overall code EXPORT_PARTIAL_FAILURE. The export job must produce a manifest listing successful and failed objects and a deterministic retry plan. Partial export results must not be applied to downstream systems unless the manifest indicates all critical objects succeeded.
- Acceptance criteria:
 - Audit integrity: automated verification compares daily manifest signature with stored manifest and detects any tamper attempts.
 - Export partial-failure handling: simulated partial export produces 207 with manifest and retry plan; downstream consumer can reconcile using the manifest.

4. Contested votes and abuse scenarios
- Edge-case handling and process constraints:
 - Contested votes: product staff must be able to mark a request as "contested" or "under-audit"; when marked, public aggregate vote counts for that request enter a read-only "pending" state and an audit entry is created. Resolution of contested flags requires an explicit decision recorded by the decision owner (see item 6).
 - Abuse detection thresholds:
 - If >100 distinct votes for the same request originate from the same IP address or same /24 subnet within a 24-hour window, the system must flag the request for review and temporarily freeze vote-count changes for that request until audit completes.
 - Suspicious account behaviour (e.g., >500 vote actions across distinct requests in 24 hours) triggers automated rate-limiting and a soft account suspension workflow: return HTTP 403 with code ACCOUNT_SUSPENDED_FLAGGED for write operations; staff-facing investigation UI is populated.
 - Verification & remediation: flagged events must be visible in an internal audit dashboard and a remediation workflow documented; mitigation actions are logged and audited.
- Acceptance criteria:
 - Abuse detection test harness triggers flags at defined thresholds and causes freeze and dashboard visibility.
 - Contested vote lifecycle test: marking contested freezes public counts, records audit entry, and requires decision owner approval to unfreeze.

5. Privacy, retention and data access controls
- Constraints and requirements:
 - Telemetry and retention:
 - Required telemetry: MAU, DAU, submission_count, vote_count, unique_voter_count, time_to_decision, task_success events, error_rates (4xx/5xx).
 - Raw event retention: 90 days. Aggregated metrics retention: 24 months.
 - Daily and 30-day aggregates must be available and queryable; daily aggregate generation must complete within 4 hours of the UTC day boundary.
 - PII handling and deletion:
 - Public exports must not contain PII without explicit user consent. Internal staff access to PII requires role-based authorization and is logged.
 - Account deletion request: votes associated with deleted accounts must be pseudonymized (replace account_id with a one-way hash) in audit logs for 90 days, after which personally-identifiable fields must be purged in compliance with privacy policy.
- Acceptance criteria:
 - Retention and aggregation: telemetry APIs return required aggregates and raw data for the specified retention periods; daily aggregations complete within 4 hours.
 - Privacy: export tests confirm PII exclusion for public exports unless consent flag is present.

6. Decision owner, approval workflow and SLA (open question / blocking assumption)
- Explicit assumption and constraint: a named decision owner and an approval workflow with SLAs must be defined and assigned before production go‑live. The system and reporting rely on a human-owner workflow to transition items from voting to a public roadmap state.
- Minimum required workflow and SLA (must be adopted or release is blocked):
 - RACI: Product Management must assign a named Decision Owner (role/title sufficient) and document the approval workflow, accessible to system UI and integrations, prior to production launch.
 - SLA targets: triage of newly qualifying requests within 3 business days; formal decision (accepted/rejected/more-info) recorded within 30 calendar days (median target <30 days across decisions).
 - System support: the portal must provide UI affordances and APIs to record decision owner, decision timestamp, decision rationale, and link to roadmap entry; all decision events recorded in audit log.
- Acceptance criteria:
 - Governance: an approved RACI document naming the Decision Owner and the documented approval workflow/SLA exists and is referenced in release checklist.
 - System capability: the portal records decision events per SLA, and a telemetry check shows median time_to_decision <30 days for a representative sample after launch; if the Decision Owner is not assigned before launch, the release shall be blocked per product governance.

7. Interoperability and out-of-scope interfaces
- Constraint: full ticketing backends, SLA management systems, and other non-feature-request workflows are out of scope for the initial deliverable (see BRD Scope Out). Integrations with external ticketing systems are allowed only via documented, opt-in export connectors and must honor privacy and export partial-failure semantics described above.
- Acceptance criteria:
 - Any integration connector used in pilot is documented, opt-in, and passes export partial-failure tests and privacy checks.

8. Testability and operational checks (summary)
- All constraints above include explicit, testable acceptance criteria. Tests to be included in the release checklist:
 - Adoption telemetry threshold validation (rolling 30-day).
 - Load and spike performance tests for concurrency and error-rate targets.
 - Duplicate vote and concurrency conflict tests (409 and 412 behaviours).
 - Audit-manifest integrity and export partial-failure handling (207 responses).
 - Abuse/flagging scenarios and contested-vote lifecycle tests.
 - Privacy/export tests ensuring no PII in public exports.
 - Governance check: Decision Owner and SLA documented (blocking if absent).

Notes:
- Where mitigations are required due to thresholds not being met or abuse detected, refer to RISK-001 for the approved mitigation runbooks and owners.
