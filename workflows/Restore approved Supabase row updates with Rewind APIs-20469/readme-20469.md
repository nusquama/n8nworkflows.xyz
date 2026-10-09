Restore approved Supabase row updates with Rewind APIs

https://n8nworkflows.xyz/workflows/restore-approved-supabase-row-updates-with-rewind-apis-20469


# Restore approved Supabase row updates with Rewind APIs

### 1. Workflow Overview

This workflow automates the recovery of human-approved Supabase row updates captured via Rewind APIs. Its primary purpose is to safely process an approved recovery job, inspect the current state of the target Supabase record to detect any subsequent conflicting edits, conditionally restore the original pre-update data using a transactional RPC call, and report the final outcome back to Rewind.

The logic is grouped into the following functional blocks:
- **1.1 Initialization and Validation:** Handles manual or scheduled execution, sets up target service endpoints and connection parameters, and strictly validates URL formats.
- **1.2 Job Claiming and Analysis:** Claims an approved recovery job from Rewind and validates the payload structure, table alias, row ID, and operation constraints.
- **1.3 State Checking:** Reads the current record from Supabase and checks whether the row version and field values still match the expected revision before restoration.
- **1.4 Conditional Restoration and Reporting:** Evaluates whether a restore is permitted, executes the transactional database restore if valid, classifies the write result, and reports the outcome back to Rewind.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Initialization and Validation
- **Overview:** Initializes the execution via manual trigger or schedule, defines the base URLs and connection parameters, and sanitizes the endpoints to ensure secure and valid communication.
- **Nodes Involved:** 
  - `Run once`
  - `Every minute`
  - `Configure endpoints`
  - `Validate endpoints`

- **Node Details:**
  - **Run once**
    - *Type and Technical Role:* `n8n-nodes-base.manualTrigger` (v1) – Manual entry point for testing and initial setup.
    - *Configuration Choices:* Standard manual trigger configuration with no parameters.
    - *Input/Output:* No inputs; outputs to `Configure endpoints`.
    - *Edge Cases/Failures:* None.
  - **Every minute**
    - *Type and Technical Role:* `n8n-nodes-base.scheduleTrigger` (v1.2) – Automated entry point running the workflow every minute.
    - *Configuration Choices:* Interval rule set to 1 minute.
    - *Input/Output:* No inputs; outputs to `Configure endpoints`.
    - *Edge Cases/Failures:* Ensure the workflow remains inactive until disposable-data checks pass.
  - **Configure endpoints**
    - *Type and Technical Role:* `n8n-nodes-base.set` (v3.4) – Sets initial environment variables.
    - *Configuration Choices:* Assigns `rewindBaseUrl` (`https://rewind.kanishq.dev`), `supabaseBaseUrl` (`https://example-project.supabase.co`), and `connection` (`campaign_demo`).
    - *Input/Output:* Receives input from triggers; outputs to `Validate endpoints`.
    - *Edge Cases/Failures:* Incorrect base URL strings.
  - **Validate endpoints**
    - *Type and Technical Role:* `n8n-nodes-base.code` (v2) – Validates and sanitizes base URLs and connection strings.
    - *Configuration Choices:* Custom JavaScript enforcing HTTPS for remote endpoints, validating URL formatting, and filtering connection string syntax (`/^[a-z][a-z0-9_-]{0,62}$/`).
    - *Input/Output:* Receives configuration from `Configure endpoints`; outputs to `Claim approved job`.
    - *Edge Cases/Failures:* Throws an error if URLs contain whitespace, queries, fragments, or invalid protocols.

---

#### Block 1.2: Job Claiming and Analysis
- **Overview:** Requests an approved recovery job from Rewind using worker authentication and validates that the job operation targets a supported Supabase row update.
- **Nodes Involved:**
  - `Claim approved job`
  - `Validate job`

