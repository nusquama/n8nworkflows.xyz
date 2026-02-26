Assess technical documentation compliance with GPT‑4o and send Slack alerts

https://n8nworkflows.xyz/workflows/assess-technical-documentation-compliance-with-gpt-4o-and-send-slack-alerts-13597


# Assess technical documentation compliance with GPT‑4o and send Slack alerts

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** This workflow fetches technical documentation (by URL), cleans/prepares it, sends it to **OpenAI GPT‑4o** to score compliance across four dimensions, determines an overall **PASS/WARNING/FAIL** status, sends a **Slack alert** to the appropriate channel, and logs results to **Google Sheets**.

**Target use cases:**
- Engineering or platform teams validating API documentation quality
- Product documentation governance (terminology consistency, required sections)
- Localization readiness checks before translation handoff
- Automated quality gates for doc release cycles

### Logical Blocks
**1.1 Input Reception & Metadata Setup**  
Manual execution triggers a Set node that defines the document URL and metadata.

**1.2 Fetch & Prepare Documentation**  
HTTP fetches document content as text; a Code node cleans HTML/whitespace, truncates content, and adds heuristic flags (headers, code blocks, examples, warnings).

**1.3 AI Scoring (GPT‑4o via OpenAI Chat Completions)**  
HTTP Request sends a strict “return ONLY valid JSON” prompt to GPT‑4o to obtain per-dimension scores and gaps.

**1.4 Parse, Score Aggregation & Routing**  
A Code node parses model output, computes a weighted overall score, assigns PASS/WARNING/FAIL; Switch routes by status.

**1.5 Slack Reporting & Audit Logging**  
A Code node builds Slack Block Kit payload; three Slack webhook calls exist (FAIL/WARNING/PASS). All paths append a row to Google Sheets.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception & Metadata Setup
**Overview:** Starts the workflow manually and defines the documentation source and metadata used downstream (title/type/date).  
**Nodes involved:**  
- When clicking 'Execute workflow'  
- Set Document Input

#### Node: When clicking 'Execute workflow'
- **Type / role:** `Manual Trigger` — entry point for ad-hoc runs.
- **Configuration:** No parameters; user clicks “Execute workflow”.
- **Connections:** Outputs to **Set Document Input**.
- **Edge cases:** None (only runs when executed manually).
- **Version notes:** TypeVersion 1 (standard).

#### Node: Set Document Input
- **Type / role:** `Set` — creates/overrides fields used across the workflow.
- **Configuration choices:**
  - Sets:
    - `doc_url` (string): default placeholder `https://your-docs-site.com/api/endpoint-reference`
    - `doc_title` (string): `API Endpoint Reference v2.1`
    - `doc_type` (string): `API Reference`
    - `review_date` (string): expression `new Date().toISOString().split('T')[0]` (YYYY-MM-DD)
- **Key expressions/variables:**
  - `={{ new Date().toISOString().split('T')[0] }}`
- **Connections:** Input from Manual Trigger; output to **Fetch Document Content**.
- **Edge cases / failures:**
  - Invalid or unreachable `doc_url` will break the next HTTP Request.
- **Version notes:** TypeVersion 3.4; uses assignments array style.

---

### 2.2 Fetch & Prepare Documentation
**Overview:** Downloads the document as raw text and normalizes it into an AI-friendly input (clean text + truncation + basic heuristics).  
**Nodes involved:**  
- Fetch Document Content  
- Prepare Document for AI

#### Node: Fetch Document Content
- **Type / role:** `HTTP Request` — fetches the documentation from `doc_url`.
- **Configuration choices:**
  - **URL:** `={{ $json.doc_url }}`
  - **Response format:** `text`
- **Connections:** Input from **Set Document Input**; output to **Prepare Document for AI**.
- **Edge cases / failures:**
  - 401/403 on protected docs
  - 404 or DNS errors
  - Large responses (performance/timeouts)
  - Non-text/binary responses (still forced to text)
- **Version notes:** TypeVersion 4.2.

