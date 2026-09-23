Moderate toxic text, redact PII, and audit with a Gemini HTTP API

https://n8nworkflows.xyz/workflows/moderate-toxic-text--redact-pii--and-audit-with-a-gemini-http-api-17517


# Moderate toxic text, redact PII, and audit with a Gemini HTTP API

### 1. Workflow Overview

This workflow implements a production-grade content moderation API. It exposes an HTTP POST webhook that accepts single text entries or batches of text along with optional context. Each input is processed via Google Gemini to evaluate safety, detect PII, rate toxicity, and generate cleaned outputs. Results are logged to Google Sheets, aggregated into a summary response, returned to the caller via webhook, and optionally trigger a Slack notification if content is blocked.

The workflow logic is grouped into the following functional blocks:
- **1.1 Input Reception & Normalization:** Captures incoming HTTP POST requests and normalizes payloads into individual items per message.
- **1.2 AI Processing & Auditing:** Evaluates text via Google Gemini using a structured output parser, formats the moderation schema, and logs each item to an audit spreadsheet.
- **1.3 Aggregation & Response:** Bundles individual item results into a structured summary response and sends it back to the webhook caller.
- **1.4 Alerting:** Evaluates whether any submitted messages were blocked and conditionally posts a notification to Slack.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Normalization
- **Overview:** Receives raw HTTP POST payloads containing text strings and prepares them for individual item processing.
- **Nodes Involved:** `Moderation API Endpoint`, `Normalize Request`

- **Node Details:**
  - **Moderation API Endpoint**
    - *Type & Technical Role:* `n8n-nodes-base.webhook` (Trigger)
    - *Configuration Choices:* Listens for incoming `POST` requests on the `/moderate` path using manual response mode.
    - *Key Expressions/Variables:* None.
    - *Input/Output Connections:* Output connects to `Normalize Request`.
    - *Version-specific Requirements:* Version 2.1.
    - *Edge Cases/Failure Types:* Invalid HTTP methods return 404/405; malformed JSON payloads fail upstream.

  - **Normalize Request**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Data Transformation)
    - *Configuration Choices:* Executes JavaScript to check if `body.texts` is an array or `body.text` is a string, mapping them into individual execution items with an assigned ID, text, context, and requester. Defaults to empty strings if no text is provided.
    - *Key Expressions/Variables:* Uses `$json.body` and `$json.headers`.
    - *Input/Output Connections:* Input from `Moderation API Endpoint`; output connects to `Moderate Text with Gemini`.
    - *Version-specific Requirements:* Version 2.
    - *Edge Cases/Failure Types:* Missing or null body properties handled gracefully with defaults.

---

#### 1.2 AI Processing & Auditing
- **Overview:** Sends normalized texts to Google Gemini for classification, parses structured output, shapes the result, and logs entries into Google Sheets.
- **Nodes Involved:** `Moderate Text with Gemini`, `Google Gemini Chat Model`, `Moderation Parser`, `Shape Result`, `Log to Audit Trail`

