Find and analyze salvage car deals with Apify, Google Sheets and Gemini

https://n8nworkflows.xyz/workflows/find-and-analyze-salvage-car-deals-with-apify--google-sheets-and-gemini-19659


# Find and analyze salvage car deals with Apify, Google Sheets and Gemini

### 1. Workflow Overview

This workflow automates the discovery, filtering, AI-powered evaluation, and notification of salvage car deals sourced from UK Copart auctions via Apify. It connects auction data scraping with Google Sheets for inventory tracking, Google Gemini (via LangChain) for multimodal damage analysis and financial estimations, and Slack for high-value deal alerts.

The workflow logic is divided into two primary functional blocks:

- **1.1 Data Ingestion & Filtering:** Periodically or manually triggers an Apify HTTP request to fetch auction listings, limits the dataset to 100 vehicles, separates them into filtered categories (Pure Sale and Curated low-mileage/non-flood vehicles), and updates a Google Sheets tracking document.
- **1.2 AI Evaluation & Notification:** Triggers hourly when new rows are detected in Google Sheets, processes vehicle image URLs and metadata through a LangChain agent using Google Gemini with structured output parsing, logs the repairability analysis back to the sheet, and conditionally alerts a Slack channel if the vehicle's reparability score exceeds 80.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Ingestion & Filtering

- **Overview:** This block handles the acquisition of raw vehicle auction data from an external scraping actor, enforces volume caps, splits the records based on predefined business criteria, and persists the filtered items into Google Sheets.
- **Nodes Involved:** 
  - `Scheduled Vehicle Fetch`
  - `Manual Trigger`
  - `Fetch Auction Vehicle Listings`
  - `Limit to 100 Vehicles`
  - `Filter Pure Sale Auctions`
  - `Filter Curated Auctions`
  - `Update Pure Sale Auction Results`
  - `Update Curated Auction Results`

- **Node Details:**
  - **Scheduled Vehicle Fetch**
    - *Type and Technical Role:* `n8n-nodes-base.scheduleTrigger` (Trigger)
    - *Configuration:* Set to execute on a recurrent time-based interval.
    - *Input/Output:* Output connects to `Fetch Auction Vehicle Listings`.
    - *Edge Cases:* Timezone configuration mismatches.
  - **Manual Trigger**
    - *Type and Technical Role:* `n8n-nodes-base.manualTrigger` (Trigger)
    - *Configuration:* Allows manual user execution for testing.
    - *Input/Output:* Output connects to `Fetch Auction Vehicle Listings`.
  - **Fetch Auction Vehicle Listings**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (Core Action)
    - *Configuration:* Executes an HTTP request to the Apify actor endpoint using Apify Header Auth credentials to fetch UK Copart auction listings.
    - *Input/Output:* Inputs from triggers; output connects to `Limit to 100 Vehicles`.
    - *Credentials:* Apify HTTP Header Auth.
    - *Edge Cases:* API rate limits, invalid tokens, or upstream Apify actor downtime.
  - **Limit to 100 Vehicles**
    - *Type and Technical Role:* `n8n-nodes-base.limit` (Flow Control)
    - *Configuration:* Restricts the incoming data stream to a maximum of 100 items.
    - *Input/Output:* Input from HTTP request; outputs connect to both filter nodes.
  - **Filter Pure Sale Auctions**
    - *Type and Technical Role:* `n8n-nodes-base.filter` (Flow Control)
    - *Configuration:* Filters items matching specific criteria (e.g., `PURE_SALE` with `NEVERBID` status).
    - *Input/Output:* Input from limit node; output connects to `Update Pure Sale Auction Results`.
  - **Filter Curated Auctions**
    - *Type and Technical Role:* `n8n-nodes-base.filter` (Flow Control)
    - *Configuration:* Filters items based on curation rules (e.g., under 60,000 miles, automobiles, excluding water/flood damage).
    - *Input/Output:* Input from limit node; output connects to `Update Curated Auction Results`.
  - **Update Pure Sale Auction Results**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Data Output)
    - *Configuration:* Appends or updates rows in Google Sheets using the lot number as a unique key for pure sale listings.
    - *Credentials:* Google Sheets Service Account.
    - *Input/Output:* Input from `Filter Pure Sale Auctions`.
    - *Edge Cases:* Sheet permission issues, quota limits, or duplicate key conflicts.
  - **Update Curated Auction Results**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Data Output)
    - *Configuration:* Appends or updates rows in Google Sheets using the lot number as a unique key for curated listings.
    - *Credentials:* Google Sheets Service Account.
    - *Input/Output:* Input from `Filter Curated Auctions`.
    - *Edge Cases:* Sheet permission issues or schema mismatches.

