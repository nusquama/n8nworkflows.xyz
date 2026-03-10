Generate an AI YouTube trend report with GPT-4o, Google Sheets and PDF.co

https://n8nworkflows.xyz/workflows/generate-an-ai-youtube-trend-report-with-gpt-4o--google-sheets-and-pdf-co-13659


# Generate an AI YouTube trend report with GPT-4o, Google Sheets and PDF.co

This technical reference document provides a detailed analysis of the n8n workflow: **AI YouTube Trend Intelligence Report**.

---

### 1. Workflow Overview
This workflow is an end-to-end competitive intelligence pipeline designed for YouTube content creators and strategists in the AI and automation niche. It automates the discovery of high-performing content, performs statistical analysis, generates a dashboard in Google Sheets, and utilizes AI to provide actionable content recommendations.

**Key Logical Blocks:**
*   **1.1 Input & Discovery:** Triggers weekly and generates search queries based on 10 predefined high-intent keywords.
*   **1.2 Data Enrichment:** Fetches raw video metadata and detailed statistics (views, likes, comments) via the YouTube Data API.
*   **1.3 Ranking & Relevance Engine:** Filters and ranks videos using a custom engagement rate formula and keyword-based relevance checks.
*   **1.4 Spreadsheet Intelligence:** Creates a multi-tab Google Sheets dashboard containing Channel Stats, Top Videos, and Weekly Summaries.
*   **1.5 AI Synthesis:** Uses GPT-4o to analyze the top-ranked data and generate an executive summary and trend report.
*   **1.6 Branded PDF Generation:** Builds a dynamic HTML report with charts (via QuickChart.io), converts it to a PDF (via PDF.co), and emails it.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Discovery
*   **Overview:** Sets the timing and the scope of the search.
*   **Nodes Involved:** `Weekly Schedule`, `Set Keywords`.
*   **Node Details:**
    *   **Weekly Schedule:** A Cron-based trigger set to run every Monday at 07:00 AM.
    *   **Set Keywords:** A Code node that outputs a list of 10 keywords (e.g., "AI automation 2025", "n8n automation tutorial") and calculates a ISO string for the `publishedAfter` parameter (current date minus 7 days).

#### 2.2 YouTube Data Fetch
*   **Overview:** Interacts with external APIs to gather video data.
*   **Nodes Involved:** `Search YouTube`, `Flatten Videos`, `Prep ID Batches`, `Get Video Stats`.
*   **Node Details:**
    *   **Search YouTube:** HTTP Request node calling the YouTube v3 Search API. It retrieves up to 50 videos per keyword, ordered by `viewCount`. 
    *   **Flatten Videos:** Removes duplicates (using a `Set` on `videoId`) and normalizes snippet data.
    *   **Prep ID Batches:** Groups unique video IDs into strings of 50 to minimize API calls to the video statistics endpoint.
    *   **Get Video Stats:** HTTP Request node fetching `statistics`, `snippet`, and `contentDetails` for the specific batch of IDs.

#### 2.3 Ranking & Relevance Engine
*   **Overview:** Transforms raw statistics into ranked intelligence.
*   **Nodes Involved:** `Rank Videos`.
*   **Node Details:**
    *   **Rank Videos (Code):** 
        *   **Logic:** Calculates engagement rate: `((likes + comments) / views) * 100`. 
        *   **Filtering:** Filters for high relevance using a secondary keyword list (e.g., "agentic", "orchestration", "llm").
        *   **Ranking:** Sorts by view count and identifies the Top 10 and Top 50 videos.
        *   **Aggregation:** Maps channel performance and keyword volume distribution.

#### 2.4 Google Sheets Export
*   **Overview:** Records the data for long-term tracking.
*   **Nodes Involved:** `Create Analytics Spreadsheet`, `Setup Tabs`, `Append Channel Stats`, `Append Top Videos`, `Append Weekly Summary`, `Finalize Spreadsheet`.
*   **Node Details:**
    *   **Setup Tabs (HTTP Request):** Uses the Google Sheets BatchUpdate API to rename the default sheet and create/header two additional tabs: "Top Videos" and "Weekly Summary".
    *   **Data Preparation:** Three Code nodes (`Prep Channel Stats`, etc.) format the ranked data into flat JSON arrays compatible with the Google Sheets "Append" operation.

#### 2.5 AI Synthesis & Branded Delivery
*   **Overview:** Interprets data and creates the final document.
*   **Nodes Involved:** `Prep AI Prompt`, `Analyze Trends with AI`, `Build HTML Report1`, `PDFco Api`, `Download PDF`, `Send Report Email`.
*   **Node Details:**
    *   **Prep AI Prompt:** Constructs a structured prompt containing the top 20 trending videos and top 5 channels. It forces the AI to output strictly valid JSON.
    *   **Analyze Trends with AI (OpenAI):** Uses the GPT-4o model to generate a content strategy report.
    *   **Build HTML Report1:** A massive Code node that embeds CSS, base64 logo assets, and dynamic Chart URLs (QuickChart.io) to create a professional multi-page HTML document.
    *   **PDFco Api:** Converts the generated HTML into a Letter-sized PDF with zero margins.
    *   **Send Report Email:** Attaches the binary PDF and provides the Google Sheets link in the message body.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Weekly Schedule | ScheduleTrigger | Workflow Trigger | None | Set Keywords | Section 1 - Trigger & Keywords |