- **Node Details:**
  - **Moderate Text with Gemini**
    - *Type & Technical Role:* `@n8n/n8n-nodes-langchain.chainLlm` (AI Chain)
    - *Configuration Choices:* Configured with an inline prompt instructing Gemini to act as a content moderation engine returning allow/flag/block decisions, toxicity levels, categories, PII types, reasons, and cleaned text. Error handling set to `continueRegularOutput`.
    - *Key Expressions/Variables:* `{{ $json.context }}` and `{{ $json.text }}`.
    - *Input/Output Connections:* Input from `Normalize Request`; linked to `Google Gemini Chat Model` and `Moderation Parser`. Output connects to `Shape Result`.
    - *Version-specific Requirements:* Version 1.9.
    - *Edge Cases/Failure Types:* LLM timeout or rate limits handled by `continueRegularOutput` to prevent workflow crashes.

  - **Google Gemini Chat Model**
    - *Type & Technical Role:* `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` (AI Language Model)
    - *Configuration Choices:* Uses model `models/gemini-3.1-flash-lite` with a temperature of `0` for deterministic outputs.
    - *Key Expressions/Variables:* None.
    - *Input/Output Connections:* Connects as a model provider to `Moderate Text with Gemini`.
    - *Credentials:* `googlePalmApi` (Google Gemini(PaLM) Api account).
    - *Version-specific Requirements:* Version 1.1.
    - *Edge Cases/Failure Types:* Invalid credentials or API quota exhaustion.

  - **Moderation Parser**
    - *Type & Technical Role:* `@n8n/n8n-nodes-langchain.outputParserStructured` (AI Output Parser)
    - *Configuration Choices:* Defines a JSON schema example enforcing structured output fields (`action`, `toxicity`, `categories`, `contains_pii`, `pii_found`, `reason`, `cleaned_text`).
    - *Key Expressions/Variables:* None.
    - *Input/Output Connections:* Connects as an output parser to `Moderate Text with Gemini`.
    - *Version-specific Requirements:* Version 1.3.
    - *Edge Cases/Failure Types:* Non-compliant model outputs failing JSON validation.

  - **Shape Result**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Data Transformation)
    - *Configuration Choices:* Runs once for each item to normalize AI responses, falling back to safe defaults ("flag" action and human review reason) if the AI output fails to parse.
    - *Key expressions/Variables:* Uses `$('Normalize Request').item.json` and `$json.output`.
    - *Input/Output Connections:* Input from `Moderate Text with Gemini`; output connects to `Log to Audit Trail`.
    - *Version-specific Requirements:* Version 2.

  - **Log to Audit Trail**
    - *Type & Technical Role:* `n8n-nodes-base.googleSheets` (Data Integration)
    - *Configuration Choices:* Appends records to a spreadsheet (`Select your spreadsheet`, tab: `Moderation Log`).
    - *Key Expressions/Variables:* None.
    - *Input/Output Connections:* Input from `Shape Result`; output connects to `Bundle Results`.
    - *Credentials:* `googleSheetsOAuth2Api` (Google Sheets account).
    - *Version-specific Requirements:* Version 4.7.
    - *Edge Cases/Failure Types:* Spreadsheet sheet name mismatches or permission errors.

---

#### 1.3 Aggregation & Response
- **Overview:** Bundles all individual message moderation results into a unified summary payload and responds to the webhook client.
- **Nodes Involved:** `Bundle Results`, `Build API Response`, `Return Result`

- **Node Details:**
  - **Bundle Results**
    - *Type & Technical Role:* `n8n-nodes-base.aggregate` (Data Transformation)
    - *Configuration Choices:* Aggregates all item data into a single array structure.
    - *Key Expressions/Variables:* None.
    - *Input/Output Connections:* Input from `Log to Audit Trail`; output connects to `Build API Response`.
    - *Version-specific Requirements:* Version 1.

  - **Build API Response**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Data Transformation)
    - *Configuration Choices:* JavaScript execution calculating summary totals (total, blocked, flagged, allowed) and formatting the response array.
    - *Key Expressions/Variables:* Uses `$json.data`.
    - *Input/Output Connections:* Input from `Bundle Results`; outputs connect to `Return Result` and `Any Blocked`.
    - *Version-specific Requirements:* Version 2.

  - **Return Result**
    - *Type & Technical Role:* `n8n-nodes-base.respondToWebhook` (Webhook Response)
    - *Configuration Choices:* Responds with HTTP status code `200`.
    - *Key Expressions/Variables:* None.
    - *Input/Output Connections:* Input from `Build API Response`.
    - *Version-specific Requirements:* Version 1.5.

---

#### 1.4 Alerting
- **Overview:** Evaluates the summary counts and sends a Slack alert if any content was flagged as blocked.
- **Nodes Involved:** `Any Blocked`, `Alert on Blocked Content`

