Benchmark procurement KPIs with Google Sheets, Gmail, and Google Gemini

https://n8nworkflows.xyz/workflows/benchmark-procurement-kpis-with-google-sheets--gmail--and-google-gemini-20454


# Benchmark procurement KPIs with Google Sheets, Gmail, and Google Gemini

### 1. Workflow Overview

The "Procurement Metrics Benchmarking Workflow" automates the monthly performance assessment of procurement KPIs against industry and peer benchmarks. It retrieves records flagged as ready from Google Sheets, cross-references them against active KPI definitions and peer benchmarks, computes gap percentages and severity levels, leverages Google Gemini for management-ready analysis, logs finalized results back to Google Sheets, and dispatches automated email reports and leadership alerts via Gmail.

The workflow logic is categorized into the following functional blocks:
- **1.1 Trigger & Configuration:** Initiates the workflow on a monthly schedule and defines persistent configuration parameters (statuses, thresholds, and placeholders).
- **1.2 Data Retrieval & Validation (Preprocessing):** Queries procurement records, validates the existence of active KPI definitions and external benchmarks, and ensures numeric integrity before processing.
- **1.3 Gap Calculation & AI Analysis:** Computes absolute and percentage gaps, performance status, and severity, then uses Google Gemini to generate structured executive summaries, causes, and recommendations.
- **1.4 Results Persistence, Alerting & Reporting:** Appends finalized results to Google Sheets, flags source records, routes significant gaps for leadership email alerts, and compiles a consolidated monthly HTML summary report.
- **1.5 Exception Handling & Review Routing:** Catches missing definitions, missing benchmarks, or invalid numeric data, routing problematic records to a review status in Google Sheets.

---

### 2. Block-by-Block Analysis

---

### Block 1.1: Trigger & Configuration

#### Overview
This block schedules the automated execution of the workflow on a monthly basis and instantiates a central reference object containing all required configuration constants, environment thresholds, and email routing variables.

#### Nodes Involved
- Monthly KPI Benchmark Trigger
- Load Benchmark Settings

#### Node Details

- **Monthly KPI Benchmark Trigger**
  - *Type & Role:* `n8n-nodes-base.scheduleTrigger` (Trigger Node). Executes the workflow automatically once a month at 9:00 AM.
  - *Configuration:* Rule interval configured to monthly execution.
  - *Expressions / Variables:* None.
  - *Connections:* Input: None; Output: `Load Benchmark Settings`.
  - *Version Requirements:* v1.3.
  - *Edge Cases / Failures:* Manual testing requires manual execution since schedule triggers do not fire on demand unless manually triggered via the n8n editor.

- **Load Benchmark Settings**
  - *Type & Role:* `n8n-nodes-base.set` (Data Transformation). Establishes static global parameters and default strings used throughout the workflow.
  - *Configuration:* Assigns string and numeric constants (e.g., `ready_status` = "Ready", `processed_status` = "Processed", `near_benchmark_threshold_pct` = 5, `max_recommendations` = 3, message strings, and blank email placeholders).
  - *Expressions / Variables:* Static values.
  - *Connections:* Input: `Monthly KPI Benchmark Trigger`; Output: `Get Ready Procurement KPIs`.
  - *Version Requirements:* v3.4.
  - *Edge Cases / Failures:* Missing configuration properties will cause downstream expressions attempting to reference `$('Load Benchmark Settings').first().json.*` to evaluate as undefined.

---

### Block 1.2: Data Retrieval & Validation (Preprocessing)

#### Overview
This block queries Google Sheets for procurement records marked as ready, performs lookups for matching KPI definitions and external peer benchmarks, and validates that all numerical inputs are safe for mathematical operations.

#### Nodes Involved
- Get Ready Procurement KPIs
- Get KPI Definition
- KPI Definition Found?
- Handle Missing KPI Definition
- Get External Benchmark
- External Benchmark Found?
- Handle Missing External Benchmark
- Build Benchmark Comparison Context
- Validate Benchmark Inputs
- Benchmark Inputs Valid?
- Handle Invalid Benchmark Data
- Mark KPI as Review Required

#### Node Details

