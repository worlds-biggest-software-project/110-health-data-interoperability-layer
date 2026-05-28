# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Health Data Interoperability Layer · Created: 2026-05-19

## Philosophy

This model follows the Aidbox/Fhirbase pattern: store FHIR resources as native JSONB documents in PostgreSQL with a thin relational wrapper for identity, indexing, and operational metadata. Core operational tables (tenants, channels, patients, OAuth2) are fully relational, but FHIR clinical data lives in JSONB columns — one table per FHIR resource type — preserving the hierarchical, polymorphic structure of FHIR without decomposing it into dozens of normalized tables.

The key insight is that FHIR resources are inherently document-shaped. A Patient resource contains nested arrays of names, addresses, telecom contacts, and identifiers that vary per record. Decomposing these into normalized tables creates enormous schema complexity (the full FHIR R4 spec defines 145+ resource types with deeply nested structures). Instead, PostgreSQL's JSONB with GIN indexes enables efficient querying directly on the document structure while maintaining ACID transactions, foreign keys for operational data, and standard SQL for everything else.

This is the production-proven approach used by Aidbox (Health Samurai), Fhirbase, and partially by Google Cloud Healthcare API's internal storage. It provides the fastest path to a working FHIR server while allowing relational precision where it matters most — patient identity, message routing, authorization, and audit.

**Best for:** Rapid MVP development; teams that want to ship a working FHIR server quickly; multi-jurisdiction deployments where resource profiles vary by region; environments where FHIR resource types may expand frequently.

**Trade-offs:**
- (+) Natural fit for FHIR's document-shaped resources — no impedance mismatch
- (+) Much lower table count than fully normalized (~25 tables vs ~80+)
- (+) Adding new FHIR resource types requires only a new table with the same structure
- (+) JSONB GIN indexes support containment queries, path queries, and full-text search
- (+) Relational tables for operational data maintain integrity where it matters
- (+) PostgreSQL JSONB is battle-tested at scale (Aidbox serves major health systems)
- (-) Complex cross-resource joins require JSONB path extraction, which is less readable than column joins
- (-) JSONB storage is larger than normalized columns (field names repeated per row)
- (-) No foreign key enforcement within JSONB documents
- (-) Query optimization requires understanding JSONB index strategies (GIN, expression indexes)
- (-) Schema validation must be enforced at the application layer, not the database

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| HL7 FHIR R4 | Resources stored as native FHIR JSON in JSONB columns — zero transformation from wire format to storage |
| HL7 v2.x | Parsed messages stored with relational metadata + raw message text; FHIR translation stored as JSONB |
| HIPAA Security Rule | Relational `audit_event` table with structured columns for efficient compliance queries |
| IHE PIXm / PDQm | Relational MPI tables with JSONB `identifiers` array for flexible identifier storage |
| SMART on FHIR | Relational OAuth2 tables for token management; SMART scopes stored as PostgreSQL arrays |
| X12 EDI | Relational transaction tracking; parsed segments stored as JSONB for flexible field access |
| SQL on FHIR v2.0 | JSONB storage aligns with the SQL on FHIR specification for querying FHIR data with standard SQL |
| USCDI v3 | Profile conformance tracked per resource via JSONB `meta.profile` extraction |

---

## Multi-Tenancy & Configuration

```sql
-- ============================================================
-- TENANT & ORGANIZATION (relational — integrity matters here)
-- ============================================================

CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    config          JSONB NOT NULL DEFAULT '{}',
    -- Example config:
    -- {
    --   "default_jurisdiction": "US",
    --   "fhir_version": "R4",
    --   "enabled_resource_types": ["Patient", "Observation", "Condition", ...],
    --   "retention_days": 2555,
    --   "timezone": "America/New_York"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organization (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    fhir_id         VARCHAR(64) NOT NULL,
    resource        JSONB NOT NULL,                  -- Full FHIR Organization resource
    name            VARCHAR(255) GENERATED ALWAYS AS (resource->>'name') STORED,
    npi             VARCHAR(10),
    active          BOOLEAN GENERATED ALWAYS AS ((resource->>'active')::boolean) STORED,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, fhir_id)
);

CREATE INDEX idx_org_tenant ON organization(tenant_id);
CREATE INDEX idx_org_npi ON organization(npi) WHERE npi IS NOT NULL;
```

## FHIR Resource Tables (Table-per-Type with JSONB Body)

