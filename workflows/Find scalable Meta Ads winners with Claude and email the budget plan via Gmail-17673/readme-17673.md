Find scalable Meta Ads winners with Claude and email the budget plan via Gmail

https://n8nworkflows.xyz/workflows/find-scalable-meta-ads-winners-with-claude-and-email-the-budget-plan-via-gmail-17673


# Find scalable Meta Ads winners with Claude and email the budget plan via Gmail

### 1. Workflow Overview

This workflow is designed to automate the discovery of scaling opportunities within Meta Ads accounts. Instead of focusing on ad waste, it identifies winning ads that are currently outperforming account benchmarks but are restricted by budget caps or frequency constraints. It calculates safe budget increases, uses Anthropic Claude to formulate a structured scaling plan, and delivers the finalized report via Gmail.

The workflow is grouped into four logical blocks:
- **1.1 Initialization and Configuration:** Triggers the workflow execution manually and initializes operational thresholds (such as minimum spend, result types, and target lift percentages).
- **1.2 Meta Marketing API Data Ingestion:** Pulls performance insights and entity budgets for the account, ads, ad sets, and campaigns from the Meta Marketing API.
- **1.3 Data Processing and AI Synthesis:** Normalizes metrics, calculates performance objectives (ROAS for sales vs. cost-per-result for leads), ranks winners based on budget constraints, and constructs an optimized prompt for Anthropic Claude.
- **1.4 Reporting and Output Delivery:** Parses Claude's generated scale plan, constructs a formatted text report, and dispatches it via Gmail.

---

### 2. Block-by-Block Analysis

#### 2.1 Initialization and Configuration
This block serves as the entry point for manual execution and establishes the parameters that govern performance evaluation thresholds.

- **Nodes Involved:** `Run manually`, `Set config: winners`
- **Node Details:**
  - **Run manually (`n8n-nodes-base.manualTrigger`)**
    - *Type and Role:* Manual execution trigger node. Initiates the workflow on demand.
    - *Configuration Choices:* Default configuration with no parameters required.
    - *Input/Output:* No inputs; outputs to `Set config: winners`.
    - *Edge Cases:* Manual execution only; requires external scheduling (such as a Cron or Schedule trigger) if automated weekly runs are desired.
  - **Set config: winners (`n8n-nodes-base.code`)**
    - *Type and Role:* JavaScript code execution node. Sets configuration constants used downstream.
    - *Configuration Choices:* Defines `DATE_PRESET` (`last_7d`), conversion actions (`RESULT_ACTIONS`), `MIN_AD_SPEND` (100), `MIN_AD_RESULTS` (3), `BEAT_PCT` (20), `FREQ_HEADROOM` (2.0), `CAP_UTILISATION_PCT` (85), and `SUGGESTED_LIFT_PCT` (30).
    - *Key Expressions:* Returns a JSON array containing the configuration parameters.
    - *Input/Output:* Input from `Run manually`; output to `Fetch account benchmark (Meta)`.
    - *Edge Cases:* Modifying financial or performance thresholds directly impacts downstream filtering and qualification criteria.

---

#### 2.2 Meta Marketing API Data Ingestion
This block sequentially fetches performance insights and configuration entities from the Meta Marketing API across account, ad, ad set, and campaign levels.

