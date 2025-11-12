Real Estate Intelligence Tracker with Bright Data & OpenAI

https://n8nworkflows.xyz/workflows/real-estate-intelligence-tracker-with-bright-data---openai-4281


# Real Estate Intelligence Tracker with Bright Data & OpenAI

---

### 1. Workflow Overview

This workflow, titled **Real Estate Intelligence Tracker with Bright Data & OpenAI**, is designed to automate the extraction, processing, and storage of real estate listing data from web sources. It leverages Bright Data’s Web Unlocker to scrape markdown-formatted real estate content and utilizes OpenAI’s GPT-4o language models for converting markdown into structured textual data and extracting key information such as property details and user reviews. The processed data is then aggregated, saved locally, pushed to Google Sheets, and sent via webhook notifications.

**Target Use Cases:**  
- Automated real estate data aggregation and intelligence tracking  
- Structured extraction of property details and user reviews from online listings  
- Integration with data storage and notification systems for real-time updates  

**Logical Blocks:**  
- **1.1 Input Reception and Configuration:** Manual trigger and setting of target URL and Bright Data zone  
- **1.2 Data Acquisition:** Requesting raw markdown data from Bright Data Web Unlocker API  
- **1.3 Markdown Processing:** Conversion of markdown content into plain textual data via OpenAI GPT-4o  
- **1.4 Information Extraction:** Splitting into two parallel extractors for structured property data and user reviews, both powered by OpenAI models  
- **1.5 Data Aggregation and Output:** Merging, aggregating, saving to file, pushing to Google Sheets, and triggering webhook notifications  
- **1.6 Documentation and Notes:** Sticky notes providing guidance and contextual information  

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Configuration

- **Overview:**  
  This block initiates workflow execution manually and sets required input parameters: the real estate listing URL and the corresponding Bright Data zone for scraping.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Set URL and Bright Data Zone (Set Node)  
  - Sticky Note (Instructional note)

- **Node Details:**  

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point enabling manual start of the workflow  
    - Configuration: Default, no parameters  
    - Input: None  
    - Output: Triggers the Set URL and Bright Data Zone node  
    - Edge Cases: None (manual trigger)  

  - **Set URL and Bright Data Zone**  
    - Type: Set Node  
    - Role: Defines key input variables for the scraping request  
    - Configuration:  
      - `url`: Target real estate listing URL (example: Redfin Chicago property)  
      - `zone`: Bright Data zone name (“web_unlocker1”) to specify scraping proxy settings  
    - Input: Trigger from manual node  
    - Output: Passes `url` and `zone` variables to the HTTP request node  
    - Edge Cases: Incorrect URL or zone name may cause request failures downstream  

  - **Sticky Note**  
    - Content: Advises setting the real estate website URL and Bright Data zone correctly; also mentions webhook notification URL setup  
    - Applies to: Input reception and configuration block  
    - Notes: Important for user awareness and preventing misconfiguration  

#### 1.2 Data Acquisition

- **Overview:**  
  Sends a POST request to Bright Data’s API to fetch the raw markdown data of the target real estate webpage using the provided URL and zone.

- **Nodes Involved:**  
  - Perform Bright Data Web Request (HTTP Request)  

- **Node Details:**  

  - **Perform Bright Data Web Request**  
    - Type: HTTP Request  
    - Role: Interacts with Bright Data API to scrape webpage content  
    - Configuration:  
      - Method: POST  
      - URL: “https://api.brightdata.com/request”  
      - Body Parameters: `zone`, `url`, `format=raw`, `data_format=markdown`  
      - Authentication: Header Auth via stored credentials (Bright Data API key)  
    - Input: Receives `url` and `zone` from Set node  
    - Output: Returns raw markdown data from the webpage under the `data` field  
    - Edge Cases:  
      - Authentication failure (invalid API key)  
      - Network issues or timeouts  
      - Invalid zone or URL causing no data or errors  
      - Unexpected response format  

#### 1.3 Markdown Processing

- **Overview:**  
  Converts the raw markdown content into plain textual data without markdown elements, links, scripts, or styles, preparing it for structured extraction.

- **Nodes Involved:**  
  - Markdown to Textual Data Extractor (Langchain LLM Chain)  
  - OpenAI Chat Model for Markdown to Textual (OpenAI GPT-4o model)  
  - Sticky Note1 (LLM Usage Information)  

