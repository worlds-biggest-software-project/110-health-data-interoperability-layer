# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Health Data Interoperability Layer · Created: 2026-05-19

## Philosophy

This model follows a fully normalized relational design where every FHIR resource type, messaging concept, patient identity record, and administrative entity gets its own dedicated table with explicit foreign keys. The schema mirrors the HAPI FHIR JPA approach — a resource body table plus separate search-parameter index tables — but extends it with first-class tables for HL7 v2 channels, X12 EDI transactions, patient matching, and compliance tracking.

The principle is that data integrity and referential consistency are paramount in a healthcare system handling PHI under HIPAA. Every relationship is enforced at the database level. Cross-entity queries (e.g., "find all observations for patients matched across two source systems") use standard SQL joins rather than application-level denormalization. Audit trails are embedded as relational records rather than event streams.

This is the approach used by HAPI FHIR's JPA persistence layer, the IBM LinuxForHealth FHIR Server, and Smile CDR's PostgreSQL backend. It is well understood by healthcare IT teams and aligns naturally with ORM frameworks like Hibernate.

**Best for:** Teams with strong SQL/relational expertise who need maximum data integrity, complex cross-entity reporting, and straightforward ORM integration.

**Trade-offs:**
- (+) Full referential integrity enforced by the database
- (+) Complex joins and aggregations are natural in SQL
- (+) Well-understood by healthcare IT developers; mirrors HAPI FHIR JPA patterns
- (+) Straightforward schema migration with tools like Flyway or Liquibase
- (-) High table count (~80-100 tables) increases schema complexity
- (-) FHIR's polymorphic and deeply nested structures require many junction tables
- (-) Schema changes needed when supporting new FHIR resource types or search parameters
- (-) Performance degrades on wide joins across many index tables at high volume

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| HL7 FHIR R4 (v4.0.1) | Each FHIR resource type maps to a dedicated table; search parameter indexes follow HAPI FHIR's HFJ_SPIDX pattern |
| HL7 v2.x | Dedicated `hl7v2_message` and `hl7v2_segment` tables store parsed ADT/ORM/ORU messages with segment-level indexing |
| IHE PIXm / PDQm | `patient_identifier` and `patient_match_result` tables implement cross-referencing and match scoring |
| SMART on FHIR | `oauth2_client`, `oauth2_token`, and `smart_launch_context` tables model the authorization lifecycle |
| X12 EDI (005010) | `x12_transaction` and `x12_segment` tables store parsed 837/835/270/271/278 transactions |
| HIPAA Security Rule | `audit_event` table aligned with FHIR AuditEvent resource structure; `access_control_entry` for RBAC |
| USCDI v3 | US Core profiles enforced via `fhir_profile_validation` table tracking conformance per resource |
| ISO 3166 | `jurisdiction` reference table with ISO 3166-1/2 codes for multi-region support |
| LOINC / SNOMED CT / ICD-10 | `terminology_code_system` and `terminology_value_set` tables for code validation |

---

## Core Identity & Multi-Tenancy

```sql
-- ============================================================
-- TENANT & ORGANIZATION
-- ============================================================

CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',  -- active, suspended, archived
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organization (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    fhir_id         VARCHAR(64) NOT NULL,           -- FHIR Organization.id
    name            VARCHAR(255) NOT NULL,
    type_code       VARCHAR(50),                     -- provider, payer, pharmacy, lab
    npi             VARCHAR(10),                     -- National Provider Identifier
    tax_id          VARCHAR(20),                     -- EIN for X12 transactions
    active          BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, fhir_id)
);

CREATE INDEX idx_organization_tenant ON organization(tenant_id);
CREATE INDEX idx_organization_npi ON organization(npi) WHERE npi IS NOT NULL;
```

## FHIR Resource Storage

