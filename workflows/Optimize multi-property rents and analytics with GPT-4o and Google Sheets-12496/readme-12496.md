Optimize multi-property rents and analytics with GPT-4o and Google Sheets

https://n8nworkflows.xyz/workflows/optimize-multi-property-rents-and-analytics-with-gpt-4o-and-google-sheets-12496


# Optimize multi-property rents and analytics with GPT-4o and Google Sheets

## 1. Workflow Overview

**Purpose:**  
This workflow runs on a schedule to pull multi-source property portfolio data (rent rolls, operating costs, utilities, mortgages, and market stats), orchestrate GPT-4o-based specialist “agent tools” to generate rent optimization decisions and actionable recommendations, compute ROI projections and portfolio summaries, update an external financial dashboard via API, and email a portfolio-level report.

**Target use cases:**
- Ongoing portfolio rent optimization across many properties/units
- Performance monitoring (under/over-performing units)
- Market-context pricing strategy guidance
- Automated reporting to dashboards and stakeholders

### 1.1 Scheduled start & configuration
- Triggers daily at a configured hour and sets all API endpoints, thresholds, and sub-workflow IDs.

### 1.2 Data acquisition & aggregation (property + market)
- Pulls multiple property datasets in parallel (rent rolls, costs, utilities, mortgages), merges them, aggregates them into a single “allProperties” structure, and fetches market data summarized to portfolio-level benchmarks.

### 1.3 AI orchestration for rent optimization
- A main AI agent (GPT-4o) coordinates multiple specialist tools:
  - Market analysis agent tool (GPT-4o)
  - Performance analysis agent tool (GPT-4o)
  - Risk assessment via calling another n8n workflow
  - Recommendation generator agent tool (GPT-4o)
  - Calculator tool (built-in)
- Produces structured output: rent adjustments, performance flags, and recommendations.

### 1.4 Per-unit reporting & ROI projections
- Splits rent adjustments into per-unit items, routes by property type, enriches each with report metadata, calculates 5-year ROI projections, and generates a per-unit financial report object.

### 1.5 Alerts, dashboard update & portfolio email
- Converts AI results into alert flags based on a performance threshold, posts updates to an external dashboard API, aggregates portfolio metrics, and emails a summary.

---

## 2. Block-by-Block Analysis

### Block 2.1 — Scheduled start & workflow configuration
**Overview:** Starts on a schedule and centralizes endpoints/parameters used across the workflow so downstream nodes can reference them consistently.

**Nodes involved:**
- Schedule Trigger
- Workflow Configuration

**Node details**

1) **Schedule Trigger**  
- **Type / role:** `n8n-nodes-base.scheduleTrigger` — entry point, time-based execution.  
- **Config:** Runs at **06:00** (server time) via `triggerAtHour: 6`.  
- **Outputs:** → Workflow Configuration  
- **Version notes:** typeVersion **1.3**.  
- **Failure/edge cases:** timezone expectations; workflow won’t run if n8n scheduler disabled.

2) **Workflow Configuration**  
- **Type / role:** `n8n-nodes-base.set` — defines reusable configuration fields.  
- **Config choices (interpreted):**
  - Defines placeholder URLs:
    - `rentRollsApiUrl`, `operatingCostsApiUrl`, `utilitiesApiUrl`, `mortgageApiUrl`
    - `dashboardApiUrl`, `alertWebhookUrl`
    - `marketDataApiUrl`, `historicalDataApiUrl`
    - `executiveSummaryWebhook`
  - Defines:
    - `performanceThreshold` = **0.85**
    - `riskAssessmentWorkflowId` (placeholder)
  - `includeOtherFields: true` (keeps upstream fields if any)
- **Key expressions used downstream:**
  - `$('Workflow Configuration').first().json.<fieldName>`
- **Outputs:** fans out in parallel to all data fetch nodes + market fetch.  
- **Version notes:** typeVersion **3.4**.  
- **Failure/edge cases:** placeholders must be replaced; missing/invalid URL causes HTTP Request failures; threshold is not actually consumed later (see Block 2.6 bug note).

---

### Block 2.2 — Multi-source property data acquisition & merge
**Overview:** Fetches four independent property datasets, then merges them by position into combined items for downstream analysis.

**Nodes involved:**
- Fetch Rent Rolls
- Fetch Operating Costs
- Fetch Utilities Data
- Fetch Mortgage Schedules
- Merge Property Data
- Aggregate All Properties

**Node details**

1) **Fetch Rent Rolls**  
- **Type / role:** `n8n-nodes-base.httpRequest` — GET JSON data.  
- **Config:** `url = $('Workflow Configuration').first().json.rentRollsApiUrl`, responseFormat JSON.  
- **Outputs:** → Merge Property Data (input 0)  
- **Version notes:** typeVersion **4.3**.  
- **Failure/edge cases:** auth not configured (node has no creds in JSON); non-JSON response; 4xx/5xx; pagination not handled.

2) **Fetch Operating Costs**  
- **Type / role:** HTTP Request — GET JSON.  
- **Config:** URL from `operatingCostsApiUrl`.  
- **Outputs:** → Merge Property Data (input 1)  
- **Failure modes:** same as above.

3) **Fetch Utilities Data**  
- **Type / role:** HTTP Request — GET JSON.  
- **Config:** URL from `utilitiesApiUrl`.  
- **Outputs:** → Merge Property Data (input 2)

4) **Fetch Mortgage Schedules**  
- **Type / role:** HTTP Request — GET JSON.  
- **Config:** URL from `mortgageApiUrl`.  
- **Outputs:** → Merge Property Data (input 3)

