Diagnose workflow failures with Google Gemini, Google Sheets and Slack

https://n8nworkflows.xyz/workflows/diagnose-workflow-failures-with-google-gemini--google-sheets-and-slack-17693


# Diagnose workflow failures with Google Gemini, Google Sheets and Slack

### 1. Workflow Overview

This workflow acts as a centralized error-handling safety net for n8n automation architectures. Its primary purpose is to automatically capture failures from any monitored n8n workflow, process the failure data through Google Gemini to generate a plain-language diagnosis and actionable resolution steps, log the structured incident data into a Google Sheet, and broadcast a rich alert message to a designated Slack channel.

The execution logic is divided into three functional blocks:
- **1.1 Failure Reception & Extraction:** Captures unhandled exceptions from connected workflows via the error trigger and cleans raw execution payloads into standardized metadata.
- **1.2 AI-Powered Analysis:** Leverages Google Gemini and structured output parsing to categorize the failure, assign a severity score, and formulate human-readable explanations and n8n-specific remediation steps.
- **1.3 Data Persistence & Notification:** Appends the structured diagnostic record to a tracking spreadsheet and dispatches an alert card to Slack containing execution links.

---

### 2. Block-by-Block Analysis

#### 2.1 Failure Reception & Extraction
**Overview:**  
This block listens for runtime failures across the n8n instance, intercepting the raw execution context and extracting essential properties like workflow names, execution URLs, failing nodes, and error messages.

**Nodes Involved:**
- `On Workflow Error`
- `Extract Error Details`

**Node Details:**
- **On Workflow Error**
  - **Type & Technical Role:** `n8n-nodes-base.errorTrigger` (v1). Acts as the primary entry point when any linked workflow crashes.
  - **Configuration Choices:** Default trigger configuration.
  - **Key Expressions / Variables:** None (receives global execution data payload).
  - **Input / Output Connections:** Input: None (Trigger); Output: Connects to `Extract Error Details`.
  - **Version Requirements:** v1.
  - **Edge Cases & Failures:** Will not fire if target workflows do not explicitly reference this workflow in their *Settings > Error Workflow* configuration.

- **Extract Error Details**
  - **Type & Technical Role:** `n8n-nodes-base.code` (v2). Runs once per item to normalize nested error objects.
  - **Configuration Choices:** JavaScript execution mode (`runOnceForEachItem`). Truncates message and description strings to 500 characters to prevent payload overflow downstream.
  - **Key Expressions / Variables:** 
    - `const w = $json.workflow || {};`
    - `const ex = $json.execution || {};`
    - `const err = ex.error || $json.error || {};`
  - **Input / Output Connections:** Input: `On Workflow Error`; Output: Connects to `Diagnose Error with Gemini`.
  - **Version Requirements:** v2.
  - **Edge Cases & Failures:** Handles missing nested keys gracefully using default fallback strings (`'Unknown workflow'`, `'No message'`).

---

#### 2.2 AI-Powered Analysis
**Overview:**  
This block interfaces with Google Gemini to analyze the normalized error context, enforcing a structured JSON output schema that categorizes the failure, evaluates its severity, and drafts an n8n-specific solution.

**Nodes Involved:**
- `Diagnose Error with Gemini`
- `Google Gemini Chat Model`
- `Diagnosis Parser`
- `Shape Diagnosis`

**Node Details:**
- **Diagnose Error with Gemini**
  - **Type & Technical Role:** `@n8n/n8n-nodes-langchain.chainLlm` (v1.9). Executes an advanced prompt chain utilizing a connected Language Model and Output Parser.
  - **Configuration Choices:** Error handling is set to `continueRegularOutput` to ensure that if the AI times out or fails, execution proceeds smoothly without halting the pipeline.
  - **Key Expressions / Variables:** 
    - Prompt references: `{{ $json.workflowName }}`, `{{ $json.lastNode }}`, `{{ $json.errorMessage }}`, `{{ $json.errorDescription }}`.
  - **Input / Output Connections:** Input: `Extract Error Details` (Main), `Google Gemini Chat Model` (AI Language Model), `Diagnosis Parser` (AI Output Parser); Output: Connects to `Shape Diagnosis`.
  - **Version Requirements:** v1.9+.
  - **Edge Cases & Failures:** API rate limits or connectivity drops with Google Gemini; mitigated by the `continueRegularOutput` setting.

