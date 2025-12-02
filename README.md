# TCPSENDER – README

## 1. Overview

**Channel Name:** TCPSENDER  

**Purpose:**  This is a simple utility channel that receives messages internally (via Channel Reader / VM Receiver) and forwards them to an external HTTP endpoint using an HTTP Sender connector.

**Typical Use Cases:**
- Testing outbound HTTP APIs  
- Forwarding messages from other channels  
- Triggered programmatically using `Channel Writer` from another channel  

**Message Flow Source → Destination**
```

Internal Channel Writer → TCPSENDER → HTTP Sender → http://10.254.212.219:4001

````

---

## 2. Workflow Summary

1. **Source Connector:**  
   - Channel Reader (VM Receiver)  
   - Accepts RAW data from other channels  

2. **Destination Connector:**  
   - HTTP Sender (POST request)  
   - Sends message to configured server with optional query parameters  

3. **Response:**  
   - No validation  
   - No transformation  
   - No filtering  
   - Response returned to calling channel (if used with Channel Writer)

---

## 3. Input / Output

### 3.1 Input Format
- **RAW message content**  
- This channel does **not** parse HL7, JSON, XML — it sends whatever data is received.

### 3.2 Output
The outbound HTTP POST sends the **raw input content** as the HTTP body.

`Content-Type: text/plain`

---

## 4. Source Connector Configuration

**Type:** Channel Reader (VM Receiver)  
**Inbound Datatype:** RAW  
**Outbound Datatype:** RAW  

**Important Properties**
- `respondAfterProcessing = true` → calling channel waits for HTTP response  
- No filter or transformer  
- No batch processing  

This means the channel is used like:

```javascript
var resp = ChannelUtil.sendMessage('TCPSENDER', 'Hello world');
````

---

## 5. Destination Connector – HTTP Sender

### 5.1 Configuration

| Setting               | Value                                          |
| --------------------- | ---------------------------------------------- |
| **Transport**         | HTTP Sender                                    |
| **Method**            | POST                                           |
| **URL**               | `http://10.254.212.219:4001?encounterid=72654` |
| **Headers**           | None                                           |
| **Parameters**        | None                                           |
| **Content-Type**      | text/plain                                     |
| **Charset**           | UTF-8                                          |
| **Use Proxy**         | No                                             |
| **Retry Count**       | 0                                              |
| **Timeout**           | 30000 ms                                       |
| **Process Multipart** | true                                           |
| **Store Attachments** | false                                          |

### 5.2 Payload

The **entire incoming RAW message** becomes the HTTP body:

```
<incoming data>
```

### 5.3 Response Handling

* Response is returned to the caller
* No validation
* Response metadata is not included

---

## 6. Scripts

All scripts are default/no-op:

| Script Type     | Purpose           | Status     |
| --------------- | ----------------- | ---------- |
| Preprocessor    | Before source     | No changes |
| Postprocessor   | After destination | No changes |
| Deploy Script   | On channel deploy | Empty      |
| Undeploy Script | On channel stop   | Empty      |

---

## 7. Code Templates / Libraries

The channel includes references to large Mirth code template libraries (CDA, HL7 DT, custom functions), but:

**None of these templates are used by this channel.**

They only appear because the libraries are globally applied.

---

## 8. Message Storage & Metadata

* **Storage Mode:** DEVELOPMENT
* **Attachments:** stored
* **Metadata Columns:**

  * SOURCE → `mirth_source`
  * TYPE → `mirth_type`

---

## 9. Deployment Requirements

* Mirth Connect 4.5.2
* Outbound HTTP connectivity to
  `10.254.212.219:4001`
* No SSL (plain HTTP)
* No authentication required

---

## 10. Testing the Channel

### 10.1 Manual Test Using Channel Writer

```javascript
var resp = ChannelUtil.sendMessage("TCPSENDER", "Test message");
logger.info(resp);
```

### 10.2 Expected Behavior

* Channel receives `"Test message"`
* Sends POST:

```
POST /?encounterid=72654
Host: 10.254.212.219:4001
Content-Type: text/plain
Content-Length: 12

Test message
```

* Returns any HTTP response back to caller.

---

## 11. Summary

The **TCPSENDER** channel is a lightweight, utility-style channel used to forward RAW data to an HTTP API. It includes:

* No filters
* No transformers
* No message parsing
* Simple RAW → HTTP passthrough

It is primarily designed for:

* Forwarding messages via Mirth pipelines
* Integrating with lightweight HTTP endpoints
* Prototyping / testing outbound calls