- **Nodes Involved:** `Fetch account benchmark (Meta)`, `Fetch ad insights (Meta)`, `Fetch ad set insights (Meta)`, `Fetch ad set entities (Meta)`, `Fetch campaign insights (Meta)`, `Fetch campaign entities (Meta)`
- **Node Details:**
  - **Fetch account benchmark (Meta) (`n8n-nodes-base.httpRequest`)**
    - *Type and Role:* HTTP Request node. Retrieves account-level performance metrics to establish a baseline benchmark.
    - *Configuration Choices:* GET request to Meta Graph API (`v21.0` or custom environment version), querying spend, impressions, clicks, CTR, actions, and purchase ROAS for the preset date range.
    - *Key Expressions:* Uses `{{ $env.META_API_VERSION || 'v21.0' }}`, `{{ $env.META_AD_ACCOUNT_ID }}`, `{{ $env.META_ACCESS_TOKEN }}`, and references `{{ $('Set config: winners').first().json.DATE_PRESET }}`.
    - *Input/Output:* Input from `Set config: winners`; output to `Fetch ad insights (Meta)`.
    - *Edge Cases:* Authentication failures if `META_ACCESS_TOKEN` is invalid; API rate limits if executed concurrently across multiple accounts.
  - **Fetch ad insights (Meta) (`n8n-nodes-base.httpRequest`)**
    - *Type and Role:* HTTP Request node. Pulls performance metrics at the individual ad level.
    - *Configuration Choices:* Queries ad-level insights including spend, impressions, CTR, actions, and purchase ROAS with a limit of 500 records.
    - *Key Expressions:* Uses environment variables for credentials and configuration variables for date presets.
    - *Input/Output:* Input from `Fetch account benchmark (Meta)`; output to `Fetch ad set insights (Meta)`.
    - *Edge Cases:* Large ad accounts may exceed the 500-record limit, requiring pagination adjustments.
  - **Fetch ad set insights (Meta) (`n8n-nodes-base.httpRequest`)**
    - *Type and Role:* HTTP Request node. Retrieves performance data at the ad set level, specifically frequency metrics.
    - *Configuration Choices:* Queries ad set insights for spend, impressions, frequency, and CTR.
    - *Input/Output:* Input from `Fetch ad insights (Meta)`; output to `Fetch ad set entities (Meta)`.
    - *Edge Cases:* Missing frequency data can cause headroom calculations to evaluate incorrectly.
  - **Fetch ad set entities (Meta) (`n8n-nodes-base.httpRequest`)**
    - *Type and Role:* HTTP Request node. Fetches ad set configuration metadata, including budgets and statuses.
    - *Configuration Choices:* Queries ad set entities for `id`, `name`, `status`, `effective_status`, `daily_budget`, `lifetime_budget`, `start_time`, and `stop_time`.
    - *Input/Output:* Input from `Fetch ad set insights (Meta)`; output to `Fetch campaign insights (Meta)`.
    - *Edge Cases:* Advantage+ or CBO campaigns store budgets at the campaign level rather than the ad set level, returning empty daily budgets here.
  - **Fetch campaign insights (Meta) (`n8n-nodes-base.httpRequest`)**
    - *Type and Role:* HTTP Request node. Pulls campaign-level performance metrics as a secondary fallback.
    - *Configuration Choices:* Queries campaign-level insights for spend, impressions, and frequency.
    - *Input/Output:* Input from `Fetch ad set entities (Meta)`; output to `Fetch campaign entities (Meta)`.
    - *Edge Cases:* Failure to retrieve campaign insights prevents fallback resolution for campaign-budget-optimized (CBO) structures.
  - **Fetch campaign entities (Meta) (`n8n-nodes-base.httpRequest`)**
    - *Type and Role:* HTTP Request node. Retrieves campaign configuration metadata for budget-optimized structures.
    - *Configuration Choices:* Queries campaign entities for metadata, status, and budget allocations.
    - *Input/Output:* Input from `Fetch campaign insights (Meta)`; output to `Rank winners and size the lift`.
    - *Edge Cases:* Missing budget fields on both ad set and campaign levels will mark ads as uncapped.

---

#### 2.3 Data Processing and AI Synthesis
This block evaluates raw API payloads, determines the account objective mode (Sales vs. Leads), identifies budget-capped winners, calculates budget scaling steps, and prepares the prompt for Claude AI.

