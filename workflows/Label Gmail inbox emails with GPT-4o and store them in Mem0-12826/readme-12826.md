Label Gmail inbox emails with GPT-4o and store them in Mem0

https://n8nworkflows.xyz/workflows/label-gmail-inbox-emails-with-gpt-4o-and-store-them-in-mem0-12826


# Label Gmail inbox emails with GPT-4o and store them in Mem0

## 1. Workflow Overview

**Title:** Label Gmail inbox emails with GPT-4o and store them in Mem0  
**Purpose:** Automatically triage new Gmail inbox emails every 5 minutes, apply Gmail labels based on AI classification (primary: GPT‑4o information extractor; fallback: JigsawStack classifier), optionally handle a separate “internal/domain” branch, and persist a structured representation of each email into **Mem0** (long‑term memory) using a **Mem0 v2** payload.

### 1.1 Entry Points
- **Automated run:** Gmail Trigger polls inbox every 5 minutes.
- **Manual run (setup utility):** Manual trigger creates a predefined set of Gmail labels.

### 1.2 Logical Blocks
1. **Label Setup Utility (manual)**: Creates Gmail labels in bulk and summarizes success/errors.
2. **Inbox Intake & Pre-processing**: Watches inbox, sanitizes email content, extracts metadata, and detects marketing signals.
3. **Batching Loop**: Processes emails one-by-one (Split in Batches) and loops back until done.
4. **Triage Router**: Routes each email into Marketing, Internal(optional), or Main AI branch.
5. **Marketing Fast-Path**: Applies a “Marketing” label (optional filter gate).
6. **Main AI Extraction + Gmail Labeling**: Uses GPT‑4o extractor to produce structured fields and category → applies mapped Gmail label; on extractor failure uses Jigsaw classification fallback → applies mapped Gmail label.
7. **Finalize + Persist to Mem0**: Builds a final payload, formats Mem0 v2 schema in Python, sends to Mem0 API, then returns to batch loop.
8. **Optional Internal Branch**: Separate extractor + labeling path for internal-domain emails (and an optional continuation point).

---

## 2. Block-by-Block Analysis

### Block A — Label Setup Utility (Manual)
**Overview:** Creates a predefined list of Gmail labels (e.g., “📌 Urgent”, “✅ Action Required”) and reports how many were created successfully. This block is independent from the inbox automation.

**Nodes Involved:**
- Manual Trigger
- SETUP LIST
- Split Labels
- Create Gmail Simple
- Check Success
- Success Summary
- Error Details

#### Node: **Manual Trigger**
- **Type / Role:** Manual Trigger; starts the label-setup flow on demand.
- **Config:** No parameters.
- **Outputs:** Into **SETUP LIST**.
- **Failure modes:** None typical (manual execution only).

#### Node: **SETUP LIST**
- **Type / Role:** Set node; defines an array of label names.
- **Config choices:** Creates field `labels` (array) with emoji-prefixed names.
- **Key data:** `labels` array is the source for splitting.
- **Outputs:** To **Split Labels**.
- **Edge cases:** If label names already exist, downstream Gmail create may error.

#### Node: **Split Labels**
- **Type / Role:** Split Out; turns the `labels` array into one item per label.
- **Config:** Field to split: `labels`.
- **Outputs:** To **Create Gmail Simple**.
- **Edge cases:** If `labels` is empty/missing → no items produced.

#### Node: **Create Gmail Simple**
- **Type / Role:** Gmail node; creates a Gmail label.
- **Config choices:**
  - **Resource:** Label
  - **Operation:** Create
  - **Name:** from item: `{{$json.labels}}`
  - Visibility options set to show label in list and messages.
- **Credentials:** Gmail OAuth2 (`ZERO_INBOX_PROJECT`).
- **Outputs:** To **Check Success**.
- **Failure modes:** OAuth issues, insufficient permissions, label already exists, Gmail API quota.

#### Node: **Check Success**
- **Type / Role:** IF; routes based on whether an `error` field exists.
- **Config:** Checks `{{$json.error}}` **notExists**.
- **Outputs:**  
  - True → **Success Summary**  
  - False → **Error Details**
