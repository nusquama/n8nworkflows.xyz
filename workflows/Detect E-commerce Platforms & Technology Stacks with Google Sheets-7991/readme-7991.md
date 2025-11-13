Detect E-commerce Platforms & Technology Stacks with Google Sheets

https://n8nworkflows.xyz/workflows/detect-e-commerce-platforms---technology-stacks-with-google-sheets-7991


# Detect E-commerce Platforms & Technology Stacks with Google Sheets

### 1. Workflow Overview

This workflow is designed to detect e-commerce platforms and technology stacks used by websites listed in a Google Sheet. It reads domain URLs from the sheet, normalizes and processes each URL, fetches the website HTML content, analyzes it for e-commerce platform signatures and related technologies, then writes the enhanced detection results back to the Google Sheet. The workflow is suitable for market researchers, competitive intelligence analysts, or developers seeking to automate technology stack discovery for multiple e-commerce domains.

The logic is organized into these main functional blocks:

- **1.1 Input Reception:** Triggering the workflow manually or on a schedule and fetching domain rows from Google Sheets.
- **1.2 URL Preprocessing:** Normalizing domain URLs to ensure proper protocol and formatting before HTTP requests.
- **1.3 Batch Processing & HTTP Fetch:** Splitting domains into manageable batches, fetching their HTML content with error handling and rate limiting.
- **1.4 Technology Detection & Analysis:** Parsing the fetched HTML content to detect e-commerce platforms, frameworks, payment gateways, and features.
- **1.5 Results Update:** Writing the analyzed results back into the Google Sheet, matching by domain.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** This block initiates the workflow either manually or on a time schedule, then retrieves domain data from a Google Sheet.
- **Nodes Involved:** 
  - `When clicking 'Execute workflow'`
  - `Schedule Trigger`
  - `Get row(s) in sheet1`
- **Node Details:**

1. **When clicking 'Execute workflow'**  
   - Type: Manual Trigger  
   - Role: Allows manual initiation of the workflow.  
   - Configuration: No parameters, triggers on user execution.  
   - Connections: Outputs to `Get row(s) in sheet1`.  
   - Edge cases: None significant; manual trigger depends on user action.

2. **Schedule Trigger**  
   - Type: Schedule Trigger  
   - Role: Triggers workflow automatically every hour.  
   - Configuration: Interval set to 1 hour.  
   - Connections: Outputs to `Get row(s) in sheet1`.  
   - Edge cases: Scheduler downtime or n8n instance offline could delay execution.

3. **Get row(s) in sheet1**  
   - Type: Google Sheets  
   - Role: Reads rows from the configured Google Sheet named "Sheet1" in the document "Technology Finder with Domain".  
   - Configuration: Reads all rows; no filter applied. Uses OAuth2 credentials for Google Sheets access.  
   - Key variables: `documentId = 1Tog_qW5NGJwCgk1t4_IpKC4rMurEx-tXuaapWgVli7w`, `sheetName = gid=0`  
   - Input: Trigger nodes  
   - Output: Data rows passed to `URL Preprocessor`.  
   - Edge cases: Google API limits, auth expiration, empty sheet.

---

#### 2.2 URL Preprocessing

