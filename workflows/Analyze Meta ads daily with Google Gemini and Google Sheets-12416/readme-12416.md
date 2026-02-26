Analyze Meta ads daily with Google Gemini and Google Sheets

https://n8nworkflows.xyz/workflows/analyze-meta-ads-daily-with-google-gemini-and-google-sheets-12416


# Analyze Meta ads daily with Google Gemini and Google Sheets

## 1. Workflow Overview

**Purpose:** Run once per day to pull **yesterday’s Meta (Facebook) Ads performance**, store **raw campaign + ad-level metrics** in **Google Sheets**, then send a **structured dataset** to **Google Gemini** to generate **actionable insights** (winners/losers + recommendations) and store those insights back into Google Sheets. It also includes **error handling** that emails a configured address when the workflow fails.

**Primary use cases:**
- Daily performance monitoring for a Meta ad account
- Producing consistent, client-ready qualitative insights from quantitative Meta metrics
- Building a historical dataset in Google Sheets for reporting dashboards

### 1.1 Configuration & Scheduling
Defines when the workflow runs and where account/report parameters live.

### 1.2 Campaign-level Data Collection → Clean → Store
Pull campaign insights (yesterday), remove zero-spend campaigns, append rows to a “Campaigns” sheet.

### 1.3 Ad-level Data Collection → Normalize → Store
Pull ad-level insights (yesterday), split records, append to an “Ads” sheet.

### 1.4 Data Structuring for AI
Aggregate raw ad rows into a hierarchical structure (campaign → adset → ads) so the LLM can reason across entities.

### 1.5 AI Analysis (Gemini) with Structured Output Parsing
Send structured data to Gemini with strict JSON requirements, parse result into machine-usable objects.

### 1.6 Insight Storage
Split AI output into per-row items and append to “AI_Insights” sheet.

### 1.7 Error Handling
On any failure, send an email containing message/stack/last node executed.

---

## 2. Block-by-Block Analysis

### Block 1 — Configuration & Scheduling
**Overview:** Triggers daily execution and sets shared configuration values (ad account + notification email).  
**Nodes involved:** `Schedule - Once per day`, `Set Configuration`, `Template Guide`, `Sticky Note1`

#### Node: Schedule - Once per day
- **Type / role:** Schedule Trigger; starts workflow automatically.
- **Configuration:** Runs daily at **06:00** (instance timezone).
- **Connections:**
  - Outputs to `Set Configuration`
  - Outputs directly to `Get Ad Insights - Ad Level` (parallel branch)
- **Edge cases / failures:**
  - Timezone mismatch can cause “yesterday” to not align with business day.
  - If n8n is down at trigger time, the run is missed unless you add retry/backfill logic.
- **Version notes:** `typeVersion 1.2`

#### Node: Set Configuration
- **Type / role:** Set node; centralizes constants.
- **Configuration choices:**
  - Sets:
    - `Ad Account ID` (string): `YOUR_ACT_ID_HERE`
    - `Email` (string): `user@example.com`
- **Key variables:** Exposed via expressions like:
  - `$('Set Configuration').first().json['Ad Account ID']`
- **Connections:**
  - Main output → `Get Ad Insights - Campaign Level`
- **Edge cases / failures:**
  - If left as placeholder, Meta API nodes will query `act_YOUR_ACT_ID_HERE` and fail.
- **Version notes:** `typeVersion 3.4`

#### Node: Template Guide (Sticky note)
- **Type / role:** Sticky note documentation.
- **Notable embedded resources:**
  - Facebook Graph API Guide: https://docs.google.com/document/d/1ydWNom0TUVh5SkU9IymlpnM_NXbTqHeL_YetDLn-9m0/edit?usp=sharing
  - Google Sheets template: https://docs.google.com/spreadsheets/d/11M6Co13t9t2P7NtFaTm9VXHnzUHXO_RGYf-FRnHkS28/edit?usp=sharing
- **Operational notes in content:**
  - Mentions date-range query format: `time_range[since]=YYYY-MM-DD&time_range[until]=YYYY-MM-DD`

---

