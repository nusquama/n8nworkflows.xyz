Find outdated statistics in an article with DuckDuckGo

https://n8nworkflows.xyz/workflows/find-outdated-statistics-in-an-article-with-duckduckgo-20511


# Find outdated statistics in an article with DuckDuckGo

### 1. Workflow Overview

This workflow is designed to audit published articles for outdated statistics. By receiving a web address via a user-facing form, it extracts the article's text, identifies dated figures using an AI model with structured validation code, performs targeted DuckDuckGo searches to find more recent figures from web sources, and compiles a comprehensive side-by-side HTML comparison table for review. 

The logic is grouped into five functional blocks:
- **1.1 Input Reception & Validation:** Handles the initial form trigger, sanitizes user input, validates public URL structures, and screens out private or local endpoints.
- **1.2 Article Retrieval & Extraction:** Fetches the article content using the DuckDuckGo community node, checks for bot walls, paywalls, or truncation issues, and prepares a securely fenced data payload.
- **1.3 Figure Identification & Validation:** Uses an AI model coupled with a structured JSON output parser to propose up to five dated figures, followed by programmatic validation code to strip out forecasts, targets, and unverified data points.
- **1.4 Web Search & Source Verification:** Iterates through valid figures to query DuckDuckGo, grades the resulting pages, introduces pacing intervals to prevent rate limiting, aggregates and indexes valid sources, and prompts the AI model to locate newer figures.
- **1.5 Review Table Compilation & Result Rendering:** Verifies candidate quotes against source texts, builds an escaped HTML review table, handles alternative workflow exit paths, and displays the final output via an n8n form completion page.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Input Reception & Validation

- **Overview:** Receives the target article URL and optional geographic region parameters from a web form, ensuring that only valid public web addresses are processed.
- **Nodes Involved:** 
  - `Article to check` (`n8n-nodes-base.formTrigger`)
  - `Check the request` (`n8n-nodes-base.code`)
  - `Address accepted?` (`n8n-nodes-base.if`)

##### Node Details:

- **Article to check**
  - **Type & Technical Role:** Form Trigger (`n8n-nodes-base.formTrigger`). Serves as the workflow entry point, displaying an input form to the user.
  - **Configuration:** Configured with two text fields (`url` and `region`), requiring user interaction and utilizing n8n User Authentication.
  - **Key Expressions:** Uses form variables to capture inputs.
  - **Connections:** Input: None (Trigger). Output: `Check the request`.
  - **Edge Cases:** Invalid or empty submissions handled downstream by validation logic.

- **Check the request**
  - **Type & Technical Role:** Code Node (`n8n-nodes-base.code`). Validates the structure and public availability of the provided URL.
  - **Configuration:** JavaScript mode (`runOnceForAllItems`). Performs regex and string manipulation to strip dangerous patterns, block local/private network ranges, and clean the region query.
  - **Key Expressions:** 
    - `={{ $now.toFormat('yyyy-MM-dd') }}`
    - `={{ Number($now.toFormat('yyyy')) }}`
  - **Connections:** Input: `Article to check`. Output: `Address accepted?`.
  - **Edge Cases:** Rejects non-HTTP(S) schemes, credentials in URLs, non-standard ports, and local loopbacks (`localhost`, `.local`, `.internal`).

- **Address accepted?**
  - **Type & Technical Role:** If Node (`n8n-nodes-base.if`). Directs workflow execution based on the validation status.
  - **Configuration:** Evaluates boolean condition.
  - **Key Expressions:** `={{ $json.ok }}`
  - **Connections:** Input: `Check the request`. Outputs: 
    - `true`: `Read the article`
    - `false`: `Explain why there is no table`
  - **Edge Cases:** Routes invalid requests directly to error rendering nodes.

---

#### Block 1.2: Article Retrieval & Extraction

- **Overview:** Downloads the article's text content from the validated web address, verifies its readability, and checks for paywalls, bot barriers, or excessive length.
- **Nodes Involved:**
  - `Read the article` (`n8n-nodes-duckduckgo-search.duckDuckGo`)
  - `Check the article` (`n8n-nodes-base.code`)
  - `Article readable?` (`n8n-nodes-base.if`)

