Recover abandoned Shopify carts with Llama 3.2, Gmail, and Google Sheets

https://n8nworkflows.xyz/workflows/recover-abandoned-shopify-carts-with-llama-3-2--gmail--and-google-sheets-20509


# Recover abandoned Shopify carts with Llama 3.2, Gmail, and Google Sheets

### 1. Workflow Overview

This workflow automates the Shopify cart recovery process by capturing checkout and order webhooks, tracking states in Google Sheets, generating context-aware multi-channel follow-up emails via an Ollama-hosted Llama 3.2 model, and dispatching summaries via Gmail.

The architecture is divided into three primary logical blocks:
- **1.1 Input Reception & Dispatch:** Captures inbound Shopify events via Webhook and branches execution paths depending on whether the event represents a checkout lifecycle event or a completed order.
- **1.2 Checkout Logging & Multi-Stage Recovery Sequence:** Logs new checkouts, waits through a multi-tier sequence (1 hour, 24 hours, and an additional 24 hours), checks recovery status before each stage, uses Llama 3.2 to compose escalating reminders (including a 10% discount code `CART10`), sends emails via Gmail, and logs state updates.
- **1.3 Order Recovery & Weekly Reporting:** Marks checkouts as "Recovered" when orders complete, triggers a secondary AI thank-you/upsell sequence, and maintains a scheduled cron job to compute and email weekly revenue performance metrics.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Input Reception & Dispatch
- **Overview:** Serves as the central webhook entry point, sorting incoming Shopify payloads into checkout events and order completion events.
- **Nodes Involved:** 
  - `On Shopify checkout or order event`
  - `Filter Checkout Events`
  - `Filter Orders`
- **Node Details:**
  - **`On Shopify checkout or order event`**
    - *Type & Technical Role:* Webhook node (v2.1) acting as the HTTP listener for Shopify event notifications.
    - *Configuration Choices:* Listens for POST requests at a unique path (`1691d4d5-82ec-4bc7-9309-955d2954e693`).
    - *Key Expressions/Variables:* None (captures raw request headers and body).
    - *Input/Output:* No inputs; outputs two parallel flows (`Filter Checkout Events` and `Filter Orders`).
    - *Edge Cases/Failures:* Signature validation failures if Shopify webhook secrets mismatch; network timeouts if Shopify retries rapidly.
  - **`Filter Checkout Events`**
    - *Type & Technical Role:* Filter node (v2.3) routing events tagged as checkout creations or updates.
    - *Configuration Choices:* Evaluates `{{ $json.headers['x-shopify-topic'] }}` to match either `checkouts/create` or `checkouts/update`.
    - *Input/Output:* Input from Webhook; output to `Log Checkout`.
  - **`Filter Orders`**
    - *Type & Technical Role:* Filter node (v2.3) isolating completed order payloads.
    - *Configuration Choices:* Evaluates `{{ $json.headers['x-shopify-topic'] }}` to match `orders/create`.
    - *Input/Output:* Input from Webhook; output to `Sheets Order Recovered`.

---

#### Block 1.2: Checkout Logging & Multi-Stage Recovery Sequence
- **Overview:** Logs new checkouts, enforces a delayed sequence of three escalating emails (Reminder, Discount, Urgency) while validating real-time conversion status in Google Sheets.
- **Nodes Involved:** 
  - `Log Checkout`, `Filter New Checkouts`, `Wait 1h Before Reminder`, `Get Checkout Row`, `Filter Reminder`, `HTTP Request Reminder`, `Code Reminder`, `Gmail Reminder`, `Sheets Reminder Sent`, `Wait 24h Before Discount`, `Get Checkout Row 2`, `Filter Discount`, `HTTP Request Discount`, `Code Discount`, `Gmail Discount`, `Sheets Discount Sent`, `Wait 24h Before Urgency`, `Get Checkout Row 3`, `Filter Urgency`, `HTTP Request Urgency`, `Code Urgency`, `Gmail Urgency`, `Sheets Urgency Sent`
