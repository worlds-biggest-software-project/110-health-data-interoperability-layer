# Health Data Interoperability Layer -- Development Plan

> Project: Health Data Interoperability Layer (Candidate #110)
> Created: 2026-05-25
> Based on: research.md, features.md, standards.md, README.md, data-model-suggestion-1 through 4

---

## Technology Decisions & Rationale

### Primary Language & Runtime: Java 21+ (Spring Boot 3.x)

**Rationale:** HAPI FHIR -- the de facto open-source FHIR reference implementation -- is written in Java. Building on HAPI FHIR as a library dependency avoids reimplementing the FHIR R4 specification from scratch (140+ resource types, search parameters, validation, terminology operations). Spring Boot provides mature production tooling (actuator, security, configuration), and the healthcare IT workforce is predominantly Java-literate. HAPI FHIR's Apache 2.0 licence is fully compatible with any commercial or open-source distribution model.

### Data Model: Hybrid Relational + JSONB (Suggestion 3) with Event Sourcing for Audit (from Suggestion 2)

**Rationale:** Data Model Suggestion 3 (Hybrid Relational + JSONB) provides the fastest path to a working FHIR server with the lowest schema complexity (~28 tables). It stores FHIR resources as native JSONB -- zero impedance mismatch with the wire format -- while keeping security-critical tables (OAuth2, audit, RBAC) fully relational. PostgreSQL's GENERATED ALWAYS AS columns and GIN indexes handle search parameter indexing without the ~80-table overhead of the fully normalized approach (Suggestion 1).

From Suggestion 2 (Event-Sourced/CQRS), we adopt an append-only `audit_event` table partitioned by month. This gives HIPAA-grade immutable audit trails without requiring the full CQRS complexity across the entire system. The graph layer from Suggestion 4 is deferred to the patient matching phase, where graph clustering for identity resolution will be introduced as a targeted subsystem rather than a system-wide architectural commitment.

### Database: PostgreSQL 16+

**Rationale:** PostgreSQL provides JSONB with GIN indexes, generated columns, table partitioning, row-level security, and the `ltree` extension -- all required by the chosen data model. It is the database behind HAPI FHIR JPA, Aidbox/Fhirbase, and Medplum. No additional database is needed for MVP; Redis is added in Phase 4 for caching and session management.

### Message Broker: Apache Kafka

**Rationale:** The HL7 v2 message pipeline, FHIR subscription delivery, X12 EDI routing, and AI anomaly detection all require durable, ordered, replayable event streams. Kafka provides exactly-once delivery semantics, partition-based parallelism, and a mature healthcare ecosystem (Confluent Healthcare connectors). For teams that cannot operate Kafka, a Kafka-compatible abstraction (Redpanda or even PostgreSQL LISTEN/NOTIFY for low-volume deployments) can be substituted.

### Authentication/Authorization: Keycloak + SMART on FHIR

**Rationale:** Keycloak is the most widely deployed open-source OAuth2/OIDC server compatible with SMART on FHIR. It supports SMART App Launch v2, PKCE, UDAP dynamic client registration, and can federate with Active Directory, LDAP, and SAML IdPs -- covering the full range of hospital and payer identity infrastructure. HAPI FHIR has native Keycloak interceptor support.

### Containerization & Orchestration: Docker + Kubernetes (Helm charts)

**Rationale:** Self-hosted, customer-managed cloud, and SaaS deployment modes all require container-based distribution. Kubernetes with Helm charts is the standard for healthcare IT infrastructure. The platform ships with docker-compose for local development and single-node deployment.

### AI/ML Runtime: Python sidecar services (FastAPI)

**Rationale:** AI-native features (probabilistic patient matching, LLM-assisted mapping, anomaly detection, NL query) use Python ML libraries (scikit-learn, PyTorch, transformers). These run as separate microservices communicating with the Java core via gRPC or REST, avoiding JVM/Python impedance and allowing independent scaling and model updates.

### Frontend: React + TypeScript (Next.js)

**Rationale:** The admin dashboard, channel builder, patient matching review UI, and compliance dashboard are built as a single React application. React dominates healthcare IT frontend development, and Next.js provides SSR for compliance-sensitive pages. The FHIR test panel reuses HAPI FHIR's existing web components where possible.

---

## Project Structure

```
health-data-interop/
  core/                          # Java (Spring Boot) -- FHIR server, HL7v2, X12, auth
    src/main/java/
      com/healthinterop/
        fhir/                    # FHIR R4 resource handlers, search, subscriptions
        hl7v2/                   # HL7 v2 MLLP listener, parser, transformer
        x12/                     # X12 EDI parser, router
        mpi/                     # Master Patient Index, deterministic matching
        auth/                    # SMART on FHIR, OAuth2, RBAC
        audit/                   # Audit event capture, compliance checks
        terminology/             # CodeSystem, ValueSet operations
        channel/                 # Channel management, routing engine
        config/                  # Spring configuration, multi-tenancy
    src/main/resources/
      db/migration/              # Flyway SQL migrations
      application.yml
  ai-services/                   # Python (FastAPI) -- ML/AI microservices
    patient-matching/            # Probabilistic MPI model
    mapping-engine/              # LLM-assisted HL7v2-to-FHIR mapping
    anomaly-detection/           # Data quality monitoring
    nl-query/                    # Natural language FHIR query
  dashboard/                     # React + TypeScript (Next.js) -- Admin UI
    src/
      app/                       # Next.js app router
      components/                # Shared UI components
      features/
        channels/                # Channel builder and monitoring
        patients/                # MPI review, merge/split
        compliance/              # Compliance dashboard
        fhir-browser/            # FHIR resource explorer
  deploy/                        # Deployment configurations
    docker/                      # docker-compose for local dev
    helm/                        # Kubernetes Helm charts
    terraform/                   # Cloud infrastructure as code
  docs/                          # Technical documentation
  e2e-tests/                     # End-to-end integration tests
```

---

## Phase Dependency Graph

```
Phase 1: Foundation
    |
    v
Phase 2: FHIR Server Core
    |
    +-------+-------+
    |               |
    v               v
Phase 3:        Phase 4:
HL7 v2          Auth & Security
Pipeline        (SMART on FHIR)
    |               |
    +-------+-------+
            |
            v
        Phase 5:
        Patient Identity (MPI)
            |
            +-------+-------+
            |               |
            v               v
        Phase 6:        Phase 7:
        Dashboard &     X12 EDI &
        Operations      Payer Workflows
            |               |
            +-------+-------+
                    |
                    v
                Phase 8:
                AI-Native Features
                    |
                    v
                Phase 9:
                Bulk Data & Analytics
                    |
                    v
                Phase 10:
                Compliance & TEFCA
                    |
                    v
                Phase 11:
                Production Hardening
                    |
                    v
                Phase 12:
                FHIR R5 & Extensibility
```

**Dependency Rules:**
- Phase 2 requires Phase 1 (database, project skeleton)
- Phases 3 and 4 can run in parallel after Phase 2
- Phase 5 requires both Phase 3 (HL7 v2 feeds create patients) and Phase 4 (auth protects MPI)
- Phases 6 and 7 can run in parallel after Phase 5
- Phase 8 requires Phase 5 (AI matching) and benefits from Phase 6 (dashboard to display AI results)
- Phases 9-12 are sequential but can overlap with ongoing stabilization work

---

## Phase 1: Foundation & Project Scaffolding

**Goal:** Establish the project skeleton, CI/CD pipeline, database schema, multi-tenancy foundation, and local development environment.

**Definition of Done:** A developer can clone the repo, run `docker-compose up`, and see a running (empty) Spring Boot application connected to PostgreSQL with all base tables created via Flyway migration.

### Task 1.1: Repository & Build System Setup

**What:** Initialize the monorepo with Gradle (multi-project build) for the Java core, Python AI services, and Next.js dashboard. Configure linting, formatting, and dependency management.

**Design:**
```
// build.gradle.kts (root)
plugins {
    id("org.springframework.boot") version "3.4.0" apply false
    id("io.spring.dependency-management") version "1.1.7" apply false
}

subprojects {
    group = "com.healthinterop"
    version = "0.1.0-SNAPSHOT"
}

// core/build.gradle.kts
plugins {
    id("org.springframework.boot")
    id("io.spring.dependency-management")
    java
}

dependencies {
    implementation("ca.uhn.hapi.fhir:hapi-fhir-jpaserver-base:7.6.0")
    implementation("ca.uhn.hapi.fhir:hapi-fhir-structures-r4:7.6.0")
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    implementation("org.flywaydb:flyway-core")
    runtimeOnly("org.postgresql:postgresql")
}

java {
    toolchain {
        languageVersion.set(JavaLanguageVersion.of(21))
    }
}
```

**Testing:**
- `./gradlew build` completes without errors
- `./gradlew check` runs Checkstyle and SpotBugs with zero violations
- Verify Java 21 toolchain is resolved correctly
- Verify HAPI FHIR 7.6.0 dependency resolves

### Task 1.2: Docker Compose Local Development Environment

**What:** Create docker-compose.yml with PostgreSQL 16, Kafka (KRaft mode), and the Spring Boot application. Include health checks and volume mounts for persistent data.

**Design:**
```yaml
# deploy/docker/docker-compose.yml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: healthinterop
      POSTGRES_USER: healthinterop
      POSTGRES_PASSWORD: localdev
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-LINE", "pg_isready", "-U", "healthinterop"]
      interval: 5s
      retries: 5

  kafka:
    image: apache/kafka:3.9.0
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093
      KAFKA_LOG_DIRS: /var/lib/kafka/data
    ports:
      - "9092:9092"

  core:
    build:
      context: ../../core
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/healthinterop
      SPRING_DATASOURCE_USERNAME: healthinterop
      SPRING_DATASOURCE_PASSWORD: localdev
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
    depends_on:
      postgres:
        condition: service_healthy

volumes:
  pgdata:
```

**Testing:**
- `docker-compose up -d` starts all services within 60 seconds
- `curl http://localhost:8080/actuator/health` returns `{"status":"UP"}`
- PostgreSQL accepts connections on port 5432
- Kafka broker is reachable on port 9092

### Task 1.3: Database Schema -- Multi-Tenancy & Core Tables

**What:** Create Flyway migrations for the tenant, organization, app_user, role, and permission tables. Implement row-level security policies for multi-tenant isolation.

**Design:**
```sql
-- V001__tenant_and_org.sql
CREATE TABLE tenant (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name        VARCHAR(255) NOT NULL,
    slug        VARCHAR(100) NOT NULL UNIQUE,
    status      VARCHAR(20) NOT NULL DEFAULT 'active',
    config      JSONB NOT NULL DEFAULT '{}',
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organization (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID NOT NULL REFERENCES tenant(id),
    fhir_id     VARCHAR(64) NOT NULL,
    resource    JSONB NOT NULL,
    name        VARCHAR(255) GENERATED ALWAYS AS (resource->>'name') STORED,
    npi         VARCHAR(10),
    active      BOOLEAN GENERATED ALWAYS AS ((resource->>'active')::boolean) STORED,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, fhir_id)
);

-- V002__rbac.sql
CREATE TABLE app_user (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id     UUID NOT NULL REFERENCES tenant(id),
    external_id   VARCHAR(255),
    email         VARCHAR(255) NOT NULL,
    display_name  VARCHAR(255) NOT NULL,
    status        VARCHAR(20) NOT NULL DEFAULT 'active',
    preferences   JSONB NOT NULL DEFAULT '{}',
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE TABLE role (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID NOT NULL REFERENCES tenant(id),
    name        VARCHAR(100) NOT NULL,
    permissions JSONB NOT NULL DEFAULT '[]',
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
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

**Testing:**
- Flyway migrations run on application startup without errors
- Inserting a tenant and organization succeeds; foreign key constraints are enforced
- Attempting to insert a duplicate tenant slug fails with unique violation
- Row-level security policy blocks cross-tenant reads when enabled

### Task 1.4: CI/CD Pipeline

**What:** Configure GitHub Actions for build, test, lint, and Docker image publishing. Include PostgreSQL service container for integration tests.

**Design:**
```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]
jobs:
  build:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: healthinterop_test
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
        ports: ["5432:5432"]
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
      - run: ./gradlew build
      - run: ./gradlew test
        env:
          SPRING_DATASOURCE_URL: jdbc:postgresql://localhost:5432/healthinterop_test