### Block 2 — Campaign-level Data Collection → Filter → Store
**Overview:** Pulls yesterday’s campaign insights from Meta, splits into items, filters out zero-spend campaigns, then appends to Google Sheets.  
**Nodes involved:** `Get Ad Insights - Campaign Level`, `Split Campaign Data`, `Filter Zero Spend`, `Save Campaign Data`, `Sticky Note2`

#### Node: Get Ad Insights - Campaign Level
- **Type / role:** Facebook Graph API; fetch campaign-level insights.
- **Configuration choices:**
  - **Graph API version:** `v24.0`
  - **Node (object):** `act_{{ $('Set Configuration').first().json['Ad Account ID'] }}`
  - **Edge:** `insights?...&level=campaign&date_preset=yesterday`
  - Requests fields: `campaign_id,campaign_name,spend,impressions,clicks,results,marketing_messages_delivered,actions,reach,ctr,cost_per_result`
- **Connections:** Main output → `Split Campaign Data`
- **Credentials:** `facebookGraphApi`
- **Edge cases / failures:**
  - Auth/permissions: missing `ads_read` / access token expired.
  - API rate limits.
  - `results`/`cost_per_result` shape varies by objective; may be empty arrays.
  - If no spend yesterday, response `data` may be empty (downstream nodes must handle 0 items).
- **Version notes:** `typeVersion 1`

#### Node: Split Campaign Data
- **Type / role:** Split Out; converts `data[]` array into one item per campaign.
- **Configuration:** `fieldToSplitOut = "data"`
- **Connections:** Main output → `Filter Zero Spend`
- **Edge cases / failures:**
  - If Meta response has no `data` field or it’s not an array, this node errors.
- **Version notes:** `typeVersion 1`

#### Node: Filter Zero Spend
- **Type / role:** Filter; removes campaigns where spend == "0".
- **Configuration choices:**
  - Condition: `$json.spend` **string** `notEquals` `"0"`
  - Strict type validation enabled in node options.
- **Connections:** “true” output → `Save Campaign Data`
- **Edge cases / failures:**
  - Meta often returns spend as `"0.00"` rather than `"0"`. Those would **pass** this filter unintentionally and be stored.
  - If spend is numeric rather than string, strict validation can behave unexpectedly.
- **Version notes:** `typeVersion 2.2`

#### Node: Save Campaign Data
- **Type / role:** Google Sheets; append campaign metrics to a sheet.
- **Configuration choices:**
  - Operation: **Append**
  - Spreadsheet: `YOUR_SPREADSHEET_ID_HERE`
  - Sheet name: `Campaigns`
  - Column mapping (examples):
    - `Date` = `$json.date_start`
    - `Campaign ID` = `$json.campaign_id`
    - `Spend` = `$json.spend`
    - `Conversations Started` = `$json.results[0].values[0].value`
    - `CPL` = `$json.cost_per_result[0].values[0].value`
- **Credentials:** `googleSheetsOAuth2Api`
- **Edge cases / failures:**
  - If `results` or `cost_per_result` arrays are empty, expressions like `[0].values[0].value` will throw.
  - Spreadsheet/sheet not found, insufficient permissions, or wrong Spreadsheet ID.
- **Version notes:** `typeVersion 4.7`

---

### Block 3 — Ad-level Data Collection → Store
**Overview:** Pulls yesterday’s ad-level insights from Meta, splits rows, stores raw data in Sheets, and simultaneously forwards data for aggregation.  
**Nodes involved:** `Get Ad Insights - Ad Level`, `Split Ad Data`, `Save Ad Data`, `Sticky Note4`

#### Node: Get Ad Insights - Ad Level
- **Type / role:** Facebook Graph API; fetch ad-level insights.
- **Configuration choices:**
  - Graph API version: `v24.0`
  - Node: `act_{{ $('Set Configuration').first().json['Ad Account ID'] }}`
  - Edge: `insights?fields=ad_id,ad_name,adset_id,adset_name,campaign_id,campaign_name,spend,reach,impressions,clicks,results,ctr,cpc,cpm&level=ad&date_preset=yesterday`
