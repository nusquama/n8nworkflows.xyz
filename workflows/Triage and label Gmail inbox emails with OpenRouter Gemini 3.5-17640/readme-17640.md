Triage and label Gmail inbox emails with OpenRouter Gemini 3.5

https://n8nworkflows.xyz/workflows/triage-and-label-gmail-inbox-emails-with-openrouter-gemini-3-5-17640


# Triage and label Gmail inbox emails with OpenRouter Gemini 3.5

### 1. Workflow Overview

This workflow automates the triage of unread incoming emails from a Gmail inbox using the OpenRouter API (Google Gemini 3.5 Flash Lite). It retrieves unread messages, normalizes their fields, requests a structured classification (category, priority, language, confidence, and summary), resolves the matching Gmail label dynamically, applies that label to the message, and marks the email as read.

The workflow logic is grouped into five distinct functional blocks:

- **1.1 Input Reception & Normalization:** Polls the inbox for unread messages, captures raw email details, and normalizes headers and bodies into a secure, clean format.
- **1.2 Classification Preparation & AI Execution:** Constructs a strict JSON schema prompt, sends the sanitized payload to OpenRouter, and receives structured classification data.
- **1.3 AI Response Validation:** Validates the AI-generated classification against predefined constraints and maps it to a target Gmail label name.
- **1.4 Label Resolution:** Fetches all labels from the connected Gmail account and matches the target label name to its system ID.
- **1.5 Execution & Post-Processing:** Applies the resolved category label to the Gmail thread and removes the `UNREAD` label to conclude processing.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Input Reception & Normalization
- **Overview:** This block monitors the Gmail inbox on a schedule, pulls unread messages, and extracts essential safe text fields (sender, subject, body, timestamp) while stripping away unwanted payload noise.
- **Nodes Involved:** 
  - `When Email Unread in Inbox`
  - `Normalize Email Data`

- **Node Details:**
  - **When Email Unread in Inbox**
    - *Type and Technical Role:* `n8n-nodes-base.gmailTrigger` (Trigger node). Polls Gmail for incoming unread messages.
    - *Configuration:* Polls every minute; filters for `is:unread in:inbox -in:drafts`; sets maximum results to 50 per execution; omits downloading attachments.
    - *Key Expressions/Variables:* Uses Gmail query syntax (`q: "is:unread in:inbox -in:drafts"`).
    - *Connections:* Input: None (Trigger); Output: Connects to `Normalize Email Data`.
    - *Version-specific Requirements:* Version 1.4.
    - *Edge Cases / Potential Failures:* API rate limits, OAuth token expiration, or network timeouts during polling.

  - **Normalize Email Data**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Code node). Normalizes raw incoming Gmail objects into clean, standardized properties.
    - *Configuration:* Runs once for each incoming item using JavaScript.
    - *Key Expressions/Variables:* Extracts `messageId`, `threadId`, parsed sender email/name, subject, text body, and receipt date.
    - *Connections:* Input: `When Email Unread in Inbox`; Output: Connects to `Build Classification Request`.
    - *Version-specific Requirements:* Version 2.
    - *Edge Cases / Potential Failures:* Malformed email headers or missing `from`/`body` fields handled by fallback objects.

---

#### Block 1.2: Classification Preparation & AI Execution
- **Overview:** Constructs a constrained system prompt with a strict JSON schema and submits the normalized email payload to OpenRouter for AI-driven classification.
- **Nodes Involved:**
  - `Build Classification Request`
  - `Classify with OpenRouter`

- **Node Details:**
  - **Build Classification Request**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Code node). Prepares the body payload and system prompt for the OpenRouter chat completion endpoint.
    - *Configuration:* Runs once per item. Enforces a strict JSON schema containing category enums (`new_lead`, `existing_client`, `pricing`, `invoice`, `vendor_pitch`, `spam`), priority enums (`P1`, `P2`, `P3`), language pattern, confidence boundaries, and summary length limits.
    - *Key Expressions/Variables:* `model: 'google/gemini-3.5-flash-lite'`, dynamic injection of email sender, subject, and body.
    - *Connections:* Input: `Normalize Email Data`; Output: Connects to `Classify with OpenRouter`.
    - *Version-specific Requirements:* Version 2.
    - *Edge Cases / Potential Failures:* Prompt injection attempts inside email bodies are mitigated by treating input as untrusted string data.

  - **Classify with OpenRouter**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (HTTP Request node). Sends the classification request to the OpenRouter API.
    - *Configuration:* Method: `POST`, URL: `https://openrouter.ai/api/v1/chat/completions`, timeout: 45,000ms. Configured with generic HTTP Header Authentication. Sends custom headers (`Content-Type`, `HTTP-Referer`, `X-Title`). Enabled auto-retry on failure (up to 3 tries with 1200ms delay).
    - *Key Expressions/Variables:* `={{ JSON.stringify($json.openrouterRequest) }}`.
    - *Connections:* Input: `Build Classification Request`; Output: Connects to `Verify Classification Output`.
    - *Version-specific Requirements:* Version 4.4.
    - *Edge Cases / Potential Failures:* Invalid API keys, HTTP 429 rate limits, OpenRouter gateway timeouts, or empty response choices.

