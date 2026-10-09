Create balanced quiz videos and answer packs with Google Sheets and Zvid

https://n8nworkflows.xyz/workflows/create-balanced-quiz-videos-and-answer-packs-with-google-sheets-and-zvid-20486


# Create balanced quiz videos and answer packs with Google Sheets and Zvid

### 1. Workflow Overview

This workflow automates the creation of balanced, multi-round quiz shorts (Easy, Medium, Hard) using questions sourced from a reviewed Google Sheets question bank. It rotates answer positions, avoids previously used questions, incorporates optional background music, and interacts with the Zvid rendering engine to generate vertical short-form videos with accompanying source-backed answer CSV packs.

The workflow logic is grouped into the following functional blocks:
- **1.1 Input Reception & Configuration:** Initializes global episode rules, parameters, and triggers the process either manually or on a daily schedule.
- **1.2 Source Data Retrieval & Planning:** Reads approved quiz questions and historical episode logs, verifies uniqueness, balances difficulty tiers, and handles recovery of incomplete or previously saved rendering tasks.
- **1.3 Optional Media Check:** Validates optional background music URLs and file sizes prior to final scene compilation.
- **1.4 Video Design, Validation & Budgeting:** Compiles the structured multi-scene video payload, calls the Zvid validation endpoint, and enforces a strict per-video credit ceiling.
- **1.5 Render Submission & Reservation:** Reserves episode render keys in Google Sheets, submits the render payload to Zvid, and implements bounded retry logic for capacity limits (HTTP 429).
- **1.6 Polling, Outcome Persistence & Delivery:** Polls the Zvid rendering job until completion, saves the final outcome or failure status, downloads the final MP4 video, and exports an editorial review CSV packet.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Configuration
- **Overview:** Sets up global variables, visual branding rules, timing intervals, and entry points for manual or scheduled execution.
- **Nodes Involved:** `Start the scheduled quiz shorts`, `Test the quiz shorts workflow`, `Set quiz episode and rotation rules`.
- **Node Details:**
  - **Start the scheduled quiz shorts** (`n8n-nodes-base.scheduleTrigger`)
    - *Role:* Triggers the workflow automatically on a daily schedule (configured for 09:00 UTC).
    - *Configuration:* Cron/interval rule set to trigger at hour 9.
    - *Inputs:* None | *Outputs:* `Set quiz episode and rotation rules`
    - *Failure Modes:* Missed executions if n8n instance is offline.
  - **Test the quiz shorts workflow** (`n8n-nodes-base.manualTrigger`)
    - *Role:* Allows manual execution for testing and configuration validation.
    - *Configuration:* Default manual trigger.
    - *Inputs:* None | *Outputs:* `Set quiz episode and rotation rules`
    - *Failure Modes:* None.
  - **Set quiz episode and rotation rules** (`n8n-nodes-base.set`)
    - *Role:* Defines all global configuration parameters, including API endpoints, brand names, font/color schemes, scene durations, timezone, dry-run flags, and credit ceilings.
    - *Configuration:* Raw JSON output containing sizing, timing, and operational parameters (e.g., `dryRun: false`, `maxCreditsPerVideo: 60`, `episodeRevision: 1`).
    - *Inputs:* `Start the scheduled quiz shorts` or `Test the quiz shorts workflow` | *Outputs:* `Read the approved question bank`
    - *Failure Modes:* Invalid JSON structure or misconfigured parameters (e.g., negative credit limits).

---