- **Node Details:**
  - **`Log Checkout`**
    - *Type & Technical Role:* Google Sheets node (v4.7) appending or updating checkout metadata.
    - *Configuration Choices:* Uses `appendOrUpdate` operation matching on `Checkout Token`. Maps email, price, product, created date, recovery link, and execution timestamp.
    - *Credentials:* `Google Sheets account`
    - *Input/Output:* Input from `Filter Checkout Events`; output to `Filter New Checkouts`.
    - *Edge Cases/Failures:* API rate limits, authentication token expiration.
  - **`Filter New Checkouts`**
    - *Type & Technical Role:* Filter node (v2.3) restricting the reminder pipeline exclusively to new cart creations.
    - *Configuration Choices:* Asserts header topic equals `checkouts/create`.
    - *Input/Output:* Input from `Log Checkout`; output to `Wait 1h Before Reminder`.
  - **`Wait 1h Before Reminder`**
    - *Type & Technical Role:* Wait node (v1.1) pausing execution.
    - *Configuration Choices:* Pauses for 1 hour.
    - *Input/Output:* Input from `Filter New Checkouts`; output to `Get Checkout Row`.
    - *Edge Cases/Failures:* Server reboots persisting wait states incorrectly if unmanaged.
  - **`Get Checkout Row`**
    - *Type & Technical Role:* Google Sheets node (v4.7) retrieving current status.
    - *Configuration Choices:* Filters row where `Checkout Token` matches `{{ $('On Shopify checkout or order event').first().json.body.cart_token }}`.
    - *Credentials:* `Google Sheets account`
    - *Input/Output:* Input from `Wait 1h Before Reminder`; output to `Filter Reminder`.
  - **`Filter Reminder`**
    - *Type & Technical Role:* Filter node (v2.3) halting execution if the cart converted.
    - *Configuration Choices:* Checks that `{{ $json.Status }}` does not equal `Recovered`.
    - *Input/Output:* Input from `Get Checkout Row`; output to `HTTP Request Reminder`.
  - **`HTTP Request Reminder`**
    - *Type & Technical Role:* HTTP Request node (v4.5) communicating with a local Ollama model instance.
    - *Configuration Choices:* POST to `http://host.docker.internal:11434/api/chat` with model `llama3.2`. Sets JSON body prompt requesting a light, non-pushy cart reminder. Timeout set to 300,000ms.
    - *Input/Output:* Input from `Filter Reminder`; output to `Code Reminder`.
    - *Edge Cases/Failures:* Ollama service unavailability, connection refusal, model inference timeouts.
  - **`Code Reminder`**
    - *Type & Technical Role:* Code node (v2) parsing raw LLM output into structured email fields.
    - *Configuration Choices:* Extracts subject and body regex matches, strips conversational filler, sanitizes line breaks, and wraps output in HTML template with corporate signature (`StrideWell Footwear`).
    - *Input/Output:* Input from `HTTP Request Reminder`; output to `Gmail Reminder`.
  - **`Gmail Reminder`**
    - *Type & Technical Role:* Gmail node (v2.2) dispatching outbound messages.
    - *Configuration Choices:* Sends message body (`{{ $json.email_body }}`) to recipient (`{{ $json.to_email }}`) with subject (`{{ $json.email_subject }}`). Disables attribution appending.
    - *Credentials:* `Gmail account`
    - *Input/Output:* Input from `Code Reminder`; output to `Sheets Reminder Sent`.
  - **`Sheets Reminder Sent`**
    - *Type & Technical Role:* Google Sheets node (v4.7) updating status.
    - *Configuration Choices:* Updates matching `Checkout Token` row, setting `Status` to `Reminder Sent` and updating `Last Updated` timestamp.
    - *Credentials:* `Google Sheets account`
    - *Input/Output:* Input from `Gmail Reminder`; output to `Wait 24h Before Discount`.
  - **`Wait 24h Before Discount`**
    - *Type & Technical Role:* Wait node (v1.1) pausing for 24 hours.
    - *Input/Output:* Input from `Sheets Reminder Sent`; output to `Get Checkout Row 2`.
  - **`Get Checkout Row 2`**
    - *Type & Technical Role:* Google Sheets node (v4.7) querying current row status.
    - *Input/Output:* Input from `Wait 24h Before Discount`; output to `Filter Discount`.
  - **`Filter Discount`**
    - *Type & Technical Role:* Filter node (v2.3) validating status is not `Recovered`.
    - *Input/Output:* Input from `Get Checkout Row 2`; output to `HTTP Request Discount`.
  - **`HTTP Request Discount`**
    - *Type & Technical Role:* HTTP Request node (v4.5) calling Ollama for the second follow-up.
    - *Configuration Choices:* Requests a warmer tone mentioning a 10% discount using code `CART10`.
    - *Input/Output:* Input from `Filter Discount`; output to `Code Discount`.
  - **`Code Discount`**
    - *Type & Technical Role:* Code node (v2) parsing the second LLM response.
    - *Input/Output:* Input from `HTTP Request Discount`; output to `Gmail Discount`.
  - **`Gmail Discount`**
    - *Type & Technical Role:* Gmail node (v2.2) sending the discount follow-up email.
    - *Credentials:* `Gmail account`
    - *Input/Output:* Input from `Code Discount`; output to `Sheets Discount Sent`.
  - **`Sheets Discount Sent`**
    - *Type & Technical Role:* Google Sheets node (v4.7) logging status update.
    - *Configuration Choices:* Updates matching row status to `Discount Sent`.
    - *Credentials:* `Google Sheets account`
    - *Input/Output:* Input from `Gmail Discount`; output to `Wait 24h Before Urgency`.
  - **`Wait 24h Before Urgency`**
    - *Type & Technical Role:* Wait node (v1.1) pausing for final 24 hours.
    - *Input/Output:* Input from `Sheets Discount Sent`; output to `Get Checkout Row 3`.
  - **`Get Checkout Row 3`**
    - *Type & Technical Role:* Google Sheets node (v4.7) fetching status.
    - *Input/Output:* Input from `Wait 24h Before Urgency`; output to `Filter Urgency`.
  - **`Filter Urgency`**
    - *Type & Technical Role:* Filter node (v2.3) confirming status is not `Recovered`.
    - *Input/Output:* Input from `Get Checkout Row 3`; output to `HTTP Request Urgency`.
  - **`HTTP Request Urgency`**
    - *Type & Technical Role:* HTTP Request node (v4.5) querying Ollama for the final urgency email.
    - *Configuration Choices:* Requests last-chance urgency framing without inventing arbitrary stock counts, reinforcing code `CART10`.
    - *Input/Output:* Input from `Filter Urgency`; output to `Code Urgency`.
  - **`Code Urgency`**
    - *Type & Technical Role:* Code node (v2) formatting final email text.
    - *Input/Output:* Input from `HTTP Request Urgency`; output to `Gmail Urgency`.
  - **`Gmail Urgency`**
    - *Type & Technical Role:* Gmail node (v2.2) dispatching final email.
    - *Credentials:* `Gmail account`
    - *Input/Output:* Input from `Code Urgency`; output to `Sheets Urgency Sent`.
  - **`Sheets Urgency Sent`**
    - *Type & Technical Role:* Google Sheets node (v4.7) updating sheet status to `Urgency Sent`.
    - *Credentials:* `Google Sheets account`
    - *Input/Output:* Input from `Gmail Urgency`; terminal node for this sequence.

