Answer Gmail support emails with OpenAI and a Pinecone knowledge base

https://n8nworkflows.xyz/workflows/answer-gmail-support-emails-with-openai-and-a-pinecone-knowledge-base-20940


# Answer Gmail support emails with OpenAI and a Pinecone knowledge base

### 1. Workflow Overview

This workflow automates the process of monitoring, parsing, classifying, and answering customer support emails received in a Gmail inbox. It uses OpenAI for intelligent classification, vector embeddings, and strict context-bound reply generation, combined with a Pinecone vector database for Retrieval-Augmented Generation (RAG). 

The logic includes safe fallback mechanisms (keyword rules and built-in demo datasets) to ensure the workflow can run end-to-end without external API credentials during testing. It strictly separates automated replies from human escalation pathways to prevent incorrect or risky answers from reaching customers.

The workflow logic is divided into the following functional blocks:
- **1.1 Input Reception & Sanitization:** Triggers on new inbox mail or manual runs, normalizes global configuration settings, parses raw email payloads, strips quote histories, and filters out unreadable or self-generated messages.
- **1.2 Classification & Routing:** Determines whether an email is a support request or irrelevant mail using OpenAI (with keyword fallback) and routes high-risk intents (e.g., refunds, legal threats, complaints) or low-confidence results directly to human handlers.
- **1.3 Knowledge Retrieval (RAG):** Embeds the incoming support query using OpenAI and queries Pinecone for relevant documentation chunks, falling back to a word-overlap search over a built-in demo knowledge base if unconfigured.
- **1.4 Answer Generation & Safety Gates:** Evaluates knowledge match scores, validates OpenAI draft generation against strict source-citation rules, and applies safety checks (caps per run, preview/test/live modes).
- **1.5 Dispatch & Escalation:** Either replies directly inside the original Gmail thread, sends test copies to a configured test address, logs previews on stop nodes, or routes unanswerable/high-risk cases as alerts to the support team.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Input Reception & Sanitization
**Overview:**  
This block ingests incoming mail from a live Gmail trigger or manual test execution, centralizes workflow settings, parses raw message structures into a clean schema, and discards invalid, bulk, or self-sent emails.

**Nodes Involved:**
- `[cred] Gmail - new email in the inbox`
- `Run the demo emails`
- `config`
- `read the email`
- `a readable email?`
- `STOP: unreadable email`

**Node Details:**

- **`[cred] Gmail - new email in the inbox`**
  - *Type & Technical Role:* `n8n-nodes-base.gmailTrigger` (Polling Trigger). Polls the specified Gmail inbox every minute for unread messages.
  - *Configuration Choices:* Filtered by `in:inbox -from:me` with `readStatus: unread`. "Simplify" option is turned off to preserve full body text, thread IDs, and RFC Message-ID headers.
  - *Inputs / Outputs:* Inputs: None (Trigger). Outputs: Connects to `config`.
  - *Edge Cases / Potential Failures:* API authentication expiration, rate limits, or network timeouts during polling.

- **`Run the demo emails`**
  - *Type & Technical Role:* `n8n-nodes-base.manualTrigger` (Manual Trigger). Allows users to manually initiate a test run using built-in demo data.
  - *Configuration Choices:* Standard manual execution trigger with no parameters required.
  - *Inputs / Outputs:* Inputs: None. Outputs: Connects to `config`.

- **`config`**
  - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript Data Transformation). Acts as a centralized configuration repository holding company details, model names, Pinecone parameters, confidence thresholds, and demo datasets.
  - *Key Expressions/Variables:* Exports flags such as `openai_enabled`, `retrieval_enabled`, `send_enabled`, and `alert_enabled` based on credential availability and execution mode.
  - *Inputs / Outputs:* Inputs: `[cred] Gmail - new email in the inbox`, `Run the demo emails`. Outputs: `read the email`.

- **`read the email`**
  - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript Data Transformation). Parses raw email objects (or fallback demo emails) into a standardized structure: sender, subject, cleaned body text (stripping quoted reply history), Gmail message ID, thread ID, and RFC Message-ID headers.
  - *Key Expressions/Variables:* Checks `cfg.own_addresses` to prevent self-reply loops and validates regex patterns for proper sender email syntax.
  - *Inputs / Outputs:* Inputs: `config`. Outputs: `a readable email?`.

- **`a readable email?`**
  - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional Router). Evaluates whether the email has a valid sender, non-empty body, and is not an automated/bulk response.
  - *Key Expressions/Variables:* Condition: `{{ $json.usable === true }}`.
  - *Inputs / Outputs:* Inputs: `read the email`. Outputs: `OpenAI configured?` (True branch), `STOP: unreadable email` (False branch).

- **`STOP: unreadable email`**
  - *Type & Technical Role:* `n8n-nodes-base.code` (Terminal Log Node). Captures dropped unreadable emails and logs the specific reason for exclusion.
  - *Inputs / Outputs:* Inputs: `a readable email?` (False). Outputs: None.

---

#### Block 1.2: Classification & Routing
**Overview:**  
This block categorizes incoming messages as support requests or irrelevant communication (such as sales pitches or newsletters) and identifies sensitive intents that mandate human intervention.

