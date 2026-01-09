Detect and correct claims cost leakage with GPT-4 and automated alerts

https://n8nworkflows.xyz/workflows/detect-and-correct-claims-cost-leakage-with-gpt-4-and-automated-alerts-12275


# Detect and correct claims cost leakage with GPT-4 and automated alerts

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title (given):** Detect and correct claims cost leakage with GPT-4 and automated alerts  
**Workflow name (JSON):** Detect and correct claims cost leakage using AI analysis and automation

**Purpose:**  
Runs daily to pull recent claims, detect potential “cost leakage” anomalies (overpayment, reserve mismatch, vendor billing issues), enriches each anomaly with vendor/policy/history data, uses GPT‑4 (structured output) to classify root cause and severity, generates recommended corrective adjustments, escalates critical cases by email, and sends a consolidated report.

**Primary use cases:**
- Claims operations: identify and prioritize suspicious/incorrect payouts.
- Finance/audit: quantify potential leakage and produce recovery/adjustment candidates.
- Vendor management: surface duplicate/inflated vendor charges.

### 1.1 Logical Blocks (by dependency)
1. **Trigger & Configuration**
2. **Claims Data Retrieval**
3. **Rule-based Anomaly Detection**
4. **Leakage Gate (IF) + Split to anomaly items**
5. **Parallel Enrichment (vendor, policy, historical DB) + merge**
6. **Risk Scoring + AI Root Cause Classification (GPT‑4 structured)**
7. **Corrective Adjustment Generation**
8. **Severity Routing & Escalation**
9. **Aggregation & Reporting**
10. **No-leakage branch**

---

## 2. Block-by-Block Analysis

### Block 1 — Trigger & Configuration
**Overview:** Starts on a daily schedule and sets workflow-wide configuration variables (API URLs, thresholds, recipients).  
**Nodes involved:**  
- Daily Claims Analysis Schedule  
- Workflow Configuration  

#### Node: Daily Claims Analysis Schedule
- **Type / role:** `Schedule Trigger` — entry point; runs on a fixed schedule.
- **Configuration:** Triggers daily at **02:00** (server time) via `triggerAtHour: 2`.
- **Outputs:** Connects to **Workflow Configuration**.
- **Edge cases / failures:** Timezone expectations; if instance timezone differs from business timezone, the run time may be off.

#### Node: Workflow Configuration
- **Type / role:** `Set` — central parameter store for downstream expressions.
- **Configuration choices (interpreted):**
  - Defines placeholders for:
    - `claimsApiUrl`, `vendorApiUrl`, `policyApiUrl`
    - `auditDbConnectionString` (note: not actually used by the Postgres node as a value)
    - `reportRecipients`, `escalationRecipients`
  - Defines numeric thresholds:
    - `excessivePayoutThreshold` = 10000
    - `reserveVarianceThreshold` = 0.25
    - `criticalThreshold` = 100000
  - **Include other fields:** enabled.
- **Outputs:** Connects to **Fetch Historical Claims Data**.
- **Edge cases / failures:** Placeholders must be replaced; otherwise HTTP nodes will fail with invalid URL; email nodes will fail if recipients empty.

---

### Block 2 — Claims Data Retrieval
**Overview:** Pulls last 30 days of claims from an external claims API.  
**Nodes involved:**  
- Fetch Historical Claims Data  

#### Node: Fetch Historical Claims Data
- **Type / role:** `HTTP Request` — retrieves claims dataset.
- **Configuration choices:**
  - `url` from `Workflow Configuration.claimsApiUrl`
  - Query params:
    - `startDate = $today.minus({ days: 30 }).toFormat('yyyy-MM-dd')`
    - `endDate = $today.toFormat('yyyy-MM-dd')`
  - Sends header `Content-Type: application/json`
  - `sendQuery: true`, `sendHeaders: true`
- **Outputs:** To **Detect Cost Leakage Anomalies**
- **Edge cases / failures:**
  - Auth not configured (node shows only content-type header; most APIs need auth headers).
  - API pagination not handled (if API returns partial results).
  - Response shape mismatch (workflow assumes items are claims; if API returns `{data:[...]}` you must split/transform first).

---

### Block 3 — Rule-based Anomaly Detection
**Overview:** Applies deterministic checks to claims to build an anomalies array and summary metrics.  
**Nodes involved:**  
- Detect Cost Leakage Anomalies  

