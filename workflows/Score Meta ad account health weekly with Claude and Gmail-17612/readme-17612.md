Score Meta ad account health weekly with Claude and Gmail

https://n8nworkflows.xyz/workflows/score-meta-ad-account-health-weekly-with-claude-and-gmail-17612


# Score Meta ad account health weekly with Claude and Gmail

### 1. Workflow Overview

This workflow automates the weekly health assessment of a Meta Ads account. It pulls performance metrics from the Meta Marketing API, runs programmatic evaluation logic to score account health across six categories (efficiency, waste, fatigue, delivery, structure, and momentum), uses Anthropic Claude to rank the top three fixes, and outputs a formatted scorecard.

The logical execution is grouped into four distinct functional blocks:
- **1.1 Trigger & Configuration:** Initializes execution on a weekly schedule and establishes configuration parameters (KPI thresholds, conversion action types, and category weights).
- **1.2 Data Ingestion:** Computes date ranges for comparison and queries the Meta Marketing API sequentially at the account, ad set, and ad levels.
- **1.3 Processing & AI Triage:** Transforms raw metrics into structured health checks, calculates an overall score (0–100) and severity band, and prompts Claude to generate executive-level copy and fix rankings.
- **1.4 Output Generation:** Assembles the final report payload containing the text scorecard and underlying structured JSON data for downstream integrations.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Configuration
- **Overview:** Starts the workflow on a weekly schedule (Mondays) and defines environment thresholds, conversion event definitions, and scoring parameters.
- **Nodes Involved:** `Schedule Trigger`, `Set config: health score`

- **Node Details:**
  - **Schedule Trigger**
    - *Type and Technical Role:* `n8n-nodes-base.scheduleTrigger` (v1.3) – Initiates the workflow execution automatically.
    - *Configuration Choices:* Configured to trigger weekly on Mondays.
    - *Input/Output:* No inputs; outputs main connection to `Set config: health score`.
    - *Potential Failures:* Execution skipped if workflow is inactive.
  - **Set config: health score**
    - *Type and Technical Role:* `n8n-nodes-base.code` (v2) – Defines configuration constants (date presets, target conversion actions, KPI flag percentages, frequency ceilings, waste limits, and scoring weights).
    - *Configuration Choices:* Custom JavaScript returning a configuration JSON object.
    - *Input/Output:* Input from `Schedule Trigger`; output to `Build comparison windows`.
    - *Potential Failures:* Syntax errors in the JavaScript return statement.

---

#### 2.2 Data Ingestion
- **Overview:** Calculates time windows relative to yesterday and sequentially fetches performance insights and entity data from the Meta Marketing API.
- **Nodes Involved:** `Build comparison windows`, `Fetch account this week (Meta)`, `Fetch account prior week (Meta)`, `Fetch ad set insights (Meta)`, `Fetch ad set entities (Meta)`, `Fetch ad insights (Meta)`