5) **Merge Property Data**  
- **Type / role:** `n8n-nodes-base.merge` — combines 4 inputs into a single stream.  
- **Config:**  
  - `mode: combine`  
  - `combineBy: combineByPosition`  
  - `numberInputs: 4`  
- **Implication:** Items are matched strictly by array position (item 0 with item 0, etc.).  
- **Outputs:** → Aggregate All Properties  
- **Version notes:** typeVersion **3.2**.  
- **Edge cases:** if one API returns fewer items or different ordering, mismatched properties will be combined incorrectly. Prefer “merge by key” (propertyId/unitId) if available.

6) **Aggregate All Properties**  
- **Type / role:** `n8n-nodes-base.aggregate` — wraps all items into one object field.  
- **Config:** `aggregateAllItemData` into `allProperties`.  
- **Outputs:** → Rent Optimization Agent and → Enrich with Market Context  
- **Version notes:** typeVersion **1**.  
- **Edge cases:** large portfolios can create very large prompt payloads to the AI agent; may hit token limits or timeouts.

---

### Block 2.3 — Market data acquisition & summarization
**Overview:** Fetches market data and summarizes portfolio-level benchmarks (average/median rent, vacancy, growth) for enriching downstream analysis.

**Nodes involved:**
- Fetch Market Data
- Summarize Market Trends
- Enrich with Market Context

**Node details**

1) **Fetch Market Data**  
- **Type / role:** HTTP Request — GET JSON.  
- **Config:** URL from `marketDataApiUrl`, responseFormat JSON.  
- **Outputs:** → Summarize Market Trends  
- **Edge cases:** missing fields expected by summarizer; non-uniform data types.

2) **Summarize Market Trends**  
- **Type / role:** `n8n-nodes-base.summarize` — produces a single item with aggregated values.  
- **Config:** Output single item; averages for:
  - `averageRent`, `medianRent`, `vacancyRate`, `marketGrowth`
- **Outputs:** → Enrich with Market Context  
- **Version notes:** typeVersion **1.1**.  
- **Edge cases:** if fields are absent or strings, aggregation may yield null/NaN.

3) **Enrich with Market Context**  
- **Type / role:** Set node — adds market benchmark fields to upstream items.  
- **Config:** Adds:
  - `marketAverageRent = $('Summarize Market Trends').first().json.averageRent`
  - `marketMedianRent`, `marketVacancyRate`, `marketGrowthRate`
  - `includeOtherFields: true`  
- **Input:** comes from Aggregate All Properties (and relies on Summarize output via expression).  
- **Outputs:** → Rent Optimization Agent  
- **Version notes:** typeVersion **3.4**.  
- **Edge cases:** if Summarize hasn’t produced data (fetch failed), expressions can resolve to null and weaken AI output.

---

### Block 2.4 — AI orchestration: rent optimization + specialist tools
**Overview:** A main agent (GPT-4o) receives the aggregated property+market context and calls specialized tools (market/performance/recommendation agents, calculator, and a risk sub-workflow). The main agent outputs structured rent adjustments, performance flags, and recommendations.

**Nodes involved:**
- Rent Optimization Agent
- OpenAI Model - Main
- Structured Output - Rent Recommendations
- Calculator Tool
- Market Analysis Agent Tool
- OpenAI Model - Market
- Structured Output - Market Analysis
- Performance Analysis Agent Tool
- OpenAI Model - Performance
- Structured Output - Performance Metrics
- Recommendation Generator Agent Tool
- OpenAI Model - Recommendations
- Structured Output - Recommendations
- Risk Assessment Workflow Tool

**Node details**

1) **Rent Optimization Agent**  
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — orchestrator agent.  
- **Input text:** `{{$json.allProperties}}` (from Aggregate All Properties).  
- **System message:** instructs calling tools in sequence and returning structured output (rent adjustments, performance flags, market insights, recommendations).  
- **Tooling connections:**  
  - Receives **language model** from OpenAI Model - Main  
  - Has AI tools connected:
    - Calculator Tool
    - Market Analysis Agent Tool
    - Performance Analysis Agent Tool
    - Risk Assessment Workflow Tool
    - Recommendation Generator Agent Tool
  - Has **output parser**: Structured Output - Rent Recommendations  
- **Outputs (main):** → Calculate Performance Metrics and → Split Recommendations by Property  
- **Version notes:** typeVersion **3.1**.  
- **Edge cases:**
  - Token limits: `allProperties` may be huge.
  - Tool call failures: if a tool errors, the agent may return incomplete/invalid schema.
  - Schema enforcement depends on output parser success; if model deviates, parsing fails.

2) **OpenAI Model - Main**  
- **Type / role:** `lmChatOpenAi` — provides GPT-4o chat model to the main agent.  
- **Model:** `gpt-4o`  
- **Credentials:** OpenAI account (must be valid).  
- **Outputs:** (AI language model) → Rent Optimization Agent  
- **Version notes:** typeVersion **1.3**.  
- **Failure modes:** invalid API key, rate limits, model unavailability, timeouts.

3) **Structured Output - Rent Recommendations**  
- **Type / role:** structured output parser — enforces JSON schema for the main agent final output.  
- **Schema (manual):**
  - `rentAdjustments[]`: propertyId, unitId, currentRent, recommendedRent, adjustmentPercentage
  - `performanceFlags[]`: propertyId, unitId, status, performanceScore, reason
  - `recommendations[]`: propertyId, recommendation, priority, estimatedImpact