- **Connections:** Main output → `Split Ad Data`
- **Credentials:** `facebookGraphApi`
- **Edge cases / failures:**
  - Same auth/rate-limit risks as campaign-level.
  - Some ads may have missing metrics or zero impressions; later AI prompt says to ignore zero-impression ads, but code does not filter them.
- **Version notes:** `typeVersion 1`

#### Node: Split Ad Data
- **Type / role:** Split Out; one item per ad insight row.
- **Configuration:** `fieldToSplitOut = "data"`
- **Connections:**
  - → `Save Ad Data`
  - → `Structure Data Hierarchy`
- **Edge cases / failures:**
  - If `data` missing/non-array, node errors.
- **Version notes:** `typeVersion 1`

#### Node: Save Ad Data
- **Type / role:** Google Sheets; append raw ad metrics to `Ads`.
- **Configuration choices:**
  - Operation: **Append**
  - Spreadsheet: `YOUR_SPREADSHEET_ID_HERE`
  - Sheet: `Ads`
  - Mapping highlights:
    - `date` = `$json.date_stop`
    - `results` = `$json.results[0].values[0].value`
    - plus spend/reach/impressions/clicks/ctr/cpc/cpm and ids/names
- **Credentials:** `googleSheetsOAuth2Api`
- **Edge cases / failures:**
  - `results[0]...` may not exist depending on account/objective → expression failure.
  - Wrong sheet column names/order can cause missing data or append errors.
- **Version notes:** `typeVersion 4.7`

---

### Block 4 — Data Structuring for AI
**Overview:** Converts flat ad-level rows into a nested hierarchy grouped by campaign and adset, with per-ad metrics included.  
**Nodes involved:** `Structure Data Hierarchy`

#### Node: Structure Data Hierarchy
- **Type / role:** Code node; aggregation/grouping.
- **Configuration choices (interpreted):**
  - Reads **all items** from input (`$input.all()`), each representing one ad insight row.
  - Groups into:
    - `campaigns[campaign_id] = { campaign_id, campaign_name, adsets: { ... } }`
    - `adsets[adset_id] = { adset_id, adset_name, ads: [] }`
  - Pushes `ads[]` entries containing:
    - `ad_id`, `ad_name`
    - `metrics`: spend, reach, impressions, clicks, ctr, cpc, cpm
    - `results`
    - `date`: start/stop
  - Outputs a single item: `{ campaigns: [...] }`
- **Connections:** Main output → `Write Insights`
- **Edge cases / failures:**
  - If the Meta API returns numbers as strings, comparisons/logic are deferred to Gemini; no numeric normalization is done here.
  - Missing `campaign_id` or `adset_id` would create `undefined` keys and messy grouping.
- **Version notes:** `typeVersion 2`

---

### Block 5 — AI Analysis & Structured Output
**Overview:** Sends hierarchical JSON to Gemini with strict output constraints and parses the response into structured items for downstream storage.  
**Nodes involved:** `Google Gemini Chat Model`, `Structured Output Parser`, `Write Insights`, `Sticky Note5`

#### Node: Google Gemini Chat Model
- **Type / role:** LangChain chat model wrapper for Google Gemini; provides LLM to the chain node.
- **Configuration choices:** Defaults (no special options set).
- **Connections:** AI language model output → `Write Insights` (as the model provider)
- **Credentials:** Not shown in JSON, but required in n8n (Gemini/Google AI).
- **Edge cases / failures:**
  - Model credential misconfiguration, quota exceeded, or safety blocks.
  - Latency/timeouts with large payloads (campaign/adset/ad volume).
- **Version notes:** `typeVersion 1`

#### Node: Structured Output Parser
- **Type / role:** LangChain structured output parser; enforces JSON schema.
- **Configuration choices:**
  - Provides an example schema expecting:
    - array of campaigns, each with `campaign_name`, and `adsets[]`
    - each adset contains `adset_name`, `best_performing_ad`, `worst_performing_ad`, `suggestion_for_agency`, `suggestion_for_client`
