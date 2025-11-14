N8N Advanced URL Parsing and Shortening Workflow - Switchy.io Integration

https://n8nworkflows.xyz/workflows/n8n-advanced-url-parsing-and-shortening-workflow---switchy-io-integration-2064


# N8N Advanced URL Parsing and Shortening Workflow - Switchy.io Integration

### 1. Workflow Overview

This workflow titled **"N8N Advanced URL Parsing and Shortening Workflow - Switchy.io Integration"** automates the process of shortening URLs with enriched metadata and advanced URL parsing features. Its key use case is to intake long URLs, extract comprehensive metadata (titles, descriptions, images, favicons), perform safety and phishing checks, optionally generate or retrieve screenshots, and then create or update shortened links via the Switchy.io API. It also hosts generated images on GitHub repositories and supports multiple metadata extraction strategies.

The workflow is logically grouped into the following blocks:

- **1.1 Input Reception & Initialization:** Receives user input from an n8n form trigger and initializes variables including API keys and user preferences.
- **1.2 URL Safety & Phishing Scanning:** Checks the submitted URL against Norton, Bitdefender, PhishTank, and URLVoid services to detect malicious or unsafe URLs.
- **1.3 Metadata Extraction:** Uses multiple APIs and methods (OpenGraph API, headers, dub.sh scraper) to extract metadata and determine the best available data.
- **1.4 Metadata Validation & Fallbacks:** Validates metadata completeness and applies fallback methods if primary metadata sources fail.
- **1.5 Screenshot Generation & Handling:** Depending on user preference, generates screenshots or branded OpenGraph images for the URL.
- **1.6 Image Hosting on GitHub:** Uploads screenshots, OpenGraph images, and favicons to a GitHub repository for CDN delivery.
- **1.7 Switchy.io API Integration:** Creates or updates shortened URL records on Switchy.io, including enriched metadata and custom options.
- **1.8 Final Response:** Sends the shortened URL back to the user via webhook response.
- **1.9 Error Handling:** Stops workflow execution with specific error messages when critical failures occur.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Initialization

- **Overview:**  
  This block captures user inputs from a web form and sets up essential variables and API keys for subsequent processing.

- **Nodes Involved:**  
  - `n8n Form Trigger`  
  - `Split In Batches`  
  - `API Auth`  
  - `Respond to Webhook`  

- **Node Details:**  

  - **n8n Form Trigger**  
    - Type: FormTrigger  
    - Role: Entry point for user data submission. Collects Switchy API key, long URL, OpenGraph image mode, dark mode preference, and URL scan preference.  
    - Config: Form fields include required fields for API key, URL, dropdowns for image mode (Screenshot, Source, Brand), dark mode (yes/no), and URL scanning (Yes/No).  
    - Output connects to `Split In Batches`.  
    - Edge Cases: Missing required fields cause form submission errors; malformed URLs may cause issues downstream.  

  - **Split In Batches**  
    - Type: SplitInBatches  
    - Role: Processes URLs one at a time (batch size = 1) to avoid API rate limits and manage flow.  
    - Output connects to `API Auth` and `Respond to Webhook`.  
    - Edge Cases: Batch size too large may exceed API limits; batch processing prevents overload.  

  - **API Auth**  
    - Type: Set  
    - Role: Sets workflow-level variables including OpenGraph image mode (normalized to lowercase), dark mode boolean, brand name, Switchy API key, parsed long URL, scan preference, optional slug, tags, folder ID, and custom domain.  
    - Key Expressions: Uses regex to extract base URL from user input; converts text inputs to booleans.  
    - Output connects to URL scanning decision node (`If3`).  
    - Edge Cases: Improperly formatted URLs or missing API keys affect downstream API calls.  

  - **Respond to Webhook**  
    - Type: RespondToWebhook  
    - Role: Sends the final shortened URL text response back to the client after processing is complete.  
    - Input from `Split In Batches` (after processing).  
    - Edge Cases: If workflow errors, no valid response may be sent.

---

#### 2.2 URL Safety & Phishing Scanning

- **Overview:**  
  This block scans the submitted URL against multiple security services to detect phishing, fraud, or unsafe status before continuing.

- **Nodes Involved:**  
  - `If3`  
  - `Scan URL (Community)` (Norton Safe Web)  
  - `If`  
  - `If1`  
  - `Check Reviews (Community)`  
  - `Norton` (Set node)  
  - `(Fraud Check)` (Bitdefender)  
  - `HTML` (extract HTML content)  
  - `bitdefender` (Set with blocklist score)  
  - `(Others)` (URLVoid)  
  - `PhishTank`  
  - `If2`  
  - `set unsafe`  

