Aggregate error alerts and send consolidated reports via Email and Jira

https://n8nworkflows.xyz/workflows/aggregate-error-alerts-and-send-consolidated-reports-via-email-and-jira-12989


# Aggregate error alerts and send consolidated reports via Email and Jira

## 1. Workflow Overview

**Purpose:** Poll a log source every hour, extract error events, deduplicate and classify them, then:
- Send **immediate email alerts** for **critical** errors
- Create **Jira issues** for **all unique** errors
- Send a **consolidated digest email** summarizing what happened (including created Jira issues)

**Target use cases:** Centralizing error alerting from any JSON log API (monitoring platform, ELK, custom endpoint) into email + Jira, while reducing noise via batching and deduplication.

### 1.1 Trigger & Data Fetch
Runs hourly, fetches raw logs via HTTP, and exits early if no logs are present.

### 1.2 Processing & Deduplication
Flattens the log array into items, batches them, deduplicates within each batch, and checks if any actionable errors remain.

### 1.3 Severity Assessment & Critical Notifications
Tags each error as `critical` or `normal` using keyword matching; sends immediate email for critical ones.

### 1.4 Jira Ticketing, Collection & Summary Digest
Builds Jira issue payloads, rate-limits requests, creates Jira issues, merges Jira results with critical-email path, then generates and emails a digest.

---

## 2. Block-by-Block Analysis

### Block 1 — Trigger & Data Fetch

**Overview:** Starts every hour and retrieves logs from an external API. If the response has no logs, processing is skipped and the workflow proceeds to the summary path (as currently wired).

**Nodes Involved:**
- Hourly Trigger
- Fetch Raw Logs
- Has Logs?

#### Node: Hourly Trigger
- **Type / Role:** `Schedule Trigger` — entry point on a timer.
- **Configuration:** Runs every **1 hour** (`interval -> hours`).
- **Inputs/Outputs:**
  - Output → **Fetch Raw Logs**
- **Version notes:** TypeVersion 1; standard schedule trigger.
- **Edge cases / failures:**
  - If n8n instance is down, scheduled executions are missed unless using an external scheduler pattern.

#### Node: Fetch Raw Logs
- **Type / Role:** `HTTP Request` — pulls raw log payload from a REST endpoint.
- **Configuration choices (interpreted):**
  - URL: `https://example.com/api/logs` (placeholder; must be replaced)
  - Authentication: `genericCredentialType` (requires configured HTTP credentials)
  - No explicit method shown; defaults typically to GET.
- **Inputs/Outputs:**
  - Input ← **Hourly Trigger**
  - Output → **Has Logs?**
- **Edge cases / failures:**
  - Auth failures (401/403), DNS/TLS issues, timeouts, non-JSON responses.
  - Response shape assumptions: downstream expects `$json.logs` to exist and be an array.

#### Node: Has Logs?
- **Type / Role:** `IF` — early gate to skip processing when empty.
- **Key expression:**
  - `Array.isArray($json.logs) ? $json.logs.length : 0` **larger than** `0`
- **Outputs (two branches):**
  - **True** → Parse & Flatten Logs
  - **False** → Generate Summary
- **Edge cases / failures:**
  - If the API uses a different field than `logs`, this IF will treat it as empty.
  - Current wiring sends “no logs” directly into **Generate Summary**, which expects items shaped like Jira issue results (see Block 4). This can produce misleading/empty summaries unless Format Summary Email handles it.

---

### Block 2 — Processing & Deduplication

**Overview:** Converts the logs array into individual items, processes them in batches of 20, deduplicates repeated errors inside each batch, and checks whether anything remains worth processing.

**Nodes Involved:**
- Parse & Flatten Logs
- Batch Logs
- Deduplicate Batch
- Any New Errors?

#### Node: Parse & Flatten Logs
- **Type / Role:** `Code` — transforms `{ logs: [...] }` into separate n8n items.
- **Configuration (logic):**
  - Reads `items[0].json.logs || []`
  - Returns `logs.map(l => ({ json: l }))`
- **Inputs/Outputs:**
  - Input ← Has Logs? (true branch)
  - Output → Batch Logs
