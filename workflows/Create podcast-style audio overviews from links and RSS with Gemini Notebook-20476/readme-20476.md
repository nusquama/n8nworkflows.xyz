Create podcast-style audio overviews from links and RSS with Gemini Notebook

https://n8nworkflows.xyz/workflows/create-podcast-style-audio-overviews-from-links-and-rss-with-gemini-notebook-20476


# Create podcast-style audio overviews from links and RSS with Gemini Notebook

### 1. Workflow Overview

This workflow automates the creation of podcast-style Audio Overviews from web pages, YouTube videos, or an RSS feed using the Gemini Notebook (NotebookLM) API via *useapi.net*. It features a dual-entry architecture supporting both interactive user submissions through an n8n form and automated weekly batch processing via an RSS feed. 

The logical execution follows a pipeline divided into distinct functional blocks:
- **1.1 Input Reception & Normalization:** Captures input either interactively via a form or scheduled via an RSS feed, parsing up to 50 source URLs and configuration parameters (format, length, language, focus).
- **1.2 Notebook Creation & Source Ingestion:** Initializes a Gemini notebook, bulk-adds source links with fallback logic for individual link failures, and polls until sources are marked as readable.
- **1.3 Audio Generation & Polling:** Requests an asynchronous Audio Overview artifact generation, handling retries for busy states, and polling until the `.m4a` file is ready.
- **1.4 Result Delivery & Output Handling:** Delivers an interactive in-browser audio player/download page for form-based requests, or extracts binary `.m4a` file payloads for scheduled downstream automated pipelines.
- **1.5 Error Handling & Webhook Audio Proxy:** Manages failures gracefully with contextual error messages and provides a secured webhook proxy for downloading/streaming audio artifacts without exposing API tokens to the client browser.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Reception & Normalization
- **Overview:** Receives raw input from users via an HTML-styled n8n form or triggers weekly via a schedule node parsing an RSS feed, normalizing parameters into a consistent execution state.
- **Nodes Involved:** `Make a podcast`, `Form settings`, `Every Monday at 7:00`, `Feed settings`, `Read feed`, `Newest posts`, `Any links?`
- **Node Details:**
  - `Make a podcast` (`n8n-nodes-base.formTrigger`)
    - *Type/Role:* Form trigger node capturing episode title, raw source links, format, length, language, optional focus instructions, and Google account email.
    - *Configuration:* Custom HTML/CSS styling applied; collects up to 50 links.
    - *Input/Output:* Triggers on form submission -> Output to `Form settings`.
    - *Edge Cases:* Empty input values handled downstream; malformed emails or unsupported formats rejected at normalization.
  - `Form settings` (`n8n-nodes-base.code`)
    - *Type/Role:* Code node parsing form fields into a standardized pipeline state object (`source: 'form'`).
    - *Configuration:* Extracts unique valid HTTP(S) links, limits arrays to 50 elements, maps UI labels to API tokens.
    - *Input/Output:* Input from `Make a podcast` -> Output to `Any links?`.
  - `Every Monday at 7:00` (`n8n-nodes-base.scheduleTrigger`)
    - *Type/Role:* Cron/Schedule trigger node executing weekly on Mondays.
    - *Configuration:* Disabled by default; configured for weekly intervals at hour 7.
    - *Input/Output:* Scheduled trigger -> Output to `Feed settings`.
  - `Feed settings` (`n8n-nodes-base.set`)
    - *Type/Role:* Set/Edit Fields node defining RSS configurations (feed URL, lookback window, max posts, format settings).
    - *Configuration:* Hardcodes target RSS feed URL (`https://blog.google/...`) and default podcast parameters.
    - *Input/Output:* Input from schedule -> Output to `Read feed`.
  - `Read feed` (`n8n-nodes-base.rssFeedRead`)
    - *Type/Role:* RSS Read node fetching feed entries.
    - *Configuration:* Evaluates URL dynamically from `={{ $json.feedUrl }}`.
    - *Input/Output:* Input from `Feed settings` -> Output to `Newest posts`.
  - `Newest posts` (`n8n-nodes-base.code`)
    - *Type/Role:* Code node filtering RSS items within the lookback window.
    - *Configuration:* Sorts by publication date, limits items to `maxPosts`, formats state as `source: 'rss'`.
    - *Input/Output:* Input from `Read feed` -> Output to `Any links?`.
  - `Any links?` (`n8n-nodes-base.if`)
    - *Type/Role:* Conditional router validating if valid source URLs exist.
    - *Configuration:* Checks `{{ $json.state.urls.length > 0 }}`.
    - *Input/Output:* Input from `Form settings` or `Newest posts` -> True branch to `Create notebook`, False branch to error handling (`Show a page?` / `Stop with error`).

