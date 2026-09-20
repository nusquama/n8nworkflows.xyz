Review cannabis orders for compliance with GPT-5 Mini, Gmail and Google Sheets

https://n8nworkflows.xyz/workflows/review-cannabis-orders-for-compliance-with-gpt-5-mini--gmail-and-google-sheets-19038


# Review cannabis orders for compliance with GPT-5 Mini, Gmail and Google Sheets

### 1. Workflow Overview

This workflow automates regulatory compliance checks for completed cannabis dispensary orders. It receives order payloads from a Point-of-Sale (POS) system via webhook, evaluates them using an AI agent powered by GPT-5 Mini against state possession limits, METRC seed-to-sale tag integrity, and active product recalls, and then routes alerts to the appropriate personnel via Gmail while logging an immutable audit record in Google Sheets.

The workflow logic is categorized into the following functional blocks:
- **1.1 Input Reception & Normalization:** Captures incoming webhooks and normalizes raw payload properties into clean, standardized data fields.
- **1.2 AI Compliance Audit:** Utilizes an AI Agent configured with a detailed regulatory system message, supported by an OpenAI language model and a structured output parser, to evaluate the order.
- **1.3 Decision Shaping & Routing:** Consolidates the AI output and order metadata into a unified format, then branches execution based on the compliance decision category using a conditional switch node.
- **1.4 Notification & Audit Logging:** Dispatches targeted email alerts via Gmail depending on the violation type, merges the output branches, and appends a structured record into a Google Sheets compliance ledger.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Input Reception & Normalization
**Overview:** This block acts as the entry point for completed dispensary orders, capturing the raw JSON data and normalizing fields to ensure downstream consistency regardless of payload variations.

**Nodes Involved:**
- Receive Dispensary Order Webhook
- Normalize Order Intake

**Node Details:**
- **Receive Dispensary Order Webhook**
  - *Type & Technical Role:* `n8n-nodes-base.webhook` — Acts as an HTTP POST endpoint listening for external dispensary order submissions.
  - *Configuration:* Listens on path `dispensary-order-review` for HTTP POST requests.
  - *Key Expressions/Variables:* None.
  - *Connections:* Input: None (Trigger); Output: `Normalize Order Intake`.
  - *Version-specific Requirements:* Version 2.1.
  - *Edge Cases / Potential Failures:* Network timeouts, malformed JSON bodies, or unauthorized IP calls to the webhook URL.
  - *Sub-workflow Reference:* None.
  - *Sticky Note:* `📥 Order Ingestion` (Captures completed order data from the POS system before fulfillment. Payload: `orderId`, `customerType`, `items`, `thcWeight`, `metrcTags`).

- **Normalize Order Intake**
  - *Type & Technical Role:* `n8n-nodes-base.set` — Transforms and standardizes incoming data structures with fallback default values.
  - *Configuration:* Assigns properties using fallback logic (e.g., checking both root and nested `body` objects).
  - *Key Expressions/Variables:* 
    - `orderId`: `={{ $json.body?.orderId ?? $json.orderId ?? "" }}`
    - `dispensaryId`: `={{ $json.body?.dispensaryId ?? $json.dispensaryId ?? "" }}`
    - `customerType`: `={{ $json.body?.customerType ?? $json.customerType ?? "ADULT_USE" }}`
    - `lineItems`: `={{ $json.body?.lineItems ?? $json.lineItems ?? [] }}`
    - `customerDailyPurchasedGramsSoFar`: `={{ $json.body?.customerDailyPurchasedGramsSoFar ?? $json.customerDailyPurchasedGramsSoFar ?? 0 }}`
    - `state`: `={{ $json.body?.state ?? $json.state ?? "" }}`
    - `budtenderId`: `={{ $json.body?.budtenderId ?? $json.budtenderId ?? "" }}`
  - *Connections:* Input: `Receive Dispensary Order Webhook`; Output: `Review Order Compliance`.
  - *Version-specific Requirements:* Version 3.5.
  - *Edge Cases / Potential Failures:* Expression evaluation failures if data types mismatch expected schemas (e.g., arrays passed as strings).
  - *Sub-workflow Reference:* None.
  - *Sticky Note:* `⚙️ Intake Normalization` (Cleanses raw JSON order data, formatting THC weights and METRC batch tags for accurate AI evaluation).

---

#### 1.2 AI Compliance Audit
**Overview:** Evaluates the normalized order details against regulatory limits, METRC tag requirements, customer classifications, and active product recalls using an LLM-backed agent with strict JSON schema enforcement.