- **Get Ready Procurement KPIs**
  - *Type & Role:* `n8n-nodes-base.googleSheets` (Data Input). Retrieves procurement records from the `Procurement_KPIs` worksheet filtered by status.
  - *Configuration:* Operation: `list` (lookup). Filter condition: `status` equals `{{ $('Load Benchmark Settings').item.json.ready_status }}`.
  - *Expressions / Variables:* `={{ $('Load Benchmark Settings').item.json.ready_status }}`
  - *Connections:* Input: `Load Benchmark Settings`; Output: `Get KPI Definition`.
  - *Version Requirements:* v4.7.
  - *Edge Cases / Failures:* Google Sheets API rate limits or invalid sheet IDs will cause execution halts.

- **Get KPI Definition**
  - *Type & Role:* `n8n-nodes-base.googleSheets` (Data Input). Fetches KPI metadata from the `KPI_Definitions` sheet.
  - *Configuration:* Lookup columns: `kpi_code` matched against the incoming record and `active` matched against the configuration constant.
  - *Expressions / Variables:* `={{ $json.kpi_code }}`, `={{ $('Load Benchmark Settings').first().json.active_kpi_definition_value }}`
  - *Connections:* Input: `Get Ready Procurement KPIs`; Output: `KPI Definition Found?`.
  - *Version Requirements:* v4.7.

- **KPI Definition Found?**
  - *Type & Role:* `n8n-nodes-base.if` (Flow Control). Verifies whether a valid KPI definition record was returned.
  - *Configuration:* Condition checks if `Boolean($json.kpi_code)` is true.
  - *Expressions / Variables:* `={{ Boolean($json.kpi_code) }}`
  - *Connections:* Input: `Get KPI Definition`; Outputs: True path to `Get External Benchmark`, False path to `Handle Missing KPI Definition`.
  - *Version Requirements:* v2.3.

- **Handle Missing KPI Definition**
  - *Type & Role:* `n8n-nodes-base.set` (Data Transformation). Prepares error metadata when a KPI definition is missing.
  - *Configuration:* Sets `processing_status` to "Review Required", sets `should_continue` to false, and captures identification fields.
  - *Expressions / Variables:* References settings for review status and error messages.
  - *Connections:* Input: `KPI Definition Found?` (False); Output: `Mark KPI as Review Required`.
  - *Version Requirements:* v3.4.

- **Get External Benchmark**
  - *Type & Role:* `n8n-nodes-base.googleSheets` (Data Input). Retrieves external benchmark values from the `External_Benchmarks` sheet.
  - *Configuration:* Lookup columns: `kpi_code` matched against the KPI code and `active` matched against the active benchmark flag.
  - *Expressions / Variables:* `={{ $json.kpi_code }}`, `={{ $('Load Benchmark Settings').first().json.active_benchmark_value }}`
  - *Connections:* Input: `KPI Definition Found?` (True); Output: `External Benchmark Found?`.
  - *Version Requirements:* v4.7.

- **External Benchmark Found?**
  - *Type & Role:* `n8n-nodes-base.if` (Flow Control). Validates whether an active external benchmark record exists.
  - *Configuration:* Condition checks if `Boolean($json.benchmark_id)` is true.
  - *Expressions / Variables:* `={{ Boolean($json.benchmark_id) }}`
  - *Connections:* Input: `Get External Benchmark`; Outputs: True path to `Build Benchmark Comparison Context`, False path to `Handle Missing External Benchmark`.
  - *Version Requirements:* v2.3.

- **Handle Missing External Benchmark**
  - *Type & Role:* `n8n-nodes-base.set` (Data Transformation). Formats error details when an external benchmark cannot be located.
  - *Configuration:* Sets error flags and review statuses.
  - *Expressions / Variables:* Pulls missing benchmark messages from `Load Benchmark Settings`.
  - *Connections:* Input: `External Benchmark Found?` (False); Output: `Mark KPI as Review Required`.
  - *Version Requirements:* v3.4.

- **Build Benchmark Comparison Context**
  - *Type & Role:* `n8n-nodes-base.code` (Data Transformation). Combines data from the KPI record, KPI definition, and external benchmark into a single unified JSON payload.
  - *Configuration:* Custom JavaScript mapping numeric values via `Number()` conversion.
  - *Expressions / Variables:* References upstream node outputs using `$('Get Ready Procurement KPIs')`, `$('Get KPI Definition')`, and `$json`.
  - *Connections:* Input: `External Benchmark Found?` (True); Output: `Validate Benchmark Inputs`.
  - *Version Requirements:* v2.

