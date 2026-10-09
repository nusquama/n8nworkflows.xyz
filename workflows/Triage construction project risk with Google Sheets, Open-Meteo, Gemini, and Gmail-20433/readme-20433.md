Triage construction project risk with Google Sheets, Open-Meteo, Gemini, and Gmail

https://n8nworkflows.xyz/workflows/triage-construction-project-risk-with-google-sheets--open-meteo--gemini--and-gmail-20433


# Triage construction project risk with Google Sheets, Open-Meteo, Gemini, and Gmail

### 1. Workflow Overview

This workflow functions as an automated decision-support engine for commercial construction projects. Running every weekday morning, it assesses active projects by combining baseline schedules, real-time weather forecasts, financial budgets, procurement tracking, labor data, site reports, and open disputes. It computes deterministic risk metrics locally, queries an LLM for qualitative reasoning and validation, enforces decision-making guardrails, logs every assessment to a centralized spreadsheet, and automatically triggers email alerts to project managers when intervention or project suspension is required.

The system execution is organized into the following logical blocks:
- **1.1 Schedule Project Intake:** Initiates the automation on a cron schedule and queries Google Sheets for all active construction projects.
- **1.2 Gather Operating Inputs:** Fetches real-time 7-day weather forecasts via HTTP and loads supporting operational data (budgets, procurement, and labor requirements) from corresponding Google Sheets tabs.
- **1.3 Compile Risk Context:** Pulls site inspection logs and open disputes, aggregates all data points into a deterministic risk-scoring engine, and limits batch sizes for processing.
- **1.4 Generate Final Decision:** Passes the computed risk context through a Gemini-powered language model chain using structured output parsing, then applies validation guardrails to prevent unverified overrides.
- **1.5 Log and Alert:** Records the final decision and operational context to an audit sheet, filters out projects marked as "CONTINUE", and dispatches formatted email notifications to project managers for items requiring intervention or review.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule Project Intake
- **Overview:** Triggers the workflow execution at 7:00 AM from Monday through Friday and pulls the initial dataset of active construction projects from a specified Google Sheet.
- **Nodes Involved:** 
  - `When Weekday at 7AM`
  - `Read Projects from Sheets`
- **Node Details:**
  - **When Weekday at 7AM**
    - *Type and Technical Role:* `n8n-nodes-base.scheduleTrigger` (Schedule Trigger)
    - *Configuration:* Cron expression configured for `0 7 * * 1-5`.
    - *Key Expressions:* None.
    - *Input/Output:* Output connects to `Read Projects from Sheets`.
    - *Edge Cases/Failures:* Timezone discrepancies if the n8n instance host time differs from local expectations.
  - **Read Projects from Sheets**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Google Sheets Reader)
    - *Configuration:* Reads from document ID `1YDO5pfSMqem58WqDcFu5212rmKZQIlSVXHuL9P9gEt4`, tab `Projects`, filtering rows where `status` equals `Active`. Authentication via Google Service Account.
    - *Key Expressions:* None.
    - *Input/Output:* Input from schedule trigger; output connects to `Fetch Weather Forecast`.
    - *Edge Cases/Failures:* Authentication revocation, API rate limits, or missing columns (`status`, `latitude`, `longitude`) causing downstream parsing errors.

#### 2.2 Gather Operating Inputs
- **Overview:** Iterates through active projects to query external meteorological forecasts via API, then sequentially loads financial, procurement, and labor allocation details from auxiliary spreadsheet tabs.
- **Nodes Involved:**
  - `Fetch Weather Forecast`
  - `Read Budget from Sheets`
  - `Read Procurement Details`
  - `Read Labor Data Sheets`
- **Node Details:**
  - **Fetch Weather Forecast**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (HTTP Request)
    - *Configuration:* GET request to Open-Meteo API with a 15-second timeout. Configured with error handling set to `continueRegularOutput`.
    - *Key Expressions:* URL dynamically builds query parameters using `{{ $json.latitude }}` and `{{ $json.longitude }}`.
    - *Input/Output:* Input from `Read Projects from Sheets`; output connects to `Read Budget from Sheets`.
    - *Edge Cases/Failures:* API timeouts or invalid geographic coordinates; mitigated via error continuation allowing the workflow to process weather warnings internally.
  - **Read Budget from Sheets**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Google Sheets Reader)
    - *Configuration:* Reads tab `Budget_Accounting` with `executeOnce` enabled.
    - *Key Expressions:* None.
    - *Input/Output:* Input from `Fetch Weather Forecast`; output connects to `Read Procurement Details`.
  - **Read Procurement Details**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Google Sheets Reader)
    - *Configuration:* Reads tab `Procurement` with `executeOnce` enabled.
    - *Key Expressions:* None.
    - *Input/Output:* Input from `Read Budget from Sheets`; output connects to `Read Labor Data Sheets`.
  - **Read Labor Data Sheets**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Google Sheets Reader)
    - *Configuration:* Reads tab `Labor` with `executeOnce` enabled.
    - *Key Expressions:* None.
    - *Input/Output:* Input from `Read Procurement Details`; output connects to `Read Site Reports Sheets` (in the next block).

