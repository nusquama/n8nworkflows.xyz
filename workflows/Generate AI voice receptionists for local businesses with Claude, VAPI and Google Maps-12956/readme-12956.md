Generate AI voice receptionists for local businesses with Claude, VAPI and Google Maps

https://n8nworkflows.xyz/workflows/generate-ai-voice-receptionists-for-local-businesses-with-claude--vapi-and-google-maps-12956


# Generate AI voice receptionists for local businesses with Claude, VAPI and Google Maps

## 1. Workflow Overview

**Workflow name (internal):** `AI_Local_Business_Receptionist_Agent_Generator_v2`  
**Workflow title (provided):** *Generate AI voice receptionists for local businesses with Claude, VAPI and Google Maps*

### Purpose
This workflow collects a location + keyword query from a user via an n8n Form, scrapes matching local businesses from Google Maps using an Apify actor, fetches each business website, extracts clean text, then uses Claude (Anthropic) to generate voice-agent messaging (system prompt + short greetings). It then provisions a **Vapi assistant** (voice receptionist) through Vapi’s API and logs the created assistant to Google Sheets. A separate “preview” branch shows a results page for the first scraped business.

### Typical use cases
- Rapidly generating AI phone receptionists for a batch of local businesses (e.g., dentists, salons, restaurants).
- Lead-gen / prospecting: build a list of businesses and instantly create a working voice agent demo link.
- Automating agent configuration based on business metadata (category, hours, address, website content).

### Logical blocks
**1.1 Input Reception & Query Construction**  
Collects user inputs and converts them into Apify “Google Maps Scraper” actor parameters.

**1.2 Google Maps Scraping & Iteration**  
Runs Apify actor, receives business results, and iterates over each business.

**1.3 Website Retrieval & Website Text Extraction**  
Fetches each business website HTML and uses an LLM-based extractor to return cleaned text.

**1.4 Business Data Normalization & Storage**  
Reshapes business + website data into a consistent schema and appends it to Google Sheets.

**1.5 Agent Messaging Generation (Claude)**  
Builds a large prompt template and asks Claude to output strict JSON messages (system + short messages).

**1.6 Vapi Assistant Provisioning & Logging**  
Creates a Vapi assistant via HTTP request and stores the assistant ID + URL in Google Sheets.

**1.7 Result Preview Page (first result only)**  
Takes only the first business from the batch and renders an HTML “results” page.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception & Query Construction

**Overview:**  
Captures user inputs (city/region/zip/keywords/count) and transforms them into an Apify actor input payload, including search strings array.

**Nodes involved:**
- **Form Trigger - Business Location Input**
- **Build Apify Request**

#### Node: Form Trigger - Business Location Input
- **Type / role:** `n8n-nodes-base.formTrigger` — public form entry point that starts the workflow.
- **Configuration (interpreted):**
  - Form fields: City, State/Region, ZIPCODE (number), Number of businesses to do (number, default 1), Keywords (comma-separated).
  - Custom CSS and UI text; response mode: **lastNode** (the workflow’s final node response is shown to the user).
- **Outputs:** JSON with submitted form fields.
- **Connections:** → `Build Apify Request`
- **Edge cases / failures:**
  - Users entering non-numeric ZIPCODE despite number type (browser usually enforces, but can still be malformed).
  - “Keywords” empty string or only commas → results in empty `searchStringsArray`.
  - Response mode depends on execution reaching the intended “last” node; if the workflow doesn’t end on the preview branch, user may see unexpected response.

#### Node: Build Apify Request
- **Type / role:** `n8n-nodes-base.code` — builds the request body for Apify Google Maps Scraper.
- **Key logic:**
  - Reads: `$('Form Trigger - Business Location Input').first().json`
  - Splits keywords by comma, trims, filters empty.
  - Builds JSON payload with parameters like:
    - `countryCode: "[Change here]"`
    - `language: "[Input your Language here]"`
    - `locationQuery: `${City}, [Input Here your country]``
    - `maxCrawledPlacesPerSearch` from “Number of businesses to do”
    - `website: "withWebsite"`
