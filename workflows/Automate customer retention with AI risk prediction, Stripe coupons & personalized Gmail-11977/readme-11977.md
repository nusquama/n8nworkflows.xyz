Automate customer retention with AI risk prediction, Stripe coupons & personalized Gmail

https://n8nworkflows.xyz/workflows/automate-customer-retention-with-ai-risk-prediction--stripe-coupons---personalized-gmail-11977


# Automate customer retention with AI risk prediction, Stripe coupons & personalized Gmail

## 1. Workflow Overview

**Workflow name (in JSON):** Predict churn risk from customer data and send retention emails via OpenAI  
**Title provided:** Automate customer retention with AI risk prediction, Stripe coupons & personalized Gmail

This workflow is an automated customer retention system that runs on a schedule, aggregates customer signals (CRM + support + product usage), predicts churn risk with OpenAI, and—when risk is high—creates a Stripe coupon and sends a personalized retention email via Gmail. It also logs actions and later updates campaign engagement (via SendGrid) and 30‑day retention outcomes back into Google Sheets.

### 1.1 Scheduled Start
- Runs periodically via Schedule Trigger.

### 1.2 Data Aggregation (CRM + Support + Product Usage)
- Pulls a list of active customers from a CRM.
- Fetches each customer’s support tickets, computes support metrics.
- Queries Postgres product events to compute usage metrics.
- Merges these into one “360° customer profile” per customer.

### 1.3 AI Risk Prediction
- Sends the merged profile to OpenAI to obtain a churn risk score.
- Filters customers with risk score ≥ 0.7.

### 1.4 Retention Action (Offer + Email + Logging)
- Generates a Stripe coupon with discount dependent on MRR.
- Chooses an offer type/value based on heuristics (MRR, open tickets, login days).
- Uses OpenAI to generate a personalized email.
- Sends the email via Gmail and logs the attempt into Google Sheets.

### 1.5 Tracking & Feedback Loop (Engagement + 30-Day Outcome)
- Reads “not yet updated” rows from Google Sheets.
- Queries SendGrid for open/click/bounce signals and updates the sheet.
- Reads rows that are 30 days old, checks CRM status, and writes final retention/churn outcomes back to the sheet.

---

## 2. Block-by-Block Analysis

### Block 0 — Documentation / On-canvas notes
**Overview:** Sticky notes describe intent, setup, and the main four sections. They do not execute.  
**Nodes involved:** `Workflow Overview`, `Note - Schedule`, `Sticky Note`, `Sticky Note1`, `Sticky Note2`

#### Node: Workflow Overview
- **Type / role:** Sticky Note; describes workflow purpose and setup checklist.
- **Key content:** Mentions credentials (OpenAI, Stripe, Gmail, SendGrid, Postgres, Google Sheets), sheet columns, updating URLs and SQL query, and enabling schedule.
- **Connections:** None.
- **Failure modes:** None (non-executing).

#### Node: Note - Schedule
- **Type / role:** Sticky Note; describes “Data Aggregation” and suggests adjusting SQL query.
- **Connections:** None.

#### Node: Sticky Note
- **Type / role:** Sticky Note; explains AI churn risk scoring and risk threshold (> 0.7).
- **Connections:** None.

#### Node: Sticky Note1
- **Type / role:** Sticky Note; explains Stripe coupon + OpenAI email + Gmail send + Google Sheets logging.
- **Connections:** None.

#### Node: Sticky Note2
- **Type / role:** Sticky Note; explains tracking via SendGrid and 30‑day outcome verification in CRM, then syncing to Sheets.
- **Connections:** None.

---

### Block 1 — Scheduled start + customer list (CRM)
**Overview:** Triggers the workflow periodically and fetches the target active customers from the CRM.  
**Nodes involved:** `Schedule Trigger - 毎日03:00実行`, `対象顧客リスト取得 (CRM)`

#### Node: Schedule Trigger - 毎日03:00実行
- **Type / role:** Schedule Trigger; entry point.
- **Config (interpreted):** Runs every 3 hours (`hoursInterval: 3`). Despite the name suggesting 03:00 daily, the actual rule is “every 3 hours”.
- **Outputs:** To `対象顧客リスト取得 (CRM)`.
- **Edge cases:** Timezone differences (n8n instance timezone), schedule mismatch with node name.

