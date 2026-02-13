Score and route leads with Telegram alerts and Box storage

https://n8nworkflows.xyz/workflows/score-and-route-leads-with-telegram-alerts-and-box-storage-13065


# Score and route leads with Telegram alerts and Box storage

## 1. Workflow Overview

**Workflow name:** Lead Scoring Pipeline with Telegram and Box  
**Purpose:** Receive lead submissions via a webhook, normalize/validate the payload, enrich it with Clearbit, compute a lead score, route the lead as *qualified* or *unqualified*, store a JSON record in Box, and notify the appropriate Telegram channel. Invalid submissions are flagged to Marketing immediately.

### 1.1 Input Reception & Normalization
Receives inbound form data and standardizes it so downstream nodes can rely on consistent field names.

### 1.2 Validation & Invalid-Lead Alerting
Checks for required fields (email + company). Invalid leads are diverted to a Telegram alert for Marketing.

### 1.3 Enrichment & Scoring
Calls Clearbit to enrich the lead, merges enrichment with the original submission, calculates a numeric score, and prepares the final lead record.

### 1.4 Qualification, Storage (Box) & Notifications (Telegram)
Routes based on score threshold (≥ 70). Each branch serializes the lead into a JSON file, uploads it to Box, and notifies Sales or Marketing.

### 1.5 (Documented but **Missing**) Error Handling
A sticky note describes a “Workflow Error Trigger” path, but **no such node exists** in this JSON. Error handling is therefore not implemented.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & Normalization

**Overview:** Accepts new lead submissions via POST webhook and passes data through a Set node intended to normalize fields (though no explicit mappings are configured).  
**Nodes involved:** `Lead Form Webhook`, `Normalize Lead Data`

#### Node: Lead Form Webhook
- **Type / role:** Webhook trigger (`n8n-nodes-base.webhook`) — entry point for external form providers.
- **Key configuration:**
  - **HTTP Method:** POST
  - **Path:** `lead-intake` (workflow will expose a URL ending in `/webhook/lead-intake` or `/webhook-test/lead-intake` depending on mode)
  - **Response mode:** `lastNode` (response returned from the last executed node in the run)
- **Inputs/Outputs:**
  - **Output:** to `Normalize Lead Data`
- **Edge cases / failures:**
  - Incorrect HTTP method/path → 404/405 at the caller
  - If downstream errors occur, webhook response may be error/timeout depending on n8n settings and runtime

#### Node: Normalize Lead Data
- **Type / role:** Set node (`n8n-nodes-base.set`) — intended to standardize incoming fields.
- **Key configuration choices (interpreted):**
  - No explicit “Set” fields are defined; it currently behaves like a pass-through unless defaults are applied by UI options.
- **Connections:**
  - **Output →** `Validate Required Fields`
  - **Also output →** `Merge Enriched Data` **(input index 1)** to preserve original data for merging
- **Edge cases / failures:**
  - If the upstream payload doesn’t contain the expected keys (`email`, `company`, etc.), validation/scoring will be impacted.
- **Version notes:** Node typeVersion `3.4` (n8n Set node behavior differs slightly across versions; ensure you’re on a compatible n8n release that supports Set v3.x).

**Sticky note covering this block:** “📥 Intake & Validation” (applies conceptually to nodes here and in Block 2).

---

### Block 2 — Validation & Invalid-Lead Alerting

**Overview:** Validates that `email` and `company` exist. If missing, it alerts Marketing via Telegram and stops the enrichment/scoring path.  
**Nodes involved:** `Validate Required Fields`, `Has Required Fields?`, `Notify Marketing (Invalid Submission)`

#### Node: Validate Required Fields
- **Type / role:** Code node (`n8n-nodes-base.code`) — computes a validation result.
- **Key logic:**
  - Reads `email` and `company` from `item.json`
  - Produces `validation.valid` boolean and `validation.message`
- **Key variables/fields created:**
  - `validation.valid` (boolean)
  - `validation.message` (string: `"OK"` or `"Missing required email or company"`)
- **Connections:**
  - **Output →** `Has Required Fields?`
  - **Also output →** `Merge Enriched Data` **(input index 1)** (original+validation stream is merged later)
