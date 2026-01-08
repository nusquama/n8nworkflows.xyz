Generate LinkedIn agency content with GPT‑4o, Claude 4.5 and Gemini

https://n8nworkflows.xyz/workflows/generate-linkedin-agency-content-with-gpt-4o--claude-4-5-and-gemini-12302


# Generate LinkedIn agency content with GPT‑4o, Claude 4.5 and Gemini

## 1. Workflow Overview

**Title:** Generate LinkedIn agency content with GPT‑4o, Claude 4.5 and Gemini  
**Purpose:** Maintain a consistent, high-authority LinkedIn presence by automatically generating content from two sources—(A) new portfolio/case-study assets in Google Drive and (B) a strategic topic bank in Google Sheets—then enforcing **human approval** before either logging (strategy track) or publishing (portfolio track).

### 1.1 Logical blocks (high-level)
1. **Intake & Routing (Schedule → Portfolio scan → Branching)**  
   Runs twice weekly, compiles “already published” topics, scans Drive for new portfolio files, and routes either to **Portfolio path** (if new assets exist) or **Strategy path** (if not).
2. **Portfolio Extraction & Post Creation (Drive → Sheets lookup → 2 model drafts)**  
   Downloads the selected PDF, looks up a matching summary in Sheets, generates two draft post options (GPT-4o + Claude).
3. **Portfolio Editorial Gatekeeper → LinkedIn Publishing**  
   Sends approval email, initializes LinkedIn document upload, merges approved text with the binary PDF, uploads it, publishes the LinkedIn post, then archives it to Sheets.
4. **Strategy Engineering & Multi-model Creative (Sheets topic → 3 model drafts)**  
   Pulls the oldest queued topic, generates a strategic brief, produces three options (GPT/Claude/Gemini), cleans formatting, then emails for approval.
5. **Topic Bank Delivery (log + update)**  
   Logs the approved strategic post to a history sheet and updates the topic bank row to “Posted” (intended).
6. **System Error Handling (Error Trigger → Gmail alert)**  
   Any node failure triggers an alert email naming the failing node and error message.

---

## 2. Block-by-Block Analysis

### Block 0 — Global Documentation / Notes (Sticky Notes)
**Overview:** Provides intent, setup checklist, and phase labeling for maintainers.  
**Nodes involved:** Sticky Note, Sticky Note1–Sticky Note11 (all are `stickyNote` nodes).  
**Node details:** Sticky notes do not execute; they annotate phases, setup steps, and requirements. See Summary Table for mapping.

---

### Block 1 — Intake & Routing (PHASE 1: INTAKE & ROUTING)
**Overview:** Twice weekly trigger; pulls publishing history and scans Drive for new portfolio assets; routes execution based on whether new assets exist.

**Nodes involved**
- **Bi-Weekly Content Trigger**
- **Fetch Published History**
- **Compile History Array**
- **Scan Portfolio Assets**
- **Identify New Portfolio Items**
- **New Portfolio Found?**

#### Node: Bi-Weekly Content Trigger
- **Type/role:** `scheduleTrigger` — workflow entry point
- **Config:** Cron `0 9 * * 2,4` (Tuesday + Thursday at 09:00). Note says “Tuesday and Friday” but cron is Tue/Thu.
- **Output:** Starts the chain.
- **Edge cases:** Timezone depends on n8n instance; note says Europe/Madrid but schedule node itself doesn’t show timezone setting here.

#### Node: Fetch Published History
- **Type/role:** `googleSheets` — fetch historical posts/topics already published
- **Config:** Uses placeholder Spreadsheet ID/Name (`YOUR_SPREADSHEET_ID`, `YOUR_SPREADSHEET_NAME`). No operation specified in JSON excerpt (defaults vary by node version); it’s being used as a “get many rows” source.
- **Credentials:** Google Sheets OAuth2.
- **Edge cases:** Wrong sheet/doc IDs; missing headers; permission issues.

#### Node: Compile History Array
- **Type/role:** `aggregate` — aggregates all incoming rows into a single array
- **Config:** `aggregateAllItemData`
- **Output:** Single item with `json.data` array used by later code.
- **Edge cases:** Empty history results in `data: []` which must be handled downstream.

#### Node: Scan Portfolio Assets
- **Type/role:** `googleDrive` — list files in a folder (portfolio assets)
- **Config:** Folder ID `YOUR_PORTFOLIO_FOLDER_ID`, return all files, exclude trashed. `executeOnce: true` (node will not re-run in same execution context).
- **Credentials:** Google Drive OAuth2.
- **Output:** One item per file; includes `json.name`, `json.id`, etc.
- **Edge cases:** Folder permissions; large folder may hit API limits; file naming conventions matter later.

#### Node: Identify New Portfolio Items
- **Type/role:** `code` — diff Drive files vs published topics
- **Key logic:**
  - Pulls aggregated history via `$items("Compile History Array")` and expects `aggregatedItems[0].json.data`.
  - Extracts `post.Topic` values; trims; filters blanks.
  - For each Drive file, strips extension and compares to published topics list.
- **Output:** Only unposted Drive files.
- **Edge cases/failures:**
  - If `Fetch Published History` returns unexpected structure, `aggregatedItems[0].json.data.map` can throw.
  - Matching is exact and case-sensitive; “Topic” must exactly equal Drive filename (without extension).

