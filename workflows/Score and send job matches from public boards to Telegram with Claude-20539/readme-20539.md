Score and send job matches from public boards to Telegram with Claude

https://n8nworkflows.xyz/workflows/score-and-send-job-matches-from-public-boards-to-telegram-with-claude-20539


# Score and send job matches from public boards to Telegram with Claude

### 1. Workflow Overview

This workflow automates the discovery, filtering, evaluation, and delivery of job postings from multiple public company boards (Ashby, Greenhouse, Lever) and open job aggregators (Jobicy, Himalayas). It uses Anthropic Claude to score each filtered job opportunity against a user-defined candidate profile and delivers matches and fallback unjudged roles directly to Telegram.

The functional execution is organized into the following logical blocks:
- **1.1 Trigger and Configuration Setup:** Initializes execution daily at 09:00 or via manual input, loads user profile settings, and validates foundational configuration parameters.
- **1.2 Data Ingestion & Normalization:** Fetches company board APIs and aggregator feeds concurrently, normalizes heterogeneous data structures into a unified format, merges the inputs, and removes duplicate openings.
- **1.3 Pre-Filtering & Deduplication:** Applies deterministic filtering for geographical regions, target role titles, exclusion of junior roles, and filters out jobs processed in previous executions, while capping the total number of items per run.
- **1.4 AI Evaluation:** Evaluates candidate fit via an Anthropic Claude language model using structured JSON outputs containing fit status, a 1–10 score, and a concise decision reason.
- **1.5 Notification & Delivery:** Compiles a run summary and pushes formatted, individual HTML job cards to a designated Telegram chat channel.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Trigger and Configuration Setup
- **Overview:** Initializes the workflow execution on a daily schedule or manual prompt, sets parameters via a centralized variable block, and performs validation to prevent runtime failures due to missing configurations.
- **Nodes Involved:** `Every day 09:00`, `Run now (manual)`, `Configure me`, `Check settings`.
- **Node Details:**
  - **`Every day 09:00`**
    - *Type and technical role:* `Schedule Trigger` node. Fires automatically every day at 09:00 system time.
    - *Configuration choices:* Interval set to days at hour 9.
    - *Input/Output:* No inputs; outputs execution flow to `Configure me`.
    - *Edge cases:* System timezone configuration mismatches.
  - **`Run now (manual)`**
    - *Type and technical role:* `Manual Trigger` node. Allows on-demand execution of the workflow.
    - *Input/Output:* No inputs; outputs execution flow to `Configure me`.
  - **`Configure me`**
    - *Type and technical role:* `Set` (Edit Fields) node. Stores global operational variables.
    - *Configuration choices:* Sets `telegramChatId`, `allowedRegion`, `minScore`, `maxPerRun`, `boards`, `useAggregators`, `feedKeywords`, and `profile`.
    - *Input/Output:* Inputs from triggers; outputs to `Check settings`.
    - *Edge cases:* Leaving placeholder values (`YOUR_TELEGRAM_CHAT_ID`) unconfigured.
  - **`Check settings`**
    - *Type and technical role:* `Code` node. Validates parameters before proceeding.
    - *Configuration choices:* Throws an explicit JavaScript error if `telegramChatId` contains placeholders or if both company boards and aggregators are disabled.
    - *Input/Output:* Input from `Configure me`; output to `Boards to watch` and `Feeds to read`.
    - *Edge cases:* Stops execution cleanly with a descriptive error message if misconfigured.

---

