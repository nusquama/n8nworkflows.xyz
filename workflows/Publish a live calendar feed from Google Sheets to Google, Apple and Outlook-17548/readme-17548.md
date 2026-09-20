Publish a live calendar feed from Google Sheets to Google, Apple and Outlook

https://n8nworkflows.xyz/workflows/publish-a-live-calendar-feed-from-google-sheets-to-google--apple-and-outlook-17548


# Publish a live calendar feed from Google Sheets to Google, Apple and Outlook

### 1. Workflow Overview

This workflow publishes a live, token-protected iCalendar (`.ics`) feed from a Google Sheets "events" tab. It allows external calendar clients (Google Calendar, Apple Calendar, and Outlook) to subscribe to an up-to-date schedule URL. When a client requests the feed, the workflow validates the access token, checks an in-memory TTL cache, reads and rigorously validates spreadsheet rows if the cache is stale, serializes the data into RFC 5545-compliant iCalendar format, and serves the response with fail-safe mechanisms to ensure subscriptions are never broken by transient backend errors.

The logical execution blocks are organized as follows:
- **1.1 Input Reception & Configuration:** Receives the incoming HTTP webhook request and injects global feed parameters (timezone, TTL, tokens, date windows).
- **1.2 Authorization & Caching Layer:** Validates the query-string security token, rejects unauthorized requests cleanly, and checks workflow static memory for a fresh cache.
- **1.3 Data Ingestion & Strict Validation:** Pulls records from Google Sheets and passes them through a strict validator that filters malformed cells, out-of-window dates, and duplicate records without guessing or coercion.
- **1.4 RFC 5545 Serialization & Caching:** Renders validated events into properly folded, escaped, and formatted iCalendar text streams complete with health monitoring components, updating the internal static cache.
- **1.5 Response Finalization & Delivery:** Evaluates workflow status, manages fallback logic (serving stale cached copies on sheet or render failures), constructs security headers, and outputs the final HTTP response.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Configuration
- **Overview:** Initializes the HTTP listener for incoming calendar sync requests and defines the core static parameters, metadata, and caching configuration for the feed.
- **Nodes Involved:** `Receive Calendar Request`, `Set Feed Config`
- **Node Details:**
  - **Receive Calendar Request**
    - *Type and technical role:* `n8n-nodes-base.webhook` (Trigger)
    - *Configuration choices:* Listens on path `calendar.ics` with response mode set to `responseNode` (delegating response handling to downstream nodes).
    - *Key expressions or variables:* None.
    - *Input and output connections:* Input: External HTTP request; Output: `Set Feed Config`.
    - *Version-specific requirements:* Version 2.1.
    - *Edge cases or potential failure types:* Public endpoint exposure; unauthenticated spam or port scanning.
  - **Set Feed Config**
    - *Type and technical role:* `n8n-nodes-base.set` (Data Transformation)
    - *Configuration choices:* Sets hardcoded feed configuration variables including security tokens, namespace IDs, display names, timezones, cache TTLs, and validation windows.
    - *Key expressions or variables:* Static string and numeric assignments (`feed_token`, `feed_namespace`, `timezone`, `cache_ttl_seconds`, etc.).
    - *Input and output connections:* Input: `Receive Calendar Request`; Output: `Check Feed Token`.
    - *Version-specific requirements:* Version 3.4.
    - *Edge cases or potential failure types:* Placeholder values (`REPLACE-WITH-A-LONG-RANDOM-STRING`) must be updated by the administrator; otherwise, token verification will fail.