```

**Testing:**
- Push to `main` triggers the CI pipeline
- Pipeline completes green with all tests passing
- Docker image is built and tagged with commit SHA
- Integration tests connect to the PostgreSQL service container

---

## Phase 2: FHIR R4 Server Core

**Goal:** Deliver a standards-compliant FHIR R4 server with CRUD, search, history, transactions, subscriptions, and terminology services -- backed by the Hybrid JSONB data model.

**Definition of Done:** The server passes the FHIR R4 Touchstone Basic Server test suite for Patient, Observation, Condition, Encounter, and MedicationRequest resources. The CapabilityStatement accurately reflects supported operations.

### Task 2.1: FHIR Resource Storage Layer

**What:** Implement the table-per-type JSONB storage pattern for the 9 core FHIR resource types (Patient, Observation, Condition, Encounter, MedicationRequest, DiagnosticReport, AllergyIntolerance, DocumentReference, Claim) plus the catch-all `fhir_resource_other` table. Include history tables for version tracking.

**Design:**
```sql
-- V003__fhir_patient.sql
CREATE TABLE fhir_patient (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id     UUID NOT NULL REFERENCES tenant(id),
    fhir_id       VARCHAR(64) NOT NULL,
    version_id    INTEGER NOT NULL DEFAULT 1,
    last_updated  TIMESTAMPTZ NOT NULL DEFAULT now(),
    is_deleted    BOOLEAN NOT NULL DEFAULT false,
    source        VARCHAR(255),
    resource      JSONB NOT NULL,
    family_name   VARCHAR(255) GENERATED ALWAYS AS (
                      resource#>>'{name,0,family}'
                  ) STORED,
    birth_date    DATE GENERATED ALWAYS AS (
                      (resource->>'birthDate')::date
                  ) STORED,
    gender        VARCHAR(20) GENERATED ALWAYS AS (
                      resource->>'gender'
                  ) STORED,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, fhir_id)
);

CREATE INDEX idx_patient_tenant ON fhir_patient(tenant_id, last_updated DESC);
CREATE INDEX idx_patient_name ON fhir_patient(tenant_id, family_name);
CREATE INDEX idx_patient_dob ON fhir_patient(tenant_id, birth_date);
CREATE INDEX idx_patient_resource ON fhir_patient USING gin(resource jsonb_path_ops);

