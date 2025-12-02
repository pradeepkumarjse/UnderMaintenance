# VITAL_LISTNER – README

## 1. Overview

**Channel Name:** VITAL_LISTNER  
**Type:** TCP Listener
**Inbound Format:** JSON

**Purpose:** This channel receives **JSON arrays of vital-sign messages** over TCP, splits them, and forwards each individual JSON object to another Mirth channel named **Vital_Sender**.

It is a **router-only** channel (no database, no API calls, no transformations).

---

## 2. High-Level Workflow

```

Device / External System → TCP (port 8900, JSON array)
↓
TCP Listener receives JSON
↓
Preprocess: remove control characters
↓
Transformer: loop through each JSON object
↓
routeMessage("Vital_Sender", <JSON object>)
↓
Bypass destination (no-op)

````

---

## 3. Input Format

### The channel expects incoming **JSON array**, for example:

```json
[
  {
    "patient_id": "12345",
    "spo2": 98,
    "hr": 77,
    "rr": 18,
    "timestamp": "2025-03-12T10:20:11Z"
  },
  {
    "patient_id": "12345",
    "spo2": 97,
    "hr": 79,
    "rr": 20,
    "timestamp": "2025-03-12T10:20:25Z"
  }
]
````

Each object becomes **one routed message**.

---

## 4. TCP Listener Configuration

| Setting              | Value                           |
| -------------------- | ------------------------------- |
| Host                 | `0.0.0.0`                       |
| Port                 | **8900**                        |
| Keep Connection Open | Yes                             |
| Max Connections      | 10                              |
| Start/End Bytes      | None (Basic framing)            |
| Response             | Auto-generated after processing |

**Inbound Datatype:** JSON
**Outbound Datatype:** JSON

---

## 5. Source Transformer Logic

The main processor of the channel is this script:

```javascript
MessageLength = (msg.length);  

for (var i = 0; i < MessageLength; i++) {
    router.routeMessage('Vital_Sender', JSON.stringify(msg[i]));
}
```

### What this does:

✔ Counts number of JSON objects in the array
✔ Loops through each one
✔ Sends each JSON object individually to **Vital_Sender**
✔ Ensures downstream channels receive clean, one-record-per-message data

---

## 6. Preprocessing Script

Before the transformer runs, the channel executes:

```javascript
message = message.replace(/[\u0000-\u001F]+/g, "");
return message;
```

Purpose:

* Removes hidden control/escape characters
* Prevents JSON parse errors

---

## 7. Destination Connector

### Destination Name: `bypass`

**Type:** JavaScript Writer
**Script:**

```javascript
return 0;
```

This confirms:

✔ No transformation
✔ No external HTTP/TCP calls
✔ No file writes
✔ The channel acts as a pure **router**

All real work is done in the source transformer.

---

## 8. Message Storage

| Setting           | Value        |
| ----------------- | ------------ |
| Mode              | PRODUCTION   |
| Store Attachments | true         |
| Metadata Columns  | SOURCE, TYPE |

---

## 9. Downstream Channel Requirement

This channel depends on:

### **Vital_Sender**

Every incoming JSON object is forwarded to this channel.
You must ensure:

* Vital_Sender is **enabled**
* It expects **JSON input**
* It performs the final action (DB insert, API call, etc.)

---

## 10. Deployment Requirements

* Mirth Connect 4.5.2
* Open firewall for inbound TCP **8900**
* Device/system must send **valid JSON array**
* Downstream channel **Vital_Sender** must exist

---

## 11. Example End-to-End Flow

1. Device sends:

```
[{"spo2":96}, {"spo2":97}]
```

2. VITAL_LISTNER receives it
3. Preprocess removes invalid characters
4. Transformer loops:

```
routeMessage("Vital_Sender", {"spo2":96})
routeMessage("Vital_Sender", {"spo2":97})
```

5. Destination bypass does nothing
6. Vital_Sender handles actual processing
