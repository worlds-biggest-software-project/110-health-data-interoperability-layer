# Standards & API Reference

> Project: Health Data Interoperability Layer · Generated: 2026-05-06

## Industry Standards & Specifications

### ISO Standards

**ISO 13606 — Electronic Health Record Communication**
- URL: https://www.iso.org/standard/67868.html (Part 1: Reference Model, 2019)
- A five-part standard specifying the information models and vocabularies required for EHR interoperability. Uses a dual-model approach (Reference Model + Archetype Object Model) to define how EHR data is structured and exchanged between systems. Less prevalent in the US than HL7 FHIR, but underpins openEHR and several European national EHR frameworks. Relevant as a reference model for any platform targeting international markets.

**ISO 27799 — Health Informatics Information Security Management**
- URL: https://www.iso.org/standard/62777.html
- Provides guidance on the application of ISO/IEC 27001 to the health sector. Specifies security controls for processing, storing, and transmitting personally identifiable health information. Applicable to any HIPAA/GDPR-adjacent security design for a health data platform.

**ISO/TS 21547 — Electronic Health Record Retention**
- URL: https://www.iso.org/standard/41871.html
- Technical specification for the retention of EHRs. Relevant when designing audit-log durability, data lifecycle policies, and record expiry mechanics in a FHIR repository.

---

### W3C & IETF Standards

**RFC 6749 — The OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- Foundation for all delegated authorization in health APIs. SMART on FHIR, UDAP, and TEFCA FHIR security all build on top of OAuth 2.0 authorization code and client credentials flows.

**RFC 7519 — JSON Web Token (JWT)**
- URL: https://datatracker.ietf.org/doc/html/rfc7519
- JWT is the token format used for SMART on FHIR access tokens, UDAP client assertions, and FHIR Subscription payloads. Any health interoperability platform must validate and issue JWTs for API security.

**RFC 7591 — OAuth 2.0 Dynamic Client Registration Protocol**
- URL: https://datatracker.ietf.org/doc/html/rfc7591
- Defines how clients (EHR apps, third-party integrations) register dynamically with an authorization server without manual pre-registration. Required by UDAP for scalable B2B FHIR exchange across organizational boundaries.

**RFC 8288 — Web Linking**
- URL: https://datatracker.ietf.org/doc/html/rfc8288
- Defines the `Link` header used in FHIR paging (`rel=next`, `rel=prev`) for paginated search result sets. Required for correct FHIR client implementation of bundle traversal.

**W3C SPARQL (informational)**
- URL: https://www.w3.org/TR/sparql11-query/
- Not natively required by FHIR R4, but relevant if the platform exposes RDF-based graph queries over clinical terminologies (SNOMED CT, LOINC). Emerging in FHIR R5 as an optional query mechanism.

---

### Health-Specific Data Exchange Standards

**HL7 FHIR R4 (v4.0.1) — Fast Healthcare Interoperability Resources**
- URL: https://hl7.org/fhir/R4/
- The primary modern standard for clinical data exchange. Defines a RESTful API, 140+ resource types (Patient, Observation, Condition, MedicationRequest, etc.), subscription framework, bulk data export, and terminology services. R4 is the mandatory baseline for ONC-certified EHR APIs and CMS-mandated payer APIs through at least 2027. R5 (published 2023) adds improved subscriptions, cross-version analysis, and enhanced search, but production adoption lags R4.

**HL7 FHIR R5 (v5.0.0)**
- URL: https://hl7.org/fhir/R5/
- Released March 2023. Key improvements include a rewritten subscription/notification framework (topic-based subscriptions replace STU3-era channel subscriptions), enhanced bulk data, cross-version analysis support, and expanded clinical resources. Expected production adoption 2026–2028 as EHRs certify to the next ONC certification cycle.

**HL7 v2.x — Health Level Seven Version 2 Messaging**
- URL: https://www.hl7.org/implement/standards/product_brief.cfm?product_id=185
- Legacy pipe-delimited message format for ADT (admissions/discharges/transfers), ORM (orders), ORU (results), MDM (documents), and other transaction types. Still active in >80% of US hospital interface engine traffic via MLLP (Minimal Lower Layer Protocol, RFC 2575). Any production interoperability layer must ingest, parse, and translate HL7 v2 messages.

**HL7 CDA / C-CDA (Consolidated CDA)**
- URL: https://www.hl7.org/implement/standards/product_brief.cfm?product_id=492
- XML-based clinical document architecture used for care summaries (Continuity of Care Document), discharge summaries, and referral notes. C-CDA documents are mandated for transitions of care under ONC certification. Must be parsed and converted to FHIR DocumentReference or Composition resources.