- **Edge cases:** If Gmail node returns errors in a different shape than `error`, this IF won’t catch them.

#### Node: **Success Summary**
- **Type / Role:** Code (JS); aggregates counts of non-error items.
- **Config:** Counts items where `item.json.error` is falsy.
- **Output:** A single summary item.

#### Node: **Error Details**
- **Type / Role:** Code (JS); returns per-label error records.
- **Config:** Filters items where `item.json.error` exists; maps to `{labelName, error, timestamp}`.
- **Output:** One item per error.

**Sticky note coverage (applies to all nodes in this block):**
- “## Set up your labels (simple way)”

---

### Block B — Inbox Intake & Pre-processing
**Overview:** Polls Gmail inbox every 5 minutes and standardizes each email into a consistent object (clean body, sender domain, marketing flags, unsubscribe detection) to make downstream AI more reliable.

**Nodes Involved:**
- [Trigger]: Watch Inbox (5m)
- [JS]: Sanitize & Detect Marketing

#### Node: **[Trigger]: Watch Inbox (5m)**
- **Type / Role:** Gmail Trigger; polls for new inbox emails.
- **Config choices:**
  - Poll every **5 minutes**
  - `simple: false` (expects richer payloads/structure)
- **Credentials:** Gmail OAuth2 (`ZERO_INBOX_PROJECT`)
- **Outputs:** To **[JS]: Sanitize & Detect Marketing**
- **Failure modes:** OAuth expiry, Gmail API quota, missing scopes, polling delays.

#### Node: **[JS]: Sanitize & Detect Marketing**
- **Type / Role:** Code (JavaScript); preprocesses each email.
- **Key behaviors:**
  - Extracts sender email and `senderDomain`
  - Detects marketing using:
    - `List-Unsubscribe` header presence
    - known marketing domains list
    - Gmail `CATEGORY_PROMOTIONS` label
    - unsubscribe links + no-reply heuristic
  - Sanitizes HTML → clean text, trims, cuts signatures, limits length (1500 chars)
  - Emits `metrics` including flags and header signals
- **Output fields (core):**
  - `id`, `subject`, `from`, `senderDomain`
  - `isMarketingEmail`, `marketingReason`
  - `cleanedBody`
  - `metrics.labels`, `metrics.hasListUnsubscribeHeader`, etc.
- **Outputs:** To **[Batch]: Process Emails**
- **Edge cases / failures:**
  - Unexpected Gmail payload shapes (headers/html/text fields vary)
  - Very large HTML bodies (mitigated by slicing)
  - Encoding/HTML entity issues (partially handled)

**Sticky note coverage:**
- “What This Workflow Does…” (describes the 4 stages and credential requirements)

---

### Block C — Batching Loop Controller
**Overview:** Ensures emails are processed sequentially and loops back after each email’s labeling + Mem0 persistence.

**Nodes Involved:**
- [Batch]: Process Emails
- End

#### Node: **[Batch]: Process Emails**
- **Type / Role:** Split In Batches; iterates through preprocessed emails.
- **Config choices:** `reset: false` (keeps batch state during the loop).
- **Connections:**
  - **Input:** from **[JS]: Sanitize & Detect Marketing** (or returns from downstream)
  - **Output 1:** to **End** (when completed / no more items)
  - **Output 2:** to **[Router]: Triage Streams** (per-item processing)
- **Edge cases:** Incorrect loop wiring can cause infinite loops or early termination.

#### Node: **End**
- **Type / Role:** NoOp; a visual terminator for the “done” output of the batch node.
- **Edge cases:** None.

**Sticky note coverage:** Same main note as Block B (overall workflow explanation).

---

### Block D — Triage Router (Marketing vs Internal vs Main)
**Overview:** Routes each email into one of three streams:
- **Marketing:** Fast label application
- **Exclude domain (optional):** Internal/domain filtering branch
- **Main:** Full AI extraction and persistence

**Nodes Involved:**
- [Router]: Triage Streams

#### Node: **[Router]: Triage Streams**
- **Type / Role:** Switch; routes based on conditions.
- **Rules:**
  1. **Marketing** if `{{$json.isMarketingEmail}}` is true
  2. **Exclude domain (optional)** if `{{$json.senderDomain}}` equals `""` (currently blank placeholder; intended to be set to your internal domain)
  - **Fallback output:** renamed to **Main**