**Nodes Involved:**
- Review Order Compliance
- GPT-5 Mini Compliance Model
- Compliance Decision Schema

**Node Details:**
- **Review Order Compliance**
  - *Type & Technical Role:* `@n8n/n8n-nodes-langchain.agent` — Advanced AI Agent that processes the order metadata according to an explicit system policy.
  - *Configuration:* Configured with a system message establishing priority rules (Recall check $\to$ METRC tag verification $\to$ Quantity limit math) and a structured prompt passing order properties and line items.
  - *Key Expressions/Variables:* 
    - `={{ $json.orderId }}`
    - `={{ $json.dispensaryId }}`
    - `={{ $json.state }}`
    - `={{ $json.customerType }}`
    - `={{ $json.budtenderId }}`
    - `={{ $json.customerDailyPurchasedGramsSoFar }}`
    - `={{ JSON.stringify($json.lineItems) }}`
  - *Connections:* Input: `Normalize Order Intake`; Output: `Shape Compliance Decision`. AI Subnode connections link to `GPT-5 Mini Compliance Model` and `Compliance Decision Schema`.
  - *Version-specific Requirements:* Version 3.1.
  - *Edge Cases / Potential Failures:* LLM hallucinations, schema violations, or rate-limit issues on the OpenAI API.
  - *Sub-workflow Reference:* None.
  - *Sticky Note:* `🧠 Compliance Audit Agent` (Audits order against state rules: limits, customer type (Med/Rec), active recalls, and METRC tag validity).

- **GPT-5 Mini Compliance Model**
  - *Type & Technical Role:* `@n8n/n8n-nodes-langchain.lmChatOpenAi` — Provides the foundational language model logic for the AI agent.
  - *Configuration:* Uses model `gpt-5-mini` with a low temperature of `0.1` for deterministic evaluation.
  - *Key Expressions/Variables:* None.
  - *Connections:* Connected as an AI language model provider to `Review Order Compliance`.
  - *Version-specific Requirements:* Version 1.3.
  - *Credentials Required:* OpenAI API (`openAiApi`).
  - *Edge Cases / Potential Failures:* Invalid API keys, depleted API credits, or upstream API outages.
  - *Sub-workflow Reference:* None.
  - *Sticky Note:* `🤖 LLM & Schema Config` (Powers the compliance evaluation via GPT-5 Mini and guarantees structured JSON response parameters).

- **Compliance Decision Schema**
  - *Type & Technical Role:* `@n8n/n8n-nodes-langchain.outputParserStructured` — Enforces a strict JSON output format for the AI agent's final answer.
  - *Configuration:* Defined with a JSON schema example containing `category`, `confidence`, `reasoning`, `gramsRemainingAfterOrder`, and `correctedAction`.
  - *Key Expressions/Variables:* None.
  - *Connections:* Connected as an AI output parser to `Review Order Compliance`.
  - *Version-specific Requirements:* Version 1.3.
  - *Edge Cases / Potential Failures:* Parsing failures if the model outputs text outside the strict schema constraints.
  - *Sub-workflow Reference:* None.
  - *Sticky Note:* `🤖 LLM & Schema Config` (Powers the compliance evaluation via GPT-5 Mini and guarantees structured JSON response parameters).

---

#### 1.3 Decision Shaping & Routing
**Overview:** Aggregates the structured AI decision alongside initial order metadata into a single data object, then routes the flow down dedicated paths according to the compliance category.

**Nodes Involved:**
- Shape Compliance Decision
- Route by Compliance Category

**Node Details:**
- **Shape Compliance Decision**
  - *Type & Technical Role:* `n8n-nodes-base.set` — Restructures and combines data from the initial intake node and the AI output node.
  - *Configuration:* Assigns properties mapping back to the normalized order and AI outputs, adding a timestamp for the review time.
  - *Key Expressions/Variables:* 
    - `orderId`: `={{ $('Normalize Order Intake').item.json.orderId }}`
    - `dispensaryId`: `={{ $('Normalize Order Intake').item.json.dispensaryId }}`
    - `budtenderId`: `={{ $('Normalize Order Intake').item.json.budtenderId }}`
    - `state`: `={{ $('Normalize Order Intake').item.json.state }}`
    - `customerType`: `={{ $('Normalize Order Intake').item.json.customerType }}`
    - `category`: `={{ $json.output.category }}`
    - `confidence`: `={{ $json.output.confidence }}`
    - `reasoning`: `={{ $json.output.reasoning }}`
    - `gramsRemainingAfterOrder`: `={{ $json.output.gramsRemainingAfterOrder }}`
    - `correctedAction`: `={{ $json.output.correctedAction }}`
    - `reviewedAt`: `={{ $now.toISO() }}`
  - *Connections:* Input: `Review Order Compliance`; Output: `Route by Compliance Category`.
  - *Version-specific Requirements:* Version 3.5.
  - *Edge Cases / Potential Failures:* Broken node references using `$()` if upstream node names are altered.
  - *Sub-workflow Reference:* None.
  - *Sticky Note:* `🎯 Decision Formatting` (Extracts structured compliance metrics (`complianceCategory`, `flaggedIssues`, `actionRequired`)).