- **Edge cases / failures:**
  - If incoming item is not an object or missing `.json`, the code node will throw.
  - If email/company are nested under different keys (common with form tools), `valid` will be false and lead will be treated as invalid.

#### Node: Has Required Fields?
- **Type / role:** IF node (`n8n-nodes-base.if`) — branching based on validation boolean.
- **Condition:**
  - Boolean `isTrue` on `={{ $json.validation.valid }}`
- **Connections:**
  - **True →** `Enrich with Clearbit`
  - **False →** `Notify Marketing (Invalid Submission)`
- **Edge cases / failures:**
  - If `validation.valid` is undefined (because upstream logic changed), the condition evaluates falsy and routes to invalid branch.

#### Node: Notify Marketing (Invalid Submission)
- **Type / role:** Telegram node (`n8n-nodes-base.telegram`) — sends an alert message.
- **Key configuration:**
  - **Chat ID:** `={{ $env.TELEGRAM_MARKETING_CHAT_ID || 'YOUR_CHAT_ID' }}`
  - **Text:** `❗️ Received invalid lead submission. Reason: {{$json.validation.message}}`
- **Connections:**
  - Terminal node for invalid path (no outgoing connections)
- **Edge cases / failures:**
  - Missing/incorrect Telegram credentials → auth errors
  - Missing env var and left as `'YOUR_CHAT_ID'` → message will fail or go to an unintended chat if not replaced
  - Rate limits if volume is high

**Sticky note covering this block:** “📥 Intake & Validation”.

---

### Block 3 — Enrichment & Scoring

**Overview:** Enriches valid leads with Clearbit, merges enrichment with original submission, calculates a score, and prepares the lead record for routing.  
**Nodes involved:** `Enrich with Clearbit`, `Merge Enriched Data`, `Calculate Lead Score`, `Prepare Lead Record`

#### Node: Enrich with Clearbit
- **Type / role:** HTTP Request node (`n8n-nodes-base.httpRequest`) — calls Clearbit Combined Find endpoint.
- **Key configuration:**
  - **URL:** `https://company.clearbit.com/v2/combined/find?email={{$json.email}}`
  - **Auth:** predefined credential type `clearbitApi`
- **Connections:**
  - **Output →** `Merge Enriched Data` (input index 0)
- **Edge cases / failures:**
  - Clearbit 4xx (invalid email, not found, invalid key) or 5xx
  - Network timeouts
  - Clearbit response shape changes could break scoring expressions
- **Important integration note:**
  - The scoring node expects enrichment under `data.enrichment...`, but this workflow **does not explicitly wrap** the Clearbit response into an `enrichment` field. Unless Clearbit’s response is already being placed under `enrichment` by some hidden option (not shown), scoring may not find `data.enrichment` as expected.

#### Node: Merge Enriched Data
- **Type / role:** Merge node (`n8n-nodes-base.merge`) — merges two streams by item index.
- **Mode:** `mergeByIndex`
- **Inputs:**
  - **Input 0:** from `Enrich with Clearbit`
  - **Input 1:** from `Normalize Lead Data` and `Validate Required Fields` (both connect to input 1)
- **Connections:**
  - **Output →** `Calculate Lead Score`
- **Edge cases / failures:**
  - **Dual wiring into the same merge input (index 1)** means `Merge Enriched Data` may receive **two separate item streams** on input 1 (one from Normalize, one from Validate). This can lead to:
    - unexpected item counts
    - misaligned merge-by-index results
    - duplicated/incorrect merged objects
  - If Clearbit returns no item (or errors and stops), merge will not behave as intended.

#### Node: Calculate Lead Score
- **Type / role:** Code node — computes numeric `leadScore`.
- **Key scoring rules:**
  - `+20` if `company` exists
  - Employee-based points (looks in `data.enrichment?.company?.metrics?.employees` or `data.enrichment?.company?.employees`)
    - `>500` → `+30`
    - `>100` → `+20`
    - else → `+10`
  - `+20` if `data.enrichment?.person?.linkedin?.handle` exists
  - `+10` if `email` matches basic regex `^.+@.+\..+$`
- **Outputs:**
  - Adds `leadScore` to the item JSON
