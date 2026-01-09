Validate & score leads with ZeroBounce AI before sending Gmail emails

https://n8nworkflows.xyz/workflows/validate---score-leads-with-zerobounce-ai-before-sending-gmail-emails-11902


# Validate & score leads with ZeroBounce AI before sending Gmail emails

## 1. Workflow Overview

**Title:** Validate & score leads with ZeroBounce AI before sending Gmail emails

**Purpose:**  
This workflow monitors a Google Sheet containing lead emails, validates each email with ZeroBounce, then (if valid) scores lead quality using ZeroBounce AI Scoring. Only **high-scoring leads (≥ 9)** are emailed via Gmail. The workflow writes all outcomes (validation details, score, emailed status, reason) back to Google Sheets to prevent re-processing and protect sender reputation.

**Primary use cases:**
- Lead list hygiene before outreach (reduce bounces, protect deliverability)
- Automated qualification of contacts using AI scoring
- Spreadsheet-driven outreach with audit trail (Validated/Scored/Emailed flags)

### Logical blocks
**1.1 Sheet Monitoring & Dedup Guards**
- Trigger on sheet updates and check whether validation/scoring/emailing has already happened.

**1.2 ZeroBounce Email Validation + Sheet Update**
- Validate email, store validation metadata back to the sheet, then decide whether it’s valid.

**1.3 ZeroBounce AI Scoring + Sheet Update**
- If valid, score the email, store score in a (separate) sheet, then route by score bucket.

**1.4 Outreach Decision + Gmail Send + Final Sheet Update**
- High score → send Gmail message → mark emailed true; otherwise mark emailed false with reason.

---

## 2. Block-by-Block Analysis

### 2.1 Sheet Monitoring & Dedup Guards

**Overview:**  
Polls a Google Sheet every minute for new/updated rows, then uses flags in the row (`Validated`, `Scored`, `Emailed`) to avoid repeating steps for the same contact.

**Nodes involved:**
- Google Sheets Trigger
- Validate?
- Score?
- Email?

#### Node: Google Sheets Trigger
- **Type / role:** `n8n-nodes-base.googleSheetsTrigger` — entry point; polls a sheet for changes.
- **Configuration (interpreted):**
  - Poll frequency: **every minute**
  - Document: “Emails” (Google Sheet ID present)
  - Sheet/tab: “Sheet1” (`gid=0`)
- **Key data produced:** Each triggered item represents a row, expected to include at least `Email`, and boolean-ish flags like `Validated`, `Scored`, `Emailed`.
- **Outputs:** Connected to **Validate?**
- **Failure modes / edge cases:**
  - OAuth permission issues (token expired / missing scopes)
  - Polling can re-deliver updated rows; flags are essential to prevent duplicate outreach
  - Sheet schema mismatches (missing `Email` column) break downstream expressions

#### Node: Validate?
- **Type / role:** `n8n-nodes-base.if` — gate: only validate if not already validated.
- **Condition logic:**  
  - `{{$json.Validated}} != true` (loose validation enabled)
- **True output:** Validate email  
- **False output:** Email is valid? (note: this is a design choice; see edge cases)
- **Edge cases / risks:**
  - If `Validated` is stored as the string `"true"` in Sheets, loose boolean handling may misbehave depending on how Sheets returns values.
  - **Potential logic hazard:** When `Validated` is already true, the workflow jumps directly to **Email is valid?**, which expects a field `ZB Status` to exist in the current item. That field is only guaranteed if validation results were written back *and* the current trigger payload includes it. If the trigger payload does not include `ZB Status`, the check may fail and route incorrectly.

#### Node: Score?
- **Type / role:** `n8n-nodes-base.if` — gate: only score if not already scored.
- **Condition logic:**  
  - `{{$json.Scored}} != true` (loose type validation)
- **True output:** Score email  
- **False output:** Filter by score
- **Edge cases / risks:**
  - Same string/boolean issue as above for `Scored`.
  - If scoring was done earlier but the current item doesn’t contain `Score`, routing to “Filter by score” can fail (missing/empty score).

#### Node: Email?
- **Type / role:** `n8n-nodes-base.if` — gate: only send email if not already emailed.
- **Condition logic:**  
  - `{{ $('Google Sheets Trigger').item.json.Emailed }} != true`
  - Uses the trigger item explicitly (not the current item), ensuring it checks the original row.
- **True output:** Send a message  
- **False output:** (unused)
- **Edge cases / risks:**
  - If downstream nodes change item structure, referencing the trigger item can be safer—but it also assumes the trigger item has `Emailed`.
  - If `Emailed` is missing/blank, condition evaluates as “not equals true” → may send.

