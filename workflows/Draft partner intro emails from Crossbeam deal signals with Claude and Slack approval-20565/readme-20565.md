Draft partner intro emails from Crossbeam deal signals with Claude and Slack approval

https://n8nworkflows.xyz/workflows/draft-partner-intro-emails-from-crossbeam-deal-signals-with-claude-and-slack-approval-20565


# Draft partner intro emails from Crossbeam deal signals with Claude and Slack approval

### 1. Workflow Overview

This workflow automates the process of identifying business opportunities from Crossbeam partner deal signals, matching them to relevant Salesforce accounts, drafting partner introduction emails using Anthropic Claude with Crossbeam MCP context, routing them through Slack for human review, and executing outreach via Gmail along with Salesforce task logging.

The execution logic is organized into the following functional blocks:
- **1.1 Start and Settings:** Establishes configuration variables and handles entry points (scheduled weekday execution or incoming webhooks).
- **1.2 Signal Ingestion & Verification:** Handles webhook signature validation or polls the Crossbeam API for account signals.
- **1.3 Filtering & Deduplication:** Normalizes signal fields, applies scope/lookback filters, removes duplicates, and filters out previously handled events via an n8n Data Table.
- **1.4 Salesforce Account Matching:** Aggregates account IDs, builds dynamic SOQL queries, and queries Salesforce in batches to retrieve historical opportunity context.
- **1.5 Data Merging & Capping:** Joins signals to Salesforce account records, sorts by recency, removes duplicate pairings, and caps processing volume per run.
- **1.6 AI Processing & Drafting:** Leverages an Anthropic Claude agent equipped with read-only Crossbeam MCP tools to research partner ownership, retrieve shared contacts, and generate structured intro drafts.
- **1.7 Human Review in Slack:** Posts context briefs to Slack and uses interactive form elements to collect user approvals, edits, or skip decisions.
- **1.8 Action, Logging & Reporting:** Executes Gmail sends/drafts, logs Salesforce follow-up tasks, records execution outcomes in the Data Table, and updates the Slack thread.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Start and Settings
**Overview:** This block initializes global workflow settings and routes execution based on whether the trigger originated from a scheduled cron event or an external webhook.

**Nodes Involved:**
- `Every weekday at 08:45`
- `Receive Crossbeam webhook`
- `Configure me`
- `Webhook or schedule?`

**Node Details:**
1. **Every weekday at 08:45**
   - *Type & Technical Role:* `n8n-nodes-base.scheduleTrigger` (Trigger)
   - *Configuration:* Cron expression `45 8 * * 1-5`.
   - *Connections:* Outputs to `Configure me`.
   - *Failure Modes:* Missed executions if n8n instance is offline.
2. **Receive Crossbeam webhook**
   - *Type & Technical Role:* `n8n-nodes-base.webhook` (Webhook Trigger)
   - *Configuration:* HTTP POST method on path `crossbeam-partner-intro-requests` with raw body capture enabled.
   - *Connections:* Outputs to `Configure me`.
   - *Failure Modes:* Network routing or endpoint misconfiguration.
3. **Configure me**
   - *Type & Technical Role:* `n8n-nodes-base.set` (Data Transformation / Settings Store)
   - *Configuration:* Defines global assignment variables including `crossbeam_org_uuid`, `crossbeam_api_base_url`, `signal_lookback_days`, `event_types`, `match_mode`, `salesforce_base_url`, `slack_channel`, `send_mode`, `dry_run`, and `webhook_signing_secret`.
   - *Connections:* Input from triggers; output to `Webhook or schedule?`.
4. **Webhook or schedule?**
   - *Type & Technical Role:* `n8n-nodes-base.switch` (Flow Control)
   - *Configuration:* Evaluates whether `$json.headers` is defined to route webhook invocations versus scheduled runs.
   - *Connections:* Input from `Configure me`; output 0 (`Webhook`) to `Verify webhook signature`, output 1 (`Schedule`) to `Fetch partner deal signals`.

---

#### Block 1.2: Signal Ingestion & Verification
**Overview:** This block verifies HMAC signatures and timestamps for incoming webhooks or polls the Crossbeam Signals API for account event data.

**Nodes Involved:**
- `Verify webhook signature`
- `Signature valid?`
- `Acknowledge webhook (200)`
- `Reject webhook (401)`
- `Fetch partner deal signals`
- `One item per signal`

**Node Details:**
1. **Verify webhook signature**
   - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript Execution)
   - *Configuration:* Validates `X-Crossbeam-Signature-256` and `X-Crossbeam-Timestamp` headers against `webhook_signing_secret` using crypto HMAC-SHA256. Checks timestamp age against `max_timestamp_age_seconds`.
   - *Connections:* Input from `Webhook or schedule?` (Webhook path); output to `Signature valid?`.
   - *Failure Modes:* Crypto errors, missing headers, or stale timestamps throwing validation failures.