```sql
-- ============================================================
-- FHIR RESOURCE BODY (one row per resource version)
-- ============================================================

CREATE TABLE fhir_resource (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    resource_type       VARCHAR(50) NOT NULL,        -- Patient, Observation, Condition, etc.
    fhir_id             VARCHAR(64) NOT NULL,        -- FHIR resource logical ID
    version_id          INTEGER NOT NULL DEFAULT 1,
    status              VARCHAR(20) NOT NULL DEFAULT 'active',
    last_updated        TIMESTAMPTZ NOT NULL DEFAULT now(),
    source_system       VARCHAR(255),                -- originating system URI
    resource_body       TEXT NOT NULL,                -- Full FHIR JSON (optionally gzipped)
    hash_sha256         VARCHAR(64) NOT NULL,        -- Integrity check
    is_deleted          BOOLEAN NOT NULL DEFAULT false,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, resource_type, fhir_id, version_id)
);

CREATE INDEX idx_fhir_resource_type ON fhir_resource(tenant_id, resource_type, fhir_id);
CREATE INDEX idx_fhir_resource_updated ON fhir_resource(tenant_id, resource_type, last_updated);

-- ============================================================
-- FHIR RESOURCE HISTORY (immutable previous versions)
-- ============================================================

CREATE TABLE fhir_resource_history (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    resource_id         UUID NOT NULL REFERENCES fhir_resource(id),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    resource_type       VARCHAR(50) NOT NULL,
    fhir_id             VARCHAR(64) NOT NULL,
    version_id          INTEGER NOT NULL,
    resource_body       TEXT NOT NULL,
    last_updated        TIMESTAMPTZ NOT NULL,
    request_method      VARCHAR(10),                 -- PUT, POST, DELETE
    request_url         VARCHAR(500),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_fhir_history_resource ON fhir_resource_history(resource_id, version_id);
```

## Search Parameter Index Tables (HAPI-Style)

```sql
-- ============================================================
-- SEARCH INDEX: STRING (name, address, etc.)
-- ============================================================

CREATE TABLE spidx_string (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    resource_id     UUID NOT NULL REFERENCES fhir_resource(id) ON DELETE CASCADE,
    tenant_id       UUID NOT NULL,
    resource_type   VARCHAR(50) NOT NULL,
    param_name      VARCHAR(100) NOT NULL,           -- e.g., "name", "address"
    value_normalized VARCHAR(500) NOT NULL,           -- lowercased, accent-folded
    value_exact     VARCHAR(500) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_spidx_string_search ON spidx_string(tenant_id, resource_type, param_name, value_normalized);

-- ============================================================
-- SEARCH INDEX: TOKEN (code, identifier, status)
-- ============================================================

CREATE TABLE spidx_token (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    resource_id     UUID NOT NULL REFERENCES fhir_resource(id) ON DELETE CASCADE,
    tenant_id       UUID NOT NULL,
    resource_type   VARCHAR(50) NOT NULL,
    param_name      VARCHAR(100) NOT NULL,           -- e.g., "code", "identifier"
    system_uri      VARCHAR(500),                    -- e.g., "http://loinc.org"
    value           VARCHAR(500) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_spidx_token_search ON spidx_token(tenant_id, resource_type, param_name, system_uri, value);

-- ============================================================
-- SEARCH INDEX: DATE
-- ============================================================

CREATE TABLE spidx_date (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    resource_id     UUID NOT NULL REFERENCES fhir_resource(id) ON DELETE CASCADE,
    tenant_id       UUID NOT NULL,
    resource_type   VARCHAR(50) NOT NULL,
    param_name      VARCHAR(100) NOT NULL,           -- e.g., "date", "birthdate"
    value_low       TIMESTAMPTZ NOT NULL,
    value_high      TIMESTAMPTZ NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_spidx_date_search ON spidx_date(tenant_id, resource_type, param_name, value_low, value_high);

-- ============================================================
-- SEARCH INDEX: REFERENCE (subject, patient, encounter)
-- ============================================================

CREATE TABLE spidx_reference (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    resource_id         UUID NOT NULL REFERENCES fhir_resource(id) ON DELETE CASCADE,
    tenant_id           UUID NOT NULL,
    resource_type       VARCHAR(50) NOT NULL,
    param_name          VARCHAR(100) NOT NULL,       -- e.g., "subject", "patient"
    target_resource_type VARCHAR(50) NOT NULL,
    target_fhir_id     VARCHAR(64) NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_spidx_reference_search ON spidx_reference(tenant_id, resource_type, param_name, target_resource_type, target_fhir_id);
CREATE INDEX idx_spidx_reference_reverse ON spidx_reference(tenant_id, target_resource_type, target_fhir_id);

-- ============================================================
-- SEARCH INDEX: QUANTITY
-- ============================================================

CREATE TABLE spidx_quantity (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    resource_id     UUID NOT NULL REFERENCES fhir_resource(id) ON DELETE CASCADE,
    tenant_id       UUID NOT NULL,
    resource_type   VARCHAR(50) NOT NULL,
    param_name      VARCHAR(100) NOT NULL,
    system_uri      VARCHAR(500),                    -- e.g., "http://unitsofmeasure.org"
    units           VARCHAR(100),
    value           NUMERIC(18,6) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_spidx_quantity_search ON spidx_quantity(tenant_id, resource_type, param_name, system_uri, units, value);

-- ============================================================
-- SEARCH INDEX: URI
-- ============================================================

CREATE TABLE spidx_uri (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    resource_id     UUID NOT NULL REFERENCES fhir_resource(id) ON DELETE CASCADE,
    tenant_id       UUID NOT NULL,
    resource_type   VARCHAR(50) NOT NULL,
    param_name      VARCHAR(100) NOT NULL,
    value           VARCHAR(2000) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_spidx_uri_search ON spidx_uri(tenant_id, resource_type, param_name, value);
```