- **Connections:** → `Scrape Local Businesses`
- **Edge cases / failures:**
  - Placeholder strings not replaced (`countryCode`, `language`, `locationQuery`) leads to irrelevant/failed scraping.
  - If `Keywords` contains unusual separators (semicolon), the split won’t work as intended.

---

### 2.2 Google Maps Scraping & Iteration

**Overview:**  
Runs the Apify actor to scrape Google Maps businesses and loops through the dataset in batches.

**Nodes involved:**
- **Scrape Local Businesses**
- **Process Businesses**
- **Limit to 1st result for preview** (preview branch)

#### Node: Scrape Local Businesses
- **Type / role:** `@apify/n8n-nodes-apify.apify` — executes Apify actor and returns dataset items.
- **Configuration:**
  - Actor: “Google Maps Scraper (compass/crawler-google-places)”
  - Operation: **Run actor and get dataset**
  - Input body: `={{ JSON.stringify($json) }}` (forwards JSON from Build Apify Request)
- **Connections:** → `Process Businesses`
- **Edge cases / failures:**
  - Apify token/actor permissions (401/403).
  - Actor run fails due to invalid `countryCode`/`locationQuery`, rate limits, or missing paid plan.
  - Dataset may return businesses without `website` even if requested (depends on scraping success).

#### Node: Process Businesses
- **Type / role:** `n8n-nodes-base.splitInBatches` — iterates through scraped businesses.
- **Configuration:** default batch settings (no explicit batch size shown).
- **Connections:**
  - Output 1 → `Limit to 1st result for preview`
  - Output 2 → `Fetch Website`
- **Edge cases / failures:**
  - With default settings, batch size behavior depends on n8n defaults; could unintentionally process many items per execution cycle.
  - If dataset is empty, downstream nodes won’t run; the form response may be blank/unhelpful.

#### Node: Limit to 1st result for preview
- **Type / role:** `n8n-nodes-base.limit` — ensures only the first business is used for HTML preview.
- **Connections:** → `Display Results Page`
- **Edge cases / failures:**
  - If no items, the node outputs nothing and preview page won’t render.

---

### 2.3 Website Retrieval & Website Text Extraction

**Overview:**  
Fetches each business website and extracts simplified “clean text” from the page content for later prompt customization.

**Nodes involved:**
- **Fetch Website**
- **Grok-4-fast**
- **Extract Website Text / Clean**

#### Node: Fetch Website
- **Type / role:** `n8n-nodes-base.httpRequest` — downloads the business website page.
- **Configuration:**
  - URL expression: `={{ $json.website.match(/https:\/\/.*/)[0] }}`
    - Forces HTTPS and takes substring matched by regex.
  - Timeout: 10s
  - Allow unauthorized certs: true
  - `continueOnFail: true` (workflow continues even if HTTP fails)
- **Connections:** → `Extract Website Text / Clean`
- **Edge cases / failures:**
  - If `website` is missing or is `http://...`, the regex may return `null` and the expression will throw (even with continueOnFail, expression evaluation can fail before request).
  - Non-HTML responses, redirects, bot protection (Cloudflare), long pages > timeout.
  - SSL issues mitigated by allowUnauthorizedCerts, but not all.