#### Node: 対象顧客リスト取得 (CRM)
- **Type / role:** HTTP Request; fetch active customers.
- **Config (interpreted):**
  - GET `https://api.your-crm.com/v1/customers`
  - Query params: `status=active`, `last_billing_days=30`
- **Inputs:** Trigger output (no fields required).
- **Outputs:** To `サポートチケット情報取得`.
- **Key data expectations:** Response should include at least `customer_id` per item; also likely includes `mrr` and email fields used later (not explicitly mapped in workflow).
- **Failure modes:**
  - Auth missing (node currently shows no predefined credential; CRM may require headers/token).
  - Rate limits/timeouts; pagination not handled (only a single call).
  - If CRM returns a list nested in a field (e.g., `data[]`), downstream nodes may not iterate as expected unless n8n auto-splits or a Split Out node is added.

---

### Block 2 — Support ticket enrichment + metric aggregation
**Overview:** Pulls support tickets per customer and calculates support metrics for the last 30 days.  
**Nodes involved:** `サポートチケット情報取得`, `サポート指標集計`

#### Node: サポートチケット情報取得
- **Type / role:** HTTP Request; fetch tickets for a single customer.
- **Config (interpreted):**
  - URL built via expression:  
    `https://api.support-tool.com/v1/tickets?customer_id={{$json.customer_id}}`
- **Inputs:** Each item must have `customer_id`.
- **Outputs:** To `サポート指標集計`.
- **Failure modes:**
  - Missing/invalid `customer_id` causes invalid URL.
  - Auth not configured (node shows none).
  - API schema mismatch: expects `item.json.tickets` array downstream.

#### Node: サポート指標集計
- **Type / role:** Code node; computes support KPIs.
- **Config (interpreted):**
  - Computes:
    - `ticket_count_last_30d` (tickets created within last 30 days)
    - `open_ticket_count`
    - `avg_csatscore`
    - `avg_first_response_time`
  - Uses `item.json.tickets || []`
- **Outputs / connections:**
  - Output **0** → `製品利用ログ取得`
  - Output **1** (Merge input 2) → `データ統合 (CRM+サポート)` (connected to Merge index 1)
- **Edge cases:**
  - Ticket objects missing `created_at`, `status`, `csat_score`, `first_response_hours` → computations may degrade but won’t crash unless date parsing fails unexpectedly.
  - Time interpretation: `new Date(t.created_at)` depends on ISO format; non-ISO formats can yield invalid dates (NaN comparisons).

---

### Block 3 — Product usage enrichment (Postgres) + merges
**Overview:** Queries product usage logs for each customer and merges CRM+Support+Usage into a single record per customer.  
**Nodes involved:** `製品利用ログ取得`, `データ統合 (CRM+サポート)`, `データ統合 (全データ)`

#### Node: 製品利用ログ取得
- **Type / role:** Postgres node; executes SQL usage aggregation.
- **Config (interpreted):**
  - Aggregates last 30 days:
    - `login_days_last_30d` = count of distinct event days
    - `feature_a_usage_count`, `feature_b_usage_count`
    - `last_active_date`, `session_avg_duration`
  - WHERE clause uses:  
    `customer_id IN ({{ $json.customer_id }})`
- **Inputs:** Expects `customer_id` from upstream item.
- **Outputs:** To `データ統合 (全データ)` (connected to Merge index 1).
- **Version-specific:** Postgres node v2.5.
- **Failure modes / important edge cases:**
  - **SQL injection / invalid SQL:** `IN ({{ $json.customer_id }})` is unsafe if `customer_id` is not numeric or not properly quoted. If `customer_id` is a string (UUID), it must be quoted: `IN ('{{$json.customer_id}}')`.
  - **Incorrect “IN” usage for single value:** If a single scalar is used, `customer_id = ...` is safer.
  - If Postgres returns 0 rows for a customer, the merge logic later may produce incomplete records.

