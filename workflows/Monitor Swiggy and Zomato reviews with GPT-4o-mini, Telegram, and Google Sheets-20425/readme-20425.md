Monitor Swiggy and Zomato reviews with GPT-4o-mini, Telegram, and Google Sheets

https://n8nworkflows.xyz/workflows/monitor-swiggy-and-zomato-reviews-with-gpt-4o-mini--telegram--and-google-sheets-20425


# Monitor Swiggy and Zomato reviews with GPT-4o-mini, Telegram, and Google Sheets

### 1. Workflow Overview

This workflow automates the collection, sentiment analysis, logging, alerting, and reporting of restaurant reviews originating from platforms like Swiggy and Zomato. It is specifically designed for restaurant owners and operations teams who need real-time awareness of customer feedback, automated public response drafting, and week-over-week analytical insights.

The workflow logic is divided into two primary execution schedules:
- **Real-Time Monitoring Flow:** Polls or receives new reviews every 30 minutes, normalizes payloads, removes duplicates via Google Sheets check, maps mentions to the restaurant menu, processes feedback through GPT-4o-mini, logs data into Google Sheets, and routes alerts to Telegram and Gmail.
- **Weekly Reporting Flow:** Triggers every Monday at 9 AM, aggregates two weeks of historical review and dish data, generates AI-driven business insights, and delivers a formatted HTML report via email alongside a summary on Telegram.

---

### 2. Block-by-Block Analysis

#### 1.1 Review Intake & Normalization
- **Overview:** Captures incoming reviews either via a scheduled 30-minute HTTP poll or an instant webhook endpoint, then normalizes irregular JSON payloads into a unified schema.
- **Nodes Involved:** `Poll Reviews Every 30 Min`, `Receive Reviews via Webhook`, `Fetch Swiggy & Zomato Reviews`, `Normalize Review Payloads`.
- **Node Details:**
  - `Poll Reviews Every 30 Min`
    - *Type/Role:* `n8n-nodes-base.scheduleTrigger` (Trigger)
    - *Configuration:* Executes every 30 minutes.
    - *Input/Output:* Output connects to `Fetch Swiggy & Zomato Reviews`.
  - `Receive Reviews via Webhook`
    - *Type/Role:* `n8n-nodes-base.webhook` (Trigger)
    - *Configuration:* Listens on path `restaurant-reviews` for incoming `POST` requests.
    - *Input/Output:* Output connects to `Normalize Review Payloads`.
  - `Fetch Swiggy & Zomato Reviews`
    - *Type/Role:* `n8n-nodes-base.httpRequest` (Action)
    - *Configuration:* Uses HTTP Header Authentication with a 30,000ms timeout and automatic retry logic (3 tries, 5000ms interval). Continues regular output on error.
    - *Input/Output:* Input from schedule trigger; output connects to `Normalize Review Payloads`.
    - *Edge Cases:* Network timeouts or incorrect review-feed endpoints will trigger the retry mechanism or output empty payloads safely.
  - `Normalize Review Payloads`
    - *Type/Role:* `n8n-nodes-base.code` (Data Transformation)
    - *Configuration:* JavaScript execution to extract review texts, ratings, platforms, dates, and order items from disparate JSON structures, yielding a normalized array of objects.
    - *Input/Output:* Inputs from `Receive Reviews via Webhook` and `Fetch Swiggy & Zomato Reviews`; output connects to `Set Monitor Config`.

#### 1.2 Configuration, Deduplication & Menu Matching
- **Overview:** Applies global configurations, reads historical log data from Google Sheets to prevent duplicate processing, and fetches the restaurant menu for AI entity mapping.
- **Nodes Involved:** `Set Monitor Config`, `Get Logged Reviews`, `Get Menu Items`, `Filter New Reviews`.
- **Node Details:**
  - `Set Monitor Config`
    - *Type/Role:* `n8n-nodes-base.set` (Parameter Assignment)
    - *Configuration:* Sets operational variables including `restaurant_name`, `brand_voice`, Google Sheet identifiers, Telegram Chat ID, escalation email address, and critical rating thresholds.
    - *Input/Output:* Input from `Normalize Review Payloads`; output connects to `Get Logged Reviews`.
  - `Get Logged Reviews`
    - *Type/Role:* `n8n-nodes-base.googleSheets` (Data Retrieval)
    - *Configuration:* Reads rows from the configured "Reviews Log" sheet. Executed once.
    - *Input/Output:* Input from `Set Monitor Config`; output connects to `Get Menu Items`.
    - *Edge Cases:* Google Sheets API limits or incorrect Sheet IDs/tab names will cause authentication or structural failures.
  - `Get Menu Items`
    - *Type/Role:* `n8n-nodes-base.googleSheets` (Data Retrieval)
    - *Configuration:* Reads canonical menu items from the designated menu sheet. Executed once.
    - *Input/Output:* Input from `Get Logged Reviews`; output connects to `Filter New Reviews`.
  - `Filter New Reviews`
    - *Type/Role:* `n8n-nodes-base.code` (Filtering & Preparation)
    - *Configuration:* JavaScript execution to compare incoming review IDs against logged IDs, filtering out duplicates and formatting the menu list string for AI processing.
    - *Input/Output:* Input from `Get Menu Items`; output connects to `AI Analyze Review & Draft Reply`.

