Send weekly WooCommerce finance KPIs to Slack using HTTP APIs

https://n8nworkflows.xyz/workflows/send-weekly-woocommerce-finance-kpis-to-slack-using-http-apis-13272


# Send weekly WooCommerce finance KPIs to Slack using HTTP APIs

## 1. Workflow Overview

**Workflow name:** Weekly WooCommerce Finance KPI Automation with HTTP APIs & Slack  
**Purpose:** Automatically fetch WooCommerce **orders** and **refunds** on a weekly schedule, calculate finance KPIs (sales, refunds, ratios, risk flags), and post a CFO-style digest to a Slack channel.

**Primary use cases**
- Weekly finance reporting for e-commerce leadership
- Monitoring refund rates and potential revenue leakage
- Lightweight KPI reporting without a data warehouse (direct HTTP API pull)

### 1.1 Schedule & Store Setup
Runs weekly and defines the WooCommerce store domain used by all API calls.

### 1.2 Order Data Collection & Preparation
Fetches WooCommerce orders, keeps only `completed` orders, then normalizes fields used in KPI math.

### 1.3 Refund Data Collection & Preparation
Fetches refunds, normalizes key refund fields for later aggregation.

### 1.4 KPI Calculation & Executive Reporting
Merges orders + refunds, computes totals/ratios/flags in code, then posts a formatted digest to Slack.

---

## 2. Block-by-Block Analysis

### Block 1 — Schedule & Store Setup
**Overview:** Triggers the workflow weekly and sets the base WooCommerce domain used by downstream HTTP requests. This is the “configuration anchor” for the two parallel data pulls.

**Nodes involved**
- Weekly KPI Scheduler
- Configure WooCommerce Store

#### Node: Weekly KPI Scheduler
- **Type / role:** `Schedule Trigger` — entry point; runs on a weekly timer.
- **Configuration (interpreted):** Runs **every week**, on **day 1 (Monday)** at **10:00** (instance timezone applies).
- **Inputs/outputs:** No inputs (trigger). Outputs to **Configure WooCommerce Store**.
- **Version notes:** typeVersion `1.2` (standard schedule trigger).
- **Edge cases / failures:**
  - Timezone mismatch between expected business timezone and n8n instance timezone.
  - If the instance is down at trigger time, the run may be skipped (depends on n8n deployment/restart behavior).

#### Node: Configure WooCommerce Store
- **Type / role:** `Set` — defines `wc_domain` used to build API URLs.
- **Configuration choices:**
  - Creates field **`wc_domain`** (string). **Currently empty** in the workflow JSON, so it must be filled (e.g., `example.com` or `store.example.com`).
- **Key expressions/variables:** Produces `$json.wc_domain`.
- **Inputs/outputs:** Input from **Weekly KPI Scheduler**. Outputs in parallel to:
  - **Fetch WooCommerce Orders**
  - **Fetch WooCommerce Refunds**
- **Version notes:** typeVersion `3.4`.
- **Edge cases / failures:**
  - Empty or malformed domain will cause invalid URLs and HTTP failures.
  - If you include protocol in `wc_domain` (like `https://…`), the downstream URL expressions will produce `https://https://…` unless adjusted.

**Sticky note coverage (content applies to this block’s nodes):**  
“## Schedule & Store Setup … scheduler triggers … store configuration ensures all API requests point to the correct WooCommerce domain.”

---

### Block 2 — Order Data Collection & Preparation
**Overview:** Pulls orders from the WooCommerce REST API, filters to completed orders only, then extracts a small, consistent schema used in KPI computations.

**Nodes involved**
- Fetch WooCommerce Orders
- Filter Completed Orders
- Normalize Order Data

#### Node: Fetch WooCommerce Orders
- **Type / role:** `HTTP Request` — calls WooCommerce REST API to list orders.
- **Configuration choices:**
  - **URL:** `https://{{$json.wc_domain}}/wp-json/wc/v3/orders`
  - **Authentication:** Generic credential type → **HTTP Basic Auth**
  - **Credentials used:** “Woo-Commerece Auth” (HTTP Basic)
