Monitor competitors and generate market insights with Claude AI and Notion

https://n8nworkflows.xyz/workflows/monitor-competitors-and-generate-market-insights-with-claude-ai-and-notion-13983


# Monitor competitors and generate market insights with Claude AI and Notion

# 1. Workflow Overview

This workflow performs scheduled competitor monitoring. Each week, it fetches competitor web pages, extracts selected signals from the HTML, computes a content hash to detect changes, sends detected changes to Claude for strategic analysis, stores the result in Notion, optionally alerts Slack for urgent findings, and also generates a weekly email summary.

The workflow is organized into four logical blocks:

## 1.1 Scheduled Input and Website Collection
The workflow starts on a weekly schedule, prepares a list of competitor URLs, initializes indexing data, downloads page content, and extracts structured website signals plus a hash for change detection.

## 1.2 Change Detection
The extracted content is compared against a previous hash value to determine whether the monitored page appears to have changed.

## 1.3 AI Analysis, Persistence, and Alerting
If changes are detected, Claude analyzes the page update, the result is saved to Notion, and a code step decides whether the update is important enough to trigger a Slack alert.

## 1.4 Weekly Reporting
Independently of whether the AI analysis branch runs, detected scan results are summarized into a weekly report and emailed through Gmail.

---

# 2. Block-by-Block Analysis

## 2.1 Scheduled Input and Website Collection

### Overview
This block launches the workflow once per week and prepares the competitor page data for processing. It then downloads the configured URL, extracts useful text patterns from the HTML, and builds a hash used later for change detection.

### Nodes Involved
- Weekly competitor scan
- Set competitor URLs
- Initialize loop counter
- Fetch competitor content
- Extract and hash content

### Node Details