- **Connected as:** `ai_outputParser` → Rent Optimization Agent  
- **Version notes:** typeVersion **1.3**.  
- **Edge cases:** if model returns strings for numbers, missing required fields, or additional nesting, parsing can fail.

4) **Calculator Tool**  
- **Type / role:** `toolCalculator` — tool for arithmetic used by the main agent.  
- **Connected as:** AI tool → Rent Optimization Agent  
- **Failure modes:** minimal; mostly expression/formatting mismatches in tool call arguments.

5) **Market Analysis Agent Tool**  
- **Type / role:** `agentTool` — specialist agent callable by the orchestrator.  
- **Input mapping:** `{{$fromAI('propertyData', 'Property and market data for analysis', 'json')}}`  
- **System message:** analyze trends, positioning, risks/opportunities, pricing strategy; return structured insights.  
- **Connected:** uses OpenAI Model - Market + Structured Output - Market Analysis; exposed as AI tool to main agent.  
- **Edge cases:** if orchestrator doesn’t pass expected JSON; output parser failure.

6) **OpenAI Model - Market**  
- **Type / role:** GPT-4o model for market tool.  
- **Connected:** AI language model → Market Analysis Agent Tool  
- **Failure modes:** OpenAI API issues.

7) **Structured Output - Market Analysis**  
- **Type / role:** structured parser for market tool.  
- **Schema:** `marketInsights[]` with propertyId, marketPosition, competitiveAdvantage, marketRisk, pricingStrategy.  
- **Connected:** output parser → Market Analysis Agent Tool

8) **Performance Analysis Agent Tool**  
- **Type / role:** `agentTool` — performance metrics specialist.  
- **Input mapping:** `{{$fromAI('propertyData', 'Property data to analyze including rent rolls, costs, utilities, and mortgage info', 'json')}}`  
- **System message:** compute NOI, cash flow, occupancy, cost ratios; classify units; score 0–1.  
- **Connected:** uses OpenAI Model - Performance + Structured Output - Performance Metrics; exposed to main agent.  
- **Edge cases:** unit-level fields absent (occupancy, expenses, debt service), leading to hallucinated metrics unless you enforce numeric inputs.

9) **OpenAI Model - Performance**  
- **Type / role:** GPT-4o model for performance tool.  
- **Connected:** AI language model → Performance Analysis Agent Tool

10) **Structured Output - Performance Metrics**  
- **Type / role:** structured parser for performance tool.  
- **Schema:** `performanceMetrics[]` with propertyId, unitId, noi, cashFlow, occupancyRate, costRatio, performanceScore, classification.

11) **Recommendation Generator Agent Tool**  
- **Type / role:** `agentTool` — recommendations specialist.  
- **Input mapping:** `{{$fromAI('analysisData', 'Performance analysis and rent adjustment data', 'json')}}`  
- **System message:** produce prioritized actions and estimate impact.  
- **Connected:** uses OpenAI Model - Recommendations + Structured Output - Recommendations; exposed to main agent.  
- **Edge cases:** estimates may be non-numeric strings; ensure downstream consumers tolerate that.

12) **OpenAI Model - Recommendations**  
- **Type / role:** GPT-4o model for recommendation tool.

13) **Structured Output - Recommendations**  
- **Type / role:** structured parser for recommendation tool.  
- **Config note:** schema is not specified in the node parameters (empty). This means parsing may be ineffective or rely on defaults—high risk of unstructured outputs.

14) **Risk Assessment Workflow Tool**  
- **Type / role:** `toolWorkflow` — calls another n8n workflow as a tool for risk evaluation.  
- **Config:** workflowId is taken from configuration:  
  - `={{ $('Workflow Configuration').first().json.riskAssessmentWorkflowId }}`  
- **Connected as:** AI tool → Rent Optimization Agent  
- **Version notes:** typeVersion **2.2**.  
- **Edge cases/failures:**
  - Missing/invalid workflow ID
  - Called workflow must be configured to accept the tool input and return tool-friendly JSON
  - Permission issues if workflows are restricted
  - Latency/timeouts when nested executions are slow

---

### Block 2.5 — Per-unit routing, ROI projections, and report generation
**Overview:** Takes the AI rent adjustments, splits them into per-unit records, routes by property type, annotates report metadata, computes ROI projections, and generates a per-unit report object.

**Nodes involved:**
- Split Recommendations by Property
- Route by Property Type
- Format Residential Report
- Format Commercial Report
- Format Mixed-Use Report
- Calculate ROI Projections
- Generate Financial Reports

**Node details**

1) **Split Recommendations by Property**  
- **Type / role:** `n8n-nodes-base.splitOut` — converts `rentAdjustments[]` into separate items.  
- **Config:** `fieldToSplitOut: rentAdjustments`  
- **Input:** from Rent Optimization Agent main output.  
- **Outputs:** → Route by Property Type  
- **Edge cases:** if `rentAdjustments` missing or not array, yields zero items.

2) **Route by Property Type**  
- **Type / role:** `n8n-nodes-base.switch` — routes items into 3 named outputs.  
- **Rules:** checks `$json.propertyType` equals:
  - `residential` → “Residential”
  - `commercial` → “Commercial”
  - `mixed-use` → “Mixed-Use”
- **Outputs:**  
  - Residential → Format Residential Report  
  - Commercial → Format Commercial Report  
  - Mixed-Use → Format Mixed-Use Report  
- **Version notes:** typeVersion **3.4**.  
- **Edge cases / functional issue:** the `rentAdjustments` schema produced by the main agent does **not** include `propertyType`. Unless the main agent adds it (not required by schema) or upstream enrichment adds it, routing will drop items (no matching rule). Consider adding `propertyType` to the rentAdjustments schema or enrich items before routing.

