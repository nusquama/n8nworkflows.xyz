Track calories and weight using Gemini, Google Sheets, Gmail and QuickChart

https://n8nworkflows.xyz/workflows/track-calories-and-weight-using-gemini--google-sheets--gmail-and-quickchart-12844


# Track calories and weight using Gemini, Google Sheets, Gmail and QuickChart

## 1. Workflow Overview

**Purpose:** Collect daily health inputs (weight, steps, meal description, email) via an n8n Form, use **Gemini** to estimate calorie intake and compute daily balance, store results in **Google Sheets**, generate a **7-day weight trend chart** via **QuickChart**, and email a daily report via **Gmail**.

**Target use cases:**
- Personal daily health journaling (weight + steps + meal notes)
- Quick calorie estimation and simple daily energy balance tracking
- Lightweight reporting with an embedded trend chart

### 1.1 Config & Input Reception
Takes user input from the Form and injects configuration values (Sheet ID, BMR) used downstream.

### 1.2 AI Analysis (Calories & Advice)
Sends steps/meals/BMR to Gemini, expecting strict JSON output. Then parses the model response safely and provides fallbacks if parsing fails.

### 1.3 Database Logging & 7-Day History + Graph
Appends (or updates) today’s row in Google Sheets, reads the last 7 entries, and generates a QuickChart URL for a line chart of weight.

### 1.4 Delivery (Email Report)
Sends an HTML email to the user with the calculated summary and embeds the chart image using the QuickChart URL.

---

## 2. Block-by-Block Analysis

### Block 1 — Config & Input Reception
**Overview:** Captures the daily entry via a public form and sets required constants (BMR and Google Sheet document ID) used by later nodes.

**Nodes involved:**
- **Health Form**
- **Configuration**

#### Node: Health Form
- **Type / role:** `Form Trigger` — workflow entry point that collects user input and triggers execution.
- **Key configuration:**
  - Title: “Daily Health Log”
  - Button label: “Analyze & Log”
  - Description: “Just type what you ate. AI does the math.”
  - Fields (all required):
    - Weight (kg) — number
    - Steps Today — number
    - Meals (Text description) — textarea
    - Your Email — email
  - Webhook ID: `health-tracker-webhook` (used internally by n8n to identify the form endpoint)
- **Inputs:** None (trigger).
- **Outputs:** One item with form fields accessible as:
  - `$('Health Form').first().json['Weight (kg)']`
  - `$('Health Form').first().json['Steps Today']`
  - `$('Health Form').first().json['Meals (Text description)']`
  - `$('Health Form').first().json['Your Email']`
- **Connections:**
  - Output → **Configuration**
- **Edge cases / failures:**
  - User submits non-sensical values (e.g., negative steps, unrealistic weight). No validation is implemented beyond “required”.
  - If form is not reachable (n8n instance networking), the workflow can’t be triggered.
- **Version notes:** Node version shown as `2.3` (behavior can vary slightly across n8n versions in UI options).

#### Node: Configuration
- **Type / role:** `Set` — defines configuration variables needed downstream.
- **Key configuration choices:**
  - Adds/overrides:
    - `sheetId` (string): **empty in JSON** → must be filled with the Google Sheets document ID.
    - `userBMR` (string): default `"1500"` (Basal Metabolic Rate)
  - `includeOtherFields: true` ensures form fields continue downstream alongside these config values.
- **Key expressions/variables used downstream:**
  - `$('Configuration').first().json.sheetId`
  - `$('Configuration').first().json.userBMR`
- **Connections:**
  - Input ← **Health Form**
  - Output → **Gemini: Calc**
- **Edge cases / failures:**
  - Empty/invalid `sheetId` will cause Google Sheets nodes to fail (document not found / permission errors).
  - `userBMR` is stored as a string; Gemini will interpret it fine, but if you later add numeric calculations in-code, you may need casting.
- **Version notes:** Node version `3.4`.

---

### Block 2 — AI Analysis (Calories & Advice)
**Overview:** Uses Gemini to estimate calories from meal text and compute calories out from BMR + steps. Parses the model output as JSON with a robust fallback.

**Nodes involved:**
- **Gemini: Calc**
- **Parse JSON**

#### Node: Gemini: Calc
- **Type / role:** `Google Gemini (LangChain)` — LLM call to produce calories in/out/balance and short advice.
- **Key configuration choices:**
  - Model: `models/gemini-1.5-flash`
  - Prompt instructs:
    1. Estimate **Calories In** from meals “Be conservative.”
    2. **Calories Out = BMR + (Steps * 0.04)**
    3. **Balance = In - Out**
    4. Output **JSON ONLY** with keys: `calories_in`, `calories_out`, `balance`, `advice`
  - Uses expressions for dynamic values:
    - Steps from form: `{{ $('Health Form').first().json['Steps Today'] }}`
    - Meals text: `{{ $('Health Form').first().json['Meals (Text description)'] }}`
    - BMR from config: `{{ $('Configuration').first().json.userBMR }}`
