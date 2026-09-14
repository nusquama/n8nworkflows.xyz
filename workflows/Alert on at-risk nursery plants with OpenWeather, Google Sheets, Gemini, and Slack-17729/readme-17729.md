Alert on at-risk nursery plants with OpenWeather, Google Sheets, Gemini, and Slack

https://n8nworkflows.xyz/workflows/alert-on-at-risk-nursery-plants-with-openweather--google-sheets--gemini--and-slack-17729


# Alert on at-risk nursery plants with OpenWeather, Google Sheets, Gemini, and Slack

### 1. Workflow Overview

This workflow automates nightly risk monitoring for specialty plant nurseries. Operating on a daily schedule, it correlates real-time regional weather forecasts with an inventory database of plant species, delegates logical cross-referencing to an AI model, and issues high-priority notifications if any inventory is threatened by extreme temperatures.

The execution logic groups into five distinct functional blocks:
- **1.1 Trigger & Weather Retrieval:** Initiates the cycle daily at 7:00 PM IST, defines geographic and unit parameters, and pulls the OpenWeatherMap multi-day forecast.
- **1.2 Inventory Processing:** Summarizes the raw meteorological window, fetches live plant records from a Google Sheets database, and structures them into a unified payload.
- **1.3 AI-Powered Risk Analysis:** Compiles a targeted analysis prompt, utilizes a Google Gemini language model paired with a structured JSON output parser, and evaluates plant vulnerabilities.
- **1.4 Evaluation & Formatting:** Parses the AI output, verifies whether any risks exist using a conditional filter, and formats a human-readable notification payload.
- **1.5 Notification Delivery:** Dispatches the final aggregated emergency message to a designated Slack channel.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Weather Retrieval
**Overview:** This block establishes the execution schedule, initializes location configuration variables, and queries the OpenWeatherMap API for an incoming meteorological forecast.
- **Nodes Involved:** `Every Day at 7pm IST`, `Set Weather Parameters`, `Fetch Weather Forecast`

##### Node Details:
- **Every Day at 7pm IST**
  - **Type & Role:** `n8n-nodes-base.scheduleTrigger` (Schedule Trigger). Initiates the workflow.
  - **Configuration:** Cron expression set to `30 13 * * *` (corresponding to 7:00 PM IST / 1:30 PM UTC).
  - **Connections:** Output connects to `Set Weather Parameters`.
  - **Edge Cases:** Execution depends on the n8n instance time zone settings.

- **Set Weather Parameters**
  - **Type & Role:** `n8n-nodes-base.set` (Edit Fields). Defines geographic coordinates and metric preferences.
  - **Configuration:** Assigns string values: `lat` ("17.3850"), `lon` ("78.4867"), and `units` ("imperial").
  - **Connections:** Input from `Every Day at 7pm IST`; output to `Fetch Weather Forecast`.
  - **Edge Cases:** Invalid coordinate formats will cause the subsequent HTTP request to fail.

- **Fetch Weather Forecast**
  - **Type & Role:** `n8n-nodes-base.httpRequest` (HTTP Request). Queries the OpenWeatherMap API.
  - **Configuration:** GET request to `https://api.openweathermap.org/data/2.5/forecast`. Uses generic query authentication (`httpQueryAuth`). Maps query parameters dynamically from preceding fields (`lat`, `lon`, `units`).
  - **Credentials:** Uses credential `weatherapi` (`httpQueryAuth`).
  - **Input/Output Connections:** Input from `Set Weather Parameters`; output to `Summarize Overnight Weather`.
  - **Edge Cases:** API rate limits, invalid API keys, or network timeouts.

---

#### 2.2 Inventory Processing
**Overview:** This block extracts the relevant overnight metrics from the raw forecast, pulls raw inventory rows from a Google Sheets document, and aggregates them into a consolidated array.
- **Nodes Involved:** `Summarize Overnight Weather`, `Read Plants from Sheets`, `Aggregate Plants List`

##### Node Details:
- **Summarize Overnight Weather**
  - **Type & Role:** `n8n-nodes-base.set` (Edit Fields). Computes forecast extremes and conditions.
  - **Key Expressions:** 
    - `overnight_low_f`: `={{ Math.min(...$json.list.slice(0,8).map(i => i.main.temp_min)) }}`
    - `overnight_high_f`: `={{ Math.max(...$json.list.slice(0,8).map(i => i.main.temp_max)) }}`
    - `conditions`: `={{ $json.list[0].weather[0].description }}`
    - `forecast_window`: `={{ $json.list[0].dt_txt }} to {{ $json.list[7].dt_txt }}`
  - **Input/Output Connections:** Input from `Fetch Weather Forecast`; output to `Read Plants from Sheets`.
  - **Edge Cases:** Empty array responses from OpenWeatherMap will cause Math operations to evaluate incorrectly (`Infinity` or `-Infinity`).