CREATE TABLE fhir_patient_history (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    current_id    UUID NOT NULL REFERENCES fhir_patient(id),
    tenant_id     UUID NOT NULL,
    fhir_id       VARCHAR(64) NOT NULL,
    version_id    INTEGER NOT NULL,
    resource      JSONB NOT NULL,
    request_method VARCHAR(10),
    last_updated  TIMESTAMPTZ NOT NULL,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

```java
// FhirResourceRepository.java
@Repository
public class FhirPatientRepository {

    @Autowired
    private JdbcTemplate jdbc;

    public FhirPatient create(UUID tenantId, IBaseResource resource) {
        String fhirId = UUID.randomUUID().toString();
        String json = FhirContext.forR4().newJsonParser()
            .encodeResourceToString(resource);

        return jdbc.queryForObject(
            """
            INSERT INTO fhir_patient (tenant_id, fhir_id, resource)
            VALUES (?, ?, ?::jsonb)
            RETURNING id, fhir_id, version_id, last_updated
            """,
            new FhirPatientRowMapper(),
            tenantId, fhirId, json
        );
    }

    public Optional<FhirPatient> read(UUID tenantId, String fhirId) {
        return jdbc.query(
            """
            SELECT id, fhir_id, version_id, last_updated, resource
            FROM fhir_patient
            WHERE tenant_id = ? AND fhir_id = ? AND is_deleted = false
            """,
            new FhirPatientRowMapper(),
            tenantId, fhirId
        ).stream().findFirst();
    }
}
```

**Testing:**
- Create a Patient resource via POST; verify 201 with Location header containing FHIR ID
- Read the created Patient via GET; verify returned JSON matches the submitted resource
- Update the Patient via PUT; verify version_id increments to 2
- Verify the previous version is stored in fhir_patient_history
- Delete via DELETE; verify is_deleted=true and GET returns 410 Gone
- Verify generated columns (family_name, birth_date, gender) are populated correctly
- Verify GIN index is used via EXPLAIN ANALYZE on a containment query

### Task 2.2: FHIR Search Implementation

**What:** Implement FHIR search across all supported resource types. Support string, token, date, reference, quantity, and composite search parameters. Implement pagination, _include, _revinclude, and _sort.

**Design:**
```java
// FhirSearchService.java
@Service
public class FhirSearchService {

    public Bundle search(UUID tenantId, String resourceType,
                         Map<String, List<String>> params) {
        SearchQueryBuilder builder = new SearchQueryBuilder(tenantId, resourceType);

        for (var entry : params.entrySet()) {
            String paramName = entry.getKey();
            SearchParamDefinition def = searchParamRegistry.get(resourceType, paramName);

            switch (def.getType()) {
                case STRING -> builder.addStringCondition(paramName, entry.getValue());
                case TOKEN -> builder.addTokenCondition(paramName, entry.getValue());
                case DATE -> builder.addDateCondition(paramName, entry.getValue());
                case REFERENCE -> builder.addReferenceCondition(paramName, entry.getValue());
                case QUANTITY -> builder.addQuantityCondition(paramName, entry.getValue());
            }
        }

        // Pagination
        int count = parseCount(params, 20);
        int offset = parseOffset(params);
        builder.setLimit(count + 1); // +1 to detect next page
        builder.setOffset(offset);

        List<IBaseResource> results = builder.execute();
        return bundleBuilder.buildSearchSet(results, count, offset);
    }
}

// SearchQueryBuilder -- generates SQL for JSONB queries
// For generated columns: WHERE family_name = ?
// For GIN-indexed paths: WHERE resource @> ?::jsonb
// For complex paths: WHERE resource #>> '{code,coding,0,code}' = ?
```

**Testing:**
- `GET /fhir/Patient?name=Smith` returns patients with family name Smith
- `GET /fhir/Patient?birthdate=1985-03-15` returns exact date match
- `GET /fhir/Patient?birthdate=gt2000-01-01` returns patients born after 2000
- `GET /fhir/Observation?code=http://loinc.org|8867-4` returns heart rate observations
- `GET /fhir/Observation?subject=Patient/pat-123` returns observations for a specific patient
- `GET /fhir/Patient?_count=10&_offset=20` returns correct pagination with next/prev links
- `GET /fhir/Patient?_include=Patient:organization` includes referenced organizations
- `GET /fhir/Patient?_sort=-birthdate` returns patients sorted by birth date descending
- Verify search against 10,000+ resources completes in under 200ms

### Task 2.3: FHIR Transactions & Batch Bundles

**What:** Implement FHIR transaction and batch bundle processing. Transactions are atomic (all-or-nothing); batches process each entry independently.

**Design:**
```java
@Service
public class FhirBundleProcessor {

    @Transactional
    public Bundle processTransaction(UUID tenantId, Bundle transactionBundle) {
        // Validate all entries first
        for (BundleEntryComponent entry : transactionBundle.getEntry()) {
            validateEntry(entry);
        }

        // Resolve conditional references (e.g., Patient?identifier=MRN|12345)
        Map<String, String> referenceMap = resolveConditionalReferences(
            tenantId, transactionBundle);

        // Process in FHIR-specified order: DELETE, POST, PUT, GET
        Bundle responseBundle = new Bundle();
        responseBundle.setType(Bundle.BundleType.TRANSACTIONRESPONSE);

        List<BundleEntryComponent> sorted = sortByMethod(transactionBundle.getEntry());
        for (BundleEntryComponent entry : sorted) {
            BundleEntryResponseComponent response = processEntry(tenantId, entry, referenceMap);
            responseBundle.addEntry().setResponse(response);
        }

        return responseBundle;
    }
}
```

**Testing:**
- POST a transaction bundle with 3 resources (Patient, Encounter, Observation); all succeed atomically
- POST a transaction bundle with a deliberate error in entry 3; verify all 3 are rolled back
- POST a batch bundle with a deliberate error in entry 2; verify entries 1 and 3 succeed, entry 2 returns error
- Verify conditional references (e.g., `Patient?identifier=MRN|12345`) resolve correctly within the transaction
- Verify transaction processing order: DELETE before POST before PUT before GET

### Task 2.4: FHIR Subscriptions

**What:** Implement FHIR R4 Subscription resources with rest-hook and websocket channel types. Include the subscription delivery engine with retry logic and dead-letter handling.

**Design:**
```sql
-- V010__subscriptions.sql
CREATE TABLE fhir_subscription (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    fhir_id         VARCHAR(64) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'requested',
    resource        JSONB NOT NULL,
    criteria        VARCHAR(500) GENERATED ALWAYS AS (resource->>'criteria') STORED,
    channel_type    VARCHAR(30) GENERATED ALWAYS AS (resource#>>'{channel,type}') STORED,
    channel_endpoint VARCHAR(500) GENERATED ALWAYS AS (resource#>>'{channel,endpoint}') STORED,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, fhir_id)
);

CREATE TABLE subscription_delivery (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subscription_id UUID NOT NULL REFERENCES fhir_subscription(id),
    resource_type   VARCHAR(50) NOT NULL,
    resource_fhir_id VARCHAR(64) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    attempt_count   INTEGER NOT NULL DEFAULT 0,
    next_attempt_at TIMESTAMPTZ,
    error_message   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

```java
@Component
public class SubscriptionMatcher {

    public List<FhirSubscription> findMatchingSubscriptions(
            UUID tenantId, String resourceType, IBaseResource resource) {
        // Parse criteria string into search parameters
        // Evaluate resource against each active subscription's criteria
        // Return matching subscriptions for delivery
    }
}

@Component
public class SubscriptionDeliveryEngine {

    @Scheduled(fixedDelay = 1000)
    public void deliverPending() {
        List<SubscriptionDelivery> pending = deliveryRepo.findPendingDeliveries(100);
        for (SubscriptionDelivery delivery : pending) {
            try {
                deliverToEndpoint(delivery);
                delivery.setStatus("delivered");
            } catch (Exception e) {
                delivery.incrementAttempts();
                if (delivery.getAttemptCount() >= MAX_RETRIES) {
                    delivery.setStatus("failed");
                } else {
                    delivery.setNextAttemptAt(calculateBackoff(delivery));
                }
            }
            deliveryRepo.save(delivery);
        }
    }
}
```

**Testing:**
- Create a Subscription for `Observation?code=http://loinc.org|8867-4` with rest-hook channel
- POST an Observation with LOINC code 8867-4; verify the webhook is called within 5 seconds
- POST an Observation with a different code; verify no webhook is triggered
- Simulate webhook endpoint failure; verify retry with exponential backoff (1s, 2s, 4s)
- Verify delivery moves to `failed` status after max retries (3)
- Verify Subscription status can be toggled to `off` and deliveries stop

### Task 2.5: Terminology Services

**What:** Implement FHIR terminology operations: CodeSystem lookup, ValueSet $expand, $validate-code. Load LOINC, SNOMED CT (US Edition), and ICD-10-CM code systems.

**Design:**
```sql
-- V011__terminology.sql
CREATE TABLE terminology_code_system (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    url         VARCHAR(500) NOT NULL UNIQUE,
    name        VARCHAR(255) NOT NULL,
    version     VARCHAR(50),
    status      VARCHAR(20) NOT NULL DEFAULT 'active',
    content     VARCHAR(20) NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE terminology_concept (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code_system_id  UUID NOT NULL REFERENCES terminology_code_system(id),
    code            VARCHAR(100) NOT NULL,
    display         VARCHAR(500) NOT NULL,
    definition      TEXT,
    active          BOOLEAN NOT NULL DEFAULT true,
    properties      JSONB NOT NULL DEFAULT '{}',
    UNIQUE (code_system_id, code)
);

CREATE TABLE terminology_value_set (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    url         VARCHAR(500) NOT NULL UNIQUE,
    name        VARCHAR(255) NOT NULL,
    version     VARCHAR(50),
    status      VARCHAR(20) NOT NULL DEFAULT 'active',
    compose     JSONB NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

**Testing:**
- `GET /fhir/CodeSystem/$lookup?system=http://loinc.org&code=8867-4` returns "Heart rate"
- `POST /fhir/ValueSet/$expand` with a ValueSet containing LOINC vital signs returns all vital sign codes
- `POST /fhir/CodeSystem/$validate-code` returns true for valid SNOMED code, false for invalid
- Verify LOINC code system loads with 90,000+ concepts
- Verify $expand with filter parameter narrows results correctly

### Task 2.6: CapabilityStatement & Conformance

**What:** Generate a dynamic CapabilityStatement resource reflecting the server's supported resources, search parameters, operations, and security configuration.

**Design:**
```java
@RestController
public class MetadataController {

    @GetMapping("/fhir/metadata")
    public CapabilityStatement getCapabilityStatement() {
        CapabilityStatement cs = new CapabilityStatement();
        cs.setStatus(Enumerations.PublicationStatus.ACTIVE);
        cs.setFhirVersion(Enumerations.FHIRVersion._4_0_1);
        cs.setFormat(List.of(new CodeType("json"), new CodeType("xml")));

        // Dynamically enumerate supported resource types and search params
        CapabilityStatementRestComponent rest = cs.addRest();
        rest.setMode(RestfulCapabilityMode.SERVER);

        for (String resourceType : supportedResourceTypes) {
            CapabilityStatementRestResourceComponent resource = rest.addResource();
            resource.setType(resourceType);
            resource.addInteraction().setCode(TypeRestfulInteraction.READ);
            resource.addInteraction().setCode(TypeRestfulInteraction.CREATE);
            resource.addInteraction().setCode(TypeRestfulInteraction.UPDATE);
            resource.addInteraction().setCode(TypeRestfulInteraction.DELETE);
            resource.addInteraction().setCode(TypeRestfulInteraction.SEARCHTYPE);
            resource.addInteraction().setCode(TypeRestfulInteraction.HISTORYINSTANCE);

            for (SearchParamDef sp : searchParamRegistry.getParams(resourceType)) {
                resource.addSearchParam()
                    .setName(sp.getName())
                    .setType(sp.getType());
            }
        }

        return cs;
    }
}
```

**Testing:**
- `GET /fhir/metadata` returns a valid CapabilityStatement
- CapabilityStatement lists all 9 core resource types with correct interactions
- Each resource type lists its supported search parameters
- Security section advertises SMART on FHIR (once Phase 4 is complete)
- CapabilityStatement validates against the FHIR R4 CapabilityStatement profile

---

## Phase 3: HL7 v2 Message Pipeline

**Goal:** Accept HL7 v2 messages via MLLP and HTTP, parse them, apply configurable transformation rules to produce FHIR resources, route them to the FHIR server, and send ACK/NAK responses.

**Definition of Done:** The system can receive an ADT^A01 message via MLLP, transform it into a FHIR Patient + Encounter transaction bundle, store the resources, and return an ACK. Channel monitoring shows message counts and error rates.

### Task 3.1: MLLP Listener & HL7 v2 Parser

**What:** Implement a TCP/MLLP listener that accepts HL7 v2 messages, validates the MSH segment, parses the message into segments, and publishes a `HL7v2MessageReceived` event to Kafka.

**Design:**
```java
// MllpListener.java -- Netty-based MLLP server
@Component
public class MllpListener {

    private static final byte MLLP_START = 0x0B;
    private static final byte MLLP_END_1 = 0x1C;
    private static final byte MLLP_END_2 = 0x0D;

    @PostConstruct
    public void start() {
        ServerBootstrap bootstrap = new ServerBootstrap();
        bootstrap.group(bossGroup, workerGroup)
            .channel(NioServerSocketChannel.class)
            .childHandler(new ChannelInitializer<SocketChannel>() {
                @Override
                protected void initChannel(SocketChannel ch) {
                    ch.pipeline().addLast(
                        new MllpFrameDecoder(),
                        new Hl7v2MessageHandler(messageService)
                    );
                }
            });
        bootstrap.bind(port);
    }
}

// Hl7v2Parser.java -- parse pipe-delimited HL7 v2 into structured segments
public class Hl7v2Parser {
    public ParsedMessage parse(String rawMessage) {
        String[] segments = rawMessage.split("\r");
        MshSegment msh = parseMsh(segments[0]);
        Map<String, List<ParsedSegment>> segmentMap = new LinkedHashMap<>();
        for (String segment : segments) {
            String segType = segment.substring(0, 3);
            segmentMap.computeIfAbsent(segType, k -> new ArrayList<>())
                .add(parseSegment(segType, segment));
        }
        return new ParsedMessage(msh, segmentMap, rawMessage);
    }
}
```

**Testing:**
- Send an ADT^A01 message via MLLP to port 6661; verify ACK response received
- Send a malformed message (missing MSH); verify NAK response with error code AE
- Verify raw message is stored in `hl7v2_message` table with `processing_status = 'received'`
- Verify `parsed_segments` JSONB column contains correct PID, PV1, and other segments
- Send 100 messages concurrently; verify all are received without data loss
- Verify HTTP endpoint `/api/hl7v2/ingest` accepts messages via POST

### Task 3.2: Channel Configuration & Routing Engine

**What:** Implement the channel management system that defines source-destination pairs with transformation rules. Channels can be started, stopped, and configured via API.

**Design:**
```sql
-- V020__hl7v2_channels.sql
CREATE TABLE hl7v2_channel (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    source_config   JSONB NOT NULL,
    dest_config     JSONB NOT NULL,
    transform_rules JSONB NOT NULL DEFAULT '[]',
    status          VARCHAR(20) NOT NULL DEFAULT 'stopped',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

```java
@RestController
@RequestMapping("/api/channels")
public class ChannelController {

    @PostMapping
    public ResponseEntity<Channel> createChannel(@RequestBody ChannelCreateRequest req) {
        Channel channel = channelService.create(req);
        return ResponseEntity.status(HttpStatus.CREATED).body(channel);
    }

    @PutMapping("/{channelId}/start")
    public ResponseEntity<Channel> startChannel(@PathVariable UUID channelId) {
        Channel channel = channelService.start(channelId);
        return ResponseEntity.ok(channel);
    }

    @PutMapping("/{channelId}/stop")
    public ResponseEntity<Channel> stopChannel(@PathVariable UUID channelId) {
        Channel channel = channelService.stop(channelId);
        return ResponseEntity.ok(channel);
    }
}
```

**Testing:**
- Create a channel via POST /api/channels with MLLP source and FHIR destination
- Start the channel; verify MLLP listener opens on the configured port
- Stop the channel; verify MLLP port is closed and new connections are refused
- Update channel transform rules; verify new rules apply to subsequent messages
- Verify channel status is persisted across application restarts

### Task 3.3: HL7 v2 to FHIR Transformation Engine

**What:** Build the configurable transformation engine that converts HL7 v2 segments into FHIR resources using mapping rules. Support direct field mapping, date format conversion, code translation (via terminology), and script-based transforms.

**Design:**
```java
@Service
public class Hl7v2ToFhirTransformer {

    public Bundle transform(UUID tenantId, UUID channelId, ParsedMessage message) {
        Channel channel = channelService.getChannel(channelId);
        List<TransformRule> rules = channel.getTransformRules();

        Bundle bundle = new Bundle();
        bundle.setType(Bundle.BundleType.TRANSACTION);

        // Group rules by target resource type
        Map<String, List<TransformRule>> rulesByTarget = rules.stream()
            .collect(Collectors.groupingBy(TransformRule::getTargetResourceType));

        for (var entry : rulesByTarget.entrySet()) {
            IBaseResource resource = createResource(entry.getKey());
            for (TransformRule rule : entry.getValue()) {
                String sourceValue = message.getField(rule.getSourcePath());
                Object transformedValue = applyTransform(rule, sourceValue);
                setFhirField(resource, rule.getTargetFhirPath(), transformedValue);
            }
            bundle.addEntry()
                .setResource((Resource) resource)
                .getRequest()
                .setMethod(Bundle.HTTPVerb.POST)
                .setUrl(entry.getKey());
        }

        return bundle;
    }

    private Object applyTransform(TransformRule rule, String sourceValue) {
        return switch (rule.getTransformType()) {
            case "direct" -> sourceValue;
            case "date_format" -> convertDate(sourceValue, rule.getConfig());
            case "code_lookup" -> terminologyService.translate(sourceValue, rule.getConfig());
            case "script" -> scriptEngine.evaluate(rule.getConfig().get("script"), sourceValue);
            default -> throw new UnsupportedTransformException(rule.getTransformType());
        };
    }
}
```

**Testing:**
- Transform ADT^A01 with PID segment into a FHIR Patient resource; verify name, DOB, gender, address map correctly
- Transform ADT^A01 with PV1 segment into a FHIR Encounter resource; verify class, status, period map correctly
- Transform ORU^R01 with OBX segments into FHIR Observation resources; verify code, value, units map correctly
- Verify date format conversion: HL7 v2 `19850315` converts to FHIR `1985-03-15`
- Verify code translation: HL7 v2 gender `M` maps to FHIR `male` via code lookup rule
- Verify script-based transform: custom JavaScript extracts a value from a composite field
- Verify the resulting FHIR Bundle is a valid transaction bundle

### Task 3.4: Message Processing Pipeline & Error Handling

**What:** Implement the end-to-end message processing pipeline: receive, parse, transform, store FHIR resources, update message status, and send ACK/NAK. Include dead-letter queue for failed messages.

**Design:**
```java
@Service
public class MessagePipelineService {

    @KafkaListener(topics = "hl7v2.messages.received")
    public void processMessage(Hl7v2MessageEvent event) {
        HL7v2Message message = messageRepo.findById(event.getMessageId());
        try {
            message.setProcessingStatus("transforming");
            messageRepo.save(message);

            Bundle fhirBundle = transformer.transform(
                message.getTenantId(), message.getChannelId(), message.getParsedMessage());

            message.setProcessingStatus("storing");
            messageRepo.save(message);

            Bundle result = fhirBundleProcessor.processTransaction(
                message.getTenantId(), fhirBundle);

            message.setProcessingStatus("completed");
            message.setFhirBundleId(extractBundleId(result));
            message.setProcessedAt(Instant.now());
            messageRepo.save(message);

        } catch (TransformationException e) {
            message.setProcessingStatus("transform_error");
            message.setErrorMessage(e.getMessage());
            messageRepo.save(message);
            deadLetterService.send(message, e);

        } catch (FhirStorageException e) {
            message.setProcessingStatus("storage_error");
            message.setErrorMessage(e.getMessage());
            messageRepo.save(message);
            deadLetterService.send(message, e);
        }
    }
}
```

**Testing:**
- End-to-end: send ADT^A01 via MLLP; verify Patient and Encounter are created in FHIR server
- Verify message status transitions: received -> transforming -> storing -> completed
- Send a message with unmappable segment; verify status = `transform_error` and dead-letter entry created
- Reprocess a failed message from the dead-letter queue; verify it completes successfully after rule fix
- Verify 1,000 messages/minute throughput without message loss
- Verify ACK is sent only after FHIR resources are persisted (not before)

---

## Phase 4: Authentication, Authorization & Security

**Goal:** Implement SMART on FHIR OAuth2 authorization, RBAC enforcement on all FHIR and admin endpoints, TLS configuration, and HIPAA-compliant audit logging.

**Definition of Done:** A SMART on FHIR app can complete the EHR launch flow, receive scoped tokens, and access only authorized resources. All API calls produce audit events. TLS terminates at the application or load balancer.

### Task 4.1: Keycloak Integration & SMART on FHIR

**What:** Configure Keycloak as the OAuth2/OIDC authorization server with SMART on FHIR extensions. Implement the `.well-known/smart-configuration` discovery endpoint and SMART App Launch v2 flow.

**Design:**
```java
@RestController
public class SmartConfigController {

    @GetMapping("/fhir/.well-known/smart-configuration")
    public SmartConfiguration getSmartConfig() {
        return SmartConfiguration.builder()
            .issuer(keycloakIssuerUrl)
            .authorizationEndpoint(keycloakIssuerUrl + "/protocol/openid-connect/auth")
            .tokenEndpoint(keycloakIssuerUrl + "/protocol/openid-connect/token")
            .tokenEndpointAuthMethodsSupported(List.of(
                "client_secret_basic", "client_secret_post", "private_key_jwt"))
            .grantTypesSupported(List.of("authorization_code", "client_credentials"))
            .scopesSupported(List.of(
                "openid", "fhirUser", "launch", "launch/patient",
                "patient/*.read", "patient/*.write",
                "user/*.read", "user/*.write",
                "system/*.read", "system/*.write"))
            .responseTypesSupported(List.of("code"))
            .codeChallengeMethodsSupported(List.of("S256"))
            .capabilities(List.of(
                "launch-ehr", "launch-standalone",
                "client-public", "client-confidential-symmetric",
                "sso-openid-connect", "context-ehr-patient",
                "permission-patient", "permission-user"))
            .build();
    }
}
```

```sql
-- V030__oauth2.sql
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
```

**Testing:**
- `GET /fhir/.well-known/smart-configuration` returns valid SMART configuration JSON
- Complete a SMART EHR Launch flow with a test app; receive access token with patient context
- Complete a SMART Standalone Launch flow; receive access token with user-selected patient
- Verify PKCE is enforced for public clients (no client_secret)
- Verify access token contains correct SMART scopes (e.g., `patient/Observation.read`)
- Verify token expiration is enforced (request after expiry returns 401)

### Task 4.2: FHIR Scope Enforcement

**What:** Implement a Spring Security interceptor that validates SMART on FHIR scopes on every FHIR API request. Enforce patient-level access compartments and resource-type restrictions.

**Design:**
```java
@Component
public class SmartScopeInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response,
                             Object handler) {
        SmartToken token = extractSmartToken(request);

        String resourceType = extractResourceType(request);
        String action = mapHttpMethodToAction(request.getMethod());

        // Check resource-type scope
        if (!token.hasScope(resourceType, action)) {
            throw new ForbiddenException("Insufficient scope for " + resourceType + "." + action);
        }

        // Enforce patient compartment for patient-level scopes
        if (token.isPatientScope()) {
            String patientId = token.getPatientId();
            request.setAttribute("SMART_PATIENT_COMPARTMENT", patientId);
            // All queries will be filtered to this patient's compartment
        }

        return true;
    }
}
```

**Testing:**
- Token with `patient/Observation.read` can GET /fhir/Observation?patient=pat-123 (if pat-123 is the launch patient)
- Token with `patient/Observation.read` is rejected when reading a different patient's observations (403)
- Token with `user/Patient.read` can read any patient the user has access to
- Token with `system/*.read` can read all resources (no compartment restriction)
- Token without any scope matching the requested resource type returns 403
- Write request with read-only scope returns 403

### Task 4.3: HIPAA Audit Logging

**What:** Implement an append-only, partitioned audit_event table that captures every FHIR API access, authentication event, data export, and administrative action. Expose audit events as FHIR AuditEvent resources.

**Design:**
```sql
-- V031__audit.sql
CREATE TABLE audit_event (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    event_type      VARCHAR(50) NOT NULL,
    event_action    VARCHAR(10) NOT NULL,
    event_outcome   VARCHAR(10) NOT NULL,
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    agent_id        VARCHAR(255) NOT NULL,
    agent_type      VARCHAR(30) NOT NULL,
    agent_ip        INET,
    entity_type     VARCHAR(50),
    entity_fhir_id  VARCHAR(64),
    entity_query    TEXT,
    detail          JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (recorded_at);

-- Create monthly partitions
CREATE TABLE audit_event_2026_01 PARTITION OF audit_event
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
-- ... auto-create future partitions via scheduled job
```

```java
@Aspect
@Component
public class AuditAspect {

    @AfterReturning(pointcut = "@annotation(Audited)", returning = "result")
    public void auditSuccess(JoinPoint joinPoint, Object result) {
        AuditEventRecord record = buildAuditRecord(joinPoint, "0"); // success
        auditService.record(record);
    }

    @AfterThrowing(pointcut = "@annotation(Audited)", throwing = "ex")
    public void auditFailure(JoinPoint joinPoint, Exception ex) {
        AuditEventRecord record = buildAuditRecord(joinPoint, "8"); // serious failure
        record.setDetail(Map.of("error", ex.getMessage()));
        auditService.record(record);
    }
}
```

**Testing:**
- Every FHIR CRUD operation creates an audit_event record with correct event_type and entity info
- Failed authentication attempts are logged with event_outcome = '8' (serious failure)
- Audit events include agent IP address and user/system identity
- `GET /fhir/AuditEvent?date=ge2026-05-01&agent=user-123` returns audit events as FHIR resources
- Audit table partitions are created automatically for future months
- Verify audit records cannot be updated or deleted (append-only enforcement)

### Task 4.4: TLS & Data Encryption

**What:** Configure TLS termination, at-rest encryption for PHI columns, and secure secrets management.

**Design:**
```yaml
# application-production.yml
server:
  ssl:
    enabled: true
    key-store: ${SSL_KEYSTORE_PATH}
    key-store-password: ${SSL_KEYSTORE_PASSWORD}
    key-store-type: PKCS12

spring:
  datasource:
    url: jdbc:postgresql://${DB_HOST}:5432/healthinterop?sslmode=verify-full
    # pgcrypto extension for column-level encryption
```

```java
@Component
public class PhiEncryptionService {
    // Encrypt sensitive fields before storage
    // SSN, certain demographic fields, clinical notes
    // Uses AES-256-GCM with per-tenant key managed via AWS KMS / HashiCorp Vault
    public String encrypt(String plaintext, UUID tenantId) { ... }
    public String decrypt(String ciphertext, UUID tenantId) { ... }
}
```

**Testing:**
- Application rejects plain HTTP connections when TLS is enabled
- Database connection uses SSL (`sslmode=verify-full`)
- Sensitive fields (SSN hash, certain notes) are encrypted at rest in the database
- Encryption keys are fetched from Vault/KMS, not hardcoded in config
- Certificate rotation does not cause downtime

---

## Phase 5: Patient Identity & Master Patient Index

**Goal:** Implement deterministic and probabilistic patient matching, a manual merge/split workflow, and IHE PIXm/PDQm profile support.

**Definition of Done:** When a new patient arrives via HL7 v2 or FHIR, the MPI automatically checks for potential duplicates, proposes matches above a configurable threshold, and queues ambiguous matches for human review. Confirmed matches produce a golden record linking all source identifiers.

### Task 5.1: Deterministic Patient Matching

**What:** Implement rule-based matching on exact identifier matches (MRN, SSN hash, insurance member ID) across source systems. Automatically link patients with matching identifiers.

**Design:**
```java
@Service
public class DeterministicMatcher {

    public List<MatchResult> findExactMatches(UUID tenantId, PatientIdentity newPatient) {
        List<MatchResult> results = new ArrayList<>();

        for (PatientIdentifier id : newPatient.getIdentifiers()) {
            List<PatientIdentity> matches = identityRepo.findByIdentifier(
                tenantId, id.getSystem(), id.getValue());

            for (PatientIdentity existing : matches) {
                if (!existing.getId().equals(newPatient.getId())) {
                    results.add(MatchResult.builder()
                        .patientA(newPatient)
                        .patientB(existing)
                        .score(1.0)
                        .method("deterministic")
                        .status("confirmed")
                        .fieldScores(Map.of(id.getSystem(), 1.0))
                        .build());
                }
            }
        }

        return results;
    }
}
```

**Testing:**
- Two patients from different hospitals with the same MRN system and value are matched automatically
- Two patients with the same SSN hash are matched automatically
- Patients with different identifiers are NOT matched deterministically
- Matched patients share a golden_record_id
- The merge creates a golden Patient resource combining identifiers from both sources

### Task 5.2: Probabilistic Patient Matching

**What:** Implement weighted probabilistic matching using demographic field comparisons (name, DOB, gender, address, phone). Use Jaro-Winkler for name similarity, exact match for DOB, and configurable thresholds.

**Design:**
```java
@Service
public class ProbabilisticMatcher {

    private static final Map<String, FieldWeight> DEFAULT_WEIGHTS = Map.of(
        "family_name", new FieldWeight(0.30, new JaroWinklerComparator()),
        "given_name",  new FieldWeight(0.15, new JaroWinklerComparator()),
        "birth_date",  new FieldWeight(0.25, new ExactDateComparator()),
        "gender",      new FieldWeight(0.05, new ExactComparator()),
        "postal_code", new FieldWeight(0.10, new ExactComparator()),
        "phone",       new FieldWeight(0.10, new PhoneNormalizedComparator()),
        "ssn_last4",   new FieldWeight(0.05, new ExactComparator())
    );

    public List<MatchResult> findProbabilisticMatches(UUID tenantId,
            PatientIdentity newPatient, double threshold) {
        // Blocking pass: narrow candidates by DOB +/- 1 year and first 3 chars of family name
        List<PatientIdentity> candidates = identityRepo.findCandidates(
            tenantId, newPatient.getBirthDate(), newPatient.getFamilyNamePrefix());

        List<MatchResult> matches = new ArrayList<>();
        for (PatientIdentity candidate : candidates) {
            Map<String, Double> fieldScores = new LinkedHashMap<>();
            double totalScore = 0;

            for (var entry : DEFAULT_WEIGHTS.entrySet()) {
                double fieldScore = entry.getValue().comparator().compare(
                    newPatient.getField(entry.getKey()),
                    candidate.getField(entry.getKey()));
                fieldScores.put(entry.getKey(), fieldScore);
                totalScore += fieldScore * entry.getValue().weight();
            }

            if (totalScore >= threshold) {
                String status = totalScore >= autoLinkThreshold ? "confirmed" : "proposed";
                matches.add(MatchResult.builder()
                    .patientA(newPatient)
                    .patientB(candidate)
                    .score(totalScore)
                    .method("probabilistic")
                    .status(status)
                    .fieldScores(fieldScores)
                    .build());
            }
        }

        return matches;
    }
}
```

**Testing:**
- Patients with identical demographics score 1.0 and auto-link (status = confirmed)
- Patient "Jon Smith" vs "John Smith" with same DOB scores > 0.85 (proposed match)
- Patient "John Smith DOB 1985-03-15" vs "Jane Doe DOB 1990-01-01" scores < 0.3 (no match)
- Configurable threshold: lowering threshold from 0.8 to 0.6 produces more proposed matches
- Configurable auto-link threshold: score > 0.95 auto-confirms without review
- Blocking pass performance: matching against 100,000 patients completes in < 2 seconds

### Task 5.3: Manual Merge/Split Workflow

**What:** Build the API and workflow for human review of proposed matches. Reviewers can confirm, reject, or split previously merged patients.

**Design:**
```java
@RestController
@RequestMapping("/api/mpi")
public class MpiReviewController {

    @GetMapping("/review-queue")
    public Page<MatchResult> getReviewQueue(
            @RequestParam UUID tenantId,
            @RequestParam(defaultValue = "proposed") String status,
            Pageable pageable) {
        return matchResultRepo.findByTenantAndStatus(tenantId, status, pageable);
    }

    @PutMapping("/matches/{matchId}/confirm")
    @Audited(action = "patient_match_confirm")
    public MatchResult confirmMatch(@PathVariable UUID matchId,
                                     @AuthenticationPrincipal UserDetails user) {
        return mpiService.confirmMatch(matchId, user.getId());
    }

    @PutMapping("/matches/{matchId}/reject")
    @Audited(action = "patient_match_reject")
    public MatchResult rejectMatch(@PathVariable UUID matchId,
                                    @AuthenticationPrincipal UserDetails user) {
        return mpiService.rejectMatch(matchId, user.getId());
    }

    @PostMapping("/patients/{patientId}/split")
    @Audited(action = "patient_split")
    public PatientIdentity splitPatient(@PathVariable UUID patientId,
                                         @RequestBody SplitRequest request) {
        return mpiService.splitPatient(patientId, request);
    }
}
```

**Testing:**
- GET /api/mpi/review-queue returns proposed matches sorted by score descending
- Confirming a match merges two patient identities under one golden record
- Confirming a match updates all FHIR resources referencing patient B to reference patient A
- Rejecting a match marks the pair as rejected; they are not proposed again
- Splitting a previously merged patient creates a new golden record for the split-out identifiers
- All merge/split actions produce audit events with reviewer identity and timestamp

### Task 5.4: IHE PIXm & PDQm Profile Support

**What:** Implement the IHE PIXm (Patient Identifier Cross-referencing for Mobile) and PDQm (Patient Demographics Query for Mobile) profiles as FHIR operations.

**Design:**
```java
// PIXm: $ihe-pix operation
@Operation(name = "$ihe-pix", type = Patient.class)
public Parameters pixmLookup(
        @OperationParam(name = "sourceIdentifier") Identifier sourceId,
        @OperationParam(name = "targetSystem") List<UriType> targetSystems) {

    PatientIdentity identity = mpiService.findByIdentifier(
        sourceId.getSystem(), sourceId.getValue());

    Parameters result = new Parameters();
    for (PatientIdentifier crossRef : identity.getAllIdentifiers()) {
        if (targetSystems.isEmpty() ||
            targetSystems.stream().anyMatch(t -> t.getValue().equals(crossRef.getSystem()))) {
            result.addParameter()
                .setName("targetIdentifier")
                .setValue(new Identifier()
                    .setSystem(crossRef.getSystem())
                    .setValue(crossRef.getValue()));
        }
    }
    return result;
}

// PDQm: $match operation
@Operation(name = "$match", type = Patient.class)
public Bundle pdqmMatch(@OperationParam(name = "resource") Patient inputPatient,
                         @OperationParam(name = "onlyCertainMatches") BooleanType onlyCertain,
                         @OperationParam(name = "count") IntegerType count) {
    // Convert input Patient to PatientIdentity
    // Run probabilistic matching
    // Return Bundle of matching Patient resources with match scores in search.score
}
```

**Testing:**
- PIXm: query with source MRN returns cross-referenced identifiers from other systems
- PIXm: query with targetSystem filter returns only identifiers from that system
- PDQm $match: submit a Patient resource; receive a ranked list of matches with search.score
- PDQm $match with onlyCertainMatches=true returns only high-confidence matches
- Verify responses conform to IHE PIXm/PDQm response profiles

---

## Phase 6: Admin Dashboard & Operations

**Goal:** Deliver a web-based admin dashboard for channel monitoring, message inspection, patient matching review, and system health visualization.

**Definition of Done:** An operator can log in to the dashboard, see all channels with real-time message counts, drill into error queues, review proposed patient matches, and see system health metrics.

### Task 6.1: Dashboard Shell & Authentication

**What:** Scaffold the Next.js application with layout, navigation, Keycloak SSO integration, and role-based menu visibility.

**Design:**
```typescript
// dashboard/src/app/layout.tsx
import { KeycloakProvider } from '@/lib/auth/keycloak-provider';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <KeycloakProvider>
          <div className="flex h-screen">
            <Sidebar />
            <main className="flex-1 overflow-auto p-6">
              {children}
            </main>
          </div>
        </KeycloakProvider>
      </body>
    </html>
  );
}

// Sidebar navigation items:
// - Dashboard (home/overview)
// - Channels (HL7 v2 channel management)
// - Messages (message browser with filters)
// - Patients (MPI review queue)
// - FHIR Browser (resource explorer)
// - Compliance (compliance dashboard)
// - Settings (tenant configuration)
```

**Testing:**
- Unauthenticated users are redirected to Keycloak login
- After login, dashboard renders with correct user name and role
- Admin role sees all menu items; analyst role sees only Dashboard and FHIR Browser
- Session timeout redirects to login after configured inactivity period

### Task 6.2: Channel Monitoring Dashboard

**What:** Build the channel management and monitoring UI showing real-time message volumes, error rates, latency, and channel status with start/stop controls.

**Design:**
```typescript
// dashboard/src/features/channels/ChannelList.tsx
interface ChannelStats {
  channelId: string;
  name: string;
  status: 'running' | 'stopped' | 'error';
  messagesToday: number;
  messagesHour: number;
  errorsToday: number;
  avgLatencyMs: number;
  lastMessageAt: string;
}

export function ChannelList() {
  const { data: channels } = useQuery({
    queryKey: ['channels'],
    queryFn: () => api.get<ChannelStats[]>('/api/channels/stats'),
    refetchInterval: 5000, // Poll every 5 seconds
  });

  return (
    <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
      {channels?.map(channel => (
        <ChannelCard key={channel.channelId} channel={channel} />
      ))}
    </div>
  );
}
```

**Testing:**
- Channel cards show real-time message counts that update every 5 seconds
- Clicking a channel opens the detail view with message volume chart (last 24 hours)
- Start/stop buttons update channel status immediately
- Error count is highlighted in red when > 0
- Channel detail shows last 50 messages with status indicators

### Task 6.3: Message Browser & Error Queue

**What:** Build a searchable message browser with filters for status, message type, date range, and channel. Include a reprocessing action for failed messages.

**Design:**
```typescript
// dashboard/src/features/messages/MessageBrowser.tsx
interface MessageFilter {
  channelId?: string;
  status?: string;
  messageType?: string;
  dateFrom?: string;
  dateTo?: string;
  search?: string;
}

export function MessageBrowser() {
  const [filters, setFilters] = useState<MessageFilter>({});
  const { data, isLoading } = useQuery({
    queryKey: ['messages', filters],
    queryFn: () => api.get('/api/messages', { params: filters }),
  });

  // Table with columns: Control ID, Type, Channel, Status, Received, Actions
  // Actions: View raw HL7, View FHIR Bundle, Reprocess (for failed messages)
}
```

**Testing:**
- Message browser loads and displays messages with pagination
- Filtering by status = 'failed' shows only failed messages
- Filtering by message type = 'ADT' shows only ADT messages
- Clicking "View raw HL7" shows the original pipe-delimited message with syntax highlighting
- Clicking "Reprocess" on a failed message re-submits it through the pipeline
- After reprocessing, message status updates to 'completed'

### Task 6.4: Patient Matching Review UI

**What:** Build the MPI review queue interface where operators can review proposed patient matches, compare demographics side-by-side, and confirm or reject matches.

**Design:**
```typescript
// dashboard/src/features/patients/MatchReview.tsx
export function MatchReview() {
  const { data: queue } = useQuery({
    queryKey: ['mpi-review-queue'],
    queryFn: () => api.get('/api/mpi/review-queue'),
  });

  return (
    <div>
      <h2>Patient Match Review Queue ({queue?.totalElements} pending)</h2>
      {queue?.content.map(match => (
        <MatchCard key={match.id} match={match}>
          {/* Side-by-side comparison of Patient A and Patient B */}
          {/* Field-level match scores with color coding */}
          {/* Confirm / Reject buttons */}
        </MatchCard>
      ))}
    </div>
  );
}
```

**Testing:**
- Review queue shows proposed matches sorted by score descending
- Side-by-side comparison highlights matching fields in green, differences in amber
- Confirming a match removes it from the queue and merges the patients
- Rejecting a match removes it from the queue and prevents re-proposal
- Empty queue shows a "no pending reviews" message
- Match score is displayed as a percentage with field-level breakdown

---

## Phase 7: X12 EDI & Payer Workflows

**Goal:** Add X12 EDI transaction processing for 270/271 (eligibility), 837 (claims), 835 (remittance), and 278 (prior authorization). Implement FHIR-based prior authorization aligned with Da Vinci PAS.

**Definition of Done:** The system can receive an X12 270 eligibility inquiry, parse it, route it to a payer endpoint, receive the 271 response, and expose the result as FHIR CoverageEligibilityRequest/Response resources.

### Task 7.1: X12 EDI Parser

**What:** Implement an X12 005010 parser that handles ISA/GS/ST envelope extraction, segment parsing, and element-level field extraction for 270, 271, 837P, 835, and 278 transaction sets.

**Design:**
```java
@Service
public class X12Parser {

