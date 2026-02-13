Track job applications and prep interviews with Notion and GPT-5-mini

https://n8nworkflows.xyz/workflows/track-job-applications-and-prep-interviews-with-notion-and-gpt-5-mini-13104


# Track job applications and prep interviews with Notion and GPT-5-mini

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name:** AI job application tracker and interview prep assistant  
**User-provided title:** Track job applications and prep interviews with Notion and GPT-5-mini

**Purpose:**  
Automate job application tracking and interview preparation. The workflow accepts a job application via (1) a webhook form containing a job URL or (2) a forwarded Gmail message. It extracts job details with GPT-5-mini, generates interview prep materials, stores everything in a Notion database, and sends confirmations. A separate scheduled path checks Notion daily and sends Slack reminders for follow-ups.

### 1.1 Input Reception (two entry points)
- Webhook (manual form submission with job URL)
- Gmail Trigger (unread forwarded application emails)

### 1.2 Routing & Normalization
- Routes based on whether the payload looks like form data or email content.

### 1.3 Data Acquisition (URL scraping)
- If a URL was provided, fetches the posting content via a Jina AI “reader” proxy.

### 1.4 AI Extraction (structured job details)
- GPT-5-mini extracts structured fields (company, role, requirements, etc.) as JSON.

### 1.5 Parsing, Enrichment & Tracking Metadata
- Code node parses JSON, adds applied date, follow-up date, status, and jobUrl.

### 1.6 AI Interview Preparation
- GPT-5-mini generates interview questions, talking points, and negotiation notes.
- **Important:** This step references a node **“Research Company Online”** that does not exist in the provided workflow (see Block 2.6).

### 1.7 Persistence & User Feedback
- Saves the result to Notion.
- Responds to webhook requests and/or emails the candidate (email confirmation path).

### 1.8 Daily Follow-up Reminders
- A schedule trigger runs daily and checks Notion for follow-ups, then posts to Slack.

---

## 2. Block-by-Block Analysis

### Block 2.1 — Input Reception (Form + Email)
**Overview:** Receives application data either from a webhook form (job URL) or Gmail (forwarded confirmations). Both feed into a shared routing node.

**Nodes involved:**
- Receive Job Application Form
- Check for Forwarded Applications

#### Node: Receive Job Application Form
- **Type / role:** Webhook (n8n-nodes-base.webhook) — HTTP entry point for manual submissions.
- **Key configuration:**
  - Method: **POST**
  - Path: **/job-application**
  - Response mode: **Respond with a “Respond to Webhook” node** (responseNode)
- **Inputs / outputs:**
  - Output → **Route by Input Source**
- **Key data expected:**
  - `{{$json.body.jobUrl}}` should be present for form submissions.
- **Edge cases / failures:**
  - Missing `body.jobUrl` → routing might not match any rule; execution ends without downstream processing.
  - If used publicly, consider auth/signature to prevent abuse.

#### Node: Check for Forwarded Applications
- **Type / role:** Gmail Trigger (n8n-nodes-base.gmailTrigger) — polls inbox for new items.
- **Key configuration:**
  - Filter: **unread**
  - Poll: **every minute**
- **Inputs / outputs:**
  - Output → **Route by Input Source**
- **Key data expected:**
  - `{{$json.textPlain}}` used by routing and AI extraction.
  - `from.value[0].address` used later to send confirmation.
- **Edge cases / failures:**
  - Gmail OAuth/permissions issues (token expired, scope missing).
  - Messages without `textPlain` (HTML-only) may not route correctly.
  - Polling every minute may hit rate limits in some environments.

**Sticky note coverage (context):**
- “📥 Input Methods … Manual form … Email forward …” applies conceptually to both input nodes and the routing node.

---

### Block 2.2 — Route by Input Source
**Overview:** Determines whether the incoming item is from the form or from Gmail by checking presence of jobUrl vs email text.

**Nodes involved:**
- Route by Input Source

#### Node: Route by Input Source
- **Type / role:** Switch (n8n-nodes-base.switch) — conditional branching.
- **Key configuration (rules):**
  - Output **“From Form”** if `{{$json.body?.jobUrl}}` is **not empty**
  - Output **“From Email”** if `{{$json.textPlain}}` is **not empty**
- **Inputs / outputs:**
  - Input: from **Receive Job Application Form** and **Check for Forwarded Applications**
  - Output 0 (“From Form”) → **Scrape Job Posting URL**
  - Output 1 (“From Email”) → **Extract Job Details with AI**
