# PDF_Listener – README

## 1. Overview

**Channel Name:** PDF_Listener  
**Protocol:** HTTP Listener (JSON + PDF binary)  

**Purpose:** This channel receives **JSON payloads (with optional PDF content or URL)** from a remote system.  
It extracts *patient information + order details + reviewer info*, enriches data from PostgreSQL, and sets all values into `channelMap` for downstream processing by other channels.

---

## 2. High-Level Workflow

```
External System → POST JSON/PDF → HTTP Listener (port 4001)
↓
Parse JSON payload
↓
If EMR order ID missing → lookup from rpm_orders table
↓
Lookup demographics from adt_messages table using MRN
↓
If message_type = "Review" → capture URL
Else → store PDF binary
↓
Populate channelMap variables
↓
Forward processing by Destination Channel(s) (not included here)

````
---

## 3. Input Format

### 3.1 JSON Input (example)

```json
{
  "patient_id": "217057",
  "message_type": "PDF",
  "order_emr_id": 5814773,
  "child_order_id": 5817367,
  "rpm_order_id": 126,
  "url": null,
  "reviewer": null,
  "reviewed_at": null,
  "pdf": "<base64 PDF binary>"
}
````

The channel supports **multipart and binary payloads**, automatically parsed by Mirth.

---

## 4. Message Parsing Logic

### Extracted fields:

| Field            | Source                   | Required             |
| ---------------- | ------------------------ | -------------------- |
| `patient_id`     | JSON                     | Yes                  |
| `message_type`   | JSON (`Review` or other) | Yes                  |
| `order_emr_id`   | JSON or DB lookup        | Optional             |
| `child_order_id` | JSON                     | Optional             |
| `rpm_order_id`   | JSON                     | Yes                  |
| `url`            | JSON                     | Only for Review      |
| `pdf`            | JSON                     | Only for PDF uploads |
| `reviewer`       | JSON                     | Optional             |
| `reviewed_at`    | JSON                     | Optional             |

---

## 5. EMR Order ID Auto-Lookup

If the incoming JSON does **not** contain `order_emr_id`, the channel will:

### Query:

```sql
select order_emr_id 
from rpm_orders 
where medical_record_number = ?
order by whenadded desc 
limit 1;
```

Extracted value → stored as `emr_order_id`.

If no record found → `child_order_id = ""`.

---

## 6. Reviewed_at Timestamp Reformatting

Incoming format:

```
2025-03-12T05:30:22.047266+00:00
```

Converted to HL7/DB friendly format:

```
yyyyMMddHHmmss
```

Example: `20250312053022`

---

## 7. Patient Demographic Lookup (adt_messages)

Query:

```sql
SELECT pv1,first_name,last_name,date_of_birth,gender,
       marital_status,phone_number,address1,city,state,zip 
FROM adt_messages 
WHERE medical_record_number = ?
ORDER BY id 
LIMIT 1;
```

If found, the channel populates:

* `pv1`
* `first_name`
* `last_name`
* `date_of_birth` (normalized into `YYYYMMDD`)
* `gender`
* `marital_status`
* `phone_number`
* `address1`
* `city`
* `state`
* `zip`

If not found → `filter = 1` (used by downstream routing).

---

## 8. PDF & URL Handling

| message_type   | Behavior                              |
| -------------- | ------------------------------------- |
| `"Review"`     | Uses `msg.url`, no PDF                |
| Any other type | Extracts `msg.pdf` (base64 or binary) |

Stored into:

* `channelMap['url']`
* `channelMap['PDF']`

---

## 9. ChannelMap Variables (Final Output)

The script outputs the following variables for downstream connectors:

```
patient_id
message_type
emr_order_id
child_order_id
rpm_order_id
reviewer
reviewed_at
datetime
medical_record_number
pv1
last_name
first_name
date_of_birth
gender
marital_status
phone_number
address1
city
state
zip
url (conditional)
PDF (conditional)
filter (determines routing path)
```

---

## 10. Source Connector Configuration

| Setting                | Value                                                    |
| ---------------------- | -------------------------------------------------------- |
| Type                   | HTTP Listener                                            |
| Host                   | 0.0.0.0                                                  |
| Port                   | 4001                                                     |
| parseMultipart         | true                                                     |
| xmlBody                | false                                                    |
| responseContentType    | application/json                                         |
| respondAfterProcessing | true                                                     |
| binaryMimeTypes        | `application/* (except json, xml)` + images/videos/audio |
| timeout                | 30000 ms                                                 |

---

## 11. Database Connection

All DB calls use:

```javascript
DatabaseConnectionFactory.createDatabaseConnection(
  'org.postgresql.Driver',
  'jdbc:postgresql://db1:5432/mirthdb',
  'mirthdb', 
  'mirthdb'
);
```

---

## 12. Deployment Requirements

* Mirth Connect v4.5.2
* Windows/Linux server
* PostgreSQL available at `db1:5432`
* Inbound firewall rule:

  * **TCP 4001** for HTTP POST
* Table requirements:

  * `rpm_orders`
  * `adt_messages`

---

## 13. Example Review-Type Request

```json
{
  "patient_id": "217057",
  "message_type": "Review",
  "url": "https://example.com/review.pdf",
  "reviewer": "Dr. Smith",
  "reviewed_at": "2025-03-12T05:30:22.047266+00:00"
}
```

---

## 14. Example PDF Upload Request

```json
{
  "patient_id": "217057",
  "message_type": "PDF",
  "pdf": "<base64_encoded_pdf_binary>",
  "rpm_order_id": 126
}
```