#### 2.2 Source Data Retrieval & Planning
- **Overview:** Pulls questions from Google Sheets, cross-references historical episode logs to avoid duplication, and builds a balanced 3-round quiz plan (Easy, Medium, Hard).
- **Nodes Involved:** `Read the approved question bank`, `Read Quiz Episodes recovery log`, `Select an easy medium and hard question`, `Eligible source content or saved render exists?`, `Explain skipped content or recovery needs`, `Resume the accepted render job?`, `Restore the saved result and handoff context`.
- **Node Details:**
  - **Read the approved question bank** (`n8n-nodes-base.googleSheets`)
    - *Role:* Fetches raw question rows from the configured Google Sheets spreadsheet.
    - *Configuration:* Uses list mode for document and sheet selection; set with retry logic (`maxTries: 3`, wait 5000ms).
    - *Inputs:* `Set quiz episode and rotation rules` | *Outputs:* `Read Quiz Episodes recovery log`
    - *Failure Modes:* Authentication failure, missing sheet tabs, or API rate limits.
  - **Read Quiz Episodes recovery log** (`n8n-nodes-base.googleSheets`)
    - *Role:* Reads the `Quiz Episodes` recovery log tab to check for historical completions or active jobs.
    - *Configuration:* Resource: sheet, operation: read, sheetName: `Quiz Episodes`.
    - *Inputs:* `Read the approved question bank` | *Outputs:* `Select an easy medium and hard question`
    - *Failure Modes:* Missing columns (`RenderKey`, `Status`, `JobId`, etc.) or sheet read errors.
  - **Select an easy medium and hard question** (`n8n-nodes-base.code`)
    - *Role:* Executes complex JavaScript business logic to filter out used questions, validate constraints (character limits, HTTPS source URLs, required FunFacts), randomize answer placements reproducibly, and construct an active episode plan.
    - *Configuration:* Custom JavaScript implementing helper validation functions, duplicate checks, and difficulty tier grouping.
    - *Inputs:* `Read Quiz Episodes recovery log` | *Outputs:* `Eligible source content or saved render exists?`
    - *Failure Modes:* Insufficient approved questions in any difficulty tier, throwing an error if requirements are unmet.
  - **Eligible source content or saved render exists?** (`n8n-nodes-base.if`)
    - *Role:* Branching node that checks whether valid source content or a resumable render plan was successfully generated.
    - *Configuration:* Condition: `{{ $json.ready === true }}`.
    - *Inputs:* `Select an easy medium and hard question` | *Outputs:* True branch -> `Resume the accepted render job?`; False branch -> `Explain skipped content or recovery needs`.
    - *Failure Modes:* Evaluation mismatch.
  - **Explain skipped content or recovery needs** (`n8n-nodes-base.code`)
    - *Role:* Passes through information regarding skipped items or missing work.
    - *Configuration:* Simple array pass-through (`return $input.all();`).
    - *Inputs:* `Eligible source content or saved render exists?` (False) | *Outputs:* None (Terminal node for unready states).
    - *Failure Modes:* None.
  - **Resume the accepted render job?** (`n8n-nodes-base.if`)
    - *Role:* Determines whether an existing accepted job ID exists and needs resumption.
    - *Configuration:* Condition: `{{ Boolean($json.jobId) }}`.
    - *Inputs:* `Eligible source content or saved render exists?` (True) | *Outputs:* True branch -> `Restore the saved result and handoff context`; False branch -> `A background music URL is configured?`.
    - *Failure Modes:* Evaluation error.
  - **Restore the saved result and handoff context** (`n8n-nodes-base.code`)
    - *Role:* Preserves and formats state for resumed render jobs.
    - *Configuration:* Pass-through code node.
    - *Inputs:* `Resume the accepted render job?` (True) | *Outputs:* `Track the saved job and its source context` (bypassing music/build/validate stages).
    - *Failure Modes:* None.

---

#### 2.3 Optional Media Check
- **Overview:** Validates optional background music assets for availability and compliance with size limits.
- **Nodes Involved:** `A background music URL is configured?`, `Check the optional music file size and availability`, `Keep the source plan and usable music only`, `Prepare the selected source for design`.
- **Node Details:**
  - **A background music URL is configured?** (`n8n-nodes-base.if`)
    - *Role:* Checks if a background music URL has been provided in the global configuration.
    - *Configuration:* Condition checks if `musicUrl` evaluates to a non-empty boolean.
    - *Inputs:* `Resume the accepted render job?` (False) | *Outputs:* True branch -> `Check the optional music file size and availability`; False branch -> `Prepare the selected source for design`.
    - *Failure Modes:* None.
  - **Check the optional music file size and availability** (`n8n-nodes-base.httpRequest`)
    - *Role:* Performs a HTTP HEAD request to check music file reachability, headers, and content length.
    - *Configuration:* Method: HEAD, timeout: 8000ms, error handling: `continueRegularOutput`.
    - *Inputs:* `A background music URL is configured?` (True) | *Outputs:* `Keep the source plan and usable music only`
    - *Failure Modes:* Network timeout or unreachable URL.
  - **Keep the source plan and usable music only** (`n8n-nodes-base.code`)
    - *Role:* Evaluates HTTP response status and `content-length` against `maxMusicBytes`. Sets `musicOk` to true or false.
    - *Configuration:* JavaScript validation logic.
    - *Inputs:* `Check the optional music file size and availability` | *Outputs:* `Prepare the selected source for design`
    - *Failure Modes:* Invalid headers.
  - **Prepare the selected source for design** (`n8n-nodes-base.code`)
    - *Role:* Prepares data context before handing off to the scene builder.
    - *Configuration:* Simple pass-through.
    - *Inputs:* `Keep the source plan and usable music only` or `A background music URL is configured?` (False) | *Outputs:* `Build the three-round quiz episode`
    - *Failure Modes:* None.

---

