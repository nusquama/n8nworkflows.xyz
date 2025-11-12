✨ Vision-Based AI Agent Scraper - with Google Sheets, ScrapingBee, and Gemini

https://n8nworkflows.xyz/workflows/--vision-based-ai-agent-scraper---with-google-sheets--scrapingbee--and-gemini-2563


# ✨ Vision-Based AI Agent Scraper - with Google Sheets, ScrapingBee, and Gemini

### 1. Workflow Overview

This workflow implements a **vision-based AI agent scraper** designed to extract structured data from webpages, primarily targeting e-commerce sites. It automates the process of data extraction by combining visual analysis of full-page screenshots with HTML scraping as a fallback, thus eliminating the need for complex DOM-based selectors like XPath or CSS selectors.

The workflow is organized into the following logical blocks:

- **1.1 Input Reception and URL Retrieval**  
  Starts with a manual trigger and fetches the list of URLs to scrape from a Google Sheets document.

- **1.2 Request Preparation and Screenshot Capture**  
  Sets required fields and interacts with ScrapingBee API to capture full-page screenshots of each URL.

- **1.3 Vision-Based AI Scraping Agent**  
  Uses the Gemini-1.5-Pro Google Gemini Chat Model as a multimodal LLM to extract product data from screenshots. If visual extraction is insufficient, the AI agent triggers an HTML scraping fallback.

- **1.4 HTML Scraping Fallback Tool**  
  When the vision-based extraction finds missing or ambiguous data, this block calls a sub-workflow to retrieve HTML content via ScrapingBee and converts it to Markdown for token-efficient AI parsing.

- **1.5 Structured Output Parsing and Data Storage**  
  Parses the AI agent’s output into structured JSON, splits the results into individual entries, and appends these to the Google Sheets “Results” sheet.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and URL Retrieval

**Overview:**  
This block triggers the workflow manually and loads the URLs to be scraped from a Google Sheets spreadsheet.

**Nodes Involved:**  
- When clicking ‘Test workflow’  
- Google Sheets - Get list of URLs  
- Set fields

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Starts the workflow on manual command.  
  - Configuration: Default (no parameters).  
  - Inputs: None  
  - Outputs: To “Google Sheets - Get list of URLs”  
  - Edge cases: None; requires manual start.

- **Google Sheets - Get list of URLs**  
  - Type: Google Sheets node  
  - Role: Fetches the list of URLs to scrape from the first sheet ("List of URLs") of a Google Sheets document, authenticated via a service account.  
  - Configuration: Reads from a specified sheet and document ID (customizable).  
  - Inputs: From manual trigger  
  - Outputs: JSON array containing URLs  
  - Edge cases: Authentication errors, empty or malformed sheet data.

- **Set fields**  
  - Type: Set node  
  - Role: Defines fields for downstream nodes, specifically passing the URL field as is.  
  - Configuration: Assigns `url` field from the JSON input.  
  - Inputs: URLs from Google Sheets  
  - Outputs: Passes data to ScrapingBee screenshot node  
  - Edge cases: Missing or malformed URL field could cause downstream failures.

---

#### 2.2 Request Preparation and Screenshot Capture

**Overview:**  
This block uses ScrapingBee to capture full-page screenshots of the URLs. The screenshots serve as inputs for the vision-based AI agent.

**Nodes Involved:**  
- ScrapingBee - Get page screenshot

**Node Details:**

- **ScrapingBee - Get page screenshot**  
  - Type: HTTP Request node  
  - Role: Calls the ScrapingBee API to capture full-page screenshots of URLs.  
  - Configuration:  
    - URL: `https://app.scrapingbee.com/api/v1`  
    - Query parameters: API key, URL to capture, `screenshot_full_page=true`  
    - Headers: User-Agent string mimicking a desktop browser  
  - Inputs: URL from “Set fields” node  
  - Outputs: Screenshot image data (binary) and metadata  
  - Edge cases: API key misconfiguration, request timeouts, rate limits, invalid URLs.

---

