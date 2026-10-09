Confirm COD Shopify orders and manage RTO with GPT-4o-mini, WATI, and Shiprocket

https://n8nworkflows.xyz/workflows/confirm-cod-shopify-orders-and-manage-rto-with-gpt-4o-mini--wati--and-shiprocket-20523


# Confirm COD Shopify orders and manage RTO with GPT-4o-mini, WATI, and Shiprocket

### 1. Workflow Overview

This workflow is designed for Indian Direct-to-Consumer (D2C) e-commerce brands to mitigate Return to Origin (RTO) rates and combat Cash on Delivery (COD) fraud. It automatically ingests new Shopify orders, evaluates them via a dual risk engine (rule-based scoring combined with OpenAI GPT-4o-mini address analysis), initiates automated WhatsApp confirmations via WATI, and coordinates fulfillment through Shiprocket. It also manages order exceptions, operational holds, automated reminders, and delivery exceptions (NDR - Non-Delivery Reports) while maintaining an updated central ledger in Google Sheets and alerting operations staff via Telegram.

The workflow logic is categorized into the following functional blocks:

*   **1.1 Input Reception:** Ingests external triggers from Shopify webhooks, incoming WATI WhatsApp customer interactions, Shiprocket courier tracking webhooks, an internal operational review form, and a 15-minute scheduled timer.
*   **1.2 Configuration & Data Loading:** Establishes global store environment variables and loads the existing state of all tracked orders from Google Sheets.
*   **1.3 Order Evaluation & AI Risk Scoring:** Normalizes order data, calculates rule-based RTO risk points, queries OpenAI (`gpt-4o-mini`) for qualitative address validation, and blends the scores to categorize orders into Low, Medium, or High-risk bands.
*   **1.4 Action Routing & Execution:** Evaluates state machines to dispatch specific tasks—such as inserting/updating Google Sheets, modifying Shopify order tags or canceling orders, transmitting WATI message templates, pushing shipments to Shiprocket, and sending Telegram notifications.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception
*   **Overview:** Serves as the multi-channel entry point for the automation. It captures real-time store events, customer replies, delivery updates, human interventions, and periodic batch cycles.
*   **Nodes Involved:** `New Shopify Order`, `Receive WATI & Shiprocket Events`, `Ops Review Form (Held Orders)`, `Check Every 15 Minutes`.
*   **Node Details:**
    *   **New Shopify Order**
        *   *Type & Technical Role:* `n8n-nodes-base.shopifyTrigger` (Webhook Trigger). Listens for newly created orders (`orders/create`).
        *   *Configuration:* Authenticates via Shopify Admin Access Token.
        *   *Inputs / Outputs:* No inputs; outputs order JSON payload to `Set Store Config`.
        *   *Edge Cases:* Webhook delivery retries could trigger duplicates; handled downstream in code via ID checking.
    *   **Receive WATI & Shiprocket Events**
        *   *Type & Technical Role:* `n8n-nodes-base.webhook` (Webhook Trigger). Receives POST payloads from WATI (customer responses) and Shiprocket (tracking scans/NDR updates).
        *   *Configuration:* Path set to `cod-events`, accepting `POST` requests.
        *   *Inputs / Outputs:* No inputs; outputs event payload to `Set Store Config`.
        *   *Edge Cases:* Unauthenticated Shiprocket requests must be filtered using `x-api-key` headers matching store configuration.
    *   **Ops Review Form (Held Orders)**
        *   *Type & Technical Role:* `n8n-nodes-base.formTrigger` (Form Trigger). Presents an internal UI form to the operations team to review, release, or cancel held orders.
        *   *Configuration:* Path set to `cod-ops-review`. Defines fields for Order Number, Decision (Dropdown: "Release to Shiprocket", "Cancel Order", "Keep On Hold"), and Notes.
        *   *Inputs / Outputs:* No inputs; outputs form submission data to `Set Store Config`.
    *   **Check Every 15 Minutes**
        *   *Type & Technical Role:* `n8n-nodes-base.scheduleTrigger` (Timer Trigger). Fires every 15 minutes to evaluate timed thresholds (reminders, auto-cancellations, shipment dispatches).
        *   *Configuration:* Interval set to `15` minutes.
        *   *Inputs / Outputs:* No inputs; outputs execution tick to `Set Store Config`.