- **Inputs/outputs:**
  - Input: from **Configure WooCommerce Store** (needs `wc_domain`)
  - Output: to **Filter Completed Orders**
- **Version notes:** typeVersion `4.3`.
- **Edge cases / failures:**
  - **401/403** if credentials are wrong or API keys lack permissions.
  - Pagination: WooCommerce orders endpoint is usually paginated (often `per_page` default 10). This node does **not** set `per_page` or pagination handling, so results may be incomplete.
  - Large stores may require date filters and pagination to avoid timeouts and memory pressure.
  - Depending on WooCommerce configuration, Basic Auth over HTTPS is typical; over HTTP it may be blocked.

#### Node: Filter Completed Orders
- **Type / role:** `Filter` — keeps only finance-relevant orders.
- **Configuration choices:**
  - Condition: `{{ $json.status }} equals "completed"`
  - Combiner: `and` (single condition)
- **Inputs/outputs:**
  - Input: from **Fetch WooCommerce Orders**
  - Output (matched items): to **Normalize Order Data**
- **Version notes:** typeVersion `2.2`.
- **Edge cases / failures:**
  - If your store uses other “paid” statuses (`processing`, `on-hold`, custom statuses), they will be excluded.
  - If the HTTP node returns a non-array structure (error payload), filter may behave unexpectedly.

#### Node: Normalize Order Data
- **Type / role:** `Set` — normalizes the order schema.
- **Configuration choices (fields created):**
  - `order_id` = `{{ $json.id }}` (string)
  - `order_date` = `{{ $json.date_created }}` (string)
  - `order_total` = `{{ $json.total }}` (string as stored here)
  - `line_items` = `{{ $json.line_items }}` (array)
- **Inputs/outputs:**
  - Input: from **Filter Completed Orders**
  - Output: to **Combine Orders & Refunds** (merge input index 0)
- **Version notes:** typeVersion `3.4`.
- **Edge cases / failures:**
  - `order_total` is stored as string; downstream code converts with `Number()`. Non-numeric formats could convert to `NaN` (mitigated partially by `|| 0`, but `Number('NaN')` is `NaN` and will propagate in sums).
  - If `line_items` is missing, set will output `undefined` for that field (usually fine since it’s unused in KPI math).

**Sticky note coverage (content applies to this block’s nodes):**  
“## Order Data Collection & Preparation … Only completed or paid orders are considered … data is cleaned and structured …”

---

### Block 3 — Refund Data Collection & Preparation
**Overview:** Pulls refund records from WooCommerce and normalizes key fields so they can be deduplicated and aggregated.

**Nodes involved**
- Fetch WooCommerce Refunds
- Normalize Refund Data

#### Node: Fetch WooCommerce Refunds
- **Type / role:** `HTTP Request` — calls WooCommerce REST API to list refunds.
- **Configuration choices:**
  - **URL:** `https://{{ $('Configure WooCommerce Store').item.json.wc_domain }}/wp-json/wc/v3/refunds`
    - Note: uses a **node reference** to “Configure WooCommerce Store” rather than `$json.wc_domain`.
  - **Authentication:** Generic credential type → **HTTP Basic Auth**
  - **Credentials used:** “Woo-Commerece Auth”
- **Inputs/outputs:**
  - Input: from **Configure WooCommerce Store**
  - Output: to **Normalize Refund Data**
- **Version notes:** typeVersion `4.3`.
- **Edge cases / failures:**
  - Same auth/pagination risks as orders endpoint; refunds are also typically paginated.
  - The expression uses `$('Configure WooCommerce Store').item.json.wc_domain`. If the workflow ever runs with multiple items, `.item` selection can become ambiguous; here it’s safe because the trigger produces a single item.

