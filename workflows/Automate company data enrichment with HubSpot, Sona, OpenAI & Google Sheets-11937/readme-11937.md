Automate company data enrichment with HubSpot, Sona, OpenAI & Google Sheets

https://n8nworkflows.xyz/workflows/automate-company-data-enrichment-with-hubspot--sona--openai---google-sheets-11937


# Automate company data enrichment with HubSpot, Sona, OpenAI & Google Sheets

## 1. Workflow Overview

**Purpose:** This workflow enriches a list of companies (provided as website domains in Google Sheets) by:
1) scraping each company’s website,
2) extracting AI-derived company intelligence (positioning, features, personas, pricing, pros/cons, etc.) via OpenAI,
3) enriching firmographic/technographic data via Sona,
4) creating/updating HubSpot Company records with both Sona enrichment + AI custom attributes.

**Target use cases**
- Building enriched HubSpot company profiles for sales/marketing ops
- Automating account research and CRM data hygiene
- Creating structured “company intel” fields from website copy at scale

### Logical Blocks
**1.1 Setup & Input (Google Sheets)**
- Manual start → read list of company domains from a sheet.

**1.2 Web Scraping & Content Cleaning**
- Fetch HTML for each domain → extract likely “main content” → remove noise and normalize text.

**1.3 AI Extraction (OpenAI + structured JSON)**
- Send cleaned content to an AI agent using an OpenAI chat model.
- Force structured JSON output via a structured output parser.
- Aggregate AI results across all companies.

**1.4 HubSpot Preparation**
- Create HubSpot custom properties (schema) for all Sona + AI fields.
- Convert aggregated AI results back into item-per-company to prepare looping.

**1.5 Per-Company Enrichment & HubSpot Sync**
- Loop through companies in batches (one-by-one).
- Call Sona enrichment API for firmographics/tech.
- Create a HubSpot Company record.
- Format and PATCH custom properties onto the created company.
- Repeat until completion.

---

## 2. Block-by-Block Analysis

### 2.1 Setup & Input (Google Sheets)

**Overview:** Starts the workflow manually and loads the list of companies (website domains) from Google Sheets.

**Nodes involved**
- Start
- Get Company List from Sheet

#### Node: **Start**
- **Type / role:** Manual Trigger; entry point for testing/runs on demand.
- **Config:** No parameters.
- **Outputs:** Sends a single empty item to the next node.
- **Failure modes:** None.

#### Node: **Get Company List from Sheet**
- **Type / role:** Google Sheets node; reads rows from a spreadsheet.
- **Config choices:**
- Uses OAuth2 credentials (`googleSheetsOAuth2Api`).
- Targets a specific documentId and a sheet tab (gid=0; cached as “Sheet1”).
- **Data expectations:** Sheet must contain a column named **`Website Domain`** (explicitly referenced later).
- **Connections:**  
  Input: Start → Output: Scrape Website Content
- **Failure modes / edge cases:**
  - OAuth token expired / insufficient permissions.
  - Wrong sheet/tab selected.
  - Missing `Website Domain` column leads to downstream expression errors or bad URLs.

**Sticky note covering this block**
- “📥 Step 1: Data Input & Web Scraping … Reads company domains from Google Sheets …”

---

### 2.2 Web Scraping & Content Cleaning

**Overview:** Fetches each website’s HTML, extracts “main” text-like content, then cleans it to remove navigation/footer/noise and pairs it with the corresponding domain.

**Nodes involved**
- Scrape Website Content
- Extract HTML Content
- Clean & Format Text

#### Node: **Scrape Website Content**
- **Type / role:** HTTP Request; downloads the website HTML.
- **Config choices:**
  - **URL expression:**  
    `={{ ($json['Website Domain'].match(/^https?:\/\//) ? $json['Website Domain'] : 'https://' + $json['Website Domain']).toLowerCase() }}`
    - Ensures a scheme exists; defaults to `https://`.
    - Lowercases the URL.
  - **Batching:** batchSize=1, batchInterval=2000ms (throttles requests to be polite/avoid rate limits).