##### Node Details:

- **Read the article**
  - **Type & Technical Role:** DuckDuckGo Node (`n8n-nodes-duckduckgo-search.duckDuckGo`). Extracts webpage content and metadata.
  - **Configuration:** Operation set to `extractContent`. Timeout: 15,000ms. Max length: 40,000 characters. Error handling configured to continue regular output on failure.
  - **Key Expressions:** `={{ $json.url }}`
  - **Connections:** Input: `Address accepted?`. Output: `Check the article`.
  - **Version Requirements:** Requires the community node `n8n-nodes-duckduckgo-search` installed on a self-hosted n8n instance.
  - **Edge Cases:** Network timeouts, HTTP errors, or missing pages return failure payloads processed by downstream code.

- **Check the article**
  - **Type & Technical Role:** Code Node (`n8n-nodes-base.code`). Evaluates the extracted text for minimum length requirements and common bot/paywall indicators.
  - **Configuration:** JavaScript mode (`runOnceForAllItems`). Implements regex checks for strings like "accept cookies", "access denied", or "are you a robot". Generates a unique execution nonce to sandbox untrusted text.
  - **Key Expressions:** Accesses preceding node payloads via `$('Check the request').first().json`.
  - **Connections:** Input: `Read the article`. Output: `Article readable?`.
  - **Edge Cases:** Text lengths under 800 characters or pages serving consent walls are flagged as unreadable.

- **Article readable?**
  - **Type & Technical Role:** If Node (`n8n-nodes-base.if`). Checks if the article text passed all quality and readability validations.
  - **Configuration:** Evaluates boolean condition.
  - **Key Expressions:** `={{ $json.ok }}`
  - **Connections:** Input: `Check the article`. Outputs:
    - `true`: `Find dated figures`
    - `false`: `Explain why there is no table`
  - **Edge Cases:** Unreadable articles bypass AI processing and route to explanation nodes.

---

#### Block 1.3: Figure Identification & Validation

- **Overview:** Sends the securely fenced article text to an LLM to identify up to five measurable dated figures, then programmatically validates that sentences and values exist within the article text.
- **Nodes Involved:**
  - `Find dated figures` (`@n8n/n8n-nodes-langchain.chainLlm`)
  - `Figure format` (`@n8n/n8n-nodes-langchain.outputParserStructured`)
  - `OpenRouter model` (`@n8n/n8n-nodes-langchain.lmChatOpenRouter`)
  - `Check the figures` (`n8n-nodes-base.code`)
  - `Any figures to check?` (`n8n-nodes-base.if`)

##### Node Details:

- **Find dated figures**
  - **Type & Technical Role:** Basic LLM Chain (`@n8n/n8n-nodes-langchain.chainLlm`). Coordinates prompt execution with the language model.
  - **Configuration:** Max retries: 2. Uses prompt templates instructing the model to extract measured values, units, metrics, scopes, years, and search queries while ignoring forecasts and projections.
  - **Key Expressions:** 
    - `=Current year: {{ $json.year }}`
    - `Article title: {{ $json.title }}`
    - `{{ $json.fenced }}`
  - **Connections:** Input: `Article readable?`. Output: `Check the figures`. Linked to `OpenRouter model` (Language Model) and `Figure format` (Output Parser).
  - **Edge Cases:** Malformed JSON outputs trigger retry attempts via built-in retry configurations.

- **Figure format**
  - **Type & Technical Role:** Structured Output Parser (`@n8n/n8n-nodes-langchain.outputParserStructured`). Enforces strict JSON schemas on LLM outputs.
  - **Configuration:** Manual schema defining an array of objects containing `sentence`, `value`, `unit`, `metric`, `scope`, `year`, and `search_query`.
  - **Connections:** Output: Connected to `Find dated figures` via AI output parser connection.

