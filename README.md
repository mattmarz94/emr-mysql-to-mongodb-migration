# EMR Database Migration: MySQL → MongoDB

## Project Overview

This project implements an end-to-end **ETL (Extract, Transform, Load)** pipeline in Python to migrate Electronic Medical Record (EMR) data from a **normalized MySQL relational database** into a **MongoDB document-oriented database**.

The objective was to redesign a healthcare data model from a join-heavy relational schema into a denormalized, read-optimized MongoDB document structure while preserving referential integrity.

This project demonstrates practical experience in:

- Relational schema analysis
- NoSQL schema design
- Embedding vs referencing decisions
- Many-to-many relationship transformation
- Python-based data pipeline development

---

## Architecture Summary

### Source: Relational Model (MySQL)

The original schema included:

- Core entities: `patient`, `visit`, `provider`, `diagnosis`, `lab`, `clinical_procedures`, `symptom`
- Junction tables implementing many-to-many relationships:
  - `visit_diagnosis`
  - `visit_lab`
  - `visit_procedure`
  - `visit_symptom`

Retrieving a complete patient visit history required multiple joins across tables.

---

### Target: Document Model (MongoDB)

The MongoDB schema was redesigned as follows:

- **Patients** stored as root documents
- **Visits embedded** within each patient document
- **Diagnoses, labs, procedures, and symptoms embedded** within each visit
- **Providers stored as a separate collection** and referenced using `ObjectId()`

This structure:

- Eliminates join complexity
- Supports efficient retrieval of full patient histories
- Preserves relationship integrity
- Reduces duplication where appropriate

---

## ETL Pipeline Overview

The migration pipeline was built using Python:

### Extract

- Connected to MySQL using `pymysql`
- Pulled data from core and junction tables

### Transform

- Converted normalized relational records into nested document structures
- Converted MySQL `DATE` values to MongoDB-compatible `datetime`
- Transformed many-to-many relationships into embedded arrays
- Created `ObjectId()` references for provider documents

### Load

- Inserted reference collections (providers, diagnoses, labs, procedures, symptoms)
- Constructed and inserted fully embedded patient documents
- Generated structured JSON output for validation

---

## Example Document Structure

```json
{
  "patient_id": 1,
  "first_name": "John",
  "last_name": "Doe",
  "visits": [
    {
      "visit_id": 12,
      "visit_date": "2023-05-10T00:00:00",
      "provider": {
        "provider_id": 5,
        "provider_ref": "ObjectId(...)"
      },
      "diagnoses": [...],
      "labs": [...],
      "procedures": [...],
      "symptoms": [...]
    }
  ]
}
```