- **Connections:**  
  Input: Get Company List from Sheet → Output: Extract HTML Content
- **Failure modes / edge cases:**
  - 4xx/5xx from websites, WAF blocks, bot protection.
  - Non-HTML responses (PDF, SPA shells, redirects).
  - TLS handshake issues; invalid domains.
  - Lowercasing can be problematic for some paths (rare) but usually fine for domains.

#### Node: **Extract HTML Content**
- **Type / role:** HTML node; extracts text from HTML using CSS selectors.
- **Config choices:**
  - Operation: “extractHtmlContent”
  - Extract key `main_content` using selector: `main, body, article, .main-content, #content, p`
  - Skip selectors: `script, img, style, nav, footer, header`
- **Connections:**  
  Input: Scrape Website Content → Output: Clean & Format Text
- **Failure modes / edge cases:**
  - If the HTTP node returns empty/invalid HTML, extraction yields empty `main_content`.
  - Selector breadth (`body` + `p`) can capture lots of irrelevant text; cleaning tries to mitigate.

#### Node: **Clean & Format Text**
- **Type / role:** Code node; removes noise lines and maps content back to the corresponding domain.
- **Config choices / key logic:**
  - Reads **all incoming items**: `const items = $input.all();`
  - Also reads the full sheet data: `const sheetData = $('Get Company List from Sheet').all();`
  - **Matches by index**: pairs `items[index]` with `sheetData[index]`.
  - Cleans text line-by-line (removes common CTA strings, legal/footer bits, too-short lines, social-only lines, etc.).
  - Outputs per company:
    - `main_content` (cleaned)
    - `website_domain` (from sheet)
- **Connections:**  
  Input: Extract HTML Content → Output: Analyze with AI
- **Failure modes / edge cases:**
  - **Index misalignment risk:** If Scrape Website Content fails for one row and returns fewer items (or errors change item ordering), `sheetData[index]` may no longer correspond to the same company. This is the most important structural fragility in the workflow.
  - `main_content` can be undefined; `.split('\n')` would throw if not a string (though extraction usually returns a string).
  - Large pages → very large prompt → token limits or high cost.

**Sticky note covering this block**
- Same “📥 Step 1: Data Input & Web Scraping …”

---

### 2.3 AI Extraction (OpenAI + Structured Output)

**Overview:** Uses an AI agent with an OpenAI chat model to extract a fixed JSON structure from the scraped text and aggregates outputs into a single array.

**Nodes involved**
- OpenAI Chat Model
- Structured Output Parser
- Analyze with AI
- Collect AI Results

#### Node: **OpenAI Chat Model**
- **Type / role:** LangChain Chat Model node; provides the underlying LLM.
- **Config choices:**
  - Model: `gpt-5.1`
  - Credentials: OpenAI API key (`openAiApi`).
- **Connections:**  
  Output (AI languageModel) → Analyze with AI (as its model)
- **Failure modes / edge cases:**
  - Invalid API key, quota exceeded, model not available in the workspace/region.
  - Timeouts on long prompts.

#### Node: **Structured Output Parser**
- **Type / role:** LangChain structured output parser; enforces output schema.
- **Config choices:**
  - Schema example defines fields such as `website_name`, `website_domain`, arrays for positioning, prices, pros/cons, etc.
- **Connections:**  
  Output (ai_outputParser) → Analyze with AI
- **Failure modes / edge cases:**
  - If the model returns invalid JSON or violates schema, parsing can fail (node behavior depends on agent + parser; may error the execution).

#### Node: **Analyze with AI**
- **Type / role:** LangChain Agent node; prompts LLM to produce structured JSON.
- **Config choices:**
  - Prompt includes:
    - Instruction to respond **ONLY** in valid JSON.
    - Injects `{{ $json.main_content }}` and uses `{{ $json.website_domain }}` in the required JSON.
    - Guidance for missing data: “Not mentioned” or `[]`.
  - `hasOutputParser: true` (uses Structured Output Parser).
- **Connections:**  
  Inputs: Clean & Format Text (main), OpenAI Chat Model (ai_languageModel), Structured Output Parser (ai_outputParser)  
  Output: Collect AI Results
