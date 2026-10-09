Audit AI and search bot access using robots.txt checks and Gmail reports

https://n8nworkflows.xyz/workflows/audit-ai-and-search-bot-access-using-robots-txt-checks-and-gmail-reports-20481


# Audit AI and search bot access using robots.txt checks and Gmail reports

### 1. Workflow Overview

This workflow automates a monthly audit to determine whether popular AI and search engine crawlers can successfully access a specific web page. It accomplishes this by comparing theoretical access permissions defined in a site’s `robots.txt` file against live HTTP responses collected using custom User-Agent strings. The results are aggregated, analyzed for discrepancies (such as firewall blocks or contradictory policies), and delivered via an email report using Gmail.

The workflow logic is divided into five distinct operational blocks:
- **1.1 Schedule & Configuration:** Initializes the workflow on a monthly basis and defines execution parameters including the target URL, the list of bots to audit, a control browser User-Agent, and pacing delays.
- **1.2 Robots.txt Ingestion & Interpretation:** Fetches the domain's `robots.txt` file and evaluates rules per bot using standard parsing logic (longest-path matching, wildcard fallbacks, and Allow/Disallow precedence).
- **1.3 Live Bot Probing & Browser Control:** Sequentially requests the target page using each bot's User-Agent with built-in rate-limiting delays, performs an additional request using a normal browser User-Agent as a control, and structures the metrics for comparison.
- **1.4 Verification & Aggregation:** Compares the theoretical `robots.txt` access permissions against live server HTTP status codes and response sizes, classifying each bot into a verdict category (e.g., `ALLOWED`, `BLOCKED`, `PARTIAL`, `CONTRADICTION`, `RATE_LIMITED`, `UNKNOWN`), sorts the findings, and aggregates them into a summary payload.
- **1.5 Reporting:** Formats the audit results and transmits the final report via Gmail to a designated recipient.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule & Configuration
- **Overview:** Establishes the recurring execution schedule and supplies all necessary global variables, including the target page URL, target crawler definitions, browser control settings, and request throttling intervals.
- **Nodes Involved:** `Monthly Audit Trigger`, `Set Audit Parameters`
- **Node Details:**
  - **Monthly Audit Trigger**
    - *Type & Technical Role:* `n8n-nodes-base.scheduleTrigger` (v1.2). Serves as the primary time-based entry point.
    - *Configuration Choices:* Configured to run on a monthly interval at hour 8.
    - *Input/Output:* No inputs; outputs a single execution trigger event to `Set Audit Parameters`.
    - *Edge Cases/Failure Types:* Minimal failure risk; dependent on n8n instance uptime.
  - **Set Audit Parameters**
    - *Type & Technical Role:* `n8n-nodes-base.set` (v3.4). Assigns static parameters used across subsequent nodes.
    - *Configuration Choices:* Sets `testUrl` to a target domain, defines an array of AI/search bots (`GPTBot`, `ClaudeBot`, `PerplexityBot`, `Googlebot`, etc.) complete with names, User-Agent strings, and descriptions, specifies a `browserUserAgent` control string, and defines `secondsBetweenRequests` as `10`.
    - *Input/Output:* Input from `Monthly Audit Trigger`; outputs configuration properties to `Fetch robots.txt`.
    - *Edge Cases/Failure Types:* Malformed JSON structures in the bot array will cause downstream evaluation errors.

