Track Technology Stacks & Find Decision Makers with BuiltWith to Google Sheets

https://n8nworkflows.xyz/workflows/track-technology-stacks---find-decision-makers-with-builtwith-to-google-sheets-4784


# Track Technology Stacks & Find Decision Makers with BuiltWith to Google Sheets

---
### 1. Workflow Overview

This workflow automates the process of tracking websites that use a specific technology stack by querying the BuiltWith API and logging the results into a Google Sheets spreadsheet. It targets users interested in lead generation, competitive analysis, or technology stack monitoring by providing a streamlined, manual-triggered data retrieval and storage flow.

**Logical Blocks:**

- **1.1 Manual Input and Technology Setup:** Allows the user to manually start the workflow and specify the technology to search for.
- **1.2 Fetch and Process BuiltWith Data:** Sends a request to the BuiltWith API to retrieve websites using the specified technology and processes the response to extract relevant site information.
- **1.3 Log Data to Google Sheets:** Appends the cleaned and structured data into a Google Sheets document for further use and analysis.

---

### 2. Block-by-Block Analysis

#### 2.1 Manual Input and Technology Setup

- **Overview:**  
  This block initiates the workflow manually and sets the target technology to search on BuiltWith, enabling dynamic input without changing the workflow’s core.

- **Nodes Involved:**  
  - Manual Trigger  
  - Set Technology

- **Node Details:**

  - **Manual Trigger**  
    - Type: Trigger node (Manual)  
    - Role: Allows manual initiation of the workflow for testing or one-off runs.  
    - Configuration: No parameters; simply triggers flow on user command.  
    - Inputs: None  
    - Outputs: Connects to "Set Technology" node.  
    - Version: 1  
    - Edge Cases: None typical; user must trigger manually. No auth dependencies.  

  - **Set Technology**  
    - Type: Set node  
    - Role: Sets a static field named "Technology" with the value of the technology to be searched (default is "Shopify").  
    - Configuration: Defines a string field `Technology` with a fixed value `"Shopify"` (modifiable to any tech).  
    - Expressions: The value is static here, but could be dynamic if edited.  
    - Inputs: From Manual Trigger  
    - Outputs: To "Fetch BuiltWith Data"  
    - Version: 3.4  
    - Edge Cases: If the technology field is empty or misspelled, API queries may fail or return empty results.

---

#### 2.2 Fetch and Process BuiltWith Data

- **Overview:**  
  This block communicates with the BuiltWith API to fetch websites using the specified technology and extracts key data fields from the API response for further use.

- **Nodes Involved:**  
  - Fetch BuiltWith Data  
  - Extract Site Info

- **Node Details:**

  - **Fetch BuiltWith Data**  
    - Type: HTTP Request  
    - Role: Performs a GET request to BuiltWith’s API endpoint to retrieve technology usage data.  
    - Configuration:  
      - URL: `https://api.builtwith.com/v20/api.json`  
      - Query Parameters:  
        - `KEY`: API key placeholder (`YOUR_API_KEY` to be replaced by user’s actual key).  
        - `TECH`: Set dynamically from the `Technology` field (`={{ $json.Technology }}`), e.g., "Shopify".  
        - `START`: Pagination start index, fixed at `0`.  
      - Method: GET  
    - Inputs: From Set Technology  
    - Outputs: To Extract Site Info  
    - Version: 4.2  
    - Edge Cases:  
      - Invalid or missing API key → authentication failure.  
      - API rate limits or quota exceeded → request fails.  
      - Empty or invalid technology parameter → no results returned.  
      - Network errors or timeouts.  

  - **Extract Site Info**  
    - Type: Code (Function) node  
    - Role: Processes the raw API response JSON to extract and format relevant fields for logging.  
    - Configuration:  
      - JavaScript code extracts `Results` array and maps each entry to a simplified object with fields:  
        - `domain`: from `ResultPath`  
        - `technology`: from input JSON field `technologies` or defaults to `"Unknown"`  
        - `firstIndexed`: from `FirstIndexed`  
        - `vertical`: from `Vertical`  
    - Key Expressions: Uses standard JavaScript to map and return a simplified array of JSON objects.  
    - Inputs: From Fetch BuiltWith Data  
    - Outputs: To Log to Google Sheet  
    - Version: 2  
    - Edge Cases:  
      - Missing or malformed `Results` will result in an empty output array.  
      - Unexpected JSON structure changes from API may cause errors or missing data.  
      - Defensive fallback for missing `technologies` field.  

---

#### 2.3 Log Data to Google Sheets

- **Overview:**  
  Appends each extracted website record as a new row in a designated Google Sheet, enabling data persistence and further analysis.

- **Nodes Involved:**  
  - Log to Google Sheet

- **Node Details:**

  - **Log to Google Sheet**  
    - Type: Google Sheets node (Append Operation)  
    - Role: Inserts processed data rows into a Google Sheets spreadsheet with defined columns.  
    - Configuration:  
      - Operation: Append rows  
      - Target Spreadsheet ID: `"1lDu3HvxcpA1nRDJq6Ge-OrMO0v592UfLBIjt3Ylc3Mo"`  
      - Sheet: Sheet1 (gid=0)  
      - Columns mapped:  
        - `Domain` ← `domain` field from JSON  
        - `Vertical` ← `vertical` field  
        - `Technology` ← `technology` field  
        - `First Indexed` ← `firstIndexed` field  
      - Mapping mode: Explicit mapping of fields  
      - Authentication: Uses Google OAuth2 credentials stored in n8n (`Google Sheets account`).  
    - Inputs: From Extract Site Info  
    - Outputs: None (end of workflow)  
    - Version: 4.5  
    - Edge Cases:  
      - Authentication failure or expired Google OAuth token.  
      - Spreadsheet or sheet not found or access denied.  
      - Data type mismatches (though type conversion is disabled).  
      - API rate limits from Google Sheets.  

