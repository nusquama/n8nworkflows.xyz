Create and validate Meta ad copy with GPT-4o, OriginalVoices, and Sheets

https://n8nworkflows.xyz/workflows/create-and-validate-meta-ad-copy-with-gpt-4o--originalvoices--and-sheets-13222


# Create and validate Meta ad copy with GPT-4o, OriginalVoices, and Sheets

## 1. Workflow Overview

**Purpose:** Collect a Meta ads brief via an n8n Form, generate **50 Meta ad copy variations** with **GPT-4o**, then **validate and rank the top 10** using **OriginalVoices Digital Twins** (human-feedback-based AI representations). Finally, append the ranked results to a **Google Sheet**.

**Target use cases:**
- Rapid ideation of Meta ad primary text + headline variations
- Audience-informed filtering using Digital Twins (simulated real audience reactions)
- Standardized reporting to Google Sheets for review/iteration

### 1.1 Input Reception (Form)
Collects product, audience, tone, and destination Sheet URL.

### 1.2 AI Generation (GPT-4o)
An AI Agent prompts GPT-4o to output **exactly 50** JSON-formatted variations.

### 1.3 Parsing + Validation (GPT-4o + OriginalVoices Digital Twins)
Parses JSON from the generator, then a second AI Agent uses the **OriginalVoices MCP tool** (`ask_twins`) to filter and deeply validate, returning **top 10** ranked ads as JSON.

### 1.4 Output (Google Sheets)
Parses top-10 JSON, extracts the Google Sheet ID from the provided URL, formats rows, and appends them into a `Results` tab.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception (Form)

**Overview:** Captures the ad brief and where to store results. This is the workflow entry point.

**Nodes involved:**
- **Ad Brief Form** (form trigger)

#### Node: Ad Brief Form
- **Type / role:** `n8n-nodes-base.formTrigger` — public form + workflow trigger
- **Key configuration (interpreted):**
  - Form title: **Meta Ad Copy Generator**
  - Description: “Generate and validate Meta ad copy variations using AI and real human feedback.”
  - Fields (required unless noted):
    - Product/Brand Name (required)
    - Product Description (required, textarea)
    - Target Audience (required)
    - Key Benefits (required, textarea; user likely enters one per line)
    - Tone (required)
    - Landing Page URL (optional)
    - Google Sheet URL (required)
  - Webhook ID: `meta-ad-generator` (used by n8n internally for the form endpoint)
- **Expressions/variables used downstream:** Output JSON keys exactly match field labels (e.g., `$json['Target Audience']`)
- **Connections:**
  - **Main output →** Generate 50 Variations
- **Potential failures / edge cases:**
  - Users may paste an invalid Google Sheet URL (later causes sheet ID extraction failure).
  - “Key Benefits” may include unexpected formatting; the prompt treats it as raw text.
- **Version-specific notes:** Node shown as **typeVersion 2.2** (form behavior/options can vary across n8n versions).

---

### Block 2 — Generate 50 Variations (GPT-4o)

**Overview:** Uses an AI Agent backed by GPT-4o to produce 50 structured ad variations (JSON array).

**Nodes involved:**
- **Generate 50 Variations** (AI Agent)
- **OpenAI Chat Model** (language model provider)

#### Node: OpenAI Chat Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — provides the chat model to the Agent
- **Key configuration:**
  - Model: **gpt-4o**
  - Options: default (no custom temperature/etc. specified)
- **Connections:**
  - **AI languageModel output →** Generate 50 Variations (AI Agent)
- **Credentials required:** OpenAI API credentials (configured in this node)
- **Potential failures / edge cases:**
  - Auth/insufficient quota/model access
  - Model may return non-JSON or not exactly 50 items (handled later only by parsing, not by strict validation)

#### Node: Generate 50 Variations
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — orchestrates prompt + model response
- **Key configuration (interpreted):**
  - Prompt instructs:
    - Act as expert Meta ads copywriter
    - Generate **50 unique** variations across 6 angles with specific distribution guidance
    - For each variation: `id`, `angle`, `primary_text` (2–4 sentences, strong first 125 chars), `headline` (< 40 chars)
    - Output: **ONLY a valid JSON array**, no markdown/explanation
  - Uses form inputs via expressions, including conditional inclusion of Landing Page URL:
    - `{{ $json['Landing Page URL'] ? '- Landing Page: ' + $json['Landing Page URL'] : '' }}`
