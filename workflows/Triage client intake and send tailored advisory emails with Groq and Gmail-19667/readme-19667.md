Triage client intake and send tailored advisory emails with Groq and Gmail

https://n8nworkflows.xyz/workflows/triage-client-intake-and-send-tailored-advisory-emails-with-groq-and-gmail-19667


# Triage client intake and send tailored advisory emails with Groq and Gmail

### 1. Workflow Overview

The "Light in the Dark — AI Advisory Pipeline v2" workflow automates client intake management by capturing form submissions via webhook, providing immediate confirmation to the user, generating personalized advisory content using Groq's LLM, assessing case urgency, notifying internal stakeholders (Cyrus), and delivering the tailored AI-generated response back to the client.

The workflow logic is categorized into three functional blocks:
- **1.1 Input Reception & Initial Acknowledgement:** Captures the incoming HTTP POST request, responds instantly to the calling frontend, and sends an HTML confirmation email to the client based on the selected service type.
- **1.2 AI Generation & Urgency Classification:** Interfaces with Groq's Chat Completions API to produce customized advisory text and evaluates the submission's urgency score to determine downstream routing.
- **1.3 Internal Notification & Client Delivery:** Routes internal alerts to Cyrus via Gmail (applying standard or URGENT formatting based on classification) and dispatches the final AI-generated advice to the client.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Initial Acknowledgement
- **Overview:** This block acts as the entry point of the pipeline. It listens for HTTP POST submissions containing client intake data, instantly responds to the webhook caller to close the connection, and triggers an automated service-specific confirmation email to the client via Gmail.
- **Nodes Involved:** 
  - `When Intake Form Submitted`
  - `Respond to Intake Submission`
  - `Email Confirmation to Client`
- **Node Details:**
  - **When Intake Form Submitted**
    - *Type & Role:* Webhook trigger node (`n8n-nodes-base.webhook`). Listens for incoming HTTP POST requests.
    - *Configuration:* Path set to `intake-form`, response mode set to `responseNode` to allow explicit asynchronous response handling.
    - *Key Expressions:* N/A (Triggers on raw payload).
    - *Connections:* Outputs to both `Respond to Intake Submission` and `Email Confirmation to Client`.
    - *Failure Types:* HTTP method mismatch (requires POST), network timeouts, or invalid payload structure.
  - **Respond to Intake Submission**
    - *Type & Role:* Webhook response node (`n8n-nodes-base.respondToWebhook`). Sends an immediate HTTP acknowledgement back to the client frontend.
    - *Configuration:* Responds with JSON data.
    - *Key Expressions:* 
      ```json
      {"status": "received", "message": "Your intake has been received. You will hear from us within 24 hours."}
      ```
    - *Connections:* Input from `When Intake Form Submitted`. No output connections.
    - *Failure Types:* Execution failure if the upstream webhook fails to pass context.
  - **Email Confirmation to Client**
    - *Type & Role:* Gmail action node (`n8n-nodes-base.gmail`). Sends an HTML confirmation email.
    - *Configuration:* Uses Gmail OAuth2 credentials to send messages.
    - *Key Expressions:* 
      - Recipient: `={{ $('When Intake Form Submitted').item.json.body.email }}`
      - Message Body (conditional based on service): 
        ```javascript
        ={{ $('When Intake Form Submitted').item.json.body.service === 'crisis-light' ? `<!DOCTYPE html><html><body><p>Emergency Advisory Received.</p></body></html>` : `<!DOCTYPE html><html><body><p>Intake Received.</p></body></html>` }}
        ```
    - *Connections:* Input from `When Intake Form Submitted`.
    - *Failure Types:* OAuth2 authentication expiration, invalid recipient address syntax, or HTML parsing errors.

#### 1.2 AI Generation & Urgency Classification
- **Overview:** *(Note: While the underlying JSON currently references nodes only up to the initial block, the conceptual design of this block processes intake data through an external LLM and evaluates urgency scoring).*
- **Nodes Involved:** 
  - *(Nodes defined conceptually in workflow documentation/notes: Groq HTTP Request node, Urgency evaluation IF node).*
- **Node Details:**
  - Configuration requires a valid Groq API key (Bearer token authentication), model selection, and prompt mapping using input variables (email, name, service, urgency, area, story).
  - Evaluates urgency scores against a threshold (e.g., score >= 7) to determine escalation paths.

