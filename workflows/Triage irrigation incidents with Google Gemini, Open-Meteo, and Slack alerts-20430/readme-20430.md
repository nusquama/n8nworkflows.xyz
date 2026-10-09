Triage irrigation incidents with Google Gemini, Open-Meteo, and Slack alerts

https://n8nworkflows.xyz/workflows/triage-irrigation-incidents-with-google-gemini--open-meteo--and-slack-alerts-20430


# Triage irrigation incidents with Google Gemini, Open-Meteo, and Slack alerts

### 1. Workflow Overview

This workflow automates the ingestion, evaluation, and triage of irrigation system telemetry. Its primary purpose is to ingest field sensor readings, combine them with hyperlocal weather forecasts, use a Gemini-powered Large Language Model (LLM) to diagnose issues, and route work orders through appropriate communication and escalation paths.

The workflow logic is categorized into six functional blocks:
- **1.1 Input Reception & Normalization:** Ingests raw telemetry payloads via webhook and normalizes missing sensor attributes with default fallback values.
- **1.2 Weather Enrichment:** Queries the Open-Meteo API using zone coordinates to retrieve historical and forecasted meteorological data.
- **1.3 AI Diagnosis Generation:** Formats an agronomic evaluation prompt and uses Google Gemini with a structured output parser to analyze root causes and determine service levels.
- **1.4 Diagnostic Enhancement & Routing:** Normalizes AI output parameters, generates a unique ticket ID, and evaluates incident severity to branch execution.
- **1.5 Work Order Synthesis:** Assigns specific escalation channels, target audiences, operational urgencies, and compiles a comprehensive work-order object.
- **1.6 Notification & Response:** Publishes formatted incident updates to Slack and returns a structured JSON response to the original webhook caller.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Normalization
- **Overview:** Receives inbound irrigation data via HTTP POST and standardizes incoming metrics to ensure consistent data structures for downstream APIs.
- **Nodes Involved:** 
  - `When Telemetry Ingested`
  - `Set Sensor Data Parameters`
- **Node Details:**
  - **When Telemetry Ingested**
    - *Type and Role:* Webhook trigger (`n8n-nodes-base.webhook`). Listens for HTTP POST requests at `/irrigation-incident-inbound`.
    - *Configuration Choices:* Response mode set to wait for the response node.
    - *Input/Output:* No incoming connections; outputs to `Set Sensor Data Parameters`.
    - *Edge Cases:* Unauthenticated endpoints may receive malformed JSON; missing request bodies are handled by subsequent default assignments.
  - **Set Sensor Data Parameters**
    - *Type and Role:* Set/Edit Fields node (`n8n-nodes-base.set`). Normalizes raw webhook payloads or provides fallback defaults (e.g., `ZONE-4B-ORCHARD`, Almonds, default soil moisture and geographic coordinates).
    - *Input/Output:* Inputs from `When Telemetry Ingested`; outputs to `Fetch Local Weather Data`.
    - *Edge Cases:* Missing fields default to safe operational presets.

#### 2.2 Weather Enrichment
- **Overview:** Fetches historical and forecast precipitation and temperature data for the specific geographic coordinates of the irrigation zone.
- **Nodes Involved:** 
  - `Fetch Local Weather Data`
- **Node Details:**
  - **Fetch Local Weather Data**
    - *Type and Role:* HTTP Request node (`n8n-nodes-base.httpRequest`). Queries the Open-Meteo API.
    - *Configuration Choices:* GET request to `https://api.open-meteo.com/v1/forecast` using dynamic latitude and longitude query parameters with 1 day of past data and 3 days of forecast.
    - *Input/Output:* Inputs from `Set Sensor Data Parameters`; outputs to `Prepare AI Input Prompt`.
    - *Edge Cases:* API timeouts or rate limits from Open-Meteo; handled by workflow execution settings or external monitoring.