- **Connections:** AI output parser → `Write Insights`
- **Edge cases / failures:**
  - If the model returns invalid JSON or deviates from schema, parsing fails and triggers error flow.
- **Version notes:** `typeVersion 1.3`

#### Node: Write Insights
- **Type / role:** LangChain “Chain LLM”; orchestrates prompt + model + parser.
- **Configuration choices:**
  - **Prompt text** embeds `{{ JSON.stringify($json.campaigns, null, 2) }}`.
  - Instructs: “Return only valid JSON following this schema…”
  - **System/message content** defines “MetaInsightsGPT” behavior and evaluation rules (CTR/conversions prioritized, ignore zero-impression ads, quantify insights, etc.).
  - Has output parser enabled (`hasOutputParser: true`), connected to `Structured Output Parser`.
- **Connections:** Main output → `Split AI Insights`
- **Critical schema mismatch to note:**
  - The **prompt’s required JSON schema** asks for `adset_suggestion` and `campaign_suggestion`.
  - The **parser schema** expects `suggestion_for_agency` and `suggestion_for_client`.
  - The **Save AI Insights** node expects `suggestion_for_agency` and `suggestion_for_client` but the prompt doesn’t require them.
  - This mismatch is a common cause of parsing or missing-field issues.
- **Version notes:** `typeVersion 1.7`

---

### Block 6 — Insights Storage
**Overview:** Splits the AI output array into separate items and appends rows into Google Sheets.  
**Nodes involved:** `Split AI Insights`, `Save AI Insights`, `Sticky Note6`

#### Node: Split AI Insights
- **Type / role:** Split Out; turns AI output array into one item per campaign (or per element).
- **Configuration:** `fieldToSplitOut = "output"`
- **Connections:** Main output → `Save AI Insights`
- **Edge cases / failures:**
  - Requires that `Write Insights` produces an `output` field containing an array. If the chain node outputs a different property name, this fails.
- **Version notes:** `typeVersion 1`

#### Node: Save AI Insights
- **Type / role:** Google Sheets; append insights rows.
- **Configuration choices:**
  - Operation: **Append**
  - Spreadsheet: `YOUR_SPREADSHEET_ID_HERE`
  - Sheet: `AI_Insights`
  - Column mapping highlights:
    - `Date` = `$now.minus({ days: 1 }).toFormat('yyyy-MM-dd')`
    - `Campaign` = `$json.campaign_name`
    - Adset fields use index `[0]`: `$json.adsets[0].adset_name`, etc.
    - `Agency Suggestion` = `$json.adsets[0].suggestion_for_agency`
    - `Client Suggestion` = `$json.adsets[0].suggestion_for_client`
- **Credentials:** `googleSheetsOAuth2Api`
- **Edge cases / failures:**
  - Only the **first adset** (`adsets[0]`) is saved per campaign item. If the AI returns multiple adsets, they are ignored.
  - If `adsets` is empty, `[0]` access fails.
  - If the model returned `adset_suggestion` instead of `suggestion_for_*`, fields will be blank or expression errors.
- **Version notes:** `typeVersion 4.7`

---

### Block 7 — Error Handling
**Overview:** If any node fails, the workflow sends an email containing error details.  
**Nodes involved:** `Error Trigger`, `Get Config`, `Send Error Email`, `Sticky Note7`

#### Node: Error Trigger
- **Type / role:** Error Trigger; starts this branch on workflow failure.
- **Configuration:** Default.
- **Connections:** Main output → `Get Config`
- **Edge cases / failures:**
  - If error branch nodes also fail (e.g., Gmail auth), you may lose visibility.
- **Version notes:** `typeVersion 1`

#### Node: Get Config
- **Type / role:** Set node; duplicates config for the error branch.
- **Configuration choices:** Same fields as `Set Configuration`:
  - `Ad Account ID`, `Email`
- **Connections:** Main output → `Send Error Email`
- **Edge cases / failures:**
  - If email is placeholder, notifications won’t reach you.
- **Version notes:** `typeVersion 3.4`