- **Node Details:**
  - **Any Blocked**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Flow Control)
    - *Configuration Choices:* Evaluates whether `{{ $json.summary.blocked }}` is strictly greater than `0`.
    - *Key Expressions/Variables:* `={{ $json.summary.blocked }}`
    - *Input/Output Connections:* Input from `Build API Response`; output connects to `Alert on Blocked Content`.
    - *Version-specific Requirements:* Version 2.2.

  - **Alert on Blocked Content**
    - *Type & Technical Role:* `n8n-nodes-base.slack` (Notification)
    - *Configuration Choices:* Posts a message to the `#moderation` channel using OAuth2 authentication.
    - *Key Expressions/Variables:* `=*:no_entry: Moderation API blocked content*\n{{ $json.summary.blocked }} of {{ $json.summary.total }} submitted messages were blocked. Check the audit log for details.`
    - *Input/Output Connections:* Input from `Any Blocked`.
    - *Credentials:* `slackOAuth2Api` (Slack account 2).
    - *Version-specific Requirements:* Version 2.5.
    - *Edge Cases/Failure Types:* Slack channel permission issues or token expiration.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Moderation API Endpoint** | `n8n-nodes-base.webhook` | Webhook Trigger | None | Normalize Request | ## Moderate toxic text and redact PII with a batch Gemini HTTP API<br><br>### How it works<br>This is a production-style moderation API, not a single classify-and-return call. POST a JSON body with either one text or an array of texts (plus an optional requester ID) to the webhook. The request is normalised into one item per message, and a Basic LLM Chain with Google Gemini classifies each: an allow / flag / block decision, toxicity, categories, any personal data found, a reason, and a cleaned version with slurs masked and PII redacted. Every message is written to a Google Sheets audit trail, all results are aggregated into a single JSON response with a summary count, and the Respond to Webhook node returns it to the caller. If any message is blocked, a parallel branch posts a Slack alert. Moderation failures fail safe to human review, so the API never errors out.<br><br>### Setup<br>1. Connect Google Gemini (PaLM) API, Google Sheets and Slack.<br>2. Point the audit-log node at your spreadsheet (tab: Moderation Log) and pick a Slack channel.<br>3. Activate the workflow and POST `{"texts": ["."]}` to the production URL.<br><br>### Customization tips<br>Enable webhook Header Auth to require an API key, or return HTTP 422 when anything is blocked.<br>## 1. Accept a batch<br>The webhook accepts one text or an array. A Code node normalises it into one item per message with a requester ID. |