#### Node: Detect Cost Leakage Anomalies
- **Type / role:** `Code` — analyzes all incoming claim items.
- **Configuration choices:**
  - Hardcoded thresholds inside code:
    - `EXCESSIVE_PAYOUT_THRESHOLD = 50000`
    - `RESERVE_VARIANCE_THRESHOLD = 0.25`
  - Detects:
    1. Excessive payout (`payoutAmount > 50000`)
    2. Reserve variance (`abs(actual-reserve)/reserve > 0.25`)
    3. Duplicate vendor charge (same `vendorId_serviceType_date`)
    4. Inflated charge (if `expectedAmount` exists and `amount > expectedAmount*1.25`)
  - Output: **single item** with fields:
    - `leakageDetected` (boolean)
    - `totalAnomalies`, `totalLeakageAmount`, `anomalies[]`, `analysisDate`, `claimsAnalyzed`
- **Outputs:** To **Check If Leakage Detected**
- **Key variables / expressions:** Uses `$input.all()`; uses claim fields like `claimId`, `payoutAmount`, `reserveAmount`, `actualAmount`, `vendorCharges[]`.
- **Edge cases / failures:**
  - **Mismatch with Workflow Configuration thresholds:** The node ignores configured `excessivePayoutThreshold` (10000) and uses 50000. If you intended configurability, replace constants with values from the Set node.
  - If `reserveAmount` is 0, reserve variance division may create `Infinity`.
  - If claims API doesn’t return vendorCharges array, vendor checks do nothing.
  - Returns a single item; downstream must expect that.

---

### Block 4 — Leakage Gate + Split to Items
**Overview:** Branches depending on leakage detection, and splits the anomalies array into per-anomaly items for enrichment/AI.  
**Nodes involved:**  
- Check If Leakage Detected  
- Split Anomalies Into Items  
- No Leakage Found  

#### Node: Check If Leakage Detected
- **Type / role:** `IF` — decides whether to process anomalies or exit.
- **Configuration choices:**
  - Condition checks: `{{ $('Detect Cost Leakage Anomalies').item.json.hasLeakage }}`
- **Connections:**
  - **True:** to **AI Root Cause Classifier** and **Split Anomalies Into Items**
  - **False:** to **No Leakage Found**
- **Critical issue (logic bug):**
  - The previous code outputs `leakageDetected`, **not** `hasLeakage`.
  - As written, the IF will usually evaluate false (undefined), sending execution to **No Leakage Found** even when leakage exists.
  - Fix: use `{{ $('Detect Cost Leakage Anomalies').item.json.leakageDetected }}`.
- **Edge cases / failures:** Expression failure if node name changes; boolean evaluation with “loose” validation may still treat undefined as false.

#### Node: Split Anomalies Into Items
- **Type / role:** `Split Out` — converts `anomalies[]` array into individual items.
- **Configuration choices:** splits field `anomalies`.
- **Outputs:** fan-out to:
  - Fetch Vendor History
  - Fetch Policy Rules
  - Check Historical Patterns
  - Merge Enrichment Data (direct feed)
- **Edge cases / failures:** If `anomalies` is missing/non-array, node errors.

#### Node: No Leakage Found
- **Type / role:** `Set` — produces a simple “no issues” status object.
- **Configuration:** sets:
  - `status = "No cost leakage detected"`
  - `analysisDate = $today (yyyy-MM-dd)`
  - `message = "All claims within acceptable parameters"`
- **Outputs:** None in the provided connections (workflow ends here).
- **Edge cases:** None; but you might want to still email a “no issues” summary—currently it doesn’t.

---

### Block 5 — Parallel Enrichment + Merge
**Overview:** For each anomaly, pulls additional context from vendor API, policy rules API, and a Postgres history table, then merges data before AI analysis.  
**Nodes involved:**  
- Fetch Vendor History  
- Fetch Policy Rules  
- Check Historical Patterns  
- Merge Enrichment Data  

#### Node: Fetch Vendor History
- **Type / role:** `HTTP Request` — vendor behavior lookback.
- **Configuration choices:**
  - URL from `Workflow Configuration.vendorApiUrl`
  - Query:
    - `vendorId = {{ $json.vendorId }}`
    - `lookbackDays = 90`