---

#### Block 1.3: Order Recovery & Weekly Reporting
- **Overview:** Handles conversion confirmation upon order creation, triggers post-purchase upsell recommendations via LLM, and administers weekly cron-based metrics reporting.
- **Nodes Involved:** 
  - `Sheets Order Recovered`, `Pick Upsell`, `HTTP Request Thank you`, `Code Thank you`, `Gmail Thank you`, `Every Monday at 9am`, `Get All Rows`, `Weekly Stats`, `Gmail Weekly Summary`
- **Node Details:**
  - **`Sheets Order Recovered`**
    - *Type & Technical Role:* Google Sheets node (v4.7) updating checkout state upon conversion.
    - *Configuration Choices:* Updates row matching `Checkout Token`, setting `Status` to `Recovered`.
    - *Credentials:* `Google Sheets account`
    - *Input/Output:* Input from `Filter Orders`; output to `Pick Upsell`.
  - **`Pick Upsell`**
    - *Type & Technical Role:* Code node (v2) randomly selecting a complementary product from a hardcoded catalog.
    - *Configuration Choices:* Filters out the purchased item from a catalog array containing `Trail Runner Pro`, `Classic Canvas Sneaker`, and `All-Day Comfort Slip-On`, picking one at random.
    - *Input/Output:* Input from `Sheets Order Recovered`; output to `HTTP Request Thank you`.
  - **`HTTP Request Thank you`**
    - *Type & Technical Role:* HTTP Request node (v4.5) generating post-purchase thank-you and upsell copy via Ollama.
    - *Input/Output:* Input from `Pick Upsell`; output to `Code Thank you`.
  - **`Code Thank you`**
    - *Type & Technical Role:* Code node (v2) parsing thank-you response data.
    - *Input/Output:* Input from `HTTP Request Thank you`; output to `Gmail Thank you`.
  - **`Gmail Thank you`**
    - *Type & Technical Role:* Gmail node (v2.2) emailing the customer their order confirmation and upsell.
    - *Credentials:* `Gmail account`
    - *Input/Output:* Input from `Code Thank you`; terminal node for order recovery.
  - **`Every Monday at 9am`**
    - *Type & Technical Role:* Schedule Trigger node (v1.4) cron scheduler.
    - *Configuration Choices:* Configured to trigger weekly on Mondays at 09:00.
    - *Input/Output:* No inputs; output to `Get All Rows`.
  - **`Get All Rows`**
    - *Type & Technical Role:* Google Sheets node (v4.7) reading tracker records for reporting.
    - *Credentials:* `Google Sheets account`
    - *Input/Output:* Input from `Every Monday at 9am`; output to `Weekly Stats`.
  - **`Weekly Stats`**
    - *Type & Technical Role:* Code node (v2) calculating aggregate metrics over the past 7 days.
    - *Configuration Choices:* Filters tracker rows created within the last 7 days, computes total abandoned carts, recovered counts, recovered revenue sum, and recovery percentage rate.
    - *Input/Output:* Input from `Get All Rows`; output to `Gmail Weekly Summary`.
  - **`Gmail Weekly Summary`**
    - *Type & Technical Role:* Gmail node (v2.2) sending performance reports.
    - *Configuration Choices:* Sends summary to configured recipient address (`your-email@example.com`).
    - *Credentials:* `Gmail account`
    - *Input/Output:* Input from `Weekly Stats`; terminal reporting node.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| On Shopify checkout or order event | n8n-nodes-base.webhook | Receives incoming Shopify webhooks | None | Filter Orders, Filter Checkout Events | ### How it works<br>Shopify webhooks (checkout creation, checkout update, order creation) feed one entry point... |
