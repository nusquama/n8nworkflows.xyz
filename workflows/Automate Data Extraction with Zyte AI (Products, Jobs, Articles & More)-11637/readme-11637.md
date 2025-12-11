Automate Data Extraction with Zyte AI (Products, Jobs, Articles & More)

https://n8nworkflows.xyz/workflows/automate-data-extraction-with-zyte-ai--products--jobs--articles---more--11637


# Automate Data Extraction with Zyte AI (Products, Jobs, Articles & More)

---

## 1. Workflow Overview

This workflow, titled **"Automate Data Extraction with Zyte AI (Products, Jobs, Articles & More)"**, is designed to provide a universal scraping solution leveraging the Zyte AI API. It targets users who want to extract structured data (such as product details, job listings, articles, Google search results, or general web content) from a wide variety of websites without needing to create custom selectors or parsers.

The workflow is logically divided into the following core blocks:

- **1.1 Input Reception and Initialization:** Captures user input (URL, site category, API key, and extraction goal) via a web form and transforms this input into actionable configuration parameters for Zyte's AI API.

- **1.2 Routing Logic:** Routes the workflow execution into one of three main pipelines based on the selected site category and extraction goal:
  - AI Extraction Pipeline: For structured data extraction using Zyte AI schemas.
  - Google Search Pipeline: For scraping Google SERP results.
  - Manual/General Pipeline: For raw data extraction (e.g., raw HTML, network captures, screenshots).

- **1.3 AI Extraction Pipeline:** Handles five scenarios of extraction goals:
  - Single item extraction.
  - List extraction (current page).
  - Detail extraction (items on current page).
  - Crawl list (all pages).
  - Crawl details (all pages + item details).

- **1.4 Manual/General Extraction Pipeline:** Supports raw data extraction use cases such as capturing browser HTML, HTTP response bodies, network API captures, infinite scroll pages, and screenshots.

- **1.5 Data Aggregation and Output Formatting:** Consolidates data collected from multiple pages or items into final outputs suitable for CSV export or further processing.

- **1.6 Utility and Control Nodes:** Includes state initialization, pagination control, URL collection, batch processing, and error handling.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception and Initialization

**Overview:**  
This block collects user inputs through a web form, including the target URL, site category, extraction goal, and Zyte API key. It then generates internal configuration variables to guide the workflow's routing and extraction logic.

**Nodes Involved:**  
- Main form submission  
- Route by Category  
- Zyte Config Generator  
- Product Extraction Goal  
- AI Extraction Goal Form  
- General Extraction Goal Form  

**Node Details:**  

- **Main form submission**  
  - Type: Form Trigger  
  - Role: Entry point; collects user data (URL, site category, API key) via a web form titled "AI Web Scraper".  
  - Configuration: Fields for Target URL, Site Category (dropdown with categories like Online Store, News, Jobs, Google Search Results, General), and Zyte API Key (required).  
  - Inputs: User form submission.  
  - Outputs: JSON with user inputs to Route by Category.  
  - Edge cases: Missing or invalid API key or URL; ensure required fields are enforced.  

- **Route by Category**  
  - Type: Switch  
  - Role: Routes flow based on the "Select Site Category" field from user input.  
  - Configuration: Routes to AI Extraction Goal Form for product, news, or job sites; to HTTP Google Search node for Google Search Results; to General Extraction Goal Form for General/Other Website.  
  - Inputs: JSON from Main form submission.  
  - Outputs: Routes to different sub-pipelines.  
  - Edge cases: Unknown category defaults to General Extraction.  

- **Zyte Config Generator**  
  - Type: Code  
  - Role: Translates user inputs into internal keys for extraction goal, target schema, and navigation schema to be used with Zyte API.  
  - Configuration: Uses mapping tables for category and goal, applies logic to set target and navigation schemas (e.g., productList, jobPostingNavigation).  
  - Inputs: Data from AI Extraction Goal Form.  
  - Outputs: JSON with `url`, `extraction_goal`, `target_schema`, `navigation_schema`, and `raw_goal`.  
  - Edge cases: Unmapped inputs default to product and single extraction mode.  
  - Notes: Essential for dynamic Zyte API calls.  

