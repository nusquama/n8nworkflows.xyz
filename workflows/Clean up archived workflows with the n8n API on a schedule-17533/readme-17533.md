Clean up archived workflows with the n8n API on a schedule

https://n8nworkflows.xyz/workflows/clean-up-archived-workflows-with-the-n8n-api-on-a-schedule-17533


# Clean up archived workflows with the n8n API on a schedule

### 1. Workflow Overview

This workflow automates the identification and optional cleanup of archived n8n workflows based on a defined retention period, safety tags, and batch limits. It is designed for system administrators and DevOps teams managing large n8n instances who need to maintain hygiene by pruning old, unneeded archived workflows without accidental data loss.

The logic operates in four sequential blocks:
- **1.1 Initialization and Discovery:** Triggers on a daily schedule, establishes global configuration variables, fetches all workflows from the n8n instance via the n8n API, and filters for those currently marked as archived.
- **1.2 Validation and Classification:** Validates parameters (such as retention duration and project bindings), classifies each archived workflow into deletion candidates or excluded groups based on age and protected tags, and handles structural errors.
- **1.3 Execution Routing:** Evaluates whether candidates exist and checks the safety toggle (`dry_run`). If enabled or if no candidates exist, it routes directly to reporting; otherwise, it splits the list for batched processing.
- **1.4 Deletion and Summarization:** Executes batched deletions via the n8n API (when live mode is active) and compiles a comprehensive JSON summary detailing counts, successes, failures, and execution status.

---

### 2. Block-by-Block Analysis

#### 2.1 Initialization and Discovery
- **Overview:** Sets up the execution schedule, initializes configuration variables, retrieves instance workflows using the n8n API, and isolates archived workflows for downstream evaluation.
- **Nodes Involved:** `daily_cleanup_schedule`, `config`, `get_workflows`, `filter_archived_workflows`.
- **Node Details:**
  - `daily_cleanup_schedule`
    - *Type and technical role:* Schedule Trigger (v1.3). Initiates execution daily at 09:00 in the instance timezone.
    - *Configuration choices:* Interval set to run at hour 9.
    - *Input/Output:* No inputs; outputs to `config`.
    - *Failure types:* None typical.
  - `config`
    - *Type and technical role:* Set Node (v3.4). Establishes configuration variables for the workflow run.
    - *Configuration choices:* Defines variables: `retention_days` (number: 7), `dry_run` (boolean: true), `project_id` (string: ""), `protected_tag` (string: "keep"), `batch_size` (number: 10), `batch_interval_ms` (number: 1000).
    - *Input/Output:* Input from `daily_cleanup_schedule`; output to `get_workflows`.
    - *Failure types:* Expression evaluation failures if data types mismatch.
  - `get_workflows`
    - *Type and technical role:* n8n Native Node (v1). Interacts with the n8n API to retrieve all workflows.
    - *Configuration choices:* Filters by `projectId` using `={{ $('config').first().json.project_id || undefined }}` and excludes pinned data (`excludePinnedData: true`). Request timeout set to 30,000ms.
    - *Credentials:* `n8n_main` (n8n API credential).
    - *Input/Output:* Input from `config`; output to `filter_archived_workflows`.
    - *Failure types:* API authentication failure, timeout, or insufficient permissions.
  - `filter_archived_workflows`
    - *Type and technical role:* Filter Node (v2.3). Restricts the dataset to workflows marked as archived.
    - *Configuration choices:* Condition checks if `={{ $json.isArchived }}` equals `true`.
    - *Input/Output:* Input from `get_workflows`; output to `select_candidates`.
    - *Failure types:* Missing `isArchived` property in payload.

#### 2.2 Validation and Classification
- **Overview:** Validates configuration parameters and workflow schemas, partitioning archived workflows into deletion candidates, age-retained workflows, and tag-protected workflows.
- **Nodes Involved:** `select_candidates`, `has_candidates`, `validation_error`.
- **Node Details:**
  - `select_candidates`
    - *Type and technical role:* Code Node (v2). Executes custom JavaScript to validate constraints and categorize items.
    - *Configuration choices:* Checks that `retention_days` is a finite number greater than 0, ensures `project_id` is specified if `dry_run` is false, computes item ages against a cutoff timestamp, and checks normalized tags against `protected_tag`.
    - *Error handling:* Configured with `onError: continueErrorOutput` to route validation exceptions to the error branch.
    - *Input/Output:* Input from `filter_archived_workflows`; primary output to `has_candidates`, error output to `validation_error`.
    - *Failure types:* Unhandled exceptions thrown in code (e.g., missing workflow ID or `updatedAt` properties, missing `project_id` during live mode).
  - `has_candidates`
    - *Type and technical role:* IF Node (v2.3). Evaluates whether the number of workflows targeted for deletion is greater than zero.
    - *Configuration choices:* Condition: `={{ $json.stats.to_delete }}` > 0.
    - *Input/Output:* Input from `select_candidates`; true output to `dry_run`, false output to `cleanup_summary`.
    - *Failure types:* Type coercion issues if stats payload is malformed.
  - `validation_error`
    - *Type and technical role:* Stop and Error Node (v1). Halts execution and raises a descriptive error when validation fails.
    - *Configuration choices:* Formats error message dynamically: `={{ 'Hubo un error al procesar los flujos. Detalle: ' + ($json.error?.message ?? $json.error ?? $json.message ?? 'sin detalle') }}`.
    - *Input/Output:* Input from error output of `select_candidates`; terminal node.
    - *Failure types:* Triggered intentionally upon configuration or data schema violations.