- **Credentials:** Google Gemini(PaLM) API account (`googlePalmApi`).
- **Connections:**
  - Input ← **Configuration**
  - Output → **Parse JSON**
- **Edge cases / failures:**
  - Model may return non-JSON or wrap JSON in code fences (handled later).
  - Rate limits, quota exhaustion, invalid API key, or region restrictions can fail the node.
  - Prompt injection risk: meal text is user-provided; the prompt asks “JSON ONLY”, but user input could still steer formatting. Parsing node mitigates but cannot guarantee semantic correctness.
- **Version notes:** Node type version `1` (LangChain-based Gemini node behavior can differ between n8n releases).

#### Node: Parse JSON
- **Type / role:** `Code` — extracts text from Gemini response, strips code fences, parses JSON, and returns a safe object.
- **Key configuration choices (logic):**
  - Reads: `const text = $input.first().json.content.parts[0].text;`
  - Sanitizes:
    - Removes ```json and ``` fences
  - Attempts `JSON.parse(cleanText)`
  - Fallback result if parsing fails:
    ```js
    { calories_in: 0, calories_out: 0, balance: 0, advice: "Could not calculate details. Please check your input." }
    ```
- **Connections:**
  - Input ← **Gemini: Calc**
  - Output → **Save Today**
- **Edge cases / failures:**
  - If Gemini node output structure changes (e.g., missing `content.parts[0].text`), this code throws before `try/catch` catches JSON parsing (because the property access happens before parsing). A safer approach would wrap that access too.
  - If Gemini returns additional text before/after JSON, parsing may still fail unless it’s cleanly removable.
- **Version notes:** Node version `2`.

---

### Block 3 — Database Logging & 7-Day History + Graph
**Overview:** Writes today’s results into Google Sheets, reads the last 7 rows, then creates a QuickChart line chart for weight over time.

**Nodes involved:**
- **Save Today**
- **Get History**
- **Generate Graph**

#### Node: Save Today
- **Type / role:** `Google Sheets` — append/update today’s data row.
- **Operation:** `appendOrUpdate`
- **Key configuration choices:**
  - Document ID: `={{ $('Configuration').first().json.sheetId }}`
  - Sheet name: `HealthLog`
  - Column mapping (define below):
    - `Date`: `{{ $today.format('yyyy-MM-dd') }}`
    - `Meals`: from form text
    - `Steps`: from form
    - `Weight`: from form
    - `Calories_In`: from parsed AI JSON (`$json.calories_in`)
    - `Calories_Out`: from parsed AI JSON (`$json.calories_out`)
    - `Balance`: from parsed AI JSON (`$json.balance`)
- **Credentials:** Google Sheets OAuth2.
- **Connections:**
  - Input ← **Parse JSON**
  - Output → **Get History**
- **Edge cases / failures:**
  - The sheet must exist and include headers matching the column keys used (`Date`, `Weight`, `Steps`, `Meals`, `Calories_In`, `Calories_Out`, `Balance`), otherwise mapping may fail or write into wrong columns.
  - `appendOrUpdate` typically relies on a matching column/key behavior; if not configured with a unique key column in the node UI, updates may not occur as expected (it may behave like append). Confirm how your n8n version defines the “update” match rule.
  - Date format must match the sheet’s expected format if you later use it for lookups/updates.
- **Version notes:** Node version `4.7`.

#### Node: Get History
- **Type / role:** `Google Sheets` — fetch recent rows for visualization.
- **Key configuration choices:**
  - Document ID: `={{ $('Configuration').first().json.sheetId }}`
  - Sheet name: `HealthLog`
  - `limit: 7`
  - Option `returnAllMatches: returnAllMatches` (ensures matches are returned; actual behavior depends on read mode defaults)
- **Credentials:** Google Sheets OAuth2.
- **Connections:**
  - Input ← **Save Today**
  - Output → **Generate Graph**
- **Edge cases / failures:**
  - Order ambiguity: this node does not explicitly sort by date. “Last 7” depends on how the Google Sheets node returns rows (often top-to-bottom). If your sheet is oldest-first, you might get the first 7, not the latest 7.
  - If there are fewer than 7 rows, chart still generates but with fewer points.
  - Missing/blank `Date` or `Weight` values will produce gaps or string values in chart data.
- **Version notes:** Node version `4.7`.

