Send weekly competitor content digests with RSS, Google Sheets, OpenAI and Gmail

https://n8nworkflows.xyz/workflows/send-weekly-competitor-content-digests-with-rss--google-sheets--openai-and-gmail-19676


# Send weekly competitor content digests with RSS, Google Sheets, OpenAI and Gmail

### 1. Workflow Overview

This workflow automates a weekly competitive intelligence gathering process. It runs on a scheduled basis every Monday, collects recent articles from up to three competitor RSS feeds, archives the data in Google Sheets, uses OpenAI (GPT-4o-mini) to analyze industry trends, content gaps, and opportunities, and delivers a consolidated summary digest via Gmail.

The workflow logic is categorized into the following functional blocks:
- **1.1 Schedule & Configuration:** Initializes the run and establishes global parameters (competitor details, target niche, recipient email, Google Sheet identifiers, and date formatting).
- **1.2 Competitor Processing & Logging:** Iterates through each valid competitor, fetches their RSS XML feed, extracts the latest post titles and metadata, and records them into a Google Sheet.
- **1.3 Aggregation & AI Analysis:** Evaluates whether all competitors have been processed. Once the final competitor is logged, the weekly entries are retrieved from Google Sheets, structured, and sent to an OpenAI agent along with a structured output parser.
- **1.4 Email Delivery & Formatting:** Formats the AI-generated analysis into a plain-text executive summary and dispatches it via Gmail to the configured recipient.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Schedule & Configuration
**Overview:** This block triggers the workflow automatically every week and establishes global environmental variables, sheet names, and competitor definitions.

- **Nodes Involved:**
  - `Every Monday at 8am`
  - `Set config`
  - `Build the competitor list`

##### Node Details:
- **Every Monday at 10am / Every Monday at 8am**
  - **Type & Technical Role:** `n8n-nodes-base.scheduleTrigger` (v1.2) — Execution trigger.
  - **Configuration:** Cron expression set to run weekly (`0 8 * * 1`).
  - **Expressions / Variables:** None.
  - **Connections:** Input: None | Output: `Set config`.
  - **Version Requirements:** v1.2+.
  - **Edge Cases / Failure Types:** None (no external credentials required).

- **Set config**
  - **Type & Technical Role:** `n8n-nodes-base.set` (v3.4) — Data assignment node.
  - **Configuration:** Assigns manual string values for competitor names, RSS URLs, target niche, recipient email, sender name, Google Sheet ID, and sheet tab name.
  - **Expressions / Variables:** 
    - `weekLabel`: `={{ $now.toFormat('dd MMM yyyy') }}`
  - **Connections:** Input: `Every Monday at 8am` | Output: `Build the competitor list`.
  - **Version Requirements:** v3.4+.
  - **Edge Cases / Failure Types:** Invalid Sheet ID format or misconfigured variable placeholders (`YOUR_...`).

- **Build the competitor list**
  - **Type & Technical Role:** `n8n-nodes-base.code` (v2) — JavaScript data transformer.
  - **Configuration:** Takes the config payload, maps up to 3 competitors into individual items, filters out entries with blank URLs or placeholder strings, and injects execution flags.
  - **Expressions / Variables:** JavaScript logic utilizing `$input.first().json`.
  - **Connections:** Input: `Set config` | Output: `Fetch the RSS feed`.
  - **Version Requirements:** v2+.
  - **Edge Cases / Failure Types:** Throws an explicit JavaScript error if zero valid competitor URLs are detected.

---

#### Block 1.2: Competitor Processing & Logging
**Overview:** This block executes sequentially for each competitor, downloads public RSS feeds, parses the XML strings into structured post titles, and permanently records the entries into Google Sheets.

- **Nodes Involved:**
  - `Fetch the RSS feed`
  - `Read the posts from the feed`
  - `Log the posts`