---

### 3. Summary Table

| Node Name           | Node Type          | Functional Role                          | Input Node(s)       | Output Node(s)       | Sticky Note                                                                                         |
|---------------------|--------------------|----------------------------------------|---------------------|----------------------|---------------------------------------------------------------------------------------------------|
| Manual Trigger      | Manual Trigger     | Starts workflow manually                | None                | Set Technology       | ## 🧩 SECTION 1: **🔘 Manual Start + 📝 Set Target Technology**<br>Manual start for testing and ad-hoc runs. |
| Set Technology      | Set                | Sets target technology for lookup      | Manual Trigger      | Fetch BuiltWith Data  | See Manual Trigger sticky note.                                                                    |
| Fetch BuiltWith Data| HTTP Request       | Calls BuiltWith API to fetch websites  | Set Technology      | Extract Site Info     | ## 🔗 SECTION 2: **🌐 Fetch + 🧠 Process BuiltWith Data**<br>Connects to BuiltWith API and extracts data. |
| Extract Site Info   | Function (Code)    | Parses and cleans API response          | Fetch BuiltWith Data | Log to Google Sheet   | See Fetch BuiltWith Data sticky note.                                                              |
| Log to Google Sheet | Google Sheets Append | Appends extracted data to spreadsheet | Extract Site Info    | None                 | ## 📊 SECTION 3: **📄 Log to Google Sheets**<br>Stores data for lead gen and analysis.               |
| Sticky Note         | Sticky Note        | Documentation and instructions          | None                | None                 | Various notes covering sections 1, 2, 3, and assistance information.                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Add a `Manual Trigger` node named "Manual Trigger". This node requires no parameters. It allows manual execution.

2. **Add Set Node to Define Technology:**  
   - Create a `Set` node named "Set Technology".  
   - Add a field named `Technology` of type `string`.  
   - Set its value to `"Shopify"` (or any technology you want to track).  
   - Connect "Manual Trigger" output to this node’s input.

3. **Add HTTP Request Node for BuiltWith API Call:**  
   - Add an `HTTP Request` node named "Fetch BuiltWith Data".  
   - Configure:  
     - Method: GET  
     - URL: `https://api.builtwith.com/v20/api.json`  
     - Query Parameters:  
       - `KEY`: Your BuiltWith API key (replace `"YOUR_API_KEY"`).  
       - `TECH`: Use expression `{{$json.Technology}}` to pass the technology field dynamically.  
       - `START`: `0` (for pagination start index).  
   - Connect "Set Technology" output to this node’s input.  
   - Ensure no authentication is needed here beyond the API key in query.

4. **Add Function Node to Extract Relevant Data:**  
   - Add a `Function` node named "Extract Site Info".  
   - Paste in the following JavaScript code:

     ```javascript
     const techSearched = $json["technologies"] || "Unknown";
     const results = $json.Results || [];

     return results.map(site => {
       return {
         json: {
           domain: site.ResultPath || "",
           technology: techSearched,
           firstIndexed: site.FirstIndexed || "",
           vertical: site.Vertical || ""
         }
       };
     });
     ```

   - Connect "Fetch BuiltWith Data" output to this node’s input.

5. **Add Google Sheets Node to Append Data:**  
   - Add a `Google Sheets` node named "Log to Google Sheet".  
   - Set the operation to `Append`.  
   - Configure the following:  
     - Document ID: your Google Sheet ID (e.g., `1lDu3HvxcpA1nRDJq6Ge-OrMO0v592UfLBIjt3Ylc3Mo`).  
     - Sheet Name: `Sheet1` (or the name/gid of your target sheet).  
     - Mapping Mode: Define below.  
     - Map columns as:  
       - `Domain` ← `{{$json.domain}}`  
       - `Technology` ← `{{$json.technology}}`  
       - `First Indexed` ← `{{$json.firstIndexed}}`  
       - `Vertical` ← `{{$json.vertical}}`  
   - Connect "Extract Site Info" output to this node’s input.  
   - Set up Google OAuth2 credentials for this node if not already done.

6. **Validate and Save:**  
   - Ensure all connections are correct and nodes have no errors.  
   - Test the workflow by running the "Manual Trigger" node.  
   - Verify data is appended correctly to your Google Sheet.

---

### 5. General Notes & Resources

| Note Content                                                                                                                             | Context or Link                                                                                         |
|------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| For questions or support, contact: Yaron@nofluff.online                                                                                  | Workflow Assistance sticky note                                                                        |
| YouTube Channel with more tips and tutorials: https://www.youtube.com/@YaronBeen/videos                                                  | Workflow Assistance sticky note                                                                        |
| LinkedIn Profile: https://www.linkedin.com/in/yaronbeen/                                                                                | Workflow Assistance sticky note                                                                        |
| Workflow is adaptable to any API → process → log pattern by swapping BuiltWith API with other APIs                                        | Final workflow recap sticky note                                                                       |
| BuiltWith API documentation must be consulted for API key acquisition and parameter details                                              | HTTP Request node sticky note                                                                           |
| Google Sheets OAuth2 credentials must be configured and authorized before running the workflow                                           | Google Sheets node sticky note                                                                          |

---

**Disclaimer:**  
The text provided is generated solely from an automated workflow created with n8n, respecting applicable content policies and containing no illegal or protected elements. All data processed is legal and publicly accessible.