#### Block 1.2: Notebook Creation & Source Ingestion
- **Overview:** Creates a dedicated Gemini notebook for the execution session, adds source links with bulk and single-item fallback mechanisms, and polls until sources are fully processed.
- **Nodes Involved:** `Create notebook`, `Notebook created`, `Notebook ok?`, `Add sources`, `Sources added`, `Sources route`, `One link each`, `Add each link`, `Links added`, `Wait 10s`, `Read notebook`, `Source status`, `Sources next`, `Sources`
- **Node Details:**
  - `Create notebook` (`n8n-nodes-base.httpRequest`)
    - *Type/Role:* HTTP Request node calling the *useapi.net* API to instantiate a new notebook.
    - *Configuration:* POST `https://api.useapi.net/v1/gemini-notebook/notebooks`, HTTP Header Auth credential, timeout 120s, max tries 3.
    - *Expressions:* JSON body maps `title` and optional account `email`.
    - *Input/Output:* Input from `Any links?` -> Output to `Notebook created`.
    - *Failure Modes:* Handles 401 (token error), 429 (quota/rate limit), 596 (signed out account) in downstream parsing code.
  - `Notebook created` (`n8n-nodes-base.code`)
    - *Type/Role:* Code node validating notebook creation API responses.
    - *Configuration:* Parses error response objects, handles API quotas, token failures, and temporary network issues.
    - *Input/Output:* Input from `Create notebook` -> Output to `Notebook ok?`.
  - `Notebook ok?` (`n8n-nodes-base.if`)
    - *Type/Role:* Conditional node checking if notebook creation succeeded (`route === 'ok'`).
    - *Input/Output:* Input from `Notebook created` -> True to `Add sources`, False to error routing (`Show a page?`).
  - `Add sources` (`n8n-nodes-base.httpRequest`)
    - *Type/Role:* HTTP Request node performing batch source addition.
    - *Configuration:* POST `https://api.useapi.net/v1/gemini-notebook/sources`, sends notebook ID and array of URLs.
    - *Input/Output:* Input from `Notebook ok?` -> Output to `Sources added`.
  - `Sources added` (`n8n-nodes-base.code`)
    - *Type/Role:* Code node evaluating batch source addition results.
    - *Configuration:* Detects HTTP 409 errors (indicating Google rejected a URL in the batch) and routes to individual ingestion (`each`).
    - *Input/Output:* Input from `Add sources` -> Output to `Sources route`.
  - `Sources route` (`n8n-nodes-base.switch`)
    - *Type/Role:* Switch node routing execution based on source ingestion strategy (`stop`, `each`, or `ok`).
    - *Input/Output:* Input from `Sources added`.
  - `One link each` (`n8n-nodes-base.code`)
    - *Type/Role:* Code node splitting URLs into individual items for sequential/parallel processing upon batch rejection.
    - *Input/Output:* Input from `Sources route` (`each`) -> Output to `Add each link`.
  - `Add each link` (`n8n-nodes-base.httpRequest`)
    - *Type/Role:* HTTP Request node adding a single source URL.
    - *Configuration:* POST `https://api.useapi.net/v1/gemini-notebook/sources` with a single-item array.
    - *Input/Output:* Input from `One link each` -> Output to `Links added`.
  - `Links added` (`n8n-nodes-base.code`)
    - *Type/Role:* Code node collecting individual link addition results, filtering out failed ones.
    - *Input/Output:* Input from `Add each link` -> Output to `Sources route` (or stop if all fail).
  - `Wait 10s` (`n8n-nodes-base.wait`)
    - *Type/Role:* Wait node introducing a 10-second delay between source status checks.
    - *Input/Output:* Input from `Sources route` (`ok`) -> Output to `Read notebook`.
  - `Read notebook` (`n8n-nodes-base.httpRequest`)
    - *Type/Role:* HTTP Request node fetching current notebook state and source processing statuses.
    - *Configuration:* GET `https://api.useapi.net/v1/gemini-notebook/notebooks/{notebook}`.
    - *Input/Output:* Input from `Wait 10s` -> Output to `Source status`.
  - `Source status` (`n8n-nodes-base.code`)
    - *Type/Role:* Code node checking if all notebook sources are ready, failed, or still processing.
    - *Configuration:* Implements a polling loop capped at 60 polls (10 minutes); compiles lists of ready and skipped sources.
    - *Input/Output:* Input from `Read notebook` -> Output to `Sources next`.
  - `Sources next` (`n8n-nodes-base.switch`)
    - *Type/Role:* Switch node routing based on source status (`wait`, `stop`, `page`, or `generate`).
    - *Input/Output:* Input from `Source status`.
  - `Sources` (`n8n-nodes-base.form`)
    - *Type/Role:* Interactive n8n Form node presenting a review page of parsed sources to the user.
    - *Configuration:* Operation set to `page`, renders dynamic HTML table showing ready vs. skipped sources.
    - *Input/Output:* Input from `Sources next` -> Output to `Generate request`.

