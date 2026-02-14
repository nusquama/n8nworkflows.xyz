Analyze website page gaps with GPT-4.1-mini, Perplexity, and Google Docs

https://n8nworkflows.xyz/workflows/analyze-website-page-gaps-with-gpt-4-1-mini--perplexity--and-google-docs-13249


# Analyze website page gaps with GPT-4.1-mini, Perplexity, and Google Docs

## 1. Workflow Overview

**Purpose:**  
This workflow accepts a website URL, fetches the site’s HTML, detects which common high-value pages already exist (About, Contact, Services, etc.), infers a rough business type, then uses an AI agent (GPT‑4.1‑mini) plus Perplexity competitor research to produce a prioritized “missing pages” gap analysis report. The final report is saved into a Google Doc.

**Target use cases:**
- SEO/content planning and site UX improvement
- Competitive benchmarking (page architecture comparison)
- Quick audits for agencies/consultants to identify missing conversion/support pages

### 1.1 Input Reception & HTML Retrieval
User submits a URL via an n8n Form Trigger; the workflow fetches raw HTML from that URL via HTTP.

### 1.2 Page Detection & Business Classification (Local Code)
A Code node parses the HTML to extract internal links, detects presence/absence of standard page types, extracts title/description, and labels an approximate business type.

### 1.3 AI + Competitor Research (Perplexity + GPT‑4.1‑mini)
An n8n LangChain Agent uses:
- **GPT‑4.1‑mini** (with built-in web search enabled)
- **Perplexity Tool** for competitor/page-structure research  
It outputs a structured gap analysis report (text only).

### 1.4 Report Persistence
The generated report text is inserted into a specified Google Doc.

---

## 2. Block-by-Block Analysis

### Block 1.1 — Input Reception & HTML Retrieval
**Overview:** Collects a target URL from a form and downloads the page HTML that will be analyzed downstream.

**Nodes involved:**
- Submit Website URL for Analysis
- Fetch Website HTML

#### Node: “Submit Website URL for Analysis”
- **Type / Role:** `Form Trigger` (`n8n-nodes-base.formTrigger`) — entry point that hosts an n8n form and starts the workflow on submission.
- **Configuration (interpreted):**
  - Form title: **“URL”**
  - One required field labeled **“URL”**
- **Key variables/fields produced:**
  - Output JSON includes `URL` (string) from the form field.
- **Connections:**
  - **Main output →** Fetch Website HTML
- **Version notes:** typeVersion **2.3** (formTrigger behavior varies across n8n versions; ensure your instance supports Form Trigger v2+).
- **Edge cases / failures:**
  - User submits malformed URL (missing scheme, invalid domain). The next HTTP node may fail or fetch unexpected content.
  - Internal/private URLs may be unreachable from n8n environment.

#### Node: “Fetch Website HTML”
- **Type / Role:** `HTTP Request` (`n8n-nodes-base.httpRequest`) — downloads the URL content.
- **Configuration (interpreted):**
  - URL: `={{ $json.URL }}` (from the form trigger output)
  - Uses default options (no explicit headers, timeout, redirects, response type overrides shown).
- **Connections:**
  - **Input:** Submit Website URL for Analysis
  - **Main output →** Detect Existing Pages and Business Type
- **Version notes:** typeVersion **4.3**.
- **Edge cases / failures:**
  - **403/401** due to bot protection (Cloudflare, WAF), missing user-agent headers, or blocked server-to-server requests.
  - **Non-HTML** responses (PDF, JSON, SPA shell) can reduce detection quality.
  - **Redirect loops / too many redirects** (if the default HTTP settings don’t handle them as expected).
  - **Timeouts** on slow websites.

---

### Block 1.2 — Page Detection & Business Classification (Local Code)
**Overview:** Parses fetched HTML to detect which standard page types exist and which are missing, infers a business category, and extracts basic metadata for the AI prompt.

**Nodes involved:**
- Detect Existing Pages and Business Type

