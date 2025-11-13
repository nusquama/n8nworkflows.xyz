Generate Business Lists from Google Maps Search to Sheets or Excel with Serper

https://n8nworkflows.xyz/workflows/generate-business-lists-from-google-maps-search-to-sheets-or-excel-with-serper-7542


# Generate Business Lists from Google Maps Search to Sheets or Excel with Serper

### 1. Workflow Overview

This workflow automates the process of generating business contact lists from Google Maps search results using the Serper API, then outputs the data into Google Sheets or Excel files. It is designed for users who need to collect, structure, and store location-based business data efficiently.

The workflow is logically divided into the following main blocks:

- **1.1 Input Reception:** Receives user input via a chat message trigger, initializing variables.
- **1.2 Data Extraction:** Calls the Serper API to get Google Maps search results, then parses and extracts relevant business data.
- **1.3 Data Preparation:** Formats and structures the extracted data for output.
- **1.4 Data Output:** Converts data into XLSX format and upserts it into Google Sheets.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block handles incoming chat messages that trigger the workflow. It initializes variables for use downstream.

**Nodes Involved:**  
- When chat message received  
- Variables Chat  
- Initialization  

**Node Details:**

- **When chat message received**  
  - *Type:* ChatTrigger (LangChain)  
  - *Role:* Entry point of the workflow, listens for chat input messages that trigger the workflow execution.  
  - *Config:* Uses a webhook ID to receive messages. No additional parameters configured.  
  - *Input:* External chat message trigger  
  - *Output:* Passes message data to the next node  
  - *Potential Failures:* Webhook connectivity issues, malformed input messages  

- **Variables Chat**  
  - *Type:* Set  
  - *Role:* Initializes or sets variables based on chat input for consistency in downstream processing.  
  - *Config:* Typically sets variables such as search parameters, query strings, or context data.  
  - *Input:* From "When chat message received"  
  - *Output:* Passes initialized data to "Initialization"  
  - *Potential Failures:* Expression evaluation errors if variables depend on missing input fields  

- **Initialization**  
  - *Type:* Set  
  - *Role:* Further refines variables or prepares additional parameters required for the API call.  
  - *Config:* Sets parameters such as API keys, search queries, or request formatting.  
  - *Input:* From "Variables Chat"  
  - *Output:* Passes configured parameters to "Extract Serper Map"  
  - *Potential Failures:* Misconfiguration of required parameters, missing API key variables  

---

#### 2.2 Data Extraction

**Overview:**  
This block performs the core data retrieval by calling Serper’s Google Maps API, then splits and extracts the relevant business information.

**Nodes Involved:**  
- Extract Serper Map  
- Split Out Serper  
- Extract Data   

**Node Details:**

- **Extract Serper Map**  
  - *Type:* HTTP Request  
  - *Role:* Calls Serper API to request Google Maps search results with provided query parameters.  
  - *Config:* Configured with necessary URL, HTTP method (usually GET or POST), headers including API key, and query body with search parameters.  
  - *Input:* From "Initialization"  
  - *Output:* Raw API response containing map search results  
  - *Potential Failures:* API authentication errors, rate limiting, network timeouts, invalid query parameters  

- **Split Out Serper**  
  - *Type:* SplitOut  
  - *Role:* Splits the array of map results into individual items for separate processing.  
  - *Config:* No parameters needed, automatically splits array output into separate items.  
  - *Input:* From "Extract Serper Map"  
  - *Output:* Individual business entries passed to "Extract Data" node  
  - *Potential Failures:* Failures if input is not an array or empty responses  

- **Extract Data**  
  - *Type:* Set  
  - *Role:* Extracts and maps specific fields from each business entry, such as name, address, phone, rating, etc.  
  - *Config:* Uses expressions to access nested JSON data fields from the split Serper response.  
  - *Input:* From "Split Out Serper"  
  - *Output:* Structured business data objects passed to "Prepare Data"  
  - *Potential Failures:* Expression errors if fields are missing or data structures vary  

---

#### 2.3 Data Preparation

**Overview:**  
This block prepares the extracted data for final output by formatting, cleaning, and structuring it appropriately.

**Nodes Involved:**  
- Prepare Data  

**Node Details:**

- **Prepare Data**  
  - *Type:* Set  
  - *Role:* Finalizes data structure, possibly adds headers, filters or transforms data fields for compatibility with output nodes.  
  - *Config:* Sets or modifies fields, may transform data types or compose new fields from existing data.  
  - *Input:* From "Extract Data"  
  - *Output:* Prepared data for file conversion and Google Sheets upload  
  - *Potential Failures:* Data inconsistency, transformation errors  

---

#### 2.4 Data Output

**Overview:**  
This block converts the prepared data into an XLSX file and uploads or updates it in Google Sheets.

**Nodes Involved:**  
- Get Data in XLSX  
- Upsert Data in Sheets  

**Node Details:**

- **Get Data in XLSX**  
  - *Type:* ConvertToFile  
  - *Role:* Converts JSON data into an Excel XLSX file format.  
  - *Config:* Configured to convert input JSON array into XLSX with appropriate sheet name and formatting defaults.  
  - *Input:* From "Prepare Data"  
  - *Output:* Binary XLSX file passed to the next node  
  - *Potential Failures:* Conversion errors if data is malformed, file size limits  