---

### 2.2 ZeroBounce Email Validation + Sheet Update

**Overview:**  
Validates the lead email using ZeroBounce and writes detailed validation metadata back to the “Emails” spreadsheet, then checks whether the email is “valid”.

**Nodes involved:**
- Validate email
- Add validation results
- Email is valid?

#### Node: Validate email
- **Type / role:** `@zerobounce/n8n-nodes-zerobounce.zeroBounce` — calls ZeroBounce validation endpoint.
- **Configuration (interpreted):**
  - Email: `{{ $('Google Sheets Trigger').item.json.Email }}`
  - Timeout: 10 seconds
- **Output shape (typical):**
  - Fields like `address`, `status`, `sub_status`, `domain`, `account`, `mx_found`, `mx_record`, `free_email`, `did_you_mean`, `processed_at`, geo fields, etc.
- **Outputs:** Add validation results
- **Failure modes / edge cases:**
  - Invalid/missing email field
  - ZeroBounce API key issues / quota exceeded
  - Timeout (10s) on slow responses; can cause workflow errors unless error handling is configured globally

#### Node: Add validation results
- **Type / role:** `n8n-nodes-base.googleSheets` — updates the lead row with validation details.
- **Configuration (interpreted):**
  - Operation: **Update**
  - Match row by column: `Email`
  - Values written (core ones):
    - `Email = {{$json.address}}`
    - `Validated = "true"`
    - `ZB Status = {{$json.status}}`
    - `ZB Sub Status = {{$json.sub_status}}`
    - plus: domain/account/geo, MX info, provider, free_email, did_you_mean, processed_at, domain_age_days…
  - Document: “Emails”; Sheet: “Sheet1”
- **Input:** From Validate email
- **Output:** Email is valid?
- **Failure modes / edge cases:**
  - If the sheet’s `Email` value differs from `$json.address` (case/whitespace), matching may fail and update 0 rows.
  - Columns must exist with exact names; otherwise update may fail or silently omit fields (depends on node behavior/version).
  - Writes `"true"` as a string, not boolean—this impacts the IF checks.

#### Node: Email is valid?
- **Type / role:** `n8n-nodes-base.if` — routes valid vs invalid.
- **Condition logic:**  
  - `{{ $json['ZB Status'] }} == "valid"` (strict string equals)
- **True output:** Score?  
- **False output:** Not valid
- **Edge cases / risks:**
  - This node expects `ZB Status` in the incoming item. If the upstream item does not include it (e.g., due to earlier “Validate?” false path), it will evaluate as not equal and treat as invalid.

---

### 2.3 ZeroBounce AI Scoring + Sheet Update

**Overview:**  
Scores valid emails (0–10) via ZeroBounce “scoring” resource, stores score results into a separate Google Sheet (“ZeroBounce”), then routes based on score thresholds.

**Nodes involved:**
- Score email
- Add Scoring Results
- Filter by score

#### Node: Score email
- **Type / role:** `@zerobounce/n8n-nodes-zerobounce.zeroBounce` — calls ZeroBounce AI scoring.
- **Configuration (interpreted):**
  - Resource: **scoring**
  - Email: `{{ $json.Email }}`
- **Input expectations:** The current item must have `Email`. (After validation update, the item typically contains sheet fields including `Email`.)
- **Output shape (typical):**
  - `email`, `score` (numeric), and potentially additional scoring context depending on API/node.
- **Output:** Add Scoring Results
- **Failure modes / edge cases:**
  - If `Email` field is absent (or renamed), scoring fails.
  - API quota/auth errors.

#### Node: Add Scoring Results
- **Type / role:** `n8n-nodes-base.googleSheets` — updates score fields in a sheet.
- **Configuration (interpreted):**
  - Operation: **Update**
  - Match by: `Email`
  - Writes:
    - `Email = {{$json.email}}`
    - `Score = {{$json.score.toString()}}` (stored as string)
    - `Scored = "true"`
  - Document: “ZeroBounce”; Sheet: “Sheet1”
- **Input:** Score email
- **Output:** Filter by score
- **Important integration note:** This writes scoring into a *different spreadsheet* (“ZeroBounce”), not the original “Emails” sheet. That may be intentional (separate scoring log) but can break later steps if they expect `Score` in the trigger sheet.
- **Failure modes / edge cases:**
  - Same “true as string” issue for `Scored`
  - If the “ZeroBounce” sheet does not contain matching Email rows, update will not find a row unless the node supports upsert (not indicated here).