| Filter Checkout Events | n8n-nodes-base.filter | Isolates checkout create/update events | On Shopify checkout or order event | Log Checkout | ### How it works<br>Shopify webhooks (checkout creation, checkout update, order creation) feed one entry point... |
| Log Checkout | n8n-nodes-base.googleSheets | Logs new checkouts to Google Sheets | Filter Checkout Events | Filter New Checkouts | ### How it works<br>Shopify webhooks (checkout creation, checkout update, order creation) feed one entry point... |
| Filter New Checkouts | n8n-nodes-base.filter | Restricts execution to cart creation events | Log Checkout | Wait 1h Before Reminder | ### How it works<br>Shopify webhooks (checkout creation, checkout update, order creation) feed one entry point... |
| Wait 1h Before Reminder | n8n-nodes-base.wait | Pauses workflow for 1 hour | Filter New Checkouts | Get Checkout Row | ## Checkout Logging & Reminder Sequence<br>Logs every checkout, then waits 1h → 24h → 24h, checking for a purchase before each AI email. |
| Get Checkout Row | n8n-nodes-base.googleSheets | Fetches latest checkout row status | Wait 1h Before Reminder | Filter Reminder | ## Checkout Logging & Reminder Sequence<br>Logs every checkout, then waits 1h → 24h → 24h, checking for a purchase before each AI email. |
| Filter Reminder | n8n-nodes-base.filter | Skips if cart status is Recovered | Get Checkout Row | HTTP Request Reminder | ## Checkout Logging & Reminder Sequence<br>Logs every checkout, then waits 1h → 24h → 24h, checking for a purchase before each AI email. |
| HTTP Request Reminder | n8n-nodes-base.httpRequest | Calls Ollama to draft reminder email | Filter Reminder | Code Reminder | ## Checkout Logging & Reminder Sequence<br>Logs every checkout, then waits 1h → 24h → 24h, checking for a purchase before each AI email. |
| Code Reminder | n8n-nodes-base.code | Parses and formats reminder email content | HTTP Request Reminder | Gmail Reminder | ## Checkout Logging & Reminder Sequence<br>Logs every checkout, then waits 1h → 24h → 24h, checking for a purchase before each AI email. |
| Gmail Reminder | n8n-nodes-base.gmail | Sends reminder email via Gmail | Code Reminder | Sheets Reminder Sent | ## Checkout Logging & Reminder Sequence<br>Logs every checkout, then waits 1h → 24h → 24h, checking for a purchase before each AI email. |
| Sheets Reminder Sent | n8n-nodes-base.googleSheets | Updates status to Reminder Sent | Gmail Reminder | Wait 24h Before Discount | ## Checkout Logging & Reminder Sequence<br>Logs every checkout, then waits 1h → 24h → 24h, checking for a purchase before each AI email. |
| Wait 24h Before Discount | n8n-nodes-base.wait | Pauses workflow for 24 hours | Sheets Reminder Sent | Get Checkout Row 2 | ## Checkout Logging & Reminder Sequence<br>Logs every checkout, then waits 1h → 24h → 24h, checking for a purchase before each AI email. |
| Get Checkout Row 2 | n8n-nodes-base.googleSheets | Fetches checkout row status before discount | Wait 24h Before Discount | Filter Discount | ## Checkout Logging & Reminder Sequence<br>Logs every checkout, then waits 1h → 24h → 24h, checking for a purchase before each AI email. |
| Filter Discount | n8n-nodes-base.filter | Checks if cart is unrecovered | Get Checkout Row 2 | HTTP Request Discount | ## Checkout Logging & Reminder Sequence<br>Logs every checkout, then waits 1h → 24h → 24h, checking for a purchase before each AI email. |
| HTTP Request Discount | n8n-nodes-base.httpRequest | Requests 10% discount email copy from Ollama | Filter Discount | Code Discount | ## Checkout Logging & Reminder Sequence<br>Logs every checkout, then waits 1h → 24h → 24h, checking for a purchase before each AI email. |
| Code Discount | n8n-nodes-base.code | Formats discount email HTML and subject | HTTP Request Discount | Gmail Discount | ## Checkout Logging & Reminder Sequence<br>Logs every checkout, then waits 1h → 24h → 24h, checking for a purchase before each AI email. |
| Gmail Discount | n8n-nodes-base.gmail | Sends discount follow-up email | Code Discount | Sheets Discount Sent | ## Checkout Logging & Reminder Sequence<br>Logs every checkout, then waits 1h → 24h → 24h, checking for a purchase before each AI email. |
| Sheets Discount Sent | n8n-nodes-base.googleSheets | Updates sheet status to Discount Sent | Gmail Discount | Wait 24h Before Urgency | ## Checkout Logging & Reminder Sequence<br>Logs every checkout, then waits 1h → 24h → 24h, checking for a purchase before each AI email. |
| Wait 24h Before Urgency | n8n-nodes-base.wait | Pauses workflow for final 24 hours | Sheets Discount Sent | Get Checkout Row 3 | ## Checkout Logging & Reminder Sequence<br>Logs every checkout, then waits 1h → 24h → 24h, checking for a purchase before each AI email. |
| Get Checkout Row 3 | n8n-nodes-base.googleSheets | Fetches row status prior to final email | Wait 24h Before Urgency | Filter Urgency | ## Checkout Logging & Reminder Sequence<br>Logs every checkout, then waits 1h → 24h → 24h, checking for a purchase before each AI email. |
| Filter Urgency | n8n-nodes-base.filter | Validates cart has not been recovered | Get Checkout Row 3 | HTTP Request Urgency | ## Checkout Logging & Reminder Sequence<br>Logs every checkout, then waits 1h → 24h → 24h, checking for a purchase before each AI email. |
| HTTP Request Urgency | n8n-nodes-base.httpRequest | Requests final urgency copy from Ollama | Filter Urgency | Code Urgency | ## Checkout Logging & Reminder Sequence<br>Logs every checkout, then waits 1h → 24h → 24h, checking for a purchase before each AI email. |
| Code Urgency | n8n-nodes-base.code | Formats final urgency email | HTTP Request Urgency | Gmail Urgency | ## Checkout Logging & Reminder Sequence<br>Logs every checkout, then waits 1h → 24h → 24h, checking for a purchase before each AI email. |
| Gmail Urgency | n8n-nodes-base.gmail | Dispatches final urgency email | Code Urgency | Sheets Urgency Sent | ## Checkout Logging & Reminder Sequence<br>Logs every checkout, then waits 1h → 24h → 24h, checking for a purchase before each AI email. |
| Sheets Urgency Sent | n8n-nodes-base.googleSheets | Updates sheet status to Urgency Sent | Gmail Urgency | None | ## Checkout Logging & Reminder Sequence<br>Logs every checkout, then waits 1h → 24h → 24h, checking for a purchase before each AI email. |
| Filter Orders | n8n-nodes-base.filter | Isolates orders/create webhook events | On Shopify checkout or order event | Sheets Order Recovered | ### How it works<br>Shopify webhooks (checkout creation, checkout update, order creation) feed one entry point... |
| Sheets Order Recovered | n8n-nodes-base.googleSheets | Updates status to Recovered in sheet | Filter Orders | Pick Upsell | ## Order Recovery & Upsell<br>Fires on order completion. Marks the checkout Recovered and sends an AI thank-you + upsell email. |
| Pick Upsell | n8n-nodes-base.code | Selects a random upsell product | Sheets Order Recovered | HTTP Request Thank you | ## Order Recovery & Upsell<br>Fires on order completion. Marks the checkout Recovered and sends an AI thank-you + upsell email. |
| HTTP Request Thank you | n8n-nodes-base.httpRequest | Requests post-purchase thank-you copy | Pick Upsell | Code Thank you | ## Order Recovery & Upsell<br>Fires on order completion. Marks the checkout Recovered and sends an AI thank-you + upsell email. |
| Code Thank you | n8n-nodes-base.code | Formats thank-you email content | HTTP Request Thank you | Gmail Thank you | ## Order Recovery & Upsell<br>Fires on order completion. Marks the checkout Recovered and sends an AI thank-you + upsell email. |
| Gmail Thank you | n8n-nodes-base.gmail | Sends thank-you and upsell email | Code Thank you | None | ## Order Recovery & Upsell<br>Fires on order completion. Marks the checkout Recovered and sends an AI thank-you + upsell email. |
| Every Monday at 9am | n8n-nodes-base.scheduleTrigger | Triggers weekly report execution | None | Get All Rows | ## Weekly Report<br>Every Monday 9am: reads the tracker and emails the store owner recovered revenue and recovery rate. |
| Get All Rows | n8n-nodes-base.googleSheets | Reads all rows from the tracking sheet | Every Monday at 9am | Weekly Stats | ## Weekly Report<br>Every Monday 9am: reads the tracker and emails the store owner recovered revenue and recovery rate. |
| Weekly Stats | n8n-nodes-base.code | Computes revenue and recovery rate | Get All Rows | Gmail Weekly Summary | ## Weekly Report<br>Every Monday 9am: reads the tracker and emails the store owner recovered revenue and recovery rate. |
| Gmail Weekly Summary | n8n-nodes-base.gmail | Sends weekly summary report | Weekly Stats | None | ## Weekly Report<br>Every Monday 9am: reads the tracker and emails the store owner recovered revenue and recovery rate. |
| Filter Checkout Events | n8n-nodes-base.filter | Matches checkout create/update headers | On Shopify checkout or order event | Log Checkout | ### How it works<br>Shopify webhooks (checkout creation, checkout update, order creation) feed one entry point... |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Entry Webhook Node**
   - Create a **Webhook** node named `On Shopify checkout or order event`.
   - Set HTTP Method to `POST`. Note the generated Webhook URL to register in Shopify.

