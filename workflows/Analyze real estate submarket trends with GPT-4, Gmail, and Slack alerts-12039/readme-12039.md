Analyze real estate submarket trends with GPT-4, Gmail, and Slack alerts

https://n8nworkflows.xyz/workflows/analyze-real-estate-submarket-trends-with-gpt-4--gmail--and-slack-alerts-12039


# Analyze real estate submarket trends with GPT-4, Gmail, and Slack alerts

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Analyze real estate submarket trends with GPT-4, Gmail, and Slack alerts  
**Workflow name (internal):** Autonomous Submarket Trend & Investment Opportunity Analyzer

**Purpose:**  
Runs a daily, fully automated pipeline that pulls multiple real-estate and macro data sources for a target market, aggregates them, asks GPT‑4 to identify investment opportunities and predicted value growth, and alerts the acquisition team/investors via Gmail and Slack when the opportunity score meets a configured threshold.

**Target use cases:**
- Real estate investment firms automating market scanning and deal sourcing
- Acquisition teams needing daily “what changed” and “where to focus” summaries
- Analysts who want consistent scoring + structured outputs for downstream systems

### 1.1 Scheduling & Runtime Configuration
Daily trigger starts the workflow and sets runtime variables such as API endpoints, target market, score threshold, and notification destinations.

### 1.2 Data Collection (Parallel HTTP Requests)
Fetches MLS, public records, demographics, and macroeconomic data in parallel from configured endpoints, scoped to the target market.

### 1.3 Aggregation & Normalization
Combines the multiple inbound datasets into a single payload for AI analysis.

### 1.4 AI Investment Analysis (GPT‑4 + Tools + Structured Output)
A LangChain agent calls GPT‑4o with a system role describing the analyst persona, can use a calculator tool and an optional external “market research” HTTP tool, and must return a JSON object matching a strict schema.

### 1.5 Threshold Gate & Notifications
If an investment score meets/exceeds the configured threshold, sends an HTML email to the acquisition team and posts a formatted Slack message.

---

## 2. Block-by-Block Analysis

### Block 2.1 — Scheduling & Workflow Configuration
**Overview:** Starts the workflow daily and defines all configurable parameters in one place (endpoints, market, threshold, and notification targets).  
**Nodes involved:** `Daily Analysis Schedule`, `Workflow Configuration`

#### Node: Daily Analysis Schedule
- **Type / role:** `Schedule Trigger` — entry point, time-based execution.
- **Configuration (interpreted):** Runs daily at **06:00** (instance timezone).
- **Connections:**  
  - **Output →** `Workflow Configuration`
- **Edge cases / failures:**
  - Timezone mismatch (server vs. expected local time).
  - If n8n is down at trigger time, the run may be missed unless using queue/retry strategies at platform level.
- **Version notes:** typeVersion **1.3**.

#### Node: Workflow Configuration
- **Type / role:** `Set` — central configuration object for the run.
- **Configuration (interpreted):**
  - Defines placeholders for:
    - `mlsApiUrl`, `publicRecordsApiUrl`, `demographicApiUrl`, `macroeconomicApiUrl`
    - `targetMarket`
    - `investmentThreshold` (default **75**)
    - `acquisitionTeamEmail`
    - `slackChannel` (channel ID)
  - **Include other fields:** enabled (passes through any inbound fields too).
- **Key variables used by other nodes:**
  - `$('Workflow Configuration').first().json.targetMarket`
  - `$('Workflow Configuration').first().json.investmentThreshold`
  - and the various `*ApiUrl` strings
- **Connections:**  
  - **Output →** `Fetch MLS Data`, `Fetch Public Records`, `Fetch Demographic Data`, `Fetch Macroeconomic Data` (fan-out / parallel)
- **Edge cases / failures:**
  - Placeholders not replaced → downstream HTTP nodes will fail (invalid URL) or notify nodes will fail (invalid email/channel).
  - `investmentThreshold` must be numeric; non-numeric will make IF comparisons unreliable.
- **Version notes:** typeVersion **3.4**.

---