2. **Signature valid?**
   - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional Branching)
   - *Configuration:* Checks boolean value of `$json.valid`.
   - *Connections:* Input from `Verify webhook signature`; true branch to `Acknowledge webhook (200)`, false branch to `Reject webhook (401)`.
3. **Acknowledge webhook (200)**
   - *Type & Technical Role:* `n8n-nodes-base.respondToWebhook` (Webhook Response)
   - *Configuration:* Returns HTTP 200 with body `{"received": true}`.
   - *Connections:* Input from `Signature valid?`; output to `Shape signal fields`.
4. **Reject webhook (401)**
   - *Type & Technical Role:* `n8n-nodes-base.respondToWebhook` (Webhook Response)
   - *Configuration:* Returns HTTP 401 with error reason JSON string.
   - *Connections:* Input from `Signature valid?`.
5. **Fetch partner deal signals**
   - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (API Request)
   - *Configuration:* GET request to `{{ $json.crossbeam_api_base_url }}/v1/signals/accounts?limit=1000` with pagination handling (`next_href`, max 60 requests) and Crossbeam OAuth2 authentication.
   - *Connections:* Input from `Webhook or schedule?` (Schedule path); output to `One item per signal`.
   - *Credentials:* `genericCredentialType` (`oAuth2Api`).
   - *Failure Modes:* API rate limits, expired OAuth tokens, or timeout limits.
6. **One item per signal**
   - *Type & Technical Role:* `n8n-nodes-base.splitOut` (Data Transformation)
   - *Configuration:* Splits array field `data` into individual n8n items.
   - *Connections:* Input from `Fetch partner deal signals`; output to `Shape signal fields`.

---

#### Block 1.3: Filtering & Deduplication
**Overview:** This block normalizes raw signal fields, applies lookback window and event type filters, and removes duplicate or previously processed events using an n8n Data Table.

**Nodes Involved:**
- `Shape signal fields`
- `Keep fresh, in-scope signals`
- `Remove duplicate events`
- `Skip events already handled`

**Node Details:**
1. **Shape signal fields**
   - *Type & Technical Role:* `n8n-nodes-base.set` (Data Transformation)
   - *Configuration:* Maps raw payload properties to standard fields: `event_key`, `event_type`, `triggered_at`, `source`, `partner_name`, `account_name`, `account_domain`, `sfdc_account_id`, `sfdc_id15`, `partner_deal_name`, `partner_deal_stage`, `partner_deal_status`, and `partner_deal_contacts`.
   - *Connections:* Input from `Acknowledge webhook (200)` or `One item per signal`; output to `Keep fresh, in-scope signals`.
2. **Keep fresh, in-scope signals**
   - *Type & Technical Role:* `n8n-nodes-base.filter` (Conditional Filtering)
   - *Configuration:* Validates that `sfdc_account_id` is present, event timestamp is within `signal_lookback_days`, `event_type` matches configured list, `partner_name` passes filter, and `partner_deal_status` is not `closed lost`.
   - *Connections:* Input from `Shape signal fields`; output to `Remove duplicate events`.
3. **Remove duplicate events**
   - *Type & Technical Role:* `n8n-nodes-base.removeDuplicates` (Deduplication)
   - *Configuration:* Compares selected field `event_key`.
   - *Connections:* Input from `Keep fresh, in-scope signals`; output to `Skip events already handled`.
4. **Skip events already handled**
   - *Type & Technical Role:* `n8n-nodes-base.dataTable` (Database Operation)
   - *Configuration:* Operation `rowNotExists` checking Data Table rows where `event_key` matches and `status` is not `dry_run`.
   - *Connections:* Input from `Remove duplicate events`; outputs to `Join signals to accounts` (index 0) and `Batches of 40 accounts` (index 0).
   - *Failure Modes:* Unconfigured Data Table ID.

---

#### Block 1.4: Salesforce Account Matching
**Overview:** This block aggregates account IDs from filtered signals, constructs dynamic SOQL queries based on matching criteria, and queries Salesforce in batches to retrieve account and opportunity context.

**Nodes Involved:**
- `Batches of 40 accounts`
- `Collect Account Ids`
- `Build SOQL query`
- `Find matching Salesforce accounts`
- `Keep accounts found`
- `Shape Salesforce account`

**Node Details:**
1. **Batches of 40 accounts**
   - *Type & Technical Role:* `n8n-nodes-base.splitInBatches` (Flow Control)
   - *Configuration:* Batch size set to `40`.
   - *Connections:* Input from `Skip events already handled`; output 0 to `Keep accounts found`, output 1 to `Collect Account Ids`.
2. **Collect Account Ids**
   - *Type & Technical Role:* `n8n-nodes-base.aggregate` (Data Transformation)
   - *Configuration:* Aggregates field `sfdc_account_id`.
   - *Connections:* Input from `Batches of 40 accounts`; output to `Build SOQL query`.