#### 2.3 AI Diagnosis Generation
- **Overview:** Constructs an agronomist prompt using sensor and weather data, invokes Google Gemini, and parses the response into a strict JSON schema.
- **Nodes Involved:** 
  - `Prepare AI Input Prompt`
  - `Evaluate Agronomy with AI`
  - `Activate Gemini Chat Model`
  - `Extract Structured Output`
- **Node Details:**
  - **Prepare AI Input Prompt**
    - *Type and Role:* Code node (`n8n-nodes-base.code`). Aggregates sensor metrics and weather forecasts to compose a structured evaluation prompt.
    - *Input/Output:* Inputs from `Fetch Local Weather Data` and `Set Sensor Data Parameters`; outputs to `Evaluate Agronomy with AI`.
  - **Evaluate Agronomy with AI**
    - *Type and Role:* Advanced AI Basic LLM Chain (`@n8n/n8n-nodes-langchain.chainLlm`). Executes the prompt against the language model.
    - *Input/Output:* Connects to model and parser sub-nodes via AI connection types; outputs to `Enhance Diagnostic Analysis`.
  - **Activate Gemini Chat Model**
    - *Type and Role:* Google Gemini Chat Model (`@n8n/n8n-nodes-langchain.lmChatGoogleGemini`). Configures the `models/gemini-3.1-flash-lite` model with a low temperature of `0.1`.
    - *Credentials:* Requires Google Palm API credentials (`STRICTLY USE GEMINI-3.1-FLASH-LITE`).
  - **Extract Structured Output**
    - *Type and Role:* Structured Output Parser (`@n8n/n8n-nodes-langchain.outputParserStructured`). Enforces a JSON schema containing `severity`, `incidentTitle`, `rootCauseAnalysis`, `slaResponseHours`, `actionItems`, and `requiresEscalation`.

#### 2.4 Diagnostic Enhancement & Routing
- **Overview:** Normalizes raw AI output, generates a unique ticket ID, and branches workflow execution based on the incident severity level.
- **Nodes Involved:** 
  - `Enhance Diagnostic Analysis`
  - `Route by Severity Level`
- **Node Details:**
  - **Enhance Diagnostic Analysis**
    - *Type and Role:* Code node (`n8n-nodes-base.code`). Parses raw LLM text outputs, normalizes severity enums (`CRITICAL`, `HIGH`, `ADVISORY`, `NORMAL`), computes SLA hours, and generates a unique ticket string (`IRR-[ZONE]-[TIMESTAMP}`).
    - *Input/Output:* Inputs from `Evaluate Agronomy with AI`; outputs to `Route by Severity Level`.
  - **Route by Severity Level**
    - *Type and Role:* Switch node (`n8n-nodes-base.switch`). Evaluates the normalized `severity` field and splits execution into four distinct conditional paths.
    - *Input/Output:* Inputs from `Enhance Diagnostic Analysis`; outputs to the four parameter-setting nodes.

#### 2.5 Work Order Synthesis
- **Overview:** Configures escalation channels, target audiences, urgency ratings, and work order statuses tailored to the specific severity branch before synthesizing the final object.
- **Nodes Involved:** 
  - `Set Critical Escalation Params`
  - `Set High Priority Params`
  - `Set Advisory Rain Params`
  - `Set Nominal Health Params`
  - `Generate Work Order Details`
