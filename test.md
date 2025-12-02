# GetPatient – README

## 1. Overview

**Channel Name:** GetPatient  
**Type:** Polling JavaScript Receiver → JavaScript Destination  

**Purpose:**  
This channel periodically retrieves a **Bearer token**, calls an external **Patient API**, receives a JSON array of patients, and updates a PostgreSQL table (`adt_messages`) by inserting the **patient_id** for matching MRNs.

The workflow repeats automatically every **5 minutes**.

---

## 2. High-Level Workflow

```

Scheduled Poll (every 5 minutes)
↓
Step 1: GetToken (POST /api-token-auth/)
↓
Step 2: GetPatient (GET /api/patient)
↓
Step 3: Parse patient list + update DB
↓
End

```

---

## 3. Polling Configuration

The Source Connector is a **JavaScript Reader** with these settings:

| Setting | Value |
|---------|-------|
| Poll Type | Interval |
| Frequency | 300000 ms (5 minutes) |
| Poll on Start | Yes |
| Inbound/Outbound Data Type | JSON |

The source simply returns `1`, triggering the workflow during each poll.

---

## 4. API Interactions

### 4.1 Step 1 – GetToken

**Method:** `POST`  
**Endpoint:**  
```

[https://telemetry-rhm.select.corp.sem/core/api-token-auth/](https://telemetry-rhm.select.corp.sem/core/api-token-auth/)

```

**Headers:**
```

Content-Type: application/json

````

**Payload (example):**
```json
{
  "username": "XXXXXXXXX",
  "password": "XXXXXXXXX!"
}
````

**Response Handling:**

The channel extracts:

```javascript
var token = jsonResponse.tokens.access;
channelMap.put('token', token);
```

The token is stored and reused for the next API call.

**Note:**
The script uses a custom SSL context that **trusts all certificates**.

---

### 4.2 Step 2 – GetPatient

**Method:** `GET`
**Endpoint:**

```
https://telemetry-rhm.select.corp.sem/core/api/patient
```

**Headers:**

```
Accept: application/json
Authorization: Bearer <token>
```

**Response:**

The API returns a JSON **array** of patients:

```json
[
  {
    "id": 16,
    "medical_record_number": "432553",
    "date_of_birth": "1970-01-01",
    "gender": "M",
    "alertness_level": "ALERT",
    "patient_type": 0
  },
  ...
]
```

The full raw JSON is stored in:

```javascript
channelMap.put('patientresponse', result);
```

---

## 5. Database Update Logic

### Database Connection

```javascript
DatabaseConnectionFactory.createDatabaseConnection(
    'org.postgresql.Driver',
    'jdbc:postgresql://db1:5432/mirthdb',
    'mirthdb',
    'mirthdb'
);
```

### Logic Flow

For every JSON object in the patient list:

1. Extract fields:

   * `jsonData.id`
   * `jsonData.medical_record_number` (MRN)

2. Check if the MRN exists in `adt_messages`:

```sql
SELECT COUNT(medical_record_number)
FROM adt_messages
WHERE medical_record_number = ?
```

3. If exists → update the patient_id:

```sql
UPDATE adt_messages
SET patient_id = ?
WHERE medical_record_number = ?
```

4. Log each update:

```
ID: <id> MRN: <mrn>
```

If database connection or query fails, errors are logged with stack trace.

---

## 6. Destination Connector Summary

Only **one destination** exists: **GetToken → GetPatient → Parse Patient** inside a single chain.

| Step          | Type       | Purpose                    |
| ------------- | ---------- | -------------------------- |
| GetToken      | JavaScript | Obtain Bearer token        |
| GetPatient    | JavaScript | Retrieve full patient list |
| parse patient | JavaScript | Update DB for each MRN     |

There is **no outbound response** returned to source because the channel is polling, not externally triggered.

---

## 7. Channel Behavior Summary

* Runs automatically every 5 minutes.
* Makes HTTPS calls with SSL validation disabled.
* Logs token, API response, and DB operations.
* Updates existing rows only—no inserts.
* Matches patients strictly by **medical_record_number**.
* Stores:

  * `patient_id` in `adt_messages` table.

---

## 8. Error Handling

### API Errors

* Logged using:

```javascript
logger.error("Error executing POST/GET request: " + e);
```

* Does **not** break polling; next cycle continues normally.

### Database Errors

* Errors logged, connection closed gracefully.

---

## 9. Deployment Requirements

* Mirth Connect **v4.5.2**
* Windows Server or Linux
* PostgreSQL server:

  * Host: `db1`
  * Port: `5432`
  * DB: `mirthdb`
  * User/password: `mirthdb`
* Outbound HTTPS allowed to:

  * `telemetry-rhm.select.corp.sem`
* Firewall must allow outbound 443
* Ensure table `adt_messages` contains:

  * `medical_record_number`
  * `patient_id`

---