3) **Format Residential Report / Format Commercial Report / Format Mixed-Use Report**  
- **Type / role:** Set nodes — add report metadata.  
- **Config:** adds:
  - `reportType` (different per branch)
  - `propertyCategory` (Residential/Commercial/Mixed-Use)
  - `analysisDate = {{$now.toISO()}}`
  - `includeOtherFields: true`
- **Outputs:** each → Calculate ROI Projections  
- **Version notes:** typeVersion **3.4**.

4) **Calculate ROI Projections**  
- **Type / role:** Code node — computes 5-year revenue projection and ROI %.  
- **Mode:** `runOnceForEachItem`  
- **Logic:**
  - Uses `currentRent`, `recommendedRent`, `adjustmentPercentage`
  - Projects 5 years with 3% annual growth after year 1
  - Computes `additionalRevenue` and `roi = additionalRevenue / currentRevenue5Year * 100`
- **Outputs:** → Generate Financial Reports  
- **Edge cases:**
  - Missing rents default to 0 → division by zero risk (`currentRevenue5Year` can be 0) resulting in Infinity/NaN.
  - Assumes monthly rent; if inputs are weekly/annual, results wrong.

5) **Generate Financial Reports**  
- **Type / role:** Code node — builds a report object for each unit.  
- **Mode:** `runOnceForEachItem`  
- **Output structure:** propertyId/unitId, reportType/category/date, current metrics, recommendations, ROI projection, market context, and a summary string.  
- **Outputs:** → Calculate Performance Metrics AND → Aggregate Portfolio Metrics  
- **Edge cases:**
  - Fields like `noi`, `occupancyRate`, `marketAverageRent` might not exist on the split item unless merged/enriched earlier.
  - Summary uses `data.roiProjection?.roi?.toFixed(2)`; if roi is undefined/NaN, this can throw (in JS, calling toFixed on non-number throws). In the provided code it uses optional chaining on `roi` but not type checking; if `roi` is a string, `toFixed` will throw.

---

### Block 2.6 — Performance thresholding, dashboard update, portfolio aggregation, and email
**Overview:** Converts AI performance flags into alertable records, posts to a dashboard API, aggregates portfolio-level metrics, and emails a summary.

**Nodes involved:**
- Calculate Performance Metrics
- Format Dashboard Update
- Update Financial Dashboard
- Aggregate Portfolio Metrics
- Send a message

**Node details**

1) **Calculate Performance Metrics**  
- **Type / role:** Code node — transforms main AI output performance flags into per-flag items and attaches alerts + full recommendation context.  
- **Logic highlights:**
  - `agentOutput = $input.first().json`
  - `threshold = 0.85` (hard-coded)
  - For each `performanceFlags[]`, sets `hasAlert = performanceScore < threshold`
  - Adds `rentAdjustments` and `recommendations` arrays and `timestamp`
- **Inputs:** from Rent Optimization Agent **and** from Generate Financial Reports (two incoming connections).  
  - **Important:** This node uses `$input.first()` and expects `performanceFlags` to exist. If the incoming item is a financial report (from Generate Financial Reports), `performanceFlags` won’t exist → it will return `[]` (no output).  
- **Outputs:** → Format Dashboard Update  
- **Version notes:** typeVersion **2**.  
- **Edge cases / bugs:**
  - Threshold ignores `Workflow Configuration.performanceThreshold`. If you change config, it won’t affect alerts unless you modify the code.
  - Dual inputs are not merged; execution semantics may be confusing. If you intended to combine AI output with generated reports, a Merge node is needed.

2) **Format Dashboard Update**  
- **Type / role:** Set node — shapes fields for dashboard API.  
- **Config:** sends:
  - propertyId, unitId, performanceScore, status
  - rentAdjustments (object), recommendations (array)
  - lastUpdated = timestamp
- **Outputs:** → Update Financial Dashboard  
- **Edge cases:** `rentAdjustments` is an array in earlier steps; here it’s stored as an “object” assignment. If the dashboard expects an array, keep type consistent.

3) **Update Financial Dashboard**  
- **Type / role:** HTTP Request — POST update payload to external dashboard API.  
- **Config:**  
  - `url = $('Workflow Configuration').first().json.dashboardApiUrl`
  - method POST
  - JSON body = `{{$json}}`
- **Outputs:** none downstream.  
- **Failure modes:** auth missing; 4xx/5xx; schema mismatch; retries not configured.

4) **Aggregate Portfolio Metrics**  
- **Type / role:** Summarize node — produces single-item portfolio metrics for emailing.  
- **Config:** output single item; summarizes:
  - average of `roiProjection.roi`
  - sum of `roiProjection.additionalRevenue`
  - includes `priority` (aggregation unspecified/odd; may produce unexpected output)
- **Input:** from Generate Financial Reports.  
- **Outputs:** → Send a message  
- **Edge cases:** if ROI is NaN/Infinity from earlier division, average becomes invalid.