#### 2.3 Execution Routing
- **Overview:** Inspects the safety flag (`dry_run`) to determine whether candidates should bypass deletion and go straight to summary generation or proceed to batched deletion.
- **Nodes Involved:** `dry_run`.
- **Node Details:**
  - `dry_run`
    - *Type and technical role:* IF Node (v2.3). Checks if dry-run mode is active.
    - *Configuration choices:* Condition: `={{ $('config').first().json.dry_run }}` equals `true`.
    - *Input/Output:* Input from `has_candidates` (true branch); true output to `cleanup_summary`, false output to `split_candidates`.
    - *Failure types:* Reference evaluation failure if config node data is unavailable.

#### 2.4 Deletion and Summarization
- **Overview:** Splits candidate lists into individual flow items, executes batched API deletions in live mode, and aggregates execution metrics into a unified summary report.
- **Nodes Involved:** `split_candidates`, `delete_workflows_in_batches`, `cleanup_summary`.
- **Node Details:**
  - `split_candidates`
    - *Type and technical role:* Split Out Node (v1). Flattens the array of candidates into individual items.
    - *Configuration choices:* Splits field `candidates` with destination field name `target`, including all other fields.
    - *Input/Output:* Input from `dry_run` (false branch); output to `delete_workflows_in_batches`.
    - *Failure types:* Attempting to split a non-array or missing field.
  - `delete_workflows_in_batches`
    - *Type and technical role:* n8n Native Node (v1). Deletes workflows via the n8n API using native batching configurations.
    - *Configuration choices:* Operation set to delete. Target ID set to `={{ $json.target.id }}`. Timeout set to 30,000ms. Request options include batching configuration using `={{ $('config').first().json.batch_size }}` and `={{ $('config').first().json.batch_interval_ms }}`.
    - *Error handling:* Configured with `onError: continueRegularOutput` and `alwaysOutputData: true` to ensure failures do not halt the summary aggregation.
    - *Credentials:* `n8n_main` (n8n API credential).
    - *Input/Output:* Input from `split_candidates`; output to `cleanup_summary`.
    - *Failure types:* API rate limits, network timeouts, or authorization errors during deletion.
  - `cleanup_summary`
    - *Type and technical role:* Code Node (v2). Aggregates results from either the dry-run path or the live deletion path to build a final execution report.
    - *Configuration choices:* Calculates success counts, failure counts, unprocessed items, and assigns a final status (`NOTHING_TO_DO`, `PREVIEW`, `PARTIAL`, or `SUCCESS`).
    - *Input/Output:* Inputs from `has_candidates` (false branch), `dry_run` (true branch), and `delete_workflows_in_batches`. Outputs terminal summary item.
    - *Failure types:* Reference errors if upstream nodes were skipped or returned empty datasets.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `workflow_overview_note` | stickyNote | Documents overall workflow architecture, flow, safety controls, and trade-offs. | None | None | # Workflow overview<br><br>Safely previews and optionally deletes archived n8n workflows older than a configurable retention period.<br><br>**Flow:** schedule → retrieve workflows → keep archived → classify → preview/delete → summarize.<br><br>**Safety:** `dry_run` is enabled by default, `project_id` is required for live mode, `protected_tag` prevents deletion, and deletions are batched.<br><br>**Tradeoffs**<br>- The n8n API returns all workflows in scope before the Filter node, so large instances may use more memory and execution time. |
