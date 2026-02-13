Monitor Reddit for sales opportunities with GPT-4o and create Asana tasks

https://n8nworkflows.xyz/workflows/monitor-reddit-for-sales-opportunities-with-gpt-4o-and-create-asana-tasks-12102


# Monitor Reddit for sales opportunities with GPT-4o and create Asana tasks

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow monitors Reddit (indirectly via Google search results) every 2 hours, uses GPT‑4o to analyze each post for sales opportunity signals, and creates corresponding Asana tasks. It also notifies a Google Chat space and emails errors when the workflow fails.

**Primary use cases:**
- Lightweight “social listening” for inbound demand (people asking for tools, recommendations, help).
- Automated lead triage (high/medium/low intent) with task creation for follow-up.
- Team notifications for timely outreach.

### 1.1 Data Collection (scheduled search → results splitting)
- Runs on a 2-hour schedule.
- Queries Google (via SerpApi) for Reddit posts matching buying-intent phrases.
- Splits results into individual items for analysis.

### 1.2 AI Analysis (LLM + structured parsing)
- Sends each post snippet to a LangChain Agent using GPT‑4o.
- Produces structured JSON fields such as `is_opportunity`, `intent_level`, summary, action, confidence.

### 1.3 Filtering & Lead Shaping
- Filters out non-opportunities.
- Extracts and normalizes the fields needed downstream.

### 1.4 Intent Routing → Actions
- Routes leads by intent (high/medium/low).
- Creates Asana tasks and sends Google Chat alerts.

### 1.5 Error Handling
- Any workflow error triggers a separate error workflow branch.
- Sends a Gmail email with the node name and error message.

---

## 2. Block-by-Block Analysis

### Block A — Workflow Notes / Documentation (UI only)

**Overview:**  
Sticky notes document the intent of the workflow and visually annotate major sections. They do not affect execution.

**Nodes involved:**
- Note
- Note1
- Note2
- Note4
- Note5
- Sticky Note7

**Node details**
- **Note** (Sticky Note)
  - **Type/role:** `n8n-nodes-base.stickyNote` — visual description of what the workflow does and required services.
  - **Content highlights:** SerpApi + OpenAI (GPT‑4o) + Asana + Google Chat; reminder to update search query and IDs.
  - **Failure modes:** None (non-executing).

- **Note1** (Sticky Note)
  - **Role:** Labels the “Data Collection” section.
  - **Failure modes:** None.

- **Note2** (Sticky Note)
  - **Role:** Labels “AI Analysis & Filtering”.
  - **Failure modes:** None.

- **Note4** (Sticky Note)
  - **Role:** Labels “Intent-Based Routing”.
  - **Failure modes:** None.

- **Note5** (Sticky Note)
  - **Role:** Labels “Actions & Notifications”.
  - **Failure modes:** None.

- **Sticky Note7** (Sticky Note)
  - **Role:** Labels “Error Handling”.
  - **Failure modes:** None.

---

### Block B — Data Collection (Cron → SerpApi → Split)

**Overview:**  
Every 2 hours, the workflow searches Google for Reddit posts matching predefined buying-intent keywords, then splits the returned results into individual items.

**Nodes involved:**
- Every 2 Hours
- Search Reddit Posts
- Split Search Results
- Loop Over Items

#### Node details

- **Every 2 Hours**
  - **Type/role:** `n8n-nodes-base.cron` — scheduled trigger.
  - **Configuration (interpreted):** Uses default Cron settings (in this JSON, no explicit schedule fields are shown). The node name indicates a 2-hour cadence, but verify the Cron rule in UI.
  - **Outputs:** To **Search Reddit Posts**.
  - **Failure/edge cases:** Cron misconfiguration or timezone mismatches can cause unexpected run times.

- **Search Reddit Posts**
  - **Type/role:** `n8n-nodes-serpapi.serpApi` — Google search via SerpApi.
  - **Key config:**
    - Query `q`:  
      `site:reddit.com ("looking for" OR "need help with" OR "any tool for" OR "recommend software")`
    - `location`: India
  - **Credentials:** SerpApi credential required.
  - **Outputs:** To **Split Search Results**.
  - **Failure/edge cases:**
    - SerpApi quota exceeded / rate limits.
    - Google results variability (missing or renamed fields).
    - Location bias may exclude relevant posts if you want global monitoring.

- **Split Search Results**
  - **Type/role:** `n8n-nodes-base.splitOut` — converts an array into multiple items.
  - **Key config:** Splits the field `organic_results`.
  - **Inputs:** From **Search Reddit Posts** (expects `organic_results` array).
  - **Outputs:** To **Loop Over Items**.
  - **Failure/edge cases:**
    - If `organic_results` is absent or not an array, the node may output 0 items or error (depends on n8n version and settings).