- **Overview:** Normalizes domain URLs to ensure they include a protocol (https://) and removes trailing slashes, storing original and cleaned URLs.
- **Nodes Involved:** 
  - `URL Preprocessor`
- **Node Details:**

1. **URL Preprocessor**  
   - Type: Code (JavaScript)  
   - Role: Cleans and standardizes domain URLs from the sheet data.  
   - Configuration:  
     - Trims whitespace from domain strings.  
     - Adds `https://` if missing protocol.  
     - Removes trailing slashes.  
     - Stores original domain in `OriginalDomain` field.  
   - Expressions: Uses JavaScript code with `$input.all()`.  
   - Input: Rows from Google Sheets.  
   - Output: Cleaned domain data passed to `Split In Batches1`.  
   - Edge cases: Empty or malformed domain values, domains already containing protocols, domains with trailing slashes in various forms.

---

#### 2.3 Batch Processing & HTTP Fetch

- **Overview:** Processes domains in batches of 5 to manage load, performs HTTP requests to fetch website HTML content, applies retry and rate limiting to avoid request throttling.
- **Nodes Involved:**  
  - `Split In Batches1`  
  - `HTTP Request1`  
  - `Rate Limiting Wait1`
- **Node Details:**

1. **Split In Batches1**  
   - Type: SplitInBatches  
   - Role: Divides the domain list into batches of 5 for sequential processing.  
   - Configuration: Batch size set to 5.  
   - Input: Cleaned domain data from `URL Preprocessor`.  
   - Output: Batches sent to `HTTP Request1`.  
   - Edge cases: Very large input lists handled gracefully; batch size can be adjusted.

2. **HTTP Request1**  
   - Type: HTTP Request  
   - Role: Fetches the HTML content of each domain URL.  
   - Configuration:  
     - URL is dynamically set from the `Domain` field.  
     - Timeout of 20 seconds.  
     - Follows up to 5 redirects.  
     - Output response saved as raw text under `html` property.  
     - Retries up to 3 times on failure; continues workflow on error.  
   - Input: Domain batch from `Split In Batches1`.  
   - Output: Response HTML content passed to `Rate Limiting Wait1`.  
   - Edge cases: Network errors, DNS failures, timeouts, unreachable domains, SSL issues, HTTP errors (403, 404, 500).

3. **Rate Limiting Wait1**  
   - Type: Wait  
   - Role: Pauses for 2 seconds between HTTP requests to prevent hitting rate limits.  
   - Configuration: Wait 2 seconds.  
   - Input: HTML content from `HTTP Request1`.  
   - Output: Passes data to `Enhanced Technology Detection1`.  
   - Edge cases: Delay ensures API compliance; if wait node fails, requests may be too rapid.

---

#### 2.4 Technology Detection & Analysis

- **Overview:** Analyzes the fetched HTML content to detect e-commerce platforms, frontend frameworks, payment gateways, and e-commerce features through pattern matching and heuristic checks. Handles error detection and confidence scoring.
- **Nodes Involved:**  
  - `Enhanced Technology Detection1`
- **Node Details:**

1. **Enhanced Technology Detection1**  
   - Type: Code (JavaScript)  
   - Role: Performs complex domain extraction and content analysis to identify technology stacks.  
   - Configuration:  
     - Extracts domain from multiple possible fields for robustness.  
     - Cleans domain strings by removing protocols, www, paths, and query strings.  
     - If no domain found, attempts extraction from HTML meta tags (`og:url`, canonical link).  
     - Checks for error conditions (DNS errors, connection refused, timeouts, SSL errors, HTTP status codes) with detailed remarks.  
     - Uses regex and string matching to detect:  
       - E-commerce platforms (Magento, Shopify, WooCommerce, BigCommerce, Squarespace, Wix, Webflow, PrestaShop, OpenCart, Salesforce Commerce Cloud, NetSuite SuiteCommerce, etc.)  
       - Frontend frameworks (Next.js, Nuxt.js, React, Vue.js, Angular, Gatsby, WordPress)  
       - Payment gateways (Stripe, PayPal, Square, Klarna, Razorpay, Braintree)  
       - Features like cart, catalog, checkout, wishlist, PWA presence.  
     - Computes a confidence score based on detected features.  
     - Generates remarks summarizing detection results or errors.  
     - Returns a structured JSON object with all detected attributes per domain.  
   - Input: HTML content with domain metadata from `Rate Limiting Wait1`.  
   - Output: Structured detection results passed to `Update Enhanced Results1`.  
   - Edge cases: Malformed HTML, missing or minimal content, unexpected error objects, pattern matching failures, JavaScript runtime errors (try-catch included).

---

#### 2.5 Results Update

- **Overview:** Updates the Google Sheet with the detection results matching each domain, writing platform, framework, features, confidence score, and remarks.
- **Nodes Involved:**  
  - `Update Enhanced Results1`
- **Node Details:**

1. **Update Enhanced Results1**  
   - Type: Google Sheets  
   - Role: Updates existing rows in the Google Sheet with enhanced detection data.  
   - Configuration:  
     - Matches rows by `Domain` column.  
     - Writes columns: Platform, Cart, Catalog, Checkout, Wishlist, PWA, Payment Gateway, Framework, Confidence Score, Last Checked, Remarks.  
     - Uses OAuth2 credentials for Google Sheets.  
     - Sheet and document same as input.  
   - Input: Detection results from `Enhanced Technology Detection1`.  
   - Output: Final node, no output connections.  
   - Edge cases: Sheet write conflicts, missing domains in sheet, API errors.

---

### 3. Summary Table

| Node Name                   | Node Type           | Functional Role                      | Input Node(s)                   | Output Node(s)                   | Sticky Note                                                                                   |
|-----------------------------|---------------------|------------------------------------|--------------------------------|---------------------------------|----------------------------------------------------------------------------------------------|
| When clicking 'Execute workflow' | Manual Trigger      | Manual start of workflow            | -                              | Get row(s) in sheet1            |                                                                                              |
| Schedule Trigger            | Schedule Trigger    | Automated hourly start              | -                              | Get row(s) in sheet1            | Schedule a Cron from Schedule Trigger                                                        |
| Get row(s) in sheet1       | Google Sheets       | Read domains from Google Sheet      | When clicking 'Execute workflow', Schedule Trigger | URL Preprocessor               | Create your google sheet and add domain column                                               |
| URL Preprocessor           | Code                | Normalize domain URLs               | Get row(s) in sheet1            | Split In Batches1               |                                                                                              |
| Split In Batches1          | SplitInBatches      | Batch domain list for HTTP requests | URL Preprocessor                | HTTP Request1                  |                                                                                              |
| HTTP Request1              | HTTP Request        | Fetch website HTML content          | Split In Batches1               | Rate Limiting Wait1             |                                                                                              |
| Rate Limiting Wait1        | Wait                | Enforce delay between requests      | HTTP Request1                  | Enhanced Technology Detection1  |                                                                                              |
| Enhanced Technology Detection1 | Code              | Detect platforms & features from HTML | Rate Limiting Wait1             | Update Enhanced Results1        |                                                                                              |
| Update Enhanced Results1   | Google Sheets       | Update Google Sheet with detection results | Enhanced Technology Detection1 | -                              | Create your google sheet, add domain, Platform, etc. column                                  |
| Sticky Note                | Sticky Note         | Documentation note                  | -                              | -                              | Schedule a Cron from Schedule Trigger                                                        |
| Sticky Note1               | Sticky Note         | Documentation note                  | -                              | -                              | Create your google sheet and add domain column                                               |
| Sticky Note2               | Sticky Note         | Documentation note                  | -                              | -                              | Create your google sheet, add domain, Platform, etc. column                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node** named "When clicking 'Execute workflow'". No parameters needed.

2. **Create a Schedule Trigger node** named "Schedule Trigger".  
   - Set interval to every 1 hour.

3. **Create a Google Sheets node** named "Get row(s) in sheet1".  
   - Operation: Read Rows  
   - Document ID: Use your Google Sheet ID (e.g. `1Tog_qW5NGJwCgk1t4_IpKC4rMurEx-tXuaapWgVli7w`)  
   - Sheet Name: Use the sheet containing domains, e.g. "Sheet1" or gid=0  
   - Credentials: Configure OAuth2 Google Sheets account with access to this document.

4. **Connect** both "When clicking 'Execute workflow'" and "Schedule Trigger" to "Get row(s) in sheet1".

5. **Create a Code node** named "URL Preprocessor".  
   - Paste the JavaScript code that:  
     - Trims domains  
     - Adds `https://` protocol if missing  
     - Removes trailing slashes  
     - Stores original domain in `OriginalDomain`  
   - Input: Output of "Get row(s) in sheet1".

6. **Create a SplitInBatches node** named "Split In Batches1".  
   - Batch Size: 5  
   - Input: Output of "URL Preprocessor".

7. **Create an HTTP Request node** named "HTTP Request1".  
   - Set URL to `{{$json["Domain"]}}` (make dynamic from domain field)  
   - Method: GET  
   - Timeout: 20 seconds  
   - Follow Redirects: Enabled, max redirects 5  
   - Response Format: Text  
   - Retry: Enabled, 3 retries  
   - On error: Continue (do not fail workflow)  
   - Input: Output of "Split In Batches1".

8. **Create a Wait node** named "Rate Limiting Wait1".  
   - Wait for 2 seconds  
   - Input: Output of "HTTP Request1".

9. **Connect "Split In Batches1" to "HTTP Request1"** and then to "Rate Limiting Wait1".

10. **Create a Code node** named "Enhanced Technology Detection1".  
    - Paste the provided complex JavaScript that:  
      - Extracts domain robustly  
      - Checks errors and status codes  
      - Matches HTML content with multiple regexes and string patterns  
      - Detects platforms, frameworks, payment gateways, features, confidence score, and remarks  
    - Input: Output of "Rate Limiting Wait1".

11. **Create a Google Sheets node** named "Update Enhanced Results1".  
    - Operation: Update Rows  
    - Document ID and Sheet Name: Same as "Get row(s) in sheet1"  
    - Matching Columns: "Domain" to match existing rows  
    - Map columns: Domain, Platform, Cart, Catalog, Checkout, Wishlist, PWA, Payment Gateway, Framework, Confidence Score, Last Checked, Remarks  
    - Credentials: Same Google Sheets OAuth2 credentials  
    - Input: Output of "Enhanced Technology Detection1".

12. **Wire "Enhanced Technology Detection1" to "Update Enhanced Results1".**

13. **Verify all connections and credentials are correct.**

14. **Test workflow** by running manual trigger or waiting for schedule. Check Google Sheet updates.

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                                                                                  |
|--------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Schedule a Cron from Schedule Trigger                                                                   | Sticky note suggests using Schedule Trigger for automation                                      |
| Create your google sheet and add domain column                                                          | Setup note for input Google Sheet                                                              |
| Create your google sheet, add domain, Platform, etc. column                                             | Setup note for output Google Sheet                                                             |

---

**Disclaimer:** The text provided is exclusively derived from an automated n8n workflow. It strictly adheres to content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.