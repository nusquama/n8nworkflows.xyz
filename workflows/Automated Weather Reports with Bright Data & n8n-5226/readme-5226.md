Automated Weather Reports with Bright Data & n8n

https://n8nworkflows.xyz/workflows/automated-weather-reports-with-bright-data---n8n-5226


# Automated Weather Reports with Bright Data & n8n

### 1. Workflow Overview

This workflow automates the process of scraping current weather data for Paris, France, from a weather website using Bright Data’s proxy service to bypass anti-bot protections. It then extracts specific weather details (like temperature) from the fetched HTML page and logs them into a Google Sheet for historical tracking and further analysis.

The workflow is logically structured into three main blocks:

- **1.1 Input Trigger & Data Fetch:** Starts the workflow manually and uses Bright Data’s proxy API to retrieve the full HTML content of the weather page.
- **1.2 Data Extraction:** Parses the fetched HTML to extract targeted weather information using CSS selectors.
- **1.3 Data Logging:** Appends the extracted weather data into a Google Sheet, creating a persistent log for future use.

This design is ideal for users needing automated, reliable weather data collection without being blocked by websites, facilitating trend analysis, dashboards, or alerting based on weather changes.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Trigger & Data Fetch

**Overview:**  
This block initiates the workflow manually and fetches the raw HTML of the weather page via Bright Data’s proxy API, ensuring the request bypasses bot detection.

**Nodes Involved:**  
- Start Workflow  
- RequestFetch Weather via Bright data

**Node Details:**

- **Start Workflow**  
  - Type: Manual Trigger  
  - Role: Entry point to start the workflow manually or via scheduling if extended.  
  - Configuration: Default manual trigger with no parameters.  
  - Inputs: None  
  - Outputs: One output to the HTTP Request node  
  - Edge cases: None typical; failures could occur if workflow is not activated.

- **RequestFetch Weather via Bright data**  
  - Type: HTTP Request  
  - Role: Sends a POST request to Bright Data’s proxy API to retrieve the full HTML content of the weather page for Paris, France.  
  - Configuration:  
    - Method: POST  
    - URL: https://api.brightdata.com/request  
    - Body parameters:  
      - zone: "n8n_unblocker" (Bright Data zone configured for unblocking)  
      - url: "https://www.weather.com/weather/today/l/Paris,FR" (target weather page)  
      - country: "fr" (France)  
      - format: "raw" (request raw HTML)  
    - Headers: Authorization with Bearer token (API_KEY placeholder)  
  - Inputs: From Start Workflow  
  - Outputs: One output to the HTML Extractor node  
  - Edge cases:  
    - Authorization errors if API_KEY invalid or expired  
    - Network timeouts or API rate limits  
    - HTML format changes at source site affecting downstream parsing  

---

#### 1.2 Data Extraction

**Overview:**  
This block parses the raw HTML to extract specific weather data points such as temperature using CSS selectors.

**Nodes Involved:**  
- Extract Weather Info

**Node Details:**

- **Extract Weather Info**  
  - Type: HTML Extractor  
  - Role: Parses the HTML response and extracts weather info using CSS selectors.  
  - Configuration:  
    - Operation: Extract HTML content  
    - Extraction values:  
      - Key: "Temperature"  
      - CSS selector: Targets the `<span>` element with specific classes and data-testid attribute containing the temperature value (example selector provided).  
  - Inputs: From HTTP Request node  
  - Outputs: One output to Google Sheets node  
  - Edge cases:  
    - If the HTML structure changes, selector may fail to extract data  
    - Missing or empty values if page content is incomplete or blocked  
    - Handling encoding or special characters in extracted data  

---

#### 1.3 Data Logging

**Overview:**  
This block logs the extracted weather data into a Google Sheet, appending a new row with city, country, and temperature values.

**Nodes Involved:**  
- Log to Weather Sheet

**Node Details:**

- **Log to Weather Sheet**  
  - Type: Google Sheets  
  - Role: Appends extracted weather data as a new row in a specified Google Sheet.  
  - Configuration:  
    - Operation: Append  
    - Document ID: Google Sheet with weather report  
    - Sheet Name: "Sheet1" (gid=0)  
    - Columns mapped:  
      - City: Fixed value "Paris"  
      - Country: Fixed value "France"  
      - Temperature: Dynamically mapped from extracted HTML data (`={{ $json.a }}`, where `$json.a` corresponds to extracted temperature)  
    - Credential: Google Sheets OAuth2 with authorized account  
  - Inputs: From HTML Extractor node  
  - Outputs: None (terminal node)  
  - Edge cases:  
    - Authentication errors if OAuth token expired or revoked  
    - API rate limits or quota exceeded  
    - Data type mismatches or invalid mappings causing row append failures  

---

### 3. Summary Table

