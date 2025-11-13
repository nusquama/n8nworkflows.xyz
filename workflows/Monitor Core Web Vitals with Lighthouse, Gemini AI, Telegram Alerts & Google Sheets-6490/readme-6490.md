Monitor Core Web Vitals with Lighthouse, Gemini AI, Telegram Alerts & Google Sheets

https://n8nworkflows.xyz/workflows/monitor-core-web-vitals-with-lighthouse--gemini-ai--telegram-alerts---google-sheets-6490


# Monitor Core Web Vitals with Lighthouse, Gemini AI, Telegram Alerts & Google Sheets

### 1. Workflow Overview

This workflow automates the monitoring of Core Web Vitals (CWV) for a website using Google’s PageSpeed Insights API (Lighthouse data), processes and summarizes the performance audits via Google Gemini AI, alerts via Telegram if any CWV scores fall below a threshold, and logs results into Google Sheets for historical tracking.

**Target Use Cases:**  
- Web developers and SEO specialists seeking automated, periodic monitoring of site performance metrics.  
- Performance engineers aiming to detect regressions early and get actionable insights.  
- Digital agencies providing reporting and alerts to clients.  

**Logical Blocks:**

- **1.1 Trigger & Data Collection:** Scheduled trigger initiates the workflow, calls Google PageSpeed Insights API for Lighthouse report.  
- **1.2 Data Processing & Filtering:** Extract CWV metrics and audit descriptions from raw API response via JavaScript code node.  
- **1.3 AI Summarization:** Use Google Gemini language model to generate a concise summary of audit descriptions into actionable items.  
- **1.4 Conditional Alerting:** Evaluate if any Core Web Vital score is below threshold (0.95) and trigger Telegram alert if so.  
- **1.5 Data Archival:** Append summarized data and metrics into a specific Google Sheets document for tracking.  
- **1.6 Ancillary:** Sticky notes provide documentation and guidance throughout.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Data Collection

- **Overview:**  
  Initiates workflow on a schedule (every minute), then fetches Lighthouse report data from Google PageSpeed Insights API for a specified URL and device strategy.

- **Nodes Involved:**  
  - Schedule Trigger  
  - call page speed insights  

- **Node Details:**  

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Starts the workflow on a recurring schedule (every minute).  
    - Config: Interval set to every 1 minute.  
    - Input: None  
    - Output: Triggers downstream HTTP request node.  
    - Edge Cases: Scheduling delays or system downtime may cause missed runs.  

  - **call page speed insights**  
    - Type: HTTP Request  
    - Role: Calls Google PageSpeed Insights API v5 to get Lighthouse performance report for a URL.  
    - Config:  
      - URL: `https://www.googleapis.com/pagespeedonline/v5/runPagespeed`  
      - Query parameters:  
        - `url` (dynamic, presumably set via credentials or environment, not visible in JSON)  
        - `key` (API key for authentication)  
        - `strategy`: "mobile" (to simulate mobile device performance)  
    - Input: Trigger from Schedule Trigger  
    - Output: JSON response with Lighthouse report data  
    - Edge Cases:  
      - API quota limits or authentication failures cause errors.  
      - Network timeouts or invalid URLs.  
      - API response changes or schema updates.  

---

#### 1.2 Data Processing & Filtering

- **Overview:**  
  Extracts specific Core Web Vitals metrics and scores, collects all other audit descriptions into a single summarized field.

- **Nodes Involved:**  
  - processing data cwv and audits  

- **Node Details:**  

  - **processing data cwv and audits**  
    - Type: Code (JavaScript)  
    - Role: Parses Lighthouse JSON output to isolate CWV metrics and aggregate audit descriptions.  
    - Configuration:  
      - Defines a list of CWV metrics: first-contentful-paint, speed-index, largest-contentful-paint, total-blocking-time, interactive, cumulative-layout-shift.  
      - Constructs an object mapping each CWV metric to its `value` and `score`.  
      - Filters all other audits (non-CWV) and concatenates their titles and cleaned descriptions into a single string.  
      - Returns JSON with two keys: `cwv` (object of CWV metrics) and `audits` (concatenated description).  
    - Input: JSON from HTTP Request node  
    - Output: Processed JSON with `cwv` and `audits` fields  
    - Edge Cases:  
      - Missing audit keys or unexpected Lighthouse schema changes could cause runtime errors.  
      - Empty or malformed API responses.  
      - String manipulations may fail if description formats change.  