#### Node: New Portfolio Found?
- **Type/role:** `if` — route to Portfolio path or Strategy path
- **Condition:** `{{$json}}` is “not empty” (object notEmpty)
- **True output:** Portfolio path (Select Oldest New Item)
- **False output:** Strategy path (Fetch Content Queue)
- **Edge cases:** If code node returns an empty list, IF receives no items; routing depends on n8n behavior. Current condition checks each item; with zero items, “false branch” may run with no items (often results in no execution). Consider using “Always Output Data” upstream (it is enabled in code node) which helps.

---

### Block 2 — Portfolio Extraction (PHASE 2: PORTFOLIO EXTRACTION)
**Overview:** Selects one new portfolio file and downloads it for later LinkedIn upload and summary lookup.

**Nodes involved**
- **Select Oldest New Item**
- **Retrieve Case Study File**
- **Lookup Project Summary**

#### Node: Select Oldest New Item
- **Type/role:** `limit` — pick one file to process
- **Config:** Default limit (implicitly 1).
- **Edge cases:** Despite name “Oldest”, it does not sort; it just takes first item from upstream order (Drive listing order can be inconsistent).

#### Node: Retrieve Case Study File
- **Type/role:** `googleDrive` download — obtains binary PDF/doc
- **Config:** `fileId: {{$json.id}}`, operation `download`.
- **Output:** Binary data (likely in `binary.data`), plus metadata (`json.name`, etc.).
- **Edge cases:** Large files may exceed memory/time; missing permissions.

#### Node: Lookup Project Summary
- **Type/role:** `googleSheets` lookup — gets case study summary matching the file name
- **Config:** Filter: lookup column `Workflow`, lookup value = Drive file name without extension: `{{$json.name.replace(/\.[^/.]+$/, "")}}`
- **Output:** A row containing at least `Summary`.
- **Edge cases:** If no match, `Summary` is missing and drafting agents may produce poor output or fail.

---

### Block 3 — Portfolio Post Creation (PHASE 3: PORTFOLIO POST CREATION)
**Overview:** Produces two post options (GPT-4o + Claude) from the sheet summary, then cleans/labels outputs into a package for email approval.

**Nodes involved**
- **GPT-4o (Creative Logic)** (LLM connector)
- **Claude 4.5 Sonnet (Analytical Logic)** (LLM connector)
- **Draft Portfolio Post (GPT-4o)**
- **Draft Portfolio Post (Claude 4.5)**
- **Combine Draft Options**
- **Format Portfolio Drafts**
- **Prepare Draft Package**

#### Node: GPT-4o (Creative Logic)
- **Type/role:** `lmChatOpenAi` — language model for the GPT portfolio agent
- **Config:** `gpt-4o`, temperature 0.7.
- **Connection:** Provides `ai_languageModel` input to **Draft Portfolio Post (GPT-4o)**.
- **Edge cases:** OpenAI auth, model availability, quota, latency.

#### Node: Draft Portfolio Post (GPT-4o)
- **Type/role:** `langchain.agent` — generates punchy “curiosity gap” post
- **Input prompt:** Uses `{{$json.Summary}}` from Lookup Project Summary.
- **System constraints:** Max 80 words, no emojis/styling/em-dashes; must end with specific CTA.
- **Output:** `json.output` (agent output).
- **Edge cases:** Output can violate constraints; later cleanup partially fixes dashes/quotes, not word count.

#### Node: Claude 4.5 Sonnet (Analytical Logic)
- **Type/role:** `lmChatAnthropic` — language model connector for Claude portfolio agent
- **Model:** `claude-sonnet-4-5-20250929`
- **Connection:** Provides `ai_languageModel` to **Draft Portfolio Post (Claude 4.5)**.
- **Edge cases:** Anthropic auth/quota; model string must exist in your Anthropic account/region.

#### Node: Draft Portfolio Post (Claude 4.5)
- **Type/role:** `langchain.agent` — PAS framework teaser post
- **Input prompt:** Uses `{{$json.Summary}}`
- **Constraints:** Under 120 words, no emojis/bolding/hashtags; ends with same “Swipe…” CTA.
- **Edge cases:** Similar constraint drift.

#### Node: Combine Draft Options
- **Type/role:** `merge` — merges the two drafts into one stream
- **Config:** Default merge behavior (in n8n v3 merge node, default is often “append”).
- **Output:** Two items (one per model), used by formatter.
- **Edge cases:** If one draft fails, merge behavior may drop items depending on mode.

#### Node: Format Portfolio Drafts
- **Type/role:** `code` — sanitize + label each option
- **Key logic:**
  - Reads all incoming items (`$input.all()`).
  - Cleans: strips surrounding quotes, replaces em-dashes with `-`, ensures CTA gets newline separation.
  - Determines `agent` name via `pairedItem?.node?.name` (falls back to fixed mapping by index).
  - Produces items: `{ agent, option, content }`.
- **Edge cases:**
  - `pairedItem` metadata may not exist across merges; fallback handles typical order only.
  - Dash replacement may violate “no em-dash” but turns it into hyphen.

#### Node: Prepare Draft Package
- **Type/role:** `aggregate` — package both formatted options into `json.data` array
- **Used by:** Approval email and later publish node for selecting which option to post.
- **Edge cases:** If only one option exists, email template references `data[1]` and will break.

---

### Block 4 — Portfolio Editorial Gatekeeper (PHASE 4) → Posting (PHASE 5)
**Overview:** Sends email for approval, routes based on approve/decline, initializes LinkedIn document upload, uploads the PDF binary, publishes the post, and archives the publication.