#### 1.3 AI Review Analysis
- **Overview:** Sends unique reviews along with menu context to GPT-4o-mini to classify sentiment, extract dish-level mentions, identify operational issues, and draft a response adhering to brand guidelines.
- **Nodes Involved:** `AI Analyze Review & Draft Reply`, `Merge Review with AI Analysis`.
- **Node Details:**
  - `AI Analyze Review & Draft Reply`
    - *Type/Role:* `@n8n/n8n-nodes-langchain.openAi` (AI Processing)
    - *Configuration:* Uses model `gpt-4o-mini` with a temperature of `0.4` and structured JSON output prompts enforcing strict schema rules for sentiment analysis, urgency tagging, dish mapping, and reply drafting.
    - *Input/Output:* Input from `Filter New Reviews`; output connects to `Merge Review with AI Analysis`.
    - *Edge Cases:* Rate limiting or API downtime on OpenAI will halt review enrichment.
  - `Merge Review with AI Analysis`
    - *Type/Role:* `n8n-nodes-base.code` (Data Transformation)
    - *Configuration:* Merges raw review objects with AI analytical responses, computes HTML email payloads, Telegram messages, and checks critical thresholds.
    - *Input/Output:* Input from `AI Analyze Review & Draft Reply`; outputs branch to sheet logging and alerting paths.

#### 1.4 Sheet Logging
- **Overview:** Persists incoming reviews and granular dish-level mentions into separate tabs within the designated Google Sheets file.
- **Nodes Involved:** `Build Reviews Log Row`, `Append to Reviews Log`, `Split Dish Mentions`, `Append to Dish Mentions`.
- **Node Details:**
  - `Build Reviews Log Row`
    - *Type/Role:* `n8n-nodes-base.code` (Data Preparation)
    - *Configuration:* Formats review metadata, sentiments, replies, and status fields into a structured row format.
    - *Input/Output:* Input from `Merge Review with AI Analysis`; output connects to `Append to Reviews Log`.
  - `Append to Reviews Log`
    - *Type/Role:* `n8n-nodes-base.googleSheets` (Data Persistence)
    - *Configuration:* Appends rows to the "Reviews Log" sheet using auto-mapped input data.
    - *Input/Output:* Input from `Build Reviews Log Row`.
  - `Split Dish Mentions`
    - *Type/Role:* `n8n-nodes-base.code` (Data Transformation)
    - *Configuration:* Iterates over AI-extracted dish mentions and produces individual row objects per dish.
    - *Input/Output:* Input from `Merge Review with AI Analysis`; output connects to `Append to Dish Mentions`.
  - `Append to Dish Mentions`
    - *Type/Role:* `n8n-nodes-base.googleSheets` (Data Persistence)
    - *Configuration:* Appends dish mention rows to the "Dish Mentions" sheet.
    - *Input/Output:* Input from `Split Dish Mentions`.

