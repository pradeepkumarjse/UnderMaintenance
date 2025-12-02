# RPM_Orders – README

## 1. Overview

**Channel Name:** RPM_Orders  
**Type:** TCP Listener (MLLP)  

**Purpose:** Receives HL7 ORM messages from Epic, extracts order identifiers (ORC/OBR/PID), stores and updates order information in PostgreSQL, builds JSON order payload for telemetry API, queues HL7 messages into filesystem, and posts/cancels orders to the remote API based on ORC.1 (Order Control Code).

This channel is part of the RPM workflow alongside:
- RPM_Orders_Queued_Process  
- GetPatient  
- FukudaAPI  
- PDF_Listener  

---

## 2. High-Level Workflow

```

Epic ORM → MLLP Port 2002
↓
HL7 Parsing (PID, ORC, OBR, PV1)
↓
Insert/Update rpm_orders table
↓
Calculate needToSend based on timestamp
↓
Generate JSON order payload
↓
Destination 1: destresp → Returns ACK
↓
Destination 2: SentToQueue → Saves HL7 file for queued processing
↓
If needToSend >= 0:
- GetToken → POST /api-token-auth/
- PostOrder → POST /api/orders/  (for NW)
- CancelOrder → DELETE /api/orders/delete/ (for CA/CO/OC)

````

---

## 3. HL7 Input Message Structure

Main HL7 segments used:

| Segment | Fields Used | Purpose |
|--------|-------------|---------|
| **PID** | PID.3.1 | MRN |
| **PV1** | PV1.50.1 | Encounter ID |
| **ORC** | ORC.1, ORC.2.1, ORC.7.4, ORC.8.1 | Order metadata |
| **OBR** | OBR.2.1 | EMR Order ID |
| **MSH** | MSH.4, MSH.10 | ACK generation |

Inbound datatype: **HL7 v2.x**

---

## 4. Core Logic (Source Transformer)

### 4.1 Extract key fields

Extracts:

- Encounter ID  
- MRN  
- patient_id (optional lookup)  
- EMR order ID  
- EMR child ID  
- Message timestamp (ORC.7)  
- Current timestamp (UTC)  

### 4.2 Calculate `needToSend`

```javascript
needToSend = currentTsInt - msgTsInt;
````

If **>= 0**, the order is due for processing.

### 4.3 Insert into *rpm_orders* if new

```
INSERT INTO rpm_orders (emr_child_id, whensend, order_emr_id, medical_record_number)
```

Only inserted if no existing matching emr_child_id + timestamp.

### 4.4 Update adt_messages table

```
UPDATE adt_messages 
SET emr_order_id=?, emr_id=?
WHERE medical_record_number=?
```

### 4.5 Cleanup old records

Deletes older than 30 days (720 hours):

```
DELETE FROM rpm_orders WHERE whenadded < NOW() - INTERVAL '720 hours';
```

---

## 5. Destination Connector: **destresp** (JavaScript Writer)

### Purpose:

* Returns ACK back to Epic
* Triggers API calls (Token, PostOrder, CancelOrder)

### ACK Generated:

```
MSH|^~\&|EpicSelect|<siteid>|Fukuda|<siteid>|<timestamp>||ACK|<msgctlID>|T|2.6
MSA|AA|<msgctlID>
```

Stored in:

```
channelMap.put('ACKMSG', ACKMSG);
```

Returned by script:

```
return $('ACKMSG');
```

---

## 6. API Integrations

All HTTPS calls use **custom SSL context that ignores certificates**.

### 6.1 GetToken

POST →

```
https://telemetry-tst.select.corp.sem/core/api-token-auth/
```

Payload:

```json
{
  "username": "XXXXXXXXX",
  "password": "XXXXXXXXX!"
}
```

Stores:

```
channelMap.put('token', token);
```

### 6.2 PostOrder (OrderControl == "NW")

POST →

```
https://telemetry-tst.select.corp.sem/core/api/orders/
```

Payload comes from:

```
channelMap['response_patientData']
```

Sample constructed payload:

```json
{
  "type": "Tele",
  "patient": 12345,
  "vital": 16,
  "status": "open",
  "emr_id": "<value>",
  "emr_child_id": "<value>"
}
```

### 6.3 CancelOrder (OrderControl IN CA, CO, OC)

DELETE →

```
/core/api/orders/delete/?emr_child_id=<id>&patient_id=<id>
```

Log responses for monitoring.

---

## 7. Destination Connector: **SentToQueue**

### Purpose

Saves inbound HL7 message to file system for the channel **RPM_Orders_Queued_Process** to transmit later.

### File Path

```
/opt/connect/appdata
```

### File Name Format

```
${emr_child_id}_${msgTs}.hl7
```

### Stored Content

```
${message.encodedData}
```

---

## 8. Order Timing Logic

* `msgTs` is the HL7 order timestamp
* `currentTs` is current UTC time
* If current >= HL7 timestamp → order is ready to send

This ensures orders with future timestamps aren’t posted prematurely.

---

## 9. Filters

### Destination 1 (destresp)

Runs only when:

```
needToSend >= 0
```

### Destination 2 (SentToQueue)

Runs when:

```
needToSend < 0
```

---

## 10. Database Requirements

### rpm_orders table

Must contain:

* emr_child_id
* whensend
* whenadded
* order_emr_id
* medical_record_number
* issent
* patient_id

### adt_messages table

Fields used:

* medical_record_number
* patient_id
* emr_order_id
* emr_id

---

## 11. Deployment Requirements

* Mirth Connect 4.5.2
* PostgreSQL at `db1:5432/mirthdb`
* Outbound HTTPS → telemetry-tst.select.corp.sem
* Inbound firewall allow: **TCP 2002**
* Folder `/opt/connect/appdata` writable
* JVM must allow custom SSL context

---

## 12. Summary of Channel Behavior

1. Receives HL7 ORM message
2. Parses MRN, timestamps, EMR IDs
3. Manages order queue state in DB
4. Generates JSON payload
5. Posts orders or cancels them via API
6. Returns HL7 ACK to Epic
7. Saves future-scheduled orders into filesystem
8. Immediate orders trigger API calls

This is the core engine for RPM order intake and dispatch.
