Parse documents in real time via webhook with Parseur

https://n8nworkflows.xyz/workflows/parse-documents-in-real-time-via-webhook-with-parseur-17631


# Parse documents in real time via webhook with Parseur

### 1. Workflow Overview

This workflow provides a synchronous API endpoint to receive, process, and return parsed document data in real-time. It accepts documents via a secure webhook, uploads them to Parseur for automated data extraction, polls the service until parsing completes or times out, and responds to the original caller with the structured extraction results or an appropriate error code.

The workflow logic is categorized into the following functional blocks:
- **1.1 Input Reception & Upload:** Captures incoming HTTP POST requests containing binary document files and uploads them to Parseur using designated credentials, returning an immediate upload failure response if the transfer fails.
- **1.2 Polling Initialization:** Introduces a brief initial delay and establishes a tracking variable (`retryCount`) to prepare for status querying.
- **1.3 Status Polling & Decision Routing:** Periodically queries the Parseur Document API to evaluate whether document parsing is complete (`PARSEDOK`), routing successful parses to the final response node and managing connection faults.
- **1.4 Retry & Timeout Management:** Manages iterative polling attempts up to a defined threshold (10 attempts), enforcing wait intervals between checks and generating timeout errors if processing exceeds the limit.
- **1.5 Counter Increment Loopback:** Updates the retry counter variable and loops execution back to the status check node.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Upload
**Overview:** This block establishes the webhook entry point for incoming documents, initiates the Parseur upload process, and handles early file-level exceptions.

- **Nodes Involved:**
  - `When Document Requested`
  - `Upload to Parseur`
  - `Respond with Upload Error`

- **Node Details:**
  - **When Document Requested**
    - *Type and technical role:* Webhook trigger (`n8n-nodes-base.webhook`) receiving external POST requests.
    - *Configuration choices:* Path set to `parse-document`, HTTP method set to `POST`, response mode delegated to a downstream response node, authentication enabled via header auth, binary property name configured as `data`.
    - *Key expressions or variables:* None.
    - *Input and output connections:* Input: External HTTP caller. Output: `Upload to Parseur`.
    - *Edge cases / Failures:* Unauthorized requests or payloads missing binary property `data`.
  - **Upload to Parseur**
    - *Type and technical role:* Parseur integration node (`n8n-nodes-parseur.parseur`) for sending files to parsers.
    - *Configuration choices:* Parser ID set to `202635`, error output handling set to continue on error (`continueErrorOutput`).
    - *Key expressions or variables:* Default binary handling for uploaded data.
    - *Input and output connections:* Input: `When Document Requested`. Outputs: `Wait 2 Seconds` (main branch), `Respond with Upload Error` (error branch).
    - *Edge cases / Failures:* Invalid file format or rejected API tokens will trigger the error path.
  - **Respond with Upload Error**
    - *Type and technical role:* Webhook response node (`n8n-nodes-base.respondToWebhook`) returning HTTP 400.
    - *Configuration choices:* Response code `400`, JSON response body containing an error message (`"Could not process the uploaded file..."`).
    - *Key expressions or variables:* Static JSON payload.
    - *Input and output connections:* Input: `Upload to Parseur` (error branch). Output: Terminal node.
    - *Edge cases / Failures:* None.

---

#### 1.2 Polling Initialization
**Overview:** Pauses execution briefly to allow Parseur to register the document, then sets an initial retry counter variable.

- **Nodes Involved:**
  - `Wait 2 Seconds`
  - `Prepare Retry Counter`

- **Node Details:**
  - **Wait 2 Seconds**
    - *Type and technical role:* Wait node (`n8n-nodes-base.wait`) introducing a fixed time delay.
    - *Configuration choices:* Amount set to `2` seconds.
    - *Key expressions or variables:* None.
    - *Input and output connections:* Input: `Upload to Parseur`. Output: `Prepare Retry Counter`.
    - *Edge cases / Failures:* None.
  - **Prepare Retry Counter**
    - *Type and technical role:* Set node (`n8n-nodes-base.set`) initializing control variables.
    - *Configuration choices:* Assigns `retryCount` as a number with value `0`, retains other input fields.
    - *Key expressions or variables:* None.
    - *Input and output connections:* Input: `Wait 2 Seconds`. Output: `Fetch Parsing Status`.
    - *Edge cases / Failures:* None.

