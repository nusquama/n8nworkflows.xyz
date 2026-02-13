Manage Patreon and Ko-fi leads with Gmail, OpenAI and Google Sheets

https://n8nworkflows.xyz/workflows/manage-patreon-and-ko-fi-leads-with-gmail--openai-and-google-sheets-12665


# Manage Patreon and Ko-fi leads with Gmail, OpenAI and Google Sheets

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Manage Patreon and Ko-fi leads with Gmail, OpenAI and Google Sheets

**Purpose / Use cases**
- Receives incoming **webhooks** from **Patreon** and **Ko‑fi** on a single endpoint.
- Identifies the source system via the **User-Agent** header.
- For **Ko‑fi**: parses payload, validates it using a **verification token**, then (optionally) adds the supporter to a **Google Sheets newsletter list**, and sends a **thank-you email** depending on the Ko‑fi transaction type.
- For **Patreon**: checks/creates the contact in the same **newsletter sheet**, then uses **OpenAI** to classify the event into an operational status (new follower/patron, upgrades/downgrades/cancel), and routes to the corresponding **Gmail** message branch.

**Logical blocks**
1.1 **Webhook Reception & Source Routing** (Webhook → Switch)  
1.2 **Ko‑fi Security & Normalization** (Set Token → Parse Kofi Payload → Token Validation)  
1.3 **Ko‑fi Newsletter Sync** (Get Newsletter Subs1 → IF → Append → Merge1)  
1.4 **Ko‑fi Transaction Type Routing & Emails** (Switch1 → Thank you Letter*)  
1.5 **Patreon Newsletter Sync** (Get Newsletter Subs → IF → Append → Merge)  
1.6 **Patreon AI Classification & Status Routing** (Merge → Message a model → Switch2 → Send a message*)  
1.7 **Alerting** (Fake Payload Alert)

---

## 2. Block-by-Block Analysis

### 1.1 Webhook Reception & Source Routing

**Overview:** Receives POST requests and splits execution into Patreon vs Ko‑fi branches based on the `user-agent` header.

**Nodes involved:** `Webhook`, `Switch`

#### Node: Webhook
- **Type / Role:** `n8n-nodes-base.webhook` — entry point HTTP listener.
- **Configuration (interpreted):**
  - Method: **POST**
  - Path: `e180c04c-2d55-44b8-8f82-800cad5ac10e` (forms the webhook URL).
- **Key data used later:**
  - Patreon payload: `$json.body.data` (full object)
  - Patreon event header: `$json.headers['x-patreon-event']`
  - Source identification: `$json.headers['user-agent']`
- **Outputs / Connections:** Main output → `Switch`
- **Edge cases / failures:**
  - Missing/changed headers (User-Agent not containing expected substring).
  - Different Patreon event header name or absent `x-patreon-event`.
  - If payload structure differs (e.g., Patreon v2 changes), downstream expressions can fail.

#### Node: Switch
- **Type / Role:** `n8n-nodes-base.switch` — routes by conditions.
- **Configuration choices:**
  - Rules check `={{ $json.headers['user-agent'] }}`:
    - Contains `"Patreon"` → output key **patreon**
    - Contains `"Kofi"` → output key **kofi**
- **Outputs / Connections:**
  - Output 0 (patreon) → `Get Newsletter Subs`
  - Output 1 (kofi) → `Set Token`
- **Edge cases:**
  - Ko‑fi User-Agent may not contain “Kofi” exactly; real-world strings can be “Ko-fi”.
  - Requests not matching any rule will effectively be dropped (no default branch configured).

**Sticky note context (applies to this block):**
- **“Patreon & Ko-fi Webhook Processor”** (setup/customization checklist; see Section 5).

---

### 1.2 Ko‑fi Security & Normalization

**Overview:** Normalizes Ko‑fi’s nested JSON payload, then validates authenticity by comparing the payload’s `verification_token` against a token stored in the workflow.

**Nodes involved:** `Set Token`, `Parse Kofi Payload`, `Token Validation`, `Fake Payload Alert`

#### Node: Set Token
- **Type / Role:** `n8n-nodes-base.set` — stores a constant used for validation.
- **Configuration choices:**
  - Sets field `token` (string). In the provided JSON it is empty, but in pinned data it’s populated.