#### Weekly competitor scan
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`; entry-point trigger node that runs the workflow on a recurring schedule.
- **Configuration choices:** Configured to run every week, on day `1`, at hour `9`. In n8n scheduling, `triggerAtDay: 1` typically means a specific weekday depending on instance locale/version behavior; verify after import.
- **Key expressions or variables used:** None.
- **Input and output connections:** No input; outputs to **Set competitor URLs**.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** Misinterpreted timezone, wrong weekday expectation, disabled workflow, server clock drift.
- **Sub-workflow reference:** None.

#### Set competitor URLs
- **Type and technical role:** `n8n-nodes-base.set`; intended to define arrays of competitor URLs and competitor names.
- **Configuration choices:** The JSON shows no explicit fields configured, but downstream nodes clearly expect:
  - `competitor_urls`
  - `competitor_names`
  - possibly aligned indexing
  This node is conceptually the configuration store for monitored targets.
- **Key expressions or variables used:** Downstream references expect:
  - `$json.competitor_urls`
  - `$json.competitor_names`
- **Input and output connections:** Input from **Weekly competitor scan**; output to **Initialize loop counter**.
- **Version-specific requirements:** Type version `3`.
- **Edge cases or potential failure types:** If arrays are missing, empty, or lengths differ, later nodes will fail or return undefined URLs/names. As provided, this is a structural gap in the current workflow JSON and must be completed manually.
- **Sub-workflow reference:** None.

#### Initialize loop counter
- **Type and technical role:** `n8n-nodes-base.set`; intended to initialize an index such as `current_index` for choosing which competitor URL to process.
- **Configuration choices:** The JSON shows no explicit values, but downstream expressions expect `current_index`.
- **Key expressions or variables used:** Downstream nodes rely on:
  - `$json.current_index`
- **Input and output connections:** Input from **Set competitor URLs**; output to **Fetch competitor content**.
- **Version-specific requirements:** Type version `3`.
- **Edge cases or potential failure types:** If `current_index` is not set to a valid integer, the HTTP Request URL expression will fail. There is also no actual loop node in the workflow, so only one index would be processed unless the workflow is redesigned.
- **Sub-workflow reference:** None.

#### Fetch competitor content
- **Type and technical role:** `n8n-nodes-base.httpRequest`; downloads the competitor page HTML.
- **Configuration choices:** URL is dynamically set from `competitor_urls[current_index]`. Timeout is set to `10000` ms.
- **Key expressions or variables used:**
  - `={{ $json.competitor_urls[$json.current_index] }}`
- **Input and output connections:** Input from **Initialize loop counter**; output to **Extract and hash content**.
- **Version-specific requirements:** Type version `4`.
- **Edge cases or potential failure types:**
  - Invalid or undefined URL if array/index setup is wrong
  - HTTP 403/404/429/500 errors
  - Timeout after 10 seconds
  - Redirect or anti-bot blocking
  - Some pages may require headers, user-agent, or JavaScript rendering not present here
- **Sub-workflow reference:** None.

#### Extract and hash content
- **Type and technical role:** `n8n-nodes-base.code`; parses HTML, extracts selected content snippets, computes an MD5 hash, and emits a normalized object for change tracking.
- **Configuration choices:** Uses `cheerio` and `crypto` inside JavaScript code. Extracts:
  - page title
  - concatenated `<h1>` values
  - text snippets matching pricing-related terms
  - text snippets matching feature-related terms
  - MD5 hash of combined extracted content
  - truncated raw HTML
- **Key expressions or variables used:**
  - `$input.first()?.body`
  - attempts to access prior node data for URL and competitor name
- **Input and output connections:** Input from **Fetch competitor content**; output to **Detect content changes**.
- **Version-specific requirements:** Type version `2`. Requires Code node runtime support for `require('cheerio')` and `require('crypto')`. In some n8n environments, external modules must be enabled explicitly.
- **Edge cases or potential failure types:**
  - `cheerio` unavailable in hosted/restricted environments
  - malformed HTML
  - empty response body
  - selector overmatching and noisy extracted text
  - the code contains likely expression mistakes:
    - `const url = $('$node["Fetch competitor content"].json.url');`
    - `const competitorName = $('$node["Set competitor URLs"].json.competitor_names[$node["Set competitor URLs"].json.current_index]');`
    These are written as Cheerio selectors, not n8n variable references, so they will not return the intended values. They should be rewritten using plain JavaScript/n8n data access.
- **Sub-workflow reference:** None.

---

## 2.2 Change Detection

### Overview
This block determines whether the fetched page content differs from the previously known version. It uses a placeholder comparison approach rather than a true historical lookup.

### Nodes Involved
- Detect content changes
- Changes detected?

### Node Details

#### Detect content changes
- **Type and technical role:** `n8n-nodes-base.code`; compares the current content hash to a previous hash and marks whether a change occurred.
- **Configuration choices:** Uses a hardcoded simulated previous hash (`abc123def456`) instead of querying Notion or another datastore.
- **Key expressions or variables used:**
  - `$input.first()`
  - `currentData.content_hash`
- **Input and output connections:** Input from **Extract and hash content**; outputs to:
  - **Changes detected?**
  - **Generate weekly report**
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - Since the previous hash is hardcoded, almost all real pages will appear changed
  - If `content_hash` is missing, output logic is unreliable
  - This node is explicitly a simplified placeholder and not production-grade as-is
- **Sub-workflow reference:** None.

#### Changes detected?
- **Type and technical role:** `n8n-nodes-base.if`; gates AI analysis so only changed pages are analyzed.
- **Configuration choices:** Evaluates whether `has_changes` is `true`.
- **Key expressions or variables used:**
  - `={{ $json.has_changes }}`
- **Input and output connections:** Input from **Detect content changes**; true branch outputs to **Analyze competitor changes**. No false-branch node is connected.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - If `has_changes` is not a boolean, branch evaluation may behave unexpectedly
  - False cases simply terminate this branch silently
- **Sub-workflow reference:** None.

---

## 2.3 AI Analysis, Persistence, and Alerting

### Overview
This block uses Claude to produce strategic competitor intelligence from detected changes, persists the result in Notion, and then decides whether the content is urgent enough to send to Slack.

### Nodes Involved
- Analyze competitor changes
- Claude AI model
- Save to Notion database
- Evaluate notification priority
- Send urgent notification?
- Send Slack alert

### Node Details

#### Analyze competitor changes
- **Type and technical role:** `@n8n/n8n-nodes-langchain.chainLlm`; prompts an LLM to analyze website changes and return structured strategic insight.
- **Configuration choices:** Prompt is fully defined in the node. It asks for:
  1. likely changes
  2. strategic implications
  3. recommended actions
  4. market trends
  5. threats/opportunities  
  It requests structured JSON output.
- **Key expressions or variables used:**
  - `{{ $json.competitor_name }}`
  - `{{ $json.url }}`
  - `{{ $json.title }}`
  - `{{ $json.headings }}`
  - `{{ $json.pricing_mentions }}`
  - `{{ $json.feature_mentions }}`
  - `{{ $json.previous_hash }}`
  - `{{ $json.content_hash }}`
- **Input and output connections:** Main input from **Changes detected?** true branch. AI model input from **Claude AI model** via `ai_languageModel`. Output goes to **Save to Notion database**.
- **Version-specific requirements:** Type version `1.4`; requires LangChain-capable n8n version and compatible Anthropic chat model node.
- **Edge cases or potential failure types:**
  - Anthropic credential issues
  - model quota/rate limits
  - prompt output may not be valid JSON despite instruction
  - weak analysis if extraction upstream is sparse or noisy
- **Sub-workflow reference:** None.

#### Claude AI model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatAnthropic`; provides the Anthropic Claude language model used by the analysis chain.
- **Configuration choices:** Model set to `claude-3-5-sonnet-20241022`.
- **Key expressions or variables used:** None.
- **Input and output connections:** Connected to **Analyze competitor changes** as its language model.
- **Version-specific requirements:** Type version `1`; requires Anthropic API credentials and availability of this exact model in the connected account/region.
- **Edge cases or potential failure types:**
  - invalid credentials
  - unavailable or deprecated model ID
  - usage limits or token constraints
