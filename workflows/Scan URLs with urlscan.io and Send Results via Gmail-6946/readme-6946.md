Scan URLs with urlscan.io and Send Results via Gmail

https://n8nworkflows.xyz/workflows/scan-urls-with-urlscan-io-and-send-results-via-gmail-6946


# Scan URLs with urlscan.io and Send Results via Gmail

### 1. Workflow Overview

This n8n workflow automates the scanning of URLs via the urlscan.io service and subsequently sends the scan results through Gmail. It is designed to receive URLs via an HTTP POST request, submit them for scanning, wait for the scan to complete, and email the relevant results to a specified recipient.

The workflow consists of four main logical blocks:

- **1.1 Input Reception:** Receives URL data via a webhook POST request.
- **1.2 URL Scanning:** Submits the URL to urlscan.io and obtains scan identifiers.
- **1.3 Wait and Processing Delay:** Pauses execution to allow urlscan.io to generate scan artifacts.
- **1.4 Notification:** Sends an email with links to the scan result, screenshot, and API JSON output.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block exposes a webhook endpoint to accept incoming HTTP POST requests containing the URL to scan.

- **Nodes Involved:**  
  - Webhook

- **Node Details:**

  - **Webhook**  
    - **Type:** Webhook Trigger  
    - **Role:** Entry point for the workflow; accepts POST requests at the path `/urlscan`.  
    - **Configuration:**  
      - HTTP Method: POST  
      - Path: `urlscan`  
    - **Expressions/Variables:**  
      - Receives JSON payload expected to contain a `url` field, e.g., `{ "url": "https://example.com" }`  
    - **Connections:**  
      - Outputs data to the "Perform a scan" node.  
    - **Edge Cases/Potential Failures:**  
      - Missing or invalid `url` field in incoming JSON may cause downstream nodes to fail or scan invalid URLs.  
      - Unauthorized or malformed requests are not explicitly handled here.  
    - **Version Requirements:**  
      - Uses n8n version 2 webhook node (typeVersion 2).

#### 2.2 URL Scanning

- **Overview:**  
  Submits the received URL to urlscan.io for scanning and retrieves a scan task ID.

- **Nodes Involved:**  
  - Perform a scan

- **Node Details:**

  - **Perform a scan**  
    - **Type:** urlScanIo Node  
    - **Role:** Submits the URL to urlscan.io API using configured credentials; fetches scan UUID.  
    - **Configuration:**  
      - URL to scan is extracted flexibly from the webhook payload using expression:  
        `={{ $json.body?.url?.[0] ?? $json.body?.url ?? $json.url }}`  
      - No additional fields set; default scan options used.  
    - **Credentials:**  
      - Uses `urlscan.io` API key credential stored securely in n8n.  
    - **Connections:**  
      - Outputs to the "Wait" node.  
    - **Edge Cases/Potential Failures:**  
      - API key invalid or missing causes authentication errors.  
      - URL extraction expression failing if payload structure changes.  
      - API rate limits or urlscan.io service downtime.  
    - **Version Requirements:**  
      - Uses n8n urlScanIo node, typeVersion 1.

#### 2.3 Wait and Processing Delay

- **Overview:**  
  Introduces a fixed 30-second delay to allow urlscan.io to process the scan and generate artifacts such as screenshots.

- **Nodes Involved:**  
  - Wait

- **Node Details:**

  - **Wait**  
    - **Type:** Wait Node  
    - **Role:** Pauses workflow execution for a fixed duration.  
    - **Configuration:**  
      - Wait time set to 30 seconds.  
    - **Connections:**  
      - Outputs to the "Send a message" node.  
    - **Edge Cases/Potential Failures:**  
      - Timeout too short may result in incomplete scan artifacts, e.g., missing screenshots.  
      - Excessive wait time may delay notification unnecessarily.  
    - **Version Requirements:**  
      - Uses n8n Wait node typeVersion 1.1.

#### 2.4 Notification

- **Overview:**  
  Sends an email via Gmail containing links to the scan result page, the screenshot, and the API JSON output.

- **Nodes Involved:**  
  - Send a message

- **Node Details:**

  - **Send a message**  
    - **Type:** Gmail Node  
    - **Role:** Sends an email notification with scan details and links.  
    - **Configuration:**  
      - Recipient email address hard-coded in the node parameter `sendTo` (e.g., `email@domain.com`).  
      - Email subject dynamically includes scanned URL using expression:  
        `=URLScan Submitted for {{ $json.url || $item(0).$node["Webhook"].json.body.url?.[0] || $item(0).$node["Webhook"].json.body.url }}`  
      - Email body is HTML formatted, embedding:  
        - The submitted URL  
        - Result page link from urlscan.io using scan ID  
        - Screenshot link formed dynamically from scan ID  
        - Raw API JSON output  
      - Option `appendAttribution` disabled to avoid additional text.  
    - **Credentials:**  
      - Uses Gmail OAuth2 credentials configured in n8n.  
    - **Connections:**  
      - Final node; no outputs.  
    - **Edge Cases/Potential Failures:**  
      - Gmail OAuth2 token expiry or invalid credentials causing authentication failure.  
      - Email address incorrectly set or invalid leading to undelivered emails.  
      - Missing scan IDs causing broken links in email body.  
    - **Version Requirements:**  
      - Uses Gmail node typeVersion 2.1.