#### 2.4 Video Design, Validation & Budgeting
- **Overview:** Generates the multi-scene JSON payload, validates the design against the Zvid API, and enforces credit limits.
- **Nodes Involved:** `Build the three-round quiz episode`, `Validate the video and request its credit quote`, `Enforce the per-video credit ceiling`, `Return a validation-only preview?`, `Return the source plan design quote and handoff`.
- **Node Details:**
  - **Build the three-round quiz episode** (`n8n-nodes-base.code`)
    - *Role:* Generates the complete scene graph (Question, Countdown, Reveal scenes for all 3 rounds), applying dynamic typography scaling, SVG background decorations, and color palettes.
    - *Configuration:* Extensive JavaScript project builder script.
    - *Inputs:* `Prepare the selected source for design` | *Outputs:* `Validate the video and request its credit quote`
    - *Failure Modes:* Text length violations or missing options.
  - **Validate the video and request its credit quote** (`@zvid/n8n-nodes-zvid.zvid`)
    - *Role:* Communicates with the Zvid API to validate the generated project JSON and return a cost estimate.
    - *Configuration:* Resource: `render`, Operation: `validate`, Source: `json`, projectJson mapped to stringified payload.
    - *Inputs:* `Build the three-round quiz episode` | *Outputs:* `Enforce the per-video credit ceiling`
    - *Failure Modes:* Zvid API authentication failure or schema validation errors.
  - **Enforce the per-video credit ceiling** (`n8n-nodes-base.code`)
    - *Role:* Verifies that the quoted credit cost does not exceed `maxCreditsPerVideo` and establishes the recovery checkpoint snapshot.
    - *Configuration:* JavaScript validation; throws an error if credits required exceed the ceiling or if the recovery snapshot exceeds 45,000 characters.
    - *Inputs:* `Validate the video and request its credit quote` | *Outputs:* `Return a validation-only preview?`
    - *Failure Modes:* Credit limit exceeded or invalid JSON payload.
  - **Return a validation-only preview?** (`n8n-nodes-base.if`)
    - *Role:* Checks if `dryRun` is enabled to terminate execution early with design data only.
    - *Configuration:* Condition: `{{ $('Set quiz episode and rotation rules').first().json.dryRun === true }}`.
    - *Inputs:* `Enforce the per-video credit ceiling` | *Outputs:* True branch -> `Return the source plan design quote and handoff`; False branch -> `Reserve this render key before spending credits`.
    - *Failure Modes:* None.
  - **Return the source plan design quote and handoff** (`n8n-nodes-base.code`)
    - *Role:* Formats preview-mode output when `dryRun` is active.
    - *Configuration:* Code node returning preview status object.
    - *Inputs:* `Return a validation-only preview?` (True) | *Outputs:* None (Terminal node for dry runs).
    - *Failure Modes:* None.

---

