Track menu food costs and flag margin erosion with GPT-6-luna, Google Sheets and Gmail

https://n8nworkflows.xyz/workflows/track-menu-food-costs-and-flag-margin-erosion-with-gpt-6-luna--google-sheets-and-gmail-20442


# Track menu food costs and flag margin erosion with GPT-6-luna, Google Sheets and Gmail

### 1. Workflow Overview

This workflow is an automated financial monitoring system designed for restaurant and kitchen operations. It tracks supplier price fluctuations, dynamically recalculates menu item production costs, flags instances of margin erosion when food costs exceed target thresholds, and provides AI-powered weekly executive summaries with pricing recommendations. 

The implementation logic is divided into three primary operational blocks:
- **1.1 Price Intake & Validation:** Receives incoming supplier price updates via webhook, normalizes the data payload, validates required fields, logs valid pricing updates into a centralized Google Sheets tracker, and issues email confirmations or rejections accordingly.
- **1.2 Daily Margin Sweep:** Executes daily at 06:00 to ingest pricing, recipe composition, and menu pricing data from Google Sheets, compute exact recipe portion costs, evaluate profit margins against predefined targets, log over-budget items to an alert sheet, and distribute immediate email alerts to management.
- **1.3 Weekly Menu Digest:** Triggers every Monday at 07:00 to aggregate weekly cost metrics, process them through an advanced AI agent utilizing a language model, and dispatch a concise plain-text executive summary detailing margin health and strategic pricing suggestions.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Price Intake & Validation

- **Overview:** This block acts as the system's entry point for external price modifications. It captures webhook data, standardizes field entries, validates data integrity, updates the master sheet, and notifies stakeholders.
- **Nodes Involved:** 
  - `When Price Update Received`
  - `Normalize Price Update`
  - `Validate Price Update`
  - `Log New Price`
  - `Email Confirmation`
  - `Email Reject Notice`
- **Node Details:**
  - **When Price Update Received**
    - *Type & Role:* Webhook node (`n8n-nodes-base.webhook`) acting as the primary trigger listening for incoming HTTP POST requests.
    - *Configuration:* Listens on path `menu-cost-intake-0926` using the POST method.
    - *Key Expressions:* None (triggers on payload arrival).
    - *Connections:* Input: None (Trigger) | Output: `Normalize Price Update`.
    - *Edge Cases / Failure Types:* Network timeout, invalid HTTP method, or missing endpoint route configuration.
  - **Normalize Price Update**
    - *Type & Role:* Set node (`n8n-nodes-base.set`) used to format and clean incoming JSON data properties.
    - *Configuration:* Assigns dynamic string values to standard fields (`update_id`, `update_received_at`, `ingredient`, `unit_price`, `unit`, `supplier`, `submitted_by`, `owner_email`).
    - *Key Expressions:* Generates an alphanumeric ID using `={{ 'PRC-' + $now.toFormat('yyyyLLdd-HHmmss') + '-' + String(Math.floor(Math.random() * 9000) + 1000) }}` and formats incoming strings using standard null-coalescing and `.trim()` operations.
    - *Connections:* Input: `When Price Update Received` | Output: `Validate Price Update`.
    - *Edge Cases / Failure Types:* Malformed JSON payloads causing runtime reference errors.
  - **Validate Price Update**
    - *Type & Role:* If node (`n8n-nodes-base.if`) implementing conditional logic to verify mandatory parameters.
    - *Configuration:* Validates that both `ingredient` and `unit_price` fields are not empty using loose type validation.
    - *Key Expressions:* `={{ $json.ingredient }}` and `={{ $json.unit_price }}` checked for non-empty string values.
    - *Connections:* Input: `Normalize Price Update` | Output (True): `Log New Price` | Output (False): `Email Reject Notice`.
    - *Edge Cases / Failure Types:* Missing properties throwing undefined reference warnings if safe navigation operators are bypassed.
  - **Log New Price**
    - *Type & Role:* Google Sheets node (`n8n-nodes-base.googleSheets`) operating in append mode.
    - *Configuration:* Appends validated row elements into the "Prices" sheet within the target document (`Menu Cost Tracker 0926`).
    - *Key Expressions:* Maps properties (`ingredient`, `unit_price`, `unit`, `price_date`, `supplier`, `submitted_by`, `update_id`) directly to corresponding spreadsheet columns.
    - *Credentials:* `Feedback Analyzer Sheets` (Google Sheets OAuth2 API).
    - *Connections:* Input: `Validate Price Update` (True) | Output: `Email Confirmation`.
    - *Edge Cases / Failure Types:* API rate limits, revoked sheet permissions, or structural schema mismatches.
  - **Email Confirmation**
    - *Type & Role:* Gmail node (`n8n-nodes-base.gmail`) handling outbound transactional email notifications.
    - *Configuration:* Sends plain-text confirmation emails detailing logged updates to the submitting party or owner.
    - *Key Expressions:* Dynamic recipient assignment `={{ $json.owner_email || 'user@example.com' }}` and template-driven message string arrays joined by newlines.
    - *Credentials:* `Gmail Fresh Sep05` (Gmail OAuth2).
    - *Connections:* Input: `Log New Price` | Output: None.
    - *Edge Cases / Failure Types:* Invalid destination email address strings or authentication token expirations.
  - **Email Reject Notice**
    - *Type & Role:* Gmail node (`n8n-nodes-base.gmail`) managing error feedback dispatch.
    - *Configuration:* Sends a plain-text notification detailing payload rejections due to omitted mandatory fields.
    - *Key Expressions:* Dynamic error description utilizing JSON stringification of the rejected payload body.
    - *Credentials:* `Gmail Fresh Sep05` (Gmail OAuth2).
    - *Connections:* Input: `Validate Price Update` (False) | Output: None.
    - *Edge Cases / Failure Types:* Token expiration or unhandled recipient formatting errors.