#### Node: Prepare Document for AI
- **Type / role:** `Code` — cleans content and enriches metadata for scoring.
- **Configuration choices (interpreted):**
  - Reads fetched content from: `rawContent = $input.item.json.data || ''`  
    (n8n HTTP Request commonly returns body in `json.data` when responseFormat is text)
  - Strips HTML tags via regex, decodes a few HTML entities, collapses whitespace.
  - Truncates clean content to ~6000 characters to reduce token usage.
  - Computes:
    - `word_count`
    - `pre_analysis` flags:
      - `has_code_blocks` (``` or `<code>`)
      - `has_headers` (`#` or `<h1-6>`)
      - `has_examples` (regex match on “example/sample/e.g.”)
      - `has_warnings` (warning/caution/note/important/deprecated)
  - Outputs: `doc_*`, `review_date`, `clean_content`, `word_count`, `pre_analysis`
- **Key expressions/variables:** Uses `$input.item.json` to merge metadata with fetched content.
- **Connections:** Input from **Fetch Document Content**; output to **Score Documentation with AI**.
- **Edge cases / failures:**
  - If HTTP node returns text under a different field than `.data`, `rawContent` becomes empty, producing low scores.
  - Regex HTML stripping can remove meaningful code formatting or inline symbols.
  - Truncation can cut mid-section and bias scoring (especially completeness).
- **Version notes:** TypeVersion 2.

---

### 2.3 AI Scoring (GPT‑4o via OpenAI)
**Overview:** Sends the prepared document to GPT‑4o with a strict instruction to return a specific JSON schema containing scores and gaps.  
**Nodes involved:**  
- Score Documentation with AI

#### Node: Score Documentation with AI
- **Type / role:** `HTTP Request` — calls OpenAI Chat Completions API.
- **Configuration choices:**
  - **POST** `https://api.openai.com/v1/chat/completions`
  - Auth: **predefined credential** `openAiApi`
  - Body includes:
    - `model: "gpt-4o"`
    - `temperature: 0.2` (more deterministic)
    - `messages`:
      - system prompt defines 4 dimensions and requires **ONLY valid JSON** with an exact structure
      - user message injects:
        - `{{ $json.doc_title }}`
        - `{{ $json.doc_type }}`
        - `{{ $json.word_count }}`
        - `{{ $json.clean_content }}`
- **Connections:** Input from **Prepare Document for AI**; output to **Parse AI Compliance Report**.
- **Edge cases / failures:**
  - Credential missing/invalid: 401
  - Model name unavailable for the account/region
  - Rate limits (429) or temporary outages (5xx)
  - Model sometimes returns non-JSON despite instruction (handled later)
  - Payload too large (400) if content not properly truncated
- **Version notes:** TypeVersion 4.2; uses `nodeCredentialType: openAiApi`.

---

### 2.4 Parse, Score Aggregation & Routing
**Overview:** Parses the model response, computes a weighted overall score, assigns a compliance status, and routes to the correct Slack path.  
**Nodes involved:**  
- Parse AI Compliance Report  
- Route by Compliance Status

#### Node: Parse AI Compliance Report
- **Type / role:** `Code` — robust parsing + scoring logic.
- **Configuration choices (interpreted):**
  - Reads AI output from: `choices[0].message.content`
  - Strips common markdown fences (` ```json ` / ` ``` `), then `JSON.parse`
  - On parse failure: produces a fallback report with structure score 0 and a “Failed to parse AI response” gap
  - Extracts scores (default 0 if missing)
  - Computes weighted `overallScore`:
    - Structure 25%
    - Terminology 25%
    - Localization 20%
    - Completeness 30%
  - Determines:
    - `FAIL` if score < 60
    - `WARNING` if score < 80
    - else `PASS`
  - Adds `statusEmoji` and `statusColor` (note: `statusColor` is not used later in Slack payload)
  - Flattens all gaps into `[{dimension, gap}, ...]`, counts them, and generates `reportId = DOC-<timestamp>`
  - Also preserves `pre_analysis` from upstream
- **Connections:** Input from **Score Documentation with AI**; output to **Route by Compliance Status**.
- **Edge cases / failures:**
  - OpenAI response shape changes (missing `choices[0]`) → will parse `'{}'` and yield all zeros (PASS/WARNING/FAIL determined by overallScore = 0 ⇒ FAIL).
  - If AI returns correct JSON but with wrong keys/types, defaults may hide issues (scores become 0).
- **Version notes:** TypeVersion 2.

#### Node: Route by Compliance Status
- **Type / role:** `Switch` — routes by `complianceStatus`.
- **Configuration choices:**
  - Three explicit equals conditions:
    - `FAIL`
    - `WARNING`
    - `PASS`
  - Outputs are renamed: `FAIL`, `WARNING`, `PASS`
- **Key expressions:** `={{ $json.complianceStatus }}`
- **Connections:**
  - Input from **Parse AI Compliance Report**
  - All three outputs connect to **Build Slack Report**
- **Edge cases / failures:**
  - If `complianceStatus` is anything else (unexpected value), item may not route anywhere (no default route configured).
- **Version notes:** TypeVersion 3.

---

### 2.5 Slack Reporting & Audit Logging
**Overview:** Builds a Block Kit Slack message and posts it via different webhooks depending on status; logs every run into Google Sheets.  
**Nodes involved:**  
- Build Slack Report  
- Slack — FAIL Alert (Engineering Channel)  
- Slack — WARNING Alert (Review Channel)  
- Slack — PASS Notice (Docs Channel)  
- Log Result to Google Sheets

#### Node: Build Slack Report
- **Type / role:** `Code` — constructs Slack payload (text + blocks).
- **Configuration choices (interpreted):**
  - Builds a 10-segment bar visualization per dimension using `█` and `░`
  - Includes:
    - Header with status emoji and compliance status
    - Fields: doc title, doc type, overall score, review date
    - Dimension score breakdown
    - AI summary
    - Gap list (up to 6, then “…and N more”)
    - Context line: report ID + source URL
  - Outputs `slackPayload` under the item JSON
- **Connections:** Input from **Route by Compliance Status**; output fan-out to all three Slack webhook nodes (see note below).
- **Edge cases / failures:**
  - Slack Block Kit limits: very long gap strings may exceed block text limits; only first 6 gaps are included but a single gap can still be too long.
  - Special characters may need escaping in rare cases.
- **Version notes:** TypeVersion 2.
- **Important behavior note:** Although the Switch routes to Build Slack Report, **Build Slack Report is connected to all three Slack webhook nodes simultaneously**. Unless n8n’s execution path prevents unintended fan-out in your version/config, this wiring risks posting to **all** webhook nodes. In most n8n setups, a node runs once per incoming item, then sends its output to *every* connected downstream node. If you intend “only one Slack channel per status”, connect each Switch output directly to its corresponding Slack node (or add another Switch/IF after building the payload).

#### Node: Slack — FAIL Alert (Engineering Channel)
- **Type / role:** `HTTP Request` — posts payload to Slack Incoming Webhook for FAIL.
- **Configuration choices:**
  - URL: `YOUR_SLACK_FAIL_WEBHOOK_URL` (placeholder)
  - Method: POST
  - Body: `={{ JSON.stringify($json.slackPayload) }}`
  - Body type: JSON
- **Connections:** Input from **Build Slack Report**; output to **Log Result to Google Sheets**.
- **Edge cases / failures:**
  - Invalid webhook URL → 404/410
  - Slack rejects payload if malformed or too large → 400
- **Version notes:** TypeVersion 4.2.

#### Node: Slack — WARNING Alert (Review Channel)
- **Type / role:** `HTTP Request` — posts to WARNING channel webhook.
- **Configuration:** Same pattern as FAIL, with `YOUR_SLACK_WARNING_WEBHOOK_URL`.
- **Connections:** Input from **Build Slack Report**; output to **Log Result to Google Sheets**.
- **Edge cases:** Same as FAIL.
- **Version notes:** TypeVersion 4.2.

#### Node: Slack — PASS Notice (Docs Channel)
- **Type / role:** `HTTP Request` — posts to PASS channel webhook.
- **Configuration:** Same pattern as FAIL, with `YOUR_SLACK_PASS_WEBHOOK_URL`.
- **Connections:** Input from **Build Slack Report**; output to **Log Result to Google Sheets**.
- **Edge cases:** Same as FAIL.
- **Version notes:** TypeVersion 4.2.

#### Node: Log Result to Google Sheets
- **Type / role:** `Google Sheets` — appends execution result for auditing/tracking.
- **Configuration choices:**
  - Operation: **append**
  - Document: `YOUR_GOOGLE_SHEET_ID` (placeholder)
  - Sheet: `Sheet1`
  - Mapping: `autoMapInputData` (implicitly writes columns matching input keys)
  - Auth: `serviceAccount`
- **Connections:** Inputs from all three Slack nodes.
- **Edge cases / failures:**
  - Service account not shared on the sheet → permission error
  - Auto-mapping may create inconsistent columns if input schema changes
  - Large nested objects (like `slackPayload.blocks`) may not fit well into sheet cells; consider flattening to selected fields (overallScore, status, summary, top gaps).
- **Version notes:** TypeVersion 4.5.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking 'Execute workflow' | Manual Trigger | Manual entry point | — | Set Document Input |  |
| Set Document Input | Set | Define doc URL + metadata | When clicking 'Execute workflow' | Fetch Document Content | ## 1. Fetch & Prepare Documentation; https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest/ |
| Fetch Document Content | HTTP Request | Download document text | Set Document Input | Prepare Document for AI | ## 1. Fetch & Prepare Documentation; https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest/ |
| Prepare Document for AI | Code | Clean/normalize/truncate + heuristics | Fetch Document Content | Score Documentation with AI | ## 1. Fetch & Prepare Documentation; https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest/ |
| Score Documentation with AI | HTTP Request | Call OpenAI GPT‑4o for scoring | Prepare Document for AI | Parse AI Compliance Report | ## 2. Score Documentation with AI; https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest/ |
| Parse AI Compliance Report | Code | Parse JSON, compute weighted score, set status | Score Documentation with AI | Route by Compliance Status | ## 3. Parse Results & Route by Status; https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.switch/ |
| Route by Compliance Status | Switch | Route FAIL/WARNING/PASS | Parse AI Compliance Report | Build Slack Report (x3 outputs) | ## 3. Parse Results & Route by Status; https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.switch/ |
| Build Slack Report | Code | Build Slack Block Kit payload | Route by Compliance Status | Slack — FAIL Alert; Slack — WARNING Alert; Slack — PASS Notice | ## 4. Send Slack Compliance Report; https://api.slack.com/messaging/webhooks |
| Slack — FAIL Alert (Engineering Channel) | HTTP Request | Post FAIL message to Slack | Build Slack Report | Log Result to Google Sheets | ## 4. Send Slack Compliance Report; https://api.slack.com/messaging/webhooks |
| Slack — WARNING Alert (Review Channel) | HTTP Request | Post WARNING message to Slack | Build Slack Report | Log Result to Google Sheets | ## 4. Send Slack Compliance Report; https://api.slack.com/messaging/webhooks |
| Slack — PASS Notice (Docs Channel) | HTTP Request | Post PASS message to Slack | Build Slack Report | Log Result to Google Sheets | ## 4. Send Slack Compliance Report; https://api.slack.com/messaging/webhooks |
| Log Result to Google Sheets | Google Sheets | Append audit log row | Slack — FAIL Alert; Slack — WARNING Alert; Slack — PASS Notice | — |  |
| Sticky Note | Sticky Note | General description & requirements | — | — | ### This n8n template demonstrates how to use AI… Discord: https://discord.com/invite/XPKeKXeB7d ; Forum: https://community.n8n.io/ |
| Sticky Note2 | Sticky Note | Block label: Fetch & Prepare | — | — | ## 1. Fetch & Prepare Documentation; https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest/ |
| Sticky Note3 | Sticky Note | Block label: AI scoring | — | — | ## 2. Score Documentation with AI; https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest/ |
| Sticky Note4 | Sticky Note | Block label: Parse & route | — | — | ## 3. Parse Results & Route by Status; https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.switch/ |
| Sticky Note5 | Sticky Note | Block label: Slack reporting | — | — | ## 4. Send Slack Compliance Report; https://api.slack.com/messaging/webhooks |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
   - Name: *Assess Technical Documentation Compliance with AI and Trigger Slack Alerts* (or your preferred title).
   - Keep workflow inactive until credentials and URLs are configured.

2) **Add “Manual Trigger”**
   - Node: **Manual Trigger**
   - Name: `When clicking 'Execute workflow'`