#### Node: Grok-4-fast
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenRouter` — provides an LLM for the extraction node (LangChain model provider).
- **Configuration:**
  - Model: `x-ai/grok-4-fast` via OpenRouter credentials.
- **Connections:** Supplies `ai_languageModel` to `Extract Website Text / Clean`.
- **Edge cases / failures:**
  - OpenRouter auth errors or quota.
  - Model availability changes on OpenRouter.

#### Node: Extract Website Text / Clean
- **Type / role:** `@n8n/n8n-nodes-langchain.informationExtractor` — extracts structured fields from unstructured text/HTML.
- **Configuration:**
  - Input text: `={{ $json.data || "Aucun//tu peux output un item qui dit que le site n'était pas accesible" }}`
    - Uses the HTTP response `data` field if present, otherwise French fallback string.
  - Attributes requested:
    - `Clean Text` (required) — described as “Text without HTML tags.”
  - Uses LLM from `Grok-4-fast` via the AI connection.
- **Connections:** → `Format Business Data`
- **Edge cases / failures:**
  - If `Fetch Website` returns binary or different schema (not `data`), extraction may run on fallback text, degrading quality.
  - LLM extractor can hallucinate; output field name must match exactly.
  - Cost/time: LLM calls per business can be expensive.

---

### 2.4 Business Data Normalization & Storage

**Overview:**  
Combines Apify business fields with extracted website text and stores a normalized row in Google Sheets.

**Nodes involved:**
- **Format Business Data**
- **Sync to Google Sheet**

#### Node: Format Business Data
- **Type / role:** `n8n-nodes-base.set` — maps/renames fields into a target schema.
- **Configuration highlights:**
  - Builds fields like:
    - `Nom du business` = `$('Scrape Local Businesses').item.json.title`
    - `Catégorie` = `...categoryName`
    - `Rue` = `...street`
    - `Ville` = `...city`
    - `Code postal` = `...postalCode`
    - `Site web` = `...website`
    - `N° Tél` = `...phone`
    - `Opening Hours` as array from `...openingHours`
    - `Infos` as object from `...additionalInfo`
    - `Site Web Data` = `={{ $json.output['Texte clean'] }}`
      - **Potential mismatch:** the extractor defines attribute `Clean Text`, but this references French key `Texte clean`. This likely breaks or leaves empty.
  - `include: none` (only set fields are kept)
  - ignore conversion errors enabled
- **Connections:** → `Sync to Google Sheet`
- **Edge cases / failures:**
  - Misaligned extractor output key (likely the biggest functional bug here).
  - Using `$('Scrape Local Businesses').item.json...` inside a loop: pairing depends on item linkage; if pairing breaks, fields can mismatch across businesses.

#### Node: Sync to Google Sheet
- **Type / role:** `n8n-nodes-base.googleSheets` — appends a row with normalized business data.
- **Configuration:**
  - Operation: **append**
  - DocumentId / SheetName are placeholders: `=[PASTE URL HERE]`, `=[PASTE ID HERE]`
  - Column mapping expects keys like:
    - `Business Name`, `Category`, `Street`, `City`, `ZIPCODE`, `Phone Number`, `Website`, `Opening Hours`, `Website Data`, `Infos`
  - Phone formatting: `={{ $json["Phone Number"].replace(/[\s+]/g, '') }}`
- **Connections:** → `Generate Agent Messages`
- **Edge cases / failures:**
  - Required setup not completed: if Sheet ID/URL not set, node fails.
  - Mapping key mismatch: this node expects English keys that are **not** produced by `Format Business Data` (which produces French labels like “Nom du business”, “Rue”, etc.). Unless there is missing transformation, this is another major mismatch.
  - OAuth token expired or insufficient permissions.

---

### 2.5 Agent Messaging Generation (Claude)

**Overview:**  
Uses Claude to generate a professional system prompt and short voice-agent messages tailored to the business.

**Nodes involved:**
- **Generate Agent Messages**
- **Parse Agent Responses**

#### Node: Generate Agent Messages
- **Type / role:** `@n8n/n8n-nodes-langchain.anthropic` — calls Anthropic Claude chat model.
- **Configuration:**
  - Model: `claude-sonnet-4-20250514`
  - Strong system instruction: produce **only strict JSON**, no line breaks, professional tone; system prompt 300–500 words; voicemail/endCall 1–4 words.
  - User message content is a very large template with multiple business archetypes (hair salon, dentist, restaurant, spa) plus “intelligent customization” rules.
  - Uses expressions referencing:
    - `$('Business Parameters').item.json[...]` (node does not exist in this workflow)
    - `$('Scrape Local Businesses').item.json.categoryName`
- **Connections:** → `Parse Agent Responses`
- **Edge cases / failures:**
  - **Critical:** references to non-existent node **Business Parameters** will throw expression errors.
  - Mixed languages: “Perfect English” required, but later Vapi `endCallMessage` is in French.
  - Claude may sometimes wrap JSON in text; downstream parser tries to extract JSON anyway, but can fail if malformed.
  - Token limits: prompt is very long; may exceed model limits depending on plan.

#### Node: Parse Agent Responses
- **Type / role:** `n8n-nodes-base.code` — extracts JSON from the LLM response and produces a `messages` array.
- **Key logic:**
  - Reads: `$input.first().json.content[0].text`
  - Extracts substring between first `{` and last `}` and `JSON.parse`.
  - Outputs:
    - `messages[0]` system_message
    - `messages[1]` first_message
    - `messages[2]` voicemail_message
  - **Note:** `endCallMessage` from Claude is ignored here (not included in messagesArray).
- **Connections:** → `Create Vapi Assistant`
- **Edge cases / failures:**
  - If Claude returns no braces or additional braces in text, substring extraction can produce invalid JSON.
  - If response schema changes (Anthropic node output structure differs), `content[0].text` may not exist.

---

### 2.6 Vapi Assistant Provisioning & Logging

**Overview:**  
Creates a Vapi assistant using the generated system prompt + greetings, then logs the created assistant to Google Sheets.

**Nodes involved:**
- **Create Vapi Assistant**
- **Log Agent to Sheet**

#### Node: Create Vapi Assistant
- **Type / role:** `n8n-nodes-base.httpRequest` — POST to Vapi Assistant API.
- **Configuration:**
  - POST `https://api.vapi.ai/assistant`
  - Auth: `httpHeaderAuth` (generic header auth credential)
  - JSON body includes:
    - `name`: from `$('Scrape Local Businesses').item.json.title`
    - `voice`: 11labs voice settings (model `eleven_turbo_v2_5`, voiceId hardcoded, stability/similarity)
    - `model`: provider `openai`, model `gpt-4o-mini`, system message from `$json.messages[0].content`, maxTokens 200, temperature 0.5
    - `firstMessage`: `$json.messages[1].content`
    - `voicemailMessage`: `$json.messages[2].content`
    - `endCallMessage`: hardcoded French: `Merci de votre appel, à bientôt chez ...`
    - `transcriber`: deepgram `nova-2`, language `fr`
  - `onError: continueRegularOutput` (continues even if API fails)