**SMART on FHIR App Launch Framework v2.2 (HL7 Implementation Guide)**
- URL: https://hl7.org/fhir/smart-app-launch/
- OAuth2-based authorization framework enabling third-party apps to launch within EHR context (EHR launch) or standalone. Defines token scopes (clinical, system, user), token introspection, and the `.well-known/smart-configuration` discovery document. Mandatory for ONC-certified EHR APIs and CMS patient access APIs.

**HL7 FAST UDAP Security for Scalable Registration, Authentication, and Authorization (STU 2)**
- URL: https://hl7.org/fhir/us/udap-security/
- Extends OAuth 2.0 using Unified Data Access Profiles (UDAP) to enable dynamic client registration and JWT-based B2B authentication without manual credential exchange. Mandated by TEFCA for FHIR exchange among QHINs from January 1, 2026.

**US Core FHIR Implementation Guide (STU 6.1 / v9.0.0 build)**
- URL: https://hl7.org/fhir/us/core/
- Defines minimum constraints on FHIR R4 resources to satisfy USCDI data requirements for ONC-certified EHR APIs. Covers profiles for Patient, Observation, Condition, Medication, AllergyIntolerance, DiagnosticReport, and 30+ other resource types. USCDI v3 is the mandatory baseline as of January 1, 2026 under the HTI-1 Final Rule.

**HL7 Da Vinci Prior Authorization Support (PAS) v2.1.0**
- URL: https://hl7.org/fhir/us/davinci-pas/
- FHIR implementation guide for the electronic submission of prior authorization requests and decisions, directly required by CMS-0057-F for payer Prior Authorization APIs going live January 1, 2027. Uses X12 278 as the payer back-end but presents FHIR as the provider-facing API.

**HL7 Da Vinci Payer Data Exchange (PDex) v2.2**
- URL: https://build.fhir.org/ig/HL7/davinci-epdx/
- Defines how payers expose member clinical data via FHIR APIs, including payer-to-payer data exchange required by CMS-0057-F Payer-to-Payer API (deadline January 1, 2027).

**HL7 Da Vinci Coverage Requirements Discovery (CRD) & Documentation Templates and Rules (DTR)**
- URL (CRD): https://build.fhir.org/ig/HL7/davinci-crd/
- URL (DTR): https://build.fhir.org/ig/HL7/davinci-dtr/
- CRD uses CDS Hooks to surface coverage and prior auth requirements in real time within the EHR workflow. DTR retrieves payer-specific documentation questionnaires using SMART embedded apps. Both are part of the Da Vinci burden-reduction suite referenced by CMS-0057-F.

**IHE ITI Mobile access to Health Documents (MHD) v4.2**
- URL: https://profiles.ihe.net/ITI/MHD/
- Lightweight FHIR-based profile for sharing and accessing clinical documents, replacing XDS (Cross-Enterprise Document Sharing) in mobile/REST contexts. Defines document submission sets, lists, and query operations using FHIR DocumentReference and List resources.

**IHE PDQm — Patient Demographics Query for Mobile**
- URL: https://profiles.ihe.net/ITI/PDQm/
- FHIR-based profile for querying patient demographics across systems; supports `$match` for probabilistic patient matching. Required when implementing cross-facility patient identity resolution.

**IHE PIXm — Patient Identifier Cross-referencing for Mobile**
- URL: https://profiles.ihe.net/ITI/PIXm/
- FHIR-based profile for registering patient identifiers from source systems and cross-referencing them to a master patient index. Essential for any platform building a Master Patient Index (MPI).

**DICOM PS3.18 — DICOMweb Services (WADO-RS, STOW-RS, QIDO-RS)**
- URL: https://www.dicomstandard.org/dicomweb
- Defines RESTful web services for querying (QIDO-RS), retrieving (WADO-RS), and storing (STOW-RS) DICOM medical imaging objects. Relevant when the interoperability layer must handle imaging metadata alongside clinical FHIR data.

**NCPDP SCRIPT Standard v2017071**
- URL: https://www.ncpdp.org/Standards-Development/Standards-Information
- EDI standard for electronic prescribing (NewRx, CancelRx, RxFill, MedicationHistory, and ePA). Mandatory for ONC-certified EHR systems. Coexists with FHIR MedicationRequest — a platform may need to translate between the two for Surescripts connectivity.