##### Node Details:
- **Fetch the RSS feed**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` (v4.2) — HTTP client.
  - **Configuration:** Performs a GET request on the dynamic RSS URL with a 15-second timeout and text response handling. Error handling set to continue regular output.
  - **Expressions / Variables:** `={{ $json.rssUrl }}`
  - **Connections:** Input: `Build the competitor list` | Output: `Read the posts from the feed`.
  - **Version Requirements:** v4.2+.
  - **Edge Cases / Failure Types:** Timeout errors, DNS resolution failures, or target websites blocking bot User-Agents. Managed via `continueRegularOutput`.

- **Read the posts from the feed**
  - **Type & Technical Role:** `n8n-nodes-base.code` (v2) — JavaScript data parser.
  - **Configuration:** Parses raw RSS 2.0 (`<item>`) or Atom (`<entry>`) XML strings, extracts up to 5 recent post titles and publication dates while handling CDATA blocks.
  - **Expressions / Variables:** Reads input from `$input.first().json` and correlates with `$('[Build the competitor list]').item.json`.
  - **Connections:** Input: `Fetch the RSS feed` | Output: `Log the posts`.
  - **Version Requirements:** v2+.
  - **Edge Cases / Failure Types:** Malformed XML or empty response strings return an empty array with a `parseNote` rather than breaking the execution flow.

- **Log the posts**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheets` (v4.5) — Google Sheets integration.
  - **Configuration:** Appends a new row to the specified Google Sheets document containing week labels, competitor names, extracted post titles (1 to 5), post counts, and timestamps. Cell format set to `USER_ENTERED`.
  - **Expressions / Variables:** 
    - Document ID: `={{ $json.sheetId }}`
    - Sheet Name: `={{ $json.sheetName }}`
    - Mapped columns use item properties for `Week`, `Logged At`, `Competitor`, `Post Count`, and `Post 1` through `Post 5 Title`.
  - **Connections:** Input: `Read the posts from the feed` | Output: `Last competitor?`.
  - **Credentials:** Google Sheets OAuth2.
  - **Edge Cases / Failure Types:** API rate limits, invalid sheet names, or missing header rows in the target spreadsheet.

---

#### Block 1.3: Aggregation & AI Analysis
**Overview:** This block checks if the active item is the last competitor in the loop. Once confirmed, it retrieves all logged records for the current week, formats the data, and passes it to an OpenAI advanced agent supported by a strict JSON output parser.

- **Nodes Involved:**
  - `Last competitor?`
  - `More competitors to go`
  - `Read this week's posts`
  - `Prepare the data for GPT`
  - `Chat model`
  - `Output Parser - Digest sections`
  - `Write the digest`

##### Node Details:
- **Last competitor?**
  - **Type & Technical Role:** `n8n-nodes-base.if` (v2.2) — Conditional branching node.
  - **Configuration:** Evaluates whether the current item represents the final competitor in the processing queue.
  - **Expressions / Variables:** `={{ $('Build the competitor list').item.json.isLastCompetitor }}`
  - **Connections:** Input: `Log the posts` | Outputs: 
    - True branch -> `Read this week's posts`
    - False branch -> `More competitors to go`
  - **Version Requirements:** v2.2+.
  - **Edge Cases / Failure Types:** None.

- **More competitors to go**
  - **Type & Technical Role:** `n8n-nodes-base.set` (v3.4) — Data assignment node.
  - **Configuration:** Dummy assignment node representing the FALSE branch termination point for non-final competitors.
  - **Connections:** Input: `Last competitor?` (False) | Output: None.

- **Read this week's posts**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheets` (v4.5) — Google Sheets integration.
  - **Configuration:** Retrieves all rows from the specified Google Sheet tab.
  - **Expressions / Variables:** 
    - Document ID: `={{ $('Build the competitor list').item.json.sheetId }}`
  - **Connections:** Input: `Last competitor?` (True) | Output: `Prepare the data for GPT`.
  - **Credentials:** Google Sheets OAuth2.
  - **Edge Cases / Failure Types:** Empty dataset returns if no rows match or sheet access is denied.

- **Prepare the data for GPT**
  - **Type & Technical Role:** `n8n-nodes-base.code` (v2) — JavaScript data transformer.
  - **Configuration:** Consolidates all rows retrieved from Google Sheets into a structured multi-line text block representing individual competitor publications for the week.
  - **Connections:** Input: `Read this week's posts` | Output: `Write the digest`.

