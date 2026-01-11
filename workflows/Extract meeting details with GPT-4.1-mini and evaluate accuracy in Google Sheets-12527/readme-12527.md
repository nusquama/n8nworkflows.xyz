Extract meeting details with GPT-4.1-mini and evaluate accuracy in Google Sheets

https://n8nworkflows.xyz/workflows/extract-meeting-details-with-gpt-4-1-mini-and-evaluate-accuracy-in-google-sheets-12527


# Extract meeting details with GPT-4.1-mini and evaluate accuracy in Google Sheets

## 1. Workflow Overview

**Title:** Extract meeting details with GPT-4.1-mini and evaluate accuracy in Google Sheets

**Purpose:**  
This workflow is a reusable **AI agent subworkflow** that extracts structured meeting/event details (title, date, time, location, link, attendees, notes) from natural-language messages. It also includes an **evaluation mode** that loads test cases from Google Sheets, runs the extractor, compares output vs. expected, scores accuracy (1–5), and writes results back to the sheet.

**Primary use cases**
- Embed as a subworkflow in larger scheduling/assistant automations (Slack/Email → extract details → create calendar event).
- Regression testing and quality tracking of extraction performance over time using Google Sheets.

### 1.1 Entry & Input Normalization (two entry modes)
- **Normal mode:** called by a parent workflow via *Execute Workflow* with `message_text` and `timezone`.
- **Evaluation mode:** triggered from an n8n Evaluation run that iterates over Google Sheets rows.

### 1.2 AI Extraction (agent + schema enforcement)
- Uses **GPT-4.1-mini** as the chat model for an **AI Agent**.
- Uses a **Structured Output Parser** with a strict JSON schema and `autoFix` to coerce valid JSON.

### 1.3 Output Validation & Error Handling
- Ensures the AI Agent actually produced `output` (because “continue on error output” can yield empty output while still following a “success” path).
- On failure: raises a structured error that bubbles up to the parent workflow.

### 1.4 Mode Routing (normal return vs evaluation path)
- A manual IF router uses `is_evaluating` to ensure that in **normal mode** the workflow ends with data returned to the caller, while **evaluation mode** continues to scoring + recording.

### 1.5 Evaluation (LLM-based semantic scoring + Google Sheets writeback)
- Compares actual extracted JSON vs expected JSON using an evaluation prompt (semantic + exact-match rules).
- Writes `actual_output`, `metadata` (reasoning), and `match` score back to the Google Sheet.

---

## 2. Block-by-Block Analysis

### Block A — Normal Mode Entry & Normalization
**Overview:** Receives `message_text` and `timezone` from a parent workflow trigger and normalizes inputs (including a runtime `now` timestamp) into a consistent shape.

**Nodes involved:** `trigger`, `normalize_trigger_input`, `merged_inputs`

#### Node: `trigger`
- **Type / role:** `Execute Workflow Trigger` — entry point when called as a subworkflow.
- **Key configuration:**
  - Declares required workflow inputs: `message_text`, `timezone`.
  - Pinned sample data exists (for manual testing).
- **Connections:** → `normalize_trigger_input`
- **Edge cases / failures:**
  - If parent workflow doesn’t pass `message_text` or `timezone`, downstream expressions may become empty/undefined.

#### Node: `normalize_trigger_input`
- **Type / role:** `Set` — standardizes inbound fields.
- **Configuration choices:**
  - Sets:
    - `message_text` = value from `trigger`
    - `timezone` = value from `trigger`
    - `now` = `new Date().toISOString()` at execution time
- **Key expressions:**
  - `{{ $('trigger').first().json.message_text }}`
  - `{{ new Date().toISOString() }}`
  - `{{ $('trigger').first().json.timezone }}`
- **Connections:** → `merged_inputs`
- **Edge cases:**
  - If `timezone` is not an IANA timezone, the AI may resolve relative dates incorrectly (no strict validation here).

#### Node: `merged_inputs`
- **Type / role:** `No Operation (NoOp)` — acts as a join point so both normal-mode and evaluation-mode normalized inputs share the same downstream graph.
- **Connections:** → `extract_meeting_details`
- **Edge cases:** None (pass-through).

---

### Block B — Evaluation Mode Entry & Normalization
**Overview:** Loads test rows from a Google Sheet via n8n’s Evaluation Trigger, then reshapes them to match the same fields used in normal mode, including a fixed reference time for reproducible results.