---

#### Block 1.3: AI Response Validation
- **Overview:** Validates the returned JSON payload from the AI model to guarantee format integrity and maps categories to target label names before updating Gmail.
- **Nodes Involved:**
  - `Verify Classification Output`

- **Node Details:**
  - **Verify Classification Output**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Code node). Validates types, ranges, enums, and regular expressions of the model response.
    - *Configuration:* Runs once per item. Throws explicit errors if classification parameters are malformed.
    - *Key Expressions/Variables:* Maps validated categories to exact target label strings (e.g., `"new_lead"` $\rightarrow$ `"AI Triage/New lead"`).
    - *Connections:* Input: `Classify with OpenRouter`; Output: Connects to `Fetch Gmail Labels`.
    - *Version-specific Requirements:* Version 2.
    - *Edge Cases / Potential Failures:* Fails the execution explicitly if the LLM returns invalid JSON or hallucinates unallowed categories.

---

#### Block 1.4: Label Resolution
- **Overview:** Retrieves all existing labels from the connected Gmail account and resolves the target category name to its exact internal Gmail label ID.
- **Nodes Involved:**
  - `Fetch Gmail Labels`
  - `Determine Category Label`

- **Node Details:**
  - **Fetch Gmail Labels**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (Gmail node). Pulls the complete list of labels configured in the target mailbox.
    - *Configuration:* Resource: `label`, Operation: `getAll`, Return All: `true`. Uses Gmail OAuth2 credentials.
    - *Connections:* Input: `Verify Classification Output`; Output: Connects to `Determine Category Label`.
    - *Version-specific Requirements:* Version 2.2.
    - *Edge Cases / Potential Failures:* API authentication failure or insufficient mailbox permissions.

  - **Determine Category Label**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Code node). Cross-references fetched Gmail labels against the target label name.
    - *Configuration:* Runs once for all items. Builds a lookup map and matches names.
    - *Key Expressions/Variables:* Throws an error if a required `AI Triage/...` label is missing from the mailbox.
    - *Connections:* Input: `Fetch Gmail Labels`; Output: Connects to `Label Email by Category`.
    - *Version-specific Requirements:* Version 2.
    - *Edge Cases / Potential Failures:* Missing predefined Gmail labels cause immediate workflow stoppage to prevent mislabeling.

---

#### Block 1.5: Execution & Post-Processing
- **Overview:** Applies the resolved label to the email message in Gmail and removes the unread flag to finalize processing.
- **Nodes Involved:**
  - `Label Email by Category`
  - `Mark as read`

- **Node Details:**
  - **Label Email by Category**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (Gmail node). Applies the mapped classification label to the specific message.
    - *Configuration:* Resource: `message`, Operation: `addLabels`. Retries up to 3 times on failure with an 800ms wait interval. Uses Gmail OAuth2 credentials.
    - *Key Expressions/Variables:* Label IDs: `={{ [$json.targetLabelId] }}`, Message ID: `={{ $json.messageId }}`.
    - *Connections:* Input: `Determine Category Label`; Output: Connects to `Mark as read`.
    - *Version-specific Requirements:* Version 2.2.
    - *Edge Cases / Potential Failures:* Message ID not found or API rate limits.

  - **Mark as read**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (Gmail node). Removes the `UNREAD` system label from the processed message.
    - *Configuration:* Resource: `message`, Operation: `removeLabels`, Label IDs: `UNREAD`. Retries up to 3 times on failure with an 800ms wait interval. Uses Gmail OAuth2 credentials.
    - *Key Expressions/Variables:* Message ID: `={{ $('Determine Category Label').item.json.messageId }}`.
    - *Connections:* Input: `Label Email by Category`; Output: None (Terminal node).
    - *Version-specific Requirements:* Version 2.2.
    - *Edge Cases / Potential Failures:* Network drop during state update.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Sticky Note** | `n8n-nodes-base.stickyNote` | Overview documentation and setup guide. | None | None | ## AI Gmail Triage — Free<br><br>### How it works<br><br>This workflow watches Gmail for unread inbox messages and prepares each email for automated triage. It sends the normalized email content to an AI classifier through OpenRouter, validates the category response, resolves the matching Gmail label, applies it to the message, and marks the message as processed.<br><br>### Setup steps<br><br>- Connect Gmail OAuth credentials for the trigger and Gmail action nodes.<br>- Configure the Gmail trigger to watch the intended inbox or unread message scope.<br>- Add an OpenRouter API key or authorization header to the HTTP request node used for classification.<br>- Create these six Gmail labels exactly as written, they are case-sensitive: "AI Triage/New lead", "AI Triage/Existing client", "AI Triage/Pricing", "AI Triage/Invoice", "AI Triage/Vendor pitch", "AI Triage/Spam".<br>- Review the code nodes for required category names, fallback behavior, and processed-message handling before activating the workflow.<br><br>### Customization<br><br>Adjust the AI prompt, model, and allowed categories in the preparation and validation code to match your triage system. You can also change the Gmail labels, trigger search criteria, or final processed action to fit your mailbox workflow. |
