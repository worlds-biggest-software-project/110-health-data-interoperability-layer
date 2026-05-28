# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Health Data Interoperability Layer · Created: 2026-05-19

## Philosophy

This model combines a relational PostgreSQL backbone for operational CRUD with a property graph layer for relationship-heavy queries — patient identity resolution, care team networks, referral chains, document provenance, and conflict-of-interest detection. The graph layer is implemented using PostgreSQL's `ltree` extension for hierarchies and explicit `graph_node` / `graph_edge` tables for arbitrary relationship traversal, with the option to synchronize to Neo4j or Apache AGE for complex graph analytics.

Healthcare data is inherently graph-shaped. A patient has relationships with providers, encounters, conditions, medications, organizations, and insurance plans. A single HL7 v2 message creates a web of linked FHIR resources. Patient identity resolution is fundamentally a graph problem: clusters of potentially matching records connected by similarity edges. Care coordination requires traversing provider-patient-encounter networks. Neo4j is already used in healthcare for patient journey analysis, drug interaction graphs, and clinical ontology traversal (SNOMED CT, ICD-10 hierarchies).

This approach keeps transactional FHIR operations in relational tables (where ACID guarantees and SQL joins are strong) while exposing a graph query interface for analytics, identity resolution, and network analysis. The graph layer is populated asynchronously from FHIR resource changes.

**Best for:** Organizations that need advanced patient identity resolution (probabilistic matching as graph clustering), care network analysis, referral pattern optimization, or clinical ontology traversal alongside standard FHIR operations.

**Trade-offs:**
- (+) Patient identity resolution becomes a natural graph clustering problem
- (+) Relationship traversal (care teams, referral chains, document provenance) is orders of magnitude faster than recursive SQL
- (+) Clinical ontologies (SNOMED CT, ICD-10, LOINC) are natively graph structures
- (+) Enables AI-powered insights: "find all patients connected to this provider network within 3 hops"
- (+) Graph visualization tools provide intuitive UI for identity resolution review
- (-) Dual storage (relational + graph) increases operational complexity
- (-) Graph synchronization introduces eventual consistency for graph queries
- (-) Team needs familiarity with graph query patterns (Cypher or recursive CTEs)
- (-) PostgreSQL-native graph queries (recursive CTEs, ltree) are slower than dedicated graph databases for deep traversals
- (-) More infrastructure to manage if using an external graph database

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| HL7 FHIR R4 | FHIR resources stored in relational tables; FHIR references (`subject`, `encounter`, `performer`) extracted as graph edges |
| HL7 v2.x | Message routing channels modeled as graph nodes with edges to source/destination systems |
| IHE PIXm / PDQm | Patient identity resolution modeled as a graph: patient records are nodes, similarity scores are weighted edges, golden records are cluster centroids |
| SNOMED CT / ICD-10 / LOINC | Clinical terminologies loaded as graph hierarchies using `ltree` paths or dedicated ontology graph |
| SMART on FHIR | Authorization relationships (user → role → permission → resource) modeled as graph paths |
| HIPAA Security Rule | Access audit trail captured as graph edges (who accessed what, when, via which app) |
| TEFCA / QHIN | Network topology (QHIN → participant → endpoint) modeled as a graph for routing decisions |

---

## Relational Core (Operational FHIR Storage)

```sql
-- ============================================================
-- TENANT
-- ============================================================

CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    config          JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- FHIR RESOURCE STORAGE (single table, JSONB body)
-- ============================================================

CREATE TABLE fhir_resource (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    resource_type   VARCHAR(50) NOT NULL,
    fhir_id         VARCHAR(64) NOT NULL,
    version_id      INTEGER NOT NULL DEFAULT 1,
    last_updated    TIMESTAMPTZ NOT NULL DEFAULT now(),
    is_deleted      BOOLEAN NOT NULL DEFAULT false,
    source          VARCHAR(255),
    resource_body   JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, resource_type, fhir_id)
);

CREATE INDEX idx_fhir_resource_type ON fhir_resource(tenant_id, resource_type, last_updated DESC);
CREATE INDEX idx_fhir_resource_body ON fhir_resource USING gin(resource_body jsonb_path_ops);

CREATE TABLE fhir_resource_history (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    resource_id     UUID NOT NULL REFERENCES fhir_resource(id),
    tenant_id       UUID NOT NULL,
    resource_type   VARCHAR(50) NOT NULL,
    fhir_id         VARCHAR(64) NOT NULL,
    version_id      INTEGER NOT NULL,
    resource_body   JSONB NOT NULL,
    request_method  VARCHAR(10),
    last_updated    TIMESTAMPTZ NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_fhir_history ON fhir_resource_history(resource_id, version_id DESC);
```

