Monitor backlinks and generate SEO insights with SE Ranking and GPT-4.1-mini

https://n8nworkflows.xyz/workflows/monitor-backlinks-and-generate-seo-insights-with-se-ranking-and-gpt-4-1-mini-12529


# Monitor backlinks and generate SEO insights with SE Ranking and GPT-4.1-mini

## 1. Workflow Overview

**Purpose:**  
This workflow monitors backlinks for a given target (domain/host/URL) using **SE Ranking Backlinks API** and generates **SEO insights** via an **AI Agent powered by OpenAI GPT‑4.1‑mini**. It then persists the AI-generated summary into storage destinations (n8n DataTable, Google Sheets) and exports it as files (JSON → CSV).

**Target use cases (from workflow notes):**
- Ongoing backlink health monitoring  
- SEO audits and reporting  
- Link quality and risk assessment  
- Agency client reporting  
- Competitive backlink analysis  

### 1.1 Trigger & Input Preparation
The workflow starts manually and sets a single input field (`query`) that instructs the AI what to analyze.

### 1.2 AI Orchestration (SE Ranking + GPT)
An **AI Agent** receives the query, uses GPT‑4.1‑mini as its language model, and can call SE Ranking API tools (summary, metrics, all backlinks). The Agent produces a structured natural-language output (`$json.output`).

### 1.3 Storage & Export
The AI output is persisted to:
- **n8n DataTable** (for internal logging / retrieval)
- **Google Sheets** (append/update by query)
And is also transformed into downloadable files:
- Extract summary → convert to binary JSON → convert to CSV

---

## 2. Block-by-Block Analysis

### Block 2.1 — Trigger & Input Reception

**Overview:**  
Starts execution manually and defines the analysis request as a single `query` string passed to the AI Agent.

**Nodes involved:**
- **When clicking ‘Execute workflow’** (Manual Trigger)
- **Set Input Fields** (Set)

#### Node: When clicking ‘Execute workflow’
- **Type / role:** `n8n-nodes-base.manualTrigger` — manual entry point.
- **Configuration:** No parameters; runs only when manually executed.
- **Outputs:** Main output → **Set Input Fields**.
- **Failure modes / edge cases:** None specific (except workflow not active/scheduled).

#### Node: Set Input Fields
- **Type / role:** `n8n-nodes-base.set` — constructs the request payload.
- **Key configuration:**
  - Creates field:
    - `query` (string) = `Backlinks Summary for https://www.aezion.com/`
- **Outputs:** Main output → **AI Agent**
- **Failure modes / edge cases:**
  - If `query` is empty/invalid, the AI Agent may call tools with missing target, causing SE Ranking API errors.

**Sticky note context covering this area:**
- **How It Works / Setup / Customize** (see Section 3 tables for mapping)

---

### Block 2.2 — AI Intelligence Layer (Agent + Model + SE Ranking Tools)

**Overview:**  
The AI Agent interprets the user’s request and can call SE Ranking Backlinks API endpoints as “tools”, then returns an AI-generated summary.

**Nodes involved:**
- **AI Agent**
- **OpenAI Chat Model**
- **HTTP Request for Backlink Summary**
- **HTTP Request for Backlink Metrics**
- **HTTP Request for All Backlinks**

#### Node: OpenAI Chat Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — provides the LLM used by the agent.
- **Configuration choices:**
  - Model: **`gpt-4.1-mini`**
  - Uses OpenAI credentials: **“OpenAi account”**
- **Connections:**
  - Output (AI language model channel) → **AI Agent**
- **Failure modes / edge cases:**
  - Invalid/expired OpenAI API key
  - Model not available in the account/region
  - Rate limits / timeouts

#### Node: AI Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — orchestrates reasoning and tool usage.
- **Prompting configuration:**
  - Prompt text:
    - `{{ $json.query }}`
    - Plus instruction: **“Do not include your own thoughts or suggestions”** (intended to suppress chain-of-thought and extra advice)
  - `promptType: define`
- **Tooling configuration:**
  - Has access to the three SE Ranking HTTP Request Tool nodes (connected via `ai_tool`).
