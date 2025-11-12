Scrape Amazon Keyboard Products with ScrapeGraphAI to Google Sheets

https://n8nworkflows.xyz/workflows/scrape-amazon-keyboard-products-with-scrapegraphai-to-google-sheets-6394


# Scrape Amazon Keyboard Products with ScrapeGraphAI to Google Sheets

---

### 1. Workflow Overview

This workflow automates the scraping of Amazon keyboard product listings using an AI-powered scraping service (ScrapeGraphAI) and stores the structured results in Google Sheets. It is designed for users who want to monitor and analyze Amazon product data regularly without manual effort.

**Use Cases:**  
- Automated market research for Amazon keyboard products  
- Periodic data collection to track product availability and trends  
- Integration of AI-based web scraping with spreadsheet data storage  

**Logical Blocks:**  
- **1.1 Automated Trigger:** Scheduling periodic execution of the workflow  
- **1.2 AI-Powered Scraping:** Using ScrapeGraphAI to extract structured product data from Amazon search results  
- **1.3 Data Formatting:** Processing and cleaning the scraped data into a standardized schema  
- **1.4 Data Storage:** Appending processed data into a Google Sheets document for analysis

---

### 2. Block-by-Block Analysis

#### 1.1 Automated Trigger

- **Overview:**  
This block initiates the workflow automatically at regular intervals, ensuring fresh data is collected without manual intervention.

- **Nodes Involved:**  
  - Automated Schedule Trigger

- **Node Details:**  

  - **Automated Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Starts workflow execution on a recurring schedule  
    - Configuration: Runs at a defined interval (default unspecified, typically daily or custom)  
    - Expressions/Variables: None specified  
    - Inputs: None (trigger node)  
    - Outputs: Connected to AI-Powered Amazon Product Scraper node  
    - Version: 1.2  
    - Edge Cases:  
      - Misconfigured intervals may cause excessive executions or no runs  
      - Time zone mismatches may shift execution times  
      - Workflow disabled state will prevent triggering  
    - Notes: Sticky Note describes best practices for scheduling and frequency selection  

#### 1.2 AI-Powered Scraping

- **Overview:**  
Invokes the ScrapeGraphAI node to extract product information from the Amazon keyboard search results page using natural language instructions.

- **Nodes Involved:**  
  - AI-Powered Amazon Product Scraper  
  - Sticky Note1 (documentation)

