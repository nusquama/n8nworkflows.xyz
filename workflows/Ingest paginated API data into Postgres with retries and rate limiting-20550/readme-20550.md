Ingest paginated API data into Postgres with retries and rate limiting

https://n8nworkflows.xyz/workflows/ingest-paginated-api-data-into-postgres-with-retries-and-rate-limiting-20550


# Ingest paginated API data into Postgres with retries and rate limiting

### 1. Workflow Overview

This workflow is a universal, production-ready pagination engine designed to extract data from any paginated REST API, normalize the records according to a predefined mapping schema, and securely upsert them into a PostgreSQL database. It features built-in API retry logic, rate-limiting delays between pages, safety limits to prevent runaway loops, and flexible pagination strategies (page-based, offset-based, cursor-based, or link-based).

The logic is organized into three distinct operational blocks:
- **1.1 Initialization & API Request:** Manually triggers the workflow, establishes global configuration settings (such as endpoint URLs, credentials, pagination parameters, mapping rules, and safety thresholds), and executes the initial HTTP request with retry handling.
- **1.2 Response Processing & Database Upsert:** Parses the incoming API response payload, normalizes extracted records, checks whether records are present, and performs a batch `INSERT ... ON CONFLICT` operation directly into PostgreSQL.
- **1.3 Pagination Control & Finalization:** Calculates the next pagination token or counter, evaluates whether further pages exist within the safety limits, throttles requests via a rate-limit delay loop, and compiles a final ingestion summary report.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Initialization & API Request
- **Overview:** This block kicks off the workflow on demand, constructs the configuration parameters and initial pagination state in JavaScript, and executes the first API call with automated retry behavior.
- **Nodes Involved:** 
  - `Manual Trigger`
  - `Code - Initialize Config`
  - `HTTP Request - Fetch Page`