#### Node: データ統合 (CRM+サポート)
- **Type / role:** Merge node; combines CRM baseline with support metrics.
- **Config (interpreted):** Mode `combine` (pairs items by position, not by key).
- **Inputs / connections:**
  - Input 0: from `対象顧客リスト取得 (CRM)`
  - Input 1: from `サポート指標集計` (second connection)
- **Outputs:** To `データ統合 (全データ)`.
- **Edge cases:**
  - **Item alignment risk:** “Combine” depends on both inputs having the same number of items in the same order. If the support API fails for one customer, order/length mismatch can corrupt merges.
  - If CRM response is a single array item rather than individual items, combine will not behave as intended.

#### Node: データ統合 (全データ)
- **Type / role:** Merge node; combines (CRM+support) with Postgres usage.
- **Config (interpreted):** Mode `combine` (again, positional pairing).
- **Inputs / connections:**
  - Input 0: from `データ統合 (CRM+サポート)`
  - Input 1: from `製品利用ログ取得`
- **Outputs:** To `AI解約リスク予測`.
- **Edge cases:** Same positional pairing risks as above; additionally, Postgres may return multiple rows per customer if grouping is wrong, causing duplication/misalignment.

---

### Block 4 — AI churn risk prediction + gating
**Overview:** Uses OpenAI to compute churn risk score and filters to only high-risk customers.  
**Nodes involved:** `AI解約リスク予測`, `高リスク顧客フィルタ`

#### Node: AI解約リスク予測
- **Type / role:** OpenAI node (Chat Completion); produces `churn_risk_score` (expected).
- **Config (interpreted):**
  - Resource: `chatCompletion`
  - **Missing in JSON:** Prompt/messages are not defined in this export. To work, you must configure model + system/user messages, and ensure output is parsed into a numeric field such as `churn_risk_score`.
- **Inputs:** The merged customer profile.
- **Outputs:** To `高リスク顧客フィルタ`.
- **Failure modes:**
  - Missing OpenAI credentials.
  - Model/prompt not configured → output won’t include `churn_risk_score`.
  - If OpenAI returns text and not structured JSON, the IF node numeric comparison may fail or evaluate unexpectedly.

#### Node: 高リスク顧客フィルタ
- **Type / role:** IF node; route by churn risk.
- **Config (interpreted):**
  - Condition: `{{$json.churn_risk_score}} >= 0.7`
- **Outputs / connections:**
  - **True** → `特別オファー生成` (retention action)
  - **False** → `未更新ログ取得` (tracking path)
- **Edge cases:**
  - `churn_risk_score` missing/null/string → loose validation may coerce unexpectedly; ensure it is a number.
  - Business logic oddity: low-risk customers trigger the tracking branch, which then expects SendGrid message IDs from Google Sheets; typically tracking should be scheduled independently or run for previously-sent emails, not for low-risk customers.

---

### Block 5 — Retention action (Stripe coupon → offer decision → AI email → Gmail → log)
**Overview:** For high-risk customers, creates a coupon, determines offer type/value, drafts a personalized email, sends it, and logs the attempt in Google Sheets.  
**Nodes involved:** `特別オファー生成`, `オファー内容決定`, `リテンションメール生成`, `リテンションメール送信`, `施策ログ保存`

#### Node: 特別オファー生成
- **Type / role:** HTTP Request; creates a Stripe coupon.
- **Config (interpreted):**
  - POST `https://api.stripe.com/v1/coupons`
  - Auth: predefined credential type `stripeApi`
  - Body params:
    - `percent_off` = `{{$json.mrr > 10000 ? 20 : 10}}`
    - `duration=repeating`
    - `duration_in_months=3`
    - `id=RETAIN_{{$json.customer_id}}_{{Date.now()}}`
- **Inputs:** Needs `customer_id` and `mrr`.
- **Outputs:** To `オファー内容決定`.
- **Failure modes / edge cases:**
  - Stripe API key missing/invalid.
  - Stripe coupon `id` uniqueness: using `Date.now()` helps, but collisions still possible in parallel executions; also Stripe has ID format constraints.
  - Some Stripe accounts restrict coupon creation or require additional fields; errors return 4xx.