#### Node: Normalize Refund Data
- **Type / role:** `Set` — normalizes refund schema.
- **Configuration choices (fields created):**
  - `refund_id` = `{{ $json.id }}` (string)
  - `order_id` = `{{ $json.parent_id }}` (string)
  - `refund_amount` = `{{ $json.amount }}` (number)
- **Inputs/outputs:**
  - Input: from **Fetch WooCommerce Refunds**
  - Output: to **Combine Orders & Refunds** (merge input index 1)
- **Version notes:** typeVersion `3.4`.
- **Edge cases / failures:**
  - Refund `amount` may arrive as string in some WooCommerce versions/plugins; the node forces number via expression evaluation, but malformed values could become `NaN`.

**Sticky note coverage (content applies to this block’s nodes):**  
“## Refund Data Collection & Preparation … retrieves refund information … cleaned and standardized …”

---

### Block 4 — KPI Calculation & Executive Reporting
**Overview:** Combines order and refund streams, calculates weekly finance KPIs and risk flags, then posts a formatted digest to Slack.

**Nodes involved**
- Combine Orders & Refunds
- Calculate Finance KPIs
- Send Weekly KPI Digest to Slack

#### Node: Combine Orders & Refunds
- **Type / role:** `Merge` — combines two inputs into a single node for downstream computation.
- **Configuration choices:** Default merge behavior (two input streams). It’s used here primarily to make both datasets available to the Code node via `$input.all(0)` and `$input.all(1)`.
- **Inputs/outputs:**
  - Input 0: from **Normalize Order Data**
  - Input 1: from **Normalize Refund Data**
  - Output: to **Calculate Finance KPIs**
- **Version notes:** typeVersion `3.2`.
- **Edge cases / failures:**
  - If one branch returns **zero items**, the merge still executes but downstream code must handle empty arrays (it does).
  - If one branch errors, merge never runs.

#### Node: Calculate Finance KPIs
- **Type / role:** `Code` — computes totals, ratios, deduplicates refunds, and generates a single CFO summary object.
- **Configuration choices (logic):**
  - Reads:
    - `orders = $input.all(0).map(i => i.json)`
    - `refundsRaw = $input.all(1).map(i => i.json)`
  - **Sales KPIs:**
    - `totalSalesAmount` = sum of `Number(order.order_total || 0)`
    - `totalOrderCount` = `orders.length`
  - **Refund deduplication:**
    - Uses `Map()` keyed by `refund_id` to remove duplicates
  - **Refund KPIs:**
    - `totalRefundAmount` = sum of `Number(refund.refund_amount || 0)`
    - `totalRefundCount` = `refunds.length`
  - **Ratios (safe divide):**
    - `refund_ratio_amount_pct` = `(refund/sales)*100` if sales>0 else 0 (formatted with `toFixed(2)` when computed)
    - `refund_ratio_count_pct` = `(refundCount/orderCount)*100` if orderCount>0 else 0
  - **Risk flags:**
    - If refund ratio amount > 10% → “High refund ratio (>10%)”
    - If refund amount exceeds sales → “Refund amount exceeds sales”
    - If sales 0 and refunds >0 → “Refunds exist with zero sales”
  - Output: returns **one item** with fields:
    - `total_sales_amount`, `total_order_count`, `total_refund_amount`, `total_refund_count`,
      `refund_ratio_amount_pct`, `refund_ratio_count_pct`, `flags`, `period_generated_at`
- **Inputs/outputs:**
  - Input: from **Combine Orders & Refunds**
  - Output: to **Send Weekly KPI Digest to Slack**
- **Version notes:** typeVersion `2`.
- **Edge cases / failures:**
  - `refundRatioAmount` is a string when computed with `toFixed(2)`; comparison `refundRatioAmount > 10` relies on JS coercion (usually works, but better to wrap with `Number()` to be explicit).
  - If any `Number(...)` produces `NaN`, totals can become `NaN` and ruin the report (consider sanitizing with `isFinite` checks).
  - This computes KPIs across **all returned orders/refunds**, not “last 7 days” (no date filtering exists).