- **Chat model**
  - **Type & Technical Role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` (v1.2) — Language model sub-node.
  - **Configuration:** Configured to use model `gpt-4o-mini` with a temperature of `0.4` and a maximum token limit of `1000`.
  - **Connections:** Connected via AI language model connection to `Write the digest`.
  - **Credentials:** OpenAI API.
  - **Edge Cases / Failure Types:** API key expiration, billing limits, or rate-limiting errors.

- **Output Parser - Digest sections**
  - **Type & Technical Role:** `@n8n/n8n-nodes-langchain.outputParserStructured` (v1.3) — LangChain structured output parser.
  - **Configuration:** Enforces a manual JSON schema requiring 5 specific properties: `weekSummary`, `competitorHighlights`, `trendingTopics`, `contentGaps`, and `urgentOpportunity`.
  - **Connections:** Connected via AI output parser connection to `Write the digest`.

- **Write the digest**
  - **Type & Technical Role:** `@n8n/n8n-nodes-langchain.agent` (v1.7) — LangChain AI Agent.
  - **Configuration:** Processes the prompt injected with competitor data and niche definitions, forcing strict adherence to the output schema via the attached parser.
  - **Expressions / Variables:** Uses prompt text referencing `{{ $json.yourNiche }}`, `{{ $json.competitorData }}`, and `{{ $json.weekLabel }}`.
  - **Connections:** Input: `Prepare the data for GPT` | Output: `Build the email`. Links to `Chat model` and `Output Parser - Digest sections`.

---

#### Block 1.4: Email Delivery & Formatting
**Overview:** This block constructs a plain-text layout using the structured JSON returned by the AI agent and sends the completed intelligence brief to the designated recipient via Gmail.

- **Nodes Involved:**
  - `Build the email`
  - `Send the digest`

##### Node Details:
- **Build the email**
  - **Type & Technical Role:** `n8n-nodes-base.code` (v2) — JavaScript data transformer.
  - **Configuration:** Reads the structured JSON output from the AI agent (`$input.first().json.output`) and assembles a human-readable plain-text email containing a week-in-review summary, competitor highlights, trending topics, content gaps, and an urgent action item.
  - **Connections:** Input: `Write the digest` | Output: `Send the digest`.
  - **Edge Cases / Failure Types:** Throws an explicit error if the AI output object is undefined or missing.

- **Send the digest**
  - **Type & Technical Role:** `n8n-nodes-base.gmail` (v2.1) — Gmail integration.
  - **Configuration:** Sends a plain-text email message. Attribution appending is disabled.
  - **Expressions / Variables:**
    - Recipient: `={{ $json.recipientEmail }}`
    - Subject: `={{ $json.emailSubject }}`
    - Message: `={{ $json.emailBody }}`
  - **Connections:** Input: `Build the email` | Output: None.
  - **Credentials:** Gmail OAuth2.
  - **Edge Cases / Failure Types:** OAuth token revocation, sending limit restrictions, or invalid recipient address formats.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Every Monday at 8am` | `scheduleTrigger` | Schedule trigger | None | `Set config` | Runs every Monday at 8AM automatically.<br><br>Change the schedule if needed:<br>- Every Friday: 0 8 * * 5<br>- Twice a week: 0 8 * * 1,4<br>- Every day: 0 8 * * *<br><br>No credentials needed. |