### Block 2.2 — Market Data Collection (Parallel)
**Overview:** Retrieves market datasets from four sources, each scoped to the configured target market.  
**Nodes involved:** `Fetch MLS Data`, `Fetch Public Records`, `Fetch Demographic Data`, `Fetch Macroeconomic Data`

#### Node: Fetch MLS Data
- **Type / role:** `HTTP Request` — pulls listing/MLS dataset.
- **Configuration (interpreted):**
  - **URL:** from `mlsApiUrl`
  - **Query param:** `market={{ targetMarket }}`
  - Sends query parameters enabled.
- **Connections:**  
  - **Input ←** `Workflow Configuration`  
  - **Output →** `Aggregate All Data Sources`
- **Edge cases / failures:**
  - Auth missing (if endpoint requires headers/API key; not configured here).
  - Non-JSON response may break later assumptions.
  - Rate limits/timeouts.
- **Version notes:** typeVersion **4.3**.

#### Node: Fetch Public Records
- **Type / role:** `HTTP Request` — pulls deeds/permits/ownership or other public-record dataset.
- **Configuration (interpreted):**
  - **URL:** from `publicRecordsApiUrl`
  - **Query param:** `region={{ targetMarket }}`
- **Connections:**  
  - **Input ←** `Workflow Configuration`  
  - **Output →** `Aggregate All Data Sources`
- **Edge cases / failures:** same class as above (auth, rate limits, schema drift).
- **Version notes:** typeVersion **4.3**.

#### Node: Fetch Demographic Data
- **Type / role:** `HTTP Request` — pulls demographic indicators.
- **Configuration (interpreted):**
  - **URL:** from `demographicApiUrl`
  - **Query param:** `location={{ targetMarket }}`
- **Connections:**  
  - **Input ←** `Workflow Configuration`  
  - **Output →** `Aggregate All Data Sources`
- **Edge cases / failures:** same class as above.
- **Version notes:** typeVersion **4.3**.

#### Node: Fetch Macroeconomic Data
- **Type / role:** `HTTP Request` — pulls macro indicators (rates, employment, GDP proxies, etc.).
- **Configuration (interpreted):**
  - **URL:** from `macroeconomicApiUrl`
  - No explicit query parameters configured (endpoint must infer target market or be global).
- **Connections:**  
  - **Input ←** `Workflow Configuration`  
  - **Output →** `Aggregate All Data Sources`
- **Edge cases / failures:**
  - If this endpoint is global, the AI prompt may misinterpret non-market-specific data.
- **Version notes:** typeVersion **4.3**.

---

### Block 2.3 — Aggregation & Preparation for AI
**Overview:** Consolidates results from all fetch nodes into one combined object for analysis.  
**Nodes involved:** `Aggregate All Data Sources`

#### Node: Aggregate All Data Sources
- **Type / role:** `Aggregate` — merges multiple inbound items into one aggregate payload.
- **Configuration (interpreted):**
  - Mode: **Aggregate All Item Data** (collects all incoming data into one item).
- **Connections:**  
  - **Inputs ←** `Fetch MLS Data`, `Fetch Public Records`, `Fetch Demographic Data`, `Fetch Macroeconomic Data`
  - **Output →** `Investment Opportunity Analyzer`
- **Edge cases / failures:**
  - If one fetch node returns 0 items or errors (and workflow continues), aggregated structure may be missing expected keys.
  - Key naming: the downstream agent prompt expects fields like `mls_data`, `public_records`, `demographic_data`, `macroeconomic_data`, but this Aggregate node does not explicitly map names—so the actual aggregated JSON keys depend on how the HTTP responses are shaped and how Aggregate composes them. This is a primary integration risk.
- **Version notes:** typeVersion **1**.

---

### Block 2.4 — AI Analysis (Agent + GPT‑4 + Tools + Structured JSON)
**Overview:** A LangChain agent sends the aggregated data to GPT‑4o with a real-estate analyst system message, can use a calculator and an extra HTTP research tool, and must output JSON validated against a schema.  
**Nodes involved:** `Investment Opportunity Analyzer`, `OpenAI GPT-4`, `Structured Investment Output`, `Calculator Tool`, `Market Research Tool`

