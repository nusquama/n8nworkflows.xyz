Send a daily AI morning briefing with Google Gemini, RSS, Sheets, Gmail and weather

https://n8nworkflows.xyz/workflows/send-a-daily-ai-morning-briefing-with-google-gemini--rss--sheets--gmail-and-weather-17519


# Send a daily AI morning briefing with Google Gemini, RSS, Sheets, Gmail and weather

### 1. Workflow Overview

This workflow automates the generation and delivery of a personalized daily morning briefing. Triggered every morning at 06:30, it concurrently gathers external news headlines, internal task lists, and local weather forecasts. It then leverages Google Gemini to synthesize this disparate data into a cohesive, motivating narrative before dispatching it via email.

The execution logic is structured into four distinct functional blocks:

- **1.1 Trigger & Parallel Data Acquisition:** Initiates execution on a daily schedule and fans out simultaneously to fetch raw data from an RSS feed, a Google Sheets document, and an external weather API.
- **1.2 Data Transformation & Shaping:** Processes the raw inputs from each branch independently, formatting them into standardized text summaries and metadata objects.
- **1.3 Data Merging & Prompt Construction:** Consolidates the three parallel data streams into a single aggregated item and structures it into a prompt template for the language model.
- **1.4 AI Synthesis & Dispatch:** Utilizes Google Gemini to generate a narrative briefing based on the combined context and delivers the final message through Gmail.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Parallel Data Acquisition
**Overview:** This block establishes the schedule-based entry point and concurrently queries three independent external and internal data sources to gather the raw components of the morning briefing.

- **Nodes Involved:** 
  - `Schedule Every Morning`
  - `Read News Feed`
  - `Get Today Tasks`
  - `Get Weather`

- **Node Details:**
  - **Schedule Every Morning**
    - *Type & Role:* `n8n-nodes-base.scheduleTrigger` (v1.3) — Acts as the primary time-based trigger for the workflow.
    - *Configuration:* Configured to trigger daily at hour 6, minute 30.
    - *Input/Output:* No inputs; outputs to `Read News Feed`, `Get Today Tasks`, and `Get Weather`.
    - *Edge Cases:* Missed executions if the instance is offline at 06:30.
  - **Read News Feed**
    - *Type & Role:* `n8n-nodes-base.rssFeedRead` (v1.2) — Fetches and parses an RSS feed to retrieve current news items.
    - *Configuration:* Points to the BBC News RSS URL (`https://feeds.bbci.co.uk/news/rss.xml`).
    - *Input/Output:* Input from `Schedule Every Morning`; output to `Take Top Headlines`.
    - *Edge Cases:* Network timeouts or RSS feed structural changes causing parsing errors.
  - **Get Today Tasks**
    - *Type & Role:* `n8n-nodes-base.googleSheets` (v4.7) — Retrieves task rows from a specified Google Sheets document.
    - *Configuration:* Targets a list-mode sheet named "Tasks" within a user-selected spreadsheet. Uses Google Sheets OAuth2 credentials.
    - *Input/Output:* Input from `Schedule Every Morning`; output to `List Open Tasks`.
    - *Edge Cases:* Authentication expiration, missing spreadsheet/worksheet names, or API rate limits.
  - **Get Weather**
    - *Type & Role:* `n8n-nodes-base.httpRequest` (v4.4) — Queries an external weather forecasting service.
    - *Configuration:* Calls `https://api.open-meteo.com/v1/forecast` with query parameters for latitude (`35.68`), longitude (`139.76`), daily forecast metrics (temperature max/min, precipitation probability), automatic timezone, and a 1-day forecast limit. Configured with `"onError": "continueRegularOutput"` to ensure failure softness.
    - *Input/Output:* Input from `Schedule Every Morning`; output to `Shape Weather`.
    - *Edge Cases:* API unavailability or network failures (mitigated by error continuation settings).

---

#### 2.2 Data Transformation & Shaping
**Overview:** This block standardizes the output structures from the acquisition phase, reducing raw data feeds into concise, structured strings representing news headlines, task inventories, and weather summaries.

- **Nodes Involved:** 
  - `Take Top Headlines`
  - `List Open Tasks`
  - `Shape Weather`