#### 2.3 Vision-Based AI Scraping Agent

**Overview:**  
The core AI agent analyzes screenshots to extract product data. It uses a customized system prompt instructing how to parse data visually and a fallback mechanism triggering HTML scraping if needed.

**Nodes Involved:**  
- Vision-based Scraping Agent  
- Google Gemini Chat Model  
- HTML-based Scraping Tool (sub-workflow)  
- Structured Output Parser

**Node Details:**

- **Vision-based Scraping Agent**  
  - Type: LangChain Agent node  
  - Role: Coordinates the AI scraping process using multimodal input (screenshot + URL).  
  - Configuration:  
    - System prompt defines extraction tasks (titles, prices, brands, promo info) and fallback to HTML scraping tool if vision fails.  
    - User message provides page URL context.  
    - Passthrough binary images enabled to send screenshot.  
    - Output parser enabled for structured JSON output.  
  - Inputs: Screenshot from ScrapingBee, URL field  
  - Outputs: Parsed JSON data ready for splitting  
  - Edge cases: AI misinterpretation, incomplete data extraction, fallback triggering logic errors.

- **Google Gemini Chat Model**  
  - Type: LangChain LLM node  
  - Role: Executes the Gemini-1.5-Pro model for vision and text understanding.  
  - Configuration: Model set to `models/gemini-1.5-pro-latest` with Google Palm API credentials.  
  - Inputs: From agent node as language model  
  - Outputs: AI responses for agent processing  
  - Edge cases: API quota limits, authentication errors, latency.

- **HTML-based Scraping Tool**  
  - Type: LangChain tool invoking a sub-workflow  
  - Role: Fallback tool called by the agent to scrape HTML if screenshot extraction fails.  
  - Configuration: Calls a sub-workflow named “HTMLScrapingTool” within the same workflow ID.  
  - Inputs: URL query from agent  
  - Outputs: HTML content as `data` property  
  - Edge cases: Sub-workflow errors, missing URL parameter.

- **Structured Output Parser**  
  - Type: LangChain Structured Output Parser  
  - Role: Converts raw AI output into a JSON structure matching the defined schema for e-commerce data.  
  - Configuration: JSON schema example includes product title, price, brand, promo flag, and promo percentage.  
  - Inputs: AI agent output  
  - Outputs: Structured JSON array for further processing  
  - Edge cases: Output not matching schema, parsing errors.

---

#### 2.4 HTML Scraping Fallback Tool

**Overview:**  
This sub-workflow is triggered by the AI agent when screenshot-based extraction is insufficient. It fetches the page HTML, converts it to Markdown, and sends it back to the agent for parsing.

**Nodes Involved:**  
- HTML-Scraping Tool (Execute Workflow Trigger)  
- Set fields - from AI agent query  
- ScrapingBee- Get page HTML  
- HTML to Markdown

**Node Details:**

- **HTML-Scraping Tool**  
  - Type: Execute Workflow Trigger node  
  - Role: Entry point for the HTML scraping sub-workflow, triggered on-demand.  
  - Inputs: From AI agent query  
  - Outputs: URL parameter passed to “Set fields - from AI agent query”  
  - Edge cases: Trigger failures, parameter missing.

- **Set fields - from AI agent query**  
  - Type: Set node  
  - Role: Extracts the `query` parameter (URL) from agent and sets it as `url` field for HTTP request.  
  - Inputs: URL query from trigger  
  - Outputs: To ScrapingBee HTML fetch node  
  - Edge cases: Missing or invalid query parameter.

- **ScrapingBee- Get page HTML**  
  - Type: HTTP Request node  
  - Role: Calls ScrapingBee API to retrieve raw HTML of the page.  
  - Configuration:  
    - URL: `https://app.scrapingbee.com/api/v1`  
    - Query parameters: API key, URL to scrape  
  - Inputs: URL from “Set fields - from AI agent query”  
  - Outputs: Raw HTML as `data` property  
  - Edge cases: API limits, malformed URLs.