#### 1.2 Data Ingestion & Normalization
- **Overview:** Transforms board and feed definitions into HTTP requests, fetches raw job board schemas, normalizes structural differences across various ATS providers, merges results, and eliminates duplicate items.
- **Nodes Involved:** `Boards to watch`, `Fetch company board`, `Normalize board postings`, `Feeds to read`, `Fetch feed`, `Normalize feed postings`, `Merge sources`, `One per opening`.
- **Node Details:**
  - **`Boards to watch`**
    - *Type and technical role:* `Code` node. Parses the comma-separated `boards` string configuration into structured target URLs.
    - *Key expressions:* Iterates over `String(cfg.boards || '').split('\n')`.
    - *Input/Output:* Input from `Check settings`; output to `Fetch company board`.
  - **`Fetch company board`**
    - *Type and technical role:* `HTTP Request` node. Calls third-party public ATS endpoints.
    - *Configuration choices:* Uses dynamic URL expressions (`{{ $json.url }}`), 20-second timeout, handles errors by continuing regular output (`neverError: true`), retries on failure with max 2 tries.
    - *Input/Output:* Input from `Boards to watch`; output to `Normalize board postings`.
    - *Edge cases:* API downtime (handled via status checks); rate limiting.
  - **`Normalize board postings`**
    - *Type and technical role:* `Code` node. Sanitizes and maps raw responses from Ashby, Greenhouse, and Lever into a unified schema.
    - *Key expressions:* Extracts `company`, `title`, `location`, `remote`, `url`, `source`, `description`, and a unique deduplication `key`.
    - *Input/Output:* Input from `Fetch company board`; output to `Merge sources` (Input 0).
  - **`Feeds to read`**
    - *Type and technical role:* `Code` node. Generates external aggregator endpoints for Jobicy and Himalayas based on configured keywords and regional configurations.
    - *Input/Output:* Input from `Check settings`; output to `Fetch feed`.
  - **`Fetch feed`**
    - *Type and technical role:* `HTTP Request` node. Queries job feeds.
    - *Configuration choices:* Dynamic URL mapping, timeout configuration, batch interval of 1000ms.
    - *Input/Output:* Input from `Feeds to read`; output to `Normalize feed postings`.
  - **`Normalize feed postings`**
    - *Type and technical role:* `Code` node. Parses job aggregator payloads into the shared normalized schema, retaining necessary attribution (`via`).
    - *Input/Output:* Input from `Fetch feed`; output to `Merge sources` (Input 1).
  - **`Merge sources`**
    - *Type and technical role:* `Merge` node. Combines data streams from company boards and open feeds.
    - *Configuration choices:* Append/combiner mode.
    - *Input/Output:* Inputs from `Normalize board postings` and `Normalize feed postings`; output to `One per opening`.
  - **`One per opening`**
    - *Type and technical role:* `Remove Duplicates` node. Drops duplicate job entries based on the composite `key` field.
    - *Configuration choices:* Selected fields comparison using `key`.
    - *Input/Output:* Input from `Merge sources`; output to `Region check`.

---

#### 1.3 Pre-Filtering & Deduplication
- **Overview:** Applies low-cost text and logic filters to drop irrelevant geographic locations, title mismatches, and junior positions, filters out previously seen runs, and enforces maximum processing caps.
- **Nodes Involved:** `Region check`, `Role and location filter`, `New since last run`, `Cap per run`.
- **Node Details:**
  - **`Region check`**
    - *Type and technical role:* `Code` node. Evaluates whether a candidate can work from a specific job location using regex rules mapped to macro regions.
    - *Key expressions:* Evaluates location text against `allowedRegion` configuration.
    - *Input/Output:* Input from `One per opening`; output to `Role and location filter`.
  - **`Role and location filter`**
    - *Type and technical role:* `Filter` node. Executes structured programmatic conditions against job properties.
    - *Configuration choices:* Combines regular expression checks ensuring target senior titles match while excluding junior designations (e.g., "associate", "intern"), alongside matching `inRegion` boolean evaluation.
    - *Input/Output:* Input from `Region check`; output to `New since last run`.
  - **`New since last run`**
    - *Type and technical role:* `Remove Duplicates` node. Filters out items processed in prior workflow runs.
    - *Configuration choices:* Operation set to `removeItemsSeenInPreviousExecutions` tracking the `key` property.
    - *Input/Output:* Input from `Role and location filter`; output to `Cap per run`.
  - **`Cap per run`**
    - *Type and technical role:* `Limit` node. Restricts the number of items passed downstream to control LLM API consumption costs.
    - *Key expressions:* Max items bound to `{{ $('Configure me').first().json.maxPerRun }}`.
    - *Input/Output:* Input from `New since last run`; output to `Judge fit (LLM)`.

---