- **Outputs:**
  - Marketing → **[Filter]: Needs Marketing Tag?**
  - Exclude domain (optional) → **[AI]: Internal Branch Extractor**
  - Main → **[AI]: Main Branch Extractor**
- **Edge cases:**
  - The “Exclude domain” rule is currently configured with an empty string; as-is, it likely never matches real domains.
  - If an email is marketing *and* internal, Marketing rule wins due to rule order.

**Sticky note coverage:**
- Main workflow note includes instruction: update internal domain rule here.

---

### Block E — Marketing Fast-Path (Optional Filter + Label)
**Overview:** For marketing emails, optionally checks if the marketing label is missing (or sender is Upwork) and then applies a specific Gmail label.

**Nodes Involved:**
- [Filter]: Needs Marketing Tag?
- [Gmail]: Tag as Marketing

#### Node: **[Filter]: Needs Marketing Tag?**
- **Type / Role:** Filter; gates whether to apply Marketing label.
- **Conditions (OR):**
  - `{{$json.metrics.labels.includes('Label_5435203421409704777')}}` is **false** (label not already applied)
  - OR `{{$json.senderDomain}} == "upwork.com"`
- **Output:** If passes → **[Gmail]: Tag as Marketing**
- **Failure modes:** If `metrics.labels` is missing/not an array, expression may error or behave unexpectedly.

#### Node: **[Gmail]: Tag as Marketing**
- **Type / Role:** Gmail; adds label to the **thread**.
- **Config choices:**
  - Resource: **thread**
  - Operation: **addLabels**
  - `labelIds`: `["Label_5435203421409704777"]`
  - `threadId`: `{{ $('[Trigger]: Watch Inbox (5m)').item.json.id }}`
    - Note: This references **id**, not `threadId`. In Gmail trigger payloads, `id` often refers to message id; labeling a thread typically needs the actual thread id. This is a potential bug unless the trigger returns thread id in `id`.
- **Output:** Loops back to **[Batch]: Process Emails**
- **Failure modes:** Wrong threadId → Gmail API error; OAuth/quota.

**Sticky note coverage (this block):**
- “The filter node is optional and can be deleted…”

---

### Block F — Main AI Extraction (GPT‑4o) + Gmail Labeling, with Jigsaw fallback
**Overview:** Attempts structured extraction with an Information Extractor driven by GPT‑4o. If it errors (node is set to continue on error), the workflow also calls Jigsaw classification as a backup and applies the mapped label from that result.

**Nodes Involved:**
- [LLM]: GPT-4o
- [AI]: Main Branch Extractor
- [Gmail]: Apply AI Category
- [API]: Jigsaw Classification
- [Gmail]: Apply Jigsaw Category

#### Node: **[LLM]: GPT-4o**
- **Type / Role:** LangChain Chat Model; provides the model to the extractor.
- **Config choices:** `chatgpt-4o-latest`, `temperature: 0.6`, `maxRetries: 2`
- **Connection:** Feeds the **ai_languageModel** input of **[AI]: Main Branch Extractor**
- **Failure modes:** OpenAI auth, rate limits, model availability, timeouts.

#### Node: **[AI]: Main Branch Extractor**
- **Type / Role:** LangChain Information Extractor; produces structured fields from the email.
- **Error handling:** `onError: continueErrorOutput` (important: downstream must handle partial/error outputs).
- **Prompting / input text:**
  - Builds a text block containing subject/from, metrics, and `<email_body>` from `cleanedBody`.
- **Schema (attributes):** Extracts:
  - `core_objective`, `email_summary`, `sender_email`, `requires_reply`, `requested_action`, `deadline`
  - `is_marketing_email`, `category` (must be one of defined set), `threadId` (forced from trigger)
  - plus some fields that appear misused as “description” fields (`GMAIL_LABEL`, `userId`)—these do not automatically populate unless the extractor interprets them.
- **Outputs:**
  - Main output → **[Gmail]: Apply AI Category**
  - Error/alternate path still triggers **[API]: Jigsaw Classification** due to wiring (both are connected from the extractor’s output in this workflow).