- **Validate Benchmark Inputs**
  - *Type & Role:* `n8n-nodes-base.code` (Data Transformation). Validates that actual values, benchmarks, and warning/critical thresholds are present, numeric, non-zero for benchmarks, and logically ordered (warning $\le$ critical).
  - *Configuration:* Custom JavaScript validation helper pushing errors into an array.
  - *Expressions / Variables:* Processes input object fields.
  - *Connections:* Input: `Build Benchmark Comparison Context`; Output: `Benchmark Inputs Valid?`.
  - *Version Requirements:* v2.

- **Benchmark Inputs Valid?**
  - *Type & Role:* `n8n-nodes-base.if` (Flow Control). Branches execution based on whether validation passed.
  - *Configuration:* Evaluates `{{ $json.benchmark_inputs_valid }}`.
  - *Expressions / Variables:* `={{ $json.benchmark_inputs_valid }}`
  - *Connections:* Input: `Validate Benchmark Inputs`; Outputs: True path to `Calculate Benchmark Gap`, False path to `Handle Invalid Benchmark Data`.
  - *Version Requirements:* v2.3.

- **Handle Invalid Benchmark Data**
  - *Type & Role:* `n8n-nodes-base.set` (Data Transformation). Consolidates validation error messages for items failing numeric or threshold checks.
  - *Configuration:* Assigns review status and formats joined validation errors.
  - *Expressions / Variables:* `={{ $('Validate Benchmark Inputs').item.json.validation_errors.join(' | ') }}`
  - *Connections:* Input: `Benchmark Inputs Valid?` (False); Output: `Mark KPI as Review Required`.
  - *Version Requirements:* v3.4.

- **Mark KPI as Review Required**
  - *Type & Role:* `n8n-nodes-base.googleSheets` (Data Output). Updates the source KPI row status in Google Sheets to "Review Required" when validation or data retrieval fails.
  - *Configuration:* Operation: `update`. Matching column: `kpi_record_id`. Updates `status` field.
  - *Expressions / Variables:* `={{ $('Load Benchmark Settings').first().json.review_required_status }}`, `={{ $json.kpi_record_id }}`
  - *Connections:* Input: `Handle Missing KPI Definition`, `Handle Missing External Benchmark`, `Handle Invalid Benchmark Data`; Output: None (Terminal branch for invalid rows).
  - *Version Requirements:* v4.7.

---

### Block 1.3: Gap Calculation & AI Analysis

#### Overview
This block computes the absolute gap, percentage difference, performance status ("Above", "Near", or "Below" benchmark), and severity level ("Low", "Medium", "High", "Critical"). It then queries Google Gemini with structured rules to generate management commentary and actionable recommendations.

#### Nodes Involved
- Calculate Benchmark Gap
- Generate KPI Gap Analysis
- Gemini Model for KPI Analysis
- Parse Structured KPI Analysis

#### Node Details

- **Calculate Benchmark Gap**
  - *Type & Role:* `n8n-nodes-base.code` (Data Transformation). Computes numerical gaps, directional performance, performance status, and severity classifications.
  - *Configuration:* Custom JavaScript utilizing direction rules ("Higher Is Better" vs "Lower Is Better") and threshold percentages.
  - *Expressions / Variables:* References `near_benchmark_threshold_pct` from `Load Benchmark Settings`.
  - *Connections:* Input: `Benchmark Inputs Valid?` (True); Output: `Generate KPI Gap Analysis`.
  - *Version Requirements:* v2.

- **Generate KPI Gap Analysis**
  - *Type & Role:* `@n8n/n8n-nodes-langchain.chainLlm` (AI / LangChain Node). Sends KPI context and calculated metrics to Google Gemini to generate structured gap analysis.
  - *Configuration:* System prompt configured with procurement analyst personas and strict adherence rules (prohibiting the overriding of calculated metrics). Connected to a Gemini chat model and a structured output parser.
  - *Expressions / Variables:* Interpolates KPI codes, names, actual values, benchmark values, gap percentages, and severity levels.
  - *Connections:* Input: `Calculate Benchmark Gap`; Output: `Prepare Benchmark Result Row`. Linked sub-nodes: `Gemini Model for KPI Analysis` and `Parse Structured KPI Analysis`.
  - *Version Requirements:* v1.9.
  - *Edge Cases / Failures:* API authentication failures or rate limits on Google AI Studio will trigger workflow execution errors.