- **Node Details:**
  - **Claim approved job**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (v4.2) – POST request to claim an approved recovery job.
    - *Configuration Choices:* Uses Generic Header Auth (`Authorization = Bearer YOUR_WORKER_KEY`). Sends a JSON body specifying `kinds: ['supabase_row']` and the target connection name. Timeout set to 15,000ms.
    - *Input/Output:* Receives input from `Validate endpoints`; outputs to `Validate job`.
    - *Edge Cases/Failures:* Authentication errors or network timeouts.
  - **Validate job**
    - *Type and Technical Role:* `n8n-nodes-base.code` (v2) – Inspects the claimed job structure.
    - *Configuration Choices:* Custom JavaScript parsing the operation resource string (`alias:rowId`), checking `supabase_row` kind, and verifying version/field integrity.
    - *Input/Output:* Receives input from `Claim approved job`; outputs to `Read current resource`.
    - *Edge Cases/Failures:* Throws an error if the job structure is unsupported or malformed.

---

#### Block 1.3: State Checking
- **Overview:** Reads the current record from Supabase via a remote procedure call (RPC) and compares it against recorded values to detect potential conflicts or newer edits.
- **Nodes Involved:**
  - `Read current resource`
  - `Check current version`
  - `Version still matches`

- **Node Details:**
  - **Read current resource**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (v4.2) – RPC call to read the current resource state from Supabase.
    - *Configuration Choices:* Uses Generic Header Auth (`apikey = sb_secret` secret key). Calls endpoint `.../rest/v1/rpc/rewind_read_row` with parameters `p_alias` and `p_id`. Configured with `neverError: true` to handle non-200 responses gracefully.
    - *Input/Output:* Receives input from `Validate job`; outputs to `Check current version`.
    - *Edge Cases/Failures:* Auth errors, missing keys, or database connectivity issues. Bypasses RLS using secret keys.
  - **Check current version**
    - *Type and Technical Role:* `n8n-nodes-base.code` (v2) – Evaluates read response against expected pre-update conditions.
    - *Configuration Choices:* Custom JavaScript determining whether the row status is `failed`, `conflict`, `manual`, or valid for restoration (`canRestore`).
    - *Input/Output:* Receives input from `Read current resource`; outputs to `Version still matches`.
    - *Edge Cases/Failures:* Null fields or unreadable row versions.
  - **Version still matches**
    - *Type and Technical Role:* `n8n-nodes-base.if` (v1) – Conditional branching based on version match status.
    - *Configuration Choices:* Evaluates `{{ $json.canRestore }}` equals `true`.
    - *Input/Output:* Receives input from `Check current version`. True branch connects to `Conditional restore`; False branch connects directly to `Report outcome`.
    - *Edge Cases/Failures:* None.

---

#### Block 1.4: Conditional Restoration and Reporting
- **Overview:** Executes a transactional database restore when versions match, classifies the outcome of the database write, and reports the final status back to the Rewind API.
- **Nodes Involved:**
  - `Conditional restore`
  - `Classify write result`
  - `Report outcome`

