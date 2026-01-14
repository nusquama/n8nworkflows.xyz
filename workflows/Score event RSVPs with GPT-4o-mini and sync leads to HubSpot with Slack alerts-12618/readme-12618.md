Score event RSVPs with GPT-4o-mini and sync leads to HubSpot with Slack alerts

https://n8nworkflows.xyz/workflows/score-event-rsvps-with-gpt-4o-mini-and-sync-leads-to-hubspot-with-slack-alerts-12618


# Score event RSVPs with GPT-4o-mini and sync leads to HubSpot with Slack alerts

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name:** Event Lead Scorer  
**Purpose:** Receive event RSVP/registration submissions, validate email, enrich and score the lead using **GPT-4o-mini**, then **upsert** the contact into **HubSpot** and send **Slack alerts** to Sales or Marketing based on score, while logging outcomes to **Google Sheets**.

**Primary use cases**
- Automated lead qualification for event registrations (web forms, landing pages, RSVP tools).
- Centralized CRM sync (HubSpot) with AI-based scoring fields stored as custom properties.
- Real-time team routing via Slack and operational logging to Sheets.

### Logical blocks
1. **1.1 Input Reception & Global Configuration**
2. **1.2 Email Validation & Invalid Logging**
3. **1.3 AI Enrichment & Score Parsing**
4. **1.4 Lead Triage (Qualified vs Nurture)**
5. **1.5 HubSpot Upsert + Slack Notifications**
6. **1.6 Final Logging (All processed valid leads)**

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception & Global Configuration

**Overview:** Accepts POSTed RSVP payloads via webhook and sets workflow-wide configuration values (thresholds, channel IDs, HubSpot URL template) used later via expressions.

**Nodes involved**
- Event Registration Webhook
- Workflow Configuration
- Note - Trigger (sticky note)

#### Node: Event Registration Webhook
- **Type / role:** `Webhook` — entry point receiving form submissions.
- **Key configuration:**
  - **Path:** `event-rsvp`
  - **Method:** `POST`
  - **Response mode:** `lastNode` (the webhook response will be whatever the final node returns).
- **Input/Output:**
  - **Input:** external HTTP request body (expected fields described in Trigger note).
  - **Output:** to **Workflow Configuration**.
- **Edge cases / failures:**
  - If the caller sends unexpected structure (e.g., nested fields), downstream expressions expecting `$json.email` etc. may be empty.
  - `lastNode` can create long response times because the caller waits until the workflow completes (OpenAI + HubSpot + Slack + Sheets latency).

#### Node: Workflow Configuration
- **Type / role:** `Set` — injects constants + keeps other incoming fields.
- **Configuration choices (interpreted):**
  - Adds:
    - `hubspotContactUrl`: `https://app.hubspot.com/contacts/<PortalID>/contact/` (placeholder)
    - `qualifiedLeadThreshold`: `70` (note: later IF uses a hard-coded `70`, not this variable)
    - `slackSalesChannel`: placeholder Slack channel ID
    - `slackMarketingChannel`: placeholder Slack channel ID
  - **Include other fields:** enabled (preserves webhook payload such as `email`, `firstName`, etc.).
- **Key expressions/vars used later:**
  - Referenced as `$('Workflow Configuration').first().json.*` inside Slack messages and email validation.
- **Input/Output:**
  - From **Event Registration Webhook**
  - To **Check Email Valid**
- **Edge cases:**
  - Placeholders must be replaced; otherwise Slack messages and HubSpot links will be wrong.
  - If the webhook payload doesn’t include `email`, the IF condition will route to invalid logging.

#### Sticky note: Note - Trigger
Content (applies to the input block):
- “Webhook receives event registration form submissions with: firstName, lastName, email, company, jobTitle, country, utmSource”

---

### 2.2 Email Validation & Invalid Logging

**Overview:** Validates that the email exists and matches a regex. Invalid submissions are written to a Google Sheet for review.

**Nodes involved**
- Check Email Valid
- Log Invalid Submissions
- Note - Validation (sticky note)
- Note - Invalid Branch (sticky note)

#### Node: Check Email Valid
- **Type / role:** `IF` — branching validation.
- **Configuration choices:**
  - Condition 1: `email` **not empty**
    - `={{ $('Workflow Configuration').item.json.email }}`
  - Condition 2: `email` matches regex:
    - `^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$`
- **Input/Output:**
  - Input from **Workflow Configuration**
  - **True** → AI processing (**AI Lead Enrichment & Scoring**)
  - **False** → invalid logging (**Log Invalid Submissions**)