---

### 3. Summary Table

| Node Name          | Node Type         | Functional Role                      | Input Node(s)    | Output Node(s)   | Sticky Note                                                                                                                |
|--------------------|-------------------|------------------------------------|------------------|------------------|----------------------------------------------------------------------------------------------------------------------------|
| Sticky Note — How it works | Sticky Note      | Provides workflow explanation      | —                | —                | Explains workflow steps, input example, and expected output links                                                         |
| Sticky Note — Setup | Sticky Note      | Provides setup instructions        | —                | —                | Details credentials setup, webhook path, wait time notes, and best practices                                               |
| Webhook            | Webhook           | Receives incoming URL data via POST| —                | Perform a scan   |                                                                                                                            |
| Perform a scan     | urlScanIo         | Submits URL for scanning via API   | Webhook          | Wait             |                                                                                                                            |
| Wait               | Wait              | Pauses execution to allow scan processing| Perform a scan | Send a message   |                                                                                                                            |
| Send a message     | Gmail             | Sends email with scan results      | Wait             | —                |                                                                                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `urlscan`  
   - No authentication configured (optional depending on use case).  
   - Purpose: Accept JSON payload with a `url` field.

2. **Create urlScanIo Node ("Perform a scan")**  
   - Type: urlScanIo  
   - Credentials: Create and select `urlscan.io` API Key credential in n8n credentials manager.  
   - URL Parameter: Set to expression:  
     `={{ $json.body?.url?.[0] ?? $json.body?.url ?? $json.url }}`  
   - Connect input from Webhook node output.

3. **Create Wait Node ("Wait")**  
   - Type: Wait  
   - Set wait time to 30 seconds (adjustable as needed).  
   - Connect input from Perform a scan node output.

4. **Create Gmail Node ("Send a message")**  
   - Type: Gmail  
   - Credentials: Configure Gmail OAuth2 credentials in n8n.  
   - Set **To** address to the intended recipient email (e.g., `email@domain.com`).  
   - Subject: Use expression:  
     `=URLScan Submitted for {{ $json.url || $item(0).$node["Webhook"].json.body.url?.[0] || $item(0).$node["Webhook"].json.body.url }}`  
   - Message (HTML):  
     ```html
     <p><strong>URL submitted to <a href="https://urlscan.io">urlscan.io</a></strong></p>
     <p><strong>Scanned URL:</strong> {{ $json.url }}</p>
     <p><strong>Result URL:</strong> {{ $json.result }}</p>
     <p><strong>Scan ID:</strong> {{ $json.scanId }}</p>
     <p><strong>Screenshot (when ready):</strong> <a href="https://urlscan.io/screenshots/{{ $json.scanId || $json.uuid || $json.task?.uuid }}.png">https://urlscan.io/screenshots/{{ $json.scanId || $json.uuid || $json.task?.uuid }}.png</a></p>
     <p><strong>API result (JSON):</strong><br>{{ $json.api }}</p>
     <hr>
     ```  
   - Disable "Append Attribution".  
   - Connect input from Wait node output.

5. **Connect Nodes Sequentially**  
   - Webhook → Perform a scan → Wait → Send a message

6. **Save and Activate Workflow**

7. **Webhook Testing**  
   - Use test URL `/webhook-test/urlscan` for dry runs.  
   - Post JSON payload with a `url` field to trigger scan.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                      |
|---------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| Workflow requires valid urlscan.io API Key and Gmail OAuth2 credentials configured in n8n credentials manager.| Credential setup note from Sticky Note — Setup node |
| Wait node set to 30 seconds; increase if screenshots are not ready on time.                                    | Sticky Note — Setup                                  |
| Webhook path is `/urlscan`; test URL is `/webhook-test/urlscan`.                                              | Sticky Note — Setup                                  |
| Sticky notes use Markdown formatting for clarity and documentation.                                           | Sticky Note — How it works                           |
| Input example JSON for webhook: `{ "url": "https://example.com" }`                                            | Sticky Note — How it works                           |
| Output email includes links to: result page, screenshot, and raw API JSON.                                    | Sticky Note — How it works                           |
| No hard-coded API keys in HTTP Request nodes; all credentials managed securely via n8n credential system.     | Sticky Note — Setup                                  |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This workflow respects all current content policies and contains no illegal, offensive, or protected elements. All data processed are legal and public.