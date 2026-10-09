Score Immoweb real estate deals with Gemini and log to Sheets and Slack

https://n8nworkflows.xyz/workflows/score-immoweb-real-estate-deals-with-gemini-and-log-to-sheets-and-slack-20453


# Score Immoweb real estate deals with Gemini and log to Sheets and Slack

### 1. Workflow Overview

This workflow automates the collection, processing, and evaluation of real estate listings from Immoweb using Apify and Google Gemini. Its primary purpose is to scan the Belgian property market, track price fluctuations against historical data stored in Google Sheets, evaluate investment quality using AI, and send instant alerts to Slack for high-potential deals.

The workflow logical blocks are structured as follows:
- **1.1 Input Reception & Configuration:** Initializes the workflow either on a daily schedule or manually, establishing baseline configurations such as target search URLs, batch size limits, and destination sheet/channel IDs.
- **1.2 Scraping & Error Handling:** Triggers the Apify Immoweb scraper actor via an HTTP request, capturing real estate data with built-in error handling that alerts Slack if the scrape fails.
- **1.3 Data Preparation & Deduplication:** Queries Google Sheets for historical records, normalizes scraped listings, calculates metrics like price per square meter ($\text{€/m}^2$) and comparative medians, and filters out duplicates.
- **1.4 AI Evaluation:** Limits the active batch to top items and sends structured property data to Google Gemini to generate investment verdicts (**BUY**, **MAYBE**, or **AVOID**), scores, reasons, and red flags.
- **1.5 Persistence & Notifications:** Merges AI verdicts back into the property dataset, updates or creates records in Google Sheets, and conditionally dispatches rich deal alerts to Slack.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Configuration
This block initiates the pipeline on a set schedule or via manual invocation and sets parameters that dictate search constraints and storage endpoints.

- **Nodes Involved:** `Every Morning at 7 AM`, `When Manually Triggered`, `Set Workflow Configuration`
- **Node Details:**
  - **Every Morning at 7 AM**
    - *Type and Technical Role:* Schedule Trigger node executing daily at hour 7.
    - *Configuration:* Cron-style interval set to trigger at 7:00 AM.
    - *Inputs / Outputs:* No inputs; outputs execution flow to configuration.
    - *Failure Handling:* None required.
  - **When Manually Triggered**
    - *Type and Technical Role:* Manual Trigger node for ad-hoc execution.
    - *Configuration:* Default empty parameters.
    - *Inputs / Outputs:* No inputs; outputs execution flow to configuration.
    - *Failure Handling:* None required.
  - **Set Workflow Configuration**
    - *Type and Technical Role:* Set node used to define global workflow variables.
    - *Configuration:* Assigns string and numeric constants (`searchUrl`, `maxItems`, `minDropPct`, `slackChannelName`, `sheetId`, `sheetName`, `emailTo`).
    - *Key Expressions:* Hardcoded parameters exposed globally to downstream nodes via `$('Set Workflow Configuration').first().json`.
    - *Inputs / Outputs:* Inputs from triggers; outputs to Apify HTTP request.
    - *Failure Handling:* Expression execution failures result in immediate stop.

#### 2.2 Scraping & Error Handling
Executes the remote extraction of real estate listings and handles runtime failures by notifying administrators on Slack.

- **Nodes Involved:** `Run Immoweb Scraper on Apify`, `Send Scrape Error to Slack`
- **Node Details:**
  - **Run Immoweb Scraper on Apify**
    - *Type and Technical Role:* HTTP Request node communicating with the Apify v2 API.
    - *Configuration:* POST request to `https://api.apify.com/v2/acts/7hrExNTiDRs0nPUXh/run-sync-get-dataset-items` with a 330-second timeout. Sends JSON payload mapping `searchUrl` and `maxItems`. Uses HTTP Header Authentication.
    - *Key Expressions:* `={{ JSON.stringify({ startUrl: $json.searchUrl, startUrls: [{ url: $json.searchUrl }], maxItems: $json.maxItems }) }}`
    - *Inputs / Outputs:* Input from configuration node; dual outputs (success path to Google Sheets reader, error path to Slack alert via `onError: continueErrorOutput`).
    - *Edge Cases:* API timeouts or invalid credentials. Handled via retry settings (max 2 retries) and error branch routing.
  - **Send Scrape Error to Slack**
    - *Type and Technical Role:* Slack notification node.
    - *Configuration:* Sends text message to a designated channel ID retrieved from the configuration node.
    - *Key Expressions:* `={{ ⚠️ Immoweb Deal Detector: Apify scrape failed. ${ JSON.stringify($json).slice(0, 400) } }}`
    - *Inputs / Outputs:* Input from error output of the Apify HTTP node; outputs message delivery status.
    - *Credentials:* Uses Slack API authentication.