- **Edge cases / failures:**
  - If both are empty, nothing matches and workflow stops.
  - If both are present (rare), only the first matching rule path may be taken depending on n8n switch behavior/settings.
  - Strict type validation may cause unexpected non-string values to fail matches.

---

### Block 2.3 — Data Acquisition (Scrape Job Posting URL)
**Overview:** Fetches the job posting content via a Jina reader endpoint, returning text for AI extraction.

**Nodes involved:**
- Scrape Job Posting URL

#### Node: Scrape Job Posting URL
- **Type / role:** HTTP Request (n8n-nodes-base.httpRequest) — fetch posting text.
- **Key configuration:**
  - URL expression: `https://r.jina.ai/{{ $json.body.jobUrl }}`
  - Method: **POST**
  - Response format: **text**
- **Inputs / outputs:**
  - Input: “From Form” branch from **Route by Input Source**
  - Output → **Extract Job Details with AI**
- **Edge cases / failures:**
  - Invalid/missing `body.jobUrl` → malformed URL.
  - Target site blocks scraping; Jina endpoint may return error or partial content.
  - Very large pages: response size and downstream token limits.
  - Using POST for a “reader” endpoint may work but can be brittle if the service expects GET.

**Sticky note coverage (context):**
- “🔍 Data Extraction … Extracts job details from URL or email content using AI” applies to this node and the AI extraction node.

---

### Block 2.4 — AI Extraction (Job Details)
**Overview:** Uses GPT-5-mini to extract structured job data as strict JSON from either scraped content or email text.

**Nodes involved:**
- Extract Job Details with AI

#### Node: Extract Job Details with AI
- **Type / role:** OpenAI (LangChain) node (@n8n/n8n-nodes-langchain.openAi) — LLM call.
- **Key configuration choices:**
  - Model: **gpt-5-mini**
  - System message: instructs “Always respond with valid JSON only”
  - Prompt content uses:
    - Primary content: `{{ $json.data || $json.textPlain }}`
    - Also references scraped content via: `{{ $('Scrape Job Posting URL').item.json.data }}`
- **Inputs / outputs:**
  - Input: from **Scrape Job Posting URL** (form path) or directly from **Route by Input Source** (email path)
  - Output → **Parse and Structure Job Data**
- **Variables / expressions of note:**
  - `$('Scrape Job Posting URL').item.json.data` assumes that node executed and has `.data`. On the email path, it did not execute.
- **Edge cases / failures:**
  - **Branch-dependent reference risk:** On the email path, referencing `Scrape Job Posting URL` may return undefined and could degrade prompt quality or error depending on n8n expression handling.
  - Model may still produce non-JSON output; downstream parser handles some markdown fences but not arbitrary text.
  - Token limits if scraped content is long.

---

### Block 2.5 — Parsing, Enrichment & Metadata
**Overview:** Converts the AI output into a clean object, adds dates (applied + follow-up), status, and jobUrl.

**Nodes involved:**
- Parse and Structure Job Data

