Extract Citation Sources from Google AI Overview to Google Sheets with DataForSEO

https://n8nworkflows.xyz/workflows/extract-citation-sources-from-google-ai-overview-to-google-sheets-with-dataforseo-7539


# Extract Citation Sources from Google AI Overview to Google Sheets with DataForSEO

### 1. Workflow Overview

This workflow automates the extraction of source references from Google's AI Overview search results using DataForSEO API, and records the extracted data into a Google Sheet. It is designed to run periodically (every 7 days by default) to track and archive source citations related to any specified keyword, location, and language.

Logical blocks included:

- **1.1 Scheduled Trigger:** Initiates the workflow automatically every 7 days.
- **1.2 DataForSEO API Request:** Fetches the AI Overview organic search results from Google for a given keyword, language, and location.
- **1.3 Data Flattening and Extraction:** Processes the nested API response to isolate individual tasks and reference items.
- **1.4 Data Recording:** Appends the extracted source references (source title, domain, URL, title, and text snippet) into a configured Google Sheet for tracking and analysis.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
Starts the workflow automatically on a fixed schedule, enabling periodic data refresh and extraction without manual intervention.

- **Nodes Involved:**  
  - Run every 7 days  
  - Sticky Note (instruction about schedule)

- **Node Details:**

  - **Run every 7 days**  
    - Type: Schedule Trigger  
    - Configuration: Triggers every 7 days (can be adjusted)  
    - Key Expressions/Variables: None  
    - Input: None (trigger node)  
    - Output: Connects to "Get Google AI Overview SERP Data" node  
    - Edge Cases: Misconfiguration can cause workflow not to run on expected intervals  
    - Version: 1.2

  - **Sticky Note (positioned near schedule node)**  
    - Type: Sticky Note  
    - Role: Reminds user to adjust schedule interval if needed  
    - Content: "The flow will run every 7 days. Adjust the schedule if needed."  
    - No input/output connections

---

#### 1.2 DataForSEO API Request

- **Overview:**  
Sends a query to DataForSEO’s SERP API to retrieve live Google organic search results, including the AI Overview, for a specified keyword, language, and location.

- **Nodes Involved:**  
  - Get Google AI Overview SERP Data  
  - Sticky Note (instruction about keyword, language, location)

- **Node Details:**

  - **Get Google AI Overview SERP Data**  
    - Type: DataForSEO SERP API Node  
    - Configuration:  
      - Operation: get-live-google-organic-serp-advanced  
      - Keyword: "why sky is blue" (default example)  
      - Language: English  
      - Location: United States  
      - AI Overview loading enabled (load_async_ai_overview=true)  
      - Other parameters left default/empty  
    - Credentials: Uses DataForSEO API credentials  
    - Input: Receives trigger from schedule node  
    - Output: Sends data to the "Split Out (items)" node  
    - Edge Cases: API authentication errors, quota limits, network timeouts, invalid keyword or location may cause failures  
    - Version: 1

  - **Sticky Note (near API node)**  
    - Content: "Specify the necessary **Keyword**, **Location**, and **Language** you want to get data for."  
    - Purpose: Guides user to configure relevant query parameters

---

#### 1.3 Data Flattening and Extraction

- **Overview:**  
Processes the API response to flatten nested arrays and extract each individual reference item for further processing.

- **Nodes Involved:**  
  - Split Out (items)  
  - Split Out (references)

- **Node Details:**

  - **Split Out (items)**  
    - Type: Split Out  
    - Configuration: Splits data by the JSON path `tasks[0].result[0].items`  
    - Input: Receives array of results from API node  
    - Output: Passes individual items to "Split Out (references)"  
    - Edge Cases: If API response structure changes or is empty, may produce errors or no outputs  
    - Version: 1

  - **Split Out (references)**  
    - Type: Split Out  
    - Configuration: Splits each item further by the JSON field `references`  
    - Input: Receives individual items from previous split  
    - Output: Sends each reference object to Google Sheets node  
    - Edge Cases: Empty or missing `references` field can cause no records to be processed  
    - Version: 1

---

#### 1.4 Data Recording

- **Overview:**  
Appends each extracted reference’s details (source, domain, URL, title, text snippet) as a new row in a designated Google Sheet.

- **Nodes Involved:**  
  - Record references to your Google Sheet  
  - Sticky Note (instruction about Google Sheet columns)

- **Node Details:**

  - **Record references to your Google Sheet**  
    - Type: Google Sheets (Append)  
    - Configuration:  
      - Operation: Append rows only (no update or delete)  
      - Columns mapped explicitly: Source, Domain, URL, Title, Text  
      - Data extracted from incoming JSON fields with expressions like `{{$json.url}}` etc.  
      - Target Sheet: Sheet1 (gid=0) of document ID `1XCjkjyVxrtpTUQenHeR3B07xfEZ489mhuVidjhGOO7I`  
    - Credentials: Uses authorized Google Sheets OAuth2 credentials  
    - Input: Receives individual reference objects from "Split Out (references)" node  
    - Output: None (terminal node)  
    - Edge Cases: Permissions errors if credentials lack edit rights; sheet not found; rate limits on Google API  
    - Version: 4.5

  - **Sticky Note (near Google Sheets node)**  
    - Content: "Select the necessary Google Sheet that contains the **Source**, **Domain**, **URL**, **Title**, and **Text** columns."  
    - Purpose: Informs user to prepare and select the correct Google Sheet structure before running

---

#### Additional Sticky Note (Workflow Description)