#### 2.5 Render Submission & Reservation
- **Overview:** Reserves render keys in Google Sheets, submits jobs to Zvid, handles capacity limitations (HTTP 429) via retry loops, and tracks job IDs.
- **Nodes Involved:** `Reserve this render key before spending credits`, `Prepare a bounded render submission attempt`, `Submit this approved video to Zvid`, `Distinguish a capacity rejection from an uncertain submission`, `Record the rejected or uncertain submission`, `Restore the submission rejection and retry timing`, `Retry this explicit capacity rejection?`, `Wait before retrying available render capacity`, `Report why the render submission stopped`, `Capture the accepted render job ID`, `Persist the accepted job for recovery`, `Restore the accepted render context`.
- **Node Details:**
  - **Reserve this render key before spending credits** (`n8n-nodes-base.googleSheets`)
    - *Role:* Appends or updates the Google Sheet row with a `submitting` status before triggering paid rendering.
    - *Configuration:* Operation: `appendOrUpdate`, matching columns: `RenderKey`.
    - *Inputs:* `Return a validation-only preview?` (False) | *Outputs:* `Prepare a bounded render submission attempt`
    - *Failure Modes:* Google Sheets API write errors.
  - **Prepare a bounded render submission attempt** (`n8n-nodes-base.code`)
    - *Role:* Initializes submission attempt counters and timestamps.
    - *Configuration:* Code node tracking `submitAttempts` and `submitStartedAt`.
    - *Inputs:* `Reserve this render key before spending credits` | *Outputs:* `Submit this approved video to Zvid`
    - *Failure Modes:* None.
  - **Submit this approved video to Zvid** (`@zvid/n8n-nodes-zvid.zvid`)
    - *Role:* Submits the video render job to Zvid in asynchronous mode (`waitForCompletion: false`).
    - *Configuration:* Resource: `render`, Operation: `create`, renderType: `video`, error handling: `continueErrorOutput`.
    - *Inputs:* `Prepare a bounded render submission attempt` | *Outputs:* Main success output -> `Capture the accepted render job ID`; Error output -> `Distinguish a capacity rejection from an uncertain submission`.
    - *Failure Modes:* HTTP 429 rate limiting, network failure, or API rejection.
  - **Distinguish a capacity rejection from an uncertain submission** (`n8n-nodes-base.code`)
    - *Role:* Analyzes submission errors to classify whether an error is a temporary rate limit (HTTP 429) or an uncertain failure requiring manual inspection.
    - *Configuration:* JavaScript error classifier parsing HTTP status codes and headers.
    - *Inputs:* `Submit this approved video to Zvid` (Error) | *Outputs:* `Record the rejected or uncertain submission`
    - *Failure Modes:* Unparseable error structures.
  - **Record the rejected or uncertain submission** (`n8n-nodes-base.googleSheets`)
    - *Role:* Updates the Google Sheets log with the rejected/uncertain status.
    - *Configuration:* Operation: `update`, matching on `RenderKey`.
    - *Inputs:* `Distinguish a capacity rejection from an uncertain submission` | *Outputs:* `Restore the submission rejection and retry timing`
    - *Failure Modes:* Sheet write errors.
  - **Restore the submission rejection and retry timing** (`n8n-nodes-base.code`)
    - *Role:* Preserves rejection state data.
    - *Configuration:* Code node pass-through.
    - *Inputs:* `Record the rejected or uncertain submission` | *Outputs:* `Retry this explicit capacity rejection?`
    - *Failure Modes:* None.
  - **Retry this explicit capacity rejection?** (`n8n-nodes-base.if`)
    - *Role:* Checks if retry attempts are within bounds (max 3 attempts and within timeout deadline).
    - *Configuration:* Condition: `{{ $json.retry === true }}`.
    - *Inputs:* `Restore the submission rejection and retry timing` | *Outputs:* True branch -> `Wait before retrying available render capacity`; False branch -> `Report why the render submission stopped`.
    - *Failure Modes:* None.
  - **Wait before retrying available render capacity** (`n8n-nodes-base.wait`)
    - *Role:* Pauses execution for the specified retry delay seconds before attempting re-submission.
    - *Configuration:* Unit: seconds, amount derived from retry headers (`{{ $json.retrySeconds }}`).
    - *Inputs:* `Retry this explicit capacity rejection?` (True) | *Outputs:* `Prepare a bounded render submission attempt` (Loop back).
    - *Failure Modes:* Wait node timeout issues.
  - **Report why the render submission stopped** (`n8n-nodes-base.code`)
    - *Role:* Terminates workflow execution by throwing an error with the detailed rejection reason.
    - *Configuration:* `throw Error($json.reason);`
    - *Inputs:* `Retry this explicit capacity rejection?` (False) | *Outputs:* None (Terminal node).
    - *Failure Modes:* Intentional error throw.
  - **Capture the accepted render job ID** (`n8n-nodes-base.code`)
    - *Role:* Extracts and records the Zvid job ID returned upon successful submission.
    - *Configuration:* JavaScript extractor validating job ID presence.
    - *Inputs:* `Submit this approved video to Zvid` (Success) | *Outputs:* `Persist the accepted job for recovery`
    - *Failure Modes:* Missing job ID in API response.
  - **Persist the accepted job for recovery** (`n8n-nodes-base.googleSheets`)
    - *Role:* Updates the Google Sheets log status to `rendering` and records the Zvid `JobId`.
    - *Configuration:* Operation: `update`, matching `RenderKey`.
    - *Inputs:* `Capture the accepted render job ID` | *Outputs:* `Restore the accepted render context`
    - *Failure Modes:* Sheet write failure.
  - **Restore the accepted render context** (`n8n-nodes-base.code`)
    - *Role:* Passes the accepted render context forward.
    - *Configuration:* Code node pass-through.
    - *Inputs:* `Persist the accepted job for recovery` | *Outputs:* `Track the saved job and its source context`
    - *Failure Modes:* None.

---