## Patient Identity & Master Patient Index

```sql
-- ============================================================
-- PATIENT IDENTITY MANAGEMENT
-- ============================================================

CREATE TABLE patient_identity (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    master_id       UUID,                            -- golden record reference (self-join)
    fhir_patient_id UUID NOT NULL REFERENCES fhir_resource(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'active',  -- active, merged, inactive
    merged_into_id  UUID REFERENCES patient_identity(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_patient_identity_master ON patient_identity(tenant_id, master_id);

CREATE TABLE patient_identifier (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_identity_id UUID NOT NULL REFERENCES patient_identity(id),
    tenant_id       UUID NOT NULL,
    identifier_system VARCHAR(500) NOT NULL,          -- e.g., "http://hospital-a.org/mrn"
    identifier_value VARCHAR(255) NOT NULL,
    assigning_authority VARCHAR(255),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_patient_identifier_lookup ON patient_identifier(tenant_id, identifier_system, identifier_value);

CREATE TABLE patient_demographics (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_identity_id UUID NOT NULL REFERENCES patient_identity(id),
    tenant_id           UUID NOT NULL,
    family_name         VARCHAR(255),
    given_names         VARCHAR(500),
    birth_date          DATE,
    gender              VARCHAR(20),
    ssn_hash            VARCHAR(64),                 -- SHA-256 of SSN (never store plaintext)
    address_line        VARCHAR(500),
    city                VARCHAR(255),
    state               VARCHAR(10),
    postal_code         VARCHAR(20),
    country             VARCHAR(3),                  -- ISO 3166-1 alpha-3
    phone               VARCHAR(50),
    email               VARCHAR(255),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_patient_demo_name ON patient_demographics(tenant_id, family_name, given_names);
CREATE INDEX idx_patient_demo_dob ON patient_demographics(tenant_id, birth_date);

CREATE TABLE patient_match_result (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL,
    patient_a_id        UUID NOT NULL REFERENCES patient_identity(id),
    patient_b_id        UUID NOT NULL REFERENCES patient_identity(id),
    match_score         NUMERIC(5,4) NOT NULL,       -- 0.0000 to 1.0000
    match_method        VARCHAR(30) NOT NULL,        -- deterministic, probabilistic, ml
    match_status        VARCHAR(20) NOT NULL,        -- confirmed, pending_review, rejected
    reviewed_by         UUID,
    reviewed_at         TIMESTAMPTZ,
    field_scores        JSONB NOT NULL DEFAULT '{}', -- per-field breakdown
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_patient_match_status ON patient_match_result(tenant_id, match_status);
CREATE INDEX idx_patient_match_score ON patient_match_result(tenant_id, match_score DESC);
```

## HL7 v2 Message Pipeline