**Nodes involved:** `load_eval_data`, `normalize_eval_data`, `merged_inputs`

#### Node: `load_eval_data`
- **Type / role:** `Evaluation Trigger` — entry point for batch test execution from a Google Sheet dataset.
- **Configuration choices:**
  - Limits rows (enabled).
  - Targets a Google Sheet named `meeting_extractor` (gid=0) with `documentId = 1U89nPsasM2WNv1D7gEYINhDwylyxYw7BOd_i8ipFC0M`.
- **Credentials:** Google Sheets OAuth2 (`Google Sheets account`)
- **Connections:** → `normalize_eval_data`
- **Edge cases / failures:**
  - OAuth issues (expired token / missing scopes).
  - Sheet structure mismatches: must contain expected columns used by the evaluation framework (notably `input` and `expected_output` as referenced later).
  - Document ID mismatch (common when copying the template sheet).

#### Node: `normalize_eval_data`
- **Type / role:** `Set` — normalizes evaluation-row fields into the same schema as normal mode.
- **Configuration choices:**
  - `message_text` = `load_eval_data.item.json.input`
  - `is_evaluating` = `true` (explicit flag for routing)
  - `=now` set to a fixed ISO time derived from `2026-01-06T15:30:00Z`
  - `timezone` hardcoded to `America/Argentina/Buenos_Aires`
- **Key expressions:**
  - `{{ $('load_eval_data').item.json.input }}`
  - `{{ new Date('2026-01-06T15:30:00Z').toISOString() }}`
- **Important note:** The field name is literally **`=now`** (including the equals sign), not `now`. This is unusual and can break downstream logic if downstream expects `now`.
- **Connections:** → `merged_inputs`
- **Edge cases / failures:**
  - If downstream expects `now` (without `=`), the AI agent will receive `undefined` for reference time, harming relative date resolution. (In this workflow, `extract_meeting_details` uses `merged_inputs...json.now`, so evaluation mode likely won’t supply it correctly unless n8n strips the `=` in the UI—verify in your instance.)
  - Hardcoded timezone/date must match the sheet’s expected outputs; otherwise tests will fail by design.

---

### Block C — AI Extraction with Structured Schema Enforcement
**Overview:** Runs an AI Agent using GPT-4.1-mini to extract meeting details and forces the output into a strict JSON schema via the Structured Output Parser.

**Nodes involved:** `OpenAI Chat Model`, `Structured Output Parser`, `output_fixer`, `extract_meeting_details`

#### Node: `OpenAI Chat Model`
- **Type / role:** `LangChain Chat Model (OpenAI)` — provides the LLM for the AI Agent.
- **Configuration choices:**
  - Model: `gpt-4.1-mini`
  - Temperature: `0` (deterministic bias)
  - Max retries: `3`
  - Response format: `json_object` (strongly nudges JSON)
- **Credentials:** OpenAI API (`scheduler_bot_api_key`)
- **Connections:** (AI language model) → `extract_meeting_details`
- **Edge cases / failures:**
  - OpenAI auth/quota errors.
  - Model unavailability / name mismatch.
  - Output may still violate schema; parser handles with auto-fix when possible.

#### Node: `Structured Output Parser`
- **Type / role:** `Structured Output Parser (LangChain)` — validates/coerces the agent output to a JSON schema.
- **Configuration choices:**
  - Schema is manually defined JSON Schema (draft 2020-12).
  - `autoFix: true` (attempts to repair malformed output).
  - Requires:
    - `meeting` object with required keys `title`, `date`, `time` (these can be `null` but are required to exist).
    - `reasoning` string (1–200 chars).
  - Enforces date pattern `YYYY-MM-DD` and time pattern `HH:mm`.
  - `meeting_link` must be a valid URI when not null.
- **Connections:** (AI output parser) → `extract_meeting_details`
- **Edge cases / failures:**
  - If the model outputs invalid URI strings for `meeting_link`, schema validation can fail.
  - `reasoning` > 200 chars will fail validation unless auto-fix trims (not guaranteed).

#### Node: `output_fixer`
- **Type / role:** `LangChain Chat Model (OpenAI)` — intended as a fixing model for parser auto-fix flows.
- **Configuration choices:**
  - Model: `gpt-5-mini`
- **Credentials:** OpenAI API (`scheduler_bot_api_key`)
- **Connections:** (AI language model) → `Structured Output Parser`
- **Important:** In this JSON, `output_fixer` is connected to the parser, but the parser’s settings don’t explicitly reference it beyond normal LangChain wiring. Whether it is actually used depends on node internals/version.
- **Edge cases / failures:** Same OpenAI issues as above.