```sql
-- ============================================================
-- FHIR RESOURCE TEMPLATE
-- Each FHIR resource type gets a table with this structure.
-- Shown here for the most common types; additional types follow
-- the same pattern.
-- ============================================================

-- Generic function to create a FHIR resource table
-- In practice, use a migration script or code generator.

-- PATIENT
CREATE TABLE fhir_patient (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    fhir_id         VARCHAR(64) NOT NULL,
    version_id      INTEGER NOT NULL DEFAULT 1,
    last_updated    TIMESTAMPTZ NOT NULL DEFAULT now(),
    is_deleted      BOOLEAN NOT NULL DEFAULT false,
    source          VARCHAR(255),                    -- Meta.source
    resource        JSONB NOT NULL,                  -- Full FHIR Patient JSON
    -- Generated columns for frequently searched fields
    family_name     VARCHAR(255) GENERATED ALWAYS AS (
        resource#>>'{name,0,family}'
    ) STORED,
    birth_date      DATE GENERATED ALWAYS AS (
        (resource->>'birthDate')::date
    ) STORED,
    gender          VARCHAR(20) GENERATED ALWAYS AS (
        resource->>'gender'
    ) STORED,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, fhir_id)
);

CREATE INDEX idx_patient_tenant ON fhir_patient(tenant_id, last_updated DESC);
CREATE INDEX idx_patient_name ON fhir_patient(tenant_id, family_name);
CREATE INDEX idx_patient_dob ON fhir_patient(tenant_id, birth_date);
CREATE INDEX idx_patient_resource ON fhir_patient USING gin(resource jsonb_path_ops);

-- PATIENT HISTORY (immutable)
CREATE TABLE fhir_patient_history (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    current_id      UUID NOT NULL REFERENCES fhir_patient(id),
    tenant_id       UUID NOT NULL,
    fhir_id         VARCHAR(64) NOT NULL,
    version_id      INTEGER NOT NULL,
    resource        JSONB NOT NULL,
    request_method  VARCHAR(10),
    last_updated    TIMESTAMPTZ NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_patient_history ON fhir_patient_history(current_id, version_id DESC);

-- OBSERVATION
CREATE TABLE fhir_observation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    fhir_id         VARCHAR(64) NOT NULL,
    version_id      INTEGER NOT NULL DEFAULT 1,
    last_updated    TIMESTAMPTZ NOT NULL DEFAULT now(),
    is_deleted      BOOLEAN NOT NULL DEFAULT false,
    source          VARCHAR(255),
    resource        JSONB NOT NULL,
    -- Generated columns for common search paths
    subject_ref     VARCHAR(100) GENERATED ALWAYS AS (
        resource#>>'{subject,reference}'
    ) STORED,
    code_system     VARCHAR(500) GENERATED ALWAYS AS (
        resource#>>'{code,coding,0,system}'
    ) STORED,
    code_value      VARCHAR(100) GENERATED ALWAYS AS (
        resource#>>'{code,coding,0,code}'
    ) STORED,
    effective_date  TIMESTAMPTZ GENERATED ALWAYS AS (
        CASE WHEN resource ? 'effectiveDateTime'
             THEN (resource->>'effectiveDateTime')::timestamptz
             ELSE NULL END
    ) STORED,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, fhir_id)
);

CREATE INDEX idx_observation_subject ON fhir_observation(tenant_id, subject_ref);
CREATE INDEX idx_observation_code ON fhir_observation(tenant_id, code_system, code_value);
CREATE INDEX idx_observation_date ON fhir_observation(tenant_id, effective_date DESC);
CREATE INDEX idx_observation_resource ON fhir_observation USING gin(resource jsonb_path_ops);

CREATE TABLE fhir_observation_history (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    current_id      UUID NOT NULL REFERENCES fhir_observation(id),
    tenant_id       UUID NOT NULL,
    fhir_id         VARCHAR(64) NOT NULL,
    version_id      INTEGER NOT NULL,
    resource        JSONB NOT NULL,
    request_method  VARCHAR(10),
    last_updated    TIMESTAMPTZ NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- CONDITION
CREATE TABLE fhir_condition (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    fhir_id         VARCHAR(64) NOT NULL,
    version_id      INTEGER NOT NULL DEFAULT 1,
    last_updated    TIMESTAMPTZ NOT NULL DEFAULT now(),
    is_deleted      BOOLEAN NOT NULL DEFAULT false,
    source          VARCHAR(255),
    resource        JSONB NOT NULL,
    subject_ref     VARCHAR(100) GENERATED ALWAYS AS (
        resource#>>'{subject,reference}'
    ) STORED,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, fhir_id)
);

CREATE INDEX idx_condition_subject ON fhir_condition(tenant_id, subject_ref);
CREATE INDEX idx_condition_resource ON fhir_condition USING gin(resource jsonb_path_ops);

-- ENCOUNTER
CREATE TABLE fhir_encounter (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    fhir_id         VARCHAR(64) NOT NULL,
    version_id      INTEGER NOT NULL DEFAULT 1,
    last_updated    TIMESTAMPTZ NOT NULL DEFAULT now(),
    is_deleted      BOOLEAN NOT NULL DEFAULT false,
    source          VARCHAR(255),
    resource        JSONB NOT NULL,
    subject_ref     VARCHAR(100) GENERATED ALWAYS AS (
        resource#>>'{subject,reference}'
    ) STORED,
    status          VARCHAR(30) GENERATED ALWAYS AS (
        resource->>'status'
    ) STORED,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, fhir_id)
);

CREATE INDEX idx_encounter_subject ON fhir_encounter(tenant_id, subject_ref);
CREATE INDEX idx_encounter_resource ON fhir_encounter USING gin(resource jsonb_path_ops);

-- MEDICATION REQUEST
CREATE TABLE fhir_medication_request (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    fhir_id         VARCHAR(64) NOT NULL,
    version_id      INTEGER NOT NULL DEFAULT 1,
    last_updated    TIMESTAMPTZ NOT NULL DEFAULT now(),
    is_deleted      BOOLEAN NOT NULL DEFAULT false,
    source          VARCHAR(255),
    resource        JSONB NOT NULL,
    subject_ref     VARCHAR(100) GENERATED ALWAYS AS (
        resource#>>'{subject,reference}'
    ) STORED,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, fhir_id)
);

CREATE INDEX idx_medrx_subject ON fhir_medication_request(tenant_id, subject_ref);
CREATE INDEX idx_medrx_resource ON fhir_medication_request USING gin(resource jsonb_path_ops);

-- DIAGNOSTIC REPORT
CREATE TABLE fhir_diagnostic_report (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    fhir_id         VARCHAR(64) NOT NULL,
    version_id      INTEGER NOT NULL DEFAULT 1,
    last_updated    TIMESTAMPTZ NOT NULL DEFAULT now(),
    is_deleted      BOOLEAN NOT NULL DEFAULT false,
    source          VARCHAR(255),
    resource        JSONB NOT NULL,
    subject_ref     VARCHAR(100) GENERATED ALWAYS AS (
        resource#>>'{subject,reference}'
    ) STORED,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, fhir_id)
);

CREATE INDEX idx_diagreport_subject ON fhir_diagnostic_report(tenant_id, subject_ref);
CREATE INDEX idx_diagreport_resource ON fhir_diagnostic_report USING gin(resource jsonb_path_ops);

-- ALLERGY INTOLERANCE
CREATE TABLE fhir_allergy_intolerance (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    fhir_id         VARCHAR(64) NOT NULL,
    version_id      INTEGER NOT NULL DEFAULT 1,
    last_updated    TIMESTAMPTZ NOT NULL DEFAULT now(),
    is_deleted      BOOLEAN NOT NULL DEFAULT false,
    source          VARCHAR(255),
    resource        JSONB NOT NULL,
    patient_ref     VARCHAR(100) GENERATED ALWAYS AS (
        resource#>>'{patient,reference}'
    ) STORED,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, fhir_id)
);

CREATE INDEX idx_allergy_patient ON fhir_allergy_intolerance(tenant_id, patient_ref);
CREATE INDEX idx_allergy_resource ON fhir_allergy_intolerance USING gin(resource jsonb_path_ops);

-- DOCUMENT REFERENCE (CDA, clinical documents)
CREATE TABLE fhir_document_reference (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    fhir_id         VARCHAR(64) NOT NULL,
    version_id      INTEGER NOT NULL DEFAULT 1,
    last_updated    TIMESTAMPTZ NOT NULL DEFAULT now(),
    is_deleted      BOOLEAN NOT NULL DEFAULT false,
    source          VARCHAR(255),
    resource        JSONB NOT NULL,
    subject_ref     VARCHAR(100) GENERATED ALWAYS AS (
        resource#>>'{subject,reference}'
    ) STORED,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, fhir_id)
);

CREATE INDEX idx_docref_subject ON fhir_document_reference(tenant_id, subject_ref);
CREATE INDEX idx_docref_resource ON fhir_document_reference USING gin(resource jsonb_path_ops);

-- CLAIM (X12 837 → FHIR)
CREATE TABLE fhir_claim (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    fhir_id         VARCHAR(64) NOT NULL,
    version_id      INTEGER NOT NULL DEFAULT 1,
    last_updated    TIMESTAMPTZ NOT NULL DEFAULT now(),
    is_deleted      BOOLEAN NOT NULL DEFAULT false,
    source          VARCHAR(255),
    resource        JSONB NOT NULL,
    patient_ref     VARCHAR(100) GENERATED ALWAYS AS (
        resource#>>'{patient,reference}'
    ) STORED,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, fhir_id)
);

CREATE INDEX idx_claim_patient ON fhir_claim(tenant_id, patient_ref);
CREATE INDEX idx_claim_resource ON fhir_claim USING gin(resource jsonb_path_ops);

-- GENERIC CATCH-ALL for less common resource types
CREATE TABLE fhir_resource_other (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    resource_type   VARCHAR(50) NOT NULL,
    fhir_id         VARCHAR(64) NOT NULL,
    version_id      INTEGER NOT NULL DEFAULT 1,
    last_updated    TIMESTAMPTZ NOT NULL DEFAULT now(),
    is_deleted      BOOLEAN NOT NULL DEFAULT false,
    source          VARCHAR(255),
    resource        JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, resource_type, fhir_id)
);

CREATE INDEX idx_other_type ON fhir_resource_other(tenant_id, resource_type, last_updated DESC);
CREATE INDEX idx_other_resource ON fhir_resource_other USING gin(resource jsonb_path_ops);
```