- **Key variables:**
  - `$('Set Token').item.json.token`
- **Outputs / Connections:** → `Parse Kofi Payload`
- **Edge cases:**
  - If left blank, all validations will fail and trigger the fake payload branch.
  - Storing secrets directly in node parameters may be undesirable; consider using n8n credentials or environment variables.

**Sticky note:** “Set Token — Get an auth token from Kofi to validate your payloads.”

#### Node: Parse Kofi Payload
- **Type / Role:** `n8n-nodes-base.code` — parses nested JSON string into an object.
- **Configuration choices:**
  - Reads `$('Webhook').first().json.body.data` (expects a **JSON string** inside `body.data`)
  - `JSON.parse(input)` inside try/catch; returns `{}` if invalid.
  - Returns an array with one item `{ json: parsed }` (n8n Code node requirement).
- **Inputs / Outputs:**
  - Input connection from `Set Token` (but it explicitly pulls data from `Webhook` via selector).
  - Output → `Token Validation`
- **Edge cases / failures:**
  - If Ko‑fi changes to send `body.data` as an object instead of a string, `JSON.parse()` will throw.
  - If `body.data` is missing, `input` becomes undefined, parse fails, outputs `{}` which will fail token validation and/or downstream expressions (like `type`, `email`).

#### Node: Token Validation
- **Type / Role:** `n8n-nodes-base.if` — gates Ko‑fi branch based on token match.
- **Condition:**
  - `={{ $json.verification_token }} == {{ $('Set Token').item.json.token }}`
  - Left side refers to **current item** (output of Parse Kofi Payload).
- **Outputs / Connections:**
  - **True** → `Get Newsletter Subs1`
  - **False** → `Fake Payload Alert`
- **Edge cases:**
  - If Parse produced `{}`, `$json.verification_token` is undefined → condition false → alert is sent (good fail-closed behavior).

#### Node: Fake Payload Alert
- **Type / Role:** `n8n-nodes-base.gmail` — sends an alert email when validation fails.
- **Configuration choices:**
  - To: `user@example.com`
  - Subject: `fake payload detected!`
  - Body: `Someone just sent a fake payload to your Kofi webhook.`
- **Credentials:** Gmail OAuth2 credential “Gmail account”
- **Outputs / Connections:** none
- **Edge cases / failures:**
  - Gmail OAuth token expiry/invalid scope.
  - Rate limits if attacked (many fake requests).

---

### 1.3 Ko‑fi Newsletter Sync

**Overview:** Checks whether the Ko‑fi supporter email exists in Google Sheets; if not, appends them. Then merges the branch back for downstream routing.

**Nodes involved:** `Get Newsletter Subs1`, `Person Exists on the List?1`, `Add New Sub to Newsletter1`, `Merge1`

#### Node: Get Newsletter Subs1
- **Type / Role:** `n8n-nodes-base.googleSheets` — lookup rows by email.
- **Configuration choices:**
  - Document: Spreadsheet “Newsletter” (`1uaPJWmAa8yEUPHad6OvDkoqLrahzZORDFjExyxR3064`)
  - Sheet: “Sheet1” (`gid=0`)
  - Filter: lookupColumn `email`, lookupValue `={{ $json.email }}`
  - `alwaysOutputData: true` (important so downstream IF runs even if no rows found)
- **Outputs / Connections:** → `Person Exists on the List?1`
- **Edge cases:**
  - If `email` is missing from Ko‑fi payload, lookup is invalid and may return empty or error depending on node behavior.
  - Sheets API quota / permission errors.

#### Node: Person Exists on the List?1
- **Type / Role:** `n8n-nodes-base.if` — decides whether to append.
- **Condition (as configured):**
  - Checks `={{ $json.isEmpty() }}` is **true**
  - Interpretation: If the Google Sheets node returned an “empty” result, treat as “not found”.
- **Outputs / Connections:**
  - **True** (empty → not found) → `Add New Sub to Newsletter1`
  - **False** → `Merge1` (skip append)