#### Node: `extract_meeting_details`
- **Type / role:** `AI Agent (LangChain)` — core extraction logic.
- **Configuration choices:**
  - Prompt includes:
    - `message_text` from `merged_inputs`
    - `reference_datetime_utc` from `merged_inputs.json.now`
    - `timezone` from `merged_inputs`
  - System message defines strict JSON output with nulls for missing fields and explicit relative-time reasoning rules.
  - **hasOutputParser: true** — uses `Structured Output Parser`.
  - `onError: continueErrorOutput` + `retryOnFail: true`
- **Key expressions:**
  - Uses `{{ $('merged_inputs').first().json.message_text }}`
  - Uses `{{ $('merged_inputs').first().json.now }}`
  - Uses `{{ $('merged_inputs').first().json.timezone }}`
- **Connections:**
  - Main output → `validate_output`
  - Error output (because continueErrorOutput) → `handle_error` (second branch in connections)
  - AI model input from `OpenAI Chat Model`
  - Output parsing from `Structured Output Parser`
- **Edge cases / failures:**
  - If `now` is missing (likely in evaluation mode due to `=now`), relative date resolution can drift.
  - Agent may return empty `output` despite “success”; handled by validation block.
  - “Always future date” rule: messages referring to past dates may produce unexpected “corrected” future values.

---

### Block D — Output Validation & Structured Error Bubble-Up
**Overview:** Ensures that the AI Agent returned a non-empty `output`. If missing, execution is stopped with a structured error object (useful when this is called as a subworkflow).

**Nodes involved:** `validate_output`, `handle_error`, `normalize_agent_output`

#### Node: `validate_output`
- **Type / role:** `IF` — checks presence of AI output.
- **Configuration choices:**
  - Condition: `extract_meeting_details.first().json.output` is **not empty**.
- **Connections:**
  - TRUE → `normalize_agent_output`
  - FALSE → `handle_error`
- **Edge cases / failures:**
  - If AI Agent changes its internal output structure (e.g., `output` renamed), condition fails and routes to error.

#### Node: `handle_error`
- **Type / role:** `Stop and Error` — throws a custom structured error.
- **Configuration choices:**
  - Throws an “errorObject” containing:
    - message: `AI agent failed to normalize event strings`
    - execution and workflow metadata (`$execution`, `$workflow`)
    - timestamp
- **Connections:** terminal
- **Edge cases / failures:**
  - Parent workflows must handle thrown errors; otherwise the whole parent run fails (often desired in production).

#### Node: `normalize_agent_output`
- **Type / role:** `Set` — produces clean top-level fields for downstream evaluation/routing.
- **Configuration choices:**
  - `meeting` = `extract_meeting_details.first().json.output.meeting`
  - `reasoning` = `extract_meeting_details.first().json.output.reasoning`
- **Connections:** → `not_evaluating`
- **Edge cases:**
  - If output parser coerces fields unexpectedly (e.g., link nulling), evaluation scoring may drop.

---

### Block E — Mode Router (Normal Return vs Evaluation Continuation)
**Overview:** Routes execution based on `is_evaluating`. Normal mode should end after producing `meeting` and `reasoning`; evaluation mode continues to scoring nodes.

**Nodes involved:** `not_evaluating`

#### Node: `not_evaluating`
- **Type / role:** `IF` — manual router.
- **Configuration choices:**
  - Primary check: `!!$('merged_inputs').item.json.is_evaluating` is **false**.
  - There is a second empty string equals empty string condition, which is always true; effectively it doesn’t change behavior but is confusing.
- **Connections:**
  - **TRUE branch (not evaluating):** no connected nodes (workflow ends; in subworkflow context, output is returned).
  - **FALSE branch (evaluating):** → `evaluate_match`
- **Edge cases / failures:**
  - If `is_evaluating` is missing, `!!undefined` is `false`, so condition “boolean false” becomes true → treated as **not evaluating** (desired default).
  - If you accidentally set `is_evaluating` truthy in normal calls, evaluation nodes will run and may fail due to missing expected sheet fields.

---

### Block F — Evaluation: Semantic Scoring + Writeback to Google Sheets
**Overview:** Uses an LLM-based evaluator to compare actual vs expected outputs and scores them 1–5, then records outputs and score into Google Sheets.

