Match sponsors to event packages with Google Sheets, GPT-4o and Gmail

https://n8nworkflows.xyz/workflows/match-sponsors-to-event-packages-with-google-sheets--gpt-4o-and-gmail-12620


# Match sponsors to event packages with Google Sheets, GPT-4o and Gmail

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Match sponsors to event packages with Google Sheets, GPT-4o and Gmail  
**Workflow name (in JSON):** Sponsor-Package Matching Agent

**Purpose:**  
Automatically reads a sponsor list and a sponsorship package catalog from Google Sheets, uses GPT‑4o to recommend the top 1–3 best packages for each sponsor, filters out weak matches, then emails the sponsor contact and logs the match results back to Google Sheets.

**Target use cases:**
- Event organizers matching sponsor leads to tiered packages
- Sales ops/partnership teams doing semi-automated qualification + outreach
- Lightweight recommendation/lead routing with an auditable log

### Logical blocks
1. **1.1 Scheduling + Config Bootstrap**  
   Starts on a weekday schedule and sets configurable Google Sheet IDs used throughout.
2. **1.2 Data Acquisition (Google Sheets)**  
   Fetches sponsors and packages from two separate spreadsheets/tabs.
3. **1.3 Package Catalog Aggregation**  
   Aggregates all package rows into a single field so each sponsor can be evaluated against the full catalog.
4. **1.4 Sponsor Iteration + Context Assembly**  
   Loops sponsor-by-sponsor and merges each sponsor with the full package catalog.
5. **1.5 AI Matching (GPT‑4o)**  
   Prompts GPT‑4o to output strict JSON recommendations (name/score/reason + summary).
6. **1.6 Parsing + Normalization + Threshold Filter**  
   Parses the model output, computes top matches + matchScore, and filters out scores below threshold.
7. **1.7 Outreach (Gmail) + Analytics Logging (Google Sheets)**  
   Sends a personalized email and appends a row to a match log sheet; then continues loop.

---

## 2. Block-by-Block Analysis

### 2.1 Scheduling + Config Bootstrap

**Overview:**  
Triggers the workflow on a weekly schedule (weekdays at 09:00) and defines the spreadsheet IDs as variables to avoid hardcoding IDs across nodes.

**Nodes involved:**
- Execute to Start
- Workflow Configuration

#### Node: Execute to Start
- **Type / role:** Schedule Trigger — workflow entry point.
- **Configuration (interpreted):**
  - Runs at **09:00** on **Monday–Friday** (week interval rule with `triggerAtDay` 1..5 and `triggerAtHour` 9).
- **Outputs:** Triggers the flow into **Workflow Configuration**.
- **Edge cases / failures:**
  - n8n timezone settings matter (instance timezone vs expected business timezone).
  - If the instance is asleep/down at trigger time, execution may be missed depending on n8n setup.

#### Node: Workflow Configuration
- **Type / role:** Set node — central configuration variables.
- **Configuration:**
  - Creates three string fields:
    - `sponsorsSheetId` (placeholder value)
    - `packagesSheetId` (placeholder value)
    - `logSheetId` (placeholder value)
  - **Include other fields:** enabled (keeps incoming data, though trigger produces minimal fields).
- **Key expressions/variables:**
  - Downstream nodes reference:
    - `{{ $('Workflow Configuration').first().json.sponsorsSheetId }}`
    - `{{ $('Workflow Configuration').first().json.packagesSheetId }}`
    - `{{ $('Workflow Configuration').first().json.logSheetId }}`
- **Connections:**
  - Outputs to both **Fetch Sponsors Sheet** and **Fetch Packages Sheet** in parallel.
- **Edge cases / failures:**
  - If placeholders are not replaced with valid Google Sheet IDs, all downstream Google Sheets calls will fail (file not found / permission errors).

---

### 2.2 Data Acquisition (Google Sheets)

**Overview:**  
Reads raw sponsor rows and package rows from Google Sheets tabs named “Sponsors” and “Packages”.

**Nodes involved:**
- Fetch Sponsors Sheet
- Fetch Packages Sheet