**ANSI ASC X12 Healthcare Transaction Sets (005010)**
- URL: https://x12.org/products/transaction-sets
- Defines administrative transactions: 837P/I/D (claims), 835 (remittance advice), 270/271 (eligibility), 276/277 (claim status), 278 (prior authorization). Still dominant for payer/provider billing. Any platform targeting payer connectivity must support X12 translation to/from FHIR.

---

### Regulatory & Compliance Frameworks

**ONC Trusted Exchange Framework and Common Agreement (TEFCA) — QTF v2.0**
- URL: https://healthit.gov/policy/tefca/
- National framework governing exchange among Qualified Health Information Networks (QHINs: CommonWell, eHealth Exchange, Carequality, KONZA, KLAS, Health Gorilla). TEFCA FHIR exchange pilots ran in 2025; FHIR APIs are mandatory for QHIN participants by January 1, 2026, with UDAP FAST security required.

**ONC HTI-1 Final Rule (USCDI v3 and US Core Alignment)**
- URL: https://www.healthit.gov/topic/laws-regulation-and-policy/health-data-technology-and-interoperability-hti-1
- Mandates USCDI v3 as the minimum data set for ONC-certified EHR systems effective January 1, 2026. Aligns with US Core 6.x FHIR profiles.

**CMS Interoperability and Prior Authorization Final Rule (CMS-0057-F)**
- URL: https://www.cms.gov/newsroom/fact-sheets/cms-interoperability-prior-authorization-final-rule-cms-0057-f
- Requires Medicare Advantage plans, Medicaid, CHIP, and QHP issuers on FFE to implement: Patient Access API, Provider Access API, Payer-to-Payer API, and Prior Authorization API — all FHIR R4-based. Operational deadlines begin January 1, 2026; API deadlines begin January 1, 2027.

**HIPAA Privacy and Security Rules (45 CFR Parts 160 and 164)**
- URL: https://www.hhs.gov/hipaa/for-professionals/security/index.html
- Federal regulations governing electronic protected health information (ePHI): access controls, audit logging, encryption, breach notification, and Business Associate Agreements (BAAs). Applies to all systems processing PHI. HHS published a proposed rule in January 2025 to strengthen cybersecurity requirements.

**NIST SP 800-66 Rev. 2 — Implementing HIPAA Security Rule**
- URL: https://csrc.nist.gov/pubs/sp/800/66/r2/final
- Provides a cybersecurity resource guide for implementing HIPAA Security Rule requirements using NIST controls. Crosswalk between HIPAA safeguards and NIST Cybersecurity Framework 2.0.

---

### Security & Authentication Standards

**OAuth 2.0 (RFC 6749) + PKCE (RFC 7636)**
- URL: https://datatracker.ietf.org/doc/html/rfc7636
- PKCE (Proof Key for Code Exchange) is mandatory for public clients (mobile and single-page apps) under SMART App Launch v2 to prevent authorization code interception.

**OpenID Connect 1.0 (OIDC)**
- URL: https://openid.net/connect/
- Identity layer on top of OAuth 2.0 used for SMART on FHIR user authentication and EHR launch context. Provides ID tokens with patient or encounter context.

---

## Similar Products — Developer Documentation & APIs

### HAPI FHIR Server (Open Source)

- **Description:** The reference open-source Java implementation of the full HL7 FHIR standard (R4 and R5). Used as the foundation of Azure Health Data Services and Smile CDR.
- **API Documentation:** https://hapifhir.io/hapi-fhir/docs/
- **SDKs/Libraries:** Java (primary); Python client via fhirclient; .NET via Hl7.Fhir.R4 NuGet
- **Developer Guide (Getting Started):** https://hapifhir.io/hapi-fhir/docs/server_jpa/get_started.html
- **Starter Project (Docker + JPA):** https://github.com/hapifhir/hapi-fhir-jpaserver-starter
- **GitHub:** https://github.com/hapifhir/hapi-fhir
- **Standards:** HL7 FHIR R4/R5; SMART on FHIR via plug-in (Keycloak); OpenAPI/Swagger auto-generation; Bulk FHIR $export
- **Authentication:** Pluggable — delegates to external IdP (Keycloak, Okta) via interceptor

---

### Medplum (Open Source)

