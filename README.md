# Health Data Interoperability Layer

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source layer for FHIR/HL7 data normalisation, routing, and patient matching across fragmented healthcare systems.

The Health Data Interoperability Layer is a healthcare integration platform that ingests HL7 v2, FHIR, CDA, and X12 messages, normalises them into a coherent FHIR R4 representation, and routes them across providers, payers, and analytics destinations. It is built for hospital IT architects, payer CIOs, and digital health teams who need EHR connectivity without locking into a single cloud or paying six-figure enterprise contracts. The project closes the open-source gap left after Mirth Connect went fully commercial in v4.6.

---

## Why Health Data Interoperability Layer?

- Mirth Connect — long the default open-source interface engine — transitioned to a commercial-only licence at v4.6, leaving organisations on MPL 2.0 v4.5 or earlier with a forced commercial migration or fork decision.
- Enterprise interface engines (Cloverleaf, Rhapsody, Edifecs, InterSystems HealthShare) typically cost $300K–$2M per year and are inaccessible to smaller providers, startups, and research groups.
- Cloud-managed FHIR services (Azure Health Data Services, AWS HealthLake, Google Cloud Healthcare API) lock customers into a single cloud and have unpredictable consumption-based pricing at scale.
- HAPI FHIR is permissively licensed (Apache 2.0) but ships without HL7 v2 translation, an admin UI, or a built-in Master Patient Index — operators must assemble these themselves.
- Probabilistic patient matching, AI-assisted HL7-to-FHIR mapping, and continuous compliance auditing remain absent or nascent across every reviewed product.

---

## Key Features

### FHIR & HL7 Core

- HL7 FHIR R4 RESTful server with CRUD, search, history, transactions, and batch bundles
- HL7 v2.x ingest via MLLP and HTTP with configurable routing and transformation
- Subscription framework for push notifications on resource changes
- Bulk FHIR `$export` for population-level analytics handoff
- SMART on FHIR OAuth2 authorisation server for third-party app launches

### Patient Identity & Matching

- Deterministic patient matching with manual merge/split workflow as the MVP baseline
- Probabilistic Master Patient Index with configurable match score thresholds
- IHE PIXm and PDQm profile support for cross-system patient identity resolution
- Manual reconciliation queues for ambiguous matches with audit trail

### Routing, Transformation & Operations

- Multi-transport connectors (TCP/MLLP, HTTP/S, SFTP, database, REST/SOAP)
- Channel-based routing with error queues, dead-letter handling, and message reprocessing
- Operational dashboard covering channel health, message volumes, and error rates
- Role-based access control, TLS in transit, encryption at rest, and HIPAA-grade audit logging

### Administrative & Payer Workflows

- X12 EDI handling for 270/271 eligibility, 837 claims, 835 remittance, and 278 prior authorisation
- FHIR-based prior authorisation aligned with CMS-0057-F Da Vinci implementation guides
- Compliance monitoring against USCDI versions and the ONC Information Blocking Rule
- TEFCA/QHIN connectivity module for nationwide health information exchange

### Extensibility

- Pluggable persistence (PostgreSQL, MySQL, Oracle) via a JPA layer
- Terminology services with `CodeSystem`, `ValueSet`, `$validate-code`, and `$expand`
- DICOM service for medical imaging metadata alongside FHIR clinical data
- FHIR R5 support ahead of its expected production adoption wave

---

## AI-Native Advantage

Unlike incumbent interface engines whose mapping logic is hand-written and whose patient matching is rule-based, this project treats AI as a first-class component. LLM-assisted transformation rule inference can generate HL7 v2-to-FHIR mappings from sample messages, eliminating weeks of manual interface development per integration. Probabilistic patient matching trained on demographic variations, typos, and aliases reduces both false-match and false-no-match rates without manual configuration. A FHIR-native natural-language query layer exposes clinical data to non-technical analysts, while continuous AI-driven anomaly detection and compliance auditing flag data-quality drift and regulatory gaps in real time.

---

## Tech Stack & Deployment

The platform is designed around HL7 FHIR R4 (with R5 on the roadmap), HL7 v2.x, CDA/C-CDA, X12 EDI, IHE PIXm/PDQm, and SMART on FHIR. Deployment modes include self-hosted on-premises, customer-managed cloud (Kubernetes or container runtimes), and a hosted SaaS tier. Integration is via FHIR REST APIs, MLLP listeners for HL7 v2, X12 connectors for payer transactions, and webhook delivery for downstream consumers. The architecture aligns with TEFCA/QHIN connectivity and HIPAA Privacy and Security Rule requirements.

---

## Market Context

The global healthcare interoperability market was valued at approximately $3.4 billion in 2024 and is projected to grow at roughly 13% CAGR through 2030, with the broader health IT integration segment exceeding $5 billion annually once cloud-managed services are included (research.md). Commercial interface engines run $50K–$300K per year for community hospitals and $300K–$2M+ for large IDNs and national payers, while iPaaS brokers like Redox charge $20K–$150K per connection. Primary buyers are hospital and IDN IT architects, payer CIOs facing CMS-0057-F and TEFCA mandates, digital health startups needing multi-EHR connectivity, and government and life-sciences organisations building population-health and real-world-data pipelines.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