| **Normalize Request** | `n8n-nodes-base.code` | Normalization | Moderation API Endpoint | Moderate Text with Gemini | ## Moderate toxic text and redact PII with a batch Gemini HTTP API<br><br>### How it works<br>This is a production-style moderation API, not a single classify-and-return call. POST a JSON body with either one text or an array of texts (plus an optional requester ID) to the webhook. The request is normalised into one item per message, and a Basic LLM Chain with Google Gemini classifies each: an allow / flag / block decision, toxicity, categories, any personal data found, a reason, and a cleaned version with slurs masked and PII redacted. Every message is written to a Google Sheets audit trail, all results are aggregated into a single JSON response with a summary count, and the Respond to Webhook node returns it to the caller. If any message is blocked, a parallel branch posts a Slack alert. Moderation failures fail safe to human review, so the API never errors out.<br><br>### Setup<br>1. Connect Google Gemini (PaLM) API, Google Sheets and Slack.<br>2. Point the audit-log node at your spreadsheet (tab: Moderation Log) and pick a Slack channel.<br>3. Activate the workflow and POST `{"texts": ["."]}` to the production URL.<br><br>### Customization tips<br>Enable webhook Header Auth to require an API key, or return HTTP 422 when anything is blocked.<br>## 1. Accept a batch<br>The webhook accepts one text or an array. A Code node normalises it into one item per message with a requester ID. |
| **Moderate Text with Gemini** | `@n8n/n8n-nodes-langchain.chainLlm` | AI Moderation Chain | Normalize Request | Shape Result | ## Moderate toxic text and redact PII with a batch Gemini HTTP API<br><br>### How it works<br>This is a production-style moderation API, not a single classify-and-return call. POST a JSON body with either one text or an array of texts (plus an optional requester ID) to the webhook. The request is normalised into one item per message, and a Basic LLM Chain with Google Gemini classifies each: an allow / flag / block decision, toxicity, categories, any personal data found, a reason, and a cleaned version with slurs masked and PII redacted. Every message is written to a Google Sheets audit trail, all results are aggregated into a single JSON response with a summary count, and the Respond to Webhook node returns it to the caller. If any message is blocked, a parallel branch posts a Slack alert. Moderation failures fail safe to human review, so the API never errors out.<br><br>### Setup<br>1. Connect Google Gemini (PaLM) API, Google Sheets and Slack.<br>2. Point the audit-log node at your spreadsheet (tab: Moderation Log) and pick a Slack channel.<br>3. Activate the workflow and POST `{"texts": ["."]}` to the production URL.<br><br>### Customization tips<br>Enable webhook Header Auth to require an API key, or return HTTP 422 when anything is blocked.<br>## 2. Moderate & audit (Basic LLM Chain)<br>Gemini classifies each message and every result is written to a Google Sheets audit trail. Model failures fail safe to human review. |
| **Google Gemini Chat Model** | `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` | LLM Provider | None | Moderate Text with Gemini | ## Moderate toxic text and redact PII with a batch Gemini HTTP API<br><br>### How it works<br>This is a production-style moderation API, not a single classify-and-return call. POST a JSON body with either one text or an array of texts (plus an optional requester ID) to the webhook. The request is normalised into one item per message, and a Basic LLM Chain with Google Gemini classifies each: an allow / flag / block decision, toxicity, categories, any personal data found, a reason, and a cleaned version with slurs masked and PII redacted. Every message is written to a Google Sheets audit trail, all results are aggregated into a single JSON response with a summary count, and the Respond to Webhook node returns it to the caller. If any message is blocked, a parallel branch posts a Slack alert. Moderation failures fail safe to human review, so the API never errors out.<br><br>### Setup<br>1. Connect Google Gemini (PaLM) API, Google Sheets and Slack.<br>2. Point the audit-log node at your spreadsheet (tab: Moderation Log) and pick a Slack channel.<br>3. Activate the workflow and POST `{"texts": ["."]}` to the production URL.<br><br>### Customization tips<br>Enable webhook Header Auth to require an API key, or return HTTP 422 when anything is blocked.<br>## 2. Moderate & audit (Basic LLM Chain)<br>Gemini classifies each message and every result is written to a Google Sheets audit trail. Model failures fail safe to human review. |
| **Moderation Parser** | `@n8n/n8n-nodes-langchain.outputParserStructured` | Structured Parser | None | Moderate Text with Gemini | ## Moderate toxic text and redact PII with a batch Gemini HTTP API<br><br>### How it works<br>This is a production-style moderation API, not a single classify-and-return call. POST a JSON body with either one text or an array of texts (plus an optional requester ID) to the webhook. The request is normalised into one item per message, and a Basic LLM Chain with Google Gemini classifies each: an allow / flag / block decision, toxicity, categories, any personal data found, a reason, and a cleaned version with slurs masked and PII redacted. Every message is written to a Google Sheets audit trail, all results are aggregated into a single JSON response with a summary count, and the Respond to Webhook node returns it to the caller. If any message is blocked, a parallel branch posts a Slack alert. Moderation failures fail safe to human review, so the API never errors out.<br><br>### Setup<br>1. Connect Google Gemini (PaLM) API, Google Sheets and Slack.<br>2. Point the audit-log node at your spreadsheet (tab: Moderation Log) and pick a Slack channel.<br>3. Activate the workflow and POST `{"texts": ["."]}` to the production URL.<br><br>### Customization tips<br>Enable webhook Header Auth to require an API key, or return HTTP 422 when anything is blocked.<br>## 2. Moderate & audit (Basic LLM Chain)<br>Gemini classifies each message and every result is written to a Google Sheets audit trail. Model failures fail safe to human review. |
| **Shape Result** | `n8n-nodes-base.code` | Result Formatting | Moderate Text with Gemini | Log to Audit Trail | ## Moderate toxic text and redact PII with a batch Gemini HTTP API<br><br>### How it works<br>This is a production-style moderation API, not a single classify-and-return call. POST a JSON body with either one text or an array of texts (plus an optional requester ID) to the webhook. The request is normalised into one item per message, and a Basic LLM Chain with Google Gemini classifies each: an allow / flag / block decision, toxicity, categories, any personal data found, a reason, and a cleaned version with slurs masked and PII redacted. Every message is written to a Google Sheets audit trail, all results are aggregated into a single JSON response with a summary count, and the Respond to Webhook node returns it to the caller. If any message is blocked, a parallel branch posts a Slack alert. Moderation failures fail safe to human review, so the API never errors out.<br><br>### Setup<br>1. Connect Google Gemini (PaLM) API, Google Sheets and Slack.<br>2. Point the audit-log node at your spreadsheet (tab: Moderation Log) and pick a Slack channel.<br>3. Activate the workflow and POST `{"texts": ["."]}` to the production URL.<br><br>### Customization tips<br>Enable webhook Header Auth to require an API key, or return HTTP 422 when anything is blocked.<br>## 2. Moderate & audit (Basic LLM Chain)<br>Gemini classifies each message and every result is written to a Google Sheets audit trail. Model failures fail safe to human review. |
| **Log to Audit Trail** | `n8n-nodes-base.googleSheets` | Audit Logging | Shape Result | Bundle Results | ## Moderate toxic text and redact PII with a batch Gemini HTTP API<br><br>### How it works<br>This is a production-style moderation API, not a single classify-and-return call. POST a JSON body with either one text or an array of texts (plus an optional requester ID) to the webhook. The request is normalised into one item per message, and a Basic LLM Chain with Google Gemini classifies each: an allow / flag / block decision, toxicity, categories, any personal data found, a reason, and a cleaned version with slurs masked and PII redacted. Every message is written to a Google Sheets audit trail, all results are aggregated into a single JSON response with a summary count, and the Respond to Webhook node returns it to the caller. If any message is blocked, a parallel branch posts a Slack alert. Moderation failures fail safe to human review, so the API never errors out.<br><br>### Setup<br>1. Connect Google Gemini (PaLM) API, Google Sheets and Slack.<br>2. Point the audit-log node at your spreadsheet (tab: Moderation Log) and pick a Slack channel.<br>3. Activate the workflow and POST `{"texts": ["."]}` to the production URL.<br><br>### Customization tips<br>Enable webhook Header Auth to require an API key, or return HTTP 422 when anything is blocked.<br>## 2. Moderate & audit (Basic LLM Chain)<br>Gemini classifies each message and every result is written to a Google Sheets audit trail. Model failures fail safe to human review. |
| **Bundle Results** | `n8n-nodes-base.aggregate` | Data Aggregation | Log to Audit Trail | Build API Response | ## Moderate toxic text and redact PII with a batch Gemini HTTP API<br><br>### How it works<br>This is a production-style moderation API, not a single classify-and-return call. POST a JSON body with either one text or an array of texts (plus an optional requester ID) to the webhook. The request is normalised into one item per message, and a Basic LLM Chain with Google Gemini classifies each: an allow / flag / block decision, toxicity, categories, any personal data found, a reason, and a cleaned version with slurs masked and PII redacted. Every message is written to a Google Sheets audit trail, all results are aggregated into a single JSON response with a summary count, and the Respond to Webhook node returns it to the caller. If any message is blocked, a parallel branch posts a Slack alert. Moderation failures fail safe to human review, so the API never errors out.<br><br>### Setup<br>1. Connect Google Gemini (PaLM) API, Google Sheets and Slack.<br>2. Point the audit-log node at your spreadsheet (tab: Moderation Log) and pick a Slack channel.<br>3. Activate the workflow and POST `{"texts": ["."]}` to the production URL.<br><br>### Customization tips<br>Enable webhook Header Auth to require an API key, or return HTTP 422 when anything is blocked.<br>## 3. Aggregate & respond<br>All per-message results are bundled into one JSON response with a summary count and returned to the caller. |
| **Build API Response** | `n8n-nodes-base.code` | Response Construction | Bundle Results | Return Result, Any Blocked | ## Moderate toxic text and redact PII with a batch Gemini HTTP API<br><br>### How it works<br>This is a production-style moderation API, not a single classify-and-return call. POST a JSON body with either one text or an array of texts (plus an optional requester ID) to the webhook. The request is normalised into one item per message, and a Basic LLM Chain with Google Gemini classifies each: an allow / flag / block decision, toxicity, categories, any personal data found, a reason, and a cleaned version with slurs masked and PII redacted. Every message is written to a Google Sheets audit trail, all results are aggregated into a single JSON response with a summary count, and the Respond to Webhook node returns it to the caller. If any message is blocked, a parallel branch posts a Slack alert. Moderation failures fail safe to human review, so the API never errors out.<br><br>### Setup<br>1. Connect Google Gemini (PaLM) API, Google Sheets and Slack.<br>2. Point the audit-log node at your spreadsheet (tab: Moderation Log) and pick a Slack channel.<br>3. Activate the workflow and POST `{"texts": ["."]}` to the production URL.<br><br>### Customization tips<br>Enable webhook Header Auth to require an API key, or return HTTP 422 when anything is blocked.<br>## 3. Aggregate & respond<br>All per-message results are bundled into one JSON response with a summary count and returned to the caller. |
| **Return Result** | `n8n-nodes-base.respondToWebhook` | Webhook Response | Build API Response | None | ## Moderate toxic text and redact PII with a batch Gemini HTTP API<br><br>### How it works<br>This is a production-style moderation API, not a single classify-and-return call. POST a JSON body with either one text or an array of texts (plus an optional requester ID) to the webhook. The request is normalised into one item per message, and a Basic LLM Chain with Google Gemini classifies each: an allow / flag / block decision, toxicity, categories, any personal data found, a reason, and a cleaned version with slurs masked and PII redacted. Every message is written to a Google Sheets audit trail, all results are aggregated into a single JSON response with a summary count, and the Respond to Webhook node returns it to the caller. If any message is blocked, a parallel branch posts a Slack alert. Moderation failures fail safe to human review, so the API never errors out.<br><br>### Setup<br>1. Connect Google Gemini (PaLM) API, Google Sheets and Slack.<br>2. Point the audit-log node at your spreadsheet (tab: Moderation Log) and pick a Slack channel.<br>3. Activate the workflow and POST `{"texts": ["."]}` to the production URL.<br><br>### Customization tips<br>Enable webhook Header Auth to require an API key, or return HTTP 422 when anything is blocked.<br>## 3. Aggregate & respond<br>All per-message results are bundled into one JSON response with a summary count and returned to the caller. |
| **Any Blocked** | `n8n-nodes-base.if` | Condition Evaluation | Build API Response | Alert on Blocked Content | ## Moderate toxic text and redact PII with a batch Gemini HTTP API<br><br>### How it works<br>This is a production-style moderation API, not a single classify-and-return call. POST a JSON body with either one text or an array of texts (plus an optional requester ID) to the webhook. The request is normalised into one item per message, and a Basic LLM Chain with Google Gemini classifies each: an allow / flag / block decision, toxicity, categories, any personal data found, a reason, and a cleaned version with slurs masked and PII redacted. Every message is written to a Google Sheets audit trail, all results are aggregated into a single JSON response with a summary count, and the Respond to Webhook node returns it to the caller. If any message is blocked, a parallel branch posts a Slack alert. Moderation failures fail safe to human review, so the API never errors out.<br><br>### Setup<br>1. Connect Google Gemini (PaLM) API, Google Sheets and Slack.<br>2. Point the audit-log node at your spreadsheet (tab: Moderation Log) and pick a Slack channel.<br>3. Activate the workflow and POST `{"texts": ["."]}` to the production URL.<br><br>### Customization tips<br>Enable webhook Header Auth to require an API key, or return HTTP 422 when anything is blocked.<br>## 4. Alert on blocks<br>If any message was blocked, a parallel branch posts a Slack alert - without delaying the API response. |
| **Alert on Blocked Content** | `n8n-nodes-base.slack` | Slack Notification | Any Blocked | None | ## Moderate toxic text and redact PII with a batch Gemini HTTP API<br><br>### How it works<br>This is a production-style moderation API, not a single classify-and-return call. POST a JSON body with either one text or an array of texts (plus an optional requester ID) to the webhook. The request is normalised into one item per message, and a Basic LLM Chain with Google Gemini classifies each: an allow / flag / block decision, toxicity, categories, any personal data found, a reason, and a cleaned version with slurs masked and PII redacted. Every message is written to a Google Sheets audit trail, all results are aggregated into a single JSON response with a summary count, and the Respond to Webhook node returns it to the caller. If any message is blocked, a parallel branch posts a Slack alert. Moderation failures fail safe to human review, so the API never errors out.<br><br>### Setup<br>1. Connect Google Gemini (PaLM) API, Google Sheets and Slack.<br>2. Point the audit-log node at your spreadsheet (tab: Moderation Log) and pick a Slack channel.<br>3. Activate the workflow and POST `{"texts": ["."]}` to the production URL.<br><br>### Customization tips<br>Enable webhook Header Auth to require an API key, or return HTTP 422 when anything is blocked.<br>## 4. Alert on blocks<br>If any message was blocked, a parallel branch posts a Slack alert - without delaying the API response. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Trigger:**
   - Add a **Webhook** node named `Moderation API Endpoint`.
   - Set HTTP Method to `POST`, path to `moderate`, and Response Mode to `Response Node`.