---

#### Block 1.2: Daily Margin Sweep

- **Overview:** Executes a daily operational routine at 06:00 to query historical pricing data, recipes, and menu structures, recalculating production costs and logging alerts for items exceeding target financial thresholds.
- **Nodes Involved:**
  - `Daily Margin Sweep`
  - `Read Prices`
  - `Collapse For Recipe Lookup`
  - `Read Recipes`
  - `Collapse For Menu Lookup`
  - `Read Menu`
  - `Compute Recipe Costs`
  - `If Margin Eroded`
  - `Log Cost Alerts`
  - `Build Margin Alert`
  - `Email Margin Alert`
- **Node Details:**
  - **Daily Margin Sweep**
    - *Type & Role:* Schedule Trigger node (`n8n-nodes-base.scheduleTrigger`) initiating automated periodic sweeps.
    - *Configuration:* Configured with a cron expression rule set to run daily at 06:00 (`0 6 * * *`).
    - *Connections:* Input: None (Trigger) | Output: `Read Prices`.
    - *Edge Cases / Failure Types:* Timezone misalignment between system server settings and user expectations.
  - **Read Prices**
    - *Type & Role:* Google Sheets node (`n8n-nodes-base.googleSheets`) fetching pricing records.
    - *Configuration:* Reads all rows from the "Prices" worksheet.
    - *Credentials:* `Feedback Analyzer Sheets` (Google Sheets OAuth2 API).
    - *Connections:* Input: `Daily Margin Sweep` | Output: `Collapse For Recipe Lookup`.
    - *Edge Cases / Failure Types:* Empty sheets returning null arrays.
  - **Collapse For Recipe Lookup**
    - *Type & Role:* Code node (`n8n-nodes-base.code`) functioning as a structural checkpoint.
    - *Configuration:* JavaScript execution block returning a static collapsed state object.
    - *Key Expressions:* `return [{ json: { _collapsed: true } }];`
    - *Connections:* Input: `Read Prices` | Output: `Read Recipes`.
    - *Edge Cases / Failure Types:* None (purely administrative flow control node).
  - **Read Recipes**
    - *Type & Role:* Google Sheets node (`n8n-nodes-base.googleSheets`) pulling structural recipe compositions.
    - *Configuration:* Reads all rows from the "Recipes" worksheet.
    - *Credentials:* `Feedback Analyzer Sheets` (Google Sheets OAuth2 API).
    - *Connections:* Input: `Collapse For Recipe Lookup` | Output: `Collapse For Menu Lookup`.
    - *Edge Cases / Failure Types:* Missing recipe relations or typographical errors in ingredient naming conventions.
  - **Collapse For Menu Lookup**
    - *Type & Role:* Code node (`n8n-nodes-base.code`) serving as a flow separator.
    - *Configuration:* JavaScript execution returning a static token object.
    - *Key Expressions:* `return [{ json: { _collapsed: true } }];`
    - *Connections:* Input: `Read Recipes` | Output: `Read Menu`.
    - *Edge Cases / Failure Types:* None.
  - **Read Menu**
    - *Type & Role:* Google Sheets node (`n8n-nodes-base.googleSheets`) retrieving retail menu configurations.
    - *Configuration:* Reads all rows from the "Menu" worksheet.
    - *Credentials:* `Feedback Analyzer Sheets` (Google Sheets OAuth2 API).
    - *Connections:* Input: `Collapse For Menu Lookup` | Output: `Compute Recipe Costs`.
    - *Edge Cases / Failure Types:* Missing sale prices or target percentages causing parsing NaN values.
  - **Compute Recipe Costs**
    - *Type & Role:* Code node (`n8n-nodes-base.code`) performing heavy data aggregation and margin math.
    - *Configuration:* JavaScript code parsing data dependencies across prior sheet reads (`Read Prices`, `Read Recipes`, `Read Menu`), building a temporal price lookup map, calculating portion expenses, and evaluating variance percentages.
    - *Key Expressions:* Custom iterative reduction scripts matching ingredient strings against most recent `price_date` values, computing totals, and establishing boolean flags.
    - *Connections:* Input: `Read Menu` | Output: `If Margin Eroded`.
    - *Edge Cases / Failure Types:* Unformatted numerical price inputs throwing parsing errors; missing recipe entries resulting in silent filtration.
  - **If Margin Eroded**
    - *Type & Role:* If node (`n8n-nodes-base.if`) checking evaluation flags.
    - *Configuration:* Evaluates whether the computed `flag` property equals the boolean string `true`.
    - *Key Expressions:* `={{ String($json.flag) }}` evaluated against `true`.
    - *Connections:* Input: `Compute Recipe Costs` | Output (True): `Log Cost Alerts` | Output (False): None.
    - *Edge Cases / Failure Types:* Type mismatch issues if boolean values evaluate as strings.
  - **Log Cost Alerts**
    - *Type & Role:* Google Sheets node (`n8n-nodes-base.googleSheets`) logging regulatory warnings.
    - *Configuration:* Appends rows containing flagged recipe data into the "CostAlerts" worksheet.
    - *Key Expressions:* Maps `recipe`, `status` (hardcoded to `flagged`), `cost_pct`, `alert_date`, `target_cost_pct`, and `cost_per_portion`.
    - *Credentials:* `Feedback Analyzer Sheets` (Google Sheets OAuth2 API).
    - *Connections:* Input: `If Margin Eroded` (True) | Output: `Build Margin Alert`.
    - *Edge Cases / Failure Types:* Sheet lock contention or API timeout errors.
  - **Build Margin Alert**
    - *Type & Role:* Code node (`n8n-nodes-base.code`) constructing notification templates.
    - *Configuration:* Iterates over flagged items to generate structured email subjects and structured multi-line text bodies.
    - *Key Expressions:* JavaScript array mapping functions computing percentages and assembling contextual warnings.
    - *Connections:* Input: `Log Cost Alerts` | Output: `Email Margin Alert`.
    - *Edge Cases / Failure Types:* Empty array handling if upstream conditions pass false positives.
  - **Email Margin Alert**
    - *Type & Role:* Gmail node (`n8n-nodes-base.gmail`) dispatching management warnings.
    - *Configuration:* Sends plain-text margin alert emails to designated recipients.
    - *Key Expressions:* Subject mapped to `={{ $json.alert_subject }}` and message body to `={{ $json.alert_body }}`.
    - *Credentials:* `Gmail Fresh Sep05` (Gmail OAuth2).
    - *Connections:* Input: `Build Margin Alert` | Output: None.
    - *Edge Cases / Failure Types:* Invalid static recipient address (`user@example.com` placeholder requires manual substitution).