- **Edge cases / risks:**
  - If extractor returns no `output.category`, label mapping defaults to FYI.
  - The schema includes several “description” entries that look like expressions; information extractor tools typically use “description” as guidance, not as an expression to evaluate. If you intended to inject runtime values, use proper fields supported by the node.
  - `threadId` is hard-coded from the trigger item; if batching changes item context, this can mismatch.

#### Node: **[Gmail]: Apply AI Category**
- **Type / Role:** Gmail; applies category label based on extractor output.
- **Config:**
  - Resource: thread; Operation: addLabels
  - `threadId`: `{{$json.output.threadId}}`
  - `labelIds`: computed mapping from `output.category`:
    - Urgent→Label_4, Action Required→Label_5, Time‑Sensitive→Label_6, FYI→Label_7, Calendar→Label_8, System Notification/Newsletter→Label_9, Cold Outreach→Label_10
    - Default: Label_7 (FYI)
- **Output:** To **[Set]: Finalize Payload Fields**
- **Failure modes:** Missing `output.threadId` → Gmail error.

#### Node: **[API]: Jigsaw Classification**
- **Type / Role:** HTTP Request; fallback classification.
- **Config:**
  - POST `https://api.jigsawstack.com/v1/classification`
  - Header auth credential: `jigsaw`
  - JSON body is dynamically built; notably it says “PARSER STATUS: FAILED (Using Raw Fallback)” and uses `$json.cleanedBody` substring as context.
  - Labels: Urgent, Action Required, Calendar, FYI / Notification, Newsletter / Promotional, Spam / Cold Outreach
  - `multiple_labels: false`
  - Timeout 10s; batch size 1
- **Outputs:** To **[Gmail]: Apply Jigsaw Category**
- **Edge cases:**
  - If `$json.cleanedBody` is missing, context becomes weak.
  - API rate limits/timeouts.
  - The “parser failed” message is always included; consider making it conditional if desired.

#### Node: **[Gmail]: Apply Jigsaw Category**
- **Type / Role:** Gmail; applies label based on Jigsaw prediction.
- **Config:**
  - Category = `$json.predictions[0]`
  - Mapping:
    - Urgent→Label_4, Action Required→Label_5, Calendar→Label_8,
    - FYI / Notification→Label_7,
    - Newsletter / Promotional→Label_9,
    - Spam / Cold Outreach→Label_10
    - Default→Label_7
  - Thread target: `{{ $('[Trigger]: Watch Inbox (5m)').item.json.id }}`
    - Same potential threadId vs messageId issue as Marketing tag node.
- **Output:** To **[Set]: Finalize Payload Fields**
- **Failure modes:** Wrong thread ID, missing predictions.

**Sticky note coverage (main extractor area):**
- “## Your Main Prompt Add or delete categories as you see fit…”

---

### Block G — Finalize Fields + Mem0 Persistence
**Overview:** Collects final email fields, formats them into Mem0 v2 schema with precedence rules for categories, then posts to Mem0. After persistence, loops back to the batch node.

**Nodes Involved:**
- [Set]: Finalize Payload Fields
- [Python]: Format Mem0 V2 Schema
- [API]: Push to Mem0 Long-term

#### Node: **[Set]: Finalize Payload Fields**
- **Type / Role:** Set; normalizes the payload used for Mem0 formatting.
- **Config choices:** `ignoreConversionErrors: true`
- **Assignments:**
  - `cleaned_email_body` from **[JS]: Sanitize & Detect Marketing** `cleanedBody`
  - `metrics.hasListUnsubscribeHeader`, `from`, `subject`, `id`
  - `threadId` from trigger `threadId`
- **Outputs:** To **[Python]: Format Mem0 V2 Schema**
- **Edge cases:**
  - Uses cross-node item references (`$('[JS]:...').item.json...`); if item pairing is off, it can pull the wrong email’s content.
  - `threadId` is pulled from trigger, not necessarily the current item’s thread.

