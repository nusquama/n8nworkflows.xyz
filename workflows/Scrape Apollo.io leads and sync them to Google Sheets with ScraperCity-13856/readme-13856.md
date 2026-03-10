Scrape Apollo.io leads and sync them to Google Sheets with ScraperCity

https://n8nworkflows.xyz/workflows/scrape-apollo-io-leads-and-sync-them-to-google-sheets-with-scrapercity-13856


# Scrape Apollo.io leads and sync them to Google Sheets with ScraperCity

This document provides a technical breakdown of the n8n workflow designed to automate lead extraction from Apollo.io via the ScraperCity API and synchronize the results into Google Sheets.

### 1. Workflow Overview
The workflow automates the end-to-end process of B2B lead generation. It replaces manual CSV exports by programmatically applying search filters to Apollo.io, monitoring a long-running extraction job (polling), and processing the resulting data into a clean spreadsheet format.

**Logical Blocks:**
*   **1.1 Configuration & Initiation:** Defines target search criteria and triggers the initial scraping request.
*   **1.2 Async Polling Loop:** Manages the waiting period while ScraperCity processes the request, checking status every 60 seconds.
*   **1.3 Data Processing:** Downloads the raw CSV, parses it into JSON objects, and removes duplicate entries based on email addresses.
*   **1.4 Delivery:** Appends the cleaned lead data to a specified Google Sheet.

---

### 2. Block-by-Block Analysis

#### 2.1 Configuration & Initiation
This block handles the user inputs and the initial handshake with the ScraperCity API.

*   **When clicking 'Execute workflow'**: Manual trigger to start the process.
*   **Configure Search Parameters (Set Node)**:
    *   **Role**: Centralized configuration for lead filters.
    *   **Variables**: `jobTitles` (comma-separated), `industry`, `companySize`, `leadCount`, and `exportFileName`.
*   **Start Apollo Lead Scrape (HTTP Request)**:
    *   **Role**: Sends a POST request to `https://scrapercity.com/api/v1/scrape/apollo-filters`.
    *   **Configuration**: Uses Header Auth (Bearer Token). It transforms the comma-separated job titles into a JSON array using an expression.
    *   **Failure Modes**: Invalid API key (401), malformed filter syntax (400), or API rate limits.

#### 2.2 Async Polling Loop
Since scraping thousands of leads is not instantaneous (10–60 minutes), this block manages the "Wait and Check" logic.

*   **Store Run ID (Set Node)**: Extracts and stores the `runId` from the initial API response to track the job.
*   **Polling Loop Controller (Split In Batches)**: Used to control the flow and prevent potential infinite loops by processing the single Run ID item.
*   **Wait 60 Seconds**: Pauses execution to avoid hammering the API.
*   **Check Scrape Status (HTTP Request)**: Calls the status endpoint using the stored `runId`.
*   **Is Scrape Complete? (If Node)**: Checks if the `status` field equals `SUCCEEDED`. If `false`, it routes back to the loop controller.

#### 2.3 Data Processing & Delivery
Once the job is done, this block cleans and saves the data.

*   **Download Scraped Results (HTTP Request)**: Fetches the final dataset. It is configured to receive the response as "Text" (CSV format).
*   **Parse CSV and Format Leads (Code Node)**:
    *   **Logic**: A custom JavaScript parser that handles CSV quoting and commas. It maps the header row to JSON keys for every subsequent row.
*   **Remove Duplicate Leads**: Filters the list to ensure each `Email` address appears only once.
*   **Save Leads to Google Sheets**:
    *   **Role**: Appends data to a specific spreadsheet.
    *   **Configuration**: Requires a valid Document ID and Sheet Name. It uses "Auto Map" to align CSV headers with Sheet columns.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| When clicking 'Execute workflow' | manualTrigger | Manual Start | None | Configure Search Parameters | The "When clicking Execute workflow" trigger starts the run manually. |
