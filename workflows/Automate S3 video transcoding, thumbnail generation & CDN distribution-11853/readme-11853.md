Automate S3 video transcoding, thumbnail generation & CDN distribution

https://n8nworkflows.xyz/workflows/automate-s3-video-transcoding--thumbnail-generation---cdn-distribution-11853


# Automate S3 video transcoding, thumbnail generation & CDN distribution

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name:** *Video Processing Pipeline with Thumbnail Generation and CDN Distribution*  
**Stated title:** *Automate S3 video transcoding, thumbnail generation & CDN distribution*

**Purpose:** Automate end-to-end processing of a video stored in Amazon S3: detect upload/manual request, validate video type, analyze with FFprobe, generate thumbnails and a GIF preview, transcode into multiple resolutions, invalidate Cloudflare CDN cache, generate delivery URLs, log metrics to Google Sheets, notify via Slack, and respond to the triggering webhook.

### 1.1 Input Reception & Trigger Normalization
Two webhooks act as entry points (S3 event vs manual). They are merged and normalized into a single job payload (bucket/key/URLs/job_id).

### 1.2 Validation (Video File Gate)
Checks file extension to ensure it’s a supported video format; otherwise returns an error payload.

### 1.3 Media Analysis (FFprobe)
Calls an external FFmpeg/FFprobe API endpoint to fetch metadata, then derives duration, resolution, thumbnail timestamps, and whether transcoding is needed.

### 1.4 Processing Fan-out (Thumbnails, GIF, Transcode)
Runs three HTTP jobs (thumbnails, preview GIF, transcode) in parallel, then aggregates the results.

### 1.5 Distribution & Reporting
Purges Cloudflare cache for generated asset prefixes, builds CDN URLs (called “signed URLs” though no cryptographic signing is performed), logs to Google Sheets, sends Slack notification, merges outputs, and responds to webhook.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & Trigger Normalization

**Overview:** Receives either an S3 event payload or a manual request, then converts both into a unified structure containing S3 bucket/key, derived URLs, and a unique job ID.

**Nodes involved:**
- **S3 Event Webhook1**
- **Manual Process Trigger1**
- **Merge Triggers1**
- **Extract S3 Info1**

#### Node: S3 Event Webhook1
- **Type / role:** Webhook trigger; receives S3 event notifications.
- **Configuration (interpreted):**
  - **POST** `/s3-video-upload`
  - **responseMode:** handled by a later **Respond to Webhook** node.
  - **onError:** `continueRegularOutput` (workflow continues even if this node errors).
- **Inputs/outputs:**
  - Output → **Merge Triggers1** (input 0).
- **Version notes:** Webhook node v2.
- **Edge cases / failures:**
  - If S3 sends a different event schema than expected, later parsing may fail (e.g., missing `Records[0].s3.object.key`).

#### Node: Manual Process Trigger1
- **Type / role:** Webhook trigger for manual processing/testing or external systems.
- **Configuration (interpreted):**
  - **POST** `/process-video`
  - response handled by Respond node.
  - **onError:** `continueRegularOutput`.
- **Inputs/outputs:**
  - Output → **Merge Triggers1** (input 1).
- **Version notes:** Webhook node v2.
- **Edge cases / failures:**
  - Missing required parameters (`bucket`, `key`, or `video_key`) will fall back to defaults in the code node.

#### Node: Merge Triggers1
- **Type / role:** Merge; selects one trigger branch (acts as a unifier).
- **Configuration (interpreted):**
  - **Mode:** “chooseBranch”
  - **Output:** “empty” (n8n merge behavior for unselected branches)
- **Inputs/outputs:**
  - Inputs: S3 webhook, Manual webhook
  - Output → **Extract S3 Info1**
- **Version notes:** Merge node v3.
- **Edge cases / failures:**
  - If both triggers fire nearly simultaneously (rare), “chooseBranch” behavior may not aggregate both; it will effectively proceed with whichever item arrives in that execution path.