- **Node Details:**
  - **Conditional restore**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (v4.2) – RPC call to perform the conditional row restoration.
    - *Configuration Choices:* Uses Supabase `apikey` Header Auth. Calls `.../rest/v1/rpc/rewind_restore_row` with expected version, alias, row ID, and before/after payloads. `neverError: true`, `retryOnFail: false`.
    - *Input/Output:* Receives input from `Version still matches` (True branch); outputs to `Classify write result`.
    - *Edge Cases/Failures:* Database conflicts (HTTP 409/412) or server errors. Never enable write retries here.
  - **Classify write result**
    - *Type and Technical Role:* `n8n-nodes-base.code` (v2) – Interprets HTTP status codes and response bodies from the restore operation.
    - *Configuration Choices:* Custom JavaScript returning status classifications (`conflict`, `uncertain`, `failed`, or `restored`).
    - *Input/Output:* Receives input from `Conditional restore`; outputs to `Report outcome`.
    - *Edge Cases/Failures:* Malformed response bodies or uncertain write outcomes.
  - **Report outcome**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (v4.2) – POST request to report the recovery job outcome to Rewind.
    - *Configuration Choices:* Uses Rewind Worker Header Auth (`Authorization = Bearer YOUR_WORKER_KEY`). URL dynamically includes the job ID. Sends lease token and result payload. Configured with `maxTries: 3` and a 2000ms delay between retries (`retryOnFail: true`).
    - *Input/Output:* Receives input from `Classify write result` (and `Version still matches` False branch); outputs terminal execution.
    - *Edge Cases/Failures:* Network failures during reporting (safely retried up to 3 times).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Run once** | `manualTrigger` | Manual entry point for testing | None | Configure endpoints | ### How it works<br>This worker is for n8n builders who have already captured a supported Supabase row update with Rewind. It claims one approved recovery job for the configured connection, reads the row and compares the current revision with the recorded result. If it still matches, the connector restores the original fields in a conditional transaction. A later edit stops recovery and preserves current data.<br><br>The workflow reports the outcome to Rewind. A report is not independent verification: inspect the actual Supabase row afterward. Capture and human approval happen separately; this worker cannot recover changes recorded after the fact or approve its own work.<br><br>### Setup<br>Use a dedicated development project with the connector and disposable demo table from https://rewind.kanishq.dev/#connect. Configure the Rewind and Supabase origins and the same connection name used by capture. Select Rewind worker Header Auth on Claim approved job and Report outcome. Select Supabase apikey Header Auth on Read current resource and Conditional restore. Credentials are not included.<br><br>Keep the workflow inactive. Capture one update, approve it in Rewind, then use Run once. Verify active/5 is restored. Repeat with a fresh captured update followed by a later edit and require that edit to remain. Publish the schedule only after both checks pass.<br><br>### Customization<br>Use only registered rows and supported scalar updates. No inserts, deletes or trigger-side-effect rollback. Inspect unknown outcomes manually; do not repeat an uncertain restore. |
