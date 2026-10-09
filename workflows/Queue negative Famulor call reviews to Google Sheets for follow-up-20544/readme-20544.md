Queue negative Famulor call reviews to Google Sheets for follow-up

https://n8nworkflows.xyz/workflows/queue-negative-famulor-call-reviews-to-google-sheets-for-follow-up-20544


# Queue negative Famulor call reviews to Google Sheets for follow-up

### 1. Workflow Overview

This workflow automates the retrieval of negative-sentiment customer calls from the Famulor API, validating and transforming the data, and updating a Google Sheets tracking queue for customer success follow-up. It is designed to run on a recurring hourly schedule or via manual execution, targeting calls created within a configurable lookback window (defaulting to the past 48 hours).

The workflow's execution logic is organized into three distinct functional blocks:

- **1.1 Input Reception & Configuration:** Handles workflow triggers (hourly schedule or manual run) and sets up global runtime parameters such as timezones, lookback periods, target Google Sheet IDs, and preview toggles.
- **1.2 Data Ingestion & Validation:** Calculates UTC time windows, queries the Famulor API with pagination, flattens multi-page responses, deduplicates records, and filters for completed, negative-sentiment calls.
- **1.3 Filtering, Preview, & Delivery:** Evaluates whether the run is in "Preview Mode" and either outputs a payload preview or performs an upsert operation into Google Sheets using the Call ID as the unique matching key.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Configuration

- **Overview:** Initializes the workflow execution either hourly or manually, and injects global configuration variables to control operational bounds, target spreadsheets, and execution modes.
- **Nodes Involved:**
  - `Every hour`
  - `Run a manual preview`
  - `Configure workflow`
  - `Build date window`

- **Node Details:**
  - **Every hour**
    - *Type and Role:* `n8n-nodes-base.scheduleTrigger` (Trigger). Executes the workflow automatically on a fixed hourly interval.
    - *Configuration:* Interval set to every 1 hour.
    - *Input/Output:* Output connects to `Configure workflow`.
    - *Edge Cases:* Overlapping runs if executions exceed one hour; concurrent runs should be avoided.
  - **Run a manual preview**
    - *Type and Role:* `n8n-nodes-base.manualTrigger` (Trigger). Allows ad-hoc testing and inspection of workflow output.
    - *Configuration:* Default manual trigger.
    - *Input/Output:* Output connects to `Configure workflow`.
  - **Configure workflow**
    - *Type and Role:* `n8n-nodes-base.set` (Data Transformation). Defines static environment parameters.
    - *Configuration:* Sets `businessTimezone` ("UTC"), `lookbackHours` (48), `previewOnly` (true), `sheetDocumentId` ("REPLACE_WITH_YOUR_SHEET_ID"), and `sheetTab` ("Call review").
    - *Input/Output:* Receives input from either trigger; outputs to `Build date window`.
  - **Build date window**
    - *Type and Role:* `n8n-nodes-base.set` (Data Transformation). Calculates the ISO 8601 start and end timestamps in UTC based on the business timezone and lookback hours.
    - *Key Expressions:* 
      - `windowStart`: `={{ $now.setZone($json.businessTimezone).minus({hours:$json.lookbackHours}).toUTC().toISO() }}`
      - `windowEnd`: `={{ $now.toUTC().toISO() }}`
    - *Input/Output:* Input from `Configure workflow`; output to `Read Famulor pages`.
    - *Edge Cases:* Invalid timezone strings will cause expression failures.

---

#### 2.2 Data Ingestion & Validation

- **Overview:** Queries the Famulor API using pagination, validates response integrity across pages, deduplicates records, and filters for completed calls with negative sentiment.
- **Nodes Involved:**
  - `Read Famulor pages`
  - `Validate and prepare`

- **Node Details:**
  - **Read Famulor pages**
    - *Type and Role:* `n8n-nodes-base.httpRequest` (Integration). Fetches paginated call records from the Famulor API.
    - *Configuration:* HTTP GET method targeting `https://app.famulor.io/api/v1/calls`. Uses Generic HTTP Header Auth (`Authorization: Bearer <API_KEY>`). Implements pagination fetching up to 10 pages (200 records per page, offset-based) with a 300ms interval.
    - *Key Parameters/Query:* `limit=200`, `from=={{ $json.windowStart }}`, `to=={{ $json.windowEnd }}`, `status=completed`, `sentiment=negative`, `sort=updated_at`.
    - *Input/Output:* Input from `Build date window`; output to `Validate and prepare`.
    - *Edge Cases:* Authentication failures (401/403), rate-limiting, network timeouts, or exceeding the 2,000-record pagination cap (which halts execution).
  - **Validate and prepare**
    - *Type and Role:* `n8n-nodes-base.code` (Custom JavaScript). Validates API response structures, checks pagination integrity, deduplicates records by `Call ID`, filters by sentiment and status, and formats rows for the destination sheet.
    - *Configuration:* JavaScript execution mode `runOnceForAllItems`. Limits summaries to 1,500 characters and labels missing success analyses.
    - *Input/Output:* Input from `Read Famulor pages`; output to `Has review rows?`.
    - *Edge Cases:* Throws explicit errors if API responses are malformed, result caps are reached, or incomplete page sequences occur. Returns a fallback `noData` object if no calls match.