- **Output:** to **Merge Enrichment Data** (input index 1)
- **Edge cases / failures:**
  - Many anomalies created earlier do **not** include `vendorId` (only duplicate/inflated vendor charge anomalies infer vendorId in `details` but don’t store it as a field). This will send `vendorId=undefined`.
  - API auth not configured.

#### Node: Fetch Policy Rules
- **Type / role:** `HTTP Request` — fetches policy constraints for the claim.
- **Configuration choices:**
  - URL from `Workflow Configuration.policyApiUrl`
  - Query:
    - `claimType = {{ $json.claimType }}`
    - `policyId = {{ $json.policyId }}`
- **Output:** Not connected onward in the JSON (it runs, but results are not merged/used).
- **Edge cases / failures:** Same as above: anomalies likely don’t include `claimType`/`policyId`, causing undefined parameters.

#### Node: Check Historical Patterns
- **Type / role:** `Postgres` — queries historical leakage patterns.
- **Configuration choices:**
  - Query:
    - `SELECT COUNT(*) as pattern_count, AVG(leakage_amount) as avg_amount FROM claim_leakage_history WHERE claim_id = $1 OR vendor_id = $2 AND detected_date > NOW() - INTERVAL '90 days'`
  - Query replacement set as: `={{ $json.claimId }},={{ $json.vendorId }}`
- **Outputs:** to **Calculate Risk Score**
- **Edge cases / failures:**
  - **SQL logic precedence:** `claim_id = $1 OR vendor_id = $2 AND detected_date > ...` means `claim_id=$1` matches regardless of date. Likely intended `(claim_id=$1 OR vendor_id=$2) AND detected_date > ...`.
  - Credentials for Postgres are not shown; `auditDbConnectionString` from Set node is not wired automatically—must configure Postgres credentials in n8n.
  - Again, anomalies may not include `vendorId`.

#### Node: Merge Enrichment Data
- **Type / role:** `Merge` — combines streams.
- **Configuration choices:**
  - Mode: `combine`
  - Match field: `claimId`
  - Receives from:
    - Split Anomalies Into Items (index 0)
    - Fetch Vendor History (index 1)
- **Output:** to **AI Root Cause Classifier**
- **Edge cases / failures:**
  - If vendor history response doesn’t contain `claimId`, combining by `claimId` may drop/duplicate incorrectly.
  - If anomaly items don’t have `claimId` (some are set; code tries claimId/id/UNKNOWN), merge may group under UNKNOWN.

---

### Block 6 — Risk Scoring + AI Root Cause Classification (GPT‑4 structured)
**Overview:** Calculates a risk score from history/amount/vendor reputation, then uses a LangChain agent with GPT‑4o and a structured output parser.  
**Nodes involved:**  
- Calculate Risk Score  
- AI Root Cause Classifier  
- OpenAI GPT-4  
- Classification Output Parser  
- Calculator Tool  

#### Node: Calculate Risk Score
- **Type / role:** `Code` (run once per item) — computes risk score and category.
- **Inputs:** From **Check Historical Patterns** results (but it expects merged/enriched fields that may not exist).
- **Logic (interpreted):**
  - Uses weighted factors: historical frequency (30%), leakage amount (40%), vendor reputation (20%), policy violation (10%).
  - Normalizations:
    - historicalFrequency assumed 0–10 → 0–100
    - leakageAmount assumed 0–50000 → 0–100
    - vendorReputation 0–100 inverted
    - policyViolationSeverity assumed 0–10 → 0–100
  - Outputs: adds `riskScore`, `riskCategory`, `riskFactors`, `calculatedAt`.
- **Output:** to **AI Root Cause Classifier**
- **Edge cases / failures:**
  - Upstream Postgres query returns `pattern_count`/`avg_amount`, but this code expects `historicalFrequency` or `pattern_frequency`. Mapping is missing.
  - Leakage amount field name mismatches across workflow (`amount`, `leakageAmount`, `estimatedLeakageAmount`, etc.).
  - If vendor reputation not present, default 50.

#### Node: AI Root Cause Classifier
- **Type / role:** `LangChain Agent` — LLM-based classification with tool + structured output.
- **Configuration choices:**
  - Prompt text: `{{ $json.leakageDetails }}` (expects a field not created upstream).
  - System message defines root cause taxonomy and required fields per anomaly; instructs JSON output as per output parser.
  - Has Output Parser enabled.