#### Node: Investment Opportunity Analyzer
- **Type / role:** `LangChain Agent` — orchestrates LLM calls + tools + structured output parsing.
- **Configuration (interpreted):**
  - **Prompt text:** Asks to analyze market data for `targetMarket` and includes:
    - `MLS Data: {{ JSON.stringify($json.mls_data) }}`
    - `Public Records: {{ JSON.stringify($json.public_records) }}`
    - `Demographic Data: {{ JSON.stringify($json.demographic_data) }}`
    - `Macroeconomic Data: {{ JSON.stringify($json.macroeconomic_data) }}`
  - **System message:** Defines analyst responsibilities; requires:
    - opportunity identification
    - 0–100 scoring
    - value growth prediction
    - actionable recommendations
    - JSON output matching the output parser schema
  - **Output parser:** enabled (`hasOutputParser: true`)
- **Connections:**
  - **Main input ←** `Aggregate All Data Sources`
  - **AI language model ←** `OpenAI GPT-4`
  - **AI tools ←** `Calculator Tool`, `Market Research Tool`
  - **AI output parser ←** `Structured Investment Output`
  - **Main output →** `Check Investment Threshold`
- **Edge cases / failures:**
  - **Critical mapping risk:** `$json.mls_data` etc. may be undefined if aggregation didn’t create those exact keys → prompt becomes “MLS Data: undefined”, reducing quality.
  - Token/size issues: stringifying large datasets can exceed model context.
  - Output parser failures if the model returns invalid JSON or misses required fields.
- **Version notes:** typeVersion **3.1**.

#### Node: OpenAI GPT-4
- **Type / role:** `lmChatOpenAi` — the LLM backend for the agent.
- **Configuration (interpreted):**
  - Model set to **gpt-4o**
  - Uses OpenAI API credential.
- **Connections:**  
  - **Output (AI language model) →** `Investment Opportunity Analyzer`
- **Failure modes:**
  - Invalid/expired API key, quota exceeded.
  - Model not available in the account/region.
  - Rate limiting.
- **Version notes:** typeVersion **1.3**.

#### Node: Structured Investment Output
- **Type / role:** `Structured Output Parser` — validates/coerces the LLM output into a JSON schema.
- **Configuration (interpreted):**
  - Manual JSON schema requiring:
    - `marketSummary` (string)
    - `opportunities` (array of objects with required fields including `location`, `propertyType`, `investmentScore`, `predictedValueGrowth`, `keyDrivers`, `recommendedAction`)
    - optional `topOpportunity` object
- **Connections:**  
  - **Output parser →** `Investment Opportunity Analyzer` (agent uses this to parse/validate)
- **Failure modes:**
  - Any schema mismatch (missing required fields, wrong types) causes parsing failure and may stop the workflow.
- **Version notes:** typeVersion **1.3**.

#### Node: Calculator Tool
- **Type / role:** `LangChain Calculator Tool` — allows the agent to compute metrics.
- **Configuration:** Default (no parameters).
- **Connections:**  
  - **Tool →** `Investment Opportunity Analyzer`
- **Edge cases:**
  - Only helps if the agent decides to call it; it does not compute anything by itself.
- **Version notes:** typeVersion **1**.

#### Node: Market Research Tool
- **Type / role:** `HTTP Request Tool` — optional tool the agent can call for extra data.
- **Configuration (interpreted):**
  - URL is a placeholder: “Additional market research API endpoint”
  - Description indicates it can fetch extra listings/economic indicators.
- **Connections:**  
  - **Tool →** `Investment Opportunity Analyzer`
- **Edge cases / failures:**
  - Placeholder not replaced → tool calls fail.
  - If it requires auth/headers, not configured here.
- **Version notes:** typeVersion **4.3**.

---

### Block 2.5 — Threshold Check & Notifications
**Overview:** Filters results using a numeric threshold and, if passed, sends an email and a Slack alert summarizing top opportunities and market context.  
**Nodes involved:** `Check Investment Threshold`, `Email Acquisition Team`, `Notify Investors on Slack`

