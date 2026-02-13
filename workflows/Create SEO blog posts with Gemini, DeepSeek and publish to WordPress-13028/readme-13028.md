Create SEO blog posts with Gemini, DeepSeek and publish to WordPress

https://n8nworkflows.xyz/workflows/create-seo-blog-posts-with-gemini--deepseek-and-publish-to-wordpress-13028


# Create SEO blog posts with Gemini, DeepSeek and publish to WordPress

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Create SEO blog posts with Gemini, DeepSeek and publish to WordPress

**Purpose & use cases:**  
This workflow automates daily SEO blog production for multiple client projects: it loads a list of active clients, checks whether each client is scheduled to publish today, pulls an approved blog topic from that client’s Google Sheet, performs SEO/competitor research via Perplexity + DeepSeek, generates an HTML article via Gemini, formats/extracts title/body, then triggers a separate (Part 2) WordPress publishing sub-workflow.

### 1.1 Project Scheduling & Validation
Runs daily at 7 AM, loads the “active client projects” sheet, and filters clients based on “Weekly Frequency”.

### 1.2 Client Data Processing
Iterates through eligible clients, fetches their approved topic list sheet, validates a focus keyword exists, and selects the first pending topic.

### 1.3 AI-Powered Content Creation
Performs web research and competitor insights (Perplexity tool + DeepSeek agent) and writes an SEO-optimized HTML article (Gemini agent) tailored to Indian investors with FAQs and internal links.

### 1.4 Content Formatting
Removes newlines/em dashes, extracts the first `<h1>` as title, removes the `<h1>` from body.

### 1.5 WordPress Publishing (Sub-workflow trigger)
Sends prepared content and metadata to a separate workflow that posts to WordPress (with images/categories, not included here).

---

## 2. Block-by-Block Analysis

### Block 1 — Project Scheduling & Validation

**Overview:**  
Triggers daily and determines which client projects should publish a post today based on the “Weekly Frequency” field.

**Nodes involved:**
- Daily Blog Publishing Schedule
- Load Active Client Projects
- Verify Publishing Day & Frequency

#### Node: Daily Blog Publishing Schedule
- **Type / role:** Schedule Trigger; initiates workflow automatically.
- **Configuration:** Runs daily at **07:00** (instance time zone dependent).
- **Outputs:** Sends an empty trigger item to “Load Active Client Projects”.
- **Edge cases / failures:** If n8n instance timezone differs from expectation, “7 AM” may not align with business timezone.

#### Node: Load Active Client Projects
- **Type / role:** Google Sheets; loads the client project registry.
- **Configuration choices (interpreted):**
  - Uses **Google Sheets OAuth2** credential (`Rahul Sheet Testing`).
  - `documentId` is configured as a **URL placeholder** (`=Your_Sheet_URL`).
  - `sheetName` is a placeholder (`=Select_Your_Column`) and must be replaced with the actual sheet/tab name (or configured properly).
- **Key data expected per row:**  
  `Client ID`, `Website URL`, `Blog API`, `GMB Name`, `Weekly Frequency`, `On Page Sheet` (or “On Page Sheet URL” per sticky note).
- **Connections:**
  - Output → Verify Publishing Day & Frequency
- **Edge cases / failures:**
  - Wrong sheet URL/tab name → “sheet not found” / permission errors.
  - Missing columns → downstream expressions like `$json['Weekly Frequency']` will be null.

#### Node: Verify Publishing Day & Frequency
- **Type / role:** IF; filters clients eligible to publish today.
- **Logic (OR combinator):**
  1) `Weekly Frequency` **contains** `{{$now.toFormat('ccc')}}`  
     - Example: if today is Mon, `toFormat('ccc')` returns `Mon` (Luxon format).
  2) `Weekly Frequency` **equals** `"Daily"`
- **Connections:**
  - **True** output → Process Each Client Project  
  - **False** output: not connected (client rows not scheduled are dropped).
- **Edge cases / failures:**
  - If sheet uses different day abbreviations (e.g., “Monday” instead of “Mon”), condition won’t match.
  - If frequency field has unexpected casing/formatting (e.g., “mon”, “MON”), “contains” is case sensitive by default.

**Sticky note covering this block:**
- “## Project Scheduling & Validation … Runs daily at 7 AM … verify which clients should publish today …”