**Nodes Involved:**
- `OpenAI configured?`
- `build the classification request`
- `[cred] OpenAI - classify the email`
- `apply the AI classification`
- `classify by keywords`
- `a support request?`
- `STOP: not a support email`
- `needs a person?`

**Node Details:**

- **`OpenAI configured?`**
  - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional Router). Determines whether OpenAI credentials are active.
  - *Key Expressions/Variables:* Condition: `{{ $('config').first().json.openai_enabled }}`.
  - *Inputs / Outputs:* Inputs: `a readable email?`. Outputs: `build the classification request` (True), `classify by keywords` (False).

- **`build the classification request`**
  - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript Data Transformation). Formats a structured JSON prompt instructing the LLM to classify the email category, intent, handoff requirement, confidence score, and rationale.
  - *Inputs / Outputs:* Inputs: `OpenAI configured?` (True). Outputs: `[cred] OpenAI - classify the email`.

- **`[cred] OpenAI - classify the email`**
  - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (API Integration Node). Calls the OpenAI Chat Completions API (`gpt-4o-mini`).
  - *Configuration Choices:* POST request with a 60-second timeout, JSON response format enforced, configured with OpenAI API credentials. Retries up to 3 times on failure with error output continuation enabled.
  - *Inputs / Outputs:* Inputs: `build the classification request`. Outputs: `apply the AI classification`.

- **`apply the AI classification`**
  - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript Data Transformation). Parses the OpenAI JSON response, validates its structure, and falls back to keyword rules if parsing fails or confidence is invalid.
  - *Inputs / Outputs:* Inputs: `[cred] OpenAI - classify the email`. Outputs: `a support request?`.

- **`classify by keywords`**
  - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript Data Transformation). Executes local keyword-matching logic to categorize emails when OpenAI is disabled or fails.
  - *Inputs / Outputs:* Inputs: `OpenAI configured?` (False). Outputs: `a support request?`.

- **`a support request?`**
  - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional Router). Filters out non-support emails (e.g., spam, marketing, cold pitches).
  - *Key Expressions/Variables:* Condition: `{{ $json.route !== 'not_support' }}`.
  - *Inputs / Outputs:* Inputs: `apply the AI classification` / `classify by keywords`. Outputs: `needs a person?` (True), `STOP: not a support email` (False).

- **`STOP: not a support email`**
  - *Type & Technical Role:* `n8n-nodes-base.code` (Terminal Log Node). Logs filtered-out non-support emails, leaving them untouched in the inbox.
  - *Inputs / Outputs:* Inputs: `a support request?` (False). Outputs: None.

- **`needs a person?`**
  - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional Router). Checks if the email requires human handling due to sensitive intent (complaints, refunds, legal matters) or low classification confidence.
  - *Key Expressions/Variables:* Condition: `{{ $json.route === 'person' }}`.
  - *Inputs / Outputs:* Inputs: `a support request?` (True). Outputs: `prepare the escalation` (True), `knowledge search configured?` (False).

---

#### Block 1.3: Knowledge Retrieval (RAG)
**Overview:**  
This block converts support queries into vector embeddings and retrieves matching knowledge chunks from Pinecone or a local demo knowledge base.

**Nodes Involved:**
- `knowledge search configured?`
- `build the embedding request`
- `[cred] OpenAI - embed the question`
- `build the Pinecone query`
- `question embedded?`
- `[cred] Pinecone - find matching knowledge`
- `read the matches`
- `search the demo knowledge base`

**Node Details:**

- **`knowledge search configured?`**
  - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional Router). Checks if both OpenAI and a Pinecone host index are configured.
  - *Key Expressions/Variables:* Condition: `{{ $('config').first().json.retrieval_enabled }}`.
  - *Inputs / Outputs:* Inputs: `needs a person?` (False). Outputs: `build the embedding request` (True), `search the demo knowledge base` (False).

- **`build the embedding request`**
  - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript Data Transformation). Prepares payload parameters for vector embedding generation.
  - *Key Expressions/Variables:* Uses model `text-embedding-3-small` with configured vector dimensions (`1536`).
  - *Inputs / Outputs:* Inputs: `knowledge search configured?` (True). Outputs: `[cred] OpenAI - embed the question`.

- **`[cred] OpenAI - embed the question`**
  - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (API Integration Node). Calls the OpenAI Embeddings API.
  - *Configuration Choices:* POST request to `https://api.openai.com/v1/embeddings`, utilizing OpenAI API credentials with up to 3 retries.
  - *Inputs / Outputs:* Inputs: `build the embedding request`. Outputs: `build the Pinecone query`.

- **`build the Pinecone query`**
  - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript Data Transformation). Validates the generated embedding vector and structures the downstream Pinecone index query.
  - *Inputs / Outputs:* Inputs: `[cred] OpenAI - embed the question`. Outputs: `question embedded?`.