#### Node: オファー内容決定
- **Type / role:** Code node; sets offer strategy fields.
- **Config (interpreted):**
  - Uses `mrr`, `open_ticket_count`, `login_days_last_30d` to set:
    - `offer_type`
    - `offer_value`
    - `offer_expire_at` = now + 14 days
- **Outputs:** To `リテンションメール生成`.
- **Edge cases:** If upstream merges didn’t supply those fields, defaults apply; offer may be less tailored than intended.

#### Node: リテンションメール生成
- **Type / role:** OpenAI Chat Completion; drafts email subject + HTML body.
- **Config (interpreted):**
  - Resource: `chatCompletion`
  - **Missing in JSON:** prompts/messages and output mapping. Downstream expects:
    - `email_subject`
    - `email_body_html`
- **Outputs:** To `リテンションメール送信`.
- **Failure modes:** Same as prior OpenAI node; additionally HTML output may need sanitization/encoding.

#### Node: リテンションメール送信
- **Type / role:** Gmail node; sends the email.
- **Config (interpreted):**
  - Subject: `{{$json.email_subject}}`
  - Message (body): `{{$json.email_body_html}}`
  - Options: `bccList=user@example.com`, `replyTo=user@example.com`
- **Inputs:** Must have recipient email address—however, **the “to” field is not shown/configured in this JSON**. In Gmail node, missing recipient typically fails.
- **Outputs:** To `施策ログ保存`.
- **Failure modes / edge cases:**
  - Gmail OAuth2 not configured/expired.
  - Missing `to` recipient.
  - Sending limits, spam policies, HTML formatting issues.
  - The workflow later queries SendGrid for message events, but Gmail does not natively produce SendGrid `msg_id`; this mismatch must be resolved (see Tracking block).

#### Node: 施策ログ保存
- **Type / role:** Google Sheets; appends a log row.
- **Config (interpreted):** Operation `appendPage` to a document (documentId not set in export).
- **Inputs:** Should include columns like `customer_id`, `risk_score`, `offer_type`, `email_status`, `email_provider_message_id`, timestamps, etc. (as described in sticky note).
- **Outputs:** Not connected further in the retention branch.
- **Failure modes:**
  - Google Sheets credentials missing.
  - `documentId` not selected.
  - Column mapping not defined; appended data may be empty unless configured.

---

### Block 6 — Tracking & feedback loop (SendGrid engagement → Sheets update → 30-day CRM status)
**Overview:** Reads pending log rows, updates email engagement from SendGrid, then checks 30-day retention outcomes in CRM and updates Sheets.  
**Nodes involved:** `未更新ログ取得`, `メール開封状況取得`, `開封データ処理`, `効果測定データ更新`, `30日経過レコード取得`, `顧客ステータス確認`, `継続状況判定`, `最終結果更新`

#### Node: 未更新ログ取得
- **Type / role:** Google Sheets; fetches rows to update.
- **Config (interpreted):** Operation not shown in export (likely “Read” / “Get many”). `documentId` and `sheetName` are unset.
- **Inputs:** Triggered from IF **False** branch (low-risk), which is logically unrelated; typically this should run on its own schedule or after logging.
- **Outputs:** To `メール開封状況取得`.
- **Failure modes:** Missing sheet selection/credentials; filtering criteria for “未更新” not defined (so it may fetch everything).

#### Node: メール開封状況取得
- **Type / role:** HTTP Request; fetch SendGrid message events.
- **Config (interpreted):**
  - GET `https://api.sendgrid.com/v3/messages?msg_id={{$json.email_provider_message_id}}`
  - Auth: predefined credential `sendGridApi`
- **Inputs:** Must have `email_provider_message_id`.
- **Outputs:** To `開封データ処理`.
- **Edge cases / integration concern:**
  - If emails are sent via Gmail, you won’t have SendGrid `msg_id`. Either:
    - Send emails via SendGrid instead of Gmail, or
    - Implement tracking differently (e.g., Gmail + pixel, or use a provider that exposes message IDs/events).