#### Node: Extract S3 Info1
- **Type / role:** Code node; normalizes incoming event into canonical job payload.
- **Configuration choices:**
  - Detects payload type:
    - S3 event: `data.Records[0]...`
    - Manual: `data.body.bucket`, `data.body.key` or `data.body.video_key`
    - Fallback: `data.bucket`, `data.key`, defaults to `video-uploads` and `sample-video.mp4`
  - Derives:
    - `job_id` = `job_<timestamp>_<random>`
    - `extension`, `is_video` based on a fixed list
    - `s3_url` (s3://…) and `https_url` (https://bucket.s3.amazonaws.com/encodedKey)
- **Key variables produced (downstream contract):**
  - `job_id`, `bucket`, `key`, `https_url`, `filename`, `extension`, `is_video`, `event_type`, `started_at`
- **Inputs/outputs:**
  - Input: merged webhook payload(s)
  - Output → **Check Is Video1**
- **Version notes:** Code node v2.
- **Edge cases / failures:**
  - If `key` is undefined/null, extension becomes `''` and `is_video=false`.
  - `https_url` assumes the object is publicly accessible or accessible by the FFmpeg service; private buckets will break later API calls unless the FFmpeg service can access S3 privately.
  - URL encoding: uses `encodeURIComponent(key)` but does not preserve path slashes; this can produce an URL that encodes `/` as `%2F`, which is **not** the standard S3 URL form for keys with folders. Many systems accept it, but some won’t. (Common fix: encode each path segment, not the slashes.)

**Sticky note coverage (context):**
- “### Step 1: Video Detection …” applies to this block.

---

### Block 2 — Validation (Video File Gate)

**Overview:** Ensures the input file is a supported video based on extension; routes to processing or produces an error response payload.

**Nodes involved:**
- **Check Is Video1**
- **Invalid File Response1**

#### Node: Check Is Video1
- **Type / role:** IF router.
- **Configuration choices:**
  - Condition: `{{$json.is_video}}` equals `true` (boolean strict validation).
- **Inputs/outputs:**
  - True → **Get Video Metadata (FFprobe)1**
  - False → **Invalid File Response1**
- **Version notes:** IF node v2.2.
- **Edge cases / failures:**
  - Extension-based detection can misclassify: files without extension or uncommon extensions will fail even if valid video.
  - A `.mp4` that is actually not a video will pass and later FFprobe will fail.

#### Node: Invalid File Response1
- **Type / role:** Set node; prepares error JSON for webhook response.
- **Configuration choices:**
  - Sets:
    - `status = "error"`
    - `message = "File is not a supported video format"`
    - `job_id = {{$json.job_id}}`
- **Inputs/outputs:**
  - Output → **Merge All Paths1** (input 1)
- **Version notes:** Set node v3.4.
- **Edge cases / failures:**
  - If `job_id` missing upstream, response will include empty/undefined value.

---

### Block 3 — Media Analysis (FFprobe)

**Overview:** Calls an external FFprobe API to retrieve stream/format metadata, then calculates useful derived fields (duration, resolution, thumbnail timestamps, transcode decision).

**Nodes involved:**
- **Get Video Metadata (FFprobe)1**
- **Parse Video Metadata1**

#### Node: Get Video Metadata (FFprobe)1
- **Type / role:** HTTP Request to FFmpeg-service probe endpoint.
- **Configuration choices:**
  - `POST https://api.ffmpeg-service.com/v1/probe`
  - JSON body includes:
    - `input`: `{{$json.https_url}}`
    - options: `show_format`, `show_streams`
  - **Authentication:** Generic credential type → **HTTP Header Auth** (expects header-based token).
  - **neverError:** enabled (HTTP errors won’t stop execution; response still returned).
- **Inputs/outputs:**
  - Input: validated job payload
  - Output → **Parse Video Metadata1**
- **Version notes:** HTTP Request v4.2.
- **Edge cases / failures:**
  - With `neverError`, downstream code must detect failures (e.g., 401/500) or missing `streams/format`; otherwise it will compute defaults and proceed incorrectly.
  - If FFmpeg service cannot access the S3 URL (private ACL, region restrictions, VPC-only), probe fails.
  - Timeouts for large files or slow network.

#### Node: Parse Video Metadata1
- **Type / role:** Code node; extracts and enriches metadata.
- **Configuration choices (logic):**
  - Reads `probeResult = data.result || data`
  - Finds first `video` and `audio` stream by `codec_type`
  - Computes:
    - `duration_seconds` from `format.duration` or stream duration
    - `duration_formatted` HH:MM:SS
    - `thumbnail_times` at 10%, 30%, 50%, 70%, 90% of duration (floored)
    - `width/height` defaults to 1920x1080 if missing
    - `needs_transcode` if width > 1920 or codec not in `['h264','hevc']`
    - Adds other fields: bitrate, frame_rate, file size, aspect ratio, etc.
  - Sets `target_resolutions = ['1080p','720p','480p']`
- **Inputs/outputs:**
  - Output fan-out → **Generate Thumbnails (FFmpeg)1**, **Generate Preview GIF1**, **Transcode Video1**
- **Version notes:** Code node v2.
- **Edge cases / failures:**
  - If duration is 0/NaN, thumbnail times become all 0 → repeated thumbnail extraction at 0s.
  - Assumes `probeResult.streams` exists; if not, it falls back to `{}` and defaults (may mask upstream probe failure).
  - `needs_transcode` is computed but not actually used to skip transcoding in this workflow (transcode always runs).

**Sticky note coverage (context):**
- “### Step 2: Media Analysis …” applies to this block.

---

### Block 4 — Processing Fan-out (Thumbnails, GIF, Transcode) + Aggregation

**Overview:** Runs three external FFmpeg jobs in parallel and aggregates their results into a single payload for downstream distribution steps.

**Nodes involved:**
- **Generate Thumbnails (FFmpeg)1**
- **Generate Preview GIF1**
- **Transcode Video1**
- **Aggregate Processing Results1**

#### Node: Generate Thumbnails (FFmpeg)1
- **Type / role:** HTTP Request; instructs FFmpeg service to render thumbnails and upload to S3.
- **Configuration choices:**
  - `POST https://api.ffmpeg-service.com/v1/thumbnails`
  - Output:
    - `output_bucket = {{$json.bucket}}`
    - `output_prefix = thumbnails/{{$json.job_id}}/`
  - Times: `{{ JSON.stringify($json.thumbnail_times) }}`
  - Sizes: 1280x720 (_large), 640x360 (_medium), 320x180 (_small)
  - Format: jpg, quality 85
  - Auth: HTTP Header Auth
  - `neverError: true`
- **Inputs/outputs:**
  - Output → **Aggregate Processing Results1**
- **Version notes:** HTTP Request v4.2.
- **Edge cases / failures:**
  - If `thumbnail_times` contains out-of-range timestamps, FFmpeg may error or clamp.
  - Bucket write permissions must allow the FFmpeg service to PUT to the target prefix.

#### Node: Generate Preview GIF1
- **Type / role:** HTTP Request; creates GIF preview and uploads to S3.
- **Configuration choices:**
  - `POST https://api.ffmpeg-service.com/v1/gif`
  - `output_key = previews/{{$json.job_id}}/preview.gif`
  - `start_time = floor(duration_seconds * 0.2)`, duration=5s, fps=10, width=480
  - Auth: HTTP Header Auth
  - `neverError: true`
- **Inputs/outputs:**
  - Output → **Aggregate Processing Results1**
- **Version notes:** HTTP Request v4.2.
- **Edge cases / failures:**
  - For very short videos, `start_time` may be near/end beyond duration; FFmpeg service may error.
  - GIF generation can be CPU heavy; timeouts possible.

#### Node: Transcode Video1
- **Type / role:** HTTP Request; transcodes to multiple presets and uploads to S3.
- **Configuration choices:**
  - `POST https://api.ffmpeg-service.com/v1/transcode`
  - Output prefix: `transcoded/{{$json.job_id}}/`
  - Presets:
    - 1080p 1920x1080 5000k
    - 720p 1280x720 2500k
    - 480p 854x480 1000k
  - Codec h264, audio aac, format mp4
  - Auth: HTTP Header Auth
  - `neverError: true`
- **Inputs/outputs:**
  - Output → **Aggregate Processing Results1**
- **Version notes:** HTTP Request v4.2.
- **Edge cases / failures:**
  - Always runs even if `needs_transcode` is false (wasted compute).
  - Source might be smaller than target (upscaling) depending on FFmpeg service behavior (not controlled here).
  - Large uploads may cause long processing times; consider async job polling if the API is non-blocking.

#### Node: Aggregate Processing Results1
- **Type / role:** Aggregate node; merges results from thumbnail/GIF/transcode into one item.
- **Configuration choices:**
  - Mode: `aggregateAllItemData` (collect all incoming item data under `data` array)
- **Inputs/outputs:**
  - Output → **Invalidate CDN Cache1**
- **Version notes:** Aggregate v1.
- **Edge cases / failures:**
  - If upstream HTTP calls return different shapes, aggregation will still bundle them; downstream assumes `data[0].job_id` exists.

**Sticky note coverage (context):**
- “### Step 3: Processing …” applies to this block.

---

### Block 5 — Distribution, Reporting & Response

**Overview:** Purges CDN cache for generated prefixes, constructs CDN URLs for assets, logs metrics to Google Sheets, sends Slack notification, then responds to the original webhook call with the final JSON.

**Nodes involved:**
- **Invalidate CDN Cache1**
- **Generate Signed URLs1**
- **Log Processing Metrics1**
- **Send Slack Notification**
- **Merge Output Paths1**
- **Merge All Paths1**
- **Respond to Webhook1**

#### Node: Invalidate CDN Cache1
- **Type / role:** HTTP Request; calls Cloudflare purge cache endpoint.
- **Configuration choices:**
  - `POST https://api.cloudflare.com/client/v4/zones/YOUR_ZONE_ID/purge_cache`
  - Body uses `prefixes`:
    - `thumbnails/<job_id>/`
    - `previews/<job_id>/`
    - `transcoded/<job_id>/`
  - Job id read from aggregated data: `{{$json.data[0]?.job_id || 'unknown'}}`
  - Auth: HTTP Header Auth
  - `neverError: true`
- **Inputs/outputs:**
  - Output → **Generate Signed URLs1**
- **Version notes:** HTTP Request v4.2.
- **Edge cases / failures:**
  - Placeholder `YOUR_ZONE_ID` must be replaced.
  - Cloudflare purge API may require different payload (`files` vs `prefixes`) depending on plan/features; “prefixes” is not universally supported for all accounts/endpoints.
  - With `neverError`, purge failures won’t stop notifications/logging.

#### Node: Generate Signed URLs1
- **Type / role:** Code node; builds final response object with CDN links.
- **Configuration choices (logic):**
  - Takes `jobData = data.data?.[0] || data` (assumes aggregated structure).
  - Sets `cdnBase = 'https://cdn.example.com'` (placeholder).
  - Creates URLs for:
    - thumbnails: `thumb_0_large/medium/small.jpg`
    - preview gif
    - transcoded: `video_1080p/720p/480p.mp4`
  - Sets `urls_expire_at` to now + 24h, but **does not actually sign URLs**.
- **Inputs/outputs:**
  - Output → **Log Processing Metrics1** and **Send Slack Notification** (in parallel)
- **Version notes:** Code node v2.
- **Edge cases / failures:**
  - Assumes naming conventions (`thumb_0_*`, `video_*.mp4`) that must match what the FFmpeg service actually writes; if different, links will be wrong.
  - “Signed URLs” are only static links; if you require signed access (Cloudflare signed URLs/cookies, S3 presigned URLs), additional signing logic is needed.

#### Node: Log Processing Metrics1
- **Type / role:** Google Sheets append; logs processing summary.
- **Configuration choices:**
  - Operation: **append**
  - Document and sheet are set via resource locator but currently blank in JSON (must be configured).
- **Inputs/outputs:**
  - Output → **Merge Output Paths1** (input 0)
- **Version notes:** Google Sheets v4.5.
- **Edge cases / failures:**
  - Missing OAuth credentials or missing document/sheet selection will fail.
  - If the sheet columns don’t match incoming fields, appended rows may be empty or misaligned.

#### Node: Send Slack Notification
- **Type / role:** Slack message; notifies channel of completion.
- **Configuration choices:**
  - Posts formatted message including job id, filename, resolution, duration.
  - Channel: `#video-processing` (selected by name).
- **Inputs/outputs:**
  - Output → **Merge Output Paths1** (input 1)
- **Version notes:** Slack v2.2.
- **Edge cases / failures:**
  - Slack credentials/token scopes missing (`chat:write` etc.).
  - Channel name resolution can fail if workspace restricts access; using channel ID is more reliable.

#### Node: Merge Output Paths1
- **Type / role:** Merge; joins the parallel “log” and “slack” branches back toward a single response path.
- **Configuration choices:** chooseBranch + output empty.
- **Inputs/outputs:**
  - Output → **Merge All Paths1** (input 0)
- **Version notes:** Merge v3.
- **Edge cases / failures:**
  - Because it’s “chooseBranch”, it doesn’t truly combine results; it simply forwards one branch’s item depending on execution behavior. If you need deterministic combined output, use Merge “combine”/“multiplex” patterns instead.

#### Node: Merge All Paths1
- **Type / role:** Merge; unifies success path with invalid-file error path for final response.
- **Configuration choices:** chooseBranch + output empty.
- **Inputs/outputs:**
  - Input 0: success path
  - Input 1: invalid file path
  - Output → **Respond to Webhook1**
- **Version notes:** Merge v3.
- **Edge cases / failures:**
  - Similar caveat: “chooseBranch” means the returned object might be from Slack/Sheets node output rather than the curated payload unless data flow is carefully controlled. In this workflow, the intent is to respond with `$json` at the final node—ensure the item reaching here is the desired response object.

#### Node: Respond to Webhook1
- **Type / role:** Respond; returns JSON to the triggering webhook call.
- **Configuration choices:**
  - Respond with JSON body: `={{ $json }}`
- **Inputs/outputs:**
  - Terminal node.
- **Version notes:** Respond to Webhook v1.1.
- **Edge cases / failures:**
  - If upstream merges forward an unexpected structure, the response will not match expectations (e.g., Google Sheets API response instead of final URLs).

**Sticky note coverage (context):**
- “### Step 4: Distribution …” applies to this block.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note - Overview1 | Sticky Note | Documentation | — | — | ## Video Processing Pipeline … (overview of end-to-end pipeline and required credentials) |
| Sticky Note - Step  | Sticky Note | Documentation | — | — | ### Step 1: Video Detection … |
| Sticky Note - Step 5 | Sticky Note | Documentation | — | — | ### Step 2: Media Analysis … |
| Sticky Note - Step 6 | Sticky Note | Documentation | — | — | ### Step 3: Processing … |
| Sticky Note - Step 7 | Sticky Note | Documentation | — | — | ### Step 4: Distribution … |
| S3 Event Webhook1 | Webhook | Entry point for S3 upload events | — | Merge Triggers1 | ### Step 1: Video Detection … |
| Manual Process Trigger1 | Webhook | Entry point for manual processing | — | Merge Triggers1 | ### Step 1: Video Detection … |
| Merge Triggers1 | Merge | Unify entry paths (choose one) | S3 Event Webhook1, Manual Process Trigger1 | Extract S3 Info1 | ### Step 1: Video Detection … |
| Extract S3 Info1 | Code | Normalize payload; derive job_id, URLs; detect extension | Merge Triggers1 | Check Is Video1 | ### Step 1: Video Detection … |
| Check Is Video1 | IF | Gate on supported video extension | Extract S3 Info1 | Get Video Metadata (FFprobe)1; Invalid File Response1 | ### Step 1: Video Detection … |
| Invalid File Response1 | Set | Build error response JSON | Check Is Video1 (false) | Merge All Paths1 |  |
| Get Video Metadata (FFprobe)1 | HTTP Request | Call FFprobe API for metadata | Check Is Video1 (true) | Parse Video Metadata1 | ### Step 2: Media Analysis … |
| Parse Video Metadata1 | Code | Enrich/derive metadata and processing parameters | Get Video Metadata (FFprobe)1 | Generate Thumbnails (FFmpeg)1; Generate Preview GIF1; Transcode Video1 | ### Step 2: Media Analysis … |
| Generate Thumbnails (FFmpeg)1 | HTTP Request | Create multi-size thumbnails to S3 | Parse Video Metadata1 | Aggregate Processing Results1 | ### Step 3: Processing … |
| Generate Preview GIF1 | HTTP Request | Create preview GIF to S3 | Parse Video Metadata1 | Aggregate Processing Results1 | ### Step 3: Processing … |
| Transcode Video1 | HTTP Request | Transcode video to 1080p/720p/480p to S3 | Parse Video Metadata1 | Aggregate Processing Results1 | ### Step 3: Processing … |
| Aggregate Processing Results1 | Aggregate | Collect results into single item | Generate Thumbnails (FFmpeg)1; Generate Preview GIF1; Transcode Video1 | Invalidate CDN Cache1 | ### Step 3: Processing … |
| Invalidate CDN Cache1 | HTTP Request | Purge Cloudflare cache for new assets | Aggregate Processing Results1 | Generate Signed URLs1 | ### Step 4: Distribution … |
| Generate Signed URLs1 | Code | Build final output object with CDN URLs | Invalidate CDN Cache1 | Log Processing Metrics1; Send Slack Notification | ### Step 4: Distribution … |
| Log Processing Metrics1 | Google Sheets | Append metrics row | Generate Signed URLs1 | Merge Output Paths1 | ### Step 4: Distribution … |
| Send Slack Notification | Slack | Notify channel of completion | Generate Signed URLs1 | Merge Output Paths1 | ### Step 4: Distribution … |
| Merge Output Paths1 | Merge | Rejoin Slack + Sheets branches (choose one) | Log Processing Metrics1; Send Slack Notification | Merge All Paths1 | ### Step 4: Distribution … |
| Merge All Paths1 | Merge | Unify success vs invalid-file response | Merge Output Paths1; Invalid File Response1 | Respond to Webhook1 |  |
| Respond to Webhook1 | Respond to Webhook | Return JSON response to caller | Merge All Paths1 | — |  |

---

## 4. Reproducing the Workflow from Scratch

1) **Create workflow**
   - Name it: **Video Processing Pipeline with Thumbnail Generation and CDN Distribution**
   - Keep workflow inactive until credentials are set.