#### Node: Generate Graph
- **Type / role:** `HTTP Request` — calls QuickChart to generate a chart URL (image is referenced in email).
- **Key configuration choices:**
  - URL: `https://quickchart.io/chart`
  - Sends query parameters (`sendQuery: true`):
    - `c` parameter is a JSON-stringified Chart.js config built via expression:
      - `labels`: `$('Get History').all().map(i => i.json.Date)`
      - dataset `data`: `$('Get History').all().map(i => i.json.Weight)`
      - line styling: `borderColor: 'rgb(75, 192, 192)'`, `fill: false`
- **Input/Output:**
  - Input: items from **Get History** (but expression explicitly re-reads all items from that node)
  - Output: QuickChart response JSON typically includes a `url` field (used later as `$json.url` in email)
- **Connections:**
  - Input ← **Get History**
  - Output → **Send Email**
- **Edge cases / failures:**
  - If QuickChart changes API response format or doesn’t return `url`, the email embed breaks.
  - Long query strings can be an issue if labels/data grow; currently limited to 7 rows so it’s safe.
  - If `Weight` values are strings with commas or non-numeric, Chart.js may render unexpectedly.
- **Version notes:** Node version `4.3`.

---

### Block 4 — Delivery (Email Report)
**Overview:** Builds an HTML email summarizing the entry and includes the generated chart as an `<img>` tag sourced from QuickChart.

**Nodes involved:**
- **Send Email**

#### Node: Send Email
- **Type / role:** `Gmail` — sends the final report to the user.
- **Key configuration choices:**
  - To: `={{ $('Health Form').first().json['Your Email'] }}`
  - Subject: `💪 Health Report: {{ $today.format('yyyy-MM-dd') }}`
  - Email type: HTML
  - Message content includes:
    - Weight and Steps from the form
    - Calories In/Out/Balance from **Parse JSON**
    - Advice from **Parse JSON**
    - Embedded chart:
      - `<img src="{{ $json.url }}" width="400" />`
      - Here `$json` refers to **Generate Graph** output
- **Credentials:** Gmail OAuth2.
- **Connections:**
  - Input ← **Generate Graph**
  - Output: none (end)
- **Edge cases / failures:**
  - Gmail OAuth token expiry/revocation; insufficient scopes for sending.
  - Email may be blocked by user’s policy or end in spam due to automated content.
  - If QuickChart returns HTTP instead of HTTPS URL (unlikely here), some email clients may block mixed content.
- **Version notes:** Node version `2.1`.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note Main | Sticky Note | Documentation / canvas annotation |  |  | # AI Health Tracker; How it works (Input→Analyze→Log→Visualize→Report); Setup steps (Sheet headers, credentials, set Sheet ID & BMR) |
| Sticky Note Config | Sticky Note | Documentation / canvas annotation |  |  | ## 1. Config & Input; Set BMR and Sheet ID here. |
| Sticky Note AI | Sticky Note | Documentation / canvas annotation |  |  | ## 2. AI Analysis; Gemini calculates "Calories In" and "Calories Out". |
| Sticky Note DB | Sticky Note | Documentation / canvas annotation |  |  | ## 3. Database & Graph; Logs data, fetches history, draws trend line. |
| Sticky Note Email | Sticky Note | Documentation / canvas annotation |  |  | ## 4. Delivery; Sends the graph and advice to your email. |
| Health Form | Form Trigger | Collect daily stats & start workflow | — | Configuration | ## 1. Config & Input; Set BMR and Sheet ID here. |
| Configuration | Set | Inject sheetId and BMR into flow | Health Form | Gemini: Calc | ## 1. Config & Input; Set BMR and Sheet ID here. |
| Gemini: Calc | Google Gemini (LangChain) | Estimate calories and advice from meals/steps/BMR | Configuration | Parse JSON | ## 2. AI Analysis; Gemini calculates "Calories In" and "Calories Out". |
| Parse JSON | Code | Parse/validate Gemini JSON output with fallback | Gemini: Calc | Save Today | ## 2. AI Analysis; Gemini calculates "Calories In" and "Calories Out". |
| Save Today | Google Sheets | Append/update today’s metrics | Parse JSON | Get History | ## 3. Database & Graph; Logs data, fetches history, draws trend line. |
| Get History | Google Sheets | Fetch last 7 rows for chart | Save Today | Generate Graph | ## 3. Database & Graph; Logs data, fetches history, draws trend line. |
| Generate Graph | HTTP Request | Build QuickChart line chart URL | Get History | Send Email | ## 3. Database & Graph; Logs data, fetches history, draws trend line. |
| Send Email | Gmail | Email summary + embed chart image | Generate Graph | — | ## 4. Delivery; Sends the graph and advice to your email. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a Google Sheet**
   - Create a spreadsheet (any name).
   - Create a sheet/tab named **`HealthLog`**.
   - Add headers in the first row (exactly):  
     `Date`, `Weight`, `Steps`, `Meals`, `Calories_In`, `Calories_Out`, `Balance`

