Convert Time Zones with TimeZoneDB API Integration

https://n8nworkflows.xyz/workflows/convert-time-zones-with-timezonedb-api-integration-4624


# Convert Time Zones with TimeZoneDB API Integration

### 1. Workflow Overview

This workflow provides an API endpoint to convert a Unix timestamp from one IANA time zone to another using the TimeZoneDB API. It is designed for scenarios where external clients need reliable and dynamic time zone conversions based on standardized identifiers and precise timestamps.

The workflow is logically divided into three main blocks:

- **1.1 Input Reception:** A webhook node listens for incoming HTTP POST requests containing JSON data specifying the source time zone, target time zone, and the Unix timestamp to convert.

- **1.2 Time Zone Conversion via TimeZoneDB API:** An HTTP Request node calls the TimeZoneDB API’s `convert-time-zone` endpoint, securely including the API key via n8n credentials, and dynamically passing user parameters.

- **1.3 Response Delivery:** A node sends the API conversion results back to the webhook caller, including the converted time details and status.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block handles incoming requests by exposing a webhook that accepts JSON payloads specifying the time zones and time to convert.

- **Nodes Involved:**  
  - Receive Time Conversion Request (Webhook)  
  - Note: Webhook Input (Sticky Note)

- **Node Details:**

  - **Receive Time Conversion Request**  
    - Type: Webhook  
    - Role: Entry point that listens for HTTP POST requests at `/convert-timezone`.  
    - Configuration:  
      - HTTP Method: POST  
      - Path: `convert-timezone`  
      - Response Mode: `responseNode` (response is deferred to a downstream node)  
    - Inputs: External HTTP client request  
    - Outputs: Passes incoming JSON body to the next node  
    - Edge Cases:  
      - Invalid or missing JSON fields (`fromZone`, `toZone`, `time`) could cause downstream errors.  
      - Unsupported HTTP methods will be rejected automatically by n8n.  
    - Sticky Note Content (Note: Webhook Input):  
      Specifies expected JSON properties:  
      - `fromZone`: source timezone IANA name (e.g., `America/New_York`)  
      - `toZone`: target timezone IANA name (e.g., `Europe/London`)  
      - `time`: Unix timestamp in seconds (e.g., `1678886400`)  
      Also highlights that the API key is secure and not part of the request body.

---

#### 1.2 Time Zone Conversion via TimeZoneDB API

- **Overview:**  
  This block performs the actual time zone conversion by making an authenticated HTTP GET request to TimeZoneDB’s `convert-time-zone` API endpoint, dynamically inserting parameters from the webhook request.

- **Nodes Involved:**  
  - Convert Timezone (TimeZoneDB) (HTTP Request)  
  - Note: TimeZoneDB API Call (Sticky Note)

- **Node Details:**

  - **Convert Timezone (TimeZoneDB)**  
    - Type: HTTP Request  
    - Role: Calls the TimeZoneDB API to convert the timestamp from one time zone to another.  
    - Configuration:  
      - HTTP Method: GET  
      - URL Template:  
        ```
        http://api.timezonedb.com/v2.1/convert-time-zone?format=json&from={{$json.body.fromZone}}&to={{$json.body.toZone}}&time={{$json.body.time}}
        ```  
      - Authentication: `HTTP Query Authentication` via n8n credentials (TimeZoneDB API key) — API key is appended as a query parameter securely.  
    - Inputs: JSON body from the webhook node containing `fromZone`, `toZone`, and `time`.  
    - Outputs: JSON response from TimeZoneDB API including converted time details and status.  
    - Edge Cases:  
      - Invalid or misspelled time zone names may cause API errors or empty responses.  
      - Incorrect Unix timestamp format or missing parameters.  
      - API key errors (expired, revoked, or invalid) will result in authentication failures.  
      - Network timeouts or HTTP errors from the API server.  
    - Sticky Note Content (Note: TimeZoneDB API Call):  
      Describes the secure use of credentials and dynamic URL construction.

---

#### 1.3 Response Delivery

- **Overview:**  
  This block returns the API conversion results back to the original client that called the webhook, ensuring the client receives the converted time and relevant metadata.

- **Nodes Involved:**  
  - Respond with Converted Time (Respond to Webhook)  
  - Note: Webhook Response (Sticky Note)

- **Node Details:**

  - **Respond with Converted Time**  
    - Type: Respond to Webhook  
    - Role: Sends the full JSON output from the TimeZoneDB API back as the HTTP response to the webhook caller.  
    - Configuration:  
      - Respond with: `allIncomingItems` (sends the entire data payload from the previous node)  
      - No additional transformations applied.  
    - Inputs: API response JSON from the HTTP Request node.  
    - Outputs: HTTP 200 response to the original request with the conversion data.  
    - Edge Cases:  
      - If the previous node failed or returned no data, the response may be empty or error-prone.  
      - Malformed API responses could cause response errors.  
    - Sticky Note Content (Note: Webhook Response):  
      Explains the content of the response: converted time, time zone names, and API status info.