2) **Add triggers**
   1. Add **Webhook** node → name **S3 Event Webhook1**
      - HTTP Method: **POST**
      - Path: `s3-video-upload`
      - Response mode: **Using “Respond to Webhook” node**
      - (Optional) On Error: continue (to match provided behavior)
   2. Add **Webhook** node → name **Manual Process Trigger1**
      - HTTP Method: **POST**
      - Path: `process-video`
      - Response mode: **Using “Respond to Webhook” node**

3) **Merge triggers**
   - Add **Merge** node → **Merge Triggers1**
     - Mode: **Choose Branch**
     - Connect:
       - S3 Event Webhook1 → Merge Triggers1 (Input 1 / index 0)
       - Manual Process Trigger1 → Merge Triggers1 (Input 2 / index 1)

4) **Normalize input (job payload)**
   - Add **Code** node → **Extract S3 Info1**
   - Paste logic equivalent to:
     - Parse S3 event at `Records[0]` and read `bucket.name` and `object.key` (decode `+` and URI encoding)
     - Or parse manual body: `bucket`, `key` or `video_key`
     - Derive:
       - `job_id`, `bucket`, `key`, `s3_url`, `https_url`, `filename`, `extension`, `is_video`, `event_type`, `started_at`
     - Supported extensions: `.mp4 .mov .avi .mkv .webm .m4v`
   - Connect Merge Triggers1 → Extract S3 Info1