#### Node: **[Python]: Format Mem0 V2 Schema**
- **Type / Role:** Code (Python); creates `mem0Payload` per item.
- **Key behaviors:**
  - Normalizes sender email, decides role (AGENT vs CUSTOMER) based on allowed domains
  - Enforces label precedence:
    1) Spam overrides all  
    2) Newsletter/Promotional overrides all  
    3) else Urgent > Action Required > Calendar > FYI / Notification
  - Run scoping strategy:
    - Spam/Newsletter → `run_id = message_id`
    - Otherwise → `run_id = thread_id`
  - Builds Mem0 message content including Subject/From/Summary/Body
  - Outputs `{ mem0Payload: {...} }`
- **Important config constants to edit:**
  - `MAILBOX_USER_ID = "yourname_inbox"`
  - `AGENT_DOMAINS = ["yourdomain.nl"]`
  - `APP_ID`, `AGENT_ID`
- **Edge cases / failures:**
  - `message_id` selection falls back across multiple keys; if none exist, run_id can become empty.
  - If upstream `output` structure isn’t present (e.g., Jigsaw path), category may default.
  - Produces `{error: True}` items on exception; downstream HTTP node is set to continue on error.

#### Node: **[API]: Push to Mem0 Long-term**
- **Type / Role:** HTTP Request; persists memory to Mem0.
- **Config:**
  - POST `https://api.mem0.ai/v1/memories/`
  - Body: `{{$json.mem0Payload}}`
  - Auth: Header auth credential `Mem0`
  - `onError: continueErrorOutput`
- **Outputs:** Loops back to **[Batch]: Process Emails**
- **Failure modes:** Auth errors, schema mismatch, Mem0 downtime, rate limits.

---

### Block H — Optional Internal Branch (Domain-specific handling)
**Overview:** If enabled via the router’s “Exclude domain (optional)” output, this branch uses a separate extractor model (Mistral) and applies labels. It then ends at an “Optional Branch” NoOp intended to be linked back into the main batch loop if desired.

**Nodes Involved:**
- [LLM]: Mistral
- [AI]: Internal Branch Extractor
- [Gmail]: Tag Domain (Always runs first. Tags it as "Agentive Concepts")
- [Gmail]: Apply category for branch
- Optional Branch

#### Node: **[LLM]: Mistral**
- **Type / Role:** LangChain Chat Model (Mistral Cloud) for internal extractor.
- **Config:** `mistral-medium-latest`, temperature 0.6, maxRetries 2
- **Connection:** ai_languageModel → **[AI]: Internal Branch Extractor**
- **Failure modes:** API key invalid, rate limits.

#### Node: **[AI]: Internal Branch Extractor**
- **Type / Role:** Information Extractor for internal emails.
- **Error handling:** continue on error.
- **Special triage rule:** If automated/unsubscribe, sets `requires_reply=false` and simplifies output.
- **Outputs:** Two parallel Gmail labeling nodes:
  - **[Gmail]: Apply category for branch**
  - **[Gmail]: Tag Domain (Always runs first...)**
- **Edge cases:** Same schema concerns as main extractor (descriptions vs runtime expressions).

#### Node: **[Gmail]: Tag Domain (Always runs first. Tags it as "Agentive Concepts")**
- **Type / Role:** Gmail; adds a constant label `Label_11`.
- **Thread:** `{{$json.output.threadId}}`
- **Output:** To **Optional Branch**
- **Failure modes:** Missing `output.threadId`.

#### Node: **[Gmail]: Apply category for branch**
- **Type / Role:** Gmail; maps `output.category` to label IDs (same mapping style as main).
- **Thread:** `{{$json.output.threadId}}`
- **Output:** To **Optional Branch**

#### Node: **Optional Branch**
- **Type / Role:** NoOp; placeholder end-point.
- **Note in flow:** “I you want to use it link it back to the split batch node”
- **Edge cases:** As shipped, it does not loop back; internal items stop here.