- **Node Details:**  

  - **Markdown to Textual Data Extractor**  
    - Type: Langchain Chain LLM Node  
    - Role: Uses OpenAI GPT-4o to transform markdown into clean textual data  
    - Configuration:  
      - Prompt: “You need to analyze the below markdown and convert to textual data. Please do not output with your own thoughts. Make sure to output with textual data only with no links, scripts, css etc.”  
      - Messages: System message setting role as “markdown expert”  
      - Input Expression: `{{ $json.data }}` (markdown content)  
    - Input: Raw markdown from Bright Data request  
    - Output: Clean textual data passed to subsequent extractors  
    - Edge Cases:  
      - LLM API errors (rate limits, network)  
      - Unexpected markdown formatting causing incomplete conversion  

  - **OpenAI Chat Model for Markdown to Textual**  
    - Type: Langchain OpenAI Chat Model  
    - Role: GPT-4o-mini model for supporting the markdown to text chain  
    - Configuration: Model name “gpt-4o-mini”  
    - Credentials: OpenAI API key configured  
    - Input/Output: Linked via ai_languageModel interface with Markdown to Textual Data Extractor  

  - **Sticky Note1**  
    - Content: Explains LLM usage with GPT-4o, basic chain for markdown conversion, and info extractor usage  
    - Applies to: Markdown processing and extraction block  

#### 1.4 Information Extraction

- **Overview:**  
  Extracts structured real estate data and user reviews from the textual content using OpenAI-powered information extractor nodes, producing JSON-formatted outputs.

- **Nodes Involved:**  
  - Review Data Extractor (Information Extractor Langchain)  
  - Structured Data Extractor (Information Extractor Langchain)  
  - OpenAI Chat Model for Review Data Extractor (OpenAI GPT-4o)  
  - OpenAI Chat Model for Structured Data (OpenAI GPT-4o)  

- **Node Details:**  

  - **Review Data Extractor**  
    - Type: Langchain Information Extractor  
    - Role: Extracts all review entries from textual content in JSON schema format  
    - Configuration:  
      - Text: `Extract all the reviews from the provided content {{ $json.text }}`  
      - JSON Schema Example: Array of review objects with date, text, and rating fields  
    - Input: Textual data from Markdown to Textual Data Extractor  
    - Output: JSON array of reviews  
    - Edge Cases:  
      - Missing or malformed review data in text  
      - LLM extraction errors  

  - **Structured Data Extractor**  
    - Type: Langchain Information Extractor  
    - Role: Extracts structured real estate listing data (property details, pricing, agent info) as JSON  
    - Configuration:  
      - Text: `Extract structured data from the provided content {{ $json.text }}`  
      - JSON Schema Example: Detailed schema including address, property features, price, seller info, etc.  
    - Input: Textual data from Markdown to Textual Data Extractor  
    - Output: JSON object of structured listing data  
    - Edge Cases:  
      - Incomplete data extraction if text lacks details  
      - Schema mismatch or LLM errors  

  - **OpenAI Chat Model for Review Data Extractor**  
    - Type: Langchain OpenAI Chat Model  
    - Role: GPT-4o-mini model supporting review extraction  
    - Configuration: Model “gpt-4o-mini” with OpenAI API credentials  

  - **OpenAI Chat Model for Structured Data**  
    - Type: Langchain OpenAI Chat Model  
    - Role: GPT-4o-mini model supporting structured data extraction  
    - Configuration: Model “gpt-4o-mini” with OpenAI API credentials  

#### 1.5 Data Aggregation and Output

- **Overview:**  
  Merges outputs from review and structured data extractors, aggregates the collected data, and pushes it to multiple outputs: Google Sheets, local file system, and a webhook notification.

- **Nodes Involved:**  
  - Merge the responses (Merge Node)  
  - Aggregate the responses (Aggregate Node)  
  - Google Sheets (Google Sheets Node)  
  - Create a binary data for Structured Data Extract (Function Node)  
  - Write the structured content to disk (Read/Write File Node)  
  - Initiate a Webhook Notification for the Structured Data (HTTP Request)  
  - Sticky Note4 (Outbound Data Push explanation)  