- **Sub-workflow reference:** None.

#### Save to Notion database
- **Type and technical role:** `n8n-nodes-base.notion`; creates a database page containing competitor monitoring results.
- **Configuration choices:** Resource is `databasePage`. Database ID comes from environment variable `NOTION_COMPETITOR_DB_ID`. Title is generated from competitor name and current date. Configured properties include:
  - Competitor
  - URL
  - Content Hash
  - AI Analysis
  - Scan Date
  The JSON shows the property keys configured, but not all mapped values are visible in detail.
- **Key expressions or variables used:**
  - `{{ $json.competitor_name }} - {{ $now.format('YYYY-MM-DD') }}`
  - `{{ $vars.NOTION_COMPETITOR_DB_ID }}`
  - `={{ $json.url }}`
- **Input and output connections:** Input from **Analyze competitor changes**; output to **Evaluate notification priority**.
- **Version-specific requirements:** Type version `2`; requires a Notion integration with write access to the target database.
- **Edge cases or potential failure types:**
  - database ID missing
  - property names do not exactly match the Notion database schema
  - insufficient integration permission on the database
  - data type mismatch between n8n field mapping and Notion property types
- **Sub-workflow reference:** None.

#### Evaluate notification priority
- **Type and technical role:** `n8n-nodes-base.code`; checks the AI response text for urgent keywords and prepares a Slack message if needed.
- **Configuration choices:** Uses a keyword list:
  - price
  - pricing
  - launch
  - feature
  - acquisition
  - funding
  - partnership
- **Key expressions or variables used:**
  - `$node['Analyze competitor changes'].json.response`
  - `$json.competitor_name`
  - `$json.url`
- **Input and output connections:** Input from **Save to Notion database**; output to **Send urgent notification?**
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - Depends on the analysis output being available as `.response`
  - If the LLM output is object-shaped JSON rather than plain text, `.toLowerCase()` may fail unless coerced to string
  - Slack formatting may not render as expected because message contains Markdown-style syntax that may differ from Slack’s exact parsing
- **Sub-workflow reference:** None.

#### Send urgent notification?
- **Type and technical role:** `n8n-nodes-base.if`; routes only urgent analyses to Slack.
- **Configuration choices:** Checks whether `should_notify` is `true`.
- **Key expressions or variables used:**
  - `={{ $json.should_notify }}`
- **Input and output connections:** Input from **Evaluate notification priority**; true branch outputs to **Send Slack alert**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:** If the code node returns malformed boolean values, branching may not behave as expected.
- **Sub-workflow reference:** None.

#### Send Slack alert
- **Type and technical role:** `n8n-nodes-base.slack`; sends the generated urgent message to Slack.
- **Configuration choices:** Sends `slack_message` from previous node. The node JSON does not explicitly show channel selection; this may depend on credential defaults or missing configuration.
- **Key expressions or variables used:**
  - `={{ $json.slack_message }}`