    public ParsedX12Transaction parse(String rawEdi) {
        X12Envelope envelope = parseEnvelope(rawEdi); // ISA, GS, ST segments
        String transactionSet = envelope.getTransactionSetIdentifier(); // 270, 837, etc.

        List<X12Segment> segments = new ArrayList<>();
        for (String line : rawEdi.split("~")) {
            String segId = line.substring(0, Math.min(3, line.length())).trim();
            String[] elements = line.split("\\*");
            segments.add(new X12Segment(segId, elements));
        }

        return ParsedX12Transaction.builder()
            .envelope(envelope)
            .transactionSet(transactionSet)
            .segments(segments)
            .rawEdi(rawEdi)
            .build();
    }
}
```

**Testing:**
- Parse a valid X12 270 eligibility inquiry; verify ISA, GS, ST segments extracted correctly
- Parse a valid X12 837P professional claim; verify CLM, SV1, and DTP segments extracted
- Parse a 271 eligibility response; verify EB (eligibility/benefit) segments extracted
- Invalid EDI (missing ISA) returns a descriptive parse error
- Verify segment element count matches expected for each transaction set

### Task 7.2: X12-to-FHIR Mapping

**What:** Implement bidirectional mapping between X12 transaction sets and their FHIR resource equivalents (270 to CoverageEligibilityRequest, 271 to CoverageEligibilityResponse, 837 to Claim, 835 to ClaimResponse, 278 to Claim with prior-auth use).

**Design:**
```java
@Service
public class X12FhirMapper {