#### Node: Parse and Structure Job Data
- **Type / role:** Code (n8n-nodes-base.code) — JavaScript post-processing.
- **Key configuration choices:**
  - `DAYS_UNTIL_FOLLOWUP = 7` controls follow-up date.
  - Parses `input.message.content` as JSON string and strips ```json fences.
  - On parse error: returns `{ error: true, rawResponse: ... }` instead of throwing.
  - Adds:
    - `appliedDate` = today (YYYY-MM-DD)
    - `followUpDate` = today + 7 days
    - `status` = `"Applied"`
    - `jobUrl` = `$('Receive Job Application Form').first().json.body?.jobUrl || 'From email'`
- **Inputs / outputs:**
  - Input: **Extract Job Details with AI**
  - Output → **Generate Interview Prep Materials**
- **Edge cases / failures:**
  - If AI returns valid JSON but not matching expected schema, Notion mapping (later) may fail or store incomplete data.
  - `$('Receive Job Application Form')...` reference will be missing on email-trigger runs; fallback “From email” is fine, but any other form-specific fields will be absent.
  - Timezone: uses server timezone when creating dates; might not match user locale.

---

### Block 2.6 — AI Interview Prep
**Overview:** Generates interview prep content from the extracted job data and “company research”. However, the workflow references a missing node for research.

**Nodes involved:**
- Generate Interview Prep Materials  
*(Referenced but missing: “Research Company Online”)*

#### Node: Generate Interview Prep Materials
- **Type / role:** OpenAI (LangChain) node (@n8n/n8n-nodes-langchain.openAi) — LLM generation.
- **Key configuration choices:**
  - Model: **gpt-5-mini**
  - Prompt includes:
    - Company: `{{$json.company}}`
    - Role: `{{$json.role}}`
    - Requirements/responsibilities joined from arrays
    - Company research snippet:  
      `{{ $('Research Company Online').first().json.data?.substring(0, 3000) || 'No additional research available' }}`
- **Inputs / outputs:**
  - Input: **Parse and Structure Job Data**
  - Output → **Save Application to Notion**
- **Critical integration issue:**
  - **Node “Research Company Online” does not exist in the provided JSON**, so the expression will resolve to an error or undefined depending on runtime behavior.
  - Expected fix: add a node that performs company research (HTTP Request / SerpAPI / Tavily / Apify / browserless scrape) and name it **exactly** “Research Company Online”, or update the expression to reference the correct node.
- **Edge cases / failures:**
  - If requirements/responsibilities aren’t arrays, `.join()` could fail (the prompt guards with optional chaining, so it’s mostly safe).
  - Token limits if research text is long (substring to 3000 helps).

**Sticky note coverage (context):**
- “🎯 Interview Prep … AI generates customized questions and talking points” applies to this node.

---

### Block 2.7 — Persistence to Notion & Confirmations
**Overview:** Stores the application and prep materials in Notion, then sends a response to the webhook and/or emails the applicant.

**Nodes involved:**
- Save Application to Notion
- Prepare Success Response
- Send Confirmation Response
- Send Email Confirmation

#### Node: Save Application to Notion
- **Type / role:** Notion (n8n-nodes-base.notion) — create a database page.
- **Key configuration choices:**
  - Resource: **databasePage**
  - Operation: implied **create** (default when not specified in UI for databasePage)
  - Database ID: **not set** (empty in JSON)
- **Inputs / outputs:**
  - Input: **Generate Interview Prep Materials**
  - Output → **Prepare Success Response**
- **Edge cases / failures:**
  - Missing Notion credentials or integration not shared with the database.
  - DatabaseId is empty → node will fail until configured.
  - Property mapping is not shown in the JSON snippet (likely configured in UI but absent here); if not configured, it will create a page with minimal/default fields or fail validation.
  - Notion rate limits for high volume.

#### Node: Prepare Success Response
- **Type / role:** Set (n8n-nodes-base.set) — formats a summary message.
- **Key configuration choices:**
  - Creates `summary` string with:
    - Company/role/location/salary/followUpDate from **Parse and Structure Job Data**
- **Inputs / outputs:**
  - Input: **Save Application to Notion**
  - Output → **Send Confirmation Response** and **Send Email Confirmation** (both in parallel)
- **Edge cases / failures:**
  - If parse node output is error object, fields will be missing and summary will contain blanks/undefined.

#### Node: Send Confirmation Response
- **Type / role:** Respond to Webhook (n8n-nodes-base.respondToWebhook) — returns HTTP response to the form submitter.
- **Key configuration:**
  - Respond with: **text**
  - Body: `{{$json.summary}}`
- **Inputs / outputs:**
  - Input: **Prepare Success Response**
  - Output: ends webhook execution.
- **Edge cases / failures:**
  - Only meaningful for webhook-triggered runs; for Gmail-triggered runs there is no requester waiting for a response (node may be unnecessary or can be conditioned).

#### Node: Send Email Confirmation
- **Type / role:** Gmail (n8n-nodes-base.gmail) — sends email to the forwarding sender.
- **Key configuration:**
  - To: `{{ $('Check for Forwarded Applications').first().json.from?.value?.[0]?.address }}`
  - Subject includes company name.
  - Body includes summary fields plus a truncated excerpt of interview prep:  
    `{{ $('Generate Interview Prep Materials').first().json.message?.content?.substring(0, 1500) }}...`
- **Inputs / outputs:**
  - Input: **Prepare Success Response**
  - Output: end.
- **Edge cases / failures:**
  - On form-triggered runs, `Check for Forwarded Applications` did not execute → recipient resolves undefined; Gmail node will fail unless you add a conditional branch.
  - Email content length and formatting; `message.content` may be missing if OpenAI node returns a different schema.

**Sticky note coverage (context):**
- “📊 Tracking & Alerts … Saves to Notion and schedules follow-up reminders” applies to Notion + follow-up scheduling/reminder logic (Notion save + scheduled block).

---

### Block 2.8 — Daily Follow-up Reminders (Notion → Slack)
**Overview:** Every day at 9 AM, retrieves applications due for follow-up from Notion; if any exist, posts a Slack reminder listing them.

**Nodes involved:**
- Daily Follow-up Check
- Get Applications Due for Follow-up
- Has Pending Follow-ups?
- Send Slack Reminder

#### Node: Daily Follow-up Check
- **Type / role:** Schedule Trigger (n8n-nodes-base.scheduleTrigger) — time-based entry point.
- **Key configuration:**
  - Triggers at **09:00** (server timezone).
- **Inputs / outputs:**
  - Output → **Get Applications Due for Follow-up**
- **Edge cases / failures:**
  - Timezone mismatch if server timezone differs from user expectations.

#### Node: Get Applications Due for Follow-up
- **Type / role:** Notion (n8n-nodes-base.notion) — query database pages.
- **Key configuration:**
  - Resource: **databasePage**
  - Operation: **getAll**
  - Database ID: **not set** (empty in JSON)
  - Filter type: **manual** (but actual filter rules are not defined in the JSON shown)
- **Inputs / outputs:**
  - Input: **Daily Follow-up Check**
  - Output → **Has Pending Follow-ups?**
- **Edge cases / failures:**
  - DatabaseId empty → will fail until configured.
  - Missing filter conditions: may return all rows (potential Slack spam) or none, depending on how it’s configured in UI.
  - Notion pagination/limits for large databases.

#### Node: Has Pending Follow-ups?
- **Type / role:** IF (n8n-nodes-base.if) — checks if any items exist.
- **Key configuration:**
  - Condition: `{{$input.all().length}} > 0`
- **Inputs / outputs:**
  - Input: **Get Applications Due for Follow-up**
  - True output → **Send Slack Reminder**
- **Edge cases / failures:**
  - If Notion node errors, IF node may never run (depends on “continue on fail” settings, not indicated).

#### Node: Send Slack Reminder
- **Type / role:** Slack (n8n-nodes-base.slack) — posts a message.
- **Key configuration:**
  - Message builds a bullet list from Notion properties:
    - Company from `properties.Company.title[0].plain_text`
    - Role from `properties.Role.rich_text[0].plain_text`
  - Includes `{{$input.all().length}}` count.
- **Inputs / outputs:**
  - Input: **Has Pending Follow-ups?** (true branch)
- **Edge cases / failures:**
  - Slack auth/webhook misconfiguration.
  - Notion property names must match exactly (**Company**, **Role**) and types must match (Title, Rich text). If your database differs, mapping will show “Unknown/Role”.

**Sticky note coverage (context):**
- “⏰ Daily Follow-up Reminders … Checks Notion daily at 9 AM …” applies to all nodes in this block.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Documentation / canvas annotation |  |  | ## AI Job Application Tracker & Interview Prep Assistant … (includes Notion property list) |
| Sticky Note1 | Sticky Note | Documentation / canvas annotation |  |  | ### 📥 Input Methods … Manual form … Email forward … |
| Sticky Note2 | Sticky Note | Documentation / canvas annotation |  |  | ### 🔍 Data Extraction … |
| Sticky Note4 | Sticky Note | Documentation / canvas annotation |  |  | ### 🎯 Interview Prep … |
| Sticky Note5 | Sticky Note | Documentation / canvas annotation |  |  | ### 📊 Tracking & Alerts … |
| Sticky Note6 | Sticky Note | Documentation / canvas annotation |  |  | ### ⏰ Daily Follow-up Reminders … |
| Receive Job Application Form | Webhook | Receives manual job URL submissions | — | Route by Input Source | ### 📥 Input Methods… |
| Check for Forwarded Applications | Gmail Trigger | Polls unread forwarded application emails | — | Route by Input Source | ### 📥 Input Methods… |
| Route by Input Source | Switch | Branches between form vs email payload | Receive Job Application Form; Check for Forwarded Applications | Scrape Job Posting URL; Extract Job Details with AI | ### 📥 Input Methods… |
| Scrape Job Posting URL | HTTP Request | Retrieves job posting text via Jina reader | Route by Input Source (From Form) | Extract Job Details with AI | ### 🔍 Data Extraction… |
| Extract Job Details with AI | OpenAI (LangChain) | Extracts structured job fields as JSON | Scrape Job Posting URL; Route by Input Source (From Email) | Parse and Structure Job Data | ### 🔍 Data Extraction… |
| Parse and Structure Job Data | Code | Parses AI JSON, adds dates/status/jobUrl | Extract Job Details with AI | Generate Interview Prep Materials |  |
| Generate Interview Prep Materials | OpenAI (LangChain) | Generates interview questions/talking points | Parse and Structure Job Data | Save Application to Notion | ### 🎯 Interview Prep… |
| Save Application to Notion | Notion | Creates a Notion database page | Generate Interview Prep Materials | Prepare Success Response | ### 📊 Tracking & Alerts… |
| Prepare Success Response | Set | Builds human-friendly confirmation summary | Save Application to Notion | Send Confirmation Response; Send Email Confirmation | ### 📊 Tracking & Alerts… |
| Send Confirmation Response | Respond to Webhook | Returns response to webhook caller | Prepare Success Response | — | ### 📊 Tracking & Alerts… |
| Send Email Confirmation | Gmail | Emails confirmation + prep excerpt | Prepare Success Response | — | ### 📊 Tracking & Alerts… |
| Daily Follow-up Check | Schedule Trigger | Starts daily follow-up scan at 9 AM | — | Get Applications Due for Follow-up | ### ⏰ Daily Follow-up Reminders… |
| Get Applications Due for Follow-up | Notion | Queries Notion for items due for follow-up | Daily Follow-up Check | Has Pending Follow-ups? | ### ⏰ Daily Follow-up Reminders… |
| Has Pending Follow-ups? | IF | Checks if any follow-ups exist | Get Applications Due for Follow-up | Send Slack Reminder (true) | ### ⏰ Daily Follow-up Reminders… |
| Send Slack Reminder | Slack | Posts follow-up reminder list to Slack | Has Pending Follow-ups? (true) | — | ### ⏰ Daily Follow-up Reminders… |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name: *AI job application tracker and interview prep assistant*
   - (Optional) Add tags: AI, Productivity, Job Search

2. **Add Notion database (prerequisite)**
   - Create a Notion database with properties (as described in the sticky note):
     - Company (Title)
     - Role (Text or Rich text)
     - Status (Select: Applied, Interviewing, Offer, Rejected)
     - Applied Date (Date)
     - Salary Range (Text)
     - Job URL (URL)
     - Company Research (Text)
     - Interview Prep (Text)
     - Follow Up Date (Date)
     - Notes (Text)
   - Share the database with your Notion integration (Notion credential in n8n).

3. **Add credentials**
   - **OpenAI credential** for the LangChain OpenAI nodes.
   - **Gmail OAuth2** credential for Gmail Trigger + Gmail Send node.
   - **Notion API** credential for Notion nodes.
   - **Slack** credential (bot token or webhook) for posting reminders.

4. **Create the form entry node**
   - Add **Webhook** node named **Receive Job Application Form**
   - Method: POST
   - Path: `job-application`
   - Response: “Using Respond to Webhook node”

5. **Create the email entry node**
   - Add **Gmail Trigger** named **Check for Forwarded Applications**
   - Filter: unread
   - Polling: every minute (or adjust)
   - Configure Gmail credential.

6. **Add routing**
   - Add **Switch** node named **Route by Input Source**
   - Rule 1 (rename output: “From Form”): `{{$json.body?.jobUrl}}` is not empty
   - Rule 2 (rename output: “From Email”): `{{$json.textPlain}}` is not empty
   - Connect:
     - Webhook → Switch
     - Gmail Trigger → Switch

7. **Add URL scraping path**
   - Add **HTTP Request** node named **Scrape Job Posting URL**
   - URL: `https://r.jina.ai/{{ $json.body.jobUrl }}`
   - Method: POST
   - Response format: text
   - Connect Switch “From Form” → Scrape node