- **Read Plants from Sheets**
  - **Type & Role:** `n8n-nodes-base.googleSheets` (Google Sheets). Retrieves plant inventory records.
  - **Configuration:** Retrieves rows from document ID `1IaS6CTDF3zjndUy_fqKfx4AsoQjdvmUsawVRFqowI8g`, Sheet name `Sheet1` (`gid=0`) using Google Service Account authentication.
  - **Credentials:** Uses Google Service Account credential.
  - **Input/Output Connections:** Input from `Summarize Overnight Weather`; output to `Aggregate Plants List`.
  - **Edge Cases:** Missing spreadsheet columns, broken share permissions on the Service Account, or empty sheets.

- **Aggregate Plants List**
  - **Type & Role:** `n8n-nodes-base.aggregate` (Aggregate). Combines individual spreadsheet rows into a single list item.
  - **Configuration:** Aggregation mode set to `aggregateAllItemData`, assigning the collected items to the destination field `plants`.
  - **Input/Output Connections:** Input from `Read Plants from Sheets`; output to `Build Plant Risk Prompt`.

---

#### 2.3 AI-Powered Risk Analysis
**Overview:** This block formats the prompt using aggregated inventory and weather summaries, routes the data to a Google Gemini language model, and parses the output into a strict schema.
- **Nodes Involved:** `Build Plant Risk Prompt`, `Plant Risk Analyzer Agent`, `Google Gemini Model`, `Parse AI Output Structure`

##### Node Details:
- **Build Plant Risk Prompt**
  - **Type & Role:** `n8n-nodes-base.set` (Edit Fields). Constructs the contextual analysis prompt string.
  - **Key Expressions:** Evaluates window, temperatures, conditions, and uses `JSON.stringify($json.plants)` to inject inventory items into a structured prompt template referencing upstream nodes (`$('Summarize Overnight Weather')`).
  - **Input/Output Connections:** Input from `Aggregate Plants List`; output to `Plant Risk Analyzer Agent`.

- **Plant Risk Analyzer Agent**
  - **Type & Role:** `@n8n/n8n-nodes-langchain.agent` (AI Agent). Orchestrates the reasoning task.
  - **Configuration:** Text input driven by `={{ $json.prompt }}`. System message enforces a strict horticulturist persona focused entirely on structured alert output.
  - **Input/Output Connections:** Main input from `Build Plant Risk Prompt`; connected to `Google Gemini Model` (via `ai_languageModel`) and `Parse AI Output Structure` (via `ai_outputParser`). Main output connects to `Parse AI Response`.

- **Google Gemini Model**
  - **Type & Role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` (Google Gemini Chat Model). Provides the underlying LLM engine.
  - **Configuration:** Model name set to `models/gemini-3.1-flash-lite`.
  - **Credentials:** Uses Google Palm API credential (`vaar@blankarray`).
  - **Input/Output Connections:** Connects to `Plant Risk Analyzer Agent` via port `ai_languageModel`.

- **Parse AI Output Structure**
  - **Type & Role:** `@n8n/n8n-nodes-langchain.outputParserStructured` (Structured Output Parser). Enforces schema compliance on the LLM's response.
  - **Configuration:** Manual schema type defining an `alerts` array containing objects with properties: `plant_name`, `batch_id`, `risk_type`, and `message`.
  - **Input/Output Connections:** Connects to `Plant Risk Analyzer Agent` via port `ai_outputParser`.

---

#### 2.4 Evaluation & Formatting
**Overview:** This block extracts the parsed alert array, evaluates whether any risks were flagged, and maps the alerts into a unified notification string.
- **Nodes Involved:** `Parse AI Response`, `If Plants at Risk`, `Prepare Alert Message`

##### Node Details:
- **Parse AI Response**
  - **Type & Role:** `n8n-nodes-base.set` (Edit Fields). Extracts the array from the AI wrapper object.
  - **Key Expressions:** `alerts`: `={{ $json.output.alerts }}`.
  - **Input/Output Connections:** Input from `Plant Risk Analyzer Agent`; output to `If Plants at Risk`.

- **If Plants at Risk**
  - **Type & Role:** `n8n-nodes-base.if` (If). Evaluates whether the alert array contains items.
  - **Configuration:** Checks if expression `={{ $json.alerts.length }}` is greater than `0`.
  - **Input/Output Connections:** Input from `Parse AI Response`; output branch (true) connects to `Prepare Alert Message`. (False branch is left unassigned).

- **Prepare Alert Message**
  - **Type & Role:** `n8n-nodes-base.set` (Edit Fields). Formats the alerts into a stylized Markdown string.
  - **Key Expressions:** Maps over `$json.alerts` to generate itemized warning blocks featuring plant name, batch identifier, and specific advice.
  - **Input/Output Connections:** Input from `If Plants at Risk` (true branch); output to `Send a message`.

---

#### 2.5 Notification Delivery
**Overview:** Delivers the formatted critical alert message to the designated Slack channel.
- **Nodes Involved:** `Send a message`

##### Node Details:
- **Send a message**
  - **Type & Role:** `n8n-nodes-base.slack` (Slack). Sends chat notifications.
  - **Configuration:** Resource set to `message`, operation set to `post`. Selects channel mode, pointing to target channel `C0B8VH1M5PX` (`general`). Text field populated via `={{ $json.message }}`.
  - **Credentials:** Uses Slack account credential (`uBs2LjzTiloElvwT`).
  - **Input/Output Connections:** Input from `Prepare Alert Message`.
  - **Edge Cases:** Expired Slack credentials, missing bot scopes (`chat:write`), or invalid channel IDs.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Every Day at 7pm IST` | `scheduleTrigger` | Daily execution timer | None | `Set Weather Parameters` | ## Untitled workflow<br><br>### How it works<br><br>1. The workflow triggers daily at 7 PM IST to check the upcoming weather.<br>2. It retrieves local weather data and current plant inventory from Google Sheets.<br>3. An AI agent analyzes the weather forecast against plant needs to identify risks.<br>4. If any plants are at risk, the workflow constructs and sends a Slack alert.<br><br>### Setup steps<br><br>- - [ ] Configure the Schedule Trigger for your preferred time zone.<br>- - [ ] Set your OpenWeatherMap API key and coordinates in the Config node.<br>- - [ ] Connect your Google Sheet containing the plant inventory data.<br>- - [ ] Add your Google Gemini API key to the AI Agent credentials.<br>- - [ ] Choose a valid Slack channel. |
