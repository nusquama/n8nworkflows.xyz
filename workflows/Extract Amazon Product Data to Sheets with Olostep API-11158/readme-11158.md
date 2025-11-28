Extract Amazon Product Data to Sheets with Olostep API

https://n8nworkflows.xyz/workflows/extract-amazon-product-data-to-sheets-with-olostep-api-11158


# Extract Amazon Product Data to Sheets with Olostep API

### 1. Workflow Overview

This workflow automates the extraction of Amazon product data using the Olostep API and stores the results in a Google Sheet or n8n Data Table. It is designed for use cases where users want to scrape multiple pages of Amazon search results for product titles and URLs without manual browsing or complex browser automation.

The logical flow is grouped into these blocks:

- **1.1 Input Reception**: A form collects a user-defined search query.
- **1.2 Pagination Setup**: Generates a list of Amazon search result pages (1 to 10) to scrape.
- **1.3 Amazon Scraping with Olostep API**: Sends HTTP POST requests to Olostep API to scrape product data from each Amazon page.
- **1.4 Parsing and Splitting Results**: Processes the JSON response into individual product items.
- **1.5 URL Normalization and Validation**: Converts relative URLs to full URLs and filters out invalid product links.
- **1.6 Data Insertion**: Inserts validated product data into a Google Sheet or Data Table.
- **1.7 Looping and Rate Control**: Manages iteration over pages and products with wait steps to respect rate limits.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception

**Overview:**  
This block captures the user's search query through a web form and prepares initial data for pagination.

**Nodes Involved:**  
- On form submission  
- Edit Fields (counter assignment)  
- Sticky Note (Instructional)

**Node Details:**  

- **On form submission**  
  - *Type & Role:* Form Trigger — starts the workflow when a user submits the search query form.  
  - *Configuration:* Single text field labeled "search query" with a placeholder example "wireless bluetooth headphones".  
  - *Expressions:* Utilizes `$('On form submission').item.json['search query']` to reference user input downstream.  
  - *Connections:* Outputs to Edit Fields node.  
  - *Edge Cases:* Missing input or empty queries could cause empty results downstream. No explicit validation here.  
  - *Sticky Note:* Explains the search query input with example.

- **Edit Fields (counter assignment)**  
  - *Type & Role:* Set node — defines a fixed list of page numbers from 1 to 10 for pagination.  
  - *Configuration:* Creates an array named `counter` with values `[1,2,3,4,5,6,7,8,9,10]`.  
  - *Connections:* Outputs to Split Out2 node (splitting page numbers for iteration).  
  - *Edge Cases:* Static pagination; not dynamic based on results or max pages.  

---

#### 2.2 Pagination Setup

**Overview:**  
Splits the list of page numbers into individual items for sequential processing.

**Nodes Involved:**  
- Split Out2  
- Loop Over Items  

**Node Details:**  

- **Split Out2**  
  - *Type & Role:* Split Out — separates array of page numbers into individual messages.  
  - *Configuration:* Field to split out is `counter`.  
  - *Connections:* Outputs to Loop Over Items.  
  - *Edge Cases:* If `counter` is empty or incorrectly formatted, no pages will be processed.  

- **Loop Over Items**  
  - *Type & Role:* Split In Batches — controls iteration over paginated pages and manages batch size.  
  - *Configuration:* Default settings; acts as a loop controller for page requests.  
  - *Connections:* On first output branch, loops to `scrape amazon products` node; on second output branch, terminates loop.  
  - *Edge Cases:* Batch size not explicitly set; large batches may hit API rate limits.  
  - *Sticky Note:* Pagination explanation is provided nearby.

---

#### 2.3 Amazon Scraping with Olostep API

**Overview:**  
For each page number, this block sends a scraping request to the Olostep API with the user’s search query embedded in the URL. It requests product titles and URLs via LLM extraction.

**Nodes Involved:**  
- scrape amazon products  
- Sticky Note (Olostep Amazon Scrape explanation)  

**Node Details:**  