- **Node Details:**
  - **Manual Trigger**
    - *Type and Technical Role:* `n8n-nodes-base.manualTrigger` — Acts as the manual entry point to start the execution on demand.
    - *Configuration Choices:* Standard manual trigger with no required user inputs.
    - *Key Expressions or Variables:* None.
    - *Input and Output Connections:* Input: None; Output: Connects to `Code - Initialize Config`.
    - *Version-Specific Requirements:* Version 1.
    - *Edge Cases or Potential Failure Types:* None.
  
  - **Code - Initialize Config**
    - *Type and Technical Role:* `n8n-nodes-base.code` — JavaScript execution node that sets up structural configurations (API endpoints, headers, query parameters, field mappings, unique database keys, and safety bounds like `maxPages` and `maxRetries`) and initializes tracking state variables (`page`, `offset`, `cursor`, `pagesDone`, `totalRecords`).
    - *Configuration Choices:* Houses a large configuration object (`CONFIG`) that must be adjusted per target API, alongside initial state and request structures.
    - *Key Expressions or Variables:* Uses raw JavaScript objects for configuration properties.
    - *Input and Output Connections:* Input: `Manual Trigger`; Output: Connects to `HTTP Request - Fetch Page`.
    - *Version-Specific Requirements:* Version 2.
    - *Edge Cases or Potential Failure Types:* JavaScript syntax errors if the configuration block is misedited; missing authentication tokens.

  - **HTTP Request - Fetch Page**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` — Performs the HTTP call to the API endpoint utilizing dynamic query parameters and headers passed from the configuration/state.
    - *Configuration Choices:* Configured with 3 maximum tries (`maxTries: 3`), a 2000ms delay between tries (`waitBetweenTries: 2000`), a 30-second timeout, `continueOnFail: true`, and `neverError: true` to catch status codes gracefully in the subsequent code node.
    - *Key Expressions or Variables:* 
      - URL: `={{ $json.request.url || $json.config.baseUrl }}`
      - Method: `={{ $json.config.method || 'GET' }}`
      - Query Parameters: Dynamic evaluation of `limitParam` and `pageParam` based on the state.
      - Header Parameters: `Authorization` pulled from `={{ $json.config.headers.Authorization }}`.
    - *Input and Output Connections:* Input: `Code - Initialize Config` (and looping back from `Wait - Rate Limit Delay`); Output: Connects to `Code - Parse & Normalize`.
    - *Version-Specific Requirements:* Version 4.2.
    - *Edge Cases or Potential Failure Types:* Network dropouts, HTTP 4xx/5xx errors (mitigated by `continueOnFail` and built-in retries), and invalid credential headers.

---

#### Block 1.2: Response Processing & Database Upsert
- **Overview:** This block parses the raw HTTP response, extracts and normalizes arrays of data according to custom field mappings, validates if records exist, and upserts them into a target PostgreSQL database table.
- **Nodes Involved:**
  - `Code - Parse & Normalize`
  - `IF - Has Records?`
  - `Postgres - Upsert Records`

- **Node Details:**
  - **Code - Parse & Normalize**
    - *Type and Technical Role:* `n8n-nodes-base.code` — JavaScript parser that safely evaluates JSON responses, navigates nested JSON paths (`dataPath`), maps fields to target database columns (`fieldMap`), updates iteration counts, and calculates the next page pointer (`page`, `offset`, `cursor`, or `link`).
    - *Configuration Choices:* Embedded JavaScript parsing logic with safety checks for stringified bodies and nested property resolution.
    - *Key Expressions or Variables:* References upstream state using `$(name).item.json` or `$(name).first().json`.
    - *Input and Output Connections:* Input: `HTTP Request - Fetch Page`; Output: Connects to `IF - Has Records?`.
    - *Version-Specific Requirements:* Version 2.
    - *Edge Cases or Potential Failure Types:* Malformed JSON responses falling back to empty bodies; invalid nested path lookups returning null values.

  - **IF - Has Records?**
    - *Type and Technical Role:* `n8n-nodes-base.if` — Conditional branch that checks if the extracted record array contains one or more items.
    - *Configuration Choices:* Version 2 strict type validation checking if `{{ $json.records.length }}` is greater than `0`.
    - *Key Expressions or Variables:* `={{ $json.records.length }}`
    - *Input and Output Connections:* Input: `Code - Parse & Normalize`; Output: True branch connects to `Postgres - Upsert Records`, False branch connects to `Code - Prepare Next Page`.
    - *Version-Specific Requirements:* Version 2.2.
    - *Edge Cases or Potential Failure Types:* Unexpected non-array types handled gracefully upstream.

  - **Postgres - Upsert Records**
    - *Type and Technical Role:* `n8n-nodes-base.postgres` — Executes a dynamic SQL query that performs a batch insert with conflict handling (`INSERT ... ON CONFLICT DO UPDATE`).
    - *Configuration Choices:* Executes a raw query (`operation: 'executeQuery'`) generated dynamically via JavaScript template strings. Retains `continueOnFail: true`.
    - *Key Expressionsor Variables:* Evaluates dynamic SQL statements mapping table columns, record values, unique constraint keys, and excluded update attributes.
    - *Input and Output Connections:* Input: `IF - Has Records?` (True branch); Output: Connects to `Code - Prepare Next Page`.
    - *Version-Specific Requirements:* Version 2.5. Requires active PostgreSQL credentials.
    - *Edge Cases or Potential Failure Types:* Database schema mismatches, missing unique constraint keys in PostgreSQL causing conflict statement failures, or connection timeouts on massive payloads.

---

#### Block 1.3: Pagination Control & Finalization
- **Overview:** This block assesses the continuation criteria, decides whether to loop back to fetch another page or terminate the process, introduces a rate-limiting pause if looping, and produces a final summary log.
- **Nodes Involved:**
  - `Code - Prepare Next Page`
  - `IF - More Pages?`
  - `Wait - Rate Limit Delay`
  - `Code - Finalize Job`
  - `NoOp - End`

- **Node Details:**
  - **Code - Prepare Next Page**
    - *Type and Technical Role:* `n8n-nodes-base.code` — Consolidates state variables from either the upsert branch or the empty-record bypass branch and sets the `shouldContinue` flag.
    - *Configuration Choices:* JavaScript helper function evaluating upstream state objects.
    - *Input and Output Connections:* Input: Receives inputs from both `Postgres - Upsert Records` and `IF - Has Records?` (False branch); Output: Connects to `IF - More Pages?`.
    - *Version-Specific Requirements:* Version 2.
    - *Edge Cases or Potential Failure Types:* Reference resolution failures if upstream nodes are bypassed incorrectly.

  - **IF - More Pages?**
    - *Type and Technical Role:* `n8n-nodes-base.if` — Evaluates whether the pagination engine should fetch another page based on data availability and the `maxPages` safety limit.
    - *Configuration Choices:* Version 2 strict type evaluation checking if `={{ $json.shouldContinue }}` is true.
    - *Key Expressions or Variables:* `={{ $json.shouldContinue }}`
    - *Input and Output Connections:* Input: `Code - Prepare Next Page`; Output: True branch connects to `Wait - Rate Limit Delay`, False branch connects to `Code - Finalize Job`.
    - *Version-Specific Requirements:* Version 2.2.
    - *Edge Cases or Potential Failure Types:* Infinite loops averted strictly via the `maxPages` counter check implemented in the normalization script.

  - **Wait - Rate Limit Delay**
    - *Type and Technical Role:* `n8n-nodes-base.wait` — Introduces an intentional pause between consecutive API requests to respect upstream rate limits.
    - *Configuration Choices:* Configured with a 1-second delay amount (can be scaled upward depending on API limits).
    - *Input and Output Connections:* Input: `IF - More Pages?` (True branch); Output: Loops back to `HTTP Request - Fetch Page`.
    - *Version-Specific Requirements:* Version 1.1. Uses an internal webhook ID (`4f9be2ee-7a9c-4a6a-b95a-fe8c0d43e138`).
    - *Edge Cases or Potential Failure Types:* Execution time inflation on large datasets spanning hundreds of pages.

  - **Code - Finalize Job**
    - *Type and Technical Role:* `n8n-nodes-base.code` — Aggregates final metadata including total pages processed, total records ingested, completion timestamps, and status messages.
    - *Configuration Choices:* JavaScript execution returning a clean JSON summary object.
    - *Input and Output Connections:* Input: `IF - More Pages?` (False branch); Output: Connects to `NoOp - End`.
    - *Version-Specific Requirements:* Version 2.
    - *Edge Cases or Potential Failure Types:* Missing state attributes defaulting to zero counts.

  - **NoOp - End**
    - *Type and Technical Role:* `n8n-nodes-base.noOp` — Acts as a clean terminal node for the workflow execution path.
    - *Configuration Choices:* Standard no-operation node.
    - *Input and Output Connections:* Input: `Code - Finalize Job`; Output: None.
    - *Version-Specific Requirements:* Version 1.
    - *Edge Cases or Potential Failure Types:* None.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Sticky Note - Overview` | `n8n-nodes-base.stickyNote` | Overview documentation container | None | None | ## Universal API → Database Ingestion<br><br>Fetches any paginated API, normalizes records, upserts them into Postgres, and automatically continues until no more pages remain (with retries + rate-limit waits).<br><br>### All nodes (1-line each)<br>1. Manual Trigger – Starts the ingestion job on demand.<br>2. Code - Initialize Config – Loads API URL, headers, pagination type, retry policy and field mapping.<br>3. HTTP Request - Fetch Page – Calls the API with current page/cursor and built-in retries.<br>4. Code - Parse & Normalize – Extracts records from the response and calculates the next page token/page.<br>5. IF - Has Records? – Continues only when the current page actually returned data.<br>6. Postgres - Upsert Records – Inserts or updates the batch into the target table with conflict handling.<br>7. Code - Prepare Next Page – Updates cursor/page counter and checks against max-pages limit.<br>8. IF - More Pages? – Decides whether another page should be fetched.<br>9. Wait - Rate Limit Delay – Pauses between pages to respect API rate limits.<br>10. Code - Finalize Job – Builds the final summary (total records, pages, status).<br>11. NoOp - End – Terminates the workflow cleanly after the last page. |