    public CoverageEligibilityRequest map270ToFhir(ParsedX12Transaction x12) {
        CoverageEligibilityRequest req = new CoverageEligibilityRequest();
        req.setStatus(CoverageEligibilityRequest.EligibilityRequestStatus.ACTIVE);

        // Map subscriber from 2010BA loop
        X12Loop subscriberLoop = x12.getLoop("2010BA");
        Reference patientRef = resolveOrCreatePatient(subscriberLoop);
        req.setPatient(patientRef);

        // Map payer from 2010BB loop
        X12Loop payerLoop = x12.getLoop("2010BB");
        Reference insurerRef = resolveOrCreateOrganization(payerLoop);
        req.setInsurer(insurerRef);

        // Map service type from SV1/SV2
        req.addItem().addCategory().addCoding()
            .setSystem("https://x12.org/codes/service-type-codes")
            .setCode(x12.getElement("SV1", 1));

        return req;
    }
}
```

**Testing:**
- X12 270 maps to a valid CoverageEligibilityRequest with correct patient, insurer, and service type
- X12 837P maps to a valid FHIR Claim with diagnosis codes, procedure codes, and charges
- X12 835 maps to a valid ClaimResponse with payment and adjustment amounts
- FHIR Claim round-trips back to X12 837 format for outbound payer submission
- Mapping preserves all clinically relevant fields without data loss

### Task 7.3: Da Vinci Prior Authorization (PAS) Support

**What:** Implement FHIR-based prior authorization following the Da Vinci PAS v2.1.0 implementation guide, aligned with CMS-0057-F requirements.

**Design:**
```java
@Operation(name = "$submit", type = Claim.class)
public ClaimResponse submitPriorAuth(
        @OperationParam(name = "claim") Claim priorAuthClaim,
        @OperationParam(name = "organization") Organization provider) {

    // Validate against Da Vinci PAS profile
    ValidationResult validation = profileValidator.validate(
        priorAuthClaim, "http://hl7.org/fhir/us/davinci-pas/StructureDefinition/profile-claim");

    if (!validation.isSuccessful()) {
        throw new UnprocessableEntityException(validation.getMessages());
    }

    // Convert FHIR Claim to X12 278 for payer backend
    String x12_278 = x12FhirMapper.mapClaimToX12_278(priorAuthClaim);

    // Route to payer endpoint
    String x12Response = payerConnector.submit(x12_278, provider);

    // Convert X12 278 response back to FHIR ClaimResponse
    ClaimResponse response = x12FhirMapper.map278ResponseToFhir(x12Response);

    // Store both FHIR resources
    fhirResourceService.create(priorAuthClaim);
    fhirResourceService.create(response);

    return response;
}
```

**Testing:**
- $submit operation accepts a valid Da Vinci PAS Claim and returns a ClaimResponse
- Invalid Claim (missing required PAS profile elements) returns OperationOutcome with errors
- Prior auth request is correctly converted to X12 278 format for payer submission
- Payer response (approved/denied/pended) is correctly reflected in ClaimResponse
- Both the request and response are stored as FHIR resources with audit trail

---

## Phase 8: AI-Native Features

**Goal:** Deliver the four AI-native capabilities that differentiate this platform: LLM-assisted HL7 v2-to-FHIR mapping, ML-based probabilistic patient matching, data quality anomaly detection, and natural language FHIR query.

**Definition of Done:** An operator can upload sample HL7 v2 messages and receive auto-generated transformation rules. The ML patient matcher outperforms the weighted probabilistic matcher on a benchmark dataset. Anomaly detection flags data quality issues in real-time message flows.

### Task 8.1: LLM-Assisted HL7 v2-to-FHIR Mapping

**What:** Build a Python FastAPI service that accepts sample HL7 v2 messages and generates transformation rule configurations using an LLM (Claude API). Generated rules are stored in the same format as manually created rules.

**Design:**
```python
# ai-services/mapping-engine/main.py
from fastapi import FastAPI
from anthropic import Anthropic

