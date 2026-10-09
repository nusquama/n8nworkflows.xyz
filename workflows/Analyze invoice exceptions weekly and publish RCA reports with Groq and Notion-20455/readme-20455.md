Analyze invoice exceptions weekly and publish RCA reports with Groq and Notion

https://n8nworkflows.xyz/workflows/analyze-invoice-exceptions-weekly-and-publish-rca-reports-with-groq-and-notion-20455


# Analyze invoice exceptions weekly and publish RCA reports with Groq and Notion

### 1. Workflow Overview

This workflow functions as an automated **Process Improvement Engine** for procurement and finance operations. It aggregates weekly invoice exceptions from Google Sheets, leverages AI to identify systemic patterns causing these delays, recommends proactive upstream process fixes, and publishes a structured report to a Notion database before alerting stakeholders via Slack.

The workflow logic is grouped into the following functional blocks:
- **1.1 Data Ingestion & Prep:** Triggers on a weekly schedule to pull raw invoice exception rows from Google Sheets, aggregating them by exception category and calculating counts, total amounts, and associated metadata.
- **1.2 Root Cause Analysis (AI):** Uses a Groq-hosted large language model with a structured output parser to analyze the grouped exception summaries and generate hypotheses for why these delays occur.
- **1.3 Process Improvement (AI):** Passes the root cause analysis into a second Groq-hosted LLM to generate strategic, proactive, and category-specific upstream process fixes.
- **1.4 Report Generation & Formatting:** Converts the structured AI outputs into a standardized Markdown report string and parses it line-by-line into native Notion block JSON schemas.
- **1.5 Publishing & Notification:** Creates a new page shell in a target Notion database, injects the formatted blocks via the Notion REST API, and dispatches a notification message via Slack.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Ingestion & Prep
**Overview:** Initiates execution on a weekly schedule, queries a Google Sheet to retrieve raw invoice exceptions, and groups the dataset by exception reason to summarize financial impact and list relevant invoice details for the AI.

**Nodes Involved:**
- `Trigger: Weekly Schedule`
- `Fetch: Invoice Exceptions`
- `Group Exceptions`

**Node Details:**
- **Trigger: Weekly Schedule**
  - *Type & Role:* `n8n-nodes-base.scheduleTrigger` (Trigger node)
  - *Configuration:* Configured to trigger every Monday at 09:00 AM.
  - *Expressions/Variables:* None.
  - *Connections:* Input: None; Output: `Fetch: Invoice Exceptions`.
  - *Edge Cases/Failures:* Failure to execute if the n8n instance is offline during the scheduled window.
- **Fetch: Invoice Exceptions**
  - *Type & Role:* `n8n-nodes-base.googleSheets` (Data retrieval node)
  - *Configuration:* Reads from document ID `1lzCKhzXrxjrbtZ_OHAbH_4xi9EIAjF70yweuJ9U-IPY` using list mode on `Sheet1` (`gid=0`).
  - *Expressions/Variables:* None.
  - *Connections:* Input: `Trigger: Weekly Schedule`; Output: `Group Exceptions`.
  - *Edge Cases/Failures:* Authentication revocation, network timeouts, or missing expected columns (`Exception_Reason`, `Amount`, `Invoice_ID`, `Vendor_Name`, `Notes`).
- **Group Exceptions**
  - *Type & Role:* `n8n-nodes-base.code` (JavaScript transformation node)
  - *Configuration:* Maps incoming sheet rows, creates a nested dictionary grouped by `Exception_Reason`, increments counts, aggregates `Amount`, and pushes individual invoice metadata into arrays.
  - *Expressions/Variables:* `$input.all()`
  - *Connections:* Input: `Fetch: Invoice Exceptions`; Output: `AI: Root Cause Analysis`.
  - *Edge Cases/Failures:* JavaScript execution error if sheet rows lack the `Exception_Reason` or `Amount` properties.

---

#### 2.2 Root Cause Analysis (AI)
**Overview:** Sends the grouped exception summary to a Groq LLM to identify operational patterns and financial impact, enforcing a structured output schema via a parser.

**Nodes Involved:**
- `AI: Root Cause Analysis`
- `LLM: Groq RCA`
- `Parser: RCA Output`
- `Extract: RCA Data`

**Node Details:**
- **AI: Root Cause Analysis**
  - *Type & Role:* `@n8n/n8n-nodes-langchain.chainLlm` (AI Agent/Chain node)
  - *Configuration:* Defines a system prompt instructing the model to act as a senior Procurement and Finance Operations Analyst.
  - *Expressions/Variables:* `{{ JSON.stringify($json.exception_summary, null, 2) }}`
  - *Connections:* Input: `Group Exceptions`, `LLM: Groq RCA` (AI model sub-node), `Parser: RCA Output` (Output parser sub-node); Output: `Extract: RCA Data`.
  - *Edge Cases/Failures:* API timeouts, rate-limiting on Groq, or schema validation failures if the model output deviates from the expected structure.