---

#### 1.3 Status Polling & Decision Routing
**Overview:** Queries the Parseur API to check document status, routing successful extractions to the final response and capturing upstream communication failures.

- **Nodes Involved:**
  - `Fetch Parsing Status`
  - `Check Parsing Complete`
  - `Check Parsing Success`
  - `Respond with Success`
  - `Respond with API Error`

- **Node Details:**
  - **Fetch Parsing Status**
    - *Type and technical role:* HTTP Request node (`n8n-nodes-base.httpRequest`) polling the Parseur document endpoint.
    - *Configuration choices:* Generic HTTP Header authentication, error output handling set to continue on error (`continueErrorOutput`).
    - *Key expressions or variables:* URL uses `={{ 'https://api.parseur.com/document/' + $('Upload to Parseur').item.json.response.attachments[0].DocumentID }}`.
    - *Input and output connections:* Inputs: `Prepare Retry Counter` or `Increment Retry Counter`. Outputs: `Check Parsing Complete` (main branch), `Respond with API Error` (error branch).
    - *Edge cases / Failures:* Network dropouts, rate limiting, or invalid Document IDs trigger the error output.
  - **Respond with API Error**
    - *Type and technical role:* Webhook response node (`n8n-nodes-base.respondToWebhook`) returning HTTP 503.
    - *Configuration choices:* Response code `503`, JSON response body indicating service unavailability.
    - *Key expressions or variables:* Static JSON payload.
    - *Input and output connections:* Input: `Fetch Parsing Status` (error branch). Output: Terminal node.
    - *Edge cases / Failures:* None.
  - **Check Parsing Complete**
    - *Type and technical role:* IF condition node (`n8n-nodes-base.if`) evaluating if parsing is unfinished and retries remain.
    - *Configuration choices:* Loose type validation; checks if status is not equal to `PARSEDOK` and retry count is less than `10`.
    - *Key expressions or variables:* Status condition checks `={{ $json.status }}`, limit check evaluates `={{ $('Prepare Retry Counter').first().json.retryCount }}`.
    - *Input and output connections:* Input: `Fetch Parsing Status`. Outputs: `Check Parsing Success` (true branch), `Respond with Success` (false branch).
    - *Edge cases / Failures:* Unexpected status payload formats.
  - **Check Parsing Success**
    - *Type and technical role:* IF condition node (`n8n-nodes-base.if`) determining whether to continue polling or abort due to timeout.
    - *Configuration choices:* Strict string comparison verifying if status equals `PARSEDOK`.
    - *Key expressions or variables:* `={{ $json.status }}`.
    - *Input and output connections:* Input: `Check Parsing Complete` (true branch). Outputs: `Wait for Retry` (true branch), `Respond with Timeout Error` (false branch).
    - *Edge cases / Failures:* None.
  - **Respond with Success**
    - *Type and technical role:* Webhook response node (`n8n-nodes-base.respondToWebhook`) returning HTTP 200 with extracted data.
    - *Configuration choices:* Responds with JSON.
    - *Key expressions or variables:* `={{ JSON.parse($json.result) }}`.
    - *Input and output connections:* Input: `Check Parsing Complete` (false branch). Output: Terminal node.
    - *Edge cases / Failures:* Malformed JSON strings in Parseur results.

---

#### 1.4 Retry & Timeout Management
**Overview:** Enforces wait times between polling iterations and handles threshold breaches by returning a timeout error response.

- **Nodes Involved:**
  - `Wait for Retry`
  - `Respond with Timeout Error`