3) **Add “Set” node for document input**
   - Node: **Set**
   - Name: `Set Document Input`
   - Add fields:
     - `doc_url` (string): your documentation URL
     - `doc_title` (string): e.g., `API Endpoint Reference v2.1`
     - `doc_type` (string): e.g., `API Reference`
     - `review_date` (string expression): `{{ new Date().toISOString().split('T')[0] }}`
   - Connect: Manual Trigger → Set Document Input

4) **Add HTTP Request to fetch the document**
   - Node: **HTTP Request**
   - Name: `Fetch Document Content`
   - Method: GET (default)
   - URL: `{{ $json.doc_url }}`
   - Response: set **Response Format = Text**
   - Connect: Set Document Input → Fetch Document Content

5) **Add Code node to clean/prepare content**
   - Node: **Code**
   - Name: `Prepare Document for AI`
   - Mode: “Run once for each item”
   - Paste logic equivalent to:
     - read fetched text (`$input.item.json.data`)
     - strip HTML, normalize whitespace, truncate to ~6000 chars
     - compute `word_count` and `pre_analysis` flags
     - output `clean_content` plus doc metadata
   - Connect: Fetch Document Content → Prepare Document for AI

6) **Add HTTP Request to call OpenAI (Chat Completions)**
   - Node: **HTTP Request**
   - Name: `Score Documentation with AI`
   - Method: POST
   - URL: `https://api.openai.com/v1/chat/completions`
   - Authentication: **Predefined Credential Type**
     - Credential type: **OpenAI API**
     - Configure credential with your OpenAI API key
   - Body type: JSON
   - JSON body:
     - `model: gpt-4o`
     - `temperature: 0.2`
     - `messages`: system prompt requiring strict JSON + user content embedding `doc_title`, `doc_type`, `word_count`, `clean_content`
   - Connect: Prepare Document for AI → Score Documentation with AI