- **Edge cases:**
  - `isEmpty()` depends on the exact object returned by the Sheets node. If the node returns an array of rows, `isEmpty()` behavior may differ across versions. Test in your environment.

#### Node: Add New Sub to Newsletter1
- **Type / Role:** `n8n-nodes-base.googleSheets` — append new subscriber row.
- **Configuration choices:**
  - Operation: **append**
  - Columns mapping:
    - `name = {{ $('Parse Kofi Payload').item.json.from_name }}`
    - `email = {{ $('Parse Kofi Payload').item.json.email }}`
- **Outputs / Connections:** → `Merge1`
- **Edge cases:**
  - If email already exists (race condition), duplicates can occur. Consider using “Upsert” pattern if required.

#### Node: Merge1
- **Type / Role:** `n8n-nodes-base.merge` — re-joins “added” and “skipped” paths.
- **Configuration choices:** Default merge behavior (no explicit mode shown).
- **Outputs / Connections:** → `Switch1`
- **Edge cases:**
  - If merge mode expects matching items, it may behave unexpectedly. In many webhook flows, “Pass-through” is safer. Verify merge mode in UI.

**Sticky note (covers this block):**
- “Newsletter — check if person is on list; create spreadsheet with columns email and name…”

---

### 1.4 Ko‑fi Transaction Type Routing & Emails

**Overview:** Routes Ko‑fi events by `type` (Donation/Subscription/Shop Order) and sends a thank-you email.

**Nodes involved:** `Switch1`, `Thank you Letter`, `Thank you Letter1`, `Thank you Letter2`

#### Node: Switch1
- **Type / Role:** `n8n-nodes-base.switch` — routes by Ko‑fi `type`.
- **Rules:**
  - `$('Parse Kofi Payload').item.json.type == "Donation"` → **Donation**
  - `... == "Subscription"` → **Subscription**
  - `... == "Shop Order"` → **Shop Order**
- **Outputs / Connections:**
  - Donation → `Thank you Letter`
  - Subscription → `Thank you Letter1`
  - Shop Order → `Thank you Letter2`
- **Edge cases:**
  - If Parse failed or `type` missing, no branch is taken.
  - Ko‑fi may use different type casing or values; keep list aligned with Ko‑fi docs.

#### Node: Thank you Letter / Thank you Letter1 / Thank you Letter2
- **Type / Role:** `n8n-nodes-base.gmail` — sends thank-you emails (one per type).
- **Configuration choices (all three):**
  - To: `={{ $json.email }}`
  - Subject: `=Thanks!` (note: leading `=` is unnecessary; harmless but can confuse)
  - Body: “Say how happy you are.”
  - Email type: text
- **Credentials:** Gmail OAuth2 “Gmail account”
- **Edge cases:**
  - Missing `email` field in Ko‑fi payload will cause send failure.
  - Gmail sending limits / spam reputation.

**Sticky note (covers this block):**
- “Letters — Set up letters that you're gonna send to members.”

---

### 1.5 Patreon Newsletter Sync

**Overview:** For Patreon events, ensures the user is present in the newsletter Google Sheet, then continues to AI classification.

**Nodes involved:** `Get Newsletter Subs`, `Person Exists on the List?`, `Add New Sub to Newsletter`, `Merge`

#### Node: Get Newsletter Subs
- **Type / Role:** `n8n-nodes-base.googleSheets` — lookup by Patreon email.
- **Configuration choices:**
  - Filter: lookupColumn `email`, lookupValue `={{ $json.body.data.attributes.email }}`
  - Same spreadsheet/sheet as Ko‑fi flow.
  - `alwaysOutputData: true`
- **Outputs / Connections:** → `Person Exists on the List?`
- **Edge cases:**
  - Patreon can omit email depending on permissions/settings; expression may be undefined.

#### Node: Person Exists on the List?
- **Type / Role:** `n8n-nodes-base.if`
- **Condition:**
  - `={{ $json.isEmpty() }}` is true → treat as “not found”
- **Outputs / Connections:**
  - True → `Add New Sub to Newsletter`
  - False → `Merge` (skip append)
- **Edge cases:** same `isEmpty()` considerations as Ko‑fi IF node.