#### Node: Fetch Sponsors Sheet
- **Type / role:** Google Sheets — reads sponsor dataset.
- **Configuration:**
  - Document ID comes from `Workflow Configuration.sponsorsSheetId`.
  - Sheet/tab name: **Sponsors**
  - `returnFirstMatch: false` (ensures all rows are returned rather than a single match).
- **Credentials:** Google Sheets OAuth2 (`Google Sheets account`).
- **Outputs:** Sponsor items (each row becomes an item).
- **Connections:** To **Loop Over Sponsors**.
- **Edge cases / failures:**
  - Missing tab name “Sponsors” → sheet not found error.
  - OAuth scope/permissions: the account must have at least read access.
  - Column naming consistency: later nodes assume fields like `SponsorName`, `Industry`, `Budget`, `Goals`, `Notes`, `OwnerEmail`.

#### Node: Fetch Packages Sheet
- **Type / role:** Google Sheets — reads package catalog.
- **Configuration:**
  - Document ID from `Workflow Configuration.packagesSheetId`.
  - Sheet/tab name: **Packages**
- **Credentials:** same Google Sheets OAuth2 account.
- **Outputs:** Package items (each row becomes an item).
- **Connections:** To **Aggregate Packages**.
- **Edge cases / failures:**
  - Missing tab name “Packages”.
  - Inconsistent package schema can reduce AI recommendation quality (e.g., missing price/benefits).

---

### 2.3 Package Catalog Aggregation

**Overview:**  
Converts multiple package rows into a single aggregated field (`allPackages`) so each sponsor can be scored against the entire catalog in one prompt.

**Nodes involved:**
- Aggregate Packages

#### Node: Aggregate Packages
- **Type / role:** Aggregate — combine many items into one item.
- **Configuration:**
  - Aggregation mode: **aggregate all item data**
  - Destination field: `allPackages`
- **Input:** All package row items from **Fetch Packages Sheet**.
- **Output:** A single item containing `allPackages` (an array-like structure of all packages).
- **Connections:** To **Merge Sponsor with Packages** (input index 1).
- **Edge cases / failures:**
  - Very large package catalogs can create huge prompts later (token limits / cost).
  - Data shape of `allPackages` affects prompt readability; if packages contain nested objects/empty rows, GPT output reliability can drop.

---

### 2.4 Sponsor Iteration + Context Assembly

**Overview:**  
Processes sponsors one-by-one, pairing each sponsor item with the aggregated package catalog, and ensures `allPackages` is available as a string for the AI prompt.

**Nodes involved:**
- Loop Over Sponsors
- Merge Sponsor with Packages
- Prepare AI Context

#### Node: Loop Over Sponsors
- **Type / role:** Split In Batches — iteration controller.
- **Configuration:**
  - Default batch behavior (no explicit batch size shown; uses node default).
  - `reset: false` (keeps state behavior as configured).
- **Input:** Sponsor items from **Fetch Sponsors Sheet**.
- **Output:** One batch at a time to **Merge Sponsor with Packages** (input index 0).
- **Connections:**
  - Receives a “continue” loop-back from **Log Match Results** to process next sponsor.
- **Edge cases / failures:**
  - If downstream nodes error, the loop stops mid-list.
  - Batch size defaults can impact execution time and rate limits (Gmail/OpenAI).

#### Node: Merge Sponsor with Packages
- **Type / role:** Merge — attach package catalog to each sponsor item.
- **Configuration:**
  - Mode: **combine**
  - Combine by: **position**
  - `includeUnpaired: true` (if one input has fewer items, still outputs items).
- **Inputs:**
  - Input 0: sponsor batch item(s) from **Loop Over Sponsors**
  - Input 1: aggregated packages (single item) from **Aggregate Packages**
- **Output:** Sponsor item enriched with package catalog (position-based combine).
- **Connections:** To **Prepare AI Context**.
- **Edge cases / failures:**
  - Combine-by-position assumes the first sponsor aligns with the first (and only) aggregated packages item. This generally works because packages side is a single item; if it ever becomes multiple items, pairing may break.

#### Node: Prepare AI Context
- **Type / role:** Set — normalize `allPackages` for the prompt.
- **Configuration:**
  - Sets `allPackages` as **string**:
    - `={{ $input.last().json.allPackages }}`
  - Keeps all other sponsor fields (`includeOtherFields: true`).
