Pinterest Keyword-Based Content Scraper with AI Agent & BrightData Automation

https://n8nworkflows.xyz/workflows/pinterest-keyword-based-content-scraper-with-ai-agent---brightdata-automation-5193


# Pinterest Keyword-Based Content Scraper with AI Agent & BrightData Automation

### 1. Workflow Overview

This workflow automates Pinterest content scraping based on user-provided keywords, leveraging AI to orchestrate scraping jobs and data extraction. It integrates BrightData’s Pinterest scraping API to collect content snapshots, monitors scraping progress, fetches the completed data, formats and extracts relevant fields, and finally saves the structured results into Google Sheets for further use or reporting.

The workflow is logically divided into these main blocks:

- **1.1 Input Reception:** Receives keywords via a user form.
- **1.2 AI Agent Control:** Uses an AI agent powered by the Anthropic Claude Sonnet 4 model to manage the scraping job lifecycle, including launching the job, checking status, and fetching data.
- **1.3 BrightData Scraping Interaction:** Sends requests to BrightData API to start scraping, monitor progress, and retrieve snapshot data.
- **1.4 Data Formatting and Storage:** Parses the raw scraped data into structured fields and saves the output to Google Sheets.
- **1.5 Utility and Documentation Nodes:** Includes wait timer (currently disabled) and multiple sticky notes for documentation and user guidance.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
Handles user input of keywords through a web form which triggers the start of the scraping process.

**Nodes Involved:**  
- Pinterest Keyword Input  
- Sticky Note6 (Keyword Input Form)

**Node Details:**

- **Pinterest Keyword Input**  
  - Type: Form Trigger  
  - Configuration: Webhook-based form with a single required field labeled “Keywords” to accept user input.  
  - Input: External HTTP request (user form submission)  
  - Output: JSON object containing the keyword(s) submitted  
  - Edge Cases: Missing required field triggers rejection; malformed requests may fail webhook trigger.  
  - Notes: Entry point for the workflow.

- **Sticky Note6**  
  - Type: Sticky Note  
  - Role: Documentation for the input form node, explaining its purpose as the initiation trigger.

---

#### 2.2 AI Agent Control

**Overview:**  
An AI-powered agent orchestrates the scraping job lifecycle: it processes keywords, triggers scraping, polls status until completion, and retrieves scraped data.

**Nodes Involved:**  
- Anthropic Chat Model  
- Keyword-based Scraping Agent  
- Wait for 1 Minute (disabled)  
- Sticky Note (AI Agent explanation)  
- Sticky Note5 (Claude Sonnet 4 model explanation)  
- Sticky Note1 (Wait Timer explanation)

**Node Details:**

- **Anthropic Chat Model**  
  - Type: LangChain Chat Model (Anthropic Claude Sonnet 4)  
  - Configuration: Uses "claude-sonnet-4-20250514" model to process keyword input intelligently.  
  - Input: Receives keyword from form trigger node.  
  - Output: Processed keyword data to the AI agent node.  
  - Edge Cases: API authentication errors, rate limits, or model unavailability could cause failures.  
  - Credentials: Requires valid Anthropic API key.

- **Keyword-based Scraping Agent**  
  - Type: LangChain AI Agent  
  - Configuration: Custom prompt instructs the agent to:  
    1. Receive keyword input,  
    2. Launch scraping job via BrightData API,  
    3. Poll scraping status until ready,  
    4. Fetch scraped data,  
    5. Output raw scraped content.  
  - Inputs: Receives processed keyword from Anthropic Chat Model and user keyword from form trigger.  
  - Outputs: Raw scraped content JSON to formatting node.  
  - Edge Cases: Handling polling delays, API timeouts, or unexpected status values; logic depends heavily on API responses.  
  - Notes: Central controller node coordinating the scraping lifecycle.

- **Wait for 1 Minute** (Disabled)  
  - Type: LangChain Tool Code node (JavaScript sleep function)  
  - Configuration: Pauses execution for 60 seconds before retrying status check.  
  - Disabled currently, useful for throttling or longer scraping jobs.  
  - Edge Cases: When enabled, may delay workflow execution unnecessarily if scraping is fast.

- **Sticky Note** (Positioned near AI agent nodes)  
  - Explains the AI Agent's role as the orchestrator of scraping logic.