---

#### Block 1.3: Weekly Menu Digest

- **Overview:** Triggers every Monday at 07:00 to assemble weekly metrics, pass JSON data to an AI model agent, and email a tailored executive summary containing cost analysis and corrective pricing recommendations.
- **Nodes Involved:**
  - `Weekly Menu Digest`
  - `Read Prices Weekly`
  - `Collapse For Recipe Lookup Weekly`
  - `Read Recipes Weekly`
  - `Collapse For Menu Lookup Weekly`
  - `Read Menu Weekly`
  - `Compute Week Stats`
  - `AI Write Digest`
  - `Weekly Digest Model`
  - `Email Weekly Digest`
- **Node Details:**
  - **Weekly Menu Digest**
    - *Type & Role:* Schedule Trigger node (`n8n-nodes-base.scheduleTrigger`) managing weekly execution timing.
    - *Configuration:* Cron expression set to run every Monday at 07:00 (`0 7 * * 1`).
    - *Connections:* Input: None (Trigger) | Output: `Read Prices Weekly`.
    - *Edge Cases / Failure Types:* Server time zone misalignments.
  - **Read Prices Weekly**
    - *Type & Role:* Google Sheets node (`n8n-nodes-base.googleSheets`) fetching pricing datasets.
    - *Configuration:* Reads all rows from the "Prices" worksheet.
    - *Credentials:* `Feedback Analyzer Sheets` (Google Sheets OAuth2 API).
    - *Connections:* Input: `Weekly Menu Digest` | Output: `Collapse For Recipe Lookup Weekly`.
    - *Edge Cases / Failure Types:* Empty sheets returning null datasets.
  - **Collapse For Recipe Lookup Weekly**
    - *Type & Role:* Code node (`n8n-nodes-base.code`) acting as a structural divider.
    - *Configuration:* JavaScript execution block returning static collapsed parameters.
    - *Connections:* Input: `Read Prices Weekly` | Output: `Read Recipes Weekly`.
    - *Edge Cases / Failure Types:* None.
  - **Read Recipes Weekly**
    - *Type & Role:* Google Sheets node (`n8n-nodes-base.googleSheets`) fetching recipe definitions.
    - *Configuration:* Reads all rows from the "Recipes" worksheet.
    - *Credentials:* `Feedback Analyzer Sheets` (Google Sheets OAuth2 API).
    - *Connections:* Input: `Collapse For Recipe Lookup Weekly` | Output: `Collapse For Menu Lookup Weekly`.
    - *Edge Cases / Failure Types:* Unmatched relational keys.
  - **Collapse For Menu Lookup Weekly**
    - *Type & Role:* Code node (`n8n-nodes-base.code`) managing data flow sequencing.
    - *Configuration:* JavaScript node returning static tracking properties.
    - *Connections:* Input: `Read Recipes Weekly` | Output: `Read Menu Weekly`.
    - *Edge Cases / Failure Types:* None.
  - **Read Menu Weekly**
    - *Type & Role:* Google Sheets node (`n8n-nodes-base.googleSheets`) reading menu pricing profiles.
    - *Configuration:* Reads all rows from the "Menu" worksheet.
    - *Credentials:* `Feedback Analyzer Sheets` (Google Sheets OAuth2 API).
    - *Connections:* Input: `Collapse For Menu Lookup Weekly` | Output: `Compute Week Stats`.
    - *Edge Cases / Failure Types:* Missing or malformed data attributes.
  - **Compute Week Stats**
    - *Type & Role:* Code node (`n8n-nodes-base.code`) aggregating weekly statistical aggregates.
    - *Configuration:* Processes price maps, calculates individual recipe costs, isolates worst-performing items, sorts margins, and serializes analytics into a structured JSON string object (`stats_json`).
    - *Key Expressions:* Comprehensive JavaScript block executing custom date mapping, sorting algorithms, and statistical averaging.
    - *Connections:* Input: `Read Menu Weekly` | Output: `AI Write Digest`.
    - *Edge Cases / Failure Types:* Division-by-zero errors when analyzing empty dataset collections.
  - **AI Write Digest**
    - *Type & Role:* LangChain Agent node (`@n8n/n8n-nodes-langchain.agent`) powering the AI summarization workflow.
    - *Configuration:* Operates in define prompt mode utilizing a specialized system persona instruction set to draft plain-text executive emails under 180 words.
    - *Key Expressions:* `={{ 'Here are the menu cost stats for the week:\n\n' + $json.stats_json }}`
    - *Connections:* Input: `Compute Week Stats`, `Weekly Digest Model` (AI Model link) | Output: `Email Weekly Digest`.
    - *Edge Cases / Failure Types:* API timeouts, token quota exhaustion, or LLM service outages.
  - **Weekly Digest Model**
    - *Type & Role:* OpenAI Chat Model node (`@n8n/n8n-nodes-langchain.lmChatOpenAi`) providing underlying intelligence.
    - *Configuration:* Configured to use the `gpt-6-luna` model profile.
    - *Credentials:* `jonathan` (OpenAI API).
    - *Connections:* Input: None (AI Sub-node) | Output (AI Language Model): `AI Write Digest`.
    - *Edge Cases / Failure Types:* Invalid API key parameters or model deprecation errors.
  - **Email Weekly Digest**
    - *Type & Role:* Gmail node (`n8n-nodes-base.gmail`) executing final output distribution.
    - *Configuration:* Sends plain-text weekly digest reports.
    - *Key Expressions:* Subject set to `Weekly menu cost digest` and message mapped to `={{ $json.output }}`.
    - *Credentials:* `Gmail Fresh Sep05` (Gmail OAuth2).
    - *Connections:* Input: `AI Write Digest` | Output: None.
    - *Edge Cases / Failure Types:* Placeholder recipient addresses requiring production updates.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation & Guide | None | None | ## track menu food costs and flag margin erosion with AI<br><br>### How it works<br><br>Small restaurants run on thin margins and supplier prices change quietly. This workflow keeps every menu item's cost fresh so a price hike shows up as a warning, not a surprise at month end.<br><br>1. A supplier price change is posted to the intake webhook, or the Prices sheet is updated directly.<br>2. The daily sweep recomputes what each recipe costs to make, using the latest price for every ingredient.<br>3. Any item above its target food cost triggers an alert email with the recipe, the cost, and the percentage.<br>4. Every Monday an AI assistant reviews the week and emails one digest naming the biggest margin problems and suggesting menu price fixes.<br><br>### Setup steps<br><br>- Connect Google Sheets and Gmail.<br>- Open the Menu Cost Tracker spreadsheet and replace the sample data in Prices, Recipes and Menu.<br>- Point the intake webhook at your ordering form or price sheet.<br>- Set the manager email on the email nodes and activate.<br><br>### Customization<br><br>- Change the target cost per menu item in the Menu tab.<br>- Adjust sweep and digest times in the schedule triggers. |