- **OpenRouter model**
  - **Type & Technical Role:** OpenRouter Chat Model (`@n8n/n8n-nodes-langchain.lmChatOpenRouter`). Provides AI inference capabilities.
  - **Configuration:** Model: `google/gemini-3.8-flash`. Temperature: 0.1. Provider routing configured with fallback options.
  - **Connections:** Output: Connected to LLM chains via AI language model connections.
  - **Credentials Required:** OpenRouter API Credential.

- **Check the figures**
  - **Type & Technical Role:** Code Node (`n8n-nodes-base.code`). Programmatically validates model-proposed figures against the raw article text.
  - **Configuration:** JavaScript mode (`runOnceForAllItems`). Verifies exact sentence inclusion, presence of numerical values, exclusion of forecasts, and ensures the referenced year is in the past. Limits results to a maximum of 5 figures.
  - **Connections:** Input: `Find dated figures`. Output: `Any figures to check?`.
  - **Edge Cases:** Drops figures where quoted sentences fail exact substring matching or refer to current/future years.

- **Any figures to check?**
  - **Type & Technical Role:** If Node (`n8n-nodes-base.if`). Determines whether valid figures remain for web searching.
  - **Configuration:** Evaluates boolean condition.
  - **Key Expressions:** `={{ $json.figure }}`
  - **Connections:** Input: `Check the figures`. Outputs:
    - `true`: `Loop over figures`
    - `false`: `Explain why there is no table`

---

#### Block 1.4: Web Search & Source Verification

- **Overview:** Iterates through each validated figure, executes targeted DuckDuckGo searches, paces requests to avoid rate limits, indexes source documents securely, and queries the AI model for newer statistics.
- **Nodes Involved:**
  - `Loop over figures` (`n8n-nodes-base.splitInBatches`)
  - `Search for a later figure` (`n8n-nodes-duckduckgo-search.duckDuckGo`)
  - `Grade the sources` (`n8n-nodes-base.code`)
  - `Pace the searches` (`n8n-nodes-base.wait`)
  - `Number the sources` (`n8n-nodes-base.code`)
  - `Find later figures` (`@n8n/n8n-nodes-langchain.chainLlm`)
  - `Candidate format` (`@n8n/n8n-nodes-langchain.outputParserStructured`)
  - `Verify the candidates` (`n8n-nodes-base.code`)

##### Node Details:

- **Loop over figures**
  - **Type & Technical Role:** Split In Batches (`n8n-nodes-base.splitInBatches`). Processes figures sequentially to manage API and search request rates.
  - **Configuration:** Batch size: 1.
  - **Connections:** Input: `Any figures to check?`, `Pace the searches`. Outputs: Loop back to `Search for a later figure`, and progress to `Number the sources` when complete.

- **Search for a later figure**
  - **Type & Technical Role:** DuckDuckGo Node (`n8n-nodes-duckduckgo-search.duckDuckGo`). Performs web searches using generated search queries.
  - **Configuration:** Operation set to `search`. Max results: 6. Fetches page content with a 10,000ms timeout and a maximum of 5 page content results. Error handling set to continue on fail.
  - **Key Expressions:** `={{ $json.searchQuery }}`
  - **Connections:** Input: `Loop over figures`. Output: `Grade the sources`.
  - **Edge Cases:** DuckDuckGo blocks or rate limits are captured and processed by downstream grading code.

- **Grade the sources**
  - **Type & Technical Role:** Code Node (`n8n-nodes-base.code`). Filters search results to retain only readable, valid HTML source pages.
  - **Configuration:** JavaScript mode (`runOnceForAllItems`). Inspects pages for bot walls, garbled text encoding, and minimum length requirements.
  - **Connections:** Input: `Search for a later figure`. Output: `Pace the searches`.

- **Pace the searches**
  - **Type & Technical Role:** Wait Node (`n8n-nodes-base.wait`). Introduces a delay between search iterations to comply with DuckDuckGo IP rate limits.
  - **Configuration:** Amount: 6 seconds. Resume: Time interval.
  - **Connections:** Input: `Grade the sources`. Output: `Loop over figures`.