- **Route by Compliance Category**
  - *Type & Technical Role:* `n8n-nodes-base.switch` — Evaluates the compliance category and directs execution to the corresponding alert path.
  - *Configuration:* Four primary rules matching `CLEAR_TO_FULFILL`, `QUANTITY_LIMIT_BREACH`, `METRC_TAG_MISMATCH`, and `RECALL_FLAG`, with an extra fallback output labeled `Unclassified (treat as Recall)`.
  - *Key Expressions/Variables:* Evaluates `={{ $json.category }}` against literal condition strings.
  - *Connections:* Input: `Shape Compliance Decision`; Outputs: Connects to four distinct Gmail notification nodes (plus fallback directed to the recall notification node).
  - *Version-specific Requirements:* Version 3.4.
  - *Edge Cases / Potential Failures:* Unhandled category values triggering fallback pathways.
  - *Sub-workflow Reference:* None.
  - *Sticky Note:* `🔀 Risk Category Router` (Routes execution path based on audit status: `0`: Clear Order, `1`: Quantity Breach, `2`: METRC Mismatch, `3`: Recall Flag).

---

#### 1.4 Notification & Audit Logging
**Overview:** Sends email notifications to relevant personnel (budtender, store manager, or compliance officer) based on the routing category, merges execution paths, and writes an audit log row to Google Sheets.

**Nodes Involved:**
- Alert Budtender Order Clear to Fulfill
- Alert Manager of Quantity Limit Breach
- Alert Compliance Officer of METRC Mismatch
- Alert Compliance Officer of Recall Flag
- Combine Alert Outcomes
- Append Compliance Review Log

**Node Details:**
- **Alert Budtender Order Clear to Fulfill**
  - *Type & Technical Role:* `n8n-nodes-base.gmail` — Sends an approval email to the budtender or POS station.
  - *Configuration:* Configured with placeholder recipient email, clear-to-fulfill subject, and plain text message detailing order status, remaining limits, and AI reasoning. Error handling set to continue regular output with 3 max retry attempts.
  - *Key Expressions/Variables:* References properties from `Shape Compliance Decision` (e.g., `={{ $('Shape Compliance Decision').item.json.orderId }}`).
  - *Connections:* Input: `Route by Compliance Category` (Branch 0); Output: `Combine Alert Outcomes` (Input Index 0).
  - *Credentials Required:* Gmail OAuth2 (`gmailOAuth2`).
  - *Edge Cases / Potential Failures:* Invalid recipient email address or transient SMTP/Gmail API failures.
  - *Sub-workflow Reference:* None.
  - *Sticky Note:* `✅ Budtender Clearance Alert` (Notifies dispensary staff that the order meets all state guidelines and is cleared for packaging).

- **Alert Manager of Quantity Limit Breach**
  - *Type & Technical Role:* `n8n-nodes-base.gmail` — Sends an escalation email regarding a daily possession limit breach.
  - *Configuration:* Configured with placeholder store manager email, warning subject, and plain text message detailing over-limit metrics. Error handling set to continue regular output with 3 retries.
  - *Key Expressions/Variables:* References properties from `Shape Compliance Decision`.
  - *Connections:* Input: `Route by Compliance Category` (Branch 1); Output: `Combine Alert Outcomes` (Input Index 1).
  - *Credentials Required:* Gmail OAuth2 (`gmailOAuth2`).
  - *Edge Cases / Potential Failures:* Invalid recipient email address or API rate limits.
  - *Sub-workflow Reference:* None.
  - *Sticky Note:* `⚠️ Quantity Limit Alert` (Flags daily state possession limit overages to the store manager for manual review).

