Manage idempotent lead intake via webhook, Postgres, and HTTP APIs

https://n8nworkflows.xyz/workflows/manage-idempotent-lead-intake-via-webhook--postgres--and-http-apis-20541


# Manage idempotent lead intake via webhook, Postgres, and HTTP APIs

### 1. Workflow Overview

This workflow manages idempotent lead intake via a webhook endpoint backed by a PostgreSQL database and third-party HTTP APIs. Its primary purpose is to ensure that lead submissions identified by a client-supplied `Idempotency-Key` are processed at most once. This prevents duplicate record creation and inconsistent states caused by retries, network timeouts, or concurrent requests.

#### Target Use Cases
- High-reliability lead capture pipelines where incoming requests from multiple sources (websites, ads, partner systems) must be deduplicated.
- Integration architectures requiring safe HTTP request retries without duplicating data in downstream CRMs.

#### Logical Blocks
- **1.1 Input Reception & Schema Provisioning:** Receives the raw HTTP POST request and automatically ensures the PostgreSQL storage table and indexes exist.
- **1.2 Validation, Fingerprinting, & Atomic Claim:** Validates incoming headers and payload structure, generates a SHA-256 fingerprint of the canonical payload, and attempts to atomically claim or fetch the idempotency lease in PostgreSQL.
- **1.3 Request Routing & Decision Engine:** Evaluates the database claim result to decide whether the workflow should execute processing, replay a previously completed response, or return a conflict/error status.
- **1.4 API Processing Pipeline:** Executes sequential external service calls (Enrichment, CRM, and Notification APIs) with scoring and pacing pauses when a fresh execution is authorized.
- **1.5 Persistence & Response Delivery:** Classifies the execution outcome, stores the final response code and body back into PostgreSQL under strict fencing constraints, and responds to the original webhook caller.

---

### 2. Block-by-Block Analysis

---

### 2.1 Input Reception & Schema Provisioning

#### Overview
This block receives the incoming lead submission via webhook and initializes the database schema required for idempotency tracking if it does not already exist.

#### Nodes Involved
- `Webhook - Lead Submission`
- `Postgres - Ensure Idempotency Store`

#### Node Details

##### Webhook - Lead Submission
- **Type and Technical Role:** `n8n-nodes-base.webhook` (v2) - Acts as the primary HTTP entry point for incoming lead payloads.
- **Configuration Choices:** Configured to listen for `POST` requests on the path `idempotent-lead` with response mode set to `responseNode`.
- **Key Expressions or Variables:** N/A (Entry point node).
- **Input and Output Connections:** No input nodes; outputs to `Postgres - Ensure Idempotency Store`.
- **Version-Specific Requirements:** Type version 2.
- **Edge Cases / Failure Types:** Incoming requests missing required HTTP headers or payload formatting issues.
- **Sub-workflow Reference:** None.

##### Postgres - Ensure Idempotency Store
- **Type and Technical Role:** `n8n-nodes-base.postgres` (v2.5) - Executes DDL queries to self-provision the `idempotency_keys` table and associated indexes.
- **Configuration Choices:** Operation set to `executeQuery`. Contains SQL statements creating the `idempotency_keys` table, status/lock indexes, and a unique partial index for active business keys.
- **Key Expressions or Variables:** N/A.
- **Input and Output Connections:** Connected from `Webhook - Lead Submission`; outputs to `Code - Validate & Fingerprint`.
- **Version-Specific Requirements:** Type version 2.5. `continueOnFail` is enabled.
- **Edge Cases / Failure Types:** Database connection failures or insufficient user permissions to create tables/indexes.
- **Sub-workflow Reference:** None.

---

### 2.2 Validation, Fingerprinting, & Atomic Claim

#### Overview
This block validates the structure and contents of the lead payload, scopes the idempotency key per client, computes a deterministic SHA-256 fingerprint, and attempts an atomic SQL transaction to claim the lease.

#### Nodes Involved
- `Code - Validate & Fingerprint`
- `Postgres - Claim Idempotency Key`

#### Node Details