## Property Graph Layer

```sql
-- ============================================================
-- GRAPH NODES
-- Every entity in the system gets a graph node.
-- Nodes are thin: the full data lives in the relational tables.
-- ============================================================

CREATE TABLE graph_node (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    node_type       VARCHAR(50) NOT NULL,
    -- Node types: Patient, Practitioner, Organization, Encounter,
    --             Observation, Condition, Device, Location,
    --             HL7v2Channel, HL7v2Message, X12Transaction,
    --             TermConcept, OAuthClient
    external_id     VARCHAR(255) NOT NULL,           -- FHIR ID or system identifier
    label           VARCHAR(500),                    -- Human-readable label
    properties      JSONB NOT NULL DEFAULT '{}',     -- Indexed properties for graph queries
    -- Example (Patient node):
    -- {
    --   "family_name": "Smith",
    --   "birth_date": "1985-03-15",
    --   "gender": "male",
    --   "source_system": "http://hospital-a.org"
    -- }
    source_table    VARCHAR(100),                    -- e.g., "fhir_resource"
    source_id       UUID,                            -- FK to source record
    active          BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, node_type, external_id)
);

CREATE INDEX idx_gn_tenant_type ON graph_node(tenant_id, node_type);
CREATE INDEX idx_gn_properties ON graph_node USING gin(properties jsonb_path_ops);
CREATE INDEX idx_gn_source ON graph_node(source_table, source_id) WHERE source_id IS NOT NULL;

-- ============================================================
-- GRAPH EDGES
-- Directed, typed, weighted relationships between nodes.
-- ============================================================

CREATE TABLE graph_edge (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    source_node_id  UUID NOT NULL REFERENCES graph_node(id) ON DELETE CASCADE,
    target_node_id  UUID NOT NULL REFERENCES graph_node(id) ON DELETE CASCADE,
    edge_type       VARCHAR(80) NOT NULL,
    -- Edge types:
    --   FHIR references: HAS_SUBJECT, HAS_ENCOUNTER, HAS_PERFORMER, HAS_AUTHOR,
    --                     BELONGS_TO_ORG, HAS_CONDITION, HAS_MEDICATION
    --   Identity:        POSSIBLY_SAME_AS, CONFIRMED_SAME_AS, MERGED_INTO
    --   Care network:    REFERS_TO, TREATS, MEMBER_OF_CARE_TEAM
    --   Messaging:       SENT_VIA_CHANNEL, TRANSFORMED_TO, ROUTED_TO
    --   Terminology:     IS_A, PART_OF, HAS_COMPONENT
    --   Authorization:   HAS_ROLE, GRANTS_PERMISSION, AUTHORIZED_ACCESS
    weight          NUMERIC(8,4) DEFAULT 1.0,        -- Match scores, confidence, priority
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Example (POSSIBLY_SAME_AS):
    -- {
    --   "match_score": 0.9234,
    --   "match_method": "probabilistic",
    --   "field_scores": {"family_name": 0.95, "birth_date": 1.0, "address": 0.75}
    -- }
    valid_from      TIMESTAMPTZ NOT NULL DEFAULT now(),
    valid_to        TIMESTAMPTZ,                     -- NULL = currently valid
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ge_source ON graph_edge(source_node_id, edge_type);
CREATE INDEX idx_ge_target ON graph_edge(target_node_id, edge_type);
CREATE INDEX idx_ge_type ON graph_edge(tenant_id, edge_type);
CREATE INDEX idx_ge_weight ON graph_edge(tenant_id, edge_type, weight DESC);
CREATE INDEX idx_ge_temporal ON graph_edge(tenant_id, edge_type, valid_from, valid_to);
CREATE INDEX idx_ge_properties ON graph_edge USING gin(properties jsonb_path_ops);

-- ============================================================
-- GRAPH PATHS (materialized for common traversal patterns)
-- ============================================================

CREATE EXTENSION IF NOT EXISTS ltree;

CREATE TABLE graph_path (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    root_node_id    UUID NOT NULL REFERENCES graph_node(id),
    node_id         UUID NOT NULL REFERENCES graph_node(id),
    path            LTREE NOT NULL,                  -- e.g., 'org_a.dept_cardiology.dr_smith.patient_123'
    depth           INTEGER NOT NULL,
    path_type       VARCHAR(50) NOT NULL,            -- org_hierarchy, terminology_hierarchy, care_chain
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_gp_path ON graph_path USING gist(path);
CREATE INDEX idx_gp_root ON graph_path(root_node_id);
CREATE INDEX idx_gp_node ON graph_path(node_id);
```