- **HTML to Markdown**  
  - Type: Markdown node  
  - Role: Converts raw HTML into Markdown format to reduce tokens when sending data back to the AI model.  
  - Inputs: HTML string from ScrapingBee  
  - Outputs: Markdown text fed back to AI agent through the tool’s response property  
  - Edge cases: Conversion errors, malformed HTML.

---

#### 2.5 Structured Output Parsing and Data Storage

**Overview:**  
This block formats the AI agent’s JSON output into rows and appends them to a Google Sheets “Results” sheet for easy data consumption.

**Nodes Involved:**  
- Split Out  
- Google Sheets - Create Rows

**Node Details:**

- **Split Out**  
  - Type: Split Out node  
  - Role: Splits the parsed JSON array from the AI output into individual items (rows) for insertion.  
  - Inputs: Structured JSON array from the agent  
  - Outputs: Individual JSON objects to the Google Sheets node  
  - Edge cases: Empty or malformed JSON arrays.

- **Google Sheets - Create Rows**  
  - Type: Google Sheets node  
  - Role: Appends rows to the “Results” sheet in the configured Google Sheets document.  
  - Configuration:  
    - Maps JSON fields such as promo, category (mapped from URL), product_url (mapped from product_title), product_brand, product_price, promo_percent.  
    - Authenticated via service account.  
  - Inputs: Individual JSON product entries  
  - Outputs: Confirmation of row creation  
  - Edge cases: Authentication issues, sheet schema mismatches, quota limits.

---

### 3. Summary Table

| Node Name                     | Node Type                          | Functional Role                              | Input Node(s)                        | Output Node(s)                      | Sticky Note                                                                                                                         |
|-------------------------------|-----------------------------------|----------------------------------------------|------------------------------------|-----------------------------------|------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’  | Manual Trigger                    | Starts workflow manually                      | None                               | Google Sheets - Get list of URLs   | The default trigger is **When clicking ‘Test workflow’**, meaning the workflow will **need to be triggered manually**. You can replace this by selecting a **trigger of your choice**. |
| Google Sheets - Get list of URLs | Google Sheets                    | Retrieves URLs list to scrape                 | When clicking ‘Test workflow’      | Set fields                        | The Google Sheet will contain two sheets:  \n- **List of URLs to** scrape  \n- **Results** page, populated with the scraping results and AI-extracted data.\n\nExample sheet link included. |
| Set fields                    | Set                              | Defines fields for scraping API calls        | Google Sheets - Get list of URLs   | ScrapingBee - Get page screenshot | This node allows you to **define the fields** that will be sent to the **ScrapingBee HTTP Node** and the AI Agent. Only `url` is pre-configured here.                                   |
| ScrapingBee - Get page screenshot | HTTP Request                    | Captures full-page screenshot via ScrapingBee | Set fields                        | Vision-based Scraping Agent       | Uses ScrapingBee to capture full-page screenshots. Screenshot parameter `screenshot_full_page` set to true. User-Agent header mimics browser.                                     |
| Vision-based Scraping Agent   | LangChain Agent                  | AI agent that extracts data visually and triggers fallback | ScrapingBee - Get page screenshot | Split Out                         | Central AI node using Gemini-1.5-Pro. Uses screenshot and URL for data extraction. Calls HTML scraping fallback if needed. Prompts detailed in sticky note.                         |
| Google Gemini Chat Model      | LangChain LLM                   | Provides multimodal AI model for vision-based extraction | Vision-based Scraping Agent (LM)  | Vision-based Scraping Agent       | Gemini-1.5-Pro model chosen for superior vision capabilities.                                                                                                                |
| HTML-based Scraping Tool      | LangChain Tool (Sub-workflow)   | Fallback tool to scrape HTML content          | Vision-based Scraping Agent        | Vision-based Scraping Agent       | Invoked only when image-based extraction fails. Calls a sub-workflow to retrieve HTML and convert it to Markdown.                                                             |
| Structured Output Parser      | LangChain Output Parser          | Parses AI output into structured JSON         | Vision-based Scraping Agent        | Vision-based Scraping Agent       | Organizes extracted data into JSON for e-commerce scraping. Schema customizable.                                                                                              |
| Split Out                    | Split Out                        | Splits JSON array into individual items       | Vision-based Scraping Agent        | Google Sheets - Create Rows       | Splits parsed JSON into individual rows for Google Sheets insertion.                                                                                                        |
| Google Sheets - Create Rows   | Google Sheets                    | Appends extracted data to “Results” sheet     | Split Out                        | None                            | Appends structured data as rows in Google Sheets. Columns mapped from AI output fields. Example sheet provided for reference.                                                   |
| HTML-Scraping Tool           | Execute Workflow Trigger         | Entry point for HTML scraping sub-workflow    | Vision-based Scraping Agent        | Set fields - from AI agent query  | Triggered by AI agent fallback only. Receives URL as parameter.                                                                                                               |
| Set fields - from AI agent query | Set                           | Sets `url` field from agent’s query parameter  | HTML-Scraping Tool                | ScrapingBee- Get page HTML       | Defines URL field for the HTML scraping API call.                                                                                                                              |
| ScrapingBee- Get page HTML    | HTTP Request                    | Retrieves raw HTML content of the page         | Set fields - from AI agent query  | HTML to Markdown                 | Calls ScrapingBee to fetch page HTML for fallback extraction.                                                                                                                  |
| HTML to Markdown             | Markdown                        | Converts HTML to Markdown to save tokens        | ScrapingBee- Get page HTML         | HTML-based Scraping Tool          | Converts HTML to Markdown to optimize token usage when feeding AI.                                                                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node**  
   - Add a **Manual Trigger** node named “When clicking ‘Test workflow’”.