app = FastAPI()
client = Anthropic()

@app.post("/api/ai/generate-mapping")
async def generate_mapping(request: MappingRequest):
    """Accept sample HL7 v2 messages and generate transformation rules."""

    prompt = f"""Analyze these sample HL7 v2 messages and generate transformation rules
    that map each field to its corresponding FHIR R4 resource path.

    Sample messages:
    {request.sample_messages}

    Target FHIR resource types: {request.target_resource_types}

    Return a JSON array of transformation rules in this format:
    [
      {{
        "source_segment": "PID",
        "source_field": "PID.5",
        "target_fhir_path": "Patient.name.family",
        "transform_type": "direct",
        "transform_config": {{}},
        "confidence": 0.95
      }}
    ]
    """

    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=4096,
        messages=[{"role": "user", "content": prompt}]
    )

    rules = parse_rules(response.content[0].text)

    # Validate rules against FHIR path definitions
    validated_rules = [r for r in rules if validate_fhir_path(r["target_fhir_path"])]

    return {"rules": validated_rules, "confidence_avg": avg_confidence(validated_rules)}
```

**Testing:**
- Upload 5 sample ADT^A01 messages; receive transformation rules for Patient and Encounter resources
- Generated rules correctly map PID.5 to Patient.name.family, PID.7 to Patient.birthDate, etc.
- Rules include confidence scores; low-confidence rules are flagged for human review
- Generated rules are in the same JSON format as manually created channel rules
- Applying generated rules to a new ADT^A01 produces a valid FHIR transaction bundle
- Rules for non-standard/custom Z-segments are generated with lower confidence scores

### Task 8.2: ML-Based Patient Matching Model

**What:** Train a supervised ML model for patient matching that outperforms the weighted probabilistic matcher (Task 5.2). Use features derived from demographic field comparisons. Deploy as a Python FastAPI service.

**Design:**
```python
# ai-services/patient-matching/model.py
import numpy as np
from sklearn.ensemble import GradientBoostingClassifier
from jellyfish import jaro_winkler_similarity

class PatientMatchModel:
    def __init__(self):
        self.model = GradientBoostingClassifier(
            n_estimators=200,
            max_depth=5,
            learning_rate=0.1
        )

    def extract_features(self, patient_a: dict, patient_b: dict) -> np.ndarray:
        return np.array([
            jaro_winkler_similarity(
                patient_a.get("family_name", ""),
                patient_b.get("family_name", "")),
            jaro_winkler_similarity(
                patient_a.get("given_name", ""),
                patient_b.get("given_name", "")),
            1.0 if patient_a.get("birth_date") == patient_b.get("birth_date") else 0.0,
            1.0 if patient_a.get("gender") == patient_b.get("gender") else 0.0,
            jaro_winkler_similarity(
                patient_a.get("address_line", ""),
                patient_b.get("address_line", "")),
            1.0 if patient_a.get("postal_code") == patient_b.get("postal_code") else 0.0,
            self._phone_similarity(patient_a.get("phone"), patient_b.get("phone")),
            self._ssn_last4_match(patient_a.get("ssn_last4"), patient_b.get("ssn_last4")),
        ])

    def predict(self, patient_a: dict, patient_b: dict) -> dict:
        features = self.extract_features(patient_a, patient_b).reshape(1, -1)
        probability = self.model.predict_proba(features)[0][1]
        return {
            "match_score": float(probability),
            "match_method": "ml",
            "feature_importances": dict(zip(self.feature_names, features[0]))
        }
```

**Testing:**
- Train on labeled match/non-match dataset (minimum 10,000 pairs); achieve > 95% precision and > 90% recall
- ML model outperforms weighted probabilistic matcher on F1 score by > 5 percentage points
- Predictions complete in < 50ms per pair
- Model handles missing fields gracefully (e.g., no SSN) without crashing
- Feature importances are returned for explainability
- Model can be retrained on new labeled data without service downtime (model versioning)

### Task 8.3: Real-Time Data Quality Anomaly Detection

**What:** Build a streaming anomaly detector that monitors message flows for missing segments, unexpected value distributions, volume anomalies, and data quality drift. Publishes alerts to the dashboard.

**Design:**
```python
# ai-services/anomaly-detection/detector.py
from collections import defaultdict
from datetime import datetime, timedelta

class AnomalyDetector:
    def __init__(self):
        self.baseline_stats = {}  # Per-channel baseline statistics
        self.alert_buffer = []

    async def analyze_message(self, message: dict):
        channel_id = message["channel_id"]
        alerts = []

        # Check for missing expected segments
        expected_segments = self.get_expected_segments(message["message_type"])
        actual_segments = set(message["parsed_segments"].keys())
        missing = expected_segments - actual_segments
        if missing:
            alerts.append(Alert(
                type="missing_segment",
                severity="warning",
                message=f"Missing expected segments: {missing}",
                channel_id=channel_id
            ))

        # Check for value distribution anomalies
        for field_path, value in self.extract_monitored_fields(message):
            if self.is_anomalous(channel_id, field_path, value):
                alerts.append(Alert(
                    type="value_anomaly",
                    severity="info",
                    message=f"Unusual value for {field_path}: {value}",
                    channel_id=channel_id
                ))

        # Check for volume anomalies (sudden drops or spikes)
        if self.is_volume_anomaly(channel_id):
            alerts.append(Alert(
                type="volume_anomaly",
                severity="warning",
                message=f"Message volume anomaly detected for channel {channel_id}",
                channel_id=channel_id
            ))

        return alerts