- **Google Gemini Chat Model**
  - **Type & Technical Role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` (v1.1). Provides the underlying LLM engine for the LangChain node.
  - **Configuration Choices:** Uses model `models/gemini-3.1-flash-lite` with a low temperature of `0.1` to maximize determinism and factual adherence.
  - **Key Expressions / Variables:** None.
  - **Input / Output Connections:** Output: Connects to `Diagnose Error with Gemini` via `ai_languageModel`.
  - **Credentials:** Requires `googlePalmApi` credentials.

- **Diagnosis Parser**
  - **Type & Technical Role:** `@n8n/n8n-nodes-langchain.outputParserStructured` (v1.3). Enforces strict JSON formatting matching a predefined schema.
  - **Configuration Choices:** Configured with a JSON schema example containing keys: `category`, `severity`, `plain_explanation`, `likely_cause`, and `suggested_fix`.
  - **Input / Output Connections:** Output: Connects to `Diagnose Error with Gemini` via `ai_outputParser`.

- **Shape Diagnosis**
  - **Type & Technical Role:** `n8n-nodes-base.code` (v2). Merges raw execution metadata with AI-generated diagnosis properties, providing robust default fallbacks if the AI output is malformed or missing.
  - **Configuration Choices:** JavaScript execution mode (`runOnceForEachItem`).
  - **Key Expressions / Variables:** 
    - `const meta = $('Extract Error Details').item.json;`
    - Fallback logic for `plainExplanation`, `likelyCause`, and `suggestedFix`.
  - **Input / Output Connections:** Input: `Diagnose Error with Gemini`; Output: Connects to `Log Error`.

---

#### 2.3 Data Persistence & Notification
**Overview:**  
This block records the final structured incident payload into a centralized Google Sheet for historical tracking and sends an actionable notification card to a Slack channel.

**Nodes Involved:**
- `Log Error`
- `Alert Slack`

**Node Details:**
- **Log Error**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheets` (v4.7). Appends a new row to a designated spreadsheet tracking historical failures.
  - **Configuration Choices:** Operation set to `append`. Document and sheet names must be configured via parameters.
  - **Input / Output Connections:** Input: `Shape Diagnosis`; Output: Connects to `Alert Slack`.
  - **Credentials:** Requires `googleSheetsOAuth2Api` credentials.
  - **Edge Cases & Failures:** Missing sheet tabs (e.g., missing "Errors" sheet) will cause row insertion failures.

- **Alert Slack**
  - **Type & Technical Role:** `n8n-nodes-base.slack` (v2.5). Sends a formatted warning notification card to a target workspace channel.
  - **Configuration Choices:** Channel selection via list parameter targeting a specific ID/name (`#n8n-alerts`). Uses OAuth2 authentication.
  - **Key Expressions / Variables:** 
    - Utilizes expressions referencing `$('Shape Diagnosis').item.json` for workflow name, node name, category, severity, plain explanation, likely cause, suggested fix, and execution URL.
  - **Input / Output Connections:** Input: `Log Error`; Output: None (Terminal node).
  - **Credentials:** Requires `slackOAuth2Api` credentials.
  - **Edge Cases & Failures:** Insufficient bot permissions to post messages in the targeted channel will cause the node to fail.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| On Workflow Error | `n8n-nodes-base.errorTrigger` | Captures unhandled workflow failures instance-wide | None | Extract Error Details | AI error handler: diagnose any workflow failure and alert Slack in plain language...<br><br>## 1. Catch the failure<br>The Error Trigger fires when any workflow that points here fails. A Code node extracts the workflow, node and error. |