- **Model configuration:**
  - Uses **OpenAI Chat Model** via `ai_languageModel` connection.
- **Outputs:**
  - Main output → **Persist on DataTable**, **Append or update row in sheet**, **Extract Summary** (three parallel branches)
- **Retry behavior:**
  - `retryOnFail: true` (useful for transient API failures)
- **Failure modes / edge cases:**
  - If the agent cannot extract a valid target from `query`, tool calls may be executed with empty `target`.
  - Output field expectations: downstream nodes assume `output` exists in the Agent result (`$json.output`). If the agent returns a different structure, storage/export nodes may write blanks or error.
  - Tool-call failures (401/403/429/5xx) can propagate or lead to incomplete summaries depending on agent behavior.

#### Node: HTTP Request for Backlink Summary
- **Type / role:** `n8n-nodes-base.httpRequestTool` — tool endpoint for backlink summary stats.
- **Endpoint:** `GET https://api.seranking.com/v1/backlinks/summary`
- **Auth:** HTTP Header Auth credential **“SE Ranking”**
- **Query parameters (key choices):**
  - `target` = `{{ $fromAI('parameters0_Value', '', 'string') }}`
  - `mode` = `host`
  - `output` = `json`
- **Tool description (intent):**
  - Returns aggregated stats (backlinks, referring domains, anchors, TLDs, etc.). Supports batching (per description).
- **Connections:** Tool output → **AI Agent**
- **Failure modes / edge cases:**
  - Missing/invalid header token → 401/403
  - Invalid `target` format
  - Endpoint-specific quotas/rate limits
  - Because this is an *AI tool node*, the `target` is expected to be supplied by the agent via `$fromAI(...)`. If the agent fails to provide it, requests may be malformed.

#### Node: HTTP Request for Backlink Metrics
- **Type / role:** `n8n-nodes-base.httpRequestTool` — tool endpoint for backlink metrics.
- **Endpoint:** `GET https://api.seranking.com/v1/backlinks/metrics`
- **Auth:** HTTP Header Auth **“SE Ranking”**
- **Query parameters:**
  - `target` = `{{ $fromAI('parameters0_Value', '', 'string') }}`
  - `mode` = `domain`
  - `output` = `json`
- **Connections:** Tool output → **AI Agent**
- **Failure modes / edge cases:** Same as above; additionally mismatch between requested scope (domain) and what the user wrote in `query` could affect relevance.

#### Node: HTTP Request for All Backlinks
- **Type / role:** `n8n-nodes-base.httpRequestTool` — tool endpoint to fetch backlink list/details.
- **Endpoint:** `GET https://api.seranking.com/v1/backlinks/all`
- **Auth:** HTTP Header Auth **“SE Ranking”**
- **Query parameters:**
  - `target` = `{{ $fromAI('parameters0_Value', '', 'string') }}`
  - `mode` = `url`
  - `output` = `json`
- **Notes from configuration:** Tool description states **no batching** and accepts a single target.
- **Connections:** Tool output → **AI Agent**
- **Failure modes / edge cases:**
  - Potentially large responses (performance/memory considerations)
  - Pagination not shown here; if SE Ranking paginates, the agent/tool may only retrieve first page unless additional parameters are added.

**Sticky note context covering this block:**
- “SE Ranking Backlinks API”
- “SE Ranking AI Agent”
- “Common Use cases”

---

### Block 2.3 — Persistence (DataTable + Google Sheets)

**Overview:**  
Stores the AI Agent’s final summary output into internal n8n storage and into a Google Sheet for reporting/history.

**Nodes involved:**
- **Persist on DataTable**
- **Append or update row in sheet**

#### Node: Persist on DataTable
- **Type / role:** `n8n-nodes-base.dataTable` — persists records into an n8n DataTable.
- **Operation:** (implied by node) insert/upsert based on matching columns.
- **Mapping:**
  - `Query` = `{{ $('Set Input Fields').item.json.query }}`
  - `Summary` = `{{ $json.output }}`