#### Node: 開封データ処理
- **Type / role:** Code node; normalizes engagement events.
- **Config (interpreted):**
  - From `events` array:
    - `email_opened` true if any `event === 'open'`
    - `email_clicked` true if any `event === 'click'`
    - `delivery_status` = `failed` if any `bounce`, else `delivered`
    - `metrics_status = 'updated'`
- **Outputs:** To `効果測定データ更新`.
- **Failure modes:** If SendGrid schema differs (no `events`), fields become defaults (false/delivered).

#### Node: 効果測定データ更新
- **Type / role:** Google Sheets; updates rows with engagement metrics.
- **Config (interpreted):** Operation `update` (requires row identification / key column mapping).
- **Outputs:** To `30日経過レコード取得`.
- **Failure modes:** Without specifying row ID / lookup, updates may fail or overwrite wrong rows.

#### Node: 30日経過レコード取得
- **Type / role:** Google Sheets; fetches rows eligible for 30‑day outcome checking.
- **Config (interpreted):** Operation not shown; needs a filter like “sent_at <= today-30d and followup_status != done”.
- **Outputs:** To `顧客ステータス確認`.
- **Failure modes:** Same as other sheets read; missing filters means excessive CRM calls.

#### Node: 顧客ステータス確認
- **Type / role:** HTTP Request; fetch customer status from CRM.
- **Config (interpreted):**
  - GET `https://api.your-crm.com/v1/customers/{{$json.customer_id}}/status`
- **Inputs:** Requires `customer_id`.
- **Outputs:** To `継続状況判定`.
- **Failure modes:** Auth/pagination; mismatched status field names.

#### Node: 継続状況判定
- **Type / role:** Code node; classifies churn/retention/upgrade outcome.
- **Config (interpreted):**
  - Reads:
    - `status`
    - `current_mrr` (from CRM status API result)
    - `mrr` (original MRR stored in sheet/log)
  - Writes:
    - `churned`, `retained`, `upgraded`
    - `followup_status = 'done'`
- **Outputs:** To `最終結果更新`.
- **Edge cases:** If `current_mrr` not returned, upgrades won’t be detected.