| Set Keywords | Code | Keyword Definition | Weekly Schedule | Search YouTube | Section 1 - Trigger & Keywords |
| Search YouTube | HttpRequest | API Data Fetch | Set Keywords | Flatten Videos | Section 2 - YouTube Data Fetch |
| Flatten Videos | Code | Deduplication | Search YouTube | Prep ID Batches | Section 2 - YouTube Data Fetch |
| Prep ID Batches | Code | Batch Processing | Flatten Videos | Get Video Stats | Section 2 - YouTube Data Fetch |
| Get Video Stats | HttpRequest | Stats Retrieval | Prep ID Batches | Rank Videos | Section 2 - YouTube Data Fetch |
| Rank Videos | Code | Data Ranking | Get Video Stats | Create Spreadsheet | Section 3 - Rank & AI Analysis |
| Create Analytics Spreadsheet | GoogleSheets | Sheet Creation | Rank Videos | Setup Tabs | Section 5 - Google Sheets Export |
| Setup Tabs | HttpRequest | Sheet Formatting | Create Spreadsheet | Prep Channel Stats | Section 5 - Google Sheets Export |
| Prep Channel Stats | Code | Data Prep | Setup Tabs | Append Channel Stats | Section 5 - Google Sheets Export |
| Append Channel Stats | GoogleSheets | Data Write | Prep Channel Stats | Prep Top Videos | Section 5 - Google Sheets Export |
| Prep Top Videos | Code | Data Prep | Append Channel Stats | Append Top Videos | Section 5 - Google Sheets Export |
| Append Top Videos | GoogleSheets | Data Write | Prep Top Videos | Prep Weekly Summary | Section 5 - Google Sheets Export |
| Prep Weekly Summary | Code | Data Prep | Append Top Videos | Append Weekly Summary | Section 5 - Google Sheets Export |
| Append Weekly Summary | GoogleSheets | Data Write | Prep Weekly Summary | Finalize Spreadsheet | Section 5 - Google Sheets Export |
| Finalize Spreadsheet | Code | URL Generation | Append Weekly Summary | Prep AI Prompt | Section 5 - Google Sheets Export |
| Prep AI Prompt | Code | AI Orchestration | Finalize Spreadsheet | Analyze Trends | Section 3 - Rank & AI Analysis |
| Analyze Trends with AI | OpenAI | Insight Generation | Prep AI Prompt | Build HTML Report | Section 3 - Rank & AI Analysis |
| Build HTML Report1 | Code | UI/UX Generation | Analyze Trends | PDFco Api | Section 4 - PDF Report & Email |
| PDFco Api | PDFco Api | PDF Conversion | Build HTML Report | Download PDF | Section 4 - PDF Report & Email |
| Download PDF | HttpRequest | Binary Retrieval | PDFco Api | Send Report Email | Section 4 - PDF Report & Email |
| Send Report Email | Gmail | Report Delivery | Download PDF | None | Section 4 - PDF Report & Email |

---

### 4. Reproducing the Workflow from Scratch

1.  **Credentials Setup:**
    *   **YouTube API:** Create a project in Google Cloud Console and enable "YouTube Data API v3". Use an "HTTP Query Auth" credential in n8n.
    *   **Google Sheets:** Set up Google Sheets OAuth2.
    *   **OpenAI:** Obtain an API Key.
    *   **PDF.co:** Obtain an API Key.
    *   **Gmail:** Set up Gmail OAuth2.

2.  **Discovery Logic:**
    *   Create a **Schedule Trigger** (Weekly).
    *   Add a **Code Node** defining an array of strings (keywords) and a variable for the date 7 days ago.

3.  **YouTube Integration:**
    *   Add an **HTTP Request** node to `https://www.googleapis.com/youtube/v3/search`. Pass the keyword in query params.
    *   Follow with a **Code Node** to flatten the nested JSON items and remove duplicates using `videoId`.
    *   Add a **Code Node** to batch IDs into comma-separated strings (limit 50).
    *   Add a second **HTTP Request** to `https://www.googleapis.com/youtube/v3/videos` to get `statistics`.

4.  **Intelligence Engine:**
    *   Add a **Code Node** to calculate engagement rates. Use `Array.prototype.sort` to find the highest view counts.
    *   Implement an "Aggregation" object to group videos by channel for the summary.

5.  **Dashboard Creation:**
    *   Add a **Google Sheets** node (Operation: Create).
    *   Add an **HTTP Request** node calling the Google Sheets `batchUpdate` endpoint. Send a JSON body to create sheets named "Top Videos" and "Weekly Summary" and add header rows to each.
    *   Chain three **Google Sheets** (Append) nodes to populate these tabs.

6.  **AI Analysis:**
    *   Add a **Code Node** to transform the ranked data into a text summary.
    *   Add the **OpenAI Node** (Model: gpt-4o). Set the prompt to ask for JSON output.

7.  **Final Document:**
    *   Add a **Code Node** containing a full HTML template. Inside, use expressions to map the AI's JSON summary and the ranked video list.
    *   Use the **PDF.co Node** (HTML to PDF) to process the HTML string.
    *   Add an **HTTP Request** node to download the URL provided by PDF.co as a binary file.
    *   Add the **Gmail Node** to send the message with the binary file as an attachment.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Branded Report Design | Built using QuickChart.io for server-side image rendering. [QuickChart.io](https://quickchart.io) |
| YouTube API Limits | Be aware of quota limits (10,000 units/day). This workflow uses batching to stay efficient. |
| PDF.co paper size | Configured for "Letter" size with 0 margins for professional full-bleed design. |
| Project Credits | Created by Milo Bravo | BRaiA Labs. [LinkedIn Profile](https://linkedin.com/in/MiloBravo/) |
| Feedback/Support | Integrated feedback form in the original template. [Tally Form](https://tally.so/r/EkKGgB) |