#### Block 1.3: Audio Generation & Polling
- **Overview:** Initiates asynchronous generation of the Audio Overview artifact, handles account busy states and rate-limit retries, and polls job execution status until completion.
- **Nodes Involved:** `Generate request`, `Generate Audio Overview`, `Generate check`, `Generate route`, `Retry in 15s`, `Wait 20s`, `Check job`, `Job status`, `Job next`
- **Node Details:**
  - `Generate request` (`n8n-nodes-base.code`)
    - *Type/Role:* Code node preparing the configuration payload for Audio Overview generation.
    - *Input/Output:* Input from `Sources` form submission, `Source status`, or `Retry in 15s` -> Output to `Generate Audio Overview`.
  - `Generate Audio Overview` (`n8n-nodes-base.httpRequest`)
    - *Type/Role:* HTTP Request node submitting an asynchronous generation job.
    - *Configuration:* POST `https://api.useapi.net/v1/gemini-notebook/artifacts` with body `{ type: 'audio', mode: 'async', ... }`.
    - *Input/Output:* Input from `Generate request` -> Output to `Generate check`.
  - `Generate check` (`n8n-nodes-base.code`)
    - *Type/Role:* Code node validating generation job initiation responses.
    - *Configuration:* Detects job IDs, handles transient API timeouts, account busy states (429), and unsettled sources (409).
    - *Input/Output:* Input from `Generate Audio Overview` -> Output to `Generate route`.
  - `Generate route` (`n8n-nodes-base.switch`)
    - *Type/Role:* Switch node routing between `retry`, `stop`, and `ok` paths.
    - *Input/Output:* Input from `Generate check`.
  - `Retry in 15s` (`n8n-nodes-base.wait`)
    - *Type/Role:* Wait node pausing execution for 15 seconds before retrying generation requests.
    - *Input/Output:* Input from `Generate route` (`retry`) -> Output to `Generate request`.
  - `Wait 20s` (`n8n-nodes-base.wait`)
    - *Type/Role:* Wait node introducing a 20-second polling delay for job completion checks.
    - *Input/Output:* Input from `Generate route` (`ok`) or `Job next` (`wait`) -> Output to `Check job`.
  - `Check job` (`n8n-nodes-base.httpRequest`)
    - *Type/Role:* HTTP Request node polling asynchronous job status.
    - *Configuration:* GET `https://api.useapi.net/v1/gemini-notebook/jobs/{jobid}`.
    - *Input/Output:* Input from `Wait 20s` -> Output to `Job status`.
  - `Job status` (`n8n-nodes-base.code`)
    - *Type/Role:* Code node evaluating job completion status (`completed`, `failed`, or ongoing).
    - *Configuration:* Implements max poll threshold (90 polls / 30 minutes); extracts resulting `.m4a` file metadata and artifact identifiers.
    - *Input/Output:* Input from `Check job` -> Output to `Job next`.
  - `Job next` (`n8n-nodes-base.switch`)
    - *Type/Role:* Switch node routing based on job status (`wait`, `stop`, or `done`).
    - *Input/Output:* Input from `Job status`.