## Patient Identity & MPI

```sql
-- ============================================================
-- MASTER PATIENT INDEX (relational for matching precision)
-- ============================================================

CREATE TABLE patient_identity (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    golden_record_id    UUID REFERENCES patient_identity(id),
    fhir_patient_id     UUID NOT NULL REFERENCES fhir_patient(id),
    status              VARCHAR(20) NOT NULL DEFAULT 'active',
    merged_into_id      UUID REFERENCES patient_identity(id),
    identifiers         JSONB NOT NULL DEFAULT '[]',
    -- Example: [
    --   {"system": "http://hospital-a.org/mrn", "value": "MRN12345"},
    --   {"system": "urn:oid:2.16.840.1.113883.4.1", "value": "***-**-6789"}
    -- ]
    demographics        JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "family_name": "Smith",
    --   "given_names": ["John", "Michael"],
    --   "birth_date": "1985-03-15",
    --   "gender": "male",
    --   "address": {"city": "Boston", "state": "MA", "postal_code": "02101"}
    -- }
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pi_tenant ON patient_identity(tenant_id, status);
CREATE INDEX idx_pi_golden ON patient_identity(golden_record_id) WHERE golden_record_id IS NOT NULL;
CREATE INDEX idx_pi_identifiers ON patient_identity USING gin(identifiers jsonb_path_ops);
CREATE INDEX idx_pi_demographics ON patient_identity USING gin(demographics jsonb_path_ops);

-- Expression indexes for common demographic searches
CREATE INDEX idx_pi_family_name ON patient_identity(tenant_id, (demographics->>'family_name'));
CREATE INDEX idx_pi_birth_date ON patient_identity(tenant_id, ((demographics->>'birth_date')::date));

CREATE TABLE patient_match_job (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    patient_a_id    UUID NOT NULL REFERENCES patient_identity(id),
    patient_b_id    UUID NOT NULL REFERENCES patient_identity(id),
    match_score     NUMERIC(5,4) NOT NULL,
    match_method    VARCHAR(30) NOT NULL,
    match_status    VARCHAR(20) NOT NULL DEFAULT 'proposed',
    field_scores    JSONB NOT NULL DEFAULT '{}',
    reviewed_by     UUID,
    reviewed_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_match_status ON patient_match_job(tenant_id, match_status);
```