- **Loop Over Items**
  - **Type/role:** `n8n-nodes-base.splitInBatches` — batch/loop controller.
  - **Key config:** Options left default (batch size not explicitly set in JSON; check UI).
  - **Connections:** In this workflow, the **second output** (index 1) goes to **AI Lead Analyzer**.
    - In n8n, Split In Batches typically has:
      - Output 0: current batch
      - Output 1: “No Items Left” (or continuation, depending on version)
    - Here, the wiring suggests a likely misconnection: analysis is connected from output 1, meaning the agent may run only when batches are exhausted (or not at all).
  - **Failure/edge cases:**
    - **High risk configuration issue:** If output ports are misused, items may never be analyzed.
  - **Version note:** Split In Batches behavior and port meanings can vary across versions; validate wiring in your n8n instance.

---

### Block C — AI Analysis (Agent + GPT‑4o + Structured Parser)

**Overview:**  
For each Reddit search result item, the workflow uses GPT‑4o to decide whether it’s a sales opportunity, assign intent level, summarize the problem, recommend an action, and output a confidence score in structured JSON.

**Nodes involved:**
- OpenAI Chat Model
- Structured Output Parser
- AI Lead Analyzer

#### Node details

- **OpenAI Chat Model**
  - **Type/role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — provides an LLM to LangChain nodes.
  - **Key config:** `model = gpt-4o`, `temperature = 0.3`.
  - **Credentials:** OpenAI API credential required.
  - **Connections:** Connected to **AI Lead Analyzer** via `ai_languageModel`.
  - **Failure/edge cases:**
    - Invalid API key / revoked key.
    - Model access restrictions or region/org policy.
    - Token limits if snippets become large (less likely with short snippets).

- **Structured Output Parser**
  - **Type/role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — forces/validates structured JSON output.
  - **Key config:** Not shown in JSON; relies on agent’s “structured JSON format” instruction and parser configuration in node UI.
  - **Connections:** Feeds **AI Lead Analyzer** via `ai_outputParser`.
  - **Failure/edge cases:**
    - Parsing failure if the model returns non-JSON or malformed JSON.
    - Missing expected keys (e.g., `intent_level`) can break downstream expressions.

- **AI Lead Analyzer**
  - **Type/role:** `@n8n/n8n-nodes-langchain.agent` — orchestrates prompt + LLM + structured parsing.
  - **Key config:**
    - System message instructs analysis using: `{{ $json.snippet }}` as the post content.
    - Required outputs: `is_opportunity` (boolean), `intent_level` (high/medium/low), `problem_summary`, `recommended_action`, `confidence_score` (0–100).
    - `hasOutputParser = true` (so it expects structured output).
  - **Inputs:** Receives a single search result item (should contain `snippet` from SerpApi organic result).
  - **Outputs:** To **Filter Valid Opportunities**.
  - **Failure/edge cases:**
    - If the SerpApi item does not include `snippet`, prompt becomes empty → degraded classification.
    - If OpenAI returns unexpected schema, downstream uses `$json.output.*` will fail.
  - **Version note:** Node type versions used: Agent 1.8, Parser 1.3, OpenAI model node 1.

---

### Block D — Filtering & Lead Data Extraction

**Overview:**  
Only items flagged as opportunities proceed. The workflow then reshapes the AI output into top-level fields for routing and task creation.

**Nodes involved:**
- Filter Valid Opportunities
- Extract Lead Data

#### Node details

- **Filter Valid Opportunities**
  - **Type/role:** `n8n-nodes-base.if` — boolean gate.
  - **Condition:** `{{ $json.output.is_opportunity }}` is `true`.
  - **Connections:**
    - **True** output → **Extract Lead Data**
    - **False** output → also connected to **Extract Lead Data** (in the JSON, both outputs point to the same node).
  - **Important issue:** Because both IF branches feed the same node, **non-opportunities will also continue** unless **Extract Lead Data** or downstream routing blocks them. This defeats the filter’s purpose.
  - **Failure/edge cases:**
    - If `$json.output.is_opportunity` is missing or not boolean, the IF may evaluate unexpectedly or error (strict type validation is enabled).

- **Extract Lead Data**
  - **Type/role:** `n8n-nodes-base.set` — maps values into a simplified schema.
  - **Key assignments (all sourced from `$json.output.*`):**
    - `is_opportunity`
    - `intent_level`
    - `problem_summary`
    - `recommended_action`
    - `confidence_score`
  - **Outputs:** To **Route by Intent Level**.
  - **Failure/edge cases:**
    - If `output` object is missing, these expressions resolve to `null` / error depending on expression handling.
    - Types are set as `string` even for boolean/number fields; downstream comparisons might require normalization.