| **Every minute** | `scheduleTrigger` | Automated interval entry point | None | Configure endpoints | ### How it works<br>This worker is for n8n builders who have already captured a supported Supabase row update with Rewind. It claims one approved recovery job for the configured connection, reads the row and compares the current revision with the recorded result. If it still matches, the connector restores the original fields in a conditional transaction. A later edit stops recovery and preserves current data.<br><br>The workflow reports the outcome to Rewind. A report is not independent verification: inspect the actual Supabase row afterward. Capture and human approval happen separately; this worker cannot recover changes recorded after the fact or approve its own work.<br><br>### Setup<br>Use a dedicated development project with the connector and disposable demo table from https://rewind.kanishq.dev/#connect. Configure the Rewind and Supabase origins and the same connection name used by capture. Select Rewind worker Header Auth on Claim approved job and Report outcome. Select Supabase apikey Header Auth on Read current resource and Conditional restore. Credentials are not included.<br><br>Keep the workflow inactive. Capture one update, approve it in Rewind, then use Run once. Verify active/5 is restored. Repeat with a fresh captured update followed by a later edit and require that edit to remain. Publish the schedule only after both checks pass.<br><br>### Customization<br>Use only registered rows and supported scalar updates. No inserts, deletes or trigger-side-effect rollback. Inspect unknown outcomes manually; do not repeat an uncertain restore. |
| **Configure endpoints** | `set` | Defines base URLs and connection name | Run once, Every minute | Validate endpoints | ### How it works<br>This worker is for n8n builders who have already captured a supported Supabase row update with Rewind. It claims one approved recovery job for the configured connection, reads the row and compares the current revision with the recorded result. If it still matches, the connector restores the original fields in a conditional transaction. A later edit stops recovery and preserves current data.<br><br>The workflow reports the outcome to Rewind. A report is not independent verification: inspect the actual Supabase row afterward. Capture and human approval happen separately; this worker cannot recover changes recorded after the fact or approve its own work.<br><br>### Setup<br>Use a dedicated development project with the connector and disposable demo table from https://rewind.kanishq.dev/#connect. Configure the Rewind and Supabase origins and the same connection name used by capture. Select Rewind worker Header Auth on Claim approved job and Report outcome. Select Supabase apikey Header Auth on Read current resource and Conditional restore. Credentials are not included.<br><br>Keep the workflow inactive. Capture one update, approve it in Rewind, then use Run once. Verify active/5 is restored. Repeat with a fresh captured update followed by a later edit and require that edit to remain. Publish the schedule only after both checks pass.<br><br>### Customization<br>Use only registered rows and supported scalar updates. No inserts, deletes or trigger-side-effect rollback. Inspect unknown outcomes manually; do not repeat an uncertain restore. |
| **Validate endpoints** | `code` | Sanitizes and validates base URLs and connection | Configure endpoints | Claim approved job | ### How it works<br>This worker is for n8n builders who have already captured a supported Supabase row update with Rewind. It claims one approved recovery job for the configured connection, reads the row and compares the current revision with the recorded result. If it still matches, the connector restores the original fields in a conditional transaction. A later edit stops recovery and preserves current data.<br><br>The workflow reports the outcome to Rewind. A report is not independent verification: inspect the actual Supabase row afterward. Capture and human approval happen separately; this worker cannot recover changes recorded after the fact or approve its own work.<br><br>### Setup<br>Use a dedicated development project with the connector and disposable demo table from https://rewind.kanishq.dev/#connect. Configure the Rewind and Supabase origins and the same connection name used by capture. Select Rewind worker Header Auth on Claim approved job and Report outcome. Select Supabase apikey Header Auth on Read current resource and Conditional restore. Credentials are not included.<br><br>Keep the workflow inactive. Capture one update, approve it in Rewind, then use Run once. Verify active/5 is restored. Repeat with a fresh captured update followed by a later edit and require that edit to remain. Publish the schedule only after both checks pass.<br><br>### Customization<br>Use only registered rows and supported scalar updates. No inserts, deletes or trigger-side-effect rollback. Inspect unknown outcomes manually; do not repeat an uncertain restore. |
| **Claim approved job** | `httpRequest` | Claims an approved recovery job from Rewind | Validate endpoints | Validate job | ### Start and route approved work<br>Keep inactive for the first drill. Run once claims an already approved job for this connection; it does not approve recovery. Select the worker credential on claim and report nodes.<br><br>Choose a Header Auth credential: Authorization = Bearer YOUR_WORKER_KEY. Use the same credential on Report outcome. |
| **Validate job** | `code` | Validates job payload, alias, and row ID | Claim approved job | Read current resource | ### Start and route approved work<br>Keep inactive for the first drill. Run once claims an already approved job for this connection; it does not approve recovery. Select the worker credential on claim and report nodes. |
| **Read current resource** | `httpRequest` | Reads current row state from Supabase RPC | Validate job | Check current version | ### Check current state<br>Read the provider with Supabase credentials. Compare row revision and recorded values. The false branch reports a conflict without restoring.<br><br>Header Auth: apikey = a server-side Supabase sb_secret key. Keep this credential only in n8n. Secret keys bypass RLS; use the dedicated test project first. |
| **Check current version** | `code` | Compares current revision with recorded state | Read current resource | Version still matches | ### Check current state<br>Read the provider with Supabase credentials. Compare row revision and recorded values. The false branch reports a conflict without restoring. |
| **Version still matches** | `if` | Branches execution based on version match | Check current version | Conditional restore (true), Report outcome (false) | ### Check current state<br>Read the provider with Supabase credentials. Compare row revision and recorded values. The false branch reports a conflict without restoring. |
| **Conditional restore** | `httpRequest` | Restores original fields via transactional RPC | Version still matches | Classify write result | ### Restore and report<br>Restore only when the condition still matches. Report the result, then independently inspect the Supabase row. An uncertain write must not be retried automatically.<br><br>Same Supabase apikey credential as Read current resource. The installed RPC locks and checks the row in one transaction. Never enable write retries. |
| **Classify write result** | `code` | Classifies outcome of the restore write | Conditional restore | Report outcome | ### Restore and report<br>Restore only when the condition still matches. Report the result, then independently inspect the Supabase row. An uncertain write must not be retried automatically. |
| **Report outcome** | `httpRequest` | Reports recovery outcome to Rewind API | Classify write result, Version still matches | None | ### Restore and report<br>Restore only when the condition still matches. Report the result, then independently inspect the Supabase row. An uncertain write must not be retried automatically.<br><br>Use your Rewind WORKER key. Retrying this exact result report is safe. Do not rerun the provider write to retry reporting. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Triggers & Configuration Nodes:**
   - **Node 1 (`Run once`):** Create a *Manual Trigger* node.
   - **Node 2 (`Every minute`):** Create a *Schedule Trigger* node with interval set to `1` minute.
   - **Node 3 (`Configure endpoints`):** Create a *Set* node. Add string assignments: `rewindBaseUrl` = `https://rewind.kanishq.dev`, `supabaseBaseUrl` = `https://example-project.supabase.co`, and `connection` = `campaign_demo`.
   - **Node 4 (`Validate endpoints`):** Create a *Code* node. Insert JavaScript validation logic to ensure secure HTTPS base URLs, correct formatting, and valid connection string syntax.
   - **Connections:** Link both `Run once` and `Every minute` to `Configure endpoints`, then link `Configure endpoints` to `Validate endpoints`.