| `Sticky Note - Config & Fetch` | `n8n-nodes-base.stickyNote` | Block 1 documentation container | None | None | ## 1. Config & First Fetch<br><br>Manual Trigger – Starts the ingestion job on demand.<br>Code - Initialize Config – Loads API URL, headers, pagination type, retry policy and field mapping.<br>HTTP Request - Fetch Page – Calls the API with current page/cursor and built-in retries. |
| `Sticky Note - Process & Store` | `n8n-nodes-base.stickyNote` | Block 2 documentation container | None | None | ## 2. Process & Store<br><br>Code - Parse & Normalize – Extracts records from the response and calculates the next page token/page.<br>IF - Has Records? – Continues only when the current page actually returned data.<br>Postgres - Upsert Records – Inserts or updates the batch into the target table with conflict handling. |
| `Sticky Note - Pagination Control` | `n8n-nodes-base.stickyNote` | Block 3 documentation container | None | None | ## 3. Pagination & Finish<br><br>Code - Prepare Next Page – Updates cursor/page counter and checks against max-pages limit.<br>IF - More Pages? – Decides whether another page should be fetched.<br>Wait - Rate Limit Delay – Pauses between pages to respect API rate limits.<br>Code - Finalize Job – Builds the final summary (total records, pages, status).<br>NoOp - End – Terminates the workflow cleanly after the last page. |
| `Manual Trigger` | `n8n-nodes-base.manualTrigger` | Starts the ingestion job on demand. | None | `Code - Initialize Config` | ## 1. Config & First Fetch<br><br>Manual Trigger – Starts the ingestion job on demand.<br>Code - Initialize Config – Loads API URL, headers, pagination type, retry policy and field mapping.<br>HTTP Request - Fetch Page – Calls the API with current page/cursor and built-in retries. |
| `Code - Initialize Config` | `n8n-nodes-base.code` | Loads API URL, headers, pagination type, retry policy and field mapping. | `Manual Trigger` | `HTTP Request - Fetch Page` | ## 1. Config & First Fetch<br><br>Manual Trigger – Starts the ingestion job on demand.<br>Code - Initialize Config – Loads API URL, headers, pagination type, retry policy and field mapping.<br>HTTP Request - Fetch Page – Calls the API with current page/cursor and built-in retries. |
| `HTTP Request - Fetch Page` | `n8n-nodes-base.httpRequest` | Calls the API with current page/cursor and built-in retries. | `Code - Initialize Config`, `Wait - Rate Limit Delay` | `Code - Parse & Normalize` | ## 1. Config & First Fetch<br><br>Manual Trigger – Starts the ingestion job on demand.<br>Code - Initialize Config – Loads API URL, headers, pagination type, retry policy and field mapping.<br>HTTP Request - Fetch Page – Calls the API with current page/cursor and built-in retries. |
| `Code - Parse & Normalize` | `n8n-nodes-base.code` | Extracts records from the response and calculates the next page token/page. | `HTTP Request - Fetch Page` | `IF - Has Records?` | ## 2. Process & Store<br><br>Code - Parse & Normalize – Extracts records from the response and calculates the next page token/page.<br>IF - Has Records? – Continues only when the current page actually returned data.<br>Postgres - Upsert Records – Inserts or updates the batch into the target table with conflict handling. |
| `IF - Has Records?` | `n8n-nodes-base.if` | Continues only when the current page actually returned data. | `Code - Parse & Normalize` | `Postgres - Upsert Records`, `Code - Prepare Next Page` | ## 2. Process & Store<br><br>Code - Parse & Normalize – Extracts records from the response and calculates the next page token/page.<br>IF - Has Records? – Continues only when the current page actually returned data.<br>Postgres - Upsert Records – Inserts or updates the batch into the target table with conflict handling. |
| `Postgres - Upsert Records` | `n8n-nodes-base.postgres` | Inserts or updates the batch into the target table with conflict handling. | `IF - Has Records?` | `Code - Prepare Next Page` | ## 2. Process & Store<br><br>Code - Parse & Normalize – Extracts records from the response and calculates the next page token/page.<br>IF - Has Records? – Continues only when the current page actually returned data.<br>Postgres - Upsert Records – Inserts or updates the batch into the target table with conflict handling. |
| `Code - Prepare Next Page` | `n8n-nodes-base.code` | Updates cursor/page counter and checks against max-pages limit. | `IF - Has Records?`, `Postgres - Upsert Records` | `IF - More Pages?` | ## 3. Pagination & Finish<br><br>Code - Prepare Next Page – Updates cursor/page counter and checks against max-pages limit.<br>IF - More Pages? – Decides whether another page should be fetched.<br>Wait - Rate Limit Delay – Pauses between pages to respect API rate limits.<br>Code - Finalize Job – Builds the final summary (total records, pages, status).<br>NoOp - End – Terminates the workflow cleanly after the last page. |
| `IF - More Pages?` | `n8n-nodes-base.if` | Decides whether another page should be fetched. | `Code - Prepare Next Page` | `Wait - Rate Limit Delay`, `Code - Finalize Job` | ## 3. Pagination & Finish<br><br>Code - Prepare Next Page – Updates cursor/page counter and checks against max-pages limit.<br>IF - More Pages? – Decides whether another page should be fetched.<br>Wait - Rate Limit Delay – Pauses between pages to respect API rate limits.<br>Code - Finalize Job – Builds the final summary (total records, pages, status).<br>NoOp - End – Terminates the workflow cleanly after the last page. |
| `Wait - Rate Limit Delay` | `n8n-nodes-base.wait` | Pauses between pages to respect API rate limits. | `IF - More Pages?` | `HTTP Request - Fetch Page` | ## 3. Pagination & Finish<br><br>Code - Prepare Next Page – Updates cursor/page counter and checks against max-pages limit.<br>IF - More Pages? – Decides whether another page should be fetched.<br>Wait - Rate Limit Delay – Pauses between pages to respect API rate limits.<br>Code - Finalize Job – Builds the final summary (total records, pages, status).<br>NoOp - End – Terminates the workflow cleanly after the last page. |
| `Code - Finalize Job` | `n8n-nodes-base.code` | Builds the final summary (total records, pages, status). | `IF - More Pages?` | `NoOp - End` | ## 3. Pagination & Finish<br><br>Code - Prepare Next Page – Updates cursor/page counter and checks against max-pages limit.<br>IF - More Pages? – Decides whether another page should be fetched.<br>Wait - Rate Limit Delay – Pauses between pages to respect API rate limits.<br>Code - Finalize Job – Builds the final summary (total records, pages, status).<br>NoOp - End – Terminates the workflow cleanly after the last page. |
| `NoOp - End` | `n8n-nodes-base.noOp` | Terminates the workflow cleanly after the last page. | `Code - Finalize Job` | None | ## 3. Pagination & Finish<br><br>Code - Prepare Next Page – Updates cursor/page counter and checks against max-pages limit.<br>IF - More Pages? – Decides whether another page should be fetched.<br>Wait - Rate Limit Delay – Pauses between pages to respect API rate limits.<br>Code - Finalize Job – Builds the final summary (total records, pages, status).<br>NoOp - End – Terminates the workflow cleanly after the last page. |