- **`question embedded?`**
  - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional Router). Verifies whether the embedding vector was successfully generated.
  - *Key Expressions/Variables:* Condition: `{{ $json.embedded }}`.
  - *Inputs / Outputs:* Inputs: `build the Pinecone query`. Outputs: `[cred] Pinecone - find matching knowledge` (True), `prepare the escalation` (False).

- **`[cred] Pinecone - find matching knowledge`**
  - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (API Integration Node). Queries the Pinecone vector database index.
  - *Configuration Choices:* POST request to `{{ $('config').first().json.pinecone_index_host + '/query' }}`. Authenticated via Generic Header Auth (`Api-Key`) with header `X-Pinecone-API-Version: 2025-01`. Retries up to 3 times.
  - *Inputs / Outputs:* Inputs: `question embedded?` (True). Outputs: `read the matches`.

- **`read the matches`**
  - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript Data Transformation). Parses Pinecone search results, extracts metadata (title, text, source), calculates top match scores, and evaluates against `MIN_MATCH_SCORE`.
  - *Inputs / Outputs:* Inputs: `[cred] Pinecone - find matching knowledge`. Outputs: `enough knowledge?`.

- **`search the demo knowledge base`**
  - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript Data Transformation). Computes word-overlap similarity scores against built-in demo knowledge entries when Pinecone is unconfigured.
  - *Inputs / Outputs:* Inputs: `knowledge search configured?` (False). Outputs: `enough knowledge?`.

---

#### Block 1.4: Answer Generation & Safety Gates
**Overview:**  
This block checks whether retrieved knowledge meets the minimum score threshold, generates context-restricted drafts using OpenAI (or templates), and enforces run-caps and safety checks.

**Nodes Involved:**
- `enough knowledge?`
- `OpenAI writes the reply?`
- `build the reply request`
- `[cred] OpenAI - write the reply`
- `apply the AI reply`
- `write the reply from the knowledge`
- `a reply we can send?`
- `address the reply`

**Node Details:**

- **`enough knowledge?`**
  - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional Router). Acts as a retrieval quality gate.
  - *Key Expressions/Variables:* Condition: `{{ $json.knowledge_ok }}`.
  - *Inputs / Outputs:* Inputs: `read the matches` / `search the demo knowledge base`. Outputs: `OpenAI writes the reply?` (True), `prepare the escalation` (False).

- **`OpenAI writes the reply?`**
  - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional Router). Verifies if OpenAI is enabled for writing natural language replies.
  - *Key Expressions/Variables:* Condition: `{{ $('config').first().json.openai_enabled }}`.
  - *Inputs / Outputs:* Inputs: `enough knowledge?` (True). Outputs: `build the reply request` (True), `write the reply from the knowledge` (False).

- **`build the reply request`**
  - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript Data Transformation). Constructs an LLM prompt restricting the model to answer *only* from provided knowledge excerpts and requiring explicit source citation IDs.
  - *Inputs / Outputs:* Inputs: `OpenAI writes the reply?` (True). Outputs: `[cred] OpenAI - write the reply`.

- **`[cred] OpenAI - write the reply`**
  - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (API Integration Node). Calls OpenAI Chat Completions to generate the customer response draft.
  - *Configuration Choices:* POST request to OpenAI API with JSON response formatting, 60-second timeout, and OpenAI API credentials.
  - *Inputs / Outputs:* Inputs: `build the reply request`. Outputs: `apply the AI reply`.

- **`apply the AI reply`**
  - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript Data Transformation). Validates generated drafts to ensure they contain valid source citations, lack external claims, and do not request self-escalation.
  - *Inputs / Outputs:* Inputs: `[cred] OpenAI - write the reply`. Outputs: `a reply we can send?`.

- **`write the reply from the knowledge`**
  - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript Data Transformation). Assembles a fallback response using a standard template populated from the best-matching knowledge entry text.
  - *Inputs / Outputs:* Inputs: `OpenAI writes the reply?` (False). Outputs: `a reply we can send?`.

- **`a reply we can send?`**
  - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional Router). Validates whether the generated reply passed all validation rules.
  - *Key Expressions/Variables:* Condition: `{{ $json.reply_ok }}`.
  - *Inputs / Outputs:* Inputs: `apply the AI reply` / `write the reply from the knowledge`. Outputs: `address the reply` (True), `prepare the escalation` (False).

- **`address the reply`**
  - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript Data Transformation). Configures reply metadata based on mode (`live`, `test`, or `preview`) and enforces `REPLY_CAP_PER_RUN` limits.
  - *Inputs / Outputs:* Inputs: `a reply we can send?`. Outputs: `sending on?`.

---

#### Block 1.5: Dispatch & Escalation
**Overview:**  
This block executes final outbound actions, sending replies via Gmail, routing test messages to designated test addresses, logging previews, or escalating unanswerable cases to support team inboxes.

**Nodes Involved:**
- `sending on?`
- `live reply?`
- `[cred] Gmail - reply in the thread`
- `STOP: replied in the thread`
- `[cred] Gmail - send the test copy`
- `STOP: test copy sent`
- `STOP: preview - reply not sent`
- `prepare the escalation`
- `alerts on?`
- `[cred] Gmail - alert the support team`
- `STOP: escalated to a person`
- `ESC-preview` (`esc-preview`)
- `STOP: Gmail refused the send`