---

### Block 2 — Client Data Processing

**Overview:**  
Loops through each scheduled client, loads approved topic rows from that client’s content sheet, ensures a focus keyword exists, and chooses the first pending topic to write.

**Nodes involved:**
- Process Each Client Project
- Get Approved Blog Topics from Sheet
- Validate Focus Keyword Exists
- Select First Pending Topic

#### Node: Process Each Client Project
- **Type / role:** Split In Batches; iterates client projects.
- **Configuration:** Default options (batch size not specified in JSON; n8n defaults apply).
- **Connections (important):**
  - Receives input from Verify Publishing Day & Frequency (true path).
  - Its **second output (index 1)** goes to Get Approved Blog Topics from Sheet.  
    This indicates the workflow likely uses “Split in Batches” looping mechanics:
    - Output 1 often emits the “current batch”
    - Output 2 is used to continue/loop (implementation depends on node version)
- **Edge cases / failures:**
  - If batch size defaults to 1000 and there are many clients, execution time may be large.
  - Misunderstanding which output is “loop/continue” can break iteration logic if modified.

#### Node: Get Approved Blog Topics from Sheet
- **Type / role:** Google Sheets; loads topic candidates for the current client.
- **Configuration choices:**
  - Credential: Google Sheets OAuth2 (`Sheet Systems`).
  - `documentId` is currently a placeholder: `=your_sheet_url` (should likely be taken dynamically from the current client row, e.g., “On Page Sheet URL” or a content sheet URL column).
  - `sheetName` is hardcoded to `ID` (likely the tab name).
  - **Error handling:** `onError = continueRegularOutput` and `alwaysOutputData = true`  
    → even if the sheet call fails, the workflow continues (but downstream data may be empty/invalid).
- **Connections:**
  - Output → Validate Focus Keyword Exists
- **Edge cases / failures:**
  - Placeholder sheet URL means it will not actually load per-client topics unless replaced.
  - If the sheet has no rows (or API returns empty), downstream `items[0]` selection can fail logically (see next nodes).
  - “Continue on fail” can hide failures; you may publish empty/incorrect content unless you add guards.

#### Node: Validate Focus Keyword Exists
- **Type / role:** IF; ensures a usable topic row exists.
- **Condition (AND):**
  - `Focus Keyword` is **not empty**
- **Connections:**
  - **True** → Select First Pending Topic
  - **False** → Process Each Client Project (feeds back to batching node, effectively “skip and continue”)
- **Edge cases / failures:**
  - If sheet returns rows but column name differs (e.g., “FocusKeyword”), it will be treated as empty.
  - If sheet is empty but node still outputs due to `alwaysOutputData`, this IF might evaluate against unexpected structure.

#### Node: Select First Pending Topic
- **Type / role:** Code; selects exactly one topic row.
- **Logic:** `return [items[0]];`
- **Connections:**
  - Output → Research Keyword & Topics
- **Edge cases / failures:**
  - If `items` is empty, `items[0]` is `undefined` → likely causes runtime error or produces invalid output.  
    Mitigation: add a guard (`if (!items.length) return [];`) or route empty to loop/stop.

**Sticky note covering this block:**
- “## Client Data Processing … Loops through each client … fetches approved blog topics … validates keyword … selects first pending topic …”

---

### Block 3 — AI-Powered Content Creation

**Overview:**  
Uses a research agent (DeepSeek + Perplexity tool) to produce an actionable research report, then uses a writing agent (Gemini) to produce an HTML blog post incorporating the research and client constraints.

**Nodes involved:**
- Research Keyword & Topics
- DeepSeek - Research Model
- Perplexity - Web Search Tool
- Generate SEO-Optimized Article
- Gemini - Content Writing Model

#### Node: Research Keyword & Topics
- **Type / role:** LangChain Agent; orchestrates research using an LLM + web search tool.
- **Inputs:** From Select First Pending Topic.
- **Prompt input (`text`):**
  - `Keyword: {{$json['Focus Keyword']}}`
  - `Content Topic: {{$json['Content Topic']}}`
- **System message (key constraints):**
  - Must deliver: search intent, top competitor insights (5–10), gaps/opportunities, audience profile, trending subtopics, LSI + related keywords, outline, trusted sources.
  - Conditional instruction: if topic is Top/Best type, do the research “for this website” using `Website URL` from **Load Active Client Projects**.