- **Node Details:**
  - **Build comparison windows**
    - *Type and Technical Role:* `n8n-nodes-base.code` (v2) – Performs date math to establish two comparative 7-day windows (`this_window` and `prior_window`).
    - *Configuration Choices:* Reads configuration from the previous node and outputs ISO date strings and JSON-stringified time ranges.
    - *Input/Output:* Input from `Set config: health score`; output to `Fetch account this week (Meta)`.
    - *Potential Failures:* Date calculation errors if timezone offsets drift.
  - **Fetch account this week (Meta)**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (v4.2) – Queries Meta Marketing API for account-level metrics for the current window.
    - *Configuration Choices:* Uses endpoint `https://graph.facebook.com/{{ $env.META_API_VERSION || 'v21.0' }}/{{ $env.META_AD_ACCOUNT_ID }}/insights`. Passes `access_token`, `level=account`, and `time_range`. Requests fields: `spend,impressions,reach,frequency,clicks,ctr,cpm,cpc,actions,purchase_roas`.
    - *Input/Output:* Input from `Build comparison windows`; output to `Fetch account prior week (Meta)`.
    - *Potential Failures:* Authentication errors (`META_ACCESS_TOKEN`), invalid account ID (`META_AD_ACCOUNT_ID`), or API rate limits.
  - **Fetch account prior week (Meta)**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (v4.2) – Queries Meta Marketing API for account-level metrics for the prior comparative window.
    - *Configuration Choices:* Identical endpoint configuration to the previous node, substituting `prior_range` for the time range.
    - *Input/Output:* Input from `Fetch account this week (Meta)`; output to `Fetch ad set insights (Meta)`.
    - *Potential Failures:* Token expiration or network timeouts.
  - **Fetch ad set insights (Meta)**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (v4.2) – Fetches ad set level performance insights for the current window.
    - *Configuration Choices:* Queries with `level=adset`, `time_range={{ $('Build comparison windows').first().json.this_range }}`, fields: `adset_id,adset_name,spend,impressions,frequency,ctr,cpc,actions,purchase_roas`, and `limit=500`.
    - *Input/Output:* Input from `Fetch account prior week (Meta)`; output to `Fetch ad set entities (Meta)`.
    - *Potential Failures:* Truncated results if ad sets exceed pagination limits (`limit=500`).
  - **Fetch ad set entities (Meta)**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (v4.2) – Retrieves ad set metadata to inspect active status and budgets.
    - *Configuration Choices:* Queries endpoint `/adsets` with fields `id,name,status,effective_status,daily_budget,lifetime_budget` and `limit=500`.
    - *Input/Output:* Input from `Fetch ad set insights (Meta)`; output to `Fetch ad insights (Meta)`.
    - *Potential Failures:* Permissions issues on ad set reading scopes.
  - **Fetch ad insights (Meta)**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (v4.2) – Retrieves ad-level performance insights for waste identification.
    - *Configuration Choices:* Queries `/insights` with `level=ad`, fields `ad_id,ad_name,adset_id,adset_name,spend,impressions,ctr,actions,purchase_roas`, and `limit=500`.
    - *Input/Output:* Input from `Fetch ad set entities (Meta)`; output to `Run checks and score`.
    - *Potential Failures:* Large payload sizes causing memory warnings on high-volume accounts.

---

#### 2.3 Processing & AI Triage
- **Overview:** Evaluates account health checks, calculates category-weighted scores, detects sales vs. lead-gen modes, and prepares a structured payload for Anthropic Claude.
- **Nodes Involved:** `Run checks and score`, `Build AI triage prompt`, `Rank fixes with Claude AI`

- **Node Details:**
  - **Run checks and score**
    - *Type and Technical Role:* `n8n-nodes-base.code` (v2) – Aggregates API responses, executes health check batteries (efficiency, fatigue, waste, delivery, structure, momentum), and calculates a final score (0–100).
    - *Configuration Choices:* Pure JavaScript transformations. Automatically detects objective mode (`SALES` via ROAS presence or defaults to `LEADS`).
    - *Input/Output:* Input from `Fetch ad insights (Meta)`; output to `Build AI triage prompt`.
    - *Potential Failures:* Division by zero if inputs contain malformed numbers (handled via safety helper functions).
  - **Build AI triage prompt**
    - *Type and Technical Role:* `n8n-nodes-base.code` (v2) – Formulates system and user prompts using structured scorecard data for Anthropic Claude.
    - *Configuration Choices:* Sets model to `claude-haiku-4-5` and constructs strict JSON-only instructions.
    - *Input/Output:* Input from `Run checks and score`; output to `Rank fixes with Claude AI`.
    - *Potential Failures:* Context extraction failures if upstream data lacks expected schema.
  - **Rank fixes with Claude AI**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (v4.2) – Sends the triage prompt to Anthropic’s Messages API.
    - *Configuration Choices:* POST request to `https://api.anthropic.com/v1/messages` using headers `x-api-key`, `anthropic-version: 2023-06-01`, and `content-type: application/json`.
    - *Input/Output:* Input from `Build AI triage prompt`; output to `Print health scorecard`.
    - *Potential Failures:* API authentication failure (`ANTHROPIC_API_KEY`), rate limiting, or invalid JSON response formatting from the model.