**Node Details:**

- **`sending on?`**
  - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional Router). Checks if sending is enabled and execution is within rate limits.
  - *Key Expressions/Variables:* Condition: `{{ $('config').first().json.send_enabled && !$json.held_by_cap }}`.
  - *Inputs / Outputs:* Inputs: `address the reply`. Outputs: `live reply?` (True), `STOP: preview - reply not sent` (False).

- **`live reply?`**
  - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional Router). Determines whether to send a live customer reply or a test copy.
  - *Key Expressions/Variables:* Condition: `{{ $json.live }}`.
  - *Inputs / Outputs:* Inputs: `sending on?` (True). Outputs: `[cred] Gmail - reply in the thread` (True), `[cred] Gmail - send the test copy` (False).

- **`[cred] Gmail - reply in the thread`**
  - *Type & Technical Role:* `n8n-nodes-base.gmail` (Gmail Integration Node). Sends a reply message attached to the customer's original Gmail thread.
  - *Configuration Choices:* Resource: `message`, Operation: `reply`. Uses Gmail credentials, attaching message text and original message ID to maintain thread continuity.
  - *Inputs / Outputs:* Inputs: `live reply?` (True). Outputs: `STOP: replied in the thread`, `STOP: Gmail refused the send` (Error branch).

- **`STOP: replied in the thread`**
  - *Type & Technical Role:* `n8n-nodes-base.code` (Terminal Log Node). Logs successfully sent live thread replies.
  - *Inputs / Outputs:* Inputs: `[cred] Gmail - reply in the thread`. Outputs: None.

- **`[cred] Gmail - send the test copy`**
  - *Type & Technical Role:* `n8n-nodes-base.gmail` (Gmail Integration Node). Sends test-mode output to a designated test email address.
  - *Configuration Choices:* Sends formatted message with `[TEST]` prefix using Gmail credentials.
  - *Inputs / Outputs:* Inputs: `live reply?` (False). Outputs: `STOP: test copy sent`, `STOP: Gmail refused the send` (Error branch).

- **`STOP: test copy sent`**
  - *Type & Technical Role:* `n8n-nodes-base.code` (Terminal Log Node). Logs successful dispatch of test copies.
  - *Inputs / Outputs:* Inputs: `[cred] Gmail - send the test copy`. Outputs: None.

- **`STOP: preview - reply not sent`**
  - *Type & Technical Role:* `n8n-nodes-base.code` (Terminal Log Node). Displays built reply payloads in preview mode without transmitting them externally.
  - *Inputs / Outputs:* Inputs: `sending on?` (False). Outputs: None.

- **`prepare the escalation`**
  - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript Data Transformation). Aggregates context for cases requiring human intervention (unmatched queries, low confidence, handoff intents, or parsing failures) into a detailed alert payload.
  - *Inputs / Outputs:* Inputs: Multiple upstream `if` false branches. Outputs: `alerts on?`.

- **`alerts on?`**
  - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional Router). Checks if escalation alerts are enabled and within rate limits.
  - *Key Expressions/Variables:* Condition: `{{ $('config').first().json.alert_enabled && !$json.held_by_cap }}`.
  - *Inputs / Outputs:* Inputs: `prepare the escalation`. Outputs: `[cred] Gmail - alert the support team` (True), `ESC-preview` (False).

- **`[cred] Gmail - alert the support team`**
  - *Type & Technical Role:* `n8n-nodes-base.gmail` (Gmail Integration Node). Sends an internal alert email to the support team with customer details, thread links, and retrieved knowledge excerpts.
  - *Inputs / Outputs:* Inputs: `alerts on?` (True). Outputs: `STOP: escalated to a person`, `STOP: Gmail refused the send` (Error branch).

- **`STOP: escalated to a person`**
  - *Type & Technical Role:* `n8n-nodes-base.code` (Terminal Log Node). Logs successful internal team escalation alerts.
  - *Inputs / Outputs:* Inputs: `[cred] Gmail - alert the support team`. Outputs: None.

- **`ESC-preview`**
  - *Type & Technical Role:* `n8n-nodes-base.code` (Terminal Log Node). Displays un-sent escalation alert payloads in preview mode.
  - *Inputs / Outputs:* Inputs: `alerts on?` (False). Outputs: None.

- **`STOP: Gmail refused the send`**
  - *Type & Technical Role:* `n8n-nodes-base.code` (Terminal Log Node). Captures Gmail API transmission errors (e.g., authentication failure, quota limits) to ensure failed sends are visible for manual handling.
  - *Inputs / Outputs:* Inputs: Gmail error output connectors. Outputs: None.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `[cred] Gmail - new email in the inbox` | `n8n-nodes-base.gmailTrigger` | Polls the support inbox every minute for unread messages. | None | `config` | ## Answer support emails in Gmail from a Pinecone knowledge base with OpenAI... |
