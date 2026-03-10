Generate business research reports with Claude, web search and Google Drive

https://n8nworkflows.xyz/workflows/generate-business-research-reports-with-claude--web-search-and-google-drive-13805


# Generate business research reports with Claude, web search and Google Drive

This document provides a technical analysis and structural breakdown of the **n8n OpenCWL AI — Business Research & Report Generator** workflow.

---

### 1. Workflow Overview
The workflow is a fully automated Business Intelligence (BI) agent. It triggers via a webhook, collects real-time data from web searches, news outlets, and financial markets, and then processes this data through a multi-pass Claude AI (Anthropic) pipeline to generate a professional, structured executive report.

**Logical Blocks:**
*   **1.1 Request Intake & Validation:** Captures incoming data and ensures required parameters exist.
*   **1.2 Parallel Data Collection:** Simultaneously queries Google Search, NewsAPI, and Alpha Vantage.
*   **1.3 Data Synthesis:** Merges and cleans disparate data sources into a structured context for the AI.
*   **1.4 Multi-Pass AI Pipeline:** A 3-stage analysis (Research -> Strategy -> Reporting) using Claude 3.5 Sonnet.
*   **1.5 Multi-Channel Delivery:** Distributes the final report via Google Drive, Slack, Email, and logs metadata to Google Sheets.

---

### 2. Block-by-Block Analysis

#### 2.1 Request Intake & Validation
This block prepares the environment and validates the payload.
*   **Nodes:** `Receive Research Request`, `Set Credentials and Config`, `Validate Input`, `Return Validation Error`.
*   **Details:**
    *   **Receive Research Request (Webhook):** Listens for POST requests at `/business-report`.
    *   **Set Credentials and Config (Set):** Normalizes inputs (topic, company, ticker, industry, reportType, recipientEmail). It also serves as the central repository for API keys (Anthropic, Serper, NewsAPI, Alpha Vantage, SendGrid).
    *   **Validate Input (IF):** Checks if the `topic` field is present.
    *   **Edge Cases:** If `topic` is empty, it returns a 400 error immediately via `Return Validation Error`.

#### 2.2 Parallel Data Collection
Aggregates raw intelligence from three distinct sources.
*   **Nodes:** `Search Google via Serper`, `Fetch News Articles`, `Fetch Market Data`.
*   **Details:**
    *   **Search Google (HTTP Request):** Queries Serper.dev for the top 10 Google results.
    *   **Fetch News (HTTP Request):** Queries NewsAPI for articles from the last 7 days related to the topic.
    *   **Fetch Market Data (HTTP Request):** Queries Alpha Vantage for company overview metrics. If no ticker is provided, it defaults to "IBM" (or fails gracefully).
    *   **Error Handling:** All nodes use `continueOnFail: true`, ensuring the workflow proceeds even if one source is unavailable.

#### 2.3 Data Synthesis & Processing
Prepares the data for the LLM.
*   **Nodes:** `Merge and Clean Data`.
*   **Details:**
    *   **Merge and Clean Data (Code):** A JavaScript node that extracts titles, snippets, and URLs from search/news and parses financial ratios (P/E, Market Cap, etc.). It creates three distinct context variables: `searchContext`, `newsContext`, and `financialContext`.

#### 2.4 Multi-Pass AI Pipeline (Claude)
The core intelligence engine that builds the report incrementally.
*   **Nodes:** `Claude Pass 1`, `Extract Pass 1`, `Wait For Process`, `Claude Pass 2`, `Extract Pass 2`, `Claude Pass 3`, `Format Final Report Package`.
*   **Details:**
    *   **Pass 1 (Research Synthesis):** Claude identifies facts, trends, and sentiment from raw data.
    *   **Pass 2 (Strategic SWOT):** Claude uses Pass 1 results to build a SWOT analysis and risk matrix.
    *   **Wait For Process (Wait):** A brief pause to ensure API rate limits or processing sequences are respected.
    *   **Pass 3 (Final Report):** Claude writes the full Markdown report based on previous steps.
    *   **Format Final Report (Code):** Uses Regex to extract metadata injected by the AI (Title, Summary, Score) and cleans the final Markdown body.