##### Code - Validate & Fingerprint
- **Type and Technical Role:** `n8n-nodes-base.code` (v2) - JavaScript execution node that validates request data against policy configurations and generates a SHA-256 hash of the canonicalized payload.
- **Configuration Choices:** Defines operational parameters such as timeouts, lease durations, allowed lead sources, and endpoint URLs within a `CONFIG` object.
- **Key Expressions or Variables:** Accesses upstream webhook headers and body fields (`Webhook - Lead Submission`).
- **Input and Output Connections:** Connected from `Postgres - Ensure Idempotency Store`; outputs to `Postgres - Claim Idempotency Key`.
- **Version-Specific Requirements:** Type version 2.
- **Edge Cases / Failure Types:** Missing `Idempotency-Key` headers, malformed email addresses, or unallowed lead sources resulting in validation error arrays.
- **Sub-workflow Reference:** None.

##### Postgres - Claim Idempotency Key
- **Type and Technical Role:** `n8n-nodes-base.postgres` (v2.5) - Executes a complex parameterized CTE (Common Table Expression) to insert or update the lease status atomically.
- **Configuration Choices:** Operation set to `executeQuery`. Uses query replacements mapped from the preceding code node.
- **Key Expressions or Variables:** `={{ [ $json.idemKey, $json.businessKey, $json.fingerprint, $json.config.leaseSeconds, $json.valid ] }}`
- **Input and Output Connections:** Connected from `Code - Validate & Fingerprint`; outputs to `Code - Route Request`.
- **Version-Specific Requirements:** Type version 2.5. `continueOnFail` and `alwaysOutputData` are enabled.
- **Edge Cases / Failure Types:** Database deadlocks under high concurrency or constraint violations.
- **Sub-workflow Reference:** None.

---

### 2.3 Request Routing & Decision Engine

#### Overview
This block interprets the status of the database claim and request validation flags to determine the exact execution route (`execute`, `replay`, `in_progress`, `mismatch`, `duplicate`, `invalid`, or `store_unavailable`).

#### Nodes Involved
- `Code - Route Request`
- `IF - Execute Processing?`

#### Node Details

##### Code - Route Request
- **Type and Technical Role:** `n8n-nodes-base.code` (v2) - JavaScript node evaluating the database claim outcome and comparing payloads.
- **Configuration Choices:** Evaluates error flags, claim states, fingerprint matches, and completion status.
- **Key Expressions or Variables:** References data from `Code - Validate & Fingerprint` and the output row of `Postgres - Claim Idempotency Key`.
- **Input and Output Connections:** Connected from `Postgres - Claim Idempotency Key`; outputs to `IF - Execute Processing?`.
- **Version-Specific Requirements:** Type version 2.
- **Edge Cases / Failure Types:** Handling payload mismatches on reused idempotency keys (returning HTTP 422).
- **Sub-workflow Reference:** None.

##### IF - Execute Processing?
- **Type and Technical Role:** `n8n-nodes-base.if` (v2.2) - Conditional router branch node.
- **Configuration Choices:** Evaluates whether the evaluated route equals `'execute'`.
- **Key Expressions or Variables:** `={{ $json.route === 'execute' }}`
- **Input and Output Connections:** Connected from `Code - Route Request`; true branch outputs to `HTTP Request - Enrichment API`, false branch outputs directly to `Respond to Webhook - Result`.
- **Version-Specific Requirements:** Type version 2.2.
- **Edge Cases / Failure Types:** None.
- **Sub-workflow Reference:** None.

---

### 2.4 API Processing Pipeline

#### Overview
This block executes the sequence of external API calls for lead enrichment, scoring, CRM synchronization, and team notifications when a fresh execution lock is acquired.

#### Nodes Involved
- `HTTP Request - Enrichment API`
- `Wait - After Enrichment`
- `Code - Score Lead`
- `HTTP Request - CRM API`
- `Wait - After CRM`
- `HTTP Request - Notification API`
- `Code - Classify & Finalize`

#### Node Details