- **Edge cases / failures:**
  - Using `$('Workflow Configuration').item.json.email` assumes the current item context matches; generally safe here, but if later changed to multi-item flows, `.first()` would be safer.
  - Regex rejects some valid but uncommon emails (internationalized domains, new TLD edge cases, quoted local parts).

#### Node: Log Invalid Submissions
- **Type / role:** `Google Sheets` — append/update invalid submissions.
- **Operation:** `appendOrUpdate`
- **Target:**
  - Spreadsheet: placeholder “Google Sheet ID for Invalid Submissions”
  - Sheet name: `Invalid Submissions`
  - Matching column: `email`
- **Columns written:**
  - `firstName, lastName, email, company, jobTitle, country, utmSource, timestamp`
  - `timestamp = new Date().toISOString()`
- **Input/Output:**
  - Input from **Check Email Valid** (false branch)
  - No downstream nodes connected.
- **Credentials:** Google Sheets OAuth2.
- **Edge cases / failures:**
  - If `email` is missing/blank, “appendOrUpdate” with matching on `email` can behave unexpectedly (may create duplicates or fail depending on sheet constraints).
  - Sheet schema must exist with matching headers/columns; otherwise mapping issues occur.
  - OAuth token expiration / permission errors.

#### Sticky note: Note - Validation
- “Check if email is present and valid. Invalid submissions → logged to Google Sheets”

#### Sticky note: Note - Invalid Branch
- “Log invalid/missing email submissions to Google Sheets for review”

---

### 2.3 AI Enrichment & Score Parsing

**Overview:** Sends lead attributes to OpenAI (GPT-4o-mini) to get structured scoring output, then parses the returned JSON into dedicated fields.

**Nodes involved**
- AI Lead Enrichment & Scoring
- Parse AI Response
- Note - AI Scoring (sticky note)

#### Node: AI Lead Enrichment & Scoring
- **Type / role:** `OpenAI (LangChain)` — LLM call for enrichment/scoring.
- **Model:** `gpt-4o-mini` (selected by modelId).
- **Prompt behavior (interpreted):**
  - Provides lead fields (name, email, company, job title, country, UTM source).
  - Requests **ONLY valid JSON** with:
    - `fitLevel` = High/Medium/Low
    - `leadScore` = 0–100
    - `nextAction`
    - `reason`
- **Input/Output:**
  - Input from **Check Email Valid** (true branch)
  - Output to **Parse AI Response**
- **Credentials:** OpenAI API (n8n credentials).
- **Edge cases / failures:**
  - LLM may return invalid JSON (extra text, trailing commas, code fences) despite instruction → will break `JSON.parse`.
  - Rate limits/timeouts from OpenAI.
  - Token/context issues if upstream fields are very large (unlikely here).

#### Node: Parse AI Response
- **Type / role:** `Set` — transforms LLM output into structured fields.
- **Configuration choices:**
  - `aiResponse` (object): `={{ JSON.parse($json.message.content) }}`
  - `fitLevel`: `={{ JSON.parse($json.message.content).fitLevel }}`
  - `leadScore`: `={{ JSON.parse($json.message.content).leadScore }}`
  - `nextAction`: `={{ JSON.parse($json.message.content).nextAction }}`
  - `reason`: `={{ JSON.parse($json.message.content).reason }}`
  - **Include other fields:** enabled (preserves lead input fields).
- **Input/Output:**
  - From **AI Lead Enrichment & Scoring**
  - To **Check Lead Score >= 70**
- **Edge cases / failures:**
  - Any non-JSON response causes an exception and stops the workflow.
  - `leadScore` could be a string; IF node uses strict numeric compare later (potential type mismatch).
  - Re-parsing JSON five times is redundant; if performance/robustness matters, parse once into `aiResponse` and reference it.

#### Sticky note: Note - AI Scoring
- “OpenAI analyzes lead data and returns: ICP fit level, Lead score, Next action recommendation, Reasoning”

---

### 2.4 Lead Triage (Qualified vs Nurture)

**Overview:** Splits leads into “Qualified” or “Nurture” based on score threshold.

**Nodes involved**
- Check Lead Score >= 70
- Note - Lead Triage (sticky note)

#### Node: Check Lead Score >= 70
- **Type / role:** `IF` — routing logic.
- **Configuration choices:**
  - Numeric condition: `leadScore >= 70`
  - Left value: `={{ $('Parse AI Response').item.json.leadScore }}`
  - Right value: `70` (hard-coded)
  - Type validation: strict