8. **Add AI extraction**
   - Add **OpenAI (LangChain)** node named **Extract Job Details with AI**
   - Model: `gpt-5-mini`
   - System message: instruct JSON-only output
   - User message: ask for the exact JSON structure (company, role, etc.) using content `{{$json.data || $json.textPlain}}`
   - Connect:
     - Scrape node → AI extraction
     - Switch “From Email” → AI extraction

9. **Add parsing/enrichment**
   - Add **Code** node named **Parse and Structure Job Data**
   - Paste logic equivalent to:
     - Parse `input.message.content` JSON
     - Set `DAYS_UNTIL_FOLLOWUP = 7`
     - Add `appliedDate`, `followUpDate`, `status: 'Applied'`
     - `jobUrl` from webhook body else “From email”
   - Connect AI extraction → Code

10. **(Fix required) Add “Research Company Online” OR edit interview prompt**
   - Option A (recommended to match existing expressions): add an **HTTP Request** node named **Research Company Online**
     - Use a search/research provider of your choice (SerpAPI/Tavily/etc.) to fetch a short company summary.
     - Output should contain a `data` field (text) because the interview node expects `...json.data`.
     - Connect **Parse and Structure Job Data → Research Company Online → Generate Interview Prep Materials**.
   - Option B: remove/replace the expression referencing `$('Research Company Online')` in the interview prep prompt.