##### HTTP Request - Enrichment API
- **Type and Technical Role:** `n8n-nodes-base.httpRequest` (v4.2) - Sends lead data to an external enrichment service.
- **Configuration Choices:** Method set to `POST` with JSON body payload and custom headers including `Idempotency-Key`.
- **Key Expressions or Variables:** `={{ $json.provider.enrichUrl }}`, `={{ $json.config.enrichTimeoutMs }}`, `={{ JSON.stringify($json.provider.enrichBody) }}`, `={{ $json.idemKey }}`
- **Input and Output Connections:** Connected from `IF - Execute Processing?` (True branch); outputs to `Wait - After Enrichment`.
- **Version-Specific Requirements:** Type version 4.2. `continueOnFail` is enabled.
- **Edge Cases / Failure Types:** External API timeouts, HTTP 4xx/5xx responses.
- **Sub-workflow Reference:** None.

##### Wait - After Enrichment
- **Type and Technical Role:** `n8n-nodes-base.wait` (v1.1) - Introduces a brief delay for rate-limiting or sequencing.
- **Configuration Choices:** Pause duration set to 2 seconds.
- **Key Expressions or Variables:** N/A.
- **Input and Output Connections:** Connected from `HTTP Request - Enrichment API`; outputs to `Code - Score Lead`.
- **Version-Specific Requirements:** Type version 1.1.
- **Edge Cases / Failure Types:** None.
- **Sub-workflow Reference:** None.

##### Code - Score Lead
- **Type and Technical Role:** `n8n-nodes-base.code` (v2) - Computes a deterministic lead score and tier classification based on enrichment results and request attributes.
- **Configuration Choices:** Applies point thresholds based on company presence, phone availability, lead source, and enrichment response codes.
- **Key Expressions or Variables:** References `Code - Route Request` and input data from `Wait - After Enrichment`.
- **Input and Output Connections:** Connected from `Wait - After Enrichment`; outputs to `HTTP Request - CRM API`.
- **Version-Specific Requirements:** Type version 2.
- **Edge Cases / Failure Types:** Malformed JSON responses from the enrichment service.
- **Sub-workflow Reference:** None.

##### HTTP Request - CRM API
- **Type and Technical Role:** `n8n-nodes-base.httpRequest` (v4.2) - Creates or updates the lead record in the CRM system.
- **Configuration Choices:** Method set to `POST` with JSON body and idempotency header.
- **Key Expressions or Variables:** `={{ $json.provider.crmUrl }}`, `={{ $json.config.crmTimeoutMs }}`, `={{ JSON.stringify($json.provider.crmBody) }}`, `={{ $json.idemKey }}`
- **Input and Output Connections:** Connected from `Code - Score Lead`; outputs to `Wait - After CRM`.
- **Version-Specific Requirements:** Type version 4.2. `continueOnFail` is enabled.
- **Edge Cases / Failure Types:** CRM validation rejection or network timeouts.
- **Sub-workflow Reference:** None.

##### Wait - After CRM
- **Type and Technical Role:** `n8n-nodes-base.wait` (v1.1) - Pause node before triggering notifications.
- **Configuration Choices:** Pause duration set to 1 second.
- **Key Expressions or Variables:** N/A.
- **Input and Output Connections:** Connected from `HTTP Request - CRM API`; outputs to `HTTP Request - Notification API`.
- **Version-Specific Requirements:** Type version 1.1.
- **Edge Cases / Failure Types:** None.
- **Sub-workflow Reference:** None.

##### HTTP Request - Notification API
- **Type and Technical Role:** `n8n-nodes-base.httpRequest` (v4.2) - Sends team notifications (e.g., Slack or internal webhook).
- **Configuration Choices:** Method set to `POST` with JSON body containing notification text and lead payload details.
- **Key Expressions or Variables:** `={{ $json.provider.notifyUrl }}`, `={{ $json.config.notifyTimeoutMs }}`, `={{ JSON.stringify($json.provider.notifyBody) }}`
- **Input and Output Connections:** Connected from `Wait - After CRM`; outputs to `Code - Classify & Finalize`.
- **Version-Specific Requirements:** Type version 4.2. `continueOnFail` is enabled.
- **Edge Cases / Failure Types:** Notification endpoint unavailability.
- **Sub-workflow Reference:** None.