- **Edge cases / failures:**
  - If `logs` is huge, item explosion can increase memory usage and execution time.
  - If logs contain nested objects, they are passed through as-is.

#### Node: Batch Logs
- **Type / Role:** `Split In Batches` — throttles processing by chunking items.
- **Configuration:**
  - Batch size: **20**
- **Inputs/Outputs:**
  - Input ← Parse & Flatten Logs
  - Output → Deduplicate Batch
- **Edge cases / failures:**
  - As wired, there is **no loop-back** connection to request the next batch (typical SplitInBatches pattern connects the downstream node back to SplitInBatches “Continue”). Without that, only the **first batch** may be processed depending on execution semantics and node behavior/version.
  - If you intend to process all logs, you likely need the standard batching loop.

#### Node: Deduplicate Batch
- **Type / Role:** `Code` — removes duplicates within the current batch.
- **Configuration (logic):**
  - Builds `dedupeKey = "${id}-${message}-${timestamp}"`
  - Filters out already-seen keys within the batch; adds `item.json.dedupeKey`
- **Inputs/Outputs:**
  - Input ← Batch Logs
  - Output → Any New Errors?
- **Edge cases / failures:**
  - Deduplication is **only in-memory per batch**, not across batches or across runs.
  - If `message`/`timestamp` differ slightly, near-duplicates won’t be removed.
  - If `id` is missing, key relies on message+timestamp.

#### Node: Any New Errors?
- **Type / Role:** `IF` — check whether the current item is non-empty.
- **Key expression:**
  - `Object.keys($json).length > 0` is true
- **Outputs:**
  - **True** → Assess Severity
  - **False** → Generate Summary
- **Edge cases / failures:**
  - After a filter, n8n typically produces **zero items** rather than “empty objects”. This IF checks per-item content, not item count; if there are zero items, the node may not run at all. This branch logic may not behave as intended for “no items left”.
  - If deduplication returns items with minimal fields, still passes.

---

### Block 3 — Severity Assessment & Critical Notifications

**Overview:** Tags each error as critical or normal using keyword matching, sends immediate emails for critical items, and in parallel prepares Jira issues for every unique error.

**Nodes Involved:**
- Assess Severity
- Critical Errors?
- Critical Alert Email
- Prepare Jira Issue

#### Node: Assess Severity
- **Type / Role:** `Code` — enriches each error with a `severity` field.
- **Configuration (logic):**
  - `crit = /critical|fatal|panic/i.test($json.level || $json.message)`
  - Adds `severity: 'critical'` else `severity: 'normal'`
- **Inputs/Outputs:**
  - Input ← Any New Errors? (true branch)
  - Output → **Critical Errors?** and **Prepare Jira Issue** (fan-out)
- **Edge cases / failures:**
  - If neither `level` nor `message` exists, test uses `undefined` and returns false → severity normal.
  - Keyword matching can create false positives/negatives; consider structured numeric levels.

#### Node: Critical Errors?
- **Type / Role:** `IF` — routes only critical items to alerting.
- **Condition:**
  - `$json.severity` equals `"critical"`
- **Inputs/Outputs:**
  - Input ← Assess Severity
  - True output → Critical Alert Email
- **Edge cases / failures:**
  - No “false” output used; non-critical items simply do not send immediate email.

#### Node: Critical Alert Email
- **Type / Role:** `Email Send` — immediate notification for critical errors.
- **Configuration:**
  - To: `user@example.com` (placeholder)
  - From: `user@example.com` (placeholder)
  - Subject: `Critical Error Alert – {{ $json.message }}`
  - Body not configured (defaults depend on node; often empty unless set)
- **Inputs/Outputs:**
  - Input ← Critical Errors? (true)
  - Output → Collect Issue Keys (input index 1)
- **Edge cases / failures:**
  - SMTP credential missing/misconfigured, relay rejection, invalid from/to.
  - Subject uses `$json.message`; if absent, subject becomes incomplete.
  - If you need rich formatting, configure HTML/text body.

#### Node: Prepare Jira Issue
- **Type / Role:** `Code` — builds a minimal Jira issue creation payload.
- **Configuration (logic):**
  - `projectKey: 'PROJ'` (placeholder)
  - `summary: [SEVERITY] <first 80 chars of message>`
  - `description: JSON.stringify($json, null, 2)`