#### 1.5 Alerts & Reply Drafts
- **Overview:** Evaluates review criticality and routes escalations (Telegram alerts and manager emails for critical feedback) or queue-ready reply drafts (for positive or low-urgency reviews).
- **Nodes Involved:** `Is Critical Review?`, `Send Critical Alert to Telegram`, `Email Manager Escalation`, `Send Reply Draft to Telegram`.
- **Node Details:**
  - `Is Critical Review?`
    - *Type/Role:* `n8n-nodes-base.if` (Conditional Branching)
    - *Configuration:* Evaluates whether the boolean expression `{{ $json.is_critical }}` equates to true.
    - *Input/Output:* Input from `Merge Review with AI Analysis`; true branch connects to critical alerts, false branch connects to standard draft notifications.
  - `Send Critical Alert to Telegram`
    - *Type/Role:* `n8n-nodes-base.telegram` (Messaging)
    - *Configuration:* Sends formatted HTML message to the specified Telegram chat ID via bot credentials without attribution.
    - *Input/Output:* Input from the true branch of `Is Critical Review?`.
  - `Email Manager Escalation`
    - *Type/Role:* `n8n-nodes-base.gmail` (Email Dispatch)
    - *Configuration:* Sends HTML escalation emails to the designated manager address using OAuth2 credentials.
    - *Input/Output:* Input from the true branch of `Is Critical Review?`.
  - `Send Reply Draft to Telegram`
    - *Type/Role:* `n8n-nodes-base.telegram` (Messaging)
    - *Configuration:* Sends ready-to-copy reply drafts to the Telegram chat ID.
    - *Input/Output:* Input from the false branch of `Is Critical Review?`.

#### 1.6 Weekly Data Pull & Insights
- **Overview:** Executes every Monday to retrieve the previous two weeks of review data, aggregates sentiment and performance metrics, and prompts OpenAI to synthesize operational insights.
- **Nodes Involved:** `Every Monday 9 AM`, `Set Report Config`, `Get Reviews for Report`, `Get Dish Mentions for Report`, `Aggregate Weekly Dish Sentiment`, `AI Write Weekly Insights`.
- **Node Details:**
  - `Every Monday 9 AM`
    - *Type/Role:* `n8n-nodes-base.scheduleTrigger` (Trigger)
    - *Configuration:* Configured to trigger weekly on Mondays at 09:00 AM.
    - *Input/Output:* Output connects to `Set Report Config`.
  - `Set Report Config`
    - *Type/Role:* `n8n-nodes-base.set` (Parameter Assignment)
    - *Configuration:* Sets report parameters including sheet IDs, recipient email addresses, threshold values, and ranking constraints.
    - *Input/Output:* Input from `Every Monday 9 AM`; output connects to `Get Reviews for Report`.
  - `Get Reviews for Report`
    - *Type/Role:* `n8n-nodes-base.googleSheets` (Data Retrieval)
    - *Configuration:* Retrieves the contents of the reviews sheet. Executed once.
    - *Input/Output:* Input from `Set Report Config`; output connects to `Get Dish Mentions for Report`.
  - `Get Dish Mentions for Report`
    - *Type/Role:* `n8n-nodes-base.googleSheets` (Data Retrieval)
    - *Configuration:* Retrieves the contents of the dish mentions sheet. Executed once.
    - *Input/Output:* Input from `Get Reviews for Report`; output connects to `Aggregate Weekly Dish Sentiment`.
  - `Aggregate Weekly Dish Sentiment`
    - *Type/Role:* `n8n-nodes-base.code` (Data Analysis)
    - *Configuration:* Processes raw records into comparative week-over-week analytical metrics, platform splits, issue frequencies, and dish sentiment rankings.
    - *Input/Output:* Input from `Get Dish Mentions for Report`; output connects to `AI Write Weekly Insights`.
  - `AI Write Weekly Insights`
    - *Type/Role:* `@n8n/n8n-nodes-langchain.openAi` (AI Processing)
    - *Configuration:* Uses model `gpt-4o-mini` with a temperature of `0.3` to generate qualitative executive summaries, wins, concerns, actionable steps, and menu ideas based on aggregated metrics.
    - *Input/Output:* Input from `Aggregate Weekly Dish Sentiment`; output connects to `Build Weekly Report`.

