Audit site AI search readiness and open fix tickets with Apify, OpenAI, Sheets and Telegram

https://n8nworkflows.xyz/workflows/audit-site-ai-search-readiness-and-open-fix-tickets-with-apify--openai--sheets-and-telegram-20497


# Audit site AI search readiness and open fix tickets with Apify, OpenAI, Sheets and Telegram

### 1. Workflow Overview

This workflow automates a weekly SEO, GEO (Generative Engine Optimization), and AEO (Answer Engine Optimization) audit for a target website. It crawls pages via an external actor, evaluates generative search readiness, compares current scores against a historical baseline stored in Google Sheets, uses an LLM to generate actionable engineering tickets for underperforming or regressed pages, archives historical data, and dispatches real-time alerts to Telegram.

The processing logic is structured into five distinct operational blocks:
- **1.1 Schedule & Configuration:** Triggers the workflow on a weekly cron schedule and injects baseline environment variables (target URLs, thresholds, sheet names, and chat IDs).
- **1.2 Data Collection & Baseline Retrieval:** Executes the Apify audit actor, aggregates multi-page item outputs, and fetches historical scores from Google Sheets.
- **1.3 Comparative Analysis & Routing:** Joins current audit metrics with historical records via custom JavaScript logic, categorizes performance shifts, logs all pages to a historical scoreboard tab, and filters out pages that require developer intervention.
- **1.4 AI Ticket Generation:** Caps the volume of flagged pages and passes structured failure data through an OpenAI language model configured with a strict output schema to draft developer-ready tickets.
- **1.5 Archiving & Alerting:** Formats generated tickets, appends them to an issue tracker tab in Google Sheets, and pushes rich HTML-formatted notifications to a designated Telegram chat.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule & Configuration
- **Overview:** Initializes the execution pipeline on a rigid weekly cadence and provides immutable configuration parameters for downstream nodes.
- **Nodes Involved:** 
  - `Every Monday at 7am`
  - `Set your site`

- **Node Details:**
  - **Every Monday at 7am**
    - *Type and Technical Role:* `n8n-nodes-base.scheduleTrigger` (Trigger Node). Initiates execution every Monday at 07:00.
    - *Configuration Choices:* Trigger interval set to weeks (`weeksInterval: 1`, `triggerAtDay: [1]`, `triggerAtHour: 7`).
    - *Input/Output:* No inputs; outputs a single execution timestamp object.
    - *Edge Cases/Failure Types:* Execution skipped if n8n instance is offline or paused during the cron window.
  - **Set your site**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data Transformation / Setup Node). Stores configuration variables.
    - *Configuration Choices:* Assigns string and numeric constants: `websiteUrl`, `maxPages` (10), `dropPoints` (5), `minScore` (70), `maxTickets` (5), `scoreTab` ("Scoreboard"), `ticketTab` ("Tickets"), `googleSheetUrl`, and `telegramChatId`.
    - *Input/Output:* Input from `Every Monday at 7am`; outputs an item containing all defined configuration key-value pairs.
    - *Edge Cases/Failure Types:* Invalid placeholders (e.g., leaving `PASTE_THE_SITE_TO_AUDIT_HERE`) will cause downstream API rejections.

#### 2.2 Data Collection & Baseline Retrieval
- **Overview:** Crawls the target website via Apify, aggregates individual page metrics into a single collection, and queries Google Sheets for historical records.
- **Nodes Involved:**
  - `Audit the site`
  - `Collect the audited pages`
  - `Read the last audit`

- **Node Details:**
  - **Audit the site**
    - *Type and Technical Role:* `@apify/n8n-nodes-apify.apify` (External API Integration Node). Runs the Apify SEO/GEO/AEO audit actor.
    - *Configuration Choices:* Actor ID set to `rufzbCa6yiJQbfHcw`. Custom JSON body maps `websiteUrl` and `maxPages` from the Setup node, disabling HTML reports for payload efficiency.
    - *Input/Output:* Input from `Set your site`; outputs a stream of individual page audit items from Apify.
    - *Edge Cases/Failure Types:* Apify token authentication errors, actor timeout on massive sites, or running out of Apify compute credits.
  - **Collect the audited pages**
    - *Type and Technical Role:* `n8n-nodes-base.aggregate` (Data Transformation Node). Consolidates streaming page items into a single array.
    - *Configuration Choices:* Aggregation mode set to `aggregateAllItemData`, assigning the array to destination field `pages`.
    - *Input/Output:* Input from `Audit the site`; outputs a single item containing the array of all crawled pages.
    - *Edge Cases/Failure Types:* Empty arrays if the Apify actor fails to discover valid internal links.
  - **Read the last audit**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Cloud Storage Integration Node). Reads historical scoring data from the scoreboard tab.
    - *Configuration Choices:* Document ID and sheet name dynamically resolved via expressions pointing to `Set your site`. `alwaysOutputData` set to `true`.
    - *Input/Output:* Input from `Collect the audited pages`; outputs rows from the historical tracking sheet.
    - *Edge Cases/Failure Types:* 
      - *Critical:* Without `alwaysOutputData: true`, an empty spreadsheet on the initial run causes the workflow to halt instantly.
      - Auth expiry on Google OAuth2 credentials.