- **Node Details:**
  - **Set Critical Escalation Params**
    - *Type and Role:* Set node (`n8n-nodes-base.set`). Sets escalation channel to `SMS_AND_PAGERDUTY_TIER_3`, urgency to `IMMEDIATE_ACTION_REQUIRED`, and status to `EMERGENCY_DISPATCHED`.
  - **Set High Priority Params**
    - *Type and Role:* Set node (`n8n-nodes-base.set`). Sets escalation channel to `SLACK_MAINTENANCE_CHANNEL`, audience to `Field Tech Queue`, and status to `QUEUED_HIGH_PRIORITY`.
  - **Set Advisory Rain Params**
    - *Type and Role:* Set node (`n8n-nodes-base.set`). Sets escalation channel to `DAILY_DIGEST_LOG`, urgency to `NONE_WEATHER_DEFERRED`, and status to `HOLD_PENDING_RAIN`.
  - **Set Nominal Health Params**
    - *Type and Role:* Set node (`n8n-nodes-base.set`). Sets escalation channel to `INTERNAL_AUDIT_LOG`, urgency to `NOMINAL`, and status to `NO_ACTION_REQUIRED`.
  - **Generate Work Order Details**
    - *Type and Role:* Code node (`n8n-nodes-base.code`). Calculates SLA deadlines and constructs the final structured work-order payload.
    - *Input/Output:* Inputs from all four parameter-setting nodes; outputs to `Post Update to Slack`.

#### 2.6 Notification & Response
- **Overview:** Dispatches the incident alert to Slack and returns the processing status and ticket details to the webhook initiator.
- **Nodes Involved:** 
  - `Post Update to Slack`
  - `Send Webhook Response`
- **Node Details:**
  - **Post Update to Slack**
    - *Type and Role:* Slack node (`n8n-nodes-base.slack`). Posts a formatted markdown message detailing the incident, priority, root cause, action items, and SLA deadline to a designated Slack channel.
    - *Credentials:* Requires Slack API credentials.
    - *Input/Output:* Inputs from `Generate Work Order Details`; outputs to `Send Webhook Response`.
  - **Send Webhook Response**
    - *Type and Role:* Respond to Webhook node (`n8n-nodes-base.respondToWebhook`). Returns an HTTP 200 status code containing the processed ticket payload to the original telemetry source.
    - *Input/Output:* Inputs from `Post Update to Slack`.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `When Telemetry Ingested` | `webhook` | Receive incoming telemetry | None | `Set Sensor Data Parameters` | Irrigation Incident Response<br><br>How it works<br>This workflow receives irrigation telemetry through a webhook, normalizes the sensor readings, and enriches them with hyperlocal weather data. It builds an agronomic prompt for a Gemini-powered LLM, parses the structured diagnosis, classifies severity, and routes the incident into critical, high-priority, advisory, or nominal paths. It then synthesizes a work-order style payload, sends a Slack message, and returns a webhook response.<br><br>Setup steps<br>- Configure the webhook endpoint and ensure incoming telemetry includes the expected fields such as zone ID, crop type, moisture percentage, threshold percentage, and any location data needed for weather lookup.<br>- Review the Open-Meteo HTTP request parameters so the forecast call uses the correct latitude, longitude, units, and forecast fields for each irrigation zone.<br>- Add Google Gemini credentials for the AI Agronomist Evaluator and confirm the structured output parser schema matches the fields expected by the downstream parsing code.<br>- Connect Slack credentials and set the target channel or routing logic for incident notifications.<br>- Verify the code nodes' field names align with the normalized telemetry, LLM output, and synthesized work-order payload before activating the workflow.<br><br>Customization<br>Adjust severity thresholds, escalation channels, target audiences, and Slack message formatting to match local irrigation operations and incident-response policies.<br><br>@[youtube](V2DyxYHqDnk) |
