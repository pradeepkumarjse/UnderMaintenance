# RPM_Orders_Queued_Process – README

## 1. Overview

**Channel Name:** RPM_Orders_Queued_Process  
**Source:** JavaScript Reader (polling every 60 seconds)  

**Purpose:** This channel checks for queued RPM orders in the PostgreSQL database, identifies HL7 files scheduled to be sent, reads the HL7 message from disk, transmits it, and deletes the file upon successful ACK (MSA|AA).

---

## 2. High-Level Workflow

```

Every 60 seconds →
JS Reader polls DB →
If queued order exists →
Get filename →
Read HL7 file from disk →
Send HL7 over TCP (MLLP) to port 2002 →
Wait for ACK →
If ACK contains "MSA|AA|" → delete file

````

---

## 3. Detailed Processing Steps

### 3.1 Polling (Source Connector)

- **Connector Type:** JavaScript Reader  
- **Polling Frequency:** Every 60 seconds  
- **Script Behavior:**
  - Connects to PostgreSQL: `jdbc:postgresql://db1:5432/mirthdb`
  - Queries:

    ```sql
    select concat(emr_child_id,'_',whensend,'.hl7') as filename
    from rpm_orders
    where issent = 0
      and whensend < CURRENT_TIMESTAMP
    order by whensend
    limit 1;
    ```

  - If a file is found:
    - `filter = 0`
    - `filename = "<emr_child_id>_<timestamp>.hl7"`
  - If none found:
    - `filter = 1` (destination is skipped)

### Mapped Variables:

| Variable | Description |
|---------|-------------|
| `filename` | Name of HL7 file to send |
| `filter` | 0 → send message; 1 → skip |

---

## 4. Filter Rule

Destination executes **only when**:

````

$('filter') == 0

```

This prevents unnecessary TCP dispatching when there are no pending orders.

---

## 5. Destination Connector (TCP Sender)

### 5.1 TCP Configuration

| Setting | Value |
|--------|--------|
| Remote IP | `127.0.0.1` |
| Remote Port | `2002` |
| Mode | MLLP |
| Start Bytes | 0B |
| End Bytes | 1C0D |
| ACK Byte | 06 |
| NACK Byte | 15 |
| Response Timeout | 5000 ms |

### 5.2 Read HL7 File

File path constructed as:

```

/opt/connect/appdata/${filename}

```

Then:

- Reads HL7 content using `FileUtil.read()`
- Puts result into `hl7Message` (used as TCP template payload)

### 5.3 Template Sent Over TCP

```

${hl7Message}

```

---

## 6. Response Handling (MLLP ACK Logic)

After sending:

1. Mirth receives ACK from remote system.
2. If ACK text contains:

```

MSA|AA|

````

Then:

- Delete HL7 file:

```javascript
if (file.delete()) {
    logger.info("File deleted successfully");
}
````

Else:

* File is **not deleted**
* Warning is logged: *"TCP Response does not contain MSA|AA|"*

---

## 7. File Processing Directory

**All HL7 files must be located at:**

```
/opt/connect/appdata/
```

Example filename:

```
12345_20250304123010.hl7
```

---

## 8. Database Requirements

### 8.1 Table: `rpm_orders`

Fields used:

| Column         | Purpose                         |
| -------------- | ------------------------------- |
| `emr_child_id` | Used to build filename          |
| `whensend`     | Timestamp to determine ordering |
| `issent`       | 0 = pending, 1 = sent           |

The channel chooses the **oldest** pending order each poll cycle.

---

## 9. Deployment Requirements

* Mirth Connect 4.5.2
* PostgreSQL reachable at `db1:5432`
* Directory `/opt/connect/appdata/` accessible with read/write permissions
* Local TCP server running on **127.0.0.1:2002** (MLLP listener)
* Linux file system permissions must allow file deletion

---

## 10. Logging

The channel logs:

* Database errors
* File read attempts
* TCP dispatch
* ACK responses
* File deletion result

Log examples:

```
Sending HL7 message to rmp_order: <HL7>
File deleted successfully: /opt/connect/appdata/12345_20250304123010.hl7
TCP Response does not contain 'MSA|AA|'
```

---

## 11. Summary
This channel provides automated, scheduled dispatch of HL7 messages via MLLP based on queuing logic stored in a PostgreSQL table. It safely polls, filters, transmits, and deletes files only after successful delivery.