#### Node: Filter by score
- **Type / role:** `n8n-nodes-base.switch` — routes into low/medium/high buckets.
- **Rules (in order):**
  - **low:** `{{$json.Score}} < 3`
  - **medium:** `{{$json.Score}} < 9`
  - **high:** `{{$json.Score}} >= 9`
  - Fallback output: `none`
- **Outputs:**
  - low → Low score (Set)
  - medium → Medium score (Set)
  - high → Email? (IF gate before sending)
- **Edge cases / risks:**
  - `Score` is compared numerically but is often written as a string (`toString()` earlier). Loose validation is enabled, which helps, but empty/missing score can route to fallback.
  - The “medium” rule is `< 9` and comes after `< 3`; this yields:
    - 0–2.999 → low
    - 3–8.999 → medium
    - ≥ 9 → high

---

### 2.4 Outreach Decision + Gmail Send + Final Sheet Update

**Overview:**  
For invalid or low/medium scores, mark the record as not emailed with a reason. For high scores, send a Gmail email and mark as emailed.

**Nodes involved:**
- Not valid
- Low score
- Medium score
- Add Emailed = false
- Email?
- Send a message
- Add Emailed = true

#### Node: Not valid
- **Type / role:** `n8n-nodes-base.set` — annotate item with a reason for not emailing.
- **Configuration:**
  - Adds field `reason = {{$json['ZB Status']}}`
  - Keeps other fields (`includeOtherFields: true`)
- **Output:** Add Emailed = false
- **Edge cases:**
  - If `ZB Status` missing, reason becomes empty.

#### Node: Low score
- **Type / role:** `n8n-nodes-base.set`
- **Configuration:** `reason = "Low score"`
- **Output:** Add Emailed = false

#### Node: Medium score
- **Type / role:** `n8n-nodes-base.set`
- **Configuration:** `reason = "Low score"` (note: label says “Medium score” but reason text is “Low score”)
- **Output:** Add Emailed = false
- **Potential bug:** You probably want reason = “Medium score” here.

#### Node: Add Emailed = false
- **Type / role:** `n8n-nodes-base.googleSheets` — records that outreach did not happen.
- **Configuration (interpreted):**
  - Operation: **Update**
  - Match by: `Email`
  - Writes:
    - `Email = {{$json.Email}}`
    - `Reason = {{$json.reason}}`
    - `Emailed = "false"`
  - Document: “ZeroBounce”; Sheet: “Sheet1”
- **Input:** from Not valid / Low score / Medium score
- **Output:** none (end)
- **Important integration note:** This updates the **“ZeroBounce”** spreadsheet, not necessarily the original “Emails” spreadsheet being watched. If the goal is to stop future sends, `Emailed` must be updated where the trigger reads it from.

#### Node: Send a message
- **Type / role:** `n8n-nodes-base.gmail` — sends the actual email.
- **Configuration:**
  - To: `{{$json.Email}}`
  - Subject: `Test from n8n`
  - Body: `Test message` (text email)
  - Append attribution: enabled
- **Credentials:** Gmail OAuth2
- **Input:** from Email?
- **Output:** Add Emailed = true
- **Failure modes / edge cases:**
  - Gmail OAuth expired / insufficient scopes
  - Sending limits / rate limits
  - Invalid recipient format (even if “valid” by ZB, still could be blocked)
  - If `$json.Email` differs from original sheet email field name/case, message may not send to the intended address

#### Node: Add Emailed = true
- **Type / role:** `n8n-nodes-base.googleSheets` — records successful outreach.
- **Configuration (interpreted):**
  - Operation: **Update**
  - Match by: `Email`
  - Writes:
    - `Email = {{ $('Email?').item.json.Email }}`
    - `Reason = "High score"`
    - `Emailed = "true"`
  - Document: “ZeroBounce”; Sheet: “Sheet1”