#### Block 1.4: Result Delivery & Output Handling
- **Overview:** Delivers the finished podcast either via an interactive browser form containing an embedded HTML5 audio player and download link, or passes binary `.m4a` data for downstream automation steps.
- **Nodes Involved:** `From the form?`, `Result page`, `Your podcast`, `Finish`, `Download podcast`, `Episode`
- **Node Details:**
  - `From the form?` (`n8n-nodes-base.if`)
    - *Type/Role:* Conditional node checking execution origin (`source === 'form'`).
    - *Input/Output:* Input from `Job next` (`done`) -> True branch to `Result page`, False branch to `Download podcast`.
  - `Result page` (`n8n-nodes-base.code`)
    - *Type/Role:* Code node generating HTML markup for the playback form.
    - *Configuration:* Constructs secure webhook player URLs and download links pointing to the local webhook proxy.
    - *Input/Output:* Input from `From the form?` -> Output to `Your podcast`.
  - `Your podcast` (`n8n-nodes-base.form`)
    - *Type/Role:* Interactive n8n Form node displaying the podcast player page.
    - *Configuration:* Operation `page`, renders HTML5 audio player, download link, and summary of skipped sources.
    - *Input/Output:* Input from `Result page` -> Output to `Finish`.
  - `Finish` (`n8n-nodes-base.form`)
    - *Type/Role:* n8n Form Completion node marking the workflow execution as successfully finished.
    - *Configuration:* Operation `completion`, displays completion message.
    - *Input/Output:* Input from `Your podcast`.
  - `Download podcast` (`n8n-nodes-base.httpRequest`)
    - *Type/Role:* HTTP Request node downloading the `.m4a` binary file from the artifact URL for scheduled/automated runs.
    - *Configuration:* GET request with file response format.
    - *Input/Output:* Input from `From the form?` (RSS branch) -> Output to `Episode`.
  - `Episode` (`n8n-nodes-base.code`)
    - *Type/Role:* Code node finalizing binary payload structures and JSON metadata for downstream automation nodes (e.g., S3, Google Drive, Telegram).
    - *Input/Output:* Input from `Download podcast` -> Output for custom user integrations.

#### Block 1.5: Error Handling & Webhook Audio Proxy
- **Overview:** Handles execution failures by presenting user-friendly error forms or throwing workflow errors, and provides a secure webhook proxy endpoint for streaming and downloading audio files without leaking API tokens.
- **Nodes Involved:** `Show a page?`, `Stopped`, `Stop with error`, `Podcast audio`, `Audio request`, `Fetch audio`, `Send audio`
- **Node Details:**
  - `Show a page?` (`n8n-nodes-base.if`)
    - *Type/Role:* Conditional node determining whether to display an error form (if triggered from form) or throw an error (if scheduled/automated).
    - *Input/Output:* Input from error routes -> True to `Stopped`, False to `Stop with error`.
  - `Stopped` (`n8n-nodes-base.form`)
    - *Type/Role:* Interactive n8n Form node displaying failure details and troubleshooting instructions.
    - *Configuration:* Operation `page`, renders sanitized HTML error messages.
    - *Input/Output:* Input from `Show a page?`.
  - `Stop with error` (`n8n-nodes-base.stopAndError`)
    - *Type/Role:* Stop and Error node terminating scheduled executions with a descriptive error message for error workflows.
    - *Input/Output:* Input from `Show a page?`.
  - `Podcast audio` (`n8n-nodes-base.webhook`)
    - *Type/Role:* Webhook trigger node receiving audio streaming/download requests from the browser player.
    - *Configuration:* Path `notebooklm-podcast-audio`, response mode set to `responseNode`.
    - *Input/Output:* Incoming HTTP request -> Output to `Audio request`.
  - `Audio request` (`n8n-nodes-base.code`)
    - *Type/Role:* Code node validating artifact IDs against strict regex patterns to prevent unauthorized resource access.
    - *Input/Output:* Input from `Podcast audio` -> Output to `Fetch audio`.
  - `Fetch audio` (`n8n-nodes-base.httpRequest`)
    - *Type/Role:* HTTP Request node securely downloading the audio file using server-side credentials.
    - *Configuration:* GET request to *useapi.net* artifact download endpoint with file response format.
    - *Input/Output:* Input from `Audio request` -> Output to `Send audio`.
  - `Send audio` (`n8n-nodes-base.respondToWebhook`)
    - *Type/Role:* Respond to Webhook node returning the `.m4a` binary stream with appropriate MIME types (`audio/mp4`) and Content-Disposition headers.
    - *Input/Output:* Input from `Fetch audio`.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Make a podcast | n8n-nodes-base.formTrigger | Triggers the workflow upon form submission | None | Form settings | NotebookLM Podcast App: turn links or an RSS feed into an Audio Overview<br>Two ways in |