2. **Add Request Normalization Code Node:**
   - Add a **Code** node named `Normalize Request`.
   - Set mode to run once for all items and paste the normalization JavaScript snippet handling `body.texts`, `body.text`, and `body.context`.
   - Connect `Moderation API Endpoint` output to `Normalize Request`.

3. **Configure Google Gemini LLM Chain:**
   - Add a **Basic LLM Chain** node named `Moderate Text with Gemini`.
   - Set prompt type to `Define` and paste the system prompt instructing the model to act as a moderation engine returning JSON.
   - Enable error handling option `On Error` -> `Continue (Using Regular Output)`.

4. **Attach Model and Parser to LLM Chain:**
   - Add a **Google Gemini Chat Model** node (`models/gemini-3.1-flash-lite`, temperature `0`). Configure credentials (`googlePalmApi`). Connect its output to the LLM Chain's language model input.
   - Add a **Structured Output Parser** node (`Moderation Parser`) configured with a JSON schema example reflecting action, toxicity, categories, contains_pii, pii_found, reason, and cleaned_text. Connect its output to the LLM Chain's output parser input.

5. **Format and Audit Results:**
   - Add a **Code** node named `Shape Result`. Set mode to run once for each item and add formatting logic to capture defaults on failure. Connect `Moderate Text with Gemini` output here.
   - Add a **Google Sheets** node named `Log to Audit Trail`. Set operation to `Append`, select your target spreadsheet and the `Moderation Log` sheet tab. Configure credentials (`googleSheetsOAuth2Api`). Connect `Shape Result` output here.

6. **Aggregate and Respond:**
   - Add an **Aggregate** node named `Bundle Results`. Set aggregation mode to `Aggregate All Item Data`. Connect `Log to Audit Trail` output here.
   - Add a **Code** node named `Build API Response` to calculate summary counts (`total`, `blocked`, `flagged`, `allowed`) and construct the final response object. Connect `Bundle Results` output here.
   - Add a **Respond to Webhook** node named `Return Result` with response code `200`. Connect `Build API Response` main output to this node.

7. **Implement Alerting Branch:**
   - Add an **If** node named `Any Blocked`. Configure condition to check if `{{ $json.summary.blocked }}` is greater than `0`. Connect `Build API Response` main output to this node.
   - Add a **Slack** node named `Alert on Blocked Content`. Select `Post Message` to a channel (e.g., `#moderation`), configure OAuth2 credentials (`slackOAuth2Api`), and add the alert message template referencing summary block counts. Connect the "true" branch output of `Any Blocked` to this node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Production moderation API architecture utilizing batch processing, auditing, and fail-safe error handling. | n8n Workflow Design Pattern |
| Customization tip: Enable webhook Header Auth to require an API key, or return HTTP 422 when content is blocked. | Security & Integration Best Practice |