- **Inputs/Outputs:**
  - Input ← Assess Severity
  - Output → Rate Limit Wait
- **Edge cases / failures:**
  - Jira may require `issuetype`, additional fields, or specific description formatting (ADF in Jira Cloud).
  - `message` undefined results in empty summary body; Jira may reject empty summary.

---

### Block 4 — Jira Ticketing, Collection & Summary Digest

**Overview:** Rate-limits Jira creation, creates issues, merges results with the critical-email branch, generates a run summary, formats an email payload, and sends a digest email.

**Nodes Involved:**
- Rate Limit Wait
- Create Jira Issue
- Collect Issue Keys
- Generate Summary
- Format Summary Email
- Daily Summary Email

#### Node: Rate Limit Wait
- **Type / Role:** `Wait` — throttling before Jira API call.
- **Configuration:** No explicit duration shown (defaults vary; often requires configuration to be effective).
- **Inputs/Outputs:**
  - Input ← Prepare Jira Issue
  - Output → Create Jira Issue
- **Edge cases / failures:**
  - If duration is not set, it may not rate-limit as expected.
  - Large volumes still risk Jira rate limits; consider adaptive retry/backoff.

#### Node: Create Jira Issue
- **Type / Role:** `Jira` — creates an issue in Jira.
- **Configuration:** Not specified in JSON (empty parameters). This must be configured in the UI:
  - Resource likely “Issue”, Operation “Create”
  - Map fields from **Prepare Jira Issue** output
- **Credentials:** Jira credentials required (OAuth2/API token depending on Jira Cloud/Server).
- **Inputs/Outputs:**
  - Input ← Rate Limit Wait
  - Output → Collect Issue Keys (input index 0)
- **Edge cases / failures:**
  - Missing credentials, insufficient permissions, invalid project key, required fields missing.
  - Jira Cloud description may require Atlassian Document Format (ADF) in some configurations.

#### Node: Collect Issue Keys
- **Type / Role:** `Merge` — combines two streams:
  - Jira creation results
  - Critical email results
- **Configuration:** Merge mode not specified (defaults apply; must be checked in node settings).
- **Inputs/Outputs:**
  - Input 0 ← Create Jira Issue
  - Input 1 ← Critical Alert Email
  - Output → Generate Summary
- **Edge cases / failures:**
  - If one branch doesn’t emit items (e.g., no critical emails), merge behavior depends on merge mode and may block or drop data.
  - Mixing different item schemas (email output vs Jira output) can confuse summary logic.

#### Node: Generate Summary
- **Type / Role:** `Code` — aggregates stats and packages digest data.
- **Configuration (logic):**
  - `all = items.map(i => i.json)`
  - `total = all.length`
  - `critical = all.filter(i => /critical|fatal|panic/i.test(i.summary)).length`
  - Output: `{ total, critical, issues: all, timestamp: now }`
- **Inputs/Outputs:**
  - Inputs can come from:
    - Has Logs? (false branch)
    - Any New Errors? (false branch)
    - Collect Issue Keys
  - Output → Format Summary Email
- **Edge cases / failures:**
  - It assumes items have `.summary` (typical Jira issue field). If items come from Email Send output or from “no logs” branch, `.summary` may not exist → critical count becomes 0 and “issues” becomes a mix of unrelated objects.
  - If you want critical count based on the original errors, compute from error items before Jira/email nodes.

#### Node: Format Summary Email
- **Type / Role:** `Set` — intended to shape fields for the final email (subject/body).
- **Configuration:** Empty (`options: {}`), so it currently does not set `subject` or body fields.
- **Inputs/Outputs:**
  - Input ← Generate Summary
  - Output → Daily Summary Email
- **Edge cases / failures:**
  - Daily Summary Email subject references `$json.subject`; without setting it here, subject will be empty/undefined.

#### Node: Daily Summary Email
- **Type / Role:** `Email Send` — sends consolidated digest.
- **Configuration:**
  - To: `user@example.com` (placeholder)
  - From: `user@example.com` (placeholder)
  - Subject: `{{ $json.subject }}`