- **Input/Output:**
  - From **Parse AI Response**
  - **True** → **Upsert Contact in HubSpot (Qualified)**
  - **False** → **Upsert Contact in HubSpot (Nurture)**
- **Edge cases / failures:**
  - Score threshold is duplicated (also exists as `qualifiedLeadThreshold` in config but not used).
  - If `leadScore` is missing or non-numeric, strict compare may fail or route unexpectedly.

#### Sticky note: Note - Lead Triage
- “Score >= 70 → Qualified Lead; Score < 70 → Nurture Lead”

---

### 2.5 HubSpot Upsert + Slack Notifications

**Overview:** Upserts the contact into HubSpot (same fields for both branches), then notifies the appropriate Slack channel with lead details and a HubSpot contact link.

**Nodes involved**
- Upsert Contact in HubSpot (Qualified)
- Upsert Contact in HubSpot (Nurture)
- Notify Sales - Qualified Lead
- Notify Marketing - Nurture Lead
- Note - HubSpot CRM (sticky note)
- Note - Slack Notifications (sticky note)

#### Node: Upsert Contact in HubSpot (Qualified)
- **Type / role:** `HubSpot` — create/update contact by email.
- **Operation (interpreted):** Upsert contact using `email`.
- **Fields set:**
  - Standard-ish fields: firstName, lastName, jobTitle, country, companyName
  - Custom properties:
    - `ai_fit_level` = `{{$json.fitLevel}}`
    - `ai_lead_score` = `{{$json.leadScore}}`
    - `ai_next_action` = `{{$json.nextAction}}`
    - `ai_reason` = `{{$json.reason}}`
    - `utm_source` = `{{$json.utmSource}}`
- **Input/Output:**
  - From **Check Lead Score >= 70** (true branch)
  - To **Notify Sales - Qualified Lead**
- **Edge cases / failures:**
  - Requires HubSpot credentials (not included in JSON export here).
  - Custom properties must exist in HubSpot with correct internal names, otherwise the update may fail or ignore fields.
  - Output structure must include a contact `id` for Slack link usage; if the node returns a different key, the Slack link will break.

#### Node: Upsert Contact in HubSpot (Nurture)
- **Type / role:** `HubSpot` — same upsert behavior for nurture leads.
- **Configuration:** same fields and custom properties as the qualified node.
- **Input/Output:**
  - From **Check Lead Score >= 70** (false branch)
  - To **Notify Marketing - Nurture Lead**
- **Edge cases / failures:** same as qualified upsert.

#### Node: Notify Sales - Qualified Lead
- **Type / role:** `Slack` — sends message to Sales channel.
- **Authentication:** OAuth2
- **Channel selection:** by ID from config: `={{ $('Workflow Configuration').first().json.slackSalesChannel }}`
- **Message content highlights:**
  - Includes lead details, score, fit, nextAction, reason.
  - HubSpot link: `<{{ hubspotContactUrl }}{{ $json.id }}|View in HubSpot>`
- **Input/Output:**
  - From **Upsert Contact in HubSpot (Qualified)**
  - To **Log to Event RSVP AI Leads Sheet**
- **Edge cases / failures:**
  - If `$json.id` isn’t present (or differs), the HubSpot link is invalid.
  - Wrong Slack channel ID or missing scopes (`chat:write`) causes errors.
  - Message formatting depends on Slack mrkdwn; generally fine.

#### Node: Notify Marketing - Nurture Lead
- **Type / role:** `Slack` — sends message to Marketing channel.
- **Authentication:** OAuth2
- **Channel selection:** `={{ $('Workflow Configuration').first().json.slackMarketingChannel }}`
- **Message content highlights:**
  - Similar, but omits fit/reason and uses a nurture framing.
  - HubSpot link uses the same URL + `$json.id`.
- **Input/Output:**
  - From **Upsert Contact in HubSpot (Nurture)**
  - To **Log to Event RSVP AI Leads Sheet**
- **Edge cases / failures:** same as Sales notification.

#### Sticky note: Note - HubSpot CRM
- “Upsert contact by email. Store AI fields in custom properties: ai_fit_level, ai_lead_score, ai_next_action, ai_reason”

#### Sticky note: Note - Slack Notifications
- “Qualified → #sales channel; Nurture → #marketing channel; Includes lead details, score, and HubSpot link”

---

### 2.6 Final Logging (All processed valid leads)