| `Set config` | `set` | Configuration assignment | `Every Monday at 8am` | `Build the competitor list` | REPLACE THESE VALUES:<br><br>1. competitor1RssUrl (pre-filled with Ahrefs as example)<br>   Replace with your actual competitor's RSS feed<br>   How to find it: try yourcompetitor.com/feed or yourcompetitor.com/rss<br>   For example: https://backlinko.com/feed<br><br>2. competitor2RssUrl (pre-filled with Search Engine Journal)<br>   Replace with your second competitor<br><br>3. competitor3Name and competitor3RssUrl<br>   Optional — leave URL blank to skip this competitor<br><br>4. yourNiche<br>   Your industry — GPT uses this to find relevant content gaps<br>   Example: SEO for law firms, e-commerce marketing, SaaS content marketing<br><br>5. recipientEmail<br>   Who receives the Monday morning digest<br>   Example: you@example.com<br><br>6. senderName<br>   Your name for the email sign-off<br><br>7. sheetId<br>   Open your Google Sheet in browser<br>   URL: docs.google.com/spreadsheets/d/1ABC123/edit<br>   Copy the part between /d/ and /edit<br>   Create a tab called: Competitor Content<br>   Row 1 headers:<br>   Week \| Competitor \| Post 1 Title \| Post 2 Title \| Post 3 Title \| Post 4 Title \| Post 5 Title \| Post Count \| Logged At |
| `Build the competitor list` | `code` | Competitor array builder & filter | `Set config` | `Fetch the RSS feed` | Creates one item per competitor from the config.<br>Filters out competitors with blank URLs.<br>Adds isLastCompetitor = true on the final competitor.<br>This flag is used later to trigger the Gmail digest after all are processed.<br>No changes needed. |
| `Fetch the RSS feed` | `httpRequest` | RSS XML downloader | `Build the competitor list` | `Read the posts from the feed` | Fetches the RSS XML from the competitor's blog feed.<br><br>timeout: 15 seconds<br>onError: continueRegularOutput — if a feed fails, the workflow continues<br>and the Parse node returns empty posts with an error note<br><br>If a feed fails consistently:<br>- Check the RSS URL is correct<br>- Some sites block bot requests — try adding a User-Agent header<br><br>No credentials needed — RSS feeds are public. |
| `Read the posts from the feed` | `code` | XML parser & post extractor | `Fetch the RSS feed` | `Log the posts` | Parses the RSS XML and extracts the 5 most recent post titles.<br><br>Handles both RSS 2.0 (item tags) and Atom (entry tags) formats.<br>Handles both CDATA and plain text title formats.<br>If the feed fails or returns no posts, outputs empty posts gracefully<br>so the workflow can continue without breaking.<br><br>No changes needed. |
| `Log the posts` | `googleSheets` | Database archiver | `Read the posts from the feed` | `Last competitor?` | Logs this competitor's posts to Google Sheets permanently.<br><br>How to connect:<br>1. Go to console.cloud.google.com and enable Google Sheets API<br>2. In n8n: Settings → Credentials → Add New → Google Sheets OAuth2<br>3. Authenticate and connect to this node<br><br>Sheet tab: Competitor Content<br>Row 1 headers:<br>Week \| Competitor \| Post 1 Title \| Post 2 Title \| Post 3 Title \| Post 4 Title \| Post 5 Title \| Post Count \| Logged At<br><br>Every Monday adds rows for all competitors.<br>Filter by Week to see any past week's data.<br>Filter by Competitor to track one source over time. |
| `Last competitor?` | `if` | Loop termination checker | `Log the posts` | `Read this week's posts`, `More competitors to go` | Checks if all competitors have been processed.<br><br>TRUE — last competitor logged — now build and send the digest<br>FALSE — more competitors still being processed — continue silently<br><br>Reads isLastCompetitor from Build Competitors List directly.<br>All previous competitors' posts are already in Google Sheets by the time this fires.<br>No changes needed. |
| `More competitors to go` | `set` | Loop continuation placeholder | `Last competitor?` | None | FALSE branch of Last Competitor? node.<br>This competitor's posts are logged.<br>More competitors still being processed.<br>Workflow continues silently.<br>No changes needed. |
| `Read this week's posts` | `googleSheets` | Weekly data retriever | `Last competitor?` | `Prepare the data for GPT` | Reads only this week's rows from the Competitor Content sheet.<br><br>Connect the same Google Sheets credential here.<br><br>Filter: Week column = current week label<br>This returns all competitors' posts that were logged in this run.<br>These rows are then sent to GPT for analysis. |
| `Prepare the data for GPT` | `code` | Analysis payload formatter | `Read this week's posts` | `Write the digest` | Reads all this week's rows from Google Sheets.<br><br>Builds a clean structured text showing what each competitor published.<br>This becomes the input for GPT analysis.<br>No changes needed. |
| `Write the digest` | `agent` | AI Competitive Analyst Agent | `Prepare the data for GPT` | `Build the email` | GPT analyzes all competitor posts from this week and returns 5 structured fields:<br><br>1. weekSummary — 2 to 3 sentences on what the competitive landscape looks like<br>2. competitorHighlights — top post and strategy angle per competitor<br>3. trendingTopics — 3 topics appearing across multiple competitors<br>4. contentGaps — 3 specific content ideas no competitor covered<br>5. urgentOpportunity — the one piece to publish this week<br><br>The Digest Output Parser below ensures clean JSON every time.<br>Output is in $json.output |
| `Chat model` | `lmChatOpenAi` | OpenAI Model provider | None | `Write the digest` | Connect your OpenAI API credential here.<br><br>1. Go to platform.openai.com/api-keys<br>2. Create a new key<br>3. In n8n: Settings → Credentials → Add New → OpenAI API<br>4. Connect to this node<br><br>Model: gpt-4o-mini<br>temperature: 0.4 for creative but consistent analysis<br>Cost: about $0.003 per weekly digest |
| `Output Parser - Digest sections` | `outputParserStructured` | JSON schema enforcement | None | `Write the digest` | Schema: 5 required fields for the competitive digest.<br>Ensures GPT always returns consistent clean data.<br>No changes needed. |
| `Build the email` | `code` | Email body assembler | `Write the digest` | `Send the digest` | Reads GPT output from $json.output.<br>Reads config from Format Data for GPT node.<br><br>Builds one plain text email with 5 sections:<br>1. Week in review — overall landscape summary<br>2. Competitor highlights — top post and angle per competitor<br>3. trending topics — what the market is talking about<br>4. Content gaps — what no competitor covered (your opportunity)<br>5. Urgent content opportunity — the one piece to publish now<br>No changes needed. |
| `Send the digest` | `gmail` | Email transmission node | `Build the email` | None | Sends the weekly competitive content digest.<br><br>How to connect:<br>1. Go to console.cloud.google.com and enable Gmail API<br>2. In n8n: Settings → Credentials → Add New → Gmail OAuth2<br>3. Authenticate and connect to this node<br><br>Email arrives every Monday morning with:<br>- What each competitor published this week<br>- Which topics are trending across competitors<br>- Content gaps your competitors missed<br>- One urgent content opportunity to act on |