#### 2.3 Comparative Analysis & Routing
- **Overview:** Joins current audit metrics against historical data, evaluates performance shifts, logs all data to the historical scoreboard, and filters pages requiring remediation.
- **Nodes Involved:**
  - `Decide what needs fixing`
  - `Format the scoreboard row`
  - `Save this audit to the scoreboard`
  - `Keep what needs a fix`

- **Node Details:**
  - **Decide what needs fixing**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript Execution Node). Performs relational joins between current and historical datasets, computes score deltas, and assigns remediation status flags.
    - *Configuration Choices:* Custom ES6 script iterating through pages, indexing historical URLs, evaluating regression thresholds (`dropPoints`), floor violations (`minScore`), and critical issue counts. Sorts output array by necessity and severity.
    - *Input/Output:* Input from `Collect the audited pages` and `Read the last audit`; outputs an array of processed page objects with `needsFix` boolean flags.
    - *Edge Cases/Failure Types:* Unhandled type casting errors if Google Sheets returns numeric scores as strings.
  - **Format the scoreboard row**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data Transformation Node). Maps structured code output into flat schema columns for the scoreboard.
    - *Configuration Choices:* Maps fields: `Checked` (`$now`), `URL`, `Page`, `Grade`, `Overall`, `SEO`, `GEO`, `AEO`, `Issues`, `Critical`, `Change`, and `Status`.
    - *Input/Output:* Input from `Decide what needs fixing`; outputs formatted rows.
  - **Save this audit to the scoreboard**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Cloud Storage Integration Node). Appends historical audit records.
    - *Configuration Choices:* Operation set to `append`. Document ID and Sheet name bound dynamically.
    - *Input/Output:* Input from `Format the scoreboard row`.
  - **Keep what needs a fix**
    - *Type and Technical Role:* `n8n-nodes-base.filter` (Conditional Routing Node). Isolates pages flagged for engineering intervention.
    - *Configuration Choices:* Condition evaluates `{{ $json.needsFix === true }}` using loose validation.
    - *Input/Output:* Input from `Decide what needs fixing`; outputs only items meeting the remediation criteria.

#### 2.4 AI Ticket Generation
- **Overview:** Restricts the volume of actionable items and utilizes an OpenAI language model equipped with a structured JSON output parser to draft engineering tickets.
- **Nodes Involved:**
  - `Cap the tickets`
  - `Write the fix ticket`
  - `OpenAI model`
  - `Ticket format`