- **Nodes Involved:** `Rank winners and size the lift`, `Build AI scale prompt`, `Write scale plan with Claude AI`
- **Node Details:**
  - **Rank winners and size the lift (`n8n-nodes-base.code`)**
    - *Type and Role:* JavaScript code transformation node. Processes insights and entity data to filter winners and calculate recommended budget lifts.
    - *Configuration Choices:* Calculates KPI modes (`SALES` based on ROAS, `LEADS` based on cost per result), applies threshold filters (`MIN_AD_SPEND`, `BEAT_PCT`), analyzes budget caps, and ranks candidates.
    - *Input/Output:* Input from `Fetch campaign entities (Meta)`; output to `Build AI scale prompt`.
    - *Edge Cases:* Zero results or division-by-zero errors when calculating cost-per-result metrics on low-volume accounts.
  - **Build AI scale prompt (`n8n-nodes-base.code`)**
    - *Type and Role:* JavaScript code transformation node. Constructs system and user prompts incorporating strict formatting rules for Claude.
    - *Configuration Choices:* Separates scalable and blocked ads, ensuring strict constraints on budget figures. Prepares request payload for Anthropic API.
    - *Input/Output:* Input from `Rank winners and size the lift`; output to `Write scale plan with Claude AI`.
    - *Edge Cases:* Token limit exhaustion if the winner list is excessively large.
  - **Write scale plan with Claude AI (`n8n-nodes-base.httpRequest`)**
    - *Type and Role:* HTTP Request node. Invokes the Anthropic Messages API to generate the structured scaling report.
    - *Configuration Choices:* POST request to `https://api.anthropic.com/v1/messages` using model `claude-haiku-4-5` with `max_tokens: 1400`.
    - *Key Expressions:* Uses `{{ $env.ANTHROPIC_API_KEY }}` for authentication header `x-api-key` and passes JSON body from preceding node.
    - *Input/Output:* Input from `Build AI scale prompt`; output to `Print scale plan`.
    - *Edge Cases:* API rate limits, invalid API keys, or JSON parsing failures if the model returns unstructured markdown.

---

#### 2.4 Reporting and Output Delivery
This block compiles the structured data and AI response into a readable text report and delivers it via email.

- **Nodes Involved:** `Print scale plan`, `Send a message`
- **Node Details:**
  - **Print scale plan (`n8n-nodes-base.code`)**
    - *Type and Role:* JavaScript code transformation node. Parses the AI response and compiles a human-readable text report.
    - *Configuration Choices:* Safely extracts JSON from Claude's response text and builds a multi-line report detailing account averages, top actions, and total upside projections.
    - *Input/Output:* Input from `Write scale plan with Claude AI`; output to `Send a message`.
    - *Edge Cases:* Malformed JSON response from the AI defaults to an empty scaling plan structure.
  - **Send a message (`n8n-nodes-base.gmail`)**
    - *Type and Role:* Gmail integration node. Sends the finalized scaling report via email.
    - *Configuration Choices:* Sends a message using the compiled report string.
    - *Key Expressions:* Uses `={{ $json.report }}` for the message body.
    - *Credentials:* Requires valid Gmail OAuth2 credentials.
    - *Input/Output:* Input from `Print scale plan`; no downstream output.
    - *Edge Cases:* OAuth2 token expiration, invalid recipient settings, or sending limits imposed by Google Workspace accounts.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Run manually | n8n-nodes-base.manualTrigger | Initiates workflow execution on demand | None | Set config: winners | Meta Winners to Scale: Most ad audits only find waste. This one finds upside: the ads already beating your account average that are being held back... Built by **nocode.expert** - done-for-you automation & tracking. https://nocode.expert |