| Form settings | n8n-nodes-base.code | Normalizes form inputs into pipeline state | Make a podcast | Any links? | Two ways in |
| Every Monday at 7:00 | n8n-nodes-base.scheduleTrigger | Triggers weekly RSS feed processing | None | Feed settings | Two ways in |
| Feed settings | n8n-nodes-base.set | Defines RSS feed parameters and options | Every Monday at 7:00 | Read feed | Two ways in |
| Read feed | n8n-nodes-base.rssFeedRead | Reads RSS feed items | Feed settings | Newest posts | Two ways in |
| Newest posts | n8n-nodes-base.code | Filters recent RSS feed posts | Read feed | Any links? | Two ways in |
| Any links? | n8n-nodes-base.if | Checks whether valid source URLs exist | Form settings, Newest posts | Create notebook, Show a page? | Notebook and sources |
| Create notebook | n8n-nodes-base.httpRequest | Creates a new Gemini notebook via API | Any links? | Notebook created | Notebook and sources |
| Notebook created | n8n-nodes-base.code | Validates notebook creation response | Create notebook | Notebook ok? | Notebook and sources |
| Notebook ok? | n8n-nodes-base.if | Verifies notebook creation success | Notebook created | Add sources, Show a page? | Notebook and sources |
| Add sources | n8n-nodes-base.httpRequest | Batch adds source URLs to the notebook | Notebook ok? | Sources added | Notebook and sources |
| Sources added | n8n-nodes-base.code | Handles batch source addition results | Add sources | Sources route | Notebook and sources |
| Sources route | n8n-nodes-base.switch | Routes based on batch ingestion outcome | Sources added | Show a page?, One link each, Wait 10s | Notebook and sources |
| One link each | n8n-nodes-base.code | Prepares individual URLs for fallback ingestion | Sources route | Add each link | Notebook and sources |
| Add each link | n8n-nodes-base.httpRequest | Adds a single source URL | One link each | Links added | Notebook and sources |
| Links added | n8n-nodes-base.code | Evaluates individual link addition results | Add each link | Sources route | Notebook and sources |
| Wait 10s | n8n-nodes-base.wait | Pauses before checking source read status | Sources route | Read notebook | Notebook and sources |
| Read notebook | n8n-nodes-base.httpRequest | Fetches notebook details and source statuses | Wait 10s | Source status | Notebook and sources |
| Source status | n8n-nodes-base.code | Analyzes readiness of notebook sources | Read notebook | Sources next | Notebook and sources |
| Sources next | n8n-nodes-base.switch | Routes based on source read progress | Source status | Wait 10s, Show a page?, Sources, Generate request | Notebook and sources |
| Sources | n8n-nodes-base.form | Displays interactive review page of sources | Sources next | Generate request | Notebook and sources |
| Generate request | n8n-nodes-base.code | Prepares Audio Overview generation payload | Sources, Source status, Retry in 15s | Generate Audio Overview | Audio Overview |
| Generate Audio Overview | n8n-nodes-base.httpRequest | Submits asynchronous generation job | Generate request | Generate check | Audio Overview |
| Generate check | n8n-nodes-base.code | Validates generation job submission response | Generate Audio Overview | Generate route | Audio Overview |
| Generate route | n8n-nodes-base.switch | Routes generation retry, stop, or ok states | Generate check | Retry in 15s, Show a page?, Wait 20s | Audio Overview |
| Retry in 15s | n8n-nodes-base.wait | Waits before retrying generation request | Generate route | Generate request | Audio Overview |
| Wait 20s | n8n-nodes-base.wait | Pauses between job status checks | Generate route, Job next | Check job | Audio Overview |
| Check job | n8n-nodes-base.httpRequest | Polls asynchronous job status | Wait 20s | Job status | Audio Overview |
| Job status | n8n-nodes-base.code | Evaluates job completion or failure | Check job | Job next | Audio Overview |
| Job next | n8n-nodes-base.switch | Routes based on job status | Job status | Wait 20s, Show a page?, From the form? | Audio Overview, Result |
| From the form? | n8n-nodes-base.if | Determines result delivery mode (form vs schedule) | Job next | Result page, Download podcast | Result |
| Result page | n8n-nodes-base.code | Generates HTML player page markup | From the form? | Your podcast | Result |
| Your podcast | n8n-nodes-base.form | Displays interactive podcast player form | Result page | Finish | Result |
| Finish | n8n-nodes-base.form | Marks form execution complete | Your podcast | None | Result |
| Download podcast | n8n-nodes-base.httpRequest | Downloads `.m4a` file for automated runs | From the form? | Episode | Result |
| Episode | n8n-nodes-base.code | Formats binary output and JSON metadata | Download podcast | None | Add your next node here |
| Show a page? | n8n-nodes-base.if | Decides whether to show error form or throw error | Any links?, Notebook ok?, Sources route, Sources next, Generate route, Job status | Stopped, Stop with error | Stops |
| Stopped | n8n-nodes-base.form | Displays error message form to user | Show a page? | None | Stops |
| Stop with error | n8n-nodes-base.stopAndError | Stops execution with error for scheduled runs | Show a page? | None | Stops |
| Podcast audio | n8n-nodes-base.webhook | Receives audio stream/download requests | None | Audio request | Podcast audio (player and download) |
| Audio request | n8n-nodes-base.code | Validates artifact ID query parameters | Podcast audio | Fetch audio | Podcast audio (player and download) |
| Fetch audio | n8n-nodes-base.httpRequest | Securely fetches audio file from API | Audio request | Send audio | Podcast audio (player and download) |
| Send audio | n8n-nodes-base.respondToWebhook | Streams binary `.m4a` response to browser | Fetch audio | None | Podcast audio (player and download) |