#### 2.2 Authorization & Caching Layer
- **Overview:** Validates the incoming request query token against the configured secret and checks whether a fresh cached calendar payload exists in memory.
- **Nodes Involved:** `Check Feed Token`, `Deny Unknown Token`, `Read Feed Cache`, `Check Cache Fresh`
- **Node Details:**
  - **Check Feed Token**
    - *Type and technical role:* `n8n-nodes-base.if` (Flow Control)
    - *Configuration choices:* Evaluates whether the incoming query parameter token matches the configured `feed_token` and confirms that placeholder values have been replaced.
    - *Key expressions or variables:* `={{ $('Receive Calendar Request').first().json.query.token }}` compared against `$json.feed_token`.
    - *Input and output connections:* Input: `Set Feed Config`; Outputs: True path to `Read Feed Cache`, False path to `Deny Unknown Token`.
    - *Version-specific requirements:* Version 2.3.
    - *Edge cases or potential failure types:* Missing query parameters result in evaluation mismatches and trigger denial.
  - **Deny Unknown Token**
    - *Type and technical role:* `n8n-nodes-base.set` (Data Transformation)
    - *Configuration choices:* Sets a denial flag (`deny: true`) to trigger a 404 response downstream.
    - *Key expressions or variables:* Boolean assignment `deny = true`.
    - *Input and output connections:* Input: `Check Feed Token` (False); Output: `Finalize Calendar Response`.
    - *Version-specific requirements:* Version 3.4.
    - *Edge cases or potential failure types:* None.
  - **Read Feed Cache**
    - *Type and technical role:* `n8n-nodes-base.code` (Custom JavaScript)
    - *Configuration choices:* Computes a configuration fingerprint and reads global workflow static memory (`$getWorkflowStaticData('global')`) to check for a valid TTL cache or a fallback "last-good" calendar body.
    - *Key expressions or variables:* Accesses `Set Feed Config` data and evaluates timestamp differences against `cache_ttl_seconds` and `stale_max_seconds`.
    - *Input and output connections:* Input: `Check Feed Token` (True); Output: `Check Cache Fresh`.
    - *Version-specific requirements:* Version 2.
    - *Edge cases or potential failure types:* Static memory is volatile on manual single-node testing inside the editor; cache misses during local execution are normal.
  - **Check Cache Fresh**
    - *Type and technical role:* `n8n-nodes-base.if` (Flow Control)
    - *Configuration choices:* Checks the boolean output of the cache reader to determine if fresh cached content is immediately servable.
    - *Key expressions or variables:* `={{ $json.cache_fresh }}` equals `true`.
    - *Input and output connections:* Input: `Read Feed Cache`; Outputs: True path to `Finalize Calendar Response`, False path to `Read Events Sheet`.
    - *Version-specific requirements:* Version 2.3.
    - *Edge cases or potential failure types:* None.

#### 2.3 Data Ingestion & Strict Validation
- **Overview:** Fetches event rows from the designated Google Sheet tab and passes them through a strict validation script that filters malformed dates, invalid times, control characters, and out-of-window entries.
- **Nodes Involved:** `Read Events Sheet`, `Validate Event Rows`, `Flag Sheet Failure`
- **Node Details:**
  - **Read Events Sheet**
    - *Type and technical role:* `n8n-nodes-base.googleSheets` (Integration)
    - *Configuration choices:* Reads sheet named `events` with `executeOnce: true`, `retryOnFail: true`, and error-handling routing enabled to continue on error output.
    - *Key expressions or variables:* Document ID and Sheet Name bindings.
    - *Credentials required:* Google Sheets OAuth2 API.
    - *Input and output connections:* Input: `Check Cache Fresh` (False); Outputs: Main success to `Validate Event Rows`, Error/continue-on-error path to `Flag Sheet Failure`.
    - *Version-specific requirements:* Version 4.7.
    - *Edge cases or potential failure types:* API rate limits, invalid OAuth credentials, or missing sheet headers will trigger the error output branch.
  - **Validate Event Rows**
    - *Type and technical role:* `n8n-nodes-base.code` (Custom JavaScript)
    - *Configuration choices:* Parses serial date/time numbers, validates timezones using Luxon, checks for duplicate UIDs, enforces business rules, and counts rejections without coercing dirty data.
    - *Key expressions or variables:* Utilizes `DateTime` from Luxon, processes input rows from Google Sheets, and checks configuration parameters.
    - *Input and output connections:* Input: `Read Events Sheet`; Outputs: Main success to `Build Calendar Text`, Error/continue-on-error path to `Flag Sheet Failure`.
    - *Version-specific requirements:* Version 2.
    - *Edge cases or potential failure types:* Throws execution errors if date parsing logic encounters unexpected fatal data structures, routing execution to the sheet failure handler.
  - **Flag Sheet Failure**
    - *Type and technical role:* `n8n-nodes-base.set` (Data Transformation)
    - *Configuration choices:* Assigns failure tracking flags and captures error messages when Google Sheets reads or row validation steps fail.
    - *Key expressions or variables:* `read_failed = true` and dynamic error message extraction from upstream nodes.
    - *Input and output connections:* Input: Error outputs from `Read Events Sheet`, `Validate Event Rows`, and `Build Calendar Text`; Output: `Finalize Calendar Response`.
    - *Version-specific requirements:* Version 3.4.
    - *Edge cases or potential failure types:* None.