- **Inputs:**
  - Main input: from **Ad Brief Form**
  - AI language model: from **OpenAI Chat Model**
- **Outputs / connections:**
  - **Main output →** Parse Variations
- **Potential failures / edge cases:**
  - Agent output might be wrapped in markdown fences (handled by parser).
  - Output might be invalid JSON (parser will throw).
  - Output might be valid JSON but not an array or not 50 objects (workflow does not enforce count beyond storing `variationCount`).

---

### Block 3 — Parse, Filter & Validate with Digital Twins

**Overview:** Converts generator output into structured data, then runs a two-stage evaluation process using OriginalVoices Digital Twins via an MCP tool, returning a top-10 ranked JSON array.

**Nodes involved:**
- **Parse Variations** (Code)
- **Filter & Validate with Digital Twins** (AI Agent)
- **OpenAI Chat Model1** (language model provider)
- **OriginalVoices Digital Twins** (MCP tool)
- **Parse Results** (Code)

#### Node: Parse Variations
- **Type / role:** `n8n-nodes-base.code` — parses JSON from the AI Agent and carries forward form data
- **Key configuration (interpreted):**
  - Reads generator output at: `$input.first().json.output`
  - Strips markdown code fences if present:
    - Removes ```json and ``` blocks via regex
  - Parses JSON into `variations`
  - Pulls original form data from the trigger:
    - `const formData = $('Ad Brief Form').first().json;`
  - Outputs:
    - `variations` (array)
    - `formData` (object)
    - `variationCount`
- **Connections:**
  - **Main output →** Filter & Validate with Digital Twins
- **Potential failures / edge cases:**
  - `output` missing or not a string/object as expected
  - `JSON.parse` failure if model returns malformed JSON
  - If the AI Agent returns already-parsed JSON object/array, logic supports it (`typeof jsonStr === 'string' ? JSON.parse(...) : jsonStr`)

#### Node: OpenAI Chat Model1
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — model provider for validation Agent
- **Key configuration:**
  - Model: **gpt-4o**
- **Connections:**
  - **AI languageModel output →** Filter & Validate with Digital Twins
- **Credentials required:** OpenAI API credentials
- **Potential failures:** same as the first model node (auth, quota, timeouts)

#### Node: OriginalVoices Digital Twins
- **Type / role:** `@n8n/n8n-nodes-langchain.mcpClientTool` — exposes an MCP tool to the Agent (the prompt references `ask_twins`)
- **Key configuration (interpreted):**
  - Endpoint URL: `https://api.originalvoices.ai/mcp`
  - Authentication: **Header Auth**
    - Sticky note specifies:
      - Header: `X-Api-Key`
      - Value: `YOUR_API_KEY`
- **Connections:**
  - **AI tool output →** Filter & Validate with Digital Twins
- **Credentials required:** Header-based API key for OriginalVoices
- **Potential failures / edge cases:**
  - 401/403 if header missing/invalid
  - Endpoint downtime/timeouts
  - Tool-call formatting issues: the Agent must call the correct tool name and schema as exposed by the MCP server

#### Node: Filter & Validate with Digital Twins
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — orchestrates tool calls + model reasoning to rank top ads
- **Key configuration (interpreted):**
  - Receives:
    - `TARGET AUDIENCE` from `$json.formData['Target Audience']`
    - Full variations embedded: `{{ JSON.stringify($json.variations, null, 2) }}`
  - Process defined in prompt:
    1. **Initial filter**: must call `ask_twins` with 2–3 questions; embed batches of variations inside each question (explicitly warns that twins can’t “see” separate inputs).
    2. **Deep validation**: run `ask_twins` again on selected top 10; ask clarity/believability/purchase intent; gather feedback.
  - Output demanded: **ONLY JSON array** of top 10 with fields:
    - `rank`, `id`, `primary_text`, `headline`, `angle`, `resonance_score` (1–10), `feedback`