- **Alert Compliance Officer of METRC Mismatch**
  - *Type & Technical Role:* `n8n-nodes-base.gmail` — Sends a discrepancy alert to the compliance officer.
  - *Configuration:* Configured with placeholder compliance officer email, mismatch subject, and message detailing tag verification failures. Error handling set to continue regular output with 3 retries.
  - *Key Expressions/Variables:* References properties from `Shape Compliance Decision`.
  - *Connections:* Input: `Route by Compliance Category` (Branch 2); Output: `Combine Alert Outcomes` (Input Index 2).
  - *Credentials Required:* Gmail OAuth2 (`gmailOAuth2`).
  - *Edge Cases / Potential Failures:* Invalid recipient email address or network timeouts.
  - *Sub-workflow Reference:* None.
  - *Sticky Note:* `🔍 METRC Mismatch Alert` (Notifies compliance team of untracked or invalid seed-to-sale tags before order release).

- **Alert Compliance Officer of Recall Flag**
  - *Type & Technical Role:* `n8n-nodes-base.gmail` — Sends an urgent emergency alert to the compliance officer for recalled products.
  - *Configuration:* Configured with placeholder compliance officer email, urgent recall subject, and detailed escalation message. Handles both Branch 3 and the Switch fallback output. Error handling set to continue regular output with 3 retries.
  - *Key Expressions/Variables:* References properties from `Shape Compliance Decision`.
  - *Connections:* Input: `Route by Compliance Category` (Branches 3 & 4); Output: `Combine Alert Outcomes` (Input Index 3).
  - *Credentials Required:* Gmail OAuth2 (`gmailOAuth2`).
  - *Edge Cases / Potential Failures:* Invalid recipient email address or service unavailability.
  - *Sub-workflow Reference:* None.
  - *Sticky Note:* `🚨 Recall Emergency Alert` (Triggers immediate notification to halt product distribution due to an active batch recall).

- **Combine Alert Outcomes**
  - *Type & Technical Role:* `n8n-nodes-base.merge` — Merges multiple incoming notification branches into a single stream.
  - *Configuration:* Configured with 4 number inputs to aggregate data streams from all alert paths.
  - *Key Expressions/Variables:* None.
  - *Connections:* Inputs: All four Gmail alert nodes; Output: `Append Compliance Review Log`.
  - *Version-specific Requirements:* Version 3.2.
  - *Edge Cases / Potential Failures:* Desynchronized branch arrivals if execution timings vary significantly.
  - *Sub-workflow Reference:* None.
  - *Sticky Note:* `🔗 Branch Merger` (Merges execution outputs from all alert branches back into a single unified stream for logging).

- **Append Compliance Review Log**
  - *Type & Technical Role:* `n8n-nodes-base.googleSheets` — Appends a new audit record row into a designated spreadsheet tab.
  - *Configuration:* Uses append operation mapped to specific column schemas (`OrderID`, `DispensaryID`, `BudtenderID`, `State`, `CustomerType`, `ComplianceCategory`, `Confidence`, `Reasoning`, `GramsRemainingAfterOrder`, `CorrectedAction`, `ReviewedAt`). Document ID and sheet name configured via placeholders.
  - *Key Expressions/Variables:* Maps values from `={{ $('Shape Compliance Decision').item.json.<property> }}`.
  - *Connections:* Input: `Combine Alert Outcomes`; Output: None (Terminal node).
  - *Version-specific Requirements:* Version 4.7.
  - *Credentials Required:* Google Sheets OAuth2 API (`googleSheetsOAuth2Api`).
  - *Edge Cases / Potential Failures:* Spreadsheet permission errors, missing columns in the target sheet header, or invalid document IDs.
  - *Sub-workflow Reference:* None.
  - *Sticky Note:* `📊 Audit Ledger Logging` (Writes an immutable audit log entry containing timestamp, order ID, risk category, and action details for state reporting).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Receive Dispensary Order Webhook | `n8n-nodes-base.webhook` | Webhook Trigger | None | Normalize Order Intake | 📥 Order Ingestion<br>Captures completed order data from the POS system before fulfillment. Payload: `orderId`, `customerType`, `items`, `thcWeight`, `metrcTags` |