**Nodes involved**
- **Portfolio Post Approval Gate**
- **Switch**
- **Halt: Review Required**
- **LI API: Init Image/Doc Upload**
- **Sync File & Approved Text**
- **API Throttle Delay**
- **LI API: Transfer Binary Data**
- **LI API: Publish Live Post**
- **Archive Portfolio Publication**

#### Node: Portfolio Post Approval Gate
- **Type/role:** `gmail` (`sendAndWait`) — human approval with dropdown
- **Form field:** `approved_option` ∈ {Option A, Option B, Decline all}
- **Email body:** HTML; references:
  - `$json.data[0].content`, `$json.data[1].content`
  - `$node["Lookup Project Summary"].json.Summary.substring(0,150)`
- **Output:** Response stored under `json.query.response` (n8n wait mechanics) and also mapped to `$json.data.approved_option` via node behavior.
- **Edge cases:**
  - If `data[1]` missing → template error.
  - Gmail “sendAndWait” requires n8n public webhook URL reachable.
  - Long drafts may truncate in email clients.

#### Node: Switch (portfolio)
- **Type/role:** `switch` — approve vs decline
- **Rules:**
  - Approve if `{{$json.data.approved_option}}` matches regex `Option A|Option B`
  - Decline if equals `Decline all`
- **Outputs:** Approve → LinkedIn init upload; Decline → Halt node
- **Edge cases:** If response payload differs (e.g., lowercase), may not match.

#### Node: Halt: Review Required
- **Type/role:** `noOp` — stops portfolio publish path on decline
- **Edge cases:** None; simply ends branch.

#### Node: LI API: Init Image/Doc Upload
- **Type/role:** `httpRequest` — LinkedIn REST initialize document upload
- **Endpoint:** `POST https://api.linkedin.com/rest/documents?action=initializeUpload`
- **Headers:** `LinkedIn-Version: 202502`, `X-Restli-Protocol-Version: 2.0.0`, `Content-Type: application/json`
- **Body:** owner is organization URN `urn:li:organization:110151891`
- **Auth:** LinkedIn OAuth2 credential
- **Output:** Expects `json.value.uploadUrl` and `json.value.document` later.
- **Edge cases:** LinkedIn API permissions (w_member_social/w_organization_social equivalents), org admin rights, version headers must match LinkedIn requirements.

#### Node: Sync File & Approved Text
- **Type/role:** `merge` (combine by position) — pairs:
  - Input 0: binary file from Retrieve Case Study File
  - Input 1: init upload response from LinkedIn
- **Config:** `combineByPosition`
- **Output:** Single item containing both datasets (binary + uploadUrl/document id).
- **Edge cases:** If either side has multiple items, pairing may mismatch.

#### Node: API Throttle Delay
- **Type/role:** `wait` — throttle before PUT upload
- **Config:** No duration shown (defaults to “wait for webhook” or “fixed time” depending on settings). Here parameters are empty, so verify node mode in UI.
- **Edge cases:** Misconfigured wait can stall indefinitely.

#### Node: LI API: Transfer Binary Data
- **Type/role:** `httpRequest` — PUT binary PDF to LinkedIn upload URL
- **URL:** `{{$json.value.uploadUrl}}`
- **Content type:** `binaryData`, input field name `data`, header `Content-Type: application/pdf`
- **Response format:** text
- **Output:** Always outputs data enabled.
- **Edge cases:** LinkedIn upload URL expires; binary field name must match where Drive download stores it (often `binary.data`). If n8n binary property differs, upload will be empty.

#### Node: LI API: Publish Live Post
- **Type/role:** `httpRequest` — publish LinkedIn post with document
- **Endpoint:** `POST https://api.linkedin.com/rest/posts`
- **Headers:** `LinkedIn-Version: 202511`, `X-Restli-Protocol-Version: 2.0.0`
- **Body logic:** `commentary` is selected from `Prepare Draft Package.json.data[...]` based on whether approval response includes “A” (else uses option 1).
  - Uses `JSON.stringify(...)` to safely quote commentary.
- **Media:** document id from init upload: `{{$node["LI API: Init Image/Doc Upload"].json.value.document}}`
- **Edge cases:**
  - If response is “Option B” it contains “B” so it uses index 1 (correct), but if any other text, it defaults to 1.
  - LinkedIn post API requires correct permissions and org authoring rights.

#### Node: Archive Portfolio Publication
- **Type/role:** `googleSheets` append — write posted portfolio entry to history
- **Columns written:**
  - Date = now formatted `dd-MM-yyyy`
  - Type = `Portfolio`
  - Topic = file name without extension from `$('API Throttle Delay').item.json.name...`
  - Status = `Posted`
  - Content = selected approved content from Prepare Draft Package
- **Edge cases:**
  - Reference to `API Throttle Delay` item for `name` assumes the merged item still contains file metadata at that point.
  - Sheet schema must match exactly.

---

### Block 5 — Strategy Engineering (PHASE 6) & Creative QC (PHASE 7)
**Overview:** When no new portfolio file exists, the workflow selects the oldest topic from Sheets, builds a strategic brief, generates three model-specific post drafts (GPT/Claude/Gemini), merges and sanitizes them, then prepares an approval email.