## Patient Identity as Graph Clustering

```sql
-- ============================================================
-- PATIENT IDENTITY CLUSTER
-- Golden records are cluster centroids; source records are members.
-- POSSIBLY_SAME_AS edges with match scores connect candidates.
-- ============================================================

CREATE TABLE patient_cluster (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    golden_node_id      UUID NOT NULL REFERENCES graph_node(id),  -- Cluster centroid
    status              VARCHAR(20) NOT NULL DEFAULT 'active',
    member_count        INTEGER NOT NULL DEFAULT 1,
    confidence_avg      NUMERIC(5,4),                -- Average match score within cluster
    last_evaluated_at   TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pc_tenant ON patient_cluster(tenant_id, status);

CREATE TABLE patient_cluster_member (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cluster_id      UUID NOT NULL REFERENCES patient_cluster(id),
    patient_node_id UUID NOT NULL REFERENCES graph_node(id),
    source_system   VARCHAR(255) NOT NULL,
    match_score     NUMERIC(5,4) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',  -- active, removed, disputed
    added_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (cluster_id, patient_node_id)
);

CREATE INDEX idx_pcm_cluster ON patient_cluster_member(cluster_id);
CREATE INDEX idx_pcm_patient ON patient_cluster_member(patient_node_id);

-- ============================================================
-- EXAMPLE: Find all candidate matches for a patient (graph traversal)
-- ============================================================

-- SELECT gn_target.external_id AS candidate_fhir_id,
--        gn_target.properties->>'family_name' AS candidate_name,
--        ge.weight AS match_score,
--        ge.properties->>'match_method' AS method
-- FROM graph_node gn_source
-- JOIN graph_edge ge ON ge.source_node_id = gn_source.id
--                    AND ge.edge_type = 'POSSIBLY_SAME_AS'
--                    AND ge.valid_to IS NULL
-- JOIN graph_node gn_target ON gn_target.id = ge.target_node_id
-- WHERE gn_source.tenant_id = 'tenant-uuid'
--   AND gn_source.node_type = 'Patient'
--   AND gn_source.external_id = 'pat-123'
-- ORDER BY ge.weight DESC;

-- ============================================================
-- EXAMPLE: Find connected patient cluster via recursive CTE
-- ============================================================

-- WITH RECURSIVE patient_network AS (
--     -- Start from a specific patient node
--     SELECT gn.id, gn.external_id, 0 AS depth
--     FROM graph_node gn
--     WHERE gn.external_id = 'pat-123'
--       AND gn.node_type = 'Patient'
--       AND gn.tenant_id = 'tenant-uuid'
--
--     UNION ALL
--
--     -- Traverse POSSIBLY_SAME_AS and CONFIRMED_SAME_AS edges
--     SELECT gn2.id, gn2.external_id, pn.depth + 1
--     FROM patient_network pn
--     JOIN graph_edge ge ON (ge.source_node_id = pn.id OR ge.target_node_id = pn.id)
--                        AND ge.edge_type IN ('POSSIBLY_SAME_AS', 'CONFIRMED_SAME_AS')
--                        AND ge.weight >= 0.7
--                        AND ge.valid_to IS NULL
--     JOIN graph_node gn2 ON gn2.id = CASE
--         WHEN ge.source_node_id = pn.id THEN ge.target_node_id
--         ELSE ge.source_node_id
--     END
--     WHERE pn.depth < 3  -- Limit traversal depth
-- )
-- SELECT DISTINCT external_id, depth
-- FROM patient_network
-- ORDER BY depth;
```

## HL7 v2 & X12 Pipeline (Relational + Graph Edges)