- **Input:** from Send a message
- **Edge cases / risks:**
  - Again, updates “ZeroBounce” sheet rather than the trigger sheet—may not prevent duplicates unless the trigger sheet also reflects emailed status.
  - Uses `$('Email?').item.json.Email` rather than current `$json.Email`; this is stable if there’s exactly one item and the referenced node executed in the same item context.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Google Sheets Trigger | googleSheetsTrigger | Poll Google Sheet rows/changes | — | Validate? | ## ZeroBounce Email Validation, A.I Scoring & Sending with Gmail; ![ZeroBounce Logo](https://raw.githubusercontent.com/zerobounce/n8n-nodes-zerobounce/main/icons/zerobounce-logo.svg); This workflow automates the process of validating, scoring, and emailing leads from a Google Sheet. It ensures you only send emails to high-quality contacts, protecting your sender reputation.; ### 🚀 How it Works; 1. Trigger: Watches a Google Sheet for new or updated rows (contacts).; 2. Validate: Uses ZeroBounce to check if the email address is valid.; * If Invalid: Updates the sheet with the reason and marks "Emailed" as `false`.; 3. Score: If valid, uses ZeroBounce A.I. Scoring to grade the lead quality (0-10).; * If Low/Medium Score (<9): Updates the sheet with the score and marks "Emailed" as `false`.; 4. Send: If the score is High (9-10), sends an email via Gmail.; 5. Update: Writes the final status back to the Google Sheet, preventing duplicate sends.; ### 📋 Setup Requirements; * Google Sheet: A sheet with columns for `Email`, `Validated`, `Scored`, `Score`, `Emailed`, `Reason`, etc.; * ZeroBounce Account: API Key for validation and scoring.; * Gmail Account: Connected via OAuth2 to send emails.; ### 💡 Key Features; * Cost Efficient: Checks existing `Validated` and `Scored` columns to avoid re-processing contacts.; * Risk Protection: Filters out valid but low-quality leads (e.g., catch-alls or low scores).; * Status Tracking: Keeps your Google Sheet updated with rich data (Sub-status, Domain Age, etc.). |
| Validate? | if | Skip validation if already validated | Google Sheets Trigger | Validate email; Email is valid? | (same as above) |
| Validate email | @zerobounce/n8n-nodes-zerobounce.zeroBounce | Validate email address | Validate? (true) | Add validation results | (same as above) |
| Add validation results | googleSheets | Write validation metadata back to sheet | Validate email | Email is valid? | (same as above) |
| Email is valid? | if | Route valid vs invalid | Add validation results; Validate? (false) | Score?; Not valid | (same as above) |
| Not valid | set | Attach invalid reason | Email is valid? (false) | Add Emailed = false | (same as above) |
| Score? | if | Skip scoring if already scored | Email is valid? (true) | Score email; Filter by score | (same as above) |
| Score email | @zerobounce/n8n-nodes-zerobounce.zeroBounce | AI scoring request | Score? (true) | Add Scoring Results | (same as above) |
| Add Scoring Results | googleSheets | Store score + scored flag | Score email | Filter by score | (same as above) |
| Filter by score | switch | Bucket by score thresholds | Add Scoring Results; Score? (false) | Low score; Medium score; Email? | (same as above) |
| Low score | set | Attach “Low score” reason | Filter by score (low) | Add Emailed = false | (same as above) |
| Medium score | set | Attach reason for medium bucket | Filter by score (medium) | Add Emailed = false | (same as above) |
| Add Emailed = false | googleSheets | Mark not emailed + reason | Low score; Medium score; Not valid | — | (same as above) |
| Email? | if | Skip send if already emailed | Filter by score (high) | Send a message | (same as above) |
| Send a message | gmail | Send Gmail outreach email | Email? (true) | Add Emailed = true | (same as above) |
| Add Emailed = true | googleSheets | Mark emailed true + reason | Send a message | — | (same as above) |
| Sticky Note | stickyNote | Documentation / visual note | — | — | ## ZeroBounce Email Validation, A.I Scoring & Sending with Gmail; ![ZeroBounce Logo](https://raw.githubusercontent.com/zerobounce/n8n-nodes-zerobounce/main/icons/zerobounce-logo.svg); (content as shown above) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create credentials**
   1. Google Sheets Trigger OAuth2 credential (Google account with access to the lead sheet).
   2. Google Sheets OAuth2 credential (for update operations; can be the same Google account).
   3. ZeroBounce API credential (API key).
   4. Gmail OAuth2 credential (account used to send outreach).

2) **Create the trigger**
   1. Add node: **Google Sheets Trigger**
   2. Configure:
      - Poll time: **Every Minute**
      - Document: select your spreadsheet (lead list)
      - Sheet: select tab (e.g., “Sheet1”)
   3. Ensure the sheet has columns at least: `Email`, `Validated`, `Scored`, `Score`, `Emailed`, `Reason` (and optionally the extra ZB fields).

3) **Add validation gate**
   1. Add node: **IF** named `Validate?`
   2. Condition: Boolean → **Not Equals**
      - Left: `{{$json.Validated}}`
      - Right: `true`
   3. Connect: Trigger → `Validate?`