- **Node Details:**
  - **Take Top Headlines**
    - *Type & Role:* `n8n-nodes-base.code` (v2.0) — JavaScript execution node that filters and formats RSS items.
    - *Configuration:* Slices the input array to the top 6 items, extracts the title property, and formats them into a newline-delimited string prefixed with hyphens.
    - *Expressions:* Uses `$input.all()` to access incoming items.
    - *Input/Output:* Input from `Read News Feed`; output to `Combine All Sources` (Input 0).
    - *Edge Cases:* Empty RSS arrays resulting in undefined property errors (handled safely via fallback optional chaining).
  - **List Open Tasks**
    - *Type & Role:* `n8n-nodes-base.code` (v2.0) — JavaScript execution node that structures task sheet rows.
    - *Configuration:* Iterates over sheet rows, formatting each into a priority-tagged task string, and computes the total open task count.
    - *Expressions:* Evaluates `i.json.Priority` (defaulting to 'Normal') and `i.json.Task`.
    - *Input/Output:* Input from `Get Today Tasks`; output to `Combine All Sources` (Input 1).
    - *Edge Cases:* Missing column headers (`Priority` or `Task`) returning undefined values.
  - **Shape Weather**
    - *Type & Role:* `n8n-nodes-base.code` (v2.0) — JavaScript execution node operating in `runOnceForEachItem` mode to parse forecast metrics.
    - *Configuration:* Extracts maximum/minimum temperatures and precipitation probability from the first element of the Open-Meteo daily forecast array.
    - *Expressions:* Accesses `$json.daily` with fallback default values (`?`) if data is absent due to upstream weather API failure.
    - *Input/Output:* Input from `Get Weather`; output to `Combine All Sources` (Input 2).
    - *Edge Cases:* Upstream HTTP request failure resulting in empty payloads, safely caught by fallback values.

---

#### 2.3 Data Merging & Prompt Construction
**Overview:** This block merges the three parallel processing streams back into a unified data structure, aggregates them, and maps them into variables for the downstream language model prompt.

- **Nodes Involved:** 
  - `Combine All Sources`
  - `Bundle Sources`
  - `Build Briefing Input`

- **Node Details:**
  - **Combine All Sources**
    - *Type & Role:* `n8n-nodes-base.merge` (v3.2) — Combines multiple input streams into a multi-input dataset.
    - *Configuration:* Configured with 3 numerical inputs to synchronize the news, tasks, and weather branches.
    - *Input/Output:* Inputs from `Take Top Headlines`, `List Open Tasks`, and `Shape Weather`; output to `Bundle Sources`.
    - *Edge Cases:* Branch execution desynchronization if upstream nodes experience variable latency.
  - **Bundle Sources**
    - *Type & Role:* `n8n-nodes-base.aggregate` (v1.0) — Aggregates multiple items into a single consolidated dataset item.
    - *Configuration:* Mode set to `aggregateAllItemData`.
    - *Input/Output:* Input from `Combine All Sources`; output to `Build Briefing Input`.
    - *Edge Cases:* Receiving fewer than 3 items if a branch fails completely without proper fallback handling.
  - **Build Briefing Input**
    - *Type & Role:* `n8n-nodes-base.code` (v2.0) — JavaScript execution node that normalizes the aggregated array into a single structured key-value object.
    - *Configuration:* Maps source identifiers (`news`, `tasks`, `weather`) from the data array to top-level object properties.
    - *Expressions:* Uses `.find()` logic against `$json.data` to extract source-specific attributes with fallback strings.
    - *Input/Output:* Input from `Bundle Sources`; output to `Write Morning Briefing with Gemini`.
    - *Edge Cases:* Unmatched source identifiers resulting in default fallback strings.

---

#### 2.4 AI Synthesis & Dispatch
**Overview:** This block coordinates the execution of the language model to write the briefing text based on the consolidated context and delivers the finished message via email.

- **Nodes Involved:** 
  - `Write Morning Briefing with Gemini`
  - `Google Gemini Chat Model`
  - `Email the Briefing`