- **Description:** Modern open-source FHIR R4 platform targeting digital health application developers. Includes FHIR server, auth server, bot framework for automation, and React component library.
- **API Documentation:** https://www.medplum.com/docs/api
- **SDKs/Libraries:** TypeScript/JavaScript SDK (`@medplum/core`); React UI library (`@medplum/react`)
- **Developer Guide:** https://www.medplum.com/docs
- **GitHub:** https://github.com/medplum/medplum
- **Standards:** FHIR R4 (v4.0.1); SMART on FHIR v2; GraphQL over FHIR; REST/JSON
- **Authentication:** OAuth 2.0 with PKCE; SMART App Launch; client credentials for B2B

---

### Epic on FHIR (EHR Integration)

- **Description:** Epic's public FHIR API program exposing patient clinical data from Epic EHR installations; covers 250+ million US patient records.
- **API Documentation:** https://fhir.epic.com/Documentation
- **Open.Epic Developer Portal:** https://open.epic.com/
- **Sandbox Environment:** Available via open.epic.com; requires app registration; uses synthetic patient data
- **SDK/Libraries:** None official; community wrappers available in Python and JS
- **Developer Guide:** https://fhir.epic.com/Documentation?docId=oauth2
- **Standards:** HL7 FHIR R4 (US Core profiles); SMART on FHIR v2; CDS Hooks; Bulk FHIR $export
- **Authentication:** OAuth 2.0 authorization code (patient/practitioner launch); Backend Services (JWT client assertion)

---

### Oracle Health (Cerner) Millennium FHIR API

- **Description:** Oracle Health's FHIR R4 API (Ignite) for the Cerner Millennium EHR platform, covering ~40% of US hospital beds.
- **API Documentation:** https://fhir.cerner.com/
- **Developer Portal:** https://code.cerner.com/
- **Millennium Platform Docs:** https://docs.oracle.com/en/industries/health/millennium-platform-apis/
- **GitHub (API Docs Source):** https://github.com/cerner/fhir.cerner.com
- **Standards:** HL7 FHIR R4; SMART on FHIR v1 and v2; Bulk FHIR $export; US Core profiles
- **Authentication:** OAuth 2.0 with SMART launch; SMART Backend Services (JWT); API key for some endpoints
- **Notes:** DSTU-2 APIs deprecated as of December 2025; all new integrations must use R4.

---

### Azure Health Data Services — FHIR Service

- **Description:** Microsoft's managed FHIR R4 service (HIPAA-eligible) with SMART on FHIR, bulk export, subscriptions, and DICOM service co-location.
- **API Documentation:** https://learn.microsoft.com/en-us/azure/healthcare-apis/fhir/
- **REST API Reference:** https://learn.microsoft.com/en-us/rest/api/healthcareapis/
- **GitHub (OSS FHIR Server):** https://github.com/microsoft/fhir-server
- **SDKs:** Azure SDK for .NET, Python, Java, JavaScript; FHIR-specific Python SDK via azure-health-deidentification
- **Developer Guide:** https://learn.microsoft.com/en-us/azure/healthcare-apis/fhir/overview
- **Standards:** HL7 FHIR R4; SMART on FHIR v2; DICOMweb (WADO-RS, STOW-RS, QIDO-RS); Bulk $export
- **Authentication:** Azure Active Directory (Entra ID) OAuth 2.0; SMART on FHIR scopes
- **Notes:** Azure API for FHIR (legacy) retires September 30, 2026; new deployments must use Azure Health Data Services FHIR service.

---

### AWS HealthLake

- **Description:** AWS's FHIR R4-compliant managed data store with built-in Amazon Comprehend Medical NLP for clinical entity extraction from unstructured notes.
- **API Documentation:** https://docs.aws.amazon.com/healthlake/
- **FHIR Resource Management:** https://docs.aws.amazon.com/healthlake/latest/devguide/managing-fhir-resources.html
- **SDKs:** AWS SDK for Python (boto3), Java, Go, .NET; FHIR REST operations accessed via HTTP (not native SDK)
- **Developer Guide:** https://docs.aws.amazon.com/healthlake/latest/devguide/
- **Standards:** HL7 FHIR R4; SMART on FHIR; Bulk FHIR $export; AWS Signature Version 4
- **Authentication:** AWS IAM (SigV4) or SMART on FHIR OAuth 2.0 (on SMART-enabled stores)

---

### Google Cloud Healthcare API