11. **Add interview prep generation**
   - Add **OpenAI (LangChain)** node named **Generate Interview Prep Materials**
   - Model: `gpt-5-mini`
   - Prompt: include job fields and company research snippet, request 5 sections (questions, talking points, questions-to-ask, culture, salary notes).
   - Connect:
     - If you added research node: Research node → Interview node
     - Otherwise: Parse node → Interview node

12. **Save to Notion**
   - Add **Notion** node named **Save Application to Notion**
   - Resource: Database Page (Create)
   - Select your Database ID
   - Map fields (typical mapping):
     - Company (Title) ← `{{$json.company}}`
     - Role ← `{{$json.role}}`
     - Status ← `{{$json.status}}`
     - Applied Date ← `{{$json.appliedDate}}`
     - Follow Up Date ← `{{$json.followUpDate}}`
     - Salary Range ← `{{$json.salaryRange}}`
     - Job URL ← `{{$json.jobUrl}}`
     - Company Research ← output from research node (if used)
     - Interview Prep ← `{{$('Generate Interview Prep Materials').first().json.message.content}}` (adjust if schema differs)
   - Connect Interview node → Notion node

13. **Prepare confirmation summary**
   - Add **Set** node named **Prepare Success Response**
   - Add a string field `summary` that formats company/role/location/salary/follow-up date.
   - Connect Notion → Set

