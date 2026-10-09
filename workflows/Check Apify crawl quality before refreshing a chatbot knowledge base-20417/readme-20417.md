Check Apify crawl quality before refreshing a chatbot knowledge base

https://n8nworkflows.xyz/workflows/check-apify-crawl-quality-before-refreshing-a-chatbot-knowledge-base-20417


# Check Apify crawl quality before refreshing a chatbot knowledge base

### 1. Workflow Overview

This workflow automates the validation of a completed Apify web crawler run before updating a chatbot knowledge base or external data destination. Its core purpose is to prevent corrupted, incomplete, or flawed crawler exports from poisoning downstream search indexes or databases. 

The process is divided into three functional blocks:
- **1.1 Input Reception & Validation:** Initializes user-defined configuration parameters (run ID, namespace, expected URLs) and validates them against structural and logical constraints.
- **1.2 Source Verification & Quality Audit Execution:** Communicates with the Apify API to confirm the source crawl run finished successfully, then triggers the Apify Crawl Quality Gate actor to evaluate the dataset for missing required pages, duplicates, or format violations.
- **1.3 Report Analysis, Decision Routing & Branching:** Fetches the audit results from the Apify key-value store, rigorously validates the report schema, and routes the workflow into either a `PASS` branch (ready for destination updates) or a `BLOCK` branch (retaining the existing knowledge base).

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Reception & Validation
- **Overview:** This block initializes the workflow via a manual trigger, defines the baseline parameters required to evaluate an Apify crawler run, and enforces strict pre-flight validation rules before any remote API calls are made.
- **Nodes Involved:** 
  - `Start manually`
  - `Configure source`
  - `Validate configuration`

- **Node Details:**
  - **Start manually**
    - *Type & Technical Role:* `n8n-nodes-base.manualTrigger` (Workflow Entry Point).
    - *Configuration:* Default settings.
    - *Input / Output:* No inputs; outputs a single execution trigger object.
    - *Edge Cases:* None; execution is strictly user-initiated.
  - **Configure source**
    - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript Data Injector).
    - *Configuration:* Returns a hardcoded JSON object containing placeholder values for the source crawl run ID, stable namespace, expected URL array, upstream failed requests count (`null` by default), and previous manifest reference.
    - *Key Expressions:* Custom JavaScript object definition.
    - *Input / Output:* Input: `Start manually` $\rightarrow$ Output: Single item containing the source configuration payload.
    - *Edge Cases:* Placeholder values must be manually replaced before execution.
  - **Validate configuration**
    - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript Validator).
    - *Configuration:* Validates data types and regular expression formats for `sourceRunId` (17 alphanumeric characters), `namespace` (1–80 alphanumeric characters/hyphens/underscores), `upstreamFailedRequests` (must be an integer strictly equal to `0`), and `expectedUrls` (valid HTTP/HTTPS array up to 1,000 items). Throws explicit errors if checks fail.
    - *Key Expressions:* Regex checks (`/^[A-Za-z0-9]{17}$/`, `/^https?:\/\/[^\s]+$/`).
    - *Input / Output:* Input: `Configure source` $\rightarrow$ Output: Validated configuration payload.
    - *Edge Cases:* Halts execution if `upstreamFailedRequests` is non-zero, null, or if expected URLs are missing without a `previousManifest`.

---

#### Block 2.2: Source Verification & Quality Audit Execution
- **Overview:** This block queries the Apify API to inspect the status of the target crawl run, ensures it completed without failure, and triggers an automated downstream audit via the Crawl Quality Gate actor.
- **Nodes Involved:**
  - `Read source run`
  - `Require completed source`
  - `Run crawl audit`
  - `Require successful audit`