---

#### 1.3 AI Summarization

- **Overview:**  
  Uses Google Gemini AI to generate a concise, bullet-point summary of audit descriptions as actionable recommendations.

- **Nodes Involved:**  
  - Google Gemini Chat Model  
  - summarize audit description  

- **Node Details:**  

  - **Google Gemini Chat Model**  
    - Type: AI Language Model (Google Gemini via Langchain)  
    - Role: Provides AI-powered text generation capabilities.  
    - Configuration: Uses Google PaLM API credentials.  
    - Input: Text prompt from the summarization node.  
    - Output: AI-generated text summary.  
    - Edge Cases:  
      - API quota or authentication failures.  
      - Latency or timeouts from AI service.  
      - Prompt format changes affecting output quality.  

  - **summarize audit description**  
    - Type: Langchain Chain Summarization  
    - Role: Prepares and sends audit descriptions text to Gemini for summarization.  
    - Configuration:  
      - Prompt instructs AI to act as a web engineer and core web vitals expert.  
      - Output is a bullet-point list of action items without introductory text.  
      - Uses advanced chunking for long text.  
    - Input: JSON with audit descriptions from code node.  
    - Output: Summarized text.  
    - Edge Cases:  
      - Very long input texts could challenge chunking limits.  
      - Unexpected input structure may cause summarization failure.  

---

#### 1.4 Conditional Alerting

- **Overview:**  
  Checks if any CWV score is below 0.95; if true, sends a Telegram notification with detailed CWV scores.

- **Nodes Involved:**  
  - If  
  - Send Notification  

- **Node Details:**  

  - **If**  
    - Type: Conditional (If) node  
    - Role: Evaluates multiple conditions on CWV scores to decide if alert is needed.  
    - Configuration:  
      - Checks if any of first-contentful-paint, speed-index, largest-contentful-paint, cumulative-layout-shift, total-blocking-time scores are less than 0.95.  
      - Uses logical OR to trigger on any failing metric.  
    - Input: JSON with CWV scores from code node.  
    - Output: Routes true branch to send notification, false branch does nothing.  
    - Edge Cases:  
      - Missing or malformed CWV data may cause condition evaluation errors.  

  - **Send Notification**  
    - Type: Telegram node  
    - Role: Sends alert message to Telegram chat via bot API.  
    - Configuration:  
      - Message text template includes CWV values and scores for multiple metrics.  
      - Uses Telegram OAuth2 credentials.  
    - Input: Triggered only if If condition is true.  
    - Output: Telegram message sent confirmation.  
    - Edge Cases:  
      - Telegram API failures or invalid bot token.  
      - Message formatting errors if CWV data missing.  
      - Network issues causing send failures.  

---

#### 1.5 Data Archival

- **Overview:**  
  Appends summarized audit text and CWV scores to a Google Sheets document for record-keeping and trend analysis.

- **Nodes Involved:**  
  - Append row in specific sheet  

- **Node Details:**  

  - **Append row in specific sheet**  
    - Type: Google Sheets node  
    - Role: Appends a new row with summarized data into a Google Sheets spreadsheet.  
    - Configuration:  
      - Operation: Append  
      - Target sheet name and document ID (to be configured)  
      - Uses Google OAuth2 credentials.  
    - Input: Summarized text from AI node.  
    - Output: Confirmation of row append.  
    - Edge Cases:  
      - Incorrect sheet/document ID or permission errors.  
      - API rate limits or network failures.  
      - Data format mismatch causing append failure.  

---

#### 1.6 Ancillary Nodes (Documentation)

- **Overview:**  
  Sticky Notes provide inline documentation and references for workflow users.

- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note1  
  - Sticky Note2  
  - Sticky Note3  
  - Sticky Note4  

- **Node Details:**  

  - Sticky Note (positioned top-left)  
    - Content: Describes workflow purpose and ideal users (developers, SEO, agencies).  
  - Sticky Note1  
    - Content: Reference link and usage guidance for PageSpeed Insights API.  
    - Link: https://developers.google.com/speed/docs/insights/v5/get-started  
  - Sticky Note2  
    - Content: Explanation of JavaScript node processing CWV and concatenating audits.  
  - Sticky Note3  
    - Content: Reminder that Telegram message is sent only if any CWV score is below threshold.  
  - Sticky Note4  
    - Content: Note that messenger can be replaced with preferred alternative.  

---

### 3. Summary Table

| Node Name                    | Node Type                          | Functional Role                              | Input Node(s)               | Output Node(s)              | Sticky Note                                                                                              |
|------------------------------|----------------------------------|----------------------------------------------|-----------------------------|-----------------------------|--------------------------------------------------------------------------------------------------------|
| Schedule Trigger             | Schedule Trigger                 | Initiate workflow on schedule                 |                             | call page speed insights     | Use the PageSpeed Insights API; Follow guidelines: https://developers.google.com/speed/docs/insights/v5/get-started |
| call page speed insights      | HTTP Request                    | Fetch Lighthouse report from PageSpeed API   | Schedule Trigger            | processing data cwv and audits |                                                                                                        |
| processing data cwv and audits| Code (JavaScript)               | Extract CWV metrics and audit descriptions   | call page speed insights    | summarize audit description, If | Javascript code: Processing API response to filter CWV and concatenate audit descriptions                |
| summarize audit description   | Langchain Chain Summarization   | Generate AI summary of audit descriptions    | processing data cwv and audits | Append row in specific sheet |                                                                                                        |
| Google Gemini Chat Model      | AI Language Model (Google Gemini)| Provide AI text generation for summarization | summarize audit description | summarize audit description  |                                                                                                        |
| If                           | Conditional (If)                | Check if any CWV score is below threshold    | processing data cwv and audits | Send Notification          | Send the message only if at least one of the CWV scores is below the desired threshold                   |
| Send Notification            | Telegram                        | Send Telegram alert if CWV scores low        | If (true branch)            |                             | You can change the messenger with your preference                                                       |
| Append row in specific sheet | Google Sheets                   | Append summarized data to Google Sheets      | summarize audit description |                             |                                                                                                        |
| Sticky Note                  | Sticky Note                    | Documentation                                |                             |                             | Automate Lighthouse report generation to Telegram or other messenger and Google Sheets                  |
| Sticky Note1                 | Sticky Note                    | Documentation                                |                             |                             | Use the PageSpeed Insights API; Follow guidelines: https://developers.google.com/speed/docs/insights/v5/get-started |
| Sticky Note2                 | Sticky Note                    | Documentation                                |                             |                             | Javascript code: Processing API response to filter CWV and concatenate audit descriptions                |
| Sticky Note3                 | Sticky Note                    | Documentation                                |                             |                             | Send the message only if at least one of the CWV scores is below the desired threshold                   |
| Sticky Note4                 | Sticky Note                    | Documentation                                |                             |                             | You can change the messenger with your preference                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node:**  
   - Type: Schedule Trigger  
   - Configuration: Set interval to trigger every 1 minute (or desired frequency).  
   - No credentials needed.  

2. **Create HTTP Request Node (call page speed insights):**  
   - Type: HTTP Request  
   - Configuration:  
     - URL: `https://www.googleapis.com/pagespeedonline/v5/runPagespeed`  
     - Method: GET  
     - Query Parameters:  
       - `url`: the website URL to analyze (set dynamically or as parameter)  
       - `key`: Google API key with access to PageSpeed Insights API  
       - `strategy`: "mobile" (or "desktop" as needed)  
   - Connect input from Schedule Trigger node.  
   - No additional credentials except API key.  