- **Node Details:**
  - **Write Morning Briefing with Gemini**
    - *Type & Role:* `@n8n/n8n-nodes-langchain.chainLlm` (v1.9) — Basic LLM chain node orchestrating the prompt generation and model completion.
    - *Configuration:* Uses a defined prompt type instructing the model to act as a warm personal assistant, weaving weather, headlines, and priority tasks into an approximately 120-word motivational note without bullet lists.
    - *Expressions:* Injects dynamic context variables:
      - `{{ $json.weather }}`
      - `{{ $json.headlines }}`
      - `{{ $json.openCount }}`
      - `{{ $json.tasks }}`
    - *Input/Output:* Input from `Build Briefing Input`; AI model connection from `Google Gemini Chat Model`; output to `Email the Briefing`.
    - *Edge Cases:* Token limit issues or model refusal outputs.
  - **Google Gemini Chat Model**
    - *Type & Role:* `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` (v1.1) — Sub-node providing the underlying chat model configuration for the LLM chain.
    - *Configuration:* Configured with model name `models/gemini-3.1-flash-lite` and a temperature of `0.5`. Requires Google Gemini (PaLM) API credentials.
    - *Input/Output:* Outputs an AI language model connection to `Write Morning Briefing with Gemini`.
    - *Edge Cases:* API authentication failures, quota exhaustion, or model deprecation.
  - **Email the Briefing**
    - *Type & Role:* n8n-nodes-base.gmail (v2.2) — Sends the final generated briefing text via Gmail OAuth2.
    - *Configuration:* Email type set to text. Uses dynamic expressions for recipient, subject line, and body message.
    - *Expressions:*
      - Recipient (`sendTo`): `you@example.com`
      - Subject: `={{"Your morning briefing - " + $now.toFormat("cccc d LLLL")}}`
      - Message: `={{ $json.text }}`
    - *Credentials:* Requires Gmail OAuth2 credentials.
    - *Input/Output:* Input from `Write Morning Briefing with Gemini`; terminal workflow node.
    - *Edge Cases:* OAuth2 token revocation, invalid recipient address, or SMTP sending quotas.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Schedule Every Morning | n8n-nodes-base.scheduleTrigger | Triggers the workflow daily at 06:30. | None | Read News Feed, Get Today Tasks, Get Weather | ## Combine weather, news and your tasks into one AI morning briefing... (See Overview Sticky) / ## 1. Three parallel sources... (See S1) |
| Read News Feed | n8n-nodes-base.rssFeedRead | Fetches BBC News headlines from RSS. | Schedule Every Morning | Take Top Headlines | ## 1. Three parallel sources... (See S1) |
| Take Top Headlines | n8n-nodes-base.code | Filters and formats top 6 RSS headlines. | Read News Feed | Combine All Sources | ## 1. Three parallel sources... (See S1) |
| Get Today Tasks | n8n-nodes-base.googleSheets | Retrieves task rows from Google Sheets. | Schedule Every Morning | List Open Tasks | ## 1. Three parallel sources... (See S1) |
| List Open Tasks | n8n-nodes-base.code | Formats sheet rows into open tasks list. | Get Today Tasks | Combine All Sources | ## 1. Three parallel sources... (See S1) |
| Get Weather | n8n-nodes-base.httpRequest | Fetches weather forecast from Open-Meteo. | Schedule Every Morning | Shape Weather | ## 1. Three parallel sources... (See S1) |
| Shape Weather | n8n-nodes-base.code | Parses weather metrics into summary text. | Get Weather | Combine All Sources | ## 1. Three parallel sources... (See S1) |
| Combine All Sources | n8n-nodes-base.merge | Merges parallel news, tasks, and weather streams. | Take Top Headlines, List Open Tasks, Shape Weather | Bundle Sources | ## 2. Merge & bundle... (See S2) |
| Bundle Sources | n8n-nodes-base.aggregate | Bundles merged items into a single aggregate item. | Combine All Sources | Build Briefing Input | ## 2. Merge & bundle... (See S2) |
| Build Briefing Input | n8n-nodes-base.code | Normalizes aggregated data into briefing variables. | Bundle Sources | Write Morning Briefing with Gemini | ## 3. Write & send (Basic LLM Chain)... (See S3) |
| Write Morning Briefing with Gemini | @n8n/n8n-nodes-langchain.chainLlm | Generates morning briefing narrative via LLM. | Build Briefing Input, Google Gemini Chat Model | Email the Briefing | ## 3. Write & send (Basic LLM Chain)... (See S3) |
| Google Gemini Chat Model | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | Provides Gemini model config for LLM chain. | None | Write Morning Briefing with Gemini | ## 3. Write & send (Basic LLM Chain)... (See S3) |
| Email the Briefing | n8n-nodes-base.gmail | Sends the final briefing text via Gmail. | Write Morning Briefing with Gemini | None | ## 3. Write & send (Basic LLM Chain)... (See S3) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**
   - Add a **Schedule Trigger** node named `Schedule Every Morning`.
   - Set interval rule to trigger at hour `6` and minute `30`.