- **Number the sources**
  - **Type & Technical Role:** Code Node (`n8n-nodes-base.code`). Consolidates and indexes unique sources across all figures into a canonical list (`S1`, `S2`, etc.) and generates fenced evidence blocks.
  - **Configuration:** JavaScript mode (`runOnceForAllItems`). Builds structured evidence text blocks tagged with cryptographic nonces.
  - **Connections:** Input: `Loop over figures` (loop completion). Output: `Find later figures`.

- **Find later figures**
  - **Type & Technical Role:** Basic LLM Chain (`@n8n/n8n-nodes-langchain.chainLlm`). Queries the AI model to locate newer statistical figures within the indexed source evidence.
  - **Configuration:** Max retries: 2. Uses prompt templates constraining the model to source-specific IDs and strict quote criteria.
  - **Key Expressions:**
    - `=Current year: {{ $json.year }}`
    - `Figures from the article: {{ $json.list }}`
    - `Sources: {{ $json.evidence }}`
  - **Connections:** Input: `Number the sources`. Output: `Verify the candidates`. Linked to `OpenRouter model` and `Candidate format`.

- **Candidate format**
  - **Type & Technical Role:** Structured Output Parser (`@n8n/n8n-nodes-langchain.outputParserStructured`). Enforces schema requirements on candidate search results.
  - **Configuration:** Manual schema requiring an array of objects with `figure`, `source_id`, `quote`, `value`, and `year`.
  - **Connections:** Output: Connected to `Find later figures` via AI output parser connection.

- **Verify the candidates**
  - **Type & Technical Role:** Code Node (`n8n-nodes-base.code`). Validates AI-proposed candidate quotes against retrieved source texts.
  - **Configuration:** JavaScript mode (`runOnceForAllItems`). Checks substring inclusion, proximity of numeric values and years in quotes, exclusion of forecasts, and date constraints.
  - **Connections:** Input: `Find later figures`. Output: `Build the review table`.

---

#### Block 1.5: Review Table Compilation & Result Rendering

- **Overview:** Formats verified statistical comparisons into a clean HTML review table or renders alternative error pages if processing stops early.
- **Nodes Involved:**
  - `Build the review table` (`n8n-nodes-base.code`)
  - `Explain why there is no table` (`n8n-nodes-base.code`)
  - `Show the result` (`n8n-nodes-base.form`)

##### Node Details:

- **Build the review table**
  - **Type & Technical Role:** Code Node (`n8n-nodes-base.code`). Generates escaped HTML markup containing side-by-side comparisons of article figures and verified newer figures.
  - **Configuration:** JavaScript mode (`runOnceForAllItems`). Escapes all user and model inputs to prevent XSS vulnerabilities, formats source links securely, and constructs summary drop-downs for dropped figures.
  - **Connections:** Input: `Verify the candidates`. Output: `Show the result`.

- **Explain why there is no table**
  - **Type & Technical Role:** Code Node (`n8n-nodes-base.code`). Generates fallback HTML explanation pages for runs that terminate without generating comparison tables.
  - **Configuration:** JavaScript mode (`runOnceForAllItems`). Handles scenarios where URLs are rejected, articles are unreadable, or no dated figures are detected.
  - **Connections:** Inputs: `Address accepted?`, `Article readable?`, `Any figures to check?`. Output: `Show the result`.

- **Show the result**
  - **Type & Technical Role:** Form Node (`n8n-nodes-base.form`). Renders the final HTML response to the user via form completion.
  - **Configuration:** Operation: `completion`. Respond with: `showText`. Injects custom CSS styling and displays the compiled HTML payload.
  - **Key Expressions:** `=<style>...</style>{{ $json.html }}`
  - **Connections:** Inputs: `Build the review table`, `Explain why there is no table`. Output: None (Terminal Node).
  - **Edge Cases:** Serves final completion pages regardless of whether the run succeeded with a review table or ended with an error explanation.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Overview | n8n-nodes-base.stickyNote | Workflow documentation & overview | None | None | ## Find outdated statistics in an article... |