- **Connections:**
  - Uses **OpenAI GPT‑4** as `ai_languageModel`
  - Uses **Calculator Tool** as `ai_tool`
  - Uses **Classification Output Parser** as `ai_outputParser`
  - Main output goes to **Generate Corrective Adjustments**
- **Edge cases / failures:**
  - **Input field mismatch:** `leakageDetails` is not produced; you likely want to pass the anomaly item (e.g., `{{ JSON.stringify($json) }}`) or map fields into a narrative.
  - If the agent returns non-conforming JSON, the structured parser will error.
  - OpenAI credential errors, rate limits, model access.

#### Node: OpenAI GPT-4
- **Type / role:** `lmChatOpenAi` — chat model provider for the agent.
- **Configuration:** model set to `gpt-4o`.
- **Credentials:** `OpenAi account` configured (must exist in target instance).
- **Edge cases:** Model availability; org policy; token limits if you stringify large objects.

#### Node: Classification Output Parser
- **Type / role:** `Structured Output Parser` — enforces schema.
- **Schema (manual):** object with:
  - `claimId`, `rootCause`, `confidenceScore`, `explanation`, `contributingFactors[]`, `severityLevel`, `estimatedLeakageAmount`
- **Edge cases:**
  - If the agent outputs snake_case keys (common) it will fail schema validation unless the agent is consistent.
  - Schema defines a *single object*, but the system prompt implies “for each anomaly”; if multiple anomalies were passed at once, schema mismatch.

#### Node: Calculator Tool
- **Type / role:** LangChain tool — allows arithmetic during reasoning.
- **Use:** Available to the agent; no direct workflow output.
- **Edge cases:** Minimal; only used if agent calls it.

---

### Block 7 — Corrective Adjustment Generation
**Overview:** Converts AI classification into an operational “adjustment” record with recommended action and priority.  
**Nodes involved:**  
- Generate Corrective Adjustments  

#### Node: Generate Corrective Adjustments
- **Type / role:** `Code` (run once per item).
- **Configuration choices / logic:**
  - Reads AI output using mixed key assumptions:
    - `root_cause || rootCause`
    - `leakage_amount || leakageAmount`
    - `claim_id || claimId`
    - `confidence || 0.5`
  - Maps root cause to actions:
    - DUPLICATE_PAYMENT/OVERPAYMENT → RECOVER_OVERPAYMENT (HIGH)
    - INCORRECT_PRICING/BILLING_ERROR → REDUCE_PAYMENT (MEDIUM)
    - RESERVE_ERROR → CORRECT_RESERVE (MEDIUM)
    - VENDOR_OVERCHARGE/PROVIDER_FRAUD → VENDOR_DISPUTE (HIGH)
    - default → REDUCE_PAYMENT (LOW)
  - Creates negative `adjustment_amount` (reduction/recovery).
  - Sets `status: PENDING_REVIEW`.
- **Outputs:**
  - To **Aggregate Findings Report**
  - To **Route By Severity**
- **Edge cases / failures:**
  - Schema parser produces `confidenceScore` (0–100) but code expects `confidence` as fraction; will produce wrong formatting and confidence math.
  - Parser produces `estimatedLeakageAmount` but code expects `leakageAmount/leakage_amount`.
  - Root cause categories from system message (e.g., `DUPLICATE_BILLING`) do not match switch cases (e.g., `DUPLICATE_PAYMENT`). Expect many defaults unless aligned.

---

### Block 8 — Severity Routing & Escalation
**Overview:** Routes adjustments by severity and escalates critical/high-dollar cases via email.  
**Nodes involved:**  
- Route By Severity  
- Check If Requires Escalation  
- Send Escalation Alert  

#### Node: Route By Severity
- **Type / role:** `Switch` — splits flow based on `severityLevel`.
- **Configuration:**
  - Outputs: CRITICAL, HIGH, MEDIUM, LOW based on `{{ $json.severityLevel }}`.
- **Outputs:** In the provided connections, all switch outputs go to:
  - **Check If Requires Escalation**
  - **Aggregate By Vendor**