2. **Fetch URLs from Google Sheets**  
   - Add a **Google Sheets** node named “Google Sheets - Get list of URLs”.  
   - Configure with:  
     - Authentication: Service Account credentials with access to your Google Sheet.  
     - Operation: Read rows from a sheet named “List of URLs” (gid=0).  
     - Document ID: Set to your Google Sheets document ID.

3. **Set Fields for Scraping**  
   - Add a **Set** node named “Set fields”.  
   - Assign one field: `url` = `{{$json.url}}`.  
   - Connect output from “Google Sheets - Get list of URLs”.

4. **ScrapingBee - Capture Screenshot**  
   - Add an **HTTP Request** node named “ScrapingBee - Get page screenshot”.  
   - Configure:  
     - URL: `https://app.scrapingbee.com/api/v1`  
     - Query parameters:  
       - `api_key`: Your ScrapingBee API key  
       - `url`: `{{$json.url}}`  
       - `screenshot_full_page`: `true`  
     - Headers: Set `User-Agent` to a desktop browser string.  
   - Connect from “Set fields”.

5. **Configure Vision-Based AI Agent**  
   - Add a **LangChain Agent** node named “Vision-based Scraping Agent”.  
   - Parameters:  
     - Text: Provide user message with URL (`Here is the screenshot you need to use: {{ $json.url }}`).  
     - System message: Use prompt instructing extraction of product title, price, brand, promotion info from screenshot; fallback to HTML scraping if needed.  
     - Enable passthrough binary images.  
     - Enable output parser.  
   - Connect input from “ScrapingBee - Get page screenshot”.

6. **Add Google Gemini Chat Model**  
   - Add a **LangChain LLM** node named “Google Gemini Chat Model”.  
   - Configure model to `models/gemini-1.5-pro-latest`.  
   - Set credentials for Google Palm API.  
   - Connect as the language model input of “Vision-based Scraping Agent”.