#### 2.2 Robots.txt Ingestion & Interpretation
- **Overview:** Retrieves the target domain's `robots.txt` file and parses its contents to resolve whether the target path is permitted or restricted for each individual bot based on longest-match and wildcard fallback rules.
- **Nodes Involved:** `Fetch robots.txt`, `Normalise Response`, `Resolve robots.txt Rules`
- **Node Details:**
  - **Fetch robots.txt**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (v4.2). Performs an HTTP GET request to retrieve the domain's `robots.txt`.
    - *Configuration Choices:* Dynamically extracts the domain root from `testUrl` (`={{ $json.testUrl.split('/').slice(0, 3).join('/') }}/robots.txt`). Timeout set to 30000ms. Configured with `neverError: true`, `fullResponse: true`, and text response format. Sends the browser User-Agent in headers.
    - *Input/Output:* Input from `Set Audit Parameters`; outputs raw HTTP response and status code to `Normalise Response`.
    - *Edge Cases/Failure Types:* Network timeouts, DNS failures, or HTTP 404/5xx errors. Handled gracefully via `neverError: true` to prevent workflow crashes.
  - **Normalise Response**
    - *Type & Technical Role:* `n8n-nodes-base.code` (v2). JavaScript execution node that standardizes the raw HTTP response structure.
    - *Configuration Choices:* Maps `$json.data` and `__status` from the HTTP request into a unified object.
    - *Input/Output:* Input from `Fetch robots.txt`; outputs normalized data to `Resolve robots.txt Rules`.
    - *Edge Cases/Failure Types:* Null or undefined response bodies.
  - **Resolve robots.txt Rules**
    - *Type & Technical Role:* `n8n-nodes-base.code` (v2). Custom JavaScript parser implementing standard `robots.txt` evaluation logic.
    - *Configuration Choices:* Reads the bot array and `robots.txt` text. Evaluates specific user-agent groups, wildcard (`*`) fallbacks, longest-match precedence, and Allow/Disallow tie-breakers against the target path. Assigns preliminary statuses (`ALLOWED`, `BLOCKED`, or `UNKNOWN`).
    - *Input/Output:* Input from `Normalise Response`; outputs an array of objects (one per bot) containing theoretical permissions to `Request Page as Each Bot`.
    - *Edge Cases/Failure Types:* Malformed `robots.txt` syntax or missing group declarations default to open access (`ALLOWED`).

#### 2.3 Live Bot Probing & Browser Control
- **Overview:** Sequentially requests the target page using each bot's User-Agent with rate-limiting pauses, executes an additional control request using a standard browser User-Agent, and merges the resulting metrics.
- **Nodes Involved:** `Request Page as Each Bot`, `Attach Bot Responses`, `Is This the Browser Control Run?`, `Request Page as a Browser`, `Measure Browser Response`, `Merge Bots and Control`
- **Node Details:**
  - **Request Page as Each Bot**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (v4.2). Makes HTTP GET requests for each configured bot.
    - *Configuration Choices:* URL set to `={{ $json.testUrl }}`. Implements batching (`batchSize: 1`, `batchInterval: secondsBetweenRequests * 1000`) to pace requests and prevent security plugins from flagging the audit as a malicious scan. Sends bot-specific User-Agents and Accept headers. Timeout: 30000ms.
    - *Input/Output:* Input from `Resolve robots.txt Rules`; outputs paginated HTTP responses to `Attach Bot Responses`.
    - *Edge Cases/Failure Types:* Rate limiting (HTTP 429/503) due to strict server-side firewalls.
  - **Attach Bot Responses**
    - *Type & Technical Role:* `n8n-nodes-base.code` (v2). Merges bot metadata with response metrics and appends a browser control payload.
    - *Configuration Choices:* Extracts HTTP status (`__status`) and response length (`__length`) for each bot response, then appends a final control item using `browserUserAgent`.
    - *Input/Output:* Inputs from `Resolve robots.txt Rules` and `Request Page as Each Bot`; outputs combined bot and control items to `Is This the Browser Control Run?`.
    - *Edge Cases/Failure Types:* Array index mismatch if item counts fluctuate.
  - **Is This the Browser Control Run?**
    - *Type & Technical Role:* `n8n-nodes-base.if` (v2.2). Conditional branch router.
    - *Configuration Choices:* Evaluates whether `{{ $json.bot }}` equals `__control__`.
    - *Input/Output:* Input from `Attach Bot Responses`. Branch 0 (false) routes bot items to `Merge Bots and Control`; Branch 1 (true) routes the control item to `Request Page as a Browser`.
    - *Edge Cases/Failure Types:* Missing control flag.
  - **Request Page as a Browser**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (v4.2). Fetches the target page using a standard desktop browser User-Agent to establish a baseline response size and status.
    - *Configuration Choices:* Uses `browserUserAgent` and standard headers. Timeout: 30000ms.
    - *Input/Output:* Input from `Is This the Browser Control Run?` (true branch); outputs HTTP response to `Measure Browser Response`.
    - *Edge Cases/Failure Types:* Firewall blocks or timeouts on the control request.
  - **Measure Browser Response**
    - *Type & Technical Role:* `n8n-nodes-base.code` (v2). Formats the browser control response metrics.
    - *Configuration Choices:* Extracts status and content length, preserving the `__control__` identifier.
    - *Input/Output:* Input from `Request Page as a Browser`; outputs control metrics to `Merge Bots and Control`.
    - *Edge Cases/Failure Types:* Empty response bodies yielding zero-length metrics.
  - **Merge Bots and Control**
    - *Type & Technical Role:* `n8n-nodes-base.merge` (v3). Combines the array of bot evaluation items with the single browser control item.
    - *Configuration Choices:* Default merge operation combining input streams.
    - *Input/Output:* Inputs from `Is This the Browser Control Run?` (bot items) and `Measure Browser Response` (control item); outputs a unified dataset to `Compare Promise With Reality`.
    - *Edge Cases/Failure Types:* Asynchronous payload misalignment.