- **Node Details:**
  - **Read source run**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (REST API Client).
    - *Configuration:* Performs a `GET` request against the Apify API to fetch metadata for the specified actor run. Configured with a 90-second timeout and JSON response parsing.
    - *Key Expressions:* `={ 'https://api.apify.com/v2/actor-runs/' + $json.sourceRunId }`
    - *Credentials:* Uses Apify Header Authentication (`Authorization: Bearer <token>`).
    - *Input / Output:* Input: `Validate configuration` $\rightarrow$ Output: Apify actor run details object (`data`).
    - *Edge Cases:* API authentication failures, network timeouts, or invalid run IDs causing 404 responses. Set to `onError: stopWorkflow`.
  - **Require completed source**
    - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript Gatekeeper).
    - *Configuration:* Verifies that the source run status is `SUCCEEDED`, includes a valid completion timestamp, and exposes a valid 17-character `defaultDatasetId`. Constructs the JSON payload for the audit actor.
    - *Input / Output:* Input: `Read source run` $\rightarrow$ Output: Audit input payload and source run ID.
    - *Edge Cases:* Halts execution if the source crawl failed, is still running, or lacks a valid dataset.
  - **Run crawl audit**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (REST API Client - Actor Execution).
    - *Configuration:* Performs a `POST` request to trigger the `obeying_laureate~crawl-quality-gate` actor. Passes query parameters enforcing a maximum charge cap ($0.10), execution timeout (60s), memory limits (256MB), and `waitForFinish=60`.
    - *Credentials:* Uses Apify Header Authentication (`Authorization: Bearer <token>`).
    - *Input / Output:* Input: `Require completed source` $\rightarrow$ Output: Audit actor run execution details.
    - *Edge Cases:* Timeout during audit execution or credit/permission limits exceeded on the Apify account.
  - **Require successful audit**
    - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript Validator).
    - *Configuration:* Inspects the audit actor run object to verify it completed successfully (`SUCCEEDED`) and outputs a valid 17-character `defaultKeyValueStoreId` and audit run ID.
    - *Input / Output:* Input: `Run crawl audit` $\rightarrow$ Output: Audit run ID and Key-Value Store ID.
    - *Edge Cases:* Throws an error if the audit actor fails or times out before generating a store ID.

---

#### Block 2.3: Report Analysis, Decision Routing & Branching
- **Overview:** This block fetches the audit report from the Apify Key-Value Store, verifies its cryptographic and structural schema integrity, and routes execution into either a success (`PASS`) or failure (`BLOCK`) branch.
- **Nodes Involved:**
  - `Read audit report`
  - `Validate decision`
  - `Gate passed?`
  - `PASS - connect your refresh here`
  - `BLOCK - keep existing knowledge base`