| Price updates | n8n-nodes-base.stickyNote | Section Label | None | None | ## 1. Price updates<br><br>Supplier price changes arrive by webhook. Missing fields are rejected by email. Valid changes are logged to the Prices sheet and confirmed. |
| Daily margin sweep | n8n-nodes-base.stickyNote | Section Label | None | None | ## 2. Daily margin sweep<br><br>Each morning the workflow reads prices, recipes and menu prices, recomputes every recipe cost, and emails the owner when an item passes its target food cost. |
| Weekly menu digest | n8n-nodes-base.stickyNote | Section Label | None | None | ## 3. Weekly menu digest<br><br>Every Monday an AI assistant reviews the week's costs and emails a practical digest with price recommendations. |
| When Price Update Received | n8n-nodes-base.webhook | Trigger Webhook | None | Normalize Price Update | Price updates |
| Normalize Price Update | n8n-nodes-base.set | Data Normalization | When Price Update Received | Validate Price Update | Price updates |
| Validate Price Update | n8n-nodes-base.if | Conditional Routing | Normalize Price Update | Log New Price, Email Reject Notice | Price updates |
| Log New Price | n8n-nodes-base.googleSheets | Append Price Data | Validate Price Update | Email Confirmation | Price updates |
| Email Confirmation | n8n-nodes-base.gmail | Transactional Notice | Log New Price | None | Price updates |
| Email Reject Notice | n8n-nodes-base.gmail | Error Notification | Validate Price Update | None | Price updates |
| Daily Margin Sweep | n8n-nodes-base.scheduleTrigger | Cron Schedule Trigger | None | Read Prices | Daily margin sweep |
| Read Prices | n8n-nodes-base.googleSheets | Read Prices Sheet | Daily Margin Sweep | Collapse For Recipe Lookup | Daily margin sweep |
| Collapse For Recipe Lookup | n8n-nodes-base.code | Flow Control Checkpoint | Read Prices | Read Recipes | Daily margin sweep |
| Read Recipes | n8n-nodes-base.googleSheets | Read Recipes Sheet | Collapse For Recipe Lookup | Collapse For Menu Lookup | Daily margin sweep |
| Collapse For Menu Lookup | n8n-nodes-base.code | Flow Control Checkpoint | Read Recipes | Read Menu | Daily margin sweep |
| Read Menu | n8n-nodes-base.googleSheets | Read Menu Sheet | Collapse For Menu Lookup | Compute Recipe Costs | Daily margin sweep |
| Compute Recipe Costs | n8n-nodes-base.code | Financial Math Engine | Read Menu | If Margin Eroded | Daily margin sweep |
| If Margin Eroded | n8n-nodes-base.if | Evaluation Check | Compute Recipe Costs | Log Cost Alerts | Daily margin sweep |
| Log Cost Alerts | n8n-nodes-base.googleSheets | Append Alerts Sheet | If Margin Eroded | Build Margin Alert | Daily margin sweep |
| Build Margin Alert | n8n-nodes-base.code | Template Assembly | Log Cost Alerts | Email Margin Alert | Daily margin sweep |
| Email Margin Alert | n8n-nodes-base.gmail | Management Warning | Build Margin Alert | None | Daily margin sweep |
| Weekly Menu Digest | n8n-nodes-base.scheduleTrigger | Weekly Schedule Trigger | None | Read Prices Weekly | Weekly menu digest |
| Read Prices Weekly | n8n-nodes-base.googleSheets | Read Prices Sheet | Weekly Menu Digest | Collapse For Recipe Lookup Weekly | Weekly menu digest |
| Collapse For Recipe Lookup Weekly | n8n-nodes-base.code | Flow Control Checkpoint | Read Prices Weekly | Read Recipes Weekly | Weekly menu digest |
| Read Recipes Weekly | n8n-nodes-base.googleSheets | Read Recipes Sheet | Collapse For Recipe Lookup Weekly | Collapse For Menu Lookup Weekly | Weekly menu digest |
| Collapse For Menu Lookup Weekly | n8n-nodes-base.code | Flow Control Checkpoint | Read Recipes Weekly | Read Menu Weekly | Weekly menu digest |
| Read Menu Weekly | n8n-nodes-base.googleSheets | Read Menu Sheet | Collapse For Menu Lookup Weekly | Compute Week Stats | Weekly menu digest |
| Compute Week Stats | n8n-nodes-base.code | Statistics Aggregator | Read Menu Weekly | AI Write Digest | Weekly menu digest |
| AI Write Digest | @n8n/n8n-nodes-langchain.agent | LLM Agent Processor | Compute Week Stats, Weekly Digest Model | Email Weekly Digest | Weekly menu digest |
| Weekly Digest Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | OpenAI LLM Model | None | AI Write Digest | Weekly menu digest |
| Email Weekly Digest | n8n-nodes-base.gmail | Executive Email Dispatch | AI Write Digest | None | Weekly menu digest |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Target Spreadsheet:** Set up a Google Sheets document titled `Menu Cost Tracker 0926` containing four distinct tabs: `Prices`, `Recipes`, `Menu`, and `CostAlerts`.
2. **Define Sheet Headers:**
   - *Prices:* `ingredient`, `unit_price`, `unit`, `price_date`, `supplier`, `submitted_by`, `update_id`
   - *Recipes:* `recipe`, `ingredient`, `qty_per_portion`
   - *Menu:* `recipe`, `sale_price`, `target_cost_pct`
   - *CostAlerts:* `alert_date`, `recipe`, `cost_per_portion`, `cost_pct`, `target_cost_pct`, `status`
