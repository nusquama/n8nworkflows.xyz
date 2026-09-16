Classify and label new Gmail messages with OpenAI GPT-4o-mini

https://n8nworkflows.xyz/workflows/classify-and-label-new-gmail-messages-with-openai-gpt-4o-mini-19459


# Classify and label new Gmail messages with OpenAI GPT-4o-mini

### 1. Workflow Overview

This workflow automates the categorization and labeling of incoming Gmail messages using the OpenAI GPT-4o-mini model. Designed for small businesses, it reduces manual inbox triage by reading incoming emails, determining their context and urgency, and dynamically applying corresponding Gmail labels.

The workflow executes through three main logical blocks:
- **1.1 Input Reception & Configuration:** Polls Gmail for incoming messages every minute and establishes the triage configuration (AI model selection, classification categories, system prompts, and label mappings).
- **1.2 AI Processing & Parsing:** Constructs a structured payload from the email metadata and body snippet, queries the OpenAI Chat Completions API, and safely parses the JSON response (with a fallback to "noise" if parsing fails).
- **1.3 Label Application:** Resolves the predicted category to a specific Gmail label ID via a mapping object and applies the label to the original email message.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Reception & Configuration
- **Overview:** This block triggers execution whenever a new email arrives in the connected Gmail mailbox and initializes the required variables, model parameters, and label mappings for downstream processing.
- **Nodes Involved:** 
  - `When New Email Received`
  - `Set Model and Labels`

- **Node Details:**
  - **When New Email Received**
    - *Type and Technical Role:* `n8n-nodes-base.gmailTrigger` (Trigger Node). Monitors a Gmail account for new incoming messages.
    - *Configuration Choices:* Configured to poll every minute using default filter settings.
    - *Key Expressions or Variables:* None (outputs raw email payload including `id`, `subject`, `From`, `snippet`, and `textPlain`).
    - *Input / Output Connections:* No inputs; outputs to `Set Model and Labels`.
    - *Edge Cases / Potential Failures:* Gmail API authentication expiration or rate limits during polling.
  
  - **Set Model and Labels**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data Transformation Node). Establishes static operational parameters.
    - *Configuration Choices:* Defines the OpenAI model (`gpt-4o-mini`), the system prompt for email classification, and a `labelMap` object linking categories (`new_enquiry`, `invoice`, `supplier`, `urgent`, `noise`) to placeholder Gmail label IDs.
    - *Key Expressions or Variables:* 
      ```javascript
      ={{ ({ new_enquiry: 'PASTE_NEW_ENQUIRY_LABEL_ID', invoice: 'PASTE_INVOICE_LABEL_ID', supplier: 'PASTE_SUPPLIER_LABEL_ID', urgent: 'PASTE_URGENT_LABEL_ID', noise: 'PASTE_NOISE_LABEL_ID' }) }}
      ```
    - *Input / Output Connections:* Input from `When New Email Received`; output to `Post to OpenAI API`.
    - *Edge Cases / Potential Failures:* Unreplaced placeholder IDs in `labelMap` will cause the downstream labeling step to fail.

---

#### Block 1.2: AI Processing & Parsing
- **Overview:** This block formats the email context, sends it to OpenAI's API enforcing a JSON output structure, and validates the returned classification result.
- **Nodes Involved:**
  - `Post to OpenAI API`
  - `Parse Classification Result`

- **Node Details:**
  - **Post to OpenAI API**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (API Integration Node). Communicates with the OpenAI Chat Completions endpoint.
    - *Configuration Choices:* Uses POST method targeting `https://api.openai.com/v1/chat/completions` with JSON body formatting and Generic Header Authentication.
    - *Key Expressions or Variables:*
      ```javascript
      ={{ JSON.stringify({ model: $('Set Model and Labels').item.json.model, response_format: { type: 'json_object' }, messages: [ { role: 'system', content: $('Set Model and Labels').item.json.systemPrompt }, { role: 'user', content: 'Subject: ' + ($json.subject || '') + ' | From: ' + ($json.From || '') + ' | Body: ' + String($json.snippet || $json.textPlain || '').slice(0, 1500) } ] }) }}
      ```
    - *Input / Output Connections:* Input from `Set Model and Labels`; output to `Parse Classification Result`.
    - *Edge Cases / Potential Failures:* OpenAI API key authorization errors, rate limits (HTTP 429), timeouts, or insufficient API credits.

  - **Parse Classification Result**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript Execution Node). Parses the raw JSON string returned by the LLM and handles malformed outputs.
    - *Configuration Choices:* Custom JavaScript execution block.
    - *Key Expressions or Variables:* Extracts choices content, safely executes `JSON.parse()`, and falls back to a default `noise` category if an exception occurs.
    - *Input / Output Connections:* Input from `Post to OpenAI API`; output to `Add Labels via Gmail`.
    - *Edge Cases / Potential Failures:* Unexpected schema deviations from the LLM are caught and defaulted gracefully to avoid crashing the workflow.

---

#### Block 1.3: Label Application
- **Overview:** This block matches the classified category to its corresponding Gmail label ID and applies it to the original message.
- **Nodes Involved:**
  - `Add Labels via Gmail`