2. **Setup Branching Filters**
   - Create a **Filter** node named `Filter Checkout Events`: set condition to check if `{{ $json.headers['x-shopify-topic'] }}` equals `checkouts/create` or `checkouts/update` (using an OR combination matching `Filter Checkout Events` vs `Filter Checkout Events` block logic). Alternatively, use two separate filter nodes or an `if` node. Connect `On Shopify checkout or order event` to it.
   - Create a **Filter** node named `Filter Orders`: set condition to check if `{{ $json.headers['x-shopify-topic'] }}` equals `orders/create`. Connect `On Shopify checkout or order event` to it.

3. **Rebuild the Order Recovery Branch**
   - Connect `Filter Orders` to a **Google Sheets** node named `Sheets Order Recovered`.
     - *Operation:* `appendOrUpdate`, matching columns: `Checkout Token`.
     - *Credential:* Configure `Google Sheets account`.
   - Connect `Sheets Order Recovered` to a **Code** node named `Pick Upsell`. Paste the JavaScript array catalog and random selection logic.
   - Connect `Pick Upsell` to an **HTTP Request** node named `HTTP Request Thank you`.
     - *Method:* `POST`, URL: `http://host.docker.internal:11434/api/chat`. Set JSON body prompt with system instructions for a post-purchase thank-you note.
   - Connect `HTTP Request Thank you` to a **Code** node named `Code Thank you` to parse the LLM output.
   - Connect `Code Thank you` to a **Gmail** node named `Gmail Thank you` using credential `Gmail account`.