```

**Testing:**
- Detector flags missing PID segment in an ADT^A01 message as a warning
- Detector flags a gender value of "X" when the baseline only contains "M" and "F"
- Detector flags a sudden 80% drop in message volume compared to the 24-hour baseline
- Detector flags a sudden 300% spike in message volume
- Alerts appear in the dashboard within 10 seconds of detection
- False positive rate is < 5% on a 7-day production-like message stream

### Task 8.4: Natural Language FHIR Query Interface

**What:** Build an NL-to-FHIR-search translator that converts plain English questions into FHIR search API calls.

**Design:**
```python
# ai-services/nl-query/main.py
@app.post("/api/ai/nl-query")
async def natural_language_query(request: NLQueryRequest):
    """Translate a natural language question into a FHIR search query."""

    prompt = f"""Convert this natural language question into a FHIR R4 search API call.

    Question: {request.question}

    Available resource types and their search parameters:
    {capability_statement_summary}

    Return JSON:
    {{
      "resource_type": "...",
      "search_params": {{"param": "value", ...}},
      "fhir_url": "...",
      "explanation": "..."
    }}
    """

    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=1024,
        messages=[{"role": "user", "content": prompt}]
    )

    query = parse_query(response.content[0].text)

    # Execute the FHIR search
    results = fhir_client.search(query["resource_type"], query["search_params"])

    return {
        "query": query,
        "results": results,
        "result_count": len(results)
    }
```

**Testing:**
- "Show me all patients named Smith born in 1985" translates to `GET /fhir/Patient?name=Smith&birthdate=1985`
- "Find lab results for patient John Doe" translates to `GET /fhir/Observation?subject=Patient/[id]&category=laboratory`
- "How many active encounters are there today?" translates to an appropriate Encounter search
- Invalid questions return a helpful error message rather than a malformed query
- Query explanation is human-readable and describes what the search will return

---

## Phase 9: Bulk Data & Analytics

**Goal:** Implement FHIR Bulk Data Export ($export) for population-level data extraction, and provide analytics-ready data export pipelines.

**Definition of Done:** A system-level $export operation produces NDJSON files for all patient data, streaming to cloud storage (S3/GCS/Azure Blob). Export status is tracked and resumable.

### Task 9.1: FHIR Bulk $export Implementation

**What:** Implement the FHIR Bulk Data Access specification for system-level, patient-level, and group-level $export operations.

**Design:**
```java
@RestController
public class BulkExportController {

    @GetMapping("/fhir/$export")
    public ResponseEntity<Void> initiateSystemExport(
            @RequestParam(name = "_type", required = false) List<String> types,
            @RequestParam(name = "_since", required = false) String since,
            @RequestParam(name = "_outputFormat", defaultValue = "application/fhir+ndjson")
                String outputFormat) {

        UUID jobId = bulkExportService.initiateExport(
            ExportRequest.builder()
                .level(ExportLevel.SYSTEM)
                .resourceTypes(types)
                .since(since != null ? Instant.parse(since) : null)
                .outputFormat(outputFormat)
                .build()
        );

        return ResponseEntity.accepted()
            .header("Content-Location", "/fhir/$export-poll/" + jobId)
            .build();
    }

    @GetMapping("/fhir/$export-poll/{jobId}")
    public ResponseEntity<?> pollExportStatus(@PathVariable UUID jobId) {
        ExportJob job = bulkExportService.getJob(jobId);

        if (job.getStatus() == ExportStatus.IN_PROGRESS) {
            return ResponseEntity.accepted()
                .header("X-Progress", job.getProgressDescription())
                .header("Retry-After", "10")
                .build();
        }

        // Complete -- return manifest
        return ResponseEntity.ok(job.getManifest());
    }
}
```

**Testing:**
- `GET /fhir/$export` returns 202 Accepted with Content-Location header
- Polling the Content-Location returns 202 with X-Progress until complete
- Completed export returns a manifest with NDJSON file URLs
- Downloading a manifest URL returns valid NDJSON with one FHIR resource per line
- `_type=Patient,Observation` exports only those resource types
- `_since=2026-01-01` exports only resources updated after that date
- Export with 100,000+ resources completes within 5 minutes

### Task 9.2: Analytics Export Pipeline

**What:** Build a scheduled pipeline that exports FHIR data in analytics-ready formats (Parquet, CSV) to cloud object storage for data warehouse ingestion.

**Design:**
```java
@Service
public class AnalyticsExportService {

    @Scheduled(cron = "0 0 2 * * *") // Daily at 2 AM
    public void runDailyExport() {
        for (Tenant tenant : tenantRepo.findAllActive()) {
            ExportManifest manifest = bulkExportService.exportToStorage(
                tenant.getId(),
                ExportConfig.builder()
                    .format("parquet")
                    .destination(tenant.getConfig().getAnalyticsStoragePath())
                    .partitionBy("resource_type", "date")
                    .build()
            );
            analyticsNotificationService.notify(tenant, manifest);
        }
    }
}
```

**Testing:**
- Daily export runs at configured time and produces Parquet files per resource type
- Parquet files are partitioned by date for efficient query access
- BigQuery/Athena can read the exported Parquet files directly
- Export includes a manifest file listing all produced files with record counts
- Failed exports produce alerts and can be retried manually

---

## Phase 10: Compliance & TEFCA

**Goal:** Implement continuous compliance monitoring against USCDI, ONC Information Blocking Rule, and CMS-0057-F requirements. Add TEFCA/QHIN connectivity readiness.

**Definition of Done:** The compliance dashboard shows pass/fail status for each regulatory requirement. UDAP dynamic client registration is functional. The system can participate in a TEFCA-style directed exchange.

### Task 10.1: USCDI Compliance Checker

**What:** Implement automated checks that verify the FHIR server exposes all USCDI v3 data elements with correct US Core profiles.

**Design:**
```java
@Service
public class UscdiComplianceChecker {

    private static final List<UscdiElement> USCDI_V3_ELEMENTS = List.of(
        new UscdiElement("Patient Demographics", "Patient",
            "http://hl7.org/fhir/us/core/StructureDefinition/us-core-patient",
            List.of("name", "birthDate", "gender", "race", "ethnicity", "address")),
        new UscdiElement("Allergies and Intolerances", "AllergyIntolerance",
            "http://hl7.org/fhir/us/core/StructureDefinition/us-core-allergyintolerance",
            List.of("code", "patient", "clinicalStatus")),
        // ... all 50+ USCDI v3 data classes and elements
    );

    public ComplianceReport checkCompliance(UUID tenantId) {
        List<ComplianceResult> results = new ArrayList<>();
        for (UscdiElement element : USCDI_V3_ELEMENTS) {
            boolean profileSupported = capabilityStatement.supportsProfile(element.profileUrl());
            boolean searchParamsAvailable = element.requiredSearchParams().stream()
                .allMatch(p -> capabilityStatement.supportsSearchParam(
                    element.resourceType(), p));

            results.add(new ComplianceResult(
                element, profileSupported && searchParamsAvailable,
                profileSupported, searchParamsAvailable));
        }
        return new ComplianceReport("USCDI_V3", results);
    }
}
```

**Testing:**
- Compliance check returns pass for all USCDI v3 data elements when server is fully configured
- Missing US Core profile support is flagged as a failure with remediation guidance
- Missing search parameter support is flagged separately from missing profile support
- Compliance report is exportable as JSON and PDF
- Historical compliance results are stored for trend analysis

### Task 10.2: UDAP Dynamic Client Registration

**What:** Implement UDAP (Unified Data Access Profiles) dynamic client registration for B2B FHIR exchange, as required by TEFCA from January 2026.

**Design:**
```java
@RestController
public class UdapRegistrationController {

    @PostMapping("/oauth2/register")
    public ResponseEntity<UdapRegistrationResponse> registerClient(
            @RequestBody String signedSoftwareStatement) {

        // 1. Parse the signed JWT software statement
        JWSObject jws = JWSObject.parse(signedSoftwareStatement);

        // 2. Validate the X.509 certificate chain against trusted CA bundle
        X509Certificate cert = extractCertificate(jws);
        certValidator.validateChain(cert, trustedCaBundle);

        // 3. Extract client metadata from the software statement
        UdapClientMetadata metadata = extractMetadata(jws.getPayload());

        // 4. Register the OAuth2 client
        OAuth2Client client = oauth2ClientService.register(
            metadata.getClientName(),
            metadata.getRedirectUris(),
            metadata.getGrantTypes(),
            metadata.getScopes(),
            cert
        );

        return ResponseEntity.created(URI.create("/oauth2/register/" + client.getClientId()))
            .body(new UdapRegistrationResponse(client));
    }
}
```

**Testing:**
- Valid UDAP software statement with trusted certificate chain registers a new client
- Invalid certificate chain returns 401 Unauthorized
- Expired certificate returns 401 with descriptive error
- Registered client can obtain access tokens via client_credentials grant
- Client metadata (scopes, grant types) is enforced on token requests

### Task 10.3: Compliance Dashboard

**What:** Build the compliance monitoring UI showing real-time status for USCDI, ONC Information Blocking, CMS-0057-F, and HIPAA requirements.

**Design:**
```typescript
// dashboard/src/features/compliance/ComplianceDashboard.tsx
export function ComplianceDashboard() {
  const { data: reports } = useQuery({
    queryKey: ['compliance'],
    queryFn: () => api.get('/api/compliance/reports'),
  });

  return (
    <div>
      <ComplianceScoreCard regulation="USCDI v3" report={reports?.uscdi} />
      <ComplianceScoreCard regulation="ONC Info Blocking" report={reports?.onc} />
      <ComplianceScoreCard regulation="CMS-0057-F" report={reports?.cms0057f} />
      <ComplianceScoreCard regulation="HIPAA Security" report={reports?.hipaa} />
      <ComplianceTrendChart history={reports?.history} />
    </div>
  );
}
```

**Testing:**
- Dashboard shows overall compliance percentage for each regulation
- Clicking a regulation card shows the detailed pass/fail list
- Failed checks include remediation guidance
- Trend chart shows compliance score over the last 90 days
- Compliance reports can be exported as PDF for auditors

---

## Phase 11: Production Hardening

**Goal:** Prepare the platform for production deployment with performance optimization, high availability, disaster recovery, monitoring, and security hardening.

**Definition of Done:** The system handles 10,000 FHIR operations/minute and 5,000 HL7 v2 messages/minute with P99 latency under 500ms. Automated failover completes within 30 seconds. A full security penetration test produces zero critical findings.

### Task 11.1: Performance Optimization

**What:** Optimize database queries, implement connection pooling (HikariCP), add Redis caching for hot paths (CapabilityStatement, terminology lookups, active tokens), and tune JVM settings.

**Design:**
```yaml
# application-production.yml
spring:
  datasource:
    hikari:
      maximum-pool-size: 50
      minimum-idle: 10
      connection-timeout: 5000
      idle-timeout: 300000

  cache:
    type: redis
    redis:
      time-to-live: 300000 # 5 minutes