| Set config: winners | n8n-nodes-base.code | Defines performance thresholds and operational configurations | Run manually | Fetch account benchmark (Meta) | 1. Config: MIN_AD_SPEND and MIN_AD_RESULTS stop a lucky two-conversion ad reading as a winner. BEAT_PCT is how far past the account average an ad has to be before it counts. CAP_UTILISATION_PCT is what "throttled" means... |
| Fetch account benchmark (Meta) | n8n-nodes-base.httpRequest | Pulls account-level performance to establish a baseline benchmark | Set config: winners | Fetch ad insights (Meta) | 2. Pull the account, the ads, and the budgets: The account call sets the benchmark every ad is measured against. The ad call finds performance. The ad set insight call gives frequency... |
| Fetch ad insights (Meta) | n8n-nodes-base.httpRequest | Fetches performance metrics at the individual ad level | Fetch account benchmark (Meta) | Fetch ad set insights (Meta) | 2. Pull the account, the ads, and the budgets: The account call sets the benchmark every ad is measured against. The ad call finds performance. The ad set insight call gives frequency... |
| Fetch ad set insights (Meta) | n8n-nodes-base.httpRequest | Retrieves ad set level performance and frequency data | Fetch ad insights (Meta) | Fetch ad set entities (Meta) | 2. Pull the account, the ads, and the budgets: The account call sets the benchmark every ad is measured against. The ad call finds performance. The ad set insight call gives frequency... |
| Fetch ad set entities (Meta) | n8n-nodes-base.httpRequest | Fetches ad set configuration metadata and budgets | Fetch ad set insights (Meta) | Fetch campaign insights (Meta) | 2. Pull the account, the ads, and the budgets: The account call sets the benchmark every ad is measured against. The ad call finds performance. The ad set insight call gives frequency... |
| Fetch campaign insights (Meta) | n8n-nodes-base.httpRequest | Pulls campaign-level insights as a fallback mechanism | Fetch ad set entities (Meta) | Fetch campaign entities (Meta) | 2. Pull the account, the ads, and the budgets: The account call sets the benchmark every ad is measured against. The ad call finds performance. The ad set insight call gives frequency... |
| Fetch campaign entities (Meta) | n8n-nodes-base.httpRequest | Retrieves campaign metadata and budget allocations | Fetch campaign insights (Meta) | Rank winners and size the lift | 2. Pull the account, the ads, and the budgets: The account call sets the benchmark every ad is measured against. The ad call finds performance. The ad set insight call gives frequency... |
| Rank winners and size the lift | n8n-nodes-base.code | Processes metrics, identifies winners, and calculates budget increases | Fetch campaign entities (Meta) | Build AI scale prompt | 3. Rank, size, report: Winners are ranked by why they are held back, not just by performance. Capped and unfatigued comes first... Claude only orders and phrases. Every number comes from the ranking node. |
| Build AI scale prompt | n8n-nodes-base.code | Constructs system and user prompts for Claude AI integration | Rank winners and size the lift | Write scale plan with Claude AI | 3. Rank, size, report: Winners are ranked by why they are held back, not just by performance. Capped and unfatigued comes first... Claude only orders and phrases. Every number comes from the ranking node. |
| Write scale plan with Claude AI | n8n-nodes-base.httpRequest | Calls Anthropic Claude API to generate scaling instructions | Build AI scale prompt | Print scale plan | 3. Rank, size, report: Winners are ranked by why they are held back, not just by performance. Capped and unfatigued comes first... Claude only orders and phrases. Every number comes from the ranking node. |
| Print scale plan | n8n-nodes-base.code | Parses AI output and formats the comprehensive text report | Write scale plan with Claude AI | Send a message | 3. Rank, size, report: Winners are ranked by why they are held back, not just by performance. Capped and unfatigued comes first... Claude only orders and phrases. Every number comes from the ranking node. |
| Send a message | n8n-nodes-base.gmail | Delivers the finalized email report to recipients | Print scale plan | None | |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually in n8n:

1. **Create the Trigger Node:**
   - Add a **Manual Trigger** node (`n8n-nodes-base.manualTrigger`). Name it `Run manually`.

2. **Add the Configuration Node:**
   - Add a **Code** node (`n8n-nodes-base.code`) named `Set config: winners`.
   - Configure the JavaScript code to return configuration parameters (`DATE_PRESET`, `RESULT_ACTIONS`, `MIN_AD_SPEND`, `MIN_AD_RESULTS`, `BEAT_PCT`, `FREQ_HEADROOM`, `CAP_UTILISATION_PCT`, `SUGGESTED_LIFT_PCT`).
   - Connect `Run manually` to `Set config: winners`.