- **Inputs/Outputs:**
  - Input ← Format Summary Email
- **Edge cases / failures:**
  - Subject likely blank until Format Summary Email sets it.
  - Missing email body configuration may result in empty digest content.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Hourly Trigger | Schedule Trigger | Hourly workflow entry point | — | Fetch Raw Logs | ## Trigger & Data Fetch: This group starts the automation on a fixed hourly schedule and pulls raw log data from your monitoring or logging platform. The Schedule Trigger guarantees a predictable cadence, while the HTTP Request node can be pointed at any JSON-based endpoint (REST API, AWS Gateway, ELK, etc.). The first IF node quickly exits the pipeline if the payload is empty, avoiding unnecessary processing and email noise. Make sure to secure the HTTP credentials and adjust polling frequency to respect rate limits. |
| Fetch Raw Logs | HTTP Request | Retrieve raw logs from external API | Hourly Trigger | Has Logs? | ## Trigger & Data Fetch: This group starts the automation on a fixed hourly schedule and pulls raw log data from your monitoring or logging platform. The Schedule Trigger guarantees a predictable cadence, while the HTTP Request node can be pointed at any JSON-based endpoint (REST API, AWS Gateway, ELK, etc.). The first IF node quickly exits the pipeline if the payload is empty, avoiding unnecessary processing and email noise. Make sure to secure the HTTP credentials and adjust polling frequency to respect rate limits. |
| Has Logs? | IF | Gate: continue only if logs array has items | Fetch Raw Logs | Parse & Flatten Logs; Generate Summary | ## Trigger & Data Fetch: This group starts the automation on a fixed hourly schedule and pulls raw log data from your monitoring or logging platform. The Schedule Trigger guarantees a predictable cadence, while the HTTP Request node can be pointed at any JSON-based endpoint (REST API, AWS Gateway, ELK, etc.). The first IF node quickly exits the pipeline if the payload is empty, avoiding unnecessary processing and email noise. Make sure to secure the HTTP credentials and adjust polling frequency to respect rate limits. |
| Parse & Flatten Logs | Code | Convert `logs[]` array into individual items | Has Logs? | Batch Logs | ## Processing & Deduplication: After retrieval, the logs are flattened into individual items, batched in groups of 20 and run through an in-memory deduplication routine. This reduces noise when identical errors repeat. The subsequent logic node determines whether any unique errors remain. If yes, each record is tagged with a severity flag based on keywords. You can extend this code to incorporate numeric levels or external severity look-ups. Everything beyond this point assumes each item is a unique, actionable error. |
| Batch Logs | Split In Batches | Process logs in chunks of 20 | Parse & Flatten Logs | Deduplicate Batch | ## Processing & Deduplication: After retrieval, the logs are flattened into individual items, batched in groups of 20 and run through an in-memory deduplication routine. This reduces noise when identical errors repeat. The subsequent logic node determines whether any unique errors remain. If yes, each record is tagged with a severity flag based on keywords. You can extend this code to incorporate numeric levels or external severity look-ups. Everything beyond this point assumes each item is a unique, actionable error. |
| Deduplicate Batch | Code | Remove duplicates inside batch; add `dedupeKey` | Batch Logs | Any New Errors? | ## Processing & Deduplication: After retrieval, the logs are flattened into individual items, batched in groups of 20 and run through an in-memory deduplication routine. This reduces noise when identical errors repeat. The subsequent logic node determines whether any unique errors remain. If yes, each record is tagged with a severity flag based on keywords. You can extend this code to incorporate numeric levels or external severity look-ups. Everything beyond this point assumes each item is a unique, actionable error. |
| Any New Errors? | IF | Gate: proceed only if item is non-empty | Deduplicate Batch | Assess Severity; Generate Summary | ## Processing & Deduplication: After retrieval, the logs are flattened into individual items, batched in groups of 20 and run through an in-memory deduplication routine. This reduces noise when identical errors repeat. The subsequent logic node determines whether any unique errors remain. If yes, each record is tagged with a severity flag based on keywords. You can extend this code to incorporate numeric levels or external severity look-ups. Everything beyond this point assumes each item is a unique, actionable error. |
| Assess Severity | Code | Tag each error as `critical` or `normal` | Any New Errors? | Critical Errors?; Prepare Jira Issue | ## Processing & Deduplication: After retrieval, the logs are flattened into individual items, batched in groups of 20 and run through an in-memory deduplication routine. This reduces noise when identical errors repeat. The subsequent logic node determines whether any unique errors remain. If yes, each record is tagged with a severity flag based on keywords. You can extend this code to incorporate numeric levels or external severity look-ups. Everything beyond this point assumes each item is a unique, actionable error. |
| Critical Errors? | IF | Route critical items to immediate email | Assess Severity | Critical Alert Email | ## Notification & Storage: Critical events trigger an immediate alert email so your on-call engineers respond without delay. Regardless of severity, each error is turned into a Jira issue, rate-limited by a brief wait node to avoid API abuse. All created issues flow into a Merge node that feeds a summary builder. The final set and email nodes craft a human-readable digest containing counts, critical breakdown and newly minted Jira keys—perfect for daily reporting or ChatOps ingestion. |
| Critical Alert Email | Email Send | Send immediate alert for critical errors | Critical Errors? | Collect Issue Keys | ## Notification & Storage: Critical events trigger an immediate alert email so your on-call engineers respond without delay. Regardless of severity, each error is turned into a Jira issue, rate-limited by a brief wait node to avoid API abuse. All created issues flow into a Merge node that feeds a summary builder. The final set and email nodes craft a human-readable digest containing counts, critical breakdown and newly minted Jira keys—perfect for daily reporting or ChatOps ingestion. |
| Prepare Jira Issue | Code | Build Jira issue payload from error | Assess Severity | Rate Limit Wait | ## Notification & Storage: Critical events trigger an immediate alert email so your on-call engineers respond without delay. Regardless of severity, each error is turned into a Jira issue, rate-limited by a brief wait node to avoid API abuse. All created issues flow into a Merge node that feeds a summary builder. The final set and email nodes craft a human-readable digest containing counts, critical breakdown and newly minted Jira keys—perfect for daily reporting or ChatOps ingestion. |
| Rate Limit Wait | Wait | Throttle Jira creation | Prepare Jira Issue | Create Jira Issue | ## Notification & Storage: Critical events trigger an immediate alert email so your on-call engineers respond without delay. Regardless of severity, each error is turned into a Jira issue, rate-limited by a brief wait node to avoid API abuse. All created issues flow into a Merge node that feeds a summary builder. The final set and email nodes craft a human-readable digest containing counts, critical breakdown and newly minted Jira keys—perfect for daily reporting or ChatOps ingestion. |
| Create Jira Issue | Jira | Create Jira ticket for each unique error | Rate Limit Wait | Collect Issue Keys | ## Notification & Storage: Critical events trigger an immediate alert email so your on-call engineers respond without delay. Regardless of severity, each error is turned into a Jira issue, rate-limited by a brief wait node to avoid API abuse. All created issues flow into a Merge node that feeds a summary builder. The final set and email nodes craft a human-readable digest containing counts, critical breakdown and newly minted Jira keys—perfect for daily reporting or ChatOps ingestion. |
| Collect Issue Keys | Merge | Merge Jira results with critical-email path | Create Jira Issue; Critical Alert Email | Generate Summary | ## Notification & Storage: Critical events trigger an immediate alert email so your on-call engineers respond without delay. Regardless of severity, each error is turned into a Jira issue, rate-limited by a brief wait node to avoid API abuse. All created issues flow into a Merge node that feeds a summary builder. The final set and email nodes craft a human-readable digest containing counts, critical breakdown and newly minted Jira keys—perfect for daily reporting or ChatOps ingestion. |
| Generate Summary | Code | Aggregate totals/critical count and package digest | Has Logs?; Any New Errors?; Collect Issue Keys | Format Summary Email | ## Notification & Storage: Critical events trigger an immediate alert email so your on-call engineers respond without delay. Regardless of severity, each error is turned into a Jira issue, rate-limited by a brief wait node to avoid API abuse. All created issues flow into a Merge node that feeds a summary builder. The final set and email nodes craft a human-readable digest containing counts, critical breakdown and newly minted Jira keys—perfect for daily reporting or ChatOps ingestion. |
| Format Summary Email | Set | Prepare fields (subject/body) for digest email | Generate Summary | Daily Summary Email | ## Notification & Storage: Critical events trigger an immediate alert email so your on-call engineers respond without delay. Regardless of severity, each error is turned into a Jira issue, rate-limited by a brief wait node to avoid API abuse. All created issues flow into a Merge node that feeds a summary builder. The final set and email nodes craft a human-readable digest containing counts, critical breakdown and newly minted Jira keys—perfect for daily reporting or ChatOps ingestion. |
| Daily Summary Email | Email Send | Send consolidated digest email | Format Summary Email | — | ## Notification & Storage: Critical events trigger an immediate alert email so your on-call engineers respond without delay. Regardless of severity, each error is turned into a Jira issue, rate-limited by a brief wait node to avoid API abuse. All created issues flow into a Merge node that feeds a summary builder. The final set and email nodes craft a human-readable digest containing counts, critical breakdown and newly minted Jira keys—perfect for daily reporting or ChatOps ingestion. |
| Overview | Sticky Note | Documentation / overview | — | — | # Error Alert Aggregator; ## How it works…; ## Setup steps… |
| Section – Fetch | Sticky Note | Documentation for fetch block | — | — | ## Trigger & Data Fetch… |
| Section – Process | Sticky Note | Documentation for processing block | — | — | ## Processing & Deduplication… |
| Section – Notify & Store | Sticky Note | Documentation for notify/store block | — | — | ## Notification & Storage… |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** named: *Error Alert Aggregator – Email and Jira*.