- **Edge cases:**
  - After “Generate Corrective Adjustments”, items contain `priority_level` and `root_cause`, but not `severityLevel` unless AI output passed through unchanged. Ensure field is present.
  - If severity is missing, item will not match any route (node behavior depends on n8n settings; may drop items).

#### Node: Check If Requires Escalation
- **Type / role:** `IF` — determines whether to send urgent alert.
- **Condition (OR):**
  - `severityLevel == CRITICAL` OR
  - `estimatedLeakageAmount > Workflow Configuration.criticalThreshold`
- **Outputs:**
  - True → **Send Escalation Alert**
  - False → **Aggregate Findings Report**
- **Edge cases:**
  - The adjustment node uses `adjustment_amount` and `original_leakage_amount`, not `estimatedLeakageAmount`. Condition may never trigger unless fields are preserved.
  - Threshold is 100000; earlier anomaly detection caps normalizations at 50k, but leakage could exceed.

#### Node: Send Escalation Alert
- **Type / role:** `Email Send` — urgent notification.
- **Configuration:**
  - To: `Workflow Configuration.escalationRecipients`
  - From: placeholder sender email
  - Subject: fixed urgent line
  - HTML references fields like `claim_id`, `adjustment_amount`, `root_cause`, `confidence_score`, `priority_level`
- **Edge cases / failures:**
  - No email credentials configured in node definition (n8n email node typically requires SMTP credential).
  - Confidence displayed as `%` but `confidence_score` produced by adjustment code is currently a fraction (0.5) unless fixed.

---

### Block 9 — Aggregation & Reporting
**Overview:** Aggregates items (some vendor-level aggregation exists) and sends an executive email report.  
**Nodes involved:**  
- Aggregate By Vendor  
- Aggregate Findings Report  
- Send Leakage Report  

#### Node: Aggregate By Vendor
- **Type / role:** `Aggregate` — aggregates fields `vendorId` and `leakageAmount`.
- **Configuration:** Aggregates two fields; exact aggregation functions not specified beyond default behavior.
- **Output:** to **Aggregate Findings Report**
- **Edge cases:**
  - Upstream items may not contain `vendorId` or `leakageAmount` (field naming mismatches).
  - Aggregation may not produce the report fields referenced later.

#### Node: Aggregate Findings Report
- **Type / role:** `Aggregate` — “aggregate all item data” into `adjustmentDetails`.
- **Configuration:**
  - `aggregateAllItemData`
  - destination field: `adjustmentDetails`
- **Outputs:** to **Send Leakage Report**
- **Edge cases / failures:**
  - The email template expects fields like `totalLeakageAmount`, `highSeverityCount`, `leakageSummary`, `aiClassifications`, `correctiveAdjustments`. This aggregate node configuration (as shown) does not compute these fields—only nests data under `adjustmentDetails`. Report email will likely render blanks/undefined unless you add a summarization step.

#### Node: Send Leakage Report
- **Type / role:** `Email Send` — sends periodic report.
- **Configuration:**
  - To: `Workflow Configuration.reportRecipients`
  - From: placeholder sender email
  - Subject: includes today’s date
  - HTML references:
    - `Aggregate Findings Report`.first().json.totalLeakageAmount, severity counts/amounts, summaries, etc.
- **Edge cases / failures:**
  - Missing computed summary fields (see above).
  - SMTP credentials required.
  - Recipient list must be valid; supports comma-separated addresses depending on node implementation.

---