4. **Rebuild the Checkout Logging & Reminder Pipeline**
   - Connect `Filter Checkout Events` to a **Google Sheets** node named `Log Checkout` (`appendOrUpdate` on `Checkout Token`).
   - Connect `Log Checkout` to a **Filter** node named `Filter New Checkouts` (topic equals `checkouts/create`).
   - Connect `Filter New Checkouts` to a **Wait** node named `Wait 1h Before Reminder` (amount: `1`, unit: `hours`).
   - Connect `Wait 1h Before Reminder` to a **Google Sheets** node named `Get Checkout Row` (lookup column `Checkout Token` with value `{{ $('On Shopify checkout or order event').first().json.body.cart_token }}`).
   - Connect `Get Checkout Row` to a **Filter** node named `Filter Reminder` (condition: Status `notEquals` `Recovered`).
   - Connect `Filter Reminder` to an **HTTP Request** node named `HTTP Request Reminder` (Ollama endpoint, model `llama3.2`).
   - Connect `HTTP Request Reminder` to a **Code** node named `Code Reminder`.
   - Connect `Code Reminder` to a **Gmail** node named `Gmail Reminder`.
   - Connect `Gmail Reminder` to a **Google Sheets** node named `Sheets Reminder Sent` (sets Status to `Reminder Sent`).