- **Input and output connections:** Input from **Send urgent notification?** true branch. No downstream nodes.
- **Version-specific requirements:** Type version `2`; requires Slack credentials with permission to post messages.
- **Edge cases or potential failure types:**
  - missing channel configuration
  - invalid Slack app scopes
  - message length or formatting issues
  - webhook/app posting restrictions
- **Sub-workflow reference:** None.

---

## 2.4 Weekly Reporting

### Overview
This block aggregates scan results into a compact weekly report and emails it to a team distribution list.

### Nodes Involved
- Generate weekly report
- Email weekly report

### Node Details

#### Generate weekly report
- **Type and technical role:** `n8n-nodes-base.code`; creates summary statistics and an email-friendly report body.
- **Configuration choices:** Builds:
  - report title
  - competitor count
  - number of changes detected
  - per-competitor status
  - a fixed recommendations list
  Then formats the result into plain text/Markdown-like email content.
- **Key expressions or variables used:**
  - `$input.all()`
  - item fields such as `competitor_name`, `has_changes`, `extracted_at`
- **Input and output connections:** Input from **Detect content changes**; output to **Email weekly report**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - With the current workflow structure, it likely receives items directly from the detection node, not from a true loop/aggregate across many URLs unless multiple items are emitted
  - If upstream only processes one item, the "weekly report" becomes a single-item summary
  - Markdown-style formatting may not render in plain-text email as intended
- **Sub-workflow reference:** None.

#### Email weekly report
- **Type and technical role:** `n8n-nodes-base.gmail`; sends the weekly summary by email.
- **Configuration choices:** Sends to `TEAM_EMAIL_LIST`, subject from generated field, message body from generated field, email type set to `text`.
- **Key expressions or variables used:**
  - `{{ $vars.TEAM_EMAIL_LIST }}`
  - `={{ $json.email_body }}`
  - `={{ $json.email_subject }}`