**Nodes involved**
- **Fetch Content Queue**
- **Prioritize Oldest Content**
- **Select Single Topic**
- **Generate Agency Strategy**
- **GPT-4o (Content Architect)**, **GPT-4o (Brand Voice Editor)**
- **Claude 4.5 (Content Architect)**, **Claude 4.5 (Brand Voice Editor)**
- **Gemini 2.5-Pro (Content Architect)**, **Gemini 2.5-Pro (Brand Voice Editor)**
- **Strategy: GPT Architect**, **Voice: GPT Editor**
- **Strategy: Claude Architect**, **Voice: Claude Editor**
- **Strategy: Gemini Architect**, **Voice: Gemini Editor**
- **Combine Cleaned Options**
- **Draft Review Email**
- **Human Editor Approval**
- **Approved?**
- **Discard Drafts**
- **Log Approved Post**
- **Update Topic Bank: Posted Status**

#### Node: Fetch Content Queue
- **Type/role:** `googleSheets` — pulls topic bank rows (queue)
- **Config:** placeholder sheet/doc. Operation not shown; used as list source.
- **Edge cases:** Must include columns used later: `Topic`, `Pain Point`, `Date Posted`, `row_number`.

#### Node: Prioritize Oldest Content
- **Type/role:** `sort` — sorts rows by “Date Posted”
- **Config:** Sort ascending by `Date Posted`.
- **Edge cases:** Date Posted must be consistently formatted; string sorting can misorder dates unless ISO format.

#### Node: Select Single Topic
- **Type/role:** `limit` — takes first row after sorting (oldest)
- **Output:** Single topic row.
- **Edge cases:** Empty queue → no downstream execution.

#### Node: Generate Agency Strategy
- **Type/role:** `code` — builds a structured brief for model architects
- **Output JSON:**
  - `selected_title` from `Topic`
  - `strategic_brief.what_changed` from `Pain Point` (note: sheet header mismatch: sticky note says “Pain Points”; code expects “Pain Point”)
  - constant `the_shift` and `why_it_matters`
  - passes `row_number`
- **Edge cases:** Header mismatch breaks `Pain Point` extraction.

#### Nodes: Strategy Architects (GPT/Claude/Gemini)
- **Type/role:** `langchain.agent` — each produces a different “strategy” output in plain text
- **Inputs:** use `selected_title` and `strategic_brief` fields
- **LLM connections:**
  - Strategy: GPT Architect uses **GPT-4o (Content Architect)** via `ai_languageModel`
  - Strategy: Claude Architect uses **Claude 4.5 (Content Architect)**
  - Strategy: Gemini Architect uses **Gemini 2.5-Pro (Content Architect)**
- **Edge cases:** Prompt formatting rules can be violated; output cleaning happens later but is oriented toward markdown/bullets, not deep structure.

#### Nodes: Voice Editors (GPT/Claude/Gemini)
- **Type/role:** `langchain.agent` — convert strategy into final LinkedIn post drafts
- **LLM connections:**
  - Voice: GPT Editor uses **GPT-4o (Brand Voice Editor)**
  - Voice: Claude Editor uses **Claude 4.5 (Brand Voice Editor)**
  - Voice: Gemini Editor uses **Gemini 2.5-Pro (Brand Voice Editor)**
- **Constraints:** bullet rules (•), no markdown, word limits (Claude/Gemini 200 words).
- **Edge cases:** Bullet spacing requirements are strict; code cleanup later attempts to repair run-on bullets and markdown remnants.

#### Node: Combine Cleaned Options
- **Type/role:** `merge` — combines three branches into one stream
- **Config:** `numberInputs: 3`
- **Edge cases:** If any one branch fails, downstream formatter assumptions (3 items) break.

#### Node: Draft Review Email
- **Type/role:** `code` — sanitize and package the 3 drafts into a single object `{ data: { ... } }`
- **Key logic:**
  - Reads `$input.all()` and maps item 0/1/2 to GPT/Claude/Gemini respectively (assumes stable merge ordering).
  - Pulls topic title and `row_number` from `$("Select Single Topic").first().json`.
  - `cleanContent()` removes markdown artifacts, normalizes dashes/hyphens, fixes bullet newlines.
- **Output:** `{ data: { subject, strategy, row_number, option_gpt, option_claude, option_gemini } }`
- **Edge cases:** If merge ordering changes, options get mislabeled.

#### Node: Human Editor Approval
- **Type/role:** `gmail` (`sendAndWait`) — presents three options for approval
- **Dropdown:** Option A/B/C or Decline all
- **Email template:** HTML with three sections and metadata (topic/strategy/source row).
- **Edge cases:** Requires reachable n8n webhook; responses must match switch regex.

#### Node: Approved?
- **Type/role:** `switch` — routes approval vs decline
- **Rules:** Approve if approved_option matches `Option A|Option B|Option C`; Decline if equals `Decline all`.
- **Outputs:** Approve → Log Approved Post; Decline → Discard Drafts

#### Node: Discard Drafts
- **Type/role:** `noOp` — ends execution for declined drafts

#### Node: Log Approved Post
- **Type/role:** `googleSheets` append — logs the approved strategy post to history
- **Selects content:** Based on `Human Editor Approval` response:
  - if includes “a” → GPT
  - else if includes “b” → Claude
  - else → Gemini
- **Columns:** Date, Type “News/Trend”, Topic, Status “Draft”, Content
- **Edge cases:** Status is “Draft” even though approved—may be intentional, but naming suggests mismatch.