| `Set Weather Parameters` | `set` | Defines coordinate and unit parameters | `Every Day at 7pm IST` | `Fetch Weather Forecast` | ## Fetch and process weather data<br><br>Trigger the daily weather check and retrieve the forecast. |
| `Fetch Weather Forecast` | `httpRequest` | Requests OpenWeatherMap forecast data | `Set Weather Parameters` | `Summarize Overnight Weather` | ## Fetch and process weather data<br><br>Trigger the daily weather check and retrieve the forecast. |
| `Summarize Overnight Weather` | `set` | Computes overnight low/high and conditions | `Fetch Weather Forecast` | `Read Plants from Sheets` | ## Fetch and process weather data<br><br>Trigger the daily weather check and retrieve the forecast. |
| `Read Plants from Sheets` | `googleSheets` | Pulls plant inventory rows | `Summarize Overnight Weather` | `Aggregate Plants List` | ## Retrieve inventory and build prompt<br><br>Fetch plant data and prepare the AI prompt. |
| `Aggregate Plants List` | `aggregate` | Consolidates plant rows into a single list | `Read Plants from Sheets` | `Build Plant Risk Prompt` | ## Retrieve inventory and build prompt<br><br>Fetch plant data and prepare the AI prompt. |
| `Build Plant Risk Prompt` | `set` | Generates structured prompt string | `Aggregate Plants List` | `Plant Risk Analyzer Agent` | ## Retrieve inventory and build prompt<br><br>Fetch plant data and prepare the AI prompt. |
| `Google Gemini Model` | `lmChatGoogleGemini` | Provides LLM chat capabilities | None | `Plant Risk Analyzer Agent` | ## Analyze data with Gemini AI<br><br>Analyze plant risk using the AI agent. |
| `Parse AI Output Structure` | `outputParserStructured` | Enforces JSON output schema | None | `Plant Risk Analyzer Agent` | ## Analyze data with Gemini AI<br><br>Analyze plant risk using the AI agent. |
| `Plant Risk Analyzer Agent` | `agent` | Analyzes risks via AI | `Build Plant Risk Prompt`, `Google Gemini Model`, `Parse AI Output Structure` | `Parse AI Response` | ## Analyze data with Gemini AI<br><br>Analyze plant risk using the AI agent. |
| `Parse AI Response` | `set` | Extracts alerts array from AI output | `Plant Risk Analyzer Agent` | `If Plants at Risk` | ## Conditional alert and notification delivery<br><br>Evaluate risk and notify via Slack. |
| `If Plants at Risk` | `if` | Evaluates if risk array is non-empty | `Parse AI Response` | `Prepare Alert Message` | ## Conditional alert and notification delivery<br><br>Evaluate risk and notify via Slack. |
| `Prepare Alert Message` | `set` | Formats alerts into Markdown message | `If Plants at Risk` | `Send a message` | ## Conditional alert and notification delivery<br><br>Evaluate risk and notify via Slack. |
| `Send a message` | `slack` | Posts emergency notification | `Prepare Alert Message` | None | ## Conditional alert and notification delivery<br><br>Evaluate risk and notify via Slack. |