#### 2.5 Multi-Channel Delivery & Logging
Outputs the results to various stakeholders.
*   **Nodes:** `Save Report to Google Drive`, `Post Summary to Slack`, `Email Full Report via SendGrid`, `Log to Google Sheets`, `Return Report Response`.
*   **Details:**
    *   **Save to Drive (HTTP Request):** Performs a multipart upload to save the Markdown file.
    *   **Post to Slack (HTTP Request):** Sends a Block Kit formatted summary including a "Research Score" and token usage.
    *   **Email (HTTP Request):** Sends the full Markdown report via SendGrid API.
    *   **Log to Sheets (Google Sheets):** Appends a row with metadata for auditing.
    *   **Response (Respond to Webhook):** Returns a JSON object with a report preview and delivery status.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Receive Research Request | Webhook | Entry Point | - | Set Credentials and Config | Stage 1 — Request Intake & Validation |
| Set Credentials and Config | Set | Configuration | Receive Research Request | Validate Input | Stage 1 — Request Intake & Validation |
| Validate Input | IF | Validation | Set Credentials and Config | Search Google..., Fetch News..., Fetch Market Data | Stage 1 — Request Intake & Validation |
| Return Validation Error | Respond to Webhook | Error Handling | Validate Input | - | Stage 1 — Request Intake & Validation |
| Search Google via Serper | HTTP Request | Data Source | Validate Input | Merge and Clean Data | Serper returns top 10 organic Google results. |
| Fetch News Articles | HTTP Request | Data Source | Validate Input | Merge and Clean Data | NewsAPI returns latest 10 articles from past 7 days. |
| Fetch Market Data | HTTP Request | Data Source | Validate Input | Merge and Clean Data | Alpha Vantage returns company overview and key financial metrics. |
| Merge and Clean Data | Code | Data Transformation | Google Search, News, Market Data | Claude Pass 1 | Stage 3 — Data Merge & Processing |
| Claude Pass 1 Research Synthesis | HTTP Request | AI Analysis | Merge and Clean Data | Extract Pass 1 Output | Pass 1 reads all raw data and extracts key facts. |
| Extract Pass 1 Output | Code | Data Extraction | Claude Pass 1 | Wait For Process | Stage 4 — 3-Pass Claude AI Analysis Pipeline |
| Wait For Process | Wait | Flow Control | Extract Pass 1 Output | Claude Pass 2 | Stage 4 — 3-Pass Claude AI Analysis Pipeline |
| Claude Pass 2 Strategic SWOT | HTTP Request | AI Analysis | Wait For Process | Extract Pass 2 Output | Pass 2 takes Pass 1 output and performs deep SWOT analysis. |
| Extract Pass 2 Output | Code | Data Extraction | Claude Pass 2 | Claude Pass 3 | Stage 4 — 3-Pass Claude AI Analysis Pipeline |
| Claude Pass 3 Write Final Report | HTTP Request | AI Analysis | Extract Pass 2 Output | Format Final Report Package | Pass 3 takes both previous outputs and writes final report. |
| Format Final Report Package | Code | Formatting | Claude Pass 3 | Drive, Slack, SendGrid, Sheets | Stage 5 — Multi-Channel Delivery & Logging |
| Save Report to Google Drive | HTTP Request | Storage | Format Final Report | Return Report Response | Stage 5 — Multi-Channel Delivery & Logging |
| Post Summary to Slack | HTTP Request | Notification | Format Final Report | Return Report Response | Stage 5 — Multi-Channel Delivery & Logging |
| Email Full Report via SendGrid | HTTP Request | Delivery | Format Final Report | Return Report Response | Stage 5 — Multi-Channel Delivery & Logging |
| Log to Google Sheets | Google Sheets | Logging | Format Final Report | Return Report Response | Stage 5 — Multi-Channel Delivery & Logging |
| Return Report Response | Respond to Webhook | Exit Point | Drive, Slack, Email, Sheets | - | Final webhook response returns JSON. |