#### 2.3 Data Preparation & Deduplication
Reads existing records from Google Sheets to maintain price history and filters incoming listings for normalization and deduplication.

- **Nodes Involved:** `Read Saved Listings in Sheets`, `Deduplicate and Prepare Listings`, `Take Top 10 Properties`
- **Node Details:**
  - **Read Saved Listings in Sheets**
    - *Type and Technical Role:* Google Sheets node using service account authentication.
    - *Configuration:* Reads rows using filter values matching listing IDs. Configured to execute once with up to 3 retries.
    - *Key Expressions:* Lookup value maps to `={{ $json.id }}`.
    - *Inputs / Outputs:* Input from Apify execution success path; outputs historical records array.
    - *Credentials:* Google Service Account.
  - **Deduplicate and Prepare Listings**
    - *Type and Technical Role:* Code node (JavaScript) performing heavy data transformation.
    - *Configuration:* Normalizes nested Apify schema elements, calculates square meter prices ($\text{€/m}^2$), derives median prices based on comparative tiers (postcode, region, search pool), detects anomalies, computes heuristic pre-scores, and identifies new or price-changed properties.
    - *Inputs / Outputs:* Inputs historical Google Sheet data and scraped items; outputs normalized property objects.
    - *Edge Cases:* Missing surface dimensions or malformed IDs; handled by default null checks within helper functions.
  - **Take Top 10 Properties**
    - *Type and Technical Role:* Limit node restricting the processing queue batch size.
    - *Configuration:* Keeps the first 10 items.
    - *Inputs / Outputs:* Input from code node; outputs up to 10 items to the AI chain.

#### 2.4 AI Evaluation
Applies large language model reasoning to generate investment verdicts and structured evaluations.

- **Nodes Involved:** `Generate Property Deal Verdict`, `Google Gemini Chat Model`, `Parse Structured Verdict`
- **Node Details:**
  - **Generate Property Deal Verdict**
    - *Type and Technical Role:* LangChain Advanced AI Chain node (`chainLlm`).
    - *Configuration:* Evaluates payload using a predefined system prompt acting as a skeptical real-estate analyst, applying strict rules for **BUY**, **MAYBE**, or **AVOID** classifications.
    - *Key Expressions:* `={{ Evaluate this listing:\n${ JSON.stringify($json, null, 1) } }}`
    - *Inputs / Outputs:* Input from Limit node; outputs structured AI analysis paired with source items.
    - *Failure Handling:* Configured with `onError: continueRegularOutput` and up to 3 retries.
  - **Google Gemini Chat Model**
    - *Type and Technical Role:* LangChain Google Gemini Chat Model sub-node (`lmChatGoogleGemini`).
    - *Configuration:* Uses model `models/gemini-3.1-flash-lite` with a temperature setting of `0.2`.
    - *Credentials:* Google PaLM / Gemini API.
  - **Parse Structured Verdict**
    - *Type and Technical Role:* LangChain Structured Output Parser sub-node (`outputParserStructured`).
    - *Configuration:* Enforces a JSON schema containing `verdict` (enum: BUY, MAYBE, AVOID), `score` (0-100), `one_liner`, `reasons`, `red_flags`, and `negotiation_tip`.

#### 2.5 Persistence & Notifications
Merges the AI verdict with property metadata, updates the Google Sheets database, and dispatches Slack notifications for qualified deals.

