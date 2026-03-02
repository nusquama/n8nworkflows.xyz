Research web topics and email a Claude report via Gmail using SerpApi, Jina.ai and Firecrawl

https://n8nworkflows.xyz/workflows/research-web-topics-and-email-a-claude-report-via-gmail-using-serpapi--jina-ai-and-firecrawl-13608


# Research web topics and email a Claude report via Gmail using SerpApi, Jina.ai and Firecrawl

This technical reference document provides a detailed breakdown of the **Self-Healing Research Agent** workflow. This system automates web research by intelligently scraping multiple sources, employing a fallback mechanism for blocked content, and synthesizing findings into a professional executive report delivered via email.

---

### 1. Workflow Overview

The workflow is designed to transform a single research topic into a structured HTML report. It follows a linear progression through four main functional stages:

*   **1.1 Input & Search:** Captures user requirements via a form and retrieves filtered search results from Google via SerpApi.
*   **1.2 Self-Healing Scraping Loop:** Iterates through each URL. It primarily uses Jina Reader for scraping. If the content is blocked or contains a CAPTCHA, it automatically triggers a "self-healing" fallback.
*   **1.3 Fallback & Secondary Analysis:** Uses Firecrawl as a secondary scraping engine to bypass blocks encountered in the primary stage.
*   **1.4 Synthesis & Delivery:** Aggregates all successful analyses, uses Claude to write an executive summary, converts the result into styled HTML, and sends it via Gmail.

---

### 2. Block-by-Block Analysis

#### 2.1 Input & Search
Retrieves the research topic and prepares a list of high-quality URLs.
*   **Nodes:** `Research Form`, `Set Config`, `SerpApi Search`, `Parse & Filter URLs`.
*   **Configuration:**
    *   **Form:** Collects topic, number of sources (5-7), and recipient email.
    *   **SerpApi:** Queries Google. Note: It requests `numSources * 2 + 3` results to ensure enough links remain after filtering.
    *   **Filtering Logic:** A Code node removes "junk" domains (Reddit, YouTube, social media, PDFs) to ensure the AI analyzes text-heavy articles rather than discussions or videos.
*   **Edge Cases:** If no valid URLs are found after filtering, the workflow throws an error.

#### 2.2 Self-Healing Loop (Primary Scrape)
Processes URLs one by one to avoid rate limits and handle individual failures gracefully.
*   **Nodes:** `Loop Over URLs`, `Jina Reader (Primary)`, `Normalize Jina Response`, `Page Analyzer (Round 1)`, `Claude Extractor`.
*   **Analysis Logic:**
    *   **Jina Reader:** Fetches the URL content in Markdown format.
    *   **Claude Extractor:** A "Chain LLM" node using `claude-sonnet-4-5`. It is instructed to return `RETRY_NEEDED` if it detects a CAPTCHA or insufficient content.
*   **Failure Handling:** The Jina node is set to "Continue Regular Output" to prevent a single 404 from stopping the entire workflow.

#### 2.3 Fallback Mechanism
Triggered only when the primary scraper is blocked.
*   **Nodes:** `Retry Needed? (IF)`, `Wait Before Retry`, `Firecrawl (Fallback)`, `Normalize Firecrawl`, `Page Analyzer (Round 2)`, `Claude Re-Analyzer`, `Still Blocked? (IF)`, `Mark as Failed`.
*   **Configuration:**
    *   **Wait Node:** Pauses for 3 seconds to reduce bot-detection probability.
    *   **Firecrawl:** Uses a POST request to the Firecrawl API to scrape content that Jina could not access.
*   **Outcome:** If Firecrawl also fails, the source is marked as "Unavailable" in the final report instead of crashing the process.

#### 2.4 Synthesis & Delivery
Consolidates all individual analyses into a final product.
*   **Nodes:** `Merge Page Results`, `Enrich & Forward`, `Aggregate Summaries`, `Build Synthesis Prompt`, `Synthesize Report`, `Build HTML Report`, `Send Gmail`.
*   **Technical Details:**
    *   **Build Synthesis Prompt:** A Code node that converts the array of analyses into a structured prompt, providing Claude with "Evidence" from each source.
    *   **Build HTML Report:** Uses custom JavaScript to convert Markdown to a professional HTML template with embedded CSS (Georgia font, responsive tables, and colored headers).
    *   **Gmail:** Sends the final HTML string with a dynamic subject line including the date.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Research Form** | Form Trigger | User Interface | (None) | Set Config | 1 — Input & search |
