# CS_SERACH_ID_QUERY – README

## 1. Overview

**Channel Name:** CS_SERACH_ID_QUERY  

**Purpose:**  
Receives XML messages over **TCP port 2806**, extracts **PTID (patient ID)**, queries a **PostgreSQL** database, and returns an XML response containing patient demographic details (name, sex, DOB, location).

---

## 2. Workflow

```

External System → TCP Listener (Port 2806)
→ Extract PTID from incoming XML
→ Query PostgreSQL (adt_messages table)
→ Build XML response (REP_PT_INFO)
→ Send back the response over same TCP connection

````

---

## 3. Input / Output

### 3.1 Input XML Format

Incoming message must be an XML similar to:

```xml
<REQ_PT_INFO>
    <PTID>12345</PTID>
</REQ_PT_INFO>
````

The channel extracts the value inside `<PTID>...</PTID>`.

---

### 3.2 Output XML Response

#### If patient found:

```xml
<REP_PT_INFO>
    <PTID>12345</PTID>
    <DATA_NO>1</DATA_NO>
    <DATA>
        <KANA>LastName FirstName</KANA>
        <SEX>M</SEX>
        <BIRTH>1990/12/01</BIRTH>
        <LOCATION>TOKYO</LOCATION>
    </DATA>
</REP_PT_INFO>
```

#### If patient NOT found:

```xml
<REP_PT_INFO>
    <PTID>12345</PTID>
    <DATA_NO>0</DATA_NO>
</REP_PT_INFO>
```

#### If DB error:

```xml
<REP_PT_INFO>
    <ERROR>Network Connection Lost</ERROR>
</REP_PT_INFO>
```

---

## 4. Configuration

### 4.1 Source Connector (Receiver)

* **Type:** TCP Listener
* **Host:** `0.0.0.0` (accepts all inbound connections)
* **Port:** `2806`
* **Respond After Processing:** Yes
* **DataType:** XML

Incoming XML is accepted and passed to the transformer.

---

### 4.2 Destination Connector

* **Name:** `resp`
* **Type:** JavaScript Writer
* **Function:** Build XML response and return it to TCP client.

#### Database Connection (PostgreSQL)

```javascript
dbConn = DatabaseConnectionFactory.createDatabaseConnection(
    'org.postgresql.Driver',
    'jdbc:postgresql://db1:5432/mirthdb',
    'mirthdb',
    'mirthdb'
);
```

---

## 5. Deployment Requirements

### 5.1 Server Requirements

* Windows Server with RDP access
* Mirth Connect installed (v4.5.2)
* PostgreSQL accessible on the network (`db1:5432`)
* Firewall rule allowing **Inbound TCP port 2806**