#### 2.4 RFC 5545 Serialization & Caching
- **Overview:** Transforms validated event objects into an RFC 5545 compliant iCalendar payload, handles folding, escaping, timezones, and health status reporting, then updates the internal cache.
- **Nodes Involved:** `Build Calendar Text`, `Save Feed Cache`
- **Node Details:**
  - **Build Calendar Text**
    - *Type and technical role:* `n8n-nodes-base.code` (Custom JavaScript)
    - *Configuration choices:* Implements an RFC 5545 serialization engine with strict backslash escaping, 75-octet line folding using UTF-8 byte boundaries, CRLF line endings, and optional health-event generation.
    - *Key expressions or variables:* Reads configurations, event lists, and validation statistics from upstream nodes.
    - *Input and output connections:* Input: `Validate Event Rows`; Outputs: Main success to `Save Feed Cache`, Error/continue-on-error path to `Flag Sheet Failure`.
    - *Version-specific requirements:* Version 2.
    - *Edge cases or potential failure types:* Post-assertion checks throw errors if generated strings fail RFC validation rules (e.g., missing header, incorrect trailing characters, or invalid line folding), routing to the failure handler.
  - **Save Feed Cache**
    - *Type and technical role:* `n8n-nodes-base.code` (Custom JavaScript)
    - *Configuration choices:* Writes the successfully rendered calendar payload to workflow static memory (`$getWorkflowStaticData('global')`) under TTL and "last-good" storage slots if payload size is within limits.
    - *Key expressions or variables:* Compares byte size against `max_cache_bytes`.
    - *Input and output connections:* Input: `Build Calendar Text`; Output: `Finalize Calendar Response`.
    - *Version-specific requirements:* Version 2.
    - *Edge cases or potential failure types:* Large calendars exceeding `max_cache_bytes` skip caching but still pass through to delivery.