- **Node Details:**  

  - **If3**  
    - Type: If  
    - Role: Checks if URL scanning is enabled (based on user input). If yes, proceeds to scan; otherwise, skips to metadata extraction.  
    - Edge Cases: If user input is malformed, may behave unexpectedly.  

  - **Scan URL (Community)**  
    - Type: HttpRequest  
    - Role: Calls Norton Safe Web API to check site safety and rating.  
    - Query param: Extracts domain from long URL using regex.  
    - Handles redirects and uses custom headers.  
    - Output connects to `If`.  
    - Edge Cases: API rate limits, network errors, or invalid domain inputs.  

  - **If**  
    - Type: If  
    - Role: Checks if Norton rating is "g" (good) or "r" (restricted).  
    - Output true proceeds to `If1`, false leads to `set unsafe`.  
    - Edge Cases: Missing rating field or unexpected values.  

  - **If1**  
    - Type: If  
    - Role: Checks if review count from Norton is ≥ 1 to fetch reviews.  
    - Output true proceeds to `Check Reviews (Community)`, false directly to `Norton` set node.  

  - **Check Reviews (Community)**  
    - Type: HttpRequest  
    - Role: Retrieves top 2 reviews for the domain from Norton.  
    - Query parameters include sort type and pagination.  
    - Output connects to `Norton` set node.  

  - **Norton**  
    - Type: Set  
    - Role: Aggregates Norton rating results into named fields: Risk Rate, Community Rating, Ban (boolean if globally restricted).  

  - **(Fraud Check)**  
    - Type: HttpRequest  
    - Role: Posts to Bitdefender fraud_info API with domain extracted from URL.  
    - Output connects to `(Others)`.  
    - Edge Cases: API downtime, invalid input, or malformed responses.  

  - **HTML**  
    - Type: Html  
    - Role: Extracts "result" from Bitdefender response HTML to derive blocklist score.  
    - Output connects to `bitdefender`.  

  - **bitdefender**  
    - Type: Set  
    - Role: Stores safe boolean and blocklist score extracted from Bitdefender response.  
    - Output connects to `If2`.  

  - **(Others)**  
    - Type: HttpRequest  
    - Role: Scrapes URLVoid results for the domain.  
    - Output connects to `HTML`.  

  - **PhishTank**  
    - Type: HttpRequest  
    - Role: Checks PhishTank database for phishing status of domain.  
    - User-Agent header identifies as "phishtank/anestooo".  
    - Output connects to `(Fraud Check)` node.  

  - **If2**  
    - Type: If  
    - Role: Evaluates multiple conditions: blocklist score ≤ 5, Norton rating "g" or "r", or Bitdefender safe flag true.  
    - Pass leads to OpenGraph extraction; fail leads to `set unsafe`.  

  - **set unsafe**  
    - Type: Set  
    - Role: Sets output indicating unsafe/skipped URL due to security flags.  
    - Output connects to `Split In Batches` to finish processing.

---

#### 2.3 Metadata Extraction

- **Overview:**  
  This block extracts metadata from the URL using several methods/APIs and chooses the best available metadata for further processing.

- **Nodes Involved:**  
  - `OpenGraph API`  
  - `IF OpenGraph invaild`  
  - `Get Headers`  
  - `Parse headers` (HTML extraction)  
  - `Meta tags Scraper - dub.sh`  
  - `IF dub invaild`  
  - `Method 1 - META`  
  - `Method 2 - META`  
  - `Method 3 - META1`  
  - `Method 4 - META`  
  - `IF GH invaild`  
  - `Final Meta` (Aggregate)  
  - `Stop and Error`  