- **Gemini Model for KPI Analysis**
  - *Type & Role:* `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` (AI Model Sub-Node). Provides the underlying Google Gemini chat model configuration.
  - *Configuration:* Uses Google Gemini credentials (`googlePalmApi`).
  - *Credentials:* Google Gemini (AI Studio).
  - *Connections:* Connected to `Generate KPI Gap Analysis` via `ai_languageModel`.
  - *Version Requirements:* v1.1.

- **Parse Structured KPI Analysis**
  - *Type & Role:* `@n8n/n8n-nodes-outputParserStructured` (AI Output Parser Sub-Node). Enforces a rigid JSON output schema for Gemini's response.
  - *Configuration:* Manual schema enforcing string fields for `gap_summary` and `management_comment`, and array fields with defined min/max items for `likely_causes` and `recommended_actions`.
  - *Connections:* Connected to `Generate KPI Gap Analysis` via `ai_outputParser`.
  - *Version Requirements:* v1.3.

---

### Block 1.4: Results Persistence, Alerting & Reporting

#### Overview
This block records the finalized benchmark results to Google Sheets, marks source KPI records as processed, evaluates whether a leadership alert email should be sent for significant gaps, and compiles and emails a consolidated monthly HTML report.

#### Nodes Involved
- Prepare Benchmark Result Row
- Append Benchmark Result
- Mark KPI as Processed
- Is Significant Gap?
- Prepare Leadership Alert
- Send Leadership Alert Email
- No Leadership Alert Required
- Build Monthly Benchmark Report
- Send Monthly Benchmark Report

#### Node Details

- **Prepare Benchmark Result Row**
  - *Type & Role:* `n8n-nodes-base.code` (Data Transformation). Merges raw KPI data, calculation outputs, and AI-generated text into a consolidated result object.
  - *Configuration:* Custom JavaScript generating a unique result ID (`RESULT-{code}-{period}-{timestamp}`) and joining array elements with pipeline separators.
  - *Expressions / Variables:* References `$json.output` from AI analysis and upstream node data.
  - *Connections:* Input: `Generate KPI Gap Analysis`; Outputs: `Append Benchmark Result` and `Build Monthly Benchmark Report`.
  - *Version Requirements:* v2.

- **Append Benchmark Result**
  - *Type & Role:* `n8n-nodes-base.googleSheets` (Data Output). Appends completed benchmark analyses to the `Benchmark_Results` worksheet.
  - *Configuration:* Operation: `append`. Maps fields including `result_id`, `analysis_date`, gaps, AI commentary, and processing status.
  - *Expressions / Variables:* `={{ $json.* }}`
  - *Connections:* Input: `Prepare Benchmark Result Row`; Output: `Mark KPI as Processed`.
  - *Version Requirements:* v4.7.

- **Mark KPI as Processed**
  - *Type & Role:* `n8n-nodes-base.googleSheets` (Data Output). Updates the source record in the `Procurement_KPIs` sheet to "Processed".
  - *Configuration:* Operation: `update`. Matching column: `kpi_record_id`.
  - *Expressions / Variables:* `={{ $('Load Benchmark Settings').first().json.processed_status }}`, `={{ $('Prepare Benchmark Result Row').item.json.kpi_record_id }}`
  - *Connections:* Input: `Append Benchmark Result`; Output: `Is Significant Gap?`.
  - *Version Requirements:* v4.7.

- **Is Significant Gap?**
  - *Type & Role:* `n8n-nodes-base.if` (Flow Control). Determines whether the detected gap requires an immediate leadership alert.
  - *Configuration:* Evaluates `={{ $('Calculate Benchmark Gap').item.json.benchmark_gap_detected }}`.
  - *Expressions / Variables:* `={{ $('Calculate Benchmark Gap').item.json.benchmark_gap_detected }}`
  - *Connections:* Input: `Mark KPI as Processed`; Outputs: True path to `Prepare Leadership Alert`, False path to `No Leadership Alert Required`.
  - *Version Requirements:* v2.3.

- **Prepare Leadership Alert**
  - *Type & Role:* `n8n-nodes-base.code` (Data Transformation). Formats alert payloads for significant gaps.
  - *Configuration:* Custom JavaScript generating an alert ID and summary titles.
  - *Expressions / Variables:* References `Prepare Benchmark Result Row` outputs.
  - *Connections:* Input: `Is Significant Gap?` (True); Output: `Send Leadership Alert Email`.
  - *Version Requirements:* v2.