##### Code - Classify & Finalize
- **Type and Technical Role:** `n8n-nodes-base.code` (v2) - JavaScript node determining the final execution status (`completed` or `failed`) and assembling the response object.
- **Configuration Choices:** Evaluates CRM status codes to map success, client rejection, or retryable failure outcomes.
- **Key Expressions or Variables:** References data from `Code - Score Lead`, `HTTP Request - CRM API`, and `HTTP Request - Notification API`.
- **Input and Output Connections:** Connected from `HTTP Request - Notification API`; outputs to `Postgres - Store Result`.
- **Version-Specific Requirements:** Type version 2.
- **Edge Cases / Failure Types:** Unexpected status code mappings.
- **Sub-workflow Reference:** None.

---

### 2.5 Persistence & Response Delivery

#### Overview
This block stores the final execution result back into PostgreSQL under strict fencing constraints and returns the response payload along with required idempotency headers to the webhook caller.

#### Nodes Involved
- `Postgres - Store Result`
- `Respond to Webhook - Result`

#### Node Details

##### Postgres - Store Result
- **Type and Technical Role:** `n8n-nodes-base.postgres` (v2.5) - Executes an UPDATE query using optimistic locking/fencing on `attempts` and `in_progress` status to persist the final response.
- **Configuration Choices:** Operation set to `executeQuery`. Query parameters mapped via expression replacements.
- **Key Expressions or Variables:** `={{ [ $json.idemKey, $json.attempt, $json.finalStatus, $json.httpStatus, JSON.stringify($json.body), $json.providerRef, $json.providerStatus ] }}`
- **Input and Output Connections:** Connected from `Code - Classify & Finalize`; outputs to `Respond to Webhook - Result`.
- **Version-Specific Requirements:** Type version 2.5. `continueOnFail` and `alwaysOutputData` are enabled.
- **Edge Cases / Failure Types:** Database write conflicts if lease expired concurrently.
- **Sub-workflow Reference:** None.

##### Respond to Webhook - Result
- **Type and Technical Role:** `n8n-nodes-base.respondToWebhook` (v1.1) - Returns the final HTTP response to the webhook caller.
- **Configuration Choices:** Responds with `json` data, dynamic response codes, and custom headers (`Idempotency-Key`, `Idempotent-Replayed`).
- **Key Expressions or Variables:** 
  - Status code: `={{ $json.httpStatus || $('Code - Classify & Finalize').first().json.httpStatus }}`
  - Header `Idempotency-Key`: `={{ $json.idemKey || $('Code - Classify & Finalize').first().json.idemKey }}`
  - Header `Idempotent-Replayed`: `={{ String($json.replayed === true) }}`
  - Response Body: `={{ $json.body || $('Code - Classify & Finalize').first().json.body }}`