- **Node Details:**  

  - **OpenGraph API**  
    - Type: HttpRequest  
    - Role: Queries https://www.opengraph.xyz API for metadata extraction with user-agent and headers set to mimic browser requests.  
    - Output connects to `IF OpenGraph invaild`.  
    - Edge Cases: API errors, unauthorized access, or incomplete metadata.  

  - **IF OpenGraph invaild**  
    - Type: If  
    - Role: Checks if OpenGraph API response is valid (no "Unauthorized access" or error in body).  
    - True path uses `Method 1 - META`, false path falls back to `Get Headers`.  

  - **Get Headers**  
    - Type: HttpRequest  
    - Role: Sends HTTP request to URL to retrieve headers and full response.  
    - Output connects to `Parse headers` and `Meta tags Scraper - dub.sh`.  
    - On error, continues to next node.  

  - **Parse headers**  
    - Type: Html  
    - Role: Extracts metadata from HTML meta tags including OpenGraph title, description, image, favicon, and page title.  
    - Output connects to `Method 2 - META`.  

  - **Meta tags Scraper - dub.sh**  
    - Type: HttpRequest  
    - Role: Calls Dub.sh API to scrape meta tags from page.  
    - Output connects to `IF dub invaild`.  

  - **IF dub invaild**  
    - Type: If  
    - Role: Checks if dub.sh metadata contains valid title, description, image, and no error indications.  
    - If valid, leads to `Method 4 - META`; else triggers `Stop and Error`.  

  - **Method 1 - META**  
    - Type: Set  
    - Role: Normalizes and constructs metadata fields from OpenGraph API results, applying fallbacks for title, description, and images.  
    - Builds CDN URLs for images based on OpenGraph Image Mode and user preferences.  
    - Output connects to `Final Meta`.  

  - **Method 2 - META**  
    - Type: Set  
    - Role: Similar to Method 1 but uses parsed HTML headers data.  
    - Output connects to `IF GH invaild`.  

  - **Method 3 - META1**  
    - Type: Set  
    - Role: Uses dub.sh's metadata fields as fallback with similar normalization and image URL construction.  
    - Output connects to `Final Meta`.  

  - **Method 4 - META**  
    - Type: Set  
    - Role: Another fallback method for metadata normalization using dub.sh data with error replacements.  
    - Output connects to `Final Meta`.  

  - **IF GH invaild**  
    - Type: If  
    - Role: Checks if metadata from parsed headers has missing or invalid fields (ogTitle, ogDescription, ogImage), to decide whether to use dub.sh fallback or proceed.  

  - **Final Meta**  
    - Type: Aggregate  
    - Role: Aggregates all metadata items into a single unified structure for further processing.  
    - Output connects to `If - Enable ScreenShots (yes to enable)`.  

  - **Stop and Error**  
    - Type: StopAndError  
    - Role: Stops workflow and throws error message if metadata extraction fails critically.

---

#### 2.4 Screenshot Generation & Handling

- **Overview:**  
  Depending on user preference, this block generates screenshots or brand-based OpenGraph images of the target URL.

- **Nodes Involved:**  
  - `If - Enable ScreenShots (yes to enable)`  
  - `Method 1 - SCR` (microlink API proxy)  
  - `Method 2 - SCR` (pxl.to API)  
  - `Convert to File`  
  - `Final SCR` (Aggregate)  
  - `Stop And Error`  
  - `Edit Fields`  

- **Node Details:**  

  - **If - Enable ScreenShots (yes to enable)**  
    - Type: If  
    - Role: Checks if OpenGraph Image Mode is set to "screenshot".  
    - True path uses screenshot APIs, false skips to metadata finalization.  

  - **Method 1 - SCR**  
    - Type: HttpRequest  
    - Role: Calls Switchy.io proxy to Microlink API to get screenshot & meta info of the URL.  
    - Output connects to `Final SCR` and `Method 2 - SCR`.  

  - **Method 2 - SCR**  
    - Type: HttpRequest  
    - Role: Alternative screenshot via pxl.to API to capture full page screenshot.  
    - Output connects to `Convert to File`.  

  - **Convert to File**  
    - Type: ConvertToFile  
    - Role: Converts base64 screenshot data to binary file format for GitHub upload.  
    - Output connects back to `Final SCR`.  

  - **Final SCR**  
    - Type: Aggregate  
    - Role: Aggregates screenshot data for upload nodes.  
    - Output connects to `Host Screenshot`.  

  - **Stop And Error**  
    - Type: StopAndError  
    - Role: Handles screenshot generation failures gracefully.  

  - **Edit Fields**  
    - Type: Set  
    - Role: Sets the download URL field extracted from the GitHub upload response for later use.

---

#### 2.5 Image Hosting on GitHub

- **Overview:**  
  This block uploads screenshots, OpenGraph images, and favicons to a predefined GitHub repository for CDN delivery.

- **Nodes Involved:**  
  - `Host Screenshot`  
  - `Host OGImage`  
  - `Host Favicon`  
  - `Download final OG`  
  - `Download Favicon`  
  - `Edit Fields`  
  - `Final Data`  