3. **Configure Meta API HTTP Request Nodes:**
   - **Node 1 (`Fetch account benchmark (Meta)`):**
     - Type: HTTP Request (`n8n-nodes-base.httpRequest`)
     - Method: GET
     - URL: `=https://graph.facebook.com/{{ $env.META_API_VERSION || 'v21.0' }}/{{ $env.META_AD_ACCOUNT_ID }}/insights`
     - Query Parameters: `access_token` (`={{ $env.META_ACCESS_TOKEN }}`), `level` (`account`), `date_preset` (`={{ $('Set config: winners').first().json.DATE_PRESET }}`), `fields` (`spend,impressions,clicks,ctr,actions,purchase_roas`).
   - **Node 2 (`Fetch ad insights (Meta)`):**
     - URL: `=https://graph.facebook.com/{{ $env.META_API_VERSION || 'v21.0' }}/{{ $env.META_AD_ACCOUNT_ID }}/insights`
     - Query Parameters: `access_token`, `level` (`ad`), `date_preset`, `fields` (`ad_id,ad_name,adset_id,adset_name,campaign_id,campaign_name,spend,impressions,ctr,actions,purchase_roas`), `limit` (`500`).
   - **Node 3 (`Fetch ad set insights (Meta)`):**
     - URL: `=https://graph.facebook.com/{{ $env.META_API_VERSION || 'v21.0' }}/{{ $env.META_AD_ACCOUNT_ID }}/insights`
     - Query Parameters: `access_token`, `level` (`adset`), `date_preset`, `fields` (`adset_id,adset_name,spend,impressions,frequency,ctr`), `limit` (`500`).
   - **Node 4 (`Fetch ad set entities (Meta)`):**
     - URL: `=https://graph.facebook.com/{{ $env.META_API_VERSION || 'v21.0' }}/{{ $env.META_AD_ACCOUNT_ID }}/adsets`
     - Query Parameters: `access_token`, `fields` (`id,name,status,effective_status,daily_budget,lifetime_budget,start_time,stop_time`), `limit` (`500`).
   - **Node 5 (`Fetch campaign insights (Meta)`):**
     - URL: `=https://graph.facebook.com/{{ $env.META_API_VERSION || 'v21.0' }}/{{ $env.META_AD_ACCOUNT_ID }}/insights`
     - Query Parameters: `access_token`, `level` (`campaign`), `date_preset`, `fields` (`campaign_id,campaign_name,spend,impressions,frequency`), `limit` (`500`).
   - **Node 6 (`Fetch campaign entities (Meta)`):**
     - URL: `=https://graph.facebook.com/{{ $env.META_API_VERSION || 'v21.0' }}/{{ $env.META_AD_ACCOUNT_ID }}/campaigns`
     - Query Parameters: `access_token`, `fields` (`id,name,status,effective_status,daily_budget,lifetime_budget,start_time,stop_time`), `limit` (`500`).
   - *Connection Chain:* `Set config: winners` -> `Fetch account benchmark (Meta)` -> `Fetch ad insights (Meta)` -> `Fetch ad set insights (Meta)` -> `Fetch ad set entities (Meta)` -> `Fetch campaign insights (Meta)` -> `Fetch campaign entities (Meta)`.

4. **Add Data Processing & AI Integration Nodes:**
   - **Node 7 (`Rank winners and size the lift`):** Add a Code node to process all upstream datasets, compute benchmarks, detect budget caps, and output ranked winning ads. Connect `Fetch campaign entities (Meta)` to this node.
   - **Node 8 (`Build AI scale prompt`):** Add a Code node to structure the prompt, system instructions, and JSON response format constraints for Claude. Connect `Rank winners and size the lift` to this node.
   - **Node 9 (`Write scale plan with Claude AI`):** Add an HTTP Request node configured as follows:
     - Method: POST
     - URL: `https://api.anthropic.com/v1/messages`
     - Headers: `x-api-key` (`={{ $env.ANTHROPIC_API_KEY }}`), `anthropic-version` (`2023-06-01`), `content-type` (`application/json`).
     - Body: Send JSON specified as `={{ $json.body }}`.
     - Connect `Build AI scale prompt` to this node.

5. **Add Reporting and Delivery Nodes:**
   - **Node 10 (`Print scale plan`):** Add a Code node to parse the AI output and compile the final text report string. Connect `Write scale plan with Claude AI` to this node.
   - **Node 11 (`Send a message`):** Add a Gmail node (`n8n-nodes-base.gmail`).
     - Parameters: Set Message to `={{ $json.report }}`.
     - Credentials: Configure valid Gmail OAuth2 credentials.
     - Connect `Print scale plan` to `Send a message`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Built by nocode.expert - done-for-you automation & tracking. | Project credits and external resource link: https://nocode.expert |