- **Node Details:**  

  - **Merge the responses**  
    - Type: Merge Node  
    - Role: Combines parallel outputs from Review Data Extractor and Structured Data Extractor into a single stream  
    - Configuration: Default (no special mode)  
    - Input: Two inputs from review and structured extractors  
    - Output: Merged data forwarded to aggregation  

  - **Aggregate the responses**  
    - Type: Aggregate Node  
    - Role: Aggregates all merged data items into one combined item for output  
    - Configuration: Aggregate all item data into one output  
    - Input: From Merge node  
    - Output: Aggregated combined JSON data for further processing  

  - **Google Sheets**  
    - Type: Google Sheets Node  
    - Role: Appends or updates a Google Sheet with the aggregated real estate data  
    - Configuration:  
      - Operation: appendOrUpdate  
      - Sheet name: “Sheet1” (gid=0)  
      - Document ID: Specific Google Sheet for real estate data  
      - Columns: Maps `data` field containing JSON stringified data  
    - Credentials: Google Sheets OAuth2 account  
    - Input: Aggregated data  

  - **Create a binary data for Structured Data Extract**  
    - Type: Function Node  
    - Role: Converts JSON structured data into base64-encoded binary data for file writing  
    - Configuration: JavaScript code encoding JSON stringified data into base64 buffer under binary field named `data`  
    - Input: Aggregated data  

  - **Write the structured content to disk**  
    - Type: Read/Write File Node  
    - Role: Writes the binary structured data to local disk as JSON file  
    - Configuration:  
      - Operation: write  
      - File path: `d:\Realestate-StructuredData.json`  
    - Input: Binary data from Function node  
    - Edge Cases:  
      - File system permission errors  
      - Disk space limitations  

  - **Initiate a Webhook Notification for the Structured Data**  
    - Type: HTTP Request  
    - Role: Sends aggregated data summary to a webhook URL for notification or integration  
    - Configuration:  
      - Method: POST  
      - URL: Fixed webhook.site URL (for testing or notification)  
      - Body parameter: summary with aggregated data  
    - Input: Aggregated data  
    - Edge Cases:  
      - Webhook endpoint downtime or rejection  

  - **Sticky Note4**  
    - Content: Explains outbound data push strategy to multiple destinations (Google Sheets, Disk, Webhook)  
    - Applies to: Data output block  

#### 1.6 Documentation and Notes

- **Overview:**  
  Provides visual notes with branding and usage instructions.

- **Nodes Involved:**  
  - Sticky Note5 (Logo)  

- **Node Details:**  

  - **Sticky Note5**  
    - Content: Displays Bright Data logo via embedded image URL  
    - Role: Branding and visual identification  

---

### 3. Summary Table