```sql
-- ============================================================
-- HL7 V2 MESSAGING
-- ============================================================

CREATE TABLE hl7v2_channel (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    source_type     VARCHAR(30) NOT NULL,            -- mllp, http, sftp, database
    source_config   JSONB NOT NULL,                  -- connection details (encrypted)
    dest_type       VARCHAR(30) NOT NULL,
    dest_config     JSONB NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'stopped', -- running, stopped, error
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE hl7v2_message (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    channel_id      UUID NOT NULL REFERENCES hl7v2_channel(id),
    message_type    VARCHAR(10) NOT NULL,            -- ADT, ORM, ORU, MDM
    trigger_event   VARCHAR(10) NOT NULL,            -- A01, A08, R01, etc.
    control_id      VARCHAR(50) NOT NULL,
    sending_facility VARCHAR(255),
    receiving_facility VARCHAR(255),
    raw_message     TEXT NOT NULL,                   -- Original pipe-delimited HL7 v2
    processing_status VARCHAR(20) NOT NULL DEFAULT 'received',
    error_message   TEXT,
    fhir_bundle_id  UUID REFERENCES fhir_resource(id),  -- Resulting FHIR Bundle
    received_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    processed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_hl7v2_message_channel ON hl7v2_message(tenant_id, channel_id, received_at DESC);
CREATE INDEX idx_hl7v2_message_status ON hl7v2_message(tenant_id, processing_status);
CREATE INDEX idx_hl7v2_message_type ON hl7v2_message(tenant_id, message_type, trigger_event);

CREATE TABLE hl7v2_transformation_rule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    channel_id      UUID REFERENCES hl7v2_channel(id),  -- NULL = global rule
    name            VARCHAR(255) NOT NULL,
    source_segment  VARCHAR(10) NOT NULL,            -- PID, OBX, ORC, etc.
    source_field    VARCHAR(20) NOT NULL,            -- e.g., "PID.5"
    target_fhir_path VARCHAR(255) NOT NULL,          -- e.g., "Patient.name.family"
    transform_type  VARCHAR(30) NOT NULL,            -- direct, lookup, script, ai_generated
    transform_config JSONB NOT NULL,
    priority        INTEGER NOT NULL DEFAULT 0,
    active          BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_hl7v2_transform_channel ON hl7v2_transformation_rule(tenant_id, channel_id, source_segment);
```

## X12 EDI Transactions

```sql
-- ============================================================
-- X12 EDI TRANSACTIONS
-- ============================================================

CREATE TABLE x12_transaction (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    transaction_set     VARCHAR(10) NOT NULL,        -- 837, 835, 270, 271, 278
    direction           VARCHAR(10) NOT NULL,        -- inbound, outbound
    interchange_control VARCHAR(20) NOT NULL,        -- ISA13
    group_control       VARCHAR(20) NOT NULL,        -- GS06
    sender_id           VARCHAR(50) NOT NULL,
    receiver_id         VARCHAR(50) NOT NULL,
    raw_edi             TEXT NOT NULL,
    processing_status   VARCHAR(20) NOT NULL DEFAULT 'received',
    error_message       TEXT,
    fhir_claim_id       UUID REFERENCES fhir_resource(id),
    received_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    processed_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_x12_transaction_set ON x12_transaction(tenant_id, transaction_set, received_at DESC);
CREATE INDEX idx_x12_transaction_status ON x12_transaction(tenant_id, processing_status);
```

## SMART on FHIR Authorization

```sql
-- ============================================================
-- SMART ON FHIR / OAUTH2
-- ============================================================

CREATE TABLE oauth2_client (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    client_id       VARCHAR(255) NOT NULL UNIQUE,
    client_name     VARCHAR(255) NOT NULL,
    client_type     VARCHAR(20) NOT NULL,            -- public, confidential
    redirect_uris   TEXT[] NOT NULL,
    grant_types     TEXT[] NOT NULL,                  -- authorization_code, client_credentials
    scopes          TEXT[] NOT NULL,                  -- patient/*.read, system/*.write, etc.
    jwks_uri        VARCHAR(500),                    -- For UDAP dynamic registration
    logo_uri        VARCHAR(500),
    active          BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE oauth2_authorization (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    client_id       UUID NOT NULL REFERENCES oauth2_client(id),
    user_id         UUID,
    patient_fhir_id VARCHAR(64),                     -- SMART launch context patient
    encounter_fhir_id VARCHAR(64),                   -- SMART launch context encounter
    scopes_granted  TEXT[] NOT NULL,
    code            VARCHAR(255),                    -- Authorization code (hashed)
    code_expires_at TIMESTAMPTZ,
    access_token_hash VARCHAR(64),
    access_token_expires_at TIMESTAMPTZ,
    refresh_token_hash VARCHAR(64),
    refresh_token_expires_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_oauth2_auth_client ON oauth2_authorization(tenant_id, client_id);
CREATE INDEX idx_oauth2_auth_token ON oauth2_authorization(access_token_hash) WHERE access_token_hash IS NOT NULL;
```