---

### 4. Reproducing the Workflow from Scratch

Follow these sequential steps to rebuild the workflow manually in n8n:

1. **Create Credentials:**
   - Set up an **HTTP Header Auth** credential named `useapi.net API token` with Name: `Authorization`, Value: `Bearer <your-useapi-net-token>`.

2. **Trigger & Input Setup:**
   - Add a **Form Trigger** node named `Make a podcast`. Configure path `notebooklm-podcast`, add form fields (`Episode title` (text), `Links` (textarea), `Format` (dropdown), `Length` (dropdown), `Language` (dropdown), `Focus (optional)` (textarea), `Google account (optional)` (email)).
   - Add a **Code** node named `Form settings`. Paste the form normalization JavaScript code.
   - Add a **Schedule Trigger** node named `Every Monday at 7:00`. Set interval to weekly on Monday at 7 AM (disable by default).
   - Add a **Set** node named `Feed settings` to define RSS variables (`feedUrl`, `showName`, `lookbackDays`, `maxPosts`, etc.).
   - Add an **RSS Feed Read** node named `Read feed` connected to `Feed settings`.
   - Add a **Code** node named `Newest posts` to process RSS items.
   - Add an **If** node named `Any links?` evaluating `{{ $json.state.urls.length > 0 }}`. Connect both input branches (`Form settings` and `Newest posts`).