5. **Rebuild the Discount Follow-Up Branch**
   - Connect `Sheets Reminder Sent` to a **Wait** node named `Wait 24h Before Discount` (24 hours).
   - Connect it to `Get Checkout Row 2`, then `Filter Discount`, `HTTP Request Discount`, `Code Discount`, `Gmail Discount`, and `Sheets Discount Sent` (sets Status to `Discount Sent`).

6. **Rebuild the Urgency Follow-Up Branch**
   - Connect `Sheets Discount Sent` to a **Wait** node named `Wait 24h Before Urgency` (24 hours).
   - Connect it to `Get Checkout Row 3`, then `Filter Urgency`, `HTTP Request Urgency`, `Code Urgency`, `Gmail Urgency`, and `Sheets Urgency Sent` (sets Status to `Urgency Sent`).

7. **Rebuild the Weekly Report Pipeline**
   - Create a **Schedule Trigger** node named `Every Monday at 9am` (Trigger at day: Monday, Hour: 9).
   - Connect to a **Google Sheets** node named `Get All Rows` (reads all spreadsheet records).
   - Connect to a **Code** node named `Weekly Stats` to compute 7-day metrics.
   - Connect to a **Gmail** node named `Gmail Weekly Summary` to email the report.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Shopify Webhook Configuration | Register webhooks in Shopify via Settings → Notifications → Webhooks using API version 2026-07. |
| Google Sheets Tracking Schema | Required headers: `Checkout Token`, `Email`, `Product`, `Price`, `Status`, `Recovery Link`, `Created At`, `Last Updated`. |
| Local AI Model Requirements | Ollama must be running locally serving `llama3.2` at `http://host.docker.internal:11434/api/chat`, or substitute with an OpenAI/Anthropic API endpoint. |