- **Model / tools attached via connections:**
  - Uses **DeepSeek - Research Model** as `ai_languageModel`.
  - Uses **Perplexity - Web Search Tool** as `ai_tool`.
- **Outputs:** Agent output stored as `output` (used downstream as `{{$json.output}}`).
- **Edge cases / failures:**
  - Tool/LLM credentials missing → agent fails.
  - Perplexity rate limits/timeouts → partial research or failure.
  - The system message references `$('Load Active Client Projects').item.json[...]`; if multiple clients are being processed, ensure the expression resolves to the correct client item in the current run (cross-item referencing can become ambiguous when batching/looping is modified).

#### Node: DeepSeek - Research Model
- **Type / role:** LangChain Chat Model (DeepSeek).
- **Configuration:** Default options; credential `DeepSeek_Testing`.
- **Connections:** Provides model to Research Keyword & Topics.
- **Edge cases:** API key issues, model availability changes, token limits for large research outputs.

#### Node: Perplexity - Web Search Tool
- **Type / role:** Perplexity tool node used by the agent for live web search.
- **Configuration:** Uses Perplexity credential `Perplexity_Testing`. Message content is AI-generated variable via `$fromAI(...)`.
- **Connections:** Provided as `ai_tool` to Research Keyword & Topics.
- **Edge cases:** Tool node may require specific Perplexity plan/features; request size and quota limits.

#### Node: Generate SEO-Optimized Article
- **Type / role:** LangChain Agent; writes the final HTML article using the research output.
- **Inputs:**
  - From Research Keyword & Topics, using `{{$json.output}}` as “Research”.
  - Uses fields from “Select First Pending Topic”: Focus Keyword, Content Topic, Internal Linking URLs, Words.
- **System message (critical formatting constraints):**
  - 800–1000 words; Indian investor audience; short sentences; high readability.
  - Must include FAQ section with 5 Q/A; FAQ heading “Frequently Asked Questions”.
  - “Start content with the Question.”
  - If Top/Best list topic: put client business (GMB Name from active projects sheet) at top rank.
  - HTML tags restricted to: `<h1>, <h2>, <p>, <ul>, <ol>, <b>, <i>, <a>` **but** also says “Keep FAQs Question in H4 Heading” (inconsistent because `<h4>` is not in allowed list).
  - No `<head>`, `<footer>`, CSS, no extra formatting, avoid em dashes, avoid `\n` line breaks.
- **Model attached via connections:**
  - Uses **Gemini - Content Writing Model** as `ai_languageModel`.
- **Output:** HTML placed in `output` field (used downstream).
- **Edge cases / failures:**
  - Conflicting instructions about `<h4>` can cause the model to output disallowed tags. Downstream extractor only looks for `<h1>`, so invalid FAQ headings won’t break extraction but may affect WordPress formatting.
  - If “Words” is not set or not numeric, the agent may ignore length constraints.
  - Large prompt may hit token limits (especially including long research output).

#### Node: Gemini - Content Writing Model
- **Type / role:** LangChain Chat Model (Google Gemini via PaLM/Google AI credential).
- **Configuration:** Default options; credential `Gemini_Testing`.
- **Connections:** Provides model to Generate SEO-Optimized Article.
- **Edge cases:** API quota, safety filters, model deprecation, or region restrictions.

**Sticky note covering this block:**
- “## AI-Powered Content Creation … Uses Perplexity … DeepSeek … Gemini … 800-1000 word …”

---

### Block 4 — Content Formatting

**Overview:**  
Normalizes HTML string, extracts `<h1>` text as a title reference, removes the `<h1>` block from the body, and prepares the final payload fields.

**Nodes involved:**
- Extract Title & Body Content
- Prepare Blog Data for Publishing

#### Node: Extract Title & Body Content
- **Type / role:** Code; parses and transforms generated HTML.
- **Logic details:**
  - Reads `content = $input.first().json.output`
  - Removes all `\n`
  - Replaces em dash `—` with space
  - Extracts first `<h1>...</h1>` inner text into `h1Only`
  - Removes all `<h1>...</h1>` blocks from content into `excludedH1`
  - Returns `{ h1Only, excludedH1 }`