#### Node: 最終結果更新
- **Type / role:** Google Sheets; updates final outcome fields.
- **Config (interpreted):** Operation `update`.
- **Failure modes:** Same update-key requirements as prior update node.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Workflow Overview | Sticky Note | On-canvas documentation | — | — | # 📉 Churn Risk Prediction & Retention Automation / Setup steps & sheet columns |
| Schedule Trigger - 毎日03:00実行 | Schedule Trigger | Periodic workflow start | — | 対象顧客リスト取得 (CRM) | ## 1️⃣ Data Aggregation … Tip: Adjust the SQL query… |
| 対象顧客リスト取得 (CRM) | HTTP Request | Fetch active customers from CRM | Schedule Trigger - 毎日03:00実行 | サポートチケット情報取得 | ## 1️⃣ Data Aggregation … |
| サポートチケット情報取得 | HTTP Request | Fetch support tickets per customer | 対象顧客リスト取得 (CRM) | サポート指標集計 | ## 1️⃣ Data Aggregation … |
| サポート指標集計 | Code | Compute ticket KPIs | サポートチケット情報取得 | 製品利用ログ取得; データ統合 (CRM+サポート) | ## 1️⃣ Data Aggregation … |
| 製品利用ログ取得 | Postgres | Aggregate product usage metrics | サポート指標集計 | データ統合 (全データ) | ## 1️⃣ Data Aggregation … |
| データ統合 (CRM+サポート) | Merge | Combine CRM + support metrics | 対象顧客リスト取得 (CRM); サポート指標集計 | データ統合 (全データ) | ## 1️⃣ Data Aggregation … |
| データ統合 (全データ) | Merge | Combine (CRM+support) + usage | データ統合 (CRM+サポート); 製品利用ログ取得 | AI解約リスク予測 | ## 1️⃣ Data Aggregation … |
| AI解約リスク予測 | OpenAI (Chat) | Predict churn risk score | データ統合 (全データ) | 高リスク顧客フィルタ | ## 2️⃣ AI Risk Analysis … Filter > 0.7 |
| 高リスク顧客フィルタ | IF | Route high-risk customers | AI解約リスク予測 | 特別オファー生成 (true); 未更新ログ取得 (false) | ## 2️⃣ AI Risk Analysis … |
| 特別オファー生成 | HTTP Request | Create Stripe coupon | 高リスク顧客フィルタ (true) | オファー内容決定 | ## 3️⃣ Personalized Retention Action … |
| オファー内容決定 | Code | Choose offer type/value | 特別オファー生成 | リテンションメール生成 | ## 3️⃣ Personalized Retention Action … |
| リテンションメール生成 | OpenAI (Chat) | Draft personalized email | オファー内容決定 | リテンションメール送信 | ## 3️⃣ Personalized Retention Action … |
| リテンションメール送信 | Gmail | Send retention email | リテンションメール生成 | 施策ログ保存 | ## 3️⃣ Personalized Retention Action … |
| 施策ログ保存 | Google Sheets | Append log row | リテンションメール送信 | — | ## 3️⃣ Personalized Retention Action … |
| 未更新ログ取得 | Google Sheets | Fetch rows pending metrics update | 高リスク顧客フィルタ (false) | メール開封状況取得 | ## 4️⃣ Campaign Tracking & Feedback Loop … |
| メール開封状況取得 | HTTP Request | Get SendGrid message events | 未更新ログ取得 | 開封データ処理 | ## 4️⃣ Campaign Tracking & Feedback Loop … |
| 開封データ処理 | Code | Normalize open/click/bounce | メール開封状況取得 | 効果測定データ更新 | ## 4️⃣ Campaign Tracking & Feedback Loop … |
| 効果測定データ更新 | Google Sheets | Update engagement metrics | 開封データ処理 | 30日経過レコード取得 | ## 4️⃣ Campaign Tracking & Feedback Loop … |
| 30日経過レコード取得 | Google Sheets | Fetch rows for 30-day check | 効果測定データ更新 | 顧客ステータス確認 | ## 4️⃣ Campaign Tracking & Feedback Loop … |
| 顧客ステータス確認 | HTTP Request | Check current CRM status | 30日経過レコード取得 | 継続状況判定 | ## 4️⃣ Campaign Tracking & Feedback Loop … |
| 継続状況判定 | Code | Determine churn/retained/upgraded | 顧客ステータス確認 | 最終結果更新 | ## 4️⃣ Campaign Tracking & Feedback Loop … |
| 最終結果更新 | Google Sheets | Update final outcome fields | 継続状況判定 | — | ## 4️⃣ Campaign Tracking & Feedback Loop … |
| Note - Schedule | Sticky Note | On-canvas explanation for data aggregation | — | — | ## 1️⃣ Data Aggregation … |
| Sticky Note | Sticky Note | On-canvas explanation for AI risk analysis | — | — | ## 2️⃣ AI Risk Analysis … |
| Sticky Note1 | Sticky Note | On-canvas explanation for retention action | — | — | ## 3️⃣ Personalized Retention Action … |
| Sticky Note2 | Sticky Note | On-canvas explanation for tracking loop | — | — | ## 4️⃣ Campaign Tracking & Feedback Loop … |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: *Predict churn risk from customer data and send retention emails via OpenAI* (or your preferred title).

2. **Add Sticky Notes (optional but matching the source)**
   - Add five Sticky Note nodes:
     - “Workflow Overview” with the provided content (purpose + setup).
     - “Note - Schedule” describing Data Aggregation.
     - “Sticky Note” describing AI Risk Analysis.
     - “Sticky Note1” describing Personalized Retention Action.
     - “Sticky Note2” describing Campaign Tracking & Feedback Loop.

3. **Add the trigger**
   - Add **Schedule Trigger** node named `Schedule Trigger - 毎日03:00実行`.
   - Configure: run **every 3 hours** (Hours interval = 3).  
     (If you truly want “daily at 03:00”, change the rule to a daily cron/time.)