- **Product Extraction Goal**  
  - Type: Switch  
  - Role: Routes based on the extracted `extraction_goal` (e.g., single, list, details_current, crawl_list, crawl_all).  
  - Inputs: Output of Zyte Config Generator.  
  - Outputs: Branches to appropriate HTTP request nodes or initialization nodes.  

- **AI Extraction Goal Form**  
  - Type: Form  
  - Role: Secondary form to select extraction goal for AI extraction pipeline, hidden fields pass URL and Site Type.  
  - Outputs: Data to Zyte Config Generator.  

- **General Extraction Goal Form**  
  - Type: Form  
  - Role: Secondary form for raw extraction goals (raw HTML, response body, network capture, infinite scroll, screenshot).  
  - Outputs: Data to General Extract Goal switch.  

---

### 2.2 AI Extraction Pipeline

**Overview:**  
This block implements Zyte AI-powered extraction scenarios. It manages different scraping goals, handling pagination, item loops, and data collection with batching.

**Nodes Involved:**  
- HTTP Node: [Single Item] Get Details  
- HTTP Node: [List] Get Current Page  
- HTTP Node: [Current Page] Get Item URLs  
- [Current Page] Split Items  
- [Current Page] Item Loop  
- HTTP Node: [Current Page] Get Item Details  
- Format Output [ Single || List ]  
- [List-All] Init State  
- [List-All] Merge Pages  
- HTTP Node: [List-All] Get Page URLs  
- [List-All] Page Controller  
- [List-All] Check Next Page  
- [List-All] Get Item List  
- [List-All] List Accumulator  
- [List-All] Set Next URL  
- [List-All] Final Output  
- [Details-All] Init State  
- [Details-All] Merge Pages  
- HTTP Node: [Details-All] Crawler (Phase 1)  
- [Details-All] URL Collector  
- [Details-All] More Pages?  
- [Details-All] Set Next URL  
- [Details-All] Unpack List (Phase 2)  
- [Details-All] Batch Processor  
- HTTP Node: [Details-All] Get Details  
- [Details-All] Accumulator  
- [Details-All] Final Output  
- Extracted AI Output  

**Node Details:**  

- **HTTP Node: [Single Item] Get Details**  
  - Type: HTTP Request  
  - Role: Fetches detailed data for a single item URL using Zyte API with the target schema.  
  - Configuration: POST to Zyte Extract API with URL and target schema. Authorization header with Zyte API key (Base64 encoded).  
  - Input: URL and schemas from Product Extraction Goal.  
  - Output: Structured JSON data of the item.  
  - Edge cases: API key invalid, network errors, invalid URL.  

- **HTTP Node: [List] Get Current Page**  
  - Type: HTTP Request  
  - Role: Retrieves a list of items on the current page using Zyte API.  
  - Configuration: POST with URL and target schema (list).  
  - Output: JSON containing list data.  

- **HTTP Node: [Current Page] Get Item URLs**  
  - Type: HTTP Request  
  - Role: Gets navigation data with item URLs to enable item-level scraping.  
  - Configuration: POST with URL and navigation schema.  

- **[Current Page] Split Items**  
  - Type: SplitOut  
  - Role: Splits array of item URLs from navigation data into separate items for processing.  
  - Configuration: Splits on `productNavigation.items`.  

- **[Current Page] Item Loop**  
  - Type: SplitInBatches  
  - Role: Processes item URLs in batches (batch size 5) to manage load and API usage.  
  - Configuration: Batches items for looping.  

- **HTTP Node: [Current Page] Get Item Details**  
  - Type: HTTP Request  
  - Role: Fetches detailed info for each item URL.  
  - On error: Continues regular output to avoid stopping workflow on single failures.  