3. **Create Code Node (processing data cwv and audits):**  
   - Type: Code (JavaScript)  
   - Configuration: Copy JavaScript code to:  
     - Extract Core Web Vitals (first-contentful-paint, speed-index, largest-contentful-paint, total-blocking-time, interactive, cumulative-layout-shift) from Lighthouse JSON.  
     - Aggregate all other audit descriptions into one string with cleaned text.  
     - Return an object with `cwv` and `audits` properties.  
   - Connect input from HTTP Request node.  

4. **Create Langchain Chain Summarization Node (summarize audit description):**  
   - Type: Langchain Chain Summarization  
   - Configuration:  
     - Set prompt to instruct the AI to act as a web engineer expert summarizing audits into bullet-point action items without introductions.  
     - Use advanced chunking for handling longer input.  
   - Connect input from Code node.  

5. **Create Google Gemini Chat Model Node:**  
   - Type: AI Language Model (Google Gemini via Langchain)  
   - Credentials: Configure Google PaLM API credentials.  
   - Connect this node as the language model for the summarization node.  

6. **Create If Node:**  
   - Type: Conditional  
   - Configuration:  
     - Conditions check if any CWV score is less than 0.95:  
       - `first-contentful-paint.score` < 0.95  
       - OR `speed-index.score` < 0.95  
       - OR `largest-contentful-paint.score` < 0.95  
       - OR `cumulative-layout-shift.score` < 0.95  
       - OR `total-blocking-time.score` < 0.95  
   - Connect input from Code node.  

7. **Create Telegram Node (Send Notification):**  
   - Type: Telegram  
   - Credentials: Setup Telegram Bot API credentials.  
   - Configuration:  
     - Message Text includes:  
       ```
       Reminder: One of the Core Web Vitals has a score below 0.95!

       Page: Home

       FCP: {{ $json.cwv['first-contentful-paint'].value }}, Score: {{ $json.cwv['first-contentful-paint'].score }}

       Speed Index: {{ $json.cwv['speed-index'].value }}, Score: {{ $json.cwv['largest-contentful-paint'].score }}

       TBT: {{ $json.cwv['total-blocking-time'].value }}, Score: {{ $json.cwv['total-blocking-time'].score }}

       Interactive: {{ $json.cwv.interactive.value }}, Score: {{ $json.cwv.interactive.score }}

       CLS: {{ $json.cwv['cumulative-layout-shift'].value }}, Score: {{ $json.cwv['cumulative-layout-shift'].score }}
       ```  
   - Connect input from True branch of If node.  

8. **Create Google Sheets Node (Append row in specific sheet):**  
   - Type: Google Sheets  
   - Credentials: Setup Google OAuth2 credentials with access to desired spreadsheet.  
   - Configuration:  
     - Operation: Append  
     - Document ID: Google Sheets document URL or ID  
     - Sheet Name: Specify target sheet name  
     - Map input data from summarization node output (e.g., summary text, CWV metrics as columns).  
   - Connect input from summarize audit description node.  

9. **Add Sticky Note Nodes:**  
   - Add sticky notes at appropriate places to document workflow purpose, API references, code explanation, alerting conditions, and messenger flexibility.  

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                                           |
|-----------------------------------------------------------------------------------------------|--------------------------------------------------------------------------|
| Automate Lighthouse report generation to Telegram or other messenger and Google Sheets        | Overview of the workflow’s purpose and ideal users                       |
| Use the PageSpeed Insights API; Follow these guidelines                                       | https://developers.google.com/speed/docs/insights/v5/get-started         |
| Javascript code: Processing the API response to filter only Core Web Vitals and concatenate audits | Explanation of the code node's function                                  |
| Send the message only if at least one of the CWV scores is below the desired threshold         | Conditional alerting logic                                               |
| You can change the messenger with your preference                                              | Flexibility note for replacing Telegram with another messaging service  |

---

**Disclaimer:**  
The provided text is generated exclusively from an automated n8n workflow export. It complies with all content policies and contains no illegal, offensive, or protected elements. All data processed is legal and publicly accessible.