- **Edge cases / failures:**
  - If enrichment is not under `enrichment`, employee/LinkedIn points will never be added (score will be lower than intended).
  - If `email` is missing, regex test may fail safely (it will test `undefined` and return false).

#### Node: Prepare Lead Record
- **Type / role:** Set node — intended to stamp status/timestamp and finalize shape.
- **Key configuration choices (interpreted):**
  - No fields configured; currently acts like a pass-through.
- **Connections:**
  - **Output →** `Qualified Lead?`
- **Edge cases:**
  - If you expect a timestamp/status downstream, it currently won’t exist unless you configure this node.

**Sticky note covering this block:** “🧠 Enrichment & Scoring”.

---

### Block 4 — Qualification, Storage (Box) & Notifications (Telegram)

**Overview:** Routes leads by score threshold. Each branch creates a JSON “file” in the item, uploads it to Box, then notifies the relevant Telegram channel.  
**Nodes involved:** `Qualified Lead?`, `Build Lead JSON (Qualified)`, `Upload to Box (Qualified)`, `Notify Sales (Qualified)`, `Build Lead JSON (Unqualified)`, `Upload to Box (Unqualified)`, `Notify Marketing (Unqualified)`

#### Node: Qualified Lead?
- **Type / role:** IF node — qualification gate.
- **Condition:** `={{ $json.leadScore }}` **largerEqual** `70`
- **Connections:**
  - **True →** `Build Lead JSON (Qualified)`
  - **False →** `Build Lead JSON (Unqualified)`
- **Edge cases:**
  - If `leadScore` is missing/null, numeric comparison can behave unexpectedly and may route to False branch.

#### Node: Build Lead JSON (Qualified)
- **Type / role:** Code node — serializes current lead record into JSON file content.
- **Key outputs:**
  - `fileName`: `lead_<timestamp>_qualified.json`
  - `fileContent`: pretty-printed JSON string of `item.json`
- **Connections:** → `Upload to Box (Qualified)`
- **Edge cases:**
  - `Date.now()` collision is unlikely but possible under extremely high concurrency.
  - Large payloads may exceed Box or node limits.

#### Node: Upload to Box (Qualified)
- **Type / role:** Box node — uploads file content.
- **Key configuration:**
  - Uses `fileName` and `fileContent` expressions from previous node.
  - **Critical missing configuration:** no folder ID is specified in the shown parameters. The sticky note says to use env vars `BOX_QUALIFIED_FOLDER_ID`, but this node does not reference them.
- **Connections:** → `Notify Sales (Qualified)`
- **Edge cases / failures:**
  - Missing Box OAuth2 credentials → auth failure
  - Missing target folder configuration → upload may fail or go to a default location depending on node defaults (often it fails)

#### Node: Notify Sales (Qualified)
- **Type / role:** Telegram node — notifies Sales.
- **Chat ID:** `={{ $env.TELEGRAM_SALES_CHAT_ID || 'YOUR_CHAT_ID' }}`
- **Text:** `🎉 New qualified lead: {{$json.firstName}} {{$json.lastName}} (Score {{$json.leadScore}}). Check Box for details.`
- **Edge cases:**
  - `firstName/lastName` may not exist unless the form provides them or normalization sets them → message may show blanks.
  - Same Telegram credential/env-var caveats as earlier.

#### Node: Build Lead JSON (Unqualified)
- **Type / role:** Code node — same as qualified but with `_unqualified`.
- **Connections:** → `Upload to Box (Unqualified)`

#### Node: Upload to Box (Unqualified)
- **Type / role:** Box node — uploads unqualified JSON.
- **Critical missing configuration:** same as qualified branch; folder ID not shown/referenced.
- **Connections:** → `Notify Marketing (Unqualified)`

#### Node: Notify Marketing (Unqualified)
- **Type / role:** Telegram node — notifies Marketing.
- **Chat ID:** `={{ $env.TELEGRAM_MARKETING_CHAT_ID || 'YOUR_CHAT_ID' }}`
- **Text:** `ℹ️ New unqualified lead: {{$json.firstName}} {{$json.lastName}} (Score {{$json.leadScore}}). Stored in Box.`
- **Edge cases:** same name/env var considerations.