- **Connections:** Output → Prepare Blog Data for Publishing
- **Edge cases / failures:**
  - If LLM output lacks `<h1>`, `h1Only` becomes empty and body remains unchanged except newline removal.
  - Regex-based HTML parsing can fail with malformed tags or nested `<h1>`; still usually acceptable for controlled outputs.

#### Node: Prepare Blog Data for Publishing
- **Type / role:** Set; maps fields for the publishing sub-workflow.
- **Assignments (outputs):**
  - `S NO` = `Select First Pending Topic.row_number`
  - `Blog Title` = `Select First Pending Topic['Content Topic']` (note: not using extracted `h1Only`)
  - `Content` = `excludedH1` (body without `<h1>`)
  - `Auth Code` = Active projects sheet `Blog API`
  - `Website URL` = Active projects sheet `Website URL`
  - `OnPage SEO` = From Process Each Client Project `On Page Sheet`
- **Connections:** Output → Trigger WordPress Publishing Workflow
- **Edge cases / failures:**
  - Referencing `row_number` assumes the Google Sheets node provides it; if disabled or different, it may be undefined.
  - Cross-node item referencing (`$node[...]`) can become inconsistent if multiple items are in play; this workflow tries to reduce to one topic item, but client project data still comes from the batching context.

**Sticky note covering this block:**
- “## Content Formatting … Extracts the H1 title … removes it … prepares all blog data …”

---

### Block 5 — WordPress Publishing (Sub-workflow)

**Overview:**  
Delegates posting to WordPress to a separate workflow (“Part 2”), passing content and identifiers. It does not wait for the sub-workflow to finish.

**Nodes involved:**
- Trigger WordPress Publishing Workflow

#### Node: Trigger WordPress Publishing Workflow
- **Type / role:** Execute Workflow; invokes another n8n workflow.
- **Configuration:**
  - `waitForSubWorkflow: false` (fire-and-forget)
  - Workflow target: **“Plan>Design>Test — Blog Publishing Automation”** (ID `NZ064TnXQVyJikPh`)
  - Inputs mapped:
    - `Content` = prepared body HTML
    - `Client ID` = current client project `Client ID`
    - `Blog S.NO.` = `Get Approved Blog Topics from Sheet.row_number` (string)
    - `Blog Title` = prepared title
    - `OnPage SEO` = prepared `OnPage SEO`
    - `Focus Keyword` = from Select First Pending Topic
- **Connections:**
  - Output → Process Each Client Project (feeds back to continue loop)
- **Edge cases / failures:**
  - If the sub-workflow expects additional fields (category, tags, featured image prompt, etc.), publishing may fail.
  - Because it does not wait, failures in the sub-workflow won’t stop or directly report back here unless you add external logging.
  - If the workflow ID changes (export/import to new workspace), you must re-select the workflow.

