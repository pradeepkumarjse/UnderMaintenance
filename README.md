# EpicIDSearch – README

## 1. Overview

**Channel Name:** EpicIDSearch  
**Protocol:** HTTP Listener  
**Port:** 4000  
**Response Format:** JSON  

The EpicIDSearch channel exposes an HTTP API that accepts **query parameters** (encounterid or medical_record_number), queries a **PostgreSQL database**, and returns **patient demographic data** in JSON format.

This channel supports:

1. **Search by Encounter ID**  
2. **Search by Medical Record Number (MRN)**  
3. **Automatic detection** of which search parameter is provided  
4. **JSON output** with patient details  
5. **Age-based classification (adult / pediatric / neonate)**  

---

## 2. Workflow

```

External System
↓ HTTP GET /?encounterid=... OR /?medical_record_number=...
HTTP Listener (Port 4000)
↓ Parse Query Parameters
↓ Evaluate Which Search to Perform
↓ Search PostgreSQL (adt_messages)
↓ Build JSON Response
↓ Return JSON to Client

```

---

## 3. Input Request Format

### Supported Query Parameters

| Parameter | Required | Purpose |
|----------|----------|---------|
| `encounterid` | Optional | Search by encounter ID |
| `medical_record_number` | Optional | Search by MRN (supports multiple comma-separated MRNs) |

You must provide **one of the two**.

### Example Requests

#### Search by Encounter ID

```

GET http://<server>:4000/?encounterid=ENC1234

```

#### Search by MRN (single)

```

GET http://<server>:4000/?medical_record_number=MRN1001

```

#### Search by MRN (multiple)

```

GET http://<server>:4000/?medical_record_number=MRN1,MRN2,MRN3

````

---

## 4. Output Response Format

### 4.1 Response for Encounter ID Search

If record found:

```json
{
  "encounterid": "12345",
  "first_name": "John",
  "last_name": "Doe",
  "medical_record_number": "MRN001",
  "date_of_birth": "1990-11-02",
  "gender": "M",
  "patient_type": "adult",
  "height": 175,
  "weight": 75
}
````

If no record found:

```json
{ "message": "No records found.." }
```

---

### 4.2 Response for MRN Search

If results contain **multiple patients**, an array is returned:

```json
[
  {
    "mrn": "MRN101",
    "first_name": "A",
    "last_name": "B",
    "encounterid": "E1",
    "date_of_birth": "1988-01-01",
    "gender": "F"
  },
  {
    "mrn": "MRN102",
    "first_name": "C",
    "last_name": "D",
    "encounterid": "E2",
    "date_of_birth": "1975-04-21",
    "gender": "M"
  }
]
```

If one record found, single JSON object returned.
If none found:

```json
{ "message": "No records found.." }
```

---

## 5. Search Logic

### Parameter Extraction (Transformer JS)

The channel extracts query variables from HTTP URL:

* Splits incoming query string
* Decodes keys & values
* Identifies which parameter is present
* Sets flags:

  * `Isencounterid` = 1 if encounterid is provided
  * `Ismedical_record_number` = 1 if MRN is provided

---

## 6. Database Queries

### PostgreSQL Connection

Connection used by both destination scripts:

```javascript
DatabaseConnectionFactory.createDatabaseConnection(
  'org.postgresql.Driver',
  'jdbc:postgresql://db1:5432/mirthdb',
  'mirthdb',
  'mirthdb'
);
```

### Encounter ID Query

Searches by encounterid, returns one row:

```sql
SELECT id, first_name, last_name, medical_record_number,
       date_of_birth, gender, height, weight
FROM adt_messages
WHERE encounterid = ?
ORDER BY id
LIMIT 1;
```

### MRN Query (supports multiple MRNs)

```sql
SELECT DISTINCT first_name, last_name, encounterid,
       date_of_birth, gender, medical_record_number
FROM adt_messages
WHERE medical_record_number IN (?, ?, ?, ...)
```

---

## 7. Business Rules

### Age Classification

Age in months is calculated:

| Age in Months | Category  |
| ------------- | --------- |
| ≥ 216 months  | adult     |
| 1–215 months  | neonate   |
| < 1 month     | pediatric |

---

## 8. Channel Structure (Summary)

### **Source Connector**

* **Type:** HTTP Listener
* **Host:** 0.0.0.0
* **Port:** 4000
* **Response Type:** JSON
* **Parses query parameters** in transformer

### **Destination Connectors**

1. **Search encounterid**

   * Executes encounterid DB query
   * Outputs JSON

2. **Search MRN**

   * Supports multiple MRNs
   * Returns single JSON or array

3. **Final Response**

   * Returns `response_patientData` JSON to caller

---

## 9. Deployment Requirements

* Windows Server with Mirth installed
* Mirth Connect v4.5.2
* PostgreSQL accessible at `db1:5432`
* DB credentials:

  * **User:** mirthdb
  * **Password:** mirthdb
* Firewall open:

  * **HTTP port 4000**

---

## 10. Example Success Response

```json
{
  "first_name": "John",
  "last_name": "Doe",
  "encounterid": "E10001",
  "medical_record_number": "MRN2001",
  "date_of_birth": "1985-03-22",
  "gender": "M",
  "patient_type": "adult",
  "height": 180,
  "weight": 80
}
```

---

## 11. Example Error Response

### Database Error Handling

If DB connection fails, the logs capture the error & stack trace.

Returned JSON may look like:

```json
{ "message": "No records found.." }
```
---
## 12. Maintainer Notes

* Query parsing logic is inside Source Transformer
* EncounterID and MRN searches are mutually exclusive
* Response always returned as JSON
* Logging includes DOB, age calculation & query diagnostics

```
```