- **Failure modes / edge cases:**
  - Prompt includes a fenced JSON block; even though it says “no markdown”, the block is shown in the prompt. The parser usually handles this, but some models may echo backticks or extra text (parser then fails).
  - Very long `main_content` may exceed context window or increase latency/cost.
  - Site content in non-English could reduce extraction quality.

#### Node: **Collect AI Results**
- **Type / role:** Aggregate node; collects all AI outputs into one array.
- **Config choices:**
  - Aggregates field: `output`
- **Connections:**  
  Input: Analyze with AI → Output: Create Custom HubSpot Fields
- **Failure modes / edge cases:**
  - If Analyze with AI output shape differs (e.g., `output` missing), aggregation may produce empty array and later loop does nothing.

**Sticky note covering this block**
- “🤖 Step 2: AI Analysis … Aggregates all AI results”

---

### 2.4 HubSpot Preparation (Schema + Loop Preparation)

**Overview:** Ensures HubSpot has all required custom company properties, then reshapes aggregated AI results into individual items so the workflow can loop per company.

**Nodes involved**
- Create Custom HubSpot Fields
- Prepare Data for Loop
- Split Companies and AI Output into Items
- Loop Through Companies

#### Node: **Create Custom HubSpot Fields**
- **Type / role:** HTTP Request; creates many HubSpot Company properties (schema).
- **Config choices:**
  - Method: POST
  - Auth: predefined credential type `hubspotAppToken`
  - Body: JSON with `"inputs": [...]` defining dozens of fields:
    - Sona fields (tech, revenue bounds, socials, tags, etc.)
    - AI fields (key positioning elements, personas, pricing model, etc.)
  - **Important:** The configured URL is `https://api.hubapi.com/YOUR_AWS_SECRET_KEY_HERE` which is a placeholder and not a valid HubSpot endpoint as-is. In practice, it should be something like:
    - `POST https://api.hubapi.com/crm/v3/properties/companies/batch/create`
- **Error handling:** `onError: continueRegularOutput` (execution continues even if this fails).
- **Connections:**  
  Input: Collect AI Results → Output: Prepare Data for Loop
- **Failure modes / edge cases:**
  - Wrong endpoint (very likely, given placeholder).
  - Missing scopes: requires at least `crm.schemas.companies.write`.
  - Properties already exist → HubSpot returns conflict errors; because errors are continued, downstream may still proceed.
  - HubSpot API limits.

#### Node: **Prepare Data for Loop**
- **Type / role:** Set node; stores aggregated AI outputs under a single field for splitting.
- **Config choices:**
  - Sets `output` to: `={{ $('Collect AI Results').first().json.output }}`
- **Connections:**  
  Input: Create Custom HubSpot Fields → Output: Split Companies and AI Output into Items
- **Failure modes / edge cases:**
  - If Collect AI Results is empty, `.first()` may be null (expression error) or `output` undefined.

#### Node: **Split Companies and AI Output into Items**
- **Type / role:** Split Out node; converts `output` array into one item per element.
- **Config choices:** `fieldToSplitOut: output`
- **Connections:**  
  Input: Prepare Data for Loop → Output: Loop Through Companies
- **Failure modes / edge cases:**
  - If `output` is not an array, split-out yields 0 items or errors.

#### Node: **Loop Through Companies**
- **Type / role:** Split In Batches; controls iteration (batch loop).
- **Config choices:** Options default (batch size default in n8n unless set; typically 1 unless changed in UI).
- **Connections:**
  - Main output 0 → End (when done / batch empty)
  - Main output 1 → Sona Enrich (each item)
  - Input from Update Company with AI Data to continue looping.
- **Failure modes / edge cases:**
  - If batch size > 1, downstream nodes referencing `.first()` can mis-associate items.
  - Loop depends on the “continue” connection from Update Company with AI Data; if that node errors hard, loop stops.

**Sticky note covering this block**
- “⚙️ Step 3: HubSpot Preparation … Creates custom fields … Splits aggregated data …”

---

### 2.5 Per-Company Enrichment & HubSpot Sync

