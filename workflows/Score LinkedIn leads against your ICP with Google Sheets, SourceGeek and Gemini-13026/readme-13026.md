Score LinkedIn leads against your ICP with Google Sheets, SourceGeek and Gemini

https://n8nworkflows.xyz/workflows/score-linkedin-leads-against-your-icp-with-google-sheets--sourcegeek-and-gemini-13026


# Score LinkedIn leads against your ICP with Google Sheets, SourceGeek and Gemini

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Score LinkedIn leads against your ICP with Google Sheets, SourceGeek and Gemini  
**Workflow name (in JSON):** LeadGen SourceGeek - Jan 26 - ICP Score  
**Purpose:** Pull LinkedIn profile URLs from a Google Sheet, enrich each profile using SourceGeek, ask a Gemini-powered AI Agent to (a) score the lead against a defined Ideal Customer Profile (ICP) with reasoning and (b) generate three outreach messages, then write the results back into the same Google Sheet.

### 1.1 Entry & Lead Selection (Google Sheets)
- Manual start, then fetches rows from a Google Sheet based on simple filters (Ready = “y” and ICP Score = “=”).
- Output is a list of leads (each lead contains at least a LinkedIn profile URL).

### 1.2 Profile Enrichment (SourceGeek)
- For each row, calls SourceGeek to retrieve full LinkedIn profile data.

### 1.3 AI Scoring & Message Generation (Gemini + AI Agent + Memory)
- Sends enriched profile JSON to an AI Agent.
- Agent uses Gemini Pro as the language model, plus a buffer memory node (configured but effectively non-persistent as sessionKey is `{}`).
- Agent returns **JSON-only**: `linkedin_profile`, `icp_score`, `icp_reasoning`, `message_1`, `message_2`, `message_3`.

### 1.4 Normalization & Batch Update Loop (Code + Split in Batches + Google Sheets Update)
- A Code node parses and normalizes the agent output into predictable fields.
- A “Loop Over Items” (Split in Batches) node is used as a loop controller for repeated updates back into the sheet.
- Google Sheets “Update row” updates matching rows by `LinkedIn profile URL`, filling score, reasoning, messages, and profile image URL.

---

## 2. Block-by-Block Analysis

### Block 1 — Entry & Lead Selection
**Overview:** Starts the workflow manually and reads qualifying rows from Google Sheets using filter conditions (Ready + missing/unset ICP Score).  
**Nodes involved:**  
- When clicking ‘Execute workflow’
- Get LinkedIn profile urls from sheet

#### Node: When clicking ‘Execute workflow’
- **Type / Role:** Manual Trigger (`n8n-nodes-base.manualTrigger`) — interactive start for testing or on-demand runs.
- **Configuration choices:** No parameters.
- **Connections:**
  - **Output →** Get LinkedIn profile urls from sheet
- **Edge cases / failures:**
  - None (only triggers execution).

#### Node: Get LinkedIn profile urls from sheet
- **Type / Role:** Google Sheets (`n8n-nodes-base.googleSheets`) — reads rows from a specific spreadsheet tab.
- **Operation intent (inferred):** Read rows (the JSON shows filtersUI; typical operation is “Read” / “Get Many”).
- **Key configuration:**
  - **Document:** “Lead Get SourceGeek - Jan 26” (Spreadsheet ID `1TrxzQGP5BetWRBcupPV1bBZrgNEQizmY5_TIolNg5Dw`)
  - **Sheet/tab:** “Data” (`gid=0`)
  - **Filters (filtersUI):**
    - `ICP Score` equals `=`
    - `Ready` equals `y`
  - **Credentials:** Google Sheets OAuth2 (“Google Sheets account”)
- **Input / Output:**
  - **Input ←** Manual Trigger
  - **Output →** Get data from a linkedin profile (one item per matching row)
- **Edge cases / failures:**
  - OAuth expired/invalid scopes; spreadsheet not shared with the credential account.
  - Filter mismatch: if “ICP Score” isn’t literally “=” (or the sheet doesn’t contain that exact value), you may pull zero rows.
  - Column name drift: if headers change, `$json['LinkedIn profile URL']` used later can break.

**Sticky note covering this block (applies to involved nodes):**
- “Retrieve the data from a Google Sheet. The only needed value is the LinkedIn profile url”