## HL7 v2 Pipeline

```sql
-- ============================================================
-- HL7 V2 CHANNELS & MESSAGES
-- ============================================================

CREATE TABLE hl7v2_channel (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    source_config   JSONB NOT NULL,
    -- Example: {
    --   "type": "mllp",
    --   "host": "0.0.0.0",
    --   "port": 6661,
    --   "tls": true,
    --   "ack_mode": "immediate"
    -- }
    dest_config     JSONB NOT NULL,
    -- Example: {
    --   "type": "fhir_server",
    --   "endpoint": "http://localhost:8080/fhir",
    --   "auth": "bearer"
    -- }
    transform_rules JSONB NOT NULL DEFAULT '[]',
    -- Example: [
    --   {"source": "PID.5", "target": "Patient.name.family", "type": "direct"},
    --   {"source": "PID.7", "target": "Patient.birthDate", "type": "date_format", "config": {"from": "yyyyMMdd"}},
    --   {"source": "OBX.5", "target": "Observation.value", "type": "ai_generated", "model_version": "v2"}
    -- ]
    status          VARCHAR(20) NOT NULL DEFAULT 'stopped',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE hl7v2_message (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    channel_id          UUID NOT NULL REFERENCES hl7v2_channel(id),
    message_type        VARCHAR(10) NOT NULL,
    trigger_event       VARCHAR(10) NOT NULL,
    control_id          VARCHAR(50) NOT NULL,
    sending_facility    VARCHAR(255),
    receiving_facility  VARCHAR(255),
    raw_message         TEXT NOT NULL,
    parsed_segments     JSONB,
    -- Example: {
    --   "MSH": {"field_separator": "|", "sending_app": "EPIC", ...},
    --   "PID": {"patient_id": "12345", "name": {"family": "Smith", ...}, ...},
    --   "OBX": [{"set_id": "1", "value_type": "NM", "value": "120", ...}]
    -- }
    processing_status   VARCHAR(20) NOT NULL DEFAULT 'received',
    error_message       TEXT,
    fhir_bundle_id      VARCHAR(64),
    received_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    processed_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_hl7v2_channel_time ON hl7v2_message(tenant_id, channel_id, received_at DESC);
CREATE INDEX idx_hl7v2_status ON hl7v2_message(tenant_id, processing_status);
CREATE INDEX idx_hl7v2_segments ON hl7v2_message USING gin(parsed_segments jsonb_path_ops);
```