**Overview:** For each company item (AI output + domain), calls Sona to enrich company data, creates the company in HubSpot, formats all custom properties, then patches them into HubSpot.

**Nodes involved**
- Sona Enrich
- Create HubSpot Company
- Format Custom Properties
- Update Company with AI Data
- End

#### Node: **Sona Enrich**
- **Type / role:** HTTP Request; calls Sona Labs enrichment API.
- **Config choices:**
  - URL: `https://api2.sonalabs.com/resource/company/enrich`
  - Query param `website` computed as:  
    `={{ $json.website_domain.toLowerCase() + (/\.[a-z]{2,}$/i.test($json.website_domain.toLowerCase()) ? '' : '.com') }}`
    - If domain doesn’t end with a TLD-like suffix, appends `.com`.
  - Headers:
    - `x-api-key` (value not set in node; must be provided)
    - content-type/accept JSON
  - `onError: continueRegularOutput`
- **Connections:**  
  Input: Loop Through Companies (per item) → Output: Create HubSpot Company
- **Failure modes / edge cases:**
  - Missing/invalid Sona API key.
  - Sona returns non-200 for unknown domains; due to continue-on-error, downstream may receive incomplete data.
  - Domain normalization may be wrong for subdomains (e.g., `app.company.com`).

#### Node: **Create HubSpot Company**
- **Type / role:** HubSpot node; creates a company object.
- **Config choices:**
  - Resource: company; Authentication: appToken.
  - Name expression:  
    `={{ $json.data.name || $('Loop Through Companies').first().json.website_name }}`
    - Prefers Sona’s `data.name`, falls back to AI `website_name`.
  - Additional fields map many properties from Sona (`$json.data.*`) with defaults like `'none'`, `0`, etc.
  - Company domain uses: `$('Loop Through Companies').item.json.website_domain`
- **Connections:**  
  Input: Sona Enrich → Output: Format Custom Properties
- **Failure modes / edge cases:**
  - If Sona failed and `$json.data` is undefined, many expressions may fail unless n8n treats missing paths as undefined (some may still error depending on expression evaluation).
  - Creates duplicates if the company already exists (no upsert/search step).
  - Requires `crm.objects.companies.write` scope.

#### Node: **Format Custom Properties**
- **Type / role:** Code node; merges Sona enrichment + AI fields and formats them as HubSpot patch payload.
- **Config choices / key logic:**
  - Reads:
    - `sonaData = $('Sona Enrich').item.json.data`
    - `loopData = $('Loop Through Companies').item.json`
  - Formats `socialHandles` into newline-delimited “Platform: handle”.
  - Joins arrays into strings (comma-separated or newline-separated) for textarea fields.
  - Returns `{ properties: { ... } }` structure expected by HubSpot CRM PATCH.
- **Connections:**  
  Input: Create HubSpot Company → Output: Update Company with AI Data
- **Failure modes / edge cases:**
  - If `sonaData` is undefined (Sona error), referencing `sonaData.socialHandles` etc. will throw and stop execution (no try/catch around the whole block, only around JSON parsing).
  - HubSpot property names must exactly match those created; mismatches are silently ignored or rejected depending on API behavior.

#### Node: **Update Company with AI Data**
- **Type / role:** HTTP Request; patches custom properties onto the created HubSpot company.
- **Config choices:**
  - Method: PATCH
  - URL:  
    `=https://api.hubapi.com/crm/v3/objects/companies/{{ $('Create HubSpot Company').first().json.companyId }}`
  - Body: `={{ $json }}` (expects input already shaped as `{ properties: {...} }`)
  - Auth: predefined `hubspotAppToken`
  - Adds `Content-Type: application/json`
- **Connections:**  
  Input: Format Custom Properties → Output: Loop Through Companies (to continue next batch)
- **Failure modes / edge cases:**
  - If Create HubSpot Company fails or returns no `companyId`, URL becomes invalid.
  - 429 rate limiting; transient 5xx.
  - If body contains properties not defined in HubSpot schema, patch may fail or partially apply.

