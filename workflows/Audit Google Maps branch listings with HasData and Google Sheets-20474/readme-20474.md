Audit Google Maps branch listings with HasData and Google Sheets

https://n8nworkflows.xyz/workflows/audit-google-maps-branch-listings-with-hasdata-and-google-sheets-20474


# Audit Google Maps branch listings with HasData and Google Sheets

### 1. Workflow Overview

This workflow is designed for local SEO and operations teams to audit physical branch Google Maps listings against an organization's approved, internal data. It fetches live data from Google Maps using exact Place IDs via the HasData API, normalizes and compares key attributes (name, address, phone number, and website hostname), and writes the evaluation state back to a Google Sheets document. 

The workflow tracks differences, handles approved aliases, identifies missing fields, and evaluates historical state transitions (detecting when a previously flagged discrepancy has been resolved on subsequent runs). It never performs automated writes or updates to Google Business Profile listings; all outputs function as a structured review queue for human validation.

The execution logic is structured into three consecutive functional blocks:

- **1.1 Initialization and Input Validation:** Initializes run configuration parameters (spreadsheet ID, batch limits, country calling code), validates settings against strict schema rules, reads the approved branch inventory from Google Sheets, and validates input data integrity (checking for duplicate branch IDs or reused Place IDs).
- **1.2 Data Retrieval and Listing Comparison:** Loads historical results from the spreadsheet, extracts saved comparison baselines, performs parallelized Google Maps Place data fetching via HasData using exact Place IDs, and compares observed attributes against expected values and custom aliases.
- **1.3 Persistence and Review Queue Generation:** Formats the final comparison records, computes field state transitions (such as resolved differences), and writes or updates the structured output records in the Google Sheets results tab.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Initialization and Input Validation
- **Overview:** Establishes execution parameters, ensures that configuration and input spreadsheets are correctly formatted, prevents oversized API batches, and guards against duplicate records before paid external requests occur.
- **Nodes Involved:** 
  - `Run manually`
  - `Settings`
  - `Validate settings`
  - `Read input`
  - `Validate input batch`