#### 2.3 Compile Risk Context
- **Overview:** Incorporates site inspection reports and dispute registers, computes comprehensive risk and schedule variance scores via custom JavaScript, and restricts the processing batch size.
- **Nodes Involved:**
  - `Read Site Reports Sheets`
  - `Read Disputes from Sheets`
  - `Calculate Risk Scores`
  - `Limit to 10 Entries`
- **Node Details:**
  - **Read Site Reports Sheets**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Google Sheets Reader)
    - *Configuration:* Reads tab `Site_Reports` with `executeOnce` enabled.
    - *Key Expressions:* None.
    - *Input/Output:* Input from `Read Labor Data Sheets`; output connects to `Read Disputes from Sheets`.
  - **Read Disputes from Sheets**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Google Sheets Reader)
    - *Configuration:* Reads tab `Disputes` with `executeOnce` enabled.
    - *Key Expressions:* None.
    - *Input/Output:* Input from `Read Site Reports Sheets`; output connects to `Calculate Risk Scores`.
  - **Calculate Risk Scores**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript Code Execution)
    - *Configuration:* Executes custom logic to parse dates, evaluate variance formulas, compute Earned Value Management (EVM) metrics, determine worst-case levers, and compile structured context strings.
    - *Key Expressions:* References preceding nodes using `$()`.
    - *Input/Output:* Input from `Read Disputes from Sheets`; output connects to `Limit to 10 Entries`.
    - *Edge Cases/Failures:* Malformed date strings or missing numeric values handled via fallback helper functions (`num`, `parseD`).
  - **Limit to 10 Entries**
    - *Type and Technical Role:* `n8n-nodes-base.limit` (Item Limiter)
    - *Configuration:* Limits execution throughput to safeguard downstream API rate boundaries.
    - *Key Expressions:* None.
    - *Input/Output:* Input from `Calculate Risk Scores`; output connects to `Execute Decision Making Chain`.

#### 2.4 Generate Final Decision
- **Overview:** Feeds the structured risk context into an LLM chain powered by Google Gemini, parses the output against a strict JSON schema, and applies algorithmic guardrails to validate the final escalation decision.
- **Nodes Involved:**
  - `Execute Decision Making Chain`
  - `Google Gemini Chat`
  - `Parse Structured Output`
  - `Format Final Decision`