#### 1.7 Report Delivery & Archival
- **Overview:** Transforms aggregated metrics and AI insights into final email markups, Telegram summaries, and archives report records to Google Sheets.
- **Nodes Involved:** `Build Weekly Report`, `Email Weekly Report to Owner`, `Send Weekly Summary to Telegram`, `Build Weekly Report Row`, `Append to Weekly Reports`.
- **Node Details:**
  - `Build Weekly Report`
    - *Type/Role:* `n8n-nodes-base.code` (Data Transformation)
    - *Configuration:* Generates clean HTML email structures, Telegram summary strings, and row logging payloads.
    - *Input/Output:* Input from `AI Write Weekly Insights`; outputs connect to email, Telegram, and archival nodes.
  - `Email Weekly Report to Owner`
    - *Type/Role:* `n8n-nodes-base.gmail` (Email Dispatch)
    - *Configuration:* Dispatches the complete weekly HTML report email via Gmail OAuth2 credentials.
    - *Input/Output:* Input from `Build Weekly Report`.
  - `Send Weekly Summary to Telegram`
    - *Type/Role:` `n8n-nodes-base.telegram` (Messaging)
    - *Configuration:* Posts the weekly analytical summary to Telegram.
    - *Input/Output:* Input from `Build Weekly Report`.
  - `Build Weekly Report Row`
    - *Type/Role:* `n8n-nodes-base.code` (Data Preparation)
    - *Configuration:* Extracts report row objects for sheet insertion.
    - *Input/Output:* Input from `Build Weekly Report`; output connects to `Append to Weekly Reports`.
  - `Append to Weekly Reports`
    - *Type/Role:* `n8n-nodes-base.googleSheets` (Data Persistence)
    - *Configuration:* Appends weekly summary statistics into the "Weekly Reports" sheet.
    - *Input/Output:* Input from `Build Weekly Report Row`.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note – Overview | n8n-nodes-base.stickyNote | Documentation | None | None | 🍛 RestaurantOS – Swiggy & Zomato Rating Monitor... |
| Sticky Note – Review Intake | n8n-nodes-base.stickyNote | Documentation | None | None | 📥 Review Intake... |
| Sticky Note – Dedupe & Menu Match | n8n-nodes-base.stickyNote | Documentation | None | None | 🧹 Config, Dedupe & Menu... |
| Sticky Note – AI Review Analysis | n8n-nodes-base.stickyNote | Documentation | None | None | 🧠 AI Review Analysis... |
| Sticky Note – Sheet Logging | n8n-nodes-base.stickyNote | Documentation | None | None | 🗂️ Sheet Logging... |
| Sticky Note – Alerts & Reply Drafts | n8n-nodes-base.stickyNote | Documentation | None | None | 🚨 Alerts & Reply Drafts... |
| Sticky Note – Weekly Data Pull | n8n-nodes-base.stickyNote | Documentation | None | None | 📅 Weekly Data Pull... |
| Sticky Note – Weekly Insights | n8n-nodes-base.stickyNote | Documentation | None | None | 📊 Weekly Insights... |
| Sticky Note – Report Delivery | n8n-nodes-base.stickyNote | Documentation | None | None | 📬 Report Delivery... |
| Sticky Note – Credentials & Security | n8n-nodes-base.stickyNote | Documentation | None | None | 🔐 Credentials & Security... |
| Poll Reviews Every 30 Min | n8n-nodes-base.scheduleTrigger | Trigger | None | Fetch Swiggy & Zomato Reviews | 📥 Review Intake |
| Receive Reviews via Webhook | n8n-nodes-base.webhook | Trigger | None | Normalize Review Payloads | 📥 Review Intake |
| Fetch Swiggy & Zomato Reviews | n8n-nodes-base.httpRequest | Action | Poll Reviews Every 30 Min | Normalize Review Payloads | 📥 Review Intake |
| Normalize Review Payloads | n8n-nodes-base.code | Transformation | Receive Reviews via Webhook, Fetch Swiggy & Zomato Reviews | Set Monitor Config | 📥 Review Intake |
| Set Monitor Config | n8n-nodes-base.set | Configuration | Normalize Review Payloads | Get Logged Reviews | 🧹 Config, Dedupe & Menu |
| Get Logged Reviews | n8n-nodes-base.googleSheets | Data Retrieval | Set Monitor Config | Get Menu Items | 🧹 Config, Dedupe & Menu |
| Get Menu Items | n8n-nodes-base.googleSheets | Data Retrieval | Get Logged Reviews | Filter New Reviews | 🧹 Config, Dedupe & Menu |
| Filter New Reviews | n8n-nodes-base.code | Filtering | Get Menu Items | AI Analyze Review & Draft Reply | 🧹 Config, Dedupe & Menu |
| AI Analyze Review & Draft Reply | @n8n/n8n-nodes-langchain.openAi | AI Processing | Filter New Reviews | Merge Review with AI Analysis | 🧠 AI Review Analysis |
| Merge Review with AI Analysis | n8n-nodes-base.code | Transformation | AI Analyze Review & Draft Reply | Build Reviews Log Row, Split Dish Mentions, Is Critical Review? | 🧠 AI Review Analysis |
| Build Reviews Log Row | n8n-nodes-base.code | Transformation | Merge Review with AI Analysis | Append to Reviews Log | 🗂️ Sheet Logging |
| Split Dish Mentions | n8n-nodes-base.code | Transformation | Merge Review with AI Analysis | Append to Dish Mentions | 🗂️ Sheet Logging |
| Is Critical Review? | n8n-nodes-base.if | Conditional | Merge Review with AI Analysis | Send Critical Alert to Telegram, Email Manager Escalation, Send Reply Draft to Telegram | 🚨 Alerts & Reply Drafts |
| Append to Reviews Log | n8n-nodes-base.googleSheets | Persistence | Build Reviews Log Row | None | 🗂️ Sheet Logging |
| Append to Dish Mentions | n8n-nodes-base.googleSheets | Persistence | Split Dish Mentions | None | 🗂️ Sheet Logging |
| Send Critical Alert to Telegram | n8n-nodes-base.telegram | Messaging | Is Critical Review? | None | 🚨 Alerts & Reply Drafts |
| Email Manager Escalation | n8n-nodes-base.gmail | Email Dispatch | Is Critical Review? | None | 🚨 Alerts & Reply Drafts |
| Send Reply Draft to Telegram | n8n-nodes-base.telegram | Messaging | Is Critical Review? | None | 🚨 Alerts & Reply Drafts |
| Every Monday 9 AM | n8n-nodes-base.scheduleTrigger | Trigger | None | Set Report Config | 📅 Weekly Data Pull |
| Set Report Config | n8n-nodes-base.set | Configuration | Every Monday 9 AM | Get Reviews for Report | 📅 Weekly Data Pull |
| Get Reviews for Report | n8n-nodes-base.googleSheets | Data Retrieval | Set Report Config | Get Dish Mentions for Report | 📅 Weekly Data Pull |
| Get Dish Mentions for Report | n8n-nodes-base.googleSheets | Data Retrieval | Get Reviews for Report | Aggregate Weekly Dish Sentiment | 📅 Weekly Data Pull |
| Aggregate Weekly Dish Sentiment | n8n-nodes-base.code | Data Analysis | Get Dish Mentions for Report | AI Write Weekly Insights | 📊 Weekly Insights |
| AI Write Weekly Insights | @n8n/n8n-nodes-langchain.openAi | AI Processing | Aggregate Weekly Dish Sentiment | Build Weekly Report | 📊 Weekly Insights |
| Build Weekly Report | n8n-nodes-base.code | Transformation | AI Write Weekly Insights | Email Weekly Report to Owner, Send Weekly Summary to Telegram, Build Weekly Report Row | 📬 Report Delivery |
| Email Weekly Report to Owner | n8n-nodes-base.gmail | Email Dispatch | Build Weekly Report | None | 📬 Report Delivery |
| Send Weekly Summary to Telegram | n8n-nodes-base.telegram | Messaging | Build Weekly Report | None | 📬 Report Delivery |
| Build Weekly Report Row | n8n-nodes-base.code | Transformation | Build Weekly Report | Append to Weekly Reports | 📬 Report Delivery |
| Append to Weekly Reports | n8n-nodes-base.googleSheets | Persistence | Build Weekly Report Row | None | 📬 Report Delivery |

---

### 4. Reproducing the Workflow from Scratch

Follow this step-by-step procedure to manually build and configure the workflow inside n8n:

1. **Prepare Google Sheets & Credentials:**
   - Create a Google Sheets file containing four sheets: `Reviews Log`, `Dish Mentions`, `Weekly Reports`, and a separate sheet named `Menu` (with a column `dish_name`).
   - Configure credentials in n8n for **Google Sheets** (OAuth2), **Gmail** (OAuth2), **Telegram** (Bot Token), **OpenAI** (API Key), and **HTTP Header Auth** (for review feeds).

2. **Build the Review Intake Stream:**
   - **Step 1:** Create a `Schedule Trigger` node (`Poll Reviews Every 30 Min`) set to run every 30 minutes. Connect its output to an `HTTP Request` node (`Fetch Swiggy & Zomato Reviews`). Configure the HTTP Request URL (`https://YOUR-REVIEW-FEED.example.com/reviews?since=last_30_min`), set authentication to Generic Credential Type (HTTP Header Auth), enable retries (3 tries, 5000ms delay), and set on-error behavior to "Continue Regular Output".
   - **Step 2:** Create a `Webhook` node (`Receive Reviews via Webhook`) with path `restaurant-reviews` and method `POST`.
   - **Step 3:** Create a `Code` node (`Normalize Review Payloads`). Connect outputs from both the HTTP Request node and the Webhook node into this node. Paste the normalization script logic to standardize payloads.