| Section - read the article | n8n-nodes-base.stickyNote | Block 1 documentation | None | None | ## 1. Read the article<br>Only public web addresses are accepted... |
| Section - find dated figures | n8n-nodes-base.stickyNote | Block 3 documentation | None | None | ## 2. Find dated figures<br>The model proposes figures... |
| Section - search for later figures | n8n-nodes-base.stickyNote | Block 4 documentation | None | None | ## 3. Search for later figures<br>One DuckDuckGo search per figure... |
| Section - match and verify | n8n-nodes-base.stickyNote | Block 4 source matching documentation | None | None | ## 4. Match and verify<br>Sources are numbered in code... |
| Section - show the review table | n8n-nodes-base.stickyNote | Block 5 documentation | None | None | ## 5. Show the review table<br>Both sentences side by side... |
| Article to check | n8n-nodes-base.formTrigger | Workflow form entry point | None | Check the request | |
| Check the request | n8n-nodes-base.code | URL validation & input sanitization | Article to check | Address accepted? | |
| Address accepted? | n8n-nodes-base.if | Conditional routing based on URL validity | Check the request | Read the article,<br>Explain why there is no table | |
| Read the article | n8n-nodes-duckduckgo-search.duckDuckGo | Fetches webpage text content | Address accepted? | Check the article | |
| Check the article | n8n-nodes-base.code | Article readability & bot-wall screening | Read the article | Article readable? | |
| Article readable? | n8n-nodes-base.if | Conditional routing based on article readability | Check the article | Find dated figures,<br>Explain why there is no table | |
| Find dated figures | @n8n/n8n-nodes-langchain.chainLlm | LLM chain to extract dated figures | Article readable? | Check the figures | |
| Figure format | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces JSON schema for extracted figures | None | Find dated figures | |
| OpenRouter model | @n8n/n8n-nodes-langchain.lmChatOpenRouter | Provides AI model (Gemini 3.8 Flash) | None | Find dated figures,<br>Find later figures | |
| Check the figures | n8n-nodes-base.code | Validates model-proposed figures against text | Find dated figures | Any figures to check? | |
| Any figures to check? | n8n-nodes-base.if | Conditional routing for valid figures | Check the figures | Loop over figures,<br>Explain why there is no table | |
| Loop over figures | n8n-nodes-base.splitInBatches | Iterates over validated figures sequentially | Any figures to check?,<br>Pace the searches | Number the sources,<br>Search for a later figure | |
| Search for a later figure | n8n-nodes-duckduckgo-search.duckDuckGo | Executes web search for newer statistics | Loop over figures | Grade the sources | |
| Grade the sources | n8n-nodes-base.code | Filters and validates search result pages | Search for a later figure | Pace the searches | |
| Pace the searches | n8n-nodes-base.wait | Rate-limiting delay between searches | Grade the sources | Loop over figures | |
| Number the sources | n8n-nodes-base.code | Canonical source indexing and fencing | Loop over figures | Find later figures | |
| Find later figures | @n8n/n8n-nodes-langchain.chainLlm | LLM chain to propose newer figures | Number the sources | Verify the candidates | |
| Candidate format | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces JSON schema for search candidates | None | Find later figures | |
| Verify the candidates | n8n-nodes-base.code | Validates candidate quotes against sources | Find later figures | Build the review table | |
| Build the review table | n8n-nodes-base.code | Compiles comparison results into HTML | Verify the candidates | Show the result | |
| Explain why there is no table | n8n-nodes-base.code | Generates error/fallback explanation pages | Address accepted?,<br>Article readable?,<br>Any figures to check? | Show the result | |
| Show the result | n8n-nodes-base.form | Renders final HTML output to user | Build the review table,<br>Explain why there is no table | None | |

---

### 4. Reproducing the Workflow from Scratch

To rebuild this workflow manually in a self-hosted n8n environment, follow these steps sequentially:

1. **Install Community Nodes:**
   - Go to **Settings > Community nodes** in your n8n instance and install `n8n-nodes-duckduckgo-search`.