- **Send Leadership Alert Email**
  - *Type & Role:* `n8n-nodes-base.gmail` (Data Output). Sends an HTML-formatted alert email to leadership regarding the below-benchmark KPI.
  - *Configuration:* Sends to recipient defined in `leadership_email`. Subject uses `={{ $json.alert_title }}`.
  - *Credentials:* Gmail OAuth2.
  - *Connections:* Input: `Prepare Leadership Alert`; Output: None.
  - *Version Requirements:* v2.2.
  - *Edge Cases / Failures:* Unconfigured `leadership_email` setting will cause email delivery failures.

- **No Leadership Alert Required**
  - *Type & Role:* `n8n-nodes-base.set` (Data Transformation). Sets a non-alert completion status for KPIs performing at or above benchmarks.
  - *Configuration:* Sets assignment variables indicating alert requirement is false.
  - *Expressions / Variables:* Static assignment values.
  - *Connections:* Input: `Is Significant Gap?` (False); Output: None.
  - *Version Requirements:* v3.4.

- **Build Monthly Benchmark Report**
  - *Type & Role:* `n8n-nodes-base.code` (Data Transformation). Aggregates all processed KPI benchmark results into a single consolidated HTML monthly executive report.
  - *Configuration:* Custom JavaScript filtering and sorting top gaps, calculating counts across status categories, and building an HTML executive summary table.
  - *Expressions / Variables:* Maps `$input.all()` and configuration settings.
  - *Connections:* Input: `Prepare Benchmark Result Row`; Output: `Send Monthly Benchmark Report`.
  - *Version Requirements:* v2.

- **Send Monthly Benchmark Report**
  - *Type & Role:* `n8n-nodes-base.gmail` (Data Output). Emails the consolidated monthly HTML report to management.
  - *Configuration:* Sends to recipient defined in `monthly_report_email`. Subject uses `={{ $json.report_title }}`.
  - *Credentials:* Gmail OAuth2.
  - *Connections:* Input: `Build Monthly Benchmark Report`; Output: None.
  - *Version Requirements:* v2.2.
  - *Edge Cases / Failures:* Unconfigured `monthly_report_email` setting will cause email delivery failures.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Monthly KPI Benchmark Trigger | n8n-nodes-base.scheduleTrigger | Triggers the workflow monthly | None | Load Benchmark Settings | Procurement Metrics Benchmarking Workflow<br>Trigger & Configuration |