### Block 10 — Sticky Notes (Documentation nodes)
**Overview:** Embedded commentary; not executed.  
**Nodes involved:** Sticky Note, Sticky Note1, Sticky Note2, Sticky Note3, Sticky Note4, Sticky Note5  
**Key observation:** Several notes mention “competitor” monitoring and Slack alerts, which do not match the actual claims leakage implementation (no Slack node exists). Treat as template text.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Daily Claims Analysis Schedule | Schedule Trigger | Daily entrypoint | — | Workflow Configuration | ## How It Works (enterprise claims leakage detection… severity routing… Gmail and Slack alerts…) |
| Workflow Configuration | Set | Centralized config variables | Daily Claims Analysis Schedule | Fetch Historical Claims Data | ## How It Works (enterprise claims leakage detection…); ## Parallel Data Collection (HTTP nodes fetch vendor history…); ## Setup Steps (configure HTTP nodes… OpenAI… Gmail… Slack… schedule…) |
| Fetch Historical Claims Data | HTTP Request | Pull last 30 days of claims | Workflow Configuration | Detect Cost Leakage Anomalies | ## Parallel Data Collection (HTTP nodes fetch vendor history…); ## How It Works (claims ingested through parallel HTTP requests…) |
| Detect Cost Leakage Anomalies | Code | Rule-based anomaly detection | Fetch Historical Claims Data | Check If Leakage Detected | ## Parallel Data Collection (…detects cost anomalies across competitors.) |
| Check If Leakage Detected | IF | Branch on leakage presence | Detect Cost Leakage Anomalies | AI Root Cause Classifier; Split Anomalies Into Items; No Leakage Found | ## Conditional Risk Scoring and AI Analysis (Historical checks trigger risk scoring…) |
| Split Anomalies Into Items | Split Out | Fan-out anomalies array | Check If Leakage Detected | Fetch Vendor History; Fetch Policy Rules; Check Historical Patterns; Merge Enrichment Data | ## Parallel Data Collection (HTTP nodes fetch vendor history…); ## Conditional Risk Scoring and AI Analysis… |
| Fetch Vendor History | HTTP Request | Vendor context enrichment | Split Anomalies Into Items | Merge Enrichment Data | ## Parallel Data Collection… |
| Fetch Policy Rules | HTTP Request | Policy rules enrichment | Split Anomalies Into Items | — | ## Parallel Data Collection… |
| Check Historical Patterns | Postgres | Query leakage history patterns | Split Anomalies Into Items | Calculate Risk Score | ## Conditional Risk Scoring and AI Analysis… |
| Calculate Risk Score | Code | Compute riskScore/riskCategory | Check Historical Patterns | AI Root Cause Classifier | ## Conditional Risk Scoring and AI Analysis… |
| Merge Enrichment Data | Merge | Combine anomaly + vendor history | Split Anomalies Into Items; Fetch Vendor History | AI Root Cause Classifier | ## Conditional Risk Scoring and AI Analysis… |
| AI Root Cause Classifier | LangChain Agent | GPT classification with structured output | Check If Leakage Detected; Calculate Risk Score; Merge Enrichment Data | Generate Corrective Adjustments | ## Conditional Risk Scoring and AI Analysis… |
| OpenAI GPT-4 | OpenAI Chat Model | LLM provider (gpt-4o) | — (AI connection) | — (AI connection) | ## Prerequisites (OpenAI API key…); ## Setup Steps (Add OpenAI API key…) |
| Classification Output Parser | Structured Output Parser | Enforce JSON schema | — (AI connection) | — (AI connection) | ## Conditional Risk Scoring and AI Analysis… |
| Calculator Tool | LangChain Tool | Optional arithmetic tool for agent | — (AI connection) | — (AI connection) | ## Conditional Risk Scoring and AI Analysis… |
| Generate Corrective Adjustments | Code | Create adjustment recommendations | AI Root Cause Classifier | Aggregate Findings Report; Route By Severity | ## Conditional Risk Scoring and AI Analysis… |
| Route By Severity | Switch | Route by severityLevel | Generate Corrective Adjustments | Check If Requires Escalation; Aggregate By Vendor | ## Severity-Based Routing (High-severity findings trigger immediate Gmail and Slack alerts) |
| Check If Requires Escalation | IF | Escalate critical/high-amount | Route By Severity | Send Escalation Alert; Aggregate Findings Report | ## Severity-Based Routing… |
| Send Escalation Alert | Email Send | Urgent email alert | Check If Requires Escalation | — | ## Severity-Based Routing… |
| Aggregate By Vendor | Aggregate | Vendor-level aggregation | Route By Severity | Aggregate Findings Report | ## Severity-Based Routing… |
| Aggregate Findings Report | Aggregate | Consolidate all adjustments | Aggregate By Vendor; Generate Corrective Adjustments; Check If Requires Escalation | Send Leakage Report | ## Severity-Based Routing… |
| Send Leakage Report | Email Send | Send daily report | Aggregate Findings Report | — | ## How It Works…; ## Setup Steps (Connect Gmail… set leadership distribution list…) |
| No Leakage Found | Set | End branch with “no leakage” status | Check If Leakage Detected | — | ## Setup Steps… |
| Sticky Note | Sticky Note | Comment | — | — |  |
| Sticky Note1 | Sticky Note | Comment | — | — |  |
| Sticky Note2 | Sticky Note | Comment | — | — |  |
| Sticky Note3 | Sticky Note | Comment | — | — |  |
| Sticky Note4 | Sticky Note | Comment | — | — |  |
| Sticky Note5 | Sticky Note | Comment | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1) **Create “Daily Claims Analysis Schedule” (Schedule Trigger)**
   - Set it to run daily at **02:00** (adjust timezone as needed).