3. **Build SOQL query**
   - *Type & Technical Role:* `n8n-nodes-base.set` (Data Transformation)
   - *Configuration:* Dynamically builds SOQL query string depending on `match_mode` (`closed_lost` vs `owned_accounts`), handling lookback dates, loss reasons, and subqueries.
   - *Connections:* Input from `Collect Account Ids`; output to `Find matching Salesforce accounts`.
4. **Find matching Salesforce accounts**
   - *Type & Technical Role:* `n8n-nodes-base.salesforce` (CRM Integration)
   - *Configuration:* Executes SOQL search query via Salesforce REST API.
   - *Connections:* Input from `Build SOQL query`; output to `Keep accounts found`.
   - *Credentials:* Salesforce OAuth2 API.
   - *Failure Modes:* SOQL syntax errors, invalid API credentials, or API limits.
5. **Keep accounts found**
   - *Type & Technical Role:* `n8n-nodes-base.filter` (Conditional Filtering)
   - *Configuration:* Asserts `Id` is not empty.
   - *Connections:* Input from `Find matching Salesforce accounts`; output to `Shape Salesforce account`.
6. **Shape Salesforce account**
   - *Type & Technical Role:* `n8n-nodes-base.set` (Data Transformation)
   - *Configuration:* Maps CRM fields including `sfdc_id15`, `our_account_name`, `owner_id`, `owner_name`, `owner_email`, `lost_opp_name`, `lost_opp_close_date`, `lost_opp_amount`, `loss_reason`, and `account_url`.
   - *Connections:* Input from `Keep accounts found`; output to `Join signals to accounts` (index 1).

---

#### Block 1.5: Data Merging & Capping
**Overview:** This block joins filtered partner signals with their corresponding Salesforce account context, sorts by recency, removes duplicate combinations, and limits execution volume per run.

**Nodes Involved:**
- `Join signals to accounts`
- `Newest signals first`
- `One request per account and partner`
- `Cap requests per run`

**Node Details:**
1. **Join signals to accounts**
   - *Type & Technical Role:* `n8n-nodes-base.merge` (Data Combination)
   - *Configuration:* Mode `combine` matching string `sfdc_id15`.
   - *Connections:* Inputs from `Skip events already handled` (index 0) and `Shape Salesforce account` (index 1); output to `Newest signals first`.
2. **Newest signals first**
   - *Type & Technical Role:* `n8n-nodes-base.sort` (Data Transformation)
   - *Configuration:* Sorts items by `triggered_at` in descending order.
   - *Connections:* Input from `Join signals to accounts`; output to `One request per account and partner`.
3. **One request per account and partner**
   - *Type & Technical Role:* `n8n-nodes-base.removeDuplicates` (Deduplication)
   - *Configuration:* Compares selected fields `sfdc_id15` and `partner_name`.
   - *Connections:* Input from `Newest signals first`; output to `Cap requests per run`.
4. **Cap requests per run**
   - *Type & Technical Role:* `n8n-nodes-base.limit` (Flow Control)
   - *Configuration:* Limits items based on expression `{{ Number($('Configure me').first().json.max_items_per_run) || 5 }}`.
   - *Connections:* Input from `One request per account and partner`; output to `Research and draft intro request`.

---

#### Block 1.6: AI Processing & Drafting
**Overview:** This block invokes an Anthropic Claude agent equipped with Crossbeam MCP tools to research account ownership and shared contacts, resulting in a structured JSON brief and drafted intro email.

**Nodes Involved:**
- `Research and draft intro request`
- `Claude Sonnet`
- `Crossbeam MCP (read only)`
- `Intro request JSON`
- `Combine context and draft`
- `Keep drafts above confidence bar`

**Node Details:**
1. **Research and draft intro request**
   - *Type & Technical Role:* `@n8n/n8n-nodes-langchain.agent` (AI Agent)
   - *Configuration:* Defines prompt input combining company, Salesforce account data, account owner, loss details, and partner deal context. Enforces a maximum of 6 iterations and a system message instructing the agent to use Crossbeam MCP tools (max 3 tool calls) to find partner account owners and shared contacts.
   - *Connections:* Inputs from `Cap requests per run`, `Claude Sonnet`, `Crossbeam MCP (read only)`, and `Intro request JSON`; output to `Combine context and draft`.
   - *Failure Modes:* Tool timeout or API quota exhaustion.
2. **Claude Sonnet**
   - *Type & Technical Role:* `@n8n/n8n-nodes-langchain.lmChatAnthropic` (Language Model)
   - *Configuration:* Model set to `claude-sonnet-5` with max tokens `2048`.
   - *Connections:* Output to `Research and draft intro request`.
   - *Credentials:* Anthropic API Key.