| `Run the demo emails` | `n8n-nodes-base.manualTrigger` | Manually triggers the workflow for demo execution. | None | `config` | ## Answer support emails in Gmail from a Pinecone knowledge base with OpenAI... |
| `config` | `n8n-nodes-base.code` | Centralizes workflow settings, parameters, and demo datasets. | `[cred] Gmail - new email in the inbox`, `Run the demo emails` | `read the email` | ## Answer support emails in Gmail from a Pinecone knowledge base with OpenAI... |
| `read the email` | `n8n-nodes-base.code` | Parses and normalizes incoming raw email payloads. | `config` | `a readable email?` | ### 1. Read the email<br><br>`read the email` keeps the sender, the subject, only the new text of the body, and the Gmail message and thread ids the reply must attach to... |
| `a readable email?` | `n8n-nodes-base.if` | Filters out empty, self-sent, or bulk unreadable emails. | `read the email` | `OpenAI configured?`, `STOP: unreadable email` | ### 1. Read the email<br><br>`read the email` keeps the sender, the subject, only the new text of the body, and the Gmail message and thread ids the reply must attach to... |
| `STOP: unreadable email` | `n8n-nodes-base.code` | Logs filtered unreadable or ignored emails. | `a readable email?` | None | ### 1. Read the email<br><br>`read the email` keeps the sender, the subject, only the new text of the body, and the Gmail message and thread ids the reply must attach to... |
| `OpenAI configured?` | `n8n-nodes-base.if` | Checks whether OpenAI integration is enabled. | `a readable email?` | `build the classification request`, `classify by keywords` | ### 2. Classify, and decide who answers<br><br>`[cred] OpenAI - classify the email` returns typed JSON: category, intent, handoff, confidence... |
| `build the classification request` | `n8n-nodes-base.code` | Formats OpenAI chat payload for email classification. | `OpenAI configured?` | `[cred] OpenAI - classify the email` | ### 2. Classify, and decide who answers<br><br>`[cred] OpenAI - classify the email` returns typed JSON: category, intent, handoff, confidence... |
| `[cred] OpenAI - classify the email` | `n8n-nodes-base.httpRequest` | Calls OpenAI API to classify incoming email category and intent. | `build the classification request` | `apply the AI classification` | ### 2. Classify, and decide who answers<br><br>`[cred] OpenAI - classify the email` returns typed JSON: category, intent, handoff, confidence... |
| `apply the AI classification` | `n8n-nodes-base.code` | Parses AI classification and falls back to keywords on failure. | `[cred] OpenAI - classify the email` | `a support request?` | ### 2. Classify, and decide who answers<br><br>`[cred] OpenAI - classify the email` returns typed JSON: category, intent, handoff, confidence... |
| `classify by keywords` | `n8n-nodes-base.code` | Classifies emails using local keyword rules. | `OpenAI configured?` | `a support request?` | ### 2. Classify, and decide who answers<br><br>`[cred] OpenAI - classify the email` returns typed JSON: category, intent, handoff, confidence... |
| `a support request?` | `n8n-nodes-base.if` | Routes non-support emails out of the workflow. | `apply the AI classification`, `classify by keywords` | `needs a person?`, `STOP: not a support email` | ### 2. Classify, and decide who answers<br><br>`[cred] OpenAI - classify the email` returns typed JSON: category, intent, handoff, confidence... |
| `STOP: not a support email` | `n8n-nodes-base.code` | Logs filtered-out non-support emails left in the inbox. | `a support request?` | None | ### 2. Classify, and decide who answers<br><br>`[cred] OpenAI - classify the email` returns typed JSON: category, intent, handoff, confidence... |
| `needs a person?` | `n8n-nodes-base.if` | Directs high-risk intents or uncertain classifications to humans. | `a support request?` | `prepare the escalation`, `knowledge search configured?` | ### 2. Classify, and decide who answers<br><br>`[cred] OpenAI - classify the email` returns typed JSON: category, intent, handoff, confidence... |
| `knowledge search configured?` | `n8n-nodes-base.if` | Checks whether Pinecone vector search is configured. | `needs a person?` | `build the embedding request`, `search the demo knowledge base` | ### 3. Retrieve, and refuse to guess<br><br>The question is embedded with the same model the knowledge base was loaded with, and `[cred] Pinecone - find matching knowledge` returns the top matches with their text... |
| `build the embedding request` | `n8n-nodes-base.code` | Prepares query parameters for OpenAI embedding generation. | `knowledge search configured?` | `[cred] OpenAI - embed the question` | ### 3. Retrieve, and refuse to guess<br><br>The question is embedded with the same model the knowledge base was loaded with, and `[cred] Pinecone - find matching knowledge` returns the top matches with their text... |
| `[cred] OpenAI - embed the question` | `n8n-nodes-base.httpRequest` | Calls OpenAI API to generate query vector embeddings. | `build the embedding request` | `build the Pinecone query` | ### 3. Retrieve, and refuse to guess<br><br>The question is embedded with the same model the knowledge base was loaded with, and `[cred] Pinecone - find matching knowledge` returns the top matches with their text... |
| `build the Pinecone query` | `n8n-nodes-base.code` | Formats vector query payload for the Pinecone index. | `[cred] OpenAI - embed the question` | `question embedded?` | ### 3. Retrieve, and refuse to guess<br><br>The question is embedded with the same model the knowledge base was loaded with, and `[cred] Pinecone - find matching knowledge` returns the top matches with their text... |
| `question embedded?` | `n8n-nodes-base.if` | Validates successful vector embedding generation. | `build the Pinecone query` | `[cred] Pinecone - find matching knowledge`, `prepare the escalation` | ### 3. Retrieve, and refuse to guess<br><br>The question is embedded with the same model the knowledge base was loaded with, and `[cred] Pinecone - find matching knowledge` returns the top matches with their text... |
| `[cred] Pinecone - find matching knowledge` | `n8n-nodes-base.httpRequest` | Queries Pinecone vector index for matching knowledge chunks. | `question embedded?` | `read the matches` | ### 3. Retrieve, and refuse to guess<br><br>The question is embedded with the same model the knowledge base was loaded with, and `[cred] Pinecone - find matching knowledge` returns the top matches with their text... |
| `read the matches` | `n8n-nodes-base.code` | Parses Pinecone matches and evaluates match scores. | `[cred] Pinecone - find matching knowledge` | `enough knowledge?` | ### 3. Retrieve, and refuse to guess<br><br>The question is embedded with the same model the knowledge base was loaded with, and `[cred] Pinecone - find matching knowledge` returns the top matches with their text... |
| `search the demo knowledge base` | `n8n-nodes-base.code` | Performs local word-overlap knowledge search for demo mode. | `knowledge search configured?` | `enough knowledge?` | ### 3. Retrieve, and refuse to guess<br><br>The question is embedded with the same model the knowledge base was loaded with, and `[cred] Pinecone - find matching knowledge` returns the top matches with their text... |
| `enough knowledge?` | `n8n-nodes-base.if` | Gate verifying if retrieved knowledge meets minimum scores. | `read the matches`, `search the demo knowledge base` | `OpenAI writes the reply?`, `prepare the escalation` | ### 3. Retrieve, and refuse to guess<br><br>The question is embedded with the same model the knowledge base was loaded with, and `[cred] Pinecone - find matching knowledge` returns the top matches with their text... |
| `OpenAI writes the reply?` | `n8n-nodes-base.if` | Checks if OpenAI reply generation is enabled. | `enough knowledge?` | `build the reply request`, `write the reply from the knowledge` | ### 4. Write only from the excerpts<br><br>`[cred] OpenAI - write the reply` sees the email and the excerpts, nothing else, must list the excerpts it used, and may refuse... |
| `build the reply request` | `n8n-nodes-base.code` | Formats OpenAI chat request for context-bound reply generation. | `OpenAI writes the reply?` | `[cred] OpenAI - write the reply` | ### 4. Write only from the excerpts<br><br>`[cred] OpenAI - write the reply` sees the email and the excerpts, nothing else, must list the excerpts it used, and may refuse... |
| `[cred] OpenAI - write the reply` | `n8n-nodes-base.httpRequest` | Calls OpenAI API to generate context-bound response draft. | `build the reply request` | `apply the AI reply` | ### 4. Write only from the excerpts<br><br>`[cred] OpenAI - write the reply` sees the email and the excerpts, nothing else, must list the excerpts it used, and may refuse... |
| `apply the AI reply` | `n8n-nodes-base.code` | Validates LLM draft for strict source citations and refunusal flags. | `[cred] OpenAI - write the reply` | `a reply we can send?` | ### 4. Write only from the excerpts<br><br>`[cred] OpenAI - write the reply` sees the email and the excerpts, nothing else, must list the excerpts it used, and may refuse... |
| `write the reply from the knowledge` | `n8n-nodes-base.code` | Assembles template reply from knowledge base when OpenAI is disabled. | `OpenAI writes the reply?` | `a reply we can send?` | ### 4. Write only from the excerpts<br><br>`[cred] OpenAI - write the reply` sees the email and the excerpts, nothing else, must list the excerpts it used, and may refuse... |
| `a reply we can send?` | `n8n-nodes-base.if` | Verifies that generated reply passed all safety and content checks. | `apply the AI reply`, `write the reply from the knowledge` | `address the reply`, `prepare the escalation` | ### 4. Write only from the excerpts<br><br>`[cred] OpenAI - write the reply` sees the email and the excerpts, nothing else, must list the excerpts it used, and may refuse... |
| `address the reply` | `n8n-nodes-base.code` | Configures mode-specific reply dispatch and enforces rate caps. | `a reply we can send?` | `sending on?` | ### 5. Reply in the customer's thread<br><br>`address the reply` decides preview, test or live and enforces `REPLY_CAP_PER_RUN`... |
| `sending on?` | `n8n-nodes-base.if` | Checks if live/test sending is enabled and within run caps. | `address the reply` | `live reply?`, `STOP: preview - reply not sent` | ### 5. Reply in the customer's thread<br><br>`address the reply` decides preview, test or live and enforces `REPLY_CAP_PER_RUN`... |
| `live reply?` | `n8n-nodes-base.if` | Routes output between live customer threads and test email addresses. | `sending on?` | `[cred] Gmail - reply in the thread`, `[cred] Gmail - send the test copy` | ### 5. Reply in the customer's thread<br><br>`address the reply` decides preview, test or live and enforces `REPLY_CAP_PER_RUN`... |
| `[cred] Gmail - reply in the thread` | `n8n-nodes-base.gmail` | Sends automated reply inside the customer's Gmail thread. | `live reply?` | `STOP: replied in the thread`, `STOP: Gmail refused the send` | ### 5. Reply in the customer's thread<br><br>`address the reply` decides preview, test or live and enforces `REPLY_CAP_PER_RUN`... |
| `STOP: replied in the thread` | `n8n-nodes-base.code` | Logs successful live customer thread replies. | `[cred] Gmail - reply in the thread` | None | ### 5. Reply in the customer's thread<br><br>`address the reply` decides preview, test or live and enforces `REPLY_CAP_PER_RUN`... |
| `[cred] Gmail - send the test copy` | `n8n-nodes-base.gmail` | Sends test-mode reply copies to designated test email addresses. | `live reply?` | `STOP: test copy sent`, `STOP: Gmail refused the send` | ### 5. Reply in the customer's thread<br><br>`address the reply` decides preview, test or live and enforces `REPLY_CAP_PER_RUN`... |
| `STOP: test copy sent` | `n8n-nodes-base.code` | Logs successful dispatch of test reply copies. | `[cred] Gmail - send the test copy` | None | ### 5. Reply in the customer's thread<br><br>`address the reply` decides preview, test or live and enforces `REPLY_CAP_PER_RUN`... |
| `STOP: preview - reply not sent` | `n8n-nodes-base.code` | Displays unsent reply payloads in preview mode. | `sending on?` | None | ### 5. Reply in the customer's thread<br><br>`address the reply` decides preview, test or live and enforces `REPLY_CAP_PER_RUN`... |
| `prepare the escalation` | `n8n-nodes-base.code` | Aggregates escalation payload for human support handlers. | Multiple upstream nodes | `alerts on?` | ## Nothing goes out until you say so<br><br>This ships with `TEST_RUN = true` and a blank `TEST_EMAIL`... |
| `alerts on?` | `n8n-nodes-base.if` | Checks if escalation alerting is enabled. | `prepare the escalation` | `[cred] Gmail - alert the support team`, `ESC-preview` | ## Nothing goes out until you say so<br><br>This ships with `TEST_RUN = true` and a blank `TEST_EMAIL`... |
| `[cred] Gmail - alert the support team` | `n8n-nodes-base.gmail` | Sends internal escalation alert email to support team. | `alerts on?` | `STOP: escalated to a person`, `STOP: Gmail refused the send` | ## Nothing goes out until you say so<br><br>This ships with `TEST_RUN = true` and a blank `TEST_EMAIL`... |
| `STOP: escalated to a person` | `n8n-nodes-base.code` | Logs successful team escalation alerts. | `[cred] Gmail - alert the support team` | None | ## Nothing goes out until you say so<br><br>This ships with `TEST_RUN = true` and a blank `TEST_EMAIL`... |
| `ESC-preview` | `n8n-nodes-base.code` | Displays unsent escalation alerts in preview mode. | `alerts on?` | None | ## Top Sticky Note Header / General Description |
| `STOP: Gmail refused the send` | `n8n-nodes-base.code` | Captures Gmail API transmission errors for failed dispatches. | Gmail error outputs | None | ### 5. Reply in the customer's thread<br><br>`address the reply` decides preview, test or live and enforces `REPLY_CAP_PER_RUN`... |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow in n8n manually without relying on the JSON export:

1. **Create Trigger Nodes:**
   - Add a **Gmail Trigger** node (`[cred] Gmail - new email in the inbox`): Set filter query to `in:inbox -from:me`, readStatus to `unread`, and polling frequency to every minute. Ensure "Simplify" is turned off.
   - Add a **Manual Trigger** node (`Run the demo emails`): Leave default parameters.

2. **Setup Configuration Node:**
   - Add a **Code** node (`config`). Paste the configuration JavaScript code block defining company name, signature, confidence thresholds (`MIN_CLASSIFY_CONFIDENCE = 0.6`, `MIN_MATCH_SCORE = 0.75`), keyword lists, and demo datasets.
   - Connect both triggers (`[cred] Gmail - new email in the inbox` and `Run the demo emails`) into `config`.

3. **Build Input Parsing & Validation Block:**
   - Add a **Code** node (`read the email`) connected from `config` to normalize headers, message IDs, thread IDs, and extract clean body text.
   - Add an **If** node (`a readable email?`): Set condition `{{ $json.usable === true }}`.
   - Connect the True branch to `OpenAI configured?` and the False branch to a **Code** node (`STOP: unreadable email`).

4. **Build Classification & Routing Block:**
   - Add an **If** node (`OpenAI configured?`): Condition `{{ $('config').first().json.openai_enabled }}`.
   - *True Path:*
     - Add a **Code** node (`build the classification request`) to format the JSON chat prompt.
     - Add an **HTTP Request** node (`[cred] OpenAI - classify the email`): POST to `https://api.openai.com/v1/chat/completions` using OpenAI API credentials. Configure JSON body, error continuation, and max retries (3).
     - Add a **Code** node (`apply the AI classification`) to parse responses and handle keyword fallbacks.
   - *False Path:*
     - Add a **Code** node (`classify by keywords`) to run local keyword categorization.
   - Connect both classification paths into an **If** node (`a support request?`): Condition `{{ $json.route !== 'not_support' }}`.
   - Connect the False branch to a **Code** node (`STOP: not a support email`).
   - Connect the True branch to an **If** node (`needs a person?`): Condition `{{ $json.route === 'person' }}`.
   - Connect the True branch of `needs a person?` to the escalation pipeline. Connect the False branch to the knowledge retrieval block.