#### Node: **End**
- **Type / role:** NoOp; marks completion path from the batch loop.
- **Connections:**  
  Input: Loop Through Companies (done branch)
- **Failure modes:** None.

**Sticky note covering this block**
- “🔄 Step 4: Enrich & Sync to HubSpot … Get your Sona API key: https://platform.sonalabs.com/onboardingv2”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Start | Manual Trigger | Manual entry point | — | Get Company List from Sheet | (none) |
| Get Company List from Sheet | Google Sheets | Read company domains from spreadsheet | Start | Scrape Website Content | ## 📥 Step 1: Data Input & Web Scraping<br>- Reads company domains from Google Sheets<br>- Scrapes each website's content<br>- Extracts and cleans main content<br>- Removes navigation, footers, and noise |
| Scrape Website Content | HTTP Request | Fetch website HTML (throttled) | Get Company List from Sheet | Extract HTML Content | ## 📥 Step 1: Data Input & Web Scraping<br>- Reads company domains from Google Sheets<br>- Scrapes each website's content<br>- Extracts and cleans main content<br>- Removes navigation, footers, and noise |
| Extract HTML Content | HTML | Extract likely main text from HTML | Scrape Website Content | Clean & Format Text | ## 📥 Step 1: Data Input & Web Scraping<br>- Reads company domains from Google Sheets<br>- Scrapes each website's content<br>- Extracts and cleans main content<br>- Removes navigation, footers, and noise |
| Clean & Format Text | Code | Remove noise, normalize content, attach domain | Extract HTML Content | Analyze with AI | ## 📥 Step 1: Data Input & Web Scraping<br>- Reads company domains from Google Sheets<br>- Scrapes each website's content<br>- Extracts and cleans main content<br>- Removes navigation, footers, and noise |
| OpenAI Chat Model | LangChain Chat Model (OpenAI) | LLM provider for the agent | — | Analyze with AI (ai_languageModel) | ## 🤖 Step 2: AI Analysis<br>- Sends cleaned content to OpenAI<br>- Extracts structured company intelligence<br>- Identifies positioning, features, personas<br>- Captures pricing, pros/cons, value props<br>- Aggregates all AI results |
| Structured Output Parser | LangChain Structured Output Parser | Enforce JSON schema | — | Analyze with AI (ai_outputParser) | ## 🤖 Step 2: AI Analysis<br>- Sends cleaned content to OpenAI<br>- Extracts structured company intelligence<br>- Identifies positioning, features, personas<br>- Captures pricing, pros/cons, value props<br>- Aggregates all AI results |
| Analyze with AI | LangChain Agent | Prompt LLM to extract fixed JSON structure | Clean & Format Text | Collect AI Results | ## 🤖 Step 2: AI Analysis<br>- Sends cleaned content to OpenAI<br>- Extracts structured company intelligence<br>- Identifies positioning, features, personas<br>- Captures pricing, pros/cons, value props<br>- Aggregates all AI results |
| Collect AI Results | Aggregate | Combine all AI outputs into one array | Analyze with AI | Create Custom HubSpot Fields | ## 🤖 Step 2: AI Analysis<br>- Sends cleaned content to OpenAI<br>- Extracts structured company intelligence<br>- Identifies positioning, features, personas<br>- Captures pricing, pros/cons, value props<br>- Aggregates all AI results |
| Create Custom HubSpot Fields | HTTP Request | Create HubSpot company custom properties (schema) | Collect AI Results | Prepare Data for Loop | ## ⚙️ Step 3: HubSpot Preparation<br>- Creates custom fields in HubSpot CRM<br>- Prepares AI-extracted data for import<br>- Splits aggregated data into individual companies<br>- Ready for batch processing |
| Prepare Data for Loop | Set | Put aggregated AI output into `output` field | Create Custom HubSpot Fields | Split Companies and AI Output into Items | ## ⚙️ Step 3: HubSpot Preparation<br>- Creates custom fields in HubSpot CRM<br>- Prepares AI-extracted data for import<br>- Splits aggregated data into individual companies<br>- Ready for batch processing |
| Split Companies and AI Output into Items | Split Out | Convert aggregated array to item-per-company | Prepare Data for Loop | Loop Through Companies | ## ⚙️ Step 3: HubSpot Preparation<br>- Creates custom fields in HubSpot CRM<br>- Prepares AI-extracted data for import<br>- Splits aggregated data into individual companies<br>- Ready for batch processing |
| Loop Through Companies | Split In Batches | Iterate per company and control loop | Split Companies and AI Output into Items; Update Company with AI Data | Sona Enrich; End | ## 🔄 Step 4: Enrich & Sync to HubSpot<br>- Loops through each company one by one<br>- Enriches with Sona API (firmographics, revenue, employees)<br>- Creates company record in HubSpot<br>- Formats and populates all custom fields<br>- Combines AI insights + Sona data in one profile<br><br>**💡 Get your Sona API key:** https://platform.sonalabs.com/onboardingv2 |
| Sona Enrich | HTTP Request | Enrich company via Sona API | Loop Through Companies | Create HubSpot Company | ## 🔄 Step 4: Enrich & Sync to HubSpot<br>- Loops through each company one by one<br>- Enriches with Sona API (firmographics, revenue, employees)<br>- Creates company record in HubSpot<br>- Formats and populates all custom fields<br>- Combines AI insights + Sona data in one profile<br><br>**💡 Get your Sona API key:** https://platform.sonalabs.com/onboardingv2 |
| Create HubSpot Company | HubSpot | Create company record | Sona Enrich | Format Custom Properties | ## 🔄 Step 4: Enrich & Sync to HubSpot<br>- Loops through each company one by one<br>- Enriches with Sona API (firmographics, revenue, employees)<br>- Creates company record in HubSpot<br>- Formats and populates all custom fields<br>- Combines AI insights + Sona data in one profile<br><br>**💡 Get your Sona API key:** https://platform.sonalabs.com/onboardingv2 |
| Format Custom Properties | Code | Merge/format Sona + AI fields into HubSpot properties payload | Create HubSpot Company | Update Company with AI Data | ## 🔄 Step 4: Enrich & Sync to HubSpot<br>- Loops through each company one by one<br>- Enriches with Sona API (firmographics, revenue, employees)<br>- Creates company record in HubSpot<br>- Formats and populates all custom fields<br>- Combines AI insights + Sona data in one profile<br><br>**💡 Get your Sona API key:** https://platform.sonalabs.com/onboardingv2 |
| Update Company with AI Data | HTTP Request | PATCH custom properties into HubSpot company | Format Custom Properties | Loop Through Companies | ## 🔄 Step 4: Enrich & Sync to HubSpot<br>- Loops through each company one by one<br>- Enriches with Sona API (firmographics, revenue, employees)<br>- Creates company record in HubSpot<br>- Formats and populates all custom fields<br>- Combines AI insights + Sona data in one profile<br><br>**💡 Get your Sona API key:** https://platform.sonalabs.com/onboardingv2 |
| End | NoOp | End marker when loop completes | Loop Through Companies | — | (none) |
| Sticky Note | Sticky Note | Comment block | — | — | ## 📥 Step 1: Data Input & Web Scraping<br>- Reads company domains from Google Sheets<br>- Scrapes each website's content<br>- Extracts and cleans main content<br>- Removes navigation, footers, and noise |
| Sticky Note1 | Sticky Note | Comment block | — | — | ## 🤖 Step 2: AI Analysis<br>- Sends cleaned content to OpenAI<br>- Extracts structured company intelligence<br>- Identifies positioning, features, personas<br>- Captures pricing, pros/cons, value props<br>- Aggregates all AI results |
| Sticky Note2 | Sticky Note | Comment block | — | — | ## ⚙️ Step 3: HubSpot Preparation<br>- Creates custom fields in HubSpot CRM<br>- Prepares AI-extracted data for import<br>- Splits aggregated data into individual companies<br>- Ready for batch processing |
| Sticky Note3 | Sticky Note | Comment block | — | — | ## 🔄 Step 4: Enrich & Sync to HubSpot<br>- Loops through each company one by one<br>- Enriches with Sona API (firmographics, revenue, employees)<br>- Creates company record in HubSpot<br>- Formats and populates all custom fields<br>- Combines AI insights + Sona data in one profile<br><br>**💡 Get your Sona API key:** https://platform.sonalabs.com/onboardingv2 |
| Sticky Note4 | Sticky Note | Comment block | — | — | # Enrich HubSpot Companies: Firmographics, tech & AI-powered custom attributes<br><br>## ✅ Setup Requirements<br>1. Google Sheets with column `Website Domain`<br>2. OpenAI API Key: https://platform.openai.com<br>3. HubSpot Legacy App token + scopes: `crm.schemas.companies.write`, `crm.objects.companies.write`, `crm.schemas.companies.read`<br>4. Sona API Key: https://platform.sonalabs.com/ |