- **Node Details:**
  - **Run manually**
    - *Type and Technical Role:* `n8n-nodes-base.manualTrigger` — Manual execution entry point.
    - *Configuration:* Default.
    - *Inputs:* None.
    - *Outputs:* `Settings`.
    - *Edge Cases:* None.
  - **Settings**
    - *Type and Technical Role:* `n8n-nodes-base.set` — Injects configuration constants into the stream.
    - *Configuration:* Assigns `spreadsheet_id` (string), `max_items` (number, default `5`), and `country_calling_code` (string, default `"1"`).
    - *Inputs:* `Run manually`.
    - *Outputs:* `Validate settings`.
    - *Edge Cases:* Unconfigured spreadsheet placeholder (`REPLACE_WITH_SPREADSHEET_ID`) will trigger downstream validation errors.
  - **Validate settings**
    - *Type and Technical Role:* `n8n-nodes-base.code` — JavaScript execution block evaluating configuration parameters.
    - *Configuration:* Runs once for all items; validates spreadsheet ID format, bounds `max_items` between 1 and 10, sets an ISO run timestamp (`run_id`), and validates the country calling code.
    - *Inputs:* `Settings`.
    - *Outputs:* `Read input`.
    - *Edge Cases:* Throws runtime errors if `spreadsheet_id` matches placeholder patterns or lacks required length.
  - **Read input**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` — Reads approved branch records from Google Sheets.
    - *Configuration:* Resource: `sheet`, Operation: `read`, Sheet Name: `Branches`, Document ID resolved via expression `{{ $('Validate settings').first().json.spreadsheet_id }}`. Always outputs data.
    - *Inputs:* `Validate settings`.
    - *Outputs:* `Validate input batch`.
    - *Edge Cases:* API authentication failures, missing sheet named "Branches", or invalid spreadsheet ID.
  - **Validate input batch**
    - *Type and Technical Role:* `n8n-nodes-base.code` — JavaScript execution block validating sheet rows.
    - *Configuration:* Runs once for all items; ensures mandatory headers (`branch_id`, `place_id`, `expected_name`, `expected_address`) are present, enforces `max_items` limits, and rejects duplicate branch IDs or reused Place IDs.
    - *Inputs:* `Read input`.
    - *Outputs:* `Prepare queue read`.
    - *Edge Cases:* Throws errors on incomplete input rows, duplicate identifiers, or empty sheets.

#### Block 1.2: Data Retrieval and Listing Comparison
- **Overview:** Prepares the environment to read historical records, fetches live Google Maps Place payloads using the verified HasData integration, and computes attribute-level comparisons between expected and observed values.
- **Nodes Involved:**
  - `Prepare queue read`
  - `Read Results`
  - `Check saved keys`
  - `Fetch search evidence`
  - `Attach search context`
- **Node Details:**
  - **Prepare queue read**
    - *Type and Technical Role:* `n8n-nodes-base.code` — Intermediary pass-through code node.
    - *Configuration:* Returns a static ready flag (`{ready: true}`).
    - *Inputs:* `Validate input batch`.
    - *Outputs:* `Read Results`.
    - *Edge Cases:* None.
  - **Read Results**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` — Reads historical comparison records from the Results tab.
    - *Configuration:* Resource: `sheet`, Operation: `read`, Sheet Name: `Results`, Document ID resolved via expression `{{ $('Validate settings').first().json.spreadsheet_id }}`.
    - *Inputs:* `Prepare queue read`.
    - *Outputs:* `Check saved keys`.
    - *Edge Cases:* Missing "Results" sheet or invalid credentials.
  - **Check saved keys**
    - *Type and Technical Role:* `n8n-nodes-base.code` — Validates historical result uniqueness.
    - *Configuration:* Iterates over existing result rows to check for duplicate `result_key` values.
    - *Inputs:* `Read Results`.
    - *Outputs:* `Fetch search evidence`.
    - *Edge Cases:* Throws an error if duplicate result keys exist in the destination sheet.
  - **Fetch search evidence**
    - *Type and Technical Role:* `@hasdata/n8n-nodes-hasdata.hasData` — External API call to retrieve Google Maps place data.
    - *Configuration:* Resource: `google_maps`, Operation: `place`, Place ID parameter resolved via expression `{{ $json.place_id }}`. Error handling set to continue regular output.
    - *Inputs:* `Check saved keys`.
    - *Outputs:* `Attach search context`.
    - *Edge Cases:* API credit exhaustion, rate limits, network timeouts, or invalid/non-existent Place IDs returning error payloads.
  - **Attach search context**
    - *Type and Technical Role:* `n8n-nodes-base.code` — JavaScript execution block performing structured comparison.
    - *Configuration:* Pairs API responses with input branch plans and historical baselines. Normalizes phone numbers, extracts website hostnames, tests custom aliases, evaluates field states (`matches`, `review_difference`, `unknown`), and calculates historical field transitions (`resolved`).
    - *Inputs:* `Fetch search evidence`.
    - *Outputs:* `Build review rows`.
    - *Edge Cases:* Malformed JSON in historical `evidence_json` cells or ambiguous item pairing between input plans and API responses.

#### Block 1.3: Persistence and Review Queue Generation
- **Overview:** Prepares the final payload structures and updates the Google Sheets results ledger with the evaluated audit states, differences, unknown fields, and resolved transitions.
- **Nodes Involved:**
  - `Build review rows`
  - `Save review queue`