---

### 4. Reproducing the Workflow from Scratch

Follow these sequential steps to rebuild the workflow manually in n8n:

1. **Create Schedule Trigger:**
   - Add a **Schedule Trigger** node. Set the interval type to `cronExpression` and value to `30 13 * * *`. Name the node `Every Day at 7pm IST`.
2. **Add Weather Parameter Setup:**
   - Add an **Edit Fields (Set)** node connected to the trigger. Name it `Set Weather Parameters`.
   - Add three string assignments:
     - `lat`: `17.3850`
     - `lon`: `78.4867`
     - `units`: `imperial`
3. **Configure Weather API Request:**
   - Add an **HTTP Request** node named `Fetch Weather Forecast`.
   - Method: `GET`, URL: `https://api.openweathermap.org/data/2.5/forecast`.
   - Set Authentication to `Generic Credential Type` -> `HTTP Query Auth`. Select or create your OpenWeatherMap credential named `weatherapi`.
   - Add query parameters mapping from node inputs: `lat` (`={{ $json.lat }}`), `lon` (`={{ $json.lon }}`), `units` (`={{ $json.units }}`).
4. **Summarize Meteorological Data:**
   - Add an **Edit Fields (Set)** node named `Summarize Overnight Weather`.
   - Configure Number assignment `overnight_low_f`: `={{ Math.min(...$json.list.slice(0,8).map(i => i.main.temp_min)) }}`.
   - Configure Number assignment `overnight_high_f`: `={{ Math.max(...$json.list.slice(0,8).map(i => i.main.temp_max)) }}`.
   - Configure String assignment `conditions`: `={{ $json.list[0].weather[0].description }}`.
   - Configure String assignment `forecast_window`: `={{ $json.list[0].dt_txt }} to {{ $json.list[7].dt_txt }}`.
5. **Retrieve Plant Inventory:**
   - Add a **Google Sheets** node named `Read Plants from Sheets`.
   - Configure authentication to use a **Google Service Account**. Select your document ID (`1IaS6CTDF3zjndUy_fqKfx4AsoQjdvmUsawVRFqowI8g`) and sheet name (`Sheet1`).
6. **Aggregate Inventory:**
   - Add an **Aggregate** node named `Aggregate Plants List`. Set aggregation mode to `Aggregate All Item Data` with destination field `plants`.
7. **Build Prompt:**
   - Add an **Edit Fields (Set)** node named `Build Plant Risk Prompt`.
   - Create a string assignment named `prompt` incorporating a horticulturist system directive, weather summary fields from upstream expressions, and `={{ JSON.stringify($json.plants) }}`.
8. **Configure AI Agent and Sub-Nodes:**
   - Add an **AI Agent** node named `Plant Risk Analyzer Agent`. Set text input to `={{ $json.prompt }}`. Set the system message to enforce structured alert output only.
   - Add a **Google Gemini Chat Model** node (`Google Gemini Model`), set the model name to `models/gemini-3.1-flash-lite`, and connect its output to the AI Agent's `ai_languageModel` input. Configure Google Palm API credentials.
   - Add a **Structured Output Parser** node (`Parse AI Output Structure`), configure a manual schema with an `alerts` array containing objects with keys `plant_name`, `batch_id`, `risk_type`, and `message`. Connect its output to the AI Agent's `ai_outputParser` input.
9. **Process AI Response:**
   - Add an **Edit Fields (Set)** node named `Parse AI Response` to map `alerts` to `={{ $json.output.alerts }}`.
   - Add an **If** node named `If Plants at Risk`. Configure condition checking if `={{ $json.alerts.length }}` is greater than `0`.
10. **Format and Deliver Alert:**
    - Add an **Edit Fields (Set)** node named `Prepare Alert Message`. Map a string assignment `message` that formats the alert array into a Markdown warning message using template literals (`⚠️ ...`).
    - Add a **Slack** node named `Send a message`. Set resource to `message`, operation to `post`, channel selection to `channel`, and select your target channel ID. Configure Slack API credentials. Connect the text parameter to `={{ $json.message }}`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Alert on at-risk nursery plants with OpenWeather, Google Sheets, Gemini, and Slack | Workflow Title / Core Project Overview |
| OpenWeatherMap API Documentation | [OpenWeather Forecast API](https://openweathermap.org/forecast5) |