| Configure Search Parameters | set | Define Filters | When clicking 'Execute workflow' | Start Apollo Lead Scrape | "Configure Search Parameters" holds all user-editable variables. |
| Start Apollo Lead Scrape | httpRequest | API Initiation | Configure Search Parameters | Store Run ID | Add your ScraperCity API key credential here. |
| Store Run ID | set | Data Persistence | Start Apollo Lead Scrape | Polling Loop Controller | "Store Run ID" saves the job identifier returned by the scrape request. |
| Polling Loop Controller | splitInBatches | Loop Management | Store Run ID, Is Scrape Complete? | Wait 60 Seconds | The splitInBatches node caps iterations to prevent infinite loops. |
| Wait 60 Seconds | wait | Delay | Polling Loop Controller | Check Scrape Status | "Wait 60 Seconds" pauses execution before each status check. |
| Check Scrape Status | httpRequest | Status Check | Wait 60 Seconds | Is Scrape Complete? | "Check Scrape Status" calls the status endpoint. |
| Is Scrape Complete? | if | Conditional Logic | Check Scrape Status | Download Scraped Results, Polling Loop Controller | Checks whether status equals SUCCEEDED. If not, the loop iterates back. |
| Download Scraped Results | httpRequest | Data Fetching | Is Scrape Complete? | Parse CSV and Format Leads | Fetches the CSV file from the ScraperCity download endpoint. |
| Parse CSV and Format Leads | code | Data Parsing | Download Scraped Results | Remove Duplicate Leads | Transforms raw CSV text into structured JSON rows. |
| Remove Duplicate Leads | removeDuplicates | Data Cleaning | Parse CSV and Format Leads | Save Leads to Google Sheets | Deduplicates by email address. |
| Save Leads to Google Sheets | googleSheets | Data Export | Remove Duplicate Leads | None | Set your Google Sheets document ID and sheet name here. |

---

### 4. Reproducing the Workflow from Scratch

1.  **Credentials Setup**:
    *   Create a **Header Auth** credential. Name: `ScraperCity API Key`. Header: `Authorization`. Value: `Bearer [YOUR_API_KEY]`.
    *   Create a **Google Sheets OAuth2** credential and authenticate.
2.  **Trigger**: Add a **Manual Trigger** node.
3.  **Configuration**: Add a **Set** node. Create string parameters for `jobTitles`, `industry`, `companySize`, and a number for `leadCount`.
4.  **API Call**: Add an **HTTP Request** node. Set method to `POST`, URL to `https://scrapercity.com/api/v1/scrape/apollo-filters`. In the body, use an expression to map the Set node values into a JSON object.
5.  **State Management**: Add a **Set** node named "Store Run ID" to capture `{{ $json.runId }}`.
6.  **Looping**:
    *   Connect a **Split In Batches** node (Batch Size 1).
    *   Connect a **Wait** node (60 seconds).
    *   Connect an **HTTP Request** node (`GET`) to the status endpoint: `https://scrapercity.com/api/v1/scrape/status/{{ $('Store Run ID').item.json.runId }}`.
    *   Add an **If** node: Condition `status` equals `SUCCEEDED`.
    *   Connect the **False** output of the "If" node back to the **Split In Batches** node.
7.  **Download**: Connect the **True** output of the "If" node to an **HTTP Request** node (`GET`) targeting the download URL. Set "Response Format" to `Text`.
8.  **Parsing**: Add a **Code** node. Paste a CSV-to-JSON parsing script that iterates through lines, splits by comma (respecting quotes), and maps to headers.
9.  **Cleaning**: Add a **Remove Duplicates** node. Set "Fields to Compare" to `Email`.
10. **Final Step**: Add a **Google Sheets** node. Set "Operation" to `Append`. Provide your spreadsheet ID and select the target sheet.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| ScraperCity Documentation | [https://scrapercity.com](https://scrapercity.com) |
| Cost Efficiency | Estimated at ~$3.90 USD per 1,000 verified leads. |
| Job Duration | Expect jobs to run between 10 to 60 minutes depending on volume. |
| CSV Formatting | The parser is optimized for standard Apollo CSV exports. |