2. **Create the News Branch:**
   - Add an **RSS Feed Read** node named `Read News Feed`. Set URL to `https://feeds.bbci.co.uk/news/rss.xml`. Connect `Schedule Every Morning` to this node.
   - Add a **Code** node named `Take Top Headlines`. Set JavaScript code to slice top 6 items, format titles with hyphens, and return `source: 'news'`. Connect `Read News Feed` output to this node.

3. **Create the Tasks Branch:**
   - Add a **Google Sheets** node named `Get Today Tasks`. Configure credentials (`Google Sheets OAuth2 account`), set document mode to list, select your spreadsheet, and select worksheet `Tasks`. Connect `Schedule Every Morning` to this node.
   - Add a **Code** node named `List Open Tasks`. Set JavaScript code to format rows into priority-tagged task strings and count items. Return `source: 'tasks'`. Connect `Get Today Tasks` output to this node.

4. **Create the Weather Branch:**
   - Add an **HTTP Request** node named `Get Weather`. Set URL to `https://api.open-meteo.com/v1/forecast`, method to `GET`, and enable query parameters: `latitude=35.68`, `longitude=139.76`, `daily=temperature_2m_max,temperature_2m_min,precipitation_probability_max`, `timezone=auto`, `forecast_days=1`. In node settings, set Error Handling to **Continue Regular Output**. Connect `Schedule Every Morning` to this node.
   - Add a **Code** node named `Shape Weather`. Set mode to `Run Once for Each Item` and add JavaScript to extract high/low temperatures and rain probability into a weather summary string (`source: 'weather'`). Connect `Get Weather` output to this node.

5. **Merge and Aggregate Streams:**
   - Add a **Merge** node named `Combine All Sources`. Set number of inputs to `3`. Connect `Take Top Headlines` to input 0, `List Open Tasks` to input 1, and `Shape Weather` to input 2.
   - Add an **Aggregate** node named `Bundle Sources`. Set aggregate mode to `Aggregate All Item Data`. Connect `Combine All Sources` to this node.
   - Add a **Code** node named `Build Briefing Input`. Set JavaScript code to map the aggregated array into structured object keys (`headlines`, `tasks`, `openCount`, `weather`). Connect `Bundle Sources` to this node.

6. **Configure AI Synthesis:**
   - Add a **Basic LLM Chain** node named `Write Morning Briefing with Gemini`. Configure the prompt definition with persona instructions and dynamic expressions (`{{ $json.weather }}`, `{{ $json.headlines }}`, `{{ $json.openCount }}`, `{{ $json.tasks }}`). Connect `Build Briefing Input` to this node.
   - Add a **Google Gemini Chat Model** node named `Google Gemini Chat Model`. Set model name to `models/gemini-3.1-flash-lite`, temperature to `0.5`, and configure credentials (`Google Gemini(PaLM) Api account`). Connect its AI model output to `Write Morning Briefing with Gemini`.

7. **Configure Delivery:**
   - Add a **Gmail** node named `Email the Briefing`. Configure credentials (`Gmail account`), set `sendTo` to your target email address, set `emailType` to `text`, set subject expression to `={{"Your morning briefing - " + $now.toFormat("cccc d LLLL")}}`, and message expression to `={{ $json.text }}`. Connect `Write Morning Briefing with Gemini` to this node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Parallel-branch and merge architectural pattern combining independent data feeds (RSS, Sheets, API) into a unified AI prompt context. | Workflow Structural Overview |
| Weather branch is configured to fail soft (`continueRegularOutput`) to ensure API failures never block the briefing generation. | Error Handling Design |