- **Format Output [ Single || List ]**  
  - Type: Set  
  - Role: Normalizes output data into a uniform `data` object regardless of extraction scenario.  
  - Logic: Picks first available data from `productNavigation`, `product`, `productList`, `browserHtml`, `articleList.articles`, `jobPosting`, or `jobPostingNavigation`.  

- **[List-All] Init State**  
  - Type: Set  
  - Role: Initializes state variables (`url`, `target_schema`, `navigation_schema`) for full list crawling.  

- **[List-All] Merge Pages**  
  - Type: Merge  
  - Role: Merges incoming data streams to consolidate navigation data across pages.  

- **HTTP Node: [List-All] Get Page URLs**  
  - Type: HTTP Request  
  - Role: Extracts page URLs from Zyte API to enable pagination crawling.  

- **[List-All] Page Controller**  
  - Type: Code  
  - Role: Controls pagination loop; tracks visited pages; prevents infinite loops by stopping if next page URL was visited. Outputs current scrape URL, next loop URL, and stop flag.  

- **[List-All] Check Next Page**  
  - Type: If  
  - Role: Checks if pagination should continue or stop based on stop flag.  

- **[List-All] Get Item List**  
  - Type: HTTP Request  
  - Role: Retrieves the list of items from the current page URL.  

- **[List-All] List Accumulator**  
  - Type: Code  
  - Role: Aggregates item lists across pages into static memory (`backpack`). Handles multiple Zyte schema types dynamically (products, articles, jobs, serp).  

- **[List-All] Set Next URL**  
  - Type: Set  
  - Role: Updates URL for next pagination iteration based on Page Controller output.  

- **[List-All] Final Output**  
  - Type: Code  
  - Role: Cleans and formats accumulated list data for export. Filters nulls/invalid data.  

- **[Details-All] Init State**  
  - Type: Set  
  - Role: Initializes state for full detail crawling including pagination.  

- **[Details-All] Merge Pages**  
  - Type: Merge  
  - Role: Combines navigation data for detail crawling.  

- **HTTP Node: [Details-All] Crawler (Phase 1)**  
  - Type: HTTP Request  
  - Role: Retrieves navigation data (item URLs and next page) to drive crawling.  

- **[Details-All] URL Collector**  
  - Type: Code  
  - Role: Collects all item URLs from navigation data into static memory; tracks visited pages; decides whether to continue or stop crawling.  

- **[Details-All] More Pages?**  
  - Type: If  
  - Role: Branches based on whether more pages remain to crawl.  

- **[Details-All] Set Next URL**  
  - Type: Set  
  - Role: Sets URL for next pagination iteration.  

- **[Details-All] Unpack List (Phase 2)**  
  - Type: Code  
  - Role: Outputs collected URLs for batch processing; clears static data for next run.  

- **[Details-All] Batch Processor**  
  - Type: SplitInBatches  
  - Role: Processes item URLs in batches (batch size 100) to fetch details. On error: continues output.  

- **HTTP Node: [Details-All] Get Details**  
  - Type: HTTP Request  
  - Role: Fetches detailed data for each item URL using Zyte API.  

- **[Details-All] Accumulator**  
  - Type: Code  
  - Role: Aggregates detailed item data into static storage; handles various Zyte schemas; records errors with fallback data.  

- **[Details-All] Final Output**  
  - Type: Code  
  - Role: Retrieves aggregated detailed items; cleans and formats for export.  

- **Extracted AI Output**  
  - Type: ConvertToFile  
  - Role: Converts final JSON output into file format for CSV export or downstream usage.  

**Edge Cases & Failures:**  
- Zyte API key invalid or expired.  
- API rate limiting or network timeouts.  
- Unexpected or missing schema fields in Zyte response.  
- Infinite pagination loops prevented by tracking visited pages.  
- Batch processing continues on error but may lose some data.  