5) **Add validation gate**
   1. Add **IF** node → **Check Is Video1**
      - Condition: Boolean equals
        - Left: `{{$json.is_video}}`
        - Right: `true`
      - Connect Extract S3 Info1 → Check Is Video1
   2. Add **Set** node → **Invalid File Response1**
      - Set fields:
        - `status` = `error`
        - `message` = `File is not a supported video format`
        - `job_id` = `{{$json.job_id}}`
      - Connect Check Is Video1 (false) → Invalid File Response1

6) **FFprobe metadata call**
   - Add **HTTP Request** node → **Get Video Metadata (FFprobe)1**
     - Method: **POST**
     - URL: `https://api.ffmpeg-service.com/v1/probe`
     - Body: JSON with:
       - `input`: `{{$json.https_url}}`
       - `options.show_format = true`, `options.show_streams = true`
     - Authentication: **Header Auth** (configure credential; see step 11)
     - Response option: enable “Never Error” equivalent
   - Connect Check Is Video1 (true) → Get Video Metadata (FFprobe)1

7) **Parse and enrich metadata**
   - Add **Code** node → **Parse Video Metadata1**
     - Implement:
       - select video/audio stream
       - compute duration, formatted duration
       - compute `thumbnail_times` at 10/30/50/70/90%
       - determine `needs_transcode` (even if not used)
       - attach derived fields (resolution, bitrates, etc.)
   - Connect Get Video Metadata (FFprobe)1 → Parse Video Metadata1