| Extract Error Details | `n8n-nodes-base.code` | Normalizes and truncates raw error execution context | On Workflow Error | Diagnose Error with Gemini | AI error handler: diagnose any workflow failure and alert Slack in plain language...<br><br>## 1. Catch the failure<br>The Error Trigger fires when any workflow that points here fails. A Code node extracts the workflow, node and error. |
| Diagnose Error with Gemini | `@n8n/n8n-nodes-langchain.chainLlm` | Analyzes failure context using an LLM chain | Extract Error Details, Google Gemini Chat Model, Diagnosis Parser | Shape Diagnosis | AI error handler: diagnose any workflow failure and alert Slack in plain language...<br><br>## 2. Diagnose (Basic LLM Chain)<br>Gemini categorises the error, rates severity, and suggests a concrete fix in plain language. Fails soft to the raw error. |
| Google Gemini Chat Model | `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` | Provides the Gemini LLM engine | None | Diagnose Error with Gemini | AI error handler: diagnose any workflow failure and alert Slack in plain language...<br><br>## 2. Diagnose (Basic LLM Chain)<br>Gemini categorises the error, rates severity, and suggests a concrete fix in plain language. Fails soft to the raw error. |
| Diagnosis Parser | `@n8n/n8n-nodes-langchain.outputParserStructured` | Enforces structured JSON output formatting | None | Diagnose Error with Gemini | AI error handler: diagnose any workflow failure and alert Slack in plain language...<br><br>## 2. Diagnose (Basic LLM Chain)<br>Gemini categorises the error, rates severity, and suggests a concrete fix in plain language. Fails soft to the raw error. |
| Shape Diagnosis | `n8n-nodes-base.code` | Combines metadata with AI outputs and fallback handlers | Diagnose Error with Gemini | Log Error | AI error handler: diagnose any workflow failure and alert Slack in plain language...<br><br>## 2. Diagnose (Basic LLM Chain)<br>Gemini categorises the error, rates severity, and suggests a concrete fix in plain language. Fails soft to the raw error. |
| Log Error | `n8n-nodes-base.googleSheets` | Appends failure records to a tracking spreadsheet | Shape Diagnosis | Alert Slack | AI error handler: diagnose any workflow failure and alert Slack in plain language...<br><br>## 3. Log & alert<br>Log every failure to Sheets and post a Slack alert with the diagnosis and a link to the execution. |
| Alert Slack | `n8n-nodes-base.slack` | Sends a formatted diagnostic alert to a Slack channel | Log Error | None | AI error handler: diagnose any workflow failure and alert Slack in plain language...<br><br>## 3. Log & alert<br>Log every failure to Sheets and post a Slack alert with the diagnosis and a link to the execution. |
| Overview Sticky | `n8n-nodes-base.stickyNote` | Visual container documenting workflow architecture | None | None | AI error handler: diagnose any workflow failure and alert Slack in plain language... |
| S1 | `n8n-nodes-base.stickyNote` | Visual container for Block 1 | None | None | ## 1. Catch the failure<br>The Error Trigger fires when any workflow that points here fails. A Code node extracts the workflow, node and error. |
| S2 | `n8n-nodes-base.stickyNote` | Visual container for Block 2 | None | None | ## 2. Diagnose (Basic LLM Chain)<br>Gemini categorises the error, rates severity, and suggests a concrete fix in plain language. Fails soft to the raw error. |
| S3 | `n8n-nodes-base.stickyNote` | Visual container for Block 3 | None | None | ## 3. Log & alert<br>Log every failure to Sheets and post a Slack alert with the diagnosis and a link to the execution. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add the Error Trigger node:**
   - Type: `n8n-nodes-base.errorTrigger`
   - Name: `On Workflow Error`
3. **Add the Extract Code node:**
   - Type: `n8n-nodes-base.code`
   - Name: `Extract Error Details`
   - Mode: `runOnceForEachItem`
   - JavaScript Code:
     ```javascript
     const w = $json.workflow || {};
     const ex = $json.execution || {};
     const err = ex.error || $json.error || {};
     return {
       workflowName: w.name || 'Unknown workflow',
       workflowId: w.id || '',
       executionUrl: ex.url || '',
       lastNode: ex.lastNodeExecuted || 'Unknown node',
       errorMessage: (err.message || 'No message').toString().slice(0, 500),
       errorDescription: (err.description || '').toString().slice(0, 500)
     };
     ```
   - Connect `On Workflow Error` to `Extract Error Details`.