#### 2.6 Polling, Outcome Persistence & Delivery
- **Overview:** Polls Zvid for render completion, records the final outcome in Google Sheets, delivers the finished MP4 video, and exports an editorial CSV review package.
- **Nodes Involved:** `Track the saved job and its source context`, `Wait for this accepted Zvid render`, `Classify completion failure or uncertain status`, `Record the render outcome and completed URL`, `Prepare the completed result and review pack`, `Completed video is ready to watch?`, `▶ Watch video`, `Download the source-specific review pack`.
- *Node Details:*
  - **Track the saved job and its source context** (`n8n-nodes-base.code`)
    - *Role:* Prepares job tracking parameters and start times for polling.
    - *Configuration:* Code node recording `pollStartedAt`.
    - *Inputs:* `Restore the accepted render context` or `Restore the saved result and handoff context` | *Outputs:* `Wait for this accepted Zvid render`
    - *Failure Modes:* None.
  - **Wait for this accepted Zvid render** (`@zvid/n8n-nodes-zvid.zvid`)
    - *Role:* Polls the Zvid API until the render job completes, fails, or times out.
    - *Configuration:* Resource: `render`, Operation: `get`, `waitForCompletion: true`, poll interval and timeout derived from global settings. Error handling: `continueRegularOutput`.
    - *Inputs:* `Track the saved job and its source context` | *Outputs:* `Classify completion failure or uncertain status`
    - *Failure Modes:* Polling timeout or API error responses.
  - **Classify completion failure or uncertain status** (`n8n-nodes-base.code`)
    - *Role:* Evaluates polling results to determine if status is `completed`, `failed`, `timed_out`, or `needs_review`.
    - *Configuration:* JavaScript outcome classifier.
    - *Inputs:* `Wait for this accepted Zvid render` | *Outputs:* `Record the render outcome and completed URL`
    - *Failure Modes:* Unrecognized job states.
  - **Record the render outcome and completed URL** (`n8n-nodes-base.googleSheets`)
    - *Role:* Persists the final status, video URL, completion timestamp, and outcome checkpoint into Google Sheets.
    - *Configuration:* Operation: `update`, matching on `RenderKey`.
    - *Inputs:* `Classify completion failure or uncertain status` | *Outputs:* `Prepare the completed result and review pack`
    - *Failure Modes:* Sheet write errors.
  - **Prepare the completed result and review pack** (`n8n-nodes-base.code`)
    - *Role:* Validates that the render successfully completed with a valid video URL before proceeding to delivery and download nodes.
    - *Configuration:* JavaScript validator throwing an error if the video is not ready.
    - *Inputs:* `Record the render outcome and completed URL` | *Outputs:* Split into two branches: `Completed video is ready to watch?` and `Download the source-specific review pack`.
    - *Failure Modes:* Incomplete status or missing video URL.
  - **Completed video is ready to watch?** (`n8n-nodes-base.if`)
    - *Role:* Conditional check verifying readiness for video binary retrieval.
    - *Configuration:* Condition checking `status === 'completed'`, `!dryRun`, and valid HTTPS video URL.
    - *Inputs:* `Prepare the completed result and review pack` | *Outputs:* True branch -> `▶ Watch video`; False branch -> empty.
    - *Failure Modes:* None.
  - **▶ Watch video** (`n8n-nodes-base.httpRequest`)
    - *Role:* Downloads the completed MP4 video file from the Zvid storage URL into binary data.
    - *Configuration:* Response format: file, data property: `data`, timeout: 30000ms, retry on fail (`maxTries: 3`, wait 5000ms), error handling: `continueRegularOutput`.
    - *Inputs:* `Completed video is ready to watch?` (True) | *Outputs:* None (Terminal output for video binary).
    - *Failure Modes:* Network timeout or expired URL.
  - **Download the source-specific review pack** (`n8n-nodes-base.code`)
    - *Role:* Generates a base64-encoded CSV file containing source questions, correct answers, fun facts, and source URLs for editorial review.
    - *Configuration:* JavaScript CSV builder outputting binary file data (`quiz-shorts-review.csv`).
    - *Inputs:* `Prepare the completed result and review pack` | *Outputs:* None (Terminal output for CSV binary).
    - *Failure Modes:* Empty handoff arrays.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Workflow overview and setup** | `n8n-nodes-base.stickyNote` | Documentation and setup instructions | None | None | ## Create balanced quiz episodes with an answer and source pack... [Background Guide](https://github.com/Zvid-io/zvid-n8n/blob/master/workflows/quiz-shorts.md) |