3. **Build the Configuration & Deduplication Block:**
   - **Step 4:** Add a `Set` node (`Set Monitor Config`). Assign parameters including `restaurant_name`, `brand_voice`, `sheet_id`, `menu_sheet_id`, sheet name variables, `telegram_chat_id`, `escalation_email`, and `critical_rating_threshold` (value: `2`). Connect `Normalize Review Payloads` to this node.
   - **Step 5:** Add a `Google Sheets` node (`Get Logged Reviews`). Set operation to "Get Many/All", use expression mapping for document ID and sheet name derived from `Set Monitor Config`. Enable "Execute Once".
   - **Step 6:** Add a second `Google Sheets` node (`Get Menu Items`) to fetch the menu sheet data. Enable "Execute Once". Connect `Get Logged Reviews` output to this node.
   - **Step 7:** Add a `Code` node (`Filter New Reviews`) to remove already processed review IDs and compile the canonical menu list string. Connect `Get Menu Items` output here.

4. **Build the AI Analysis & Logging Block:**
   - **Step 8:** Add an `OpenAI` advanced AI node (`AI Analyze Review & Draft Reply`). Select model `gpt-4o-mini`, temperature `0.4`, set response format to JSON, and configure the system prompt and user message bindings referencing the review data and menu list. Connect `Filter New Reviews` to this node.
   - **Step 9:** Add a `Code` node (`Merge Review with AI Analysis`) to combine raw reviews with AI analysis, compute sentiment scores, urgency tags, and pre-format HTML emails and Telegram text.
   - **Step 10:** Create two parallel branches from the merge node for sheet logging:
     - Add a `Code` node (`Build Reviews Log Row`) connected to a `Google Sheets` node (`Append to Reviews Log`) targeting the reviews sheet.
     - Add a `Code` node (`Split Dish Mentions`) connected to a `Google Sheets` node (`Append to Dish Mentions`) targeting the dish mentions sheet.