- **Description:** Google Cloud's managed FHIR/HL7/DICOM API with native BigQuery streaming and Vertex AI/MedPaLM integration for clinical analytics and generative AI.
- **API Documentation:** https://docs.cloud.google.com/healthcare-api/docs
- **FHIR Concepts:** https://docs.cloud.google.com/healthcare-api/docs/concepts/fhir
- **REST API Reference:** https://docs.cloud.google.com/healthcare-api/docs/reference/rest/
- **SDKs:** Google Cloud client libraries (Python, Java, Go, Node.js); gcloud CLI
- **Developer Guide:** https://docs.cloud.google.com/healthcare-api/docs/introduction
- **Standards:** HL7 FHIR R4, STU3, DSTU2; HL7 v2 (native ingest via Pub/Sub); DICOMweb; SMART on FHIR
- **Authentication:** Google IAM OAuth 2.0; SMART on FHIR (limited)

---

### Redox

- **Description:** Commercial API-first interoperability platform that normalises HL7 v2, CDA, X12, DICOM, and proprietary formats into a unified FHIR JSON API. Pre-built connectors to 800+ healthcare organisations.
- **API Documentation:** https://docs.redoxengine.com/
- **FHIR API Guide:** https://docs.redoxengine.com/basics/redox-fhir-api/
- **Data Model API:** https://docs.redoxengine.com/basics/redox-data-model-api/
- **API Reference:** https://docs.redoxengine.com/api-reference/
- **GitHub:** https://github.com/RedoxEngine/
- **Standards:** FHIR R4 (normalised output); HL7 v2.x (inbound); X12 EDI (partial); proprietary Redox Data Model
- **Authentication:** OAuth 2.0; API key for legacy endpoints

---

### Smile CDR (Smile Digital Health)

- **Description:** Commercial enterprise FHIR R4/R5 server with built-in SMART on FHIR auth server, probabilistic MPI, CDS Hooks, and subscriptions. Built on HAPI FHIR core.
- **API Documentation:** https://smilecdr.com/docs/
- **FHIR Gateway Docs:** https://smilecdr.com/docs/fhir_gateway/introduction.html
- **SMART on FHIR Docs:** https://smilecdr.com/docs/smart/smart_on_fhir_introduction.html
- **Standards:** FHIR R4/R5/DSTU2; SMART on FHIR v1/v2; CDS Hooks; Bulk $export; UDAP; HL7 v2 (listener module)
- **Authentication:** Built-in OAuth 2.0/OIDC authorization server; SMART App Launch; UDAP dynamic registration; client credentials

---

### Availity & Optum Clearinghouse APIs (X12 EDI)

- **Description:** Availity is the largest US healthcare clearinghouse (connected to 2,000+ payers); Optum (formerly Change Healthcare) is the second-largest. Both expose REST APIs wrapping X12 EDI transactions for eligibility, claims, and remittance.
- **Availity Developer Portal:** https://developer.availity.com/
- **Availity API Guide:** https://developer.availity.com/blog/2025/3/25/availity-api-guide
- **Optum Developer Portal:** https://developer.optum.com/
- **Optum Eligibility API Docs:** https://developer.optum.com/eligibilityandclaims/docs/
- **Standards:** X12 005010 (270/271, 837, 835, 276/277); REST/JSON wrappers; SMART on FHIR scopes (Availity)
- **Authentication:** OAuth 2.0; API key; partner credentialing required for production access

---

## Notes

**Emerging / Rapidly Evolving Areas:**

- **FHIR Subscriptions v2 (R5 topic-based):** Replaces the polling-heavy R4 subscription model with a topic/channel approach supporting WebSocket, REST hook, and message queue delivery. Platforms should design subscription infrastructure to accommodate R5 backport where possible.
- **TEFCA FHIR exchange (January 2026 deadline):** QHINs must have UDAP FAST security in place; implementations targeting TEFCA connectivity must support dynamic client registration and fine-grained SMART scopes alongside existing XCA/XDS document exchange.
- **USCDI versioning cadence:** ONC published USCDI v6 in July 2025 and Draft v7 in January 2026. US Core IG versions track USCDI annually; platforms should plan for regular profile updates to maintain ONC certification alignment.
- **CMS-0057-F API compliance wave (2027):** The Prior Authorization API, Provider Access API, and Payer-to-Payer API deadlines hit January 1, 2027, driving a major demand cycle for FHIR-based payer integrations using Da Vinci PAS, PDex, and CRD implementation guides.
- **AI-augmented mapping (no formal standard yet):** LLM-assisted HL7 v2-to-FHIR transformation is an active area of vendor investment (NextGen v4.6+, Redox roadmap) but no industry standard or implementation guide governs it; this remains an open opportunity for an AI-native interoperability layer.