## X12 EDI

```sql
-- ============================================================
-- X12 EDI TRANSACTIONS
-- ============================================================

CREATE TABLE x12_transaction (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    transaction_set     VARCHAR(10) NOT NULL,
    direction           VARCHAR(10) NOT NULL,
    interchange_control VARCHAR(20) NOT NULL,
    sender_id           VARCHAR(50) NOT NULL,
    receiver_id         VARCHAR(50) NOT NULL,
    raw_edi             TEXT NOT NULL,
    parsed_data         JSONB,
    -- Example (270 Eligibility Inquiry):
    -- {
    --   "subscriber": {"member_id": "ABC123", "name": "Smith, John"},
    --   "payer": {"id": "62308", "name": "Aetna"},
    --   "service_types": ["30"],
    --   "date_of_service": "2026-05-19"
    -- }
    processing_status   VARCHAR(20) NOT NULL DEFAULT 'received',
    error_message       TEXT,
    fhir_resource_ids   TEXT[],
    received_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    processed_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_x12_set ON x12_transaction(tenant_id, transaction_set, received_at DESC);
CREATE INDEX idx_x12_status ON x12_transaction(tenant_id, processing_status);
CREATE INDEX idx_x12_parsed ON x12_transaction USING gin(parsed_data jsonb_path_ops);
```

## SMART on FHIR / OAuth2

```sql
-- ============================================================
-- SMART ON FHIR AUTHORIZATION (relational — security critical)
-- ============================================================

CREATE TABLE oauth2_client (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    client_id       VARCHAR(255) NOT NULL UNIQUE,
    client_name     VARCHAR(255) NOT NULL,
    client_type     VARCHAR(20) NOT NULL,
    redirect_uris   TEXT[] NOT NULL,
    grant_types     TEXT[] NOT NULL,
    scopes          TEXT[] NOT NULL,
    jwks_uri        VARCHAR(500),
    launch_config   JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "launch_types": ["ehr", "standalone"],
    --   "required_scopes": ["openid", "fhirUser"],
    --   "allowed_resource_types": ["Patient", "Observation"]
    -- }
    active          BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE oauth2_session (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    client_id           UUID NOT NULL REFERENCES oauth2_client(id),
    user_id             UUID,
    patient_fhir_id     VARCHAR(64),
    encounter_fhir_id   VARCHAR(64),
    scopes_granted      TEXT[] NOT NULL,
    access_token_hash   VARCHAR(64),
    refresh_token_hash  VARCHAR(64),
    expires_at          TIMESTAMPTZ NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_oauth2_token ON oauth2_session(access_token_hash) WHERE access_token_hash IS NOT NULL;
CREATE INDEX idx_oauth2_expiry ON oauth2_session(expires_at);
```