**Overview:** Logs all valid processed leads (both qualified and nurture) with input + AI outputs to a Google Sheet.

**Nodes involved**
- Log to Event RSVP AI Leads Sheet
- Note - Logging (sticky note)

#### Node: Log to Event RSVP AI Leads Sheet
- **Type / role:** `Google Sheets` — append/update processed leads.
- **Operation:** `appendOrUpdate`
- **Target:**
  - Spreadsheet: placeholder “Google Sheet ID for Event RSVP AI Leads”
  - Sheet name: `Event RSVP AI Leads`
  - Matching column: `email`
- **Columns written:**
  - Input: firstName, lastName, email, company, jobTitle, country, utmSource
  - AI: leadScore, fitLevel, nextAction, reason
  - `timestamp = new Date().toISOString()`
- **Input/Output:**
  - From **Notify Sales - Qualified Lead** and **Notify Marketing - Nurture Lead**
  - No downstream nodes.
- **Credentials:** Google Sheets OAuth2.
- **Edge cases / failures:**
  - If the Slack node output does not carry forward all original fields, the log may miss values. (This depends on the HubSpot/Slack node output behavior; in many nodes, outputs change shape.)
  - Same “appendOrUpdate” considerations as above (schema alignment, permissions, duplicate behavior).

#### Sticky note: Note - Logging
- “All processed leads logged to Google Sheets: Input fields, AI scoring results, Timestamp”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Note - Trigger | Sticky Note | Documentation (trigger payload expectations) | — | — | ## 🎯 Trigger; Webhook receives event registration form submissions with:; - firstName, lastName, email; - company, jobTitle, country; - utmSource |
| Event Registration Webhook | Webhook | Entry point (POST RSVP submissions) | — | Workflow Configuration | ## 🎯 Trigger; Webhook receives event registration form submissions with:; - firstName, lastName, email; - company, jobTitle, country; - utmSource |
| Workflow Configuration | Set | Adds constants (HubSpot URL, threshold, Slack channels) | Event Registration Webhook | Check Email Valid |  |
| Note - Validation | Sticky Note | Documentation (email validation + invalid logging) | — | — | ## ✅ Validation; Check if email is present and valid.; Invalid submissions → logged to Google Sheets |
| Check Email Valid | IF | Validate email presence + regex | Workflow Configuration | AI Lead Enrichment & Scoring (true), Log Invalid Submissions (false) | ## ✅ Validation; Check if email is present and valid.; Invalid submissions → logged to Google Sheets |
| Note - Invalid Branch | Sticky Note | Documentation (invalid branch logging) | — | — | ## ❌ Invalid Submissions; Log invalid/missing email submissions to Google Sheets for review |
| Log Invalid Submissions | Google Sheets | Persist invalid submissions | Check Email Valid (false) | — | ## ❌ Invalid Submissions; Log invalid/missing email submissions to Google Sheets for review |
| Note - AI Scoring | Sticky Note | Documentation (LLM scoring outputs) | — | — | ## 🤖 AI Lead Enrichment; OpenAI analyzes lead data and returns:; - ICP fit level (High/Medium/Low); - Lead score (0-100); - Next action recommendation; - Reasoning |
| AI Lead Enrichment & Scoring | OpenAI (LangChain) | LLM call to score/enrich lead | Check Email Valid (true) | Parse AI Response | ## 🤖 AI Lead Enrichment; OpenAI analyzes lead data and returns:; - ICP fit level (High/Medium/Low); - Lead score (0-100); - Next action recommendation; - Reasoning |
| Parse AI Response | Set | JSON.parse LLM output into fields | AI Lead Enrichment & Scoring | Check Lead Score >= 70 |  |
| Note - Lead Triage | Sticky Note | Documentation (routing by score) | — | — | ## 🎯 Lead Triage; Score >= 70 → Qualified Lead; Score < 70 → Nurture Lead |
| Check Lead Score >= 70 | IF | Route qualified vs nurture | Parse AI Response | Upsert Contact in HubSpot (Qualified) (true), Upsert Contact in HubSpot (Nurture) (false) | ## 🎯 Lead Triage; Score >= 70 → Qualified Lead; Score < 70 → Nurture Lead |
| Note - HubSpot CRM | Sticky Note | Documentation (HubSpot upsert + custom properties) | — | — | ## 📊 HubSpot CRM; Upsert contact by email.; Store AI fields in custom properties:; - ai_fit_level; - ai_lead_score; - ai_next_action; - ai_reason |
| Upsert Contact in HubSpot (Qualified) | HubSpot | Upsert contact + store AI fields (qualified) | Check Lead Score >= 70 (true) | Notify Sales - Qualified Lead | ## 📊 HubSpot CRM; Upsert contact by email.; Store AI fields in custom properties:; - ai_fit_level; - ai_lead_score; - ai_next_action; - ai_reason |
| Upsert Contact in HubSpot (Nurture) | HubSpot | Upsert contact + store AI fields (nurture) | Check Lead Score >= 70 (false) | Notify Marketing - Nurture Lead | ## 📊 HubSpot CRM; Upsert contact by email.; Store AI fields in custom properties:; - ai_fit_level; - ai_lead_score; - ai_next_action; - ai_reason |
| Note - Slack Notifications | Sticky Note | Documentation (Slack routing + message content) | — | — | ## 💬 Slack Notifications; Qualified → #sales channel; Nurture → #marketing channel; Includes lead details, score, and HubSpot link |
| Notify Sales - Qualified Lead | Slack | Send qualified lead alert to Sales channel | Upsert Contact in HubSpot (Qualified) | Log to Event RSVP AI Leads Sheet | ## 💬 Slack Notifications; Qualified → #sales channel; Nurture → #marketing channel; Includes lead details, score, and HubSpot link |
| Notify Marketing - Nurture Lead | Slack | Send nurture lead alert to Marketing channel | Upsert Contact in HubSpot (Nurture) | Log to Event RSVP AI Leads Sheet | ## 💬 Slack Notifications; Qualified → #sales channel; Nurture → #marketing channel; Includes lead details, score, and HubSpot link |
| Note - Logging | Sticky Note | Documentation (final Sheets logging) | — | — | ## 📝 Logging; All processed leads logged to Google Sheets:; - Input fields; - AI scoring results; - Timestamp |
| Log to Event RSVP AI Leads Sheet | Google Sheets | Log valid processed leads (append/update) | Notify Sales - Qualified Lead; Notify Marketing - Nurture Lead | — | ## 📝 Logging; All processed leads logged to Google Sheets:; - Input fields; - AI scoring results; - Timestamp |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name: **Event Lead Scorer**
   - (Optional) Set workflow setting **Execution Order** to `v1` to match behavior.