#### Node: Send Weekly KPI Digest to Slack
- **Type / role:** `Slack` — posts the computed KPI summary to a channel.
- **Configuration choices:**
  - Operation: Send message to a **channel** (selected by ID)
  - Channel: `C09S57E2JQ2` (cached name: `n8n`)
  - Message text is an expression pulling from `$json.*`:
    - Includes `period_generated_at`, totals, ratios, and `flags`
- **Inputs/outputs:**
  - Input: from **Calculate Finance KPIs**
  - Output: none (terminal)
- **Credentials:** Slack API credential “Slack account 27”
- **Version notes:** typeVersion `2.3`.
- **Edge cases / failures:**
  - Slack auth/token revoked → `invalid_auth`.
  - Bot not in channel → Slack API error (must invite the app/bot).
  - `{{ $json.flags }}` will render as an array (e.g., `High...,...`) rather than a nicely formatted bullet list; consider joining with `\n• ` for readability.

**Sticky note coverage (content applies to this block’s nodes):**  
“## KPI Calculation & Executive Reporting … calculates key financial KPIs … automatically sent to Slack …”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Weekly KPI Scheduler | Schedule Trigger | Weekly workflow entry point | — | Configure WooCommerce Store | ## Schedule & Store Setup<br>This section controls when the workflow runs and which WooCommerce store it connects to. The scheduler triggers the automation on a weekly basis, while the store configuration ensures all API requests point to the correct WooCommerce domain. |
| Configure WooCommerce Store | Set | Define `wc_domain` configuration | Weekly KPI Scheduler | Fetch WooCommerce Orders; Fetch WooCommerce Refunds | ## Schedule & Store Setup<br>This section controls when the workflow runs and which WooCommerce store it connects to. The scheduler triggers the automation on a weekly basis, while the store configuration ensures all API requests point to the correct WooCommerce domain. |
| Fetch WooCommerce Orders | HTTP Request | Pull orders from WooCommerce REST API | Configure WooCommerce Store | Filter Completed Orders | ## Order Data Collection & Preparation<br>This section fetches order data from WooCommerce and prepares it for financial analysis. Only completed or paid orders are considered as valid sales. The data is cleaned and structured so totals, dates and line items can be used reliably for KPI calculations. |
| Filter Completed Orders | Filter | Keep only `status=completed` orders | Fetch WooCommerce Orders | Normalize Order Data | ## Order Data Collection & Preparation<br>This section fetches order data from WooCommerce and prepares it for financial analysis. Only completed or paid orders are considered as valid sales. The data is cleaned and structured so totals, dates and line items can be used reliably for KPI calculations. |
| Normalize Order Data | Set | Normalize order fields for KPI logic | Filter Completed Orders | Combine Orders & Refunds | ## Order Data Collection & Preparation<br>This section fetches order data from WooCommerce and prepares it for financial analysis. Only completed or paid orders are considered as valid sales. The data is cleaned and structured so totals, dates and line items can be used reliably for KPI calculations. |
| Fetch WooCommerce Refunds | HTTP Request | Pull refunds from WooCommerce REST API | Configure WooCommerce Store | Normalize Refund Data | ## Refund Data Collection & Preparation<br>This section retrieves refund information from WooCommerce and formats it for analysis. Refund records are cleaned and standardized so refund amounts and counts can be accurately calculated and compared against sales data in later steps. |
| Normalize Refund Data | Set | Normalize refund fields for KPI logic | Fetch WooCommerce Refunds | Combine Orders & Refunds | ## Refund Data Collection & Preparation<br>This section retrieves refund information from WooCommerce and formats it for analysis. Refund records are cleaned and standardized so refund amounts and counts can be accurately calculated and compared against sales data in later steps. |
| Combine Orders & Refunds | Merge | Provide both datasets to KPI computation | Normalize Order Data; Normalize Refund Data | Calculate Finance KPIs | ## KPI Calculation & Executive Reporting<br>This final section combines sales and refund data to calculate key financial KPIs such as refund ratios and risk indicators. A concise, CFO-ready summary is then generated and automatically sent to Slack, providing leadership with clear weekly insights. |
| Calculate Finance KPIs | Code | Compute totals, ratios, flags | Combine Orders & Refunds | Send Weekly KPI Digest to Slack | ## KPI Calculation & Executive Reporting<br>This final section combines sales and refund data to calculate key financial KPIs such as refund ratios and risk indicators. A concise, CFO-ready summary is then generated and automatically sent to Slack, providing leadership with clear weekly insights. |
| Send Weekly KPI Digest to Slack | Slack | Post KPI digest message to Slack channel | Calculate Finance KPIs | — | ## KPI Calculation & Executive Reporting<br>This final section combines sales and refund data to calculate key financial KPIs such as refund ratios and risk indicators. A concise, CFO-ready summary is then generated and automatically sent to Slack, providing leadership with clear weekly insights. |
| Sticky Note | Sticky Note | Comment block (schedule/store) | — | — | ## Schedule & Store Setup<br>This section controls when the workflow runs and which WooCommerce store it connects to. The scheduler triggers the automation on a weekly basis, while the store configuration ensures all API requests point to the correct WooCommerce domain. |
| Sticky Note1 | Sticky Note | Comment block (orders) | — | — | ## Order Data Collection & Preparation<br>This section fetches order data from WooCommerce and prepares it for financial analysis. Only completed or paid orders are considered as valid sales. The data is cleaned and structured so totals, dates and line items can be used reliably for KPI calculations. |
| Sticky Note2 | Sticky Note | Comment block (refunds) | — | — | ## Refund Data Collection & Preparation<br>This section retrieves refund information from WooCommerce and formats it for analysis. Refund records are cleaned and standardized so refund amounts and counts can be accurately calculated and compared against sales data in later steps. |
| Sticky Note3 | Sticky Note | Comment block (KPIs/reporting) | — | — | ## KPI Calculation & Executive Reporting<br>This final section combines sales and refund data to calculate key financial KPIs such as refund ratios and risk indicators. A concise, CFO-ready summary is then generated and automatically sent to Slack, providing leadership with clear weekly insights. |
| Sticky Note4 | Sticky Note | Global explanation and setup notes | — | — | ## How It Works<br>The workflow runs automatically on a weekly schedule using a Cron trigger.<br><br>It connects to your WooCommerce store and fetches all completed orders within the selected time period.<br><br>Order data is cleaned and normalized to extract only relevant finance fields such as order total and order count.<br><br>In parallel, the workflow fetches refund data from WooCommerce and removes duplicate refund records.<br><br>Both datasets are then combined to calculate key finance KPIs like total sales, total refunds and refund ratios.<br><br>Risk indicators are added automatically when refund values cross predefined thresholds.<br><br>Finally, a clear weekly KPI summary is generated and sent to Slack for easy review by finance and leadership teams.<br><br>## Setup Steps<br><br>### Cron Schedule:<br>Configure the Weekly KPI Scheduler node to define when the report should run (for example, every Monday morning).<br><br>### WooCommerce Domain:<br>Set your store URL in the Configure WooCommerce Store node so all API requests target the correct site.<br><br>### WooCommerce API Access:<br>Add your WooCommerce Consumer Key and Consumer Secret to the Fetch Orders and Fetch Refunds HTTP Request nodes using authentication.<br><br>### Data Processing Nodes:<br>No configuration is required for the filter and normalize nodes unless you want to customize fields or calculations.<br><br>### Slack Integration:<br>Connect your Slack account in the Send Weekly KPI Digest to Slack node and select the channel where reports should be posted.<br><br>### Activate Workflow:<br>Enable the workflow to start receiving automated weekly WooCommerce sales and refund KPI updates. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Trigger**
   1) Add node: **Schedule Trigger**  
   2) Set rule: **Interval = Weeks**, **Trigger day = Monday (1)**, **Trigger hour = 10**.