---

### Block E — Intent-Based Routing → Asana Tasks → Google Chat Alerts

**Overview:**  
Leads are routed by `intent_level`. For each route, an Asana task is created and a Google Chat message is posted.

**Nodes involved:**
- Route by Intent Level
- Create High Intent Task → Alert High Intent Lead
- Create Medium Intent Task → Alert Medium Intent Lead
- Create Low Intent Task → Alert Low Intent Lead

#### Node details

- **Route by Intent Level**
  - **Type/role:** `n8n-nodes-base.switch` — branching based on a field.
  - **Configuration issue (critical):**
    - The node’s rules in JSON appear effectively empty (conditions show blank left/right values).
    - Only **one output connection** is defined in `connections`: it routes to **Create High Intent Task** only.
  - **Expected behavior vs current:**
    - Expected: 3 cases (high/medium/low) routed to corresponding Asana nodes.
    - Current: likely routes everything to “High” or routes nothing depending on how n8n interprets empty rules.
  - **Failure/edge cases:** Misrouting causes wrong task priority and alert channel noise.

- **Create High Intent Task**
  - **Type/role:** `n8n-nodes-base.asana` — creates a task.
  - **Auth:** OAuth2 Asana.
  - **Key config:**
    - Workspace: `1212551193156936`
    - Task name: `Reddit Lead: {{ $json.intent_level }}`
    - Notes use `problem_summary` and `confidence_score` (author is blank in template).
  - **Outputs:** To **Alert High Intent Lead**.
  - **Failure/edge cases:**
    - Workspace mismatch, missing permissions, revoked OAuth token.
    - Notes include markdown-like formatting; Asana rendering may differ.
    - Missing fields produce empty task content.

- **Alert High Intent Lead**
  - **Type/role:** `n8n-nodes-base.googleChat` — sends a message to a space.
  - **Auth:** OAuth2 Google Chat.
  - **Key config issues:**
    - `spaceId` is set to `"="` (invalid). Must be a real space resource name/ID.
    - Message text is just `{{ $json.intent_level }}` (minimal; likely should include link/snippet/problem).
  - **Failure/edge cases:**
    - Invalid space ID → 404/400 errors.
    - Google Chat API scopes missing.

- **Create Medium Intent Task** / **Alert Medium Intent Lead**
  - Same structure as “High” versions.
  - **Not connected** from the Switch node in current JSON, so likely never runs.

- **Create Low Intent Task** / **Alert Low Intent Lead**
  - Same structure as “High” versions.
  - **Not connected** from the Switch node in current JSON, so likely never runs.

---

### Block F — Error Handling (global failure → Gmail alert)

**Overview:**  
If any node errors during execution, an Error Trigger starts a separate path that emails error details via Gmail.

**Nodes involved:**
- Workflow Error Handler
- Send Error Email

#### Node details

- **Workflow Error Handler**
  - **Type/role:** `n8n-nodes-base.errorTrigger` — triggers on workflow execution errors.
  - **Outputs:** To **Send Error Email**.
  - **Failure/edge cases:** None at runtime unless disabled; depends on n8n error trigger semantics.

- **Send Error Email**
  - **Type/role:** `n8n-nodes-base.gmail` — sends an email via Gmail.
  - **Key config:**
    - Body includes:
      - Error node name: `{{ $json.node.name }}`
      - Error message: `{{ $json.error.message }}`
      - Timestamp: `{{ $now.toISO() }}`
    - Email type: `text`
  - **Missing config risk:** Recipient / subject fields are not shown in the provided JSON snippet—verify in node UI.
  - **Failure/edge cases:**
    - Gmail OAuth token invalid/expired.
    - Missing “To” address or insufficient scopes.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Note | Sticky Note | Workflow description & requirements | — | — | ## 🎯 Reddit Sales Opportunity Monitor / Automatically scans Reddit every 2 hours… / Requirements: SerpApi, OpenAI API (GPT-4o), Asana, Google Chat / Setup: update search query, IDs |