- **Nodes Involved:** `Attach Verdict to Property`, `Upsert Properties in Sheets`, `If Deal Alert Triggered`, `Send Deal Alert to Slack`
- **Node Details:**
  - **Attach Verdict to Property**
    - *Type and Technical Role:* Code node (JavaScript) combining original property context with AI outputs.
    - *Configuration:* Parses structured verdicts, validates enums, formats Markdown texts for Slack and HTML for email payloads, and constructs Google Sheets row records.
    - *Inputs / Outputs:* Input from AI chain node; outputs enriched items containing formatted texts and row objects.
  - **Upsert Properties in Sheets**
    - *Type and Technical Role:* Google Sheets node configured for append-or-update operations.
    - *Configuration:* Matches rows on `id` and maps sheet columns to structured payload attributes (`url`, `title`, `bucket`, `postal`, `city`, `province`, `price`, `initial_price`, `surface`, `price_m2`, `bedrooms`, `epc`, `construction_year`, `first_seen`, `last_seen`, `last_price_change`, `price_drops`, `verdict`, `score`, `one_liner`, `alerted`).
    - *Inputs / Outputs:* Input from property attachment code node; outputs upsert confirmation.
    - *Credentials:* Google Service Account.
  - **If Deal Alert Triggered**
    - *Type and Technical Role:* If condition node.
    - *Configuration:* Evaluates whether the property meets alert criteria (`alert === true`).
    - *Key Expressions:* `={{ $('Attach Verdict to Property').item.json.alert }}`.
    - *Inputs / Outputs:* Input from Google Sheets upsert node; routes matching items to Slack alerts.
  - **Send Deal Alert to Slack**
    - *Type and Technical Role:* Slack notification node.
    - *Configuration:* Sends formatted Markdown notifications (`mrkdwn: true`) to the target channel name defined in global configurations.
    - *Key Expressions:* `={{ $('Attach Verdict to Property').item.json.slackText }}`.
    - *Inputs / Outputs:* Input from If condition true-branch; outputs notification delivery status.
    - *Credentials:* Slack API.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Every Morning at 7 AM` | `scheduleTrigger` | Triggers workflow daily at 7 AM. | None | `Set Workflow Configuration` | ## Trigger and initialize parameters<br>Initiates the workflow on schedule or manually and sets search parameters. |
| `When Manually Triggered` | `manualTrigger` | Manually triggers the workflow execution. | None | `Set Workflow Configuration` | ## Trigger and initialize parameters<br>Initiates the workflow on schedule or manually and sets search parameters. |
| `Set Workflow Configuration` | `set` | Sets global configuration variables and search parameters. | `Every Morning at 7 AM`<br>`When Manually Triggered` | `Run Immoweb Scraper on Apify` | ## Trigger and initialize parameters<br>Initiates the workflow on schedule or manually and sets search parameters. |
| `Run Immoweb Scraper on Apify` | `httpRequest` | Executes the Apify Immoweb scraper task. | `Set Workflow Configuration` | `Read Saved Listings in Sheets`<br>`Send Scrape Error to Slack` | ## Scrape real estate listings<br>Executes the Apify Immoweb scraper task and alerts Slack if the request fails. |
| `Send Scrape Error to Slack` | `slack` | Sends notification to Slack if Apify scrape fails. | `Run Immoweb Scraper on Apify` | None | ## Scrape real estate listings<br>Executes the Apify Immoweb scraper task and alerts Slack if the request fails. |
| `Read Saved Listings in Sheets` | `googleSheets` | Fetches historical listings from Google Sheets. | `Run Immoweb Scraper on Apify` | `Deduplicate and Prepare Listings` | ## Filter and deduplicate properties<br>Fetches known listings from Google Sheets, filters out duplicates, and limits batch size. |
| `Deduplicate and Prepare Listings` | `code` | Normalizes listings, computes $\text{€/m}^2$, and removes duplicates. | `Read Saved Listings in Sheets` | `Take Top 10 Properties` | ## Filter and deduplicate properties<br>Fetches known listings from Google Sheets, filters out duplicates, and limits batch size. |
| `Take Top 10 Properties` | `limit` | Restricts processing batch size to 10 items. | `Deduplicate and Prepare Listings` | `Generate Property Deal Verdict` | ## Filter and deduplicate properties<br>Fetches known listings from Google Sheets, filters out duplicates, and limits batch size. |
| `Generate Property Deal Verdict` | `chainLlm` | Evaluates properties using Google Gemini via structured prompt. | `Take Top 10 Properties` | `Attach Verdict to Property` | ## Evaluate deals with Gemini<br>Evaluates property details using Google Gemini with structured outputs and merges the verdict. |
| `Google Gemini Chat Model` | `lmChatGoogleGemini` | Provides Gemini LLM core configuration for the evaluation chain. | None | `Generate Property Deal Verdict` | ## Evaluate deals with Gemini<br>Evaluates property details using Google Gemini with structured outputs and merges the verdict. |
| `Parse Structured Verdict` | `outputParserStructured` | Ensures AI output matches the expected JSON schema. | None | `Generate Property Deal Verdict` | ## Evaluate deals with Gemini<br>Evaluates property details using Google Gemini with structured outputs and merges the verdict. |
| `Attach Verdict to Property` | `code` | Merges Gemini verdict and formats message strings. | `Generate Property Deal Verdict` | `Upsert Properties in Sheets` | ## Log results and send alerts<br>Records evaluated property data into Google Sheets and notifies Slack when criteria match. |
| `Upsert Properties in Sheets` | `googleSheets` | Records or updates evaluated listing data in Google Sheets. | `Attach Verdict to Property` | `If Deal Alert Triggered` | ## Log results and send alerts<br>Records evaluated property data into Google Sheets and notifies Slack when criteria match. |
| `If Deal Alert Triggered` | `if` | Checks if property qualifies for deal notifications. | `Upsert Properties in Sheets` | `Send Deal Alert to Slack` | ## Log results and send alerts<br>Records evaluated property data into Google Sheets and notifies Slack when criteria match. |
| `Send Deal Alert to Slack` | `slack` | Sends deal alerts to the designated Slack channel. | `If Deal Alert Triggered` | None | ## Log results and send alerts<br>Records evaluated property data into Google Sheets and notifies Slack when criteria match. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**
   - Add a **Schedule Trigger** (`Every Morning at 7 AM`) set to run daily at 7:00 AM.
   - Add a **Manual Trigger** (`When Manually Triggered`) for ad-hoc executions.
2. **Configure Global Variables:**
   - Add a **Set** node (`Set Workflow Configuration`).
   - Configure assignments: `searchUrl` (string), `maxItems` (number, e.g., `10`), `minDropPct` (number, e.g., `1`), `slackChannelName` (string, e.g., `general`), `sheetId` (string), `sheetName` (string, e.g., `Listings`), `emailTo` (string).
   - Connect both triggers to this node.
3. **Setup Apify Scraper Integration:**
   - Add an **HTTP Request** node (`Run Immoweb Scraper on Apify`). Set method to `POST`, URL to `https://api.apify.com/v2/acts/7hrExNTiDRs0nPUXh/run-sync-get-dataset-items`, and timeout to `330000` ms.
   - Configure query parameters: `timeout` (`280`), `clean` (`true`), `format` (`json`).
   - Configure JSON body with expressions mapping to the configuration variables (`searchUrl`, `maxItems`).
   - Authenticate via **HTTP Header Auth** credential.
   - Connect `Set Workflow Configuration` to this node.