#### Node: Update Topic Bank: Posted Status
- **Type/role:** `googleSheets` update — intended to mark the chosen topic as posted
- **Current config issues:**
  - `sheetName` and `documentId` values are set with leading `=` and placeholders (`=YOUR_SPREADSHEET_NAME`, `=YOUR_SPREADSHEET_ID`) which may cause resolution problems.
  - `columns.value` is empty and schema is empty; no update mapping is defined.
- **Edge cases:** As-is, this node likely does nothing or errors. It also needs a row identifier (row_number) mapping to update the correct row.

---

### Block 6 — System Error Handling (SYSTEM: Error Handling)
**Overview:** Sends an immediate alert email when any node fails.

**Nodes involved**
- **Error Trigger**
- **Send a message**

#### Node: Error Trigger
- **Type/role:** `errorTrigger` — separate entry point for failures
- **Output:** Provides failing `node.name` and `execution.error.message`.

#### Node: Send a message
- **Type/role:** `gmail` send — emails `user@example.com`
- **Subject:** `⚠️ Workflow Error: LinkedIn Content Engine`
- **Body:** Includes node name + reason from Error Trigger
- **Edge cases:** If Gmail credential fails, you will not receive alerts.

---

## 3. Summary Table (all nodes)

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Bi-Weekly Content Trigger | scheduleTrigger | Entry point (twice weekly) | — | Fetch Published History | ## PHASE 1: INTAKE & ROUTING |
| Fetch Published History | googleSheets | Load LinkedIn history (topics posted) | Bi-Weekly Content Trigger | Compile History Array | ## PHASE 1: INTAKE & ROUTING |
| Compile History Array | aggregate | Aggregate history rows into array | Fetch Published History | Scan Portfolio Assets | ## PHASE 1: INTAKE & ROUTING |
| Scan Portfolio Assets | googleDrive | List files in portfolio folder | Compile History Array | Identify New Portfolio Items | ## PHASE 1: INTAKE & ROUTING |
| Identify New Portfolio Items | code | Diff Drive files vs history topics | Scan Portfolio Assets | New Portfolio Found? | ## PHASE 1: INTAKE & ROUTING |
| New Portfolio Found? | if | Route: portfolio vs strategy | Identify New Portfolio Items | Select Oldest New Item; Fetch Content Queue | ## PHASE 1: INTAKE & ROUTING |
| Select Oldest New Item | limit | Pick single portfolio asset | New Portfolio Found? (true) | Retrieve Case Study File | ## PHASE 2: PORTFOLIO EXTRACTION (TRUE PATH) |
| Retrieve Case Study File | googleDrive | Download PDF binary | Select Oldest New Item | Sync File & Approved Text; Lookup Project Summary | ## PHASE 2: PORTFOLIO EXTRACTION (TRUE PATH) |
| Lookup Project Summary | googleSheets | Lookup summary by file name | Retrieve Case Study File | Draft Portfolio Post (GPT-4o); Draft Portfolio Post (Claude 4.5) | ## PHASE 3: PORTFOLIO POST CREATION |
| GPT-4o (Creative Logic) | lmChatOpenAi | LLM connector for GPT portfolio draft | — | Draft Portfolio Post (GPT-4o) (ai_languageModel) | ## PHASE 3: PORTFOLIO POST CREATION |
| Draft Portfolio Post (GPT-4o) | langchain.agent | Draft option A for portfolio | Lookup Project Summary; GPT-4o (Creative Logic) | Combine Draft Options | ## PHASE 3: PORTFOLIO POST CREATION |
| Claude 4.5 Sonnet (Analytical Logic) | lmChatAnthropic | LLM connector for Claude portfolio draft | — | Draft Portfolio Post (Claude 4.5) (ai_languageModel) | ## PHASE 3: PORTFOLIO POST CREATION |
| Draft Portfolio Post (Claude 4.5) | langchain.agent | Draft option B for portfolio | Lookup Project Summary; Claude 4.5 Sonnet (Analytical Logic) | Combine Draft Options | ## PHASE 3: PORTFOLIO POST CREATION |
| Combine Draft Options | merge | Combine 2 portfolio drafts | Draft Portfolio Post (GPT-4o); Draft Portfolio Post (Claude 4.5) | Format Portfolio Drafts | ## PHASE 3: PORTFOLIO POST CREATION |
| Format Portfolio Drafts | code | Sanitize + label each portfolio option | Combine Draft Options | Prepare Draft Package | ## PHASE 3: PORTFOLIO POST CREATION |
| Prepare Draft Package | aggregate | Package portfolio options into array | Format Portfolio Drafts | Portfolio Post Approval Gate | ## PHASE 4: PORTFOLIO EDITORIAL GATEKEEPER |
| Portfolio Post Approval Gate | gmail (sendAndWait) | Human approval for portfolio drafts | Prepare Draft Package | Switch | ## PHASE 4: PORTFOLIO EDITORIAL GATEKEEPER |
| Switch | switch | Route approve/decline (portfolio) | Portfolio Post Approval Gate | LI API: Init Image/Doc Upload; Halt: Review Required | ## PHASE 4: PORTFOLIO EDITORIAL GATEKEEPER |
| Halt: Review Required | noOp | Stop on decline (portfolio) | Switch (Decline) | — | ## PHASE 4: PORTFOLIO EDITORIAL GATEKEEPER |
| LI API: Init Image/Doc Upload | httpRequest | Initialize LinkedIn document upload | Switch (Approve) | Sync File & Approved Text | ## PHASE 5: PORTFOLIO POSTING & DELIVERY |
| Sync File & Approved Text | merge | Combine binary PDF + upload info | Retrieve Case Study File; LI API: Init Image/Doc Upload | API Throttle Delay | ## PHASE 5: PORTFOLIO POSTING & DELIVERY |
| API Throttle Delay | wait | Delay before PUT upload | Sync File & Approved Text | LI API: Transfer Binary Data | ## PHASE 5: PORTFOLIO POSTING & DELIVERY |
| LI API: Transfer Binary Data | httpRequest | PUT PDF binary to LinkedIn upload URL | API Throttle Delay | LI API: Publish Live Post | ## PHASE 5: PORTFOLIO POSTING & DELIVERY |
| LI API: Publish Live Post | httpRequest | Publish LinkedIn post with doc | LI API: Transfer Binary Data | Archive Portfolio Publication | ## PHASE 5: PORTFOLIO POSTING & DELIVERY |
| Archive Portfolio Publication | googleSheets | Append posted portfolio entry to history | LI API: Publish Live Post | — | ## PHASE 5: PORTFOLIO POSTING & DELIVERY |
| Fetch Content Queue | googleSheets | Load topic bank queue | New Portfolio Found? (false) | Prioritize Oldest Content | ## PHASE 6: STRATEGY ENGINEERING (FALSE PATH) |
| Prioritize Oldest Content | sort | Sort by Date Posted | Fetch Content Queue | Select Single Topic | ## PHASE 6: STRATEGY ENGINEERING (FALSE PATH) |
| Select Single Topic | limit | Choose oldest topic | Prioritize Oldest Content | Generate Agency Strategy | ## PHASE 6: STRATEGY ENGINEERING (FALSE PATH) |
| Generate Agency Strategy | code | Build structured strategic brief | Select Single Topic | Strategy: GPT Architect; Strategy: Claude Architect; Strategy: Gemini Architect | ## PHASE 6: STRATEGY ENGINEERING (FALSE PATH) |
| GPT-4o (Content Architect) | lmChatOpenAi | LLM connector for GPT strategy | — | Strategy: GPT Architect (ai_languageModel) | ## PHASE 7: MULTI-MODEL CREATIVE & QC |
| Strategy: GPT Architect | langchain.agent | Produce post architecture (GPT) | Generate Agency Strategy; GPT-4o (Content Architect) | Voice: GPT Editor | ## PHASE 7: MULTI-MODEL CREATIVE & QC |
| GPT-4o (Brand Voice Editor) | lmChatOpenAi | LLM connector for GPT voice editor | — | Voice: GPT Editor (ai_languageModel) | ## PHASE 7: MULTI-MODEL CREATIVE & QC |
| Voice: GPT Editor | langchain.agent | Final GPT post draft | Strategy: GPT Architect; GPT-4o (Brand Voice Editor) | Combine Cleaned Options | ## PHASE 7: MULTI-MODEL CREATIVE & QC |
| Claude 4.5 (Content Architect) | lmChatAnthropic | LLM connector for Claude strategy | — | Strategy: Claude Architect (ai_languageModel) | ## PHASE 7: MULTI-MODEL CREATIVE & QC |
| Strategy: Claude Architect | langchain.agent | Strategy analysis (Claude) | Generate Agency Strategy; Claude 4.5 (Content Architect) | Voice: Claude Editor | ## PHASE 7: MULTI-MODEL CREATIVE & QC |
| Claude 4.5 (Brand Voice Editor) | lmChatAnthropic | LLM connector for Claude voice | — | Voice: Claude Editor (ai_languageModel) | ## PHASE 7: MULTI-MODEL CREATIVE & QC |
| Voice: Claude Editor | langchain.agent | Final Claude post draft | Strategy: Claude Architect; Claude 4.5 (Brand Voice Editor) | Combine Cleaned Options | ## PHASE 7: MULTI-MODEL CREATIVE & QC |
| Gemini 2.5-Pro (Content Architect) | lmChatGoogleGemini | LLM connector for Gemini strategy | — | Strategy: Gemini Architect (ai_languageModel) | ## PHASE 7: MULTI-MODEL CREATIVE & QC |
| Strategy: Gemini Architect | langchain.agent | Sales-engineer logic (Gemini) | Generate Agency Strategy; Gemini 2.5-Pro (Content Architect) | Voice: Gemini Editor | ## PHASE 7: MULTI-MODEL CREATIVE & QC |
| Gemini 2.5-Pro (Brand Voice Editor) | lmChatGoogleGemini | LLM connector for Gemini voice | — | Voice: Gemini Editor (ai_languageModel) | ## PHASE 7: MULTI-MODEL CREATIVE & QC |
| Voice: Gemini Editor | langchain.agent | Final Gemini post draft | Strategy: Gemini Architect; Gemini 2.5-Pro (Brand Voice Editor) | Combine Cleaned Options | ## PHASE 7: MULTI-MODEL CREATIVE & QC |
| Combine Cleaned Options | merge | Combine 3 drafts | Voice: GPT Editor; Voice: Claude Editor; Voice: Gemini Editor | Draft Review Email | ## PHASE 7: MULTI-MODEL CREATIVE & QC |
| Draft Review Email | code | Sanitize + package 3 options | Combine Cleaned Options | Human Editor Approval | ## PHASE 8: EDITORIAL GATEKEEPER |
| Human Editor Approval | gmail (sendAndWait) | Human selects A/B/C or decline | Draft Review Email | Approved? | ## PHASE 8: EDITORIAL GATEKEEPER |
| Approved? | switch | Route approve/decline (strategy) | Human Editor Approval | Log Approved Post; Discard Drafts | ## PHASE 8: EDITORIAL GATEKEEPER |
| Discard Drafts | noOp | Stop on decline | Approved? (Decline) | — | ## PHASE 8: EDITORIAL GATEKEEPER |
| Log Approved Post | googleSheets | Append approved strategy post | Approved? (Approve) | Update Topic Bank: Posted Status | ## PHASE 9:  TOPIC BANK DELIVERY |
| Update Topic Bank: Posted Status | googleSheets | Mark topic row as posted (intended) | Log Approved Post | — | ## PHASE 9:  TOPIC BANK DELIVERY |
| Error Trigger | errorTrigger | Entry point for failures | — | Send a message | ## SYSTEM: Error Handling |
| Send a message | gmail | Email failure details | Error Trigger | — | ## SYSTEM: Error Handling |
| Sticky Note | stickyNote | Global overview + requirements | — | — |  |
| Sticky Note1 | stickyNote | Phase label | — | — |  |
| Sticky Note2 | stickyNote | Phase label | — | — |  |
| Sticky Note3 | stickyNote | Phase label | — | — |  |
| Sticky Note4 | stickyNote | Phase label | — | — |  |
| Sticky Note5 | stickyNote | Phase label | — | — |  |
| Sticky Note6 | stickyNote | Phase label | — | — |  |
| Sticky Note7 | stickyNote | Phase label | — | — |  |
| Sticky Note8 | stickyNote | Phase label | — | — |  |
| Sticky Note9 | stickyNote | Phase label | — | — |  |
| Sticky Note10 | stickyNote | Phase label | — | — |  |
| Sticky Note11 | stickyNote | Setup guide | — | — |  |