# Redis caching for hot paths
# - CapabilityStatement: cached indefinitely, invalidated on config change
# - Terminology lookups: cached 1 hour
# - Active OAuth2 tokens: cached until expiry
# - Channel configuration: cached 5 minutes
```

**Testing:**
- Benchmark: 10,000 FHIR Patient searches/minute with P50 < 50ms and P99 < 200ms
- Benchmark: 5,000 HL7 v2 ADT messages/minute end-to-end with P99 < 500ms
- Redis cache hit rate > 90% for terminology lookups after warm-up
- Database connection pool utilization stays below 80% under peak load
- No memory leaks over a 72-hour sustained load test

### Task 11.2: High Availability & Disaster Recovery

**What:** Configure PostgreSQL streaming replication, Kafka multi-broker cluster, application multi-instance deployment with health-check-based load balancing, and automated backup/restore.

**Design:**
```yaml
# deploy/helm/values-production.yaml
replicaCount: 3
autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

postgresql:
  architecture: replication
  readReplicas:
    replicaCount: 2
  backup:
    enabled: true
    cronjob:
      schedule: "0 */6 * * *"  # Every 6 hours
    storage:
      storageClass: gp3
      size: 100Gi

kafka:
  replicaCount: 3
  defaultReplicationFactor: 3
  minInsyncReplicas: 2
```

**Testing:**
- Kill one application instance; verify requests are served by remaining instances with no errors
- Kill the PostgreSQL primary; verify automatic failover to replica within 30 seconds
- Kill one Kafka broker; verify message processing continues without loss
- Restore from backup; verify all data is intact and the system resumes processing
- Verify zero-downtime rolling deployments with Kubernetes rolling update strategy

### Task 11.3: Observability & Alerting

**What:** Instrument the application with structured logging (JSON), distributed tracing (OpenTelemetry), metrics (Prometheus), and configure alerting rules.

**Design:**
```java
// OpenTelemetry instrumentation
@Bean
public OpenTelemetry openTelemetry() {
    return AutoConfiguredOpenTelemetrySdk.builder()
        .addResourceCustomizer((resource, config) ->
            resource.merge(Resource.builder()
                .put("service.name", "health-interop-core")
                .put("service.version", buildVersion)
                .build()))
        .build()
        .getOpenTelemetrySdk();
}

// Custom metrics
@Component
public class InteropMetrics {
    private final Counter messagesProcessed;
    private final Counter messagesErrored;
    private final Histogram messageLatency;
    private final Gauge activeChannels;

    public InteropMetrics(MeterRegistry registry) {
        this.messagesProcessed = Counter.builder("interop.messages.processed")
            .tag("type", "hl7v2")
            .register(registry);
        this.messageLatency = Timer.builder("interop.messages.latency")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(registry);
    }
}
```

**Testing:**
- Prometheus scrape endpoint returns metrics for message counts, latencies, error rates
- Grafana dashboard shows real-time message throughput and latency charts
- Distributed trace spans cover the full message pipeline (MLLP receive -> FHIR store)
- Alert fires when error rate exceeds 5% over a 5-minute window
- Alert fires when P99 latency exceeds 1 second
- Log aggregation (Loki/ELK) captures all structured logs with correlation IDs

### Task 11.4: Security Hardening & Penetration Testing

**What:** Conduct dependency vulnerability scanning (Dependabot/Snyk), container image scanning (Trivy), and commission a third-party penetration test. Remediate all critical and high findings.

**Testing:**
- Dependency scan shows zero critical vulnerabilities
- Container image scan shows zero critical vulnerabilities
- Penetration test produces zero critical findings
- OWASP Top 10 checklist is complete (SQL injection, XSS, CSRF, etc.)
- PHI fields cannot be accessed without valid SMART token
- Rate limiting prevents brute-force token attacks (429 after 10 failed attempts)

---

## Phase 12: FHIR R5 & Extensibility

**Goal:** Add FHIR R5 support (topic-based subscriptions, enhanced search), DICOM metadata service, and plugin architecture for custom extensions.

**Definition of Done:** The server advertises FHIR R5 capability alongside R4 via content negotiation. Topic-based subscriptions are functional. A plugin can register a custom operation without modifying core code.

### Task 12.1: FHIR R5 Resource Support

**What:** Add FHIR R5 resource handling alongside R4, using content negotiation (Accept header) to select the version. Implement R5 cross-version analysis support.

**Design:**
```java
@Configuration
public class FhirVersionConfig {

    @Bean("fhirContextR4")
    public FhirContext fhirContextR4() {
        return FhirContext.forR4();
    }

    @Bean("fhirContextR5")
    public FhirContext fhirContextR5() {
        return FhirContext.forR5();
    }

    @Bean
    public FhirVersionNegotiator versionNegotiator() {
        return new FhirVersionNegotiator(
            Map.of(
                "application/fhir+json; fhirVersion=4.0", fhirContextR4(),
                "application/fhir+json; fhirVersion=5.0", fhirContextR5()
            )
        );
    }
}
```

**Testing:**
- Request with `Accept: application/fhir+json; fhirVersion=4.0` returns R4 resources
- Request with `Accept: application/fhir+json; fhirVersion=5.0` returns R5 resources
- Default (no fhirVersion) returns R4 for backward compatibility
- R5 CapabilityStatement reflects R5-specific features
- Resources stored as R4 can be projected to R5 format on read

### Task 12.2: R5 Topic-Based Subscriptions

**What:** Implement the FHIR R5 topic-based subscription framework (SubscriptionTopic, Subscription, SubscriptionStatus) with support for WebSocket, REST hook, and Kafka delivery channels.

**Design:**
```java
@Service
public class TopicSubscriptionService {

    public void registerTopic(SubscriptionTopic topic) {
        // Parse topic resource triggers and filters
        // Register event listeners for matching resource changes
        topicRegistry.register(topic);
    }

    public void evaluateTrigger(String resourceType, IBaseResource resource,
                                 TriggerType triggerType) {
        List<SubscriptionTopic> matchingTopics = topicRegistry.findMatching(
            resourceType, triggerType);

        for (SubscriptionTopic topic : matchingTopics) {
            List<Subscription> subscribers = subscriptionRepo.findByTopic(topic.getUrl());
            for (Subscription sub : subscribers) {
                deliveryService.deliver(sub, resource, topic);
            }
        }
    }
}
```

**Testing:**
- Create a SubscriptionTopic for "new lab results" (Observation create with category=laboratory)
- Create a Subscription referencing that topic with rest-hook channel
- POST a laboratory Observation; verify webhook is triggered
- POST a vital-signs Observation; verify webhook is NOT triggered (wrong category)
- SubscriptionStatus resource reflects delivery statistics
- WebSocket channel delivers real-time notifications to connected clients

### Task 12.3: Plugin Architecture

**What:** Implement a plugin system that allows custom FHIR operations, interceptors, and resource validators to be loaded at runtime from JAR files or configuration.

**Design:**
```java
public interface HealthInteropPlugin {
    String getName();
    String getVersion();

    default List<FhirOperation> getOperations() { return List.of(); }
    default List<FhirInterceptor> getInterceptors() { return List.of(); }
    default List<ResourceValidator> getValidators() { return List.of(); }
}

@Service
public class PluginManager {
    private final List<HealthInteropPlugin> plugins = new ArrayList<>();

    @PostConstruct
    public void loadPlugins() {
        // Scan classpath and configured plugin directories
        ServiceLoader<HealthInteropPlugin> loader =
            ServiceLoader.load(HealthInteropPlugin.class);
        for (HealthInteropPlugin plugin : loader) {
            registerPlugin(plugin);
        }
    }

    private void registerPlugin(HealthInteropPlugin plugin) {
        plugins.add(plugin);
        plugin.getOperations().forEach(operationRegistry::register);
        plugin.getInterceptors().forEach(interceptorRegistry::register);
        plugin.getValidators().forEach(validatorRegistry::register);
    }
}
```

**Testing:**
- A sample plugin JAR defining a custom $my-operation is loaded at startup
- `POST /fhir/Patient/$my-operation` invokes the plugin's operation handler
- Plugin interceptor can modify request/response headers
- Plugin validator can reject resources that don't conform to a custom profile
- Disabling a plugin via configuration removes its operations from the CapabilityStatement
- Plugin errors are isolated and do not crash the core server

### Task 12.4: DICOM Metadata Service

**What:** Add a lightweight DICOM metadata service that stores imaging study/series/instance metadata as FHIR ImagingStudy resources, with DICOMweb QIDO-RS query support.

**Design:**
```java
@RestController
@RequestMapping("/dicomweb")
public class DicomWebController {

    @GetMapping("/studies")
    public ResponseEntity<List<JsonObject>> queryStudies(
            @RequestParam Map<String, String> params) {
        // QIDO-RS: Query for DICOM studies
        // Map DICOM query parameters to FHIR ImagingStudy search
        // Return DICOM JSON response format
    }

    @PostMapping("/studies")
    public ResponseEntity<Void> storeStudy(@RequestBody byte[] dicomData) {
        // STOW-RS: Store DICOM metadata (not pixel data)
        // Extract study/series/instance metadata
        // Create FHIR ImagingStudy resource
    }
}
```

**Testing:**
- STOW-RS accepts DICOM metadata and creates a FHIR ImagingStudy resource
- QIDO-RS queries return matching studies in DICOM JSON format
- ImagingStudy resources are searchable via standard FHIR search parameters
- Study-level, series-level, and instance-level queries are supported
- Integration with a PACS viewer (e.g., OHIF) displays study metadata correctly

---

## Summary

| Phase | Title | Key Deliverables | Est. Duration |
|-------|-------|-----------------|---------------|
| 1 | Foundation | Repo, CI/CD, DB schema, Docker dev env | 2-3 weeks |
| 2 | FHIR Server Core | CRUD, search, transactions, subscriptions, terminology | 6-8 weeks |
| 3 | HL7 v2 Pipeline | MLLP listener, parser, transformer, routing | 4-5 weeks |
| 4 | Auth & Security | SMART on FHIR, RBAC, audit logging, TLS | 4-5 weeks |
| 5 | Patient Identity | Deterministic + probabilistic matching, merge/split, PIXm/PDQm | 4-6 weeks |
| 6 | Dashboard | Channel monitoring, message browser, MPI review UI | 4-5 weeks |
| 7 | X12 EDI | X12 parser, FHIR mapping, Da Vinci PAS | 4-6 weeks |
| 8 | AI-Native Features | LLM mapping, ML matching, anomaly detection, NL query | 6-8 weeks |
| 9 | Bulk Data & Analytics | $export, analytics pipelines | 3-4 weeks |
| 10 | Compliance & TEFCA | USCDI checker, UDAP, compliance dashboard | 4-5 weeks |
| 11 | Production Hardening | Performance, HA/DR, observability, security testing | 4-6 weeks |
| 12 | FHIR R5 & Extensibility | R5 support, topic subscriptions, plugins, DICOM | 6-8 weeks |
| **Total** | | | **~51-69 weeks** |

---

## Appendix: Definition of Done (Global)

Every task across all phases must satisfy these criteria before it is considered complete:

1. **Code:** Implementation merged to `main` via reviewed pull request
2. **Tests:** Unit tests (>80% coverage for new code), integration tests for database and API interactions
3. **Documentation:** API endpoints documented in OpenAPI/Swagger; configuration parameters documented in README
4. **Security:** No hardcoded secrets; PHI access requires authentication; audit events emitted
5. **Observability:** Structured log entries for key operations; metrics exported to Prometheus
6. **Migration:** Database changes delivered as versioned Flyway migrations; rollback script included
7. **CI Green:** All CI checks pass (build, test, lint, security scan)