**Sticky note covering this block:** “🚦 Qualification & Routing”.

---

### Block 5 — Error Handling (Declared but not implemented)

**Overview:** A sticky note describes a workflow-wide error trigger that should notify Ops, but there is no node implementing it.  
**Nodes involved:** *(none present in JSON)*  
**Consequence:** Unhandled exceptions will not proactively notify Operations via Telegram.

**Sticky note covering this block:** “🛠️ Error Handling”.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| ⚡️ Lead Scoring Overview | Sticky Note | Documentation / overview | — | — | ## How it works… (includes setup steps for Clearbit, Box env vars, Telegram env vars, threshold tuning) |
| 📥 Intake & Validation | Sticky Note | Documentation / block note | — | — | ## Intake & Validation (explains webhook, normalization, validation, invalid alerting) |
| 🧠 Enrichment & Scoring | Sticky Note | Documentation / block note | — | — | ## Enrichment & Scoring (Clearbit enrichment, merge, scoring, extensibility) |
| 🚦 Qualification & Routing | Sticky Note | Documentation / block note | — | — | ## Qualification & Routing (threshold routing, Box storage, Telegram notifications) |
| 🛠️ Error Handling | Sticky Note | Documentation / block note | — | — | ## Error Handling & Ops (describes Workflow Error Trigger path, but node is missing) |
| Lead Form Webhook | Webhook | Intake entry point | — | Normalize Lead Data | ## Intake & Validation (group handles incoming lead data…) |
| Normalize Lead Data | Set | Standardize field names / pass-through | Lead Form Webhook | Validate Required Fields; Merge Enriched Data | ## Intake & Validation (standardises key field names…) |
| Validate Required Fields | Code | Add `validation` object | Normalize Lead Data | Has Required Fields?; Merge Enriched Data | ## Intake & Validation (checks for presence of email and company…) |
| Has Required Fields? | IF | Route valid vs invalid | Validate Required Fields | Enrich with Clearbit; Notify Marketing (Invalid Submission) | ## Intake & Validation (separates valid and invalid leads…) |
| Notify Marketing (Invalid Submission) | Telegram | Alert Marketing about invalid submission | Has Required Fields? (false) | — | ## Intake & Validation (invalid submissions trigger an immediate Telegram alert…) |
| Enrich with Clearbit | HTTP Request | Clearbit combined enrichment | Has Required Fields? (true) | Merge Enriched Data | ## Enrichment & Scoring (enriched with Clearbit…) |
| Merge Enriched Data | Merge | Merge original + enrichment by index | Enrich with Clearbit; Normalize Lead Data; Validate Required Fields | Calculate Lead Score | ## Enrichment & Scoring (combines original submission and Clearbit data…) |
| Calculate Lead Score | Code | Compute numeric score | Merge Enriched Data | Prepare Lead Record | ## Enrichment & Scoring (assign points based on factors…) |
| Prepare Lead Record | Set | Stamp status/timestamp (currently pass-through) | Calculate Lead Score | Qualified Lead? | ## Enrichment & Scoring (stamps a status and timestamp…) |
| Qualified Lead? | IF | Route based on score threshold | Prepare Lead Record | Build Lead JSON (Qualified); Build Lead JSON (Unqualified) | ## Qualification & Routing (threshold-based IF node…) |
| Build Lead JSON (Qualified) | Code | Serialize qualified lead to JSON file fields | Qualified Lead? (true) | Upload to Box (Qualified) | ## Qualification & Routing (builds a JSON file, stores in Box, posts message…) |
| Upload to Box (Qualified) | Box | Upload JSON to Box | Build Lead JSON (Qualified) | Notify Sales (Qualified) | ## Qualification & Routing (stores it in a dedicated Box folder…) |
| Notify Sales (Qualified) | Telegram | Notify Sales channel | Upload to Box (Qualified) | — | ## Qualification & Routing (qualified leads alert Sales…) |
| Build Lead JSON (Unqualified) | Code | Serialize unqualified lead to JSON file fields | Qualified Lead? (false) | Upload to Box (Unqualified) | ## Qualification & Routing (unqualified leads land in Marketing backlog…) |
| Upload to Box (Unqualified) | Box | Upload JSON to Box | Build Lead JSON (Unqualified) | Notify Marketing (Unqualified) | ## Qualification & Routing (folder separation keeps Box tidy…) |
| Notify Marketing (Unqualified) | Telegram | Notify Marketing channel | Upload to Box (Unqualified) | — | ## Qualification & Routing (unqualified leads alert Marketing…) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **Lead Scoring Pipeline with Telegram and Box**
   - (Optional) Add sticky notes with the same content for operational clarity.