2. **Add node: Schedule Trigger** (name it **Hourly Trigger**)
   - Set **Interval** to **Every 1 hour**.
   - This is the workflow entry.

3. **Add node: HTTP Request** (name it **Fetch Raw Logs**)
   - Method: **GET** (default)
   - URL: set to your log endpoint (replace placeholder).
   - Authentication: choose the appropriate mode (API key/Bearer/OAuth2) and create/select the credential.
   - Ensure response is JSON and includes an array field like `logs`.

4. **Connect**: Hourly Trigger → Fetch Raw Logs.

5. **Add node: IF** (name it **Has Logs?**)
   - Condition: **Number**
   - Value 1 (expression): `={{ Array.isArray($json.logs) ? $json.logs.length : 0 }}`
   - Operation: **larger**
   - Value 2: `0`

6. **Connect**: Fetch Raw Logs → Has Logs?

7. **Add node: Code** (name it **Parse & Flatten Logs**)
   - Paste code:
     ```js
     // Flatten the incoming log payload
     const logs = items[0].json.logs || [];
     return logs.map(l => ({ json: l }));
     ```

8. **Connect** (true branch): Has Logs? (true) → Parse & Flatten Logs.

9. **Add node: Split In Batches** (name it **Batch Logs**)
   - Batch Size: **20**