**Sticky note covering this block:**
- “## WordPress Publishing … Sends prepared blog content to a sub-workflow … featured images and category assignment …”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Workflow Instructions | Sticky Note | Documentation / operator guidance |  |  | # Automated Blog Content Creation & Publishing… (contains setup steps and “2-Part Automation System” note) |
| Daily Blog Publishing Schedule | Schedule Trigger | Daily execution at 7 AM |  | Load Active Client Projects | ## Project Scheduling & Validation… |
| Load Active Client Projects | Google Sheets | Load client registry rows | Daily Blog Publishing Schedule | Verify Publishing Day & Frequency | ## Project Scheduling & Validation… |
| Verify Publishing Day & Frequency | IF | Filter clients scheduled for today | Load Active Client Projects | Process Each Client Project | ## Project Scheduling & Validation… |
| Process Each Client Project | Split In Batches | Iterate eligible clients | Verify Publishing Day & Frequency; Trigger WordPress Publishing Workflow (loop-back) | Get Approved Blog Topics from Sheet | ## Client Data Processing… |
| Get Approved Blog Topics from Sheet | Google Sheets | Load approved topics for client | Process Each Client Project | Validate Focus Keyword Exists | ## Client Data Processing… |
| Validate Focus Keyword Exists | IF | Ensure topic row has Focus Keyword | Get Approved Blog Topics from Sheet | Select First Pending Topic (true); Process Each Client Project (false) | ## Client Data Processing… |
| Select First Pending Topic | Code | Keep only first topic item | Validate Focus Keyword Exists | Research Keyword & Topics | ## Client Data Processing… |
| Research Keyword & Topics | LangChain Agent | SEO/competitor research orchestration | Select First Pending Topic; (uses DeepSeek + Perplexity as attachments) | Generate SEO-Optimized Article | ## AI-Powered Content Creation… |
| DeepSeek - Research Model | LangChain Chat Model (DeepSeek) | LLM for research agent |  | Research Keyword & Topics (as ai_languageModel) | ## AI-Powered Content Creation… |
| Perplexity - Web Search Tool | Perplexity Tool | Web search tool for agent |  | Research Keyword & Topics (as ai_tool) | ## AI-Powered Content Creation… |
| Generate SEO-Optimized Article | LangChain Agent | Produce final HTML blog post | Research Keyword & Topics; (uses Gemini as attachment) | Extract Title & Body Content | ## AI-Powered Content Creation… |
| Gemini - Content Writing Model | LangChain Chat Model (Google Gemini) | LLM for writing agent |  | Generate SEO-Optimized Article (as ai_languageModel) | ## AI-Powered Content Creation… |
| Extract Title & Body Content | Code | Extract `<h1>` and body HTML | Generate SEO-Optimized Article | Prepare Blog Data for Publishing | ## Content Formatting… |
| Prepare Blog Data for Publishing | Set | Map payload fields for publishing | Extract Title & Body Content | Trigger WordPress Publishing Workflow | ## Content Formatting… |
| Trigger WordPress Publishing Workflow | Execute Workflow | Invoke WP publishing sub-workflow | Prepare Blog Data for Publishing | Process Each Client Project (loop-back) | ## WordPress Publishing… |
| Sticky Note | Sticky Note | Section label |  |  | ## Project Scheduling & Validation… |
| Sticky Note1 | Sticky Note | Section label |  |  | ## Client Data Processing… |
| Sticky Note2 | Sticky Note | Section label |  |  | ## AI-Powered Content Creation… |
| Sticky Note3 | Sticky Note | Section label |  |  | ## Content Formatting… |
| Sticky Note4 | Sticky Note | Section label |  |  | ## WordPress Publishing… |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** named: *Create SEO blog posts with Gemini, DeepSeek and publish to WordPress*.

2) **Add Schedule Trigger**
   - Node: **Schedule Trigger**
   - Name: `Daily Blog Publishing Schedule`
   - Set it to run **every day at 07:00**.

3) **Add Google Sheets node (client registry)**
   - Node: **Google Sheets**
   - Name: `Load Active Client Projects`
   - Credentials: **Google Sheets OAuth2**
   - Configure:
     - Document: select your **client registry Google Sheet URL**
     - Sheet/Tab name: the tab that contains client rows
   - Ensure columns exist (at minimum): `Client ID`, `Website URL`, `Blog API`, `GMB Name`, `Weekly Frequency`, `On Page Sheet` (and/or topic sheet URL column).

4) **Add IF node (publish today?)**
   - Node: **IF**
   - Name: `Verify Publishing Day & Frequency`
   - Add conditions (OR):
     - String **contains**: left `{{$json["Weekly Frequency"]}}`, right `{{$now.toFormat("ccc")}}`
     - String **equals**: left `{{$json["Weekly Frequency"]}}`, right `Daily`
   - Connect: Schedule Trigger → Load Active Client Projects → IF (true output continues).

5) **Add Split In Batches (iterate clients)**
   - Node: **Split In Batches**
   - Name: `Process Each Client Project`
   - Configure batch size as desired (commonly 1 for strict sequential publishing).
   - Connect IF (true) → Split In Batches.

6) **Add Google Sheets node (topics per client)**
   - Node: **Google Sheets**
   - Name: `Get Approved Blog Topics from Sheet`
   - Credentials: Google Sheets OAuth2
   - Configure:
     - Document: **use an expression** pointing to the current client’s topic sheet URL (recommended), e.g. `{{$json["Content Sheet URL"]}}` if you store it in the client registry.
     - Tab name: set to your topics tab (example in JSON: `ID`)
   - Optional: keep “Continue On Fail”, but add downstream checks to avoid empty publishing.
   - Connect Split In Batches → Google Sheets topics.