3. **Crossbeam MCP (read only)**
   - *Type & Technical Role:* `@n8n/n8n-nodes-langchain.mcpClientTool` (MCP Tool Integration)
   - *Configuration:* Endpoint URL `https://mcp.crossbeam.com/mcp` with tools `find_overlapping_partners`, `find_partner_shared_contacts`, and `get_account_context`. Timeout set to 120,000ms.
   - *Connections:* Output to `Research and draft intro request`.
   - *Credentials:* MCP OAuth2 API.
4. **Intro request JSON**
   - *Type & Technical Role:* `@n8n/n8n-nodes-langchain.outputParserStructured` (Output Parser)
   - *Configuration:* Enforces JSON schema containing fields `brief`, `partner_owner_name`, `partner_owner_email`, `email_subject`, `email_body`, and `confidence`.
   - *Connections:* Output to `Research and draft intro request`.
5. **Combine context and draft**
   - *Type & Technical Role:* `n8n-nodes-base.set` (Data Transformation)
   - *Configuration:* Raw mode merging original item data with the AI output (`$json.output`).
   - *Connections:* Input from `Research and draft intro request`; output to `Keep drafts above confidence bar`.
6. **Keep drafts above confidence bar**
   - *Type & Technical Role:* `n8n-nodes-base.filter` (Conditional Filtering)
   - *Configuration:* Compares confidence level (`high: 3, medium: 2, low: 1`) against `min_confidence` configured in `Configure me`.
   - *Connections:* Input from `Combine context and draft`; output to `Review one draft at a time`.

---

#### Block 1.7: Human Review in Slack
**Overview:** This block iterates through drafted emails one by one, posts a rich context brief to Slack, and triggers an interactive review form in a message thread.

**Nodes Involved:**
- `Review one draft at a time`
- `Post brief to Slack`
- `Ask for approval in thread`
- `Read reviewer decision`

**Node Details:**
1. **Review one draft at a time**
   - *Type & Technical Role:* `n8n-nodes-base.splitInBatches` (Flow Control)
   - *Configuration:* Processes items individually (batch size default).
   - *Connections:* Input from `Keep drafts above confidence bar` and `Reply in thread with outcome`; output 1 to `Post brief to Slack`.
2. **Post brief to Slack**
   - *Type & Technical Role:* `n8n-nodes-base.slack` (Messaging Integration)
   - *Configuration:* Posts formatted markdown text detailing account info, partner signal, loss history, ownership, and confidence score to the designated `slack_channel`.
   - *Connections:* Input from `Review one draft at a time`; output to `Ask for approval in thread`.
   - *Credentials:* Slack OAuth2 API.
   - *Failure Modes:* Invalid Slack channel ID or missing `chat:write` scopes.
3. **Ask for approval in thread**
   - *Type & Technical Role:* `n8n-nodes-base.slack` (Interactive Messaging / Wait Operation)
   - *Configuration:* Operation `sendAndWait`. Sends threaded message with custom form fields (`decision`: dropdown [Approve, Skip], `send_to`, `email_subject`, `email_body`). Configures wait time limit based on `approval_wait_hours`.
   - *Connections:* Input from `Post brief to Slack`; output to `Read reviewer decision`.
   - *Credentials:* Slack OAuth2 API.
   - *Failure Modes:* Timeout expiry without response, or missing Interactivity configuration on the Slack app.
4. **Read reviewer decision**
   - *Type & Technical Role:* `n8n-nodes-base.set` (Data Transformation)
   - *Configuration:* Extracts reviewer inputs from form response (`decision`, `send_to`, `email_subject`, `email_body`, and `dry_run`).
   - *Connections:* Input from `Ask for approval in thread`; output to `Route reviewer decision`.

---

#### Block 1.8: Action, Logging & Reporting
**Overview:** This block routes the reviewer's decision, executes Gmail sends or drafts when approved (and not in dry-run mode), logs follow-up tasks in Salesforce, records outcomes to the Data Table, and posts confirmation replies back to the Slack thread.

**Nodes Involved:**
- `Route reviewer decision`
- `Status: skipped`
- `Status: dry run`
- `Send now or save draft?`
- `Send intro email (Gmail)`
- `Save intro email as Gmail draft`
- `Log a Salesforce task?`
- `Log task for our account owner`
- `Status: sent or drafted`
- `Status: no response`
- `Record outcome`
- `Reply in thread with outcome`

**Node Details:**
1. **Route reviewer decision**
   - *Type & Technical Role:* `n8n-nodes-base.switch` (Flow Control)
   - *Configuration:* Evaluates decision type: `Skipped`, `Approved, dry run`, `Approved, live`, with fallback for no response.
   - *Connections:* Input from `Read reviewer decision`; outputs to `Status: skipped`, `Status: dry run`, `Send now or save draft?`, and `Status: no response`.