---

### 4. Reproducing the Workflow from Scratch

1.  **Trigger Setup:** Create a **Webhook node** (POST) with the path `business-report`. Set Response Mode to "When Last Node Finishes".
2.  **Configuration:** Add a **Set node**. Create fields for `topic`, `company`, `ticker`, and `industry`. Add placeholders for `ANTHROPIC_KEY`, `SERPER_KEY`, `NEWSAPI_KEY`, `ALPHAV_KEY`, `SENDGRID_KEY`, `SLACK_WEBHOOK`, `DRIVE_FOLDER_ID`, and `SHEET_ID`.
3.  **Input Logic:** Add an **IF node** to check if `topic` is not empty. Connect a **Respond to Webhook node** (Status 400) to the "False" output.
4.  **Parallel Search:**
    *   **Serper:** HTTP Request (POST) to `https://google.serper.dev/search`. Send `q` as `topic + company + 2025`.
    *   **NewsAPI:** HTTP Request (GET) to `https://newsapi.org/v2/everything`. Filter by `q` and set `from` to 7 days ago using a JS date expression.
    *   **Alpha Vantage:** HTTP Request (GET) to `https://www.alphavantage.co/query` with `function=OVERVIEW` and `symbol={{ticker}}`.
    *   *Note:* Set **Continue on Fail** to true for all three.
5.  **Data Processing:** Add a **Code node** to consolidate the JSON from the three requests. Map the arrays into clean strings (e.g., `searchResults.map(...)`).
6.  **AI Pass 1 (Research):** Add an **HTTP Request node** (POST) to Anthropic's API. Use model `claude-sonnet-4-20250514`. Instruct it to synthesize facts and trends from the context.
7.  **AI Pass 2 (Strategy):** Add an **HTTP Request node** (POST) to Anthropic. Pass the output of Pass 1 and request a SWOT analysis and Risk Matrix.
8.  **AI Pass 3 (Report Writing):** Add an **HTTP Request node** (POST) to Anthropic. Pass outputs of Pass 1 and 2. Instruct it to output Markdown with specific tags at the end: `END_REPORT_TITLE:`, `END_REPORT_SUMMARY:`, and `END_REPORT_SCORE:`.
9.  **Formatting:** Add a **Code node**. Use Regex to find the "END_REPORT" tags, store them as variables, and remove them from the main report body.
10. **Delivery:**
    *   **Google Drive:** HTTP Request (POST) to Drive API using `multipart/related` to upload the Markdown string as a `.md` file.
    *   **Slack:** HTTP Request (POST) to your Slack Webhook URL using Block Kit.
    *   **SendGrid:** HTTP Request (POST) to `/v3/mail/send` with the Markdown content.
    *   **Google Sheets:** **Google Sheets node** (Append) to log the report ID and score.
11. **Final Response:** Connect all delivery nodes to a final **Respond to Webhook node** that returns a success JSON including the `reportPreview`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Search Engine API** | [serper.dev](https://serper.dev) |
| **News Aggregator API** | [newsapi.org](https://newsapi.org) |
| **Financial Data API** | [alphavantage.co](https://www.alphavantage.co) |
| **AI Model Used** | Claude 3.5 Sonnet (claude-sonnet-4-20250514) |
| **Project Credits** | n8n OpenCWL AI Workflow |
| **Auth Requirement** | Google Drive requires a valid OAuth2 token or Service Account for the HTTP Request node. |
| **Performance** | Typical execution time is 90–180 seconds due to sequential LLM calls. |