Track and interpret whale wallet movements with Airtable, OpenAI, and Slack

https://n8nworkflows.xyz/workflows/track-and-interpret-whale-wallet-movements-with-airtable--openai--and-slack-20516


# Track and interpret whale wallet movements with Airtable, OpenAI, and Slack

### 1. Workflow Overview

This workflow automates the monitoring, classification, analysis, and alerting of high-value cryptocurrency whale wallet movements. Running on a scheduled interval, it ingests recent blockchain data, enriches it with reference datasets from Airtable, processes financial metrics, leverages OpenAI for behavioral market interpretation, prevents duplicate processing, and dispatches multi-channel alerts.

The operational logic is organized into seven functional blocks:
- **1.1 Input Reception & Normalization:** Scheduled triggers and HTTP retrieval of raw blockchain ledgers, standardized into unified transaction objects.
- **1.2 Reference Data Loading:** Batch retrieval of known whale wallets and exchange addresses from Airtable.
- **1.3 Movement Classification & Valuation:** Cross-referencing transaction participants against registries to categorize flows (e.g., deposits, withdrawals) and calculating fiat equivalent values.
- **1.4 Threshold & Duplicate Validation:** Filtering transactions against a $100,000 USD limit and querying Airtable to eliminate previously recorded events.
- **1.5 AI Behavioral Interpretation:** Utilizing OpenAI models to generate concise, risk-managed market context and confidence ratings.
- **1.6 Event Logging:** Persisting verified whale movement events into an Airtable historical database.
- **1.7 Notification:** Formatting and dispatching structural alerts to Slack.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Normalization
- **Overview:** This block triggers the processing pipeline on a fixed schedule, requests raw transaction lists from an external blockchain API, and normalizes the payload into clean, iterative execution objects.
- **Nodes Involved:** 
  - `Schedule Whale Monitoring`
  - `Fetch Recent Blockchain Transactions`
  - `Normalize Transaction Data`
- **Node Details:**
  - **Schedule Whale Monitoring**
    - *Type & Role:* `n8n-nodes-base.scheduleTrigger` (v1.2) — Acts as the primary temporal entry point.
    - *Configuration:* Interval set to fire every 15 minutes.
    - *Connections:* Input: None; Output: `Fetch Recent Blockchain Transactions`.
    - *Edge Cases:* Timezone shifts or platform uptime interruptions can delay execution windows.
  - **Fetch Recent Blockchain Transactions**
    - *Type & Role:* `n8n-nodes-base.httpRequest` (v4.2) — Performs an outbound HTTP GET request to pull recent transactions.
    - *Configuration:* Targets the dummy blockchain endpoint URL (`http://192.168.101.63:5678/webhook-test/dummy-blockchain/transactions`).
    - *Connections:* Input: `Schedule Whale Monitoring`; Output: `Normalize Transaction Data`.
    - *Edge Cases:* HTTP 5xx errors, network timeouts, or unreachable endpoints cause the workflow branch to halt.
  - **Normalize Transaction Data**
    - *Type & Role:* `n8n-nodes-base.code` (v2) — JavaScript execution node for payload restructuring.
    - *Configuration:* Extracts the root `transactions` array, normalizes missing fields, forces numeric types for amounts and prices, and sets default fallback statuses.
    - *Connections:* Input: `Fetch Recent Blockchain Transactions`; Output: `Load Tracked Whale Wallets`.
    - *Edge Cases:* Malformed JSON arrays or missing properties throw JavaScript runtime exceptions.

#### 1.2 Reference Data Loading
- **Overview:** Queries Airtable databases to retrieve active registry records containing known whale wallet addresses and exchange endpoint associations.
- **Nodes Involved:**
  - `Load Tracked Whale Wallets`
  - `Load Exchange Wallet Registry`
- **Node Details:**
  - **Load Tracked Whale Wallets**
    - *Type & Role:* `n8n-nodes-base.airtable` (v2.1) — Fetches rows from the "Tracked Whales" table.
    - *Configuration:* Uses Airtable base ID `appvyDg5xDLQjO5CF` and table `tblAx9s9fP3GPa67B`. Configured with `executeOnce: true`.
    - *Connections:* Input: `Normalize Transaction Data`; Output: `Load Exchange Wallet Registry`.
    - *Edge Cases:* Airtable API rate limits (HTTP 429) or invalid authentication keys.
  - **Load Exchange Wallet Registry**
    - *Type & Role:* `n8n-nodes-base.airtable` (v2.1) — Fetches rows from the "Exchange Addresses" table.
    - *Configuration:* Uses Airtable base ID `appvyDg5xDLQjO5CF` and table `tbl1bcnFSL7sNrMOb`. Configured with `executeOnce: true`.
    - *Connections:* Input: `Load Tracked Whale Wallets`; Output: `Classify Whale Movement`.
    - *Edge Cases:* Missing table names or missing credential grants.