2. **Create the trigger node**
   - Add node: **Form Trigger** named **“Health Form”**
   - Configure:
     - Form Title: `Daily Health Log`
     - Description: `Just type what you ate. AI does the math.`
     - Button label: `Analyze & Log`
     - Fields (required):
       1) Number: `Weight (kg)`  
       2) Number: `Steps Today`  
       3) Textarea: `Meals (Text description)` with placeholder like `e.g., Toast for breakfast, Ramen for lunch...`  
       4) Email: `Your Email`

3. **Add configuration node**
   - Add node: **Set** named **“Configuration”**
   - Turn on “Keep Only Set” = **off** (so it keeps form fields) or ensure **Include Other Fields = true**
   - Add fields:
     - `sheetId` (String) → paste your spreadsheet document ID
     - `userBMR` (String or Number) → e.g. `1500`

4. **Connect nodes**
   - Connect: **Health Form → Configuration**

5. **Add Gemini node**
   - Add node: **Google Gemini (LangChain)** named **“Gemini: Calc”**
   - Credentials: configure **Google Gemini(PaLM) API** credential (API key in n8n credential manager).
   - Model: `models/gemini-1.5-flash`
   - Prompt/message content (single user message) that:
     - Passes Steps, Meals, BMR from earlier nodes (via expressions)
     - Requires **JSON ONLY** output with keys:
       `calories_in`, `calories_out`, `balance`, `advice`
     - Uses the formula: `Calories Out = BMR + (Steps * 0.04)`
   - Connect: **Configuration → Gemini: Calc**

6. **Add parsing code node**
   - Add node: **Code** named **“Parse JSON”**
   - Paste logic that:
     - Reads the Gemini response text
     - Strips ```json fences
     - `JSON.parse()` with a fallback object on failure
   - Connect: **Gemini: Calc → Parse JSON**

7. **Add Google Sheets write node**
   - Add node: **Google Sheets** named **“Save Today”**
   - Credentials: configure **Google Sheets OAuth2** (sign in and grant access).
   - Operation: **Append or Update**
   - Document ID: expression `{{$('Configuration').first().json.sheetId}}`
   - Sheet name: `HealthLog`
   - Map columns:
     - Date: `{{$today.format('yyyy-MM-dd')}}`
     - Weight: `{{$('Health Form').first().json['Weight (kg)']}}`
     - Steps: `{{$('Health Form').first().json['Steps Today']}}`
     - Meals: `{{$('Health Form').first().json['Meals (Text description)']}}`
     - Calories_In: `{{$json.calories_in}}`
     - Calories_Out: `{{$json.calories_out}}`
     - Balance: `{{$json.balance}}`
   - Connect: **Parse JSON → Save Today**

8. **Add Google Sheets read node**
   - Add node: **Google Sheets** named **“Get History”**
   - Credentials: same Google Sheets OAuth2.
   - Document ID: expression `{{$('Configuration').first().json.sheetId}}`
   - Sheet name: `HealthLog`
   - Set **Limit = 7**
   - Connect: **Save Today → Get History**

9. **Add QuickChart HTTP node**
   - Add node: **HTTP Request** named **“Generate Graph”**
   - Method: GET
   - URL: `https://quickchart.io/chart`
   - Enable “Send Query Parameters”
   - Add query parameter `c` with an expression that builds the Chart.js config using the 7 rows:
     - labels = Dates
     - dataset = Weight values
     - type = line
   - Connect: **Get History → Generate Graph**

10. **Add Gmail send node**
   - Add node: **Gmail** named **“Send Email”**
   - Credentials: configure **Gmail OAuth2** (Google account with send permissions).
   - To: `{{$('Health Form').first().json['Your Email']}}`
   - Subject: `💪 Health Report: {{$today.format('yyyy-MM-dd')}}`
   - Email type: HTML
   - Body: include:
     - Weight/Steps from **Health Form**
     - Calories and advice from **Parse JSON**
     - Chart image referencing QuickChart output: `<img src="{{$json.url}}" width="400" />`
   - Connect: **Generate Graph → Send Email**

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| QuickChart endpoint used to generate charts via URL query config | https://quickchart.io/chart |
| Sheet must contain headers: `Date`, `Weight`, `Steps`, `Meals`, `Calories_In`, `Calories_Out`, `Balance` | Required for correct Google Sheets column mapping |
| You must set Sheet ID and BMR in the “Configuration” node | Mentioned in canvas notes |
| Credentials required: Gemini (PaLM/Gemini API), Google Sheets OAuth2, Gmail OAuth2 | Required integrations for full workflow execution |