#### Node: Send Error Email
- **Type / role:** Gmail; sends error notification.
- **Configuration choices:**
  - To: `$('Get Config').first().json.Email`
  - Subject includes Ad Account ID.
  - HTML body includes:
    - `execution.error.message`
    - `execution.error.stack`
    - `execution.lastNodeExecuted`
  - Attribution disabled.
- **Credentials:** `gmailOAuth2`
- **Edge cases / failures:**
  - Gmail OAuth expired/revoked.
  - Large stack traces could exceed email size limits (rare).
- **Version notes:** `typeVersion 2.1`

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Template Guide | stickyNote | Embedded setup notes and links | — | — | ## Analyze Meta ads with Gemini and Google Sheets… (includes Graph API guide + Sheets template links) |
| Schedule - Once per day | scheduleTrigger | Daily trigger at 06:00 | — | Set Configuration; Get Ad Insights - Ad Level | ## Configuration & schedule • Define run frequency • Set account and reporting parameters |
| Set Configuration | set | Stores Ad Account ID + Email for main flow | Schedule - Once per day | Get Ad Insights - Campaign Level | ## Configuration & schedule • Define run frequency • Set account and reporting parameters |
| Get Ad Insights - Campaign Level | facebookGraphApi | Fetch yesterday campaign-level insights | Set Configuration | Split Campaign Data | ## Campaign-level data collection • Fetch campaign performance data • Remove zero-spend campaigns • Store clean campaign metrics |
| Split Campaign Data | splitOut | Split Meta response `data[]` into items | Get Ad Insights - Campaign Level | Filter Zero Spend | ## Campaign-level data collection • Fetch campaign performance data • Remove zero-spend campaigns • Store clean campaign metrics |
| Filter Zero Spend | filter | Remove campaigns with spend == "0" | Split Campaign Data | Save Campaign Data | ## Campaign-level data collection • Fetch campaign performance data • Remove zero-spend campaigns • Store clean campaign metrics |
| Save Campaign Data | googleSheets | Append campaign metrics to sheet `Campaigns` | Filter Zero Spend | — | ## Campaign-level data collection • Fetch campaign performance data • Remove zero-spend campaigns • Store clean campaign metrics |
| Get Ad Insights - Ad Level | facebookGraphApi | Fetch yesterday ad-level insights | Schedule - Once per day | Split Ad Data | ## Ad-level data collection • Fetch ad-level performance data • Normalize and structure records • Persist raw ad metrics |
| Split Ad Data | splitOut | Split Meta response `data[]` into ad items | Get Ad Insights - Ad Level | Save Ad Data; Structure Data Hierarchy | ## Ad-level data collection • Fetch ad-level performance data • Normalize and structure records • Persist raw ad metrics |
| Save Ad Data | googleSheets | Append ad metrics to sheet `Ads` | Split Ad Data | — | ## Ad-level data collection • Fetch ad-level performance data • Normalize and structure records • Persist raw ad metrics |
| Structure Data Hierarchy | code | Group ads into campaign→adset→ads JSON | Split Ad Data | Write Insights | ## AI analysis & insights • Analyze performance with Gemini • Detect winners and underperformers • Generate actionable recommendations |
| Google Gemini Chat Model | lmChatGoogleGemini | LLM provider for analysis | — | Write Insights (AI model input) | ## AI analysis & insights • Analyze performance with Gemini • Detect winners and underperformers • Generate actionable recommendations |
| Structured Output Parser | outputParserStructured | Enforce JSON schema parsing | — | Write Insights (AI parser input) | ## AI analysis & insights • Analyze performance with Gemini • Detect winners and underperformers • Generate actionable recommendations |
| Write Insights | chainLlm | Prompt Gemini + parse into structured result | Structure Data Hierarchy; Google Gemini Chat Model; Structured Output Parser | Split AI Insights | ## AI analysis & insights • Analyze performance with Gemini • Detect winners and underperformers • Generate actionable recommendations |
| Split AI Insights | splitOut | Split AI `output[]` into items | Write Insights | Save AI Insights | ## Insights storage • Split structured AI output • Save insights for reporting |
| Save AI Insights | googleSheets | Append insights to `AI_Insights` | Split AI Insights | — | ## Insights storage • Split structured AI output • Save insights for reporting |
| Error Trigger | errorTrigger | Starts on any workflow error | — | Get Config | ## Error handling • Catch workflow failures • Send notification email |
| Get Config | set | Stores Ad Account ID + Email for error flow | Error Trigger | Send Error Email | ## Error handling • Catch workflow failures • Send notification email |
| Send Error Email | gmail | Email error details | Get Config | — | ## Error handling • Catch workflow failures • Send notification email |
| Sticky Note1 | stickyNote | Documentation block label | — | — | ## Configuration & schedule • Define run frequency • Set account and reporting parameters |
| Sticky Note2 | stickyNote | Documentation block label | — | — | ## Campaign-level data collection • Fetch campaign performance data • Remove zero-spend campaigns • Store clean campaign metrics |
| Sticky Note4 | stickyNote | Documentation block label | — | — | ## Ad-level data collection • Fetch ad-level performance data • Normalize and structure records • Persist raw ad metrics |
| Sticky Note5 | stickyNote | Documentation block label | — | — | ## AI analysis & insights • Analyze performance with Gemini • Detect winners and underperformers • Generate actionable recommendations |
| Sticky Note6 | stickyNote | Documentation block label | — | — | ## Insights storage • Split structured AI output • Save insights for reporting |
| Sticky Note7 | stickyNote | Documentation block label | — | — | ## Error handling • Catch workflow failures • Send notification email |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named: *Analyze Meta ads daily with Google Gemini and Google Sheets*.