- **Connections:** → `Log Agent to Sheet`
- **Edge cases / failures:**
  - Missing/incorrect Authorization header (explicitly noted in sticky note).
  - VoiceId may be invalid or not accessible to your ElevenLabs account (Vapi-side validation).
  - Language mismatch: transcriber `fr` while UI preview claims “English” in one place; Claude is instructed “Perfect English”.
  - If API fails but node continues, downstream may log incomplete data (e.g., missing `id`).

#### Node: Log Agent to Sheet
- **Type / role:** `n8n-nodes-base.googleSheets` — upserts assistant record into tracking sheet.
- **Configuration:**
  - Operation: **appendOrUpdate**
  - Matching column: `Nom du business`
  - Writes:
    - `Agent ID` = `{{ $json.id }}`
    - `Agent URL` = `https://dashboard.vapi.ai/assistants/{{ $json.id }}`
    - `Nom du business` = `{{ $json.name }}`
  - DocumentId / SheetName placeholders not set.
- **Connections:** → `Process Businesses` (loops to next batch item)
- **Edge cases / failures:**
  - Missing sheet configuration.
  - If `Create Vapi Assistant` failed and returned no `id`, the URL becomes invalid and update may overwrite rows unexpectedly.

---

### 2.7 Result Preview Page (first result only)

**Overview:**  
Generates a formatted HTML page showing the generated messages and the Vapi assistant link. This is intended as the form completion response.

**Nodes involved:**
- **Display Results Page**