```sql
-- ============================================================
-- HL7 V2 CHANNELS & MESSAGES
-- ============================================================

CREATE TABLE hl7v2_channel (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    source_config   JSONB NOT NULL,
    dest_config     JSONB NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'stopped',
    graph_node_id   UUID REFERENCES graph_node(id),  -- Link to graph for topology
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
    raw_message         TEXT NOT NULL,
    parsed_segments     JSONB,
    processing_status   VARCHAR(20) NOT NULL DEFAULT 'received',
    error_message       TEXT,
    graph_node_id       UUID REFERENCES graph_node(id),  -- Message in graph for provenance
    received_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    processed_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_hl7v2_channel_time ON hl7v2_message(tenant_id, channel_id, received_at DESC);
CREATE INDEX idx_hl7v2_status ON hl7v2_message(tenant_id, processing_status);

-- Graph edges created by the message pipeline:
-- HL7v2Message --SENT_VIA_CHANNEL--> HL7v2Channel
-- HL7v2Message --TRANSFORMED_TO--> FhirResource (Patient, Observation, etc.)
-- HL7v2Message --ROUTED_TO--> Organization

CREATE TABLE hl7v2_transformation_rule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    channel_id      UUID REFERENCES hl7v2_channel(id),
    name            VARCHAR(255) NOT NULL,
    source_path     VARCHAR(100) NOT NULL,           -- e.g., "PID.5"
    target_fhir_path VARCHAR(255) NOT NULL,          -- e.g., "Patient.name.family"
    transform_type  VARCHAR(30) NOT NULL,
    transform_config JSONB NOT NULL,
    active          BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- X12 EDI
-- ============================================================

CREATE TABLE x12_transaction (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    transaction_set     VARCHAR(10) NOT NULL,
    direction           VARCHAR(10) NOT NULL,
    sender_id           VARCHAR(50) NOT NULL,
    receiver_id         VARCHAR(50) NOT NULL,
    raw_edi             TEXT NOT NULL,
    parsed_data         JSONB,
    processing_status   VARCHAR(20) NOT NULL DEFAULT 'received',
    graph_node_id       UUID REFERENCES graph_node(id),
    received_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    processed_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_x12_set ON x12_transaction(tenant_id, transaction_set, received_at DESC);
```

## Clinical Terminology as Graph

```sql
-- ============================================================
-- TERMINOLOGY ONTOLOGY GRAPH
-- SNOMED CT, ICD-10, LOINC loaded as graph nodes with IS_A edges
-- ============================================================

-- Terminology concepts are loaded as graph_node entries:
-- node_type = 'TermConcept'
-- properties = {
--   "code_system": "http://snomed.info/sct",
--   "code": "73211009",
--   "display": "Diabetes mellitus",
--   "semantic_tag": "disorder"
-- }

-- IS_A relationships loaded as graph_edge entries:
-- edge_type = 'IS_A'
-- "Diabetes mellitus type 2" --IS_A--> "Diabetes mellitus" --IS_A--> "Disorder of glucose metabolism"

-- Materialized paths using ltree for fast subsumption queries:
-- path = 'sct.73211009.44054006'  (Diabetes mellitus > Type 2 diabetes)

-- ============================================================
-- EXAMPLE: Find all subtypes of "Diabetes mellitus" (SNOMED 73211009)
-- ============================================================

-- Using ltree:
-- SELECT gn.properties->>'code' AS code,
--        gn.properties->>'display' AS display,
--        gp.depth
-- FROM graph_path gp
-- JOIN graph_node gn ON gn.id = gp.node_id
-- WHERE gp.path <@ 'sct.73211009'    -- All descendants
--   AND gp.path_type = 'terminology_hierarchy'
--   AND gp.tenant_id = 'tenant-uuid';

-- Using recursive CTE on graph edges:
-- WITH RECURSIVE subtypes AS (
--     SELECT gn.id, gn.properties->>'code' AS code,
--            gn.properties->>'display' AS display, 0 AS depth
--     FROM graph_node gn
--     WHERE gn.node_type = 'TermConcept'
--       AND gn.properties->>'code' = '73211009'
--       AND gn.properties->>'code_system' = 'http://snomed.info/sct'
--
--     UNION ALL
--
--     SELECT gn2.id, gn2.properties->>'code', gn2.properties->>'display', s.depth + 1
--     FROM subtypes s
--     JOIN graph_edge ge ON ge.target_node_id = s.id AND ge.edge_type = 'IS_A'
--     JOIN graph_node gn2 ON gn2.id = ge.source_node_id
--     WHERE s.depth < 5
-- )
-- SELECT * FROM subtypes;
```

## SMART on FHIR & Audit