| Node Name                                | Node Type                        | Functional Role                           | Input Node(s)                           | Output Node(s)                                          | Sticky Note                                                                                                                       |
|-----------------------------------------|---------------------------------|-----------------------------------------|---------------------------------------|---------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’            | Manual Trigger                  | Workflow entry point                     | None                                  | Set URL and Bright Data Zone                             | Deals with the Realestate data extraction by utilizing the Bright Data Web Unlocker Product. Please set URL and Bright Data zone. |
| Set URL and Bright Data Zone             | Set                            | Sets URL and Bright Data zone parameters| When clicking ‘Test workflow’          | Perform Bright Data Web Request                          | Same as above                                                                                                                    |
| Perform Bright Data Web Request           | HTTP Request                   | Scrapes real estate page via Bright Data| Set URL and Bright Data Zone           | Markdown to Textual Data Extractor                       | Same as above                                                                                                                    |
| Markdown to Textual Data Extractor        | Langchain Chain LLM            | Converts markdown to plain text          | Perform Bright Data Web Request        | Review Data Extractor, Structured Data Extractor        | LLM Usages: OpenAI GPT-4o model used for markdown conversion and information extraction                                          |
| OpenAI Chat Model for Markdown to Textual| Langchain OpenAI Chat Model    | Supports markdown to textual conversion  | N/A (ai_languageModel for above node) | Markdown to Textual Data Extractor                       | Same as above                                                                                                                    |
| Review Data Extractor                     | Langchain Information Extractor| Extracts user reviews from text          | Markdown to Textual Data Extractor     | Merge the responses                                     | Same as above                                                                                                                    |
| OpenAI Chat Model for Review Data Extractor| Langchain OpenAI Chat Model   | Supports review data extraction           | N/A (ai_languageModel for above node) | Review Data Extractor                                    | Same as above                                                                                                                    |
| Structured Data Extractor                 | Langchain Information Extractor| Extracts structured real estate data     | Markdown to Textual Data Extractor     | Merge the responses                                     | Same as above                                                                                                                    |
| OpenAI Chat Model for Structured Data    | Langchain OpenAI Chat Model    | Supports structured data extraction       | N/A (ai_languageModel for above node) | Structured Data Extractor                                | Same as above                                                                                                                    |
| Merge the responses                       | Merge                         | Merges review and structured data results| Review Data Extractor, Structured Data Extractor | Aggregate the responses                                 | Outbound Data Push: merging and pushing data to multiple destinations                                                          |
| Aggregate the responses                   | Aggregate                     | Aggregates merged data into single item  | Merge the responses                    | Google Sheets, Initiate a Webhook Notification, Create a binary data for Structured Data Extract | Same as above                                                                                                                    |
| Google Sheets                            | Google Sheets                  | Appends aggregated data to Google Sheet  | Aggregate the responses                | None                                                    | Same as above                                                                                                                    |
| Initiate a Webhook Notification for the Structured Data| HTTP Request       | Sends aggregated data to webhook          | Aggregate the responses                | None                                                    | Same as above                                                                                                                    |
| Create a binary data for Structured Data Extract| Function                   | Encodes JSON data into base64 binary      | Aggregate the responses                | Write the structured content to disk                    | Same as above                                                                                                                    |
| Write the structured content to disk     | Read/Write File               | Writes structured JSON data to local file | Create a binary data for Structured Data Extract | None                                                    | Same as above                                                                                                                    |
| Sticky Note                             | Sticky Note                   | Instructional note on URL and zone setup  | None                                  | None                                                    | Deals with the Realestate data extraction by utilizing the Bright Data Web Unlocker Product. Please set URL and Bright Data zone. |
| Sticky Note1                            | Sticky Note                   | Explains LLM usage in markdown conversion | None                                  | None                                                    | LLM Usages: OpenAI GPT-4o model, basic chain, information extractor usage                                                    |
| Sticky Note4                            | Sticky Note                   | Explains outbound data push process        | None                                  | None                                                    | Outbound Data Push: sending to Google Sheets, disk, webhook                                                                  |
| Sticky Note5                            | Sticky Note                   | Displays Bright Data logo                    | None                                  | None                                                    |                                                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: “When clicking ‘Test workflow’”  
   - Purpose: Manual start of workflow  

2. **Create Set Node**  
   - Name: “Set URL and Bright Data Zone”  
   - Add fields:  
     - `url` (string): Set to target real estate listing URL (e.g., `https://www.redfin.com/IL/Chicago/5814-W-Roscoe-St-60634/home/13464782`)  
     - `zone` (string): Set to Bright Data zone name (e.g., `web_unlocker1`)  
   - Connect from Manual Trigger node  

3. **Create HTTP Request Node for Bright Data API**  
   - Name: “Perform Bright Data Web Request”  
   - Method: POST  
   - URL: `https://api.brightdata.com/request`  
   - Authentication: HTTP Header Auth using Bright Data API Key credentials  
   - Body Parameters (form-data):  
     - `zone`: `={{ $json.zone }}`  
     - `url`: `={{ $json.url }}`  
     - `format`: `raw`  
     - `data_format`: `markdown`  
   - Connect from Set node  

4. **Create Langchain Chain LLM Node**  
   - Name: “Markdown to Textual Data Extractor”  
   - Text input:  
     ```
     You need to analyze the below markdown and convert to textual data. Please do not output with your own thoughts. Make sure to output with textual data only with no links, scripts, css etc.

     {{ $json.data }}
     ```  
   - Prompt type: Define  
   - Messages: Add system role message “You are a markdown expert”  
   - Connect from HTTP Request node  

5. **Create OpenAI Chat Model Node**  
   - Name: “OpenAI Chat Model for Markdown to Textual”  
   - Model: `gpt-4o-mini`  
   - Credentials: OpenAI API key  
   - Connect as ai_languageModel to “Markdown to Textual Data Extractor” node  