2. **Status: skipped**
   - *Type & Technical Role:* `n8n-nodes-base.set` (Data Transformation)
   - *Configuration:* Sets status variable to `skipped`.
   - *Connections:* Input from `Route reviewer decision`; output to `Record outcome`.
3. **Status: dry run**
   - *Type & Technical Role:* `n8n-nodes-base.set` (Data Transformation)
   - *Configuration:* Sets status variable to `dry_run`.
   - *Connections:* Input from `Route reviewer decision`; output to `Record outcome`.
4. **Send now or save draft?**
   - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional Branching)
   - *Configuration:* Evaluates if `send_mode` equals `send`.
   - *Connections:* Input from `Route reviewer decision`; true branch to `Send intro email (Gmail)`, false branch to `Save intro email as Gmail draft`.
5. **Send intro email (Gmail)**
   - *Type & Technical Role:* `n8n-nodes-base.gmail` (Email Integration)
   - *Configuration:* Sends text email to `send_to` with subject and body.
   - *Connections:* Input from `Send now or save draft?`; output to `Log a Salesforce task?`.
   - *Credentials:* Gmail OAuth2 API.
   - *Failure Modes:* Invalid recipient address, Gmail API limits, or authentication revocation.
6. **Save intro email as Gmail draft**
   - *Type & Technical Role:* `n8n-nodes-base.gmail` (Email Integration)
   - *Configuration:* Resource `draft`, saving message to mailbox.
   - *Connections:* Input from `Send now or save draft?`; output to `Log a Salesforce task?`.
   - *Credentials:* Gmail OAuth2 API.
7. **Log a Salesforce task?**
   - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional Branching)
   - *Configuration:* Evaluates boolean `create_sfdc_task`.
   - *Connections:* Input from Gmail nodes; true branch to `Log task for our account owner`, false branch to `Status: sent or drafted`.
8. **Log task for our account owner**
   - *Type & Technical Role:* `n8n-nodes-base.salesforce` (CRM Integration)
   - *Configuration:* Creates task resource with `Status: Not Started`, assigned to account owner `owner_id`, linked to `sfdc_account_id` via `whatId`, with due date 3 days out.
   - *Connections:* Input from `Log a Salesforce task?`; output to `Status: sent or drafted`.
   - *Credentials:* Salesforce OAuth2 API.
9. **Status: sent or drafted**
   - *Type & Technical Role:* `n8n-nodes-base.set` (Data Transformation)
   - *Configuration:* Sets status to `sent` or `drafted` based on `send_mode`.
   - *Connections:* Inputs from `Log task for our account owner` and `Log a Salesforce task?` (false branch); output to `Record outcome`.
10. **Status: no response**
    - *Type & Technical Role:* `n8n-nodes-base.set` (Data Transformation)
    - *Configuration:* Sets status to `no_response`.
    - *Connections:* Input from `Route reviewer decision` (fallback); output to `Record outcome`.
11. **Record outcome**
    - *Type & Technical Role:* `n8n-nodes-base.dataTable` (Database Operation)
    - *Configuration:* Writes outcome row to Data Table (`event_key`, `status`, `partner_name`, `account_name`, `sfdc_account_id`, `partner_owner_email`, `decided_at`).
    - *Connections:* Inputs from status set nodes; output to `Reply in thread with outcome`.
12. **Reply in thread with outcome**
    - *Type & Technical Role:* `n8n-nodes-base.slack` (Messaging Integration)
    - *Configuration:* Posts threaded reply summarizing action outcome (sent, drafted, dry run, skipped, or no response) to `slack_channel`.
    - *Connections:* Input from `Record outcome`; output to `Review one draft at a time`.
    - *Credentials:* Slack OAuth2 API.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Configure me** | `n8n-nodes-base.set` | Holds user settings | `Every weekday at 08:45`, `Receive Crossbeam webhook` | `Webhook or schedule?` | 1. Start and settings |
