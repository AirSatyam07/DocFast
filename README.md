# DocFast

## Project Overview

DocFast is a production-oriented asynchronous document processing backend built with FastAPI.

The initial version focuses on **XLSX-based engineering documents** in a controlled domain. The system accepts structured Excel workbooks, validates them against document-specific schemas, applies deterministic business rules, normalizes the data, and produces structured results.

The long-term goal is to evolve DocFast into an enterprise document-processing and AI-assisted engineering data platform.

---

## Table of Contents

**Core design (original sections)**

1. [Problem Statement](#1-problem-statement)
2. [Core Concept](#2-core-concept)
3. [Domain Scope](#3-domain-scope)
4. [Document-Specific Schemas](#4-document-specific-schemas)
5. [Document Schema vs Database Schema](#5-document-schema-vs-database-schema)
6. [Schema Versioning](#6-schema-versioning)
7. [Initial Processing Pipeline](#7-initial-processing-pipeline)
8. [Synthetic Data](#8-synthetic-data)
9. [Business Validation](#9-business-validation)
10. [Job Concept](#10-job-concept)
11. [Planned Job Lifecycle](#11-planned-job-lifecycle)
12. [Planned Architecture](#12-planned-architecture)
13. [Initial API Direction](#13-initial-api-direction)
14. [Architectural Principles](#14-architectural-principles)
15. [Future Scope](#15-future-scope)
16. [AI Extension](#16-ai-extension)
17. [Long-Term AI Architecture](#17-long-term-ai-architecture)
18. [Final Evolution](#18-final-evolution)
19. [Current V1 Scope](#19-current-v1-scope)
20. [Initial Project Structure](#20-initial-project-structure)

**New sections**

21. [Error Model and Error Codes](#21-error-model-and-error-codes)
22. [Configuration-Driven Validation Rules](#22-configuration-driven-validation-rules)
23. [Getting Started](#23-getting-started)
24. [V1 API (Proposed Minimal Surface)](#24-v1-api-proposed-minimal-surface)
25. [Testing Strategy](#25-testing-strategy)
26. [Security and Upload Safety](#26-security-and-upload-safety)
27. [Engineering Standards and Tooling](#27-engineering-standards-and-tooling)
28. [Additional Planned Features](#28-additional-planned-features)
29. [AI Quality and Governance](#29-ai-quality-and-governance)

---

# 1. Problem Statement

Generic file-upload systems are easy to build but provide little meaningful business logic.

DocFast focuses on a specific engineering domain:

> **Aircraft Engine Maintenance and Engineering Documents**

For the first version, we deliberately restrict the system to:

- XLSX input
- One engineering domain
- One primary document type
- Schema-driven validation
- Deterministic processing
- Structured JSON output

The system will later expand to multiple document types, asynchronous processing, large-scale batch processing, AI-assisted schema mapping, RAG, anomaly detection, and controlled AI agents.

---

# 2. Core Concept

A file and a document are not the same thing.

An XLSX file is the physical representation containing bytes.

A **Document** is DocFast's application-level representation of that file, including metadata such as:

- document ID
- owner
- filename
- size
- MIME type
- checksum
- storage location
- creation/update timestamps

Conceptually:

```text
                    Document
                       |
             +---------+---------+
             |                   |
             v                   v
        Metadata              File Bytes
             |                   |
             v                   v
        PostgreSQL          File Storage
```

This separation allows the application database to manage metadata and relationships while file storage handles the actual file contents.

---

# 3. Domain Scope

## Initial Domain

**Aircraft Engine Maintenance**

## Initial Document Type

```text
ENGINE_MAINTENANCE_REPORT
```

## Initial Input Format

```text
.xlsx
```

The XLSX format is only the transport/file representation.

The actual business meaning comes from the **document type and its schema**.

For example:

```text
XLSX
  |
  v
ENGINE_MAINTENANCE_REPORT
  |
  v
Schema v1
```

---

# 4. Document-Specific Schemas

Each document type has its own business schema.

For example:

```text
ENGINE_MAINTENANCE_REPORT

report_id
aircraft_model
engine_model
engine_serial
inspection_date
flight_hours
flight_cycles
n1_max
n2_max
oil_pressure
oil_temperature
inspection_status
```

A different document type may have a completely different schema.

Example:

```text
ENGINE_TEST_REPORT

test_id
engine_serial
test_date
test_duration
n1
n2
egt
fuel_flow
test_status
```

Therefore, DocFast does not attempt to create one universal schema for every document.

---

# 5. Document Schema vs Database Schema

These are two different concepts.

## Document Schema

Defines what an engineering document contains and what values are valid.

```text
Field
  |
  +-- name
  +-- type
  +-- required/optional
  +-- validation rules
  +-- normalization rules
```

## Database Schema

Defines how DocFast stores application state.

Initial entities:

```text
users
documents
jobs
job_steps
results
audit_logs
```

The document schema describes **business data**.

The database schema describes **application persistence**.

---

# 6. Schema Versioning

Document schemas will be versioned.

Example:

```text
ENGINE_MAINTENANCE_REPORT
    |
    +-- v1
    |
    +-- v2
    |
    +-- v3
```

A document can record:

```text
document_type
schema_version
```

A processing job can additionally record:

```text
pipeline_version
```

This gives reproducibility.

For example:

```text
Document
   |
   +-- ENGINE_MAINTENANCE_REPORT
   +-- schema v2
   |
   v
Pipeline v3
   |
   v
Result
```

If the schema changes later, historical processing remains explainable.

---

# 7. Initial Processing Pipeline

V1 processing:

```text
                 XLSX
                   |
                   v
              Parse Workbook
                   |
                   v
             Identify Schema
                   |
                   v
             Schema Validation
                   |
                   v
           Business Validation
                   |
                   v
               Normalize
                   |
                   v
            Structured JSON
```

Example input:

```text
Report ID | Engine Serial | N1 | N2
------------------------------------
EMR001    | ESN123        | 97 | 101
```

Possible output:

```json
{
  "document_type": "ENGINE_MAINTENANCE_REPORT",
  "schema_version": "1.0",
  "data": {
    "report_id": "EMR001",
    "engine_serial": "ESN123",
    "n1_max": 97.0,
    "n2_max": 101.0
  }
}
```

---

# 8. Synthetic Data

No real engineering/company data is required.

DocFast will use synthetic data for development and testing.

Example:

```text
EMR-2026-000001
EMR-2026-000002
EMR-2026-000003
...
```

Synthetic documents will contain realistic but fictional values.

We will create both:

1. Valid datasets
2. Intentionally invalid datasets

Examples of invalid cases:

- missing required column
- invalid date
- invalid numeric value
- value outside allowed range
- duplicate record
- malformed workbook
- corrupted input
- inconsistent values

This allows the processing engine and tests to be developed without relying on private data.

---

# 9. Business Validation

Validation is divided into multiple layers.

## Structural validation

Does the workbook contain the expected columns?

```text
engine_serial
n1_max
n2_max
```

## Type validation

Are values actually of the expected type?

```text
n1_max → float
inspection_date → date
```

## Business validation

Are values meaningful according to domain rules?

Example:

```text
N1 must be within configured limits
N2 must be within configured limits
inspection_date cannot be invalid
```

## Result

Validation should produce useful error information.

Example:

```json
{
  "status": "FAILED",
  "errors": [
    {
      "row": 18,
      "column": "N1",
      "value": 147.3,
      "rule": "N1 must be within the configured range"
    }
  ]
}
```

See [Section 21](#21-error-model-and-error-codes) for the extended error model with stable error codes and severities.

---

# 10. Job Concept

A Document answers:

> What data are we working with?

A Job answers:

> What work should DocFast perform on that data?

Example:

```text
Document
    |
    v
invoice/engineering report
    |
    v
Job
    |
    +-- validate
    +-- extract
    +-- normalize
    +-- export
    |
    v
Result
```

The Job is the unit of processing and later becomes the basis for asynchronous execution.

---

# 11. Planned Job Lifecycle

Eventually:

```text
QUEUED
   |
   v
RUNNING
   |
   +------> COMPLETED
   |
   +------> FAILED
   |
   +------> CANCELLED
```

Failed jobs may later support:

```text
FAILED
  |
  v
RETRYING
  |
  v
QUEUED
```

State transitions will be explicitly controlled rather than allowing arbitrary status changes.

---

# 12. Planned Architecture

The initial architecture will be deliberately small.

Long-term:

```text
                         Client
                           |
                           v
                       FastAPI
                           |
             +-------------+-------------+
             |             |             |
             v             v             v
        PostgreSQL       Redis      File Storage
                           |
                           v
                     Celery Workers
                           |
                           v
                   Processing Engine
                           |
                           v
                        Result
```

### PostgreSQL

Source of truth for application state:

```text
users
documents
jobs
job_steps
results
audit_logs
```

### Redis

Used for infrastructure such as:

- queues
- caching
- short-lived state

### Celery

Used for long-running/background processing.

### File Storage

Stores actual XLSX/document bytes.

Initial development can use local storage.

Later:

```text
Local Storage
      |
      v
S3 / Azure Blob / GCS
```

---

# 13. Initial API Direction

Planned API surface:

## Authentication

```text
POST   /api/v1/auth/register
POST   /api/v1/auth/login
POST   /api/v1/auth/refresh
POST   /api/v1/auth/logout
GET    /api/v1/users/me
```

## Documents

```text
POST   /api/v1/documents
GET    /api/v1/documents
GET    /api/v1/documents/{document_id}
DELETE /api/v1/documents/{document_id}
```

## Jobs

```text
POST   /api/v1/jobs
GET    /api/v1/jobs
GET    /api/v1/jobs/{job_id}
POST   /api/v1/jobs/{job_id}/cancel
POST   /api/v1/jobs/{job_id}/retry
```

## Results

```text
GET /api/v1/jobs/{job_id}/result
```

These endpoints are a target design, not all V1 requirements. See [Section 24](#24-v1-api-proposed-minimal-surface) for the proposed minimal V1 surface.

---

# 14. Architectural Principles

DocFast will follow several engineering principles.

## Separate API from business logic

Preferred:

```text
Router
   |
   v
Schema
   |
   v
Service
   |
   v
Repository
   |
   v
Database
```

Business logic should not become a large block inside FastAPI route functions.

## Deterministic processing

Business rules should be deterministic and testable.

## AI handles ambiguity

AI should assist with tasks such as:

- classification
- schema mapping
- natural language interpretation
- explanation
- knowledge retrieval

It should not silently replace deterministic business validation.

## Explicit failure handling

Failures should be represented and observable.

## Reproducibility

Schema and pipeline versions should be recorded.

## Least necessary complexity

New infrastructure should be introduced only when the system has a concrete reason for it.

---

# 15. Future Scope

## V2 — Multiple Document Types

Support:

```text
ENGINE_MAINTENANCE_REPORT
ENGINE_TEST_REPORT
COMPONENT_INSPECTION_REPORT
```

Each document type has its own schema.

---

## V3 — Schema Registry

Introduce a schema registry:

```text
Schema Registry
    |
    +-- ENGINE_MAINTENANCE_REPORT
    |       +-- v1
    |       +-- v2
    |
    +-- ENGINE_TEST_REPORT
    |       +-- v1
    |
    +-- COMPONENT_INSPECTION_REPORT
            +-- v1
```

Schemas become configurable/versioned domain contracts.

---

## V4 — Asynchronous Processing

Move expensive processing out of HTTP requests.

```text
POST /jobs
     |
     v
202 Accepted
     |
     v
Redis
     |
     v
Celery Worker
     |
     v
Processing
```

The client can query:

```text
GET /jobs/{job_id}
```

for status.

---

## V5 — Batch Processing

Support thousands of documents:

```text
Batch Job
   |
   +-- Document 1
   +-- Document 2
   +-- Document 3
   +-- ...
   +-- Document N
```

Workers can process documents concurrently.

Potential concerns:

- concurrency
- backpressure
- rate limiting
- worker scaling
- partial failures

---

## V6 — Object Storage

Move from local storage to:

```text
AWS S3
Azure Blob Storage
Google Cloud Storage
```

This allows API instances to remain stateless.

---

## V7 — Data Quality Engine

Produce detailed quality reports:

```text
valid records
invalid records
missing fields
type errors
business-rule violations
duplicates
```

---

# 16. AI Extension

AI will be added after the deterministic processing layer is reliable.

The target architecture:

```text
                 DocFast
                    |
        +-----------+-----------+
        |                       |
        v                       v
Deterministic Layer         AI Layer
        |                       |
        +-----------+-----------+
                    |
                    v
                 Results
```

Core principle:

> **AI handles ambiguity; deterministic code handles correctness.**

---

## AI Feature 1 — Intelligent Schema Mapping

Input XLSX may contain:

```text
ESN
N1 Maximum
N2 Max RPM
Oil Press.
Inspection Dt
```

Canonical schema:

```text
engine_serial
n1_max
n2_max
oil_pressure
inspection_date
```

AI can propose:

```text
ESN
  -> engine_serial

N1 Maximum
  -> n1_max

N2 Max RPM
  -> n2_max
```

The AI result should then pass through deterministic validation.

---

## AI Feature 2 — Document Classification

Given an unknown supported workbook:

```text
XLSX
  |
  v
AI Classifier
  |
  v
ENGINE_TEST_REPORT
  |
  v
Schema v1
```

Low-confidence predictions can become:

```text
REVIEW_REQUIRED
```

rather than being automatically accepted.

---

## AI Feature 3 — Natural Language Data Query

User asks:

> Show engines where N2 exceeded 101% during September.

Architecture:

```text
Natural Language
       |
       v
LLM
       |
       v
Structured Query
       |
       v
Validation
       |
       v
SQLAlchemy
       |
       v
PostgreSQL
```

The LLM should not receive unrestricted database access.

It should produce a controlled query representation that the backend validates.

---

## AI Feature 4 — Engineering Assistant

Engineering documentation can be indexed:

```text
Maintenance Manuals
Engineering Procedures
Troubleshooting Guides
Technical Standards
```

Then:

```text
Question
   |
   v
RAG
   |
   +-- Retrieve relevant documents
   |
   v
LLM
   |
   v
Grounded response
```

Responses should include relevant source references where possible.

---

## AI Feature 5 — RAG

Future knowledge pipeline:

```text
Engineering Documents
        |
        v
Chunking
        |
        v
Embeddings
        |
        v
Vector Database
        |
        v
Retriever
        |
        v
LLM
```

This allows DocFast to answer questions using organization-specific engineering knowledge rather than relying only on model knowledge.

---

## AI Feature 6 — Anomaly Detection

Use historical structured data:

```text
N1
N2
Oil pressure
Oil temperature
Vibration
```

to identify unusual measurements.

Possible architecture:

```text
Historical Data
      |
      v
ML Model
      |
      v
Anomaly
      |
      v
RAG
      |
      v
Relevant Procedure
      |
      v
LLM Explanation
```

AI/ML can identify patterns while deterministic rules remain responsible for explicit business constraints.

---

## AI Feature 7 — Human-in-the-Loop

Low-confidence AI results should be reviewable.

```text
AI Prediction
     |
     v
Confidence
     |
 +---+---+
 |       |
High    Low
 |       |
 v       v
Accept  Human Review
```

Example:

```text
"ESN" -> "engine_serial"
confidence = 0.98
```

can be automatically accepted.

Whereas:

```text
"RPM" -> "n2_max"
confidence = 0.54
```

can require human confirmation.

---

## AI Feature 8 — Controlled AI Agents

A later version may introduce an agentic workflow.

The agent should operate through explicit tools:

```text
Agent
 |
 +-- classify_document()
 +-- get_schema()
 +-- validate_document()
 +-- search_knowledge()
 +-- get_report()
 +-- create_review_task()
```

The agent should not receive unrestricted database or infrastructure access.

---

# 17. Long-Term AI Architecture

```text
                         DocFast
                            |
          +-----------------+-----------------+
          |                                   |
          v                                   v
 Deterministic Processing                 AI Platform
          |                                   |
          |                         +---------+---------+
          |                         |         |         |
          |                         v         v         v
          |                    Classifier  RAG    Schema Mapping
          |                                   |
          |                                   v
          |                                LLM
          |                                   |
          +------------------+----------------+
                             |
                             v
                         Job Engine
                             |
                  +----------+----------+
                  |                     |
                  v                     v
             PostgreSQL              Redis
                                        |
                                        v
                                   Celery Workers
```

---

# 18. Final Evolution

The intended progression is:

```text
Phase 1
XLSX processing

Phase 2
Schema-driven validation

Phase 3
Multiple document types

Phase 4
Asynchronous jobs

Phase 5
Batch processing

Phase 6
Object storage

Phase 7
Data quality engine

Phase 8
AI schema mapping

Phase 9
AI document classification

Phase 10
Natural-language querying

Phase 11
RAG over engineering documentation

Phase 12
ML anomaly detection

Phase 13
AI-assisted investigation

Phase 14
Controlled AI agents

Phase 15
Enterprise observability, security, CI/CD and cloud deployment
```

---

# 19. Current V1 Scope

For now, freeze the scope here:

```text
Input:
    XLSX

Domain:
    Aircraft Engine Maintenance

Document Type:
    ENGINE_MAINTENANCE_REPORT

Processing:
    XLSX
      ↓
    Parse
      ↓
    Schema validation
      ↓
    Business validation
      ↓
    Normalize
      ↓
    JSON result

Infrastructure:
    FastAPI
    Python
```

No Redis, Celery, LLM, RAG, Kubernetes, or microservices in V1.

The goal is to build the core domain correctly first and introduce complexity only when there is a concrete engineering reason for it.

---

# 20. Initial Project Structure

Current target:

```text
DocFast/
│
├── app/
│   ├── __init__.py
│   └── main.py
│
├── tests/
│
├── .gitignore
├── README.md
└── requirements.txt
```

The architecture will evolve as the domain grows. We should avoid creating empty architecture folders before their responsibilities are understood.

---

# 21. Error Model and Error Codes

Validation errors should be machine-readable so the frontend, tests, and later AI explanations can rely on them.

## Error format

```json
{
  "status": "FAILED",
  "errors": [
    {
      "code": "RANGE_VIOLATION",
      "severity": "ERROR",
      "row": 18,
      "column": "N1",
      "value": 147.3,
      "rule": "N1 must be within the configured range"
    }
  ]
}
```

## Initial error codes

```text
MISSING_COLUMN
TYPE_MISMATCH
INVALID_DATE
RANGE_VIOLATION
DUPLICATE_RECORD
MALFORMED_WORKBOOK
INCONSISTENT_VALUES
```

## Severity levels

```text
ERROR     fails the document
WARNING   flagged, but the document can still pass
INFO      informational only
```

Severity prevents a single borderline value from failing an entire document when the domain does not require it.

---

# 22. Configuration-Driven Validation Rules

Business limits should not be hardcoded in Python.

Rules live in a configuration file (YAML or JSON), versioned together with the document schema.

Illustrative shape (values are placeholders, not real engineering limits):

```yaml
document_type: ENGINE_MAINTENANCE_REPORT
schema_version: "1.0"
fields:
  engine_serial:
    type: string
    required: true
  n1_max:
    type: float
    required: true
    range: { min: <configured>, max: <configured> }
  inspection_date:
    type: date
    required: true
```

Benefits:

- limits can change without code changes
- rules are reviewable by domain engineers
- prepares the system for the schema registry (V3)

---

# 23. Getting Started

```bash
# 1. Create and activate a virtual environment
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate

# 2. Install dependencies
pip install -r requirements.txt

# 3. Run the development server
uvicorn app.main:app --reload
```

Interactive API docs are available at `http://127.0.0.1:8000/docs`.

Run the tests:

```bash
pytest
```

---

# 24. V1 API (Proposed Minimal Surface)

Section 13 describes the full target design. For V1, a much smaller surface is enough:

```text
GET    /health
POST   /api/v1/documents/process     Upload an XLSX, receive the JSON result
```

The request is a multipart file upload. The response is either the structured JSON result (Section 7) or the error model (Section 21).

Authentication, persistence, and job endpoints arrive in later versions.

---

# 25. Testing Strategy

Testing relies on the synthetic datasets from Section 8.

- **Unit tests:** individual validators and normalizers
- **Golden-file tests:** each synthetic workbook has an expected JSON output that the pipeline must reproduce exactly
- **Negative tests:** every invalid dataset must produce the expected error codes
- **API tests:** FastAPI `TestClient` against the V1 endpoints
- **Determinism check:** processing the same file twice yields identical output

Tools: `pytest`, `httpx`/`TestClient`.

---

# 26. Security and Upload Safety

File-processing services accept untrusted input. Even in V1:

- enforce a maximum upload size
- check file extension and MIME type
- reject malformed or corrupted workbooks with `MALFORMED_WORKBOOK`
- do not execute macros or external links inside workbooks
- guard against oversized or decompression-bomb archives (XLSX is a zip file)
- never trust the client-supplied filename for storage paths
- deduplicate uploads using the file checksum

Later: authentication, role-based access, rate limiting, secrets management, and data retention and deletion policy.

---

# 27. Engineering Standards and Tooling

- **Formatting and linting:** ruff
- **Type checking:** mypy
- **Testing:** pytest
- **Hooks:** pre-commit
- **CI:** run lint, type check, and tests on every push (for example GitHub Actions)
- **Containers:** Docker and docker-compose once PostgreSQL and Redis are introduced
- **Migrations:** Alembic once PostgreSQL is introduced
- **Logging:** structured logs with request IDs and, later, job IDs

---

# 28. Additional Planned Features

Ideas beyond the original roadmap, to be adopted only when there is a concrete reason (Section 14):

## Domain depth

- cross-record checks (for example, flight hours must not decrease for the same engine serial)
- unit handling and normalization (psi vs kPa, °C vs °F)
- per-engine time-series storage, the foundation for anomaly detection
- lineage: raw upload, parsed data, normalized data, and result, linked to schema and pipeline versions

## Product features

- web frontend for upload, job status, and error viewing
- downloadable validation reports (Excel/PDF)
- dashboards: pass/fail rates and most common rule violations
- roles and permissions (uploader, reviewer, admin)
- audit trail UI for traceability
- webhooks or notifications on job completion
- downloadable XLSX templates per schema version

## Production concerns

- observability (metrics, tracing)
- backups and disaster recovery
- multi-tenancy if several organizations are served

---

# 29. AI Quality and Governance

To keep the AI layer trustworthy (Section 16):

- **Evaluation set:** a labeled set of messy headers (`ESN`, `N2 Max RPM`) to measure schema-mapping accuracy before and after any prompt or model change
- **Versioning:** prompt and model versions recorded on jobs, like `pipeline_version`
- **Structured outputs:** LLM responses constrained to a JSON schema and parsed defensively
- **Guardrails:** every AI output passes through deterministic validation before it affects data
- **Confidence thresholds:** configurable auto-accept and human-review thresholds
- **Cost and latency tracking:** per feature, with caching where safe
- **Explainability first:** plain-language explanations of validation failures (built on error codes) are the safest first AI feature, since they cannot change data