- **Node Details:**
  - **Build review rows**
    - *Type and Technical Role:* `n8n-nodes-base.code` — Pass-through code node formatting final collection arrays.
    - *Configuration:* Returns all input items.
    - *Inputs:* `Attach search context`.
    - *Outputs:* `Save review queue`.
    - *Edge Cases:* None.
  - **Save review queue**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` — Writes audit results to Google Sheets.
    - *Configuration:* Resource: `sheet`, Operation: `appendOrUpdate`, Sheet Name: `Results`, Document ID resolved via expression `{{ $('Validate settings').first().json.spreadsheet_id }}`. Matching column configured to `result_key`. Cell format set to `RAW`. Maps attributes (`topic`, `status`, `summary`, `place_id`, `checked_at`, `result_key`, `differences`, `evidence_json`, `unknown_fields`, `resolved_fields`).
    - *Inputs:* `Build review rows`.
    - *Outputs:* None (Terminal node).
    - *Edge Cases:* Google Sheets cell size limits (exceeding 44,000 characters in `evidence_json`), API write rate limits, or permission errors.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Run manually | n8n-nodes-base.manualTrigger | Manual execution entry point | None | Settings | |
| Settings | n8n-nodes-base.set | Injects run configuration parameters | Run manually | Validate settings | |
| Validate settings | n8n-nodes-base.code | Validates configuration constraints and formats | Settings | Read input | |
| Read input | n8n-nodes-base.googleSheets | Reads approved branch details from Google Sheets | Validate settings | Validate input batch | 1. Add your approved branch details<br>Create Branches headers: `branch_id`, `place_id`, `expected_name`, `expected_address`, `expected_phone`, `expected_website`, `name_aliases`, `address_aliases`, `phone_aliases`, `website_aliases`. Name and address are required. Other comparison fields and aliases are optional. Set spreadsheet_id and country_calling_code in Settings. |
| Validate input batch | n8n-nodes-base.code | Validates branch batch size, headers, and unique IDs | Read input | Prepare queue read | 2. Check the batch and saved results<br>Reject duplicate branch IDs, reused place IDs and oversized batches before paid requests. Read Results once and reject duplicate saved keys. Every branch is checked again. Saved comparisons provide the baseline for the next run, not a cache. |
| Prepare queue read | n8n-nodes-base.code | Intermediary pass-through node | Validate input batch | Read Results | 2. Check the batch and saved results<br>Reject duplicate branch IDs, reused place IDs and oversized batches before paid requests. Read Results once and reject duplicate saved keys. Every branch is checked again. Saved comparisons provide the baseline for the next run, not a cache. |
| Read Results | n8n-nodes-base.googleSheets | Reads historical results from Google Sheets | Prepare queue read | Check saved keys | 2. Check the batch and saved results<br>Reject duplicate branch IDs, reused place IDs and oversized batches before paid requests. Read Results once and reject duplicate saved keys. Every branch is checked again. Saved comparisons provide the baseline for the next run, not a cache. |
| Check saved keys | n8n-nodes-base.code | Validates uniqueness of historical result keys | Read Results | Fetch search evidence | 2. Check the batch and saved results<br>Reject duplicate branch IDs, reused place IDs and oversized batches before paid requests. Read Results once and reject duplicate saved keys. Every branch is checked again. Saved comparisons provide the baseline for the next run, not a cache. |
| Fetch search evidence | @hasdata/n8n-nodes-hasdata.hasData | Fetches Google Maps Place data via HasData API | Check saved keys | Attach search context | |
| Attach search context | n8n-nodes-base.code | Compares observed place data against expected values and aliases | Fetch search evidence | Build review rows | |
| Build review rows | n8n-nodes-base.code | Formats evaluated records for persistence | Attach search context | Save review queue | |
| Save review queue | n8n-nodes-base.googleSheets | Appends or updates audit results in Google Sheets | Build review rows | None | 3. Compare the listing and save review tasks<br>Create Results headers: `result_key`, `checked_at`, `topic`, `status`, `summary`, `differences`, `unknown_fields`, `resolved_fields`, `place_id`, `evidence_json`. Review differences_to_review. Missing data stays unknown. resolved_fields marks previously different values that now match unchanged expectations. Optional review_status and editor_notes remain untouched. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Entry Point:**
   - Add a **Manual Trigger** (`n8n-nodes-base.manualTrigger`) node named `Run manually`.
2. **Configure Settings:**
   - Add a **Set** (`n8n-nodes-base.set`) node named `Settings`.
   - Add string assignment `spreadsheet_id` set to your Google Sheet ID.
   - Add number assignment `max_items` set to `5` (or desired batch ceiling up to 10).
   - Add string assignment `country_calling_code` set to your target country code (e.g., `"1"`).
   - Connect `Run manually` output to `Settings`.
3. **Add Configuration Validation:**
   - Add a **Code** (`n8n-nodes-base.code`) node named `Validate settings`. Set mode to `Run Once for All Items`.
   - Paste configuration checking logic verifying spreadsheet ID format and parameter bounds.
   - Connect `Settings` output to `Validate settings`.
4. **Read Input Data:**
   - Add a **Google Sheets** (`n8n-nodes-base.googleSheets`) node named `Read input`.
   - Configure resource to `sheet`, operation to `read`, sheet name to `Branches`.
   - Set Document ID expression to: `{{ $('Validate settings').first().json.spreadsheet_id }}`. Enable "Always Output Data".
   - Connect `Validate settings` output to `Read input`.
5. **Validate Input Batch:**
   - Add a **Code** (`n8n-nodes-base.code`) node named `Validate input batch` (Run Once for All Items) to verify headers (`branch_id`, `place_id`, `expected_name`, `expected_address`), check for duplicate branch/place IDs, and enforce batch size limits.
   - Connect `Read input` output to `Validate input batch`.
6. **Prepare Queue Read:**
   - Add a **Code** (`n8n-nodes-base.code`) node named `Prepare queue read` returning `{ready: true}`.
   - Connect `Validate input batch` output to `Prepare queue read`.
7. **Read Historical Results:**
   - Add a **Google Sheets** (`n8n-nodes-base.googleSheets`) node named `Read Results`.
   - Configure resource to `sheet`, operation to `read`, sheet name to `Results`.
   - Set Document ID expression to: `{{ $('Validate settings').first().json.spreadsheet_id }}`.
   - Connect `Prepare queue read` output to `Read Results`.
8. **Validate Saved Keys:**
   - Add a **Code** (`n8n-nodes-base.code`) node named `Check saved keys` to ensure uniqueness of existing `result_key` entries.
   - Connect `Read Results` output to `Check saved keys`.
9. **Fetch External Evidence:**
   - Add the verified HasData community node (`@hasdata/n8n-nodes-hasdata.hasData`) named `Fetch search evidence`.
   - Configure resource to `google_maps`, operation to `place`. Set Place ID parameter expression to: `{{ $json.place_id }}`.
   - Configure error handling (`onError`) to `Continue Regular Output`.
   - Connect `Check saved keys` output to `Fetch search evidence`.
10. **Process and Compare Listing Data:**
    - Add a **Code** (`n8n-nodes-base.code`) node named `Attach search context` (Run Once for All Items) to pair API responses with input plans, execute normalization routines for phones and URLs, evaluate aliases, calculate status transitions, and build evidence payloads.
    - Connect `Fetch search evidence` output to `Attach search context`.
11. **Format Review Rows:**
    - Add a **Code** (`n8n-nodes-base.code`) node named `Build review rows` to pass the processed items forward.
    - Connect `Attach search context` output to `Build review rows`.
12. **Persist Output to Google Sheets:**
    - Add a **Google Sheets** (`n8n-nodes-base.googleSheets`) node named `Save review queue`.
    - Configure resource to `sheet`, operation to `appendOrUpdate`.
    - Set Document ID expression to: `{{ $('Validate settings').first().json.spreadsheet_id }}`.
    - Set sheet name to `Results`, mapping mode to `Define Below`, and matching columns to `result_key`.
    - Map output properties (`topic`, `status`, `summary`, `place_id`, `checked_at`, `result_key`, `differences`, `evidence_json`, `unknown_fields`, `resolved_fields`) using corresponding item expressions (`{{ $json.propertyName }}`).
    - Connect `Build review rows` output to `Save review queue`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Designed for local SEO and operations teams reviewing branch information. A mismatch is a review task, not proof that Google is wrong. | Operational Guidelines |
| Website comparisons use hostnames, not paths. Missing data cannot resolve a difference. Run one execution at a time. | Technical Constraints |
| Requires the verified HasData community node installed in your n8n workspace, along with valid HasData and Google Sheets credentials. | Integration Requirements |