---

### Block 2 — LinkedIn Profile Enrichment
**Overview:** For each selected row, enriches the LinkedIn URL by retrieving detailed profile data via SourceGeek.  
**Nodes involved:**  
- Get data from a linkedin profile

#### Node: Get data from a linkedin profile
- **Type / Role:** SourceGeek (`n8n-nodes-sourcegeek.sourcegeek`) — LinkedIn profile enrichment.
- **Key configuration:**
  - **LinkedIn URL input:** `={{ $json['LinkedIn profile URL'] }}`
  - **Credentials:** “Sourcegeek Credentials account”
- **Input / Output:**
  - **Input ←** Get LinkedIn profile urls from sheet
  - **Output →** AI Agent creating the ICP score and reasoning
- **Key data used downstream:**
  - The agent prompt uses `{{ JSON.stringify($json.data) }}` meaning SourceGeek must return a `data` object.
  - Update node later reads profile image via:  
    `{{ $('Get data from a linkedin profile').item.json.data.profileImageUrl }}`
- **Edge cases / failures:**
  - Invalid/empty LinkedIn URL in the sheet row.
  - SourceGeek API auth failures, rate limits, or partial data (e.g., no `profileImageUrl`).
  - If SourceGeek returns errors in a different shape (no `data`), the AI prompt and update mapping can break.

**Sticky note covering this block:**
- “Enrich the LinkedIn profile url with all the info from the profile and add soft skills on top”

---

### Block 3 — AI Scoring & Message Generation (Gemini + Agent + Memory)
**Overview:** Uses a Gemini chat model to power an AI Agent that scores each profile against a detailed ICP definition and generates three personalized outreach messages.  
**Nodes involved:**  
- Google Gemini Chat Model
- Simple Memory
- AI Agent creating the ICP score and reasoning

#### Node: Google Gemini Chat Model
- **Type / Role:** LangChain Google Gemini chat model (`@n8n/n8n-nodes-langchain.lmChatGoogleGemini`) — provides LLM capability to the agent.
- **Key configuration:**
  - **Model:** `models/gemini-pro-latest`
  - **Credentials:** Google PaLM / Gemini (“AdGibbon account”)
- **Connections:**
  - **AI language model →** AI Agent creating the ICP score and reasoning (as `ai_languageModel`)
- **Edge cases / failures:**
  - API key invalid / quota exceeded.
  - Model name not available in some regions/accounts.
  - Safety filters could refuse generation depending on content, though this use case is typical sales text.

#### Node: Simple Memory
- **Type / Role:** Buffer window memory (`@n8n/n8n-nodes-langchain.memoryBufferWindow`) — supplies conversational memory to the agent.
- **Key configuration:**
  - **sessionIdType:** customKey
  - **sessionKey:** `{}` (literal string / expression resolving to `{}` in config)
- **Connections:**
  - **AI memory →** AI Agent creating the ICP score and reasoning (as `ai_memory`)
- **Practical implication:**
  - With `sessionKey` effectively constant/empty, memory may not segment per lead; depending on node behavior, it could be unused or shared.
- **Edge cases / failures:**
  - If memory is shared across items, messages may “bleed” context between different leads. If you want isolation per lead, use a unique key (e.g., LinkedIn URL).

#### Node: AI Agent creating the ICP score and reasoning
- **Type / Role:** LangChain Agent (`@n8n/n8n-nodes-langchain.agent`) — orchestrates prompt + model + memory.
- **Key configuration choices (interpreted):**
  - **PromptType:** “define”
  - **System/task prompt includes:**
    - Role: salesperson at SourceGeek
    - Task: compute ICP score (0–100), reasoning, and 3 outreach messages
    - Rules: authority, seniority, tech savviness; DISC-style language; avoid “—”; informal Dutch pronouns (je/jij); sign as “Niels Berkhout”
    - **Input:** `{{ JSON.stringify($json.data) }}` (full enriched profile payload)
    - **Output constraint:** return only valid JSON with specified fields and no extra explanation
    - **ICP definition:** embedded long ICP segmentation and anti-ICP
    - Language note: output can be in Dutch
  - **Dependencies:** requires `$json.data.linkedinUsername` to form `linkedin_profile` example in prompt (though the agent is free to output any URL).