14. **Webhook response**
   - Add **Respond to Webhook** node named **Send Confirmation Response**
   - Respond with: text
   - Body: `{{$json.summary}}`
   - Connect Set → Respond to Webhook

15. **Email confirmation**
   - Add **Gmail** node named **Send Email Confirmation**
   - To: `{{ $('Check for Forwarded Applications').first().json.from?.value?.[0]?.address }}`
   - Subject/body as desired, include excerpt of interview prep.
   - Connect Set → Gmail send
   - **Recommended safeguard:** add an IF node so this runs only when triggered by Gmail (otherwise recipient is empty on webhook runs).

16. **Daily follow-up reminder path**
   - Add **Schedule Trigger** named **Daily Follow-up Check** set to 9 AM.
   - Add **Notion** node named **Get Applications Due for Follow-up**
     - Operation: getAll
     - Database ID: same as above
     - Add a filter for follow-ups due “today” (depends on your schema; typically Follow Up Date equals today and Status is “Applied/Interviewing”).
   - Add **IF** node named **Has Pending Follow-ups?**
     - Condition: `{{$input.all().length}} > 0`
   - Add **Slack** node named **Send Slack Reminder**
     - Compose list from Notion properties Company + Role.
   - Connect: Schedule → Notion getAll → IF (true) → Slack

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Notion database must include the listed properties (Company, Role, Status, Applied Date, Salary Range, Job URL, Company Research, Interview Prep, Follow Up Date, Notes). | From the main sticky note content in the workflow canvas. |
| Workflow has two input methods: manual form (job URL) and forwarded email (Gmail trigger). | Sticky note “📥 Input Methods”. |
| **Missing dependency:** “Research Company Online” is referenced in the interview prep prompt but not present as a node in the workflow JSON. | Must be added or the prompt expression must be updated. |
| Notion Database IDs are empty in both Notion nodes and must be configured before running. | Affects “Save Application to Notion” and “Get Applications Due for Follow-up”. |
| Email confirmation node uses sender address from Gmail trigger; it will fail on webhook runs unless guarded. | “Send Email Confirmation” node expression depends on Gmail trigger execution. |