#### Node: Display Results Page
- **Type / role:** `n8n-nodes-base.form` (completion) — returns a custom HTML page as the “completion” response.
- **Configuration:**
  - Operation: completion
  - Respond with: showText
  - HTML uses expressions referencing:
    - `$json["Business Name"]`
    - `$('Parse Agent Responses').item.json.messages[...]`
    - `$('Create Vapi Assistant').item.json.id`
- **Connections:** none (terminal)
- **Edge cases / failures:**
  - Data mismatch: the workflow’s business object likely uses `title` (Apify), not `Business Name`. If the key doesn’t exist, the page renders blanks.
  - If preview branch executes before assistant creation for that item (because it branches directly from `Process Businesses`), `Create Vapi Assistant` may not have run yet for the preview item, causing broken link rendering.
  - Mixed language: page shows “English” in transcriber config while actual body uses `language: "fr"` in Vapi request.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Form Trigger - Business Location Input | n8n-nodes-base.formTrigger | Collect user location/keyword parameters | — | Build Apify Request | ## 1. Data Collection & Scraping<br><br>Captures user input via form and scrapes matching businesses from Google Maps using Apify. |
| Build Apify Request | n8n-nodes-base.code | Build Apify actor input payload | Form Trigger - Business Location Input | Scrape Local Businesses | **⚠️ CONFIGURE FOR YOUR COUNTRY**<br><br>Update:<br>- `countryCode`: "us" (or your country)<br>- `language`: "en" (or your language)<br>- `locationQuery`: "${city}, United States" (or your country) |
| Scrape Local Businesses | @apify/n8n-nodes-apify.apify | Scrape Google Maps businesses via Apify | Build Apify Request | Process Businesses | ## 1. Data Collection & Scraping<br><br>Captures user input via form and scrapes matching businesses from Google Maps using Apify. |
| Process Businesses | n8n-nodes-base.splitInBatches | Iterate through scraped businesses | Scrape Local Businesses | Limit to 1st result for preview; Fetch Website | ## 1. Data Collection & Scraping<br><br>Captures user input via form and scrapes matching businesses from Google Maps using Apify. |
| Limit to 1st result for preview | n8n-nodes-base.limit | Limit preview to a single business | Process Businesses | Display Results Page |  |
| Display Results Page | n8n-nodes-base.form | Render HTML results page (completion) | Limit to 1st result for preview | — |  |
| Fetch Website | n8n-nodes-base.httpRequest | Download business website HTML | Process Businesses | Extract Website Text / Clean | ## 2. Business Analysis<br>Fetches each business website and extracts relevant text content using LLM. |
| Grok-4-fast | @n8n/n8n-nodes-langchain.lmChatOpenRouter | LLM provider for website extraction | — | Extract Website Text / Clean (AI model connection) |  |
| Extract Website Text / Clean | @n8n/n8n-nodes-langchain.informationExtractor | Extract “clean text” from website HTML | Fetch Website; Grok-4-fast | Format Business Data | ## 2. Business Analysis<br>Fetches each business website and extracts relevant text content using LLM. |
| Format Business Data | n8n-nodes-base.set | Normalize business fields + website text | Extract Website Text / Clean | Sync to Google Sheet | ## 2. Business Analysis<br>Fetches each business website and extracts relevant text content using LLM. |
| Sync to Google Sheet | n8n-nodes-base.googleSheets | Store business record in Sheets | Format Business Data | Generate Agent Messages |  |
| Generate Agent Messages | @n8n/n8n-nodes-langchain.anthropic | Generate system prompt + greetings via Claude | Sync to Google Sheet | Parse Agent Responses | ## 3. Agent Generation<br>Uses Claude to generate customized voice agent prompts based on business type and data. |
| Parse Agent Responses | n8n-nodes-base.code | Parse Claude JSON output into message array | Generate Agent Messages | Create Vapi Assistant | ## 4. VAPI Deployment<br>Creates the voice assistant via VAPI API and logs details to Google Sheets. |
| Create Vapi Assistant | n8n-nodes-base.httpRequest | Provision assistant in Vapi | Parse Agent Responses | Log Agent to Sheet | ## 4. VAPI Deployment<br>Creates the voice assistant via VAPI API and logs details to Google Sheets.<br>**⚠️ REQUIRED:** Add your VAPI API key in the Authorization header |
| Log Agent to Sheet | n8n-nodes-base.googleSheets | Log/Upsert created assistant in Sheets | Create Vapi Assistant | Process Businesses | ## 4. VAPI Deployment<br>Creates the voice assistant via VAPI API and logs details to Google Sheets. |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation | — | — | **⚠️ CONFIGURE FOR YOUR COUNTRY**<br><br>Update:<br>- `countryCode`: "us" (or your country)<br>- `language`: "en" (or your language)<br>- `locationQuery`: "${city}, United States" (or your country) |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation | — | — | ## 1. Data Collection & Scraping<br><br>Captures user input via form and scrapes matching businesses from Google Maps using Apify. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Documentation | — | — | ## AI Voice Agent Generator for Local Businesses<br><br>Automatically create personalized AI voice receptionist agents for local businesses using VAPI, by scraping Google Maps data and analyzing their websites.<br><br>### How it works<br>1. **Form input**: User submits city, region, postal code, and business keywords<br>2. **Google Maps scraping**: Apify scrapes local businesses matching criteria<br>3. **Website analysis**: LLM extracts relevant content from each business website<br>4. **AI prompt generation**: Claude generates customized system prompts, greetings, and voicemail messages based on business type<br>5. **VAPI provisioning**: Automatically creates voice agents via VAPI API<br>6. **Logging**: Stores all generated agents in Google Sheets for tracking<br><br>### Setup steps<br>1. Connect **Apify** credentials (Google Maps Scraper actor)<br>2. Connect **Anthropic** API key<br>3. Connect **OpenRouter** API key (for website analysis)<br>4. Add your **VAPI API key** in the HTTP Request node<br>5. Connect **Google Sheets** OAuth<br>6. Activate the workflow<br><br>### Customization<br>- Modify business category templates in the "Generate Agent Messages" node<br>- Adjust voice settings (stability, similarity) in the VAPI provisioning node<br>- Change the LLM model for prompt generation (default: Claude Sonnet) |
| Sticky Note3 | n8n-nodes-base.stickyNote | Documentation | — | — | ## 2. Business Analysis<br>Fetches each business website and extracts relevant text content using LLM. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Documentation | — | — | ## 3. Agent Generation<br>Uses Claude to generate customized voice agent prompts based on business type and data. |
| Sticky Note5 | n8n-nodes-base.stickyNote | Documentation | — | — | ## 4. VAPI Deployment<br>Creates the voice assistant via VAPI API and logs details to Google Sheets. |
| Sticky Note6 | n8n-nodes-base.stickyNote | Documentation | — | — | **⚠️ REQUIRED:** Add your VAPI API key in the Authorization header |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it (e.g.) `AI_Local_Business_Receptionist_Agent_Generator_v2`.