| **Start the scheduled quiz shorts** | `n8n-nodes-base.scheduleTrigger` | Triggers workflow on a daily schedule | None | Set quiz episode and rotation rules | ## 1. Choose source rules and schedule |
| **Test the quiz shorts workflow** | `n8n-nodes-base.manualTrigger` | Manual execution entry point | None | Set quiz episode and rotation rules | ## 1. Choose source rules and schedule |
| **Set quiz episode and rotation rules** | `n8n-nodes-base.set` | Initializes global configuration | Start the scheduled quiz shorts, Test the quiz shorts workflow | Read the approved question bank | ## 1. Choose source rules and schedule |
| **Read the approved question bank** | `n8n-nodes-base.googleSheets` | Fetches question rows from Google Sheets | Set quiz episode and rotation rules | Read Quiz Episodes recovery log | ## 2. Check the real source and plan the output |
| **Read Quiz Episodes recovery log** | `n8n-nodes-base.googleSheets` | Reads historical episode log | Read the approved question bank | Select an easy medium and hard question | ## 2. Check the real source and plan the output |
| **Select an easy medium and hard question** | `n8n-nodes-base.code` | Plans balanced 3-round quiz & filters duplicates | Read Quiz Episodes recovery log | Eligible source content or saved render exists? | ## 2. Check the real source and plan the output |
| **Eligible source content or saved render exists?** | `n8n-nodes-base.if` | Verifies question readiness | Select an easy medium and hard question | Resume the accepted render job?, Explain skipped content or recovery needs | ## 2. Check the real source and plan the output |
| **Explain skipped content or recovery needs** | `n8n-nodes-base.code` | Explains unready state or skips | Eligible source content or saved render exists? | None | ## 2. Check the real source and plan the output |
| **Resume the accepted render job?** | `n8n-nodes-base.if` | Checks for resumable job ID | Eligible source content or saved render exists? | Restore the saved result and handoff context, A background music URL is configured? | ## 3. Recover accepted work before a new render |
| **Restore the saved result and handoff context** | `n8n-nodes-base.code` | Restores prior job context | Resume the accepted render job? | Track the saved job and its source context | ## 3. Recover accepted work before a new render |
| **A background music URL is configured?** | `n8n-nodes-base.if` | Checks if background music is specified | Resume the accepted render job? | Check the optional music file size and availability, Prepare the selected source for design | ## 4. Check optional background music |
| **Check the optional music file size and availability** | `n8n-nodes-base.httpRequest` | HEAD request to verify music file | A background music URL is configured? | Keep the source plan and usable music only | ## 4. Check optional background music |
| **Keep the source plan and usable music only** | `n8n-nodes-base.code` | Validates music file size limits | Check the optional music file size and availability | Prepare the selected source for design | ## 4. Check optional background music |
| **Prepare the selected source for design** | `n8n-nodes-base.code` | Passes design source data forward | Keep the source plan and usable music only, A background music URL is configured? | Build the three-round quiz episode | ## 4. Check optional background music |
| **Build the three-round quiz episode** | `n8n-nodes-base.code` | Generates 3-scene project JSON | Prepare the selected source for design | Validate the video and request its credit quote | ## 5. Build and validate the design and credit ceiling |
| **Validate the video and request its credit quote** | `@zvid/n8n-nodes-zvid.zvid` | Validates project JSON via Zvid API | Build the three-round quiz episode | Enforce the per-video credit ceiling | ## 5. Build and validate the design and credit ceiling |
| **Enforce the per-video credit ceiling** | `n8n-nodes-base.code` | Enforces max credits per video | Validate the video and request its credit quote | Return a validation-only preview? | ## 5. Build and validate the design and credit ceiling |
| **Return a validation-only preview?** | `n8n-nodes-base.if` | Checks if dryRun mode is enabled | Enforce the per-video credit ceiling | Return the source plan design quote and handoff, Reserve this render key before spending credits | ## 5. Build and validate the design and credit ceiling |
| **Return the source plan design quote and handoff** | `n8n-nodes-base.code` | Formats preview-only output | Return a validation-only preview? | None | ## 5. Build and validate the design and credit ceiling |
| **Reserve this render key before spending credits** | `n8n-nodes-base.googleSheets` | Appends/updates sheet with submitting status | Return a validation-only preview? | Prepare a bounded render submission attempt | ## 6. Reserve the key and preserve the accepted job |
| **Prepare a bounded render submission attempt** | `n8n-nodes-base.code` | Tracks submission attempt counts | Reserve this render key before spending credits | Submit this approved video to Zvid | ## 6. Reserve the key and preserve the accepted job |
| **Submit this approved video to Zvid** | `@zvid/n8n-nodes-zvid.zvid` | Submits render job to Zvid API | Prepare a bounded render submission attempt | Capture the accepted render job ID, Distinguish a capacity rejection from an uncertain submission | ## 6. Reserve the key and preserve the accepted job |
| **Distinguish a capacity rejection from an uncertain submission** | `n8n-nodes-base.code` | Classifies HTTP 429 vs uncertain errors | Submit this approved video to Zvid | Record the rejected or uncertain submission | ## 6. Reserve the key and preserve the accepted job |
| **Record the rejected or uncertain submission** | `n8n-nodes-base.googleSheets` | Records rejection in recovery log | Distinguish a capacity rejection from an uncertain submission | Restore the submission rejection and retry timing | ## 6. Reserve the key and preserve the accepted job |
| **Restore the submission rejection and retry timing** | `n8n-nodes-base.code` | Restores rejection state and retry delay | Record the rejected or uncertain submission | Retry this explicit capacity rejection? | ## 6. Reserve the key and preserve the accepted job |
| **Retry this explicit capacity rejection?** | `n8n-nodes-base.if` | Checks retry bounds and timeout deadlines | Restore the submission rejection and retry timing | Wait before retrying available render capacity, Report why the render submission stopped | ## 6. Reserve the key and preserve the accepted job |
| **Wait before retrying available render capacity** | `n8n-nodes-base.wait` | Pauses execution before rate-limit retry | Retry this explicit capacity rejection? | Prepare a bounded render submission attempt | ## 6. Reserve the key and preserve the accepted job |
| **Report why the render submission stopped** | `n8n-nodes-base.code` | Throws terminal error for halted submissions | Retry this explicit capacity rejection? | None | ## 6. Reserve the key and preserve the accepted job |
| **Capture the accepted render job ID** | `n8n-nodes-base.code` | Extracts JobId from Zvid response | Submit this approved video to Zvid | Persist the accepted job for recovery | ## 6. Reserve the key and preserve the accepted job |
| **Persist the accepted job for recovery** | `n8n-nodes-base.googleSheets` | Updates sheet status to rendering | Capture the accepted render job ID | Restore the accepted render context | ## 6. Reserve the key and preserve the accepted job |
| **Restore the accepted render context** | `n8n-nodes-base.code` | Passes render context forward | Persist the accepted job for recovery | Track the saved job and its source context | ## 6. Reserve the key and preserve the accepted job |
| **Track the saved job and its source context** | `n8n-nodes-base.code` | Prepares parameters for job polling | Restore the accepted render context, Restore the saved result and handoff context | Wait for this accepted Zvid render | ## 7. Wait for the accepted render and record its result |
| **Wait for this accepted Zvid render** | `@zvid/n8n-nodes-zvid.zvid` | Polls Zvid until render finishes | Track the saved job and its source context | Classify completion failure or uncertain status | ## 7. Wait for the accepted render and record its result |
| **Classify completion failure or uncertain status** | `n8n-nodes-base.code` | Classifies polling completion status | Wait for this accepted Zvid render | Record the render outcome and completed URL | ## 7. Wait for the accepted render and record its result |
| **Record the render outcome and completed URL** | `n8n-nodes-base.googleSheets` | Records final render outcome in sheet | Classify completion failure or uncertain status | Prepare the completed result and review pack | ## 7. Wait for the accepted render and record its result |
| **Prepare the completed result and review pack** | `n8n-nodes-base.code` | Validates completed video URL | Record the render outcome and completed URL | Completed video is ready to watch?, Download the source-specific review pack | ## 8. Review the video and source-specific handoff |
| **Completed video is ready to watch?** | `n8n-nodes-base.if` | Verifies completion status for download | Prepare the completed result and review pack | ▶ Watch video | ## 8. Review the video and source-specific handoff |
| **▶ Watch video** | `n8n-nodes-base.httpRequest` | Downloads final MP4 video binary | Completed video is ready to watch? | None | ## 8. Review the video and source-specific handoff |
| **Download the source-specific review pack** | `n8n-nodes-base.code` | Generates review CSV file binary | Prepare the completed result and review pack | None | ## 8. Review the video and source-specific handoff |