---

## 4. Reproducing the Workflow from Scratch

1) **Create credentials**
   1. **Google Sheets OAuth2** credential (n8n Google Sheets OAuth2).
   2. **OpenAI API** credential (OpenAI key).
   3. **HubSpot App Token** credential (HubSpot private/legacy app token) with scopes:
      - `crm.schemas.companies.write`
      - `crm.schemas.companies.read`
      - `crm.objects.companies.write`
   4. **Sona API key** (stored securely; you will reference it in an HTTP header).

2) **Google Sheets setup**
   1. Create a Google Sheet with a column header exactly: **`Website Domain`**
   2. Fill rows with domains like `example.com` (optionally full URLs).

3) **Node: Manual Trigger**
   - Add **Manual Trigger** node named `Start`.

4) **Node: Google Sheets → Read rows**
   - Add **Google Sheets** node named `Get Company List from Sheet`.
   - Configure:
     - Credential: your Google Sheets OAuth2
     - Document: select the spreadsheet
     - Sheet/tab: select the correct tab containing `Website Domain`
     - Operation: read/get many rows (default “Read” behavior as per UI)
   - Connect: `Start` → `Get Company List from Sheet`.

5) **Node: HTTP Request (scrape)**
   - Add **HTTP Request** node `Scrape Website Content`.
   - Set **URL** expression to:
     - `($json['Website Domain'] starts with http) ? use it : prepend https://`
   - Enable batching/throttling:
     - Batch size **1**
     - Interval **2000 ms**
   - Connect: `Get Company List from Sheet` → `Scrape Website Content`.

