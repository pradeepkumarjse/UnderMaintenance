
# PurgeMirthDB – README

## 1. Overview

**Channel Name:** PurgeMirthDB  
**Type:** JavaScript Polling Channel  

**Purpose:** This channel automatically performs **PostgreSQL VACUUM FULL VERBOSE ANALYZE** on internal Mirth Connect database tables to reclaim disk space, remove bloat, and maintain optimal performance.

It is designed to run on a schedule **twice per day (every 12 hours)**.

---

## 2. High-Level Workflow

```

Scheduler (every 12 hours)
↓
Run JavaScript via Source Transformer
↓
Open PostgreSQL connection
↓
Execute:
VACUUM FULL VERBOSE ANALYZE d_mc6;
VACUUM FULL VERBOSE ANALYZE d_mc2;
↓
Log output / catch errors
↓
Close DB connection
↓
End

````

---

## 3. Polling Configuration

### 3.1 Poll Type  
**Interval-based polling**

| Setting | Value |
|--------|-------|
| Poll Type | INTERVAL |
| Frequency | 43,200,000 ms |
| Equivalent | **12 hours** |
| Poll on Start | true |
| Weekly advanced flags | Disabled |

This ensures the maintenance routine runs twice per day.

---

## 4. Database Maintenance Logic

Inside the JavaScript Source Transformer, the following sequence is executed:

### 4.1 Connect to PostgreSQL

```javascript
dbConn = DatabaseConnectionFactory.createDatabaseConnection(
  'org.postgresql.Driver',
  'jdbc:postgresql://db1:5432/mirthdb',
  'mirthdb',
  'mirthdb'
);
````

If connection is successful, it proceeds to maintenance operations.

---

### 4.2 Execution of VACUUM FULL Commands

```javascript
var updateQuery1 = " VACUUM FULL VERBOSE ANALYZE d_mc6;";
dbConn.executeUpdate(updateQuery1);

var updateQuery2 = " VACUUM FULL VERBOSE ANALYZE d_mc2;";
dbConn.executeUpdate(updateQuery2);
```

These target **two internal Mirth tables**:

* `d_mc6`
* `d_mc2`

Performing:

* VACUUM FULL → reclaims unused space
* VERBOSE → logs detailed output
* ANALYZE → updates planner statistics

---

## 5. Error Handling

All database exceptions are logged:

```javascript
logger.error("Database query failed. Error: " + e.name + " - " + e.message);
logger.error("Stack Trace: " + e.stack);
```

Closing the connection is guaranteed via `finally`:

```javascript
if (dbConn) dbConn.close();
```

---

## 6. Destination Behavior

The channel has a **single destination**, named `dest1`, containing:

```javascript
return 0;
```

This indicates the destination is not used for external dispatch; the maintenance task happens entirely in the Source Transformer.

---

## 7. Channel Settings Summary

| Component             | Value               |
| --------------------- | ------------------- |
| Source Connector      | JavaScript Reader   |
| Trigger               | Poll every 12 hours |
| Inbound/Outbound Type | RAW                 |
| Message Storage       | Production Mode     |
| Attachments           | Disabled            |
| Global Map Clearing   | Enabled             |

All scripts for preprocess / postprocess / deploy / undeploy are empty, ensuring no side effects.

---

## 8. Why This Channel Is Important

This channel performs essential DB maintenance:

* Prevents Mirth DB from growing indefinitely
* Eliminates table bloat
* Improves query performance
* Reduces disk usage
* Prevents server slowdowns or outages

Ideal for production environments where message volume is high.

---

## 9. Deployment Requirements

* Mirth Connect **v4.5.2**
* PostgreSQL server running on:
  `db1:5432` (database: `mirthdb`)
* `VACUUM FULL` permissions for the user `mirthdb`
* Channel must remain **Enabled** and **Started**
* No outbound firewall requirements
* Should be run on servers with low load during maintenance windows