- **Node Details:**
  - **Add Labels via Gmail**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (Action Node). Modifies Gmail message labels via the Gmail API.
    - *Configuration Choices:* Operation set to `addLabels`.
    - *Key Expressions or Variables:*
      - Label IDs: `={{ [ $('Set Model and Labels').item.json.labelMap[$json.category] ] }}`
      - Message ID: `={{ $json.id }}`
    - *Input / Output Connections:* Input from `Parse Classification Result`; terminal node.
    - *Edge Cases / Potential Failures:* Invalid label IDs result in a Gmail API 404/400 error. Ensure target labels are pre-created in the Gmail account and mapped correctly.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `When New Email Received` | `n8n-nodes-base.gmailTrigger` | Monitors inbox for incoming messages | None | `Set Model and Labels` | Detect and configure<br>Starts the workflow when a new Gmail message arrives and sets the model, prompt, and label mapping used for triage. |
| `Set Model and Labels` | `n8n-nodes-base.set` | Defines triage rules, model, and label mapping | `When New Email Received` | `Post to OpenAI API` | Detect and configure<br>Starts the workflow when a new Gmail message arrives and sets the model, prompt, and label mapping used for triage. |
| `Post to OpenAI API` | `n8n-nodes-base.httpRequest` | Sends email data to OpenAI for analysis | `Set Model and Labels` | `Parse Classification Result` | Classify email content<br>Sends the prepared email data to the OpenAI chat completions API and reads the returned classification result. |
| `Parse Classification Result` | `n8n-nodes-base.code` | Parses and validates AI JSON output | `Post to OpenAI API` | `Add Labels via Gmail` | Classify email content<br>Sends the prepared email data to the OpenAI chat completions API and reads the returned classification result. |
| `Add Labels via Gmail` | `n8n-nodes-base.gmail` | Applies matching label to Gmail message | `Parse Classification Result` | None | Apply Gmail label<br>Uses the parsed classification to label the original Gmail message. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**
   - Add a **Gmail Trigger** node (`n8n-nodes-base.gmailTrigger`).
   - Name it `When New Email Received`.
   - Set polling intervals to trigger `everyMinute`.
   - Configure and authenticate your Gmail credentials.

2. **Create the Configuration Node:**
   - Add a **Set (Edit Fields)** node (`n8n-nodes-base.set`).
   - Name it `Set Model and Labels`.
   - Add string assignment `model` with value `gpt-4o-mini`.
   - Add string assignment `systemPrompt` with value: `You are an email triage assistant for a small business. Classify the email into exactly one category: new_enquiry, invoice, supplier, urgent, noise. Reply with JSON only, using the keys: category (string), urgent (boolean), summary (string, max 20 words).`
   - Add object assignment `labelMap` with expression: `={{ ({ new_enquiry: 'PASTE_NEW_ENQUIRY_LABEL_ID', invoice: 'PASTE_INVOICE_LABEL_ID', supplier: 'PASTE_SUPPLIER_LABEL_ID', urgent: 'PASTE_URGENT_LABEL_ID', noise: 'PASTE_NOISE_LABEL_ID' }) }}`.
   - Connect `When New Email Received` output to this node.

3. **Create the API Request Node:**
   - Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`).
   - Name it `Post to OpenAI API`.
   - Set **Method** to `POST` and **URL** to `https://api.openai.com/v1/chat/completions`.
   - Set **Authentication** to `Generic Credential Type` -> `HTTP Header Auth` (configure your OpenAI API key credentials).
   - Set **Body Content Type** to `JSON`.
   - Insert the JSON body expression:
     ```javascript
     ={{ JSON.stringify({ model: $('Set Model and Labels').item.json.model, response_format: { type: 'json_object' }, messages: [ { role: 'system', content: $('Set Model and Labels').item.json.systemPrompt }, { role: 'user', content: 'Subject: ' + ($json.subject || '') + ' | From: ' + ($json.From || '') + ' | Body: ' + String($json.snippet || $json.textPlain || '').slice(0, 1500) } ] }) }}
     ```
   - Connect `Set Model and Labels` output to this node.

4. **Create the Code Parsing Node:**
   - Add a **Code** node (`n8n-nodes-base.code`).
   - Name it `Parse Classification Result`.
   - Set **Language** to JavaScript and insert the following code:
     ```javascript
     const raw = $json.choices[0].message.content;
     let parsed;
     try { parsed = JSON.parse(raw); } catch (e) { parsed = { category: 'noise', urgent: false, summary: 'Could not parse model output' }; }
     const email = $('Set Model and Labels').item.json;
     return [{ json: { category: parsed.category, urgent: parsed.urgent === true, summary: parsed.summary || '', subject: email.subject || '', id: email.id } }];
     ```
   - Connect `Post to OpenAI API` output to this node.

5. **Create the Gmail Action Node:**
   - Add a **Gmail** node (`n8n-nodes-base.gmail`).
   - Name it `Add Labels via Gmail`.
   - Set **Operation** to `Add Labels`.
   - Set **Message ID** expression to `={{ $json.id }}`.
   - Set **Label IDs** expression to `={{ [ $('Set Model and Labels').item.json.labelMap[$json.category] ] }}`.
   - Authenticate using your Gmail credentials.
   - Connect `Parse Classification Result` output to this node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Full build includes Google Sheets logging, AI-drafted replies, and instant alerts. | [Baynhams Gumroad Store](https://baynhams.gumroad.com/l/inbox-triage) |