---

## 4. Reproducing the Workflow from Scratch (step-by-step)

1. **Create the Schedule Trigger**
   1) Add **Schedule Trigger** named *Bi-Weekly Content Trigger*  
   2) Set cron to `0 9 * * 2,4` (adjust if you truly want Tue/Fri).  
   3) (Optional) Set timezone to Europe/Madrid in n8n instance or node if available.

2. **Create “Published History → Drive scan → diff” chain**
   1) Add **Google Sheets** node *Fetch Published History*  
      - Configure Spreadsheet ID + Sheet name containing LinkedIn post history (must include **Topic** column).  
      - Operation: “Get many” / “Read rows” (implementation depends on n8n version).
   2) Add **Aggregate** node *Compile History Array* → mode “Aggregate All Item Data”.
   3) Add **Google Drive** node *Scan Portfolio Assets*  
      - Resource: File/Folder, Search/List files in folder  
      - Folder ID: your portfolio folder  
      - Return All: true; exclude trashed.
   4) Add **Code** node *Identify New Portfolio Items* and paste the provided JS (adapt if your history schema differs).
   5) Add **IF** node *New Portfolio Found?*  
      - Condition: Object “not empty” on `{{$json}}`.

3. **Portfolio path (true branch): select + download + summary lookup**
   1) Add **Limit** node *Select Oldest New Item* (limit 1).
   2) Add **Google Drive** node *Retrieve Case Study File* → Operation: Download, File ID `{{$json.id}}`.
   3) Add **Google Sheets** node *Lookup Project Summary*  
      - Filter lookup: column `Workflow` equals `{{$json.name.replace(/\.[^/.]+$/, "")}}`  
      - Ensure the sheet has at least `Workflow` and `Summary`.