2. **Add “Form Trigger” node** (`Form Trigger - Business Location Input`)
   - Fields:
     - City (text, required)
     - State / Region (text, required)
     - ZIPCODE (number, required)
     - Number of businesses to do (number, required, default 1)
     - Keywords (separated by comma) (text, required)
   - Set **Response mode** to **Last node**.
   - (Optional) paste the custom CSS and button label text.

3. **Add “Code” node** (`Build Apify Request`)
   - Read form values and build:
     - `searchStringsArray` = split keywords by `,`, trim, remove empties
     - Apify payload fields: `countryCode`, `language`, `locationQuery`, `maxCrawledPlacesPerSearch`, `website: "withWebsite"`, etc.
   - **Important:** replace placeholders:
     - `countryCode` (e.g., `"fr"`, `"us"`)
     - `language` (e.g., `"fr"`, `"en"`)
     - `locationQuery` country string (e.g., `"Paris, France"`)

4. **Add Apify node** (`Scrape Local Businesses`)
   - Node type: **Apify**
   - Operation: **Run actor and get dataset**
   - Actor: **Google Maps Scraper (compass/crawler-google-places)**
   - Body: pass the JSON from previous node (`JSON.stringify($json)` or direct object depending on node options).
   - **Credentials:** add Apify API token.