#### 2.2 Configuration & Data Loading
*   **Overview:** Sets up global store constants, API endpoints, risk thresholds, and pulls the current ledger of active orders from Google Sheets to provide context for incoming requests.
*   **Nodes Involved:** `Set Store Config`, `Get COD Orders Sheet`.
*   **Node Details:**
    *   **Set Store Config**
        *   *Type & Technical Role:* `n8n-nodes-base.set` (Data Transformation / Constant Definition). Injects execution-wide parameters into the data stream.
        *   *Configuration:* Defines store parameters, API domains, threshold metrics (`hold_threshold: 70`, `medium_threshold: 40`), WATI template names, and sheet identifiers. Includes incoming payload references.
        *   *Inputs / Outputs:* Receives data from any of the four trigger nodes; outputs configuration object along with incoming payload data to `Get COD Orders Sheet`.
    *   **Get COD Orders Sheet**
        *   *Type & Technical Role:* `n8n-nodes-base.googleSheets` (Data Reader). Retrieves all rows from the designated Google Sheets order ledger.
        *   *Configuration:* References `documentId` and `sheetName` dynamically from the `Set Store Config` node. Set to execute once (`executeOnce: true`) and always output data (`alwaysOutputData: true`).
        *   *Inputs / Outputs:* Input from `Set Store Config`; outputs sheet rows to `Decide Next Actions per Order`.
        *   *Edge Cases:* API rate limits or invalid Google Sheet credentials will halt execution.

#### 2.3 Order Evaluation & AI Risk Scoring
*   **Overview:** Acts as the core processing engine. It ingests triggers and sheet rows, calculates a rule-based risk score, decides whether AI evaluation is required, queries OpenAI, and normalizes risk bands.
*   **Nodes Involved:** `Decide Next Actions per Order`, `Needs AI Score?`, `AI Assess RTO Risk`, `Apply AI Risk Score`.
*   **Node Details:**
    *   **Decide Next Actions per Order**
        *   *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript Execution Node). Evaluates the incoming event source (Shopify, WATI, Shiprocket, Form, or Schedule) against all loaded sheet rows and determines necessary follow-up tasks.
        *   *Configuration:* Custom JavaScript block containing logic for rule-based risk scoring (first-time customer, address length, pincode validation, phone number patterns, historical RTOs) and time-delta evaluations.
        *   *Inputs / Outputs:* Input from `Get COD Orders Sheet`; outputs structured action arrays and execution flags to `Needs AI Score?`.
    *   **Needs AI Score?**
        *   *Type & Technical Role:* `n8n-nodes-base.if` (Conditional Branching). Routes newly created orders to the AI risk assessor, bypassing execution for existing order updates.
        *   *Configuration:* Evaluates condition `{{ $json.needs_ai }} === true`.
        *   *Inputs / Outputs:* Input from `Decide Next Actions per Order`; True branch routes to `AI Assess RTO Risk`, False branch routes to `Split Into Actions`.
    *   **AI Assess RTO Risk**
        *   *Type & Technical Role:* `@n8n/n8n-nodes-langchain.openAi` (LLM Integration). Sends order metadata and address patterns to GPT-4o-mini for fraud and RTO probability analysis.
        *   *Configuration:* Model configured as `gpt-4o-mini` with a temperature of `0.2`. Enforces strict JSON return schema containing `risk_score`, `risk_band`, `top_reasons`, `address_quality`, `address_issues`, and `suggested_action`.
        *   *Inputs / Outputs:* Input from `Needs AI Score?` (True branch); outputs AI text response to `Apply AI Risk Score`.
        *   *Edge Cases:* API timeouts or malformed JSON responses from the LLM; handled gracefully via parsing try-catch blocks downstream.
    *   **Apply AI Risk Score**
        *   *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript Execution Node). Blends rule-based score (60%) with OpenAI's score (40%), applies history overrides, creates the Google Sheets append action, sets up Shopify tags, and queues WATI confirmation templates and Telegram alerts.
        *   *Configuration:* Custom JavaScript block processing upstream LLM output and combining it with original order contexts.
        *   *Inputs / Outputs:* Input from `AI Assess RTO Risk`; outputs flattened action arrays to `Split Into Actions`.