#### Node: Check Investment Threshold
- **Type / role:** `IF` — gating logic.
- **Configuration (interpreted):**
  - Condition: `investmentScore >= investmentThreshold`
  - **Left value:** `{{ $('Investment Opportunity Analyzer').item.json.investmentScore }}`
  - **Right value:** `{{ $('Workflow Configuration').first().json.investmentThreshold }}`
- **Connections:**  
  - **Input ←** `Investment Opportunity Analyzer`  
  - **True output →** `Email Acquisition Team`, `Notify Investors on Slack`
  - (No false-path nodes configured.)
- **Edge cases / failures:**
  - **Potential bug:** The structured schema places `investmentScore` inside each entry in `opportunities[]`, not at the root. Unless the agent also outputs a root-level `investmentScore`, the expression may evaluate to `undefined` and the IF will never pass (or behave unpredictably).
  - If the agent returns multiple items, `.item` selection may not reference what you expect.
- **Version notes:** typeVersion **2.3**.

#### Node: Email Acquisition Team
- **Type / role:** `Gmail` — sends HTML email alert.
- **Configuration (interpreted):**
  - **To:** `acquisitionTeamEmail` from config
  - **Subject:** “🚨 High-Priority Investment Opportunity Detected - {{ topOpportunity.location }}”
  - **HTML body:** Includes:
    - market, analysis date
    - topOpportunity location/reason
    - marketSummary
    - table of opportunities filtered by threshold:
      - `opportunities.filter(o => o.investmentScore >= threshold)...`
- **Credentials:** Gmail OAuth2 credential required.
- **Connections:**  
  - **Input ←** `Check Investment Threshold` (true branch)
- **Failure modes:**
  - OAuth token expired / insufficient Gmail scopes.
  - `topOpportunity` missing → subject/body expressions may error or render blank.
  - If `opportunities` isn’t an array → `.filter` expression fails.
- **Version notes:** typeVersion **2.2**.

#### Node: Notify Investors on Slack
- **Type / role:** `Slack` — posts formatted message to a channel.
- **Configuration (interpreted):**
  - Authentication: OAuth2
  - Channel: ID from `slackChannel` config
  - Message includes market/date, topOpportunity, marketSummary, and bullet list of high-score opportunities using the same `filter/map/join` approach.
- **Credentials:** Slack OAuth2 credential required.
- **Connections:**  
  - **Input ←** `Check Investment Threshold` (true branch)
- **Failure modes:**
  - Invalid channel ID, bot not in channel, missing scopes (e.g., `chat:write`).
  - Same data-shape issues as email (missing `opportunities`, missing `topOpportunity`).