#### 2.5 Response Finalization & Delivery
- **Overview:** Evaluates all execution paths, determines appropriate HTTP status codes, injects security and caching headers, and returns the final calendar stream or error message to the requesting client.
- **Nodes Involved:** `Finalize Calendar Response`, `Return Calendar Feed`
- **Node Details:**
  - **Finalize Calendar Response**
    - *Type and technical role:* `n8n-nodes-base.code` (Custom JavaScript)
    - *Configuration choices:* Fan-in node that aggregates responses from authorization failure, cache hits, fresh builds, and sheet read failures. Implements fallback logic to serve last-good cached bodies with stale age tracking or return a 503 error.
    - *Key expressions or variables:* Computes entity tags (`ETag`), content types, cache control directives, and stale tracking headers.
    - *Input and output connections:* Input: `Save Feed Cache`, `Check Cache Fresh` (True), `Deny Unknown Token`, and `Flag Sheet Failure`; Output: `Return Calendar Feed`.
    - *Version-specific requirements:* Version 2.
    - *Edge cases or potential failure types:* None.
  - **Return Calendar Feed**
    - *Type and technical role:* `n8n-nodes-base.respondToWebhook` (Response Handler)
    - *Configuration choices:* Sends an HTTP response with dynamic status codes, streaming disabled, custom response headers (Content-Type, Cache-Control, ETag, X-Feed-Stale, etc.), and text response body.
    - *Key expressions or variables:* `={{ $json.status_code }}`, `={{ $json.content_type }}`, `={{ $json.cache_control }}`, `={{ $json.response_body }}`.
    - *Input and output connections:* Input: `Finalize Calendar Response`; Output: External HTTP client.
    - *Version-specific requirements:* Version 1.5.
    - *Edge cases or potential failure types:* Client disconnection during transmission or header size limitations.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Receive Calendar Request | n8n-nodes-base.webhook | Trigger HTTP webhook for calendar sync | None | Set Feed Config | Publish a Google Sheet as a live calendar feed<br><br>Turn a sheet of events into a URL that Google Calendar, Apple Calendar and Outlook can subscribe to. Edit a row and the event changes for everyone. Delete the row and it goes away. Read-only: this workflow never writes to your sheet.<br><br>Sheet columns: `uid` \| `title` \| `start_date` \| `start_time` \| `end_date` \| `end_time` \| `location` \| `description` \| `status` \| `updated_at`<br><br>### How it works<br>1. A calendar client fetches the feed URL and the token in the query string is checked against `Set Feed Config`. A miss gets a plain 404.<br>2. A short-lived cache is checked first, so a burst of client refreshes does not turn into a burst of Google Sheets reads.<br>3. The events tab is read with date values as serial numbers, never as locale display strings, so 03/04 can never be read as the wrong month.<br>4. Every row is validated. A bad cell is rejected and counted with its row number, never guessed at and never silently coerced.<br>5. The surviving rows are serialized to RFC 5545 iCalendar text: CRLF endings, 75-octet folding, all-day dates as DATE values with the exclusive end date, timed events converted to UTC.<br>6. The feed is returned as `text/calendar`. If the sheet read fails, the last good copy is served instead, so a transient error never empties anyone's calendar.<br><br>### Setup steps<br>- [ ] Create a tab named `events` with the ten headers above, spelled exactly.<br>- [ ] Connect your Google Sheets credential on `Read Events Sheet` and pick your document.<br>- [ ] In `Set Feed Config`, replace `feed_token` with 32 or more random characters and `feed_namespace` with something you own, for example `acme-ops.example`.<br>- [ ] Set `timezone` to match File > Settings > Time zone in your spreadsheet.<br>- [ ] Activate the workflow and copy the PRODUCTION `/webhook/` URL, then add `?token=` and your token.<br>- [ ] Subscribe: Google Calendar > Other calendars > From URL. Apple: File > New Calendar Subscription. Outlook: Add calendar > Subscribe from web.<br><br>### Customization<br>Set `status` on a row to `tentative` or `cancelled` to keep it but change how it shows. Fill `updated_at` to give one event its own DTSTAMP. `window_future_days` trims far-future events; leave `window_past_days` at 0 unless you accept that trimming the past removes history from every subscriber. Set `health_event` to false once you trust the feed.<br><br>Not included on purpose: recurring events (use one row per occurrence), TZID with a generated VTIMEZONE, and conditional requests. |