## Audit & Compliance

```sql
-- ============================================================
-- AUDIT LOGGING
-- ============================================================

CREATE TABLE audit_event (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    event_type      VARCHAR(50) NOT NULL,
    event_action    VARCHAR(10) NOT NULL,
    event_outcome   VARCHAR(10) NOT NULL,
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    agent_id        VARCHAR(255) NOT NULL,
    agent_type      VARCHAR(30) NOT NULL,
    agent_ip        INET,
    entity_type     VARCHAR(50),
    entity_fhir_id  VARCHAR(64),
    detail          JSONB,
    -- Example: {
    --   "query": "Patient?name=Smith&birthdate=1985-03-15",
    --   "results_count": 3,
    --   "response_time_ms": 45
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (recorded_at);

CREATE INDEX idx_audit_time ON audit_event(tenant_id, recorded_at DESC);
CREATE INDEX idx_audit_entity ON audit_event(tenant_id, entity_type, entity_fhir_id);
CREATE INDEX idx_audit_agent ON audit_event(tenant_id, agent_id);

-- ============================================================
-- COMPLIANCE
-- ============================================================

CREATE TABLE compliance_rule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    regulation      VARCHAR(50) NOT NULL,
    rule_code       VARCHAR(100) NOT NULL UNIQUE,
    description     TEXT NOT NULL,
    severity        VARCHAR(20) NOT NULL DEFAULT 'warning',
    check_config    JSONB NOT NULL,
    -- Example: {
    --   "type": "fhir_search",
    --   "query": "Patient?_has:Observation:subject:_count=0",
    --   "expected": "empty",
    --   "message": "Patients without any observations detected"
    -- }
    active          BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE compliance_result (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    rule_id         UUID NOT NULL REFERENCES compliance_rule(id),
    status          VARCHAR(20) NOT NULL,
    details         JSONB,
    checked_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## FHIR Subscriptions

```sql
-- ============================================================
-- SUBSCRIPTIONS
-- ============================================================

CREATE TABLE fhir_subscription (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    fhir_id         VARCHAR(64) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'requested',
    resource        JSONB NOT NULL,                  -- Full FHIR Subscription resource
    criteria        VARCHAR(500) GENERATED ALWAYS AS (
        resource->>'criteria'
    ) STORED,
    channel_type    VARCHAR(30) GENERATED ALWAYS AS (
        resource#>>'{channel,type}'
    ) STORED,
    channel_endpoint VARCHAR(500) GENERATED ALWAYS AS (
        resource#>>'{channel,endpoint}'
    ) STORED,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, fhir_id)
);
```

## RBAC

```sql
-- ============================================================
-- USERS & ROLES (relational)
-- ============================================================

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    external_id     VARCHAR(255),
    email           VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    preferences     JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE TABLE role (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(100) NOT NULL,
    permissions     JSONB NOT NULL DEFAULT '[]',
    -- Example: [
    --   {"resource_type": "Patient", "actions": ["read", "search"], "scope": "patient"},
    --   {"resource_type": "*", "actions": ["read"], "scope": "user"},
    --   {"resource_type": "hl7v2_channel", "actions": ["read", "write"], "scope": "system"}
    -- ]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

CREATE TABLE user_role (
    user_id         UUID NOT NULL REFERENCES app_user(id),
    role_id         UUID NOT NULL REFERENCES role(id),
    organization_id UUID REFERENCES organization(id),
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, role_id)
);
```

## Example JSONB Queries

```sql
-- ============================================================
-- EXAMPLE: Search patients by identifier (JSONB containment)
-- ============================================================

-- Find patient with MRN "12345" from hospital-a
SELECT id, fhir_id, resource
FROM fhir_patient
WHERE tenant_id = 'tenant-uuid'
  AND resource @> '{"identifier": [{"system": "http://hospital-a.org/mrn", "value": "12345"}]}';

-- ============================================================
-- EXAMPLE: Find all observations with a specific LOINC code
-- ============================================================