- **Input and output connections:** Input from **Generate weekly report**. No downstream nodes.
- **Version-specific requirements:** Type version `2`; requires Gmail OAuth2 credentials and permission to send email.
- **Edge cases or potential failure types:**
  - invalid Gmail OAuth2 connection
  - recipient list improperly formatted
  - sending limits or organizational restrictions
  - plain-text body may display Markdown markers literally
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Weekly competitor scan | Schedule Trigger | Weekly workflow entry point |  | Set competitor URLs | ## Data collection\nScheduled scraping of competitor websites with change detection |
| Set competitor URLs | Set | Defines monitored competitors and URLs | Weekly competitor scan | Initialize loop counter | ## Data collection\nScheduled scraping of competitor websites with change detection |
| Initialize loop counter | Set | Initializes index for URL selection | Set competitor URLs | Fetch competitor content | ## Data collection\nScheduled scraping of competitor websites with change detection |
| Fetch competitor content | HTTP Request | Downloads competitor page HTML | Initialize loop counter | Extract and hash content | ## Data collection\nScheduled scraping of competitor websites with change detection |
| Extract and hash content | Code | Extracts content signals and computes content hash | Fetch competitor content | Detect content changes | ## Data collection\nScheduled scraping of competitor websites with change detection |
| Detect content changes | Code | Compares current hash to previous hash | Extract and hash content | Changes detected?\nGenerate weekly report | ## Data collection\nScheduled scraping of competitor websites with change detection |
| Changes detected? | If | Sends only changed items to AI analysis | Detect content changes | Analyze competitor changes | ## Analysis & storage\nClaude AI analyzes changes and results are saved to Notion database |
| Analyze competitor changes | LangChain Chain LLM | Produces strategic competitive analysis from page changes | Changes detected? | Save to Notion database | ## Analysis & storage\nClaude AI analyzes changes and results are saved to Notion database |
| Claude AI model | Anthropic Chat Model | Supplies Claude model to analysis chain |  | Analyze competitor changes | ## Analysis & storage\nClaude AI analyzes changes and results are saved to Notion database |
| Save to Notion database | Notion | Stores competitor insight record in Notion | Analyze competitor changes | Evaluate notification priority | ## Notifications\nImportant changes trigger immediate Slack alerts with AI insights |
| Evaluate notification priority | Code | Determines whether analysis is urgent | Save to Notion database | Send urgent notification? | ## Notifications\nImportant changes trigger immediate Slack alerts with AI insights |
| Send urgent notification? | If | Gates Slack notifications | Evaluate notification priority | Send Slack alert | ## Notifications\nImportant changes trigger immediate Slack alerts with AI insights |
| Send Slack alert | Slack | Sends urgent competitor alert | Send urgent notification? |  | ## Notifications\nImportant changes trigger immediate Slack alerts with AI insights |
| Generate weekly report | Code | Builds weekly summary email content | Detect content changes | Email weekly report | ## Reporting\nWeekly summary emails with competitive intelligence insights |
| Email weekly report | Gmail | Sends weekly summary email | Generate weekly report |  | ## Reporting\nWeekly summary emails with competitive intelligence insights |
| Main workflow overview | Sticky Note | Documentation note |  |  | # Research competitors and generate market insights\n\n### How it works\nThis workflow automatically monitors competitor websites weekly, detects content changes using hash comparison, analyzes updates with Claude AI, and stores insights in Notion. Important changes trigger immediate Slack notifications, while comprehensive weekly reports are emailed to your team.\n\n### Setup steps\n1. **Configure competitor URLs** in the Set node - add websites, pricing pages, and feature pages to monitor\n2. **Set up credentials**: Anthropic Claude API, Notion API, Slack API, Gmail OAuth2\n3. **Create Notion database** with properties: Competitor (text), URL (url), Content Hash (text), AI Analysis (text), Scan Date (date)\n4. **Set environment variables**: `NOTION_COMPETITOR_DB_ID`, `SLACK_ALERTS_CHANNEL`, `TEAM_EMAIL_LIST`\n5. **Test the workflow** with manual execution before activating the weekly schedule\n\n### Customization\n- Adjust the schedule trigger for different monitoring frequencies\n- Modify urgent keywords in the priority evaluation node\n- Customize the Claude AI prompt for specific analysis focus areas\n- Add more competitor URLs or change notification thresholds |
| Collection section | Sticky Note | Section label |  |  | ## Data collection\nScheduled scraping of competitor websites with change detection |
| Analysis section | Sticky Note | Section label |  |  | ## Analysis & storage\nClaude AI analyzes changes and results are saved to Notion database |
| Notifications section | Sticky Note | Section label |  |  | ## Notifications\nImportant changes trigger immediate Slack alerts with AI insights |
| Reporting section | Sticky Note | Section label |  |  | ## Reporting\nWeekly summary emails with competitive intelligence insights |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **Monitor competitors and generate market insights with Claude AI and Notion**.

2. **Add a Schedule Trigger node**
   - Node type: **Schedule Trigger**
   - Name: **Weekly competitor scan**
   - Configure a weekly run:
     - Interval unit: weeks
     - Day: the intended weekday for competitor scans
     - Hour: `9`
   - Confirm the workflow timezone in n8n settings.

3. **Add a Set node for competitor configuration**
   - Node type: **Set**
   - Name: **Set competitor URLs**
   - Add fields manually, for example:
     - `competitor_urls` as an array of URLs
     - `competitor_names` as an array of names aligned by index
   - Example conceptual structure:
     - `competitor_urls[0] = https://competitor-a.com/pricing`
     - `competitor_names[0] = Competitor A`
   - Connect **Weekly competitor scan → Set competitor URLs**.

4. **Add a Set node to initialize indexing**
   - Node type: **Set**
   - Name: **Initialize loop counter**
   - Add field:
     - `current_index = 0`
   - Connect **Set competitor URLs → Initialize loop counter**.
   - Important: the provided workflow does not implement a real loop. If you want to monitor multiple URLs, add a proper loop pattern such as:
     - split arrays into items before HTTP requests, or
     - use Loop Over Items / Split Out / Code-based item expansion.
   - If you reproduce the workflow exactly as shown, it only addresses one indexed entry unless expanded.

5. **Add an HTTP Request node**
   - Node type: **HTTP Request**
   - Name: **Fetch competitor content**
   - Method: `GET`
   - URL expression:
     - `{{ $json.competitor_urls[$json.current_index] }}`
   - Timeout: `10000`
   - Response format: ensure the raw page body is available to the Code node.
   - Connect **Initialize loop counter → Fetch competitor content**.