- **scrape amazon products**  
  - *Type & Role:* HTTP Request — sends POST requests to `https://api.olostep.com/v1/scrapes`.  
  - *Configuration:*  
    - Method: POST  
    - JSON body includes:  
      - `url_to_scrape`: Amazon search URL constructed dynamically from the search query and current page number (`https://www.amazon.com/s?k={{search query}}&page={{counter}}`).  
      - `formats`: ["json"]  
      - `wait_before_scraping`: 2 seconds delay before scraping (to avoid rate limits).  
      - `remove_css_selectors`: "default" (standard content cleaning).  
      - `llm_extract`: JSON schema requesting an array of product objects with `title` and `url`.  
      - `screen_size`: Desktop resolution 1920x1080.  
    - Headers: Authorization Bearer token for Olostep API.  
  - *Expressions:* Uses templated expressions to insert query and page number dynamically.  
  - *Error Handling:* Set to "continueRegularOutput" to avoid halting on errors.  
  - *Connections:* Outputs to parsedInfo node.  
  - *Edge Cases:*  
    - API rate limits or 504 Gateway Timeout errors (see warning sticky note).  
    - Authorization errors if token invalid.  
    - LLM extraction failures may produce malformed JSON.  
  - *Sticky Note:* Describes the purpose and data extracted by this node.

---

#### 2.4 Parsing and Splitting Results

**Overview:**  
Processes the JSON response from Olostep, cleans escaped characters, and splits the list into individual product items.

**Nodes Involved:**  
- parsedInfo  
- Split Out  
- Sticky Note (Parse & Split explanation)  

**Node Details:**  

- **parsedInfo**  
  - *Type & Role:* Set — cleans the JSON string by removing backslashes (`\\`) to ensure proper JSON formatting.  
  - *Configuration:* Assigns a new field `parsedJson` by applying regex replace on the `result.json_content` field.  
  - *Connections:* Outputs to Split Out node.  
  - *Edge Cases:* If response is empty or malformed, parsing will fail or produce empty output.

- **Split Out**  
  - *Type & Role:* Split Out — separates the array in `parsedJson` into individual messages for each product.  
  - *Configuration:* Splits on field `parsedJson`.  
  - *Connections:* Outputs to If node for URL validation.  
  - *Edge Cases:* If `parsedJson` is not a valid array, splitting fails or no output is generated.

---

#### 2.5 URL Normalization and Validation

**Overview:**  
This block checks that product URLs are valid and converts relative URLs to full Amazon URLs.

**Nodes Involved:**  
- If  
- Edit Fields1  

**Node Details:**  

- **If**  
  - *Type & Role:* Conditional check — ensures the product URL starts with `"https://www.amazon.com"`.  
  - *Configuration:*  
    - Condition: `$json.url` does **not** start with "https://www.amazon.com" (negated check).  
    - Only products with valid URLs pass forward.  
  - *Connections:*  
    - True branch (invalid URLs): terminates (no further output).  
    - False branch (valid URLs): proceeds to Edit Fields1 node.  
  - *Edge Cases:* May exclude legitimate URLs if Amazon uses alternative domains or URL structures.

- **Edit Fields1**  
  - *Type & Role:* Set — prepends full Amazon domain to relative URLs and assigns normalized `url` and `title`.  
  - *Configuration:*  
    - `url`: concatenates "https://www.amazon.com" with the product URL (if relative).  
    - `title`: assigned as-is from product data.  
  - *Connections:* Outputs to Insert row node.  
  - *Edge Cases:* If URL is already absolute, concatenation may produce duplicate domains.

---

#### 2.6 Data Insertion

**Overview:**  
Stores the cleaned and validated product data into a Google Sheet or n8n Data Table.

**Nodes Involved:**  
- Insert row  
- Sticky Note (Insert Row explanation)  

**Node Details:**  

- **Insert row**  
  - *Type & Role:* Data Table node — inserts each product as a new row into a specified Data Table (or Google Sheet equivalent).  
  - *Configuration:*  
    - Columns mapped: `title` and `url` from the normalized fields.  
    - Data Table ID is set (specific to the user’s environment).  
  - *Connections:* Outputs to Wait node.  
  - *Edge Cases:*  
    - Insertion failures if Data Table is inaccessible or full.  
    - Data type mismatches prevented by explicit mapping.  
  - *Sticky Note:* Describes purpose: saving each product.

---

#### 2.7 Looping and Rate Control

**Overview:**  
Controls the pacing of item processing and pagination iteration to respect API rate limits.

**Nodes Involved:**  
- Wait  
- Loop Over Items  

**Node Details:**  

- **Wait**  
  - *Type & Role:* Wait — pauses workflow briefly between inserting rows to avoid rate limiting or API overload.  
  - *Configuration:* Default wait time (unspecified, likely minimal).  
  - *Connections:* Outputs back to Loop Over Items to continue processing next page.  
  - *Edge Cases:* If wait is too short, API may throttle requests; too long increases total workflow runtime.  
  - *Sticky Note:* Warning about 504 Gateway Timeout related to LLM extraction in the HTTP request node.