#### 2.4 Verification & Aggregation
- **Overview:** Analyzes discrepancies between theoretical `robots.txt` rules and actual server responses, assigns categorical verdicts, sorts the results by severity, and compiles a summary payload.
- **Nodes Involved:** `Compare Promise With Reality`, `Aggregate Findings`
- **Node Details:**
  - **Compare Promise With Reality**
    - *Type & Technical Role:* `n8n-nodes-base.code` (v2). JavaScript analysis engine.
    - *Configuration Choices:* Evaluates HTTP status codes, response content lengths against the browser control length (flagging content < 50% as `PARTIAL`), rate limiting (HTTP 429/503 as `RATE_LIMITED`), and contradictions where `robots.txt` permits access but the server returns error statuses (classified as `CONTRADICTION`). Sorts results by verdict severity and builds summary counts.
    - *Input/Output:* Input from `Merge Bots and Control`; outputs evaluated findings to `Aggregate Findings`.
    - *Edge Cases/Failure Types:* Missing control baseline affecting byte-comparison ratios.
  - **Aggregate Findings**
    - *Type & Technical Role:* `n8n-nodes-base.aggregate` (v1). Consolidates all individual bot evaluation items into a single array structure.
    - *Configuration Choices:* Aggregates all item data into a single list.
    - *Input/Output:* Input from `Compare Promise With Reality`; outputs aggregated object to `Email the Audit`.
    - *Edge Cases/Failure Types:* Empty item sets if upstream nodes fail.