6. **Add a Code node for parsing and hashing**
   - Node type: **Code**
   - Name: **Extract and hash content**
   - Use JavaScript.
   - Implement logic to:
     - read HTML body
     - parse with `cheerio`
     - extract title, H1 headings, pricing mentions, feature mentions
     - create an MD5 hash from extracted text
     - output structured fields such as:
       - `url`
       - `competitor_name`
       - `title`
       - `headings`
       - `pricing_mentions`
       - `feature_mentions`
       - `content_hash`
       - `extracted_at`
       - `raw_html`
   - Recommended correction when rebuilding:
     - Do **not** use Cheerio selectors for node references.
     - Instead, use direct n8n access, for example:
       - URL from incoming item or HTTP node data
       - competitor name from the Set node arrays using the current index
   - Connect **Fetch competitor content → Extract and hash content**.
   - If self-hosted, ensure Code node external module support includes `cheerio`. `crypto` is normally available in Node.js.

7. **Add a Code node for change detection**
   - Node type: **Code**
   - Name: **Detect content changes**
   - Reproduce the current simplified behavior:
     - take `content_hash`
     - compare it to a previous hash value
     - output `has_changes`, `previous_hash`, `change_detected_at`
   - For an exact recreation, use a placeholder previous hash such as `abc123def456`.
   - For a production-ready rebuild, replace the placeholder with a Notion lookup or another persistent store query before comparison.
   - Connect **Extract and hash content → Detect content changes**.

8. **Add an If node for change filtering**
   - Node type: **If**
   - Name: **Changes detected?**
   - Condition:
     - boolean
     - value1: `{{ $json.has_changes }}`
     - operation: `true`
   - Connect **Detect content changes → Changes detected?**

9. **Add the Anthropic chat model node**
   - Node type: **Anthropic Chat Model**
   - Name: **Claude AI model**
   - Select model:
     - `claude-3-5-sonnet-20241022`
   - Create or attach Anthropic credentials with a valid API key.

10. **Add a LangChain Chain LLM node**
    - Node type: **Chain LLM**
    - Name: **Analyze competitor changes**
    - Prompt type: define
    - Paste a prompt equivalent to:
      - identify likely changes
      - explain implications
      - recommend actions
      - note market trends
      - identify threats/opportunities
      - ask for structured JSON output
    - Use expressions for:
      - competitor name
      - URL
      - title
      - headings
      - pricing mentions
      - feature mentions
      - previous hash
      - current hash
    - Connect:
      - **Changes detected? (true) → Analyze competitor changes**
      - **Claude AI model → Analyze competitor changes** via AI language model connector

11. **Prepare the Notion database**
    - In Notion, create a database with properties matching the workflow, such as:
      - Title
      - Competitor
      - URL
      - Content Hash
      - AI Analysis
      - Scan Date
    - Share the database with the Notion integration used by n8n.

12. **Set environment/workflow variables**
    - Add or define:
      - `NOTION_COMPETITOR_DB_ID`
      - `TEAM_EMAIL_LIST`
      - optionally `SLACK_ALERTS_CHANNEL` if you choose to use it explicitly
    - Ensure n8n can resolve `$vars.NOTION_COMPETITOR_DB_ID` and `$vars.TEAM_EMAIL_LIST`.

13. **Add a Notion node**
    - Node type: **Notion**
    - Name: **Save to Notion database**
    - Resource: **Database Page**
    - Operation: create
    - Database ID:
      - `{{ $vars.NOTION_COMPETITOR_DB_ID }}`
    - Title:
      - `{{ $json.competitor_name }} - {{ $now.format('YYYY-MM-DD') }}`
    - Map properties to the database schema:
      - Competitor ← competitor name
      - URL ← page URL
      - Content Hash ← current hash
      - AI Analysis ← Claude response
      - Scan Date ← current date/timestamp
    - Connect **Analyze competitor changes → Save to Notion database**.
    - Add Notion credentials.

14. **Add a Code node for alert prioritization**
    - Node type: **Code**
    - Name: **Evaluate notification priority**
    - Use JavaScript to:
      - read the Claude analysis result
      - convert it to lowercase text
      - scan for urgent keywords
      - output:
        - `should_notify`
        - `slack_message`
        - and analysis metadata
    - Recommended hardening:
      - cast the analysis result to string before calling `.toLowerCase()`
      - handle object/JSON outputs safely
    - Connect **Save to Notion database → Evaluate notification priority**.