- **Input and Output Connections:** Connected from `Postgres - Store Result` and the false branch of `IF - Execute Processing?`.
- **Version-Specific Requirements:** Type version 1.1.
- **Edge Cases / Failure Types:** Missing upstream payload references if execution flow paths diverge unexpectedly.
- **Sub-workflow Reference:** None.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note - Overview | n8n-nodes-base.stickyNote | Visual documentation note providing high-level architecture details. | None | None | ## Idempotent Lead Management<br><br>### How it works<br>A single POST with a client-supplied Idempotency-Key is guaranteed to process the lead at most once, even under retries, timeouts or concurrent duplicates.<br><br>1. Webhook validates + fingerprints the payload, then atomically claims the key in Postgres.<br>2. **New key** → continues to enrichment → score → CRM → notify.<br>3. **Completed key** → stored response is replayed as-is.<br>4. **Key in flight** → returns 409 so the client retries shortly.<br>5. **Key reused with different payload** → rejected with 422.<br><br>Only a freshly claimed key reaches the external APIs. The same Idempotency-Key is forwarded where useful.<br><br>### Node budget<br>- 4 Code nodes<br>- 3 HTTP Request (API) nodes<br>- 2 Wait nodes<br><br>### Setup<br>- Add Postgres credentials to the three Postgres nodes.<br>- Replace the three provider URLs in CONFIG (Code - Validate & Fingerprint).<br>- Review leaseSeconds, timeouts, allowedSources, maxScoreThreshold.<br>- Test with curl: two identical POSTs with the same Idempotency-Key must return the same body (second has Idempotent-Replayed: true).<br>- Set allowSimulation=false in production. |
| Sticky Note - Intake | n8n-nodes-base.stickyNote | Visual documentation note for intake setup. | None | None | ## 1. Intake & store setup<br>Receives the lead and ensures the idempotency_keys table + unique indexes exist (self-provisioning). |
| Sticky Note - Validate & claim | n8n-nodes-base.stickyNote | Visual documentation note for validation and claiming logic. | None | None | ## 2. Validate & claim the key<br>Validates headers/payload, scopes Idempotency-Key per client, fingerprints the canonical body, then claims the key in one atomic SQL statement. |
| Sticky Note - Route | n8n-nodes-base.stickyNote | Visual documentation note for routing logic. | None | None | ## 3. Route the request<br>Interprets claim result → execute / replay / in_progress / mismatch / duplicate / invalid / store_unavailable.<br>Only "execute" continues to the APIs. |
| Sticky Note - Process & respond | n8n-nodes-base.stickyNote | Visual documentation note covering downstream processing and response delivery. | None | None | ## 4. Process <br><br>Processes a claimed lead through enrichment, scoring, CRM update, notification, and final storage before responding. |
| Webhook - Lead Submission | n8n-nodes-base.webhook | Entry: POST /idempotent-lead with Idempotency-Key (optional X-Client-Id) + JSON lead body. | None | Postgres - Ensure Idempotency Store | ## 1. Intake & store setup<br>Receives the lead and ensures the idempotency_keys table + unique indexes exist (self-provisioning). |
| Postgres - Ensure Idempotency Store | n8n-nodes-base.postgres | Creates idempotency_keys table + indexes if missing. | Webhook - Lead Submission | Code - Validate & Fingerprint | ## 1. Intake & store setup<br>Receives the lead and ensures the idempotency_keys table + unique indexes exist (self-provisioning). |
| Code - Validate & Fingerprint | n8n-nodes-base.code | Validates lead payload, scopes key per client, computes SHA-256 fingerprint of canonical body. | Postgres - Ensure Idempotency Store | Postgres - Claim Idempotency Key | ## 2. Validate & claim the key<br>Validates headers/payload, scopes Idempotency-Key per client, fingerprints the canonical body, then claims the key in one atomic SQL statement. |
| Postgres - Claim Idempotency Key | n8n-nodes-base.postgres | Atomic claim with lease. Returns existing row if already present. | Code - Validate & Fingerprint | Code - Route Request | ## 2. Validate & claim the key<br>Validates headers/payload, scopes Idempotency-Key per client, fingerprints the canonical body, then claims the key in one atomic SQL statement. |
| Code - Route Request | n8n-nodes-base.code | Decides execute / replay / in_progress / mismatch / duplicate / invalid / store_unavailable. | Postgres - Claim Idempotency Key | IF - Execute Processing? | ## 3. Route the request<br>Interprets claim result → execute / replay / in_progress / mismatch / duplicate / invalid / store_unavailable.<br>Only "execute" continues to the APIs. |
| IF - Execute Processing? | n8n-nodes-base.if | True only for a freshly claimed key. | Code - Route Request | HTTP Request - Enrichment API, Respond to Webhook - Result | ## 3. Route the request<br>Interprets claim result → execute / replay / in_progress / mismatch / duplicate / invalid / store_unavailable.<br>Only "execute" continues to the APIs. |
| HTTP Request - Enrichment API | n8n-nodes-base.httpRequest | First external call – enrich lead data (company, social, etc.). | IF - Execute Processing? | Wait - After Enrichment | ## 4. Process <br><br>Processes a claimed lead through enrichment, scoring, CRM update, notification, and final storage before responding. |
| Wait - After Enrichment | n8n-nodes-base.wait | Short pause after enrichment (rate-limit / sequencing). | HTTP Request - Enrichment API | Code - Score Lead | ## 4. Process <br><br>Processes a claimed lead through enrichment, scoring, CRM update, notification, and final storage before responding. |
| Code - Score Lead | n8n-nodes-base.code | Combines original request + enrichment result into a lead score and prepares CRM payload. | Wait - After Enrichment | HTTP Request - CRM API | ## 4. Process <br><br>Processes a claimed lead through enrichment, scoring, CRM update, notification, and final storage before responding. |
| HTTP Request - CRM API | n8n-nodes-base.httpRequest | Second external call – create / update lead in CRM. | Code - Score Lead | Wait - After CRM | ## 4. Process <br><br>Processes a claimed lead through enrichment, scoring, CRM update, notification, and final storage before responding. |
| Wait - After CRM | n8n-nodes-base.wait | Short pause after CRM write before notification. | HTTP Request - CRM API | HTTP Request - Notification API | ## 4. Process <br><br>Processes a claimed lead through enrichment, scoring, CRM update, notification, and final storage before responding. |
| HTTP Request - Notification API | n8n-nodes-base.httpRequest | Third external call – Slack / email / internal webhook. | Wait - After CRM | Code - Classify & Finalize | ## 4. Process <br><br>Processes a claimed lead through enrichment, scoring, CRM update, notification, and final storage before responding. |
| Code - Classify & Finalize | n8n-nodes-base.code | Maps enrichment + CRM + notify outcomes into final status (completed / failed) for storage. | HTTP Request - Notification API | Postgres - Store Result | ## 4. Process <br><br>Processes a claimed lead through enrichment, scoring, CRM update, notification, and final storage before responding. |
| Postgres - Store Result | n8n-nodes-base.postgres | Fencing write – only the current owner (matching attempts + in_progress) can store the outcome. | Code - Classify & Finalize | Respond to Webhook - Result | ## 4. Process <br><br>Processes a claimed lead through enrichment, scoring, CRM update, notification, and final storage before responding. |
| Respond to Webhook - Result | n8n-nodes-base.respondToWebhook | Returns fresh, replayed or error response with Idempotency-Key + Idempotent-Replayed headers. | Postgres - Store Result, IF - Execute Processing? | None | ## 4. Process <br><br>Processes a claimed lead through enrichment, scoring, CRM update, notification, and final storage before responding. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**
   - Type: `n8n-nodes-base.webhook`
   - Name: `Webhook - Lead Submission`
   - Parameters: Set HTTP Method to `POST`, Path to `idempotent-lead`, and Response Mode to `responseNode`.
   - Credentials: None required.