#### 2.4 Action Routing & Execution
*   **Overview:** Flattens batch arrays into individual execution items and routes them to their respective integrations: Google Sheets, Shopify, WATI (WhatsApp), Shiprocket, and Telegram.
*   **Nodes Involved:** `Split Into Actions`, `Route by Action Type`, `Add New Order to Sheet`, `Prepare Row Update`, `Update Order Row in Sheet`, `Send WhatsApp via WATI`, `Update Order in Shopify`, `Log In to Shiprocket`, `Attach Shiprocket Token`, `Create Shipment or NDR Action in Shiprocket`, `Record Shiprocket Result`, `Save Shiprocket Result to Sheet`, `Shiprocket Failed?`, `Alert Ops on Shiprocket Error`, `Send Ops Alert on Telegram`.
*   **Node Details:**
    *   **Split Into Actions**
        *   *Type & Technical Role:* `n8n-nodes-base.code` (Data Normalization). Unpacks composite arrays of actions into individual items for independent branch routing.
        *   *Inputs / Outputs:* Input from `Needs AI Score?` (False branch) or `Apply AI Risk Score`; outputs individual action objects to `Route by Action Type`.
    *   **Route by Action Type**
        *   *Type & Technical Role:* `n8n-nodes-base.switch` (Multi-way Router). Evaluates `{{ $json.type }}` to distribute items across six parallel integration pipelines.
        *   *Configuration:* Routes: `append` (Output 0), `update` (Output 1), `wati` (Output 2), `shopify` (Output 3), `shiprocket` (Output 4), `telegram` (Output 5).
    *   **Add New Order to Sheet**
        *   *Type & Technical Role:* `n8n-nodes-base.googleSheets` (Data Writer). Appends newly processed order records to the tracking ledger.
        *   *Inputs / Outputs:* Input from `Route by Action Type` (Output 0).
    *   **Prepare Row Update**
        *   *Type & Technical Role:* `n8n-nodes-base.code` (Data Transformation). Extracts inner update payloads before writing to Google Sheets.
        *   *Inputs / Outputs:* Input from `Route by Action Type` (Output 1); outputs to `Update Order Row in Sheet`.
    *   **Update Order Row in Sheet**
        *   *Type & Technical Role:* `n8n-nodes-base.googleSheets` (Data Updater). Modifies existing rows in the tracking ledger matched by `order_id`.
        *   *Inputs / Outputs:* Input from `Prepare Row Update`.
    *   **Send WhatsApp via WATI**
        *   *Type & Technical Role:* `n8n-nodes-base.httpRequest` (API Integration). Sends pre-approved WhatsApp templates or session messages via WATI.
        *   *Configuration:* HTTP POST request using Generic Credential Type (`httpHeaderAuth`). Includes error continuation settings and retry logic.
        *   *Inputs / Outputs:* Input from `Route by Action Type` (Output 2).
    *   **Update Order in Shopify**
        *   *Type & Technical Role:* `n8n-nodes-base.httpRequest` (API Integration). Updates order tags or cancels orders in Shopify.
        *   *Configuration:* Authenticated via Shopify Admin API access token credentials.
        *   *Inputs / Outputs:* Input from `Route by Action Type` (Output 3).
    *   **Log In to Shiprocket**
        *   *Type & Technical Role:* `n8n-nodes-base.httpRequest` (API Authentication). Authenticates against Shiprocket API v2 to retrieve a bearer token.
        *   *Configuration:* POST request to `https://apiv2.shiprocket.in/v1/external/auth/login` using Custom Auth credentials (`httpCustomAuth`). Executes once (`executeOnce: true`).
        *   *Inputs / Outputs:* Input from `Route by Action Type` (Output 4); outputs auth token to `Attach Shiprocket Token`.
    *   **Attach Shiprocket Token**
        *   *Type & Technical Role:* `n8n-nodes-base.code` (Data Combination). Merges the authentication token with queued Shiprocket action payloads.
        *   *Inputs / Outputs:* Input from `Log In to Shiprocket` and `Route by Action Type` (Output 4); outputs authenticated request payload to `Create Shipment or NDR Action in Shiprocket`.
    *   **Create Shipment or NDR Action in Shiprocket**
        *   *Type & Technical Role:* `n8n-nodes-base.httpRequest` (API Integration). Creates ad-hoc shipments or triggers NDR actions (re-attempt/return) in Shiprocket.
        *   *Configuration:* POST request using Bearer token authentication headers.
        *   *Inputs / Outputs:* Input from `Attach Shiprocket Token`; outputs API response to `Record Shiprocket Result`.
    *   **Record Shiprocket Result**
        *   *Type & Technical Role:* `n8n-nodes-base.code` (Data Transformation). Parses Shiprocket API responses, generates success/failure audit statuses, and formats notification text.
        *   *Inputs / Outputs:* Input from `Create Shipment or NDR Action in Shiprocket`; outputs update objects to `Save Shiprocket Result to Sheet` and `Shiprocket Failed?`.
    *   **Save Shiprocket Result to Sheet**
        *   *Type & Technical Role:* `n8n-nodes-base.googleSheets` (Data Updater). Persists shipment IDs, AWB codes, couriers, and tracking statuses back to the sheet ledger.
        *   *Inputs / Outputs:* Input from `Record Shiprocket Result`.
    *   **Shiprocket Failed?**
        *   *Type & Technical Role:* `n8n-nodes-base.if` (Conditional Branching). Checks if a Shiprocket API call failed (`{{ $json.failed === true }}`).
        *   *Inputs / Outputs:* Input from `Record Shiprocket Result`; True branch routes to `Alert Ops on Shiprocket Error`.
    *   **Alert Ops on Shiprocket Error**
        *   *Type & Technical Role:* `n8n-nodes-base.telegram` (Messaging Integration). Sends operational failure alerts to the designated Telegram chat group.
        *   *Inputs / Outputs:* Input from `Shiprocket Failed?` (True branch).
    *   **Send Ops Alert on Telegram**
        *   *Type & Technical Role:* `n8n-nodes-base.telegram` (Messaging Integration). Dispatches high-risk order alerts, auto-cancellation notifications, or operational escalations to Telegram.
        *   *Configuration:* Uses HTML parse mode and dynamic Chat ID retrieval from store config.
        *   *Inputs / Outputs:* Input from `Route by Action Type` (Output 5).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note – Overview | `n8n-nodes-base.stickyNote` | Documentation block outlining purpose, workflow mechanics, and setup instructions. | None | None | ## 📦 COD Order Confirmation & RTO Reducer (Shopify + WATI + Shiprocket)<br><br>For Indian D2C brands where a big share of COD orders come back as RTO, costing two-way shipping on parcels nobody wanted. Most of that is avoidable if you confirm the order and check the address before it ships.<br><br>### How it works<br>When a COD order lands in Shopify it's scored for RTO risk, using rules plus an AI check on the address and pincode, and the customer gets a WhatsApp asking them to confirm or cancel. Confirmed low-risk orders go straight to Shiprocket. Risky ones are held for a quick ops call and released or cancelled from a form. Unanswered orders get reminders and are auto-cancelled after 24 hours.<br><br>After dispatch, courier updates trigger an out-for-delivery message, and failed deliveries (NDR) ask the customer what to do before re-attempting or escalating. Everything is tracked in one Google Sheet.<br><br>### Setup steps<br>1. Create a Google Sheet with a **COD Orders** tab.<br>2. Connect Shopify, Google Sheets, OpenAI and Telegram credentials.<br>3. Add WATI as Header Auth and Shiprocket API user login as Custom Auth (see the security note).<br>4. Get the four WATI templates approved and fill in **Set Store Config**: shop domain, WATI endpoint, sheet ID, thresholds and webhook token.<br>5. Point WATI and Shiprocket webhooks to **Receive WATI & Shiprocket Events**, place a test COD order, then activate. |