| Load Benchmark Settings | n8n-nodes-base.set | Defines global configuration variables | Monthly KPI Benchmark Trigger | Get Ready Procurement KPIs | Procurement Metrics Benchmarking Workflow<br>Trigger & Configuration |
| Get Ready Procurement KPIs | n8n-nodes-base.googleSheets | Fetches ready KPI rows from Google Sheets | Load Benchmark Settings | Get KPI Definition | Procurement Metrics Benchmarking Workflow<br>KPI & Benchmark Retrieval |
| Get KPI Definition | n8n-nodes-base.googleSheets | Retrieves active KPI definitions | Get Ready Procurement KPIs | KPI Definition Found? | Procurement Metrics Benchmarking Workflow<br>KPI & Benchmark Retrieval |
| KPI Definition Found? | n8n-nodes-base.if | Validates KPI definition existence | Get KPI Definition | Get External Benchmark, Handle Missing KPI Definition | Procurement Metrics Benchmarking Workflow<br>KPI & Benchmark Retrieval |
| Handle Missing KPI Definition | n8n-nodes-base.set | Prepares error state for missing definition | KPI Definition Found? | Mark KPI as Review Required | Procurement Metrics Benchmarking Workflow<br>KPI & Benchmark Retrieval |
| Get External Benchmark | n8n-nodes-base.googleSheets | Retrieves external peer benchmarks | KPI Definition Found? | External Benchmark Found? | Procurement Metrics Benchmarking Workflow<br>KPI & Benchmark Retrieval |
| External Benchmark Found? | n8n-nodes-base.if | Validates external benchmark existence | Get External Benchmark | Build Benchmark Comparison Context, Handle Missing External Benchmark | Procurement Metrics Benchmarking Workflow<br>KPI & Benchmark Retrieval |
| Handle Missing External Benchmark | n8n-nodes-base.set | Prepares error state for missing benchmark | External Benchmark Found? | Mark KPI as Review Required | Procurement Metrics Benchmarking Workflow<br>KPI & Benchmark Retrieval |
| Build Benchmark Comparison Context | n8n-nodes-base.code | Consolidates KPI and benchmark context | External Benchmark Found? | Validate Benchmark Inputs | Validation & Benchmark Calculation |
| Validate Benchmark Inputs | n8n-nodes-base.code | Validates numeric ranges and thresholds | Build Benchmark Comparison Context | Benchmark Inputs Valid? | Validation & Benchmark Calculation |
| Benchmark Inputs Valid? | n8n-nodes-base.if | Checks input validation status | Validate Benchmark Inputs | Calculate Benchmark Gap, Handle Invalid Benchmark Data | Validation & Benchmark Calculation |
| Handle Invalid Benchmark Data | n8n-nodes-base.set | Prepares error state for invalid data | Benchmark Inputs Valid? | Mark KPI as Review Required | Validation & Benchmark Calculation |
| Mark KPI as Review Required | n8n-nodes-base.googleSheets | Flags source KPI as review required | Handle Missing KPI Definition, Handle Missing External Benchmark, Handle Invalid Benchmark Data | None | Validation & Benchmark Calculation |
| Calculate Benchmark Gap | n8n-nodes-base.code | Calculates absolute gap, status, and severity | Benchmark Inputs Valid? | Generate KPI Gap Analysis | Validation & Benchmark Calculation |
| Gemini Model for KPI Analysis | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | Configures Google Gemini LLM model | None | Generate KPI Gap Analysis | AI Analysis & Recommendations |
| Parse Structured KPI Analysis | @n8n/n8n-nodes-outputParserStructured | Enforces JSON output schema for Gemini | None | Generate KPI Gap Analysis | AI Analysis & Recommendations |
| Generate KPI Gap Analysis | @n8n/n8n-nodes-langchain.chainLlm | Generates AI gap analysis via Gemini | Calculate Benchmark Gap, Gemini Model for KPI Analysis, Parse Structured KPI Analysis | Prepare Benchmark Result Row | AI Analysis & Recommendations |
| Prepare Benchmark Result Row | n8n-nodes-base.code | Formats combined analysis results | Generate KPI Gap Analysis | Append Benchmark Result, Build Monthly Benchmark Report | Results, Alerts & Reporting |
| Append Benchmark Result | n8n-nodes-base.googleSheets | Saves completed result to Google Sheets | Prepare Benchmark Result Row | Mark KPI as Processed | Results, Alerts & Reporting |
| Mark KPI as Processed | n8n-nodes-base.googleSheets | Updates source KPI status to Processed | Append Benchmark Result | Is Significant Gap? | Results, Alerts & Reporting |
| Is Significant Gap? | n8n-nodes-base.if | Checks if gap requires leadership alert | Mark KPI as Processed | Prepare Leadership Alert, No Leadership Alert Required | Results, Alerts & Reporting |
| Prepare Leadership Alert | n8n-nodes-base.code | Formats alert payload | Is Significant Gap? | Send Leadership Alert Email | Results, Alerts & Reporting |
| Send Leadership Alert Email | n8n-nodes-base.gmail | Sends leadership alert email | Prepare Leadership Alert | None | Results, Alerts & Reporting |
| No Leadership Alert Required | n8n-nodes-base.set | Handles non-alert execution path | Is Significant Gap? | None | Results, Alerts & Reporting |
| Build Monthly Benchmark Report | n8n-nodes-base.code | Builds consolidated HTML monthly report | Prepare Benchmark Result Row | Send Monthly Benchmark Report | Results, Alerts & Reporting |
| Send Monthly Benchmark Report | n8n-nodes-base.gmail | Emails monthly report to management | Build Monthly Benchmark Report | None | Results, Alerts & Reporting |
| Sticky Note | n8n-nodes-base.stickyNote | Workflow overview documentation | None | None | Procurement Metrics Benchmarking Workflow |
| Sticky Note1 | n8n-nodes-base.stickyNote | Trigger & Configuration documentation | None | None | Trigger & Configuration |
| Sticky Note2 | n8n-nodes-base.stickyNote | KPI & Benchmark Retrieval documentation | None | None | KPI & Benchmark Retrieval |
| Sticky Note3 | n8n-nodes-base.stickyNote | Validation & Benchmark Calculation documentation | None | None | Validation & Benchmark Calculation |
| Sticky Note4 | n8n-nodes-base.stickyNote | AI Analysis & Recommendations documentation | None | None | AI Analysis & Recommendations |
| Sticky Note5 | n8n-nodes-base.stickyNote | Results, Alerts & Reporting documentation | None | None | Results, Alerts & Reporting |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually in n8n:

1. **Create Workflow & Trigger**
   - Create a new n8n workflow.
   - Add a **Schedule Trigger** node (`n8n-nodes-base.scheduleTrigger`). Set interval to monthly at 9:00 AM. Name it `Monthly KPI Benchmark Trigger`.

2. **Configure Global Settings**
   - Add a **Set** node (`n8n-nodes-base.set`) named `Load Benchmark Settings`.
   - Add assignments: `ready_status` = "Ready", `processed_status` = "Processed", `active_benchmark_value` = "Yes", `active_kpi_definition_value` = "Yes", `near_benchmark_threshold_pct` = 5, `max_recommendations` = 3, `report_status_completed` = "Completed", `review_required_status` = "Review Required", `leadership_email` = "", `monthly_report_email` = "", `alert_open_status` = "Open", `missing_definition_message` = "No active KPI definition was found for this KPI", `missing_benchmark_message` = "No active external benchmark was found for this KPI.", `report_title_prefix` = "Procurement KPI Benchmark Report".
   - Connect `Monthly KPI Benchmark Trigger` $\rightarrow$ `Load Benchmark Settings`.

3. **Fetch Ready KPIs**
   - Add a **Google Sheets** node (`n8n-nodes-base.googleSheets`) named `Get Ready Procurement KPIs`.
   - Set operation to `list` (lookup). Select your target spreadsheet and the `Procurement_KPIs` sheet.
   - Add a filter: `status` equal to `={{ $('Load Benchmark Settings').item.json.ready_status }}`.
   - Connect `Load Benchmark Settings` $\rightarrow$ `Get Ready Procurement KPIs`.

4. **Retrieve and Validate KPI Definition**
   - Add a **Google Sheets** node named `Get KPI Definition` targeting the `KPI_Definitions` sheet.
   - Lookup columns: `kpi_code` = `={{ $json.kpi_code }}` and `active` = `={{ $('Load Benchmark Settings').first().json.active_kpi_definition_value }}`.
   - Connect `Get Ready Procurement KPIs` $\rightarrow$ `Get KPI Definition`.
   - Add an **If** node (`n8n-nodes-base.if`) named `KPI Definition Found?`. Condition: `={{ Boolean($json.kpi_code) }}`.
   - Connect `Get KPI Definition` $\rightarrow$ `KPI Definition Found?`.
   - Add a **Set** node named `Handle Missing KPI Definition` (setting processing status to Review Required and `should_continue` to false). Connect the False output of `KPI Definition Found?` here.

5. **Retrieve and Validate External Benchmark**
   - Add a **Google Sheets** node named `Get External Benchmark` targeting the `External_Benchmarks` sheet.
   - Lookup columns: `kpi_code` = `={{ $json.kpi_code }}` and `active` = `={{ $('Load Benchmark Settings').first().json.active_benchmark_value }}`.
   - Connect the True output of `KPI Definition Found?` $\rightarrow$ `Get External Benchmark`.
   - Add an **If** node named `External Benchmark Found?`. Condition: `={{ Boolean($json.benchmark_id) }}`.
   - Connect `Get External Benchmark` $\rightarrow$ `External Benchmark Found?`.
   - Add a **Set** node named `Handle Missing External Benchmark`. Connect the False output of `External Benchmark Found?` here.