| **Every weekday at 08:45** | `n8n-nodes-base.scheduleTrigger` | Scheduled cron trigger | None | `Configure me` | 1. Start and settings |
| **Receive Crossbeam webhook** | `n8n-nodes-base.webhook` | Webhook HTTP entry point | None | `Configure me` | 1. Start and settings |
| **Webhook or schedule?** | `n8n-nodes-base.switch` | Routes execution path | `Configure me` | `Verify webhook signature`, `Fetch partner deal signals` | 1. Start and settings |
| **Verify webhook signature** | `n8n-nodes-base.code` | HMAC & timestamp validation | `Webhook or schedule?` | `Signature valid?` | 2a. Webhook check (Enterprise) |
| **Signature valid?** | `n8n-nodes-base.if` | Condition check on signature | `Verify webhook signature` | `Acknowledge webhook (200)`, `Reject webhook (401)` | 2a. Webhook check (Enterprise) |
| **Acknowledge webhook (200)** | `n8n-nodes-base.respondToWebhook` | HTTP 200 response | `Signature valid?` | `Shape signal fields` | 2a. Webhook check (Enterprise) |
| **Reject webhook (401)** | `n8n-nodes-base.respondToWebhook` | HTTP 401 response | `Signature valid?` | None | 2a. Webhook check (Enterprise) |
| **Fetch partner deal signals** | `n8n-nodes-base.httpRequest` | Calls Crossbeam Signals API | `Webhook or schedule?` | `One item per signal` | 2b. Poll partner deal signals |
| **One item per signal** | `n8n-nodes-base.splitOut` | Splits API array into items | `Fetch partner deal signals` | `Shape signal fields` | 2b. Poll partner deal signals |
| **Shape signal fields** | `n8n-nodes-base.set` | Normalizes signal properties | `Acknowledge webhook (200)`, `One item per signal` | `Keep fresh, in-scope signals` | 3. Filter and dedupe |
| **Keep fresh, in-scope signals** | `n8n-nodes-base.filter` | Filters by lookback & criteria | `Shape signal fields` | `Remove duplicate events` | 3. Filter and dedupe |
| **Remove duplicate events** | `n8n-nodes-base.removeDuplicates` | Deduplicates by `event_key` | `Keep fresh, in-scope signals` | `Skip events already handled` | 3. Filter and dedupe |
| **Skip events already handled** | `n8n-nodes-base.dataTable` | Filters out processed events | `Remove duplicate events` | `Join signals to accounts`, `Batches of 40 accounts` | 3. Filter and dedupe |
| **Batches of 40 accounts** | `n8n-nodes-base.splitInBatches` | Batches account processing | `Skip events already handled` | `Keep accounts found`, `Collect Account Ids` | 4. Match in Salesforce (read only) |
| **Collect Account Ids** | `n8n-nodes-base.aggregate` | Aggregates IDs for SOQL | `Batches of 40 accounts` | `Build SOQL query` | 4. Match in Salesforce (read only) |
| **Build SOQL query** | `n8n-nodes-base.set` | Constructs dynamic SOQL | `Collect Account Ids` | `Find matching Salesforce accounts` | 4. Match in Salesforce (read only) |
| **Find matching Salesforce accounts** | `n8n-nodes-base.salesforce` | Queries Salesforce via SOQL | `Build SOQL query` | `Keep accounts found` | 4. Match in Salesforce (read only) |
| **Keep accounts found** | `n8n-nodes-base.filter` | Asserts account ID existence | `Find matching Salesforce accounts` | `Shape Salesforce account` | 4. Match in Salesforce (read only) |
| **Shape Salesforce account** | `n8n-nodes-base.set` | Maps account/opp fields | `Keep accounts found` | `Join signals to accounts` | 4. Match in Salesforce (read only) |
| **Join signals to accounts** | `n8n-nodes-base.merge` | Joins signals with CRM data | `Skip events already handled`, `Shape Salesforce account` | `Newest signals first` | 5. Pick what to review |
| **Newest signals first** | `n8n-nodes-base.sort` | Sorts signals descending | `Join signals to accounts` | `One request per account and partner` | 5. Pick what to review |
| **One request per account and partner**| `n8n-nodes-base.removeDuplicates` | Deduplicates account/partner | `Newest signals first` | `Cap requests per run` | 5. Pick what to review |
| **Cap requests per run** | `n8n-nodes-base.limit` | Limits volume per execution | `One request per account and partner` | `Research and draft intro request` | 5. Pick what to review |
| **Research and draft intro request** | `@n8n/n8n-nodes-langchain.agent` | AI agent for research & draft | `Cap requests per run` | `Combine context and draft` | 6. Claude researches and drafts |
| **Claude Sonnet** | `@n8n/n8n-nodes-langchain.lmChatAnthropic`| LLM provider for agent | None | `Research and draft intro request` | 6. Claude researches and drafts |
| **Crossbeam MCP (read only)** | `@n8n/n8n-nodes-langchain.mcpClientTool` | MCP tool integration | None | `Research and draft intro request` | 6. Claude researches and drafts |
| **Intro request JSON** | `@n8n/n8n-nodes-langchain.outputParserStructured` | Structured JSON output parser | None | `Research and draft intro request` | 6. Claude researches and drafts |
| **Combine context and draft** | `n8n-nodes-base.set` | Merges context with draft | `Research and draft intro request` | `Keep drafts above confidence bar` | 6. Claude researches and drafts |
| **Keep drafts above confidence bar** | `n8n-nodes-base.filter` | Filters by confidence level | `Combine context and draft` | `Review one draft at a time` | 6. Claude researches and drafts |
| **Review one draft at a time** | `n8n-nodes-base.splitInBatches` | Iterates drafts sequentially | `Keep drafts above confidence bar`, `Reply in thread with outcome` | `Post brief to Slack` | 7. Human review in Slack |
| **Post brief to Slack** | `n8n-nodes-base.slack` | Posts account/signal brief | `Review one draft at a time` | `Ask for approval in thread` | 7. Human review in Slack |
| **Ask for approval in thread** | `n8n-nodes-base.slack` | Slack interactive approval form | `Post brief to Slack` | `Read reviewer decision` | 7. Human review in Slack |
| **Read reviewer decision** | `n8n-nodes-base.set` | Extracts reviewer form data | `Ask for approval in thread` | `Route reviewer decision` | 7. Human review in Slack |
| **Route reviewer decision** | `n8n-nodes-base.switch` | Routes based on form decision | `Read reviewer decision` | `Status: skipped`, `Status: dry run`, `Send now or save draft?`, `Status: no response` | 8. Act and log |
| **Status: skipped** | `n8n-nodes-base.set` | Sets status to skipped | `Route reviewer decision` | `Record outcome` | 8. Act and log |
| **Status: dry run** | `n8n-nodes-base.set` | Sets status to dry_run | `Route reviewer decision` | `Record outcome` | 8. Act and log |
| **Send now or save draft?** | `n8n-nodes-base.if` | Evaluates send mode | `Route reviewer decision` | `Send intro email (Gmail)`, `Save intro email as Gmail draft` | 8. Act and log |
| **Send intro email (Gmail)** | `n8n-nodes-base.gmail` | Sends email via Gmail API | `Send now or save draft?` | `Log a Salesforce task?` | 8. Act and log |
| **Save intro email as Gmail draft** | `n8n-nodes-base.gmail` | Saves email draft in Gmail | `Send now or save draft?` | `Log a Salesforce task?` | 8. Act and log |
| **Log a Salesforce task?** | `n8n-nodes-base.if` | Checks task creation setting | `Send intro email (Gmail)`, `Save intro email as Gmail draft` | `Log task for our account owner`, `Status: sent or drafted` | 8. Act and log |
| **Log task for our account owner** | `n8n-nodes-base.salesforce` | Creates Salesforce task | `Log a Salesforce task?` | `Status: sent or drafted` | 8. Act and log |
| **Status: sent or drafted** | `n8n-nodes-base.set` | Sets status to sent/drafted | `Log task for our account owner`, `Log a Salesforce task?` | `Record outcome` | 8. Act and log |
| **Status: no response** | `n8n-nodes-base.set` | Sets status to no_response | `Route reviewer decision` | `Record outcome` | 8. Act and log |
| **Record outcome** | `n8n-nodes-base.dataTable` | Records outcome in Data Table | `Status: skipped`, `Status: dry run`, `Status: sent or drafted`, `Status: no response` | `Reply in thread with outcome` | 8. Act and log |
| **Reply in thread with outcome** | `n8n-nodes-base.slack` | Replies to Slack thread | `Record outcome` | `Review one draft at a time` | 8. Act and log |