- **Node Details:**
  - **Wait for Retry**
    - *Type and technical role:* Wait node (`n8n-nodes-base.wait`) delaying subsequent poll requests.
    - *Configuration choices:* Default wait configuration (resumed via webhook/internal timer).
    - *Key expressions or variables:* None.
    - *Input and output connections:* Input: `Check Parsing Success` (true branch). Output: `Increment Retry Counter`.
    - *Edge cases / Failures:* None.
  - **Respond with Timeout Error**
    - *Type and technical role:* Webhook response node (`n8n-nodes-base.respondToWebhook`) returning HTTP 408.
    - *Configuration choices:* Response code `408`, JSON response body indicating a timeout.
    - *Key expressions or variables:* Static JSON payload.
    - *Input and output connections:* Input: `Check Parsing Success` (false branch). Output: Terminal node.
    - *Edge cases / Failures:* None.

---

#### 1.5 Counter Increment Loopback
**Overview:** Increments the retry counter variable and routes execution back to the status check step.

- **Nodes Involved:**
  - `Increment Retry Counter`

- **Node Details:**
  - **Increment Retry Counter**
    - *Type and technical role:* Set node (`n8n-nodes-base.set`) updating loop iteration metrics.
    - *Configuration choices:* Assigns `retryCount` as a number by incrementing the previous value.
    - *Key expressions or variables:* `=={{ $('Prepare Retry Counter').first().json.retryCount + 1 }}`.
    - *Input and output connections:* Input: `Wait for Retry`. Output: `Fetch Parsing Status`.
    - *Edge cases / Failures:* Accumulation errors if scope references break.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Sticky Note` | `n8n-nodes-base.stickyNote` | Documentation placeholder | None | None | Real-Time Parse-and-Respond<br><br>### How it works<br><br>This workflow receives a document through a webhook, uploads it to Parseur, then polls Parseur until parsing completes or the retry limit is reached. It returns a real-time webhook response with the parsed result, an upload/API error, or a timeout error if parsing does not finish in time.<br><br>### Setup steps<br><br>- Configure the webhook trigger URL and ensure incoming requests include the document payload expected by the Parseur upload node.<br>- Connect valid Parseur credentials and select the correct mailbox/template settings for document upload and status checks.<br>- Verify the HTTP status-check request uses the correct Parseur document endpoint and authentication headers/API token.<br>- Review the wait durations, retry counter field, and maximum retry condition so the workflow fits the expected Parseur processing time.<br><br>### Customization<br><br>Adjust the initial wait, retry delay, and maximum retry count to balance response speed against Parseur processing latency. Customize the success, timeout, upload-error, and API-error webhook responses to match the calling application's expected schema. |
| `Sticky Note1` | `n8n-nodes-base.stickyNote` | Documentation placeholder | None | None | Receive and upload document<br><br>Handles the incoming webhook request, uploads the received document to Parseur, and returns an immediate upload error response if the upload fails. |
| `Sticky Note2` | `n8n-nodes-base.stickyNote` | Documentation placeholder | None | None | Prepare first status check<br><br>Waits briefly after a successful upload and initializes the retry counter before polling Parseur for the first time. |
| `Sticky Note3` | `n8n-nodes-base.stickyNote` | Documentation placeholder | None | None | Poll and route status<br><br>Checks the Parseur document status, routes successful parsing to the success response, and sends a dedicated response if the status API call fails. |
| `Sticky Note4` | `n8n-nodes-base.stickyNote` | Documentation placeholder | None | None | Retry or timeout decision<br><br>Verifies whether polling should continue, waits before the next retry when appropriate, or returns a timeout response when parsing has not completed within the allowed attempts. |
| `Sticky Note5` | `n8n-nodes-base.stickyNote` | Documentation placeholder | None | None | Increment retry counter<br><br>Updates the retry count in an isolated loopback step before sending execution back to the Parseur status check. |
| `Respond with API Error` | `n8n-nodes-base.respondToWebhook` | Returns HTTP 503 service unavailable response | `Fetch Parsing Status` | None | Poll and route status<br><br>Checks the Parseur document status, routes successful parsing to the success response, and sends a dedicated response if the status API call fails. |
| `Fetch Parsing Status` | `n8n-nodes-base.httpRequest` | Polls the Parseur Document API | `Prepare Retry Counter`, `Increment Retry Counter` | `Check Parsing Complete`, `Respond with API Error` | Poll and route status<br><br>Checks the Parseur document status, routes successful parsing to the success response, and sends a dedicated response if the status API call fails. |
| `Respond with Upload Error` | `n8n-nodes-base.respondToWebhook` | Returns HTTP 400 upload failure response | `Upload to Parseur` | None | Receive and upload document<br><br>Handles the incoming webhook request, uploads the received document to Parseur, and returns an immediate upload error response if the upload fails. |
| `When Document Requested` | `n8n-nodes-base.webhook` | Receives incoming document webhook | None | `Upload to Parseur` | Receive and upload document<br><br>Handles the incoming webhook request, uploads the received document to Parseur, and returns an immediate upload error response if the upload fails. |
| `Prepare Retry Counter` | `n8n-nodes-base.set` | Initializes retryCount to 0 | `Wait 2 Seconds` | `Fetch Parsing Status` | Prepare first status check<br><br>Waits briefly after a successful upload and initializes the retry counter before polling Parseur for the first time. |
| `Respond with Timeout Error` | `n8n-nodes-base.respondToWebhook` | Returns HTTP 408 timeout error response | `Check Parsing Success` | None | Retry or timeout decision<br><br>Verifies whether polling should continue, waits before the next retry when appropriate, or returns a timeout response when parsing has not completed within the allowed attempts. |
| `Respond with Success` | `n8n-nodes-base.respondToWebhook` | Returns parsed JSON data with HTTP 200 | `Check Parsing Complete` | None | Poll and route status<br><br>Checks the Parseur document status, routes successful parsing to the success response, and sends a dedicated response if the status API call fails. |
| `Upload to Parseur` | `n8n-nodes-parseur.parseur` | Uploads document to Parseur parser | `When Document Requested` | `Wait 2 Seconds`, `Respond with Upload Error` | Receive and upload document<br><br>Handles the incoming webhook request, uploads the received document to Parseur, and returns an immediate upload error response if the upload fails. |
| `Wait 2 Seconds` | `n8n-nodes-base.wait` | Pauses execution for 2 seconds | `Upload to Parseur` | `Prepare Retry Counter` | Prepare first status check<br><br>Waits briefly after a successful upload and initializes the retry counter before polling Parseur for the first time. |
| `Check Parsing Complete` | `n8n-nodes-base.if` | Evaluates if parsing is done and retries remain | `Fetch Parsing Status` | `Check Parsing Success`, `Respond with Success` | Retry or timeout decision<br><br>Verifies whether polling should continue, waits before the next retry when appropriate, or returns a timeout response when parsing has not completed within the allowed attempts. |
| `Wait for Retry` | `n8n-nodes-base.wait` | Delays before the next retry poll | `Check Parsing Success` | `Increment Retry Counter` | Retry or timeout decision<br><br>Verifies whether polling should continue, waits before the next retry when appropriate, or returns a timeout response when parsing has not completed within the allowed attempts. |
| `Increment Retry Counter` | `n8n-nodes-base.set` | Increments retryCount variable by 1 | `Wait for Retry` | `Fetch Parsing Status` | Increment retry counter<br><br>Updates the retry count in an isolated loopback step before sending execution back to the Parseur status check. |
| `Check Parsing Success` | `n8n-nodes-base.if` | Validates if parse status is PARSEDOK | `Check Parsing Complete` | `Wait for Retry`, `Respond with Timeout Error` | Retry or timeout decision<br><br>Verifies whether polling should continue, waits before the next retry when appropriate, or returns a timeout response when parsing has not completed within the allowed attempts. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Webhook Trigger Node:**
   - Add a **Webhook** node named `When Document Requested`.
   - Set **Path** to `parse-document`, **HTTP Method** to `POST`, **Response Mode** to `Response Node`, and configure **Authentication** as Header Auth. Set the binary property name to `data`.
2. **Create the Upload Node:**
   - Add a **Parseur** node named `Upload to Parseur`.
   - Set **Parser ID** to `202635`. Configure **Error Handling** to continue on error (`continueErrorOutput`).
   - Configure valid Parseur credentials.
   - Connect `When Document Requested` to `Upload to Parseur`.
3. **Create the Upload Error Response Node:**
   - Add a **Respond to Webhook** node named `Respond with Upload Error`.
   - Set **Response Code** to `400` and response body to JSON: `{"error": true, "message": "Could not process the uploaded file. Please check the file format and try again."}`.
   - Connect the error output of `Upload to Parseur` to `Respond with Upload Error`.
4. **Create the Initial Wait Node:**
   - Add a **Wait** node named `Wait 2 Seconds`.
   - Set **Amount** to `2`.
   - Connect the main output of `Upload to Parseur` to `Wait 2 Seconds`.
5. **Create the Prepare Counter Node:**
   - Add a **Set** node named `Prepare Retry Counter`.
   - Add assignment: Name `retryCount`, Type `Number`, Value `0`.
   - Connect `Wait 2 Seconds` to `Prepare Retry Counter`.
6. **Create the Status Fetch Node:**
   - Add an **HTTP Request** node named `Fetch Parsing Status`.
   - Set **URL** to `={{ 'https://api.parseur.com/document/' + $('Upload to Parseur').item.json.response.attachments[0].DocumentID }}`.
   - Set **Authentication** to Generic Credential Type -> HTTP Header Auth. Configure appropriate API credentials.
   - Set **Error Handling** to continue on error (`continueErrorOutput`).
   - Connect `Prepare Retry Counter` and `Increment Retry Counter` to `Fetch Parsing Status`.
7. **Create the API Error Response Node:**
   - Add a **Respond to Webhook** node named `Respond with API Error`.
   - Set **Response Code** to `503` and response body to JSON: `{"error": true, "message": "Unable to reach parsing service. Please try again in a few moments."}`.
   - Connect the error output of `Fetch Parsing Status` to `Respond with API Error`.
8. **Create the Check Parsing Complete Node:**
   - Add an **IF** node named `Check Parsing Complete`.
   - Set condition 1: `={{ $json.status }}` (String) **Not Equals** `PARSEDOK`.
   - Set condition 2: `={{ $('Prepare Retry Counter').first().json.retryCount }}` (Number) **Less Than** `10`.
   - Combinator: `AND`.
   - Connect the main output of `Fetch Parsing Status` to `Check Parsing Complete`.
9. **Create the Success Response Node:**
   - Add a **Respond to Webhook** node named `Respond with Success`.
   - Set **Response Body** to expression: `={{ JSON.parse($json.result) }}`.
   - Connect the false output (or success condition branch) of `Check Parsing Complete` to `Respond with Success`.
10. **Create the Check Parsing Success Node:**
    - Add an **IF** node named `Check Parsing Success`.
    - Set condition: `={{ $json.status }}` (String) **Equals** `PARSEDOK`.
    - Connect the true output of `Check Parsing Complete` to `Check Parsing Success`.
11. **Create the Timeout Error Response Node:**
    - Add a **Respond to Webhook** node named `Respond with Timeout Error`.
    - Set **Response Code** to `408` and response body to JSON: `{"error": true, "message": "Document parsing timed out after multiple attempts. Please check the file and try again."}`.
    - Connect the false output of `Check Parsing Success` to `Respond with Timeout Error`.
12. **Create the Retry Wait Node:**
    - Add a **Wait** node named `Wait for Retry`.
    - Connect the true output of `Check Parsing Success` to `Wait for Retry`.
13. **Create the Increment Counter Node:**
    - Add a **Set** node named `Increment Retry Counter`.
    - Add assignment: Name `retryCount`, Type `Number`, Value: `=={{ $('Prepare Retry Counter').first().json.retryCount + 1 }}`.
    - Connect `Wait for Retry` to `Increment Retry Counter`.
    - Connect `Increment Retry Counter` back to `Fetch Parsing Status` to close the loop.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Workflow source and process disclaimer | Provided exclusively from an automated n8n workflow export respecting content safety policies. |