- **Node Details:**  

  - **Host Screenshot**  
    - Type: GitHub  
    - Role: Uploads screenshot PNG binary to GitHub repo under `/screenshots/{domain}/scr/` with random 5-char filename.  
    - Uses owner and repository parameters set to user’s GitHub account and repo.  

  - **Host OGImage**  
    - Type: GitHub  
    - Role: Uploads OpenGraph image PNG binary under `/screenshots/{domain}/og/` with random 5-char filename.  

  - **Host Favicon**  
    - Type: GitHub  
    - Role: Uploads favicon PNG binary under `/screenshots/{domain}/icon/` with random 5-char filename.  

  - **Download final OG**  
    - Type: HttpRequest  
    - Role: Downloads final OpenGraph image, replacing placeholder "<SCR>" in URL with actual download URL from GitHub or default images.  

  - **Download Favicon**  
    - Type: HttpRequest  
    - Role: Downloads favicon image from URL specified in metadata.  

  - **Edit Fields**  
    - Type: Set  
    - Role: Extracts `download_url` from GitHub API response for final image URLs.  

  - **Final Data**  
    - Type: Set  
    - Role: Combines all final metadata fields including Title, Description, image URLs (converted to CDN format), favicon URL, slug, tags, folder ID, API key, brand name, and domain for Switchy API call.

---

#### 2.6 Switchy.io API Integration

- **Overview:**  
  This block creates or updates the shortened URL on Switchy.io using the enriched metadata.

- **Nodes Involved:**  
  - `CREATE`  
  - `IF Slug available`  
  - `UPDATE`  
  - `Shortened URL`  

- **Node Details:**  

  - **CREATE**  
    - Type: HttpRequest  
    - Role: Sends POST request to Switchy API `/links/create` endpoint with JSON body including URL, tags, domain, title, description, image, slug, folder ID, favicon, and note.  
    - Headers include API authorization from user input.  
    - Implements batching with max 15 requests per minute to respect API limits.  
    - Output connects to `IF Slug available`.  

  - **IF Slug available**  
    - Type: If  
    - Role: Checks if the creation response status is HTTP 201 (Created).  
    - True outputs to `Shortened URL` node (success path).  
    - False outputs to `UPDATE` node (assumes link exists and needs update).  

  - **UPDATE**  
    - Type: HttpRequest  
    - Role: Sends PUT request to Switchy API `/links/by-domain/{domain}/{id}` to update existing link with new metadata.  
    - Uses domain and ID from creation response.  
    - Output connects to `Shortened URL`.  

  - **Shortened URL**  
    - Type: Set  
    - Role: Constructs the final shortened URL string using domain and ID returned from Switchy API.  
    - Also sets the OG method used.  
    - Output connects to `Split In Batches` to finalize.

---

#### 2.7 Final Response & Error Handling

- **Overview:**  
  This block manages final output to user and error handling.

- **Nodes Involved:**  
  - `Respond to Webhook` (covered in 2.1)  
  - `Stop and Error` (metadata failure)  
  - `Stop And Error` (screenshot failure)  

- **Node Details:**  

  - **Stop and Error**  
    - Type: StopAndError  
    - Role: Stops workflow on metadata extraction failure with message prompting user to report issue.  

  - **Stop And Error**  
    - Type: StopAndError  
    - Role: Stops workflow on screenshot generation failure with message prompting user to report issue.  

---

### 3. Summary Table