| Set Feed Config | n8n-nodes-base.set | Assign configuration and metadata parameters | Receive Calendar Request | Check Feed Token | Serve and authorise<br><br>Calendar clients subscribe by URL only and cannot send an auth header, so the secret is a query-string token checked here. A wrong token gets a plain 404, never a 401, because 401 makes Apple and Outlook prompt for credentials that cannot exist. |
| Check Feed Token | n8n-nodes-base.if | Verify query security token | Set Feed Config | Read Feed Cache, Deny Unknown Token | Serve and authorise<br><br>Calendar clients subscribe by URL only and cannot send an auth header, so the secret is a query-string token checked here. A wrong token gets a plain 404, never a 401, because 401 makes Apple and Outlook prompt for credentials that cannot exist. |
| Deny Unknown Token | n8n-nodes-base.set | Flag unauthorized requests for 404 response | Check Feed Token | Finalize Calendar Response | Serve and authorise<br><br>Calendar clients subscribe by URL only and cannot send an auth header, so the secret is a query-string token checked here. A wrong token gets a plain 404, never a 401, because 401 makes Apple and Outlook prompt for credentials that cannot exist. |
| Read Feed Cache | n8n-nodes-base.code | Read workflow static memory for cached payload | Check Feed Token | Check Cache Fresh | Read and validate<br><br>Date cells are read as serial numbers, never as locale display strings, so 03/04 can never be read as the wrong month. A blank `start_time` means all-day; every other malformed cell is rejected and counted with its row number. |
| Check Cache Fresh | n8n-nodes-base.if | Determine if cache is fresh | Read Feed Cache | Finalize Calendar Response, Read Events Sheet | Read and validate<br><br>Date cells are read as serial numbers, never as locale display strings, so 03/04 can never be read as the wrong month. A blank `start_time` means all-day; every other malformed cell is rejected and counted with its row number. |
| Read Events Sheet | n8n-nodes-base.googleSheets | Read rows from Google Sheets event tab | Check Cache Fresh | Validate Event Rows, Flag Sheet Failure | Read and validate<br><br>Date cells are read as serial numbers, never as locale display strings, so 03/04 can never be read as the wrong month. A blank `start_time` means all-day; every other malformed cell is rejected and counted with its row number. |
| Validate Event Rows | n8n-nodes-base.code | Validate spreadsheet rows and filter invalid entries | Read Events Sheet | Build Calendar Text, Flag Sheet Failure | Read and validate<br><br>Date cells are read as serial numbers, never as locale display strings, so 03/04 can never be read as the wrong month. A blank `start_time` means all-day; every other malformed cell is rejected and counted with its row number. |
| Build Calendar Text | n8n-nodes-base.code | Serialize events into RFC 5545 iCalendar format | Validate Event Rows | Save Feed Cache, Flag Sheet Failure | Render RFC 5545<br><br>Escaping is backslash first, then semicolon, comma and newline. Colon and double quote are left alone. All-day `DTEND` is the day AFTER the last day. |
| Save Feed Cache | n8n-nodes-base.code | Write generated calendar to static memory cache | Build Calendar Text | Finalize Calendar Response | Render RFC 5545<br><br>Escaping is backslash first, then semicolon, comma and newline. Colon and double quote are left alone. All-day `DTEND` is the day AFTER the last day. |
| Flag Sheet Failure | n8n-nodes-base.set | Set error flags when reads or renders fail | Read Events Sheet, Validate Event Rows, Build Calendar Text | Finalize Calendar Response | Respond and fail safe<br><br>Four branches meet here and exactly one fires. A 200 is only ever paired with a body that opens `BEGIN:VCALENDAR`, because a client replaces its whole copy from any parseable 200. |
| Finalize Calendar Response | n8n-nodes-base.code | Aggregate paths, handle fail-safe and construct headers | Save Feed Cache, Check Cache Fresh, Deny Unknown Token, Flag Sheet Failure | Return Calendar Feed | Respond and fail safe<br><br>Four branches meet here and exactly one fires. A 200 is only ever paired with a body that opens `BEGIN:VCALENDAR`, because a client replaces its whole copy from any parseable 200. |
| Return Calendar Feed | n8n-nodes-base.respondToWebhook | Respond to HTTP request with iCalendar content | Finalize Calendar Response | None | Respond and fail safe<br><br>Four branches meet here and exactly one fires. A 200 is only ever paired with a body that opens `BEGIN:VCALENDAR`, because a client replaces its whole copy from any parseable 200. |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually in n8n:

1. **Create the Webhook Trigger:**
   - Add a **Webhook** node named `Receive Calendar Request`.
   - Set Path to `calendar.ics` and Response Mode to `Response Node`.

