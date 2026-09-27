# DocFast

## Project Overview

DocFast is a production-oriented asynchronous document processing backend built with FastAPI.

The initial version focuses on **XLSX-based engineering documents** in a controlled domain. The system accepts structured Excel workbooks, validates them against document-specific schemas, applies deterministic business rules, normalizes the data, and produces structured results.

The long-term goal is to evolve DocFast into an enterprise document-processing and AI-assisted engineering data platform.

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

These endpoints are a target design, not all V1 requirements.

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