| Node Name                  | Node Type           | Functional Role                              | Input Node(s)                | Output Node(s)                  | Sticky Note                                                                             |
|----------------------------|---------------------|----------------------------------------------|-----------------------------|--------------------------------|-----------------------------------------------------------------------------------------|
| n8n Form Trigger           | FormTrigger         | Entry point: collects user inputs             |                             | Split In Batches                |                                                                                         |
| Split In Batches           | SplitInBatches      | Batch processing to handle URLs one at a time | n8n Form Trigger, set unsafe | API Auth, Respond to Webhook    |                                                                                         |
| API Auth                  | Set                 | Setup variables and parse user inputs         | Split In Batches             | If3                            |                                                                                         |
| If3                       | If                  | Decide to scan URLs or proceed                 | API Auth                    | Scan URL (Community), OpenGraph API |                                                                                         |
| Scan URL (Community)       | HttpRequest         | Norton Safe Web URL safety check               | If3                         | If                            |                                                                                         |
| If                        | If                  | Check Norton rating for safety                  | Scan URL (Community)         | If1, set unsafe                |                                                                                         |
| If1                       | If                  | Check if reviews exist                           | If                          | Check Reviews (Community), Norton |                                                                                         |
| Check Reviews (Community)  | HttpRequest         | Fetch Norton reviews                            | If1                         | Norton                        |                                                                                         |
| Norton                    | Set                 | Aggregate Norton scan results                   | If1, Check Reviews (Community) | PhishTank                    |                                                                                         |
| PhishTank                 | HttpRequest         | Phishing check                                 | Norton                      | (Fraud Check)                 |                                                                                         |
| (Fraud Check)             | HttpRequest         | Bitdefender fraud info check                    | PhishTank                   | (Others)                      |                                                                                         |
| (Others)                  | HttpRequest         | URLVoid scan                                   | (Fraud Check)               | HTML                         |                                                                                         |
| HTML                      | Html                | Extract blocklist score from URLVoid           | (Others)                    | bitdefender                   |                                                                                         |
| bitdefender               | Set                 | Set safe flag and blocklist score              | HTML                        | If2                          |                                                                                         |
| If2                       | If                  | Final safety decision gate                      | bitdefender, If             | OpenGraph API, set unsafe     |                                                                                         |
| set unsafe                | Set                 | Mark URL as unsafe/skipped                      | If                         | Split In Batches               |                                                                                         |
| OpenGraph API             | HttpRequest         | Fetch metadata via OpenGraph API                | If3, If2                    | IF OpenGraph invaild          |                                                                                         |
| IF OpenGraph invaild      | If                  | Validate OpenGraph API response                 | OpenGraph API               | Method 1 - META, Get Headers  |                                                                                         |
| Get Headers               | HttpRequest         | Fetch HTTP headers and full response            | IF OpenGraph invaild        | Parse headers, Meta tags Scraper - dub.sh |                                                                                         |
| Parse headers             | Html                | Extract metadata from HTML headers              | Get Headers                 | Method 2 - META               |                                                                                         |
| Meta tags Scraper - dub.sh| HttpRequest         | Dub.sh API metadata scraping                     | Get Headers, IF GH invaild  | IF dub invaild                |                                                                                         |
| IF dub invaild            | If                  | Validate dub.sh metadata                         | Meta tags Scraper - dub.sh  | Method 4 - META, Stop and Error |                                                                                         |
| Method 1 - META           | Set                 | Normalize OpenGraph API metadata                | IF OpenGraph invaild        | Final Meta                   |                                                                                         |
| Method 2 - META           | Set                 | Normalize parsed header metadata                 | Parse headers               | IF GH invaild                 |                                                                                         |
| Method 3 - META1          | Set                 | Normalize dub.sh metadata (fallback)            | IF dub invaild (false path) | Final Meta                   |                                                                                         |
| Method 4 - META           | Set                 | Normalize dub.sh metadata (fallback)            | IF dub invaild (true path)  | Final Meta                   |                                                                                         |
| IF GH invaild             | If                  | Check if parsed headers metadata is invalid     | Method 2 - META             | Meta tags Scraper - dub.sh, Final Meta |                                                                                         |
| Final Meta                | Aggregate           | Aggregate all metadata results                   | Method 1 - META, Method 3 - META1, Method 4 - META, IF GH invaild | If - Enable ScreenShots (yes to enable) |                                                                                         |
| If - Enable ScreenShots (yes to enable) | If       | Check if screenshots should be generated        | Final Meta                  | Method 1 - SCR, Edit Fields   |                                                                                         |
| Method 1 - SCR            | HttpRequest         | Screenshot via Microlink API proxy               | If - Enable ScreenShots     | Final SCR, Method 2 - SCR     |                                                                                         |
| Method 2 - SCR            | HttpRequest         | Screenshot via pxl.to API                         | Method 1 - SCR              | Convert to File, Stop And Error |                                                                                         |
| Convert to File           | ConvertToFile       | Convert base64 screenshot to binary file         | Method 2 - SCR              | Final SCR                    |                                                                                         |
| Final SCR                 | Aggregate           | Aggregate screenshot data                         | Method 1 - SCR, Convert to File | Host Screenshot            |                                                                                         |
| Stop And Error            | StopAndError        | Handle screenshot generation errors               | Method 2 - SCR              |                              |                                                                                         |
| Edit Fields               | Set                 | Extract download_url from GitHub upload response | Final SCR                   | Download final OG            |                                                                                         |
| Host Screenshot           | GitHub              | Upload screenshot to GitHub repository             | Final SCR                   | Edit Fields                  |                                                                                         |
| Download final OG         | HttpRequest         | Download final OpenGraph image (replaces placeholders) | Edit Fields                  | Host OGImage                 |                                                                                         |
| Host OGImage              | GitHub              | Upload OpenGraph image to GitHub                   | Download final OG           | Download Favicon             |                                                                                         |
| Download Favicon          | HttpRequest         | Download favicon image                              | Host OGImage                | Host Favicon                 |                                                                                         |
| Host Favicon              | GitHub              | Upload favicon image to GitHub repository           | Download Favicon            | Final Data                   |                                                                                         |
| Final Data                | Set                 | Prepare final data payload for Switchy API         | Host Favicon                | CREATE                       |                                                                                         |
| CREATE                    | HttpRequest         | Create shortened link on Switchy.io                 | Final Data                  | IF Slug available            | Sticky Note1: Switchy API limits (10k links/day, 1k/hour, 16/minute)                   |
| IF Slug available         | If                  | Check if creation successful (201 Created)          | CREATE                      | Shortened URL, UPDATE        |                                                                                         |
| UPDATE                    | HttpRequest         | Update existing shortened link on Switchy.io        | IF Slug available (fail)    | Shortened URL                |                                                                                         |
| Shortened URL             | Set                 | Set final shortened URL output                        | IF Slug available, UPDATE   | Split In Batches             |                                                                                         |
| Respond to Webhook        | RespondToWebhook    | Return shortened URL to user                          | Split In Batches            |                              |                                                                                         |
| Stop and Error            | StopAndError        | Handle metadata extraction failures                  | IF dub invaild (fail path)  |                              |                                                                                         |
| Stop And Error            | StopAndError        | Handle screenshot generation failures                 | Method 2 - SCR (fail path)  |                              |                                                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create `n8n Form Trigger` Node:**  
   - Type: FormTrigger  
   - Path: `switchy`  
   - Form Fields:  
     - Textarea, label: "What's your Switchy API Key" (required)  
     - Textarea, label: "What's Your LongURL ?" (required)  
     - Dropdown, label: "What's OG Image method you like ?" with options: Screenshot, Source, brand (required)  
     - Dropdown, label: "With your brand, Do you Like dark mode ?" with options: no, yes (required)  
     - Dropdown, label: "scan Long URLs from virus/Phishing ? (Shorting a phishing URL will ban your domain from SEO)" with options: No, Skip Scan part., Yes, Scan all Links. (required)  
   - Set response mode to "responseNode".  