#### Node: Add New Sub to Newsletter
- **Type / Role:** `n8n-nodes-base.googleSheets` append.
- **Columns mapping:**
  - `name = {{ $('Webhook').item.json.body.data.attributes.full_name }}`
  - `email = {{ $('Webhook').item.json.body.data.attributes.email }}`
- **Outputs / Connections:** → `Merge`

#### Node: Merge
- **Type / Role:** `n8n-nodes-base.merge` — rejoins after optional append.
- **Outputs / Connections:** → `Message a model`
- **Edge cases:** merge-mode caveat as above.

**Sticky note (covers this block):**
- “Newsletter — … Create a ‘Newsletter’ spreadsheet with columns ‘email’ and ‘name’…”

---

### 1.6 Patreon AI Classification & Status Routing

**Overview:** Uses OpenAI to classify Patreon webhooks into one of eight statuses using `x-patreon-event` as an authoritative signal, then routes to different Gmail messages.

**Nodes involved:** `Message a model`, `Switch2`, `Send a message`, `Send a message1` … `Send a message7`

#### Node: Message a model
- **Type / Role:** `@n8n/n8n-nodes-langchain.openAi` — LLM call for classification.
- **Configuration choices:**
  - Model: `gpt-4.1-mini`
  - Response format: JSON object (`textFormat.type = json_object`)
  - Prompting:
    - System message contains strict classification rules:
      - `event_type` from header overrides inference
      - Explicit mapping constraints for `members:update`, `members:delete`, `members:create`
      - Output must be `{ "status": "<...>" }`
    - User content passes:
      - Full payload: `={{ JSON.stringify($('Webhook').item.json.body.data) }}`
      - `event_type: {{ $('Webhook').item.json.headers['x-patreon-event'] }}`
- **Outputs / Connections:** → `Switch2`
- **Edge cases / failures:**
  - OpenAI credential missing, model not available, rate limits, timeouts.
  - If OpenAI returns invalid JSON, downstream `Switch2` expressions referencing `content[0].text.status` can fail.
  - Note: pinned data shows `x-patreon-event` like `members:pledge:delete` while the system prompt expects `members:delete` / `members:update` / `members:create`. This mismatch can cause misclassification unless the model generalizes. Consider normalizing event types before the LLM.

**Sticky note:** “Status Validation — We use OpenAI to determine the status of operations”.

#### Node: Switch2
- **Type / Role:** `n8n-nodes-base.switch` — routes by AI status.
- **Rules:** checks `={{ $json.output[0].content[0].text.status }}` equals:
  - `brand_new_follower`
  - `brand_new_patron_one_time`
  - `brand_new_patron_subscription`
  - `follower_to_one_time`
  - `follower_to_subscription`
  - `subscription_upgraded`
  - `subscription_downgraded`
  - `subscription_cancelled`
- **Outputs / Connections:**
  - brand_new_follower → `Send a message`
  - brand_new_patron_one_time → `Send a message1`
  - brand_new_patron_subscription → `Send a message2`
  - follower_to_one_time → `Send a message7`
  - follower_to_subscription → `Send a message3`
  - subscription_upgraded → `Send a message4`
  - subscription_downgraded → `Send a message5`
  - subscription_cancelled → `Send a message6`
- **Edge cases:**
  - If OpenAI output path differs (node version changes), the expression path may break.
  - If status has unexpected value, nothing happens (no default case).

#### Nodes: Send a message / Send a message1 … Send a message7
- **Type / Role:** `n8n-nodes-base.gmail` — sends an email per Patreon status.
- **Configuration choices:** In provided JSON, parameters are empty excepts `options: {}` (so these are placeholders).
- **Credentials:** Gmail OAuth2 “Gmail account”
- **Outputs:** none
- **Edge cases:**
  - Because recipients/subject/body are not configured, these nodes as-is won’t send meaningful emails.
  - Gmail auth/rate limits as above.

**Sticky note (covers these Gmail nodes):**
- “Letters — Set up letters that you're gonna send to members.”

---

### 1.7 Alerting (Ko‑fi invalid payload)

**Overview:** Sends an alert email when the Ko‑fi verification token does not match.