| Node Name                    | Node Type           | Functional Role                     | Input Node(s)              | Output Node(s)                 | Sticky Note                                                                                                             |
|------------------------------|---------------------|-----------------------------------|----------------------------|-------------------------------|-------------------------------------------------------------------------------------------------------------------------|
| Start Workflow               | Manual Trigger      | Starts workflow manually           | -                          | RequestFetch Weather via Bright data | Describes manual trigger role and how to start the workflow.                                                           |
| RequestFetch Weather via Bright data | HTTP Request       | Fetches HTML from weather site via Bright Data proxy | Start Workflow             | Extract Weather Info            | Explains use of Bright Data proxy for reliable scraping.                                                                |
| Extract Weather Info         | HTML Extractor      | Extracts temperature from HTML     | RequestFetch Weather via Bright data | Log to Weather Sheet            | Explains HTML parsing and data extraction using CSS selectors.                                                          |
| Log to Weather Sheet         | Google Sheets       | Appends extracted data to Google Sheet | Extract Weather Info        | -                             | Explains logging of weather data for historical tracking.                                                               |
| Sticky Note9                 | Sticky Note         | Workflow assistance and support info | -                          | -                             | Contains contact info and links for workflow support and tutorials.                                                     |
| Sticky Note3                 | Sticky Note         | Overview and detailed workflow explanation | -                          | -                             | Provides a full description of the workflow purpose, sections, and beginner-friendly explanations.                      |
| Sticky Note4                 | Sticky Note         | Section 1 details on starting and fetching weather | -                          | -                             | Details about Section 1 nodes and their roles.                                                                           |
| Sticky Note5                 | Sticky Note         | Section 2 details on extraction and logging | -                          | -                             | Details about Section 2 nodes and their roles.                                                                           |
| Sticky Note                  | Sticky Note         | Affiliate link for Bright Data support | -                          | -                             | Contains affiliate link and request for support via Bright Data referral.                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Add a **Manual Trigger** node named `Start Workflow` to serve as the entry point.

2. **Create HTTP Request Node:**  
   - Add an **HTTP Request** node named `RequestFetch Weather via Bright data`.  
   - Set Method to **POST**.  
   - Set URL to `https://api.brightdata.com/request`.  
   - Enable sending body and headers.  
   - Configure Body Parameters as JSON with:  
     - `zone` = `"n8n_unblocker"`  
     - `url` = `"https://www.weather.com/weather/today/l/Paris,FR"`  
     - `country` = `"fr"`  
     - `format` = `"raw"`  
   - Add header parameter:  
     - `Authorization` = `"Bearer YOUR_BRIGHT_DATA_API_KEY"` (replace with valid API key)  
   - Connect `Start Workflow` output to this node’s input.

3. **Create HTML Extractor Node:**  
   - Add an **HTML Extract** node named `Extract Weather Info`.  
   - Set operation to **Extract HTML Content**.  
   - Add extraction value:  
     - Key: `Temperature`  
     - CSS Selector: use the selector targeting the temperature span element, for example:  
       ```html
       span[data-testid="TemperatureValue"]
       ```  
   - Connect `RequestFetch Weather via Bright data` output to this node.

4. **Create Google Sheets Node:**  
   - Add a **Google Sheets** node named `Log to Weather Sheet`.  
   - Set operation to **Append** row.  
   - Specify the Google Sheet document ID where weather data will be stored.  
   - Set Sheet name to `"Sheet1"` or appropriate tab name.  
   - Map columns:  
     - `City`: fixed string `"Paris"`  
     - `Country`: fixed string `"France"`  
     - `Temperature`: map to the extracted temperature value from the HTML Extract node, expression like `={{ $json["Temperature"] }}` (adjust according to actual output key)  
   - Ensure Google Sheets OAuth2 credentials are configured and authorized.  
   - Connect `Extract Weather Info` output to this node.

5. **Link Nodes:**  
   - Confirm connections:  
     - `Start Workflow` → `RequestFetch Weather via Bright data` → `Extract Weather Info` → `Log to Weather Sheet`.

6. **Test Workflow:**  
   - Manually trigger the workflow.  
   - Verify HTML is fetched, temperature extracted, and data appended to Google Sheet.

7. **Add Sticky Notes (Optional):**  
   - Add sticky notes for documentation and support info as desired.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                   | Context or Link                                               |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| Workflow assistance and support contact: Yaron@nofluff.online                                                                                                | Support contact email                                          |
| YouTube channel with tips and tutorials: https://www.youtube.com/@YaronBeen/videos                                                                             | Video tutorials and workflow tips                              |
| LinkedIn profile for further insights and updates: https://www.linkedin.com/in/yaronbeen/                                                                     | Professional profile and networking                            |
| Affiliate link to support Bright Data usage: https://get.brightdata.com/1tndi4600b25                                                                           | Bright Data referral link for supporting content creation     |
| This workflow is designed for reliable weather data scraping using proxy services to avoid IP blocks and ensure data accuracy.                               | Workflow design principle                                     |
| Google Sheets OAuth2 credentials must be authorized with write access to the target spreadsheet.                                                              | Credential setup note                                         |
| Bright Data API key must be valid and have sufficient quota for proxy requests.                                                                                | Credential setup note                                         |

---

*Disclaimer: The text provided is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.*