- **Node Details:**
  - **Execute Decision Making Chain**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.chainLlm` (Advanced AI / LLM Chain)
    - *Configuration:* Takes contextual project metrics, applies specialized prompt rules for construction controls, and links to an LLM model and output parser.
    - *Key Expressions:* `{{ $json.context_json }}` and `{{ $json.project_id }}`.
    - *Input/Output:* Input from `Limit to 10 Entries`; connects to `Format Final Decision`.
  - **Google Gemini Chat**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` (Google Gemini Model Connector)
    - *Configuration:* Uses model `models/gemini-3.1-flash-lite` with a temperature setting of `0.2`.
    - *Key Expressions:* None.
    - *Input/Output:* Connects as an AI language model provider to `Execute Decision Making Chain`.
  - **Parse Structured Output**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.outputParserStructured` (Structured Output Parser)
    - *Configuration:* Validates responses against a strict JSON schema requiring fields such as `project_id`, `decision`, `llm_override`, `confidence`, `headline`, `why`, `critical_action`, `risk_if_ignored`, and `data_quality_flags`.
    - *Key Expressions:* None.
    - *Input/Output:* Connects as an AI output parser to `Execute Decision Making Chain`.
  - **Format Final Decision**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript Code Execution)
    - *Configuration:* Enforces safety guardrails (limiting LLM decision modifications to a maximum of one level unless explicitly overridden) and merges LLM outputs with raw calculated metrics.
    - *Key Expressions:* References `$input.all()` and `$('Calculate Risk Scores').all()`.
    - *Input/Output:* Input from `Execute Decision Making Chain`; output connects to `Log Decision to Sheets`.

#### 2.5 Log and Alert
- **Overview:** Writes the audited decision record to a tracking sheet, filters out projects categorized as "CONTINUE", and dispatches HTML-formatted email alerts to project managers for all intervention or review cases.
- **Nodes Involved:**
  - `Log Decision to Sheets`
  - `Filter Intervention Decisions`
  - `Notify PM via Email`
- **Node Details:**
  - **Log Decision to Sheets**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Google Sheets Writer)
    - *Configuration:* Appends row data to the `Decision_Log` tab in spreadsheet `1YDO5pfSMqem58WqDcFu5212rmKZQIlSVXHuL9P9gEt4` using automap mode.
    - *Key Expressions:* None.
    - *Input/Output:* Input from `Format Final Decision`; output connects to `Filter Intervention Decisions`.
  - **Filter Intervention Decisions**
    - *Type and Technical Role:* `n8n-nodes-base.filter` (Item Filter)
    - *Configuration:* Evaluates condition where `$json.decision` does not equal `CONTINUE`.
    - *Key Expressions:* `{{ $json.decision }}`.
    - *Input/Output:* Input from `Log Decision to Sheets`; output connects to `Notify PM via Email`.
  - **Notify PM via Email**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (Gmail Notification Sender)
    - *Configuration:* Sends an HTML email via Gmail OAuth2 credentials.
    - *Key Expressions:* Dynamic subject and HTML message body populated using `{{ $json.decision }}`, `{{ $json.project_name }}`, `{{ $json.headline }}`, `{{ $json.critical_action }}`, and related properties.
    - *Input/Output:* Input from `Filter Intervention Decisions`.
    - *Edge Cases/Failures:* OAuth token expiration or strict recipient filter constraints.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Sticky Note** | `n8n-nodes-base.stickyNote` | General documentation note | None | None | ## Construction - Should We Stop This Project? (Decision Engine)... |
| **Sticky Note1** | `n8n-nodes-base.stickyNote` | Block documentation note | None | None | ## Schedule project intake... |
| **Sticky Note2** | `n8n-nodes-base.stickyNote` | Block documentation note | None | None | ## Gather operating inputs... |
| **Sticky Note3** | `n8n-nodes-base.stickyNote` | Block documentation note | None | None | ## Compile risk context... |
| **Sticky Note4** | `n8n-nodes-base.stickyNote` | Block documentation note | None | None | ## Generate final decision... |
| **Sticky Note5** | `n8n-nodes-base.stickyNote` | Block documentation note | None | None | ## Log and alert... |
| **When Weekday at 7AM** | `n8n-nodes-base.scheduleTrigger` | Triggers workflow weekdays at 7 AM | None | Read Projects from Sheets | ## Schedule project intake |
| **Read Projects from Sheets** | `n8n-nodes-base.googleSheets` | Reads active projects from sheet | When Weekday at 7AM | Fetch Weather Forecast | ## Schedule project intake |
| **Fetch Weather Forecast** | `n8n-nodes-base.httpRequest` | Fetches 7-day weather forecast | Read Projects from Sheets | Read Budget from Sheets | ## Gather operating inputs |
| **Read Budget from Sheets** | `n8n-nodes-base.googleSheets` | Reads budget accounting data | Fetch Weather Forecast | Read Procurement Details | ## Gather operating inputs |
| **Read Procurement Details** | `n8n-nodes-base.googleSheets` | Reads procurement PO records | Read Budget from Sheets | Read Labor Data Sheets | ## Gather operating inputs |
| **Read Labor Data Sheets** | `n8n-nodes-base.googleSheets` | Reads crew headcount requirements | Read Procurement Details | Read Site Reports Sheets | ## Gather operating inputs |
| **Read Site Reports Sheets** | `n8n-nodes-base.googleSheets` | Reads site inspection logs | Read Labor Data Sheets | Read Disputes from Sheets | ## Compile risk context |
| **Read Disputes from Sheets** | `n8n-nodes-base.googleSheets` | Reads active dispute register | Read Site Reports Sheets | Calculate Risk Scores | ## Compile risk context |
| **Calculate Risk Scores** | `n8n-nodes-base.code` | Computes risk scores and metrics | Read Disputes from Sheets | Limit to 10 Entries | ## Compile risk context |
| **Limit to 10 Entries** | `n8n-nodes-base.limit` | Restricts batch execution throughput | Calculate Risk Scores | Execute Decision Making Chain | ## Compile risk context |
| **Execute Decision Making Chain** | `@n8n/n8n-nodes-langchain.chainLlm` | Runs LLM decision chain | Limit to 10 Entries | Format Final Decision | ## Generate final decision |
| **Google Gemini Chat** | `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` | Provides Gemini model integration | None | Execute Decision Making Chain | ## Generate final decision |
| **Parse Structured Output** | `@n8n/n8n-nodes-langchain.outputParserStructured` | Parses and validates JSON schema | None | Execute Decision Making Chain | ## Generate final decision |
| **Format Final Decision** | `n8n-nodes-base.code` | Merges LLM output and enforces guardrails | Execute Decision Making Chain | Log Decision to Sheets | ## Generate final decision |
| **Log Decision to Sheets** | `n8n-nodes-base.googleSheets` | Appends audit decision to log tab | Format Final Decision | Filter Intervention Decisions | ## Log and alert |
| **Filter Intervention Decisions** | `n8n-nodes-base.filter` | Filters out non-intervention cases | Log Decision to Sheets | Notify PM via Email | ## Log and alert |
| **Notify PM via Email** | `n8n-nodes-base.gmail` | Emails project manager on warnings | Filter Intervention Decisions | None | ## Log and alert |

---

### 4. Reproducing theWorkflow from Scratch

1. **Create the Trigger Node:**
   - Add a **Schedule Trigger** (`When Weekday at 7AM`). Set rule interval to cron expression `0 7 * * 1-5`.
2. **Setup Project Reader:**
   - Add a **Google Sheets** node (`Read Projects from Sheets`). Configure with a Service Account credential, Spreadsheet ID `1YDO5pfSMqem58WqDcFu5212rmKZQIlSVXHuL9P9gEt4`, and sheet name `Projects`. Add a filter where lookupColumn is `status` and lookupValue is `Active`. Connect `When Weekday at 7AM` to this node.
3. **Add Weather Retrieval:**
   - Add an **HTTP Request** node (`Fetch Weather Forecast`). Set method to GET, URL to `https://api.open-meteo.com/v1/forecast?latitude={{ $json.latitude }}&longitude={{ $json.longitude }}&daily=precipitation_sum,wind_speed_10m_max,temperature_2m_min&forecast_days=7&timezone=auto`, timeout to 15000ms, and configure error handling to "Continue Regular Output". Connect `Read Projects from Sheets` to this node.