2. **Set Up Job Claiming & Validation:**
   - **Node 5 (`Claim approved job`):** Create an *HTTP Request* node (v4.2). Set Method to `POST`, URL to `={{ $('Validate endpoints').first().json.rewindBaseUrl + '/api/v1/recovery-jobs/claim' }}`. Set Authentication to Generic Credential Type -> Header Auth (`Authorization = Bearer YOUR_WORKER_KEY`). Configure JSON body with `kinds: ['supabase_row']` and connection name. Set timeout to 15,000ms.
   - **Node 6 (`Validate job`):** Create a *Code* node to parse the job operation, extract `alias` and `rowId`, and validate the schema structure.
   - **Connections:** Link `Validate endpoints` -> `Claim approved job` -> `Validate job`.

3. **Set Up State Checking:**
   - **Node 7 (`Read current resource`):** Create an *HTTP Request* node (v4.2). Set Method to `POST`, URL to `={{ $('Validate endpoints').first().json.supabaseBaseUrl + '/rest/v1/rpc/rewind_read_row' }}`. Set Authentication to Generic Credential Type -> Header Auth (`apikey = sb_secret` secret key). Configure JSON body with `p_alias` and `p_id`. Enable `neverError: true` in response options. Set timeout to 15,000ms.
   - **Node 8 (`Check current version`):** Create a *Code* node to compare the current record revision with the recorded operation after version.
   - **Node 9 (`Version still matches`):** Create an *If* node with condition `{{ $json.canRestore }}` equal to `true`.
   - **Connections:** Link `Validate job` -> `Read current resource` -> `Check current version` -> `Version still matches`.

4. **Set Up Restoration & Reporting:**
   - **Node 10 (`Conditional restore`):** Create an *HTTP Request* node (v4.2). Set Method to `POST`, URL to `={{ $('Validate endpoints').first().json.supabaseBaseUrl + '/rest/v1/rpc/rewind_restore_row' }}`. Set Authentication to Supabase `apikey` Header Auth. Send JSON body with `p_alias`, `p_id`, `p_expected_version`, `p_before`, and `p_after`. Set `neverError: true`, `retryOnFail: false`.
   - **Node 11 (`Classify write result`):** Create a *Code* node to evaluate the database write response status and categorize it into `restored`, `conflict`, `uncertain`, or `failed`.
   - **Node 12 (`Report outcome`):** Create an *HTTP Request* node (v4.2). Set Method to `POST`, URL to `={{ $('Validate endpoints').first().json.rewindBaseUrl + '/api/v1/recovery-jobs/' + $('Validate job').first().json.job.id + '/complete' }}`. Set Authentication to Rewind Worker Header Auth (`Authorization = Bearer YOUR_WORKER_KEY`). Send JSON body merging the lease token and result payload. Enable `retryOnFail: true`, set `maxTries: 3` with 2000ms wait.
   - **Connections:** 
     - Link `Version still matches` (True branch, index 0) -> `Conditional restore`.
     - Link `Conditional restore` -> `Classify write result`.
     - Link `Classify write result` -> `Report outcome`.
     - Link `Version still matches` (False branch, index 1) directly to `Report outcome`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Install the connector and disposable demo table in a development project. | [Rewind Connect Documentation](https://rewind.kanishq.dev/#connect) |
| Keep workflow inactive during initial setup; test an approved restore and a newer-edit conflict before enabling the schedule. | Setup instructions & safety guidelines |