- **Matching columns:** `Summary`  
  (This means rows are matched/updated based on the *summary text*, which can be unstable; typically you’d match on Query or Timestamp.)
- **Target table:** DataTable “Backlinks” (id cached in node config)
- **Connections:** Receives from **AI Agent**; no further outputs.
- **Failure modes / edge cases:**
  - If `$json.output` is large, DataTable field limits may be hit (depends on n8n/project limits).
  - Matching on `Summary` can cause duplicates or unintended overwrites.

#### Node: Append or update row in sheet
- **Type / role:** `n8n-nodes-base.googleSheets` — writes results to Google Sheets.
- **Operation:** `appendOrUpdate`
- **Document:** “SE Ranking Backlinks” (Spreadsheet ID configured)
- **Sheet:** “Sheet1”
- **Column mapping:**
  - `Query` = `{{ $('Set Input Fields').item.json.query }}`
  - `Summary` = `{{ $json.output }}`
- **Matching columns:** `Query` (stable key; good choice)
- **Credentials:** Google Sheets OAuth2 **“Google Sheets account”**
- **Connections:** Receives from **AI Agent**; no further outputs.
- **Failure modes / edge cases:**
  - OAuth token expired / insufficient permissions
  - Sheet structure changed (columns renamed/removed)
  - Large summary text may exceed Google Sheets cell limits (~50k chars)

**Sticky note context covering this block:**
- “Export Data Handling” (covers export/persistence area)

---

### Block 2.4 — Export to Files (Summary → JSON Binary → CSV)

**Overview:**  
Builds a lightweight export object with summary + timestamp, converts it into a binary JSON file, then converts to CSV.

**Nodes involved:**
- **Extract Summary**
- **Create a Binary Data**
- **Save as CSV**

#### Node: Extract Summary
- **Type / role:** `n8n-nodes-base.code` — reshapes output for export.
- **Logic (interpreted):**
  - Creates one item:
    - `summary` = `$input.first().json.output`
    - `timestamp` = current ISO datetime
- **Connections:**
  - Input from **AI Agent**
  - Output → **Create a Binary Data**
- **Failure modes / edge cases:**
  - If the AI Agent output doesn’t contain `output`, then `summary` becomes `undefined`.

#### Node: Create a Binary Data
- **Type / role:** `n8n-nodes-base.function` — converts JSON to a binary attachment.
- **Logic (interpreted):**
  - Serializes `items[0].json` to pretty JSON string
  - Creates `items[0].binary.data` with:
    - `mimeType: application/json`
    - `fileName: backlinks.json`
- **Connections:** Output → **Save as CSV**
- **Failure modes / edge cases:**
  - If no items exist (empty input), `items[0]` will throw.
  - Large JSON can increase memory usage.

#### Node: Save as CSV
- **Type / role:** `n8n-nodes-base.convertToFile` — converts input data to a file (CSV).
- **Configuration:** Default options (no custom settings shown).
- **Connections:** Receives from **Create a Binary Data**; no further outputs.
- **Failure modes / edge cases:**
  - If input is binary JSON rather than structured tabular JSON, conversion may not yield the expected CSV format. (Often, Convert to File expects JSON objects/arrays; your current flow builds a JSON file binary, then tries to “convert to file” again.)

---