6) **Node: HTML extraction**
   - Add **HTML** node `Extract HTML Content`.
   - Operation: Extract HTML Content
   - Create extraction key `main_content` with selector:
     - `main, body, article, .main-content, #content, p`
   - Skip selectors:
     - `script, img, style, nav, footer, header`
   - Connect: `Scrape Website Content` → `Extract HTML Content`.

7) **Node: Code (clean text + attach domain)**
   - Add **Code** node `Clean & Format Text`.
   - Implement:
     - Split extracted text into lines
     - Filter noise/CTAs/legal
     - Join back into cleaned text
     - Output `{ main_content, website_domain }`
   - Connect: `Extract HTML Content` → `Clean & Format Text`.
   - Note: If you want robustness, avoid index-based matching; pass the domain through from the start (recommended improvement).

8) **Nodes: OpenAI model + structured output**
   1. Add **OpenAI Chat Model** node and select model `gpt-5.1` (or available equivalent).
   2. Add **Structured Output Parser** node and define the JSON schema (fields used in the provided schema example).
   3. Add **AI Agent** node `Analyze with AI`:
      - Provide the prompt instructing JSON-only output.
      - Include `main_content` and set `website_domain` from the item.
      - Enable output parser usage.
   4. Connect:
      - `Clean & Format Text` → `Analyze with AI`
      - `OpenAI Chat Model` → `Analyze with AI` (language model connection)
      - `Structured Output Parser` → `Analyze with AI` (output parser connection)

9) **Node: Aggregate AI results**
   - Add **Aggregate** node `Collect AI Results`.
   - Aggregate field: `output`.
   - Connect: `Analyze with AI` → `Collect AI Results`.