2. **Add the Webhook trigger**
   - Add node: **Webhook**
   - Configure:
     - **HTTP Method:** POST
     - **Path:** `lead-intake`
     - **Response Mode:** Last Node
   - This is your external form endpoint.

3. **Add “Normalize Lead Data” (Set node)**
   - Add node: **Set**
   - Purpose: map your form provider’s keys to consistent keys like:
     - `email`, `company`, `firstName`, `lastName`, etc.
   - Connect: **Lead Form Webhook → Normalize Lead Data**
   - Also plan to feed this into a merge later (see step 7).

4. **Add “Validate Required Fields” (Code node)**
   - Add node: **Code**
   - Paste logic that sets:
     - `validation.valid` true when `email` and `company` are present
     - `validation.message` accordingly
   - Connect: **Normalize Lead Data → Validate Required Fields**

5. **Add “Has Required Fields?” (IF node)**
   - Add node: **IF**
   - Condition (Boolean):
     - Value 1: `={{ $json.validation.valid }}`
     - Operation: **is true**
   - Connect: **Validate Required Fields → Has Required Fields?**

6. **Invalid path Telegram alert**
   - Add node: **Telegram**
   - Operation: send message
   - Text: `❗️ Received invalid lead submission. Reason: {{$json.validation.message}}`
   - Chat ID: `={{ $env.TELEGRAM_MARKETING_CHAT_ID || 'YOUR_CHAT_ID' }}`
   - Connect: **Has Required Fields? (false) → Notify Marketing (Invalid Submission)**
   - **Credentials:** Create Telegram Bot API credentials (bot token).

7. **Add Clearbit enrichment**
   - Add node: **HTTP Request**
   - Method: GET
   - URL: `https://company.clearbit.com/v2/combined/find?email={{$json.email}}`
   - Authentication: **Clearbit API** (predefined credentials)
   - Connect: **Has Required Fields? (true) → Enrich with Clearbit**

8. **Add a Merge node**
   - Add node: **Merge**
   - Mode: **Merge By Index**
   - Connect:
     - **Enrich with Clearbit → Merge (Input 0)**
     - **Normalize Lead Data → Merge (Input 1)** *(original payload path)*
     - (Optional) If you keep the validation fields, connect **Validate Required Fields → Merge (Input 1)**, but avoid double-feeding the same merge input unless you intentionally manage item counts. Prefer a single “original+validation” stream.
   - Goal: obtain one merged object containing original lead + enrichment.

9. **Add “Calculate Lead Score” (Code)**
   - Add node: **Code**
   - Implement scoring and output `leadScore` (number).
   - Connect: **Merge → Calculate Lead Score**
   - If you want to use `data.enrichment...` as in the provided code, ensure the merged structure actually places Clearbit data into an `enrichment` key (e.g., via an intermediate Set node that nests Clearbit response under `enrichment` before merging).

10. **Add “Prepare Lead Record” (Set)**
   - Add node: **Set**
   - Add fields commonly used downstream, for example:
     - `status`: `"new"`
     - `receivedAt`: `={{ $now }}`
   - Connect: **Calculate Lead Score → Prepare Lead Record**

11. **Add “Qualified Lead?” (IF)**
   - Add node: **IF**
   - Numeric condition:
     - Value 1: `={{ $json.leadScore }}`
     - Operation: **larger or equal**
     - Value 2: `70` (adjust as desired)
   - Connect: **Prepare Lead Record → Qualified Lead?**