4) **Add ZeroBounce validation**
   1. Add node: **ZeroBounce** named `Validate email`
   2. Configure:
      - Email: `{{ $('Google Sheets Trigger').item.json.Email }}`
      - Timeout: `10` seconds
   3. Connect: `Validate?` (true) → `Validate email`

5) **Write validation results back to the lead sheet**
   1. Add node: **Google Sheets** named `Add validation results`
   2. Operation: **Update**
   3. Document: your lead spreadsheet
   4. Sheet: your lead tab
   5. Matching columns: `Email`
   6. Map fields (examples):
      - `Email = {{$json.address}}`
      - `Validated = true` (prefer writing a consistent value; if using string, keep it consistent with your IF checks)
      - `ZB Status = {{$json.status}}`
      - `ZB Sub Status = {{$json.sub_status}}`
      - Add any additional metadata you want (MX, domain age, etc.)
   7. Connect: `Validate email` → `Add validation results`

6) **Branch on validation status**
   1. Add node: **IF** named `Email is valid?`
   2. Condition: String → **Equals**
      - Left: `{{$json['ZB Status']}}`
      - Right: `valid`
   3. Connect:
      - `Add validation results` → `Email is valid?`
      - Optionally connect `Validate?` (false) → `Email is valid?` only if you are sure `ZB Status` exists in that path.

7) **Add scoring gate**
   1. Add node: **IF** named `Score?`
   2. Condition: `{{$json.Scored}} != true`
   3. Connect: `Email is valid?` (true) → `Score?`

8) **Add ZeroBounce scoring**
   1. Add node: **ZeroBounce** named `Score email`
   2. Resource: **scoring**
   3. Email: `{{$json.Email}}`
   4. Connect: `Score?` (true) → `Score email`

9) **Write scoring results**
   1. Add node: **Google Sheets** named `Add Scoring Results`
   2. Operation: **Update**
   3. Decide where scores live:
      - Either update the same lead sheet, or a separate scoring sheet.
   4. Match by `Email`
   5. Map:
      - `Email = {{$json.email}}`
      - `Score = {{$json.score}}`
      - `Scored = true`
   6. Connect: `Score email` → `Add Scoring Results`

10) **Switch by score thresholds**
   1. Add node: **Switch** named `Filter by score`
   2. Rules:
      - Output “low”: number `< 3` using `{{$json.Score}}`
      - Output “medium”: number `< 9` using `{{$json.Score}}`
      - Output “high”: number `>= 9` using `{{$json.Score}}`
      - Fallback: `none`
   3. Connect:
      - `Add Scoring Results` → `Filter by score`
      - Optionally `Score?` (false) → `Filter by score` if items already contain `Score`.

11) **Handle invalid/low/medium outcomes (mark not emailed)**
   1. Add **Set** node `Not valid`: set `reason = {{$json['ZB Status']}}`
      - Connect: `Email is valid?` (false) → `Not valid`
   2. Add **Set** node `Low score`: set `reason = "Low score"`
      - Connect: `Filter by score` (low) → `Low score`
   3. Add **Set** node `Medium score`: set `reason = "Medium score"` (recommended)
      - Connect: `Filter by score` (medium) → `Medium score`
   4. Add **Google Sheets** node `Add Emailed = false`
      - Operation: Update; match `Email`
      - Set `Emailed = false`, `Reason = {{$json.reason}}`
      - Connect: `Not valid` → it, `Low score` → it, `Medium score` → it

12) **High score path: prevent duplicates, send Gmail, mark emailed**
   1. Add **IF** node `Email?`:
      - Condition: `{{ $('Google Sheets Trigger').item.json.Emailed }} != true`
   2. Connect: `Filter by score` (high) → `Email?`
   3. Add Gmail node `Send a message`:
      - To: `{{$json.Email}}`
      - Subject/body as desired
   4. Connect: `Email?` (true) → `Send a message`
   5. Add Google Sheets node `Add Emailed = true`:
      - Operation: Update; match `Email`
      - Set `Emailed = true`, `Reason = "High score"`
   6. Connect: `Send a message` → `Add Emailed = true`

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| ZeroBounce Logo | https://raw.githubusercontent.com/zerobounce/n8n-nodes-zerobounce/main/icons/zerobounce-logo.svg |
| This workflow checks existing `Validated` and `Scored` columns to avoid re-processing contacts | Mentioned in the workflow sticky note |
| It filters out valid but low-quality leads (e.g., catch-alls or low scores) to protect sender reputation | Mentioned in the workflow sticky note |
| It keeps the Google Sheet updated with rich data (sub-status, domain age, etc.) | Mentioned in the workflow sticky note |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.