10) **Node: Create HubSpot custom properties (schema)**
   - Add **HTTP Request** node `Create Custom HubSpot Fields`.
   - Configure:
     - Auth: HubSpot App Token credential
     - Method: **POST**
     - URL: **HubSpot batch create properties endpoint** (must be correct for your portal), e.g.:
       - `https://api.hubapi.com/crm/v3/properties/companies/batch/create`
     - JSON body: `{"inputs":[...properties...]}` including all Sona + AI fields you plan to store.
   - Set **On Error**: “Continue (regular output)” if you want idempotent runs when fields already exist.
   - Connect: `Collect AI Results` → `Create Custom HubSpot Fields`.

11) **Node: Set (prepare array)**
   - Add **Set** node `Prepare Data for Loop`.
   - Set field `output` to aggregated array from `Collect AI Results`.
   - Connect: `Create Custom HubSpot Fields` → `Prepare Data for Loop`.

12) **Node: Split Out**
   - Add **Split Out** node `Split Companies and AI Output into Items`.
   - Field to split out: `output`.
   - Connect: `Prepare Data for Loop` → `Split Companies and AI Output into Items`.

13) **Node: Split In Batches (loop)**
   - Add **Split In Batches** node `Loop Through Companies`.
   - Set batch size to **1** (recommended to match the workflow’s use of `.first()` / `.item` references).
   - Connect: `Split Companies and AI Output into Items` → `Loop Through Companies`.

14) **Node: Sona enrich (HTTP)**
   - Add **HTTP Request** node `Sona Enrich`.
   - Method: GET (or as required; current setup uses query parameters).
   - URL: `https://api2.sonalabs.com/resource/company/enrich`
   - Query parameter `website`: based on current item’s `website_domain` (optionally append `.com` if missing a TLD).
   - Headers:
     - `x-api-key: <your Sona key>` (prefer referencing an n8n credential/secret)
     - `accept: application/json`
     - `content-type: application/json`
   - On Error: Continue regular output (optional).
   - Connect: `Loop Through Companies` (items output) → `Sona Enrich`.

15) **Node: HubSpot create company**
   - Add **HubSpot** node `Create HubSpot Company`.
   - Operation: Create company.
   - Map name and standard fields primarily from Sona response, with fallback to AI fields.
   - Connect: `Sona Enrich` → `Create HubSpot Company`.

16) **Node: Code (format custom properties payload)**
   - Add **Code** node `Format Custom Properties`.
   - Build output JSON like:
     - `{ "properties": { tech: "...", key_positioning_elements: "...", ... } }`
   - Join arrays into newline/comma-separated strings for textarea fields.
   - Connect: `Create HubSpot Company` → `Format Custom Properties`.

17) **Node: HTTP PATCH to HubSpot**
   - Add **HTTP Request** node `Update Company with AI Data`.
   - Method: **PATCH**
   - URL: `https://api.hubapi.com/crm/v3/objects/companies/{{companyId}}`
     - `companyId` from the previous HubSpot create node output.
   - Auth: HubSpot App Token credential.
   - Body: the incoming `{properties:{...}}`.
   - Connect: `Format Custom Properties` → `Update Company with AI Data`.

18) **Close the loop**
   - Connect: `Update Company with AI Data` → `Loop Through Companies` (to fetch next batch).
   - Add **NoOp** node `End` and connect the “done/no items” output of `Loop Through Companies` to `End`.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Get your Sona API key | https://platform.sonalabs.com/onboardingv2 |
| OpenAI platform signup/API key | https://platform.openai.com |
| Sona platform signup | https://platform.sonalabs.com/ |
| HubSpot token guidance (Legacy App, required scopes) | HubSpot Settings → Integrations → Legacy Apps; scopes: `crm.schemas.companies.write`, `crm.schemas.companies.read`, `crm.objects.companies.write` |
| Important configuration issue: HubSpot schema creation URL is a placeholder and must be replaced with the correct endpoint | Replace `https://api.hubapi.com/YOUR_AWS_SECRET_KEY_HERE` with the real HubSpot properties batch-create endpoint |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.