| Note1 | Sticky Note | Section label | — | — | ## 📡 Data Collection / Scheduled search every 2 hours → Fetches Reddit posts via Google → Splits results |
| Note2 | Sticky Note | Section label | — | — | ## 🧠 AI Analysis & Filtering / AI evaluates each post… |
| Note4 | Sticky Note | Section label | — | — | ## 🔀 Intent-Based Routing / Routes qualified leads by intent level |
| Note5 | Sticky Note | Section label | — | — | ## ✅ Actions & Notifications / Creates Asana tasks… Sends Google Chat alerts |
| Sticky Note7 | Sticky Note | Section label | — | — | ## 🚨 Error Handling / Catches workflow failures → Sends email alerts |
| Every 2 Hours | Cron | Scheduled trigger | — | Search Reddit Posts | ## 📡 Data Collection / Scheduled search every 2 hours → Fetches Reddit posts via Google → Splits results |
| Search Reddit Posts | SerpApi | Google search for Reddit posts | Every 2 Hours | Split Search Results | ## 📡 Data Collection / Scheduled search every 2 hours → Fetches Reddit posts via Google → Splits results |
| Split Search Results | Split Out | Split `organic_results` into items | Search Reddit Posts | Loop Over Items | ## 📡 Data Collection / Scheduled search every 2 hours → Fetches Reddit posts via Google → Splits results |
| Loop Over Items | Split In Batches | Batch looping over items | Split Search Results | AI Lead Analyzer (via output index 1) | ## 📡 Data Collection / Scheduled search every 2 hours → Fetches Reddit posts via Google → Splits results |
| OpenAI Chat Model | OpenAI Chat (LangChain) | LLM provider (gpt-4o) | — | AI Lead Analyzer (ai_languageModel) | ## 🧠 AI Analysis & Filtering / AI evaluates each post… |
| Structured Output Parser | Structured Output Parser (LangChain) | Enforce structured JSON output | — | AI Lead Analyzer (ai_outputParser) | ## 🧠 AI Analysis & Filtering / AI evaluates each post… |
| AI Lead Analyzer | LangChain Agent | Classify opportunity + intent + summary | Loop Over Items (likely miswired), OpenAI Chat Model, Structured Output Parser | Filter Valid Opportunities | ## 🧠 AI Analysis & Filtering / AI evaluates each post… |
| Filter Valid Opportunities | IF | Keep only `is_opportunity=true` | AI Lead Analyzer | Extract Lead Data (both branches) | ## 🧠 AI Analysis & Filtering / AI evaluates each post… |
| Extract Lead Data | Set | Normalize lead fields | Filter Valid Opportunities | Route by Intent Level | ## 🧠 AI Analysis & Filtering / AI evaluates each post… |
| Route by Intent Level | Switch | Route by `intent_level` | Extract Lead Data | Create High Intent Task | ## 🔀 Intent-Based Routing / Routes qualified leads by intent level |
| Create High Intent Task | Asana | Create task for high intent | Route by Intent Level | Alert High Intent Lead | ## ✅ Actions & Notifications / Creates Asana tasks… Sends Google Chat alerts |
| Alert High Intent Lead | Google Chat | Notify team (high) | Create High Intent Task | — | ## ✅ Actions & Notifications / Creates Asana tasks… Sends Google Chat alerts |
| Create Medium Intent Task | Asana | Create task for medium intent | (not connected) | Alert Medium Intent Lead | ## ✅ Actions & Notifications / Creates Asana tasks… Sends Google Chat alerts |
| Alert Medium Intent Lead | Google Chat | Notify team (medium) | Create Medium Intent Task | — | ## ✅ Actions & Notifications / Creates Asana tasks… Sends Google Chat alerts |
| Create Low Intent Task | Asana | Create task for low intent | (not connected) | Alert Low Intent Lead | ## ✅ Actions & Notifications / Creates Asana tasks… Sends Google Chat alerts |
| Alert Low Intent Lead | Google Chat | Notify team (low) | Create Low Intent Task | — | ## ✅ Actions & Notifications / Creates Asana tasks… Sends Google Chat alerts |
| Workflow Error Handler | Error Trigger | Triggers on workflow errors | — | Send Error Email | ## 🚨 Error Handling / Catches workflow failures → Sends email alerts |
| Send Error Email | Gmail | Email error details | Workflow Error Handler | — | ## 🚨 Error Handling / Catches workflow failures → Sends email alerts |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name it: **“Monitor Reddit for sales opportunities and create Asana tasks”**
- (Optional) Add sticky notes matching the sections described above.

2) **Add the schedule trigger**
- Add node: **Cron**
- Configure it to run **every 2 hours** (verify timezone).
- Connect Cron → SerpApi node.

3) **Add SerpApi Google search**
- Add node: **SerpApi**
- Operation: Google search (default for the SerpApi node)
- Set query `q` to:  
  `site:reddit.com ("looking for" OR "need help with" OR "any tool for" OR "recommend software")`