- **Loop Over Items**  
  - As described above, controls the iteration cycle.

---

### 3. Summary Table

| Node Name              | Node Type           | Functional Role                     | Input Node(s)            | Output Node(s)          | Sticky Note                                                                                              |
|------------------------|---------------------|-----------------------------------|--------------------------|-------------------------|---------------------------------------------------------------------------------------------------------|
| On form submission     | Form Trigger        | Receive user search query          |                          | Edit Fields             | ## Enter a search query  Example: “wireless bluetooth headphones”.                                      |
| Edit Fields            | Set                 | Create pagination counter array    | On form submission       | Split Out2              |                                                                                                         |
| Split Out2             | Split Out           | Split pagination array into items  | Edit Fields              | Loop Over Items         | ## Pagination  Iterates through multiple Amazon pages (1–10).                                           |
| Loop Over Items        | Split In Batches    | Loop over each page number          | Split Out2               | scrape amazon products  | ## Pagination  Iterates through multiple Amazon pages (1–10).                                           |
| scrape amazon products | HTTP Request        | Scrape Amazon page via Olostep API | Loop Over Items          | parsedInfo              | ## Olostep Amazon Scrape  Extracts product title + URL.                                                  |
| parsedInfo             | Set                 | Clean JSON response string          | scrape amazon products   | Split Out               | ## Parse & Split  Converts JSON output into individual products.                                        |
| Split Out              | Split Out           | Split parsed JSON into products     | parsedInfo               | If                      | ## Parse & Split  Converts JSON output into individual products.                                        |
| If                     | If                  | Filter valid product URLs           | Split Out                | Edit Fields1, (end)     |                                                                                                         |
| Edit Fields1           | Set                 | Normalize URLs and assign fields    | If                       | Insert row              |                                                                                                         |
| Insert row             | Data Table          | Insert product data into sheet      | Edit Fields1             | Wait                    | ## Insert Row  Saves each product to a Google Sheet / Data Table.                                       |
| Wait                   | Wait                | Rate control between inserts        | Insert row               | Loop Over Items         | ## WARNING If the http request runs through 504 gateway timeout error, that's because the "llm_extract". |
| Sticky Note1           | Sticky Note         | Instructional (Pagination)          |                          |                         | ## Pagination  Iterates through multiple Amazon pages (1–10).                                           |
| Sticky Note2           | Sticky Note         | Instructional (Olostep Scrape)      |                          |                         | ## Olostep Amazon Scrape  Extracts product title + URL.                                                  |
| Sticky Note3           | Sticky Note         | Instructional (Parse & Split)       |                          |                         | ## Parse & Split  Converts JSON output into individual products.                                        |
| Sticky Note5           | Sticky Note         | Instructional (Insert Row)          |                          |                         | ## Insert Row  Saves each product to a Google Sheet / Data Table.                                       |
| Sticky Note6           | Sticky Note         | Overview & setup instructions       |                          |                         | # Olostep Amazon Products Scraper  Explains full workflow steps and usage.                              |
| Sticky Note            | Sticky Note         | Instructional (Form input)           |                          |                         | ## Enter a search query  Example: “wireless bluetooth headphones”.                                      |
| Sticky Note4           | Sticky Note         | Warning about 504 gateway timeout   |                          |                         | ## WARNING If the http request runs through a 504 gateway timeout error, that's because the "llm_extract". |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node** named "On form submission":  
   - Form title: "Olostep Amazon Products Scraper"  
   - Add one text field:  
     - Label: "search query"  
     - Placeholder: "wireless bluetooth headphones"  
   - Save and connect its output to the next node.

2. **Add a Set node** named "Edit Fields":  
   - Create a new field named `counter` of type array.  
   - Assign value: `[1,2,3,4,5,6,7,8,9,10]` (pagination pages).  
   - Connect from "On form submission".

3. **Add a Split Out node** named "Split Out2":  
   - Field to split out: `counter`.  
   - Connect from "Edit Fields".

4. **Add a Split In Batches node** named "Loop Over Items":  
   - Use default batch size (can be adjusted if needed).  
   - Connect from "Split Out2".