## 3. Summary Table (All Nodes)

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | Manual Trigger | Manual start entry point | — | Set Input Fields | ## **How It Works** … Setup Instructions … Customize … (full note content in workflow) |
| Set Input Fields | Set | Defines `query` for the AI | When clicking ‘Execute workflow’ | AI Agent | ## **How It Works** … Setup Instructions … Customize … (full note content in workflow) |
| AI Agent | LangChain Agent | Orchestrates GPT + tool calls; produces final summary | Set Input Fields; OpenAI Chat Model (ai_languageModel); SE Ranking tools (ai_tool) | Persist on DataTable; Append or update row in sheet; Extract Summary | ## SE Ranking AI Agent … “acts as the intelligence layer … Using GPT-4.1-mini” |
| OpenAI Chat Model | OpenAI Chat Model (LangChain) | LLM backing the agent (gpt-4.1-mini) | — | AI Agent (ai_languageModel) | ## SE Ranking AI Agent … |
| HTTP Request for Backlink Summary | HTTP Request Tool | SE Ranking `/backlinks/summary` tool | — (tool invoked by agent) | AI Agent (ai_tool) | ## SE Ranking Backlinks API … (aggregated metrics, monthly refresh) |
| HTTP Request for Backlink Metrics | HTTP Request Tool | SE Ranking `/backlinks/metrics` tool | — (tool invoked by agent) | AI Agent (ai_tool) | ## SE Ranking Backlinks API … |
| HTTP Request for All Backlinks | HTTP Request Tool | SE Ranking `/backlinks/all` tool | — (tool invoked by agent) | AI Agent (ai_tool) | ## SE Ranking Backlinks API … |
| Persist on DataTable | DataTable | Store Query + AI Summary in n8n DataTable | AI Agent | — | ## Export Data Handling … converts summaries into DataTables/Sheets/CSV/JSON |
| Append or update row in sheet | Google Sheets | Append/update Query + Summary in Google Sheet | AI Agent | — | ## Export Data Handling … converts summaries into DataTables/Sheets/CSV/JSON |
| Extract Summary | Code | Build export object (summary + timestamp) | AI Agent | Create a Binary Data | ## Export Data Handling … converts summaries into DataTables/Sheets/CSV/JSON |
| Create a Binary Data | Function | Serialize JSON and attach as `backlinks.json` binary | Extract Summary | Save as CSV | ## Export Data Handling … converts summaries into DataTables/Sheets/CSV/JSON |
| Save as CSV | Convert to File | Convert data to CSV file | Create a Binary Data | — | ## Export Data Handling … converts summaries into DataTables/Sheets/CSV/JSON |
| Sticky Note | Sticky Note | Documentation: SE Ranking Backlinks API | — | — | ## SE Ranking Backlinks API … |
| Sticky Note1 | Sticky Note | Documentation: how it works/setup/customize | — | — | (Contains setup steps and customization ideas) |
| Sticky Note2 | Sticky Note | Documentation: export data handling | — | — | ## Export Data Handling … |
| Sticky Note3 | Sticky Note | Documentation: common use cases | — | — | ## Common Use cases … |
| Sticky Note4 | Sticky Note | Documentation: SE Ranking AI Agent | — | — | ## SE Ranking AI Agent … |
| Sticky Note5 | Sticky Note | Branding logo | — | — | ![Logo](https://s3-eu-west-1.amazonaws.com/tpd/logos/538f1575000064000578dee6/0x0.png) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add Manual Trigger**
   - Node: **Manual Trigger**
   - Name: `When clicking ‘Execute workflow’`

3. **Add Set node for input**
   - Node: **Set**
   - Name: `Set Input Fields`
   - Add a field:
     - `query` (String) = e.g. `Backlinks Summary for https://www.aezion.com/`
   - Connect: **Manual Trigger → Set Input Fields**

4. **Add OpenAI Chat Model (LangChain)**
   - Node: **OpenAI Chat Model** (LangChain)
   - Name: `OpenAI Chat Model`
   - Model: `gpt-4.1-mini`
   - Credentials:
     - Configure **OpenAI API** credential (API key) and select it in the node.

5. **Add SE Ranking HTTP Header credential**
   - Create credential: **HTTP Header Auth**
   - Name: `SE Ranking`
   - Add the required header (per SE Ranking API docs), commonly something like `Authorization: Bearer <token>` or the vendor-specified header key.

6. **Add 3 HTTP Request Tool nodes (SE Ranking endpoints)**
   - Node 1: **HTTP Request Tool**
     - Name: `HTTP Request for Backlink Summary`
     - Method/URL: `https://api.seranking.com/v1/backlinks/summary`
     - Auth: Generic Credential → HTTP Header Auth → `SE Ranking`
     - Send Query Parameters:
       - `target` = `{{$fromAI('parameters0_Value', '', 'string')}}`
       - `mode` = `host`
       - `output` = `json`
   - Node 2: **HTTP Request Tool**
     - Name: `HTTP Request for Backlink Metrics`
     - URL: `https://api.seranking.com/v1/backlinks/metrics`
     - Query:
       - `target` = `{{$fromAI('parameters0_Value', '', 'string')}}`
       - `mode` = `domain`
       - `output` = `json`
   - Node 3: **HTTP Request Tool**
     - Name: `HTTP Request for All Backlinks`
     - URL: `https://api.seranking.com/v1/backlinks/all`
     - Query:
       - `target` = `{{$fromAI('parameters0_Value', '', 'string')}}`
       - `mode` = `url`
       - `output` = `json`

7. **Add AI Agent (LangChain Agent)**
   - Node: **AI Agent**
   - Name: `AI Agent`
   - Prompt / Text:
     - `{{$json.query}}`
     - Add instruction line: `Do not include your own thoughts or suggestions`
   - Enable retry on fail (optional but matches original): **Retry On Fail = true**
   - Connect:
     - **Set Input Fields → AI Agent**
     - **OpenAI Chat Model → AI Agent** (connection type: *ai_languageModel*)
     - Each SE Ranking HTTP Request Tool → **AI Agent** (connection type: *ai_tool*)

8. **Add persistence: DataTable**
   - Node: **Data Table**
   - Name: `Persist on DataTable`
   - Create/select a DataTable named e.g. **Backlinks**
   - Map columns:
     - `Query` = `{{ $('Set Input Fields').item.json.query }}`
     - `Summary` = `{{ $json.output }}`
   - Matching columns: (as in workflow) `Summary`  
     - (Recommended adjustment: match on `Query` instead for stability.)
   - Connect: **AI Agent → Persist on DataTable**

9. **Add persistence: Google Sheets**
   - Node: **Google Sheets**
   - Name: `Append or update row in sheet`
   - Credentials: configure **Google Sheets OAuth2**
   - Operation: **Append or Update**
   - Select Document + Sheet (create a sheet with headers `Query`, `Summary`)
   - Mapping:
     - `Query` = `{{ $('Set Input Fields').item.json.query }}`
     - `Summary` = `{{ $json.output }}`
   - Matching column: `Query`
   - Connect: **AI Agent → Append or update row in sheet**

10. **Add export reshaping**
   - Node: **Code**
   - Name: `Extract Summary`
   - Code:
     - Create JSON with `summary: $input.first().json.output` and `timestamp: new Date().toISOString()`
   - Connect: **AI Agent → Extract Summary**

11. **Add binary JSON creation**
   - Node: **Function**
   - Name: `Create a Binary Data`
   - Function logic:
     - JSON.stringify first item
     - Write to `binary.data` with filename `backlinks.json` and mime `application/json`
   - Connect: **Extract Summary → Create a Binary Data**

12. **Add CSV export**
   - Node: **Convert to File**
   - Name: `Save as CSV`
   - Operation: convert incoming data to CSV (default options)
   - Connect: **Create a Binary Data → Save as CSV**

13. **Run**
   - Click **Execute workflow**
   - Verify:
     - Agent output contains `output`
     - DataTable row is created/updated
     - Google Sheet row is appended/updated
     - Export nodes produce expected files

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| SE Ranking Backlinks API retrieves aggregated metrics (backlinks, referring domains, IPs/subnets, anchors, TLDs, etc.). Data refreshed monthly. | Sticky note: “SE Ranking Backlinks API” |
| Workflow setup steps: configure SE Ranking HTTP Header Auth, OpenAI credentials (GPT‑4.1‑mini), optional Google Sheets; set target in “Set Input Fields”; execute. | Sticky note: “How It Works / Setup Instructions / Customize” |
| Export section stores AI summaries to DataTables, Google Sheets, CSV, JSON. | Sticky note: “Export Data Handling” |
| Branding logo | https://s3-eu-west-1.amazonaws.com/tpd/logos/538f1575000064000578dee6/0x0.png |

Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.