**Nodes involved:** `OpenAI Chat Model1`, `evaluate_match`, `record_eval_output`

#### Node: `OpenAI Chat Model1`
- **Type / role:** `LangChain Chat Model (OpenAI)` — evaluation model provider.
- **Configuration choices:**
  - Model: `gpt-5-mini`
  - `retryOnFail: true`
- **Credentials:** OpenAI API (`scheduler_bot_api_key`)
- **Connections:** (AI language model) → `evaluate_match`
- **Edge cases / failures:** OpenAI auth/quota/model availability.

#### Node: `evaluate_match`
- **Type / role:** `Evaluation` — sets an evaluation metric using an LLM prompt.
- **Configuration choices:**
  - Operation: `setMetrics`
  - Metric name: `match`
  - Prompt: detailed rubric:
    - Exact: `date`, `time`, `meeting_link`
    - Semantic: `title`, `location`, `attendees`, `notes`
    - Null handling guidance
    - Outputs only: 1–5
  - `actualAnswer` = JSON stringify of `normalize_agent_output.meeting`
  - `expectedAnswer` = JSON stringify of `load_eval_data.item.json.expected_output`
- **Connections:** → `record_eval_output`
- **Edge cases / failures:**
  - If expected outputs are not valid JSON objects (or stored as strings inconsistently), the evaluator may behave unpredictably.
  - LLM may output text other than a single digit; depending on node internals this may fail metric parsing.

#### Node: `record_eval_output`
- **Type / role:** `Evaluation` — writes results back to Google Sheets.
- **Configuration choices:**
  - Source: `googleSheets`
  - Writes:
    - `actual_output` = JSON stringify of extracted meeting
    - `metadata` = JSON stringify of reasoning
    - `match` = metric produced by `evaluate_match`
  - Document ID is `1U89nPsasM2WNv1D7gEYINhDwylyxYw7BOd_i8ipFC0M`
  - Note: the cachedResultUrl shown in the node references a different spreadsheet ID (`12Zpn...`) but the *actual* `documentId` parameter is what matters at runtime.
- **Credentials:** Google Sheets OAuth2 (`Google Sheets account`)
- **Connections:** terminal
- **Edge cases / failures:**
  - Missing write permissions to the sheet.
  - Column mismatches: the Evaluation framework expects specific columns for outputs.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| trigger | Execute Workflow Trigger | Subworkflow entry point (normal mode) | — | normalize_trigger_input | # AI Meeting Extractor with Evaluation Framework (setup, usage, customization, sheet link) |