5. **Build Knowledge Retrieval (RAG) Block:**
   - Add an **If** node (`knowledge search configured?`): Condition `{{ $('config').first().json.retrieval_enabled }}`.
   - *True Path:*
     - Add a **Code** node (`build the embedding request`) for text embeddings.
     - Add an **HTTP Request** node (`[cred] OpenAI - embed the question`): POST to `https://api.openai.com/v1/embeddings` using OpenAI credentials.
     - Add a **Code** node (`build the Pinecone query`) to format vector search parameters.
     - Add an **If** node (`question embedded?`): Condition `{{ $json.embedded }}`.
     - Add an **HTTP Request** node (`[cred] Pinecone - find matching knowledge`): POST to `{{ $('config').first().json.pinecone_index_host + '/query' }}`. Authenticate using Generic Header Auth (`Api-Key`) with header `X-Pinecone-API-Version: 2025-01`.
     - Add a **Code** node (`read the matches`) to parse chunk similarity scores.
   - *False Path:*
     - Add a **Code** node (`search the demo knowledge base`) to perform local word-overlap searches.
   - Connect both retrieval paths into an **If** node (`enough knowledge?`): Condition `{{ $json.knowledge_ok }}`.
   - Connect the False branch to the escalation pipeline.

6. **Build Answer Generation & Safety Gates Block:**
   - Add an **If** node (`OpenAI writes the reply?`): Condition `{{ $('config').first().json.openai_enabled }}`.
   - *True Path:*
     - Add a **Code** node (`build the reply request`) to format prompt constraints and excerpt citations.
     - Add an **HTTP Request** node (`[cred] OpenAI - write the reply`): POST to `https://api.openai.com/v1/chat/completions` using OpenAI credentials.
     - Add a **Code** node (`apply the AI reply`) to validate drafts.
   - *False Path:*
     - Add a **Code** node (`write the reply from the knowledge`) to assemble template replies.
   - Connect both reply paths into an **If** node (`a reply we can send?`): Condition `{{ $json.reply_ok }}`.
   - Connect the False branch to the escalation pipeline. Connect the True branch to a **Code** node (`address the reply`) to enforce run caps and set mode routing.
   - Add an **If** node (`sending on?`): Condition `{{ $('config').first().json.send_enabled && !$json.held_by_cap }}`. Connect False branch to `STOP: preview - reply not sent`.

