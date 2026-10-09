Create narrated KPI exception videos and action registers with Zvid and Google Sheets

https://n8nworkflows.xyz/workflows/create-narrated-kpi-exception-videos-and-action-registers-with-zvid-and-google-sheets-20487


# Create narrated KPI exception videos and action registers with Zvid and Google Sheets

### 1. Workflow Overview

This workflow automates the generation of narrated (or music-only) KPI exception videos and structured owner action registers from reporting snapshots. Its primary use cases include turning weekly reporting data into executive-ready video briefings and CSV action registers, prioritizing performance anomalies, and securely logging rendering jobs to Google Sheets to prevent duplicate spending. 

The workflow logic is categorized into the following functional blocks:
- **1.1 Input Reception & Configuration:** Initializes global execution parameters and provides manual or weekly scheduled entry points.
- **1.2 Data Ingestion & Mode Selection:** Fetches live reporting metrics via HTTPS or loads labeled sample data for testing.
- **1.3 Planning, Recovery & Validation:** Reads the Google Sheets recovery log, calculates performance changes and target misses, prioritizes metrics, and resumes unfinished jobs.
- **1.4 Asset Enhancement (Music & Voiceover):** Optionally validates background music files and generates synchronized AI voiceovers using ElevenLabs with timestamp alignment.
- **1.5 Video Design, Validation & Budget Control:** Compiles visual scenes, validates the project schema using Zvid nodes, and enforces a strict per-video credit ceiling.
- **1.6 Job Submission, Polling & Tracking:** Reserves unique render keys in Google Sheets, submits jobs to Zvid, handles capacity rate limits with exponential retries, and polls until completion.
- **1.7 Post-Processing, Review & Optional Delivery:** Delivers review packs (CSV downloads, video previews) and optionally posts summaries to Slack.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Reception & Configuration
- **Overview:** Establishes execution triggers (manual or weekly schedule) and loads core environment variables, brand aesthetics, and thresholds.
- **Nodes Involved:** `Start the scheduled kpi report video`, `Test the kpi report video workflow`, `Set reporting period and exception thresholds`
- **Node Details:**
  - `Start the scheduled kpi report video`: Schedule Trigger running weekly on Mondays at 09:00 UTC. Connects to the configuration set node.
  - `Test the kpi report video workflow`: Manual Trigger allowing on-demand execution. Connects to the configuration set node.
  - `Set reporting period and exception thresholds`: Set Node (Raw JSON mode). Establishes parameters such as `apiUrl`, `metricsUrl`, `narrate`, `brandName`, `alertPercent`, `maxCreditsPerVideo`, etc. Outputs configuration objects downstream.

#### Block 1.2: Data Ingestion & Mode Selection
- **Overview:** Evaluates whether to use live reporting endpoints or predefined sample data, ensuring secure data retrieval and schema validation.
- **Nodes Involved:** `Use explicitly selected sample metrics?`, `Create labeled sample metrics for testing`, `Require a live reporting URL before fetching`, `Fetch the reporting source snapshot`, `Choose the configured metrics input`
- **Node Details:**
  - `Use explicitly selected sample metrics?`: If Node. Evaluates `={{ $json.metricsSource === 'sample' }}`.
  - `Create labeled sample metrics for testing`: Code Node. Generates mock KPI metrics with labeled test identifiers.
  - `Require a live reporting URL before fetching`: Code Node. Validates that `metricsUrl` is a valid HTTPS endpoint when not in sample mode.
  - `Fetch the reporting source snapshot`: HTTP Request Node. Fetches external JSON reporting data with a 20-second timeout and error handling.
  - `Choose the configured metrics input`: Code Node. Normalizes incoming payload streams.

#### Block 1.3: Planning, Recovery & Validation
- **Overview:** Integrates Google Sheets logs to check for prior execution states, computes variances, flags target misses, and prioritizes metrics.
- **Nodes Involved:** `Read KPI Reports recovery log`, `Calculate KPI changes target misses and owners`, `Eligible source content or saved render exists?`, `Explain skipped content or recovery needs`, `Resume the accepted render job?`, `Restore the saved result and handoff context`
- **Node Details:**
  - `Read KPI Reports recovery log`: Google Sheets Node. Reads the `KPI Reports` tracking sheet to check execution statuses (`rendering`, `completed`, `rate_limited`).
  - `Calculate KPI changes target misses and owners`: Code Node. Computes percentage deltas, checks `lowerIsBetter` flags, identifies target misses, sorts exceptions, and determines if recovery is needed.
  - `Eligible source content or saved render exists?`: If Node. Evaluates `={{ $json.ready === true }}`.
  - `Explain skipped content or recovery needs`: Code Node. Passes through execution state when work is skipped.
  - `Resume the accepted render job?`: If Node. Evaluates `={{ Boolean($json.jobId) }}`.
  - `Restore the saved result and handoff context`: Code Node. Restores cached execution contexts for active jobs.