5. **Build the Alerting Block:**
   - **Step 11:** Add an `If` node (`Is Critical Review?`) checking whether `{{ $json.is_critical }}` is true. Connect `Merge Review with AI Analysis` output to this node.
   - **Step 12:** Configure true branch outputs:
     - Connect to a `Telegram` node (`Send Critical Alert to Telegram`) using HTML parse mode and dynamic chat ID.
     - Connect to a `Gmail` node (`Email Manager Escalation`) sending to the configured escalation email.
   - **Step 13:** Configure false branch output:
     - Connect to a `Telegram` node (`Send Reply Draft to Telegram`) for standard review draft delivery.

6. **Build the Weekly Reporting Stream:**
   - **Step 14:** Create a `Schedule Trigger` node (`Every Monday 9 AM`) configured for weekly execution on Mondays at 9 AM.
   - **Step 15:** Add a `Set` node (`Set Report Config`) to define reporting variables (`reports_sheet`, `report_email`, `min_mentions_for_ranking`, etc.).
   - **Step 16:** Add sequential `Google Sheets` nodes (`Get Reviews for Report` and `Get Dish Mentions for Report`) configured to retrieve all records from their respective sheets with "Execute Once" enabled.
   - **Step 17:** Add a `Code` node (`Aggregate Weekly Dish Sentiment`) to process week-over-week analytics, platform breakdowns, and issue frequency counts.
   - **Step 18:** Add an `OpenAI` node (`AI Write Weekly Insights`) using model `gpt-4o-mini` (temperature `0.3`) to synthesize executive summaries, wins, and action items from aggregated JSON metrics.
   - **Step 19:** Add a `Code` node (`Build Weekly Report`) to generate final report HTML markup, Telegram text summaries, and row objects.
   - **Step 20:** Connect report build outputs to:
     - A `Gmail` node (`Email Weekly Report to Owner`) for email distribution.
     - A `Telegram` node (`Send Weekly Summary to Telegram`) for chat summaries.
     - A `Code` node (`Build Weekly Report Row`) linked to a `Google Sheets` node (`Append to Weekly Reports`) for historical archival.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| RestaurantOS Rating Monitor & Weekly Dish Sentiment Report | Primary application workflow design for Swiggy and Zomato integration. |