| normalize_trigger_input | Set | Normalize parent-provided inputs; set runtime `now` | trigger | merged_inputs | # AI Meeting Extractor with Evaluation Framework (setup, usage, customization, sheet link) |
| load_eval_data | Evaluation Trigger | Load evaluation rows from Google Sheets | — | normalize_eval_data | ## Load Evaluation Data (sheet ID changes; fixed now/timezone caution) |
| normalize_eval_data | Set | Normalize evaluation row to match trigger format; set `is_evaluating` | load_eval_data | merged_inputs | ## Load Evaluation Data (sheet ID changes; fixed now/timezone caution) |
| merged_inputs | NoOp | Join point for normal + evaluation inputs | normalize_trigger_input, normalize_eval_data | extract_meeting_details | # AI Meeting Extractor with Evaluation Framework (setup, usage, customization, sheet link) |
| OpenAI Chat Model | LangChain Chat Model (OpenAI) | LLM for extraction agent (GPT-4.1-mini) | — | extract_meeting_details (ai_languageModel) | ## AI Agent - Core Extraction Logic (agent role; schema; reuse pattern) |
| Structured Output Parser | Structured Output Parser | Enforce JSON schema; auto-fix invalid output | output_fixer (ai_languageModel) | extract_meeting_details (ai_outputParser) | ## AI Agent - Core Extraction Logic (agent role; schema; reuse pattern) |
| output_fixer | LangChain Chat Model (OpenAI) | Potential fixing model for schema repair (GPT-5-mini) | — | Structured Output Parser (ai_languageModel) | ## AI Agent - Core Extraction Logic (agent role; schema; reuse pattern) |
| extract_meeting_details | AI Agent (LangChain) | Core extraction into structured meeting JSON + reasoning | merged_inputs, OpenAI Chat Model, Structured Output Parser | validate_output, handle_error | ## AI Agent - Core Extraction Logic (agent role; schema; reuse pattern) |
| validate_output | IF | Detect empty agent output; route to error | extract_meeting_details | normalize_agent_output, handle_error | ## Output Validation & Error Handling (why needed; continue-on-error caveat) |
| handle_error | Stop and Error | Throw structured error object to parent | extract_meeting_details, validate_output | — | ## Output Validation & Error Handling (traceability via execution/workflow metadata) |
| normalize_agent_output | Set | Flatten agent output to `meeting` and `reasoning` | validate_output | not_evaluating | ## Output Validation & Error Handling (fits validated output into stable fields) |
| not_evaluating | IF | Route normal mode to end; evaluation mode to scoring | normalize_agent_output | evaluate_match (false branch only) | ## Evaluation Mode Router (manual is_evaluating routing rationale) |
| OpenAI Chat Model1 | LangChain Chat Model (OpenAI) | LLM for evaluation scoring (GPT-5-mini) | — | evaluate_match (ai_languageModel) | ## Evaluation: Compare Actual vs Expected (semantic/exact rules; best practice note) |
| evaluate_match | Evaluation | LLM-based match scoring metric (1–5) | not_evaluating, OpenAI Chat Model1 | record_eval_output | ## Evaluation: Compare Actual vs Expected (semantic/exact rules; best practice note) |
| record_eval_output | Evaluation | Write evaluation outputs + score back to Google Sheets | evaluate_match | — | ## Evaluation: Compare Actual vs Expected (writeback; change spreadsheet ID) |
| Sticky Note | Sticky Note | Documentation | — | — | (self) |
| Sticky Note1 | Sticky Note | Documentation | — | — | (self) |
| Sticky Note2 | Sticky Note | Documentation | — | — | (self) |
| Sticky Note3 | Sticky Note | Documentation | — | — | (self) |
| Sticky Note4 | Sticky Note | Documentation | — | — | (self) |
| Sticky Note5 | Sticky Note | Documentation | — | — | (self) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**  
   - Name it: *Extract meeting details with GPT-4.1-mini and evaluate accuracy in Google Sheets*.

2) **Add normal-mode entry node**
   - Add node: **Execute Workflow Trigger**
   - Define **Workflow Inputs**:
     - `message_text` (string)
     - `timezone` (string)

3) **Normalize normal-mode inputs**
   - Add node: **Set** named `normalize_trigger_input`
   - Add fields:
     - `message_text` = expression from trigger input
     - `timezone` = expression from trigger input
     - `now` = `new Date().toISOString()`
   - Connect: `trigger` → `normalize_trigger_input`

4) **Add evaluation entry node (for batch tests)**
   - Add node: **Evaluation Trigger** named `load_eval_data`
   - Configure Google Sheets source:
     - Set `documentId` to your test sheet ID
     - Select sheet tab (gid=0 in the template)
     - Enable “limit rows” if desired
   - Set up **Google Sheets OAuth2** credentials.

5) **Normalize evaluation row to match normal inputs**
   - Add node: **Set** named `normalize_eval_data`
   - Set:
     - `message_text` = `load_eval_data.item.json.input`
     - `is_evaluating` = `true`
     - `now` = fixed ISO time (recommended for deterministic tests), e.g. `2026-01-06T15:30:00Z`
     - `timezone` = fixed IANA timezone used by the dataset
   - Connect: `load_eval_data` → `normalize_eval_data`

6) **Add a join point**
   - Add node: **NoOp** named `merged_inputs`
   - Connect:
     - `normalize_trigger_input` → `merged_inputs`
     - `normalize_eval_data` → `merged_inputs`

7) **Add the extraction chat model**
   - Add node: **OpenAI Chat Model (LangChain)** named `OpenAI Chat Model`
   - Choose model: `gpt-4.1-mini`
   - Set options:
     - temperature = 0
     - max retries = 3
     - response format = `json_object`
   - Configure **OpenAI API credential**.

8) **Add Structured Output Parser**
   - Add node: **Structured Output Parser**
   - Enable `autoFix`
   - Set schema to include:
     - `meeting.title` (string|null), `meeting.date` (YYYY-MM-DD string|null), `meeting.time` (HH:mm string|null), plus optional fields (location/link/attendees/notes)
     - top-level `reasoning` (short string)
   - (Optional but matching this workflow) Add a second OpenAI model as fixer.

9) **(Optional) Add fixer model used by parser**
   - Add node: **OpenAI Chat Model (LangChain)** named `output_fixer`
   - Model: `gpt-5-mini`
   - Connect: `output_fixer` → `Structured Output Parser` via the parser’s **AI Language Model** connection.