4. **Portfolio drafting (2 options)**
   1) Add **OpenAI Chat Model** node *GPT-4o (Creative Logic)* selecting model `gpt-4o`.
   2) Add **AI Agent** node *Draft Portfolio Post (GPT-4o)*  
      - Set its “Language Model” connection to *GPT-4o (Creative Logic)*  
      - Prompt includes `Project Summary: {{$json.Summary}}` and your system rules (replace “XX” and “YOUR COMPANY NAME”).
   3) Add **Anthropic Chat Model** node *Claude 4.5 Sonnet (Analytical Logic)* selecting `claude-sonnet-4-5-20250929`.
   4) Add **AI Agent** node *Draft Portfolio Post (Claude 4.5)* using that Claude model connection.
   5) Add **Merge** node *Combine Draft Options* to merge both draft outputs.
   6) Add **Code** node *Format Portfolio Drafts* with the provided sanitization/labeling JS.
   7) Add **Aggregate** node *Prepare Draft Package* (aggregate all items).

5. **Portfolio approval + LinkedIn publish**
   1) Add **Gmail** node *Portfolio Post Approval Gate* → Operation: **Send and Wait**  
      - To: your editor email  
      - Custom form dropdown: `approved_option` with options A/B/Decline  
      - HTML body referencing `$json.data[0]` and `$json.data[1]`  
      - Ensure n8n has a public webhook URL configured.
   2) Add **Switch** node named *Switch*  
      - Approve rule: regex match `Option A|Option B` on `{{$json.data.approved_option}}`  
      - Decline rule: equals `Decline all`.
   3) Decline output → **NoOp** node *Halt: Review Required*.
   4) Approve output → **HTTP Request** node *LI API: Init Image/Doc Upload*  
      - Auth: LinkedIn OAuth2 credential  
      - POST `https://api.linkedin.com/rest/documents?action=initializeUpload`  
      - JSON body with `owner` as your org URN  
      - Headers: `LinkedIn-Version`, `X-Restli-Protocol-Version`, `Content-Type: application/json`.
   5) Add **Merge** node *Sync File & Approved Text*  
      - Mode: combine by position  
      - Input 0 from *Retrieve Case Study File*  
      - Input 1 from *LI API: Init…*
   6) Add **Wait** node *API Throttle Delay*  
      - Configure a fixed delay (e.g., 2–5 seconds) to avoid upload throttling.
   7) Add **HTTP Request** node *LI API: Transfer Binary Data*  
      - PUT to `{{$json.value.uploadUrl}}`  
      - Body: binary data; set input binary field name to match your download binary property (commonly `data`)  
      - Header: `Content-Type: application/pdf`.
   8) Add **HTTP Request** node *LI API: Publish Live Post*  
      - POST `https://api.linkedin.com/rest/posts`  
      - Use the provided JSON body logic selecting option A vs B and referencing the document id.
   9) Add **Google Sheets** node *Archive Portfolio Publication* (append row to history sheet).