---

### 2.3 Google Search Pipeline

**Overview:**  
Handles extraction of Google Search Result Pages (SERP) using Zyte's 'serp' schema.

**Nodes Involved:**  
- HTTP Google Search: Serp  
- serp response  

**Node Details:**  

- **HTTP Google Search: Serp**  
  - Type: HTTP Request  
  - Role: Sends POST request to Zyte API with `serp=true` to fetch Google search results for the input URL.  
  - Inputs: User form input URL and Zyte API key.  
  - Outputs: SERP JSON data.  

- **serp response**  
  - Type: Set  
  - Role: Wraps the SERP data into `data` property for uniform handling and downstream export.  

**Edge Cases:**  
- Google blocks or CAPTCHA may cause API errors.  
- Invalid or malformed query URLs.  

---

### 2.4 Manual/General Extraction Pipeline

**Overview:**  
Supports raw data extraction scenarios where no AI parsing is used. Provides outputs like full browser-rendered HTML, raw HTTP response body, captured network API calls, infinite scrolling page HTML, and page screenshots for custom parsing.

**Nodes Involved:**  
- General Extract Goal  
- HTTP BrowserHtml  
- HTTP Response Body  
- HTTP Node: Capture Network API  
- HTTP Node: Infinite Scroll  
- HTTP Node: Capture Page Screenshot  
- Custom Output  
- Convert to File (Image)  

**Node Details:**  

- **General Extract Goal**  
  - Type: Switch  
  - Role: Routes based on user-selected extraction goal for manual mode.  

- **HTTP BrowserHtml**  
  - Type: HTTP Request  
  - Role: Fetches browser-rendered HTML from Zyte API.  

- **HTTP Response Body**  
  - Type: HTTP Request  
  - Role: Fetches raw HTTP response body (Base64 encoded).  

- **HTTP Node: Capture Network API**  
  - Type: HTTP Request  
  - Role: Captures network API calls filtered by URL containing `/api/` with response bodies.  

- **HTTP Node: Infinite Scroll**  
  - Type: HTTP Request  
  - Role: Loads page with infinite scroll action to bottom, then fetches rendered HTML.  

- **HTTP Node: Capture Page Screenshot**  
  - Type: HTTP Request  
  - Role: Captures a visual screenshot of the page.  

- **Custom Output**  
  - Type: Set  
  - Role: Consolidates output data from manual extraction nodes into a single `data` field.  

- **Convert to File (Image)**  
  - Type: ConvertToFile  
  - Role: Converts screenshot Base64 to a binary file for download or further processing.  

**Edge Cases:**  
- Large pages may time out.  
- Network captures may miss data if filters are incorrect.  
- Screenshots may fail on certain domains or require more API credits.  

---

## 3. Summary Table