---

### 4. Reproducing the Workflow from Scratch

To rebuild this workflow manually in n8n without importing the JSON, follow these sequential steps:

1. **Initialize Core Nodes & Triggers**
   - Create a `Schedule Trigger` node configured for weekdays at 08:45 (`45 8 * * 1-5`). Name it `Every weekday at 08:45`.
   - Create a `Webhook` node with HTTP method `POST`, path `crossbeam-partner-intro-requests`, and raw body capture enabled. Name it `Receive Crossbeam webhook`.
   - Create a `Set` node named `Configure me`. Add assignment variables for configuration parameters (e.g., `crossbeam_org_uuid`, `crossbeam_api_base_url`, `signal_lookback_days`, `event_types`, `match_mode`, `salesforce_base_url`, `slack_channel`, `send_mode`, `dry_run`, `webhook_signing_secret`, etc.).
   - Connect both triggers to `Configure me`.

2. **Set Up Routing & Webhook Verification**
   - Create a `Switch` node named `Webhook or schedule?`. Connect `Configure me` to it. Define two output rules: one checking if `{{ $json.headers !== undefined }}` (Webhook) and one checking if `{{ $json.headers === undefined }}` (Schedule).
   - For the webhook path, add a `Code` node named `Verify webhook signature` to perform HMAC-SHA256 verification and timestamp checks.
   - Add an `If` node named `Signature valid?` (`{{ $json.valid }}`). Connect true output to a `Respond to Webhook` node named `Acknowledge webhook (200)` (Response Code 200, JSON body `{"received": true}`) and false output to `Respond to Webhook` named `Reject webhook (401)` (Response Code 401).
   - For the schedule path, add an `HTTP Request` node named `Fetch partner deal signals` (`GET` `{{ $('Configure me').first().json.crossbeam_api_base_url }}/v1/signals/accounts?limit=1000`) with Crossbeam OAuth2 credentials and pagination settings. Connect it to a `Split Out` node (`One item per signal`) splitting on field `data`.