2. **Configure Feed Parameters:**
   - Add a **Set** node named `Set Feed Config`.
   - Add string and number assignments for: `feed_token` (replace with a secure random string), `feed_namespace` (e.g., `mycompany-calendar.invalid`), `calendar_name`, `calendar_description`, `timezone` (e.g., `America/New_York`), `feed_epoch` (`20260101T000000Z`), `prod_id` (`-//exekyute//Sheet Calendar Feed 1.0//EN`), `refresh_hint` (`PT1H`), `default_duration_minutes` (`60`), `window_past_days` (`0`), `window_future_days` (`730`), `cache_ttl_seconds` (`300`), `stale_max_seconds` (`86400`), `max_cache_bytes` (`524288`), and `health_event` (`true`).
   - Connect `Receive Calendar Request` to `Set Feed Config`.

3. **Set Up Authorization Checks:**
   - Add an **If** node named `Check Feed Token`. Configure condition to check if `{{ $('Receive Calendar Request').first().json.query.token }}` equals `{{ $json.feed_token }}` (along with checks ensuring parameters are not default placeholders).
   - Add a **Set** node named `Deny Unknown Token` assigning `deny: true`. Connect the `false` branch of `Check Feed Token` here.
   - Connect the `true` branch of `Check Feed Token` to a **Code** node named `Read Feed Cache` containing the cache checking and fingerprint generation JavaScript logic.

4. **Configure Cache Routing:**
   - Add an **If** node named `Check Cache Fresh` connected after `Read Feed Cache`. Set condition to evaluate if `{{ $json.cache_fresh }}` equals `true`.

5. **Set Up Google Sheets Integration:**
   - Add a **Google Sheets** node named `Read Events Sheet`. Configure it to read sheet name `events` from your target document. Enable `executeOnce`, `retryOnFail`, and set `onError` to continue error output.
   - Configure **Google Sheets OAuth2 API** credentials.
   - Connect the `false` branch of `Check Cache Fresh` to `Read Events Sheet`.

6. **Implement Row Validation:**
   - Add a **Code** node named `Validate Event Rows`. Set `onError` to continue error output. Insert the strict data validation and parsing script.
   - Connect the main success output of `Read Events Sheet` to `Validate Event Rows`.

7. **Implement RFC 5545 Serializer & Cache Writer:**
   - Add a **Code** node named `Build Calendar Text`. Set `onError` to continue error output. Insert the RFC 5545 serialization and post-assertion script. Connect `Validate Event Rows` success output here.
   - Add a **Code** node named `Save Feed Cache` to store payloads in workflow static memory. Connect `Build Calendar Text` success output here.

8. **Implement Error Handling & Fail-Safe Paths:**
   - Add a **Set** node named `Flag Sheet Failure` assigning `read_failed: true` and extracting error messages. Connect error outputs from `Read Events Sheet`, `Validate Event Rows`, and `Build Calendar Text` to this node.

9. **Finalize Response & Webhook Return:**
   - Add a **Code** node named `Finalize Calendar Response` acting as a fan-in point for `Save Feed Cache`, `Check Cache Fresh` (true branch), `Deny Unknown Token`, and `Flag Sheet Failure`. Insert the response evaluation and fallback logic.
   - Add a **Respond to Webhook** node named `Return Calendar Feed`. Set response code to `={{ $json.status_code }}`, response body to `={{ $json.body }}`, and configure headers (`Content-Type`, `Cache-Control`, `Content-Disposition`, `ETag`, `X-Robots-Tag`, `X-Feed-Stale`, `Retry-After`).
   - Connect `Finalize Calendar Response` to `Return Calendar Feed`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Google Sheets column schema required: `uid`, `title`, `start_date`, `start_time`, `end_date`, `end_time`, `location`, `description`, `status`, `updated_at`, `rrule`. | Spreadsheet Structure |
| The webhook URL query string (`?token=...`) acts as the security password. Anyone with the URL can read all sheet rows. | Security Consideration |
| Keep the workflow active; deactivating it breaks external subscriptions in Google Calendar, Apple Calendar, and Outlook, requiring manual re-subscription. | Lifecycle Management |