6. **Create Two Langchain Information Extractor Nodes**  
   - a) “Review Data Extractor”  
     - Text:  
       ```
       Extract all the reviews from the provided content

       {{ $json.text }}
       ```  
     - Schema Type: fromJson  
     - JSON Schema Example: Array of Review objects (datePublished, review text, rating)  
     - Connect from “Markdown to Textual Data Extractor”  

   - b) “Structured Data Extractor”  
     - Text:  
       ```
       Extract structured data from the provided content

       {{ $json.text }}
       ```  
     - Schema Type: fromJson  
     - JSON Schema Example: Real estate offer schema with property details, price, seller info  
     - Connect from “Markdown to Textual Data Extractor”  

7. **Create OpenAI Chat Model Nodes for Each Extractor**  
   - “OpenAI Chat Model for Review Data Extractor”  
     - Model: `gpt-4o-mini`  
     - Credentials: OpenAI API key  
     - Connect as ai_languageModel to “Review Data Extractor”  

   - “OpenAI Chat Model for Structured Data”  
     - Model: `gpt-4o-mini`  
     - Credentials: OpenAI API key  
     - Connect as ai_languageModel to “Structured Data Extractor”  

8. **Create Merge Node**  
   - Name: “Merge the responses”  
   - Input 1: Connect from “Review Data Extractor”  
   - Input 2: Connect from “Structured Data Extractor”  

9. **Create Aggregate Node**  
   - Name: “Aggregate the responses”  
   - Operation: Aggregate all item data  
   - Connect from “Merge the responses”  

10. **Create Google Sheets Node**  
    - Name: “Google Sheets”  
    - Operation: Append or update  
    - Document ID: Set your Google Sheet document ID  
    - Sheet Name: Set to “Sheet1” or your target sheet  
    - Columns: Map `data` field with `={{ $json.data }}`  
    - Credentials: Google Sheets OAuth2 credentials  
    - Connect from “Aggregate the responses”  

11. **Create Function Node**  
    - Name: “Create a binary data for Structured Data Extract”  
    - Code:
      ```javascript
      items[0].binary = {
        data: {
          data: Buffer.from(JSON.stringify(items[0].json, null, 2)).toString('base64')
        }
      };
      return items;
      ```
    - Connect from “Aggregate the responses”  

12. **Create Read/Write File Node**  
    - Name: “Write the structured content to disk”  
    - Operation: Write  
    - File path: `d:\Realestate-StructuredData.json` (update as needed)  
    - Connect from Function node  

13. **Create HTTP Request Node for Webhook Notification**  
    - Name: “Initiate a Webhook Notification for the Structured Data”  
    - Method: POST  
    - URL: Your webhook URL (example uses `https://webhook.site/7b5380a0-0544-48dc-be43-0116cb2d52c2`)  
    - Body Parameters: Add field `summary` with value `={{ $json.data }}`  
    - Connect from “Aggregate the responses”  

14. **Add Sticky Notes** (optional for documentation)  
    - Add notes explaining configuration of URLs, Bright Data zones, LLM usage, outbound data push, branding logo, etc., positioned near respective nodes for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                                   |
|---------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| Deals with the Realestate data extraction by utilizing the Bright Data Web Unlocker Product. Please set the real estate website URL with the Bright Data zone name. Also update the Webhook Notification URL of your interest. | Sticky note near input configuration block                                                                       |
| OpenAI GPT 4o model is used. Basic LLM Chain for converting markdown to textual content. Information Extractor is used for structured data extraction. | Sticky note near LLM nodes                                                                                        |
| Outbound data handling by merging, aggregating the data and pushing the same to multiple sources such as Google Sheets, Save to Disk, Webhook Notification. | Sticky note near data output nodes                                                                                 |
| ![Bright Data Logo](https://images.seeklogo.com/logo-png/43/1/brightdata-logo-png_seeklogo-439974.png)          | Branding note                                                                                                     |
| Bright Data Web Unlocker API documentation: https://brightdata.com/docs/web-unlocker/api                        | External resource for configuring and troubleshooting Bright Data API                                            |
| OpenAI API documentation: https://platform.openai.com/docs/api-reference                                      | Reference for OpenAI API usage and limits                                                                        |

---

**Disclaimer:**  
The provided content originates exclusively from an automated workflow created with n8n, respecting all applicable content policies and legal constraints. All processed data is legal and publicly accessible.

---