2. **Add WooCommerce domain configuration**
   1) Add node: **Set** named **Configure WooCommerce Store**  
   2) Add field:
      - `wc_domain` (string) = `your-store-domain.com`  
      (Do not include `https://` unless you also update the HTTP URL templates accordingly.)

3. **Create WooCommerce credentials (for HTTP Basic Auth)**
   1) In n8n Credentials, create **HTTP Basic Auth** credential (name it e.g., “Woo-Commerece Auth”).  
   2) Set:
      - **Username** = WooCommerce **Consumer Key**
      - **Password** = WooCommerce **Consumer Secret**
   3) Ensure WooCommerce REST API keys have appropriate read permissions.

4. **Fetch orders**
   1) Add node: **HTTP Request** named **Fetch WooCommerce Orders**  
   2) Method: `GET`  
   3) URL: `https://{{$json.wc_domain}}/wp-json/wc/v3/orders`  
   4) Authentication: **Generic credential type → HTTP Basic Auth**, select “Woo-Commerece Auth”.

5. **Filter completed orders**
   1) Add node: **Filter** named **Filter Completed Orders**  
   2) Condition: `{{ $json.status }}` **equals** `completed`.

6. **Normalize order data**
   1) Add node: **Set** named **Normalize Order Data**  
   2) Add fields:
      - `order_id` (string) = `{{ $json.id }}`
      - `order_date` (string) = `{{ $json.date_created }}`
      - `order_total` (string) = `{{ $json.total }}`
      - `line_items` (array) = `{{ $json.line_items }}`