#### 2.5 Reporting
- **Overview:** Formats the compiled audit results into a readable text report and transmits it via email.
- **Nodes Involved:** `Email the Audit`
- **Node Details:**
  - **Email the Audit**
    - *Type & Technical Role:* `n8n-nodes-base.gmail` (v2.1). Sends an email via the Gmail API.
    - *Configuration Choices:* Recipient set to `you@example.com`. Subject dynamically reflects the target URL (`=AI bot access audit - {{ $json.data[0].testUrl }}`). Message body maps summary metrics and detailed per-bot explanations using expression templating. Uses Gmail OAuth2 credentials.
    - *Input/Output:* Input from `Aggregate Findings`; execution terminal node.
    - *Edge Cases/Failure Types:* Invalid OAuth2 credentials, token expiration, or incorrect recipient addresses.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Monthly Audit Trigger` | `n8n-nodes-base.scheduleTrigger` | Triggers the audit monthly | None | `Set Audit Parameters` | ## Audit which AI and search bots can actually reach your site<br><br>### How it works<br><br>Every site owner has an opinion about AI crawlers. Very few know what their site actually does. This workflow checks both halves of the question for one page:<br><br>1. It reads your `robots.txt` and works out, per bot, whether that page is allowed - following the real rules: the most specific path wins, `Allow` beats `Disallow` on a tie, and a bot without its own group falls back to `User-agent: *`.<br>2. It then requests the page once as each bot and once as a normal browser, and compares the answers.<br><br>That second step is the one people skip, and it is where the surprises live:<br><br>- **CONTRADICTION** - your `robots.txt` welcomes the bot, but the server answers 403. A firewall, WAF or hosting bot rule is turning it away regardless of what you wrote. You think you are visible in ChatGPT or Perplexity; you are not.<br>- **PARTIAL** - the bot gets a page, but far smaller than the browser version. Usually a challenge page or a stripped render.<br>- **BLOCKED** - a deliberate block. Fine, as long as you meant it.<br>- **ALLOWED** - full access.<br>- **RATE_LIMITED** - your own firewall throttled the audit. Not a verdict about bots; run it again more slowly.<br><br>The bot list covers two different jobs, because blocking them is not the same decision. Training bots (GPTBot, ClaudeBot, Google-Extended, CCBot) feed model training. Search bots (OAI-SearchBot, ChatGPT-User, PerplexityBot, Claude-SearchBot) fetch pages to cite in AI answers. Blocking the first group is a licensing choice. Blocking the second group removes you from AI search results - and plenty of sites do it by accident, in one line.<br><br>### Setup steps<br><br>- In the **Set Audit Parameters** node, set `testUrl` to the page you care about, for example your homepage or a key service page. Everything else works out of the box.<br>- Connect a Gmail credential and set the recipient in the **Email the Audit** node, or swap that node for Slack, Telegram or a Google Sheet.<br>- Adjust the schedule trigger. Monthly is usually enough, since this only changes when someone edits robots.txt or turns on a new firewall rule.<br>- Optional: lower `secondsBetweenRequests` if you know the site has no rate limiting, or raise it if the first run comes back rate limited.<br><br>### Good to know<br><br>No credentials are needed for the audit itself - only for delivering the report.<br><br>The audit sends one request per bot to the same page. That is a tiny amount of traffic, but a dozen requests from one IP with a different bot user agent each time is exactly what a security plugin reads as a scan. So requests are throttled to one every `secondsBetweenRequests` (10 by default, so a full audit takes about two minutes).<br><br>If you still get **RATE_LIMITED** rows, your own firewall answered 429 or 503 and those rows say nothing about bot access. Three ways out, best first:<br><br>- Allowlist the IP your n8n runs from, so the audit measures how the site treats bots instead of how it treats the audit.<br>- Raise `secondsBetweenRequests` to 20 or 30 and run it again. Note that firewalls usually keep blocking for several minutes after a trip, so wait before retrying.<br>- Shorten the `bots` list. If you only care about being citable in AI answers, the six search bots are enough.<br><br>Run this against a site you own, or with the owner's agreement.<br><br>### Customization<br><br>Add or remove bots in the `bots` list in **Set Audit Parameters** - each entry needs a name, a user agent string and a short purpose. To audit several pages, put the URLs in a list and loop, or duplicate the schedule trigger per site. |
| `Set Audit Parameters` | `n8n-nodes-base.set` | Defines test URL, bot array, browser User-Agent, and delay | `Monthly Audit Trigger` | `Fetch robots.txt` | ## Schedule and settings<br><br>Runs monthly and defines the page to audit plus the list of bots to test. This is the only node you need to edit. |
| `Fetch robots.txt` | `n8n-nodes-base.httpRequest` | Fetches the site's robots.txt file | `Set Audit Parameters` | `Normalise Response` | ## Read and interpret robots.txt<br><br>Fetches robots.txt and resolves, per bot, whether the target page is allowed - longest path match wins, with a fallback to the wildcard group. |
| `Normalise Response` | `n8n-nodes-base.code` | Normalizes HTTP response data | `Fetch robots.txt` | `Resolve robots.txt Rules` | ## Read and interpret robots.txt<br><br>Fetches robots.txt and resolves, per bot, whether the target page is allowed - longest path match wins, with a fallback to the wildcard group. |
| `Resolve robots.txt Rules` | `n8n-nodes-base.code` | Parses robots.txt rules and evaluates initial bot permissions | `Normalise Response` | `Request Page as Each Bot` | ## Read and interpret robots.txt<br><br>Fetches robots.txt and resolves, per bot, whether the target page is allowed - longest path match wins, with a fallback to the wildcard group. |
| `Request Page as Each Bot` | `n8n-nodes-base.httpRequest` | Sequentially requests the target page using each bot's User-Agent | `Resolve robots.txt Rules` | `Attach Bot Responses` | ## Knock on the door as each bot<br><br>Requests the page once per bot user agent, plus once as a normal browser. That control run is what the bot responses are compared against.<br><br>This step is deliberately slow: one request every `secondsBetweenRequests` (10 s by default, about 2 minutes in total), so a security plugin does not mistake the audit for a scan. |
| `Attach Bot Responses` | `n8n-nodes-base.code` | Attaches bot metadata to response metrics and adds control placeholder | `Request Page as Each Bot` | `Is This the Browser Control Run?` | ## Knock on the door as each bot<br><br>Requests the page once per bot user agent, plus once as a normal browser. That control run is what the bot responses are compared against.<br><br>This step is deliberately slow: one request every `secondsBetweenRequests` (10 s by default, about 2 minutes in total), so a security plugin does not mistake the audit for a scan. |
| `Is This the Browser Control Run?` | `n8n-nodes-base.if` | Branches execution between bot items and the browser control item | `Attach Bot Responses` | `Request Page as a Browser`, `Merge Bots and Control` | ## Knock on the door as each bot<br><br>Requests the page once per bot user agent, plus once as a normal browser. That control run is what the bot responses are compared against.<br><br>This step is deliberately slow: one request every `secondsBetweenRequests` (10 s by default, about 2 minutes in total), so a security plugin does not mistake the audit for a scan. |
| `Request Page as a Browser` | `n8n-nodes-base.httpRequest` | Requests the target page using a standard browser User-Agent | `Is This the Browser Control Run?` | `Measure Browser Response` | ## Knock on the door as each bot<br><br>Requests the page once per bot user agent, plus once as a normal browser. That control run is what the bot responses are compared against.<br><br>This step is deliberately slow: one request every `secondsBetweenRequests` (10 s by default, about 2 minutes in total), so a security plugin does not mistake the audit for a scan. |
| `Measure Browser Response` | `n8n-nodes-base.code` | Measures browser control response size and status | `Request Page as a Browser` | `Merge Bots and Control` | ## Knock on the door as each bot<br><br>Requests the page once per bot user agent, plus once as a normal browser. That control run is what the bot responses are compared against.<br><br>This step is deliberately slow: one request every `secondsBetweenRequests` (10 s by default, about 2 minutes in total), so a security plugin does not mistake the audit for a scan. |
| `Merge Bots and Control` | `n8n-nodes-base.merge` | Merges bot results with the browser control result | `Is This the Browser Control Run?`, `Measure Browser Response` | `Compare Promise With Reality` | ## Knock on the door as each bot<br><br>Requests the page once per bot user agent, plus once as a normal browser. That control run is what the bot responses are compared against.<br><br>This step is deliberately slow: one request every `secondsBetweenRequests` (10 s by default, about 2 minutes in total), so a security plugin does not mistake the audit for a scan. |
| `Compare Promise With Reality` | `n8n-nodes-base.code` | Compares robots.txt rules against live server responses and assigns verdicts | `Merge Bots and Control` | `Aggregate Findings` | ## Compare and report<br><br>Flags contradictions between what robots.txt permits and what the server does, then sends the findings by email. |
| `Aggregate Findings` | `n8n-nodes-base.aggregate` | Aggregates all evaluated bot findings into a single array | `Compare Promise With Reality` | `Email the Audit` | ## Compare and report<br><br>Flags contradictions between what robots.txt permits and what the server does, then sends the findings by email. |
| `Email the Audit` | `n8n-nodes-base.gmail` | Sends the compiled audit report via email | `Aggregate Findings` | None | ## Compare and report<br><br>Flags contradictions between what robots.txt permits and what the server does, then sends the findings by email. |

---

### 4. Reproducing the Workflow from Scratch

Follow these steps to rebuild the workflow manually in n8n:

1. **Create the Trigger Node:**
   - Add a **Schedule Trigger** node (`n8n-nodes-base.scheduleTrigger`).
   - Configure the interval rule to run monthly at hour `8`.
   - Name it `Monthly Audit Trigger`.

2. **Add Parameters Node:**
   - Add a **Set** node (`n8n-nodes-base.set`).
   - Name it `Set Audit Parameters`.
   - Create the following string/number/array assignments:
     - `testUrl`: `https://example.com/` (string)
     - `bots`: JSON array containing bot definitions (`GPTBot`, `OAI-SearchBot`, `ChatGPT-User`, `ClaudeBot`, `Claude-SearchBot`, `Claude-User`, `PerplexityBot`, `Google-Extended`, `Googlebot`, `Bingbot`, `CCBot`, `Applebot-Extended`) with properties `name`, `ua`, and `purpose`.
     - `browserUserAgent`: `Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/128.0.0.0 Safari/537.36` (string)
     - `secondsBetweenRequests`: `10` (number)
   - Connect `Monthly Audit Trigger` output to `Set Audit Parameters`.