SELECT id, fhir_id, resource
FROM fhir_observation
WHERE tenant_id = 'tenant-uuid'
  AND code_system = 'http://loinc.org'
  AND code_value = '8867-4';     -- Heart rate

-- ============================================================
-- EXAMPLE: Cross-resource join — patient with their conditions
-- ============================================================

SELECT p.fhir_id AS patient_id,
       p.resource->>'birthDate' AS birth_date,
       c.resource#>>'{code,coding,0,display}' AS condition_name,
       c.resource->>'onsetDateTime' AS onset
FROM fhir_patient p
JOIN fhir_condition c ON c.tenant_id = p.tenant_id
                      AND c.subject_ref = 'Patient/' || p.fhir_id
WHERE p.tenant_id = 'tenant-uuid'
  AND p.family_name = 'Smith';

-- ============================================================
-- EXAMPLE: Find HL7 v2 messages with specific parsed segment data
-- ============================================================

SELECT id, control_id, parsed_segments->'PID'->>'patient_id'
FROM hl7v2_message
WHERE tenant_id = 'tenant-uuid'
  AND parsed_segments @> '{"PID": {"name": {"family": "Smith"}}}';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Tenant & Organization | 2 | Relational with JSONB config |
| FHIR Resource Tables | 10 | Patient, Observation, Condition, Encounter, MedicationRequest, DiagnosticReport, AllergyIntolerance, DocumentReference, Claim + catch-all |
| FHIR History Tables | 2 | Patient + Observation histories shown; pattern repeats per type |
| Patient Identity / MPI | 2 | Identity with JSONB demographics + match jobs |
| HL7 v2 Pipeline | 2 | Channels with JSONB transform rules + messages with JSONB parsed segments |
| X12 EDI | 1 | Transactions with JSONB parsed data |
| SMART on FHIR / OAuth2 | 2 | Clients with JSONB launch config + sessions |
| Audit & Compliance | 3 | Events (partitioned), rules with JSONB check config, results |
| Subscriptions | 1 | JSONB resource with generated columns |
| RBAC | 3 | Users, roles with JSONB permissions, user-role junction |
| **Total** | **~28** | Expands by 1-2 tables per additional FHIR resource type promoted from catch-all |

---

## Key Design Decisions

1. **Table-per-FHIR-resource-type rather than single resource table.** Following the Aidbox/Fhirbase pattern, each high-volume FHIR resource type gets its own table. This enables type-specific generated columns, targeted GIN indexes, and per-type partitioning. Less common types share `fhir_resource_other` until they warrant promotion.

2. **FHIR JSON stored as-is in JSONB columns.** Zero transformation between wire format and storage. The `resource` column holds the exact FHIR JSON, enabling round-trip fidelity and simplifying the application layer. PostgreSQL's JSONB binary format handles indexing and querying efficiently.

3. **Generated (STORED) columns for frequently searched fields.** Rather than maintaining separate search index tables, PostgreSQL's `GENERATED ALWAYS AS ... STORED` columns extract key search fields (family_name, birth_date, subject_ref, code_value) into typed relational columns. These are automatically maintained by the database and can be indexed normally.

4. **GIN indexes with `jsonb_path_ops` for containment queries.** Every resource table has a GIN index on its JSONB column, enabling FHIR search operations that map to `@>` containment queries. This handles the long tail of search parameters without dedicated index tables.

5. **Hybrid approach: relational where integrity matters, JSONB where flexibility matters.** Security-critical tables (OAuth2, users, audit) are fully relational. Clinical data that varies by profile, jurisdiction, or resource version lives in JSONB. This gives the best of both worlds.

6. **Transformation rules stored as JSONB arrays in channel configuration.** HL7 v2-to-FHIR mapping rules are stored as JSONB within the channel record rather than in a separate rules table. This keeps the channel self-contained and allows the AI-generated mapping engine to write rules in the same format as manual rules.

7. **Patient demographics stored as JSONB with expression indexes.** The MPI's demographic data is a JSONB document with expression indexes on frequently searched paths (family_name, birth_date). This accommodates jurisdiction-specific demographic fields (e.g., NHS number in the UK, Medicare number in Australia) without schema changes.

8. **History tables per resource type.** Each promoted FHIR resource type has a companion `_history` table storing previous versions. This follows the Fhirbase pattern and supports FHIR's `_history` operation efficiently without mixing current and historical data in the same table.