## Audit & Compliance

```sql
-- ============================================================
-- AUDIT LOGGING (FHIR AuditEvent aligned)
-- ============================================================

CREATE TABLE audit_event (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    event_type      VARCHAR(50) NOT NULL,            -- rest, login, export, patient-match
    event_subtype   VARCHAR(50),                     -- create, read, update, delete, search
    event_action    VARCHAR(10) NOT NULL,            -- C, R, U, D, E (execute)
    event_outcome   VARCHAR(10) NOT NULL,            -- 0 (success), 4 (minor), 8 (serious), 12 (major)
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    agent_type      VARCHAR(30) NOT NULL,            -- user, system, device
    agent_id        VARCHAR(255) NOT NULL,
    agent_name      VARCHAR(255),
    agent_ip        INET,
    source_site     VARCHAR(255),
    entity_type     VARCHAR(50),                     -- FHIR resource type
    entity_fhir_id  VARCHAR(64),
    entity_query    TEXT,                            -- Search query if applicable
    detail          JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_event_time ON audit_event(tenant_id, recorded_at DESC);
CREATE INDEX idx_audit_event_entity ON audit_event(tenant_id, entity_type, entity_fhir_id);
CREATE INDEX idx_audit_event_agent ON audit_event(tenant_id, agent_id, recorded_at DESC);

-- Partition by month for retention management
-- ALTER TABLE audit_event PARTITION BY RANGE (recorded_at);

-- ============================================================
-- COMPLIANCE TRACKING
-- ============================================================

CREATE TABLE compliance_rule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    regulation      VARCHAR(50) NOT NULL,            -- USCDI, ONC_INFO_BLOCKING, CMS_0057_F, HIPAA
    rule_code       VARCHAR(100) NOT NULL,
    description     TEXT NOT NULL,
    check_query     TEXT,                            -- SQL or FHIR search to evaluate
    severity        VARCHAR(20) NOT NULL DEFAULT 'warning',
    active          BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE compliance_check_result (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    rule_id         UUID NOT NULL REFERENCES compliance_rule(id),
    status          VARCHAR(20) NOT NULL,            -- pass, fail, warning, error
    details         JSONB,
    checked_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_compliance_result_tenant ON compliance_check_result(tenant_id, checked_at DESC);
```

## Terminology Services

```sql
-- ============================================================
-- TERMINOLOGY
-- ============================================================

CREATE TABLE terminology_code_system (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    url             VARCHAR(500) NOT NULL UNIQUE,    -- e.g., "http://loinc.org"
    name            VARCHAR(255) NOT NULL,
    version         VARCHAR(50),
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    content         VARCHAR(20) NOT NULL,            -- complete, fragment, supplement
    concept_count   INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE terminology_concept (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code_system_id  UUID NOT NULL REFERENCES terminology_code_system(id),
    code            VARCHAR(100) NOT NULL,
    display         VARCHAR(500) NOT NULL,
    definition      TEXT,
    active          BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (code_system_id, code)
);

CREATE INDEX idx_terminology_concept_code ON terminology_concept(code_system_id, code);

CREATE TABLE terminology_value_set (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    url             VARCHAR(500) NOT NULL UNIQUE,
    name            VARCHAR(255) NOT NULL,
    version         VARCHAR(50),
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE terminology_value_set_member (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    value_set_id    UUID NOT NULL REFERENCES terminology_value_set(id),
    concept_id      UUID NOT NULL REFERENCES terminology_concept(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (value_set_id, concept_id)
);
```

## RBAC & User Management

```sql
-- ============================================================
-- RBAC
-- ============================================================

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    external_id     VARCHAR(255),                    -- IdP subject identifier
    email           VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE TABLE role (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(100) NOT NULL,           -- admin, clinician, analyst, integration_engine
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

CREATE TABLE permission (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    resource_type   VARCHAR(50) NOT NULL,            -- Patient, Observation, *, hl7v2_channel
    action          VARCHAR(20) NOT NULL,            -- read, write, delete, search, export
    scope           VARCHAR(50),                     -- patient, user, system (SMART scopes)
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (resource_type, action, scope)
);

CREATE TABLE role_permission (
    role_id         UUID NOT NULL REFERENCES role(id),
    permission_id   UUID NOT NULL REFERENCES permission(id),
    PRIMARY KEY (role_id, permission_id)
);

CREATE TABLE user_role (
    user_id         UUID NOT NULL REFERENCES app_user(id),
    role_id         UUID NOT NULL REFERENCES role(id),
    organization_id UUID REFERENCES organization(id),  -- scoped to organization
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    granted_by      UUID REFERENCES app_user(id),
    PRIMARY KEY (user_id, role_id, organization_id)
);
```