2. **Add `Split In Batches` Node:**  
   - Type: SplitInBatches  
   - Batch size: 1  
   - Connect `n8n Form Trigger` main output to `Split In Batches` input.  

3. **Add `API Auth` (Set) Node:**  
   - Type: Set  
   - Extract and normalize inputs:  
     - OpenGraph Image Mode: lowercase from form input  
     - Dark Mode: convert "yes"/"no" to boolean  
     - Brand Name: default "NodeMation"  
     - Switchy API Key: from form  
     - LongURL: extract first valid URL using regex  
     - Scan LongURL: from form  
     - Custom slug, tags, Switchy Folder ID, Custom Domain: optional empty strings or placeholders  
   - Connect `Split In Batches` output to `API Auth`.  

4. **Add `If3` Node to Decide URL Scan:**  
   - Condition: If "Scan LongURL" starts with "Yes," proceed to scan, else skip.  
   - Connect `API Auth` to `If3`.  

5. **Add Norton Safe Web Scan (`Scan URL (Community)` HttpRequest):**  
   - URL: `https://safeweb.norton.com/safeweb/sites/v1/details` with query param `url` extracted domain.  
   - Set headers to mimic browser requests.  
   - Connect `If3` true output to this node.  

6. **Add `If` Node to Check Norton Rating:**  
   - Condition: `rating` equals "g" OR "r".  
   - Connect `Scan URL (Community)` to `If`.  

7. **Add `If1` Node to Check Norton Reviews:**  
   - Condition: `reviewCount` ≥ 1.  
   - Connect `If` true output to `If1`.  

8. **Add `Check Reviews (Community)` HttpRequest:**  
   - URL: Norton API to fetch reviews, paginated.  
   - Connect `If1` true output here, false output to `Norton`.  

9. **Add `Norton` Set Node:**  
   - Set Risk Rate, Community Rating, Ban fields from Norton API responses.  
   - Connect `Check Reviews (Community)` and `If1` false outputs here.  

10. **Add `PhishTank` HttpRequest Node:**  
    - URL: PhishTank check with domain from URL.  
    - Connect `Norton` to `PhishTank`.  