| Sticky Note – Event Triggers | `n8n-nodes-base.stickyNote` | Documentation block grouping all event triggers. | None | None | ## 📥 Event Triggers<br>Four ways in, one brain: new Shopify orders, customer replies and courier updates from webhooks, ops decisions from the review form, and a 15-minute schedule for follow-ups. |
| Sticky Note – Load & Decide | `n8n-nodes-base.stickyNote` | Documentation block grouping data loading and state decision logic. | None | None | ## 🧭 Load & Decide<br>Loads every tracked order and decides what each one needs right now: a confirmation, a reminder, a hold, a dispatch or an NDR follow-up. Rule-based risk signals are checked here first. |
| Sticky Note – AI Risk Score & Dispatch | `n8n-nodes-base.stickyNote` | Documentation block grouping AI evaluation and branch splitting. | None | None | ## 🧠 AI Risk Score & Dispatch<br>New COD orders get a 0–100 RTO risk score from GPT-4o-mini, which checks address quality and pincode/city mismatches. Every planned action is then split out and routed. |
| Sticky Note – Sheet Updates | `n8n-nodes-base.stickyNote` | Documentation block grouping Google Sheets write operations. | None | None | ## 🗂️ Sheet Updates<br>New orders are added to the tracker, and every status change is written back so the next run knows exactly where each order stands. |
| Sticky Note – WhatsApp & Shopify Actions | `n8n-nodes-base.stickyNote` | Documentation block grouping messaging and e-commerce platform modifications. | None | None | ## 💬 WhatsApp & Shopify Actions<br>WATI sends the confirmation, reminder, out-for-delivery and failed-delivery templates. Shopify gets tags, notes and cancellations so your store always matches reality. |
| Sticky Note – Shiprocket & Ops Alerts | `n8n-nodes-base.stickyNote` | Documentation block grouping fulfillment and operational alerting actions. | None | None | ## 🚚 Shiprocket & Ops Alerts<br>Confirmed orders are pushed to Shiprocket and NDRs are re-attempted, with the result saved to the sheet. Failed calls and orders that need a human ping the ops group on Telegram. |
| Sticky Note – Credentials & Security | `n8n-nodes-base.stickyNote` | Documentation block outlining credential storage requirements and token security. | None | None | ## 🔐 Credentials & Security<br>Shiprocket login lives in a Custom Auth credential, not the config node: `{"body":{"email":"api-user@example.com","password":"…"}}`. Use a dedicated Shiprocket API user. Change `shiprocket_webhook_token` and keep all webhook URLs private. |
| New Shopify Order | `n8n-nodes-base.shopifyTrigger` | Triggers workflow on new Shopify orders. | None | Set Store Config | |
| Receive WATI & Shiprocket Events | `n8n-nodes-base.webhook` | Ingests incoming WhatsApp responses and courier tracking webhooks. | None | Set Store Config | |
| Ops Review Form (Held Orders) | `n8n-nodes-base.formTrigger` | Ingests human operational decisions for held orders. | None | Set Store Config | |
| Check Every 15 Minutes | `n8n-nodes-base.scheduleTrigger` | Triggers periodic batch checks every 15 minutes. | None | Set Store Config | |
| Set Store Config | `n8n-nodes-base.set` | Injects global store constants and configuration variables. | New Shopify Order, Receive WATI & Shiprocket Events, Ops Review Form (Held Orders), Check Every 15 Minutes | Get COD Orders Sheet | |
| Get COD Orders Sheet | `n8n-nodes-base.googleSheets` | Loads all rows from the Google Sheets order ledger. | Set Store Config | Decide Next Actions per Order | |
| Decide Next Actions per Order | `n8n-nodes-base.code` | Evaluates events against spreadsheet data to determine necessary actions. | Get COD Orders Sheet | Needs AI Score? | |
| Needs AI Score? | `n8n-nodes-base.if` | Routes new orders to AI evaluation and existing updates to action splitting. | Decide Next Actions per Order | AI Assess RTO Risk, Split Into Actions | |
| AI Assess RTO Risk | `@n8n/n8n-nodes-langchain.openAi` | Queries GPT-4o-mini to assess address quality and RTO/fraud probability. | Needs AI Score? | Apply AI Risk Score | |
| Apply AI Risk Score | `n8n-nodes-base.code` | Blends rule and AI risk scores, constructs tracking rows, and queues actions. | AI Assess RTO Risk | Split Into Actions | |
| Split Into Actions | `n8n-nodes-base.code` | Flattens batch arrays into individual action items. | Needs AI Score?, Apply AI Risk Score | Route by Action Type | |
| Route by Action Type | `n8n-nodes-base.switch` | Routes individual actions to their respective integration pipelines. | Split Into Actions | Add New Order to Sheet, Prepare Row Update, Send WhatsApp via WATI, Update Order in Shopify, Log In to Shiprocket, Send Ops Alert on Telegram | |
| Add New Order to Sheet | `n8n-nodes-base.googleSheets` | Appends new order records to the spreadsheet ledger. | Route by Action Type | None | |
| Prepare Row Update | `n8n-nodes-base.code` | Prepares extracted update objects for Google Sheets modification. | Route by Action Type | Update Order Row in Sheet | |
| Send WhatsApp via WATI | `n8n-nodes-base.httpRequest` | Transmits WhatsApp template or session messages using WATI. | Route by Action Type | None | |
| Update Order in Shopify | `n8n-nodes-base.httpRequest` | Updates tags or cancels orders in Shopify. | Route by Action Type | None | |
| Log In to Shiprocket | `n8n-nodes-base.httpRequest` | Authenticates against Shiprocket to obtain session tokens. | Route by Action Type | Attach Shiprocket Token | |
| Send Ops Alert on Telegram | `n8n-nodes-base.telegram` | Sends operational escalation or high-risk warnings to Telegram. | Route by Action Type | None | |
| Update Order Row in Sheet | `n8n-nodes-base.googleSheets` | Modifies existing spreadsheet ledger rows by Order ID. | Prepare Row Update | None | |
| Attach Shiprocket Token | `n8n-nodes-base.code` | Attaches the retrieved Shiprocket bearer token to execution payloads. | Log In to Shiprocket | Create Shipment or NDR Action in Shiprocket | |
| Create Shipment or NDR Action in Shiprocket | `n8n-nodes-base.httpRequest` | Creates shipments or submits NDR resolution actions in Shiprocket. | Attach Shiprocket Token | Record Shiprocket Result | |
| Record Shiprocket Result | `n8n-nodes-base.code` | Audits Shiprocket API responses and structures error flags. | Create Shipment or NDR Action in Shiprocket | Save Shiprocket Result to Sheet, Shiprocket Failed? | |
| Save Shiprocket Result to Sheet | `n8n-nodes-base.googleSheets` | Persists shipment tracking numbers and statuses to the sheet. | Record Shiprocket Result | None | |
| Shiprocket Failed? | `n8n-nodes-base.if` | Checks if a Shiprocket API request resulted in an error. | Record Shiprocket Result | Alert Ops on Shiprocket Error | |
| Alert Ops on Shiprocket Error | `n8n-nodes-base.telegram` | Alerts the operations team via Telegram when Shiprocket integration fails. | Shiprocket Failed? | None | |