#### 1.4 AI Evaluation
- **Overview:** Sends structured job descriptions to an Anthropic Claude language model, constrained by a strict profile evaluation prompt and forced to respond via an enforced JSON schema output parser.
- **Nodes Involved:** `Judge fit (LLM)`, `Claude Haiku 4.5 (chat model)`, `Verdict format (fit, score, reason)`.
- **Node Details:**
  - **`Judge fit (LLM)`**
    - *Type and technical role:* LangChain `Basic LLM Chain` node. Coordinates the prompt, language model, and schema validation.
    - *Configuration choices:* Uses system instructions combining candidate profile constraints with strict negative constraints on regional limitations. Error handling configured to continue regular output on failure.
    - *Input/Output:* Input from `Cap per run`; connects to AI Chat Model and Output Parser; outputs to `Apply verdict`.
    - *Edge cases:* LLM timeouts or token limit overages (handled gracefully by downstream parsing logic marking items unjudged).
  - **`Claude Haiku 4.5 (chat model)`**
    - *Type and technical role:* LangChain `Anthropic Chat Model` sub-node. Connects to the Anthropic API.
    - *Configuration choices:* Model ID `claude-haiku-4-5`, temperature `0`, max tokens `300`. Credentials: Anthropic account.
  - **`Verdict format (fit, score, reason)`**
    - *Type and technical role:* LangChain `Structured Output Parser` sub-node. Enforces programmatic JSON validation.
    - *Configuration choices:* Schema defining boolean `fit`, integer `score` (min 1, max 10), and string `reason`.

---