5) **Send a message**  
- **Type / role:** Gmail node — sends email with aggregated metrics.  
- **Config:** not specified (no subject/body/recipients in parameters).  
- **Credentials:** Gmail OAuth2 “Gmail account 3”.  
- **Input:** from Aggregate Portfolio Metrics.  
- **Version notes:** typeVersion **2.2**.  
- **Edge cases:** incomplete node configuration will cause runtime errors (missing recipients, content); Gmail scopes/consent; quota limits.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | scheduleTrigger | Scheduled workflow entry point | — | Workflow Configuration |  |
| Workflow Configuration | set | Centralized config (API URLs, thresholds, workflow IDs) | Schedule Trigger | Fetch Rent Rolls; Fetch Operating Costs; Fetch Utilities Data; Fetch Mortgage Schedules; Fetch Market Data | ## Setup Steps<br>1. Configure real estate API credentials (Zillow/Realtor.com) <br>2. Add market data API keys for local statistics and demographics<br>3. Input NVIDIA API keys for all OpenAI Model nodes  <br>4. Set OpenAI API key in Team Collaboration Agent/Orchestrator<br>5. Configure Calculator Tool parameters for financial projections<br>6. Connect Google Sheets and specify portfolio tracking spreadsheet ID<br>7. Set up Gmail credentials and specify recipient addresses for reports |
| Fetch Rent Rolls | httpRequest | Pull rent roll dataset | Workflow Configuration | Merge Property Data | ## Multi-Source Property Data Aggregation<br>**Why:** Comprehensive data collection from MLS listings, market databases, and local statistics ensures analysis is based on complete, current information rather than fragmented sources. |
| Fetch Operating Costs | httpRequest | Pull operating cost dataset | Workflow Configuration | Merge Property Data | ## Multi-Source Property Data Aggregation<br>**Why:** Comprehensive data collection from MLS listings, market databases, and local statistics ensures analysis is based on complete, current information rather than fragmented sources. |
| Fetch Utilities Data | httpRequest | Pull utilities dataset | Workflow Configuration | Merge Property Data | ## Multi-Source Property Data Aggregation<br>**Why:** Comprehensive data collection from MLS listings, market databases, and local statistics ensures analysis is based on complete, current information rather than fragmented sources. |
| Fetch Mortgage Schedules | httpRequest | Pull mortgage schedule dataset | Workflow Configuration | Merge Property Data | ## Multi-Source Property Data Aggregation<br>**Why:** Comprehensive data collection from MLS listings, market databases, and local statistics ensures analysis is based on complete, current information rather than fragmented sources. |
| Merge Property Data | merge | Combine 4 datasets into aligned records | Fetch Rent Rolls; Fetch Operating Costs; Fetch Utilities Data; Fetch Mortgage Schedules | Aggregate All Properties | ## Multi-Source Property Data Aggregation<br>**Why:** Comprehensive data collection from MLS listings, market databases, and local statistics ensures analysis is based on complete, current information rather than fragmented sources. |
| Aggregate All Properties | aggregate | Aggregate all items into `allProperties` | Merge Property Data | Rent Optimization Agent; Enrich with Market Context | ## Multi-Source Property Data Aggregation<br>**Why:** Comprehensive data collection from MLS listings, market databases, and local statistics ensures analysis is based on complete, current information rather than fragmented sources. |
| Fetch Market Data | httpRequest | Pull market benchmark dataset | Workflow Configuration | Summarize Market Trends | ## Market Sentiment & Trend Assessment<br>**Why:** Natural language processing of reviews, news, and demographic data reveals qualitative insights that traditional metrics miss, predicting neighborhood trajectories. |
| Summarize Market Trends | summarize | Create portfolio-level market benchmarks | Fetch Market Data | Enrich with Market Context | ## Market Sentiment & Trend Assessment<br>**Why:** Natural language processing of reviews, news, and demographic data reveals qualitative insights that traditional metrics miss, predicting neighborhood trajectories. |
| Enrich with Market Context | set | Add market averages/benchmarks to property data | Aggregate All Properties; Summarize Market Trends (via expression) | Rent Optimization Agent | ## Market Sentiment & Trend Assessment<br>**Why:** Natural language processing of reviews, news, and demographic data reveals qualitative insights that traditional metrics miss, predicting neighborhood trajectories. |
| Rent Optimization Agent | langchain.agent | Orchestrate specialist tools and produce structured rent/performance outputs | Aggregate All Properties; Enrich with Market Context | Calculate Performance Metrics; Split Recommendations by Property | ## AI-Powered Performance Analysis<br>**Why:** Automated evaluation of cap rates, cash-on-cash returns, and appreciation potential identifies opportunities faster than manual spreadsheet analysis across hundreds of properties. |
| OpenAI Model - Main | lmChatOpenAi | LLM backend for main orchestrator | — | Rent Optimization Agent (ai_languageModel) | ## AI-Powered Performance Analysis<br>**Why:** Automated evaluation of cap rates, cash-on-cash returns, and appreciation potential identifies opportunities faster than manual spreadsheet analysis across hundreds of properties. |
| Structured Output - Rent Recommendations | outputParserStructured | Enforce schema for orchestrator final output | — | Rent Optimization Agent (ai_outputParser) | ## Intelligent Property Recommendations<br>**Why:** Machine learning ranks properties by alignment with investment criteria, filtering noise and surfacing optimal opportunities based on risk tolerance and goals. |
| Calculator Tool | toolCalculator | Arithmetic tool callable by agent | — | Rent Optimization Agent (ai_tool) | ## AI-Powered Performance Analysis<br>**Why:** Automated evaluation of cap rates, cash-on-cash returns, and appreciation potential identifies opportunities faster than manual spreadsheet analysis across hundreds of properties. |
| Market Analysis Agent Tool | agentTool | Tool: market positioning and pricing strategy | — | Rent Optimization Agent (ai_tool) | ## Market Sentiment & Trend Assessment<br>**Why:** Natural language processing of reviews, news, and demographic data reveals qualitative insights that traditional metrics miss, predicting neighborhood trajectories. |
| OpenAI Model - Market | lmChatOpenAi | LLM backend for market tool | — | Market Analysis Agent Tool (ai_languageModel) | ## Market Sentiment & Trend Assessment<br>**Why:** Natural language processing of reviews, news, and demographic data reveals qualitative insights that traditional metrics miss, predicting neighborhood trajectories. |
| Structured Output - Market Analysis | outputParserStructured | Enforce schema for market tool output | — | Market Analysis Agent Tool (ai_outputParser) | ## Market Sentiment & Trend Assessment<br>**Why:** Natural language processing of reviews, news, and demographic data reveals qualitative insights that traditional metrics miss, predicting neighborhood trajectories. |
| Performance Analysis Agent Tool | agentTool | Tool: compute NOI/cashflow/occupancy & scores | — | Rent Optimization Agent (ai_tool) | ## AI-Powered Performance Analysis<br>**Why:** Automated evaluation of cap rates, cash-on-cash returns, and appreciation potential identifies opportunities faster than manual spreadsheet analysis across hundreds of properties. |
| OpenAI Model - Performance | lmChatOpenAi | LLM backend for performance tool | — | Performance Analysis Agent Tool (ai_languageModel) | ## AI-Powered Performance Analysis<br>**Why:** Automated evaluation of cap rates, cash-on-cash returns, and appreciation potential identifies opportunities faster than manual spreadsheet analysis across hundreds of properties. |
| Structured Output - Performance Metrics | outputParserStructured | Enforce schema for performance tool output | — | Performance Analysis Agent Tool (ai_outputParser) | ## AI-Powered Performance Analysis<br>**Why:** Automated evaluation of cap rates, cash-on-cash returns, and appreciation potential identifies opportunities faster than manual spreadsheet analysis across hundreds of properties. |
| Recommendation Generator Agent Tool | agentTool | Tool: generate prioritized actions & impacts | — | Rent Optimization Agent (ai_tool) | ## Intelligent Property Recommendations<br>**Why:** Machine learning ranks properties by alignment with investment criteria, filtering noise and surfacing optimal opportunities based on risk tolerance and goals. |
| OpenAI Model - Recommendations | lmChatOpenAi | LLM backend for recommendations tool | — | Recommendation Generator Agent Tool (ai_languageModel) | ## Intelligent Property Recommendations<br>**Why:** Machine learning ranks properties by alignment with investment criteria, filtering noise and surfacing optimal opportunities based on risk tolerance and goals. |
| Structured Output - Recommendations | outputParserStructured | Enforce schema for recommendations tool output (currently unspecified) | — | Recommendation Generator Agent Tool (ai_outputParser) | ## Intelligent Property Recommendations<br>**Why:** Machine learning ranks properties by alignment with investment criteria, filtering noise and surfacing optimal opportunities based on risk tolerance and goals. |
| Risk Assessment Workflow Tool | toolWorkflow | Tool: call sub-workflow for risk evaluation | — | Rent Optimization Agent (ai_tool) | ## Intelligent Property Recommendations<br>**Why:** Machine learning ranks properties by alignment with investment criteria, filtering noise and surfacing optimal opportunities based on risk tolerance and goals. |
| Split Recommendations by Property | splitOut | Split `rentAdjustments[]` into per-unit items | Rent Optimization Agent | Route by Property Type | ## Intelligent Property Recommendations<br>**Why:** Machine learning ranks properties by alignment with investment criteria, filtering noise and surfacing optimal opportunities based on risk tolerance and goals. |
| Route by Property Type | switch | Branch by residential/commercial/mixed-use | Split Recommendations by Property | Format Residential Report; Format Commercial Report; Format Mixed-Use Report | ## Intelligent Property Recommendations<br>**Why:** Machine learning ranks properties by alignment with investment criteria, filtering noise and surfacing optimal opportunities based on risk tolerance and goals. |
| Format Residential Report | set | Add report metadata (Residential) | Route by Property Type | Calculate ROI Projections |  |
| Format Commercial Report | set | Add report metadata (Commercial) | Route by Property Type | Calculate ROI Projections |  |
| Format Mixed-Use Report | set | Add report metadata (Mixed-Use) | Route by Property Type | Calculate ROI Projections |  |
| Calculate ROI Projections | code | Compute 5-year ROI projection per unit | Format Residential Report; Format Commercial Report; Format Mixed-Use Report | Generate Financial Reports |  |
| Generate Financial Reports | code | Build per-unit report object | Calculate ROI Projections | Calculate Performance Metrics; Aggregate Portfolio Metrics |  |
| Calculate Performance Metrics | code | Create per-flag alert items + attach recommendations | Rent Optimization Agent; Generate Financial Reports | Format Dashboard Update |  |
| Format Dashboard Update | set | Shape payload for dashboard API | Calculate Performance Metrics | Update Financial Dashboard |  |
| Update Financial Dashboard | httpRequest | POST results to dashboard API | Format Dashboard Update | — |  |
| Aggregate Portfolio Metrics | summarize | Portfolio-level rollup for email | Generate Financial Reports | Send a message |  |
| Send a message | gmail | Email portfolio summary | Aggregate Portfolio Metrics | — |  |
| Sticky Note | stickyNote | Project notes (prereqs/use cases/benefits) | — | — | ## Prerequisites<br>NVIDIA API access, OpenAI API key, real estate data API subscriptions <br>## Use Cases<br>Multi-property portfolio analysis, acquisition opportunity screening.<br>## Customization<br>Adjust investment criteria thresholds, add custom financial metrics <br>## Benefits<br>Reduces analysis time by 90%, evaluates unlimited properties simultaneously |
| Sticky Note1 | stickyNote | Setup checklist | — | — | ## Setup Steps<br>1. Configure real estate API credentials (Zillow/Realtor.com) <br>2. Add market data API keys for local statistics and demographics<br>3. Input NVIDIA API keys for all OpenAI Model nodes  <br>4. Set OpenAI API key in Team Collaboration Agent/Orchestrator<br>5. Configure Calculator Tool parameters for financial projections<br>6. Connect Google Sheets and specify portfolio tracking spreadsheet ID<br>7. Set up Gmail credentials and specify recipient addresses for reports |
| Sticky Note2 | stickyNote | High-level “How it works” description | — | — | ## How It Works<br>This workflow automates comprehensive real estate investment analysis by orchestrating specialized AI agents to evaluate property data, market trends, and financial metrics. Designed for real estate investors, portfolio managers, and property analysts managing multiple properties or evaluating acquisition opportunities, it eliminates the manual research and analysis that typically requires days of work across multiple data sources. The system aggregates data from real estate APIs, market databases, and local statistics, then deploys specialized agents: performance analysis evaluates ROI and cash flow, recommendation engines identify optimal properties, market analysis assesses location trends, sentiment analysis mines reviews and local feedback, and workflow tools calculate financial projections. An orchestrator coordinates these agents to generate consolidated investment reports with property rankings, risk assessments, and portfolio recommendations. Results populate Google Sheets dashboards and trigger email notifications, transforming weeks of analysis into automated insights delivered in hours. |
| Sticky Note3 | stickyNote | Rationale for AI performance analysis | — | — | ## AI-Powered Performance Analysis<br>**Why:** Automated evaluation of cap rates, cash-on-cash returns, and appreciation potential identifies opportunities faster than manual spreadsheet analysis across hundreds of properties. |
| Sticky Note4 | stickyNote | Rationale for data aggregation | — | — | ## Multi-Source Property Data Aggregation<br>**Why:** Comprehensive data collection from MLS listings, market databases, and local statistics ensures analysis is based on complete, current information rather than fragmented sources. |
| Sticky Note5 | stickyNote | Rationale for market trend assessment | — | — | ## Market Sentiment & Trend Assessment<br>**Why:** Natural language processing of reviews, news, and demographic data reveals qualitative insights that traditional metrics miss, predicting neighborhood trajectories. |
| Sticky Note6 | stickyNote | Rationale for recommendations | — | — | ## Intelligent Property Recommendations<br>**Why:** Machine learning ranks properties by alignment with investment criteria, filtering noise and surfacing optimal opportunities based on risk tolerance and goals. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create the trigger**
   1. Add **Schedule Trigger**.
   2. Set it to run daily at **06:00** (or your preferred hour).