- **Node Details:**  

  - **AI-Powered Amazon Product Scraper**  
    - Type: ScrapeGraphAI Node (community node)  
    - Role: Executes AI-driven web scraping on a specified Amazon search results URL  
    - Configuration:  
      - Website URL: Amazon search page for keyboards (https://www.amazon.com/s?k=keyboard...)  
      - User Prompt: Instruction to extract products with fields title, url, category in JSON format  
      - API Credentials: Requires ScrapeGraphAI API key (needs to be set up in credentials)  
    - Expressions/Variables: Static URL and prompt text; no dynamic expressions used  
    - Inputs: Trigger from Automated Schedule Trigger  
    - Outputs: Raw scraping JSON result passed to Data Formatting node  
    - Version: 1  
    - Edge Cases:  
      - API key missing or invalid causing authentication failure  
      - Amazon site structure changes impacting scraping accuracy  
      - Network timeouts or rate limits from ScrapeGraphAI or Amazon  
      - Response parsing errors if AI output format deviates from schema  
    - Notes: Sticky Note1 explains usage, configuration, and limitations  

#### 1.3 Data Formatting

- **Overview:**  
Processes the raw JSON response from ScrapeGraphAI, extracting and mapping product fields into a clean structure compatible with Google Sheets.

- **Nodes Involved:**  
  - Data Formatting and Processing  
  - Sticky Note2 (documentation)

- **Node Details:**  

  - **Data Formatting and Processing**  
    - Type: Code (JavaScript) Node  
    - Role: Transforms scraped data to standardized format  
    - Configuration:  
      - JavaScript code extracts `result.products` array from input JSON  
      - Maps each product to an object with keys: title, url, category  
      - Returns an array of objects for downstream nodes  
    - Expressions/Variables: Uses `$input.all()[0].json` to access input data  
    - Inputs: Output from AI-Powered Amazon Product Scraper  
    - Outputs: Formatted product array JSON passed to Google Sheets node  
    - Version: 2  
    - Edge Cases:  
      - Input data missing or malformed causing code errors  
      - Absence of `products` array in response  
      - Unexpected data types or null fields in products  
    - Notes: Sticky Note2 details the processing steps and customization options  

#### 1.4 Data Storage

- **Overview:**  
Stores the cleaned product data into a Google Sheets document, appending new rows for each product to maintain historical data.

- **Nodes Involved:**  
  - Google Sheets Data Storage  
  - Sticky Note4 (named "Sticky Note" in original)

- **Node Details:**  

  - **Google Sheets Data Storage**  
    - Type: Google Sheets Node  
    - Role: Appends formatted product data to a specified Google Sheets worksheet  
    - Configuration:  
      - Operation: Append rows  
      - Sheet Name: "Sheet1"  
      - Column Mapping: Automatically maps title, url, category fields to columns  
      - Document ID: URL of target Google Sheets document (to be configured)  
      - Credentials: Google Sheets OAuth2 required  
    - Expressions/Variables: Document URL and sheet name set statically (but can be dynamic)  
    - Inputs: Receives formatted data from Data Formatting node  
    - Outputs: None (final step)  
    - Version: 4.5  
    - Edge Cases:  
      - Invalid or missing Google Sheets credentials causing auth errors  
      - Incorrect document ID or sheet name leading to write failures  
      - API rate limits or connectivity issues  
      - Data schema mismatch causing append errors  
    - Notes: Sticky Note describes usage, configuration, and data management best practices  

---

### 3. Summary Table

| Node Name                      | Node Type                     | Functional Role                          | Input Node(s)                | Output Node(s)                | Sticky Note                                                                                                                                        |
|-------------------------------|-------------------------------|----------------------------------------|-----------------------------|------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------|
| Automated Schedule Trigger     | Schedule Trigger               | Initiates workflow periodically        | None                        | AI-Powered Amazon Product Scraper | Step 1: Automated Schedule Trigger ⏱️ - describes scheduling best practices and configuration options                                               |
| AI-Powered Amazon Product Scraper | ScrapeGraphAI Node            | Scrapes Amazon product data via AI     | Automated Schedule Trigger  | Data Formatting and Processing | Step 2: AI-Powered Amazon Product Scraper 🤖 - explains AI scraping setup, configuration, and usage notes                                          |
| Data Formatting and Processing | Code (JavaScript)             | Processes and structures scraped data  | AI-Powered Amazon Product Scraper | Google Sheets Data Storage    | Step 3: Data Formatting and Processing 🧱 - details data transformation and customization options                                                  |
| Google Sheets Data Storage     | Google Sheets                 | Stores processed data into Google Sheets | Data Formatting and Processing | None                         | Step 4: Google Sheets Data Storage 📊 - outlines configuration, data management, and usage of Google Sheets                                         |
| Sticky Note1                   | Sticky Note                   | Documentation for AI-Powered Scraper   | None                        | None                         | Step 2: AI-Powered Amazon Product Scraper 🤖                                                                                                      |
| Sticky Note2                   | Sticky Note                   | Documentation for Data Formatting      | None                        | None                         | Step 3: Data Formatting and Processing 🧱                                                                                                         |
| Sticky Note3                   | Sticky Note                   | Documentation for Schedule Trigger     | None                        | None                         | Step 1: Automated Schedule Trigger ⏱️                                                                                                             |
| Sticky Note                    | Sticky Note                   | Documentation for Google Sheets Storage | None                        | None                         | Step 4: Google Sheets Data Storage 📊                                                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Automated Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Configure interval to desired frequency (e.g., daily at specific time)  
   - No credentials required  
   - Position: Starting point of the workflow  

2. **Add AI-Powered Amazon Product Scraper node (ScrapeGraphAI):**  
   - Type: ScrapeGraphAI Node (community node, ensure installed and available)  
   - Set the Website URL to: `https://www.amazon.com/s?k=keyboard&crid=2FI5OSSH1T0Q8&sprefix=keyboar%2Caps%2C191&ref=nb_sb_ss_p13n-pd-dpltr-ranker_ci_hl-bn-left_1_7`  
   - User Prompt: `Extract all the products from this site. Use the following schema for response { "title": "Logitech MX Keys Advanced Wireless Illuminated Keyboard", "url": "https://www.amazon.com/dp/B07S92QBCX", "category": "Electronics" }`  
   - Credentials: Configure ScrapeGraphAI API key in n8n credentials and assign here  
   - Connect “Automated Schedule Trigger” node output to this node input  

3. **Add Code node for Data Formatting and Processing:**  
   - Type: Code (JavaScript)  
   - Paste the following code:
     ```javascript
     // Get the input data
     const inputData = $input.all()[0].json;

     // Extract products array from result
     const products = inputData.result.products;

     // Map each product and return only title, url, category
     return products.map(product => ({
       json: {
         title: product.title,
         url: product.url,
         category: product.category
       }
     }));
     ```
   - Connect output of AI-Powered Amazon Product Scraper node to this Code node input  

4. **Add Google Sheets node for Data Storage:**  
   - Type: Google Sheets  
   - Operation: Append  
   - Sheet Name: Set to "Sheet1" (or your target sheet)  
   - Document ID: Provide the URL or ID of your Google Sheets document  
   - Columns: Configure to auto-map fields `title`, `url`, and `category` from input data  
   - Credentials: Set up Google Sheets OAuth2 credentials and assign here  
   - Connect output of Code node (Data Formatting) to this node input  

5. **Verify all connections:**  
   - Automated Schedule Trigger → AI-Powered Amazon Product Scraper → Data Formatting and Processing → Google Sheets Data Storage  

6. **Save and activate the workflow:**  
   - Ensure API keys and credentials are valid and tested  
   - Run the workflow manually to test initial execution and data storage  
   - Monitor logs for errors or data inconsistencies  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                       | Context or Link                                                               |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------|
| ScrapeGraphAI is a community node requiring self-hosting or API access; ensure you have valid API credentials and understand its usage limitations.              | See Sticky Note1 describing ScrapeGraphAI node details                      |
| When scheduling the workflow, consider Amazon’s peak traffic times and rate limits to avoid scraping issues or IP bans.                                         | See Sticky Note3 on scheduling best practices                               |
| You can customize the JavaScript code node to extract additional product fields or perform data validation before storage.                                      | See Sticky Note2 for code customization ideas                               |
| Google Sheets operation is set to append to keep historical data intact, supporting trend analysis over time.                                                     | See Sticky Note4 on Google Sheets usage                                     |
| For more information on n8n Google Sheets integration, visit: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets/                   | Official n8n documentation link                                             |
| For ScrapeGraphAI or similar AI-powered scraping tools, verify compliance with Amazon terms of service and data privacy laws.                                    | General best practice                                                        |

---

**Disclaimer:** The provided content is exclusively derived from an n8n automated workflow. It complies strictly with content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.

---