2. **Create the Entry Point:**
   - Add a **Form Trigger** node named `Article to check`. Configure form fields:
     - Field 1: Text field with name `url`, label `Article address`, placeholder `https://example.com/article`, required.
     - Field 2: Text field with name `region`, label `Region or market (optional)`, placeholder `e.g. UK, Germany, worldwide`.
     - Set response mode to `Last Node` and authentication to `n8n User Auth`.

3. **Add URL Validation:**
   - Create a **Code** node named `Check the request` connected downstream from `Article to check`. Set mode to `Run Once for All Items` and paste the URL sanitization and regex validation logic from the JSON reference.
   - Add an **If** node named `Address accepted?` checking `{{ $json.ok }}` equals `true`.

4. **Configure Article Retrieval:**
   - Connect the `true` branch of `Address accepted?` to a **DuckDuckGo** node named `Read the article`. Set operation to `Extract Content`, URL parameter to `={{ $json.url }}`, page content timeout to `15000`, and max length to `40000`. Set error handling to continue regular output on fail.
   - Connect `Read the article` to a **Code** node named `Check the article` to validate text length and screen for bot/paywalls.
   - Add an **If** node named `Article readable?` checking `{{ $json.ok }}` equals `true`.

5. **Set Up AI Figure Extraction:**
   - Connect the `true` branch of `Article readable?` to a **Basic LLM Chain** node named `Find dated figures`.
   - Connect an **OpenRouter Chat Model** node named `OpenRouter model` (configured with model `google/gemini-3.8-flash` and an OpenRouter credential) to the LLM node's language model input.
   - Connect a **Structured Output Parser** node named `Figure format` to the LLM node's parser input, defining the schema for `figures` (array of objects containing `sentence`, `value`, `unit`, `metric`, `scope`, `year`, `search_query`).
   - Connect `Find dated figures` to a **Code** node named `Check the figures` to validate extracted figures against article text.
   - Add an **If** node named `Any figures to check?` checking `{{ $json.figure }}` equals `true`.

6. **Configure Iterative Web Searches & Source Verification:**
   - Connect the `true` branch of `Any figures to check?` to a **Split In Batches** node named `Loop over figures` with batch size set to `1`.
   - Connect the batch loop output to a **DuckDuckGo** node named `Search for a later figure`. Set operation to `Search`, query to `={{ $json.searchQuery }}`, max results to `6`, and page content max results to `5`. Enable error handling to continue on fail.
   - Connect `Search for a later figure` to a **Code** node named `Grade the sources`.
   - Connect `Grade the sources` to a **Wait** node named `Pace the searches` set to `6` seconds using time interval resume. Route the wait node back into `Loop over figures`.
   - Once the batch loop completes, connect it to a **Code** node named `Number the sources`.
   - Connect `Number the sources` to a second **Basic LLM Chain** node named `Find later figures`, linking it to the same `OpenRouter model` and a **Structured Output Parser** named `Candidate format` (defining schema for `candidates` array with `figure`, `source_id`, `quote`, `value`, `year`).
   - Connect `Find later figures` to a **Code** node named `Verify the candidates`.

7. **Build Output & Rendering Logic:**
   - Connect `Verify the candidates` to a **Code** node named `Build the review table` to generate escaped comparison HTML.
   - Create a separate **Code** node named `Explain why there is no table` to handle error branches from `Address accepted?`, `Article readable?`, and `Any figures to check?`.
   - Connect both `Build the review table` and `Explain why there is no table` to a **Form** node named `Show the result`. Set operation to `Completion`, respond with `Show Text`, and insert response text using `=<style>...</style>{{ $json.html }}`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Canvas preview limitation | Community nodes do not render visually on the standard n8n canvas preview. |
| Source documentation and repository | [GitHub Documentation & Examples](https://github.com/samnodehi/n8n-nodes-duckduckgo/tree/main/docs/examples) |
| High-resolution workflow canvas screenshot | [Workflow Canvas Preview Image](https://raw.githubusercontent.com/samnodehi/n8n-nodes-duckduckgo/main/docs/examples/images/06-outdated-statistics.png) |