7. **Add HTML-based Scraping Tool Sub-workflow**  
   - Create a separate workflow named “HTMLScrapingTool” with following nodes:  
     - **Execute Workflow Trigger** node named “HTML-Scraping Tool” (entry point).  
     - **Set** node named “Set fields - from AI agent query” that assigns `url` from input query parameter.  
     - **HTTP Request** node named “ScrapingBee- Get page HTML” configured to fetch HTML from ScrapingBee with `api_key` and `url`.  
     - **Markdown** node named “HTML to Markdown” that converts HTML (`{{$json.data}}`) to Markdown.  
   - Back in main workflow, add a **LangChain Tool Workflow** node named “HTML-based Scraping Tool”.  
   - Configure to call “HTMLScrapingTool” by workflow ID and set response property as `data`.  
   - Connect as AI tool input to “Vision-based Scraping Agent”.

8. **Add Structured Output Parser**  
   - Add a **LangChain Structured Output Parser** node named “Structured Output Parser”.  
   - Define JSON schema example matching expected e-commerce output fields: product_title, product_price, product_brand, promo (boolean), promo_percentage (number).  
   - Connect as AI output parser input to “Vision-based Scraping Agent”.

9. **Split Parsed Output**  
   - Add a **Split Out** node named “Split Out”.  
   - Connect input from “Vision-based Scraping Agent” main output.

10. **Store Results in Google Sheets**  
    - Add a **Google Sheets** node named “Google Sheets - Create Rows”.  
    - Configure:  
      - Authentication: Service Account.  
      - Operation: Append rows to “Results” sheet (gid for results).  
      - Document ID: Same as “Get list of URLs”.  
      - Map columns to JSON fields: promo, category (map from URL), product_url (product_title), product_brand, product_price, promo_percent.  
    - Connect from “Split Out”.

11. **Connect Workflow Sequence**  
    - Trigger → Google Sheets - Get list of URLs → Set fields → ScrapingBee - Get page screenshot → Vision-based Scraping Agent → Split Out → Google Sheets - Create Rows.

12. **Set Credentials**  
    - Configure Google Sheets nodes with service account credentials having read/write access.  
    - Configure ScrapingBee nodes with valid API key.  
    - Configure Google Gemini Chat Model node with valid Google Palm API credentials.

13. **Test and Validate**  
    - Test the workflow manually with some URLs in the Google Sheets input sheet.  
    - Monitor fallback to HTML scraping via tool invocation if screenshot extraction is incomplete.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                       | Context or Link                                                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------|
| This workflow involves scraping, so **make sure to check the legal regulations around scraping in your country before getting started**. Better safe than sorry!                                                                                                                                                                                                | Workflow Overview and Sticky Note on legal considerations.                                                                         |
| Example Google Sheet template with “List of URLs” and “Results” sheets is available here: [Example Google Sheet](https://docs.google.com/spreadsheets/d/10Gc7ooUeTBbOOE6bgdNe5vSKRkkcAamonsFSjFevkOE/)                                                                                                                                                               | Used for managing input URLs and storing scraped data.                                                                             |
| ScrapingBee offers 1,000 free requests for new users. Their API supports full-page screenshots (required for vision-based scraping) and HTML retrieval for fallback extraction.                                                                                                                                                                                    | ScrapingBee API info and free tier trial.                                                                                         |
| Gemini-1.5-Pro model outperforms GPT-4 and GPT-4-Vision for this visual scraping use case. However, it is the most expensive model, so use it judiciously.                                                                                                                                                                                                        | AI model choice rationale.                                                                                                         |
| HTML is converted into Markdown to reduce token usage when sending to the AI model, optimizing cost and performance.                                                                                                                                                                                                                                            | Token efficiency tip in AI processing.                                                                                            |
| The Structured Output Parser node schema should be adapted if you change the target data fields or use cases beyond e-commerce.                                                                                                                                                                                                                                | Customization note.                                                                                                                |
| The AI Agent includes fallback logic to call the HTML scraping sub-workflow only if the screenshot extraction is incomplete or ambiguous, preventing unnecessary API calls and saving costs.                                                                                                                                                                    | Efficiency and cost-saving design consideration.                                                                                  |

---

This document fully describes the workflow’s structure, nodes, configuration, and logic to enable reproduction, modification, and troubleshooting for advanced users and automation agents.