| **Set Config** | Set | Variables Setup | Research Form | SerpApi Search | 1 — Input & search |
| **SerpApi Search** | HTTP Request | Search Engine | Set Config | Parse & Filter URLs | 1 — Input & search |
| **Parse & Filter URLs** | Code | Data Cleaning | SerpApi Search | Loop Over URLs | 1 — Input & search |
| **Loop Over URLs** | Split In Batches | Batch Control | Parse & Filter | Jina Reader, Aggregate | 2 — Self-healing loop |
| **Jina Reader (Primary)**| HTTP Request | Primary Scraper | Loop Over URLs | Normalize Jina | 2 — Self-healing loop |
| **Normalize Jina** | Code | Data Formatting | Jina Reader | Page Analyzer R1 | 2 — Self-healing loop |
| **Page Analyzer R1** | Chain LLM | Content Analysis| Normalize Jina | Combine Analysis R1 | 2 — Self-healing loop |
| **Claude Extractor** | Anthropic Chat| AI Brain | Page Analyzer R1 | (Internal) | 2 — Self-healing loop |
| **Combine Analysis R1**| Code | Data Merging | Page Analyzer R1 | Retry Needed? | 2 — Self-healing loop |
| **Retry Needed? (IF)** | If | Flow Control | Combine R1 | Wait, Merge Results | 2 — Self-healing loop |
| **Wait Before Retry** | Wait | Delay | Retry Needed? | Firecrawl | 3 — Firecrawl fallback |
| **Firecrawl (Fallback)**| HTTP Request | Backup Scraper | Wait Before Retry | Normalize Firecrawl | 3 — Firecrawl fallback |
| **Normalize Firecrawl** | Code | Data Formatting | Firecrawl | Page Analyzer R2 | 3 — Firecrawl fallback |
| **Page Analyzer R2** | Chain LLM | Fallback Analysis| Normalize Firecrawl| Combine Analysis R2 | 3 — Firecrawl fallback |
| **Combine Analysis R2**| Code | Data Merging | Page Analyzer R2 | Still Blocked? | 3 — Firecrawl fallback |
| **Still Blocked? (IF)** | If | Flow Control | Combine R2 | Mark Failed, Merge | 3 — Firecrawl fallback |
| **Mark as Failed** | Set | Error Flagging | Still Blocked? | Merge Page Results | 3 — Firecrawl fallback |
| **Merge Page Results** | Merge | Path Re-entry | Retry?, Still?, Mark| Enrich & Forward | 2 — Self-healing loop |
| **Enrich & Forward** | Code | Loop Cleanup | Merge Results | Loop Over URLs | 2 — Self-healing loop |
| **Aggregate Summaries**| Aggregate | Data Collection | Loop Over URLs | Build Synthesis | 4 — Synthesis & delivery|
| **Build Synthesis** | Code | Prompt Prep | Aggregate | Synthesize Report | 4 — Synthesis & delivery|
| **Synthesize Report** | Chain LLM | Report Writing | Build Synthesis | Build HTML Report | 4 — Synthesis & delivery|
| **Claude Synthesizer** | Anthropic Chat| AI Brain | Synthesize Report| (Internal) | 4 — Synthesis & delivery|
| **Build HTML Report** | Code | Styling | Synthesize Report| Send Gmail | 4 — Synthesis & delivery|
| **Send Gmail** | Gmail | Delivery | Build HTML Report | (None) | 4 — Synthesis & delivery|
| **Error Trigger** | Error Trigger | Error Monitoring | (None) | Format Error | Catches any workflow failure |
| **Format Error** | Set | Error Formatting | Error Trigger | (None) | Catches any workflow failure |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: Credentials Setup
*   **SerpApi:** Get an API key from `serpapi.com`.
*   **Firecrawl:** Get an API key from `firecrawl.dev`.
*   **Anthropic:** Set up Claude API credentials in n8n.
*   **Gmail:** Set up Gmail OAuth2 credentials.

#### Step 2: Input and Configuration
1.  Create a **Form Trigger** with three fields: `Research Topic` (String), `Number of Sources` (Dropdown: 5, 6, 7), and `Recipient Email` (Email).
2.  Add a **Set Node** to map form values to internal variables (`topic`, `numSources`, `recipientEmail`). Add a `runId` using the expression `{{ $now.toFormat('yyyyMMddHHmmss') }}`.
3.  Add an **HTTP Request Node** for SerpApi. Use `GET https://serpapi.com/search`. Parameters: `q` (topic), `num` (numSources * 2), `engine` (google).

#### Step 3: URL Filtering
1.  Add a **Code Node** to filter search results. Use a list of forbidden strings (e.g., `youtube.com`, `.pdf`).
2.  Map the remaining links to a new array of objects containing `url`, `title`, and original config data.

#### Step 4: The Processing Loop
1.  Add a **Split In Batches Node** with a batch size of 1.
2.  Add an **HTTP Request Node** for Jina: `GET https://r.jina.ai/{{ $json.url }}`. Set the timeout to 30s.
3.  Add a **Chain LLM Node** (Claude) with instructions to analyze the text. Crucial: Tell the AI to output exactly `RETRY_NEEDED` if the content is unusable.

#### Step 5: Self-Healing Logic
1.  Add an **If Node** to check if the output equals `RETRY_NEEDED`.
2.  On the **True** path: Add a **Wait Node** (3s), followed by an **HTTP Request Node** for Firecrawl (`POST https://api.firecrawl.dev/v1/scrape`).
3.  Add a second **Chain LLM Node** (Claude) to analyze the Firecrawl output.
4.  Add a **Merge Node** (Append mode) to collect the result from either the primary Jina path or the Firecrawl fallback path.
5.  Connect the end of the loop body back to the **Split In Batches Node**.

#### Step 6: Synthesis and Mailing
1.  Connect the **Done** output of the loop to an **Aggregate Node** to combine all analyses.
2.  Add a **Code Node** to build a single prompt containing all "Evidence" items.
3.  Add a **Chain LLM Node** (Claude) to write the final executive report in Markdown.
4.  Add a **Code Node** with a JavaScript function to wrap the Markdown in HTML/CSS.
5.  Add a **Gmail Node** to send the final HTML to the `recipientEmail`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **SerpApi Setup** | Sign up at [serpapi.com](https://serpapi.com) |
| **Firecrawl Setup** | Sign up at [firecrawl.dev](https://firecrawl.dev) |
| **Jina.ai Usage** | Primary scraper; used via `r.jina.ai` prefix. No API key required for basic use. |
| **Error Handling** | Go to Workflow Settings -> Error Workflow to enable the built-in handler. |
| **Custom Styling** | The `Build HTML Report` node uses a custom internal CSS stylesheet for professional email rendering. |