10. **Connect**: Parse & Flatten Logs → Batch Logs.

11. **Add node: Code** (name it **Deduplicate Batch**)
   - Paste code:
     ```js
     // Basic in-batch deduplication
     const seen = new Set();
     return items.filter(item => {
       const key = `${item.json.id || ''}-${item.json.message}-${item.json.timestamp}`;
       if (seen.has(key)) return false;
       seen.add(key);
       item.json.dedupeKey = key;
       return true;
     });
     ```

12. **Connect**: Batch Logs → Deduplicate Batch.

13. **Add node: IF** (name it **Any New Errors?**)
   - Condition: **Boolean**
   - Value 1 (expression): `={{ Object.keys($json).length > 0 }}`
   - Operation: **isTrue**

14. **Connect**: Deduplicate Batch → Any New Errors?

15. **Add node: Code** (name it **Assess Severity**)
   - Paste code:
     ```js
     // Add a severity flag
     const crit = /critical|fatal|panic/i.test($json.level || $json.message);
     return [{ json: { ...$json, severity: crit ? 'critical' : 'normal' } }];
     ```

16. **Connect** (true branch): Any New Errors? (true) → Assess Severity.

17. **Add node: IF** (name it **Critical Errors?**)
   - Condition: **String**
   - Value 1: `={{ $json.severity }}`
   - Operation: **equals**
   - Value 2: `critical`