11. **Add `(Fraud Check)` Bitdefender HttpRequest Node:**  
    - POST request with domain to Bitdefender's fraud_info endpoint.  
    - Connect `PhishTank` output here.  

12. **Add `(Others)` HttpRequest Node for URLVoid:**  
    - GET request to URLVoid scan page.  
    - Connect `(Fraud Check)` output here.  

13. **Add `HTML` Node to Extract Blocklist Score:**  
    - Extract element with CSS selector `.label`.  
    - Connect `(Others)` output here.  

14. **Add `bitdefender` Set Node:**  
    - Set safe boolean and blocklist score from HTML extraction.  
    - Connect `HTML` output here.  

15. **Add `If2` Node to Finalize Safety Decision:**  
    - Conditions: blocklist score ≤ 5 OR Norton rating "g" or "r" OR Bitdefender safe true.  
    - Connect `bitdefender` and `If` outputs here.  
    - True path to `OpenGraph API`, false path to `set unsafe`.  

16. **Add `set unsafe` Set Node:**  
    - Set output indicating unsafe/skipped URL.  
    - Connect `If2` false output here, then back to `Split In Batches` to continue.  

17. **Add `OpenGraph API` HttpRequest Node:**  
    - Call OpenGraph API with URL encoded long URL.  
    - Set headers to mimic browser and avoid errors.  
    - Connect `If3` false output and `If2` true output here.  

18. **Add `IF OpenGraph invaild` If Node:**  
    - Check if OpenGraph API response is unauthorized or error.  
    - True path to `Method 1 - META`.  
    - False path to `Get Headers`.  

19. **Add `Get Headers` HttpRequest Node:**  
    - Request to LongURL to get full headers and response.  
    - Connect `IF OpenGraph invaild` false output here.  

20. **Add `Parse headers` HTML Node:**  
    - Extract meta tags (ogTitle, ogDescription, ogImage, favicon, etc.) from response.  
    - Connect `Get Headers` output here.  

21. **Add `Meta tags Scraper - dub.sh` HttpRequest Node:**  
    - Call Dub.sh API with LongURL to scrape meta tags.  
    - Connect `Get Headers` output here (parallel).  

22. **Add `IF dub invaild` If Node:**  
    - Check if dub.sh metadata is valid or contains "No title", "No description", "No image", or "Error".  
    - True path to `Method 4 - META`.  
    - False path to `Stop and Error`.  

23. **Add `Method 1 - META` Set Node:**  
    - Normalize OpenGraph API metadata with fallbacks for title, description, image, favicon.  
    - Construct CDN URLs for images based on user preferences and OpenGraph Image Mode.  
    - Connect `IF OpenGraph invaild` true output here.  

24. **Add `Method 2 - META` Set Node:**  
    - Normalize metadata extracted from headers.  
    - Connect `Parse headers` output here.  

25. **Add `IF GH invaild` If Node:**  
    - Check if Method 2 metadata is valid.  
    - True path to `Meta tags Scraper - dub.sh`.  
    - False path to `Final Meta`.  

26. **Add `Method 3 - META1` Set Node:**  
    - Normalize dub.sh metadata as fallback.  
    - Connect `IF dub invaild` false output here.  

27. **Add `Method 4 - META` Set Node:**  
    - Another fallback normalization of dub.sh metadata.  
    - Connect `IF dub invaild` true output here.  

28. **Add `Final Meta` Aggregate Node:**  
    - Aggregate all metadata into unified output.  
    - Connect `Method 1 - META`, `Method 3 - META1`, `Method 4 - META`, and `IF GH invaild` false output here.  

29. **Add `If - Enable ScreenShots (yes to enable)` If Node:**  
    - Check if OpenGraph Image Mode = "screenshot".  
    - True path to screenshot API calls, false path to `Edit Fields`.  

30. **Add `Method 1 - SCR` HttpRequest:**  
    - Proxy call to Microlink API for screenshot and metadata.  
    - Connect `If - Enable ScreenShots` true output here.  

31. **Add `Method 2 - SCR` HttpRequest:**  
    - Optional screenshot via pxl.to API.  
    - Connect `Method 1 - SCR` output here.  

32. **Add `Convert to File` Node:**  
    - Convert base64 screenshot to binary.  
    - Connect `Method 2 - SCR` output here.  

33. **Add `Final SCR` Aggregate Node:**  
    - Aggregate screenshot data.  
    - Connect `Method 1 - SCR` and `Convert to File` outputs here.  