---

#### 2.2 AI Evaluation & Notification

- **Overview:** This block monitors sheet updates, extracts vehicle inspection images, performs multimodal AI damage assessment and economic modeling via Google Gemini, updates the tracking sheet with results, and posts high-scoring deals to Slack.
- **Nodes Involved:**
  - `When Sheet Updated`
  - `Limit to 10 Entries`
  - `Process Auction Data`
  - `Repair Feasibility Agent`
  - `Gemini Chat Model`
  - `Parse Structured Output`
  - `Update Feasibility Results`
  - `Check Reparability Score`
  - `Prepare Slack Message`
  - `Post to Slack`

- **Node Details:**
  - **When Sheet Updated**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheetsTrigger` (Trigger)
    - *Configuration:* Polls the Google Sheet hourly for newly added rows.
    - *Credentials:* Google Sheets Account.
    - *Input/Output:* Output connects to `Limit to 10 Entries`.
    - *Edge Cases:* Polling latency or missed rows if sheet is updated rapidly.
  - **Limit to 10 Entries**
    - *Type and Technical Role:* `n8n-nodes-base.limit` (Flow Control)
    - *Configuration:* Restricts batch processing size to 10 records per execution.
    - *Input/Output:* Input from sheet trigger; output connects to `Process Auction Data`.
  - **Process Auction Data**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Data Transformation)
    - *Configuration:* JavaScript/Python code block to prepare and format inspection image URLs and basic lot/price fields.
    - *Input/Output:* Input from limit node; output connects to `Repair Feasibility Agent`.
    - *Edge Cases:* Malformed image URLs or missing field attributes.
  - **Repair Feasibility Agent**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.agent` (AI / LLM)
    - *Configuration:* LangChain agent orchestrating multimodal input analysis.
    - *Input/Output:* Inputs from `Process Auction Data`, `Gemini Chat Model`, and `Parse Structured Output`; output connects to `Update Feasibility Results`.
  - **Gemini Chat Model**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` (AI Sub-node)
    - *Configuration:* Configured with model `models/gemini-3.1-flash-lite`.
    - *Credentials:* Google Gemini (Google PaLM) API.
    - *Input/Output:* Connects as an AI language model to `Repair Feasibility Agent`.
  - **Parse Structured Output**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.outputParserStructured` (AI Sub-node)
    - *Configuration:* Ensures the LLM response strictly follows a schema containing fields such as `reparability_score` and `verdict`.
    - *Input/Output:* Connects as an AI output parser to `Repair Feasibility Agent`.
  - **Update Feasibility Results**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Data Output)
    - *Configuration:* Writes AI assessment fields (`reparability_score`, financial breakdown, verdict) back to the matching row in Google Sheets.
    - *Credentials:* Google Sheets Service Account.
    - *Input/Output:* Input from `Repair Feasibility Agent`; output connects to `Check Reparability Score`.
  - **Check Reparability Score**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Flow Control)
    - *Configuration:* Evaluates whether the calculated `reparability_score` is greater than 80.
    - *Input/Output:* Input from update node; true branch connects to `Prepare Slack Message`.
  - **Prepare Slack Message**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data Transformation)
    - *Configuration:* Formats the payload and text content for the high-scoring deal alert.
    - *Input/Output:* Input from conditional true branch; output connects to `Post to Slack`.
  - **Post to Slack**
    - *Type and Technical Role:* `n8n-nodes-base.slack` (Action)
    - *Configuration:* Posts the formatted report to a designated Slack channel.
    - *Credentials:* Slack OAuth2.
    - *Input/Output:* Input from prepare message node.
    - *Edge Cases:* Missing channel permissions or token revocation.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Sticky Note` | `n8n-nodes-base.stickyNote` | Documentation / Visual Grouping | None | None | |
| `Sticky Note1` | `n8n-nodes-base.stickyNote` | Documentation / Visual Grouping | None | None | |
| `Sticky Note2` | `n8n-nodes-base.stickyNote` | Documentation / Visual Grouping | None | None | |
| `Sticky Note3` | `n8n-nodes-base.stickyNote` | Documentation / Visual Grouping | None | None | |
| `Sticky Note4` | `n8n-nodes-base.stickyNote` | Documentation / Visual Grouping | None | None | |
| `Sticky Note5` | `n8n-nodes-base.stickyNote` | Documentation / Visual Grouping | None | None | |
| `Sticky Note6` | `n8n-nodes-base.stickyNote` | Documentation / Visual Grouping | None | None | |
| `Sticky Note7` | `n8n-nodes-base.stickyNote` | Documentation / Visual Grouping | None | None | |
| `Sticky Note8` | `n8n-nodes-base.stickyNote` | Documentation / Visual Grouping | None | None | |
| `Limit to 100 Vehicles` | `n8n-nodes-base.limit` | Restrict item volume | Fetch Auction Vehicle Listings | Filter Pure Sale Auctions, Filter Curated Auctions | |
| `Filter Pure Sale Auctions` | `n8n-nodes-base.filter` | Filter pure sale records | Limit to 100 Vehicles | Update Pure Sale Auction Results | |
| `Filter Curated Auctions` | `n8n-nodes-base.filter` | Filter curated vehicle attributes | Limit to 100 Vehicles | Update Curated Auction Results | |
| `Update Curated Auction Results` | `n8n-nodes-base.googleSheets` | Save curated data to Sheet | Filter Curated Auctions | None | |
| `Update Pure Sale Auction Results` | `n8n-nodes-base.googleSheets` | Save pure sale data to Sheet | Filter Pure Sale Auctions | None | |
| `When Sheet Updated` | `n8n-nodes-base.googleSheetsTrigger`| Trigger on new sheet row | None | Limit to 10 Entries | |
| `Limit to 10 Entries` | `n8n-nodes-base.limit` | Batch size limiter | When Sheet Updated | Process Auction Data | |
| `Process Auction Data` | `n8n-nodes-base.code` | Prepare image URLs & metadata | Limit to 10 Entries | Repair Feasibility Agent | |
| `Repair Feasibility Agent` | `@n8n/n8n-nodes-langchain.agent` | Multimodal AI evaluation | Process Auction Data, Gemini Chat Model, Parse Structured Output | Update Feasibility Results | |
| `Gemini Chat Model` | `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` | LLM backend for agent | None | Repair Feasibility Agent | |
| `Parse Structured Output` | `@n8n/n8n-nodes-langchain.outputParserStructured` | Enforce JSON output schema | None | Repair Feasibility Agent | |
| `Update Feasibility Results` | `n8n-nodes-base.googleSheets` | Save AI analysis to Sheet | Repair Feasibility Agent | Check Reparability Score | |
| `Check Reparability Score` | `n8n-nodes-base.if` | Condition score > 80 | Update Feasibility Results | Prepare Slack Message | |
| `Post to Slack` | `n8n-nodes-base.slack` | Send deal alert to Slack | Prepare Slack Message | None | |
| `Prepare Slack Message` | `n8n-nodes-base.set` | Format alert payload | Check Reparability Score | Post to Slack | |
| `Fetch Auction Vehicle Listings` | `n8n-nodes-base.httpRequest` | Fetch data from Apify actor | Manual Trigger, Scheduled Vehicle Fetch | Limit to 100 Vehicles | |
| `Manual Trigger` | `n8n-nodes-base.manualTrigger` | Manual workflow initiator | None | Fetch Auction Vehicle Listings | |
| `Scheduled Vehicle Fetch` | `n8n-nodes-base.scheduleTrigger` | Periodic workflow initiator | None | Fetch Auction Vehicle Listings | |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Triggers & Data Fetching:**
   - Add a `Schedule Trigger` node (`Scheduled Vehicle Fetch`) configured for hourly or daily execution.
   - Add a `Manual Trigger` node (`Manual Trigger`) for ad-hoc runs.
   - Add an `HTTP Request` node (`Fetch Auction Vehicle Listings`). Set method to `POST`/`GET` (depending on Apify actor specification), target the Apify actor endpoint, and configure **Apify HTTP Header Auth** with your API token.
   - Connect both triggers to `Fetch Auction Vehicle Listings`.

2. **Add Volume Control & Filtering:**
   - Add a `Limit` node (`Limit to 100 Vehicles`) connected after `Fetch Auction Vehicle Listings`. Set maximum items to `100`.
   - Add two `Filter` nodes: `Filter Pure Sale Auctions` and `Filter Curated Auctions`, both connected to the output of `Limit to 100 Vehicles`.
     - Configure `Filter Pure Sale Auctions` to match `PURE_SALE` and `NEVERBID` status.
     - Configure `Filter Curated Auctions` to enforce filters (e.g., mileage `< 60000`, type automobile, excluding water/flood damage).

3. **Save Initial Auction Results:**
   - Add two `Google Sheets` nodes: `Update Pure Sale Auction Results` and `Update Curated Auction Results`.
   - Connect each filter node to its respective Google Sheets node.
   - Configure both Google Sheets nodes using **Google Sheets Service Account** credentials. Set operation to `Upsert` (or append/update), providing your Spreadsheet ID, Sheet Name, and setting `lotNumber` as the matching/unique key column.

4. **Monitor Sheet Updates for AI Processing:**
   - Add a `Google Sheets Trigger` node (`When Sheet Updated`). Configure it with Google Sheets credentials, pointing to the target spreadsheet and tab, set to trigger on row additions.
   - Add a `Limit` node (`Limit to 10 Entries`) connected to the sheet trigger to process items in batches of 10.
   - Add a `Code` node (`Process Auction Data`) to parse and format inspection image URLs and basic vehicle metadata. Connect `Limit to 10 Entries` to it.

5. **Configure LangChain & Gemini AI Agent:**
   - Add an `Advanced AI Agent` node (`Repair Feasibility Agent`). Set type to `Agent`. Connect `Process Auction Data` to its main input.
   - Add a `Google Gemini Chat Model` node (`Gemini Chat Model`). Select model `models/gemini-3.1-flash-lite`, configure with **Google Gemini (Google PaLM) API** credentials, and connect its AI output to the agent's language model input.
   - Add a `Structured Output Parser` node (`Parse Structured Output`). Define a schema with fields like `reparability_score`, financial breakdown parameters, and `verdict`. Connect its output to the agent's output parser input.

6. **Log AI Results & Conditional Notifications:**
   - Add a `Google Sheets` node (`Update Feasibility Results`). Connect the output of `Repair Feasibility Agent` to it. Configure it to update rows in the spreadsheet using `lotNumber` as the key, saving the AI assessment metrics.
   - Add an `If` node (`Check Reparability Score`) connected after `Update Feasibility Results`. Set condition to check if `reparability_score` > `80`.
   - Add a `Set` node (`Prepare Slack Message`) connected to the `true` output branch of the `If` node to format the alert text.
   - Add a `Slack` node (`Post to Slack`) connected to `Prepare Slack Message`. Configure with **Slack OAuth2** credentials, select the target destination channel, and pass the prepared message payload.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Video Explanation | [https://youtu.be/TeYVU4NSgrE](https://youtu.be/TeYVU4NSgrE) |
| Automation Solution Purpose | Automated salvage car deal sourcing, AI damage assessment, and flip profit calculator |