3. **Fetch Robots.txt:**
   - Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`).
   - Name it `Fetch robots.txt`.
   - Set Method to `GET`, URL to `={{ $json.testUrl.split('/').slice(0, 3).join('/') }}/robots.txt`.
   - In Options, set Timeout to `30000`, Response Format to `Text`, Full Response to `True`, and Never Error to `True`.
   - Add a Request Header: `User-Agent` = `={{ $json.browserUserAgent }}`.
   - Connect `Set Audit Parameters` to `Fetch robots.txt`.

4. **Normalise HTTP Response:**
   - Add a **Code** node (`n8n-nodes-base.code`).
   - Name it `Normalise Response`.
   - Set Mode to JavaScript and paste:
     ```javascript
     return [{ json: { data: $json.data, __status: $json.statusCode } }];
     ```
   - Connect `Fetch robots.txt` to `Normalise Response`.

5. **Parse Robots.txt Rules:**
   - Add a **Code** node (`n8n-nodes-base.code`).
   - Name it `Resolve robots.txt Rules`.
   - Paste the JavaScript parser code that extracts groups, evaluates user-agents, applies longest-match rules, and maps initial verdicts per bot.
   - Connect `Normalise Response` to `Resolve robots.txt Rules`.

6. **Probe Page as Each Bot:**
   - Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`).
   - Name it `Request Page as Each Bot`.
   - Set Method to `GET`, URL to `={{ $json.testUrl }}`.
   - Configure Batching: Batch Size = `1`, Batch Interval = `={{ $json.secondsBetweenRequests * 1000 }}`.
   - Set Options: Timeout = `30000`, Response Format = `Text`, Full Response = `True`, Never Error = `True`.
   - Add Request Headers: `User-Agent` = `={{ $json.userAgent }}`, `Accept` = `text/html,application/xhtml+xml`.
   - Connect `Resolve robots.txt Rules` to `Request Page as Each Bot`.