---

### 3. Summary Table

| Node Name                   | Node Type               | Functional Role                     | Input Node(s)                 | Output Node(s)                | Sticky Note                                                                                                                          |
|-----------------------------|-------------------------|-----------------------------------|------------------------------|------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| Note: Webhook Input          | Sticky Note             | Documentation of webhook input    |                              |                              | ## Webhook Input  This node listens for incoming POST requests. It expects a JSON body with the following properties: ...           |
| Receive Time Conversion Request | Webhook                 | Accepts incoming webhook requests |                              | Convert Timezone (TimeZoneDB) |                                                                                                                                      |
| Note: TimeZoneDB API Call    | Sticky Note             | Documentation of API call logic   |                              |                              | ## TimeZoneDB API Call  This node makes an HTTP GET request to the TimeZoneDB API's `convert-time-zone` endpoint. ...                  |
| Convert Timezone (TimeZoneDB)| HTTP Request            | Calls TimeZoneDB API to convert time | Receive Time Conversion Request | Respond with Converted Time  |                                                                                                                                      |
| Note: Webhook Response       | Sticky Note             | Documentation of webhook response |                              |                              | ## Webhook Response  This node sends the conversion result received from the TimeZoneDB API back to the original webhook caller. ...  |
| Respond with Converted Time  | Respond to Webhook      | Sends conversion result back      | Convert Timezone (TimeZoneDB) |                              |                                                                                                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Sticky Note Node**  
   - Name: `Note: Webhook Input`  
   - Content: Document expected webhook input JSON with properties `fromZone`, `toZone`, and `time`.  
   - Position: Place near the webhook node for clarity.

2. **Create a Webhook Node**  
   - Name: `Receive Time Conversion Request`  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `convert-timezone`  
   - Response Mode: Select `Response Node` to defer response sending downstream.  
   - No credentials needed here.  
   - Position: After the webhook input note.

3. **Create a Sticky Note Node**  
   - Name: `Note: TimeZoneDB API Call`  
   - Content: Explain the HTTP Request node’s role, dynamic URL construction, and secure credential usage.  
   - Position: Near the HTTP Request node.

4. **Create an HTTP Request Node**  
   - Name: `Convert Timezone (TimeZoneDB)`  
   - HTTP Method: GET  
   - URL:  
     ```
     http://api.timezonedb.com/v2.1/convert-time-zone?format=json&from={{$json.body.fromZone}}&to={{$json.body.toZone}}&time={{$json.body.time}}
     ```  
   - Authentication: Select `HTTP Query Authentication`  
   - Credentials: Create or select existing credentials for TimeZoneDB API key (must be pre-configured in n8n’s credential manager). The API key will be appended as a query parameter automatically.  
   - Position: Connect `Receive Time Conversion Request` → `Convert Timezone (TimeZoneDB)`

5. **Create a Sticky Note Node**  
   - Name: `Note: Webhook Response`  
   - Content: Describe that this node sends the converted time and status back to the original caller.  
   - Position: Near respond node.

6. **Create a Respond to Webhook Node**  
   - Name: `Respond with Converted Time`  
   - Configure to respond with all incoming items (`allIncomingItems`).  
   - Connect from `Convert Timezone (TimeZoneDB)` node.  
   - Position: Last node in the chain.

7. **Connect the Nodes**  
   - `Receive Time Conversion Request` → `Convert Timezone (TimeZoneDB)` → `Respond with Converted Time`

8. **Verify Credentials**  
   - Ensure TimeZoneDB API key credential exists and is valid in n8n.  
   - Test API key separately if needed.

9. **Activate Workflow**  
   - Save and activate the workflow.  
   - Test by sending a POST request with JSON body containing `fromZone`, `toZone`, and `time` fields to the webhook URL.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                             | Context or Link                                                                                                  |
|----------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| The TimeZoneDB API key is securely managed via n8n credentials and should never be exposed in webhook payloads or URL parameters directly.               | Security best practice                                                                                          |
| Time zone names must conform to IANA time zone database names (e.g., `America/New_York`, `Europe/London`). Incorrect names will cause API errors.          | API documentation: https://timezonedb.com/api                                                                 |
| Unix timestamps are expected in **seconds**, not milliseconds.                                                                                        | Clarifies timestamp format                                                                                      |
| For testing, tools like Postman or curl can be used to POST JSON data to the webhook endpoint.                                                          | External testing utilities                                                                                      |
| Workflow structure supports easy extension for additional time zone-related functionalities, such as listing zones or validating inputs.                 | Potential workflow enhancement                                                                                  |

---

**Disclaimer:** The provided content is derived solely from an automated workflow built with n8n, adhering strictly to content policies and containing no illegal or offensive elements. All data handled is legal and publicly accessible.