18. **Connect**: Assess Severity → Critical Errors?

19. **Add node: Email Send** (name it **Critical Alert Email**)
   - Configure **SMTP credentials** (or your email provider integration).
   - To: on-call address(es)
   - From: authorized sender
   - Subject: `Critical Error Alert – {{ $json.message }}`

20. **Connect** (true branch): Critical Errors? (true) → Critical Alert Email.

21. **Add node: Code** (name it **Prepare Jira Issue**)
   - Paste code:
     ```js
     // Build minimal issue payload
     return [{
       json: {
         projectKey: 'PROJ',
         summary: `[${$json.severity.toUpperCase()}] ${($json.message || '').slice(0, 80)}`,
         description: `Error details (auto-generated):\n\n${JSON.stringify($json, null, 2)}`
       }
     }];
     ```
   - Update `projectKey` and (optionally) add issue type fields required by your Jira setup.

22. **Connect**: Assess Severity → Prepare Jira Issue.

23. **Add node: Wait** (name it **Rate Limit Wait**)
   - Set a wait duration suitable for your Jira limits (e.g., 200–500 ms or 1s per issue if needed).

24. **Connect**: Prepare Jira Issue → Rate Limit Wait.

25. **Add node: Jira** (name it **Create Jira Issue**)
   - Configure **Jira credentials** (Jira Cloud typically uses API token or OAuth2).
   - Resource: **Issue**
   - Operation: **Create**
   - Map:
     - Project Key ← `{{$json.projectKey}}`
     - Summary ← `{{$json.summary}}`
     - Description ← `{{$json.description}}`
     - Issue Type: set explicitly if required (often mandatory).

26. **Connect**: Rate Limit Wait → Create Jira Issue.

27. **Add node: Merge** (name it **Collect Issue Keys**)
   - Choose a merge mode appropriate to your intent (commonly “Append” to combine item lists).
   - Connect:
     - Create Jira Issue → Merge Input 1 (or 0)
     - Critical Alert Email → Merge Input 2 (or 1)

28. **Connect**: Create Jira Issue → Collect Issue Keys.
29. **Connect**: Critical Alert Email → Collect Issue Keys.

30. **Add node: Code** (name it **Generate Summary**)
   - Paste code:
     ```js
     // Build aggregated stats for summary email
     const all = items.map(i => i.json);
     const total = all.length;
     const critical = all.filter(i => /critical|fatal|panic/i.test(i.summary)).length;
     return [{ json: { total, critical, issues: all, timestamp: new Date().toISOString() } }];
     ```

31. **Connect**: Collect Issue Keys → Generate Summary.
   - Also connect:
     - Has Logs? (false) → Generate Summary
     - Any New Errors? (false) → Generate Summary  
     (Note: this mirrors the provided workflow, but you may want a dedicated “no-op summary” path to avoid schema mismatch.)

32. **Add node: Set** (name it **Format Summary Email**)
   - Create fields at minimum:
     - `subject` (string), e.g. `Error digest: {{$json.total}} issues ({{$json.critical}} critical) – {{$json.timestamp}}`
     - Optionally `text`/`html` body fields summarizing `$json.issues`
   - In the provided JSON this node is empty; without setting `subject`, the next node’s subject will be blank.

33. **Connect**: Generate Summary → Format Summary Email.

34. **Add node: Email Send** (name it **Daily Summary Email**)
   - SMTP credentials (same or different)
   - To/From: set recipients
   - Subject: `{{ $json.subject }}`
   - Body: map from fields you set in the previous node (recommended).

35. **Connect**: Format Summary Email → Daily Summary Email.

36. **Test with sample data**, then **activate** the workflow.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow polls the log source every hour, flattens, batches (20), deduplicates, flags severity, emails critical alerts, creates Jira issues, then sends a digest. | Sticky note “Overview” and section notes embedded in the canvas |
| Setup guidance included in the canvas: create HTTP/Jira/SMTP credentials; replace log API URL; adjust batch/dedupe; update Jira project key; set recipients; test and tune severity/schedule. | Sticky note “Overview” (canvas documentation) |