---

#### 2.4 Output Generation
- **Overview:** Parses the AI response, builds a plain-text executive summary report, and packages all diagnostic metrics for export.
- **Nodes Involved:** `Print health scorecard`

- **Node Details:**
  - **Print health scorecard**
    - *Type and Technical Role:* `n8n-nodes-base.code` (v2) – Assembles the final plain-text health report string and aggregates diagnostic metadata.
    - *Configuration Choices:* Extracts JSON content from Claude's response (with fallback handling) and formats a structured text scorecard.
    - *Input/Output:* Input from `Rank fixes with Claude AI`; terminal node output.
    - *Potential Failures:* JSON parsing errors if Claude returns unescaped characters outside the JSON payload.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Schedule Trigger | n8n-nodes-base.scheduleTrigger | Initiates workflow execution weekly. | None | Set config: health score | Meta Ads Weekly Health Score |
| Set config: health score | n8n-nodes-base.code | Sets configuration variables, weights, and thresholds. | Schedule Trigger | Build comparison windows | 1. Config and windows |
| Build comparison windows | n8n-nodes-base.code | Generates 7-day comparative date windows. | Set config: health score | Fetch account this week (Meta) | 1. Config and windows |
| Fetch account this week (Meta) | n8n-nodes-base.httpRequest | Pulls current week account performance. | Build comparison windows | Fetch account prior week (Meta) | 2. Pull both windows at three levels |
| Fetch account prior week (Meta) | n8n-nodes-base.httpRequest | Pulls prior week account performance. | Fetch account this week (Meta) | Fetch ad set insights (Meta) | 2. Pull both windows at three levels |
| Fetch ad set insights (Meta) | n8n-nodes-base.httpRequest | Pulls ad set level performance insights. | Fetch account prior week (Meta) | Fetch ad set entities (Meta) | 2. Pull both windows at three levels |
| Fetch ad set entities (Meta) | n8n-nodes-base.httpRequest | Retrieves ad set status and budget metadata. | Fetch ad set insights (Meta) | Fetch ad insights (Meta) | 2. Pull both windows at three levels |
| Fetch ad insights (Meta) | n8n-nodes-base.httpRequest | Retrieves ad level insights for waste detection. | Fetch ad set entities (Meta) | Run checks and score | 2. Pull both windows at three levels |
| Run checks and score | n8n-nodes-base.code | Evaluates health checks and calculates score. | Fetch ad insights (Meta) | Build AI triage prompt | 3. Score, rank, report |
| Build AI triage prompt | n8n-nodes-base.code | Prepares system/user prompts for Claude. | Run checks and score | Rank fixes with Claude AI | 3. Score, rank, report |
| Rank fixes with Claude AI | n8n-nodes-base.httpRequest | Calls Anthropic API to rank fixes. | Build AI triage prompt | Print health scorecard | 3. Score, rank, report |
| Print health scorecard | n8n-nodes-base.code | Generates final formatted text report and payload. | Rank fixes with Claude AI | None | 3. Score, rank, report |

---

### 4. Reproducing the Workflow from Scratch

Follow these numbered steps to recreate the workflow manually in n8n:

1. **Create the Trigger Node:**
   - Add a **Schedule Trigger** node. Set the interval trigger rule to weekly on Mondays (`triggerAtDay: [1]`).
2. **Configure Global Settings:**
   - Add a **Code** node named `Set config: health score`.
   - Paste JavaScript logic containing threshold values (`DATE_PRESET`, `RESULT_ACTIONS`, `KPI_FLAG_PCT`, `WEIGHTS`, etc.).
   - Connect `Schedule Trigger` to this node.