2) **Create “Workflow Configuration” (Set)**
   - Add fields:
     - `claimsApiUrl` (string) – your claims API endpoint
     - `vendorApiUrl` (string)
     - `policyApiUrl` (string)
     - `reportRecipients` (string, comma-separated emails)
     - `escalationRecipients` (string, comma-separated emails)
     - `excessivePayoutThreshold` (number)
     - `reserveVarianceThreshold` (number)
     - `criticalThreshold` (number)
   - Enable “Include Other Fields”.
   - Connect: **Schedule Trigger → Workflow Configuration**

3) **Create “Fetch Historical Claims Data” (HTTP Request)**
   - URL: `{{ $('Workflow Configuration').first().json.claimsApiUrl }}`
   - Query parameters:
     - `startDate = {{ $today.minus({ days: 30 }).toFormat('yyyy-MM-dd') }}`
     - `endDate = {{ $today.toFormat('yyyy-MM-dd') }}`
   - Header: `Content-Type: application/json`
   - Configure authentication for your API (Bearer token/OAuth2/etc.).
   - Connect: **Workflow Configuration → Fetch Historical Claims Data**

4) **Create “Detect Cost Leakage Anomalies” (Code)**
   - Paste the anomaly detection JS logic.
   - (Recommended) Replace hardcoded thresholds with config values:
     - `const EXCESSIVE_PAYOUT_THRESHOLD = $('Workflow Configuration').first().json.excessivePayoutThreshold;`
     - `const RESERVE_VARIANCE_THRESHOLD = $('Workflow Configuration').first().json.reserveVarianceThreshold;`
   - Connect: **Fetch Historical Claims Data → Detect Cost Leakage Anomalies**

5) **Create “Check If Leakage Detected” (IF)**
   - Condition (boolean true):
     - **Use:** `{{ $('Detect Cost Leakage Anomalies').item.json.leakageDetected }}`
     - (Fixes the current `hasLeakage` mismatch.)
   - Connect: **Detect Cost Leakage Anomalies → Check If Leakage Detected**

6) **Create “No Leakage Found” (Set)**
   - Set fields: `status`, `analysisDate`, `message` as in JSON.
   - Connect IF false output → **No Leakage Found**

7) **Create “Split Anomalies Into Items” (Split Out)**
   - Field to split: `anomalies`
   - Connect IF true output → **Split Anomalies Into Items**

8) **Create enrichment nodes (run in parallel from Split Out)**
   - **Fetch Vendor History (HTTP Request)**
     - URL: `{{ $('Workflow Configuration').first().json.vendorApiUrl }}`
     - Query: `vendorId={{$json.vendorId}}`, `lookbackDays=90`
     - Add auth as required.
   - **Fetch Policy Rules (HTTP Request)**
     - URL: `{{ $('Workflow Configuration').first().json.policyApiUrl }}`
     - Query: `claimType={{$json.claimType}}`, `policyId={{$json.policyId}}`
   - **Check Historical Patterns (Postgres)**
     - Configure Postgres credentials (host/db/user/pass) in n8n Credentials.
     - Use a corrected query (recommended):
       - `SELECT COUNT(*) as pattern_count, AVG(leakage_amount) as avg_amount FROM claim_leakage_history WHERE (claim_id = $1 OR vendor_id = $2) AND detected_date > NOW() - INTERVAL '90 days'`
     - Bind parameters with `$json.claimId` and `$json.vendorId`.

9) **Create “Merge Enrichment Data” (Merge)**
   - Mode: **Combine**
   - Match field: `claimId`
   - Connect:
     - Split Out → Merge input 0
     - Fetch Vendor History → Merge input 1
   - (Optional but recommended) also merge policy rules and DB outputs; currently the JSON does not.