- **Inputs/Outputs:**
  - Input: merged sponsor+packages item.
  - Output: same item with `allPackages` ensured on the sponsor item.
- **Edge cases / failures:**
  - `$input.last()` assumes the last input contains the packages aggregate. If merge behavior changes, this could become incorrect.
  - If `allPackages` is an array/object, forcing to string may yield `[object Object]`-like results depending on n8n casting; ideally JSON.stringify could be safer.

---

### 2.5 AI Matching (GPT‑4o)

**Overview:**  
Calls GPT‑4o with sponsor attributes plus the full package list and asks for strict JSON containing top recommendations and a summary.

**Nodes involved:**
- AI Match Packages

#### Node: AI Match Packages
- **Type / role:** OpenAI (LangChain) — LLM inference for matching/scoring.
- **Configuration:**
  - Model: `gpt-4o`
  - Prompt instructs:
    - Analyze sponsor details (`SponsorName`, `Industry`, `Budget`, `Goals`, `Notes`)
    - Use `Available Packages: {{ $json.allPackages }}`
    - Return **valid JSON** in a required schema:
      - `recommendations[]`: `{packageName, score(0-100), reason}`
      - `summaryText`: paragraph
- **Credentials:** OpenAI API (`n8n free OpenAI API credits`).
- **Input:** Sponsor item enriched with `allPackages`.
- **Output:** A message object containing `message.content` (the model’s response text).
- **Connections:** To **Parse AI Response**.
- **Edge cases / failures:**
  - Model may return non-JSON (extra prose, trailing commas, code fences) → downstream parse failures.
  - Token limits: large `allPackages` + sponsor notes can exceed context window.
  - Cost/rate limit throttling if many sponsors processed in one run.

---

### 2.6 Parsing + Normalization + Threshold Filter

**Overview:**  
Attempts to parse the AI JSON, extracts top recommendation fields, then normalizes robustly in code and filters out sponsors whose best match is below 70.

**Nodes involved:**
- Parse AI Response
- Parse Matches
- Filter Low Scores

#### Node: Parse AI Response
- **Type / role:** Set — initial JSON parsing and quick field extraction.
- **Configuration:**
  - `aiResponse` (object): `={{ JSON.parse($json.message.content) }}`
  - `topPackage` (string): first recommendation package name
  - `topScore` (number): first recommendation score
  - Keeps other fields.
- **Connections:** To **Parse Matches**.
- **Edge cases / failures:**
  - If `message.content` is not valid JSON, **this node will throw** and stop execution (no try/catch here).
  - If `recommendations` is empty/missing, indexing `[0]` can fail.

#### Node: Parse Matches
- **Type / role:** Code — robust parsing + shaping final fields used by email/logging.
- **Configuration (logic):**
  - Reads AI output from `$json.message?.content` or `$json.aiResponse`.
  - Tries JSON.parse if string; on failure sets `{ recommendations: [] }`.
  - Defines:
    - `topMatches`: first 3 recommendations
    - `bestPackage`: first of topMatches
    - `matchScore`: `bestPackage.score` else 0
    - `aiResponse`: parsed object
- **Output fields used later:**
  - `topMatches` (array)
  - `bestPackage` (object)
  - `matchScore` (number)
- **Connections:** To **Filter Low Scores**.
- **Edge cases / failures:**
  - This node is robust, but it may never run if **Parse AI Response** fails first.
  - If GPT returns score as string (e.g., `"95"`), matchScore remains `"95"`; IF node uses “loose” type validation, but numeric comparisons can still be inconsistent.

#### Node: Filter Low Scores
- **Type / role:** IF — gate outreach/logging to only strong matches.
- **Condition:**
  - `{{ $json.matchScore }}` **>= 70**
- **Connections:**
  - True path → **Send match email summary** and **Log Match Results** (both in parallel).
  - False path → no further nodes (sponsor is silently skipped).
- **Edge cases / failures:**
  - If `matchScore` is undefined/null, comparison can behave unexpectedly; here it will likely fail the condition and skip.
  - Threshold mismatch with sticky notes: sticky notes mention “score 1–10” and “IF >7”, but the workflow uses **0–100** and **>=70**.

---

### 2.7 Outreach (Gmail) + Analytics Logging (Google Sheets)