- **Upsert Data in Sheets**  
  - *Type:* Google Sheets  
  - *Role:* Inserts or updates rows in a Google Sheets document with the prepared data.  
  - *Config:* Requires Google Sheets credentials (OAuth2), target spreadsheet ID, sheet name, and mapping of data fields to columns.  
  - *Input:* From "Get Data in XLSX" (may also accept JSON data depending on configuration)  
  - *Output:* Confirmation of data upsert operation  
  - *Potential Failures:* Authentication errors, permission issues, quota limits, incorrect spreadsheet or sheet IDs  

---

### 3. Summary Table

| Node Name             | Node Type                 | Functional Role                      | Input Node(s)           | Output Node(s)         | Sticky Note                         |
|-----------------------|---------------------------|------------------------------------|------------------------|------------------------|-----------------------------------|
| When chat message received | ChatTrigger (LangChain)     | Entry point, receives chat input   | -                      | Variables Chat          |                                   |
| Variables Chat         | Set                       | Initializes variables from input   | When chat message received | Initialization          |                                   |
| Initialization        | Set                       | Prepares parameters for API call   | Variables Chat          | Extract Serper Map      |                                   |
| Extract Serper Map     | HTTP Request              | Calls Serper Google Maps API       | Initialization          | Split Out Serper        |                                   |
| Split Out Serper       | SplitOut                  | Splits API response array           | Extract Serper Map      | Extract Data            |                                   |
| Extract Data           | Set                       | Extracts business data fields       | Split Out Serper        | Prepare Data            |                                   |
| Prepare Data           | Set                       | Formats and prepares data for output | Extract Data            | Get Data in XLSX, Upsert Data in Sheets |                                   |
| Get Data in XLSX       | ConvertToFile             | Converts JSON to XLSX file           | Prepare Data            | Upsert Data in Sheets   |                                   |
| Upsert Data in Sheets  | Google Sheets             | Inserts/updates data into Sheets    | Get Data in XLSX        | -                      |                                   |
| Sticky Note1           | Sticky Note               | -                                  | -                      | -                      |                                   |
| Sticky Note2           | Sticky Note               | -                                  | -                      | -                      |                                   |
| Sticky Note4           | Sticky Note               | -                                  | -                      | -                      |                                   |

*Note: Sticky Notes have no content in this workflow.*

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n and name it accordingly.**

2. **Add a "When chat message received" node (LangChain Chat Trigger):**  
   - Configure webhook ID or create a new webhook to receive chat messages.  
   - No additional parameters needed.

3. **Add a "Set" node named "Variables Chat":**  
   - Connect input from "When chat message received".  
   - Define variables needed for the search, such as the search query or location parameters extracted from incoming chat data.

4. **Add a "Set" node named "Initialization":**  
   - Connect input from "Variables Chat".  
   - Set parameters required for the API call: Serper API key, query parameters, request headers, etc.

5. **Add an "HTTP Request" node named "Extract Serper Map":**  
   - Connect input from "Initialization".  
   - Configure to make a request to Serper’s Google Maps search API endpoint.  
   - Set HTTP method (GET/POST), URL, headers including API key, and body/query parameters using expressions from "Initialization".  
   - Set response format to JSON.

6. **Add a "Split Out" node named "Split Out Serper":**  
   - Connect input from "Extract Serper Map".  
   - No special configuration; this splits the array of businesses into individual items.

7. **Add a "Set" node named "Extract Data":**  
   - Connect input from "Split Out Serper".  
   - Use expressions to extract fields such as business name, address, phone number, website, rating, and others from the split JSON items.  
   - Map these fields to new output data items.

8. **Add a "Set" node named "Prepare Data":**  
   - Connect input from "Extract Data".  
   - Format or clean data fields as needed for output compatibility, e.g., ensure consistent field naming or data types.

9. **Add a "ConvertToFile" node named "Get Data in XLSX":**  
   - Connect input from "Prepare Data".  
   - Configure to convert JSON data into an XLSX file.  
   - Optionally set sheet name and formatting options.

10. **Add a "Google Sheets" node named "Upsert Data in Sheets":**  
    - Connect input from "Get Data in XLSX".  
    - Configure Google Sheets credentials (OAuth2).  
    - Set target spreadsheet ID and sheet name.  
    - Map the data fields to columns for insertion or update.

11. **Validate all connections and parameter expressions.**

12. **Save and activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                          |
|-------------------------------------------------------------------------------------------------|----------------------------------------|
| This workflow leverages Serper API for Google Maps data extraction, which requires valid API keys. | Serper API documentation                |
| Google Sheets node requires OAuth2 credentials with appropriate permissions to edit spreadsheets. | Google Cloud Console setup for Sheets  |
| The LangChain Chat Trigger requires webhook setup to receive chat inputs externally.            | n8n LangChain integration guide        |
| For large data sets, consider API rate limits and data pagination handling for Serper API.      | Serper API rate limiting documentation |

---

**Disclaimer:**  
The text provided above is derived solely from an automated workflow created using n8n, an integration and automation tool. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.