- **LLM: Groq RCA**
  - *Type & Role:* `@n8n/n8n-nodes-langchain.lmChatGroq` (Sub-node model provider)
  - *Configuration:* Model: `openai/gpt-oss-120b`. Uses credentials ID `AnindwaoyRCy8KlP` (Groq).
  - *Connections:* Connects to `AI: Root Cause Analysis` via `ai_languageModel`.
- **Parser: RCA Output**
  - *Type & Role:* `@n8n/n8n-nodes-langchain.outputParserStructured` (Sub-node parser)
  - *Configuration:* Manual JSON schema expecting an `overall_summary` string and a `categories` array containing `exception_type`, `financial_impact`, `root_cause_hypothesis`, and `priority_level`.
  - *Connections:* Connects to `AI: Root Cause Analysis` via `ai_outputParser`.
- **Extract: RCA Data**
  - *Type & Role:* `n8n-nodes-base.set` (Data transformation node)
  - *Configuration:* Assigns the stringified output from the parser to a property named `root_cause_analysis`.
  - *Expressions/Variables:* `{{ $json.output }}`
  - *Connections:* Input: `AI: Root Cause Analysis`; Output: `AI: Recommend Fixes`.
  - *Edge Cases/Failures:* Expression evaluation error if the AI node returns malformed output.

---

#### 2.3 Process Improvement (AI)
**Overview:** Takes the root cause analysis and prompts a second Groq LLM to generate proactive, strategic upstream process fixes categorized by exception type.

**Nodes Involved:**
- `AI: Recommend Fixes`
- `LLM: Groq Fixes`
- `Parser: Fixes Output`
- `Extract: Fixes Data`

**Node Details:**
- **AI: Recommend Fixes**
  - *Type & Role:* `@n8n/n8n-nodes-langchain.chainLlm` (AI Agent/Chain node)
  - *Configuration:* System prompt instructing the model to act as a Process Improvement Consultant for Procurement and Finance, outputting structured JSON.
  - *Expressions/Variables:* `{{ $json.root_cause_analysis }}`
  - *Connections:* Input: `Extract: RCA Data`, `LLM: Groq Fixes` (AI model sub-node), `Parser: Fixes Output` (Output parser sub-node); Output: `Extract: Fixes Data`.
- **LLM: Groq Fixes**
  - *Type & Role:* `@n8n/n8n-nodes-langchain.lmChatGroq` (Sub-node model provider)
  - *Configuration:* Model: `qwen/qwen3.8-27b`. Uses credentials ID `AnindwaoyRCy8KlP` (Groq).
  - *Connections:* Connects to `AI: Recommend Fixes` via `ai_languageModel`.
- **Parser: Fixes Output**
  - *Type & Role:* `@n8n/n8n-nodes-langchain.outputParserStructured` (Sub-node parser)
  - *Configuration:* Manual JSON schema expecting an `exceptions` array with `exception_type` and `upstream_fixes` (string array).
  - *Connections:* Connects to `AI: Recommend Fixes` via `ai_outputParser`.
- **Extract: Fixes Data**
  - *Type & Role:* `n8n-nodes-base.set` (Data transformation node)
  - *Configuration:* Extracts specific exception categories (`Missing Receipt`, `Price Variance`, `Tax Mismatch`, `Missing PO`) into dedicated object fields using optional chaining with fallback empty arrays (`?? []`). Ignores conversion errors.
  - *Expressions/Variables:* `{{ $json.output.exceptions.find(e => e.exception_type === "...")?.upstream_fixes ?? [] }}`
  - *Connections:* Input: `AI: Recommend Fixes`; Output: `Generate Markdown Report`.
  - *Edge Cases/Failures:* Returns empty arrays if the exception category strings do not exactly match the AI model's output values.

---

#### 2.4 Report Generation & Formatting
**Overview:** Compiles the extracted fix arrays into a formatted Markdown document, then parses the Markdown line-by-line into native JSON blocks compatible with the Notion API.

**Nodes Involved:**
- `Generate Markdown Report`
- `Convert Markdown to Blocks`

**Node Details:**
- **Generate Markdown Report**
  - *Type & Role:* `n8n-nodes-base.code` (JavaScript transformation node)
  - *Configuration:* Iterates over input entries, appends headers, dates, and bulleted lists into a single Markdown string (`notion_report`).
  - *Expressions/Variables:* `$input.first().json`
  - *Connections:* Input: `Extract: Fixes Data`; Output: `Convert Markdown to Blocks`.