| `title_note` | stickyNote | Displays workflow title on canvas. | None | None | # Archived Workflow Cleanup |
| `daily_cleanup_schedule` | scheduleTrigger | Triggers execution daily at 09:00. | None | `config` | # Step 1 — Schedule and discover<br><br>- `daily_cleanup_schedule` runs daily at 09:00 in the workflow timezone.<br>- `get_workflows` retrieves workflows from `project_id` when provided.<br>- Pinned data is excluded.<br>- `filter_archived_workflows` sends only archived workflows to classification.<br><br>**Important:** non-archived workflows are discarded before custom code runs. |
| `config` | set | Sets global configuration variables for the workflow run. | `daily_cleanup_schedule` | `get_workflows` | # Configuration<br><br>Change these values in `config`:<br><br>- `retention_days`: minimum archived age. Default: `7`.<br>- `dry_run`: `true` previews; `false` enables deletion.<br>- `project_id`: optional scope in preview; required for live deletion.<br>- `protected_tag`: tag that prevents deletion. Default: `keep`.<br>- `batch_size`: deletions per batch. Default: `10`.<br>- `batch_interval_ms`: pause between batches. Default: `1000` ms.<br><br>Select an **n8n API credential** in both n8n nodes. Review a preview before enabling live mode. |
| `get_workflows` | n8n | Fetches instance workflows from the n8n API. | `config` | `filter_archived_workflows` | # Step 1 — Schedule and discover<br><br>- `daily_cleanup_schedule` runs daily at 09:00 in the workflow timezone.<br>- `get_workflows` retrieves workflows from `project_id` when provided.<br>- Pinned data is excluded.<br>- `filter_archived_workflows` sends only archived workflows to classification.<br><br>**Important:** non-archived workflows are discarded before custom code runs. |
| `filter_archived_workflows` | filter | Keeps only workflows where `isArchived` is true. | `get_workflows` | `select_candidates` | # Step 1 — Schedule and discover<br><br>- `daily_cleanup_schedule` runs daily at 09:00 in the workflow timezone.<br>- `get_workflows` retrieves workflows from `project_id` when provided.<br>- Pinned data is excluded.<br>- `filter_archived_workflows` sends only archived workflows to classification.<br><br>**Important:** non-archived workflows are discarded before custom code runs. |
| `select_candidates` | code | Validates configuration and categorizes archived workflows by age and protection tags. | `filter_archived_workflows` | `has_candidates`, `validation_error` | # Step 2 — Classify and decide<br><br>- `select_candidates` validates configuration and workflow fields.<br>- Every archived workflow enters one category: delete, retained by age, or protected by tag.<br>- `has_candidates` skips deletion when the candidate count is zero.<br>- `dry_run` routes candidates to preview or live deletion.<br>- Validation failures stop in `validation_error`.<br><br>**Invariant:** the three categories always add up to `total_archived`. |
| `has_candidates` | if | Branches execution based on whether deletion candidates exist. | `select_candidates` | `dry_run`, `cleanup_summary` | # Step 2 — Classify and decide<br><br>- `select_candidates` validates configuration and workflow fields.<br>- Every archived workflow enters one category: delete, retained by age, or protected by tag.<br>- `has_candidates` skips deletion when the candidate count is zero.<br>- `dry_run` routes candidates to preview or live deletion.<br>- Validation failures stop in `validation_error`.<br><br>**Invariant:** the three categories always add up to `total_archived`. |
| `validation_error` | stopAndError | Stops execution and raises an error if validation fails. | `select_candidates` | None | # Step 2 — Classify and decide<br><br>- `select_candidates` validates configuration and workflow fields.<br>- Every archived workflow enters one category: delete, retained by age, or protected by tag.<br>- `has_candidates` skips deletion when the candidate count is zero.<br>- `dry_run` routes candidates to preview or live deletion.<br>- Validation failures stop in `validation_error`.<br><br>**Invariant:** the three categories always add up to `total_archived`. |
| `dry_run` | if | Routes execution to preview summary or live deletion based on safety mode. | `has_candidates` | `cleanup_summary`, `split_candidates` | # Step 2 — Classify and decide<br><br>- `select_candidates` validates configuration and workflow fields.<br>- Every archived workflow enters one category: delete, retained by age, or protected by tag.<br>- `has_candidates` skips deletion when the candidate count is zero.<br>- `dry_run` routes candidates to preview or live deletion.<br>- Validation failures stop in `validation_error`.<br><br>**Invariant:** the three categories always add up to `total_archived`. |
| `split_candidates` | splitOut | Flattens the candidate array into individual items for batch processing. | `dry_run` | `delete_workflows_in_batches` | # Step 3 — Delete and summarize<br><br>- `split_candidates` creates one item per workflow candidate.<br>- `delete_workflows_in_batches` deletes using `batch_size` and `batch_interval_ms`.<br>- `cleanup_summary` reports status, archived totals, candidates, successes, and failures.<br><br>**Important:** deletion is permanent. Keep `dry_run = true` until the preview is approved. |
| `delete_workflows_in_batches` | n8n | Deletes workflows via the n8n API in configured batches. | `split_candidates` | `cleanup_summary` | # Step 3 — Delete and summarize<br><br>- `split_candidates` creates one item per workflow candidate.<br>- `delete_workflows_in_batches` deletes using `batch_size` and `batch_interval_ms`.<br>- `cleanup_summary` reports status, archived totals, candidates, successes, and failures.<br><br>**Important:** deletion is permanent. Keep `dry_run = true` until the preview is approved. |
| `cleanup_summary` | code | Aggregates execution statistics and generates a final report. | `has_candidates`, `dry_run`, `delete_workflows_in_batches` | None | # Step 3 — Delete and summarize<br><br>- `split_candidates` creates one item per workflow candidate.<br>- `delete_workflows_in_batches` deletes using `batch_size` and `batch_interval_ms`.<br>- `cleanup_summary` reports status, archived totals, candidates, successes, and failures.<br><br>**Important:** deletion is permanent. Keep `dry_run = true` until the preview is approved. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**
   - Add a **Schedule Trigger** node named `daily_cleanup_schedule`. Set the interval rule to trigger at hour `9`.