- **Inputs:**
  - Main: from Parse Variations
  - AI language model: from OpenAI Chat Model1
  - AI tool: from OriginalVoices Digital Twins
- **Outputs / connections:**
  - **Main output →** Parse Results
- **Potential failures / edge cases:**
  - Agent may exceed context limits when embedding all 50 variations + tool prompts (depends on length of copy).
  - Agent might not follow “JSON only” instruction (handled somewhat by Parse Results stripping fences, but not other extraneous text).
  - If the Agent fails to call `ask_twins` properly, validation quality degrades; workflow still continues if it outputs JSON.

#### Node: Parse Results
- **Type / role:** `n8n-nodes-base.code` — parses top-10 JSON and extracts Google Sheet document ID
- **Key configuration (interpreted):**
  - Reads validation output at: `$input.first().json.output`
  - Retrieves `formData` from Parse Variations: `$('Parse Variations').first().json.formData;`
  - Strips markdown fences if present; parses JSON into `topAds`
  - Extracts Sheet ID from `Google Sheet URL` using regex:
    - `/\/d\/([a-zA-Z0-9-_]+)/`
  - Outputs:
    - `topAds` (array)
    - `sheetId` (string or null)
    - `productName`
- **Connections:**
  - **Main output →** Format for Sheets
- **Potential failures / edge cases:**
  - Invalid/missing JSON from the Agent → `JSON.parse` throws
  - Google Sheet URL not matching `/d/<id>` pattern → `sheetId` becomes `null` (next node likely fails)
  - If user shares a URL variant not containing `/d/` (rare but possible), extraction fails

---

### Block 4 — Output to Google Sheets

**Overview:** Converts the top-10 array into row items and appends them into the `Results` sheet tab.

**Nodes involved:**
- **Format for Sheets** (Code)
- **Write to Google Sheet** (Google Sheets append)

#### Node: Format for Sheets
- **Type / role:** `n8n-nodes-base.code` — maps each ranked ad to a row structure
- **Key configuration (interpreted):**
  - Reads `topAds` and `productName` from input
  - Returns one item per ad with columns:
    - `Rank`, `Primary Text`, `Headline`, `Angle`, `Resonance Score`, `Feedback`, `Product`
- **Connections:**
  - **Main output →** Write to Google Sheet
- **Potential failures / edge cases:**
  - If `topAds` is not an array (e.g., parse failure upstream), `.map` throws
  - Missing fields in any `ad` object results in blank cells (not fatal)

#### Node: Write to Google Sheet
- **Type / role:** `n8n-nodes-base.googleSheets` — appends rows
- **Key configuration (interpreted):**
  - Operation: **append**
  - Document ID: expression from Parse Results:
    - `={{ $('Parse Results').first().json.sheetId }}`
  - Sheet/tab name: **Results**
  - Column mapping: **auto map input data** (expects input keys to match sheet headers)
- **Credentials required:** Google Sheets OAuth2
- **Potential failures / edge cases:**
  - `sheetId` null/invalid → document not found error
  - OAuth token expired/insufficient permissions
  - Tab “Results” missing → append fails
  - Header mismatch: if the sheet doesn’t have matching columns, auto-mapping may not behave as expected (often still appends but columns may misalign depending on node behavior/version)