**Sticky note coverage (internal branch):**
- “## THIS is OPTIONAL … link it back to the split batch node.”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Manual Trigger | manualTrigger | Manual entry to create Gmail labels | — | SETUP LIST | ## Set up your labels (simple way) |
| SETUP LIST | set | Define array of label names | Manual Trigger | Split Labels | ## Set up your labels (simple way) |
| Split Labels | splitOut | Create one item per label | SETUP LIST | Create Gmail Simple | ## Set up your labels (simple way) |
| Create Gmail Simple | gmail | Create Gmail labels | Split Labels | Check Success | ## Set up your labels (simple way) |
| Check Success | if | Route to summary vs error list | Create Gmail Simple | Success Summary; Error Details | ## Set up your labels (simple way) |
| Success Summary | code | Aggregate success counts | Check Success | — | ## Set up your labels (simple way) |
| Error Details | code | Emit per-label errors | Check Success | — | ## Set up your labels (simple way) |
| [Trigger]: Watch Inbox (5m) | gmailTrigger | Poll inbox every 5 minutes | — | [JS]: Sanitize & Detect Marketing | ## What This Workflow Does … (credentials + customization notes) |
| [JS]: Sanitize & Detect Marketing | code | Clean body + detect marketing + metrics | [Trigger]: Watch Inbox (5m) | [Batch]: Process Emails | ## What This Workflow Does … (credentials + customization notes) |
| [Batch]: Process Emails | splitInBatches | Per-email iteration + loop control | [JS]: Sanitize & Detect Marketing; [Gmail]: Tag as Marketing; [API]: Push to Mem0 Long-term | End; [Router]: Triage Streams | ## What This Workflow Does … (credentials + customization notes) |
| End | noOp | Batch завершение placeholder | [Batch]: Process Emails | — |  |
| [Router]: Triage Streams | switch | Route Marketing/Internal/Main | [Batch]: Process Emails | [Filter]: Needs Marketing Tag?; [AI]: Internal Branch Extractor; [AI]: Main Branch Extractor | ## What This Workflow Does … (mentions updating internal domain rule) |
| [Filter]: Needs Marketing Tag? | filter | Optional gate for marketing labeling | [Router]: Triage Streams | [Gmail]: Tag as Marketing | ## The filter node is optional and can be deleted , keep if you already have filters active in Gmail. |
| [Gmail]: Tag as Marketing | gmail | Add Marketing label to thread | [Filter]: Needs Marketing Tag? | [Batch]: Process Emails | ## The filter node is optional and can be deleted , keep if you already have filters active in Gmail. |
| [LLM]: GPT-4o | lmChatOpenAi | LLM for main extractor | — | [AI]: Main Branch Extractor (ai_languageModel) | ## Your Main Prompt Add or delete categories as you see fit. |
| [AI]: Main Branch Extractor | informationExtractor | Extract structured fields + category | [Router]: Triage Streams; [LLM]: GPT-4o (model input) | [Gmail]: Apply AI Category; [API]: Jigsaw Classification | ## Your Main Prompt Add or delete categories as you see fit. |
| [Gmail]: Apply AI Category | gmail | Apply label mapped from AI category | [AI]: Main Branch Extractor | [Set]: Finalize Payload Fields |  |
| [API]: Jigsaw Classification | httpRequest | Fallback classifier | [AI]: Main Branch Extractor | [Gmail]: Apply Jigsaw Category |  |
| [Gmail]: Apply Jigsaw Category | gmail | Apply label mapped from Jigsaw prediction | [API]: Jigsaw Classification | [Set]: Finalize Payload Fields |  |
| [Set]: Finalize Payload Fields | set | Normalize fields for Mem0 formatting | [Gmail]: Apply AI Category; [Gmail]: Apply Jigsaw Category | [Python]: Format Mem0 V2 Schema |  |
| [Python]: Format Mem0 V2 Schema | code (python) | Build Mem0 v2 payload + category precedence | [Set]: Finalize Payload Fields | [API]: Push to Mem0 Long-term |  |
| [API]: Push to Mem0 Long-term | httpRequest | Persist to Mem0, then loop | [Python]: Format Mem0 V2 Schema | [Batch]: Process Emails |  |
| [LLM]: Mistral | lmChatMistralCloud | LLM for internal extractor | — | [AI]: Internal Branch Extractor (ai_languageModel) | ## THIS is OPTIONAL … link it back to the split batch node. |
| [AI]: Internal Branch Extractor | informationExtractor | Internal-domain extraction path | [Router]: Triage Streams; [LLM]: Mistral (model input) | [Gmail]: Apply category for branch; [Gmail]: Tag Domain… | ## THIS is OPTIONAL … link it back to the split batch node. |
| [Gmail]: Tag Domain (Always runs first. Tags it as "Agentive Concepts") | gmail | Apply constant internal label | [AI]: Internal Branch Extractor | Optional Branch | ## THIS is OPTIONAL … link it back to the split batch node. |
| [Gmail]: Apply category for branch | gmail | Apply label mapped from internal category | [AI]: Internal Branch Extractor | Optional Branch | ## THIS is OPTIONAL … link it back to the split batch node. |
| Optional Branch | noOp | Placeholder; intended to reconnect to batch | [Gmail]: Tag Domain…; [Gmail]: Apply category for branch | — | ## THIS is OPTIONAL … link it back to the split batch node. |
| Sticky Note1 | stickyNote | Comment | — | — | (content is the note itself) |
| Sticky Note2 | stickyNote | Comment | — | — | (content is the note itself) |
| Sticky Note3 | stickyNote | Comment | — | — | (content is the note itself) |
| Sticky Note4 | stickyNote | Comment | — | — | (content is the note itself) |
| Sticky Note5 | stickyNote | Comment | — | — | (content is the note itself) |