8) **Processing fan-out (3 parallel HTTP calls)**
   1. Add **HTTP Request** → **Generate Thumbnails (FFmpeg)1**
      - POST `https://api.ffmpeg-service.com/v1/thumbnails`
      - JSON body:
        - input `{{$json.https_url}}`
        - output_bucket `{{$json.bucket}}`
        - output_prefix `thumbnails/{{$json.job_id}}/`
        - times = `{{$json.thumbnail_times}}` (ensure it is serialized as JSON array)
        - sizes for large/medium/small, jpg quality 85
      - Auth: same header auth credential
      - Never Error on
   2. Add **HTTP Request** → **Generate Preview GIF1**
      - POST `https://api.ffmpeg-service.com/v1/gif`
      - JSON body:
        - output_key `previews/{{$json.job_id}}/preview.gif`
        - start_time = `floor(duration_seconds * 0.2)`
        - duration=5 fps=10 width=480
      - Auth: header auth; Never Error on
   3. Add **HTTP Request** → **Transcode Video1**
      - POST `https://api.ffmpeg-service.com/v1/transcode`
      - JSON body:
        - output_prefix `transcoded/{{$json.job_id}}/`
        - presets 1080p/720p/480p
        - codec h264, audio aac, mp4
      - Auth: header auth; Never Error on
   - Connect Parse Video Metadata1 to all three HTTP nodes.