7. **Fetch refunds**
   1) Add node: **HTTP Request** named **Fetch WooCommerce Refunds**  
   2) Method: `GET`  
   3) URL: `https://{{ $('Configure WooCommerce Store').item.json.wc_domain }}/wp-json/wc/v3/refunds`  
   4) Authentication: **HTTP Basic Auth**, select “Woo-Commerece Auth”.

8. **Normalize refund data**
   1) Add node: **Set** named **Normalize Refund Data**  
   2) Add fields:
      - `refund_id` (string) = `{{ $json.id }}`
      - `order_id` (string) = `{{ $json.parent_id }}`
      - `refund_amount` (number) = `{{ $json.amount }}`

9. **Merge the two streams**
   1) Add node: **Merge** named **Combine Orders & Refunds**  
   2) Connect:
      - **Normalize Order Data → Merge (Input 0)**
      - **Normalize Refund Data → Merge (Input 1)**

10. **Compute KPIs**
   1) Add node: **Code** named **Calculate Finance KPIs**  
   2) Paste logic equivalent to:
      - Sum order totals
      - Deduplicate refunds by `refund_id`
      - Sum refund amounts
      - Compute ratios with safe divide
      - Create `flags` array
      - Output one JSON object with KPI fields and `period_generated_at`

11. **Configure Slack credentials**
   1) Create/select **Slack API** credentials (OAuth or bot token, depending on your n8n setup).  
   2) Ensure the Slack app/bot is a member of the destination channel.

12. **Send Slack digest**
   1) Add node: **Slack** named **Send Weekly KPI Digest to Slack**  
   2) Action: send message to **channel**  
   3) Select channel (e.g., `#n8n`)  
   4) Message text: reference fields from the Code node, including totals, ratios, and flags.

13. **Connect final nodes**
   - **Merge → Code → Slack**
   - Ensure both HTTP branches start from **Configure WooCommerce Store**.

14. **Activate**
   - Save and toggle the workflow to **Active**.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “How It Works” + “Setup Steps” content describing schedule, domain setup, WooCommerce API keys, Slack integration, and activation | From the workflow’s global sticky note (Sticky Note4) |
| Important functional limitation: no explicit date range filtering or pagination is configured for orders/refunds endpoints | Implicit from the two HTTP Request node configurations; consider adding `after/before` (or `date_created_*`) parameters and pagination handling for accurate weekly reporting |