**Nodes involved:** `Fake Payload Alert` (already covered in 1.2)

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook | Webhook | Receive Patreon/Ko‑fi POST webhooks | — | Switch | ## Patreon & Ko-fi Webhook Processor / About, Setup, Customization (see note content in Section 5) |
| Switch | Switch | Route to Patreon vs Ko‑fi by User-Agent | Webhook | Get Newsletter Subs; Set Token | ## Patreon & Ko-fi Webhook Processor / About, Setup, Customization |
| Set Token | Set | Store Ko‑fi verification token constant | Switch (kofi) | Parse Kofi Payload | ## Set Token\nGet an auth token from Kofi to validate your payloads. |
| Parse Kofi Payload | Code | Parse Ko‑fi nested JSON string into object | Set Token | Token Validation | ## Patreon & Ko-fi Webhook Processor / About, Setup, Customization |
| Token Validation | IF | Validate `verification_token` matches stored token | Parse Kofi Payload | Get Newsletter Subs1 (true); Fake Payload Alert (false) | ## Patreon & Ko-fi Webhook Processor / About, Setup, Customization |
| Fake Payload Alert | Gmail | Alert email on invalid Ko‑fi payload | Token Validation (false) | — | ## Patreon & Ko-fi Webhook Processor / About, Setup, Customization |
| Get Newsletter Subs1 | Google Sheets | Ko‑fi: lookup subscriber by email | Token Validation (true) | Person Exists on the List?1 | ## Newsletter\nWe check whether person is on our newsletter list or not... |
| Person Exists on the List?1 | IF | Ko‑fi: decide whether to append | Get Newsletter Subs1 | Add New Sub to Newsletter1 (true); Merge1 (false) | ## Newsletter\nWe check whether person is on our newsletter list or not... |
| Add New Sub to Newsletter1 | Google Sheets | Ko‑fi: append subscriber to sheet | Person Exists on the List?1 (true) | Merge1 | ## Newsletter\nWe check whether person is on our newsletter list or not... |
| Merge1 | Merge | Ko‑fi: rejoin after optional append | Add New Sub to Newsletter1; Person Exists on the List?1 (false) | Switch1 | ## Newsletter\nWe check whether person is on our newsletter list or not... |
| Switch1 | Switch | Ko‑fi: route by transaction type | Merge1 | Thank you Letter; Thank you Letter1; Thank you Letter2 | ## Letters\nSet up letters that you're gonna send to members. |
| Thank you Letter | Gmail | Ko‑fi donation thank-you email | Switch1 (Donation) | — | ## Letters\nSet up letters that you're gonna send to members. |
| Thank you Letter1 | Gmail | Ko‑fi subscription thank-you email | Switch1 (Subscription) | — | ## Letters\nSet up letters that you're gonna send to members. |
| Thank you Letter2 | Gmail | Ko‑fi shop order thank-you email | Switch1 (Shop Order) | — | ## Letters\nSet up letters that you're gonna send to members. |
| Get Newsletter Subs | Google Sheets | Patreon: lookup subscriber by email | Switch (patreon) | Person Exists on the List? | ## Newsletter\nWe check whether person is on our newsletter list or not... |
| Person Exists on the List? | IF | Patreon: decide whether to append | Get Newsletter Subs | Add New Sub to Newsletter (true); Merge (false) | ## Newsletter\nWe check whether person is on our newsletter list or not... |
| Add New Sub to Newsletter | Google Sheets | Patreon: append subscriber to sheet | Person Exists on the List? (true) | Merge | ## Newsletter\nWe check whether person is on our newsletter list or not... |
| Merge | Merge | Patreon: rejoin after optional append | Add New Sub to Newsletter; Person Exists on the List? (false) | Message a model | ## Newsletter\nWe check whether person is on our newsletter list or not... |
| Message a model | OpenAI (LangChain) | Classify Patreon event to status | Merge | Switch2 | ## Status Validation\nWe use OpenAI to determine the status of operations |
| Switch2 | Switch | Patreon: route by AI status | Message a model | Send a message; Send a message1; Send a message2; Send a message7; Send a message3; Send a message4; Send a message5; Send a message6 | ## Letters\nSet up letters that you're gonna send to members. |
| Send a message | Gmail | Patreon: email for brand_new_follower | Switch2 | — | ## Letters\nSet up letters that you're gonna send to members. |
| Send a message1 | Gmail | Patreon: email for brand_new_patron_one_time | Switch2 | — | ## Letters\nSet up letters that you're gonna send to members. |
| Send a message2 | Gmail | Patreon: email for brand_new_patron_subscription | Switch2 | — | ## Letters\nSet up letters that you're gonna send to members. |
| Send a message7 | Gmail | Patreon: email for follower_to_one_time | Switch2 | — | ## Letters\nSet up letters that you're gonna send to members. |
| Send a message3 | Gmail | Patreon: email for follower_to_subscription | Switch2 | — | ## Letters\nSet up letters that you're gonna send to members. |
| Send a message4 | Gmail | Patreon: email for subscription_upgraded | Switch2 | — | ## Letters\nSet up letters that you're gonna send to members. |
| Send a message5 | Gmail | Patreon: email for subscription_downgraded | Switch2 | — | ## Letters\nSet up letters that you're gonna send to members. |
| Send a message6 | Gmail | Patreon: email for subscription_cancelled | Switch2 | — | ## Letters\nSet up letters that you're gonna send to members. |
| Sticky Note | Sticky Note | Documentation/comment | — | — |  |
| Sticky Note1 | Sticky Note | Documentation/comment | — | — |  |
| Sticky Note2 | Sticky Note | Documentation/comment | — | — |  |
| Sticky Note3 | Sticky Note | Documentation/comment | — | — |  |
| Sticky Note4 | Sticky Note | Documentation/comment | — | — |  |
| Sticky Note5 | Sticky Note | Documentation/comment | — | — |  |
| Sticky Note6 | Sticky Note | Documentation/comment | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** in n8n.