---

## 4. Reproducing the Workflow from Scratch

### A) Create Credentials (Required)
1. **Gmail OAuth2 credential**
   - In n8n: Credentials → Gmail OAuth2
   - Ensure scopes allow reading inbox and modifying labels/threads.
2. **OpenAI API credential**
   - Credentials → OpenAI API
   - Must support `chatgpt-4o-latest` (or choose available equivalent).
3. **Mistral Cloud API Key** (only if using Optional Internal Branch)
4. **HTTP Header Auth: Jigsaw**
   - Add header (commonly `Authorization: Bearer <key>` or per JigsawStack docs).
5. **HTTP Header Auth: Mem0**
   - Add required Mem0 header (commonly `Authorization: Token <key>` or `Bearer <key>` depending on Mem0 docs).

---

### B) Label Setup Utility (Manual)
1. Add **Manual Trigger**.
2. Add **Set** node named `SETUP LIST`
   - Add field `labels` (type Array) with your desired label names.
3. Add **Split Out** node named `Split Labels`
   - Field to split out: `labels`
4. Add **Gmail** node named `Create Gmail Simple`
   - Resource: **Label**
   - Operation: **Create**
   - Name: `{{$json.labels}}`
   - Set label visibility/message visibility as desired
   - Select Gmail OAuth2 credential
5. Add **IF** node `Check Success`
   - Condition: `{{$json.error}}` → **not exists**
6. Add **Code (JS)** node `Success Summary` (aggregates successes).
7. Add **Code (JS)** node `Error Details` (lists errors).
8. Connect: Manual Trigger → SETUP LIST → Split Labels → Create Gmail Simple → Check Success → (true) Success Summary, (false) Error Details.

---

### C) Inbox Automation (Main Flow)
1. Add **Gmail Trigger** node `[Trigger]: Watch Inbox (5m)`
   - Poll every 5 minutes
   - Connect Gmail credential
2. Add **Code (JS)** node `[JS]: Sanitize & Detect Marketing`
   - Paste the sanitization/marketing detection script
3. Add **Split In Batches** node `[Batch]: Process Emails`
   - Options: `reset = false`
4. Add **NoOp** node `End`
5. Add **Switch** node `[Router]: Triage Streams`
   - Rule 1 (Marketing): `{{$json.isMarketingEmail}}` is true
   - Rule 2 (Exclude domain optional): `{{$json.senderDomain}}` equals `yourdomain.com` (replace placeholder)
   - Fallback output renamed to `Main`
6. Wire: Trigger → JS → Batch.
7. Wire Batch output(0) → End; Batch output(1) → Router.

---

### D) Marketing Branch
1. Add **Filter** `[Filter]: Needs Marketing Tag?` (optional)
   - Condition OR:
     - `={{ $json.metrics.labels.includes('YOUR_MARKETING_LABEL_ID') }}` is false
     - OR `={{ $json.senderDomain }}` equals `upwork.com`