## FHIR Subscriptions

```sql
-- ============================================================
-- FHIR SUBSCRIPTIONS
-- ============================================================

CREATE TABLE fhir_subscription (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    fhir_id         VARCHAR(64) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'requested', -- requested, active, error, off
    criteria        VARCHAR(500) NOT NULL,           -- e.g., "Observation?code=http://loinc.org|1234-5"
    channel_type    VARCHAR(30) NOT NULL,            -- rest-hook, websocket, email
    channel_endpoint VARCHAR(500) NOT NULL,
    channel_headers JSONB,
    filter_criteria JSONB,                           -- R5-backport topic-based filter
    expiration      TIMESTAMPTZ,
    error_count     INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE fhir_subscription_delivery (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subscription_id UUID NOT NULL REFERENCES fhir_subscription(id),
    resource_id     UUID NOT NULL REFERENCES fhir_resource(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'pending', -- pending, delivered, failed
    attempt_count   INTEGER NOT NULL DEFAULT 0,
    last_attempt_at TIMESTAMPTZ,
    error_message   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_subscription_delivery_pending ON fhir_subscription_delivery(status, created_at)
    WHERE status = 'pending';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Tenant & Organization | 2 | Multi-tenancy foundation |
| FHIR Resource Storage | 2 | Body + history |
| Search Parameter Indexes | 6 | String, token, date, reference, quantity, URI |
| Patient Identity / MPI | 4 | Identity, identifiers, demographics, match results |
| HL7 v2 Pipeline | 3 | Channels, messages, transformation rules |
| X12 EDI | 1 | Transaction storage |
| SMART on FHIR / OAuth2 | 2 | Client registration, authorization lifecycle |
| Audit & Compliance | 3 | Events, rules, check results |
| Terminology | 4 | Code systems, concepts, value sets, members |
| RBAC | 5 | Users, roles, permissions, junctions |
| FHIR Subscriptions | 2 | Subscriptions, delivery tracking |
| **Total** | **34** | Core schema; expands to ~50+ with additional resource-specific indexes |

---

## Key Design Decisions

1. **Single `fhir_resource` table rather than table-per-resource-type.** While HAPI FHIR uses a single table approach and Aidbox/Fhirbase use per-type tables, a single table simplifies schema management and cross-resource queries at the cost of per-type partitioning. The search index tables handle type-specific querying.

2. **Separate search index tables per data type (HAPI pattern).** This mirrors the proven HAPI FHIR HFJ_SPIDX approach where each search parameter data type gets its own table, enabling type-specific indexes and avoiding the overhead of a single polymorphic index table.

3. **Patient identity as a first-class domain, not just a FHIR resource.** The MPI tables (`patient_identity`, `patient_identifier`, `patient_demographics`, `patient_match_result`) exist alongside the FHIR Patient resource to support cross-system identity resolution with match scoring — a capability absent from HAPI FHIR's core schema.

4. **HL7 v2 messages stored in their original format alongside FHIR translations.** The `hl7v2_message` table preserves the raw pipe-delimited message and links to the resulting FHIR Bundle, enabling reprocessing and audit without data loss.

5. **UUID primary keys throughout.** Consistent with modern SaaS patterns and avoids sequential ID leakage. Foreign keys reference UUIDs, which is slightly less performant than bigint but more portable and merge-safe across distributed deployments.

6. **Tenant-scoped rows with composite indexes.** Every data table includes `tenant_id` and all indexes lead with it, enabling row-level security policies for multi-tenant isolation without schema-per-tenant overhead.

7. **Audit events aligned with FHIR AuditEvent resource structure.** The `audit_event` table columns map directly to FHIR AuditEvent fields, making it straightforward to expose audit data as FHIR resources and meet HIPAA audit logging requirements.

8. **SMART on FHIR authorization modeled as relational tables.** Token hashes (never plaintext) are stored with explicit scope arrays, enabling RBAC enforcement at the database level and supporting SMART launch context (patient, encounter) for EHR-embedded apps.