- **Version notes:** typeVersion **2.4**.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Daily Analysis Schedule | Schedule Trigger | Daily workflow entry point | — | Workflow Configuration | ## How It Works  Automates daily analysis of real estate market opportunities by aggregating MLS listings, public records, demographic data, and macroeconomic indicators, applying GPT-4 investment analysis, evaluating investment thresholds, and alerting investors to high-opportunity properties. Fetches MLS data, public property records, demographic profiles, and economic indicators simultaneously, consolidates datasets, applies GPT-4 for comprehensive investment analysis with ROI projections and risk assessment, calculates financial metrics using investment calculator, validates against investment thresholds, and notifies acquisition teams and investors via email and Slack. Targets real estate investors and property acquisition firms. |
| Workflow Configuration | Set | Central config variables (URLs, market, threshold, targets) | Daily Analysis Schedule | Fetch MLS Data; Fetch Public Records; Fetch Demographic Data; Fetch Macroeconomic Data | ## Setup Steps  1. Connect MLS data provider, public records database, and demographic data source APIs. 2. Configure OpenAI GPT-4 API for investment analysis and opportunity scoring. 3. Set up investment calculator integration and ROI calculation parameters. 4. Connect Gmail and Slack for team notifications; define investment thresholds. |
| Fetch MLS Data | HTTP Request | Pull MLS/listing data | Workflow Configuration | Aggregate All Data Sources | ## Aggregate Market Data  What: Fetches MLS listings, public records, demographics, and macroeconomic . Why: Provides comprehensive market context for data-driven investment decisions. |
| Fetch Public Records | HTTP Request | Pull public record data | Workflow Configuration | Aggregate All Data Sources | ## Aggregate Market Data  What: Fetches MLS listings, public records, demographics, and macroeconomic . Why: Provides comprehensive market context for data-driven investment decisions. |
| Fetch Demographic Data | HTTP Request | Pull demographic indicators | Workflow Configuration | Aggregate All Data Sources | ## Aggregate Market Data  What: Fetches MLS listings, public records, demographics, and macroeconomic . Why: Provides comprehensive market context for data-driven investment decisions. |
| Fetch Macroeconomic Data | HTTP Request | Pull macroeconomic indicators | Workflow Configuration | Aggregate All Data Sources | ## Aggregate Market Data  What: Fetches MLS listings, public records, demographics, and macroeconomic . Why: Provides comprehensive market context for data-driven investment decisions. |
| Aggregate All Data Sources | Aggregate | Combine all fetched datasets into one payload | Fetch MLS Data; Fetch Public Records; Fetch Demographic Data; Fetch Macroeconomic Data | Investment Opportunity Analyzer | ## Aggregate Market Data  What: Fetches MLS listings, public records, demographics, and macroeconomic . Why: Provides comprehensive market context for data-driven investment decisions. |
| Investment Opportunity Analyzer | LangChain Agent | GPT-driven opportunity detection + scoring + structured output | Aggregate All Data Sources | Check Investment Threshold | ## Evaluate Financial Metrics  What: Calculates cap rates, cash-on-cash returns, and investment thresholds  Why: Enables objective, quantitative screening of investment properties |
| OpenAI GPT-4 | OpenAI Chat Model (LangChain) | LLM used by the agent | — (AI connection) | Investment Opportunity Analyzer | ## Evaluate Financial Metrics  What: Calculates cap rates, cash-on-cash returns, and investment thresholds  Why: Enables objective, quantitative screening of investment properties |
| Structured Investment Output | Structured Output Parser (LangChain) | Enforce JSON schema for LLM output | — (AI connection) | Investment Opportunity Analyzer | ## Evaluate Financial Metrics  What: Calculates cap rates, cash-on-cash returns, and investment thresholds  Why: Enables objective, quantitative screening of investment properties |
| Calculator Tool | Calculator Tool (LangChain) | Agent tool for calculations | — (AI connection) | Investment Opportunity Analyzer | ## Evaluate Financial Metrics  What: Calculates cap rates, cash-on-cash returns, and investment thresholds  Why: Enables objective, quantitative screening of investment properties |
| Market Research Tool | HTTP Request Tool | Agent tool to fetch extra market data | — (AI connection) | Investment Opportunity Analyzer | ## Evaluate Financial Metrics  What: Calculates cap rates, cash-on-cash returns, and investment thresholds  Why: Enables objective, quantitative screening of investment properties |
| Check Investment Threshold | IF | Gate notifications based on score threshold | Investment Opportunity Analyzer | Email Acquisition Team; Notify Investors on Slack | ## Alert Acquisition Teams  What: Sends investment alerts to teams via email and Slack with analysis summaries. Why: Enables rapid response to qualifying opportunities. |
| Email Acquisition Team | Gmail | Email alert with HTML summary/table | Check Investment Threshold | — | ## Alert Acquisition Teams  What: Sends investment alerts to teams via email and Slack with analysis summaries. Why: Enables rapid response to qualifying opportunities. |
| Notify Investors on Slack | Slack | Slack channel alert | Check Investment Threshold | — | ## Alert Acquisition Teams  What: Sends investment alerts to teams via email and Slack with analysis summaries. Why: Enables rapid response to qualifying opportunities. |
| Sticky Note | Sticky Note | Documentation / prerequisites | — | — | ## Prerequisites MLS data access; public records database; demographic data provider; macroeconomic data source  ## Use Cases Real estate investment firms automating deal sourcing across markets;  ## Customization Adjust investment analysis criteria and thresholds  ## Benefits Identifies investment opportunities automatically |
| Sticky Note1 | Sticky Note | Documentation / setup steps | — | — | ## Setup Steps 1. Connect MLS data provider, public records database, and demographic data source APIs. 2. Configure OpenAI GPT-4 API for investment analysis and opportunity scoring. 3. Set up investment calculator integration and ROI calculation parameters. 4. Connect Gmail and Slack for team notifications; define investment thresholds. |
| Sticky Note2 | Sticky Note | Documentation / how it works | — | — | ## How It Works Automates daily analysis of real estate market opportunities by aggregating MLS listings, public records, demographic data, and macroeconomic indicators, applying GPT-4 investment analysis, evaluating investment thresholds, and alerting investors to high-opportunity properties. Fetches MLS data, public property records, demographic profiles, and economic indicators simultaneously, consolidates datasets, applies GPT-4 for comprehensive investment analysis with ROI projections and risk assessment, calculates financial metrics using investment calculator, validates against investment thresholds, and notifies acquisition teams and investors via email and Slack. Targets real estate investors and property acquisition firms. |
| Sticky Note4 | Sticky Note | Documentation / notifications rationale | — | — | ## Alert Acquisition Teams What: Sends investment alerts to teams via email and Slack with analysis summaries. Why: Enables rapid response to qualifying opportunities. |
| Sticky Note5 | Sticky Note | Documentation / metrics rationale | — | — | ## Evaluate Financial Metrics What: Calculates cap rates, cash-on-cash returns, and investment thresholds  Why: Enables objective, quantitative screening of investment properties |
| Sticky Note6 | Sticky Note | Documentation / aggregation rationale | — | — | ## Aggregate Market Data What: Fetches MLS listings, public records, demographics, and macroeconomic . Why: Provides comprehensive market context for data-driven investment decisions. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **Autonomous Submarket Trend & Investment Opportunity Analyzer**
   - (Optional) Add the title/description in your project documentation.