#### Block 1.4: Asset Enhancement (Music & Voiceover)
- **Overview:** Optionally inspects background music files and generates synchronized AI narration via ElevenLabs.
- **Nodes Involved:** `A background music URL is configured?`, `Check the optional music file size and availability`, `Keep the source plan and usable music only`, `Prepare the selected source for design`, `Create optional narration from checked metrics?`, `Write factual KPI narration from the checked values`, `Generate voiceover`, `Voice + timings`, `Upload voiceover`, `Restore the checked KPI plan with its narration`
- **Node Details:**
  - `A background music URL is configured?`: If Node. Checks if `musicUrl` is provided in settings.
  - `Check the optional music file size and availability`: HTTP Request Node (HEAD method). Probes music file size and availability.
  - `Keep the source plan and usable music only`: Code Node. Validates file size constraints against `maxMusicBytes`.
  - `Prepare the selected source for design`: Code Node. Normalizes data structures.
  - `Create optional narration from checked metrics?`: If Node. Evaluates `narrate === true`.
  - `Write factual KPI narration from the checked values`: Code Node. Generates the script string from metric values and baselines.
  - `Generate voiceover`: HTTP Request Node. Calls ElevenLabs API (`/v1/text-to-speech/{voiceId}/with-timestamps`) using HTTP Header Auth (`xi-api-key`).
  - `Voice + timings`: Code Node. Parses audio base64 payload and word-level character alignment markers into binary audio data.
  - `Upload voiceover`: HTTP Request Node (`POST /api/uploads`). Uploads binary audio via multipart form data using Zvid API credentials because the standard n8n Zvid node lacks multi-part upload capability.
  - `Restore the checked KPI plan with its narration`: Code Node. Merges the uploaded voiceover URL, text script, and timing data into the pipeline.

#### Block 1.5: Video Design, Validation & Budget Control
- **Overview:** Assembles the project timeline schema, queries Zvid validation endpoints, and enforces credit limits.
- **Nodes Involved:** `Build the exception-focused KPI briefing`, `Validate the video and request its credit quote`, `Enforce the per-video credit ceiling`, `Return a validation-only preview?`, `Return the source plan design quote and handoff`
- **Node Details:**
  - `Build the exception-focused KPI briefing`: Code Node. Generates the full 1920x1080 scene layout structure, text blocks, SVGs, and audio configuration arrays.
  - `Validate the video and request its credit quote`: Zvid Node (`resource: render`, `operation: validate`). Validates project JSON against Zvid servers.
  - `Enforce the per-video credit ceiling`: Code Node. Verifies that `creditsRequired` is numeric and does not exceed `maxCreditsPerVideo`. Constructs recovery checkpoints.
  - `Return a validation-only preview?`: If Node. Evaluates `dryRun === true`.
  - `Return the source plan design quote and handoff`: Code Node. Outputs preview data without incurring render costs or making log modifications.

#### Block 1.6: Job Submission, Polling & Tracking
- **Overview:** Interacts with Google Sheets to reserve render keys, submits video jobs to Zvid, handles rate-limit retries, and polls render status.
- **Nodes Involved:** `Reserve this render key before spending credits`, `Prepare a bounded render submission attempt`, `Submit this approved video to Zvid`, `Distinguish a capacity rejection from an uncertain submission`, `Record the rejected or uncertain submission`, `Restore the submission rejection and retry timing`, `Retry this explicit capacity rejection?`, `Wait before retrying available render capacity`, `Report why the render submission stopped`, `Capture the accepted render job ID`, `Persist the accepted job for recovery`, `Restore the accepted render context`, `Track the saved job and its source context`, `Wait for this accepted Zvid render`, `Classify completion failure or uncertain status`, `Record the render outcome and completed URL`
- **Node Details:**
  - `Reserve this render key before spending credits`: Google Sheets Node (`appendOrUpdate`). Saves job state as `submitting`.
  - `Prepare a bounded render submission attempt`: Code Node. Tracks submission attempt counts and start timestamps.
  - `Submit this approved video to Zvid`: Zvid Node (`resource: render`, `operation: create`). Submits project JSON asynchronously.
  - `Distinguish a capacity rejection from an uncertain submission`: Code Node. Identifies HTTP 429 rate limits vs. unknown network errors.
  - `Record the rejected or uncertain submission`: Google Sheets Node. Updates log rows with failure states (`rate_limited` or `needs_review`).
  - `Retry this explicit capacity rejection?`: If Node. Evaluates `retry === true`.
  - `Wait before retrying available render capacity`: Wait Node. Pauses execution based on server-provided backoff seconds.
  - `Report why the render submission stopped`: Code Node. Throws exceptions on unrecoverable submission failures.
  - `Capture the accepted render job ID`: Code Node. Extracts and validates job IDs returned by Zvid.
  - `Persist the accepted job for recovery`: Google Sheets Node. Updates tracking rows with status `rendering` and active `JobId`.
  - `Restore the accepted render context`: Code Node. Prepares context for status polling.
  - `Track the saved job and its source context`: Code Node. Prepares job trackers.
  - `Wait for this accepted Zvid render`: Zvid Node (`resource: render`, `operation: get`). Polls job progress using configured `pollSeconds` and `timeoutMinutes`.
  - `Classify completion failure or uncertain status`: Code Node. Inspects polling output states (`completed`, `failed`, `timed_out`).
  - `Record the render outcome and completed URL`: Google Sheets Node. Updates tracking logs with final URLs and completion timestamps.