2. **Create Postgres Ensure Store Node**
   - Type: `n8n-nodes-base.postgres`
   - Name: `Postgres - Ensure Idempotency Store`
   - Parameters: Operation = `executeQuery`. Query = DDL script creating `idempotency_keys` table and indexes (`CREATE TABLE IF NOT EXISTS idempotency_keys ...`).
   - Credentials: Configure PostgreSQL credentials. Set `continueOnFail` to true.
   - Connection: Connect `Webhook - Lead Submission` (main) to `Postgres - Ensure Idempotency Store` (main).

3. **Create Validation & Fingerprint Code Node**
   - Type: `n8n-nodes-base.code`
   - Name: `Code - Validate & Fingerprint`
   - Parameters: JavaScript code block validating payload fields, configuring policy defaults (`CONFIG`), and calculating SHA-256 fingerprints.
   - Connection: Connect `Postgres - Ensure Idempotency Store` (main) to `Code - Validate & Fingerprint` (main).

4. **Create Postgres Claim Key Node**
   - Type: `n8n-nodes-base.postgres`
   - Name: `Postgres - Claim Idempotency Key`
   - Parameters: Operation = `executeQuery`. Query = parameterized CTE performing atomic lease insertion/upsert. Query Replacements expression = `={{ [ $json.idemKey, $json.businessKey, $json.fingerprint, $json.config.leaseSeconds, $json.valid ] }}`.
   - Credentials: PostgreSQL credentials. Enable `continueOnFail` and `alwaysOutputData`.
   - Connection: Connect `Code - Validate & Fingerprint` (main) to `Postgres - Claim Idempotency Key` (main).

5. **Create Route Request Code Node**
   - Type: `n8n-nodes-base.code`
   - Name: `Code - Route Request`
   - Parameters: JavaScript code evaluating claim output rows to determine routing outcomes (`execute`, `replay`, etc.).
   - Connection: Connect `Postgres - Claim Idempotency Key` (main) to `Code - Route Request` (main).

6. **Create IF Router Node**
   - Type: `n8n-nodes-base.if`
   - Name: `IF - Execute Processing?`
   - Parameters: Condition checking `={{ $json.route === 'execute' }}`.
   - Connection: Connect `Code - Route Request` (main) to `IF - Execute Processing?` (main).