#### 1.3 Movement Classification & Valuation
- **Overview:** Compares transaction senders and receivers against active Airtable lists to categorize the economic flow type, subsequently computing total USD valuation.
- **Nodes Involved:**
  - `Classify Whale Movement`
  - `Calculate Transaction Value`
- **Node Details:**
  - **Classify Whale Movement**
    - *Type & Role:* `n8n-nodes-base.code` (v2) — Cross-references multi-node item lists using memory maps (`Map()`).
    - *Configuration:* Iterates over normalized transactions, evaluating addresses against whale and exchange maps. Assigns movement categories (`Exchange Deposit`, `Exchange Withdrawal`, `Wallet-to-Wallet`).
    - *Connections:* Input: `Load Exchange Wallet Registry`; Output: `Calculate Transaction Value`.
    - *Edge Cases:* Unrecognized internal address strings result in filtered-out iterations (`continue` statement drops unmatched non-whale payloads).
  - **Calculate Transaction Value**
    - *Type & Role:* `n8n-nodes-base.code` (v2) — Evaluates financial metrics.
    - *Configuration:* Multiplies `amount` by `price_usd`, sets a hardcoded valuation threshold of $100,000, and generates an initial rule-based interpretation string.
    - *Connections:* Input: `Classify Whale Movement`; Output: `Check Alert Threshold`.
    - *Edge Cases:* Non-numeric amount parameters evaluate to NaN values.

#### 1.4 Threshold & Duplicate Validation
- **Overview:** Evaluates the transaction's financial scale against the minimum alerting threshold and cross-references historical Airtable logs to block duplicate event dispatching.
- **Nodes Involved:**
  - `Check Alert Threshold`
  - `Check Duplicate Event`
  - `Confirm New Whale Event`
- **Node Details:**
  - **Check Alert Threshold**
    - *Type & Role:* `n8n-nodes-base.if` (v2.2) — Conditional router.
    - *Configuration:* Evaluates if `usd_value` is greater than or equal to `100000`.
    - *Connections:* Input: `Calculate Transaction Value`; Output: `Check Duplicate Event`.
    - *Edge Cases:* Expression evaluation failures if `usd_value` is unbound.
  - **Check Duplicate Event**
    - *Type & Role:* `n8n-nodes-base.airtable` (v2.1) — Searches historical log entries.
    - *Configuration:* Queries base `appvyDg5xDLQjO5CF`, table `Whale Movement Events`, applying formula: `={event_id}='{{$json.transaction_hash}}'`. Configured with `alwaysOutputData: true`.
    - *Connections:* Input: `Check Alert Threshold`; Output: `Confirm New Whale Event`.
    - *Edge Cases:* Identifier mismatches between the lookup formula (`transaction_hash`) and the logged schema (`sender-timestamp`).
  - **Confirm New Whale Event**
    - *Type & Role:* `n8n-nodes-base.if` (v2.2) — Validates uniqueness.
    - *Configuration:* Checks if the Airtable lookup record ID (`$json.id`) is empty (confirming no pre-existing log exists).
    - *Connections:* Input: `Check Duplicate Event`; Output: `Generate Market Interpretation`.
    - *Edge Cases:* Failing to catch duplicates if query formulas return partial array structures.

#### 1.5 AI Behavioral Interpretation
- **Overview:** Sends qualifying transaction payloads to an OpenAI LLM to generate cautious, risk-managed market context and extracts normalized confidence values.
- **Nodes Involved:**
  - `Generate Market Interpretation`
  - `Normalize AI Analysis`