9) **Aggregate parallel results**
   - Add **Aggregate** node → **Aggregate Processing Results1**
     - Mode: aggregate all item data
   - Connect each of the 3 processing HTTP nodes → Aggregate Processing Results1

10) **CDN purge + final output payload**
   1. Add **HTTP Request** → **Invalidate CDN Cache1**
      - POST `https://api.cloudflare.com/client/v4/zones/<ZONE_ID>/purge_cache`
      - JSON body with `prefixes` for thumbnails/previews/transcoded using job id from the aggregated structure
      - Auth: Cloudflare header auth credential (token)
      - Never Error on
   2. Add **Code** node → **Generate Signed URLs1**
      - Build a final JSON with:
        - job_id, status, completed_at
        - original metadata fields
        - CDN URLs using a base like `https://cdn.example.com`
        - urls_expire_at (informational unless you implement real signing)
   - Connect Aggregate Processing Results1 → Invalidate CDN Cache1 → Generate Signed URLs1

11) **Credentials setup**
   - **FFmpeg service (Header Auth):** create an n8n credential of type “HTTP Header Auth” (or generic header auth) with required header (commonly `Authorization: Bearer <token>` or `x-api-key: <key>` depending on your service).
   - **Cloudflare (Header Auth):** set `Authorization: Bearer <CLOUDFLARE_API_TOKEN>` and ensure token has cache purge permissions for the zone.
   - **Google Sheets:** set up Google OAuth2 credentials in n8n and authorize access.
   - **Slack:** set up Slack credentials (bot token) with permission to post messages to the target channel.