7. **Attach Bot Responses:**
   - Add a **Code** node (`n8n-nodes-base.code`).
   - Name it `Attach Bot Responses`.
   - Paste JavaScript code to map bot metadata with response status (`__status`) and length (`__length`), appending a control run object (`__control__`) at the end.
   - Connect `Request Page as Each Bot` to `Attach Bot Responses`.

8. **Conditional Control Routing:**
   - Add an **If** node (`n8n-nodes-base.if`).
   - Name it `Is This the Browser Control Run?`.
   - Set Condition: Left Value `={{ $json.bot }}` equals `__control__`.
   - Connect `Attach Bot Responses` to `Is This the Browser Control Run?`.

9. **Request Page as a Browser (Control):**
   - Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`).
   - Name it `Request Page as a Browser`.
   - Configure identically to the bot request node, but using the browser control headers.
   - Connect the `true` (index 0) output of `Is This the Browser Control Run?` to `Request Page as a Browser`.

10. **Measure Browser Response:**
    - Add a **Code** node (`n8n-nodes-base.code`).
    - Name it `Measure Browser Response`.
    - Paste code to format the control response size and status.
    - Connect `Request Page as a Browser` to `Measure Browser Response`.

11. **Merge Bots and Control:**
    - Add a **Merge** node (`n8n-nodes-base.merge`).
    - Name it `Merge Bots and Control`.
    - Connect input 1 (Index 0) from the `false` output of `Is This the Browser Control Run?`.
    - Connect input 2 (Index 1) from `Measure Browser Response`.

12. **Compare Promise With Reality:**
    - Add a **Code** node (`n8n-nodes-base.code`).
    - Name it `Compare Promise With Reality`.
    - Paste JavaScript analysis code to evaluate rules vs. live responses, detect rate limits (`RATE_LIMITED`), contradictions (`CONTRADICTION`), partial blocks (`PARTIAL`), and standard permissions, sorting by severity.
    - Connect `Merge Bots and Control` to `Compare Promise With Reality`.

13. **Aggregate Findings:**
    - Add an **Aggregate** node (`n8n-nodes-base.aggregate`).
    - Name it `Aggregate Findings`.
    - Set Aggregate to `Aggregate All Item Data`.
    - Connect `Compare Promise With Reality` to `Aggregate Findings`.

14. **Email Report:**
    - Add a **Gmail** node (`n8n-nodes-base.gmail`).
    - Name it `Email the Audit`.
    - Configure resource/operation to `Message: Send`.
    - Set Recipient (`sendTo`) to your target email address (e.g., `you@example.com`).
    - Set Subject to `=AI bot access audit - {{ $json.data[0].testUrl }}`.
    - Set Message body expression to format summary counts and per-bot explanations.
    - Configure and select a valid **Gmail OAuth2** credential.
    - Connect `Aggregate Findings` to `Email the Audit`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Audit scope and crawler definitions | Differentiates between model training bots (`GPTBot`, `ClaudeBot`, `Google-Extended`, `CCBot`) and search indexation/citation bots (`OAI-SearchBot`, `ChatGPT-User`, `PerplexityBot`, `Claude-SearchBot`). |
| Rate limiting and firewall mitigation | Requests are paced (`secondsBetweenRequests`, default 10s) to prevent security plugins from identifying the audit workflow as a vulnerability scan. If `RATE_LIMITED` results occur, allowlist the n8n runner IP or increase the delay. |
| Credentials required | No credentials are required for the HTTP request audit phase; a Gmail OAuth2 credential is required exclusively for delivering the report. |