| Normalize Order Intake | `n8n-nodes-base.set` | Pre-processing | Receive Dispensary Order Webhook | Review Order Compliance | ⚙️ Intake Normalization<br>Cleanses raw JSON order data, formatting THC weights and METRC batch tags for accurate AI evaluation. |
| Review Order Compliance | `@n8n/n8n-nodes-langchain.agent` | AI Regulation Check | Normalize Order Intake | Shape Compliance Decision | 🧠 Compliance Audit Agent<br>Audits order against state rules: limits, customer type (Med/Rec), active recalls, and METRC tag validity. |
| GPT-5 Mini Compliance Model | `@n8n/n8n-nodes-langchain.lmChatOpenAi` | Model & Schema Enforcer | None (AI Subnode) | Review Order Compliance | 🤖 LLM & Schema Config<br>Powers the compliance evaluation via GPT-5 Mini and guarantees structured JSON response parameters. |
| Compliance Decision Schema | `@n8n/n8n-nodes-langchain.outputParserStructured` | Model & Schema Enforcer | None (AI Subnode) | Review Order Compliance | 🤖 LLM & Schema Config<br>Powers the compliance evaluation via GPT-5 Mini and guarantees structured JSON response parameters. |
| Shape Compliance Decision | `n8n-nodes-base.set` | Data Structuring | Review Order Compliance | Route by Compliance Category | 🎯 Decision Formatting<br>Extracts structured compliance metrics (`complianceCategory`, `flaggedIssues`, `actionRequired`). |
| Route by Compliance Category | `n8n-nodes-base.switch` | Conditional Switch | Shape Compliance Decision | Alert Budtender Order Clear to Fulfill, Alert Manager of Quantity Limit Breach, Alert Compliance Officer of METRC Mismatch, Alert Compliance Officer of Recall Flag | 🔀 Risk Category Router<br>Routes execution path based on audit status:<br>- `0`: Clear Order<br>- `1`: Quantity Breach<br>- `2`: METRC Mismatch<br>- `3`: Recall Flag |
| Alert Budtender Order Clear to Fulfill | `n8n-nodes-base.gmail` | Fulfillment Release | Route by Compliance Category | Combine Alert Outcomes | ✅ Budtender Clearance Alert<br>Notifies dispensary staff that the order meets all state guidelines and is cleared for packaging. |
| Alert Manager of Quantity Limit Breach | `n8n-nodes-base.gmail` | Manager Escalation | Route by Compliance Category | Combine Alert Outcomes | ⚠️ Quantity Limit Alert<br>Flags daily state possession limit overages to the store manager for manual review. |
| Alert Compliance Officer of METRC Mismatch | `n8n-nodes-base.gmail` | Compliance Officer Alert | Route by Compliance Category | Combine Alert Outcomes | 🔍 METRC Mismatch Alert<br>Notifies compliance team of untracked or invalid seed-to-sale tags before order release. |
| Alert Compliance Officer of Recall Flag | `n8n-nodes-base.gmail` | Emergency Product Hold | Route by Compliance Category | Combine Alert Outcomes | 🚨 Recall Emergency Alert<br>Triggers immediate notification to halt product distribution due to an active batch recall. |
| Combine Alert Outcomes | `n8n-nodes-base.merge` | Data Consolidation | Alert Budtender Order Clear to Fulfill, Alert Manager of Quantity Limit Breach, Alert Compliance Officer of METRC Mismatch, Alert Compliance Officer of Recall Flag | Append Compliance Review Log | 🔗 Branch Merger<br>Merges execution outputs from all alert branches back into a single unified stream for logging. |
| Append Compliance Review Log | `n8n-nodes-base.googleSheets` | Google Sheets Ledger | Combine Alert Outcomes | None | 📊 Audit Ledger Logging<br>Writes an immutable audit log entry containing timestamp, order ID, risk category, and action details for state reporting. |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow in n8n manually:

1. **Create the Webhook Trigger:**
   - Add a **Webhook** node named `Receive Dispensary Order Webhook`.
   - Set **HTTP Method** to `POST` and **Path** to `dispensary-order-review`.
2. **Add Intake Normalization:**
   - Add a **Set** node named `Normalize Order Intake`.
   - Configure assignments to map incoming body fields with safe defaults:
     - `orderId`: `={{ $json.body?.orderId ?? $json.orderId ?? "" }}`
     - `dispensaryId`: `={{ $json.body?.dispensaryId ?? $json.dispensaryId ?? "" }}`
     - `customerType`: `={{ $json.body?.customerType ?? $json.customerType ?? "ADULT_USE" }}`
     - `lineItems`: `={{ $json.body?.lineItems ?? $json.lineItems ?? [] }}`
     - `customerDailyPurchasedGramsSoFar`: `={{ $json.body?.customerDailyPurchasedGramsSoFar ?? $json.customerDailyPurchasedGramsSoFar ?? 0 }}`
     - `state`: `={{ $json.body?.state ?? $json.state ?? "" }}`
     - `budtenderId`: `={{ $json.body?.budtenderId ?? $json.budtenderId ?? "" }}`
   - Connect `Receive Dispensary Order Webhook` to `Normalize Order Intake`.