```sql
-- ============================================================
-- SMART ON FHIR AUTHORIZATION
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
    active          BOOLEAN NOT NULL DEFAULT true,
    graph_node_id   UUID REFERENCES graph_node(id),  -- For auth relationship graph
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE oauth2_session (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    client_id           UUID NOT NULL REFERENCES oauth2_client(id),
    user_id             UUID,
    patient_fhir_id     VARCHAR(64),
    scopes_granted      TEXT[] NOT NULL,
    access_token_hash   VARCHAR(64),
    expires_at          TIMESTAMPTZ NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_oauth2_token ON oauth2_session(access_token_hash);

-- ============================================================
-- AUDIT EVENTS (relational for compliance, graph edges for analysis)
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
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (recorded_at);

CREATE INDEX idx_audit_time ON audit_event(tenant_id, recorded_at DESC);
CREATE INDEX idx_audit_entity ON audit_event(tenant_id, entity_type, entity_fhir_id);

-- Audit events also create graph edges:
-- User --AUTHORIZED_ACCESS--> Patient (with timestamp and scope in edge properties)
-- App --EXPORTED_DATA--> FHIRResource (with $export job details)
```

## RBAC as Graph

```sql
-- ============================================================
-- USERS & ROLES (relational + graph)
-- ============================================================

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    external_id     VARCHAR(255),
    email           VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    graph_node_id   UUID REFERENCES graph_node(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE TABLE role (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(100) NOT NULL,
    description     TEXT,
    graph_node_id   UUID REFERENCES graph_node(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

CREATE TABLE permission (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    resource_type   VARCHAR(50) NOT NULL,
    action          VARCHAR(20) NOT NULL,
    scope           VARCHAR(50),
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
    organization_id UUID,
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, role_id)
);

-- Graph edges for RBAC:
-- User --HAS_ROLE--> Role --GRANTS_PERMISSION--> Permission
-- User --MEMBER_OF--> Organization
-- Role --SCOPED_TO--> Organization

-- ============================================================
-- EXAMPLE: Check if user can access a patient (graph traversal)
-- ============================================================

-- Can user 'user-123' read Patient 'pat-456'?
-- Traverse: User --HAS_ROLE--> Role --GRANTS_PERMISSION--> Permission
-- Check: permission.resource_type IN ('Patient', '*') AND permission.action = 'read'
-- Then check scope: if scope = 'patient', verify
--   User --TREATS/MEMBER_OF_CARE_TEAM--> Patient path exists
```

## Network Topology (TEFCA / QHIN)

```sql
-- ============================================================
-- EXAMPLE: TEFCA network topology as graph
-- ============================================================

-- Graph nodes:
-- node_type = 'QHIN' (e.g., CommonWell, eHealth Exchange, Carequality)
-- node_type = 'Participant' (Health systems, payers)
-- node_type = 'Endpoint' (FHIR servers, XCA gateways)

-- Graph edges:
-- QHIN --HAS_PARTICIPANT--> Participant
-- Participant --HAS_ENDPOINT--> Endpoint
-- Endpoint --SUPPORTS_PROFILE--> profile node (US Core, Da Vinci PAS, etc.)

-- Routing query: "Find all endpoints that support Da Vinci PAS within
-- the eHealth Exchange QHIN"
-- SELECT ep.properties->>'url' AS endpoint_url,
--        p.properties->>'name' AS participant_name
-- FROM graph_node qhin
-- JOIN graph_edge e1 ON e1.source_node_id = qhin.id AND e1.edge_type = 'HAS_PARTICIPANT'
-- JOIN graph_node p ON p.id = e1.target_node_id
-- JOIN graph_edge e2 ON e2.source_node_id = p.id AND e2.edge_type = 'HAS_ENDPOINT'
-- JOIN graph_node ep ON ep.id = e2.target_node_id
-- JOIN graph_edge e3 ON e3.source_node_id = ep.id AND e3.edge_type = 'SUPPORTS_PROFILE'
-- JOIN graph_node prof ON prof.id = e3.target_node_id
--   AND prof.properties->>'url' = 'http://hl7.org/fhir/us/davinci-pas'
-- WHERE qhin.properties->>'name' = 'eHealth Exchange';
```

## Graph Synchronization