15. **Add an If node for Slack gating**
    - Node type: **If**
    - Name: **Send urgent notification?**
    - Condition:
      - boolean
      - value1: `{{ $json.should_notify }}`
      - operation: `true`
    - Connect **Evaluate notification priority → Send urgent notification?**

16. **Add a Slack node**
    - Node type: **Slack**
    - Name: **Send Slack alert**
    - Configure message text:
      - `{{ $json.slack_message }}`
    - Choose a destination channel explicitly when rebuilding; the exported JSON does not clearly show one.
    - Add Slack credentials with permission to post messages.
    - Connect **Send urgent notification? (true) → Send Slack alert**.

17. **Add a Code node for weekly summary generation**
    - Node type: **Code**
    - Name: **Generate weekly report**
    - Implement JavaScript that:
      - reads all incoming items
      - counts competitors monitored
      - counts changed items
      - groups by competitor
      - generates:
        - `email_subject`
        - `email_body`
        - `report_data`
    - Connect **Detect content changes → Generate weekly report**.
    - Note: for a true multi-competitor report, upstream must produce multiple items. If you only keep the current single-index design, the report will summarize one processed item.

18. **Add a Gmail node**
    - Node type: **Gmail**
    - Name: **Email weekly report**
    - To:
      - `{{ $vars.TEAM_EMAIL_LIST }}`
    - Subject:
      - `{{ $json.email_subject }}`
    - Message:
      - `{{ $json.email_body }}`
    - Email type:
      - `text`
    - Add Gmail OAuth2 credentials with send permission.
    - Connect **Generate weekly report → Email weekly report**.

19. **Add sticky notes for maintainability**
    - Add notes reflecting the original structure:
      - overall workflow overview
      - data collection
      - analysis & storage
      - notifications
      - reporting
    - Include setup reminders for credentials, variables, and the Notion schema.

20. **Test with manual execution**
    - Verify:
      - the schedule trigger can be replaced temporarily by manual execution during setup
      - URL arrays and names are populated correctly
      - `current_index` resolves to a valid entry
      - HTTP response body is available
      - Code node can load `cheerio`
      - Claude returns usable output
      - Notion page creation succeeds
      - Gmail sends correctly
      - Slack message posts correctly

21. **Production improvements strongly recommended**
    - Replace the placeholder previous hash with a real lookup from Notion or another datastore.
    - Implement a proper loop over all competitors/URLs.
    - Normalize and validate LLM JSON output.
    - Add retry/error handling for HTTP, Anthropic, Slack, Gmail, and Notion.
    - Add anti-bot-safe headers or scraping fallbacks if competitor sites block requests.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Research competitors and generate market insights | Workflow purpose |
| This workflow automatically monitors competitor websites weekly, detects content changes using hash comparison, analyzes updates with Claude AI, and stores insights in Notion. Important changes trigger immediate Slack notifications, while comprehensive weekly reports are emailed to your team. | Main workflow overview |
| Configure competitor URLs in the Set node - add websites, pricing pages, and feature pages to monitor | Setup guidance |
| Set up credentials: Anthropic Claude API, Notion API, Slack API, Gmail OAuth2 | Setup guidance |
| Create Notion database with properties: Competitor (text), URL (url), Content Hash (text), AI Analysis (text), Scan Date (date) | Setup guidance |
| Set environment variables: `NOTION_COMPETITOR_DB_ID`, `SLACK_ALERTS_CHANNEL`, `TEAM_EMAIL_LIST` | Setup guidance |
| Test the workflow with manual execution before activating the weekly schedule | Setup guidance |
| Adjust the schedule trigger for different monitoring frequencies | Customization guidance |
| Modify urgent keywords in the priority evaluation node | Customization guidance |
| Customize the Claude AI prompt for specific analysis focus areas | Customization guidance |
| Add more competitor URLs or change notification thresholds | Customization guidance |

## Additional implementation note
The exported workflow is conceptually complete but technically incomplete in several places:
- the competitor arrays are not actually populated in the Set node export
- the loop/index logic is not fully implemented
- the content-change comparison uses a placeholder hash rather than a database lookup
- the HTML extraction code contains incorrect node-reference syntax for `url` and `competitor_name`

A developer recreating this workflow should treat it as a strong functional blueprint, but should correct those gaps before production use.