7) **Add Code node to parse AI output and compute status**
   - Node: **Code**
   - Name: `Parse AI Compliance Report`
   - Logic to implement:
     - read `choices[0].message.content`
     - remove code fences if present, `JSON.parse`
     - fallback on parse error
     - compute weighted `overallScore` (25/25/20/30)
     - set `complianceStatus` by thresholds (<60 FAIL, <80 WARNING, else PASS)
     - flatten gaps and compute `gapCount`
     - generate `reportId`
   - Connect: Score Documentation with AI → Parse AI Compliance Report

8) **Add Switch node to route by status**
   - Node: **Switch**
   - Name: `Route by Compliance Status`
   - Add 3 rules (string equals):
     - `{{ $json.complianceStatus }}` == `FAIL` (rename output to FAIL)
     - == `WARNING` (rename output to WARNING)
     - == `PASS` (rename output to PASS)
   - Connect: Parse AI Compliance Report → Route by Compliance Status

9) **Add Code node to build Slack payload**
   - Node: **Code**
   - Name: `Build Slack Report`
   - Build:
     - `slackPayload.text`
     - `slackPayload.blocks` (header, fields, dimension bars, summary, gaps, context)
   - Connect: Route by Compliance Status → Build Slack Report  
   - **Recommended wiring to avoid multi-posting:** create **three separate connections** from each Switch output to its matching Slack node (and optionally run `Build Slack Report` before the Switch, or create 3 copies of the Slack payload node). If you keep a single Build node, ensure only the intended Slack webhook receives the item.