- Set `location` to your target geography (e.g., “India” or remove for broader results).
- Add **SerpApi credentials** (API key).
- Connect SerpApi → Split Out.

4) **Split results into individual items**
- Add node: **Split Out**
- Field to split: `organic_results`
- Connect Split Out → Split In Batches.

5) **Loop through posts**
- Add node: **Split In Batches**
- Set a batch size (recommended: 5–20) to control OpenAI/Asana rate limits.
- **Important:** Connect **Output 0 (Items)** → AI Lead Analyzer (this is the typical correct wiring).  
  (The provided JSON connects output 1, which is likely wrong.)
- (Optional) Connect AI path back into Split In Batches for continued looping if your n8n pattern requires it (depends on your version/pattern).

6) **Add OpenAI model**
- Add node: **OpenAI Chat Model (LangChain)**
- Model: **gpt-4o**
- Temperature: **0.3**
- Configure **OpenAI API credentials**.
- This node connects to the Agent via the **AI Language Model** connector.

7) **Add Structured Output Parser**
- Add node: **Structured Output Parser (LangChain)**
- Define the expected schema in the node UI (recommended):
  - `is_opportunity` (boolean)
  - `intent_level` (string; enum: high/medium/low)
  - `problem_summary` (string)
  - `recommended_action` (string)
  - `confidence_score` (number)
- Connect parser to agent via the **AI Output Parser** connector.

8) **Add AI Lead Analyzer (Agent)**
- Add node: **LangChain Agent**
- Set the system message to analyze: `{{$json.snippet}}` and to output structured JSON.
- Enable structured output parsing (so parser is used).
- Connect:
  - Split In Batches (items output) → Agent (main)
  - OpenAI Chat Model → Agent (ai_languageModel)
  - Structured Output Parser → Agent (ai_outputParser)
- Connect Agent → IF node.

9) **Filter only valid opportunities**
- Add node: **IF**
- Condition: `{{$json.output.is_opportunity}}` **is true**
- Connect **True** output → Set node.
- **Do not connect False** output forward if you want to drop non-opportunities (the provided workflow currently sends both branches forward).

10) **Extract/normalize lead fields**
- Add node: **Set**
- Create fields from `{{$json.output.*}}`:
  - `intent_level`, `problem_summary`, `recommended_action`, `confidence_score`
- Connect Set → Switch.

11) **Route by intent level**
- Add node: **Switch**
- Value to evaluate: `{{$json.intent_level}}`
- Add 3 rules:
  - equals `high` → High path
  - equals `medium` → Medium path
  - equals `low` → Low path
- Connect each output to its respective Asana node.

12) **Create Asana tasks (High/Medium/Low)**
- Add 3 nodes: **Asana**
- Operation: Create task
- Workspace: your workspace ID (e.g., `1212551193156936`)
- Name: `Reddit Lead: {{$json.intent_level}}`
- Notes: include problem summary + confidence + recommended action.
- Configure **Asana OAuth2 credentials**.
- Connect each Asana node → corresponding Google Chat node.

13) **Send Google Chat alerts**
- Add 3 nodes: **Google Chat**
- Configure OAuth2 credentials/scopes for Chat.
- Set `spaceId` to the correct space identifier (must not be `"="`).
- Message text: include at least `intent_level`, `problem_summary`, and ideally the post URL/title if available from SerpApi item fields.
- Connect from each Asana node to its Chat node.

14) **Add error handling**
- Add node: **Error Trigger**
- Add node: **Gmail**
- Configure Gmail OAuth2.
- Set To/Subject (verify in UI) and message body using:
  - `{{$json.node.name}}`
  - `{{$json.error.message}}`
  - `{{$now.toISO()}}`
- Connect Error Trigger → Gmail.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Automatically scans Reddit every 2 hours to identify potential sales opportunities using AI analysis; requires SerpApi, OpenAI (GPT‑4o), Asana workspace, Google Chat space; update search query and IDs before running | From sticky note “Reddit Sales Opportunity Monitor” |
| Scheduled search every 2 hours → Fetches Reddit posts via Google → Splits results for individual processing | From sticky note “Data Collection” |
| AI evaluates each post for sales intent → Filters out non-opportunities → Extracts structured lead data | From sticky note “AI Analysis & Filtering” |
| Routes qualified leads by intent level: high → medium → low | From sticky note “Intent-Based Routing” |
| Creates Asana tasks for each intent level → Sends Google Chat alerts to notify team | From sticky note “Actions & Notifications” |
| Catches workflow failures → Sends email alerts with error details | From sticky note “Error Handling” |