2) **Add Webhook node**
   - Node type: **Webhook**
   - Method: **POST**
   - Path: generate or set a unique path (store it; you’ll paste this URL into Patreon and Ko‑fi settings).
   - Activate “Test” URL for initial testing.

3) **Add Switch node (source router)**
   - Node type: **Switch**
   - Create 2 rules on string field:
     - Rule 1 output name: `patreon`
       - Value: `={{ $json.headers['user-agent'] }}`
       - Condition: **contains** `Patreon`
     - Rule 2 output name: `kofi`
       - Value: `={{ $json.headers['user-agent'] }}`
       - Condition: **contains** `Kofi` (adjust to `Ko-fi` if needed)
   - Connect: **Webhook → Switch**

---

### Ko‑fi branch

4) **Add Set node (“Set Token”)**
   - Node type: **Set**
   - Add field:
     - `token` (String) = your **Ko‑fi verification token**
   - Connect: **Switch (kofi) → Set Token**

5) **Add Code node (“Parse Kofi Payload”)**
   - Node type: **Code**
   - Paste logic that:
     - Reads `$('Webhook').first().json.body.data`
     - `JSON.parse()` it
     - Returns `[ { json: parsed } ]`
   - Connect: **Set Token → Parse Kofi Payload**

6) **Add IF node (“Token Validation”)**
   - Node type: **IF**
   - Condition (string equals):
     - Left: `={{ $json.verification_token }}`
     - Right: `={{ $('Set Token').item.json.token }}`
   - Connect: **Parse Kofi Payload → Token Validation**

7) **Add Gmail node (“Fake Payload Alert”)**
   - Node type: **Gmail** → Send
   - Configure:
     - To: your admin email
     - Subject/body as desired
   - Credentials:
     - Create/attach **Gmail OAuth2** credential
   - Connect: **Token Validation (false) → Fake Payload Alert**

8) **Newsletter lookup (Ko‑fi)**
   - Add Google Sheets node (“Get Newsletter Subs1”)
     - Operation: **Read / Lookup** (use “Filters” UI)
     - Spreadsheet: create/select **Newsletter**
     - Sheet: e.g. **Sheet1**
     - Filter: column `email`, value `={{ $json.email }}`
     - Enable **Always Output Data**
   - Connect: **Token Validation (true) → Get Newsletter Subs1**