#### Node: “Detect Existing Pages and Business Type”
- **Type / Role:** `Code` (`n8n-nodes-base.code`) — custom JavaScript to parse HTML and create a structured analysis object.
- **Configuration (interpreted):**
  - Reads HTML from one of:
    - `$input.first().json.body` or
    - `$input.first().json.data` or
    - `$input.first().json` (fallback)
  - Normalizes HTML to string.
  - Extracts the submitted URL from:
    - `$('Submit Website URL for Analysis').first().json.URL`
  - Extracts links:
    - Regex: `<a[^>]+href=["']([^"']+)["']`
    - Lowercases, trims
  - Normalizes link URLs by removing query params (`?`), anchors (`#`), trailing slash, lowercasing.
  - Detects page presence for these page types:
    - `home, about, contact, services, products, blog, portfolio, pricing, faq, testimonials, team, careers, privacy, terms`
    - Uses both:
      - URL pattern matches (e.g., `/about`, `/contact-us`, `/pricing`, etc.)
      - Keyword checks in anchor tags (e.g., `<a>about</a>`)
  - Produces:
    - `existing_pages[]`, `missing_pages[]`
    - `detection_details{}` with match evidence
  - Business type heuristic from HTML keywords:
    - `ecommerce`, `portfolio / agency`, `blog / content site`, `saas / software`, `service / agency`, else `general`
  - Extracts metadata:
    - `<title>...</title>`
    - `<meta name="description" content="...">`
  - Output includes diagnostics:
    - `total_links`, `html_length`, `ready_for_ai: true`
- **Connections:**
  - **Input:** Fetch Website HTML
  - **Main output →** Generate Gap Analysis Report with AI
- **Version notes:** typeVersion **2** for Code node (ensure your n8n supports this API and `$()` selector usage).
- **Edge cases / failures:**
  - If the HTTP node returns binary/non-string data or a different property structure, HTML extraction may fail → outputs `{ error: ... }` object.
  - Pages loaded dynamically via JS may not appear in HTML, causing false “missing pages.”
  - Link extraction captures **all** `<a href>` including external links, mailto, tel, fragments; normalization helps but doesn’t explicitly filter to same-domain.
  - Home detection patterns (`(^\/|^index|^home)$`) may not match fully qualified URLs; detection may undercount “home.”
  - Keyword anchor detection expects anchors exactly containing the keyword (may miss “About Our Company”).
  - Multi-language sites (non-English navigation labels) may reduce detection accuracy.

---

### Block 1.3 — AI + Competitor Research (Perplexity + GPT‑4.1‑mini)
**Overview:** An AI agent receives the site analysis (business type + existing/missing pages) and uses Perplexity plus GPT‑4.1‑mini (with web search) to confirm gaps, research competitors, and generate a final report with priorities and content suggestions.

**Nodes involved:**
- Generate Gap Analysis Report with AI
- OpenAI GPT-4.1 Mini with Web Search
- Perplexity Competitor Research Tool

#### Node: “OpenAI GPT-4.1 Mini with Web Search”
- **Type / Role:** LangChain Chat Model (`@n8n/n8n-nodes-langchain.lmChatOpenAi`) — provides the LLM for the agent.
- **Configuration (interpreted):**
  - Model: **gpt-4.1-mini**
  - Built-in tool: **webSearch enabled**, context size **medium**
  - Credential: **OpenAI - Keyword Research**
- **Connections:**
  - **AI languageModel output →** Generate Gap Analysis Report with AI (agent uses it as its LLM)
- **Version notes:** typeVersion **1.3** (LangChain nodes can change tool wiring between versions).
- **Edge cases / failures:**
  - OpenAI auth/credit/quota errors.
  - Model availability or permission issues.
  - Web search tool availability depends on n8n/OpenAI integration support; may return limited/blocked results.

#### Node: “Perplexity Competitor Research Tool”
- **Type / Role:** Tool node (`n8n-nodes-base.perplexityTool`) — provides external research capability to the agent.
- **Configuration (interpreted):**
  - Sends a single message prompt instructing Perplexity to:
    1. Crawl and list all pages on target site
    2. Find 5–7 competitors for the detected business type
    3. List competitor page structures
    4. Identify industry-standard pages
  - Explicitly requests **user-facing pages only** (exclude sitemap/robots/admin).
  - Credential: **testing_pplx**
- **Connections:**
  - **AI tool output →** Generate Gap Analysis Report with AI (agent can call this tool)
- **Version notes:** typeVersion **1**.
- **Edge cases / failures:**
  - Perplexity API auth/quota failures.
  - Incomplete crawling due to robots, geo blocks, JavaScript-heavy sites, or rate limits.
  - Competitor identification may be noisy for “general” business type.

