# FukudaAPI – README

## 1. Overview

**Channel Name:** FukudaAPI  
**Protocol:** TCP Listener (HL7 over MLLP)  

**Purpose:** This channel receives **HL7 ADT messages** from Epic over **MLLP**, extracts patient demographic and visit data, stores/updates the information in a PostgreSQL database, and returns a proper **HL7 ACK** message.

---

## 2. Workflow

```

Epic System
↓  (HL7 ADT via MLLP on port 2003)
TCP Listener (MLLP)
↓ Parse HL7 fields (PID, PV1, MSH, EVN)
↓ Construct patient & encounter data
↓ Insert or update adt_messages table
↓ Build HL7 ACK (AA)
↓ Return ACK to Epic

```

---

## 3. Input / Output

### 3.1 Input (HL7 ADT Message)

The channel receives a standard HL7 ADT message such as:

```

MSH|^~&|Epic|HOSP1|Fukuda|HOSP1|20250305114347||ADT^A01|66739|T|2.6
EVN|A01
PID|1||123456^^^MRN||Doe^John||19900909|M|||123 Main ST^^City||555-1234
PV1|1|I|WARD^ROOM^BED^BLDG^HOSP1||...|...|...|...|...|123456789|...

```

The transformer extracts:

| HL7 Field | Extracted Value |
|----------|------------------|
| PID.3 | MRN |
| PV1.19 | Epic Encounter ID |
| PV1.50 | Encounter ID (CSN) |
| PID.5 | First/Last Name |
| PID.7 | DOB |
| PID.8 | Gender |
| PV1.3 | Location string |
| PID.13 | Phone |
| PID.11 | Address |
| PID.16 | Marital Status |

---

### 3.2 Output (HL7 ACK Message)

The channel generates a dynamic HL7 ACK:

```

MSH|^~&|EpicSelect|<siteid>|Fukuda|<siteid>|<timestamp>||ACK|<msgcontrolid>|T|2.6
MSA|AA|<msgcontrolid>

````

The ACK is returned over the same MLLP connection.

---

## 4. HL7 Field Extraction Logic

The source transformer extracts and processes HL7 segments:

- **MSH.4** → site ID  
- **MSH.10** → message control ID  
- **PV1 fields** → encounter IDs, patient location, etc.  
- **PID fields** → MRN, name, DOB, gender, SSN, address, phone  
- **EVN.1** → message type (A01, A03, A08, etc.)

DOB is reformatted via a helper function:

```javascript
function formatDateOfBirth(rawDate) {
    if (!rawDate || rawDate.length !== 8) return null;
    var inputDate = new Date(Date.UTC(rawDate.substring(0,4), rawDate.substring(4,6)-1, rawDate.substring(6,8)));
    return new java.sql.Date(inputDate.getTime());
}
````

---

## 5. Database Operations

### 5.1 PostgreSQL Connection

```javascript
dbConn = DatabaseConnectionFactory.createDatabaseConnection(
    'org.postgresql.Driver',
    'jdbc:postgresql://db1:5432/mirthdb',
    'mirthdb',
    'mirthdb'
);
```

### 5.2 Table Updated

**Table:** `adt_messages`
Stores: demographic + PV1 + MSH fields.

### 5.3 Logic

1. Check if encounter already exists:

```sql
SELECT COUNT(encounterid) FROM adt_messages WHERE encounterid = ?
```

2. If **exists → UPDATE**

```sql
UPDATE adt_messages
SET siteid=?, pv1=?, marital_status=?, phone_number=?, address1=?, 
    medical_record_number=?, first_name=?, last_name=?, date_of_birth=?, 
    gender=?, height=?, weight=?, message=?, ssn=?, msgtype=?, epicencounterid=?
WHERE encounterid = ?
```

If location exists → update location:

```sql
UPDATE adt_messages SET location=? WHERE encounterid=?
```

3. If **not exists → INSERT**

```sql
INSERT INTO adt_messages (
    siteid, marital_status, phone_number, address1, medical_record_number,
    first_name, last_name, date_of_birth, gender, height, weight, message,
    ssn, encounterid, msgtype, epicencounterid, pv1
) VALUES (?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?)
```

All errors are logged but not returned to Epic.

---

## 6. ACK Message Construction

Example ACK:

```
MSH|^~\&|EpicSelect|HOSP1|Fukuda|HOSP1|20250305115000||ACK|MSG0001|T|2.6
MSA|AA|MSG0001
```

Created using:

```javascript
channelMap.put('ACKMSG', ACKMSG);
```

This ACK is the outbound data returned to Epic.

---

## 7. Destination Connector

### Name: `dest1`

### Type: JavaScript Writer

### Purpose: Return **ACK** stored in:

```javascript
return $('ACKMSG');
```

---

## 8. Transport Details

### TCP Listener Settings

| Setting              | Value   |
| -------------------- | ------- |
| Host                 | 0.0.0.0 |
| Port                 | 2003    |
| MLLP Start Block     | 0B      |
| End Block            | 1C0D    |
| ACK                  | 06      |
| NACK                 | 15      |
| Keep Connection Open | true    |
| Max Connections      | 100     |

Data type: **HL7 v2.x** (ADT)

---

## 9. Deployment Requirements

* Windows/Linux server with Mirth Connect installed
* Mirth Connect **v4.5.2**
* PostgreSQL reachable at `db1:5432`
* Required firewall rule:

  * **Inbound TCP 2003**
* Ensure the `adt_messages` table exists with appropriate columns

---

## 10. Example Flow Summary

1. Epic sends ADT^A01/A03/A08 message over MLLP → port 2003
2. Channel extracts all patient + encounter fields
3. Inserts/updates record in `adt_messages`
4. Builds HL7 ACK
5. Returns ACK back to Epic

---

## 11. Notes
* Ensure Epic uses **MLLP framing** (0x0B ... 0x1C0D)
* Channel is already set to auto-ACK with custom ACK
* All database errors log to Mirth but do NOT break ACK generation
* Important fields stored:

  * MRN, CSN, Epic Encounter ID
  * Names, DOB, Gender
  * Location (PV1.3)
  * MSH.4 site ID