#### 1.3 Internal Notification & Client Delivery
- **Overview:** *(Note: Conceptual block based on workflow specifications).*
- **Nodes Involved:**
  - *(Nodes defined conceptually: Internal Gmail notification nodes for standard/urgent alerts, and final client advisory email delivery node).*
- **Node Details:**
  - Branches into URGENT vs. standard alert formatting for internal stakeholders before dispatching the final generated advisory response to the client.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Sticky Note` | `n8n-nodes-base.stickyNote` | Workflow documentation and high-level setup overview. | None | None | ## Light in the Dark — AI Advisory Pipeline v2<br><br>### How it works<br>This workflow receives a client intake submission through a webhook, immediately acknowledges the form, and sends the client a confirmation email... |
| `Sticky Note1` | `n8n-nodes-base.stickyNote` | Documentation cluster for intake and acknowledgement. | None | None | ## Receive and acknowledge intake<br><br>Handles the incoming form submission, returns a webhook response, and sends an immediate confirmation email to the client... |
| `Sticky Note2` | `n8n-nodes-base.stickyNote` | Documentation cluster for AI generation and classification. | None | None | ## Generate and classify advice<br><br>Calls Groq to produce the AI advisory response, then checks whether the situation should be treated as urgent... |
| `Sticky Note3` | `n8n-nodes-base.stickyNote` | Documentation cluster for notifications and response delivery. | None | None | ## Notify and send response<br><br>Branches into urgent or standard Gmail notifications to Cyrus, then converges to send the AI-generated response to the client... |
| `When Intake Form Submitted` | `n8n-nodes-base.webhook` | Entry point for incoming HTTP POST intake submissions. | None | `Respond to Intake Submission`, `Email Confirmation to Client` | ## Receive and acknowledge intake<br><br>Handles the incoming form submission, returns a webhook response, and sends an immediate confirmation email to the client... |
| `Respond to Intake Submission` | `n8n-nodes-base.respondToWebhook` | Returns immediate JSON acknowledgement to the webhook caller. | `When Intake Form Submitted` | None | ## Receive and acknowledge intake<br><br>Handles the incoming form submission, returns a webhook response, and sends an immediate confirmation email to the client... |
| `Email Confirmation to Client` | `n8n-nodes-base.gmail` | Sends an HTML confirmation email to the client via Gmail. | `When Intake Form Submitted` | None | ## Receive and acknowledge intake<br><br>Handles the incoming form submission, returns a webhook response, and sends an immediate confirmation email to the client... |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Webhook Trigger:**
   - Add a **Webhook** node named `When Intake Form Submitted`.
   - Set **HTTP Method** to `POST`.
   - Set **Path** to `intake-form`.
   - Set **Response Mode** to `Respond Using 'Respond to Webhook' Node`.

2. **Create the Webhook Response Node:**
   - Add a **Respond to Webhook** node named `Respond to Intake Submission`.
   - Connect the output of `When Intake Form Submitted` to this node.
   - Set **Respond With** to `JSON`.
   - Set **Response Body** expression to:
     ```json
     {"status": "received", "message": "Your intake has been received. You will hear from us within 24 hours."}
     ```

3. **Create the Client Confirmation Email Node:**
   - Add a **Gmail** node named `Email Confirmation to Client`.
   - Connect the output of `When Intake Form Submitted` to this node.
   - Configure **Gmail OAuth2** credentials.
   - Set **Send To** expression to:
     ```javascript
     ={{ $('When Intake Form Submitted').item.json.body.email }}
     ```
   - Set **Message** expression to evaluate the service type:
     ```javascript
     ={{ $('When Intake Form Submitted').item.json.body.service === 'crisis-light' ? `<!DOCTYPE html><html><body><p>Emergency Advisory Received.</p></body></html>` : `<!DOCTYPE html><html><body><p>Intake Received.</p></body></html>` }}
     ```

4. **Add Documentation Notes:**
   - Create four **Sticky Note** nodes matching the content and dimensions specified in the summary table to maintain structural documentation across the canvas.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Light in the Dark — AI Advisory Pipeline v2 Architecture | Automated intake processing, Groq LLM integration, and Gmail routing. |