---

#### 2.3 Filtering, Preview, & Delivery

- **Overview:** Evaluates whether data exists and whether preview mode is active, routing the payload either to a debug preview node or directly to Google Sheets for upsertion.
- **Nodes Involved:**
  - `Has review rows?`
  - `Preview only?`
  - `Preview prepared output`
  - `Upsert the review queue`

- **Node Details:**
  - **Has review rows?**
    - *Type and Role:* `n8n-nodes-base.if` (Flow Control). Checks if the validation node returned active review rows or a "no data" placeholder.
    - *Key Expressions:* `={{ $json.noData === true }}` (Branch true handles empty states; branch false proceeds with data).
    - *Input/Output:* Input from `Validate and prepare`; outputs to `Preview only?` (if data exists) or `Preview prepared output` (if empty).
  - **Preview only?**
    - *Type and Role:* `n8n-nodes-base.if` (Flow Control). Determines whether to write to production storage or halt execution in a safe preview state.
    - *Key Expressions:* `={{ $("Configure workflow").first().json.previewOnly }}`
    - *Input/Output:* Input from `Has review rows?`; outputs to `Preview prepared output` (true) or `Upsert the review queue` (false).
  - **Preview prepared output**
    - *Type and Role:* `n8n-nodes-base.code` (Data Transformation). Passes through the final items for inspection during testing.
    - *Input/Output:* Receives input from `Has review rows?` or `Preview only?`; terminal node for previews.
  - **Upsert the review queue**
    - *Type and Role:* `n8n-nodes-base.googleSheets` (Integration). Inserts or updates call review rows in the destination spreadsheet.
    - *Configuration:* Operation set to `appendOrUpdate`. Matches existing rows using `Call ID`. Document ID and Sheet Name dynamically reference configuration variables.
    - *Input/Output:* Input from `Preview only?`; terminal node for production runs.
    - *Edge Cases:* Google Sheets API rate limits, permission errors, incorrect spreadsheet ID, or race conditions from concurrent executions.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Overview and setup` | `n8n-nodes-base.stickyNote` | Documentation & Setup Guide | None | None | ## Who is this for?\nSupport and customer success teams using Famulor voice assistants who want a review queue for dissatisfied callers.\n\n### How it works\nEvery hour, the workflow reads completed calls with negative sentiment that were created in the last 48 hours. It checks all response pages before writing, deduplicates call IDs and prepares a short review row. Google Sheets appends or updates by Call ID, so overlapping windows update existing rows. Operator columns such as Owner, Follow-up status and Notes are left untouched. Missing success analysis is labelled accurately. Summaries are limited to 1,500 characters; transcripts and recordings are excluded.\n\n### Setup\n1. In Famulor, enable sentiment analysis for the assistants you want to monitor.\n2. Create an n8n Header Auth credential with name Authorization and value Bearer followed by your workspace API key. Grant calls:read and select it in Read Famulor pages. Verify the selected credential before testing; imports may preselect an existing connection.\n3. Create these sheet headers: Call ID, Created at, Ended at, Direction, Caller, Recipient, Duration minutes, Sentiment, Summary, Success, Review reason. Connect Google Sheets and set its ID and tab in Configure workflow.\n4. Execute with Preview only enabled. After checking the rows, set it to false and activate the schedule.\n\n### Customization\nAdjust the creation window and polling interval. Increase throughput by narrowing the window; an incomplete scan fails before any write. API errors stop the workflow and appear in n8n Executions. This template reads Famulor data and never initiates calls. [API setup](https://docs.famulor.io/automations/n8n). |
| `Read section` | `n8n-nodes-base.stickyNote` | Documentation Block | None | None | ## 1. Read the configured window\nSelect your Header Auth credential. Requests stay on the Famulor API origin. |
| `Prepare section` | `n8n-nodes-base.stickyNote` | Documentation Block | None | None | ## 2. Validate and prepare\nIncomplete scans stop here. No full transcripts or recordings are forwarded. |
| `Deliver section` | `n8n-nodes-base.stickyNote` | Documentation Block | None | None | ## 3. Preview, then deliver\nPreview only starts enabled. Connect the destination before switching it off. |
| `Every hour` | `n8n-nodes-base.scheduleTrigger` | Hourly workflow trigger | None | `Configure workflow` | |
| `Run a manual preview` | `n8n-nodes-base.manualTrigger` | Manual test trigger | None | `Configure workflow` | |
| `Configure workflow` | `n8n-nodes-base.set` | Sets global configurations | `Every hour`, `Run a manual preview` | `Build date window` | |
| `Build date window` | `n8n-nodes-base.set` | Computes UTC start/end timestamps | `Configure workflow` | `Read Famulor pages` | |
| `Read Famulor pages` | `n8n-nodes-base.httpRequest` | Fetches paginated calls from Famulor | `Build date window` | `Validate and prepare` | |
| `Validate and prepare` | `n8n-nodes-base.code` | Validates, deduplicates, and formats rows | `Read Famulor pages` | `Has review rows?` | |
| `Preview only?` | `n8n-nodes-base.if` | Checks if preview mode is active | `Has review rows?` | `Preview prepared output`, `Upsert the review queue` | |
| `Preview prepared output` | `n8n-nodes-base.code` | Outputs preview data | `Preview only?`, `Has review rows?` | None | |
| `Upsert the review queue` | `n8n-nodes-base.googleSheets` | Upserts rows into Google Sheets | `Preview only?` | None | |
| `Has review rows?` | `n8n-nodes-base.if` | Checks for returned call records | `Validate and prepare` | `Preview only?`, `Preview prepared output` | |

---

### 4. Reproducing the Workflow from Scratch

Follow these steps to manually recreate the workflow in n8n:

1. **Create Trigger Nodes:**
   - Add a **Schedule Trigger** node (`n8n-nodes-base.scheduleTrigger`). Name it `Every hour` and set the interval to every 1 hour.
   - Add a **Manual Trigger** node (`n8n-nodes-base.manualTrigger`). Name it `Run a manual preview`.

2. **Add Configuration Nodes:**
   - Add a **Set** node (`n8n-nodes-base.set`) named `Configure workflow`. Connect both triggers to this node. Add string/number/boolean assignments:
     - `businessTimezone` (String): `UTC`
     - `lookbackHours` (Number): `48`
     - `previewOnly` (Boolean): `true`
     - `sheetDocumentId` (String): `REPLACE_WITH_YOUR_SHEET_ID`
     - `sheetTab` (String): `Call review`
   - Add a second **Set** node (`n8n-nodes-base.set`) named `Build date window`. Connect `Configure workflow` to it. Enable "Include Other Fields". Add assignments:
     - `windowStart`: `={{ $now.setZone($json.businessTimezone).minus({hours:$json.lookbackHours}).toUTC().toISO() }}`
     - `windowEnd`: `={{ $now.toUTC().toISO() }}`

3. **Configure API Ingestion:**
   - Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`) named `Read Famulor pages`. Connect `Build date window` to it.
     - Method: `GET`
     - URL: `https://app.famulor.io/api/v1/calls`
     - Authentication: Generic Credential Type -> HTTP Header Auth (Create credential with Header Name: `Authorization`, Value: `Bearer <YOUR_WORKSPACE_API_KEY>`).
     - Query Parameters: Add `limit` (200), `from` (`={{ $json.windowStart }}`), `to` (`={{ $json.windowEnd }}`), `status` (`completed`), `sentiment` (`negative`), `sort` (`updated_at`).
     - Pagination Options: Set pagination mode to "Update a parameter in each request", query parameter name `offset`, value `={{ $pageCount * 200 }}`. Set max requests to 10, limit pages fetched to true, and supply the completion expression: `={{ Array.isArray($response.body.data) && ($response.body.data.length < 200 || (Number.isInteger($response.body.meta?.pagination?.total) && $response.body.meta.pagination.offset + $response.body.data.length >= $response.body.meta.pagination.total)) }}`.