- **Input / Output:**
  - **Input ←** Get data from a linkedin profile
  - **Output →** JS code creating an array (agent output is expected in `item.json.output` or similar)
- **Edge cases / failures:**
  - Agent returns non-JSON, JSON wrapped in markdown fences, or extra commentary despite instructions (handled later by Code node partially).
  - Missing `data` fields may reduce personalization quality or produce nulls.
  - Very long profile JSON could exceed model context limits; consider trimming `data` to essentials if needed.

**Sticky note covering this block:**
- “Describe in detail your Ideal Customer Profile (ICP) and let it match”

---

### Block 4 — Output Normalization & Loop Control
**Overview:** Parses the agent’s output into structured fields and uses a batching/loop node to repeatedly update the spreadsheet per item.  
**Nodes involved:**  
- JS code creating an array
- Loop Over Items

#### Node: JS code creating an array
- **Type / Role:** Code node (`n8n-nodes-base.code`) — transforms items by parsing agent output into stable keys.
- **Key configuration:**
  - Parses `item.json?.output ?? item.output`
  - Cleans possible markdown fences ```json … ```
  - On parse failure returns an object containing:
    - `_parse_error`, `_error_message`, `_raw_output`
  - Outputs normalized structure:
    - `linkedin_profile`, `icp_score`, `icp_reasoning`, `message_1`, `message_2`, `message_3`, `_parsed`
- **Input / Output:**
  - **Input ←** AI Agent creating the ICP score and reasoning
  - **Output →** Loop Over Items
- **Edge cases / failures:**
  - If agent output is nested differently (e.g., `item.json.text`), parsing will yield nulls.
  - If parse fails, downstream update writes blanks unless you add checks/branches.

#### Node: Loop Over Items
- **Type / Role:** Split in Batches (`n8n-nodes-base.splitInBatches`) — used here as a loop controller.
- **Key configuration:**
  - Batch size not explicitly set in the provided JSON (defaults depend on node/version; commonly 1 or configured in UI).
  - Options empty.
- **Connections (important):**
  - **Input ←** JS code creating an array
  - **Output (main index 1) →** Update row in sheet
  - **Update row in sheet →** Loop Over Items (creates the loop)
- **How the loop behaves (in practice):**
  - “Split in Batches” typically sends the next batch when it receives control back. This workflow wires Update → Loop to continue iterating.
- **Edge cases / failures:**
  - Misconfigured batch size can update multiple rows at once or behave unexpectedly.
  - If “Update row” errors, the loop may stop mid-way (unless error handling is configured globally).

**Sticky note covering this block:**
- “Break up the data in a usable array”

---

### Block 5 — Writeback to Google Sheets
**Overview:** Updates the originating Google Sheet row by matching on LinkedIn profile URL, writing ICP score, reasoning, outreach messages, and profile image.  
**Nodes involved:**  
- Update row in sheet

#### Node: Update row in sheet
- **Type / Role:** Google Sheets (`n8n-nodes-base.googleSheets`) — updates existing rows.
- **Operation:** Update
- **Key configuration:**
  - **Document:** same spreadsheet as input
  - **Sheet/tab:** “Data” (`gid=0`)
  - **Matching column:** `LinkedIn profile URL`
  - **Mapping mode:** “defineBelow”
  - **Columns written (expressions):**
    - `ICP Score` = `{{ $json.icp_score }}`
    - `ICP reasoning` = `{{ $json.icp_reasoning }}`
    - `First message` = `{{ $json.message_1 }}`
    - `Second message` = `{{ $json.message_2 }}`
    - `Third message` = `{{ $json.message_3 }}`
    - `LinkedIn profile URL` = `{{ $json.linkedin_profile }}`
    - `LinkedIn profile image` = `{{ $('Get data from a linkedin profile').item.json.data.profileImageUrl }}`
  - **Credentials:** Google Sheets OAuth2 (“Google Sheets account”)
- **Input / Output:**
  - **Input ←** Loop Over Items
  - **Output →** Loop Over Items (to continue loop)
