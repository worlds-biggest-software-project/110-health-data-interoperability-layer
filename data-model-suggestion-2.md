# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Health Data Interoperability Layer · Created: 2026-05-19

## Philosophy

This model treats every state change as an immutable event in an append-only event store. The event store is the single source of truth. Current state is derived by replaying events into materialized read models (projections) optimized for specific query patterns — a classic CQRS (Command Query Responsibility Segregation) architecture.

In healthcare, this pattern is particularly compelling because HIPAA mandates comprehensive audit trails, regulators demand the ability to reconstruct "what was known at time X," and the message pipeline (HL7 v2 messages arriving, being transformed, routed, and acknowledged) is inherently event-driven. Rather than bolting audit logging onto a mutable relational schema, the event store IS the audit trail. Every FHIR resource creation, update, deletion, patient match decision, HL7 v2 message transformation, and X12 transaction is captured as a domain event with full provenance.

This approach is used in financial trading systems, banking ledgers, and is advocated for Electronic Medical Record systems by industry architects (see InfoQ's "Healthy Architectures" series on CQRS/ES for EMR). The Kodjin FHIR Server uses event-driven architecture for exactly these reasons.

**Best for:** Organizations that require complete temporal auditability, regulatory reconstruction ("show me this patient's record as it was on March 1"), and real-time event streaming to analytics/AI pipelines.

**Trade-offs:**
- (+) Complete, immutable audit trail by design — HIPAA compliance is inherent, not bolted on
- (+) Temporal queries are natural: reconstruct any state at any point in time
- (+) Event stream feeds real-time analytics, AI anomaly detection, and compliance monitoring
- (+) Write operations are simple appends — high write throughput
- (+) Easy to add new read models (projections) without changing the write path
- (-) Read models require eventual consistency — not all queries are immediately consistent
- (-) Event schema evolution is harder than ALTER TABLE — requires careful versioning
- (-) Rebuilding projections from scratch can be time-consuming for large event stores
- (-) Development team needs familiarity with event sourcing patterns
- (-) Debugging requires tracing through event sequences rather than inspecting current state

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| HL7 FHIR R4 | FHIR resource mutations are captured as ResourceCreated/Updated/Deleted events; read models project current FHIR state |
| HL7 v2.x | Each inbound HL7 v2 message is a MessageReceived event; transformation steps are TransformationApplied events |
| HIPAA Security Rule | The event store IS the audit trail — every PHI access, modification, and export is an immutable event |
| IHE PIXm / PDQm | Patient matching events (MatchProposed, MatchConfirmed, MatchRejected, PatientMerged) capture the full identity resolution lifecycle |
| SMART on FHIR | Authorization events (TokenIssued, TokenRevoked, ScopeGranted) provide a complete OAuth2 audit trail |
| X12 EDI | EDI transactions captured as EDIReceived, EDIParsed, EDIRouted events with full payload preservation |
| USCDI v3 | Compliance checks are events (ComplianceCheckRun, ComplianceViolationDetected) enabling temporal compliance reporting |
| FHIR AuditEvent | Read model projects events into FHIR AuditEvent resources for standards-compliant audit reporting |

---

## Event Store (Source of Truth)

```sql
-- ============================================================
-- CORE EVENT STORE (append-only, immutable)
-- ============================================================

CREATE TABLE event_store (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    stream_id       UUID NOT NULL,                   -- Aggregate root ID
    stream_type     VARCHAR(50) NOT NULL,            -- FhirResource, Patient, Channel, X12Transaction
    event_type      VARCHAR(100) NOT NULL,           -- e.g., ResourceCreated, PatientMatched
    event_version   INTEGER NOT NULL,                -- Sequence within stream (optimistic concurrency)
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    actor_id        VARCHAR(255) NOT NULL,           -- User, system, or device ID
    actor_type      VARCHAR(30) NOT NULL,            -- user, system, device, client_app
    actor_ip        INET,
    correlation_id  UUID,                            -- Links related events across aggregates
    causation_id    UUID,                            -- The event that caused this event
    metadata        JSONB NOT NULL DEFAULT '{}',     -- Transport headers, source info
    payload         JSONB NOT NULL,                  -- Event-specific data (see examples below)
    schema_version  INTEGER NOT NULL DEFAULT 1,      -- For event schema evolution
    UNIQUE (stream_id, event_version)                -- Optimistic concurrency control
) PARTITION BY RANGE (occurred_at);

-- Partition per month for retention management
CREATE TABLE event_store_2026_01 PARTITION OF event_store
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE event_store_2026_02 PARTITION OF event_store
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
-- ... (auto-create partitions via pg_partman or cron)

CREATE INDEX idx_event_stream ON event_store(stream_id, event_version);
CREATE INDEX idx_event_type ON event_store(tenant_id, event_type, occurred_at DESC);
CREATE INDEX idx_event_correlation ON event_store(correlation_id) WHERE correlation_id IS NOT NULL;
CREATE INDEX idx_event_tenant_time ON event_store(tenant_id, occurred_at DESC);

-- ============================================================
-- EVENT SNAPSHOTS (periodic state snapshots to avoid full replay)
-- ============================================================

CREATE TABLE event_snapshot (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    stream_id       UUID NOT NULL,
    stream_type     VARCHAR(50) NOT NULL,
    snapshot_version INTEGER NOT NULL,               -- Event version at time of snapshot
    state           JSONB NOT NULL,                  -- Serialized aggregate state
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, snapshot_version)
);

CREATE INDEX idx_snapshot_stream ON event_snapshot(stream_id, snapshot_version DESC);
```

### Event Payload Examples

```sql
-- ResourceCreated event payload:
-- {
--   "resource_type": "Patient",
--   "fhir_id": "pat-123",
--   "version_id": 1,
--   "resource_body": { ... full FHIR JSON ... },
--   "source_system": "http://hospital-a.org",
--   "profile_urls": ["http://hl7.org/fhir/us/core/StructureDefinition/us-core-patient"]
-- }

-- ResourceUpdated event payload:
-- {
--   "resource_type": "Observation",
--   "fhir_id": "obs-456",
--   "version_id": 3,
--   "resource_body": { ... full FHIR JSON ... },
--   "changes": [
--     {"path": "value.valueQuantity.value", "old": 120, "new": 125}
--   ]
-- }

-- HL7v2MessageReceived event payload:
-- {
--   "channel_id": "ch-001",
--   "message_type": "ADT",
--   "trigger_event": "A01",
--   "control_id": "MSG00001",
--   "sending_facility": "HOSPITAL_A",
--   "raw_message": "MSH|^~\\&|...",
--   "byte_size": 2048
-- }

-- PatientMatchProposed event payload:
-- {
--   "patient_a_stream_id": "...",
--   "patient_b_stream_id": "...",
--   "match_score": 0.9234,
--   "match_method": "probabilistic",
--   "field_scores": {
--     "family_name": 0.95,
--     "given_name": 0.88,
--     "birth_date": 1.00,
--     "gender": 1.00,
--     "address_postal_code": 0.75
--   }
-- }
```

## Read Models (CQRS Projections)

```sql
-- ============================================================
-- PROJECTION: CURRENT FHIR RESOURCES
-- Materialized from ResourceCreated / ResourceUpdated / ResourceDeleted events
-- ============================================================

CREATE TABLE rm_fhir_resource (
    id              UUID PRIMARY KEY,                -- Same as stream_id
    tenant_id       UUID NOT NULL,
    resource_type   VARCHAR(50) NOT NULL,
    fhir_id         VARCHAR(64) NOT NULL,
    version_id      INTEGER NOT NULL,
    last_updated    TIMESTAMPTZ NOT NULL,
    is_deleted      BOOLEAN NOT NULL DEFAULT false,
    resource_body   JSONB NOT NULL,                  -- Current state as JSONB for querying
    last_event_id   UUID NOT NULL,                   -- Last processed event
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, resource_type, fhir_id)
);

CREATE INDEX idx_rm_fhir_type ON rm_fhir_resource(tenant_id, resource_type, last_updated DESC);
CREATE INDEX idx_rm_fhir_body ON rm_fhir_resource USING gin(resource_body jsonb_path_ops);

-- ============================================================
-- PROJECTION: FHIR SEARCH INDEXES
-- Materialized from ResourceCreated / ResourceUpdated events
-- ============================================================

CREATE TABLE rm_search_token (
    resource_id     UUID NOT NULL,
    tenant_id       UUID NOT NULL,
    resource_type   VARCHAR(50) NOT NULL,
    param_name      VARCHAR(100) NOT NULL,
    system_uri      VARCHAR(500),
    value           VARCHAR(500) NOT NULL
);

CREATE INDEX idx_rm_search_token ON rm_search_token(tenant_id, resource_type, param_name, system_uri, value);

CREATE TABLE rm_search_string (
    resource_id     UUID NOT NULL,
    tenant_id       UUID NOT NULL,
    resource_type   VARCHAR(50) NOT NULL,
    param_name      VARCHAR(100) NOT NULL,
    value_normalized VARCHAR(500) NOT NULL
);

CREATE INDEX idx_rm_search_string ON rm_search_string(tenant_id, resource_type, param_name, value_normalized);

CREATE TABLE rm_search_date (
    resource_id     UUID NOT NULL,
    tenant_id       UUID NOT NULL,
    resource_type   VARCHAR(50) NOT NULL,
    param_name      VARCHAR(100) NOT NULL,
    value_low       TIMESTAMPTZ NOT NULL,
    value_high      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_search_date ON rm_search_date(tenant_id, resource_type, param_name, value_low, value_high);

CREATE TABLE rm_search_reference (
    resource_id         UUID NOT NULL,
    tenant_id           UUID NOT NULL,
    resource_type       VARCHAR(50) NOT NULL,
    param_name          VARCHAR(100) NOT NULL,
    target_resource_type VARCHAR(50) NOT NULL,
    target_fhir_id     VARCHAR(64) NOT NULL
);

CREATE INDEX idx_rm_search_ref ON rm_search_reference(tenant_id, resource_type, param_name, target_resource_type, target_fhir_id);

-- ============================================================
-- PROJECTION: PATIENT IDENTITY (MPI)
-- Materialized from PatientRegistered, PatientMatchProposed,
-- PatientMatchConfirmed, PatientMerged events
-- ============================================================

CREATE TABLE rm_patient_identity (
    stream_id           UUID PRIMARY KEY,
    tenant_id           UUID NOT NULL,
    master_patient_id   UUID,                        -- Golden record stream_id
    status              VARCHAR(20) NOT NULL DEFAULT 'active',
    fhir_patient_id     VARCHAR(64) NOT NULL,
    family_name         VARCHAR(255),
    given_names         VARCHAR(500),
    birth_date          DATE,
    gender              VARCHAR(20),
    identifiers         JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"system": "http://hospital-a.org/mrn", "value": "12345"},
    --           {"system": "http://hl7.org/fhir/sid/us-ssn", "value": "***-**-6789"}]
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_patient_name ON rm_patient_identity(tenant_id, family_name, given_names);
CREATE INDEX idx_rm_patient_dob ON rm_patient_identity(tenant_id, birth_date);
CREATE INDEX idx_rm_patient_identifiers ON rm_patient_identity USING gin(identifiers jsonb_path_ops);

CREATE TABLE rm_patient_match (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    patient_a_id    UUID NOT NULL,
    patient_b_id    UUID NOT NULL,
    match_score     NUMERIC(5,4) NOT NULL,
    match_status    VARCHAR(20) NOT NULL,            -- proposed, confirmed, rejected
    last_event_id   UUID NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_match_status ON rm_patient_match(tenant_id, match_status);

-- ============================================================
-- PROJECTION: MESSAGE PIPELINE STATUS
-- Materialized from HL7v2MessageReceived, TransformationApplied,
-- MessageRouted, MessageAcknowledged, MessageFailed events
-- ============================================================

CREATE TABLE rm_message_pipeline (
    stream_id           UUID PRIMARY KEY,
    tenant_id           UUID NOT NULL,
    channel_id          UUID NOT NULL,
    message_type        VARCHAR(10) NOT NULL,
    trigger_event       VARCHAR(10) NOT NULL,
    control_id          VARCHAR(50) NOT NULL,
    sending_facility    VARCHAR(255),
    current_status      VARCHAR(30) NOT NULL,        -- received, transforming, transformed, routing, routed, acknowledged, failed
    received_at         TIMESTAMPTZ NOT NULL,
    last_status_at      TIMESTAMPTZ NOT NULL,
    fhir_bundle_id      VARCHAR(64),
    error_message       TEXT,
    step_count          INTEGER NOT NULL DEFAULT 0,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_pipeline_status ON rm_message_pipeline(tenant_id, current_status, received_at DESC);
CREATE INDEX idx_rm_pipeline_channel ON rm_message_pipeline(tenant_id, channel_id, received_at DESC);

-- ============================================================
-- PROJECTION: CHANNEL HEALTH DASHBOARD
-- Materialized by aggregating message pipeline events per channel
-- ============================================================

CREATE TABLE rm_channel_health (
    channel_id      UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    channel_name    VARCHAR(255) NOT NULL,
    status          VARCHAR(20) NOT NULL,
    messages_today  INTEGER NOT NULL DEFAULT 0,
    messages_hour   INTEGER NOT NULL DEFAULT 0,
    errors_today    INTEGER NOT NULL DEFAULT 0,
    avg_latency_ms  NUMERIC(10,2),
    last_message_at TIMESTAMPTZ,
    last_error_at   TIMESTAMPTZ,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- PROJECTION: X12 EDI TRANSACTIONS
-- Materialized from EDIReceived, EDIParsed, EDIValidated, EDIRouted events
-- ============================================================

CREATE TABLE rm_x12_transaction (
    stream_id           UUID PRIMARY KEY,
    tenant_id           UUID NOT NULL,
    transaction_set     VARCHAR(10) NOT NULL,
    direction           VARCHAR(10) NOT NULL,
    sender_id           VARCHAR(50) NOT NULL,
    receiver_id         VARCHAR(50) NOT NULL,
    current_status      VARCHAR(30) NOT NULL,
    interchange_control VARCHAR(20) NOT NULL,
    fhir_claim_id       VARCHAR(64),
    received_at         TIMESTAMPTZ NOT NULL,
    last_status_at      TIMESTAMPTZ NOT NULL,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_x12_status ON rm_x12_transaction(tenant_id, current_status, received_at DESC);

-- ============================================================
-- PROJECTION: COMPLIANCE DASHBOARD
-- Materialized from ComplianceCheckRun, ComplianceViolationDetected events
-- ============================================================

CREATE TABLE rm_compliance_status (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    regulation      VARCHAR(50) NOT NULL,
    rule_code       VARCHAR(100) NOT NULL,
    current_status  VARCHAR(20) NOT NULL,            -- pass, fail, warning
    last_checked_at TIMESTAMPTZ NOT NULL,
    violation_count INTEGER NOT NULL DEFAULT 0,
    details         JSONB,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_compliance_tenant ON rm_compliance_status(tenant_id, regulation);
```

## Projection Management

```sql
-- ============================================================
-- PROJECTION TRACKING
-- Tracks which event each projection has processed up to
-- ============================================================

CREATE TABLE projection_checkpoint (
    projection_name VARCHAR(100) PRIMARY KEY,
    tenant_id       UUID,                            -- NULL for global projections
    last_event_id   UUID NOT NULL,
    last_event_at   TIMESTAMPTZ NOT NULL,
    events_processed BIGINT NOT NULL DEFAULT 0,
    status          VARCHAR(20) NOT NULL DEFAULT 'running', -- running, paused, rebuilding, error
    error_message   TEXT,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- DEAD LETTER QUEUE (events that failed projection)
-- ============================================================

CREATE TABLE projection_dead_letter (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    projection_name VARCHAR(100) NOT NULL,
    event_id        UUID NOT NULL,
    error_message   TEXT NOT NULL,
    retry_count     INTEGER NOT NULL DEFAULT 0,
    max_retries     INTEGER NOT NULL DEFAULT 3,
    status          VARCHAR(20) NOT NULL DEFAULT 'failed', -- failed, retrying, resolved
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    resolved_at     TIMESTAMPTZ
);

CREATE INDEX idx_dead_letter_status ON projection_dead_letter(status, created_at);
```

## Multi-Tenancy & Configuration

```sql
-- ============================================================
-- TENANT CONFIGURATION (also event-sourced but with read model)
-- ============================================================

CREATE TABLE rm_tenant (
    id              UUID PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    settings        JSONB NOT NULL DEFAULT '{}',
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE rm_channel_config (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    name            VARCHAR(255) NOT NULL,
    source_type     VARCHAR(30) NOT NULL,
    source_config   JSONB NOT NULL,
    dest_type       VARCHAR(30) NOT NULL,
    dest_config     JSONB NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'stopped',
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- SMART ON FHIR (read model for auth decisions)
-- ============================================================

CREATE TABLE rm_oauth2_client (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    client_id       VARCHAR(255) NOT NULL UNIQUE,
    client_name     VARCHAR(255) NOT NULL,
    client_type     VARCHAR(20) NOT NULL,
    redirect_uris   TEXT[] NOT NULL,
    grant_types     TEXT[] NOT NULL,
    scopes          TEXT[] NOT NULL,
    active          BOOLEAN NOT NULL DEFAULT true,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE rm_active_token (
    token_hash      VARCHAR(64) PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    client_id       UUID NOT NULL,
    user_id         UUID,
    patient_fhir_id VARCHAR(64),
    scopes_granted  TEXT[] NOT NULL,
    expires_at      TIMESTAMPTZ NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_token_expiry ON rm_active_token(expires_at);
```

## Temporal Queries

```sql
-- ============================================================
-- EXAMPLE: Reconstruct patient state at a specific point in time
-- ============================================================

-- Get all events for a patient stream up to a given timestamp
-- SELECT event_type, payload, occurred_at
-- FROM event_store
-- WHERE stream_id = 'patient-stream-uuid'
--   AND stream_type = 'FhirResource'
--   AND occurred_at <= '2026-03-01T00:00:00Z'
-- ORDER BY event_version ASC;

-- ============================================================
-- EXAMPLE: Find all changes to a patient record in the last 30 days
-- ============================================================

-- SELECT event_type, occurred_at, actor_id,
--        payload->>'changes' as changes
-- FROM event_store
-- WHERE stream_id = 'patient-stream-uuid'
--   AND occurred_at >= now() - INTERVAL '30 days'
-- ORDER BY occurred_at DESC;

-- ============================================================
-- EXAMPLE: Trace a message through the full pipeline
-- ============================================================

-- SELECT event_type, occurred_at, payload
-- FROM event_store
-- WHERE correlation_id = 'message-correlation-uuid'
-- ORDER BY occurred_at ASC;
-- Returns: HL7v2MessageReceived -> TransformationApplied -> ResourceCreated
--          -> PatientMatchProposed -> MessageRouted -> MessageAcknowledged
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store (write side) | 2 | Event store (partitioned) + snapshots |
| Projection Management | 2 | Checkpoints + dead letter queue |
| RM: FHIR Resources | 1 | Current resource state as JSONB |
| RM: FHIR Search Indexes | 4 | Token, string, date, reference |
| RM: Patient Identity / MPI | 2 | Identity + match results |
| RM: Message Pipeline | 2 | Per-message status + channel health |
| RM: X12 EDI | 1 | Transaction status |
| RM: Compliance | 1 | Per-rule status |
| RM: Configuration | 4 | Tenant, channels, OAuth2 clients, active tokens |
| **Total** | **19** | 2 write-side + 17 read-model tables; read models are disposable/rebuildable |

---

## Key Design Decisions

1. **Single event store table, partitioned by time.** All domain events go into one table partitioned by `occurred_at` month. This simplifies the write path, enables global event ordering, and allows old partitions to be archived or moved to cold storage for HIPAA retention compliance.

2. **Stream-based aggregate organization.** Events are grouped by `stream_id` + `stream_type`, where each aggregate (a FHIR resource, a patient identity, an HL7 v2 message, an X12 transaction) is a stream. The `event_version` column enforces optimistic concurrency within each stream.

3. **Correlation and causation IDs for full traceability.** The `correlation_id` links all events triggered by a single inbound message (e.g., an HL7 v2 ADT message that creates a Patient resource and triggers a match). The `causation_id` links each event to the specific event that caused it. Together they enable end-to-end pipeline tracing.

4. **Read models prefixed with `rm_` and treated as disposable.** Every read model table can be dropped and rebuilt from the event store. The `projection_checkpoint` table tracks replay progress. This means schema changes to read models are trivial — just rebuild.

5. **FHIR resource body stored as JSONB in the read model.** While the event store payload contains the FHIR JSON, the read model stores it as JSONB to enable PostgreSQL GIN index queries. This supports both FHIR search parameters (via dedicated index tables) and ad-hoc JSONB queries.

6. **Event schema versioning.** The `schema_version` field on each event enables backward-compatible evolution. Projections can handle multiple schema versions during replay, and an upcasting layer can transform old events to new schemas on read.

7. **Snapshots for performance.** For aggregates with many events (a patient with thousands of observations), periodic snapshots avoid replaying the full event history. The projection loads the latest snapshot and replays only subsequent events.

8. **Dead letter queue for projection failures.** Events that fail to project (e.g., due to a bug in the projection logic or a transient database error) are captured in `projection_dead_letter` rather than blocking the projection pipeline. This ensures the system degrades gracefully.

9. **HIPAA audit trail is free.** Because every action is an event, there is no separate audit logging mechanism. The event store itself, filtered by `actor_id` and `event_type`, produces a complete HIPAA-compliant audit trail. A projection can materialize these into FHIR AuditEvent resources for standards-compliant reporting.

10. **Real-time event streaming to AI/analytics.** The event store can be tailed (via PostgreSQL logical replication, LISTEN/NOTIFY, or a CDC tool like Debezium) to feed real-time events to anomaly detection models, compliance monitors, and analytics pipelines without impacting the operational database.