- **Sticky Note5**  
  - Documents the use of Claude Sonnet 4 model for intelligent keyword processing.

- **Sticky Note1**  
  - Notes on the wait timer purpose and its disabled state.

---

#### 2.3 BrightData Scraping Interaction

**Overview:**  
Interacts directly with BrightData’s API to launch scraping jobs, check scraping snapshot status, and retrieve scraped Pinterest data.

**Nodes Involved:**  
- BrightData Pinterest Scraping (HTTP Request)  
- Check Scraping Status (HTTP Request)  
- Fetch Pinterest Snapshot Data (HTTP Request)  
- Sticky Note4 (Launch Scraping Job)  
- Sticky Note3 (Check Snapshot Progress)  
- Sticky Note2 (Download Scraped Data)

**Node Details:**

- **BrightData Pinterest Scraping**  
  - Type: HTTP Request Tool  
  - Configuration: Sends POST request to `https://api.brightdata.com/datasets/v3/trigger`  
  - Query Parameters: dataset_id, include_errors, type (discover_new), discover_by (keyword), limit_per_input  
  - Body: JSON with sessionId, action, and user keyword as chatInput  
  - Headers: Authorization Bearer token (BrightData API key)  
  - Input: Keyword from AI agent  
  - Output: JSON containing `snapshot_id` for tracking job progress  
  - Edge Cases: Auth errors, API rate limits, malformed request body, or API downtime.  
  - Notes: Initiates the Pinterest scraping job on BrightData.

- **Check Scraping Status**  
  - Type: HTTP Request Tool  
  - Configuration: GET request to `https://api.brightdata.com/datasets/v3/progress/{snapshot_id}`  
  - Query Parameters: format=json  
  - Headers: Authorization Bearer token  
  - Input: `snapshot_id` from previous node  
  - Output: Status JSON indicating if snapshot is "running" or "ready"  
  - Edge Cases: Snapshot ID invalid, network errors, delayed status updates.  
  - Notes: Polled repeatedly by AI agent until data is ready.

- **Fetch Pinterest Snapshot Data**  
  - Type: HTTP Request Tool  
  - Configuration: GET request to `https://api.brightdata.com/datasets/v3/snapshot/{snapshot_id}`  
  - Query Parameters: format=json  
  - Headers: Authorization Bearer token  
  - Input: `snapshot_id` once status is "ready"  
  - Output: Raw scraped Pinterest content data  
  - Edge Cases: Snapshot data missing, API errors, or malformed responses.

- **Sticky Note4**  
  - Describes launching scraping job with keyword and receiving snapshot_id.

- **Sticky Note3**  
  - Explains status check loop until snapshot is ready.

- **Sticky Note2**  
  - Details about fetching the actual scraped data once ready.

---

#### 2.4 Data Formatting and Storage

**Overview:**  
Processes the raw scraped data to extract structured Pinterest content fields and saves them into a Google Sheets document.

**Nodes Involved:**  
- Format & Extract Pinterest Content (Code Node)  
- Save Pinterest Data to Google Sheets  
- Sticky Note9 (Data Extraction Explanation)  
- Sticky Note8 (Final Output Explanation)  
- Sticky Note11 (Google Sheets Sample Link)

**Node Details:**

- **Format & Extract Pinterest Content**  
  - Type: Code Node (JavaScript)  
  - Configuration: Parses raw text output from AI agent, identifies and extracts fields such as URL, Post ID, Title, Content, Date Posted, User, Likes, Comments, Media, Image URL, Categories, Hashtags, and Comments text.  
  - Input: Raw scraped content from AI agent node  
  - Output: Array of JSON objects with cleaned and structured content fields  
  - Edge Cases: Missing fields in raw data, irregular formatting, or empty records handled by skipping.  
  - Notes: Uses regex to match each field robustly.

- **Save Pinterest Data to Google Sheets**  
  - Type: Google Sheets Node  
  - Configuration: Appends rows to a sheet with columns "Title", "Content", "Post URL", and "Image URL".  
  - Input: Structured JSON from formatting node  
  - Output: Confirmation of data written to sheet  
  - Credentials: Requires Google Sheets OAuth2 credentials with write access  
  - Edge Cases: Credential expiry, API quota limits, sheet ID or name misconfiguration.