- **Version-specific notes:** Node shown as **typeVersion 4.6** (Google Sheets node options differ across versions).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | n8n-nodes-base.stickyNote | Workflow overview/instructions | — | — | ## Create relevant Meta ads with GPT-4o, OriginalVoices, and Google Sheets \nGenerate 50 ad variations **informed by audience insights**, then **validate with real human feedback** to identify the top 10.\n\n### How it works\n1. **Form** collects product details and target audience\n2. **AI gathers insights** from Digital Twins to understand your audience\n3. **Generate 50 variations** shaped by real human preferences\n4. **Digital Twins validate** each ad for resonance\n5. **Top 10 ranked** to Google Sheet with scores and feedback\n\n### Setup\n1. Connect **OpenAI API** credentials on Chat Model nodes\n2. Connect **Header Auth** on OriginalVoices node:\n   - Header: `X-Api-Key`\n   - Value: `YOUR_API_KEY`\n3. Connect **Google Sheets OAuth** credentials\n4. Create a Google Sheet with tab \"Results\"\n5. **Activate** the workflow\n\n### Customize\n- **Variation count**: Adjust agent prompt\n- **Validation criteria**: Modify Digital Twin questions\n- **Output format**: Add columns as needed\n\n### Need help?\nReach out on Discord (`vedad27`) or email `vedad@originalvoices.ai` |
| Sticky Note Trigger | n8n-nodes-base.stickyNote | Block label | — | — | ### 1. Input Form |
| Ad Brief Form | n8n-nodes-base.formTrigger | Collect inputs / entry point | — | Generate 50 Variations | ### 1. Input Form |
| Sticky Note Generate | n8n-nodes-base.stickyNote | Block label | — | — | ### 2. Generate 50 Variations\nConnect OpenAI credentials |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM provider for generation | — | Generate 50 Variations (ai_languageModel) | ### 2. Generate 50 Variations\nConnect OpenAI credentials |
| Generate 50 Variations | @n8n/n8n-nodes-langchain.agent | Generate 50 ad variations (JSON) | Ad Brief Form; OpenAI Chat Model | Parse Variations | ### 2. Generate 50 Variations\nConnect OpenAI credentials |
| Parse Variations | n8n-nodes-base.code | Parse generation JSON + carry form data | Generate 50 Variations | Filter & Validate with Digital Twins | ### 2. Generate 50 Variations\nConnect OpenAI credentials |
| Sticky Note Validate | n8n-nodes-base.stickyNote | Block label | — | — | ### 3. Filter & Validate\nConnect OriginalVoices credentials |
| OpenAI Chat Model1 | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM provider for validation | — | Filter & Validate with Digital Twins (ai_languageModel) | ### 3. Filter & Validate\nConnect OriginalVoices credentials |
| OriginalVoices Digital Twins | @n8n/n8n-nodes-langchain.mcpClientTool | Digital Twins tool (ask_twins) | — | Filter & Validate with Digital Twins (ai_tool) | ### 3. Filter & Validate\nConnect OriginalVoices credentials |
| Filter & Validate with Digital Twins | @n8n/n8n-nodes-langchain.agent | Tool-assisted ranking + feedback to top 10 | Parse Variations; OpenAI Chat Model1; OriginalVoices Digital Twins | Parse Results | ### 3. Filter & Validate\nConnect OriginalVoices credentials |
| Parse Results | n8n-nodes-base.code | Parse top-10 JSON + extract sheetId | Filter & Validate with Digital Twins | Format for Sheets | ### 3. Filter & Validate\nConnect OriginalVoices credentials |
| Sticky Note Output | n8n-nodes-base.stickyNote | Block label | — | — | ### 4. Output to Sheet\nConnect Google Sheets credentials |
| Format for Sheets | n8n-nodes-base.code | Map top-10 ads into sheet rows | Parse Results | Write to Google Sheet | ### 4. Output to Sheet\nConnect Google Sheets credentials |
| Write to Google Sheet | n8n-nodes-base.googleSheets | Append rows to Google Sheet | Format for Sheets | — | ### 4. Output to Sheet\nConnect Google Sheets credentials |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add a Form Trigger node**
   - Node type: **Form Trigger**
   - Title: `Meta Ad Copy Generator`
   - Description: “Generate and validate Meta ad copy variations using AI and real human feedback.”
   - Add fields (use these exact labels to match expressions):
     1) `Product/Brand Name` (required)  
     2) `Product Description` (required, textarea)  
     3) `Target Audience` (required)  
     4) `Key Benefits` (required, textarea)  
     5) `Tone` (required)  
     6) `Landing Page URL` (optional)  
     7) `Google Sheet URL` (required)