6. **Build Context & Validate Inputs**
   - Add a **Code** node (`n8n-nodes-base.code`) named `Build Benchmark Comparison Context` using the JavaScript implementation provided in the source workflow to map KPI, definition, and benchmark properties.
   - Connect the True output of `External Benchmark Found?` $\rightarrow$ `Build Benchmark Comparison Context`.
   - Add a **Code** node named `Validate Benchmark Inputs` to check numeric validity, non-zero benchmarks, and threshold logic.
   - Connect `Build Benchmark Comparison Context` $\rightarrow$ `Validate Benchmark Inputs`.
   - Add an **If** node named `Benchmark Inputs Valid?`. Condition: `={{ $json.benchmark_inputs_valid }}`.
   - Connect `Validate Benchmark Inputs` $\rightarrow$ `Benchmark Inputs Valid?`.
   - Add a **Set** node named `Handle Invalid Benchmark Data`. Connect the False output of `Benchmark Inputs Valid?` here.
   - Add a **Google Sheets** node named `Mark KPI as Review Required` (operation `update`, matching column `kpi_record_id`, updating `status` to review required). Connect `Handle Missing KPI Definition`, `Handle Missing External Benchmark`, and `Handle Invalid Benchmark Data` to this node.

7. **Calculate Gaps & AI Analysis**
   - Add a **Code** node named `Calculate Benchmark Gap` using the JavaScript implementation to compute absolute gaps, percentages, statuses, and severities.
   - Connect the True output of `Benchmark Inputs Valid?` $\rightarrow$ `Calculate Benchmark Gap`.
   - Add an **Advanced AI - Basic LLM Chain** node named `Generate KPI Gap Analysis`.
   - Add a **Google Gemini Chat Model** sub-node (`@n8n/n8n-nodes-langchain.lmChatGoogleGemini`) configured with Google Gemini credentials, connected to `Generate KPI Gap Analysis`.
   - Add a **Structured Output Parser** sub-node (`@n8n/n8n-nodes-outputParserStructured`) configured with the JSON schema requiring `gap_summary`, `likely_causes` (array 2-3 items), `recommended_actions` (array 1-3 items), and `management_comment`, connected to `Generate KPI Gap Analysis`.
   - Connect `Calculate Benchmark Gap` $\rightarrow$ `Generate KPI Gap Analysis`.

8. **Save Results, Flag Records & Report**
   - Add a **Code** node named `Prepare Benchmark Result Row` to format the combined AI and calculated results.
   - Connect `Generate KPI Gap Analysis` $\rightarrow$ `Prepare Benchmark Result Row`.
   - Add a **Google Sheets** node named `Append Benchmark Result` (operation `append`, targeting `Benchmark_Results` sheet) and connect `Prepare Benchmark Result Row` $\rightarrow$ `Append Benchmark Result`.
   - Add a **Google Sheets** node named `Mark KPI as Processed` (operation `update`, targeting `Procurement_KPIs`, updating status to Processed). Connect `Append Benchmark Result` $\rightarrow$ `Mark KPI as Processed`.
   - Add an **If** node named `Is Significant Gap?`. Condition: `={{ $('Calculate Benchmark Gap').item.json.benchmark_gap_detected }}`.
   - Connect `Mark KPI as Processed` $\rightarrow$ `Is Significant Gap?`.
   - Add a **Code** node named `Prepare Leadership Alert` and a **Gmail** node (`n8n-nodes-base.gmail`) named `Send Leadership Alert Email` (using Gmail OAuth2 credentials). Connect `Is Significant Gap?` (True) $\rightarrow$ `Prepare Leadership Alert` $\rightarrow$ `Send Leadership Alert Email`.
   - Add a **Set** node named `No Leadership Alert Required`. Connect `Is Significant Gap?` (False) $\rightarrow$ `No Leadership Alert Required`.
   - Add a **Code** node named `Build Monthly Benchmark Report` and a **Gmail** node named `Send Monthly Benchmark Report` (using Gmail OAuth2 credentials). Connect `Prepare Benchmark Result Row` $\rightarrow$ `Build Monthly Benchmark Report` $\rightarrow$ `Send Monthly Benchmark Report`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Mock Data Sheet Template Reference | [Google Sheets Mock Data Template](https://docs.google.com/spreadsheets/d/1Sw0ES-X9PR_fzoQLG9wYGYWMo_IsKb7KiVSouTWS-iY/edit?usp=drivesdk) |
| KPI Definitions Worksheet ID | GID: `2068041816` (`KPI_Definitions`) |
| External Benchmarks Worksheet ID | GID: `65023634` (`External_Benchmarks`) |
| Procurement KPIs Worksheet ID | GID: `838784183` (`Procurement_KPIs`) |
| Benchmark Results Worksheet ID | GID: `448979579` (`Benchmark_Results`) |