3. **Configure the AI Agent and Subnodes:**
   - Add an **Advanced AI Agent** node named `Review Order Compliance`. Set prompt type to `define`. Configure text input to evaluate order parameters and line items as a JSON string.
   - Add an **OpenAI Chat Model** node named `GPT-5 Mini Compliance Model`. Set the model to `gpt-5-mini` and temperature to `0.1`. Configure and attach an OpenAI API credential. Connect this node to the AI language model input of `Review Order Compliance`.
   - Add a **Structured Output Parser** node named `Compliance Decision Schema`. Set the JSON schema example to include fields for `category`, `confidence`, `reasoning`, `gramsRemainingAfterOrder`, and `correctedAction`. Connect this node to the AI output parser input of `Review Order Compliance`.
   - Connect `Normalize Order Intake` output to `Review Order Compliance`.
4. **Shape the AI Decision:**
   - Add a **Set** node named `Shape Compliance Decision`.
   - Map normalized order properties (`orderId`, `dispensaryId`, `budtenderId`, `state`, `customerType`) alongside parsed AI output properties (`category`, `confidence`, `reasoning`, `gramsRemainingAfterOrder`, `correctedAction`) and include `reviewedAt` set to `={{ $now.toISO() }}`.
   - Connect `Review Order Compliance` output to `Shape Compliance Decision`.
5. **Route by Compliance Category:**
   - Add a **Switch** node named `Route by Compliance Category`.
   - Define four rules evaluating `={{ $json.category }}` equal to:
     1. `CLEAR_TO_FULFILL`
     2. `QUANTITY_LIMIT_BREACH`
     3. `METRC_TAG_MISMATCH`
     4. `RECALL_FLAG`
   - Set the fallback output option to redirect unclassified output to the recall handling branch.
   - Connect `Shape Compliance Decision` output to `Route by Compliance Category`.
6. **Set Up Notification Branches:**
   - Add four **Gmail** nodes:
     - `Alert Budtender Order Clear to Fulfill` (Branch 0)
     - `Alert Manager of Quantity Limit Breach` (Branch 1)
     - `Alert Compliance Officer of METRC Mismatch` (Branch 2)
     - `Alert Compliance Officer of Recall Flag` (Branches 3 & 4 / Fallback)
   - Configure each Gmail node with a Gmail OAuth2 credential, `onError` set to `continueRegularOutput`, `retryOnFail` enabled, max tries set to `3`, and wait time set to `2000ms`.
   - Customize the `sendTo` field with your recipient email addresses and populate subjects and message bodies using expressions referencing `$('Shape Compliance Decision').item.json.<property>`.
   - Connect the respective outputs of `Route by Compliance Category` to each Gmail node.
7. **Merge Notification Outcomes:**
   - Add a **Merge** node named `Combine Alert Outcomes` configured with `4` number inputs.
   - Connect all four Gmail nodes to the respective inputs of `Combine Alert Outcomes`.
8. **Configure Audit Ledger Logging:**
   - Add a **Google Sheets** node named `Append Compliance Review Log`.
   - Configure credentials using a Google Sheets OAuth2 API account.
   - Set operation to `append`, specify your target Document ID and Sheet Tab Name, and map columns (`OrderID`, `DispensaryID`, `BudtenderID`, `State`, `CustomerType`, `ComplianceCategory`, `Confidence`, `Reasoning`, `GramsRemainingAfterOrder`, `CorrectedAction`, `ReviewedAt`) to expressions reading from `$('Shape Compliance Decision').item.json.<property>`.
   - Configure error handling to `continueRegularOutput` with retry enabled.
   - Connect `Combine Alert Outcomes` output to `Append Compliance Review Log`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Creator Credit & Support | Created by Swapnil AI Labs — [swapnil.mandloi7@gmail.com](mailto:swapnil.mandloi7@gmail.com) — [swapnilailabs.netlify.app](https://swapnilailabs.netlify.app) |
| Regulatory Disclaimer & Guardrails | Thresholds and policy rules in the workflow are illustrative defaults. Users must verify current state regulations and integrate live recall feeds for production use. |
| Data Privacy & PII | The workflow processes operational order and product data only and is designed not to collect customer names, dates of birth, ID numbers, or payment data. |