2) **Add a configuration node**
   1. Add **Set** node named **Workflow Configuration**.
   2. Add fields (strings unless noted):
      - `rentRollsApiUrl`, `operatingCostsApiUrl`, `utilitiesApiUrl`, `mortgageApiUrl`
      - `dashboardApiUrl`, `alertWebhookUrl`, `marketDataApiUrl`, `historicalDataApiUrl`
      - `executiveSummaryWebhook`
      - `riskAssessmentWorkflowId`
      - `performanceThreshold` (number, default **0.85**)
   3. Enable **Include Other Fields**.
   4. Connect: **Schedule Trigger → Workflow Configuration**.

3) **Create parallel HTTP fetches for property data**
   1. Add four **HTTP Request** nodes:
      - **Fetch Rent Rolls** (URL: `={{ $('Workflow Configuration').first().json.rentRollsApiUrl }}`)
      - **Fetch Operating Costs** (URL: `={{ $('Workflow Configuration').first().json.operatingCostsApiUrl }}`)
      - **Fetch Utilities Data** (URL: `={{ $('Workflow Configuration').first().json.utilitiesApiUrl }}`)
      - **Fetch Mortgage Schedules** (URL: `={{ $('Workflow Configuration').first().json.mortgageApiUrl }}`)
   2. Set each to parse **JSON** response.
   3. Connect **Workflow Configuration** to all four nodes.