- **Sticky Note3**  
  - Content:  
    This n8n template automatically collects all source references from Google’s AI Overview for any keyword you choose using DataForSEO and Google Sheets.  
    What it does:  
    * Uses DataForSEO API to fetch AI Overview results.  
    * Extracts the source title, URL, and domain.  
    * Stores the data in Google Sheets.  
    You can use this data to track your website’s presence in AI-generated snippets.  
  - Positioned separately to provide overall context to the workflow

---

### 3. Summary Table

| Node Name                              | Node Type                      | Functional Role                       | Input Node(s)              | Output Node(s)                  | Sticky Note                                                                                         |
|--------------------------------------|--------------------------------|-------------------------------------|----------------------------|--------------------------------|---------------------------------------------------------------------------------------------------|
| Run every 7 days                     | Schedule Trigger               | Initiates workflow on 7-day schedule| None                       | Get Google AI Overview SERP Data| The flow will run every 7 days. Adjust the schedule if needed.                                    |
| Get Google AI Overview SERP Data     | DataForSEO SERP API            | Fetches AI Overview SERP data       | Run every 7 days           | Split Out (items)               | Specify the necessary **Keyword**, **Location**, and **Language** you want to get data for.       |
| Split Out (items)                    | Split Out                     | Splits API response into individual items | Get Google AI Overview SERP Data | Split Out (references)          |                                                                                                   |
| Split Out (references)               | Split Out                     | Extracts individual references from items | Split Out (items)          | Record references to your Google Sheet |                                                                                                   |
| Record references to your Google Sheet | Google Sheets (Append)         | Appends extracted references to Google Sheet | Split Out (references)     | None                           | Select the necessary Google Sheet that contains the **Source**, **Domain**, **URL**, **Title**, and **Text** columns. |
| Sticky Note                         | Sticky Note                   | Instruction on schedule             | None                       | None                           | The flow will run every 7 days. Adjust the schedule if needed.                                    |
| Sticky Note1                        | Sticky Note                   | Instruction on keyword/location/language setup | None                       | None                           | Specify the necessary **Keyword**, **Location**, and **Language** you want to get data for.       |
| Sticky Note2                        | Sticky Note                   | Instruction on Google Sheet columns | None                       | None                           | Select the necessary Google Sheet that contains the **Source**, **Domain**, **URL**, **Title**, and **Text** columns. |
| Sticky Note3                        | Sticky Note                   | Workflow overview description       | None                       | None                           | This n8n template automatically collects all source references from Google’s AI Overview for any keyword you choose using DataForSEO and Google Sheets... |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Name: "Run every 7 days"  
   - Type: Schedule Trigger  
   - Set interval to trigger every 7 days (adjustable)  

2. **Create a DataForSEO SERP API node**  
   - Name: "Get Google AI Overview SERP Data"  
   - Type: DataForSEO SERP API  
   - Operation: `get-live-google-organic-serp-advanced`  
   - Parameters:  
     - Keyword: Input your target keyword (e.g., "why sky is blue")  
     - Language name: e.g., "english"  
     - Location name: e.g., "united states"  
     - Enable `load_async_ai_overview` = true  
     - Leave other parameters default or empty  
   - Credentials: Configure with your DataForSEO API credentials  

3. **Connect Schedule Trigger to DataForSEO node**  

4. **Add a Split Out node to flatten tasks items**  
   - Name: "Split Out (items)"  
   - Type: Split Out  
   - Field to split out: `tasks[0].result[0].items`  

5. **Connect DataForSEO node output to Split Out (items) node input**  

6. **Add another Split Out node to separate references**  
   - Name: "Split Out (references)"  
   - Type: Split Out  
   - Field to split out: `references`  

7. **Connect Split Out (items) output to Split Out (references) input**  

8. **Add Google Sheets node**  
   - Name: "Record references to your Google Sheet"  
   - Type: Google Sheets (Append)  
   - Operation: Append  
   - Document ID: Set to your Google Sheet ID containing columns: Source, Domain, URL, Title, Text  
   - Sheet Name: Select the appropriate sheet (e.g., Sheet1)  
   - Mapping: Map columns as follows using expressions:  
     - Source: `{{$json.source}}`  
     - Domain: `{{$json.domain}}`  
     - URL: `{{$json.url}}`  
     - Title: `{{$json.title}}`  
     - Text: `{{$json.text}}`  
   - Credentials: Connect your Google Sheets OAuth2 credentials  

9. **Connect Split Out (references) output to Google Sheets node input**  

10. **(Optional) Add Sticky Notes for instructions**  
    - Schedule note near Schedule Trigger: "The flow will run every 7 days. Adjust the schedule if needed."  
    - Keyword/naming note near DataForSEO node: "Specify the necessary **Keyword**, **Location**, and **Language** you want to get data for."  
    - Google Sheet note near Google Sheets node: "Select the necessary Google Sheet that contains the **Source**, **Domain**, **URL**, **Title**, and **Text** columns."  
    - Overview note describing workflow purpose for context  

11. **Test the workflow**  
    - Manually trigger or wait for scheduled trigger to verify successful data retrieval and appending to the sheet  
    - Verify API credentials and Google Sheets permissions are correct  

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                         |
|----------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| This n8n template automatically collects all source references from Google’s AI Overview for any keyword you choose using DataForSEO and Google Sheets. | Workflow overview for user understanding                                                                |
| You can use this data to track your website’s presence in AI-generated snippets and monitor citations over time. | Use case and benefit of the workflow                                                                     |
| DataForSEO API documentation: https://docs.dataforseo.com/                                         | Reference for API parameters and limits                                                                |
| Google Sheets API and OAuth2 setup documentation: https://developers.google.com/sheets/api        | Credential setup and permissions                                                                         |

---

**Disclaimer:**  
The text and data processed by this workflow come exclusively from automated n8n processing using legal and public sources. All content respects applicable content policies and contains no illegal, offensive, or protected material.