7. **Build Dispatch & Escalation Block:**
   - From `sending on?` (True), add an **If** node (`live reply?`): Condition `{{ $json.live }}`.
   - *True Branch (Live):*
     - Add a **Gmail** node (`[cred] Gmail - reply in the thread`): Resource `message`, Operation `reply`. Set Message ID to `{{ $json.reply_to_message_id }}` and message content to `{{ $json.reply_text }}`. Connect success to `STOP: replied in the thread` and error output to `STOP: Gmail refused the send`.
   - *False Branch (Test):*
     - Add a **Gmail** node (`[cred] Gmail - send the test copy`): Send to `{{ $json.test_to }}`, subject `{{ $json.test_subject }}`, message `{{ $json.test_body }}`. Connect success to `STOP: test copy sent` and error output to `STOP: Gmail refused the send`.
   - *Escalation Pipeline:*
     - Route all escalation triggers (handoff intents, low confidence, insufficient knowledge, failed validation) into a **Code** node (`prepare the escalation`).
     - Add an **If** node (`alerts on?`): Condition `{{ $('config').first().json.alert_enabled && !$json.held_by_cap }}`.
   - *Alert True Path:*
     - Add a **Gmail** node (`[cred] Gmail - alert the support team`): Send to `{{ $json.alert_to }}`, subject `{{ $json.alert_subject }}`, message `{{ $json.alert_body }}`. Connect success to `STOP: escalated to a person` and error output to `STOP: Gmail refused the send`.
   - *Alert False Path:*
     - Add a **Code** node (`ESC-preview`) to display preview alerts.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Designed for support teams and agencies handling repetitive customer email inquiries by automating answers via RAG while safely escalating complex or sensitive cases to human personnel. | Target Use Case & Architecture |
| Ships in preview mode (`TEST_RUN = true`, `TEST_EMAIL` blank) by default to ensure zero external transmissions until verified. | Safety Switch & Testing Guidelines |