#### Node: “Generate Gap Analysis Report with AI”
- **Type / Role:** LangChain Agent (`@n8n/n8n-nodes-langchain.agent`) — orchestrates model + tools to generate the final report.
- **Configuration (interpreted):**
  - Prompt is fully defined in-node (“define” prompt type).
  - Injects expressions from prior nodes:
    - URL from form trigger
    - business_type/title/description/existing_pages/missing_pages from code node
  - Strict rules: never recommend existing pages; exclude technical/admin pages; only business-value pages.
  - Process steps:
    - Use Perplexity to analyze target and competitors
    - Compare against detected pages and industry standards
    - Produce prioritized recommendations with competitor examples + content suggestions
  - Output constraints:
    - “OUTPUT ONLY THE REPORT TEXT - NO CODE, NO JSON, NO MARKDOWN FORMATTING”
  - `hasOutputParser: true` indicates structured output handling is enabled (implementation depends on node internals; here it appears to emit `output` used downstream).
- **Connections:**
  - **Main input:** Detect Existing Pages and Business Type
  - **Uses:** OpenAI GPT-4.1 Mini with Web Search as language model; Perplexity tool as tool
  - **Main output →** Save Report to Google Docs
- **Version notes:** typeVersion **3**.
- **Edge cases / failures:**
  - If the Code node returned an error JSON (e.g., `ready_for_ai` missing), prompt fields may be empty and output quality degrades.
  - The agent may still recommend a page that exists under an unusual slug/name; the prompt attempts to mitigate, but verification is probabilistic.
  - Tool call failures (Perplexity down) can reduce competitor grounding and cause generic suggestions.
  - Long outputs may exceed downstream insertion limits depending on Google Docs node behavior.

---

### Block 1.4 — Report Persistence (Google Docs)
**Overview:** Takes the agent’s generated report text and inserts it into a predefined Google Doc.

**Nodes involved:**
- Save Report to Google Docs

#### Node: “Save Report to Google Docs”
- **Type / Role:** `Google Docs` (`n8n-nodes-base.googleDocs`) — updates an existing document by inserting text.
- **Configuration (interpreted):**
  - Operation: **update**
  - Action: **insert**
  - Text inserted: `={{ $json.output }}` (expects the agent output field to be named `output`)
  - Target Document URL/ID: `1xSDUxPlpSXj2dM3vQtUMHQjVXPE4-XtoB9h4peCFRys`
  - Credential: **testing_doc**
- **Connections:**
  - **Input:** Generate Gap Analysis Report with AI
  - **Output:** none (end)
- **Version notes:** typeVersion **2**.
- **Edge cases / failures:**
  - Google OAuth token expired/revoked; missing permissions to edit the doc.
  - Document ID invalid or not shared with the credential’s Google account.
  - Large inserts may fail or truncate depending on API constraints.
  - If the agent returns a different field name than `output`, nothing meaningful is inserted (blank content).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Submit Website URL for Analysis | n8n-nodes-base.formTrigger | Entry point: collect URL via form | — | Fetch Website HTML | ## AI-Powered Website Gap Analysis Tool  ; ## URL Submission & HTML Fetch |
| Fetch Website HTML | n8n-nodes-base.httpRequest | Download HTML from submitted URL | Submit Website URL for Analysis | Detect Existing Pages and Business Type | ## AI-Powered Website Gap Analysis Tool  ; ## URL Submission & HTML Fetch |
| Detect Existing Pages and Business Type | n8n-nodes-base.code | Parse HTML, detect existing/missing pages, infer business type | Fetch Website HTML | Generate Gap Analysis Report with AI | ## AI-Powered Website Gap Analysis Tool  ; ## Page Detection & AI Analysis |
| Generate Gap Analysis Report with AI | @n8n/n8n-nodes-langchain.agent | Orchestrate LLM + tools; generate final gap report | Detect Existing Pages and Business Type | Save Report to Google Docs | ## AI-Powered Website Gap Analysis Tool  ; ## Page Detection & AI Analysis ; ## Report Generation & Save |
| OpenAI GPT-4.1 Mini with Web Search | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM provider for the agent + web search | — (wired via AI port) | Generate Gap Analysis Report with AI (AI languageModel) | ## AI-Powered Website Gap Analysis Tool  ; ## Page Detection & AI Analysis |
| Perplexity Competitor Research Tool | n8n-nodes-base.perplexityTool | Tool for crawling/competitor research | — (called by agent) | Generate Gap Analysis Report with AI (AI tool) | ## AI-Powered Website Gap Analysis Tool  ; ## Page Detection & AI Analysis |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas documentation | — | — | ## AI-Powered Website Gap Analysis Tool… (content as shown) |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas documentation | — | — | ## URL Submission & HTML Fetch… |
| Sticky Note2 | n8n-nodes-base.stickyNote | Canvas documentation | — | — | ## Page Detection & AI Analysis… |
| Sticky Note3 | n8n-nodes-base.stickyNote | Canvas documentation | — | — | ## Report Generation & Save… |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** in n8n.