---

### 4. Reproducing the Workflow from Scratch

Follow this step-by-step procedure to rebuild the workflow manually in n8n:

1.  **Initialize the Triggers:**
    *   Create a **Shopify Trigger** node (`New Shopify Order`). Set topic to `orders/create` and authenticate with your Shopify access token.
    *   Create a **Webhook** node (`Receive WATI & Shiprocket Events`). Set method to `POST` and path to `cod-events`.
    *   Create a **Form Trigger** node (`Ops Review Form (Held Orders)`). Set path to `cod-ops-review`, title to `Review Held COD Order`, and add fields: `Order Number` (Text, required), `Decision` (Dropdown with options: `Release to Shiprocket`, `Cancel Order`, `Keep On Hold`, required), and `Notes` (Textarea).
    *   Create a **Schedule Trigger** node (`Check Every 15 Minutes`). Set interval to every `15` minutes.

2.  **Configure Environment Constants & Data Loading:**
    *   Create a **Set** node (`Set Store Config`). Connect all four triggers to this node. Add string and number assignments matching your store configuration (e.g., `store_name`, `country_code` (`91`), `shopify_shop`, `shopify_api_version`, `wati_api_endpoint`, template names, `ops_telegram_chat_id`, `sheet_id`, `orders_sheet` (`COD Orders`), `shiprocket_pickup_location`, thresholds). Ensure "Include Other Fields" is enabled.
    *   Create a **Google Sheets** node (`Get COD Orders Sheet`). Connect input from `Set Store Config`. Operation: `Get Many`, Document ID: `={{ $('Set Store Config').first().json.sheet_id }}`, Sheet Name: `={{ $('Set Store Config').first().json.orders_sheet }}`. Enable `Execute Once` and `Always Output Data`.