- **Sticky Note9**  
  - Describes the extraction and structuring of Pinterest data.

- **Sticky Note8**  
  - Notes this node as the final output step.

- **Sticky Note11**  
  - Provides link and instructions to create a sample Google Sheet for saving data.

---

#### 2.5 Utility and Documentation Nodes

**Overview:**  
Includes general notes, placeholders for documentation, and disabled utility nodes.

**Nodes Involved:**  
- Sticky Note7  
- Sticky Notes scattered throughout (Sticky Note, Sticky Note1, Sticky Note4, etc.)

**Node Details:**

- **Sticky Note7**  
  - Placeholder for adding documentation, reminders, or tips.

- Other Sticky Notes  
  - Serve as inline documentation for respective nodes or blocks, improving maintainability and clarity.

---

### 3. Summary Table

| Node Name                       | Node Type                           | Functional Role                         | Input Node(s)                 | Output Node(s)                          | Sticky Note                                                                                   |
|--------------------------------|-----------------------------------|---------------------------------------|------------------------------|---------------------------------------|----------------------------------------------------------------------------------------------|
| Pinterest Keyword Input         | Form Trigger                      | Entry point: receives user keywords   | External HTTP/webhook         | Keyword-based Scraping Agent           | 📥 Keyword Input Form  User submits keywords here to initiate scraping.                       |
| Anthropic Chat Model            | LangChain Chat Model (Anthropic)  | AI model processes keywords           | Pinterest Keyword Input       | Keyword-based Scraping Agent           | 🧠 Claude Sonnet 4 Model  Used to intelligently process keywords before scraping.             |
| Keyword-based Scraping Agent    | LangChain AI Agent                | Orchestrates scraping job lifecycle   | Pinterest Keyword Input, Anthropic Chat Model, Wait for 1 Minute, BrightData Pinterest Scraping, Check Scraping Status, Fetch Pinterest Snapshot Data | Format & Extract Pinterest Content          | 🤖 AI Agent  Processes input keyword via language model (Claude). Controller for scraping.  |
| BrightData Pinterest Scraping  | HTTP Request Tool                 | Launches Pinterest scraping job       | Keyword-based Scraping Agent | Keyword-based Scraping Agent           | 🚀 Launch Scraping Job  Sends keyword to BrightData API, returns snapshot_id.                 |
| Check Scraping Status           | HTTP Request Tool                 | Polls scraping status                  | Keyword-based Scraping Agent | Keyword-based Scraping Agent           | 🔄 Check Snapshot Progress  Loops polling snapshot status until ready.                        |
| Fetch Pinterest Snapshot Data  | HTTP Request Tool                 | Fetches scraped Pinterest data        | Keyword-based Scraping Agent | Keyword-based Scraping Agent           | 📦 Download Scraped Data  Fetches data when snapshot status is ready.                         |
| Format & Extract Pinterest Content | Code Node                     | Parses and structures scraped data    | Keyword-based Scraping Agent | Save Pinterest Data to Google Sheets   | Extracts and structures scraped Pinterest data into clean fields.                            |
| Save Pinterest Data to Google Sheets | Google Sheets Node           | Saves structured data to Google Sheets| Format & Extract Pinterest Content | -                                     | "This is the final output step where formatted Pinterest content is saved for future use."   |
| Wait for 1 Minute (disabled)   | LangChain Tool Code (JS sleep)    | Optional wait timer between polls     | Keyword-based Scraping Agent | Keyword-based Scraping Agent           | ⏱️ Wait Timer (Disabled)  Useful for longer jobs, currently disabled.                        |
| Sticky Notes (multiple)         | Sticky Note                      | Various documentation and notes       | -                            | -                                     | Multiple notes explaining node roles and workflow steps, e.g. launch job, polling, formatting.|

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node: "Pinterest Keyword Input"**  
   - Type: Form Trigger  
   - Configure webhook ID (generate or assign)  
   - Add a required form field labeled “Keywords” to collect user input  

2. **Add Anthropic Chat Model Node: "Anthropic Chat Model"**  
   - Type: LangChain Chat Model (Anthropic)  
   - Configure model to "claude-sonnet-4-20250514"  
   - Connect input from "Pinterest Keyword Input" node  
   - Attach Anthropic API credentials (API key)  