9) **Add IF node (“Person Exists on the List?1”)**
   - Condition: `={{ $json.isEmpty() }}` is true (meaning not found)
   - Connect: **Get Newsletter Subs1 → Person Exists on the List?1**

10) **Add Google Sheets append (“Add New Sub to Newsletter1”)**
   - Operation: **Append**
   - Map:
     - `name = {{ $('Parse Kofi Payload').item.json.from_name }}`
     - `email = {{ $('Parse Kofi Payload').item.json.email }}`
   - Connect: **Person Exists…1 (true) → Add New Sub…1**

11) **Add Merge node (“Merge1”)**
   - Connect:
     - **Add New Sub…1 → Merge1**
     - **Person Exists…1 (false) → Merge1**
   - Verify merge mode in UI so the flow continues with one item.

12) **Add Switch node (“Switch1”) for Ko‑fi type**
   - Rules on: `={{ $('Parse Kofi Payload').item.json.type }}`
     - equals Donation → output “Donation”
     - equals Subscription → output “Subscription”
     - equals Shop Order → output “Shop Order”
   - Connect: **Merge1 → Switch1**

13) **Add 3 Gmail nodes for thank-you emails**
   - “Thank you Letter” (Donation), “Thank you Letter1” (Subscription), “Thank you Letter2” (Shop Order)
   - Configure each:
     - To: `={{ $json.email }}`
     - Subject/body specific per type
   - Connect outputs of **Switch1** to each Gmail node.

---

### Patreon branch

14) **Add Google Sheets node (“Get Newsletter Subs”)**
   - Filter by `email = {{ $json.body.data.attributes.email }}`
   - Enable **Always Output Data**
   - Connect: **Switch (patreon) → Get Newsletter Subs**

15) **Add IF node (“Person Exists on the List?”)**
   - Condition: `={{ $json.isEmpty() }}` is true → append
   - Connect: **Get Newsletter Subs → Person Exists on the List?**

16) **Add Google Sheets append (“Add New Sub to Newsletter”)**
   - Map:
     - `name = {{ $('Webhook').item.json.body.data.attributes.full_name }}`
     - `email = {{ $('Webhook').item.json.body.data.attributes.email }}`
   - Connect: **IF (true) → Add New Sub to Newsletter**

17) **Add Merge node (“Merge”)**
   - Connect:
     - **Add New Sub to Newsletter → Merge**
     - **IF (false) → Merge**

18) **Add OpenAI node (“Message a model”)**
   - Node type: **OpenAI (LangChain) / Message a model**
   - Credentials:
     - Create/attach **OpenAI API** credential
   - Model: `gpt-4.1-mini` (or your preferred)
   - Response format: JSON object
   - System prompt: include strict rules + required JSON output `{ "status": "..." }`
   - User message:
     - Payload: `{{ JSON.stringify($('Webhook').item.json.body.data) }}`
     - Event type: `{{ $('Webhook').item.json.headers['x-patreon-event'] }}`
   - Connect: **Merge → Message a model**

19) **Add Switch node (“Switch2”) for status**
   - Evaluate: `={{ $json.output[0].content[0].text.status }}`
   - Add eight equals rules (one per status).
   - Connect: **Message a model → Switch2**

20) **Add 8 Gmail nodes (“Send a message…”)**
   - One per status branch (brand new, follower conversions, upgrade/downgrade/cancel)
   - Configure recipients and content (these are placeholders in the provided workflow).
   - Connect each Switch2 output to the corresponding Gmail node.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| **Patreon & Ko-fi Webhook Processor** — About/Setup/Customization checklist: connect Gmail, Google Sheets, OpenAI; set Ko‑fi token; configure Patreon webhook and forwarded `x-patreon-event`; customize email branches; optional CRM enhancements. | Sticky note content in workflow canvas |
| **Newsletter** — Create “Newsletter” spreadsheet with columns **email** and **name**; workflow checks if user exists then appends if missing (both Patreon and Ko‑fi). | Sticky note content in workflow canvas |
| **Status Validation** — OpenAI is used to determine Patreon status. | Sticky note content in workflow canvas |
| **Letters** — Configure the actual email content for each branch (Ko‑fi types and Patreon statuses). | Sticky note content in workflow canvas |