3.  **Build the Decision & AI Scoring Engine:**
    *   Create a **Code** node (`Decide Next Actions per Order`). Connect input from `Get COD Orders Sheet`. Paste the JavaScript logic that parses incoming triggers against sheet rows, calculates rule-based risk scores, and generates action arrays.
    *   Create an **If** node (`Needs AI Score?`). Connect input from `Decide Next Actions per Order`. Set condition: `{{ $json.needs_ai }}` equals `true`.
    *   Create an **OpenAI** node (`AI Assess RTO Risk`). Connect the True branch of `Needs AI Score?`. Select model `gpt-4o-mini`, temperature `0.2`, set message role to `system` with the RTO analyst prompt, and user content to `={{ $json.ai_prompt }}`. Enable JSON output parsing.
    *   Create a **Code** node (`Apply AI Risk Score`). Connect input from `AI Assess RTO Risk`. Paste the JavaScript logic that blends rule scores and AI scores, determines risk bands (`Low`, `Medium`, `High`), formats Google Sheets append actions, and queues WATI confirmation messages and Telegram alerts.

4.  **Implement Action Splitting and Routing:**
    *   Create a **Code** node (`Split Into Actions`). Connect input from `Needs AI Score?` (False branch) and `Apply AI Risk Score`. Paste the iteration script that flattens action arrays into individual items.
    *   Create a **Switch** node (`Route by Action Type`). Connect input from `Split Into Actions`. Configure 6 output rules based on `{{ $json.type }}` equaling:
        1. `append` (Rename output: `Append Row`)
        2. `update` (Rename output: `Update Row`)
        3. `wati` (Rename output: `WhatsApp`)
        4. `shopify` (Rename output: `Shopify`)
        5. `shiprocket` (Rename output: `Shiprocket`)
        6. `telegram` (Rename output: `Telegram`)