- **Node Details:**
  - **Read audit report**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (REST API Client).
    - *Configuration:* Performs a `GET` request to retrieve the `REPORT` record from the audit run's Key-Value Store.
    - *Key Expressions:* `={ 'https://api.apify.com/v2/key-value-stores/' + $json.storeId + '/records/REPORT' }`
    - *Credentials:* Uses Apify Header Authentication (`Authorization: Bearer <token>`).
    - *Input / Output:* Input: `Require successful audit` $\rightarrow$ Output: Complete audit report JSON object.
    - *Edge Cases:* Missing record or storage read failures.
  - **Validate decision**
    - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript Schema Validator).
    - *Configuration:* Performs rigorous validation on the report object (verifying schema version, namespace match, dataset ID match, error counts, gate status booleans, and manifest acceptability). Returns a structured summary including audit and source run identifiers alongside a direct Apify console URL.
    - *Input / Output:* Input: `Read audit report` $\rightarrow$ Output: Standardized decision object (`gatePassed`, `reportUrl`, etc.).
    - *Edge Cases:* Malformed report schemas or mismatched namespaces trigger immediate execution failure.
  - **Gate passed?**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional Router).
    - *Configuration:* Evaluates whether `{{ $json.gatePassed }}` evaluates strictly to `true`.
    - *Input / Output:* Input: `Validate decision` $\rightarrow$ Output: Routes to index 0 (true) or index 1 (false).
  - **PASS - connect your refresh here**
    - *Type & Technical Role:* `n8n-nodes-base.noOp` (Placeholder / Target Node).
    - *Configuration:* Acts as an anchor point for connecting downstream vector database or knowledge base refresh workflows.
    - *Input / Output:* Input: `Gate passed?` (True branch) $\rightarrow$ Output: None (terminal branch until extended).
    - *Edge Cases:* Ensure destination writes occur *only* after this node.
  - **BLOCK - keep existing knowledge base**
    - *Type & Technical Role:* `n8n-nodes-base.noOp` (Placeholder / Terminal Node).
    - *Configuration:* Acts as an anchor point for handling blocked crawl runs. Intentionally performs no write actions to preserve existing data.
    - *Input / Output:* Input: `Gate passed?` (False branch) $\rightarrow$ Output: None (terminal branch).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Start manually** | `manualTrigger` | Workflow Entry Point | None | Configure source | Check Apify crawl exports before refreshing a chatbot<br><br>Who it is for<br>Agencies and developers refreshing a chatbot or knowledge base from an existing Apify crawl.<br><br>What it does<br>Checks a finished crawler export for missing required pages, coverage loss, duplicates, recorded failures and short text. It verifies the source run, starts Crawl Quality Gate, reads its report and routes a valid PASS or BLOCK. Errors stop the workflow. Both branches initially end without writing anywhere.<br><br>Set up<br>1. Import the workflow into n8n.<br>2. In Configure source, enter your finished crawler run ID, stable namespace, required URLs and explicitly reviewed failed-request count.<br>3. Select your Apify Header Auth credential in all three HTTP Request nodes. Keep tokens in credentials, outside the workflow JSON.<br>4. Run manually. Add an absent required URL to reproduce BLOCK.<br>5. Connect only PASS to your existing refresh. Save the candidate manifest only after that update succeeds.<br><br>Requirements and costs<br>An Apify account, a finished accessible crawl and n8n are required. The template is free. Each real completed audit, including BLOCK, costs $0.10; the workflow caps its audit charge at $0.10 and does not retry automatically. Other platform costs may apply. A free synthetic demo is available on the [product page](https://apify.com/obeying_laureate/crawl-quality-gate).<br><br>Customize carefully<br>Destination writes, scheduling and persistent baseline storage are not included. PASS checks supplied rules; it does not prove whole-site completeness or authorize deletion. Verified on n8n 2.6.4. |
| **Configure source** | `code` | Data Injector | Start manually | Validate configuration | 1. Configure and verify the source<br>Edit the grouped values in **Configure source**. The failed-request count starts as null so the example stops until you review it. The source run must be finished and successful. Keep its dataset immutable. |
| **Validate configuration** | `code` | Parameter Validator | Configure source | Read source run | 1. Configure and verify the source<br>Edit the grouped values in **Configure source**. The failed-request count starts as null so the example stops until you review it. The source run must be finished and successful. Keep its dataset immutable. |
| **Read source run** | `httpRequest` | REST API Client | Validate configuration | Require completed source | Select your Apify Header Auth credential. Name: Authorization; value: Bearer followed by your token. Never paste a token into this workflow JSON. |
| **Require completed source** | `code` | Validation Gate | Read source run | Run crawl audit | |
| **Run crawl audit** | `httpRequest` | REST API Client | Require completed source | Require successful audit | Select your Apify Header Auth credential. Name: Authorization; value: Bearer followed by your token. Never paste a token into this workflow JSON.<br><br>2. Audit and validate the result<br>Use Header Auth credentials in each HTTP node. A completed audit can be PASS or BLOCK. Errors and malformed responses stop the workflow. The $0.10 audit cap is set in the request; another start request may incur another charge. |
| **Require successful audit** | `code` | Validation Gate | Run crawl audit | Read audit report | 2. Audit and validate the result<br>Use Header Auth credentials in each HTTP node. A completed audit can be PASS or BLOCK. Errors and malformed responses stop the workflow. The $0.10 audit cap is set in the request; another start request may incur another charge. |
| **Read audit report** | `httpRequest` | REST API Client | Require successful audit | Validate decision | Select your Apify Header Auth credential. Name: Authorization; value: Bearer followed by your token. Never paste a token into this workflow JSON.<br><br>2. Audit and validate the result<br>Use Header Auth credentials in each HTTP node. A completed audit can be PASS or BLOCK. Errors and malformed responses stop the workflow. The $0.10 audit cap is set in the request; another start request may incur another charge. |
| **Validate decision** | `code` | Schema Validator | Read audit report | Gate passed? | 2. Audit and validate the result<br>Use Header Auth credentials in each HTTP node. A completed audit can be PASS or BLOCK. Errors and malformed responses stop the workflow. The $0.10 audit cap is set in the request; another start request may incur another charge. |
| **Gate passed?** | `if` | Conditional Router | Validate decision | PASS / BLOCK nodes | 3. Route before updating<br>Both branches end without writes. Attach your existing destination update only to PASS. Keep old data on BLOCK/error. Store report.manifest only after a successful update. PASS never authorizes deleting records. |
| **PASS - connect your refresh here** | `noOp` | Success Destination Anchor | Gate passed? | None | Only attach the destination refresh to this branch. Read warnings. Persist report.manifest only after the refresh succeeds. PASS is not permission to delete records.<br><br>3. Route before updating<br>Both branches end without writes. Attach your existing destination update only to PASS. Keep old data on BLOCK/error. Store report.manifest only after a successful update. PASS never authorizes deleting records. |
| **BLOCK - keep existing knowledge base** | `noOp` | Failure Destination Anchor | Gate passed? | None | Inspect report.findings. Leave the destination and committed manifest unchanged. This branch intentionally ends without writes.<br><br>3. Route before updating<br>Both branches end without writes. Attach your existing destination update only to PASS. Keep old data on BLOCK/error. Store report.manifest only after a successful update. PASS never authorizes deleting records. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Entry Point:**
   - Add a **Manual Trigger** node (`n8n-nodes-base.manualTrigger`). Name it `Start manually`.
2. **Add Source Configuration:**
   - Add a **Code** node (`n8n-nodes-base.code`) named `Configure source`. Set the mode to JavaScript and return an array containing `sourceRunId`, `namespace`, `expectedUrls`, `upstreamFailedRequests` (set to `null` initially), and `previousManifest`.
   - Connect `Start manually` $\rightarrow$ `Configure source`.
3. **Add Configuration Validation:**
   - Add a **Code** node named `Validate configuration`. Implement validation checks ensuring `sourceRunId` is a 17-character string, `namespace` matches slug constraints, `upstreamFailedRequests` is explicitly `0`, and `expectedUrls` contains valid HTTP/HTTPS URLs.
   - Connect `Configure source` $\rightarrow$ `Validate configuration`.
4. **Fetch Source Run Metadata:**
   - Add an **HTTP Request** node named `Read source run`. Set method to `GET`, URL to `={{ 'https://api.apify.com/v2/actor-runs/' + $json.sourceRunId }}`, and configure authentication to use generic HTTP Header Auth (`Authorization: Bearer <token>`). Set response format to JSON and timeout to 90 seconds.
   - Connect `Validate configuration` $\rightarrow$ `Read source run`.
5. **Validate Source Run Status:**
   - Add a **Code** node named `Require completed source`. Verify the run status is `SUCCEEDED`, has a valid completion timestamp, and exposes a `defaultDatasetId`. Construct the `auditInput` payload object.
   - Connect `Read source run` $\rightarrow$ `Require completed source`.
6. **Trigger the Quality Audit Actor:**
   - Add an **HTTP Request** node named `Run crawl audit`. Set method to `POST`, URL to `https://api.apify.com/v2/acts/obeying_laureate~crawl-quality-gate/runs`. Enable JSON body sending using `={{ $json.auditInput }}`. Add query parameters: `waitForFinish=60`, `timeout=60`, `memory=256`, `maxTotalChargeUsd=0.10`, `restartOnError=false`, `forcePermissionLevel=LIMITED_PERMISSIONS`. Configure HTTP Header Auth.
   - Connect `Require completed source` $\rightarrow$ `Run crawl audit`.
7. **Validate Audit Execution:**
   - Add a **Code** node named `Require successful audit`. Check that the audit run status is `SUCCEEDED` and extract `defaultKeyValueStoreId` and `run.id`.
   - Connect `Run crawl audit` $\rightarrow$ `Require successful audit`.
8. **Fetch Audit Report:**
   - Add an **HTTP Request** node named `Read audit report`. Set method to `GET`, URL to `={{ 'https://api.apify.com/v2/key-value-stores/' + $json.storeId + '/records/REPORT' }}`. Configure HTTP Header Auth.
   - Connect `Require successful audit` $\rightarrow$ `Read audit report`.
9. **Validate Report Schema & Decision:**
   - Add a **Code** node named `Validate decision`. Validate report properties including `schemaVersion`, `demo`, `gatePassed`, and `summary.errorCount`. Return structured decision flags and Apify console report URLs.
   - Connect `Read audit report` $\rightarrow$ `Validate decision`.
10. **Configure Conditional Routing:**
    - Add an **If** node (`n8n-nodes-base.if`) named `Gate passed?`. Set condition to evaluate whether `{{ $json.gatePassed }}` strictly equals `true`.
    - Connect `Validate decision` $\rightarrow$ `Gate passed?`.
11. **Create Termination Branches:**
    - Add a **No-Op** node named `PASS - connect your refresh here`. Connect index 0 of `Gate passed?` to this node.
    - Add a **No-Op** node named `BLOCK - keep existing knowledge base`. Connect index 1 of `Gate passed?` to this node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Product page, synthetic demo, and setup guide | [Apify Crawl Quality Gate Product Page](https://apify.com/obeying_laureate/crawl-quality-gate) |
| Platform Compatibility & Versioning | Verified on n8n version 2.6.4 |