**Overview:**  
For qualifying sponsors, sends a personalized HTML email containing ranked matches and writes an audit log row to a MatchLog sheet, then continues the sponsor loop.

**Nodes involved:**
- Send match email summary
- Log Match Results

#### Node: Send match email summary
- **Type / role:** Gmail — send email.
- **Configuration:**
  - To: `={{ $json.OwnerEmail }}`
  - Subject: `🎯 {{ $json.SponsorName }} - Top Package Matches (Score: {{ $json.matchScore }})`
  - HTML body builds a bullet list from:
    - `{{ $json.topMatches.map(...) }}`
  - Mentions best option: `{{ $json.bestPackage.packageName }}`
- **Credentials:** Gmail OAuth2 (`Gmail account`).
- **Input:** Sponsor item with `topMatches`, `bestPackage`, `matchScore`.
- **Edge cases / failures:**
  - Missing/invalid `OwnerEmail` → send failure.
  - Gmail API sending limits/quota; bulk sends can be rate limited.
  - Subject includes a leading emoji; may not be desired in some corporate environments.

#### Node: Log Match Results
- **Type / role:** Google Sheets — append row to analytics log.
- **Configuration:**
  - Operation: **append**
  - Document ID: `Workflow Configuration.logSheetId`
  - Sheet/tab name: **MatchLog**
  - Columns mapped:
    - `Timestamp`: `{{ $now.toISO() }}`
    - `SponsorName`: `{{ $json.SponsorName }}`
    - `RecommendedPackages`: `{{ $json.topMatches.map(m => m.packageName).join(', ') }}`
    - `TopScore`: `{{ $json.matchScore }}`
  - Schema defines these four columns.
- **Connections:** Output loops back into **Loop Over Sponsors** to process the next sponsor.
- **Edge cases / failures:**
  - Tab “MatchLog” must exist with matching column headers (or mapping must match how n8n writes).
  - If Google Sheets API quota is hit, the loop will stop and remaining sponsors won’t be processed.
  - If `topMatches` is empty, `RecommendedPackages` becomes empty string (fine) and `TopScore` is 0.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Execute to Start | Schedule Trigger | Entry point; runs weekdays at 09:00 | — | Workflow Configuration | # **Ai sponsor matching agent** / Matches sponsors to packages using Google Sheets + GPT-4o + Gmail / No hardcoded IDs—100% configurable. Error-proof filtering + analytics tracking. / Steps + setup notes (see sticky content). |