5.  **Build Downstream Integration Branches:**
    *   *Append Row:* Connect Switch output 0 to a **Google Sheets** node (`Add New Order to Sheet`). Operation: `Append`, mapping mode: `Define Below`, mapping sheet columns to `$json.row.*` properties.
    *   *Update Row:* Connect Switch output 1 to a **Code** node (`Prepare Row Update`) to unwrap `$json.update`, then connect to a **Google Sheets** node (`Update Order Row in Sheet`) with operation `Update`, matching column `order_id`.
    *   *WhatsApp:* Connect Switch output 2 to an **HTTP Request** node (`Send WhatsApp via WATI`). Method `POST`, URL `={{ $json.url }}`, Body `={{ JSON.stringify($json.body) }}`, Authentication: Generic Credential Type (`httpHeaderAuth`). Enable `Continue On Fail` and retry options.
    *   *Shopify:* Connect Switch output 3 to an **HTTP Request** node (`Update Order in Shopify`). Method `={{ $json.method }}`, URL `={{ $json.url }}`, Body `={{ JSON.stringify($json.body) }}`, Authentication: Predefined (`shopifyAccessTokenApi`). Enable retries.
    *   *Shiprocket (Fulfillment):*
        *   Connect Switch output 4 to an **HTTP Request** node (`Log In to Shiprocket`). Method `POST`, URL `https://apiv2.shiprocket.in/v1/external/auth/login`, Authentication: Custom Auth (`httpCustomAuth`). Enable `Execute Once`.
        *   Connect to a **Code** node (`Attach Shiprocket Token`) to combine the auth token with queued requests.
        *   Connect to an **HTTP Request** node (`Create Shipment or NDR Action in Shiprocket`). Method `POST`, URL `=https://apiv2.shiprocket.in/v1/external{{ $json.path }}`, Body `={{ JSON.stringify($json.body) }}`, Header `Authorization: =Bearer {{ $json.token }}`.
        *   Connect to a **Code** node (`Record Shiprocket Result`) to audit success/failure.
        *   Branch result to a **Google Sheets** node (`Save Shiprocket Result to Sheet`) (Operation: `Update`, matching `order_id`) and an **If** node (`Shiprocket Failed?`) (`{{ $json.failed === true }}`).
        *   Connect True branch of `Shiprocket Failed?` to a **Telegram** node (`Alert Ops on Shiprocket Error`).
    *   *Telegram:* Connect Switch output 5 to a **Telegram** node (`Send Ops Alert on Telegram`). Chat ID `={{ $('Set Store Config').first().json.ops_telegram_chat_id }}`, Text `={{ $json.text }}`, Parse Mode `HTML`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Shiprocket Custom Auth configuration requirement | Must use Custom Auth credentials structured as `{"body":{"email":"api-user@example.com","password":"…"}}` rather than standard bearer credentials. |
| Dedicated API user recommendation | Recommended to create a dedicated API user account within Shiprocket specifically for this automation integration. |
| Webhook security protocol | Ensure `shiprocket_webhook_token` is updated and all incoming webhook endpoints remain secure and private. |