4. **Add Scrape Error Handling:**
   - Add a **Slack** node (`Send Scrape Error to Slack`) using Slack API credentials.
   - Connect the error output (`onError: continueErrorOutput`) of the Apify HTTP node to this Slack node.
5. **Setup Historical Data Retrieval:**
   - Add a **Google Sheets** node (`Read Saved Listings in Sheets`) using Service Account authentication.
   - Set operation to lookup rows, referencing the sheet ID and name from global configuration, matching on listing IDs.
   - Connect the success output of the Apify HTTP node to this node.
6. **Process and Deduplicate Data:**
   - Add a **Code** node (`Deduplicate and Prepare Listings`) running JavaScript to normalize schemas, compute comparative metrics, filter duplicates, and build the AI payload.
   - Connect `Read Saved Listings in Sheets` to this node.
   - Add a **Limit** node (`Take Top 10 Properties`) configured to keep the first 10 items. Connect the Code node output to it.
7. **Configure AI Evaluation Chain:**
   - Add an **Advanced AI Chain** node (`Generate Property Deal Verdict`).
   - Connect a **Google Gemini Chat Model** sub-node (`Google Gemini Chat Model`) using model `models/gemini-3.1-flash-lite` with Google PaLM API credentials and temperature `0.2`.
   - Connect a **Structured Output Parser** sub-node (`Parse Structured Verdict`) containing the JSON schema for verdicts (`BUY`, `MAYBE`, `AVOID`), scores, reasons, red flags, and tips.
   - Connect the Limit node output to the Chain node.
8. **Format and Persist Results:**
   - Add a **Code** node (`Attach Verdict to Property`) to merge AI outputs, format Slack/Email strings, and structure sheet rows. Connect the AI Chain to this node.
   - Add a **Google Sheets** node (`Upsert Properties in Sheets`) using Service Account credentials. Set operation to `appendOrUpdate`, matching columns on `id` and mapping data fields. Connect the property attachment node here.
9. **Configure Deal Alerting:**
   - Add an **If** node (`If Deal Alert Triggered`) checking if `alert === true`. Connect the Sheets upsert node to it.
   - Add a **Slack** node (`Send Deal Alert to Slack`) using Slack API credentials, configured with Markdown enabled and channel ID/name derived from global parameters. Connect the true branch of the If node to this Slack node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Youtube Video Tutorial | [Watch on YouTube](https://youtu.be/FGzYp8k-ONY) |