| Workflow Configuration | Set | Central config variables (sheet IDs) | Execute to Start | Fetch Sponsors Sheet; Fetch Packages Sheet | # **Ai sponsor matching agent** / Matches sponsors to packages using Google Sheets + GPT-4o + Gmail / No hardcoded IDs—100% configurable. Error-proof filtering + analytics tracking. / Steps + setup notes (see sticky content). |
| Fetch Sponsors Sheet | Google Sheets | Read sponsors from “Sponsors” tab | Workflow Configuration | Loop Over Sponsors | #### Data sources / Reads sponsors and packages from two Google Sheets. / Sponsors: Name, lndustry, Budget, Goals, Email / Packages: Name, Price, Benefits |
| Fetch Packages Sheet | Google Sheets | Read packages from “Packages” tab | Workflow Configuration | Aggregate Packages | #### Data sources / Reads sponsors and packages from two Google Sheets. / Sponsors: Name, lndustry, Budget, Goals, Email / Packages: Name, Price, Benefits |
| Aggregate Packages | Aggregate | Combine all package rows into `allPackages` | Fetch Packages Sheet | Merge Sponsor with Packages |  |
| Loop Over Sponsors | Split In Batches | Iterate through sponsors list | Fetch Sponsors Sheet; Log Match Results (loopback) | Merge Sponsor with Packages |  |
| Merge Sponsor with Packages | Merge | Attach package catalog to each sponsor | Loop Over Sponsors; Aggregate Packages | Prepare AI Context |  |
| Prepare AI Context | Set | Ensure `allPackages` is present for prompt | Merge Sponsor with Packages | AI Match Packages |  |
| AI Match Packages | OpenAI (LangChain) | Score and recommend packages via GPT‑4o | Prepare AI Context | Parse AI Response | #### AI matching engine / GPT analyzes sponsor profile + full package catalog, scores 1-10 + reasoning per package / 1. GPT scores TOP3 (JSON) / 2. IF >7 → proceed |
| Parse AI Response | Set | JSON.parse of model output; extract top pkg/score | AI Match Packages | Parse Matches |  |
| Parse Matches | Code | Robust parsing; compute `topMatches`, `bestPackage`, `matchScore` | Parse AI Response | Filter Low Scores |  |
| Filter Low Scores | IF | Gate: only proceed if score >= 70 | Parse Matches | Send match email summary; Log Match Results |  |
| Send match email summary | Gmail | Email sponsor contact with recommendations | Filter Low Scores (true) | — | #### Outreach + Analytics / Sends personalized email per sponsor with top matches (Package and Score) / Records match history : who + what + scores to Google Sheets |
| Log Match Results | Google Sheets | Append match record to MatchLog; drives loop | Filter Low Scores (true) | Loop Over Sponsors | #### Outreach + Analytics / Sends personalized email per sponsor with top matches (Package and Score) / Records match history : who + what + scores to Google Sheets |
| Sticky Note | Sticky Note | Comment / credits | — | — | © 2025 Lucas Peyrin |
| Sticky Note1 | Sticky Note | Comment: data sources | — | — | © 2025 Lucas Peyrin |
| Sticky Note2 | Sticky Note | Comment: AI matching engine | — | — | © 2025 Lucas Peyrin |
| Sticky Note3 | Sticky Note | Comment: outreach + analytics | — | — | © 2025 Lucas Peyrin |
| Sticky Note12 | Sticky Note | Comment: feedback + links | — | — | © 2025 Lucas Peyrin / Includes links (see notes section). |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **Sponsor-Package Matching Agent**
   - (Optional) Add tags as needed.

2. **Add trigger: Schedule**
   - Node: **Schedule Trigger** (“Execute to Start”)
   - Configure: run **Mon–Fri at 09:00** (set timezone appropriately in n8n or in node if available).

3. **Add configuration node**
   - Node: **Set** (“Workflow Configuration”)
   - Add fields (strings):
     - `sponsorsSheetId` = your Sponsors spreadsheet ID
     - `packagesSheetId` = your Packages spreadsheet ID
     - `logSheetId` = your Match Log spreadsheet ID
   - Enable “Include Other Fields”.

4. **Add Google Sheets read: Sponsors**
   - Node: **Google Sheets** (“Fetch Sponsors Sheet”)
   - Credentials: configure **Google Sheets OAuth2** with access to the sponsor spreadsheet.
   - Operation: read/get rows (default read operation in your n8n version).
   - Document ID: expression `{{$('Workflow Configuration').first().json.sponsorsSheetId}}`
   - Sheet name: `Sponsors`

5. **Add Google Sheets read: Packages**
   - Node: **Google Sheets** (“Fetch Packages Sheet”)
   - Same credentials (or another account with access).
   - Document ID: `{{$('Workflow Configuration').first().json.packagesSheetId}}`
   - Sheet name: `Packages`

6. **Aggregate packages into one field**
   - Node: **Aggregate** (“Aggregate Packages”)
   - Mode: aggregate all item data
   - Destination field: `allPackages`

7. **Add sponsor loop**
   - Node: **Split In Batches** (“Loop Over Sponsors”)
   - Leave defaults (or set batch size if you expect many sponsors).
   - Connect: **Fetch Sponsors Sheet → Loop Over Sponsors**

8. **Merge sponsor with packages**
   - Node: **Merge** (“Merge Sponsor with Packages”)
   - Mode: **Combine**
   - Combine by: **Position**
   - Include unpaired: enabled
   - Connect:
     - **Loop Over Sponsors → Merge** (Input 1 / index 0)
     - **Aggregate Packages → Merge** (Input 2 / index 1)

9. **Prepare AI context**
   - Node: **Set** (“Prepare AI Context”)
   - Add field:
     - `allPackages` (string) = `{{$input.last().json.allPackages}}`
   - Keep “Include Other Fields” enabled.
   - Connect: **Merge Sponsor with Packages → Prepare AI Context**