3. **Notebook & Source Ingestion:**
   - Add an **HTTP Request** node named `Create notebook` (POST `https://api.useapi.net/v1/gemini-notebook/notebooks`, authenticate with `useapi.net API token`, timeout 120s, retry on fail). Connect True branch of `Any links?`.
   - Add a **Code** node named `Notebook created`.
   - Add an **If** node named `Notebook ok?`.
   - Add an **HTTP Request** node named `Add sources` (POST `https://api.useapi.net/v1/gemini-notebook/sources`).
   - Add a **Code** node named `Sources added`.
   - Add a **Switch** node named `Sources route` with 3 outputs (`stop`, `each`, `ok`).
   - Configure the fallback loop: `One link each` (Code) -> `Add each link` (HTTP Request) -> `Links added` (Code) -> loops back to `Sources route`.
   - Configure the polling loop: `Wait 10s` (Wait node, 10s amount) -> `Read notebook` (HTTP Request GET `/notebooks/{notebook}`) -> `Source status` (Code) -> `Sources next` (Switch node).
   - Connect the `Sources` page form node (`n8n-nodes-base.form`, operation `page`) to the review route.

4. **Audio Overview Generation & Polling:**
   - Add a **Code** node named `Generate request` to format the generation body.
   - Add an **HTTP Request** node named `Generate Audio Overview` (POST `https://api.useapi.net/v1/gemini-notebook/artifacts`).
   - Add a **Code** node named `Generate check`.
   - Add a **Switch** node named `Generate route`.
   - Add a **Wait** node named `Retry in 15s` (15s) for generation retries.
   - Add a **Wait** node named `Wait 20s` (20s) for job status polling.
   - Add an **HTTP Request** node named `Check job` (GET `https://api.useapi.net/v1/gemini-notebook/jobs/{jobid}`).
   - Add a **Code** node named `Job status`.
   - Add a **Switch** node named `Job next` (`wait`, `stop`, `done`).

5. **Output & Result Handling:**
   - Add an **If** node named `From the form?` checking `{{ $json.state.source === 'form' }}`.
   - For form flow: `Result page` (Code) -> `Your podcast` (Form node, operation `page`) -> `Finish` (Form node, operation `completion`).
   - For scheduled/automated flow: `Download podcast` (HTTP Request GET file) -> `Episode` (Code node formatting binary data and JSON metadata for downstream integrations).

6. **Error Handling & Webhook Proxy:**
   - Route error paths to `Show a page?` (If node).
   - If form origin: `Stopped` (Form node, operation `page`).
   - If scheduled origin: `Stop with error` (Stop and Error node).
   - Create a separate webhook pipeline: **Webhook** node named `Podcast audio` (path `notebooklm-podcast-audio`, response mode `responseNode`) -> `Audio request` (Code validation) -> `Fetch audio` (HTTP Request GET file) -> `Send audio` (Respond to Webhook node with headers `Content-Type: audio/mp4`).

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Complete walkthrough with screenshots | [useapi.net Documentation Article](https://useapi.net/docs/articles/notebooklm-n8n-podcast?utm_source=n8n&utm_medium=referral&utm_campaign=notebooklm-podcast-app) |
| Source code and minimal workflow versions | [GitHub Repository (MIT)](https://github.com/useapi/notebooklm-api/tree/main/n8n) |
| API setup guide | [useapi.net Setup Documentation](https://useapi.net/docs/start-here/setup-useapi?utm_source=n8n&utm_medium=referral&utm_campaign=notebooklm-podcast-app) |
| Gemini Notebook (NotebookLM) integration guide | [useapi.net Gemini Notebook Setup](https://useapi.net/docs/start-here/setup-gemini-notebook?utm_source=n8n&utm_medium=referral&utm_campaign=notebooklm-podcast-app) |
| API reference | [useapi.net Gemini Notebook API Docs](https://useapi.net/docs/api-gemini-notebook-v1?utm_source=n8n&utm_medium=referral&utm_campaign=notebooklm-podcast-app) |