---

### 4. Reproducing the Workflow from Scratch

To rebuild this workflow manually from scratch in n8n, follow these sequential configuration steps:

1. **Create the Trigger Node:**
   - Add a **Schedule Trigger** node. Set the interval to **Cron Expression**: `0 8 * * 1` (Every Monday at 8:00 AM). Name it `Every Monday at 8am`.

2. **Create the Configuration Node:**
   - Add a **Set** node named `Set config`.
   - Add the following string assignments:
     - `competitor1Name`: `Ahrefs Blog`
     - `competitor1RssUrl`: `https://ahrefs.com/blog/feed`
     - `competitor2Name`: `Search Engine Journal`
     - `competitor2RssUrl`: `https://searchenginejournal.com/feed`
     - `competitor3Name`: `YOUR_THIRD_COMPETITOR_NAME_OR_LEAVE_BLANK`
     - `competitor3RssUrl`: `YOUR_THIRD_COMPETITOR_RSS_URL_OR_LEAVE_BLANK`
     - `yourNiche`: `SEO and digital marketing`
     - `recipientEmail`: `YOUR_EMAIL_ADDRESS`
     - `senderName`: `YOUR_NAME`
     - `sheetId`: `YOUR_GOOGLE_SHEET_ID`
     - `sheetName`: `Competitor Content`
     - `weekLabel`: `={{ $now.toFormat('dd MMM yyyy') }}`
   - Connect `Every Monday at 8am` to `Set config`.

3. **Create the Competitor Array Builder:**
   - Add a **Code** node named `Build the competitor list`.
   - Insert JavaScript to extract configuration values, filter out items containing placeholders or empty URLs, and calculate an `isLastCompetitor` boolean flag for each item.
   - Connect `Set config` to `Build the competitor list`.