| **Sticky Note1** | `n8n-nodes-base.stickyNote` | Block documentation for capture and normalization. | None | None | ## Capture and prepare email<br><br>Starts when an unread Gmail inbox message arrives, normalizes the message content, and builds the payload needed for AI classification. |
| **Sticky Note2** | `n8n-nodes-base.stickyNote` | Block documentation for classification and validation. | None | None | ## Classify and validate<br><br>Sends the prepared email data to the OpenRouter chat completion endpoint and validates the returned classification before any Gmail changes are made. |
| **Sticky Note3** | `n8n-nodes-base.stickyNote` | Block documentation for label resolution. | None | None | ## Resolve Gmail label<br><br>Loads available Gmail labels and matches the AI-selected category to the correct Gmail label identifier. |
| **Sticky Note4** | `n8n-nodes-base.stickyNote` | Block documentation for applying labels and finishing. | None | None | ## Apply and finish<br><br>Applies the resolved category label to the message and then marks the email as processed so it is not triaged again. |
| **When Email Unread in Inbox** | `n8n-nodes-base.gmailTrigger` | Polls unread messages from the Gmail inbox. | None | Normalize Email Data | |
| **Normalize Email Data** | `n8n-nodes-base.code` | Parses and normalizes raw Gmail message objects. | When Email Unread in Inbox | Build Classification Request | |
| **Build Classification Request** | `n8n-nodes-base.code` | Formats system prompt, JSON schema, and email payload. | Normalize Email Data | Classify with OpenRouter | |
| **Classify with OpenRouter** | `n8n-nodes-base.httpRequest` | Sends structured classification request to OpenRouter API. | Build Classification Request | Verify Classification Output | Uses the OpenRouter Header Auth credential. The API key remains encrypted in the local n8n credential store and is not exported with this workflow. |
| **Verify Classification Output** | `n8n-nodes-base.code` | Validates LLM output against strict schema rules. | Classify with OpenRouter | Fetch Gmail Labels | |
| **Fetch Gmail Labels** | `n8n-nodes-base.gmail` | Retrieves all available Gmail labels. | Verify Classification Output | Determine Category Label | |
| **Determine Category Label** | `n8n-nodes-base.code` | Resolves category names to specific Gmail label IDs. | Fetch Gmail Labels | Label Email by Category | |
| **Label Email by Category** | `n8n-nodes-base.gmail` | Applies the category label to the Gmail message. | Determine Category Label | Mark as read | |
| **Mark as read** | `n8n-nodes-base.gmail` | Removes the `UNREAD` label to mark processing complete. | Label Email by Category | None | |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually in n8n:

1. **Create the Workflow:** Open n8n, create a new workflow, and name it `AI Gmail Triage — Free`.
2. **Add Node 1 (Trigger):** Create a **Gmail Trigger** node (`n8n-nodes-base.gmailTrigger`).
   - *Parameters:* Set Simple to `false`. Set Query filter (`q`) to `is:unread in:inbox -in:drafts`. Set read status to `unread`. Set poll time interval to `everyMinute`. Set Max Results to `50`.
   - *Credentials:* Connect your Gmail OAuth2 credentials.
3. **Add Node 2 (Normalize):** Create a **Code** node (`n8n-nodes-base.code`) named `Normalize Email Data`.
   - *Parameters:* Mode: `runOnceForEachItem`. Paste the normalization JavaScript parsing logic to extract `messageId`, `threadId`, `from` email and name, `subject`, `body`, and `receivedAt`.
   - *Connection:* Connect output of `When Email Unread in Inbox` to this node.