---

### 4. Reproducing the Workflow from Scratch

Follow this step-by-step procedure to rebuild the workflow manually in n8n:

1. **Create the Manual Trigger Node**
   - Add a **Manual Trigger** node (`n8n-nodes-base.manualTrigger`). Leave parameters at default.
2. **Create the Initialization Code Node**
   - Add a **Code** node (`n8n-nodes-base.code`) named `Code - Initialize Config`.
   - Set mode to JavaScript and populate the `CONFIG` object with your target API's `baseUrl`, `headers`, `paginationType` (`page`, `offset`, `cursor`, or `link`), data/cursor paths, safety limits (`maxPages: 50`, `maxRetries: 3`), and database settings (`tableName`, `uniqueKey`, and `fieldMap`). Return the structured initial state and request objects.
   - Connect `Manual Trigger` → `Code - Initialize Config`.
3. **Create the HTTP Request Node**
   - Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`) named `HTTP Request - Fetch Page`.
   - Set `Method` to `={{ $json.config.method || 'GET' }}` and `URL` to `={{ $json.request.url || $json.config.baseUrl }}`.
   - Enable `Send Query Parameters` and `Send Header Parameters`. Configure query parameters and authorization headers dynamically using expressions referencing `$json.config` and `$json.state`.
   - Under node options, set timeout to `30000ms`, `Response > Never Error` to `true`, and `Response > Full Response` to `true`. Enable retry options (`Retry On Fail`, max tries = `3`, wait between tries = `2000ms`) and `Continue On Fail`.
   - Connect `Code - Initialize Config` → `HTTP Request - Fetch Page`.
4. **Create the Parse & Normalize Code Node**
   - Add a **Code** node (`n8n-nodes-base.code`) named `Code - Parse & Normalize`.
   - Insert JavaScript logic to extract records using `dataPath`, map source keys to destination columns via `fieldMap`, evaluate pagination progression based on the configured strategy, and increment state counters (`pagesDone`, `totalRecords`).
   - Connect `HTTP Request - Fetch Page` → `Code - Parse & Normalize`.
5. **Create the Has Records IF Node**
   - Add an **IF** node (`n8n-nodes-base.if`) named `IF - Has Records?`.
   - Set condition: Left Value `={{ $json.records.length }}`, Operator: `Larger than`, Right Value: `0`.
   - Connect `Code - Parse & Normalize` → `IF - Has Records?`.
6. **Create the PostgreSQL Upsert Node**
   - Add a **Postgres** node (`n8n-nodes-base.postgres`) named `Postgres - Upsert Records`.
   - Configure credentials for your PostgreSQL database.
   - Set Operation to `Execute Query` (`executeQuery`).
   - Insert JavaScript template expression logic that dynamically builds an `INSERT INTO ... VALUES ... ON CONFLICT (...) DO UPDATE SET ...` query matching your table configuration and mapped fields. Enable `Continue On Fail`.
   - Connect `IF - Has Records?` (True branch) → `Postgres - Upsert Records`.
7. **Create the Prepare Next Page Code Node**
   - Add a **Code** node (`n8n-nodes-base.code`) named `Code - Prepare Next Page`.
   - Add JavaScript to pull the latest state and request parameters and output `shouldContinue: state.hasMore === true`.
   - Connect both `Postgres - Upsert Records` and `IF - Has Records?` (False/bypass branch) → `Code - Prepare Next Page`.
8. **Create the More Pages IF Node**
   - Add an **IF** node (`n8n-nodes-base.if`) named `IF - More Pages?`.
   - Set condition: Left Value `={{ $json.shouldContinue }}`, Operator: `True`, type validation strict.
   - Connect `Code - Prepare Next Page` → `IF - More Pages?`.
9. **Create the Rate Limit Delay Wait Node**
   - Add a **Wait** node (`n8n-nodes-base.wait`) named `Wait - Rate Limit Delay`.
   - Set amount to `1` second (adjust as needed for target API rate limits).
   - Connect `IF - More Pages?` (True branch) → `Wait - Rate Limit Delay`.
   - Connect `Wait - Rate Limit Delay` back to `HTTP Request - Fetch Page` to complete the loop.
10. **Create the Finalization Code & End Nodes**
    - Add a **Code** node (`n8n-nodes-base.code`) named `Code - Finalize Job` to assemble the completion payload and summary message.
    - Connect `IF - More Pages?` (False branch) → `Code - Finalize Job`.
    - Add a **No-Op** node (`n8n-nodes-base.noOp`) named `NoOp - End`.
    - Connect `Code - Finalize Job` → `NoOp - End`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Universal API-to-Database Ingestion Engine | Workflow designed for manual trigger executions, handling arbitrary REST APIs with configurable pagination strategies, robust retry policies, and SQL upsert statements. |