2. **Add Trigger: “Daily Analysis Schedule”**
   - Node type: **Schedule Trigger**
   - Set it to run **daily at 06:00** (adjust timezone as needed).

3. **Add “Workflow Configuration” (Set node)**
   - Node type: **Set**
   - Add fields:
     - `mlsApiUrl` (string) – your MLS endpoint
     - `publicRecordsApiUrl` (string)
     - `demographicApiUrl` (string)
     - `macroeconomicApiUrl` (string)
     - `targetMarket` (string, e.g., “Austin, TX”)
     - `investmentThreshold` (number, default 75)
     - `acquisitionTeamEmail` (string)
     - `slackChannel` (string: Slack channel ID)
   - Enable **Include Other Fields**
   - Connect: **Daily Analysis Schedule → Workflow Configuration**

4. **Add 4 HTTP Request nodes (data fetchers)**
   - Node types: **HTTP Request**
   - Configure each URL to reference the Set node:
     - **Fetch MLS Data**
       - URL: `{{ $('Workflow Configuration').first().json.mlsApiUrl }}`
       - Enable “Send Query Parameters”
       - Query: `market = {{ $('Workflow Configuration').first().json.targetMarket }}`
     - **Fetch Public Records**
       - URL: `{{ $('Workflow Configuration').first().json.publicRecordsApiUrl }}`
       - Query: `region = {{ targetMarket }}`
     - **Fetch Demographic Data**
       - URL: `{{ $('Workflow Configuration').first().json.demographicApiUrl }}`
       - Query: `location = {{ targetMarket }}`
     - **Fetch Macroeconomic Data**
       - URL: `{{ $('Workflow Configuration').first().json.macroeconomicApiUrl }}`
   - Connect: **Workflow Configuration → each HTTP node** (fan-out)

   > If your APIs require authentication, add headers (API keys/Bearer tokens) in each HTTP node or use n8n credentials where supported.

5. **Add “Aggregate All Data Sources”**
   - Node type: **Aggregate**
   - Operation: **Aggregate All Item Data**
   - Connect each HTTP node output to the Aggregate node:
     - Fetch MLS Data → Aggregate
     - Fetch Public Records → Aggregate
     - Fetch Demographic Data → Aggregate
     - Fetch Macroeconomic Data → Aggregate