7) **Add IF node (focus keyword present)**
   - Node: **IF**
   - Name: `Validate Focus Keyword Exists`
   - Condition: `{{$json["Focus Keyword"]}}` **is not empty**
   - Connect:
     - Topics → IF
     - IF false → back to `Process Each Client Project` (to skip and continue)

8) **Add Code node (pick first topic)**
   - Node: **Code**
   - Name: `Select First Pending Topic`
   - Code:
     - Return only the first item (`return [items[0]];`)
     - Recommended hardening: if empty, return `[]`.
   - Connect IF true → Code.

9) **Add credentials for AI services**
   - Create credentials in n8n:
     - **DeepSeek API** credential
     - **Perplexity API** credential
     - **Google Gemini / PaLM** credential

10) **Add DeepSeek model node**
   - Node: **DeepSeek Chat Model** (LangChain)
   - Name: `DeepSeek - Research Model`
   - Select DeepSeek credential.

11) **Add Perplexity tool node**
   - Node: **Perplexity Tool**
   - Name: `Perplexity - Web Search Tool`
   - Select Perplexity credential.

12) **Add LangChain Agent for research**
   - Node: **AI Agent** (LangChain Agent)
   - Name: `Research Keyword & Topics`
   - User message input: include keyword + topic from the selected topic item.
   - System message: include the research checklist (intent, competitors, gaps, audience, LSI, outline, sources).
   - Connect:
     - `Select First Pending Topic` → `Research Keyword & Topics` (main)
     - `DeepSeek - Research Model` → `Research Keyword & Topics` (ai_languageModel)
     - `Perplexity - Web Search Tool` → `Research Keyword & Topics` (ai_tool)

13) **Add Gemini model node**
   - Node: **Google Gemini Chat Model** (LangChain)
   - Name: `Gemini - Content Writing Model`
   - Select Gemini credential.

14) **Add LangChain Agent for writing**
   - Node: **AI Agent** (LangChain Agent)
   - Name: `Generate SEO-Optimized Article`
   - Input should include: Focus Keyword, Content Topic, Internal Linking URLs, Words, and the research output.
   - System message: enforce HTML-only output and readability requirements.
   - Connect:
     - Research agent → Writing agent (main)
     - Gemini model → Writing agent (ai_languageModel)

15) **Add Code node (extract and clean HTML)**
   - Node: **Code**
   - Name: `Extract Title & Body Content`
   - Implement:
     - Remove newlines and em dashes
     - Extract `<h1>` text
     - Remove `<h1>...</h1>` from body

16) **Add Set node (prepare publishing payload)**
   - Node: **Set**
   - Name: `Prepare Blog Data for Publishing`
   - Map fields: Blog Title, Content (without h1), Client ID, Focus Keyword, OnPage SEO reference, etc.

17) **Add Execute Workflow node (publishing sub-workflow)**
   - Node: **Execute Workflow**
   - Name: `Trigger WordPress Publishing Workflow`
   - Select your separate WordPress publishing workflow (Part 2).
   - Set `waitForSubWorkflow` to **false** (or true if you want status/handling).
   - Define inputs:
     - `Client ID`, `Blog S.NO.`, `Blog Title`, `Content`, `OnPage SEO`, `Focus Keyword`
   - Connect Set → Execute Workflow → back to Split In Batches to continue loop.

18) **Create/Configure the Part 2 publishing workflow (required)**
   - Must accept the above inputs.
   - Should post to WordPress, handle featured image/category, and ideally update the topic row status in Google Sheets to avoid reposting.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “This workflow (Part 1) handles content research and writing. To complete the automation, you need Part 2 which handles WordPress publishing with images.” | From “Workflow Instructions” sticky note |
| Credentials required: Google Sheets OAuth, Google Gemini API, DeepSeek API, Perplexity API | From “Workflow Instructions” sticky note |
| Client project sheet must include: Client ID, Website URL, Blog API, GMB Name, Weekly Frequency, On Page Sheet URL | From “Workflow Instructions” sticky note |
| Topic/content sheets per client must include: Focus Keyword, Content Topic, Internal Linking URLs, Words, Topic Approval, Content Approval | From “Workflow Instructions” sticky note |

