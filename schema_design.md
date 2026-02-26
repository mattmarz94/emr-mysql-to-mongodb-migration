Part 1: Analyze the MySQL Schema 
Overview of Tables
The emr MySQL database contains the following 11 tables:
•	clinical_procedures
•	diagnosis
•	lab
•	patient
•	provider
•	symptom
•	visit
•	visit_diagnosis
•	visit_lab
•	visit_procedure
•	visit_symptom
The schema follows a normalized relational design where core entities are separated into individual tables and many-to-many relationships are implemented using junction tables.

Primary Keys
Each core entity table uses a single-column primary key:
•	patient - patient_id
•	provider - provider_id
•	visit - visit_id
•	diagnosis - diagnosis_id
•	lab - lab_id
•	clinical_procedures - procedure_id
•	symptom - symptom_id
The junction tables use composite primary keys composed of foreign keys:
•	visit_diagnosis - (visit_id, diagnosis_id)
•	visit_lab - (visit_id, lab_id)
•	visit_procedure - (visit_id, procedure_id)
•	visit_symptom - (visit_id, symptom_id)
These composite keys enforce the uniqueness of relationships between visits and their associated clinical entities.

Foreign Key Relationships
The schema defines foreign key constraints that enforce referential integrity between tables.
Many-to-One Relationships
The following relationships are many-to-one:
•	visit.patient_id  patient.patient_id
o	Many visits belong to one patient.
•	visit.provider_id  provider.provider_id
o	Many visits are associated with one provider.
These relationships represent core clinical interactions: a patient may have multiple visits, and a provider may see multiple patients across many visits.
Many-to-Many Relationships
The schema implements many-to-many relationships using junction tables:
•	visit ↔ diagnosis (via visit_diagnosis)
•	visit ↔ lab (via visit_lab)
•	visit ↔ clinical_procedures (via visit_procedure)
•	visit ↔ symptom (via visit_symptom)
Example:
•	A single visit may have multiple diagnoses.
•	A diagnosis may appear in multiple visits.
•	A visit may include multiple labs, procedures, and symptoms.
These relationships require join operations in MySQL to retrieve complete visit information.
One-to-One Relationships
No one-to-one relationships were identified in the schema. All relationships are either many-to-one or many-to-many.
Implications for MongoDB Scheme Design
The relational analysis directly influenced the MongoDB document design:
•	Since visit has a many to one relationship with patient, visits were embedded inside patient documents to support the retrieval of the complete patient visit history.
•	Since diagnoses, labs, procedures, and symptoms are many to many with visits and require junction tables in MySQL, these entities were embedded within each visit document in MongoDB. This eliminates the need for join operations and produces self-contained visit records.
•	Since providers are referenced by many visits and shared across multiple patients, the provider entity was kept as a separate collection and referenced using ObjectId() to avoid duplication and support flexible updates.
This transformation preserves the logical relationships of the relational model while leveraging MongoDB’s document-oriented strengths to simplify retrieval and reduce join complexity.
Part 2: Design MongoDB Schema 
Collection 1: patients
{
  "_id": ObjectId(),
  "patient_id": Number,
  "first_name": String,
  "last_name": String,
  "dob": Date,
  "gender": String,
  "visits": [
    {
      "visit_id": Number,
      "visit_date": Date,
      "provider": {
        "provider_ref": ObjectId(),
        "provider_id": Number
      },
      "diagnoses": [
        {
          "diagnosis_id": Number,
          "name": String,
          "icd10_code": String
        }
      ],
      "labs": [
        {
          "lab_id": Number,
          "cpt_code": String,
          "lab_name": String
        }
      ],
      "procedures": [
        {
          "procedure_id": Number,
          "icd10_code": String,
          "proc_name": String,
          "description": String
        }
      ],
      "symptoms": [
        {
          "symptom_id": Number
        }
      ]
    }
  ]
}

Justification:
•	Why patients are separate collections, and it embeds visits: patients are the root entity in the EMR model, and the migration is designed around retrieving a patient’s complete visit history. Since all visits relate to a patient, storing patients as top-level documents provides a foundation for embedding visits and visit-level clinical details within a single document structure.
•	Example: Generating sample_output.json requires exporting a patient document that includes all embedded visits, including visit_date, and each visit’s diagnoses/labs/procedures/symptoms in one record.
Collection 2: providers
{
  "_id": ObjectId(),
  "provider_id": Number,
  "first_name": String,
  "last_name": String,
  "specialty": String
}
Justification:
•	Why providers are a separate collection and are referenced: providers is a shared entity referenced by many visits. Keeping providers in their own collection prevents duplicating provider attributes across many embedded visits and supports updates such as the providers specialty or any name changes by updating one provider document rather than many patient documents. 
•	Example: If a provider’s specialty changes, updating the single document in providers keeps all visits consistent because visits store a provider reference instead of duplicating provider details.

Collection 3: diagnoses
{
  "_id": ObjectId(),
  "diagnosis_id": Number,
  "name": String,
  "icd10_code": String
}
Justification:
•	Why diagnoses are separate collections and are embedded: diagnoses  functions as a standardized lookup table in MySQL and can be associated with many visits through visit_diagnosis. A separate collection preserves the diagnosis definitions while still allowing diagnosis details to be embedded inside each visit for complete visit records.
•	Example:  In MySQL, diagnoses for a visit require joining visit_diagnoses to diagnoses while in MongoDB the diagnoses are already embedded in the visit, so a patient visit display does no require an additional lookup.

Collection 4: labs
{
    "_id": ObjectId(),
    "lab_id": Number,
    "cpt_code": String,
    "lab_name": String
}
Justification:
•	Why labs are separate collections and are embedded: labs is a reusable reference table that links to visits via visit_lab. Keeping labs in a separate collection maintains a centralized definition of lab tests that can be reused across many visits without having to repeatedly define the tests. So a visit can contain multiple labs and embedding makes the visit record complete for “visit history” output.
•	Example: When viewing one visit in the patient portal, the lab tests tied to that visit are available directly within the visit subdocument.

Collection 5: clinical_procedures
{
    "_id": ObjectId(),
    "procedure_id": Number,
    "icd10_code": String,
    "proc_name": String,
    "description": String
}
Justification:
•	Why procedures is a separate collection and are embedded: clinical_procedures is a catalog of standardized procedures referenced via visit_procedure. A separate collection supports reuse across many visits and avoids repeating procedure metadata across embedded visit documents, 
•	Example: A visit summary should display the procedures performed during that encounter. Embedding procedures inside the visit allows this summary without requiring a join through visit_procedure.




Collection 6: symptoms
{
"_id": ObjectId(),
"symptom_id": Number,
"note": String
}
Justification:
•	Why symptoms are separate collections and are embedded: symptoms is a reusable clinical entity associated with visits through visit_symptom. Storing symptoms separately preserves a consistent symptom catalog and avoids repeated duplication of symptom definitions across many visits, while symptoms can still be embedded within each visit for self-contained visit history output.
•	Example: A visit record should list all symptoms reported during that encounter. Embedding symptoms inside the visit ensures they appear directly within the visit history output.