- **Node Details:**
  - **Generate Market Interpretation**
    - *Type & Role:* `@n8n/n8n-nodes-langchain.openAi` (v2.3) — AI execution node.
    - *Configuration:* Model set to `gpt-4.1-nano`. Instructs the model to output strict JSON formatting containing a short market interpretation (under 40 words) and a confidence score (0–100).
    - *Connections:* Input: `Confirm New Whale Event`; Output: `Normalize AI Analysis`.
    - *Credentials:* OpenAI API (`yOfw9W9XwQefVROR`).
    - *Edge Cases:* API rate-limiting, output parsing failures, or LLM hallucination ignoring JSON-only constraints.
  - **Normalize AI Analysis**
    - *Type & Role:* `n8n-nodes-base.code` (v2) — JavaScript data cleanup.
    - *Configuration:* Strips markdown block syntax (` ```json `) from LLM output, safely parses inner JSON strings, and clamps confidence values between 0 and 100.
    - *Connections:* Input: `Generate Market Interpretation`; Output: `Log Whale Movement Event`.
    - *Edge Cases:* Syntax errors thrown during standard `JSON.parse` operations fallback safely to raw text descriptions.

#### 1.6 Event Logging
- **Overview:** Commits the verified whale movement event to an Airtable database table for audit trails and historical analytics.
- **Nodes Involved:**
  - `Log Whale Movement Event`
- **Node Details:**
  - **Log Whale Movement Event**
    - *Type & Role:* `n8n-nodes-base.airtable` (v2.1) — Airtable record creator.
    - *Configuration:* Maps transaction attributes, AI interpretation, and calculation metrics to base `appvyDg5xDLQjO5CF`, table `tblk054Uran6wyDoV`. Note: Sets `event_id` to `={{ $json.sender }}-{{ $json.timestamp }}`.
    - *Connections:* Input: `Normalize AI Analysis`; Output: `Send Whale Alert to Slack`.
    - *Edge Cases:* Schema mismatches, unmapped optional fields, or authentication token expiration.

#### 1.7 Notification
- **Overview:** Formats the completed event payload into a human-readable alert message and pushes it directly to a dedicated Slack communications channel.
- **Nodes Involved:**
  - `Send Whale Alert to Slack`
- **Node Details:**
  - **Send Whale Alert to Slack**
    - *Type & Role:* `n8n-nodes-base.slack` (v2.3) — Messaging node.
    - *Configuration:* Target channel set to `C0B1LNY15GW` ("n8n-workflow-testing"). Constructs a structured multi-line text template populated via expression mappings (`$json.fields.*`).
    - *Connections:* Input: `Log Whale Movement Event`; Output: None (Terminal node).
    - *Credentials:* Slack API.
    - *Edge Cases:* Invalid webhook targets, missing channel scopes, or Slack API connection outages.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Schedule Whale Monitoring` | `scheduleTrigger` | Triggers the workflow execution every 15 minutes. | None | `Fetch Recent Blockchain Transactions` | ## Transaction Monitoring<br>This section starts the whale monitoring process on a scheduled interval and retrieves recent blockchain transactions from the internal dummy API. The normalization step converts the returned transaction collection into consistent individual records containing blockchain, asset, wallets, amount, price, timestamp, and transaction status. |
| `Fetch Recent Blockchain Transactions` | `httpRequest` | Fetches raw blockchain transaction payloads from the source endpoint. | `Schedule Whale Monitoring` | `Normalize Transaction Data` | ## Transaction Monitoring<br>This section starts the whale monitoring process on a scheduled interval and retrieves recent blockchain transactions from the internal dummy API. The normalization step converts the returned transaction collection into consistent individual records containing blockchain, asset, wallets, amount, price, timestamp, and transaction status. |
| `Normalize Transaction Data` | `code` | Standardizes raw API arrays into uniform transaction objects. | `Fetch Recent Blockchain Transactions` | `Load Tracked Whale Wallets` | ## Transaction Monitoring<br>This section starts the whale monitoring process on a scheduled interval and retrieves recent blockchain transactions from the internal dummy API. The normalization step converts the returned transaction collection into consistent individual records containing blockchain, asset, wallets, amount, price, timestamp, and transaction status. |
| `Load Tracked Whale Wallets` | `airtable` | Loads active whale wallet records from Airtable. | `Normalize Transaction Data` | `Load Exchange Wallet Registry` | ## Reference Data Loading<br><br>This section loads the reference data required to understand blockchain movements. The first Airtable table contains the curated whale wallet addresses being monitored, while the second contains known exchange wallet addresses. These datasets are used by the classification logic to identify meaningful wallet relationships. |
| `Load Exchange Wallet Registry` | `airtable` | Loads known exchange address records from Airtable. | `Load Tracked Whale Wallets` | `Classify Whale Movement` | ## Reference Data Loading<br><br>This section loads the reference data required to understand blockchain movements. The first Airtable table contains the curated whale wallet addresses being monitored, while the second contains known exchange wallet addresses. These datasets are used by the classification logic to identify meaningful wallet relationships. |
| `Classify Whale Movement` | `code` | Identifies movement types by comparing addresses against registries. | `Load Exchange Wallet Registry` | `Calculate Transaction Value` | ## Movement Classification<br>This section determines the behavioral category of each tracked whale transaction by comparing sender and receiver addresses against the Airtable reference lists. It identifies exchange deposits, exchange withdrawals, and wallet-to-wallet movements, then calculates the transaction's USD value and establishes the alert threshold and initial interpretation. |
| `Calculate Transaction Value` | `code` | Computes USD equivalents and flags threshold metrics. | `Classify Whale Movement` | `Check Alert Threshold` | ## Movement Classification<br>This section determines the behavioral category of each tracked whale transaction by comparing sender and receiver addresses against the Airtable reference lists. It identifies exchange deposits, exchange withdrawals, and wallet-to-wallet movements, then calculates the transaction's USD value and establishes the alert threshold and initial interpretation. |
| `Check Alert Threshold` | `if` | Filters out transactions worth less than $100,000. | `Calculate Transaction Value` | `Check Duplicate Event` | ## Threshold and Duplicate Validation<br>This section prevents unnecessary processing and duplicate alerts. Transactions below the configured $100,000 threshold are stopped, while qualifying transactions are checked against previously logged events in Airtable. Only transactions that meet the threshold and are not already recorded continue to AI interpretation and notification. |
| `Check Duplicate Event` | `airtable` | Searches Airtable logs to spot previously recorded event entries. | `Check Alert Threshold` | `Confirm New Whale Event` | ## Threshold and Duplicate Validation<br>This section prevents unnecessary processing and duplicate alerts. Transactions below the configured $100,000 threshold are stopped, while qualifying transactions are checked against previously logged events in Airtable. Only transactions that meet the threshold and are not already recorded continue to AI interpretation and notification. |
| `Confirm New Whale Event` | `if` | Verifies the transaction does not exist in the log history. | `Check Duplicate Event` | `Generate Market Interpretation` | ## Threshold and Duplicate Validation<br>This section prevents unnecessary processing and duplicate alerts. Transactions below the configured $100,000 threshold are stopped, while qualifying transactions are checked against previously logged events in Airtable. Only transactions that meet the threshold and are not already recorded continue to AI interpretation and notification. |
| `Generate Market Interpretation` | `openAi` | Prompts OpenAI to analyze transaction context and provide risk-managed interpretations. | `Confirm New Whale Event` | `Normalize AI Analysis` | ## AI Behavioral Interpretation<br><br>This section adds contextual analysis to the rule-based classification. The OpenAI node evaluates the transaction details and known movement classification using cautious market language. The following Code node normalizes the AI response into a consistent interpretation and confidence value before the result is stored and communicated. |
| `Normalize AI Analysis` | `code` | Parses and cleans LLM outputs into structured variables. | `Generate Market Interpretation` | `Log Whale Movement Event` | ## AI Behavioral Interpretation<br><br>This section adds contextual analysis to the rule-based classification. The OpenAI node evaluates the transaction details and known movement classification using cautious market language. The following Code node normalizes the AI response into a consistent interpretation and confidence value before the result is stored and communicated. |
| `Log Whale Movement Event` | `airtable` | Commits final processed analytics into the historical Airtable events table. | `Normalize AI Analysis` | `Send Whale Alert to Slack` | ## Event Logging and Alerting<br>This section permanently records qualifying whale movements in Airtable and sends the resulting alert to the configured Slack channel. The alert includes the blockchain, asset, movement type, whale wallet, counterparty, transaction value, transaction hash, AI interpretation, confidence, and configured alert threshold. |
| `Send Whale Alert to Slack` | `slack` | Formats and dispatches structured event alerts to a Slack channel. | `Log Whale Movement Event` | None | ## Event Logging and Alerting<br>This section permanently records qualifying whale movements in Airtable and sends the resulting alert to the configured Slack channel. The alert includes the blockchain, asset, movement type, whale wallet, counterparty, transaction value, transaction hash, AI interpretation, confidence, and configured alert threshold. |

---

### 4. Reproducing the Workflow from Scratch

Follow this step-by-step procedure to reconstruct the workflow inside n8n:

1. **Create the Trigger Node:**
   - Add a **Schedule Trigger** node. Set interval rules to fire every `15` minutes. Name it `Schedule Whale Monitoring`.
2. **Fetch Transactions:**
   - Create an **HTTP Request** node. Set the method to `GET` and URL to your transaction provider endpoint (e.g., `http://192.168.101.63:5678/webhook-test/dummy-blockchain/transactions`). Connect `Schedule Whale Monitoring` to this node. Name it `Fetch Recent Blockchain Transactions`.
3. **Normalize Data:**
   - Add a **Code** node. Set the JavaScript execution code to extract the `transactions` array, enforce explicit numeric casting on amounts and prices, and supply fallbacks for string elements. Connect `Fetch Recent Blockchain Transactions` here. Name it `Normalize Transaction Data`.
4. **Load Reference Registries (Airtable):**
   - Add an **Airtable** node configured to *Search* records. Select your target Base (`appvyDg5xDLQjO5CF`) and Table (`Tracked Whales`). Enable `Execute Once`. Connect `Normalize Transaction Data` to this node. Name it `Load Tracked Whale Wallets`.
   - Add a second **Airtable** node configured to *Search* records. Use the same Base and select the Table (`Exchange Addresses`). Enable `Execute Once`. Connect `Load Tracked Whale Wallets` to this node. Name it `Load Exchange Wallet Registry`.
5. **Classify Movements:**
   - Add a **Code** node. Write mapping loops using `Map()` objects to match transaction senders and receivers against active whale and exchange entries. Categorize movements as `Exchange Deposit`, `Exchange Withdrawal`, or `Wallet-to-Wallet`. Connect `Load Exchange Wallet Registry` here. Name it `Classify Whale Movement`.
6. **Calculate Financial Scale:**
   - Add a **Code** node to compute `usd_value` by multiplying `amount` by `price_usd`. Establish default descriptive interpretations matching specific flow categories and pass an explicit boolean flag (`alert_eligible`) for values $\ge \$100,000$. Connect `Classify Whale Movement` here. Name it `Calculate Transaction Value`.
7. **Filter Thresholds:**
   - Add an **If** node. Configure a numeric rule comparing `{{ $json.usd_value }}` with `100000` (`is greater than or equal to`). Connect `Calculate Transaction Value` to this node. Name it `Check Alert Threshold`.
8. **Deduplicate Events:**
   - Add an **Airtable** node configured to *Search* records. Connect the `true` branch of `Check Alert Threshold`. Select your base, use the `Whale Movement Events` table, set `Always Output Data` to true, and add a formula filter: `={event_id}='{{$json.transaction_hash}}'`. Name it `Check Duplicate Event`.
   - Add an **If** node to check if the returned Airtable record ID is empty (`{{ $json.id }}` equals empty string), validating that the transaction has not been logged previously. Connect `Check Duplicate Event` here. Name it `Confirm New Whale Event`.
9. **AI Integration:**
   - Add an **OpenAI** node (LangChain model connector). Set the model to `gpt-4.1-nano`. Provide system and user prompt strings enforcing JSON-only output constraints for market interpretation and confidence percentages (0–100). Connect the `true` branch of `Confirm New Whale Event` to this node. Configure your OpenAI API credentials. Name it `Generate Market Interpretation`.
   - Add a **Code** node to clean markdown wrappers from the LLM string output, parse the underlying JSON securely, and handle syntax fallbacks safely. Connect `Generate Market Interpretation` here. Name it `Normalize AI Analysis`.
10. **Persistence & Notification:**
    - Add an **Airtable** node configured to *Create* records. Connect `Normalize AI Analysis` here. Map the schema fields (such as `asset`, `amount`, `usd_value`, `confidence`, `ai_interpretation`, `transaction_hash`, etc.). *Note on duplication fix:* Ensure the unique identifier mapping strategy uses transaction hashes consistently if required. Name it `Log Whale Movement Event`.
    - Add a **Slack** node configured to send messages to a selected channel (`C0B1LNY15GW`). Write a structured multi-line text template referencing properties from the preceding Airtable output (`$json.fields.*`). Connect `Log Whale Movement Event` here. Configure valid Slack API credentials. Name it `Send Whale Alert to Slack`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| n8n Workflow Setup and Customization Services | [WeblineIndia n8n Automation Services](https://www.weblineindia.com/n8n-automation/) |
| Professional Consultation & Development Inquiries | [Contact WeblineIndia](https://www.weblineindia.com/contact-us.html) |