4) **Merge and aggregate property data**
   1. Add **Merge** node named **Merge Property Data**.
   2. Set:
      - Mode: **Combine**
      - Combine by: **Position**
      - Number of inputs: **4**
   3. Connect each fetch node into inputs 0–3 of the Merge node.
   4. Add **Aggregate** node named **Aggregate All Properties**:
      - Aggregate: **All Item Data**
      - Destination field: `allProperties`
   5. Connect **Merge Property Data → Aggregate All Properties**.

5) **Fetch and summarize market data**
   1. Add **HTTP Request** node **Fetch Market Data**:
      - URL: `={{ $('Workflow Configuration').first().json.marketDataApiUrl }}`
      - Response: JSON
   2. Add **Summarize** node **Summarize Market Trends**:
      - Output format: **Single Item**
      - Add aggregations (average): `averageRent`, `medianRent`, `vacancyRate`, `marketGrowth`
   3. Add **Set** node **Enrich with Market Context**:
      - Include other fields: enabled
      - Add:
        - `marketAverageRent = {{ $('Summarize Market Trends').first().json.averageRent }}`
        - `marketMedianRent = {{ $('Summarize Market Trends').first().json.medianRent }}`
        - `marketVacancyRate = {{ $('Summarize Market Trends').first().json.vacancyRate }}`
        - `marketGrowthRate = {{ $('Summarize Market Trends').first().json.marketGrowth }}`
   4. Connect:
      - **Workflow Configuration → Fetch Market Data → Summarize Market Trends → Enrich with Market Context**
      - Also connect **Aggregate All Properties → Enrich with Market Context** (so it carries the property payload forward).