4. **Fetch & Parse RSS Feeds:**
   - Add an **HTTP Request** node named `Fetch the RSS feed`. Set Method to `GET`, URL to `={{ $json.rssUrl }}`, timeout to `15000`ms, response format to `Text`, and error handling to **Continue Regular Output**. Connect `Build the competitor list` to it.
   - Add a **Code** node named `Read the posts from the feed`. Insert JavaScript that extracts items or entries matching RSS/Atom XML tags, isolates titles/dates, and outputs up to 5 posts. Connect `Fetch the RSS feed` to it.

5. **Log Entries to Google Sheets:**
   - Add a **Google Sheets** node named `Log the posts`. Operation: `Append`. Set Document ID to `={{ $json.sheetId }}` and Sheet Name to `={{ $json.sheetName }}`. Map columns `Week`, `Logged At`, `Competitor`, `Post Count`, and `Post 1 Title` through `Post 5 Title` using corresponding item expressions.
   - Configure **Google Sheets OAuth2** credentials.
   - Connect `Read the posts from the feed` to `Log the posts`.

6. **Implement Loop Evaluation:**
   - Add an **If** node named `Last competitor?`. Set condition evaluation to verify if `={{ $('Build the competitor list').item.json.isLastCompetitor }}` equals boolean `true`.
   - Connect `Log the posts` to `Last competitor?`.
   - Add a **Set** node named `More competitors to go` on the FALSE branch to terminate uncompleted loop iterations silently.

7. **Retrieve & Format Data for AI Analysis:**
   - On the TRUE branch of `Last competitor?`, add a **Google Sheets** node named `Read this week's posts`. Operation: `Get Many` / `Get All`, specifying the document ID from config. Use the same Google Sheets credentials.
   - Add a **Code** node named `Prepare the data for GPT` to aggregate all retrieved rows into a formatted competitor publication summary. Connect `Read this week's posts` to it.

8. **Configure the OpenAI Agent & Output Parser:**
   - Add an **AI Agent** node named `Write the digest`. Set prompt type to `Define` and paste the competitive analysis prompt containing variable injections like `{{ $json.yourNiche }}` and `{{ $json.competitorData }}`.
   - Add a **OpenAI Chat Model** sub-node (`Chat model`), select model `gpt-4o-mini`, set temperature to `0.4` and max tokens to `1000`. Connect it via the model input of `Write the digest`. Configure **OpenAI API** credentials.
   - Add a **Structured Output Parser** sub-node (`Output Parser - Digest sections`), configure manual schema properties (`weekSummary`, `competitorHighlights`, `trendingTopics`, `contentGaps`, `urgentOpportunity`), and link it to the agent's output parser input.
   - Connect `Prepare the data for GPT` to `Write the digest`.

9. **Build & Send the Email Digest:**
   - Add a **Code** node named `Build the email`. Insert JavaScript to map properties from `$input.first().json.output` into structured email body sections (`emailSubject`, `emailBody`, `recipientEmail`). Connect `Write the digest` to it.
   - Add a **Gmail** node named `Send the digest`. Set parameters to send email with `sendTo` set to `={{ $json.recipientEmail }}`, subject to `={{ $json.emailSubject }}`, and message body to `={{ $json.emailBody }}`. Disable attribution.
   - Configure **Gmail OAuth2** credentials.
   - Connect `Build the email` to `Send the digest`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Google Cloud Console API requirements | Ensure both the **Google Sheets API** and **Gmail API** are enabled in your Google Cloud Project before generating OAuth2 credentials. |
| Google Sheet Tab Header Structure | The target spreadsheet must contain a tab named `Competitor Content` with exact row 1 headers: `Week`, `Competitor`, `Post 1 Title`, `Post 2 Title`, `Post 3 Title`, `Post 4 Title`, `Post 5 Title`, `Post Count`, `Logged At`. |
| Finding Competitor RSS Feeds | Most blogging platforms expose RSS feeds natively via paths like `/feed`, `/rss`, or `/feed.xml` (e.g., `https://backlinko.com/feed`). |