2. Add **Gmail** node `[Gmail]: Tag as Marketing`
   - Resource: thread
   - Operation: addLabels
   - labelIds: array with your marketing label ID
   - threadId: **use a real thread id** (recommended: `{{$json.threadId}}` if you carry it; otherwise map from trigger output properly)
3. Connect Router(Marketing) → Filter → Gmail Tag → Batch (to continue loop).

---

### E) Main AI Branch (GPT‑4o + fallback)
1. Add **LLM Chat Model** node `[LLM]: GPT-4o`
   - Model: `chatgpt-4o-latest`
   - Temperature 0.6; Max retries 2
2. Add **Information Extractor** node `[AI]: Main Branch Extractor`
   - Connect Router(Main) → Extractor (main input)
   - Connect `[LLM]: GPT-4o` → Extractor (ai_languageModel input)
   - Configure extractor text template using subject/from/metrics/cleanedBody
   - Define attributes including required `category` and `threadId`
   - Set **Continue On Fail** / error output behavior (as in original)
3. Add **Gmail** node `[Gmail]: Apply AI Category`
   - Map `output.category` to label IDs via expression
   - threadId: `{{$json.output.threadId}}`
4. Add **HTTP Request** node `[API]: Jigsaw Classification`
   - POST to Jigsaw endpoint
   - Header auth credential
   - JSON body with dataset text and allowed labels
5. Add **Gmail** node `[Gmail]: Apply Jigsaw Category`
   - Map `predictions[0]` to label IDs
   - threadId: again, ensure you pass the real thread id
6. Wiring (to match current behavior):
   - Extractor → Apply AI Category → Set Finalize
   - Extractor → Jigsaw → Apply Jigsaw Category → Set Finalize  
   (If you want *true fallback only on error*, add an IF checking for extractor error and route accordingly.)

---

### F) Finalize + Mem0
1. Add **Set** node `[Set]: Finalize Payload Fields`
   - Create fields: `cleaned_email_body`, `from`, `subject`, `id`, `threadId`, and any metrics you want
   - Prefer using the **current item’s** fields rather than cross-node item references, to avoid mis-pairing.
2. Add **Code (Python)** node `[Python]: Format Mem0 V2 Schema`
   - Paste the Python code
   - Update constants: `MAILBOX_USER_ID`, `AGENT_DOMAINS`, etc.
3. Add **HTTP Request** node `[API]: Push to Mem0 Long-term`
   - POST `https://api.mem0.ai/v1/memories/`
   - Body: `{{$json.mem0Payload}}`
   - Header auth credential
   - Set “Continue On Fail” if you want the batch loop to keep going
4. Connect: Set → Python → Mem0 → back to **[Batch]: Process Emails**.

---

### G) Optional Internal Branch
1. Ensure Switch rule “Exclude domain (optional)” matches your internal domain.
2. Add **LLM Mistral** node and **Internal Branch Extractor** node.
3. Add two Gmail nodes:
   - Tag Domain with constant label (Label_11)
   - Apply category mapping label
4. Connect both to **Optional Branch** NoOp.
5. If you want internal emails to continue to Mem0 + batching, connect **Optional Branch** back to **[Batch]: Process Emails** (as the note suggests).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “WIth this workflow you get instant email management… four key stages… credentials needed… update internal domain in router.” | Sticky note “What This Workflow Does” (embedded in workflow canvas) |
| “The filter node is optional and can be deleted…” | Sticky note near Marketing filter/labeling |
| “THIS is OPTIONAL… link it back to the split batch node.” | Sticky note near Internal Branch |
| “Your Main Prompt… Add or delete categories…” | Sticky note near Main extractor |
| “Set up your labels (simple way)” | Sticky note near label-creation utility |

**Important implementation note (high impact):** multiple Gmail label nodes use `threadId` set to `{{ $('[Trigger]: Watch Inbox (5m)').item.json.id }}`. In many Gmail payloads, `id` is a message id, not a thread id. To avoid Gmail API errors or mislabeling, ensure you pass the true thread id (typically `threadId`) consistently through preprocessing and into labeling nodes.