6. **Add AI components**
   1) **OpenAI model node (“OpenAI GPT-4”)**
   - Node type: **OpenAI Chat Model (LangChain)** (`lmChatOpenAi`)
   - Model: **gpt-4o**
   - Credentials: create/select **OpenAI API** credential

   2) **Structured Output Parser (“Structured Investment Output”)**
   - Node type: **Structured Output Parser**
   - Schema: paste the workflow’s manual JSON schema (object with `opportunities`, `marketSummary`, optional `topOpportunity`, etc.)

   3) **Tools**
   - Add **Calculator Tool** node (no config).
   - Add **HTTP Request Tool** node (“Market Research Tool”)
     - Set URL to your external research endpoint
     - Add a clear tool description (what it returns, parameters)

   4) **Agent node (“Investment Opportunity Analyzer”)**
   - Node type: **AI Agent (LangChain)**
   - Set **Prompt type** to “Define”
   - Put the same system message (real estate analyst instructions)
   - In the user text, include target market and stringified datasets.
   - Enable output parser usage.
   - Connect:
     - **Aggregate All Data Sources → Investment Opportunity Analyzer** (main)
     - **OpenAI GPT-4 → Investment Opportunity Analyzer** (AI language model connection)
     - **Structured Investment Output → Investment Opportunity Analyzer** (AI output parser connection)
     - **Calculator Tool → Investment Opportunity Analyzer** (AI tool)
     - **Market Research Tool → Investment Opportunity Analyzer** (AI tool)

7. **Add gating logic: “Check Investment Threshold”**
   - Node type: **IF**
   - Condition: number `>=`
   - Left value (as in workflow): `{{ $('Investment Opportunity Analyzer').item.json.investmentScore }}`
   - Right value: `{{ $('Workflow Configuration').first().json.investmentThreshold }}`
   - Connect: **Investment Opportunity Analyzer → Check Investment Threshold**

   > Recommended adjustment when rebuilding: compare against the *maximum* opportunity score in `opportunities[]` (since the schema nests scores). Otherwise notifications may never trigger.

8. **Add notifications**
   1) **Gmail node (“Email Acquisition Team”)**
   - Node type: **Gmail**
   - Operation: send email
   - Credentials: **Gmail OAuth2**
   - To: `{{ acquisitionTeamEmail }}`
   - Subject/body: use the provided HTML template, including the threshold-filtered opportunities table.
   - Connect: **Check Investment Threshold (true) → Email Acquisition Team**

   2) **Slack node (“Notify Investors on Slack”)**
   - Node type: **Slack**
   - Authentication: OAuth2
   - Select: “channel”
   - Channel ID: `{{ $('Workflow Configuration').first().json.slackChannel }}`
   - Text: use the provided Slack formatted message with filtered opportunities.
   - Credentials: **Slack OAuth2** (ensure `chat:write` and channel access)
   - Connect: **Check Investment Threshold (true) → Notify Investors on Slack**

9. **(Optional) Add sticky notes**
   - Add sticky notes for prerequisites, setup steps, aggregation rationale, metrics rationale, and notification rationale as in the original.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| **Prerequisites:** MLS data access; public records database; demographic data provider; macroeconomic data source. **Use Cases:** Real estate investment firms automating deal sourcing across markets. **Customization:** Adjust investment analysis criteria and thresholds. **Benefits:** Identifies investment opportunities automatically. | Sticky note (“Prerequisites”) |
| **Setup Steps:** 1) Connect MLS/public records/demographic APIs. 2) Configure OpenAI GPT‑4 for analysis/scoring. 3) Set up investment calculator integration and ROI parameters. 4) Connect Gmail and Slack; define thresholds. | Sticky note (“Setup Steps”) |
| **How it works:** Daily aggregation → GPT‑4 analysis with ROI/risk → threshold validation → email + Slack notifications to acquisition/investor teams. | Sticky note (“How It Works”) |
| **Alert Acquisition Teams:** Sends alerts via email and Slack with summaries; enables rapid response. | Sticky note (“Alert Acquisition Teams”) |
| **Evaluate Financial Metrics:** Calculates cap rates / cash-on-cash / thresholds for quantitative screening. | Sticky note (“Evaluate Financial Metrics”) |
| **Aggregate Market Data:** Fetches MLS, public records, demographics, macroeconomic indicators to provide market context. | Sticky note (“Aggregate Market Data”) |