12) **Logging and notification (parallel)**
   1. Add **Google Sheets** node → **Log Processing Metrics1**
      - Operation: **Append**
      - Select **Document** and **Sheet**
      - Map columns to fields from Generate Signed URLs1 (job_id, filename, duration, etc.)
   2. Add **Slack** node → **Send Slack Notification**
      - Operation: send message to channel
      - Channel: `#video-processing` (or use channel ID)
      - Text uses expressions such as:
        - `{{$json.job_id}}`, `{{$json.original.filename}}`, etc.
   - Connect Generate Signed URLs1 → both nodes.

13) **Merge for response + respond**
   1. Add **Merge** node → **Merge Output Paths1**
      - Mode: **Choose Branch**
      - Connect:
        - Log Processing Metrics1 → Merge Output Paths1
        - Send Slack Notification → Merge Output Paths1
   2. Add **Merge** node → **Merge All Paths1**
      - Mode: **Choose Branch**
      - Connect:
        - Merge Output Paths1 → Merge All Paths1
        - Invalid File Response1 → Merge All Paths1
   3. Add **Respond to Webhook** node → **Respond to Webhook1**
      - Respond with: **JSON**
      - Body: `{{$json}}`
   - Connect Merge All Paths1 → Respond to Webhook1

**Important reproducibility note:** If you want the webhook response to always be the curated payload from **Generate Signed URLs1**, replace the “chooseBranch” merge pattern after Slack/Sheets with a deterministic merge (or route the response from Generate Signed URLs1 directly and make Slack/Sheets non-blocking).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “This workflow automates video processing from upload to delivery… You’ll need AWS S3 credentials, an FFmpeg API endpoint, Cloudflare API access, and a Slack bot token.” | From sticky note “## Video Processing Pipeline” (overview) |
| “Triggered by S3 events or manually via webhook… assets uploaded back to S3, CDN cache invalidated, Slack notification.” | From sticky note “## Video Processing Pipeline” (overview) |
| Step notes describing phases (Detection, Media Analysis, Processing, Distribution). | From sticky notes “Step 1” through “Step 4” |