7. **Create Enrichment HTTP Request Node**
   - Type: `n8n-nodes-base.httpRequest`
   - Name: `HTTP Request - Enrichment API`
   - Parameters: Method = `POST`, URL = `={{ $json.provider.enrichUrl }}`, Timeout = `={{ $json.config.enrichTimeoutMs }}`, Body = JSON stringified enrichment body, Headers include `Idempotency-Key` and `content-type`. Enable `continueOnFail`.
   - Connection: Connect `IF - Execute Processing?` (True branch, index 0) to `HTTP Request - Enrichment API` (main).

8. **Create Wait After Enrichment Node**
   - Type: `n8n-nodes-base.wait`
   - Name: `Wait - After Enrichment`
   - Parameters: Amount = 2 seconds.
   - Connection: Connect `HTTP Request - Enrichment API` (main) to `Wait - After Enrichment` (main).

9. **Create Score Lead Code Node**
   - Type: `n8n-nodes-base.code`
   - Name: `Code - Score Lead`
   - Parameters: JavaScript code computing lead scores and tiers.
   - Connection: Connect `Wait - After Enrichment` (main) to `Code - Score Lead` (main).

10. **Create CRM HTTP Request Node**
    - Type: `n8n-nodes-base.httpRequest`
    - Name: `HTTP Request - CRM API`
    - Parameters: Method = `POST`, URL = `={{ $json.provider.crmUrl }}`, Timeout = `={{ $json.config.crmTimeoutMs }}`, Body = JSON stringified CRM body, Headers include `Idempotency-Key`. Enable `continueOnFail`.
    - Connection: Connect `Code - Score Lead` (main) to `HTTP Request - CRM API` (main).

11. **Create Wait After CRM Node**
    - Type: `n8n-nodes-base.wait`
    - Name: `Wait - After CRM`
    - Parameters: Amount = 1 second.
    - Connection: Connect `HTTP Request - CRM API` (main) to `Wait - After CRM` (main).

12. **Create Notification HTTP Request Node**
    - Type: `n8n-nodes-base.httpRequest`
    - Name: `HTTP Request - Notification API`
    - Parameters: Method = `POST`, URL = `={{ $json.provider.notifyUrl }}`, Timeout = `={{ $json.config.notifyTimeoutMs }}`, Body = JSON stringified notification payload. Enable `continueOnFail`.
    - Connection: Connect `Wait - After CRM` (main) to `HTTP Request - Notification API` (main).

13. **Create Classify & Finalize Code Node**
    - Type: `n8n-nodes-base.code`
    - Name: `Code - Classify & Finalize`
    - Parameters: JavaScript code mapping API responses to final completion or failure states.
    - Connection: Connect `HTTP Request - Notification API` (main) to `Code - Classify & Finalize` (main).

14. **Create Postgres Store Result Node**
    - Type: `n8n-nodes-base.postgres`
    - Name: `Postgres - Store Result`
    - Parameters: Operation = `executeQuery`. UPDATE query with fencing conditions. Query replacements expression = `={{ [ $json.idemKey, $json.attempt, $json.finalStatus, $json.httpStatus, JSON.stringify($json.body), $json.providerRef, $json.providerStatus ] }}`. Enable `continueOnFail` and `alwaysOutputData`.
    - Connection: Connect `Code - Classify & Finalize` (main) to `Postgres - Store Result` (main).

15. **Create Respond to Webhook Node**
    - Type: `n8n-nodes-base.respondToWebhook`
    - Name: `Respond to Webhook - Result`
    - Parameters: Respond with JSON, configure response code and headers (`Idempotency-Key`, `Idempotent-Replayed`).
    - Connections: 
      - Connect `Postgres - Store Result` (main) to `Respond to Webhook - Result` (main).
      - Connect `IF - Execute Processing?` (False branch, index 1) to `Respond to Webhook - Result` (main).

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| External provider URL placeholders | Currently pointing to `httpbin.org`. Replace with production Enrichment, CRM, and Notification endpoints before deployment. |
| Production configuration hardening | Ensure `allowSimulation` is set to `false` in production environments within the validation code node. |