10. **Add OpenAI matching node**
   - Node: **OpenAI** (LangChain) (“AI Match Packages”)
   - Credentials: configure **OpenAI API** key/credentials.
   - Model: `gpt-4o`
   - Prompt: include sponsor fields and `{{$json.allPackages}}`, and instruct the model to return strict JSON:
     - `recommendations[]` with `packageName`, `score` (0–100), `reason`
     - `summaryText`
   - Connect: **Prepare AI Context → AI Match Packages**

11. **Parse AI response (strict parse)**
   - Node: **Set** (“Parse AI Response”)
   - Fields:
     - `aiResponse` (object) = `{{ JSON.parse($json.message.content) }}`
     - `topPackage` (string) = `{{ JSON.parse($json.message.content).recommendations[0].packageName }}`
     - `topScore` (number) = `{{ JSON.parse($json.message.content).recommendations[0].score }}`
   - Connect: **AI Match Packages → Parse AI Response**
   - Note: this step is brittle; consider removing it and letting the Code node handle parsing.

12. **Normalize matches (robust)**
   - Node: **Code** (“Parse Matches”)
   - Mode: run once for each item
   - Code logic:
     - Parse JSON if possible, else fallback to empty recommendations.
     - Compute `topMatches` (max 3), `bestPackage`, `matchScore`.
   - Connect: **Parse AI Response → Parse Matches**

13. **Filter low scores**
   - Node: **IF** (“Filter Low Scores”)
   - Condition: `{{$json.matchScore}}` **greater or equal** `70`
   - Connect: **Parse Matches → Filter Low Scores** (main)

14. **Send email via Gmail**
   - Node: **Gmail** (“Send match email summary”)
   - Credentials: configure **Gmail OAuth2** (sender mailbox).
   - To: `{{$json.OwnerEmail}}`
   - Subject: `🎯 {{$json.SponsorName}} - Top Package Matches (Score: {{$json.matchScore}})`
   - Body: HTML list built from `topMatches`, include `bestPackage.packageName`.
   - Connect: **Filter Low Scores (true) → Send match email summary**

15. **Log results to Google Sheets**
   - Node: **Google Sheets** (“Log Match Results”)
   - Credentials: Google Sheets OAuth2 (must have write access to log sheet).
   - Operation: **Append**
   - Document ID: `{{$('Workflow Configuration').first().json.logSheetId}}`
   - Sheet name: `MatchLog`
   - Map columns:
     - SponsorName = `{{$json.SponsorName}}`
     - RecommendedPackages = `{{$json.topMatches.map(m => m.packageName).join(', ')}}`
     - TopScore = `{{$json.matchScore}}`
     - Timestamp = `{{$now.toISO()}}`
   - Connect: **Filter Low Scores (true) → Log Match Results**

16. **Close the loop**
   - Connect: **Log Match Results → Loop Over Sponsors**
   - This continues processing the next sponsor batch item.

17. **Prepare your Google Sheets**
   - Sponsors tab (`Sponsors`) should include at least:
     - `SponsorName`, `Industry`, `Budget`, `Goals`, `Notes` (optional), `OwnerEmail`
   - Packages tab (`Packages`) should include useful descriptors:
     - `Name`, `Price`, `Benefits` (or similar)
   - MatchLog tab (`MatchLog`) should include headers:
     - `SponsorName`, `RecommendedPackages`, `TopScore`, `Timestamp`

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| © 2025 Lucas Peyrin | Credits shown in sticky notes |
| “Ai sponsor matching agent” summary: matches sponsors to packages using Google Sheets + GPT‑4o + Gmail; configurable IDs; filtering + analytics tracking; setup notes mention vars and columns. | Sticky note content embedded in workflow canvas |
| Feedback / contact form link: **https://tally.so/r/EkKGgB** | “Start the conversation here” (sticky note) |
| LinkedIn: **https://linkedin.com/in/MiloBravo/** | Signature in sticky note |
| BRaiA Labs | Mentioned in sticky note |
| Embedded image/gif link (as provided): https://vptkuqoipqbebipqjnqw.supabase.co/storage/v1/object/public/Milo%20Bravo/seeAxWUupcOOXY5tntexZ_video.gif | Used inside the feedback section |