4. **Add the Google Gemini Chat Model sub-node:**
   - Type: `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`
   - Name: `Google Gemini Chat Model`
   - Model Name: `models/gemini-3.1-flash-lite`
   - Temperature: `0.1`
   - Credentials: Connect a valid `googlePalmApi` credential.
5. **Add the Diagnosis Output Parser sub-node:**
   - Type: `@n8n/n8n-nodes-langchain.outputParserStructured`
   - Name: `Diagnosis Parser`
   - JSON Schema Example:
     ```json
     {"category":"rate_limit","severity":"medium","plain_explanation":"The Stripe API is rejecting requests because too many were sent too quickly.","likely_cause":"No delay or batching between Stripe calls","suggested_fix":"Add a Wait node or reduce batch size, then retry"}
     ```
6. **Add the Basic LLM Chain node:**
   - Type: `@n8n/n8n-nodes-langchain.chainLlm`
   - Name: `Diagnose Error with Gemini`
   - Prompt Type: `Define`
   - Prompt Text:
     `=You are an n8n reliability engineer. Diagnose this workflow failure and respond with JSON. Categorise it (auth, rate_limit, timeout, bad_data, expression_error, api_error, config or other), give a severity of low, medium or high, a one-sentence plain-language explanation a non-engineer would understand, the most likely cause, and a concrete suggested fix in n8n terms.\n\nWorkflow: {{ $json.workflowName }}\nFailed node: {{ $json.lastNode }}\nError: {{ $json.errorMessage }}\nDescription: {{ $json.errorDescription }}`
   - On Error: `continueRegularOutput`
   - Connections: Connect `Extract Error Details` (Main), `Google Gemini Chat Model` (AI Language Model), and `Diagnosis Parser` (AI Output Parser) to this node.
7. **Add the Shape Diagnosis Code node:**
   - Type: `n8n-nodes-base.code`
   - Name: `Shape Diagnosis`
   - Mode: `runOnceForEachItem`
   - JavaScript Code:
     ```javascript
     const meta = $('Extract Error Details').item.json;
     const d = $json.output || {};
     return Object.assign({}, meta, { category: String(d.category || 'other').toLowerCase(), severity: String(d.severity || 'medium').toLowerCase(), plainExplanation: d.plain_explanation || meta.errorMessage, likelyCause: d.likely_cause || '', suggestedFix: d.suggested_fix || 'Review the execution manually.' });
     ```
   - Connect `Diagnose Error with Gemini` to `Shape Diagnosis`.
8. **Add the Google Sheets node:**
   - Type: `n8n-nodes-base.googleSheets`
   - Name: `Log Error`
   - Operation: `append`
   - Spreadsheet / Sheet Name: Select target document and configure sheet name to `Errors`.
   - Credentials: Connect a valid `googleSheetsOAuth2Api` credential.
   - Connect `Shape Diagnosis` to `Log Error`.
9. **Add the Slack node:**
   - Type: `n8n-nodes-base.slack`
   - Name: `Alert Slack`
   - Authentication: `oAuth2`
   - Select: `channel`
   - Channel ID: Select your target alert channel (e.g., `#n8n-alerts`).
   - Message Text:
     ```text
     =*:warning: Workflow failed: {{ $("Shape Diagnosis").item.json.workflowName }}*
     *Node:* {{ $("Shape Diagnosis").item.json.lastNode }}
     *Type:* {{ $("Shape Diagnosis").item.json.category }} ({{ $("Shape Diagnosis").item.json.severity }})
     *What happened:* {{ $("Shape Diagnosis").item.json.plainExplanation }}
     *Likely cause:* {{ $("Shape Diagnosis").item.json.likelyCause }}
     *Suggested fix:* {{ $("Shape Diagnosis").item.json.suggestedFix }}
     {{ $("Shape Diagnosis").item.json.executionUrl }}
     ```
   - Credentials: Connect a valid `slackOAuth2Api` credential.
   - Connect `Log Error` to `Alert Slack`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Set this one workflow as the Error Workflow for all your other workflows in their settings. | Global Error Handling Configuration |
| Route high-severity errors to PagerDuty or customize alerting behavior based on failure thresholds. | Customization Tip |