10) **Create “Calculate Risk Score” (Code)**
   - Paste the risk scoring JS.
   - Ensure input fields exist (map Postgres output to expected names first, or update code to use `pattern_count`).
   - Connect: **Check Historical Patterns → Calculate Risk Score**

11) **Create the AI stack**
   - **OpenAI GPT-4** (OpenAI Chat Model)
     - Model: `gpt-4o`
     - Configure OpenAI API credentials.
   - **Classification Output Parser** (Structured Output Parser)
     - Paste the JSON schema from the workflow.
   - **Calculator Tool** (Tool)
     - No config required.
   - **AI Root Cause Classifier** (LangChain Agent)
     - System message: paste the root-cause taxonomy instructions.
     - Text input: ensure it references actual data. Recommended:
       - `{{ JSON.stringify($json) }}`
       - or build a composed summary with claimId/leakageType/details/amount/etc.
     - Attach:
       - Chat model: OpenAI GPT-4 node
       - Output parser: Classification Output Parser
       - Tool: Calculator Tool

12) **Create “Generate Corrective Adjustments” (Code)**
   - Paste the adjustment JS.
   - (Recommended) Align keys to parser schema:
     - use `rootCause`, `confidenceScore`, `estimatedLeakageAmount`, `claimId`.
     - Convert confidenceScore 0–100 to fraction if needed.
   - Connect: **AI Root Cause Classifier → Generate Corrective Adjustments**

13) **Create “Route By Severity” (Switch)**
   - Switch on `severityLevel`.
   - Create outputs: CRITICAL/HIGH/MEDIUM/LOW.
   - Connect: **Generate Corrective Adjustments → Route By Severity**

14) **Create “Check If Requires Escalation” (IF)**
   - OR conditions:
     - `severityLevel == CRITICAL`
     - `estimatedLeakageAmount > {{ $('Workflow Configuration').first().json.criticalThreshold }}`
   - Connect: **Route By Severity → Check If Requires Escalation**

15) **Create “Send Escalation Alert” (Email Send)**
   - Configure SMTP credential (or your email provider integration).
   - To: `{{ $('Workflow Configuration').first().json.escalationRecipients }}`
   - From: your sender address
   - Subject/HTML: as in workflow (adjust field names to your actual item).
   - Connect IF true → **Send Escalation Alert**

16) **Create aggregation & reporting**
   - **Aggregate By Vendor** (Aggregate)
     - Aggregate `vendorId`, `leakageAmount` (or your standardized fields).
     - Connect: **Route By Severity → Aggregate By Vendor**
   - **Aggregate Findings Report** (Aggregate)
     - “Aggregate all item data” into `adjustmentDetails`
     - (Recommended) Insert a Code node before emailing to compute totals and severity breakdown fields used by the email template.
     - Connect:
       - Aggregate By Vendor → Aggregate Findings Report
       - Generate Corrective Adjustments → Aggregate Findings Report (if you want all items)
       - IF false from escalation → Aggregate Findings Report
   - **Send Leakage Report** (Email Send)
     - To: `{{ $('Workflow Configuration').first().json.reportRecipients }}`
     - Configure SMTP credential
     - Subject/HTML: as in workflow, but ensure referenced summary fields exist.
     - Connect: **Aggregate Findings Report → Send Leakage Report**

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Sticky notes contain mixed domain text (“competitor feature releases”, Slack alerts). There is no Slack node in this workflow. Consider updating notes to match claims leakage context. | Embedded Sticky Notes (general) |
| The IF condition uses `hasLeakage`, but the anomaly detector outputs `leakageDetected`. This prevents the leakage branch from running until corrected. | Block 4: Check If Leakage Detected |
| Several nodes assume fields (`vendorId`, `claimType`, `policyId`, `leakageDetails`) that the anomaly detector does not output. Add mapping/enrichment or adjust prompts/queries accordingly. | Enrichment + AI blocks |
| Email nodes require SMTP (or equivalent) credentials; recipients and sender placeholders must be replaced. | Send Leakage Report / Send Escalation Alert |
| Postgres node requires a configured Postgres credential; the `auditDbConnectionString` Set value is not automatically used by the Postgres node. | Check Historical Patterns |