34. **Add `Stop And Error` Node:**  
    - Stops workflow if screenshot generation fails.  
    - Connect `Method 2 - SCR` error output here.  

35. **Add `Host Screenshot` GitHub Node:**  
    - Upload screenshot to GitHub repo under folder `/screenshots/{domain}/scr/`.  
    - Connect `Final SCR` output here.  

36. **Add `Edit Fields` Set Node:**  
    - Extract `download_url` from GitHub upload response.  
    - Connect `Host Screenshot` output here.  

37. **Add `Download final OG` HttpRequest:**  
    - Download final OpenGraph image, replacing placeholders with actual URLs.  
    - Connect `Edit Fields` output here.  

38. **Add `Host OGImage` GitHub Node:**  
    - Upload OpenGraph image to GitHub under `/screenshots/{domain}/og/`.  
    - Connect `Download final OG` output here.  

39. **Add `Download Favicon` HttpRequest:**  
    - Download favicon from URL.  
    - Connect `Host OGImage` output here.  

40. **Add `Host Favicon` GitHub Node:**  
    - Upload favicon to GitHub under `/screenshots/{domain}/icon/`.  
    - Connect `Download Favicon` output here.  

41. **Add `Final Data` Set Node:**  
    - Compose final payload including title, description, image URLs (converted to CDN), favicon, slug, tags, folder ID, API key, brand name, domain.  
    - Connect `Host Favicon` output here.  

42. **Add `CREATE` HttpRequest Node:**  
    - POST to Switchy API `/links/create` with JSON body built from `Final Data`.  
    - Add API key header.  
    - Batching: 15 requests per minute.  
    - Connect `Final Data` output here.  

43. **Add `IF Slug available` If Node:**  
    - Check if HTTP status code = 201 (Created).  
    - True path to `Shortened URL`.  
    - False path to `UPDATE`.  

44. **Add `UPDATE` HttpRequest Node:**  
    - PUT to Switchy API `/links/by-domain/{domain}/{id}` to update existing link.  
    - Connect `IF Slug available` false output here.  

45. **Add `Shortened URL` Set Node:**  
    - Compose final shortened URL string and OG method used.  
    - Connect `IF Slug available` true output and `UPDATE` output here.  
    - Connect output back to `Split In Batches` to finish processing.  

46. **Connect `Split In Batches` output to `Respond to Webhook`** to send final shortened URL back to user.

47. **Add `Stop and Error` nodes at metadata and screenshot failure points for graceful stops with error messages.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                   | Context or Link                                                                                                             |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------|
| This workflow facilitates advanced URL shortening enriched with metadata and safety checks, integrating multiple APIs including Switchy.io, OpenGraph, Dub.sh, Microlink, Pxl.to, Norton Safe Web, Bitdefender, PhishTank, URLVoid, and GitHub for image hosting.                                                                                                                   | Workflow description                                                                                                        |
| Switchy API limits: 10,000 links per day, 1,000 links per hour max, 16 links per minute max.                                                                                                                                                                                                                                                                                     | Sticky Note1                                                                                                                |
| OpenGraph Image modes supported: "screenshot" (capture webpage screenshot), "source" (use original OG image), "brand" (generate branded image with dark/light mode). Brand name max 18 characters. Dark mode only affects "brand" image mode.                                                                                                                                     | Sticky Note6, README section                                                                                                |
| Safety scanning uses Norton Safe Web, Bitdefender, PhishTank, and URLVoid to prevent shortening unsafe or phishing URLs.                                                                                                                                                                                                                                                      | Nodes in URL Safety & Phishing block                                                                                        |
| Image hosting via GitHub requires a configured GitHub API credential and a repository named (by default) `n8n-templates-demos` under user `ARHAEEM`. Modify these parameters for your own GitHub setup.                                                                                                                                                                           | GitHub nodes configuration                                                                                                 |
| This workflow is a demo template and not intended for production use or spam. API usage might be rate limited or blocked if abused. Support is community-driven via [n8n community](https://community.n8n.io), creator @Nskha. Follow updates on [Telegram](https://nodemation.t.me).                                                                                                                                       | Sticky Note8                                                                                                                |
| Video overview available: [Advanced URL Parsing and Shortening Workflow - Switchy.io Integration](https://youtu.be/c7yCZhmMjtI)                                                                                                                                                                                                                                                | Workflow description video preview                                                                                          |

---

This structured reference document should enable advanced users and automation agents to understand, reproduce, and modify the workflow confidently, while anticipating potential error states and integration requirements.