10) **Add three Slack webhook HTTP Request nodes**
   - Node: **HTTP Request** (three nodes)
   - Names:
     - `Slack — FAIL Alert (Engineering Channel)`
     - `Slack — WARNING Alert (Review Channel)`
     - `Slack — PASS Notice (Docs Channel)`
   - Each:
     - Method: POST
     - URL: your Slack incoming webhook URL for that channel
     - Body type: JSON
     - Body: `{{ JSON.stringify($json.slackPayload) }}`
   - Connect:
     - FAIL route → FAIL Slack node
     - WARNING route → WARNING Slack node
     - PASS route → PASS Slack node

11) **Add Google Sheets logging**
   - Node: **Google Sheets**
   - Name: `Log Result to Google Sheets`
   - Operation: **Append**
   - Authentication: **Service Account**
     - Configure Google credentials and share the Sheet with the service account email
   - Document ID: your sheet ID
   - Sheet name: `Sheet1` (or your choice)
   - Mapping: Auto-map (or explicitly map a flattened schema)
   - Connect: each Slack node → Log Result to Google Sheets

12) **Test**
   - Execute manually.
   - Validate:
     - document fetch returns the expected text field
     - OpenAI returns parseable JSON
     - only one Slack channel receives the message
     - Google Sheets receives a reasonable row (consider flattening to avoid huge nested JSON)

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “This n8n template demonstrates how to use AI to automatically review technical documentation against predefined compliance standards — and alert your team via Slack.” | Workflow sticky note (overview) |
| Requirements: OpenAI/Anthropic API key, Slack incoming webhooks, docs accessible via URL or text | Workflow sticky note (overview) |
| Discord community invite | https://discord.com/invite/XPKeKXeB7d |
| n8n community forum | https://community.n8n.io/ |
| HTTP Request node reference | https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest/ |
| Switch node reference | https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.switch/ |
| Slack incoming webhooks | https://api.slack.com/messaging/webhooks |