- **Node Details:**
  - **Cap the tickets**
    - *Type and Technical Role:* `n8n-nodes-base.limit` (Flow Control Node). Limits maximum item throughput per execution batch.
    - *Configuration Choices:* `maxItems` set dynamically via `{{ $('Set your site').first().json.maxTickets }}`.
    - *Input/Output:* Input from `Keep what needs a fix`; outputs up to `maxTickets` items.
  - **Write the fix ticket**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.chainLlm` (Advanced AI / LangChain Node). Executes the LLM chain for ticket synthesis.
    - *Configuration Choices:* Prompt template injects page metrics, structural check counts, and raw issue arrays. Strict system prompt enforces imperative titles, mechanical priority mapping (P1/P2/P3), behaviour-based descriptions, and exact step counts derived strictly from failed checks.
    - *Input/Output:* Inputs connected from `Cap the tickets`, `OpenAI model` (AI Language Model), and `Ticket format` (AI Output Parser); outputs structured JSON.
    - *Edge Cases/Failure Types:* LLM hallucinations or schema validation errors if the output fails to parse against the JSON schema.
  - **OpenAI model**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.lmChatOpenAi` (AI Model Provider Node). Configures the underlying chat completion engine.
    - *Configuration Choices:* Model set to `gpt-4o-mini`, temperature set to `0.3` for deterministic outputs.
    - *Input/Output:* Outputs model connection interface to `Write the fix ticket`.
  - **Ticket format**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.outputParserStructured` (AI Parser Node). Forces structured JSON compliance.
    - *Configuration Choices:* Schema defines required keys: `title` (string), `priority` (string), `whyItMatters` (string), `steps` (string), and `effort` (string).
    - *Input/Output:* Outputs parser connection interface to `Write the fix ticket`.

#### 2.5 Archiving & Alerting
- **Overview:** Structures generated tickets into database-ready rows and pushes real-time notifications to team communication channels.
- **Nodes Involved:**
  - `Format the ticket row`
  - `Save the ticket`
  - `Send the ticket to Telegram`

- **Node Details:**
  - **Format the ticket row**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data Transformation Node). Normalizes AI-generated ticket outputs and node references.
    - *Configuration Choices:* Assigns fields: `Opened`, `Priority`, `Ticket`, `URL`, `Why it matters`, `Steps`, `Effort`, `Why it is on the list`, `Overall`, `Change`, and `Critical issues` by referencing parent nodes.
    - *Input/Output:* Input from `Write the fix ticket`; outputs flattened ticket row objects.
  - **Save the ticket**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Cloud Storage Integration Node). Appends tickets to the issue tracker sheet.
    - *Configuration Choices:* Operation set to `append`. Document ID and sheet name (`ticketTab`) bound dynamically.
    - *Input/Output:* Input from `Format the ticket row`.
  - **Send the ticket to Telegram**
    - *Type and Technical Role:* `n8n-nodes-base.telegram` (Messaging Integration Node). Broadcasts ticket details via Telegram bot API.
    - *Configuration Choices:* Chat ID bound dynamically from setup parameters. HTML parse mode enabled with custom string escaping, link truncation, and code block formatting for steps.
    - *Input/Output:* Input from `Format the ticket row`.
    - *Edge Cases/Failure Types:* Telegram API rate limits or blocked bot credentials.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Sticky Note - About** | `n8n-nodes-base.stickyNote` | Documentation & Overview | None | None | ## Audit your site for AI search readiness, then open the tickets to fix it... |
| **Sticky Note - Step 1** | `n8n-nodes-base.stickyNote` | Documentation for Setup | None | None | ### 1. The site, and what deserves a ticket... |
| **Sticky Note - Step 2** | `n8n-nodes-base.stickyNote` | Documentation for Auditing | None | None | ### 2. Audit now, then read the last audit... |
| **Sticky Note - Step 3** | `n8n-nodes-base.stickyNote` | Documentation for Comparison | None | None | ### 3. Compare, then decide what is worth a developer's time... |
| **Sticky Note - Step 4** | `n8n-nodes-base.stickyNote` | Documentation for Ticketing | None | None | ### 4. Write the ticket, not the score... |
| **Sticky Note - Step 5** | `n8n-nodes-base.stickyNote` | Documentation for Delivery | None | None | ### 5. A backlog, in two places... |
| **Every Monday at 7am** | `n8n-nodes-base.scheduleTrigger` | Weekly Cron Execution | None | Set your site | |
| **Set your site** | `n8n-nodes-base.set` | Configuration Variable Store | Every Monday at 7am | Audit the site | |
| **Audit the site** | `@apify/n8n-nodes-apify.apify` | Apify Actor Execution | Set your site | Collect the audited pages | |
| **Collect the audited pages** | `n8n-nodes-base.aggregate` | Payload Aggregation | Audit the site | Read the last audit | |
| **Read the last audit** | `n8n-nodes-base.googleSheets` | Baseline Data Fetch | Collect the audited pages | Decide what needs fixing | |
| **Decide what needs fixing** | `n8n-nodes-base.code` | Comparative Delta Analysis | Read the last audit | Format the scoreboard row, Keep what needs a fix | |
| **Format the scoreboard row** | `n8n-nodes-base.set` | Scoreboard Row Normalization | Decide what needs fixing | Save this audit to the scoreboard | |
| **Save this audit to the scoreboard** | `n8n-nodes-base.googleSheets` | Historical Logging | Format the scoreboard row | None | |
| **Keep what needs a fix** | `n8n-nodes-base.filter` | Remediation Filtering | Decide what needs fixing | Cap the tickets | |
| **Cap the tickets** | `n8n-nodes-base.limit` | Volume Restrictor | Keep what needs a fix | Write the fix ticket | |
| **Write the fix ticket** | `@n8n/n8n-nodes-langchain.chainLlm` | LLM Ticket Synthesis | Cap the tickets, OpenAI model, Ticket format | Format the ticket row | |
| **OpenAI model** | `@n8n/n8n-nodes-langchain.lmChatOpenAi` | LLM Provider Configuration | None | Write the fix ticket | |
| **Ticket format** | `@n8n/n8n-nodes-langchain.outputParserStructured` | JSON Schema Enforcement | None | Write the fix ticket | |
| **Format the ticket row** | `n8n-nodes-base.set` | Ticket Normalization | Write the fix ticket | Save the ticket, Send the ticket to Telegram | |
| **Save the ticket** | `n8n-nodes-base.googleSheets` | Issue Tracker Archiving | Format the ticket row | None | |
| **Send the ticket to Telegram** | `n8n-nodes-base.telegram` | Real-time Alerting | Format the ticket row | None | |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger:** Add a `Schedule Trigger` node. Name it `Every Monday at 7am`. Set Interval to Weeks, execution day to Monday, and hour to 7.
2. **Add Configuration Node:** Create a `Set (Edit Fields)` node named `Set your site`. Add string assignments for `websiteUrl`, `scoreTab` ("Scoreboard"), `ticketTab` ("Tickets"), `googleSheetUrl`, and `telegramChatId`. Add numeric assignments for `maxPages` (10), `dropPoints` (5), `minScore` (70), and `maxTickets` (5). Connect `Every Monday at 7am` to this node.
3. **Integrate Apify Audit:** Add an `Apify` node named `Audit the site`. Select actor ID `rufzbCa6yiJQbfHcw`. Insert custom JSON expression to pass `websiteUrl`, `maxPages`, and `includeHtmlReport: false`. Connect `Set your site` output here.
4. **Aggregate Pages:** Add an `Aggregate` node named `Collect the audited pages`. Set aggregation mode to `aggregateAllItemData` with destination field `pages`. Connect `Audit the site` to this node.
5. **Fetch Historical Baseline:** Add a `Google Sheets` node named `Read the last audit`. Configure operation to read rows. Bind document ID and sheet name dynamically to configuration parameters. **Crucial:** Enable `alwaysOutputData` in node options. Connect `Collect the audited pages` here.
6. **Implement Comparison Logic:** Add a `Code` node named `Decide what needs fixing`. Insert ES6 script parsing inputs from `Set your site` and `Collect the audited pages`, cross-referencing previous scoreboard runs by URL, calculating deltas, sorting items, and assigning `needsFix` flags. Connect `Read the last audit` to this node.
7. **Branch - Scoreboard Logging:**
   - Add a `Set` node named `Format the scoreboard row`. Map execution values to sheet columns (`Checked`, `URL`, `Page`, `Grade`, `Overall`, `SEO`, `GEO`, `AEO`, `Issues`, `Critical`, `Change`, `Status`). Connect `Decide what needs fixing` to this node.
   - Add a `Google Sheets` node named `Save this audit to the scoreboard`. Set operation to `append` with dynamic binding. Connect `Format the scoreboard row` here.
8. **Branch - Remediation Filtering & Capping:**
   - Add a `Filter` node named `Keep what needs a fix`. Set condition where `{{ $json.needsFix === true }}`. Connect `Decide what needs fixing` to this node.
   - Add a `Limit` node named `Cap the tickets`. Bind `maxItems` to `{{ $('Set your site').first().json.maxTickets }}`. Connect `Keep what needs a fix` here.
9. **Configure AI Generation Chain:**
   - Add an OpenAI Chat Model node named `OpenAI model`. Select model `gpt-4o-mini` with temperature `0.3`. Configure valid OpenAI API credentials.
   - Add a Structured Output Parser node named `Ticket format`. Paste the manual JSON schema defining `title`, `priority`, `whyItMatters`, `steps`, and `effort`.
   - Add an Advanced AI / LLM Chain node named `Write the fix ticket`. Paste the specialized system prompt and connect inputs from `Cap the tickets`, `OpenAI model`, and `Ticket format`.
10. **Format and Dispatch Tickets:**
    - Add a `Set` node named `Format the ticket row`. Map AI outputs and parent JSON attributes into issue tracking columns. Connect `Write the fix ticket` here.
    - Add a `Google Sheets` node named `Save the ticket`. Set operation to `append` targeting `ticketTab`. Connect `Format the ticket row` here.
    - Add a `Telegram` node named `Send the ticket to Telegram`. Configure HTML-formatted message template, dynamic chat ID assignment, and valid Telegram bot credentials. Connect `Format the ticket row` here.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Apify Actor Documentation & Platform | [Apify Platform](https://apify.com/?fpr=youssef) |
| SEO, GEO and AEO Audit Actor | [Apify Actor Store](https://apify.com/fayoussef/seo-geo-aeo-audit?fpr=youssef) |