12. **Qualified branch: build file + upload to Box + Telegram**
   1. Add node: **Code** (“Build Lead JSON (Qualified)”)
      - Create `fileName` and `fileContent` (stringified JSON).
      - Connect: **Qualified Lead? (true) → Build Lead JSON (Qualified)**
   2. Add node: **Box** (“Upload to Box (Qualified)”)
      - Configure upload using:
        - **File Name:** `={{ $json.fileName }}`
        - **File Content:** `={{ $json.fileContent }}`
      - Also set **Folder ID** to `={{ $env.BOX_QUALIFIED_FOLDER_ID }}` (this is implied by the sticky note; ensure the node’s “Parent Folder”/folder parameter is set accordingly).
      - **Credentials:** Box OAuth2
      - Connect: **Build Lead JSON (Qualified) → Upload to Box (Qualified)**
   3. Add node: **Telegram** (“Notify Sales (Qualified)”)
      - Chat ID: `={{ $env.TELEGRAM_SALES_CHAT_ID || 'YOUR_CHAT_ID' }}`
      - Text: `🎉 New qualified lead: {{$json.firstName}} {{$json.lastName}} (Score {{$json.leadScore}}). Check Box for details.`
      - Connect: **Upload to Box (Qualified) → Notify Sales (Qualified)**

13. **Unqualified branch: build file + upload to Box + Telegram**
   1. Add node: **Code** (“Build Lead JSON (Unqualified)”)
      - Similar, with `_unqualified.json`
      - Connect: **Qualified Lead? (false) → Build Lead JSON (Unqualified)**
   2. Add node: **Box** (“Upload to Box (Unqualified)”)
      - Folder ID: `={{ $env.BOX_UNQUALIFIED_FOLDER_ID }}`
      - File Name/Content from expressions
      - Connect: **Build Lead JSON (Unqualified) → Upload to Box (Unqualified)**
   3. Add node: **Telegram** (“Notify Marketing (Unqualified)”)
      - Chat ID: `={{ $env.TELEGRAM_MARKETING_CHAT_ID || 'YOUR_CHAT_ID' }}`
      - Text: `ℹ️ New unqualified lead: {{$json.firstName}} {{$json.lastName}} (Score {{$json.leadScore}}). Stored in Box.`
      - Connect: **Upload to Box (Unqualified) → Notify Marketing (Unqualified)**

14. **Environment variables to set (as referenced by expressions/sticky note)**
   - `TELEGRAM_SALES_CHAT_ID`
   - `TELEGRAM_MARKETING_CHAT_ID`
   - `TELEGRAM_OPS_CHAT_ID` (referenced in notes, but not used by any node currently)
   - `BOX_QUALIFIED_FOLDER_ID`
   - `BOX_UNQUALIFIED_FOLDER_ID`

15. **(Recommended) Add workflow-wide error handling**
   - Add node: **Error Trigger** (called “Workflow Error Trigger” in n8n)
   - Add node: **Telegram** to send to Ops:
     - Chat ID: `={{ $env.TELEGRAM_OPS_CHAT_ID }}`
     - Text summarizing error details from the trigger output
   - This is described in the sticky note but not present in the provided workflow JSON.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Add Clearbit API credential under **Credentials → Clearbit API**. | From sticky note “⚡️ Lead Scoring Overview” |
| Add a **Box OAuth2** credential and share target folders’ IDs as env vars `BOX_QUALIFIED_FOLDER_ID` and `BOX_UNQUALIFIED_FOLDER_ID`. | From sticky note “⚡️ Lead Scoring Overview”; note that the Box nodes as provided do not visibly reference these env vars. |
| Create a Telegram bot and add **Telegram Bot API** credentials; export `TELEGRAM_SALES_CHAT_ID`, `TELEGRAM_MARKETING_CHAT_ID`, `TELEGRAM_OPS_CHAT_ID`. | From sticky note “⚡️ Lead Scoring Overview”; `TELEGRAM_OPS_CHAT_ID` isn’t used unless an error path is added. |
| Adjust the score threshold in the *Qualified Lead?* IF node if required. | From sticky note “⚡️ Lead Scoring Overview” |
| Sticky note describes error handling via “Workflow Error Trigger”, but the workflow JSON contains no such node. | From sticky note “🛠️ Error Handling”; implementation gap to address |