4. **Add Data Validation & Transformation:**
   - Add a **Code** node (`n8n-nodes-base.code`) named `Validate and prepare`. Connect `Read Famulor pages` to it. Set mode to `Run Once for All Items` and insert the validation/mapping JavaScript logic provided in the original JSON (handling pagination validation, deduplication by `Call ID`, sentiment filtering, and field mapping for Call ID, Created at, Ended at, Direction, Caller, Recipient, Duration minutes, Sentiment, Summary, Success, and Review reason).

5. **Set Up Flow Control & Conditional Branching:**
   - Add an **If** node (`n8n-nodes-base.if`) named `Has review rows?`. Connect `Validate and prepare` to it. Add condition: `={{ $json.noData === true }}` equals `false`.
   - Add a second **If** node (`n8n-nodes-base.if`) named `Preview only?`. Connect the `true` output of `Has review rows?` to it. Add condition: `={{ $("Configure workflow").first().json.previewOnly }}` equals `true`.

6. **Create Output & Destination Nodes:**
   - Add a **Code** node (`n8n-nodes-base.code`) named `Preview prepared output`. Connect the `true` output of `Preview only?` and the `false` (noData) output of `Has review rows?` to this node. Add JS code: `return $input.all();`.
   - Add a **Google Sheets** node (`n8n-nodes-base.googleSheets`) named `Upsert the review queue`. Connect the `false` output of `Preview only?` to it.
     - Setup Google Sheets OAuth2 credentials.
     - Operation: `Append or Update`.
     - Document ID: `={{ $("Configure workflow").first().json.sheetDocumentId }}`
     - Sheet Name: `={{ $("Configure workflow").first().json.sheetTab }}`
     - Mapping Mode: Define below, matching columns set to `Call ID`. Map each column (Call ID, Created at, Ended at, Direction, Caller, Recipient, Duration minutes, Sentiment, Summary, Success, Review reason) to its respective JSON property.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Famulor API Setup Guide | [Famulor API Documentation](https://docs.famulor.io/automations/n8n) |