- **Convert Markdown to Blocks**
  - *Type & Role:* `n8n-nodes-base.code` (JavaScript transformation node)
  - *Configuration:* Splits the Markdown string by newline characters and matches prefixes (`#`, `##`, `###`, `-`, `---`, `**Date:**`) to construct a native Notion blocks JSON array.
  - *Expressions/Variables:* `$input.first().json.notion_report`
  - *Connections:* Input: `Generate Markdown Report`; Output: `Notion: Create Page`.
  - *Edge Cases/Failures:* Malformed Markdown syntax could result in incorrect fallback paragraph rendering.

---

#### 2.5 Publishing & Notification
**Overview:** Creates a new database page shell in Notion, injects the converted block children via the Notion REST API, and sends a notification alert via Slack.

**Nodes Involved:**
- `Notion: Create Page`
- `Notion: Inject Blocks API`
- `Send a message`

**Node Details:**
- **Notion: Create Page**
  - *Type & Role:* `n8n-nodes-base.notion` (API integration node)
  - *Configuration:* Resource: Database Page. Targets database ID `3dc31eba-a729-8031-b1a3-000b75662fb8` (`N-170 Invoice Reports`). Sets title using expression.
  - *Expressions/Variables:* `{{$today.toFormat('yyyy-MM-dd')}}`
  - *Connections:* Input: `Convert Markdown to Blocks`; Output: `Notion: Inject Blocks API`.
  - *Edge Cases/Failures:* Integration lacks write permission on the specified Notion database.
- **Notion: Inject Blocks API**
  - *Type & Role:* `n8n-nodes-base.httpRequest` (HTTP request node)
  - *Configuration:* Method: `PATCH`. URL: `https://api.notion.com/v1/blocks/{{ $json.id }}/children`. Sends JSON body with Notion version header. Uses Notion API credentials.
  - *Expressions/Variables:* `{{ $json.id }}`, `{{ JSON.stringify($('Convert Markdown to Blocks').item.json.blocks) }}`
  - *Header Parameters:* `Notion-Version`: `2026-09-15`.
  - *Connections:* Input: `Notion: Create Page`; Output: `Send a message`.
  - *Edge Cases/Failures:* Payload size limits or incorrect Notion API version header.
- **Send a message**
  - *Type & Role:* `n8n-nodes-base.slack` (Messaging node)
  - *Configuration:* Default message delivery configuration.
  - *Connections:* Input: `Notion: Inject Blocks API`; Output: None (Terminal node).
  - *Edge Cases/Failures:* Slack credential expiration or unconfigured target channel.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Trigger: Weekly Schedule | scheduleTrigger | Triggers workflow weekly | None | Fetch: Invoice Exceptions | Data Ingestion & Prep |
| Fetch: Invoice Exceptions | googleSheets | Pulls raw invoice data | Trigger: Weekly Schedule | Group Exceptions | Data Ingestion & Prep |
| Group Exceptions | code | Groups exceptions and calculates metrics | Fetch: Invoice Exceptions | AI: Root Cause Analysis | Data Ingestion & Prep |
| AI: Root Cause Analysis | chainLlm | Analyzes root causes via LLM | Group Exceptions, LLM: Groq RCA, Parser: RCA Output | Extract: RCA Data | Root Cause Analysis (AI) |
| LLM: Groq RCA | lmChatGroq | Provides GPT model for RCA | None | AI: Root Cause Analysis | Root Cause Analysis (AI) |
| Parser: RCA Output | outputParserStructured | Enforces JSON schema for RCA | None | AI: Root Cause Analysis | Root Cause Analysis (AI) |
| Extract: RCA Data | set | Extracts RCA string from output | AI: Root Cause Analysis | AI: Recommend Fixes | Root Cause Analysis (AI) |
| AI: Recommend Fixes | chainLlm | Generates upstream fixes via LLM | Extract: RCA Data, LLM: Groq Fixes, Parser: Fixes Output | Extract: Fixes Data | Process Improvement (AI) |
| LLM: Groq Fixes | lmChatGroq | Provides Qwen model for fixes | None | AI: Recommend Fixes | Process Improvement (AI) |
| Parser: Fixes Output | outputParserStructured | Enforces JSON schema for fixes | None | AI: Recommend Fixes | Process Improvement (AI) |
| Extract: Fixes Data | set | Maps exception fixes to object fields | AI: Recommend Fixes | Generate Markdown Report | Process Improvement (AI) |
| Generate Markdown Report | code | Compiles fixes into Markdown string | Extract: Fixes Data | Convert Markdown to Blocks | Report Generation & Formatting |
| Convert Markdown to Blocks | code | Translates Markdown into Notion blocks | Generate Markdown Report | Notion: Create Page | Report Generation & Formatting |
| Notion: Create Page | notion | Creates report page in Notion | Convert Markdown to Blocks | Notion: Inject Blocks API | Publishing & Notification |
| Notion: Inject Blocks API | httpRequest | Pushes native blocks to Notion page | Notion: Create Page | Send a message | Publishing & Notification |
| Send a message | slack | Sends completion notification | Notion: Inject Blocks API | None | Publishing & Notification |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Entry Trigger:**
   - Add a **Schedule Trigger** node (`Trigger: Weekly Schedule`).
   - Configure the rule interval to trigger every week on Monday at 09:00 AM.