5. **Add “Split in Batches” node** (`Process Businesses`)
   - Connect from Apify node output.
   - Configure batch size as desired (if you need strict 1-by-1, set batch size to 1 explicitly).

6. **(Preview branch) Add “Limit” node** (`Limit to 1st result for preview`)
   - Connect from `Process Businesses` first output.
   - Limit: 1

7. **Add “Form (Completion)” node** (`Display Results Page`)
   - Operation: **completion**
   - Respond with: **showText**
   - Paste HTML template and ensure referenced variables match your actual item schema.
   - Connect from `Limit to 1st result for preview`.

8. **(Main processing branch) Add “HTTP Request” node** (`Fetch Website`)
   - Connect from `Process Businesses` second output.
   - URL: use business website field (avoid forcing https unless guaranteed).
     - Safer expression: if website starts with http use it, else prepend `https://`.
   - Timeout ~10s.
   - Enable **Continue On Fail**.

9. **Add OpenRouter LLM node** (`Grok-4-fast`)
   - Node type: **OpenRouter Chat Model**
   - Model: `x-ai/grok-4-fast`
   - **Credentials:** OpenRouter API key.

10. **Add “Information Extractor” node** (`Extract Website Text / Clean`)
   - Text input: use `Fetch Website` response body (ensure correct field; n8n HTTP node often returns `body`).
   - Add attribute:
     - `Clean Text` (required)
   - Connect AI model input from `Grok-4-fast`.

11. **Add “Set” node** (`Format Business Data`)
   - Map Apify fields (title, categoryName, street, etc.) and extracted `Clean Text`.
   - Ensure you reference the extractor output key exactly (e.g., `$json.output["Clean Text"]`).
   - Keep/merge fields as needed.

12. **Add Google Sheets node** (`Sync to Google Sheet`)
   - Operation: **Append**
   - Select Spreadsheet + Sheet.
   - Map columns to the fields created in `Format Business Data`.
   - **Credentials:** Google Sheets OAuth2.

13. **Add Anthropic node** (`Generate Agent Messages`)
   - Model: `claude-sonnet-4-20250514` (or available equivalent)
   - System message: enforce strict JSON output.
   - User message: include templates + business data.
   - **Fix references:** only reference nodes that exist (e.g., use current item fields rather than `Business Parameters`).

14. **Add “Code” node** (`Parse Agent Responses`)
   - Parse Claude output JSON and produce:
     - system message
     - first message
     - voicemail message
     - (optionally) end-call message too (recommended)

15. **Add “HTTP Request” node** (`Create Vapi Assistant`)
   - POST `https://api.vapi.ai/assistant`
   - Header: `Content-Type: application/json`
   - Auth: Header Auth with `Authorization: Bearer <VAPI_API_KEY>` (or the scheme required by Vapi).
   - Body: include voice provider settings, LLM config, messages, transcriber.
   - Consider aligning language fields (Claude English vs transcriber French).

16. **Add Google Sheets node** (`Log Agent to Sheet`)
   - Operation: **Append or Update**
   - Match on business name column.
   - Store assistant `id` and assistant dashboard URL.

17. **Loop batches**
   - Connect `Log Agent to Sheet` back to `Process Businesses` to fetch next batch.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques. | Provided disclaimer |
| **⚠️ CONFIGURE FOR YOUR COUNTRY**: Update `countryCode`, `language`, `locationQuery` | Sticky note guidance |
| **⚠️ REQUIRED:** Add your VAPI API key in the Authorization header | Sticky note guidance |
| High-level workflow description + setup steps + customization pointers | Sticky note “AI Voice Agent Generator for Local Businesses” |