2. **Add Webhook trigger**
   - Node: **Webhook**
   - HTTP Method: **POST**
   - Path: **event-rsvp**
   - Response: **Last node**
   - Expected incoming JSON fields (from your form): `firstName, lastName, email, company, jobTitle, country, utmSource`

3. **Add configuration node**
   - Node: **Set** (name it **Workflow Configuration**)
   - Enable **Include Other Fields**
   - Add fields:
     - `hubspotContactUrl` (string): `https://app.hubspot.com/contacts/<YOUR_PORTAL_ID>/contact/`
     - `qualifiedLeadThreshold` (number): `70`
     - `slackSalesChannel` (string): `C...` (Sales channel ID)
     - `slackMarketingChannel` (string): `C...` (Marketing channel ID)
   - Connect: **Webhook → Workflow Configuration**

4. **Add email validation**
   - Node: **IF** (name: **Check Email Valid**)
   - Conditions (AND):
     - String: `={{ $('Workflow Configuration').item.json.email }}` → **is not empty**
     - String regex: `={{ $('Workflow Configuration').item.json.email }}` matches `^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$`
   - Connect: **Workflow Configuration → Check Email Valid**

5. **Invalid branch logging (false output)**
   - Node: **Google Sheets** (name: **Log Invalid Submissions**)
   - Credential: **Google Sheets OAuth2** (connect your Google account)
   - Operation: **Append or Update**
   - Document: select the spreadsheet for invalid submissions
   - Sheet: `Invalid Submissions`
   - Matching column: `email`
   - Map columns:
     - `firstName={{$json.firstName}}`, `lastName={{$json.lastName}}`, `email={{$json.email}}`, `company`, `jobTitle`, `country`, `utmSource`
     - `timestamp={{new Date().toISOString()}}`
   - Connect: **Check Email Valid (false) → Log Invalid Submissions**

6. **AI enrichment (true output)**
   - Node: **OpenAI (LangChain)** (name: **AI Lead Enrichment & Scoring**)
   - Credential: **OpenAI API** (configure in n8n credentials)
   - Model: **gpt-4o-mini**
   - Prompt: provide lead fields and request JSON only with `fitLevel, leadScore, nextAction, reason` (as in the workflow).
   - Connect: **Check Email Valid (true) → AI Lead Enrichment & Scoring**