4. **Add Node 3 (Build Request):** Create a **Code** node (`n8n-nodes-base.code`) named `Build Classification Request`.
   - *Parameters:* Mode: `runOnceForEachItem`. Paste the prompt builder script containing the system prompt, target categories (`new_lead`, `existing_client`, `pricing`, `invoice`, `vendor_pitch`, `spam`), priority levels (`P1`, `P2`, `P3`), and the strict JSON schema configuration.
   - *Connection:* Connect output of `Normalize Email Data` to this node.
5. **Add Node 4 (HTTP Request):** Create an **HTTP Request** node (`n8n-nodes-base.httpRequest`) named `Classify with OpenRouter`.
   - *Parameters:* Method: `POST`, URL: `https://openrouter.ai/api/v1/chat/completions`. Specify Body as `JSON`. Set JSON Body expression to `={{ JSON.stringify($json.openrouterRequest) }}`. Under Authentication, select `Generic Credential Type` $\rightarrow$ `HTTP Header Auth`. Add headers: `Content-Type: application/json`, `HTTP-Referer: https://example.com`, `X-Title: AI Email Agent for n8n`. Set timeout to `45000`. Enable Retry on Fail (3 tries, 1200ms wait).
   - *Credentials:* Connect your OpenRouter Header Auth credential.
   - *Connection:* Connect output of `Build Classification Request` to this node.
6. **Add Node 5 (Verify Output):** Create a **Code** node (`n8n-nodes-base.code`) named `Verify Classification Output`.
   - *Parameters:* Mode: `runOnceForEachItem`. Paste the verification script to validate categories, priorities, language codes, and confidence scores, mapping them to target label names.
   - *Connection:* Connect output of `Classify with OpenRouter` to this node.
7. **Add Node 6 (Fetch Labels):** Create a **Gmail** node (`n8n-nodes-base.gmail`) named `Fetch Gmail Labels`.
   - *Parameters:* Resource: `label`, Operation: `getAll`, Return All: `true`.
   - *Credentials:* Connect your Gmail OAuth2 credentials.
   - *Connection:* Connect output of `Verify Classification Output` to this node.
8. **Add Node 7 (Determine Label):** Create a **Code** node (`n8n-nodes-base.code`) named `Determine Category Label`.
   - *Parameters:* Mode: `runOnceForAllItems`. Paste the matching logic script that resolves target label names to internal Gmail label IDs using input data and upstream verification items.
   - *Connection:* Connect output of `Fetch Gmail Labels` to this node.
9. **Add Node 8 (Apply Label):** Create a **Gmail** node (`n8n-nodes-base.gmail`) named `Label Email by Category`.
   - *Parameters:* Resource: `message`, Operation: `addLabels`. Set Message ID expression to `={{ $json.messageId }}`. Set Label IDs expression to `={{ [$json.targetLabelId] }}`. Enable Retry on Fail (3 tries, 800ms wait).
   - *Credentials:* Connect your Gmail OAuth2 credentials.
   - *Connection:* Connect output of `Determine Category Label` to this node.
10. **Add Node 9 (Mark as Read):** Create a **Gmail** node (`n8n-nodes-base.gmail`) named `Mark as read`.
    - *Parameters:* Resource: `message`, Operation: `removeLabels`. Set Message ID expression to `={{ $('Determine Category Label').item.json.messageId }}`. Set Label IDs to `UNREAD`. Enable Retry on Fail (3 tries, 800ms wait).
    - *Credentials:* Connect your Gmail OAuth2 credentials.
    - *Connection:* Connect output of `Label Email by Category` to this node.
11. **Prerequisite Configuration (External):** Ensure the following six Gmail labels are created inside your Google mailbox exactly as written (case-sensitive):
    - `AI Triage/New lead`
    - `AI Triage/Existing client`
    - `AI Triage/Pricing`
    - `AI Triage/Invoice`
    - `AI Triage/Vendor pitch`
    - `AI Triage/Spam`

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Ensure Gmail API access is enabled on your Google Workspace or consumer account. | Google Workspace / Gmail API Access |
| Create the six case-sensitive labels in Gmail before running the workflow: `AI Triage/New lead`, `AI Triage/Existing client`, `AI Triage/Pricing`, `AI Triage/Invoice`, `AI Triage/Vendor pitch`, `AI Triage/Spam`. | Mandatory Setup Requirement |
| OpenRouter API key required via HTTP Header Auth. Model utilized: `google/gemini-3.5-flash-lite`. | OpenRouter Documentation |