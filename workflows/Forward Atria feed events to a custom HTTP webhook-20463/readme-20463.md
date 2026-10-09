Forward Atria feed events to a custom HTTP webhook

https://n8nworkflows.xyz/workflows/forward-atria-feed-events-to-a-custom-http-webhook-20463


# Forward Atria feed events to a custom HTTP webhook

### 1. Workflow Overview

This workflow is designed to bridge Atria feed events with external systems by listening for incoming feed webhooks, transforming the payload, and forwarding it to a custom HTTP endpoint. Additionally, it provides a manual execution path to fetch and verify Atria feed configurations on demand. 

The logic is organized into two primary functional blocks:
- **1.1 Automated Event Forwarding Pipeline:** Listens to Atria webhook events, extracts specific metadata and counts, and POSTs the formatted data to a designated HTTP webhook receiver.
- **1.2 Manual Lifecycle Verification Path:** Manually triggers a retrieval of Atria feed details for administrative or testing purposes.

---

### 2. Block-by-Block Analysis

#### 2.1 Automated Event Forwarding Pipeline
- **Overview:** This block captures automated feed events from Atria, extracts relevant data fields (metadata and count), and securely relays them as a JSON payload to an external HTTP endpoint.
- **Nodes Involved:** 
  - Atria Trigger
  - Edit Fields
  - HTTP Request

- **Node Details:**
  - **Atria Trigger**
    - *Type and Technical Role:* `n8n-nodes-atria.atriaTrigger` (Webhook Trigger). Listens for incoming real-time events from an Atria feed.
    - *Configuration Choices:* Configured with a target `feedId` (`REPLACE_WITH_FEED_UUID`) and options set to exclude headers (`includeHeaders: false`).
    - *Key Expressions or Variables:* None (relies on incoming webhook payload).
    - *Input and Output Connections:* Input: None (Trigger node). Output: Connects to *Edit Fields*.
    - *Version-specific Requirements:* Version 1.
    - *Edge Cases or Potential Failure types:* Webhook signature verification failures, invalid Feed UUID, network drops between Atria and n8n.
  
  - **Edit Fields**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data Transformation). Extracts and restructures specific fields from the incoming Atria payload.
    - *Configuration Choices:* Assigns new properties based on incoming JSON values.
    - *Key Expressions or Variables:* 
      - `metadata`: `={{ $json.metadata }}`
      - `count`: `={{ $json.count }}`
    - *Input and Output Connections:* Input: *Atria Trigger*. Output: Connects to *HTTP Request*.
    - *Version-specific Requirements:* Version 3.5.
    - *Edge Cases or Potential Failure types:* Expression evaluation errors if the expected JSON keys (`metadata`, `count`) are missing or null in the incoming payload.

  - **HTTP Request**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (Action). Sends the formatted data outward to a target URL via an HTTP POST request.
    - *Configuration Choices:* Method set to `POST`, URL configured to `https://webhook.site/REPLACE_WITH_YOUR_TOKEN`, and configured to send body parameters.
    - *Key Expressions or Variables:* 
      - Body parameter value: `=={{ JSON.stringify($json) }}`
    - *Input and Output Connections:* Input: *Edit Fields*. Output: None (Terminal node).
    - *Version-specific Requirements:* Version 4.5.
    - *Edge Cases or Potential Failure types:* HTTP 4xx/5xx errors from the receiving endpoint, DNS resolution failures, connection timeouts.

---

#### 2.2 Manual Lifecycle Verification Path
- **Overview:** This block allows users to manually trigger a call to fetch details of a specific Atria feed, primarily used for testing or verifying configuration status.
- **Nodes Involved:** 
  - When clicking ‘Execute workflow’
  - Get feed

- **Node Details:**
  - **When clicking ‘Execute workflow’**
    - *Type and Technical Role:* `n8n-nodes-base.manualTrigger` (Manual Trigger). Acts as a manual entry point for user-initiated test runs.
    - *Configuration Choices:* None.
    - *Key Expressions or Variables:* None.
    - *Input and Output Connections:* Input: None. Output: Connects to *Get feed*.
    - *Version-specific Requirements:* Version 1.
    - *Edge Cases or Potential Failure types:* None.

  - **Get feed**
    - *Type and Technical Role:* `n8n-nodes-atria.atria` (Integration Action). Queries the Atria API to fetch feed details.
    - *Configuration Choices:* Operation set to `get` with a specified `feedId` (`REPLACE_WITH_FEED_UUID`).
    - *Key Expressions or Variables:* None.
    - *Input and Output Connections:* Input: *When clicking ‘Execute workflow’*. Output: None (Terminal node for this branch).
    - *Version-specific Requirements:* Version 1.
    - *Edge Cases or Potential Failure types:* Invalid Atria API credentials, expired API keys, invalid Feed UUID resulting in a 404 from Atria.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Atria Trigger | `n8n-nodes-atria.atriaTrigger` | Webhook event entry point | None | Edit Fields | |
| When clicking ‘Execute workflow’ | `n8n-nodes-base.manualTrigger` | Manual test execution trigger | None | Get feed | |
| Get feed | `n8n-nodes-atria.atria` | Fetch feed configuration details | When clicking ‘Execute workflow’ | None | |
| Edit Fields | `n8n-nodes-base.set` | Extract and format metadata and count | Atria Trigger | HTTP Request | |
| HTTP Request | `n8n-nodes-base.httpRequest` | Forward JSON payload to HTTP endpoint | Edit Fields | None | |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Entry Points:**
   - Add a **Manual Trigger** node (`n8n-nodes-base.manualTrigger`, named: `When clicking ‘Execute workflow’`). Leave parameters at default.
   - Add an **Atria Trigger** node (`n8n-nodes-atria.atriaTrigger`, named: `Atria Trigger`). Set `feedId` to `REPLACE_WITH_FEED_UUID` and configure the `atriaApi` credential. Ensure options have `includeHeaders` set to `false`.

2. **Build the Manual Verification Path:**
   - Add an **Atria** node (`n8n-nodes-atria.atria`, named: `Get feed`). Set operation to `get`, `feedId` to `REPLACE_WITH_FEED_UUID`, and link it to the `atriaApi` credential.
   - **Connection:** Connect `When clicking ‘Execute workflow’` output to the input of `Get feed`.

3. **Build the Automated Transformation and Forwarding Path:**
   - Add a **Set** node (`n8n-nodes-base.set`, named: `Edit Fields`). 
   - Configure assignments under parameters:
     - Field 1: Name `metadata`, Type `object`, Value `={{ $json.metadata }}`
     - Field 2: Name `count`, Type `number`, Value `={{ $json.count }}`
   - Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`, named: `HTTP Request`).
     - Set Method to `POST`.
     - Set URL to `https://webhook.site/REPLACE_WITH_YOUR_TOKEN`.
     - Enable `Send Body`.
     - Add a body parameter with name `=body` and value `=={{ JSON.stringify($json) }}`.
   - **Connections:**
     - Connect `Atria Trigger` output to `Edit Fields` input.
     - Connect `Edit Fields` output to `HTTP Request` input.

4. **Configure Credentials:**
   - Ensure an Atria API credential (`atriaApi`) is created in your n8n instance and mapped to both the `Atria Trigger` and `Get feed` nodes.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Requires an Atria free account and a valid Atria API key. | Prerequisite setup for Atria authentication. |
| Remember to replace placeholder IDs before activating. | `REPLACE_WITH_FEED_UUID`, `REPLACE_WITH_CREDENTIAL_ID`, and `REPLACE_WITH_YOUR_TOKEN` must be updated. |