- **Edge cases / failures:**
  - If `linkedin_profile` produced by AI does not exactly match the sheet’s URL format, matching may fail and no row will be updated.
    - Example mismatch: sheet stores `https://www.linkedin.com/in/foo/` but agent returns without trailing slash.
  - If multiple rows share the same LinkedIn URL, updates may affect the wrong row (or multiple rows depending on node behavior/version).
  - If SourceGeek profile image URL is missing, `LinkedIn profile image` becomes empty.
  - Rate limits or Google API errors when updating many rows.

**Sticky note covering this block:**
- “Insert all the data back in the Google Sheet. All the profile data & the ICP Score”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | Manual Trigger | Manual entry point | — | Get LinkedIn profile urls from sheet | ## ICP score and Message generator |
| Get LinkedIn profile urls from sheet | Google Sheets | Read/filter lead rows from sheet | When clicking ‘Execute workflow’ | Get data from a linkedin profile | Retrieve the data from a Google Sheet. The only needed value is the LinkedIn profile url |
| Get data from a linkedin profile | SourceGeek | Enrich LinkedIn profile | Get LinkedIn profile urls from sheet | AI Agent creating the ICP score and reasoning | Enrich the LinkedIn profile url with all the info from the profile and add soft skills on top |
| Google Gemini Chat Model | Google Gemini Chat Model (LangChain) | LLM powering the agent | — | AI Agent creating the ICP score and reasoning | Describe in detail your Ideal Customer Profile (ICP) and let it match |
| Simple Memory | Memory Buffer Window (LangChain) | Provides conversational memory to agent | — | AI Agent creating the ICP score and reasoning | Describe in detail your Ideal Customer Profile (ICP) and let it match |
| AI Agent creating the ICP score and reasoning | LangChain Agent | Compute ICP score + reasoning + 3 messages | Get data from a linkedin profile | JS code creating an array | Describe in detail your Ideal Customer Profile (ICP) and let it match |
| JS code creating an array | Code | Parse/normalize agent JSON output | AI Agent creating the ICP score and reasoning | Loop Over Items | Break up the data in a usable array |
| Loop Over Items | Split in Batches | Iteration/loop controller for updates | JS code creating an array; Update row in sheet | Update row in sheet | Break up the data in a usable array |
| Update row in sheet | Google Sheets | Update matching row with results | Loop Over Items | Loop Over Items | Insert all the data back in the Google Sheet. All the profile data & the ICP Score |
| Sticky Note | Sticky Note | Canvas annotation | — | — | ## ICP score and Message generator |
| Sticky Note1 | Sticky Note | Canvas annotation | — | — | Retrieve the data from a Google Sheet. The only needed value is the LinkedIn profile url |
| Sticky Note2 | Sticky Note | Canvas annotation | — | — | Enrich the LinkedIn profile url with all the info from the profile and add soft skills on top |
| Sticky Note3 | Sticky Note | Canvas annotation | — | — | Describe in detail your Ideal Customer Profile (ICP) and let it match |
| Sticky Note4 | Sticky Note | Canvas annotation | — | — | Break up the data in a usable array |
| Sticky Note5 | Sticky Note | Canvas annotation | — | — | Insert all the data back in the Google Sheet. All the profile data & the ICP Score |
| Sticky Note6 | Sticky Note | Canvas annotation | — | — | ## How it works<br>This template is using a LinkedIn User Profile in combination with your detailed Ideal Customer Profile (ICP) to create a score, including reasoning and outreach messages.<br><br>It is manually triggered and uses a Google Sheet as an entry point. Inside there are rows with only LinkedIn profile urls in it.<br><br>Then the SourceGeek Node is being triggered and the complete profile info is retrieved.<br><br>That info is being sent to an AI Agent where a long description of the Ideal Customer Profile is written down.<br><br>The AI Agent will process all that data and will return with three values<br><br>- ICP rating (between 0 - 100)<br>- ICP reasoning. Where does the score come from<br>- A 1st, 2nd and 3rd outreach message which you can use later on<br><br>After that the original Google Sheet row will be updated with the data created by the AI Agent |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add node: Manual Trigger**
   - Name: `When clicking ‘Execute workflow’`.