#### 1.5 Notification & Delivery
- **Overview:** Integrates the LLM assessment back into each job object, compiles execution summary metrics, and formats/dispatches Telegram messages and card notifications.
- **Nodes Involved:** `Apply verdict`, `Run summary`, `Send run summary to Telegram`, `Fits or unjudged?`, `Send job card to Telegram`.
- **Node Details:**
  - **`Apply verdict`**
    - *Type and technical role:* `Code` node. Merges raw posting data with AI evaluation attributes, formatting HTML markup strings for delivery.
    - *Input/Output:* Input from `Judge fit (LLM)`; outputs to `Run summary` and `Fits or unjudged?`.
  - **`Run summary`**
    - *Type and technical role:* `Code` node. Generates aggregated execution statistics for monitoring metrics.
    - *Input/Output:* Input from `Apply verdict`; output to `Send run summary to Telegram`.
  - **`Send run summary to Telegram`**
    - *Type and technical role:* `Telegram` node. Sends aggregate execution results.
    - *Configuration choices:* Chat ID configuration, HTML parse mode, web page previews disabled. Credentials: Telegram account.
  - **`Fits or unjudged?`**
    - *Type and technical role:* `If` node. Filters jobs meeting the minimum score threshold or fallback unjudged status.
    - *Configuration choices:* Condition checking `{{ $json.deliver }}` equals true.
    - *Input/Output:* Input from `Apply verdict`; output to `Send job card to Telegram`.
  - **`Send job card to Telegram`**
    - *Type and technical role:* `Telegram` node. Sends formatted individual job cards.
    - *Configuration choices:* HTML parse mode, payload derived from `{{ $json.card }}`. Credentials: Telegram account.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Template description` | `n8n-nodes-base.stickyNote` | Workflow documentation overview | None | None | Score new jobs from company boards and open job feeds against your profile with Claude and send matches to Telegram... |
| `Note: trigger` | `n8n-nodes-base.stickyNote` | Execution triggers documentation | None | None | ## 1. When and what Daily at 09:00 (instance timezone), plus a manual trigger for test runs... |
| `Note: fetch` | `n8n-nodes-base.stickyNote` | Company board fetching documentation | None | None | ## 2a. Company boards One HTTP call per line of `boards`, no auth... |
| `Note: filter` | `n8n-nodes-base.stickyNote` | Filtering logic documentation | None | None | ## 3. Cheap filters before the LLM **Region check** keeps postings whose location names your `allowedRegion`... |
| `Note: judge` | `n8n-nodes-base.stickyNote` | AI verdict documentation | None | None | ## 4. AI verdict Claude reads each posting against your profile and returns {fit, score, reason}... |
| `Note: deliver` | `n8n-nodes-base.stickyNote` | Telegram delivery documentation | None | None | ## 5. Telegram One summary line with the counts, then one card per job that fits... |
| `Note: feeds` | `n8n-nodes-base.stickyNote` | Aggregator feeds documentation | None | None | ## 2b. Open job feeds (no company list needed) When `useAggregators` is on, searches Jobicy and Himalayas... |
| `Every day 09:00` | `n8n-nodes-base.scheduleTrigger` | Daily automated workflow trigger | None | `Configure me` | |
| `Run now (manual)` | `n8n-nodes-base.manualTrigger` | On-demand manual workflow trigger | None | `Configure me` | |
| `Configure me` | `n8n-nodes-base.set` | Central configuration variables assignment | `Every day 09:00`, `Run now (manual)` | `Check settings` | |
| `Boards to watch` | `n8n-nodes-base.code` | Parses board configuration strings to URLs | `Check settings` | `Fetch company board` | |
| `Fetch company board` | `n8n-nodes-base.httpRequest` | Fetches jobs from company ATS endpoints | `Boards to watch` | `Normalize board postings` | |
| `Normalize board postings` | `n8n-nodes-base.code` | Normalizes raw company board payloads | `Fetch company board` | `Merge sources` | |
| `Role and location filter` | `n8n-nodes-base.filter` | Filters role regex matches and regions | `Region check` | `New since last run` | |
| `New since last run` | `n8n-nodes-base.removeDuplicates` | Removes previously processed job keys | `Role and location filter` | `Cap per run` | |
| `Cap per run` | `n8n-nodes-base.limit` | Limits executions based on max config | `New since last run` | `Judge fit (LLM)` | |
| `Judge fit (LLM)` | `@n8n/n8n-nodes-langchain.chainLlm` | Evaluates job fit via LLM and schema | `Cap per run` | `Apply verdict` | |
| `Claude Haiku 4.5 (chat model)` | `@n8n/n8n-nodes-langchain.lmChatAnthropic` | Language model provider configuration | None | `Judge fit (LLM)` | |
| `Verdict format (fit, score, reason)` | `@n8n/n8n-nodes-langchain.outputParserStructured` | Structured JSON output validation schema | None | `Judge fit (LLM)` | |
| `Apply verdict` | `n8n-nodes-base.code` | Formats verdicts into deliverables | `Judge fit (LLM)` | `Run summary`, `Fits or unjudged?` | |
| `Run summary` | `n8n-nodes-base.code` | Compiles run metrics and summary line | `Apply verdict` | `Send run summary to Telegram` | |
| `Send run summary to Telegram` | `n8n-nodes-base.telegram` | Delivers run summary message to Telegram | `Run summary` | None | |
| `Fits or unjudged?` | `n8n-nodes-base.if` | Evaluates delivery thresholds | `Apply verdict` | `Send job card to Telegram` | |
| `Send job card to Telegram` | `n8n-nodes-base.telegram` | Delivers individual job cards to Telegram | `Fits or unjudged?` | None | |
| `Region check` | `n8n-nodes-base.code` | Evaluates regional candidacy compliance | `One per opening` | `Role and location filter` | |
| `Check settings` | `n8n-nodes-base.code` | Validates configuration dependencies | `Configure me` | `Boards to watch`, `Feeds to read` | |
| `Feeds to read` | `n8n-nodes-base.code` | Generates aggregator feed request URLs | `Check settings` | `Fetch feed` | |
| `Fetch feed` | `n8n-nodes-base.httpRequest` | Fetches third-party job aggregators | `Feeds to read` | `Normalize feed postings` | |
| `Normalize feed postings` | `n8n-nodes-base.code` | Normalizes third-party job aggregator payloads | `Fetch feed` | `Merge sources` | |
| `Merge sources` | `n8n-nodes-base.merge` | Merges company boards and open feed arrays | `Normalize board postings`, `Normalize feed postings` | `One per opening` | |
| `One per opening` | `n8n-nodes-base.removeDuplicates` | Removes intra-run duplicate openings | `Merge sources` | `Region check` | |

---

### 4. Reproducing the Workflow from Scratch

To rebuild this workflow manually in an n8n instance, execute the following steps in sequence:

1. **Create Trigger Nodes:**
   - Add a `Schedule Trigger` node (`Every day 09:00`) configured to trigger daily at 09:00.
   - Add a `Manual Trigger` node (`Run now (manual)`).

2. **Setup Configuration Node:**
   - Add a `Set` (Edit Fields) node named `Configure me`.
   - Add string assignments: `telegramChatId` (default placeholder `YOUR_TELEGRAM_CHAT_ID`), `allowedRegion` (`Europe`), `boards` (multiline company list), `feedKeywords` (`product manager`), `profile` (candidate persona text).
   - Add number assignments: `minScore` (`6`), `maxPerRun` (`30`).
   - Add boolean assignment: `useAggregators` (`true`).
   - Connect both triggers to `Configure me`.

3. **Setup Validation Node:**
   - Add a `Code` node named `Check settings`.
   - Paste validation script to verify `telegramChatId` is set and ensure either boards or aggregators are active. Connect `Configure me` to `Check settings`.

4. **Build Company Boards Branch:**
   - Add a `Code` node named `Boards to watch` connected from `Check settings`.
   - Add an `HTTP Request` node named `Fetch company board` with `onError` set to continue regular output, timeout to `20000ms`, and dynamic URL mapping `{{ $json.url }}`. Connect `Boards to watch` to it.
   - Add a `Code` node named `Normalize board postings` to parse payloads from Ashby, Greenhouse, and Lever formats. Connect `Fetch company board` to it.

5. **Build Feeds Branch:**
   - Add a `Code` node named `Feeds to read` connected from `Check settings`.
   - Add an `HTTP Request` node named `Fetch feed` with timeout configuration and batch settings. Connect `Feeds to read` to it.
   - Add a `Code` node named `Normalize feed postings` to handle Jobicy and Himalayas payloads. Connect `Fetch feed` to it.

6. **Merge and Deduplicate Streams:**
   - Add a `Merge` node named `Merge sources`. Connect `Normalize board postings` to Input 0 and `Normalize feed postings` to Input 1.
   - Add a `Remove Duplicates` node named `One per opening` configured to compare by `key`. Connect `Merge sources` to it.

7. **Implement Filters:**
   - Add a `Code` node named `Region check` to evaluate location strings against regional regex patterns. Connect `One per opening` to it.
   - Add a `Filter` node named `Role and location filter` checking title regex, junior exclusions, and `inRegion`. Connect `Region check` to it.
   - Add a `Remove Duplicates` node named `New since last run` set to `removeItemsSeenInPreviousExecutions` tracking `key`. Connect `Role and location filter` to it.
   - Add a `Limit` node named `Cap per run` bound expression `{{ $('Configure me').first().json.maxPerRun }}`. Connect `New since last run` to it.

8. **Configure AI Evaluation Chain:**
   - Add a `Basic LLM Chain` node named `Judge fit (LLM)`. Connect `Cap per run` to its main input.
   - Add an `Anthropic Chat Model` node named `Claude Haiku 4.5 (chat model)` with model `claude-haiku-4-5`, temperature `0`, and set its credentials (`Anthropic account`). Connect it to the `ai_languageModel` input of `Judge fit (LLM)`.
   - Add a `Structured Output Parser` node named `Verdict format (fit, score, reason)` containing the JSON schema (`fit`, `score`, `reason`). Connect it to the `ai_outputParser` input of `Judge fit (LLM)`.

9. **Build Delivery and Notification Flow:**
   - Add a `Code` node named `Apply verdict` connected from `Judge fit (LLM)`.
   - Add a `Code` node named `Run summary` connected from `Apply verdict`.
   - Add a `Telegram` node named `Send run summary to Telegram` configured with HTML parse mode and credential configuration (`Telegram account`). Connect `Run summary` to it.
   - Add an `If` node named `Fits or unjudged?` checking `{{ $json.deliver }}`. Connect `Apply verdict` to it.
   - Add a `Telegram` node named `Send job card to Telegram` configured with HTML parse mode and credential configuration (`Telegram account`). Connect `Fits or unjudged?` (true branch) to it.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Jobicy API Endpoints | [Jobicy API Documentation](https://jobicy.com/) |
| Himalayas API Endpoints | [Himalayas API Documentation](https://himalayas.app/) |