2) **Add “Form Trigger” node**
   - Name: **Submit Website URL for Analysis**
   - Form Title: **URL**
   - Add one field:
     - Label: **URL**
     - Required: **true**
   - This node is the entry point.

3) **Add “HTTP Request” node**
   - Name: **Fetch Website HTML**
   - Method: GET (default)
   - URL expression: `={{ $json.URL }}`
   - Connect: **Submit Website URL for Analysis → Fetch Website HTML**

4) **Add “Code” node**
   - Name: **Detect Existing Pages and Business Type**
   - Paste the provided JavaScript logic (the workflow’s code) into the node.
   - Connect: **Fetch Website HTML → Detect Existing Pages and Business Type**
   - Important expectations:
     - The HTTP node should output HTML in `body` (typical) or `data`.
     - The code references the form node by name using: `$('Submit Website URL for Analysis')...` so keep the node name identical (or update the code accordingly).

5) **Add “OpenAI Chat Model (LangChain)” node**
   - Name: **OpenAI GPT-4.1 Mini with Web Search**
   - Model: **gpt-4.1-mini**
   - Enable **built-in webSearch** with **medium** context (as available in your n8n version).
   - Credentials:
     - Create/select **OpenAI API** credentials (API key) with access to the chosen model.

6) **Add “Perplexity Tool” node**
   - Name: **Perplexity Competitor Research Tool**
   - Credentials:
     - Create/select **Perplexity API** credentials (API key).
   - Message content: configure a prompt that includes expressions for:
     - Website URL from the form node
     - Business type from the code node
   - (Match the intent: crawl target site, identify 5–7 competitors, list page structures, derive industry-standard pages; exclude technical/admin pages.)

7) **Add “AI Agent (LangChain)” node**
   - Name: **Generate Gap Analysis Report with AI**
   - Prompt type: **Define**
   - Paste the long system/task prompt from the workflow, including:
     - Existing pages list (must not recommend)
     - Preliminary missing pages
     - Steps requiring Perplexity competitor research
     - Output format requirements (“text only”, no markdown)
   - Connect **main input:**  
     **Detect Existing Pages and Business Type → Generate Gap Analysis Report with AI**
   - Connect **AI ports:**
     - From **OpenAI GPT-4.1 Mini with Web Search**: connect its **AI Language Model** output to the agent’s **AI Language Model** input.
     - From **Perplexity Competitor Research Tool**: connect its **AI Tool** output to the agent’s **AI Tool** input.

8) **Add “Google Docs” node**
   - Name: **Save Report to Google Docs**
   - Operation: **Update**
   - Action: **Insert text**
   - Text expression: `={{ $json.output }}`
   - Document: set the **Document URL/ID** of the target Google Doc.
   - Credentials:
     - Create/select **Google Docs OAuth2** credentials with permission to edit that doc.
   - Connect: **Generate Gap Analysis Report with AI → Save Report to Google Docs**

9) **Validate credentials and permissions**
   - OpenAI: key valid, model allowed, billing/quota OK.
   - Perplexity: key valid, quota OK.
   - Google Docs: OAuth2 authenticated; doc shared with that Google account.

10) **Run a test**
   - Submit a URL via the form.
   - Confirm HTTP returns HTML.
   - Confirm Code node outputs `existing_pages` and `missing_pages`.
   - Confirm agent output contains an `output` field with the report text.
   - Confirm Google Doc receives inserted content.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “AI-Powered Website Gap Analysis Tool… Setup steps: Connect Google Docs OAuth credentials; Add OpenAI API key for GPT‑4.1‑mini; Add Perplexity API key; Update Google Docs document URL; Share the form link…” | Sticky note describing overall purpose and setup |
| “URL Submission & HTML Fetch… Receives website URL via form, then fetches the complete HTML source code…” | Sticky note for the first block |
| “Page Detection & AI Analysis… Analyzes HTML to detect existing pages and business type, then uses AI with Perplexity…” | Sticky note for analysis/AI block |
| “Report Generation & Save… AI creates a comprehensive gap analysis report… saves it directly to Google Docs.” | Sticky note for output block |
| Disclaimer (French): “Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n…” | Provided by requester; confirms content is legal/public and policy-compliant |