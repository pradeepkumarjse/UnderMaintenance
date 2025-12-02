# Vital_Sender – README

## 1. Overview

**Channel Name:** Vital_Sender  

**Role:** Converts incoming **JSON vital sign messages** into **HL7 ORU messages** and sends them to **EPIC** over **MLLP (port 5128)**.

This channel is the second stage in the vitals pipeline:

**Purpose:**

- Receive JSON representing patient vitals  
- Look up patient & encounter metadata from PostgreSQL  
- Build HL7 ORU^R01 (MSH, PID, PV1, OBR, OBX segments)  
- Convert JSON vital parameters into multiple OBX segments  
- Send HL7 to Epic via TCP/MLLP

```

Device → VITAL_LISTNER → Vital_Sender → Epic (MLLP/HL7)

```

---

## 2. High-Level Workflow

```

SOURCE (Channel Reader, JSON)
↓
Transformer:

* Validate message
* Extract patient_id
* Query DB (adt_messages)
* Prepare PID, PV1 data
* Compute timestamps (UTC → EST)
* Create HL7 template (tmp)
* Build OBX segments dynamically
  ↓
  DESTINATION (MLLP Sender → Epic)
  ↓
  Epic receives HL7 ORU message

````

---

## 3. Input Format (From VITAL_LISTNER)

Vital_Sender receives **JSON array** messages, example:

```json
[
  {"tag": "Header", "patient_id": "12345", ...},
  {"tag": "Patient ID", "patient_id": "12345"},
  {"tag": "Name", "first_and_last_name": "JOHN DOE"},
  {"tag": "Patient info", "sex": "M", "birthday": "19800101", "floor_number": "4"},
  {"tag": "Data Time", "data_time": "2024-07-19T23:12:08.000Z"},
  {"tag": "... more fields ..."}
]
````

Vital_Sender extracts measurements like:

```
HR_measurement_value
SPO2_measured_value
NBP_S_measured_value
T1_measurement_value
RR_measured_value
```

---

## 4. Source Connector

| Setting                | Value                        |
| ---------------------- | ---------------------------- |
| Type                   | Channel Reader (VM Receiver) |
| Inbound Datatype       | JSON                         |
| Outbound Datatype      | JSON                         |
| respondAfterProcessing | true                         |

Vital_Sender is not externally callable — only triggered internally via `routeMessage()` by VITAL_LISTNER.

---

## 5. Transformer Logic (Core)

### 5.1 Extract & Validate Patient ID

```javascript
var raw = connectorMessage.getRawData();
if (raw != '[]') {
    var patient_id = msg[1]['patient_id'];
}
```

### 5.2 DB Lookup (adt_messages)

Queries the patient encounter record:

```sql
SELECT medical_record_number, epicencounterid, first_name, last_name, siteid
FROM adt_messages
WHERE encounterid = ?
ORDER BY id LIMIT 1
```

Populates:

* **MRN**
* **Epic Encounter ID**
* **First/Last Name**
* **Site ID**

If not found → filter = 1 (skip).

### 5.3 Timestamp Conversion

Incoming timestamps are UTC → converted to EST:

```javascript
var datetime = EST_time(UTC_now);
var obr_datetime = EST_with_timezone(JSON_timestamp);
```

### 5.4 HL7 Message Build

Vital_Sender fills values into the HL7 template (`tmp`):

#### MSH

```
MSH.7  = current datetime (EST)
MSH.10 = UUID
```

#### PID

* PID-2 = patient_id
* PID-3 = MRN
* PID-5 = Last / First name
* PID-7 = DOB
* PID-8 = Sex

#### PV1

* PV1-3.3 = floor_number + "-" + siteid
* PV1-19 = Epic encounter ID

#### OBR

* OBR.7 = Observation datetime (EST)
* OBR.3 = datetime

---

## 6. OBX Segment Creation (Core Functionality)

Vital_Sender dynamically creates OBX segments for every measurement matching:

```
*_measurement_value
*_measured_value
```

### 6.1 Measurement Mapping

Each measurement code maps to:

* MDC code (OBX.3)
* Unit (OBX.6)

Examples:

| JSON Key | OBX.3 (MDC)                       | Unit                        |
| -------- | --------------------------------- | --------------------------- |
| HR       | `147842^MDC_ECG_HEART_RATE^MDC`   | `^MDC_DIM_BEAT_PER_MIN^MDC` |
| SPO2     | `150456^MDC_PULS_OXIM_SAT_O2^MDC` | `^MDC_DIM_PERCENT^MDC`      |
| RR       | `151552^MDC_RESP^MDC`             | `^MDC_DIM_RESP_PER_MIN^MDC` |
| NBP_S    | `150020^NBP_S^MDC`                | `^MDC_DIM_MMHG^MDC`         |
| T1       | `150344^MDC_TEMP1^MDC`            | `^MDC_DIM_DEGC^MDC`         |

### 6.2 OBX creation loop

For each measurement:

```javascript
tmp['OBX'][i]['OBX.1.1'] = sequenceCounter
tmp['OBX'][i]['OBX.3.1'] = measurementMDC
tmp['OBX'][i]['OBX.5.1'] = measurement_value
tmp['OBX'][i]['OBX.6.1'] = unit
tmp['OBX'][i]['OBX.14.1'] = timestamp
```

All OBX segments are numbered sequentially.

---

## 7. Destination Connector — MLLP Sender

Vital_Sender sends HL7 to Epic:

| Setting          | Value                            |
| ---------------- | -------------------------------- |
| Type             | TCP Sender (MLLP)                |
| Remote Host      | `epic-tst.et0948.epichosted.com` |
| Port             | **5128**                         |
| MLLP Start Block | `0B`                             |
| MLLP End Block   | `1C0D`                           |
| ACK byte         | `06`                             |
| Timeout          | 60000 ms                         |

Payload:

```
${message.encodedData}
```

**Epic returns HL7 ACK/NACK**, validated by Mirth.

---

## 8. Filtering Logic

The channel sets:

```
filter = 1 → skip sending
```

Conditions for filtering:

* Missing patient_id
* Missing MRN
* Missing encounter record
* Invalid raw JSON (`[]`)

This prevents corrupt HL7 messages from being sent.

---

## 9. Database Requirements

### `adt_messages` table fields used:

* medical_record_number
* epicencounterid
* first_name
* last_name
* siteid
* encounterid

---

## 10. Deployment Requirements

* Mirth 4.5.2
* PostgreSQL (db1:5432)
* Open outbound TCP port **5128 → EPIC**

---

## 11. End-to-End Flow Example

```
DEVICE sends JSON →
VITAL_LISTNER splits JSON →
routes to Vital_Sender →
Vital_Sender builds HL7 ORU →
Sends to Epic →
Epic returns ACK →
Vital_Sender logs success
```