4. **CRM customer list**
   - Add **HTTP Request** node `対象顧客リスト取得 (CRM)`.
   - Method: GET
   - URL: `https://api.your-crm.com/v1/customers`
   - Query params:
     - `status` = `active`
     - `last_billing_days` = `30`
   - Configure authentication (as required by your CRM): typically Header Auth or OAuth2.
   - Connect: **Schedule Trigger → 対象顧客リスト取得 (CRM)**.
   - Ensure output is one item per customer (use “Split Out” / “Item Lists” if CRM returns an array field).

5. **Support tickets per customer**
   - Add **HTTP Request** node `サポートチケット情報取得`.
   - URL (expression):  
     `={{ 'https://api.support-tool.com/v1/tickets?customer_id=' + $json.customer_id }}`
   - Configure auth for your support tool.
   - Connect: **対象顧客リスト取得 (CRM) → サポートチケット情報取得**.

6. **Compute support metrics**
   - Add **Code** node `サポート指標集計`.
   - Paste the JS aggregation code from the workflow (tickets last 30 days, open ticket count, averages).
   - Connect: **サポートチケット情報取得 → サポート指標集計**.

7. **Product usage query (Postgres)**
   - Add **Postgres** node `製品利用ログ取得`.
   - Credentials: configure Postgres connection.
   - Operation: Execute Query
   - Query: use the provided SQL, but **fix the customer_id filter**:
     - If `customer_id` is numeric: `WHERE customer_id = {{ $json.customer_id }}`
     - If `customer_id` is a UUID/string: `WHERE customer_id = '{{ $json.customer_id }}'`
   - Connect: **サポート指標集計 → 製品利用ログ取得**.

8. **Merge CRM + Support**
   - Add **Merge** node `データ統合 (CRM+サポート)`.
   - Mode: Combine.
   - Connect:
     - **対象顧客リスト取得 (CRM) → データ統合 (CRM+サポート)** (Input 1)
     - **サポート指標集計 → データ統合 (CRM+サポート)** (Input 2)
   - Consider switching to “Merge by Key” (if available in your n8n version) using `customer_id` to avoid positional misalignment.

9. **Merge with Usage**
   - Add **Merge** node `データ統合 (全データ)`.
   - Mode: Combine (or key-based merge by `customer_id`).
   - Connect:
     - **データ統合 (CRM+サポート) → データ統合 (全データ)** (Input 1)
     - **製品利用ログ取得 → データ統合 (全データ)** (Input 2)

10. **OpenAI churn risk prediction**
    - Add **OpenAI** node `AI解約リスク予測` (Chat Completion).
    - Credentials: OpenAI API key.
    - Configure messages so the model returns structured output (recommended):
      - Ask for JSON with a numeric `churn_risk_score` in `[0,1]` plus optional reasons.
    - Ensure the node output is mapped so that downstream sees `$json.churn_risk_score`.
    - Connect: **データ統合 (全データ) → AI解約リスク予測**.

11. **Risk filter**
    - Add **IF** node `高リスク顧客フィルタ`.
    - Condition: Number `{{$json.churn_risk_score}} >= 0.7`
    - Connect: **AI解約リスク予測 → 高リスク顧客フィルタ**.

12. **Stripe coupon creation (high-risk path)**
    - Add **HTTP Request** node `特別オファー生成`.
    - Method: POST
    - URL: `https://api.stripe.com/v1/coupons`
    - Auth: Predefined credential `stripeApi` (Stripe secret key).
    - Body parameters:
      - `percent_off`: `={{ $json.mrr > 10000 ? 20 : 10 }}`
      - `duration`: `repeating`
      - `duration_in_months`: `3`
      - `id`: `={{ 'RETAIN_' + $json.customer_id + '_' + Date.now() }}`
    - Connect: **高リスク顧客フィルタ (true) → 特別オファー生成**.

13. **Offer decision heuristics**
    - Add **Code** node `オファー内容決定`.
    - Paste the provided JS (sets `offer_type`, `offer_value`, `offer_expire_at`).
    - Connect: **特別オファー生成 → オファー内容決定**.