#### Block 1.7: Post-Processing, Review & Optional Delivery
- **Overview:** Prepares review packages, extracts final video binaries, and handles optional Slack webhooks.
- **Nodes Involved:** `▶ Watch video`, `Prepare the completed result and review pack`, `Download the source-specific review pack`, `Completed video is ready to watch?`, `Optional delivery is enabled and approved?`, `Record delivery as pending before sending`, `Post the checked KPI briefing to Slack`, `Keep the delivery acknowledgement or error`, `Record the delivery acknowledgement`
- **Node Details:**
  - `▶ Watch video`: HTTP Request Node. Fetches final video binaries (`responseFormat: file`) when completed.
  - `Prepare the completed result and review pack`: Code Node. Validates completed video accessibility via HTTPS URLs.
  - `Download the source-specific review pack`: Code Node. Generates downloadable Base64-encoded CSV review files containing metric data, owners, and action registers.
  - `Completed video is ready to watch?`: If Node. Checks if execution status is `completed` and URLs are valid.
  - `Optional delivery is enabled and approved?`: If Node. Evaluates `sendSlack === true` and validates webhook URLs.
  - `Record delivery as pending before sending`: Google Sheets Node. Sets `DeliveryStatus` to `sending`.
  - `Post the checked KPI briefing to Slack`: HTTP Request Node. Posts formatted briefing text and video links via webhook.
  - `Keep the delivery acknowledgement or error`: Code Node. Validates HTTP 200 `ok` responses from Slack.
  - `Record the delivery acknowledgement`: Google Sheets Node. Updates log rows with final delivery statuses (`accepted` or `needs_review`).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Start the scheduled kpi report video` | `n8n-nodes-base.scheduleTrigger` | Weekly cron trigger | None | `Set reporting period and exception thresholds` | ## 1. Choose source rules and schedule<br><br>Complete the overview setup and source contract. Run one execution at a time. Use the manual trigger before enabling the schedule. |
| `Test the kpi report video workflow` | `n8n-nodes-base.manualTrigger` | Manual test entry point | None | `Set reporting period and exception thresholds` | ## 1. Choose source rules and schedule<br><br>Complete the overview setup and source contract. Run one execution at a time. Use the manual trigger before enabling the schedule. |
| `Set reporting period and exception thresholds` | `n8n-nodes-base.set` | Establishes configuration variables | `Start the scheduled kpi report video`, `Test the kpi report video workflow` | `Use explicitly selected sample metrics?` | ## 1. Choose source rules and schedule<br><br>Complete the overview setup and source contract. Run one execution at a time. Use the manual trigger before enabling the schedule. |
| `Use explicitly selected sample metrics?` | `n8n-nodes-base.if` | Evaluates metrics source mode | `Set reporting period and exception thresholds` | `Create labeled sample metrics for testing`, `Require a live reporting URL before fetching` | ## 2. Choose live or labeled sample metrics<br><br>Live mode requires metricsUrl and a fresh completed-period snapshot. Explicit sample mode bypasses HTTP, labels its video and CSV, and blocks Slack delivery. Derive changes from current and previous values; rank target misses and adverse movements. |
| `Create labeled sample metrics for testing` | `n8n-nodes-base.code` | Generates sample payload data | `Use explicitly selected sample metrics?` | `Choose the configured metrics input` | ## 2. Choose live or labeled sample metrics<br><br>Live mode requires metricsUrl and a fresh completed-period snapshot. Explicit sample mode bypasses HTTP, labels its video and CSV, and blocks Slack delivery. Derive changes from current and previous values; rank target misses and adverse movements. |
| `Require a live reporting URL before fetching` | `n8n-nodes-base.code` | Validates live reporting URL | `Use explicitly selected sample metrics?` | `Fetch the reporting source snapshot` | ## 2. Choose live or labeled sample metrics<br><br>Live mode requires metricsUrl and a fresh completed-period snapshot. Explicit sample mode bypasses HTTP, labels its video and CSV, and blocks Slack delivery. Derive changes from current and previous values; rank target misses and adverse movements. |
| `Fetch the reporting source snapshot` | `n8n-nodes-base.httpRequest` | Fetches live reporting JSON | `Require a live reporting URL before fetching` | `Choose the configured metrics input` | ## 2. Choose live or labeled sample metrics<br><br>Live mode requires metricsUrl and a fresh completed-period snapshot. Explicit sample mode bypasses HTTP, labels its video and CSV, and blocks Slack delivery. Derive changes from current and previous values; rank target misses and adverse movements. |
| `Choose the configured metrics input` | `n8n-nodes-base.code` | Normalizes metrics inputs | `Create labeled sample metrics for testing`, `Fetch the reporting source snapshot` | `Read KPI Reports recovery log` | ## 2. Choose live or labeled sample metrics<br><br>Live mode requires metricsUrl and a fresh completed-period snapshot. Explicit sample mode bypasses HTTP, labels its video and CSV, and blocks Slack delivery. Derive changes from current and previous values; rank target misses and adverse movements. |
| `Read KPI Reports recovery log` | `n8n-nodes-base.googleSheets` | Reads execution history log | `Choose the configured metrics input` | `Calculate KPI changes target misses and owners` | ## 3. Recover accepted work before a new render<br><br>Saved jobs follow the upper route using their original review context. New work continues through optional media and the domain design. |
| `Calculate KPI changes target misses and owners` | `n8n-nodes-base.code` | Computes metric deltas and priorities | `Read KPI Reports recovery log` | `Eligible source content or saved render exists?` | ## 3. Recover accepted work before a new render<br><br>Saved jobs follow the upper route using their original review context. New work continues through optional media and the domain design. |
| `Eligible source content or saved render exists?` | `n8n-nodes-base.if` | Validates rendering eligibility | `Calculate KPI changes target misses and owners` | `Resume the accepted render job?`, `Explain skipped content or recovery needs` | ## 3. Recover accepted work before a new render<br><br>Saved jobs follow the upper route using their original review context. New work continues through optional media and the domain design. |
| `Explain skipped content or recovery needs` | `n8n-nodes-base.code` | Handles skipped work states | `Eligible source content or saved render exists?` | None | ## 3. Recover accepted work before a new render<br><br>Saved jobs follow the upper route using their original review context. New work continues through optional media and the domain design. |
| `Resume the accepted render job?` | `n8n-nodes-base.if` | Checks for active job IDs | `Eligible source content or saved render exists?` | `Restore the saved result and handoff context`, `A background music URL is configured?` | ## 3. Recover accepted work before a new render<br><br>Saved jobs follow the upper route using their original review context. New work continues through optional media and the domain design. |
| `Restore the saved result and handoff context` | `n8n-nodes-base.code` | Restores saved execution contexts | `Resume the accepted render job?` | `Track the saved job and its source context` | ## 3. Recover accepted work before a new render<br><br>Saved jobs follow the upper route using their original review context. New work continues through optional media and the domain design. |
| `A background music URL is configured?` | `n8n-nodes-base.if` | Checks music configuration | `Resume the accepted render job?` | `Check the optional music file size and availability`, `Prepare the selected source for design` | ## 4. Check optional background music<br><br>A failed or oversized music file is skipped. Empty musicUrl continues without an external request. This check preserves the selected source plan. |
| `Check the optional music file size and availability` | `n8n-nodes-base.httpRequest` | Probes background music file | `A background music URL is configured?` | `Keep the source plan and usable music only` | ## 4. Check optional background music<br><br>A failed or oversized music file is skipped. Empty musicUrl continues without an external request. This check preserves the selected source plan. |
| `Keep the source plan and usable music only` | `n8n-nodes-base.code` | Validates music file size constraints | `Check the optional music file size and availability` | `Prepare the selected source for design` | ## 4. Check optional background music<br><br>A failed or oversized music file is skipped. Empty musicUrl continues without an external request. This check preserves the selected source plan. |
| `Prepare the selected source for design` | `n8n-nodes-base.code` | Prepares dataset for briefing generation | `A background music URL is configured?`, `Keep the source plan and usable music only` | `Create optional narration from checked metrics?` | ## 5. Optionally narrate the checked KPI facts<br><br>narrate is off by default. ElevenLabs voice generation can charge even in previews. Configure its Header Auth credential. Multipart upload uses HTTP because the published Zvid node lacks uploads. |
| `Create optional narration from checked metrics?` | `n8n-nodes-base.if` | Checks narration flag | `Prepare the selected source for design` | `Write factual KPI narration from the checked values`, `Build the exception-focused KPI briefing` | ## 5. Optionally narrate the checked KPI facts<br><br>narrate is off by default. ElevenLabs voice generation can charge even in previews. Configure its Header Auth credential. Multipart upload uses HTTP because the published Zvid node lacks uploads. |
| `Write factual KPI narration from the checked values` | `n8n-nodes-base.code` | Generates narration script text | `Create optional narration from checked metrics?` | `Generate voiceover` | ## 5. Optionally narrate the checked KPI facts<br><br>narrate is off by default. ElevenLabs voice generation can charge even in previews. Configure its Header Auth credential. Multipart upload uses HTTP because the published Zvid node lacks uploads. |
| `Generate voiceover` | `n8n-nodes-base.httpRequest` | Calls ElevenLabs API | `Write factual KPI narration from the checked values` | `Voice + timings` | ## 5. Optionally narrate the checked KPI facts<br><br>narrate is off by default. ElevenLabs voice generation can charge even in previews. Configure its Header Auth credential. Multipart upload uses HTTP because the published Zvid node lacks uploads. |
| `Voice + timings` | `n8n-nodes-base.code` | Parses word alignments and audio binary | `Generate voiceover` | `Upload voiceover` | ## 5. Optionally narrate the checked KPI facts<br><br>narrate is off by default. ElevenLabs voice generation can charge even in previews. Configure its Header Auth credential. Multipart upload uses HTTP because the published Zvid node lacks uploads. |
| `Upload voiceover` | `n8n-nodes-base.httpRequest` | Uploads audio via HTTP multipart | `Voice + timings` | `Restore the checked KPI plan with its narration` | ## 5. Optionally narrate the checked KPI facts<br><br>narrate is off by default. ElevenLabs voice generation can charge even in previews. Configure its Header Auth credential. Multipart upload uses HTTP because the published Zvid node lacks uploads. |
| `Restore the checked KPI plan with its narration` | `n8n-nodes-base.code` | Merges narration data into plan | `Upload voiceover` | `Build the exception-focused KPI briefing` | ## 5. Optionally narrate the checked KPI facts<br><br>narrate is off by default. ElevenLabs voice generation can charge even in previews. Configure its Header Auth credential. Multipart upload uses HTTP because the published Zvid node lacks uploads. |
| `Build the exception-focused KPI briefing` | `n8n-nodes-base.code` | Compiles video project JSON | `Create optional narration from checked metrics?`, `Restore the checked KPI plan with its narration` | `Validate the video and request its credit quote` | ## 6. Build and validate the design and credit ceiling<br><br>dryRun=true returns the source plan, design JSON and quote. It performs no log writes, editor draft creation, render, or delivery. Optional voice synthesis may charge before validation. |
| `Validate the video and request its credit quote` | `@zvid/n8n-nodes-zvid.zvid` | Validates project schema with Zvid | `Build the exception-focused KPI briefing` | `Enforce the per-video credit ceiling` | ## 6. Build and validate the design and credit ceiling<br><br>dryRun=true returns the source plan, design JSON and quote. It performs no log writes, editor draft creation, render, or delivery. Optional voice synthesis may charge before validation. |
| `Enforce the per-video credit ceiling` | `n8n-nodes-base.code` | Checks credit limits and builds checkpoint | `Validate the video and request its credit quote` | `Return a validation-only preview?` | ## 6. Build and validate the design and credit ceiling<br><br>dryRun=true returns the source plan, design JSON and quote. It performs no log writes, editor draft creation, render, or delivery. Optional voice synthesis may charge before validation. |
| `Return a validation-only preview?` | `n8n-nodes-base.if` | Evaluates dry-run flag | `Enforce the per-video credit ceiling` | `Return the source plan design quote and handoff`, `Reserve this render key before spending credits` | ## 7. Reserve the key and preserve the accepted job<br><br>Save the key and accepted job ID. Retry explicit capacity rejections up to three attempts, respecting the server delay and a 30-second minimum. Uncertain submissions require inspection. Sheets is not an atomic lock. |
| `Return the source plan design quote and handoff` | `n8n-nodes-base.code` | Outputs dry-run inspection results | `Return a validation-only preview?` | None | ## 7. Reserve the key and preserve the accepted job<br><br>Save the key and accepted job ID. Retry explicit capacity rejections up to three attempts, respecting the server delay and a 30-second minimum. Uncertain submissions require inspection. Sheets is not an atomic lock. |
| `Reserve this render key before spending credits` | `n8n-nodes-base.googleSheets` | Appends submitting log row | `Return a validation-only preview?` | `Prepare a bounded render submission attempt` | ## 7. Reserve the key and preserve the accepted job<br><br>Save the key and accepted job ID. Retry explicit capacity rejections up to three attempts, respecting the server delay and a 30-second minimum. Uncertain submissions require inspection. Sheets is not an atomic lock. |
| `Prepare a bounded render submission attempt` | `n8n-nodes-base.code` | Initializes submission attempt counters | `Reserve this render key before spending credits`, `Wait before retrying available render capacity` | `Submit this approved video to Zvid` | ## 7. Reserve the key and preserve the accepted job<br><br>Save the key and accepted job ID. Retry explicit capacity rejections up to three attempts, respecting the server delay and a 30-second minimum. Uncertain submissions require inspection. Sheets is not an atomic lock. |
| `Submit this approved video to Zvid` | `@zvid/n8n-nodes-zvid.zvid` | Submits render job to Zvid API | `Prepare a bounded render submission attempt` | `Capture the accepted render job ID`, `Distinguish a capacity rejection from an uncertain submission` | ## 7. Reserve the key and preserve the accepted job<br><br>Save the key and accepted job ID. Retry explicit capacity rejections up to three attempts, respecting the server delay and a 30-second minimum. Uncertain submissions require inspection. Sheets is not an atomic lock. |
| `Distinguish a capacity rejection from an uncertain submission` | `n8n-nodes-base.code` | Categorizes submission errors | `Submit this approved video to Zvid` | `Record the rejected or uncertain submission` | ## 7. Reserve the key and preserve the accepted job<br><br>Save the key and accepted job ID. Retry explicit capacity rejections up to three attempts, respecting the server delay and a 30-second minimum. Uncertain submissions require inspection. Sheets is not an atomic lock. |
| `Record the rejected or uncertain submission` | `n8n-nodes-base.googleSheets` | Updates log with rejection status | `Distinguish a capacity rejection from an uncertain submission` | `Restore the submission rejection and retry timing` | ## 7. Reserve the key and preserve the accepted job<br><br>Save the key and accepted job ID. Retry explicit capacity rejections up to three attempts, respecting the server delay and a 30-second minimum. Uncertain submissions require inspection. Sheets is not an atomic lock. |
| `Restore the submission rejection and retry timing` | `n8n-nodes-base.code` | Restores rejection context | `Record the rejected or uncertain submission` | `Retry this explicit capacity rejection?` | ## 7. Reserve the key and preserve the accepted job<br><br>Save the key and accepted job ID. Retry explicit capacity rejections up to three attempts, respecting the server delay and a 30-second minimum. Uncertain submissions require inspection. Sheets is not an atomic lock. |
| `Retry this explicit capacity rejection?` | `n8n-nodes-base.if` | Evaluates retry eligibility | `Restore the submission rejection and retry timing` | `Wait before retrying available render capacity`, `Report why the render submission stopped` | ## 7. Reserve the key and preserve the accepted job<br><br>Save the key and accepted job ID. Retry explicit capacity rejections up to three attempts, respecting the server delay and a 30-second minimum. Uncertain submissions require inspection. Sheets is not an atomic lock. |
| `Wait before retrying available render capacity` | `n8n-nodes-base.wait` | Backs off before re-submitting | `Retry this explicit capacity rejection?` | `Prepare a bounded render submission attempt` | ## 7. Reserve the key and preserve the accepted job<br><br>Save the key and accepted job ID. Retry explicit capacity rejections up to three attempts, respecting the server delay and a 30-second minimum. Uncertain submissions require inspection. Sheets is not an atomic lock. |
| `Report why the render submission stopped` | `n8n-nodes-base.code` | Throws terminal submission errors | `Retry this explicit capacity rejection?` | None | ## 7. Reserve the key and preserve the accepted job<br><br>Save the key and accepted job ID. Retry explicit capacity rejections up to three attempts, respecting the server delay and a 30-second minimum. Uncertain submissions require inspection. Sheets is not an atomic lock. |
| `Capture the accepted render job ID` | `n8n-nodes-base.code` | Extracts accepted job identifiers | `Submit this approved video to Zvid` | `Persist the accepted job for recovery` | ## 7. Reserve the key and preserve the accepted job<br><br>Save the key and accepted job ID. Retry explicit capacity rejections up to three attempts, respecting the server delay and a 30-second minimum. Uncertain submissions require inspection. Sheets is not an atomic lock. |
| `Persist the accepted job for recovery` | `n8n-nodes-base.googleSheets` | Updates log row with `JobId` | `Capture the accepted render job ID` | `Restore the accepted render context` | ## 7. Reserve the key and preserve the accepted job<br><br>Save the key and accepted job ID. Retry explicit capacity rejections up to three attempts, respecting the server delay and a 30-second minimum. Uncertain submissions require inspection. Sheets is not an atomic lock. |
| `Restore the accepted render context` | `n8n-nodes-base.code` | Restores accepted render context | `Persist the accepted job for recovery` | `Track the saved job and its source context` | ## 8. Wait for the accepted render and record its result<br><br>Poll the saved job within the deadline. Keep failed, timed-out and uncertain job IDs for inspection. Completed keys are skipped on later runs. |
| `Track the saved job and its source context` | `n8n-nodes-base.code` | Tracks job polling timers | `Restore the accepted render context`, `Resume the accepted render job?` | `Wait for this accepted Zvid render` | ## 8. Wait for the accepted render and record its result<br><br>Poll the saved job within the deadline. Keep failed, timed-out and uncertain job IDs for inspection. Completed keys are skipped on later runs. |
| `Wait for this accepted Zvid render` | `@zvid/n8n-nodes-zvid.zvid` | Polls Zvid render job status | `Track the saved job and its source context` | `Classify completion failure or uncertain status` | ## 8. Wait for the accepted render and record its result<br><br>Poll the saved job within the deadline. Keep failed, timed-out and uncertain job IDs for inspection. Completed keys are skipped on later runs. |
| `Classify completion failure or uncertain status` | `n8n-nodes-base.code` | Classifies polling results | `Wait for this accepted Zvid render` | `Record the render outcome and completed URL` | ## 8. Wait for the accepted render and record its result<br><br>Poll the saved job within the deadline. Keep failed, timed-out and uncertain job IDs for inspection. Completed keys are skipped on later runs. |
| `Record the render outcome and completed URL` | `n8n-nodes-base.googleSheets` | Records completed video outcomes | `Classify completion failure or uncertain status` | `Prepare the completed result and review pack` | ## 8. Wait for the accepted render and record its result<br><br>Poll the saved job within the deadline. Keep failed, timed-out and uncertain job IDs for inspection. Completed keys are skipped on later runs. |
| `Prepare the completed result and review pack` | `n8n-nodes-base.code` | Validates final video URL readiness | `Record the render outcome and completed URL` | `Completed video is ready to watch?`, `Download the source-specific review pack`, `Optional delivery is enabled and approved?` | 9. Review the video and source-specific handoff<br><br>Completed renders reach Watch video → Binary → data → View. A failed render stops with its actual reason after saving the tracking row. Download the review CSV and check source facts before sharing. Optional delivery is off by default. |
| `Download the source-specific review pack` | `n8n-nodes-base.code` | Generates CSV review download | `Prepare the completed result and review pack` | None | 9. Review the video and source-specific handoff<br><br>Completed renders Reach Watch video → Binary → data → View. A failed render stops with its actual reason after saving the tracking row. Download the review CSV and check source facts before sharing. Optional delivery is off by default. |
| `▶ Watch video` | `n8n-nodes-base.httpRequest` | Downloads completed video binary | `Completed video is ready to watch?` | None | 9. Review the video and source-specific handoff<br><br>Completed renders reach Watch video → Binary → data → View. A failed render stops with its actual reason after saving the tracking row. Download the review CSV and check source facts before sharing. Optional delivery is off by default. |
| `Completed video is ready to watch?` | `n8n-nodes-base.if` | Verifies completed video download state | `Prepare the completed result and review pack` | `▶ Watch video` | 9. Review the video and source-specific handoff<br><br>Completed renders reach Watch video → Binary → data → View. A failed render stops with its actual reason after saving the tracking row. Download the review CSV and check source facts before sharing. Optional delivery is off by default. |
| `Optional delivery is enabled and approved?` | `n8n-nodes-base.if` | Evaluates Slack delivery flags | `Prepare the completed result and review pack` | `Record delivery as pending before sending` | ## 10. Deliver only when explicitly enabled<br><br>Optional delivery is off by default. Enable sendSlack only after configuring its webhook. Record sending before delivery and retain the acknowledgement. Inspect uncertain sends manually; completed runs never resend automatically. |
| `Record delivery as pending before sending` | `n8n-nodes-base.googleSheets` | Sets `DeliveryStatus` to `sending` | `Optional delivery is enabled and approved?` | `Post the checked KPI briefing to Slack` | ## 10. Deliver only when explicitly enabled<br><br>Optional delivery is off by default. Enable sendSlack only after configuring its webhook. Record sending before delivery and retain the acknowledgement. Inspect uncertain sends manually; completed runs never resend automatically. |
| `Post the checked KPI briefing to Slack` | `n8n-nodes-base.httpRequest` | Sends Slack webhook payload | `Record delivery as pending before sending` | `Keep the delivery acknowledgement or error` | ## 10. Deliver only when explicitly enabled<br><br>Optional delivery is off by default. Enable sendSlack only after configuring its webhook. Record sending before delivery and retain the acknowledgement. Inspect uncertain sends manually; completed runs never resend automatically. |
| `Keep the delivery acknowledgement or error` | `n8n-nodes-base.code` | Parses Slack delivery responses | `Post the checked KPI briefing to Slack` | `Record the delivery acknowledgement` | ## 10. Deliver only when explicitly enabled<br><br>Optional delivery is off by default. Enable sendSlack only after configuring its webhook. Record sending before delivery and retain the acknowledgement. Inspect uncertain sends manually; completed runs never resend automatically. |
| `Record the delivery acknowledgement` | `n8n-nodes-base.googleSheets` | Records final delivery status | `Keep the delivery acknowledgement or error` | None | ## 10. Deliver only when explicitly enabled<br><br>Optional delivery is off by default. Enable sendSlack only after configuring its webhook. Record sending before delivery and retain the acknowledgement. Inspect uncertain sends manually; completed runs never resend automatically. |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: Install Community Nodes & Credentials
1. Navigate to **Settings → Community nodes** in your self-hosted n8n instance and install **`@zvid/n8n-nodes-zvid`**.
2. Create a **Zvid API** credential:
   - Base URL: `https://api.zvid.io`
   - API Key: Obtain from [Zvid API Keys](https://app.zvid.io/api-keys).
3. Connect **Google Sheets** credentials and create a spreadsheet tab named `KPI Reports` with the following column headers: `RenderKey`, `Status`, `JobId`, `CreditsQuoted`, `PlanJson`, `VideoUrl`, `CompletedAt`, `DeliveryStatus`.
4. (Optional) Create an **HTTP Header Auth** credential for ElevenLabs (`xi-api-key`).

#### Step 2: Build the Core Trigger & Configuration
1. Create a **Schedule Trigger** (`Start the scheduled kpi report video`) set to run weekly on Mondays at 09:00 UTC.
2. Create a **Manual Trigger** (`Test the kpi report video workflow`) for on-demand testing.
3. Add a **Set** node (`Set reporting period and exception thresholds`) configured in raw JSON mode containing your branding, API URLs, thresholds (`alertPercent: 10`, `maxCreditsPerVideo: 65`), and feature flags (`narrate: false`, `sendSlack: false`, `dryRun: false`).

#### Step 3: Configure Data Ingestion & Recovery
1. Add an **If** node (`Use explicitly selected sample metrics?`) evaluating `={{ $json.metricsSource === 'sample' }}`.
2. Add a **Code** node (`Create labeled sample metrics for testing`) to generate mock KPI structures when testing.
3. Add a **Code** node (`Require a live reporting URL before fetching`) and an **HTTP Request** node (`Fetch the reporting source snapshot`) to retrieve live data.
4. Add a **Code** node (`Choose the configured metrics input`) to normalize input paths.
5. Add a **Google Sheets** node (`Read KPI Reports recovery log`) configured to read the `KPI Reports` tab.
6. Add a **Code** node (`Calculate KPI changes target misses and owners`) to compute variances, target misses, and sort exceptions.
7. Add an **If** node (`Eligible source content or saved render exists?`) checking `={{ $json.ready === true }}`.
8. Add a **Code** node (`Explain skipped content or recovery needs`) for negative path exits.
9. Add an **If** node (`Resume the accepted render job?`) checking `={{ Boolean($json.jobId) }}`.
10. Add a **Code** node (`Restore the saved result and handoff context`) to restore saved execution metadata.

#### Step 4: Configure Audio Assets (Music & Narration)
1. Add an **If** node (`A background music URL is configured?`) checking `={{ Boolean($json.musicUrl) }}`.
2. Add an **HTTP Request** node (`Check the optional music file size and availability`, HEAD method) and a **Code** node (`Keep the source plan and usable music only`) to validate music headers.
3. Add a **Code** node (`Prepare the selected source for design`).
4. Add an **If** node (`Create optional narration from checked metrics?`) checking `={{ $json.narrate === true }}`.
5. Add a **Code** node (`Write factual KPI narration from the checked values`) to generate speech scripts.
6. Add an **HTTP Request** node (`Generate voiceover`) connecting to ElevenLabs `/v1/text-to-speech/{voiceId}/with-timestamps`.
7. Add a **Code** node (`Voice + timings`) to parse word alignments and binary output.
8. Add an **HTTP Request** node (`Upload voiceover`) using `POST /api/uploads` with multipart form-data via Zvid API credentials.
9. Add a **Code** node (`Restore the checked KPI plan with its narration`).

#### Step 5: Design, Validation & Credit Checks
1. Add a **Code** node (`Build the exception-focused KPI briefing`) to compile the 1920x1080 project JSON schema.
2. Add a **Zvid** node (`Validate the video and request its credit quote`, resource: `render`, operation: `validate`).
3. Add a **Code** node (`Enforce the per-video credit ceiling`) to verify credits required against `maxCreditsPerVideo`.
4. Add an **If** node (`Return a validation-only preview?`) checking `={{ $json.dryRun === true }}`.
5. Add a **Code** node (`Return the source plan design quote and handoff`) for dry-run exits.

#### Step 6: Reservation, Submission & Polling
1. Add a **Google Sheets** node (`Reserve this render key before spending credits`, operation: `appendOrUpdate`) matching on `RenderKey` with status `submitting`.
2. Add a **Code** node (`Prepare a bounded render submission attempt`).
3. Add a **Zvid** node (`Submit this approved video to Zvid`, resource: `render`, operation: `create`, `waitForCompletion: false`). Enable `Continue node on error` (`onError: continueErrorOutput`).
4. Add a **Code** node (`Distinguish a capacity rejection from an uncertain submission`) to catch HTTP 429 rate limits.
5. Add a **Google Sheets** node (`Record the rejected or uncertain submission`, operation: `update`).
6. Add a **Code** node (`Restore the submission rejection and retry timing`) and an **If** node (`Retry this explicit capacity rejection?`) checking `={{ $json.retry === true }}`.
7. Add a **Wait** node (`Wait before retrying available render capacity`) resuming on time intervals.
8. Add a **Code** node (`Report why the render submission stopped`) to throw terminal errors on unrecoverable failures.
9. Add a **Code** node (`Capture the accepted render job ID`) and a **Google Sheets** node (`Persist the accepted job for recovery`, operation: `update`) setting status to `rendering`.
10. Add **Code** nodes (`Restore the accepted render context`, `Track the saved job and its source context`) and a **Zvid** node (`Wait for this accepted Zvid render`, resource: `render`, operation: `get`, `waitForCompletion: true`).
11. Add a **Code** node (`Classify completion failure or uncertain status`) and a **Google Sheets** node (`Record the render outcome and completed URL`, operation: `update`).

#### Step 7: Post-Processing, Review & Delivery
1. Add a **Code** node (`Prepare the completed result and review pack`) and a **Code** node (`Download the source-specific review pack`) to generate Base64 CSV payloads.
2. Add an **If** node (`Completed video is ready to watch?`) evaluating completion status.
3. Add an **HTTP Request** node (`▶ Watch video`, response format: File) to retrieve final MP4 binaries.
4. Add an **If** node (`Optional delivery is enabled and approved?`) evaluating `sendSlack === true`.
5. Add a **Google Sheets** node (`Record delivery as pending before sending`, operation: `update`) setting `DeliveryStatus` to `sending`.
6. Add an **HTTP Request** node (`Post the checked KPI briefing to Slack`) pointing to your Slack webhook URL.
7. Add a **Code** node (`Keep the delivery acknowledgement or error`) and a **Google Sheets** node (`Record the delivery acknowledgement`, operation: `update`) to finalize `DeliveryStatus` (`accepted` or `needs_review`).

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Background Guide & Documentation | [GitHub Repository Guide](https://github.com/Zvid-io/zvid-n8n/blob/master/workflows/kpi-report-video.md) |
| Zvid API Keys & Dashboard | [Zvid App API Keys](https://app.zvid.io/api-keys) |
| Support & Contact | [Zvid Contact Support](https://zvid.io/contact) |
| Zvid Community Node Package | `@zvid/n8n-nodes-zvid` (Installable via n8n Settings → Community nodes) |