2. **Add Data Ingestion:**
   - Add a **Google Sheets** node (`Fetch: Invoice Exceptions`).
   - Connect `Trigger: Weekly Schedule` to `Fetch: Invoice Exceptions`.
   - Configure credentials and set Document ID to your target Google Sheet and Sheet Name to `Sheet1`.
3. **Add Data Grouping Logic:**
   - Add a **Code** node (`Group Exceptions`).
   - Connect `Fetch: Invoice Exceptions` to `Group Exceptions`.
   - Paste the grouping JavaScript snippet to aggregate rows by `Exception_Reason`, sum amounts, and collect invoice metadata.
4. **Configure Root Cause Analysis AI Chain:**
   - Add an **Advanced AI / Chain LLM** node (`AI: Root Cause Analysis`).
   - Connect `Group Exceptions` to the main input of `AI: Root Cause Analysis`.
   - Set the prompt template to ingest `{{ JSON.stringify($json.exception_summary, null, 2) }}` with a procurement analyst system prompt.
   - Add a **Groq Chat Model** node (`LLM: Groq RCA`), configure model `openai/gpt-oss-120b`, connect its output to `AI: Root Cause Analysis` (`ai_languageModel`), and link your Groq API credentials.
   - Add a **Structured Output Parser** node (`Parser: RCA Output`), supply the manual JSON schema for overall summary and categories, and connect its output to `AI: Root Cause Analysis` (`ai_outputParser`).
5. **Extract Root Cause Data:**
   - Add a **Set** node (`Extract: RCA Data`).
   - Connect `AI: Root Cause Analysis` to `Extract: RCA Data`.
   - Create an assignment with name `root_cause_analysis` and value `={{ $json.output }}`.
6. **Configure Process Improvement AI Chain:**
   - Add a second **Chain LLM** node (`AI: Recommend Fixes`).
   - Connect `Extract: RCA Data` to the main input of `AI: Recommend Fixes`.
   - Set the prompt template to ingest `{{ $json.root_cause_analysis }}` with a process consultant system prompt.
   - Add a second **Groq Chat Model** node (`LLM: Groq Fixes`), configure model `qwen/qwen3.8-27b`, connect its output to `AI: Recommend Fixes` (`ai_languageModel`), and link Groq API credentials.
   - Add a second **Structured Output Parser** node (`Parser: Fixes Output`), supply the manual JSON schema for upstream fixes, and connect its output to `AI: Recommend Fixes` (`ai_outputParser`).
7. **Extract Process Fixes:**
   - Add a **Set** node (`Extract: Fixes Data`).
   - Connect `AI: Recommend Fixes` to `Extract: Fixes Data`.
   - Assign object fields (`Missing Receipt`, `Price Variance`, `Tax Mismatch`, `Missing PO`) using optional chaining against `$json.output.exceptions`.
8. **Generate Report and Convert to Notion Blocks:**
   - Add a **Code** node (`Generate Markdown Report`). Connect `Extract: Fixes Data` to it. Paste the JavaScript code to compile a Markdown report string.
   - Add a second **Code** node (`Convert Markdown to Blocks`). Connect `Generate Markdown Report` to it. Paste the JavaScript code to parse the Markdown line-by-line into Notion block JSON structures.
9. **Configure Notion Publishing:**
   - Add a **Notion** node (`Notion: Create Page`). Connect `Convert Markdown to Blocks` to it. Set Resource to `Database Page`, select your target database ID, and set the title to `={{$today.toFormat('yyyy-MM-dd')}}`.
   - Add an **HTTP Request** node (`Notion: Inject Blocks API`). Connect `Notion: Create Page` to it. Set Method to `PATCH`, URL to `https://api.notion.com/v1/blocks/{{ $json.id }}/children`, add header `Notion-Version: 2026-09-15`, and set Body content to `={"children": {{ JSON.stringify($('Convert Markdown to Blocks').item.json.blocks) }}}`.
10. **Configure Slack Notification:**
    - Add a **Slack** node (`Send a message`). Connect `Notion: Inject Blocks API` to `Send a message`. Configure your Slack authentication and target channel.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Professional implementation and customization assistance | [WeblineIndia Contact Page](https://www.weblineindia.com/contact-us.html) |