3. **Add HTTP Request Node: "BrightData Pinterest Scraping"**  
   - Method: POST  
   - URL: `https://api.brightdata.com/datasets/v3/trigger`  
   - Query parameters:  
     - dataset_id = `gd_lk0sjs4d21kdr7cnlv`  
     - include_errors = `true`  
     - type = `discover_new`  
     - discover_by = `keyword`  
     - limit_per_input = `2`  
   - Body: JSON with keys sessionId, action, chatInput using the keyword from input  
   - Headers: Authorization Bearer token with BrightData API key  

4. **Add HTTP Request Node: "Check Scraping Status"**  
   - Method: GET  
   - URL Template: `https://api.brightdata.com/datasets/v3/progress/{{snapshot_id}}` (replace snapshot_id dynamically)  
   - Query parameter: format=json  
   - Headers: Authorization Bearer token  

5. **Add HTTP Request Node: "Fetch Pinterest Snapshot Data"**  
   - Method: GET  
   - URL Template: `https://api.brightdata.com/datasets/v3/snapshot/{{snapshot_id}}` (replace snapshot_id dynamically)  
   - Query parameter: format=json  
   - Headers: Authorization Bearer token  

6. **Add LangChain AI Agent Node: "Keyword-based Scraping Agent"**  
   - Define prompt to:  
     - Receive keyword input  
     - Use "BrightData Pinterest Scraping" node to launch job  
     - Use "Check Scraping Status" node to poll status until ready  
     - Use "Fetch Pinterest Snapshot Data" node to retrieve data  
     - Output raw scraped content  
   - Connect inputs from "Pinterest Keyword Input," "Anthropic Chat Model," and the three HTTP Request nodes above  

7. **Add Code Node: "Format & Extract Pinterest Content"**  
   - Paste provided JavaScript code to parse raw output and extract fields  
   - Connect input from "Keyword-based Scraping Agent" node  

8. **Add Google Sheets Node: "Save Pinterest Data to Google Sheets"**  
   - Operation: Append  
   - Document ID: Google Sheet URL or ID for target sheet  
   - Sheet Name: Use sheet identifier or name (e.g., 'gid=0' or 'Sheet1')  
   - Map columns: Title, Content, Post URL, Image URL from JSON fields  
   - Attach Google Sheets OAuth2 credentials  
   - Connect input from "Format & Extract Pinterest Content" node  

9. **(Optional) Add Wait Node: "Wait for 1 Minute"**  
   - Use JavaScript code to pause 60 seconds between status polls (disabled by default)  

10. **Add Sticky Notes** for documentation at appropriate places for clarity and maintenance  

11. **Connect Nodes in Sequence:**  
    - Pinterest Keyword Input → Anthropic Chat Model → Keyword-based Scraping Agent  
    - Keyword-based Scraping Agent → BrightData Pinterest Scraping  
    - Keyword-based Scraping Agent → Check Scraping Status  
    - Keyword-based Scraping Agent → Fetch Pinterest Snapshot Data  
    - Keyword-based Scraping Agent → Format & Extract Pinterest Content → Save Pinterest Data to Google Sheets  

12. **Set Credentials:**  
    - Anthropic API credentials for Anthropic Chat Model  
    - BrightData API key for HTTP Request nodes interacting with BrightData  
    - Google Sheets OAuth2 credentials for saving data  

13. **Save and Activate Workflow**

---

### 5. General Notes & Resources

| Note Content                                                                                       | Context or Link                                                                                         |
|--------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Create a copy of the sample Google Sheet before running the workflow to ensure compatibility.   | https://docs.google.com/spreadsheets/d/SAMPLE_SHEET_ID/edit                                            |
| BrightData Pinterest API requires an active subscription and valid API key with correct scopes. | BrightData API documentation and account setup                                                         |
| Anthropic Claude Sonnet 4 is a specialized language model designed for intelligent text processing. | Anthropic API documentation                                                                             |
| The wait timer node is disabled by default but can be enabled for longer scraping jobs to avoid rate limits. | Workflow optimization tip                                                                                |
| This workflow demonstrates integration of AI orchestration with real-time scraping and data extraction workflows. | Conceptual model for complex automation pipelines                                                       |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.