5. **Add an HTTP Request node** named "scrape amazon products":  
   - Method: POST  
   - URL: `https://api.olostep.com/v1/scrapes`  
   - Headers: Add Authorization header with value `Bearer <your Olostep API token>` (replace `<your Olostep API token>`).  
   - Body: Raw JSON with the following template, using expressions:  
     ```json
     {
       "url_to_scrape": "https://www.amazon.com/s?k={{ $('On form submission').item.json['search query'] }}&page={{ $json.counter }}",
       "formats": ["json"],
       "wait_before_scraping": 2,
       "remove_css_selectors": "default",
       "llm_extract": {
         "schema": {
           "type": "array",
           "description": "A list of products listed on Amazon.",
           "items": {
             "type": "object",
             "properties": {
               "title": {
                 "type": "string",
                 "description": "The title of the product."
               },
               "url": {
                 "type": "string",
                 "description": "The full url for the product."
               }
             },
             "required": ["title", "url"]
           }
         }
       },
       "screen_size": {
         "screen_type": "desktop",
         "screen_width": 1920,
         "screen_height": 1080
       }
     }
     ```  
   - Enable "Send Body" as JSON.  
   - Error handling set to "Continue on Error" to avoid workflow stop on failure.  
   - Connect from "Loop Over Items".

6. **Add a Set node** named "parsedInfo":  
   - Assign a new field `parsedJson` by cleaning the HTTP response JSON string:  
     - Use expression: `{{ $json.result.json_content.replace(/\\\\/g, '') }}`  
   - Connect from "scrape amazon products".

7. **Add a Split Out node** named "Split Out":  
   - Field to split out: `parsedJson`.  
   - Connect from "parsedInfo".

8. **Add an If node** named "If":  
   - Condition: Check if `$json.url` **does not start with** `"https://www.amazon.com"` (negated condition for invalid URLs).  
   - True branch: no output (filter out).  
   - False branch: continue to next node.  
   - Connect from "Split Out".

9. **Add a Set node** named "Edit Fields1":  
   - Assign and normalize fields:  
     - `url`: `"https://www.amazon.com" + $json.url` (concatenate full URL)  
     - `title`: `$json.title` (keep as is)  
   - Connect from "If" node’s False branch.

10. **Add a Data Table node** named "Insert row":  
    - Select your target Data Table or Google Sheet.  
    - Map columns:  
      - `title` → `title` field  
      - `url` → `url` field  
    - Connect from "Edit Fields1".

11. **Add a Wait node** named "Wait":  
    - Use default wait time or set explicit wait (e.g., 1-2 seconds) to manage rate limits.  
    - Connect from "Insert row".

12. **Connect "Wait" node back to "Loop Over Items"** to continue looping through all pages.

13. **Add sticky notes to document each major block and warnings**, for example:  
    - Input instructions  
    - Pagination info  
    - Olostep scraping details  
    - Parsing explanation  
    - Data insertion purpose  
    - Warning about 504 Gateway Timeout errors related to LLM extraction schema (suggest removing schema if errors occur).

14. **Set workflow execution order to "v1" (default).**

15. **Activate the workflow and test** by submitting a search query via the form.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                        | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow leverages the Olostep API to scrape Amazon product data without browser automation, providing a clean JSON output of product titles and URLs.                                                                      | Olostep API documentation: https://api.olostep.com/                                               |
| If the HTTP Request node returns a 504 Gateway Timeout error, consider removing the `llm_extract` schema and using a separate LLM or information extractor node to parse the data.                                               | Warning sticky note inside workflow                                                                |
| The workflow saves data into a configurable Data Table or Google Sheet, allowing easy integration with other tools or reporting systems.                                                                                        | n8n Data Table documentation: https://docs.n8n.io/nodes/n8n-nodes-base.datatable/                  |
| The workflow’s pagination is statically set to pages 1 through 10; for dynamic pagination, modify the counter array or add logic to detect last page.                                                                           | Sticky note “Pagination” block explanation                                                        |
| To use the Olostep API, you need an API key with appropriate permissions; store this securely in n8n credentials and reference it in the HTTP Request node’s header.                                                              | Credentials management in n8n: https://docs.n8n.io/credentials/                                     |
| The workflow includes instructional sticky notes explaining each block for users deploying or customizing it.                                                                                                                    | Visible in the workflow UI                                                                         |

---

**Disclaimer:** The text provided is extracted exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to current content policies and contains no illegal, offensive, or proprietary elements. All data handled is legal and public.