2. **Add trigger**
   1. Add node: **Schedule Trigger** named `Schedule - Once per day`.
   2. Set rule: **Every day at 06:00** (adjust timezone in n8n settings if needed).

3. **Add configuration node**
   1. Add node: **Set** named `Set Configuration`.
   2. Add fields:
      - `Ad Account ID` (string): your Meta Ad Account numeric ID (without `act_`)
      - `Email` (string): notification email address
   3. Connect: `Schedule - Once per day` → `Set Configuration`.

4. **Campaign-level branch (from Set Configuration)**
   1. Add node: **Facebook Graph API** named `Get Ad Insights - Campaign Level`.
      - Graph API version: **v24.0**
      - Node: `act_{{$('Set Configuration').first().json['Ad Account ID']}}`
      - Edge:
        - `insights?fields=campaign_id,campaign_name,spend,impressions,clicks,results,marketing_messages_delivered,actions,reach,ctr,cost_per_result&level=campaign&date_preset=yesterday`
      - Set credentials: **Facebook Graph API** (token with ads read permissions).
   2. Add node: **Split Out** named `Split Campaign Data`
      - Field to split out: `data`
   3. Add node: **Filter** named `Filter Zero Spend`
      - Condition: `$json.spend` **not equals** `"0"`
      - (Recommended improvement: treat `"0.00"` as zero too.)
   4. Add node: **Google Sheets** named `Save Campaign Data`
      - Operation: **Append**
      - Document ID: your spreadsheet ID
      - Sheet name: `Campaigns`
      - Map columns to campaign fields (Date, Campaign ID/Name, Reach, Impressions, Spend, Clicks, CTR, etc.)
      - Credentials: **Google Sheets OAuth2**
   5. Connect in order:
      - `Set Configuration` → `Get Ad Insights - Campaign Level` → `Split Campaign Data` → `Filter Zero Spend` → `Save Campaign Data`