3. **Add node: Google Sheets** (read rows)
   - Name: `Get LinkedIn profile urls from sheet`
   - **Credentials:** configure **Google Sheets OAuth2** (account that has access to the spreadsheet).
   - **Document:** select the spreadsheet *Lead Get SourceGeek - Jan 26* (or your equivalent).
   - **Sheet:** select tab *Data*.
   - **Filters:** set:
     - Column `Ready` equals `y`
     - Column `ICP Score` equals `=` (adjust to your “not processed yet” convention; many people use empty instead)
   - Ensure the sheet has a column header exactly: **LinkedIn profile URL**.

4. **Connect:** Manual Trigger → Get LinkedIn profile urls from sheet.

5. **Add node: SourceGeek**
   - Name: `Get data from a linkedin profile`
   - **Credentials:** configure SourceGeek API credentials in n8n.
   - **LinkedIn URL field:** set expression  
     - `{{$json['LinkedIn profile URL']}}`

6. **Connect:** Get LinkedIn profile urls from sheet → Get data from a linkedin profile.

7. **Add node: Google Gemini Chat Model (LangChain)**
   - Name: `Google Gemini Chat Model`
   - **Credentials:** configure Google Gemini/PaLM credentials.
   - **Model name:** `models/gemini-pro-latest`.

8. **Add node: Memory Buffer Window (LangChain)**
   - Name: `Simple Memory`
   - Set **sessionIdType** to *customKey*.
   - Set **sessionKey** to a unique identifier if you want per-lead isolation (recommended), e.g.:  
     - `{{$json['LinkedIn profile URL']}}`  
     (The provided workflow uses `{}` which may not isolate sessions.)

9. **Add node: AI Agent (LangChain)**
   - Name: `AI Agent creating the ICP score and reasoning`
   - Set **Prompt type** to *Define*.
   - Paste the agent instruction text (role/task/rules/ICP definition).
   - Ensure the prompt includes:
     - Input section referencing SourceGeek data: `{{ JSON.stringify($json.data) }}`
     - Output requirement: **return only valid JSON** with keys:
       `linkedin_profile`, `icp_score`, `icp_reasoning`, `message_1`, `message_2`, `message_3`.

10. **Connect AI resources to the agent:**
   - Google Gemini Chat Model → AI Agent (as Language Model connection).
   - Simple Memory → AI Agent (as Memory connection).

11. **Connect main flow:** Get data from a linkedin profile → AI Agent creating the ICP score and reasoning.

12. **Add node: Code**
   - Name: `JS code creating an array`
   - Paste JS that:
     - Reads agent output from `item.json.output` (or equivalent)
     - Strips ```json fences
     - `JSON.parse` it
     - Emits normalized fields (`icp_score`, messages, etc.)

13. **Connect:** AI Agent → JS code creating an array.

14. **Add node: Split in Batches**
   - Name: `Loop Over Items`
   - Set **Batch Size** to `1` (common approach for row-by-row updates).
   - Keep other options default.

15. **Connect:** JS code creating an array → Loop Over Items.

16. **Add node: Google Sheets** (update rows)
   - Name: `Update row in sheet`
   - **Credentials:** same Google Sheets OAuth2.
   - **Operation:** *Update*.
   - **Document/Sheet:** same as step 3.
   - **Matching columns:** `LinkedIn profile URL`.
   - **Map fields** (define mapping) using expressions:
     - `ICP Score` → `{{$json.icp_score}}`
     - `ICP reasoning` → `{{$json.icp_reasoning}}`
     - `First message` → `{{$json.message_1}}`
     - `Second message` → `{{$json.message_2}}`
     - `Third message` → `{{$json.message_3}}`
     - `LinkedIn profile URL` → `{{$json.linkedin_profile}}`
     - `LinkedIn profile image` → `{{$('Get data from a linkedin profile').item.json.data.profileImageUrl}}`

17. **Connect loop:**
   - Loop Over Items → Update row in sheet
   - Update row in sheet → Loop Over Items  
   (This creates the “continue to next item” behavior.)

18. **(Optional but recommended) Add error handling**
   - Add an IF node after the Code node to check `_parsed._parse_error` and route failures to a log sheet or notification.
   - Add an IF node to ensure `linkedin_profile` matches your canonical sheet format (normalize trailing slash).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| ## ICP score and Message generator | Canvas section label (sticky note) |
| ## How it works… (full block describing the template’s intent and outputs) | Canvas note summarizing the full workflow behavior (sticky note) |