3. **Add an OpenAI Chat Model node** (for generation)
   - Node type: **OpenAI Chat Model (LangChain)**
   - Model: **gpt-4o**
   - Attach **OpenAI credentials** (API key / project as required by your n8n setup).
4. **Add an AI Agent node** named `Generate 50 Variations`
   - Node type: **AI Agent (LangChain)**
   - Prompt mode: “Define” (custom prompt)
   - Paste the generation prompt and reference form fields with expressions:
     - Include the conditional Landing Page line:
       - `{{ $json['Landing Page URL'] ? '- Landing Page: ' + $json['Landing Page URL'] : '' }}`
   - Connect:
     - **Form Trigger → Generate 50 Variations** (main)
     - **OpenAI Chat Model → Generate 50 Variations** (ai_languageModel)
5. **Add a Code node** named `Parse Variations`
   - Node type: **Code**
   - Add JS that:
     - Reads `$input.first().json.output`
     - Strips ```json fences
     - `JSON.parse` into `variations`
     - Pulls form payload from the trigger node
     - Outputs `{ variations, formData, variationCount }`
   - Connect: **Generate 50 Variations → Parse Variations**
6. **Add an OpenAI Chat Model node** (for validation)
   - Model: **gpt-4o**
   - Same OpenAI credentials (or another, if desired)
7. **Add an MCP Client Tool node** named `OriginalVoices Digital Twins`
   - Node type: **MCP Client Tool**
   - Endpoint: `https://api.originalvoices.ai/mcp`
   - Authentication: **Header Auth**
     - Configure credentials/header:
       - Header name: `X-Api-Key`
       - Value: your OriginalVoices API key
8. **Add an AI Agent node** named `Filter & Validate with Digital Twins`
   - Prompt instructs a two-step process:
     - Batch-embed all ad variations into questions when calling `ask_twins`
     - Return only a JSON array of top 10 with rank/score/feedback
   - Connect:
     - **Parse Variations → Filter & Validate with Digital Twins** (main)
     - **OpenAI Chat Model (validation) → Filter & Validate with Digital Twins** (ai_languageModel)
     - **OriginalVoices Digital Twins → Filter & Validate with Digital Twins** (ai_tool)
9. **Add a Code node** named `Parse Results`
   - Parse `$input.first().json.output` into `topAds`
   - Extract `sheetId` from `Google Sheet URL` using `/\/d\/([a-zA-Z0-9-_]+)/`
   - Output `{ topAds, sheetId, productName }`
   - Connect: **Filter & Validate with Digital Twins → Parse Results**
10. **Add a Code node** named `Format for Sheets`
    - Map `topAds` to one item per row with keys:
      - `Rank`, `Primary Text`, `Headline`, `Angle`, `Resonance Score`, `Feedback`, `Product`
    - Connect: **Parse Results → Format for Sheets**
11. **Add a Google Sheets node** named `Write to Google Sheet`
    - Operation: **Append**
    - Document: **By ID**
      - Document ID expression: `{{ $('Parse Results').first().json.sheetId }}`
    - Sheet name: `Results`
    - Mapping: auto-map input keys to columns
    - Connect: **Format for Sheets → Write to Google Sheet**
    - Attach **Google Sheets OAuth2** credentials.
12. In Google Sheets, **create a tab named `Results`** and add headers matching the keys:
    - Rank | Primary Text | Headline | Angle | Resonance Score | Feedback | Product
13. **Activate** the workflow and submit the form.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Connect OpenAI API credentials on Chat Model nodes | Applies to both OpenAI Chat Model nodes |
| Connect Header Auth on OriginalVoices node: Header `X-Api-Key`, Value `YOUR_API_KEY` | Applies to “OriginalVoices Digital Twins” MCP tool node |
| Connect Google Sheets OAuth credentials; create a Google Sheet with tab “Results”; activate the workflow | Output block requirements |
| Customization ideas: variation count (agent prompt), validation criteria (Digital Twin questions), output columns | Prompt + sheet formatting can be modified |
| Support contact: Discord `vedad27` or email `vedad@originalvoices.ai` | From workflow sticky note |
| Disclaimer (provided): Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques. | Context: compliance/attribution note provided with the workflow |