| Node Name                          | Node Type           | Functional Role                                    | Input Node(s)                       | Output Node(s)                          | Sticky Note                                                                                                  |
|-----------------------------------|---------------------|---------------------------------------------------|-----------------------------------|----------------------------------------|-------------------------------------------------------------------------------------------------------------|
| Main form submission              | Form Trigger        | Entry point; collects user inputs                  | -                                 | Route by Category                      | ## AI Web Scraper<br>Workflow using Zyte API for universal scraping.<br>Get free API key: https://www.zyte.com/?utm_campaign=Discord_n8n_tpl&utm_activity=Community&utm_medium=social&utm_source=Discord |
| Route by Category                 | Switch              | Routes based on Site Category                       | Main form submission              | AI Extraction Goal Form, HTTP Google Search: Serp, General Extraction Goal Form | ## 🕹️ Control Center<br>Routes to AI, Google, or Manual pipelines.                                         |
| AI Extraction Goal Form           | Form                | Select extraction goal for AI pipeline              | Route by Category                 | Zyte Config Generator                  |                                                                                                             |
| Zyte Config Generator             | Code                | Maps form inputs to Zyte API schemas                 | AI Extraction Goal Form           | Product Extraction Goal                |                                                                                                             |
| Product Extraction Goal           | Switch              | Routes based on extraction goal                      | Zyte Config Generator             | HTTP Nodes and Init States             |                                                                                                             |
| HTTP Node: [Single Item] Get Details | HTTP Request      | Fetch single item details                            | Product Extraction Goal           | Format Output [ Single || List ]       |                                                                                                             |
| HTTP Node: [List] Get Current Page | HTTP Request       | Fetch list of items on current page                  | Product Extraction Goal           | Format Output [ Single || List ]       |                                                                                                             |
| HTTP Node: [Current Page] Get Item URLs | HTTP Request    | Fetch navigation schema with item URLs               | Product Extraction Goal           | [Current Page] Split Items             |                                                                                                             |
| [Current Page] Split Items        | SplitOut            | Splits item URLs array into separate items           | HTTP Node: [Current Page] Get Item URLs | [Current Page] Item Loop          |                                                                                                             |
| [Current Page] Item Loop          | SplitInBatches      | Loops over items in batches (size 5)                  | [Current Page] Split Items        | HTTP Node: [Current Page] Get Item Details, Extracted AI Output |                                                                                                             |
| HTTP Node: [Current Page] Get Item Details | HTTP Request   | Fetch details for each item URL                        | [Current Page] Item Loop          | [Current Page] Item Loop, Extracted AI Output |                                                                                                             |
| Format Output [ Single || List ]  | Set                 | Normalizes output data into a single `data` object   | HTTP Node: [Single Item] Get Details, HTTP Node: [List] Get Current Page | Extracted AI Output | ## AI Output: Aggregation & CSV Export                                                                       |
| Extracted AI Output               | ConvertToFile       | Converts JSON output to file for export               | Format Output [ Single || List ]  | -                                      |                                                                                                             |
| [List-All] Init State            | Set                 | Initializes crawl state variables                      | Product Extraction Goal           | [List-All] Merge Pages                 |                                                                                                             |
| [List-All] Merge Pages           | Merge               | Merges navigation data streams                         | [List-All] Init State, [List-All] Set Next URL | HTTP Node: [List-All] Get Page URLs |                                                                                                             |
| HTTP Node: [List-All] Get Page URLs | HTTP Request      | Fetch page URLs for pagination                          | [List-All] Merge Pages            | [List-All] Page Controller             |                                                                                                             |
| [List-All] Page Controller       | Code                | Controls pagination loop; prevents infinite loops     | HTTP Node: [List-All] Get Page URLs | [List-All] Check Next Page           |                                                                                                             |
| [List-All] Check Next Page       | If                  | Checks if pagination should continue                   | [List-All] Page Controller        | [List-All] Final Output, [List-All] Get Item List |                                                                                                             |
| [List-All] Get Item List         | HTTP Request        | Extracts list of items for current page                | [List-All] Check Next Page        | [List-All] List Accumulator            |                                                                                                             |
| [List-All] List Accumulator      | Code                | Aggregates list items from multiple pages              | [List-All] Get Item List          | [List-All] Set Next URL                |                                                                                                             |
| [List-All] Set Next URL          | Set                 | Updates URL for next page crawl                         | [List-All] List Accumulator       | [List-All] Merge Pages                 |                                                                                                             |
| [List-All] Final Output          | Code                | Cleans and formats aggregated list data                | [List-All] Check Next Page        | Extracted AI Output                    |                                                                                                             |
| [Details-All] Init State         | Set                 | Initializes state for detailed crawling                 | Product Extraction Goal           | [Details-All] Merge Pages              |                                                                                                             |
| [Details-All] Merge Pages        | Merge               | Merges navigation data for detail crawling             | [Details-All] Init State, [Details-All] Set Next URL | HTTP Node: [Details-All] Crawler (Phase 1) |                                                                                                             |
| HTTP Node: [Details-All] Crawler (Phase 1) | HTTP Request  | Fetches navigation data to discover item URLs          | [Details-All] Merge Pages         | [Details-All] URL Collector            |                                                                                                             |
| [Details-All] URL Collector      | Code                | Collects item URLs; tracks visited pages; decides to continue | HTTP Node: [Details-All] Crawler (Phase 1) | [Details-All] More Pages?            |                                                                                                             |
| [Details-All] More Pages?        | If                  | Checks if more pages remain to crawl                    | [Details-All] URL Collector       | [Details-All] Unpack List, [Details-All] Set Next URL |                                                                                                             |
| [Details-All] Set Next URL       | Set                 | Sets next page URL for crawling                         | [Details-All] More Pages?         | [Details-All] Merge Pages              |                                                                                                             |
| [Details-All] Unpack List (Phase 2) | Code             | Outputs collected URLs for batch detail fetching        | [Details-All] More Pages?         | [Details-All] Batch Processor          |                                                                                                             |
| [Details-All] Batch Processor    | SplitInBatches      | Processes item URLs in batches (size 100)               | [Details-All] Unpack List (Phase 2) | HTTP Node: [Details-All] Get Details, [Details-All] Final Output |                                                                                                             |
| HTTP Node: [Details-All] Get Details | HTTP Request     | Gets detailed data for each item URL                     | [Details-All] Batch Processor     | [Details-All] Accumulator              |                                                                                                             |
| [Details-All] Accumulator        | Code                | Aggregates detailed item data; records errors           | HTTP Node: [Details-All] Get Details | [Details-All] Batch Processor          |                                                                                                             |
| [Details-All] Final Output       | Code                | Retrieves aggregated detailed items for export          | [Details-All] Batch Processor     | Extracted AI Output                    |                                                                                                             |
| HTTP Google Search: Serp         | HTTP Request        | Fetches Google SERP results                              | Route by Category                 | serp response                        | ## 🔍 Non-AI Extraction :: Google Search<br>Can modify domain for different searches.                       |
| serp response                   | Set                 | Wraps SERP data into `data` property                     | HTTP Google Search: Serp          | Extracted AI Output                    | ## Google Search Result :: serp Response                                                                  |
| General Extract Goal             | Switch              | Routes manual extraction goals                           | General Extraction Goal Form      | HTTP BrowserHtml, HTTP Response Body, HTTP Node: Capture Network API, HTTP Node: Infinite Scroll, HTTP Node: Capture Page Screenshot | ## 🛠️ Manual Mode (Raw Data)<br>No AI parsing; custom parsing needed after output.                          |
| General Extraction Goal Form     | Form                | User selects manual extraction goal                      | Route by Category                 | General Extract Goal                  |                                                                                                             |
| HTTP BrowserHtml                 | HTTP Request        | Fetches browser rendered HTML                            | General Extract Goal              | Custom Output                        |                                                                                                             |
| HTTP Response Body              | HTTP Request        | Fetches raw HTTP response body                           | General Extract Goal              | Custom Output                        |                                                                                                             |
| HTTP Node: Capture Network API   | HTTP Request        | Captures network API calls filtered by URL              | General Extract Goal              | Custom Output                        |                                                                                                             |
| HTTP Node: Infinite Scroll       | HTTP Request        | Loads page with infinite scroll and fetches HTML        | General Extract Goal              | Custom Output                        |                                                                                                             |
| HTTP Node: Capture Page Screenshot | HTTP Request     | Captures screenshot of the page                          | General Extract Goal              | Convert to File ( Image )             |                                                                                                             |
| Custom Output                   | Set                 | Aggregates manual extraction data into `data` field     | HTTP BrowserHtml, HTTP Response Body, HTTP Node: Capture Network API, HTTP Node: Infinite Scroll, HTTP Node: Capture Page Screenshot | - | ## General Output: Raw Export<br>Custom parsing needed.                                                    |
| Convert to File ( Image )        | ConvertToFile       | Converts screenshot Base64 output to binary file         | HTTP Node: Capture Page Screenshot | -                                  | ## Image Output: Save as JPEG or PNG                                                                       |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node** named "Main form submission":
   - Configure form fields:
     - Target URL (required, text input, placeholder example: https://books.toscrape.com/)
     - Select Site Category (required, dropdown with options: Online Store / Product (E-commerce), News, Blog, Forum or Article Site, Job Board / Career Site, Google Search Results, General / Other Website)
     - Zyte API Key (required, text input)
   - Add HTML field with link to Zyte API key registration.
   - Set webhook ID (auto-generated or manual).

2. **Add a Switch Node** named "Route by Category":
   - Use expression to route based on `$json["Select Site Category"]`.
   - Define rules to route:
     - To "AI Extraction Goal Form" for categories: Online Store, News, Job Board.
     - To "HTTP Google Search: Serp" for Google Search Results.
     - To "General Extraction Goal Form" for General / Other Website.

3. **Create the "AI Extraction Goal Form" Form Node**:
   - Hidden fields for URL and Site Type (set via expressions from previous node).
   - Dropdown for extraction goals (Single item, List this page, Details this page, List all pages, Details all pages).
   - Connect output to "Zyte Config Generator".

4. **Add a Code Node "Zyte Config Generator"**:
   - Implement mapping from Site Type and extraction goal dropdown to Zyte API keys:
     - Map categories to base keys (product, article, jobPosting, serp, browserHtml).
     - Map goals to extraction goals (single, list, crawl_list, details_current, crawl_all).
     - Determine target_schema and navigation_schema accordingly.
   - Output keys: url, extraction_goal, target_schema, navigation_schema, raw_goal.

5. **Create a Switch Node "Product Extraction Goal"**:
   - Route based on `extraction_goal` (single, list, details_current, crawl_list, crawl_all).
   - Connect to appropriate HTTP nodes or init state nodes as per goal.

6. **Build HTTP Request Nodes for Zyte API Calls**:
   - For single item details: "HTTP Node: [Single Item] Get Details"
   - For list current page: "HTTP Node: [List] Get Current Page"
   - For navigation URLs: "HTTP Node: [Current Page] Get Item URLs"
   - For getting page URLs during list crawl: "HTTP Node: [List-All] Get Page URLs"
   - For details crawl phase 1: "HTTP Node: [Details-All] Crawler (Phase 1)"
   - For details crawl phase 2: "HTTP Node: [Details-All] Get Details"
   - All HTTP nodes:
     - Use POST method to https://api.zyte.com/v1/extract
     - Set body parameters with "url" and the corresponding Zyte schema set to true.
     - Set header with Authorization: Basic (base64 encoded Zyte API key + ":").
     - Enable error retry where appropriate.

7. **Add State Initialization Nodes (Set nodes)**:
   - "[List-All] Init State": Initialize `url`, `target_schema`, `navigation_schema`.
   - "[Details-All] Init State": Same for details crawl.

8. **Add Pagination and Loop Control Nodes**:
   - "[List-All] Merge Pages": Merge multiple inputs.
   - "[List-All] Page Controller": Code node to track visited pages, prevent infinite loops, output next URL or stop flag.
   - "[List-All] Check Next Page": If node to check stop flag.
   - "[List-All] Set Next URL": Set node to update URL for next loop.
   - "[Details-All] Merge Pages": Merge node for details.
   - "[Details-All] URL Collector": Code node to collect URLs and track visited pages.
   - "[Details-All] More Pages?": If node.
   - "[Details-All] Set Next URL": Set node.

9. **Add Loop & Batch Processing Nodes**:
   - "[Current Page] Split Items": SplitOut node to separate item URLs.
   - "[Current Page] Item Loop": SplitInBatches with batch size 5 for item detail requests.
   - "[Details-All] Unpack List (Phase 2)": Code node to output collected URLs.
   - "[Details-All] Batch Processor": SplitInBatches with batch size 100 for details crawling.

10. **Add Accumulator and Output Formatting Nodes**:
    - "[List-All] List Accumulator": Code node to accumulate list items.
    - "[Details-All] Accumulator": Code node to accumulate detailed items, handle errors.
    - "[List-All] Final Output": Code node to filter and output final list.
    - "[Details-All] Final Output": Code node to filter and output final details.
    - "Format Output [ Single || List ]": Set node to normalize data output.
    - "Extracted AI Output": ConvertToFile node for CSV export.

11. **Build Google Search Pipeline**:
    - HTTP Request node "HTTP Google Search: Serp" set with `serp=true`.
    - Set node "serp response" to assign response to `data`.
    - Connect to "Extracted AI Output".

12. **Build Manual/General Extraction Pipeline**:
    - Form node "General Extraction Goal Form" for manual goals.
    - Switch node "General Extract Goal" routes to:
      - HTTP BrowserHtml (fetch browser rendered HTML)
      - HTTP Response Body (fetch raw HTTP body)
      - HTTP Node: Capture Network API (filter URLs containing "/api/")
      - HTTP Node: Infinite Scroll (simulate scroll to bottom)
      - HTTP Node: Capture Page Screenshot
    - "Custom Output" Set node to consolidate outputs.
    - "Convert to File ( Image )" node to convert screenshot Base64 to file.

13. **Connect all pipelines back to a final output node** ("Extracted AI Output") for downstream export.

14. **Add Sticky Notes** throughout for documentation and user guidance.

15. **Credential Setup**:
    - Configure Zyte API credentials to be used in all HTTP Request nodes for Authorization header.
    - No other external credentials required.

---

## 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                              | Context or Link                                                                                                                             |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| This workflow turns n8n into a universal scraping machine using the [Zyte API](https://www.zyte.com/?utm_campaign=Discord_n8n_tpl&utm_activity=Community&utm_medium=social&utm_source=Discord). It can crawl and extract structured data from almost any website without custom selectors. | Main sticky note at workflow start                                                                                                         |
| You can get a free Zyte API key here: https://www.zyte.com/?utm_campaign=Discord_n8n_tpl&utm_activity=Community&utm_medium=social&utm_source=Discord                                                                                                                       | Mentioned in form and sticky note                                                                                                          |
| The workflow features advanced pagination handling with state memory to prevent infinite loops. It automatically detects Zyte schema types and dynamically adapts to different site structures (products, articles, jobs, SERP).                                           | Code nodes [List-All] Page Controller and [Details-All] URL Collector                                                                        |
| Manual mode supports raw data extraction modes including browser HTML, HTTP response body, network captures, infinite scrolling, and screenshots — enabling custom parsing outside of AI extraction.                                                                       | Sticky note titled "🛠️ Manual Mode (Raw Data)"                                                                                             |
| AI Extraction pipeline handles 5 main scenarios: single item, list current page, details current page, crawl list (all pages), and crawl details (all pages + item details).                                                                                                | Sticky note titled "🤖 AI Extraction Pipeline (5 Scenarios)"                                                                                |
| The workflow utilizes batch processing (sizes 5 or 100) to efficiently handle multiple API calls while respecting rate limits and performance.                                                                                                                                 | Batch processors in [Current Page] Item Loop and [Details-All] Batch Processor                                                             |
| Google Search scraping uses Zyte's 'serp' schema. You can customize the domain or URL in the HTTP Request node to target different search result pages.                                                                                                                    | Sticky note near Google Search nodes                                                                                                       |

---

**Disclaimer:**  
The content described above originates exclusively from an automated workflow created with n8n, a workflow automation tool. All data handled is legal and public. No illegal, offensive, or protected content is involved. The workflow respects policies in force.

---