3. **Configure Credentials:** Set up OAuth2 credentials for Google Sheets (`Feedback Analyzer Sheets`), Gmail (`Gmail Fresh Sep05`), and OpenAI (`jonathan`).
4. **Build Block 1.1 (Price Intake & Validation):**
   - Place a **Webhook** node (`When Price Update Received`), set method to `POST`, and path to `menu-cost-intake-0926`.
   - Add a **Set** node (`Normalize Price Update`) to assign dynamic values for `update_id`, `update_received_at`, `ingredient`, `unit_price`, `unit`, `supplier`, `submitted_by`, and `owner_email`.
   - Add an **If** node (`Validate Price Update`) to ensure `ingredient` and `unit_price` are not empty.
   - Connect the True branch to a **Google Sheets** node (`Log New Price`) operating in `append` mode, mapping fields to the `Prices` tab.
   - Connect `Log New Price` to a **Gmail** node (`Email Confirmation`) configured to send confirmation text to the owner.
   - Connect the False branch of `Validate Price Update` to a **Gmail** node (`Email Reject Notice`) sending failure feedback.
5. **Build Block 1.2 (Daily Margin Sweep):**
   - Place a **Schedule Trigger** node (`Daily Margin Sweep`) set to a cron expression of `0 6 * * *`.
   - Connect it to a **Google Sheets** node (`Read Prices`) reading the `Prices` tab.
   - Insert a **Code** node (`Collapse For Recipe Lookup`) returning `{ _collapsed: true }`.
   - Chain to a **Google Sheets** node (`Read Recipes`) reading the `Recipes` tab, followed by another **Code** node (`Collapse For Menu Lookup`).
   - Chain to a **Google Sheets** node (`Read Menu`) reading the `Menu` tab.
   - Add a **Code** node (`Compute Recipe Costs`) executing JavaScript that aggregates prices, evaluates recipes against menu pricing, and calculates food cost percentages.
   - Add an **If** node (`If Margin Eroded`) checking if `flag === true`.
   - Connect the True branch to a **Google Sheets** node (`Log Cost Alerts`) appending records to the `CostAlerts` sheet.
   - Connect to a **Code** node (`Build Margin Alert`) to format notification text, followed by a **Gmail** node (`Email Margin Alert`) sending the alert.
6. **Build Block 1.3 (Weekly Menu Digest):**
   - Place a **Schedule Trigger** node (`Weekly Menu Digest`) set to cron `0 7 * * 1`.
   - Chain sequential **Google Sheets** and **Code** nodes mirroring the daily sweep structure (`Read Prices Weekly`, `Collapse For Recipe Lookup Weekly`, `Read Recipes Weekly`, `Collapse For Menu Lookup Weekly`, `Read Menu Weekly`).
   - Add a **Code** node (`Compute Week Stats`) compiling weekly aggregated metrics into a JSON string (`stats_json`).
   - Add an **AI Agent** node (`AI Write Digest`) paired with an **OpenAI Chat Model** node (`Weekly Digest Model`) set to model `gpt-6-luna`, passing system instructions to write an email summary under 180 words.
   - Connect the agent output to a **Gmail** node (`Email Weekly Digest`) to send the final weekly report.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| n8n Platform Creator Referral Link | [n8n Partner Link](https://n8n.partnerlinks.io/creator-khmuhtadin) |
| Professional Business Consultation Services | [Consultation Booking](https://khmuhtadin.com/consultation/) |