6) **Build the AI orchestrator (main agent)**
   1. Add **OpenAI Chat Model** node **OpenAI Model - Main**:
      - Model: **gpt-4o**
      - Configure **OpenAI API credential**.
   2. Add **Agent** node **Rent Optimization Agent**:
      - Prompt type: “Define”
      - Text: `={{ $json.allProperties }}`
      - System message: include the orchestration steps (market tool, performance tool, risk tool, recommendations tool, calculator).
   3. Add **Structured Output Parser** node **Structured Output - Rent Recommendations** with the schema for:
      - `rentAdjustments[]`, `performanceFlags[]`, `recommendations[]`
   4. Connect:
      - **OpenAI Model - Main (ai_languageModel) → Rent Optimization Agent**
      - **Structured Output - Rent Recommendations (ai_outputParser) → Rent Optimization Agent**
      - **Enrich with Market Context → Rent Optimization Agent**

7) **Create specialist AI tools**
   - **Market tool**
     1. Add **Agent Tool** named **Market Analysis Agent Tool** with its system message and input mapping using `$fromAI(...)`.
     2. Add **OpenAI Model - Market** (gpt-4o) and connect as the tool’s language model.
     3. Add **Structured Output - Market Analysis** with its `marketInsights[]` schema and connect as tool output parser.
     4. Connect **Market Analysis Agent Tool → Rent Optimization Agent** as an **AI tool** connection.
   - **Performance tool**
     1. Add **Performance Analysis Agent Tool** + **OpenAI Model - Performance** (gpt-4o) + **Structured Output - Performance Metrics** schema.
     2. Connect the tool to the orchestrator as an AI tool.
   - **Recommendations tool**
     1. Add **Recommendation Generator Agent Tool** + **OpenAI Model - Recommendations** (gpt-4o).
     2. Add **Structured Output - Recommendations** and define an explicit schema (recommended—currently missing in the provided workflow).
     3. Connect the tool to the orchestrator as an AI tool.
   - **Calculator**
     1. Add **Calculator Tool** and connect as AI tool to the orchestrator.
   - **Risk sub-workflow tool**
     1. Add **Workflow Tool** node **Risk Assessment Workflow Tool**.
     2. Set Workflow ID: `={{ $('Workflow Configuration').first().json.riskAssessmentWorkflowId }}`
     3. Ensure the called workflow accepts JSON input from the tool and returns structured JSON.
     4. Connect as AI tool to the orchestrator.

8) **Split rent adjustments and route by property type**
   1. Add **Split Out** node **Split Recommendations by Property**:
      - Field to split: `rentAdjustments`
   2. Add **Switch** node **Route by Property Type** with three rules:
      - `$json.propertyType == 'residential'`
      - `$json.propertyType == 'commercial'`
      - `$json.propertyType == 'mixed-use'`
   3. Connect **Rent Optimization Agent → Split Recommendations by Property → Route by Property Type**.
   4. Important: ensure each rent adjustment item actually contains `propertyType` (either add it to the orchestrator schema or enrich items before routing).

9) **Add report formatting + ROI computation**
   1. Add three **Set** nodes:
      - **Format Residential Report** (set reportType/category/date)
      - **Format Commercial Report**
      - **Format Mixed-Use Report**
   2. Connect each Switch output to the matching formatter node.
   3. Add **Code** node **Calculate ROI Projections** (run once per item) and paste ROI logic.
   4. Connect each formatter → Calculate ROI Projections.
   5. Add **Code** node **Generate Financial Reports** (run once per item) and paste report construction code.
   6. Connect **Calculate ROI Projections → Generate Financial Reports**.

10) **Dashboard update pipeline**
   1. Add **Code** node **Calculate Performance Metrics** to iterate `performanceFlags[]` and set `hasAlert`.
      - Recommended fix: use `threshold = $('Workflow Configuration').first().json.performanceThreshold` instead of a hard-coded 0.85.
   2. Add **Set** node **Format Dashboard Update** to build dashboard payload.
   3. Add **HTTP Request** node **Update Financial Dashboard**:
      - POST to `dashboardApiUrl`
      - Body: JSON `{{$json}}`
   4. Connect: **Rent Optimization Agent → Calculate Performance Metrics → Format Dashboard Update → Update Financial Dashboard**.

11) **Portfolio aggregation + Gmail**
   1. Add **Summarize** node **Aggregate Portfolio Metrics**:
      - Output single item
      - Average `roiProjection.roi`
      - Sum `roiProjection.additionalRevenue`
   2. Add **Gmail** node **Send a message**:
      - Configure OAuth2 credentials
      - Set recipients, subject, and body using fields from Aggregate Portfolio Metrics.
   3. Connect **Generate Financial Reports → Aggregate Portfolio Metrics → Send a message**.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Prerequisites: NVIDIA API access, OpenAI API key, real estate data API subscriptions | Sticky note “Prerequisites” |
| Use cases: multi-property portfolio analysis, acquisition opportunity screening | Sticky note “Use Cases” |
| Customization: adjust thresholds, add custom metrics | Sticky note “Customization” |
| Benefits: reduces analysis time by 90%, evaluates unlimited properties simultaneously | Sticky note “Benefits” |
| Setup steps mention Google Sheets integration, but no Google Sheets nodes exist in this workflow JSON | Alignment note: sticky content vs actual implementation |
| Recommendation tool output parser has no schema defined; this can lead to inconsistent tool outputs | Reliability note |
| Property type routing requires `propertyType` on each rent adjustment item; current rent adjustment schema does not guarantee it | Functional dependency note |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.