---

### 4. Reproducing the Workflow from Scratch

To rebuild this workflow manually from scratch, follow these step-by-step instructions:

1. **Install Prerequisites:**
   - Ensure your self-hosted n8n instance has community nodes enabled.
   - Go to **Settings → Community nodes**, install **`@zvid/n8n-nodes-zvid`**.
   - Create a Zvid API credential using your API key from [Zvid API Keys](https://app.zvid.io/api-keys) and Base URL `https://api.zvid.io`.

2. **Set up Google Sheets:**
   - Create a Google Sheets document with two tabs:
     - **Question Bank Tab:** Columns required: `QuestionKey`, `Question`, `OptionA`, `OptionB`, `OptionC`, `CorrectLetter`, `FunFact`, `Topic`, `Difficulty`, `SourceUrl`, `Approval`. Populate with approved rows (`Approval=approved`, easy/medium/hard difficulties, HTTPS sources).
     - **Quiz Episodes Tab:** Columns required: `RenderKey`, `Status`, `JobId`, `CreditsQuoted`, `PlanJson`, `VideoUrl`, `CompletedAt`.

3. **Create Entry Points & Configuration Nodes:**
   - **Node 1 (`Start the scheduled quiz shorts`):** Add a `Schedule Trigger` node. Set interval to trigger at hour `9`.
   - **Node 2 (`Test the quiz shorts workflow`):** Add a `Manual Trigger` node.
   - **Node 3 (`Set quiz episode and rotation rules`):** Add a `Set` node (mode: raw, JSON output). Paste the configuration JSON containing branding (`BRAINWAVE`), resolution (`youtube-short`), timing parameters, `dryRun: false`, and `maxCreditsPerVideo: 60`. Connect both triggers to this node.

4. **Build Source Reading & Planning Logic:**
   - **Node 4 (`Read the approved question bank`):** Add a Google Sheets node (operation: read/list) connected to your question bank sheet. Configure retry options (`maxTries: 3`, wait 5000ms).
   - **Node 5 (`Read Quiz Episodes recovery log`):** Add a Google Sheets node configured to read the `Quiz Episodes` tab.
   - **Node 6 (`Select an easy medium and hard question`):** Add a Code node containing the planning and deduplication JavaScript function that selects one easy, one medium, and one hard question while rotating answer positions reproducibly.
   - **Node 7 (`Eligible source content or saved render exists?`):** Add an IF node evaluating `{{ $json.ready === true }}`.
   - **Node 8 (`Explain skipped content or recovery needs`):** Add a Code node (`return $input.all();`) attached to the false branch.
   - **Node 9 (`Resume the accepted render job?`):** Add an IF node on the true branch evaluating `{{ Boolean($json.jobId) }}`.
   - **Node 10 (`Restore the saved result and handoff context`):** Add a Code node pass-through attached to the true branch of Node 9.

5. **Build Optional Music Check Logic:**
   - **Node 11 (`A background music URL is configured?`):** Add an IF node checking if `musicUrl` is provided in the configuration.
   - **Node 12 (`Check the optional music file size and availability`):** Add an HTTP Request node (method: HEAD, timeout: 8000ms, continue on error) for the music URL.
   - **Node 13 (`Keep the source plan and usable music only`):** Add a Code node validating `content-length` against `maxMusicBytes`.
   - **Node 14 (`Prepare the selected source for design`):** Add a Code node uniting the music check output.

6. **Build Scene Design & Zvid Validation Nodes:**
   - **Node 15 (`Build the three-round quiz episode`):** Add a Code node implementing the `buildProject` function to generate the 3-scene JSON payload (Question, Countdown, Reveal).
   - **Node 16 (`Validate the video and request its credit quote`):** Add a Zvid node (resource: `render`, operation: `validate`, source: `json`, projectJson mapped to stringified payload). Select your Zvid credential.
   - **Node 17 (`Enforce the per-video credit ceiling`):** Add a Code node validating credit quotas against `maxCreditsPerVideo` and establishing the recovery checkpoint.
   - **Node 18 (`Return a validation-only preview?`):** Add an IF node evaluating `{{ $('Set quiz episode and rotation rules').first().json.dryRun === true }}`.
   - **Node 19 (`Return the source plan design quote and handoff`):** Add a Code node for dry-run output formatting.

7. **Build Submission, Reservation & Retry Logic:**
   - **Node 20 (`Reserve this render key before spending credits`):** Add a Google Sheets node (operation: `appendOrUpdate`, matching on `RenderKey`) updating the sheet with `Status: "submitting"`.
   - **Node 21 (`Prepare a bounded render submission attempt`):** Add a Code node initializing attempt counters.
   - **Node 22 (`Submit this approved video to Zvid`):** Add a Zvid node (resource: `render`, operation: `create`, `waitForCompletion: false`, continue on error).
   - **Node 23 (`Distinguish a capacity rejection from an uncertain submission`):** Add a Code node connected to the error output of Node 22 to classify HTTP 429 rate limits versus uncertain errors.
   - **Node 24 (`Record the rejected or uncertain submission`):** Add a Google Sheets node updating the sheet with rejection status.
   - **Node 25 (`Restore the submission rejection and retry timing`):** Add a Code node pass-through.
   - **Node 26 (`Retry this explicit capacity rejection?`):** Add an IF node evaluating `{{ $json.retry === true }}`.
   - **Node 27 (`Wait before retrying available render capacity`):** Add a Wait node using retry seconds. Connect its output back to Node 21.
   - **Node 28 (`Report why the render submission stopped`):** Add a Code node throwing an error if retries are exhausted (`throw Error($json.reason);`).
   - **Node 29 (`Capture the accepted render job ID`):** Add a Code node extracting the Zvid `jobId` from successful submissions.
   - **Node 30 (`Persist the accepted job for recovery`):** Add a Google Sheets node updating sheet status to `rendering` and saving `JobId`.
   - **Node 31 (`Restore the accepted render context`):** Add a Code node pass-through.

8. **Build Polling, Persistence & Delivery Nodes:**
   - **Node 32 (`Track the saved job and its source context`):** Add a Code node preparing polling timestamps (connect from both Node 10 and Node 31).
   - **Node 33 (`Wait for this accepted Zvid render`):** Add a Zvid node (resource: `render`, operation: `get`, `waitForCompletion: true`, poll interval/timeout from config, continue on error).
   - **Node 34 (`Classify completion failure or uncertain status`):** Add a Code node evaluating polling results.
   - **Node 35 (`Record the render outcome and completed URL`):** Add a Google Sheets node updating the sheet with the final outcome, video URL, and completion timestamp.
   - **Node 36 (`Prepare the completed result and review pack`):** Add a Code node validating successful completion.
   - **Node 37 (`Completed video is ready to watch?`):** Add an IF node verifying completion status and HTTPS URL.
   - **Node 38 (`▶ Watch video`):** Add an HTTP Request node (response format: file, data property `data`, timeout: 30000ms, retry on fail).
   - **Node 39 (`Download the source-specific review pack`):** Add a Code node generating the base64 CSV attachment (`quiz-shorts-review.csv`).

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Zvid Community Node Installation** | Requires `@zvid/n8n-nodes-zvid` installed via Settings → Community nodes. |
| **Zvid API Credentials & Portal** | Create API credentials at [Zvid API Keys](https://app.zvid.io/api-keys) with Base URL `https://api.zvid.io`. |
| **Background Guide & Documentation** | Detailed background guide available in the [Zvid n8n Quiz Shorts Repository Guide](https://github.com/Zvid-io/zvid-n8n/blob/master/workflows/quiz-shorts.md). |
| **Support & Contact** | Official support channel via [Zvid Contact](https://zvid.io/contact). |