14. **OpenAI email drafting**
    - Add **OpenAI** node `リテンションメール生成` (Chat Completion).
    - Configure prompt to produce:
      - `email_subject`
      - `email_body_html`
      - (Optionally) plain text version.
    - Connect: **オファー内容決定 → リテンションメール生成**.

15. **Send email via Gmail**
    - Add **Gmail** node `リテンションメール送信`.
    - Credentials: Google OAuth2 for Gmail.
    - Set:
      - **To:** (must be configured; use something like `={{ $json.email }}` from CRM)
      - Subject: `={{ $json.email_subject }}`
      - Message: `={{ $json.email_body_html }}`
      - Options: BCC and Reply-To as desired.
    - Connect: **リテンションメール生成 → リテンションメール送信**.

16. **Append log to Google Sheets**
    - Add **Google Sheets** node `施策ログ保存`.
    - Credentials: Google Sheets OAuth2/service account.
    - Operation: Append (Append Page / Append Row depending on UI)
    - Select `documentId` and target sheet/tab.
    - Map fields (recommended minimum):
      - `customer_id`, `mrr`, `churn_risk_score`, `offer_type`, `offer_value`, `coupon_id`, `sent_at`, `email_status`
      - Tracking fields: `email_provider_message_id`, `metrics_status`, `followup_status`
    - Connect: **リテンションメール送信 → 施策ログ保存**.

17. **Tracking branch (as implemented)**
    - Add **Google Sheets** node `未更新ログ取得` to read rows where `metrics_status != updated`.
    - Connect: **高リスク顧客フィルタ (false) → 未更新ログ取得**.
    - Add **HTTP Request** node `メール開封状況取得`:
      - GET `={{ 'https://api.sendgrid.com/v3/messages?msg_id=' + $json.email_provider_message_id }}`
      - Auth: Predefined credential `sendGridApi`
    - Add **Code** node `開封データ処理` with provided JS.
    - Add **Google Sheets** node `効果測定データ更新` (Update) to write engagement fields back.
    - Chain connections:  
      **未更新ログ取得 → メール開封状況取得 → 開封データ処理 → 効果測定データ更新**

18. **30-day retention outcome**
    - Add **Google Sheets** node `30日経過レコード取得` to read rows eligible for follow-up (e.g., `sent_at <= now-30d` and `followup_status != done`).
    - Connect: **効果測定データ更新 → 30日経過レコード取得**.
    - Add **HTTP Request** node `顧客ステータス確認`:
      - GET `={{ 'https://api.your-crm.com/v1/customers/' + $json.customer_id + '/status' }}`
      - Configure CRM auth.
    - Add **Code** node `継続状況判定` with provided JS.
    - Add **Google Sheets** node `最終結果更新` (Update).
    - Chain connections:  
      **30日経過レコード取得 → 顧客ステータス確認 → 継続状況判定 → 最終結果更新**

19. **Important alignment fix (recommended)**
    - Replace both Merge nodes from “combine by position” to a key-based strategy:
      - Use `customer_id` to merge CRM/support/usage reliably.
      - If your n8n version doesn’t support key-merge in the Merge node, use Code nodes to join datasets by `customer_id`.

20. **Activate**
    - Validate credentials: OpenAI, Stripe, Gmail, SendGrid, Postgres, Google Sheets, CRM, support tool.
    - Execute once manually with test data.
    - Activate schedule.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “This workflow acts as an intelligent retention system… identifies at-risk customers and sends personalized offers.” | From sticky note “Workflow Overview” |
| Setup prerequisites: configure OpenAI, Stripe, Gmail, SendGrid, Postgres, Google Sheets; create a Google Sheet with columns like `customer_id`, `risk_score`, `offer_type`, `email_status`, `retention_result` | From sticky note “Workflow Overview” |
| “Adjust the SQL query in the Postgres node to match your table schema.” | From sticky note “Note - Schedule” |
| Risk gating rule: only risk score > 0.7 proceeds to retention flow | From sticky note “Sticky Note” |
| Tracking design: SendGrid open/click checks + 30-day active verification + sync to Sheets | From sticky note “Sticky Note2” |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensif ou protégé. Toutes les données manipulées sont légales et publiques.