3. **Filter and Deduplicate Signals**
   - Create a `Set` node named `Shape signal fields` to map raw fields (`event_key`, `event_type`, `triggered_at`, `partner_name`, `sfdc_account_id`, etc.). Connect inputs from `Acknowledge webhook (200)` and `One item per signal`.
   - Add a `Filter` node named `Keep fresh, in-scope signals` verifying account ID presence, lookback days, matching event types, partner filter, and excluding closed-lost statuses.
   - Add a `Remove Duplicates` node named `Remove duplicate events` comparing `event_key`.
   - Add a `Data Table` node named `Skip events already handled` configured with operation `rowNotExists` checking `event_key` and status not equal to `dry_run`.

4. **Salesforce Integration & Account Matching**
   - Add a `Split In Batches` node named `Batches of 40 accounts` with batch size `40`.
   - Connect batch output 1 to an `Aggregate` node (`Collect Account Ids`) aggregating `sfdc_account_id`.
   - Connect aggregate output to a `Set` node (`Build SOQL query`) generating dynamic SOQL queries based on `match_mode`.
   - Connect to a Salesforce node (`Find matching Salesforce accounts`) using SOQL search resource and Salesforce OAuth2 credentials.
   - Connect Salesforce output and batch output 0 to a `Filter` node (`Keep accounts found`) and a `Set` node (`Shape Salesforce account`) mapping CRM fields.

5. **Merge, Sort, and Cap Execution Volume**
   - Add a `Merge` node named `Join signals to accounts` combining signals and shaped accounts on `sfdc_id15`.
   - Add a `Sort` node (`Newest signals first`) sorting by `triggered_at` descending.
   - Add a `Remove Duplicates` node (`One request per account and partner`) comparing `sfdc_id15,partner_name`.
   - Add a `Limit` node (`Cap requests per run`) limiting items based on `max_items_per_run`.

6. **Configure AI Agent & MCP Tools**
   - Create an AI Agent node named `Research and draft intro request`.
   - Connect an Anthropic Chat Model node (`Claude Sonnet`) using Claude Sonnet 5 model and Anthropic API credentials.
   - Connect an MCP Client Tool node (`Crossbeam MCP (read only)`) using endpoint `https://mcp.crossbeam.com/mcp`, tools `find_overlapping_partners`, `find_partner_shared_contacts`, and `get_account_context`, with MCP OAuth2 credentials.
   - Connect a Structured Output Parser node (`Intro request JSON`) with the required JSON schema example.
   - Connect the agent to a `Set` node (`Combine context and draft`) and a `Filter` node (`Keep drafts above confidence bar`) checking minimum confidence thresholds.

7. **Slack Human Review Workflow**
   - Add a `Split In Batches` node named `Review one draft at a time`.
   - Add a Slack node (`Post brief to Slack`) posting the markdown summary brief to `slack_channel` using Slack OAuth2 credentials.
   - Add a Slack node (`Ask for approval in thread`) configured with operation `sendAndWait`, response type `customForm`, form fields for decision, send to, subject, and body, and resume timeout set to `approval_wait_hours`.
   - Add a `Set` node (`Read reviewer decision`) to parse form submission data.

8. **Action, Logging & Reporting**
   - Add a `Switch` node (`Route reviewer decision`) routing by `Skipped`, `Approved, dry run`, `Approved, live`, and fallback.
   - For live approvals, add an `If` node (`Send now or save draft?`) checking `send_mode`.
   - Add Gmail nodes (`Send intro email (Gmail)` or `Save intro email as Gmail draft`) using Gmail OAuth2 credentials.
   - Add an `If` node (`Log a Salesforce task?`) checking `create_sfdc_task`.
   - Add a Salesforce node (`Log task for our account owner`) creating a task resource assigned to the account owner.
   - Add `Set` nodes for status tracking (`Status: skipped`, `Status: dry run`, `Status: sent or drafted`, `Status: no response`).
   - Add a `Data Table` node (`Record outcome`) writing execution results.
   - Add a Slack node (`Reply in thread with outcome`) posting confirmation replies to the Slack thread, looping back to `Review one draft at a time`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Crossbeam REST API Documentation | [Crossbeam REST API Guide](https://help.crossbeam.com/en/articles/4677142-rest-api) |
| Crossbeam API Reference & Scopes | [Crossbeam Developers](https://developers.crossbeam.com/) |
| Crossbeam Signals API & Webhooks | [Signals API Documentation](https://help.crossbeam.com/en/articles/12732223-getting-started-with-signals-how-to-access-real-time-partner-data-via-api-and-webhooks) |
| Crossbeam MCP Server Availability | [Crossbeam MCP Server Guide](https://help.crossbeam.com/en/articles/12601327-crossbeam-mcp-server-limited-availability) |