10) **Add the AI Agent**
   - Add node: **AI Agent (LangChain)** named `extract_meeting_details`
   - Configure:
     - Provide a **System Message** that:
       - forces strict JSON output
       - resolves relative times using `reference_datetime_iso` and `timezone`
       - uses nulls for missing fields
     - Provide a **Text/Prompt** that injects:
       - `message_text` from `merged_inputs`
       - `now` from `merged_inputs`
       - `timezone` from `merged_inputs`
     - Enable `hasOutputParser` and connect the **Structured Output Parser** to it.
     - Enable “Continue on Error Output” (to match this design), and enable retry on fail.
   - Connect:
     - `merged_inputs` → `extract_meeting_details` (main)
     - `OpenAI Chat Model` → `extract_meeting_details` (AI Language Model)
     - `Structured Output Parser` → `extract_meeting_details` (AI Output Parser)

11) **Validate the agent output exists**
   - Add node: **IF** named `validate_output`
   - Condition: `extract_meeting_details.first().json.output` is **not empty**
   - Connect: `extract_meeting_details` → `validate_output`

12) **Add structured error node**
   - Add node: **Stop and Error** named `handle_error`
   - Set it to throw an **error object** including:
     - clear message
     - `$execution` and `$workflow`
     - timestamp
   - Connect:
     - `validate_output` FALSE → `handle_error`
     - (Optional but recommended) connect `extract_meeting_details` error output → `handle_error`

13) **Normalize final output shape**
   - Add node: **Set** named `normalize_agent_output`
   - Set:
     - `meeting` = `extract_meeting_details.first().json.output.meeting`
     - `reasoning` = `extract_meeting_details.first().json.output.reasoning`
   - Connect: `validate_output` TRUE → `normalize_agent_output`

14) **Route evaluation vs normal mode**
   - Add node: **IF** named `not_evaluating`
   - Condition: `is_evaluating` is falsey (e.g., `!!merged_inputs.item.json.is_evaluating` equals `false`)
   - Connect: `normalize_agent_output` → `not_evaluating`
   - Leave **TRUE** branch unconnected (workflow ends and returns output in normal subworkflow usage)
   - Connect **FALSE** branch → evaluation nodes (next steps)

15) **Add evaluation scoring model**
   - Add node: **OpenAI Chat Model (LangChain)** named `OpenAI Chat Model1`
   - Model: `gpt-5-mini`
   - Configure OpenAI credential.

16) **Add evaluation scoring node**
   - Add node: **Evaluation** named `evaluate_match`
   - Operation: **setMetrics**
   - Metric name: `match`
   - Prompt: include your scoring rubric (exact vs semantic fields; null rules; output only 1–5)
   - actualAnswer = JSON of extracted `meeting`
   - expectedAnswer = JSON from `load_eval_data.item.json.expected_output`
   - Connect:
     - `not_evaluating` FALSE → `evaluate_match`
     - `OpenAI Chat Model1` → `evaluate_match` (AI Language Model)

17) **Record evaluation output to Google Sheets**
   - Add node: **Evaluation** named `record_eval_output`
   - Source: `googleSheets`
   - Map outputs:
     - `actual_output` = JSON stringify extracted meeting
     - `metadata` = reasoning
     - `match` = metric value from `evaluate_match`
   - Configure Google Sheets credential and set the same document ID / sheet tab.
   - Connect: `evaluate_match` → `record_eval_output`

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Make a copy of the evaluation dataset sheet and update the workflow’s Google Sheets `documentId` in `load_eval_data` and `record_eval_output`. | https://docs.google.com/spreadsheets/d/1U89nPsasM2WNv1D7gEYINhDwylyxYw7BOd_i8ipFC0M/edit?usp=sharing |
| Evaluation mode relies on a fixed reference time and timezone; changing them requires updating expected outputs or tests will fail. | See “Load Evaluation Data” sticky note guidance |
| Manual routing via an `is_evaluating` flag is used to ensure subworkflow outputs return correctly in normal mode (instead of relying on n8n’s built-in evaluation routing behavior). | See “Evaluation Mode Router” sticky note guidance |
| Best practice: prefer exact/categorical metrics when possible; use LLM-based evaluation only for semantic/fuzzy matching needs. | See “Evaluation: Compare Actual vs Expected” sticky note guidance |

Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.