4. **Configure Supporting Data Sheets:**
   - Add three sequential **Google Sheets** nodes (`Read Budget from Sheets`, `Read Procurement Details`, `Read Labor Data Sheets`) referencing the same Spreadsheet ID and respective tabs (`Budget_Accounting`, `Procurement`, `Labor`). Enable `executeOnce` on each. Chain them consecutively starting from `Fetch Weather Forecast`.
5. **Add Inspection and Dispute Readers:**
   - Add two more **Google Sheets** nodes (`Read Site Reports Sheets`, `Read Disputes from Sheets`) targeting tabs `Site_Reports` and `Disputes` respectively (with `executeOnce` enabled). Chain them sequentially after `Read Labor Data Sheets`.
6. **Implement Risk Calculation & Limiter:**
   - Add a **Code** node (`Calculate Risk Scores`) and paste the custom JavaScript code parsing project risk metrics. Connect `Read Disputes from Sheets` into it.
   - Add a **Limit** node (`Limit to 10 Entries`) and connect `Calculate Risk Scores` to it.
7. **Configure AI Decision Engine:**
   - Add an **Advanced AI / LLM Chain** node (`Execute Decision Making Chain`). Provide the prompt instructing the model to act as a construction project-controls analyst based on provided JSON metrics.
   - Add a **Google Gemini Chat** model node (`Google Gemini Chat`), set model name to `models/gemini-3.1-flash-lite`, temperature to `0.2`, and connect its AI language model output to `Execute Decision Making Chain`. Configure Google Palm API credentials.
   - Add a **Structured Output Parser** node (`Parse Structured Output`), configure its manual JSON schema matching the required decision structure (`project_id`, `decision`, `llm_override`, `confidence`, `headline`, `why`, `critical_action`, `risk_if_ignored`, `data_quality_flags`), and connect its AI output parser output to `Execute Decision Making Chain`.
   - Connect `Limit to 10 Entries` into `Execute Decision Making Chain`.
8. **Format and Log Decisions:**
   - Add a **Code** node (`Format Final Decision`) to apply guardrails and merge outputs. Connect `Execute Decision Making Chain` to it.
   - Add a **Google Sheets** node (`Log Decision to Sheets`) configured to append data to the `Decision_Log` tab using automap mode. Connect `Format Final Decision` to it.
9. **Filter and Alert:**
   - Add a **Filter** node (`Filter Intervention Decisions`) setting a condition where `$json.decision` does not equal `CONTINUE`. Connect `Log Decision to Sheets` to it.
   - Add a **Gmail** node (`Notify PM via Email`) using Gmail OAuth2 credentials. Set dynamic HTML body and subject templates referencing `$json` attributes. Connect `Filter Intervention Decisions` to this final node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Youtube Video Overview | [https://youtu.be/Fh0QR48f9is](https://youtu.be/Fh0QR48f9is) |