```sql
-- ============================================================
-- GRAPH SYNC TRACKING
-- Tracks which relational changes have been synced to graph
-- ============================================================

CREATE TABLE graph_sync_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_table    VARCHAR(100) NOT NULL,
    source_id       UUID NOT NULL,
    operation       VARCHAR(10) NOT NULL,            -- insert, update, delete
    synced          BOOLEAN NOT NULL DEFAULT false,
    sync_error      TEXT,
    queued_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    synced_at       TIMESTAMPTZ
);

CREATE INDEX idx_graph_sync_pending ON graph_sync_log(synced, queued_at)
    WHERE synced = false;

-- Trigger function to queue graph sync on FHIR resource changes
-- CREATE OR REPLACE FUNCTION queue_graph_sync()
-- RETURNS TRIGGER AS $$
-- BEGIN
--     INSERT INTO graph_sync_log (source_table, source_id, operation)
--     VALUES (TG_TABLE_NAME, NEW.id, TG_OP);
--     RETURN NEW;
-- END;
-- $$ LANGUAGE plpgsql;
--
-- CREATE TRIGGER fhir_resource_graph_sync
-- AFTER INSERT OR UPDATE ON fhir_resource
-- FOR EACH ROW EXECUTE FUNCTION queue_graph_sync();
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Tenant | 1 | Multi-tenancy foundation |
| FHIR Resource Storage | 2 | Current + history (single table with JSONB) |
| Graph Layer | 4 | Nodes, edges, materialized paths, sync log |
| Patient Identity Clusters | 2 | Cluster centroids + members |
| HL7 v2 Pipeline | 3 | Channels, messages, transformation rules |
| X12 EDI | 1 | Transactions with graph links |
| SMART on FHIR / OAuth2 | 2 | Clients, sessions |
| Audit | 1 | Events (partitioned) with graph edges for analysis |
| RBAC | 5 | Users, roles, permissions, junctions (mirrored in graph) |
| **Total** | **21** | Core relational + graph tables; graph nodes/edges replace many junction tables |

---

## Key Design Decisions

1. **Graph layer as a projection, not source of truth.** The relational tables remain the transactional source of truth for FHIR operations. The graph layer (`graph_node`, `graph_edge`, `graph_path`) is populated asynchronously from relational changes via triggers and a sync queue. This means graph queries may lag by milliseconds to seconds, but FHIR CRUD operations maintain full ACID guarantees.

2. **Generic node/edge tables rather than typed graph tables.** A single `graph_node` table with a `node_type` discriminator and a single `graph_edge` table with an `edge_type` discriminator. This provides maximum flexibility for adding new entity types and relationship types without schema changes. The JSONB `properties` columns hold type-specific attributes.

3. **Patient identity as graph clustering.** Instead of a traditional MPI with pairwise match results, patient identity resolution is modeled as a graph clustering problem. Source patient records are nodes; similarity scores are weighted edges (`POSSIBLY_SAME_AS`). Golden records are cluster centroids. Graph algorithms (connected components, community detection) identify patient clusters more naturally than pairwise table joins.

4. **Clinical terminologies as graph hierarchies.** SNOMED CT, ICD-10, and LOINC are loaded as `TermConcept` nodes with `IS_A` edges. Subsumption queries ("find all observations classified under Diabetes mellitus") use `ltree` paths for fast ancestor/descendant lookups, or recursive CTEs for dynamic traversal.

5. **FHIR references automatically extracted as graph edges.** When a FHIR resource is stored, its references (e.g., `Observation.subject` referencing a Patient) are extracted and created as typed graph edges (`HAS_SUBJECT`, `HAS_ENCOUNTER`, `HAS_PERFORMER`). This enables relationship-centric queries without parsing JSONB at query time.

6. **Temporal edges for historical relationship analysis.** Graph edges have `valid_from` and `valid_to` timestamps, enabling temporal queries like "what was this patient's care team on January 1?" or "when was this provider-patient relationship established?" Edges are never deleted; they are closed (valid_to set) and new edges opened.

7. **`ltree` for materialized hierarchy paths.** PostgreSQL's `ltree` extension enables fast ancestor/descendant queries on organizational hierarchies, terminology trees, and care chains without recursive CTEs. The `graph_path` table pre-computes these paths for common traversal patterns.

8. **Optional external graph database sync.** While the PostgreSQL graph layer handles most use cases, organizations needing complex graph analytics (community detection, PageRank, shortest path across thousands of nodes) can synchronize the `graph_node` and `graph_edge` tables to Neo4j or Apache AGE via the `graph_sync_log` queue. The schema is designed to make this export straightforward.

9. **RBAC mirrored in graph for authorization traversal.** User-role-permission relationships exist in both relational tables (for fast CRUD) and graph edges (for complex authorization queries like "can this user access this patient through any organizational path?"). The graph representation makes SMART on FHIR scope enforcement and delegation chains traversable.

10. **Network topology for TEFCA routing.** QHIN, participant, and endpoint relationships are modeled as graph nodes and edges, enabling routing queries like "find the shortest path to deliver a FHIR resource to a patient's payer through the TEFCA network."