6. **Strategy path (false branch): fetch topic → 3-model generation**
   1) Add **Google Sheets** node *Fetch Content Queue* pointing to Topic Bank.
   2) Add **Sort** node *Prioritize Oldest Content* sorting by `Date Posted`.
   3) Add **Limit** node *Select Single Topic* (limit 1).
   4) Add **Code** node *Generate Agency Strategy* producing `selected_title`, `strategic_brief`, `row_number`.  
      - Ensure your Topic Bank headers match code expectations (decide “Pain Point” vs “Pain Points” and align).
   5) Create three model connectors:
      - *GPT-4o (Content Architect)* + *GPT-4o (Brand Voice Editor)*
      - *Claude 4.5 (Content Architect)* + *Claude 4.5 (Brand Voice Editor)*
      - *Gemini 2.5-Pro (Content Architect)* + *Gemini 2.5-Pro (Brand Voice Editor)*
   6) Create three strategy agents:
      - *Strategy: GPT Architect* (connect to GPT content architect model)
      - *Strategy: Claude Architect* (connect to Claude content architect model)
      - *Strategy: Gemini Architect* (connect to Gemini content architect model)
   7) Create three voice editor agents:
      - *Voice: GPT Editor* (connect to GPT brand voice model)
      - *Voice: Claude Editor* (connect to Claude brand voice model)
      - *Voice: Gemini Editor* (connect to Gemini brand voice model)
   8) Merge outputs using **Merge** node *Combine Cleaned Options* with 3 inputs.
   9) Add **Code** node *Draft Review Email* to sanitize and package the three outputs.

7. **Strategy editorial gatekeeper + logging**
   1) Add **Gmail** node *Human Editor Approval* (sendAndWait) with dropdown Option A/B/C/Decline.
   2) Add **Switch** node *Approved?* with rules:
      - Approve: regex `Option A|Option B|Option C`
      - Decline: equals `Decline all`
   3) Decline → **NoOp** *Discard Drafts*
   4) Approve → **Google Sheets Append** node *Log Approved Post* writing Date/Type/Topic/Status/Content.
   5) Add **Google Sheets Update** node *Update Topic Bank: Posted Status*  
      - Configure: document ID + sheet name (no leading `=`)  
      - Use `row_number` to target the correct row (or a lookup on Topic)  
      - Set `Status = Posted` and update `Date Posted = today` (recommended).  
      - As provided, this node is incomplete and must be configured.

8. **Error handling**
   1) Add **Error Trigger** node.
   2) Connect to **Gmail Send** node *Send a message* with body referencing failing node name and message.
   3) Ensure Gmail OAuth2 credential is configured and allowed to send.

**Credentials to configure**
- Google Sheets OAuth2
- Google Drive OAuth2
- Gmail OAuth2 (must support “sendAndWait”)
- OpenAI API
- Anthropic API
- Google Gemini (PaLM/Gemini) API
- LinkedIn OAuth2 (must have permissions to initialize upload + publish posts for the organization URN)

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “The Multi-Model Agency Content Engine (GPT, Claude, Gemini)” — alternates Portfolio (Drive) vs Strategy (Sheets); human gatekeeper; error handling; throttle; cleanup; setup requirements | Sticky Note (global overview) |
| “SYSTEM: Error Handling” | Sticky Note5 |
| “PHASE 1: INTAKE & ROUTING” | Sticky Note1 |
| “PHASE 2: PORTFOLIO EXTRACTION (TRUE PATH)” | Sticky Note2 |
| “PHASE 3: PORTFOLIO POST CREATION” | Sticky Note9 |
| “PHASE 4: PORTFOLIO EDITORIAL GATEKEEPER” | Sticky Note10 |
| “PHASE 5: PORTFOLIO POSTING & DELIVERY” | Sticky Note7 |
| “PHASE 6: STRATEGY ENGINEERING (FALSE PATH)” | Sticky Note3 |
| “PHASE 7: MULTI-MODEL CREATIVE & QC” | Sticky Note4 |
| “PHASE 8: EDITORIAL GATEKEEPER” | Sticky Note6 |
| “PHASE 9: TOPIC BANK DELIVERY” | Sticky Note8 |
| Setup guide: required Sheets headers, Drive folder, replacing placeholders (XX / YOUR COMPANY NAME), updating IDs/emails/credentials | Sticky Note11 |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.