2. **Create the Configuration Node:**
   - Add a **Set** node named `config`.
   - Add assignments:
     - `retention_days` (Number) = `7`
     - `dry_run` (Boolean) = `true`
     - `project_id` (String) = `""`
     - `protected_tag` (String) = `"keep"`
     - `batch_size` (Number) = `10`
     - `batch_interval_ms` (Number) = `1000`
   - Connect `daily_cleanup_schedule` output to `config`.
3. **Create Workflow Retrieval Node:**
   - Add an **n8n** node named `get_workflows`.
   - Configure credentials: select or create an **n8n API** credential (`n8n_main`).
   - Set request filter options: `projectId` = `={{ $('config').first().json.project_id || undefined }}` and `excludePinnedData` = `true`. Set request timeout to `30000`.
   - Connect `config` output to `get_workflows`.
4. **Create Archived Filter Node:**
   - Add a **Filter** node named `filter_archived_workflows`.
   - Add condition: string/boolean evaluation where `={{ $json.isArchived }}` equals `true`.
   - Connect `get_workflows` output to `filter_archived_workflows`.
5. **Create Candidate Classification Node:**
   - Add a **Code** node named `select_candidates`.
   - Insert the JavaScript logic provided in the workflow definition to validate parameters and partition workflows into deletion candidates, age-retained, and tag-protected items.
   - Set node error handling option (`onError`) to `continueErrorOutput`.
   - Connect `filter_archived_workflows` output to `select_candidates`.
6. **Create Validation Error Handler:**
   - Add a **Stop and Error** node named `validation_error`.
   - Set error message expression: `={{ 'Hubo un error al procesar los flujos. Detalle: ' + ($json.error?.message ?? $json.error ?? $json.message ?? 'sin detalle') }}`.
   - Connect the error output (index 1) of `select_candidates` to `validation_error`.
7. **Create Candidate Presence Check Node:**
   - Add an **IF** node named `has_candidates`.
   - Set condition: `={{ $json.stats.to_delete }}` greater than `0`.
   - Connect the main output (index 0) of `select_candidates` to `has_candidates`.
8. **Create Dry-Run Safety Switch Node:**
   - Add an **IF** node named `dry_run`.
   - Set condition: `={{ $('config').first().json.dry_run }}` equals `true`.
   - Connect the true branch output of `has_candidates` to `dry_run`.
   - Connect the false branch output of `has_candidates` directly to `cleanup_summary`.
9. **Create Candidate Splitter Node:**
   - Add a **Split Out** node named `split_candidates`.
   - Set field to split out to `candidates` and destination field name to `target` (including all other fields).
   - Connect the true branch output of `dry_run` to `split_candidates`.
   - Connect the false branch output of `dry_run` to `cleanup_summary`.
10. **Create Batch Deletion Node:**
    - Add an **n8n** node named `delete_workflows_in_batches`.
    - Configure credentials: select the same **n8n API** credential (`n8n_main`).
    - Set operation to `delete`, workflow ID to `={{ $json.target.id }}`, and request timeout to `30000`.
    - Configure request batching options: batch size = `={{ $('config').first().json.batch_size }}`, batch interval = `={{ $('config').first().json.batch_interval_ms }}`.
    - Set node error handling option (`onError`) to `continueRegularOutput` and enable `alwaysOutputData`.
    - Connect `split_candidates` output to `delete_workflows_in_batches`.
11. **Create Summary Aggregation Node:**
    - Add a **Code** node named `cleanup_summary`.
    - Insert the JavaScript logic provided in the workflow definition to compile metrics and determine final status.
    - Connect outputs from `has_candidates` (false branch), `dry_run` (true branch), and `delete_workflows_in_batches` to `cleanup_summary`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Deletion is permanent. Always keep `dry_run` enabled (`true`) until preview outputs have been fully reviewed. | Safety configuration guideline |
| The n8n API retrieves all workflows within scope before local filtering; large instances should monitor memory and execution time. | Instance performance consideration |