| `Set Sensor Data Parameters` | `set` | Normalize sensor properties | `When Telemetry Ingested` | `Fetch Local Weather Data` | Ingest and enrich telemetry<br><br>Receives incoming irrigation sensor telemetry, normalizes key fields, and fetches hyperlocal weather context before AI evaluation. |
| `Fetch Local Weather Data` | `httpRequest` | Query Open-Meteo API | `Set Sensor Data Parameters` | `Prepare AI Input Prompt` | Ingest and enrich telemetry<br><br>Receives incoming irrigation sensor telemetry, normalizes key fields, and fetches hyperlocal weather context before AI evaluation. |
| `Prepare AI Input Prompt` | `code` | Format prompt for LLM | `Fetch Local Weather Data` | `Evaluate Agronomy with AI` | Build AI diagnosis<br><br>Combines telemetry and weather into a prompt, runs the Gemini-backed agronomist evaluator, and uses a structured parser to produce machine-readable output. |
| `Evaluate Agronomy with AI` | `chainLlm` | Run LLM evaluation chain | `Prepare AI Input Prompt` | `Enhance Diagnostic Analysis` | Build AI diagnosis<br><br>Combines telemetry and weather into a prompt, runs the Gemini-backed agronomist evaluator, and uses a structured parser to produce machine-readable output. |
| `Activate Gemini Chat Model` | `lmChatGoogleGemini` | Provide Gemini model context | None | `Evaluate Agronomy with AI` | Build AI diagnosis<br><br>Combines telemetry and weather into a prompt, runs the Gemini-backed agronomist evaluator, and uses a structured parser to produce machine-readable output. |
| `Extract Structured Output` | `outputParserStructured` | Parse LLM output to JSON schema | None | `Evaluate Agronomy with AI` | Build AI diagnosis<br><br>Combines telemetry and weather into a prompt, runs the Gemini-backed agronomist evaluator, and uses a structured parser to produce machine-readable output. |
| `Enhance Diagnostic Analysis` | `code` | Normalize AI results and ticket ID | `Evaluate Agronomy with AI` | `Route by Severity Level` | Classify diagnosis severity<br><br>Parses and augments the AI result, then branches the workflow by severity or operational condition. |
| `Route by Severity Level` | `switch` | Branch workflow by severity | `Enhance Diagnostic Analysis` | Parameter set nodes | Classify diagnosis severity<br><br>Parses and augments the AI result, then branches the workflow by severity or operational condition. |
| `Set Critical Escalation Params` | `set` | Assign critical route metadata | `Route by Severity Level` | `Generate Work Order Details` | Route and synthesize order<br><br>Applies the selected escalation route for critical, high-priority, advisory rain-buffer, or nominal outcomes, then consolidates the selected route into a work-order payload. |
| `Set High Priority Params` | `set` | Assign high priority metadata | `Route by Severity Level` | `Generate Work Order Details` | Route and synthesize order<br><br>Applies the selected escalation route for critical, high-priority, advisory rain-buffer, or nominal outcomes, then consolidates the selected route into a work-order payload. |
| `Set Advisory Rain Params` | `set` | Assign advisory route metadata | `Route by Severity Level` | `Generate Work Order Details` | Route and synthesize order<br><br>Applies the selected escalation route for critical, high-priority, advisory rain-buffer, or nominal outcomes, then consolidates the selected route into a work-order payload. |
| `Set Nominal Health Params` | `set` | Assign nominal route metadata | `Route by Severity Level` | `Generate Work Order Details` | Route and synthesize order<br><br>Applies the selected escalation route for critical, high-priority, advisory rain-buffer, or nominal outcomes, then consolidates the selected route into a work-order payload. |
| `Generate Work Order Details` | `code` | Assemble work order payload | Parameter set nodes | `Post Update to Slack` | Route and synthesize order<br><br>Applies the selected escalation route for critical, high-priority, advisory rain-buffer, or nominal outcomes, then consolidates the selected route into a work-order payload. |
| `Post Update to Slack` | `slack` | Send alert notification | `Generate Work Order Details` | `Send Webhook Response` | Notify and acknowledge<br><br>Sends the synthesized incident or health message to Slack and returns the final response to the original webhook request. |
| `Send Webhook Response` | `respondToWebhook` | Return HTTP 200 response | `Post Update to Slack` | None | Notify and acknowledge<br><br>Sends the synthesized incident or health message to Slack and returns the final response to the original webhook request. |

---

### 4. Reproducing the Workflow from Scratch

Follow these steps to manually recreate the workflow:

1. **Create Webhook Trigger:**
   - Add a **Webhook** node (`When Telemetry Ingested`).
   - Set HTTP Method to `POST`, Path to `irrigation-incident-inbound`, and Response Mode to `Response node`.
2. **Add Sensor Normalization:**
   - Add a **Set** node (`Set Sensor Data Parameters`). Connect `When Telemetry Ingested` to it.
   - Configure string and numeric assignments for fields: `zoneId`, `cropType`, `moisturePercent`, `thresholdPercent`, `pumpState`, `lastRunHoursAgo`, `lat`, and `lon` using fallback expressions (e.g., `={{ $json.body?.zoneId || 'ZONE-4B-ORCHARD' }}`).
3. **Configure Weather API Request:**
   - Add an **HTTP Request** node (`Fetch Local Weather Data`). Connect `Set Sensor Data Parameters` to it.
   - Set URL to `https://api.open-meteo.com/v1/forecast`. Add query parameters: `latitude` (from previous node `lat`), `longitude` (`lon`), `daily` (`precipitation_sum,precipitation_probability_max,temperature_2m_max`), `past_days` (`1`), `forecast_days` (`3`), and `timezone` (`auto`).
4. **Prepare AI Input Prompt:**
   - Add a **Code** node (`Prepare AI Input Prompt`). Connect `Fetch Local Weather Data` to it.
   - Insert JavaScript code to extract weather metrics and build the agronomic prompt string.
5. **Set Up Advanced AI Components:**
   - Add an **Advanced AI Basic LLM Chain** node (`Evaluate Agronomy with AI`). Connect `Prepare AI Input Prompt` to its main input.
   - Add a **Google Gemini Chat Model** node (`Activate Gemini Chat Model`). Connect its model output port (`ai_languageModel`) to the LLM Chain. Configure model name to `models/gemini-3.1-flash-lite`, temperature to `0.1`, and provide Google Palm API credentials.
   - Add a **Structured Output Parser** node (`Extract Structured Output`). Connect its output port (`ai_outputParser`) to the LLM Chain. Define the JSON schema requiring `severity`, `incidentTitle`, `rootCauseAnalysis`, `slaResponseHours`, `actionItems`, and `requiresEscalation`.
6. **Enhance and Route Diagnostics:**
   - Add a **Code** node (`Enhance Diagnostic Analysis`). Connect `Evaluate Agronomy with AI` to it. Add parsing logic for severity and ticket ID generation.
   - Add a **Switch** node (`Route by Severity Level`). Connect `Enhance Diagnostic Analysis` to it. Configure 4 output rules matching `CRITICAL`, `HIGH`, `ADVISORY`, and `NORMAL` respectively.
7. **Configure Escalation Parameter Nodes:**
   - Add four **Set** nodes (`Set Critical Escalation Params`, `Set High Priority Params`, `Set Advisory Rain Params`, `Set Nominal Health Params`). Connect each corresponding output of the Switch node to one Set node. Configure operational metadata (escalation channel, target audience, dispatch urgency, work order status) for each severity tier.
8. **Synthesize Work Order:**
   - Add a **Code** node (`Generate Work Order Details`). Connect all four parameter-setting nodes to this single node. Implement logic to calculate SLA resolution deadlines and format the final work order object.
9. **Dispatch Notifications & Respond:**
   - Add a **Slack** node (`Post Update to Slack`). Connect `Generate Work Order Details` to it. Configure the text template using expressions referencing the ticket payload, and link valid Slack API credentials targeting the desired channel.
   - Add a **Respond to Webhook** node (`Send Webhook Response`). Connect `Post Update to Slack` to it with an HTTP response code of `200`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Irrigation Incident Response Video Tutorial | `@[youtube](V2DyxYHqDnk)` |
| Google Gemini Model Requirement | Must use `models/gemini-3.1-flash-lite` with Google Palm API credentials |