7. **Parse AI response**
   - Node: **Set** (name: **Parse AI Response**)
   - Enable **Include Other Fields**
   - Add fields using expressions:
     - `aiResponse` (object): `{{ JSON.parse($json.message.content) }}`
     - `fitLevel` (string): `{{ JSON.parse($json.message.content).fitLevel }}`
     - `leadScore` (number): `{{ JSON.parse($json.message.content).leadScore }}`
     - `nextAction` (string): `{{ JSON.parse($json.message.content).nextAction }}`
     - `reason` (string): `{{ JSON.parse($json.message.content).reason }}`
   - Connect: **AI Lead Enrichment & Scoring → Parse AI Response**

8. **Triage by score**
   - Node: **IF** (name: **Check Lead Score >= 70**)
   - Condition:
     - Number: `={{ $('Parse AI Response').item.json.leadScore }}` **greater than or equal** `70`
     - (Optional improvement: use `{{ $('Workflow Configuration').first().json.qualifiedLeadThreshold }}` instead of `70`.)
   - Connect: **Parse AI Response → Check Lead Score >= 70**

9. **HubSpot upsert (Qualified branch)**
   - Node: **HubSpot** (name: **Upsert Contact in HubSpot (Qualified)**)
   - Credential: **HubSpot** (private app token or OAuth, depending on your setup)
   - Upsert by: **Email**
   - Set properties:
     - Standard: firstName, lastName, companyName, jobTitle, country
     - Custom: `ai_fit_level, ai_lead_score, ai_next_action, ai_reason, utm_source`
   - Ensure these custom properties exist in HubSpot with matching internal names.
   - Connect: **Check Lead Score >= 70 (true) → Upsert Contact in HubSpot (Qualified)**

10. **HubSpot upsert (Nurture branch)**
   - Duplicate the previous HubSpot node, name it **Upsert Contact in HubSpot (Nurture)** (same mapping).
   - Connect: **Check Lead Score >= 70 (false) → Upsert Contact in HubSpot (Nurture)**

11. **Slack notification (Qualified)**
   - Node: **Slack** (name: **Notify Sales - Qualified Lead**)
   - Credential: **Slack OAuth2** with `chat:write`
   - Send to channel by ID:
     - Channel ID: `={{ $('Workflow Configuration').first().json.slackSalesChannel }}`
   - Message text: include lead details + score + link:
     - Link pattern: `{{ $('Workflow Configuration').first().json.hubspotContactUrl }}{{ $json.id }}`
   - Connect: **Upsert Contact in HubSpot (Qualified) → Notify Sales - Qualified Lead**

12. **Slack notification (Nurture)**
   - Node: **Slack** (name: **Notify Marketing - Nurture Lead**)
   - Channel ID: `={{ $('Workflow Configuration').first().json.slackMarketingChannel }}`
   - Message text: nurture framing + HubSpot link.
   - Connect: **Upsert Contact in HubSpot (Nurture) → Notify Marketing - Nurture Lead**

13. **Final logging to Google Sheets (both branches)**
   - Node: **Google Sheets** (name: **Log to Event RSVP AI Leads Sheet**)
   - Credential: Google Sheets OAuth2
   - Operation: **Append or Update**
   - Document: select spreadsheet for processed leads
   - Sheet: `Event RSVP AI Leads`
   - Matching column: `email`
   - Map columns: all lead input fields + AI outputs + timestamp.
   - Connect:
     - **Notify Sales - Qualified Lead → Log to Event RSVP AI Leads Sheet**
     - **Notify Marketing - Nurture Lead → Log to Event RSVP AI Leads Sheet**

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Webhook response mode is “lastNode”, meaning the form submitter may wait for OpenAI + HubSpot + Slack + Sheets to finish. Consider changing response mode if you need faster HTTP responses. | Workflow-level integration behavior |
| Score threshold is defined in config (`qualifiedLeadThreshold=70`) but the triage IF uses a hard-coded `70`. Align them if you want one source of truth. | Maintainability / configuration hygiene |
| HubSpot custom properties must exist (`ai_fit_level`, `ai_lead_score`, `ai_next_action`, `ai_reason`, `utm_source`) with correct internal names. | HubSpot configuration prerequisite |
| Parsing relies on `JSON.parse($json.message.content)`; any non-JSON LLM output will fail the run. Consider adding a fallback/repair step if reliability is critical. | Operational robustness |