5. **Ad-level branch (from Schedule Trigger)**
   1. Add node: **Facebook Graph API** named `Get Ad Insights - Ad Level`.
      - Graph API version: **v24.0**
      - Node: `act_{{$('Set Configuration').first().json['Ad Account ID']}}`
        - Note: in this workflow JSON, this node is triggered directly from the schedule; to keep that design, either:
          - also connect `Schedule - Once per day` → `Set Configuration` and then `Set Configuration` → `Get Ad Insights - Ad Level`, **or**
          - keep schedule → ad node but ensure `Set Configuration` runs before it by using a single branch. (n8n parallel branches don’t guarantee ordering.)
      - Edge:
        - `insights?fields=ad_id,ad_name,adset_id,adset_name,campaign_id,campaign_name,spend,reach,impressions,clicks,results,ctr,cpc,cpm&level=ad&date_preset=yesterday`
   2. Add node: **Split Out** named `Split Ad Data`
      - Field: `data`
   3. Add node: **Google Sheets** named `Save Ad Data`
      - Operation: **Append**
      - Document ID: your spreadsheet ID
      - Sheet: `Ads`
      - Map columns (campaign/adset/ad identifiers + metrics)
   4. Add node: **Code** named `Structure Data Hierarchy`
      - Paste logic that groups items by `campaign_id` then `adset_id` and outputs `{ campaigns: [...] }` (as in the workflow).
   5. Connect:
      - `Get Ad Insights - Ad Level` → `Split Ad Data`
      - `Split Ad Data` → `Save Ad Data`
      - `Split Ad Data` → `Structure Data Hierarchy`

6. **AI block (Gemini + parsing)**
   1. Add node: **Google Gemini Chat Model** named `Google Gemini Chat Model`
      - Select Gemini credentials (Google AI / Gemini API key depending on your n8n setup).
   2. Add node: **Structured Output Parser** named `Structured Output Parser`
      - Provide schema example (campaign → adsets with best/worst + suggestions).
   3. Add node: **Chain LLM** named `Write Insights`
      - Set prompt to include: `{{ JSON.stringify($json.campaigns, null, 2) }}`
      - Add system/instruction message to enforce:
        - identify best/worst ad per adset
        - quantify CTR/CPC/etc.
        - return valid JSON only
      - Enable structured output parsing and connect parser + model (see next step).
   4. Connect:
      - `Structure Data Hierarchy` → `Write Insights`
      - `Google Gemini Chat Model` → `Write Insights` (AI language model connection)
      - `Structured Output Parser` → `Write Insights` (AI output parser connection)

7. **Insights split + save**
   1. Add node: **Split Out** named `Split AI Insights`
      - Field to split: `output`
   2. Add node: **Google Sheets** named `Save AI Insights`
      - Operation: **Append**
      - Document ID: your spreadsheet ID
      - Sheet: `AI_Insights`
      - Map columns:
        - `Date` = `$now.minus({ days: 1 }).toFormat('yyyy-MM-dd')`
        - `Campaign` = `$json.campaign_name`
        - Adset-related fields (ensure your AI schema matches what you save)
   3. Connect:
      - `Write Insights` → `Split AI Insights` → `Save AI Insights`

8. **Error handling branch**
   1. Add node: **Error Trigger** named `Error Trigger`.
   2. Add node: **Set** named `Get Config` (duplicate configuration fields, or reference a global/static config).
   3. Add node: **Gmail** named `Send Error Email`
      - To: `={{ $('Get Config').first().json.Email }}`
      - Subject: include ad account id
      - Body: include error message/stack/last node executed from the error trigger payload.
      - Credentials: **Gmail OAuth2**
   4. Connect:
      - `Error Trigger` → `Get Config` → `Send Error Email`

9. **Google Sheets preparation**
   - Create a spreadsheet with sheets named exactly:
     - `Campaigns`, `Ads`, `AI_Insights`
   - Ensure columns match the names used in the Google Sheets nodes.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Facebook Graph API Guide | https://docs.google.com/document/d/1ydWNom0TUVh5SkU9IymlpnM_NXbTqHeL_YetDLn-9m0/edit?usp=sharing |
| Google Sheets template | https://docs.google.com/spreadsheets/d/11M6Co13t9t2P7NtFaTm9VXHnzUHXO_RGYf-FRnHkS28/edit?usp=sharing |
| Date-range query format note | `time_range[since]=YYYY-MM-DD&time_range[until]=YYYY-MM-DD` |
| Attribution/credit in sticky note | “Happy setup, Bakdaulet Abdikhan… Feel free to add slack/email nodes to notify alerts/updates ;)” |