3. **Establish Date Windows:**
   - Add a **Code** node named `Build comparison windows`.
   - Implement date math logic to calculate `this_window` and `prior_window` based on yesterday's date.
   - Connect `Set config: health score` to this node.
4. **Fetch Account Metrics (Current Period):**
   - Add an **HTTP Request** node named `Fetch account this week (Meta)`.
   - Set method to `GET`, URL to `https://graph.facebook.com/{{ $env.META_API_VERSION || 'v21.0' }}/{{ $env.META_AD_ACCOUNT_ID }}/insights`.
   - Add query parameters for `access_token`, `level=account`, `time_range` (mapped from `Build comparison windows`), and required fields (`spend,impressions,reach,frequency,clicks,ctr,cpm,cpc,actions,purchase_roas`).
   - Configure credentials for Meta API access via environment variables.
   - Connect `Build comparison windows` to this node.
5. **Fetch Account Metrics (Prior Period):**
   - Add an **HTTP Request** node named `Fetch account prior week (Meta)`.
   - Duplicate configuration from the previous node, utilizing `prior_range` for the time range.
   - Connect `Fetch account this week (Meta)` to this node.
6. **Fetch Ad Set Insights:**
   - Add an **HTTP Request** node named `Fetch ad set insights (Meta)`.
   - Set URL and query parameters with `level=adset`, `time_range`, fields (`adset_id,adset_name,spend,impressions,frequency,ctr,cpc,actions,purchase_roas`), and `limit=500`.
   - Connect `Fetch account prior week (Meta)` to this node.
7. **Fetch Ad Set Entities:**
   - Add an **HTTP Request** node named `Fetch ad set entities (Meta)`.
   - Query endpoint `/adsets` with fields (`id,name,status,effective_status,daily_budget,lifetime_budget`) and `limit=500`.
   - Connect `Fetch ad set insights (Meta)` to this node.
8. **Fetch Ad Insights:**
   - Add an **HTTP Request** node named `Fetch ad insights (Meta)`.
   - Query endpoint `/insights` with `level=ad`, fields (`ad_id,ad_name,adset_id,adset_name,spend,impressions,ctr,actions,purchase_roas`), and `limit=500`.
   - Connect `Fetch ad set entities (Meta)` to this node.
9. **Run Checks and Scoring Logic:**
   - Add a **Code** node named `Run checks and score`.
   - Implement transformation logic parsing previous node inputs to check efficiency, fatigue, waste, delivery hygiene, and structure, outputting a consolidated JSON with scores and breakdowns.
   - Connect `Fetch ad insights (Meta)` to this node.
10. **Build AI Triage Prompt:**
    - Add a **Code** node named `Build AI triage prompt`.
    - Construct system instructions and user payloads targeting Claude (`claude-haiku-4-5`) with strict JSON output formatting constraints.
    - Connect `Run checks and score` to this node.
11. **Call Anthropic API:**
    - Add an **HTTP Request** node named `Rank fixes with Claude AI`.
    - Set method to `POST`, URL to `https://api.anthropic.com/v1/messages`.
    - Set headers (`x-api-key`, `anthropic-version: 2023-06-01`, `content-type: application/json`).
    - Specify body as JSON mapped from `{{ $json.body }}`.
    - Configure environment variable `ANTHROPIC_API_KEY`.
    - Connect `Build AI triage prompt` to this node.
12. **Generate Final Scorecard Report:**
    - Add a **Code** node named `Print health scorecard`.
    - Implement response parsing and plain-text report compilation.
    - Connect `Rank fixes with Claude AI` to this node as the final step.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Built by nocode.expert - done-for-you automation & tracking. | Project credits and branding. |
| Full walkthrough, including every threshold | https://nocode.expert/resources/how-to-check-facebook-ad-account-health |