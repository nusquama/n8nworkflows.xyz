Monitor SEO Keyword Rankings with SERP API & Send Drop Alerts via Slack

https://n8nworkflows.xyz/workflows/monitor-seo-keyword-rankings-with-serp-api---send-drop-alerts-via-slack-6676


# Monitor SEO Keyword Rankings with SERP API & Send Drop Alerts via Slack

### 1. Workflow Overview

This workflow automates the daily monitoring of SEO keyword rankings using the SERP API and sends alerts on significant ranking drops via Slack and email. It targets SEO teams and digital marketers who need continuous tracking of keyword performance and timely notifications on ranking deteriorations that require action.

The workflow is logically divided into functional blocks:

- **1.1 Scheduled Trigger:** Automatically initiates the workflow daily at a fixed time.
- **1.2 Keyword Data Retrieval & Filtering:** Loads the list of keywords from Google Sheets and filters to process only active ones.
- **1.3 Ranking Data Fetching:** Queries SERP API to get current Google rankings for each keyword.
- **1.4 Ranking Analysis & Change Detection:** Parses API responses, compares with historical data, and identifies significant ranking drops or changes.
- **1.5 Alert Filtering & Notifications:** Filters keywords with notable drops and sends alerts via Slack and Email.
- **1.6 Data Update & Reporting:** Updates the Google Sheet with new ranking data and generates a summary report of the workflow execution.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:**  
Triggers the SEO monitoring workflow automatically every day at 8 AM to ensure daily updates without manual intervention.

- **Nodes Involved:**  
  - Daily SEO Check Trigger

- **Node Details:**  
  - **Daily SEO Check Trigger:**  
    - Type: Cron Trigger  
    - Configured with cron expression `0 8 * * *` to run daily at 8 AM.  
    - No input connections (start node).  
    - Outputs to "Get Keywords Database" node.  
    - Edge cases: Cron misconfiguration could prevent triggering; server downtime at trigger time could delay execution.

---

#### 2.2 Keyword Data Retrieval & Filtering

- **Overview:**  
Retrieves the keyword list and metadata from a Google Sheet and filters only those keywords marked active for monitoring.

- **Nodes Involved:**  
  - Get Keywords Database  
  - Filter Active Keywords Only

- **Node Details:**  
  - **Get Keywords Database:**  
    - Type: Google Sheets (Read Operation)  
    - Reads keywords, target URLs, previous rankings from a specified Google Sheet document and sheet.  
    - Uses Service Account authentication.  
    - Outputs full list of keywords to filtering node.  
    - Edge cases: Google Sheets API quota limits, authentication failures, empty sheets or malformed data.

  - **Filter Active Keywords Only:**  
    - Type: Filter  
    - Filters input to only process entries where `keyword` is not empty and `monitor_active` equals `"true"`.  
    - Ensures only enabled keywords proceed, saving API calls and processing time.  
    - Input from "Get Keywords Database".  
    - Outputs filtered keywords to "Fetch Google Rankings via SERP API".  
    - Edge cases: Case sensitivity in filter conditions, missing fields, data type mismatches.

---

#### 2.3 Ranking Data Fetching

- **Overview:**  
Queries the SERP API with each active keyword to fetch up-to-date Google search rankings, tailored for the US location.

- **Nodes Involved:**  
  - Fetch Google Rankings via SERP API  
  - Wait For Response

- **Node Details:**  
  - **Fetch Google Rankings via SERP API:**  
    - Type: HTTP Request  
    - Sends GET requests to `https://serpapi.com/search` with parameters including keyword, location, language, domain, API key, and number of results (top 100).  
    - Uses HTTP Header Authentication with API key.  
    - Input: Keywords from filter node.  
    - Output passes to "Wait For Response" node.  
    - Edge cases: API rate limits, invalid API key, network timeouts, unexpected API response formats.

  - **Wait For Response:**  
    - Type: Wait  
    - Pauses execution to ensure SERP API responses are fully received and processed before moving ahead.  
    - Input from HTTP Request node, output to ranking parsing node.  
    - Edge cases: Long response times could cause workflow delays.

---

#### 2.4 Ranking Analysis & Change Detection

- **Overview:**  
Processes the SERP API JSON responses to identify the current rank of the target URLs, compare with previous ranks, and determine any ranking improvements, drops, or if the keyword is missing from top 100 results.

- **Nodes Involved:**  
  - Parse Rankings & Detect Changes

- **Node Details:**  
  - **Parse Rankings & Detect Changes:**  
    - Type: Code (JavaScript)  
    - Iterates through the organic search results, matches the target URL domain to find the ranking position.  
    - Calculates ranking change (improvement or drop).  
    - Flags keywords needing alerts if drop > 2 positions or if not found in top 100.  
    - Outputs enriched data including `keyword`, `target_url`, `current_rank`, `previous_rank`, `ranking_change`, `ranking_drop`, `status` (improved, dropped, stable, not_found), `found_url`, `check_date`, and `needs_alert`.  
    - Input from "Wait For Response".  
    - Outputs to "Filter Significant Ranking Drops" and "Update Rankings in Google Sheet".  
    - Edge cases: No organic results, malformed API data, domain matching edge cases, missing previous rank data.

---

#### 2.5 Alert Filtering & Notifications

- **Overview:**  
Filters keywords with significant ranking drops and sends notifications to the SEO team via Slack and email for immediate action.

- **Nodes Involved:**  
  - Filter Significant Ranking Drops  
  - Send Slack Ranking Alert  
  - Send Email Ranking Alert

- **Node Details:**  
  - **Filter Significant Ranking Drops:**  
    - Type: Filter  
    - Passes only items where `needs_alert` is true (drops > 2 or disappeared).  
    - Input from ranking analysis node.  
    - Outputs to both Slack and Email alert nodes.  
    - Edge cases: Boolean flag misinterpretation, filtering logic errors.

  - **Send Slack Ranking Alert:**  
    - Type: Slack  
    - Sends formatted alert messages to a specific Slack channel.  
    - Message includes keyword, current and previous rank, ranking drop, URLs, status, and action recommendations.  
    - Uses Slack API credentials with OAuth2.  
    - Input from alert filter node.  
    - Edge cases: Slack API rate limits, invalid channel ID, authentication errors, message formatting issues.

  - **Send Email Ranking Alert:**  
    - Type: Email Send  
    - Sends detailed HTML-formatted emails to SEO team members.  
    - Includes ranking details, before/after comparisons, and investigation links.  
    - Uses SMTP credentials.  
    - Input from alert filter node.  
    - Edge cases: Email delivery failures, SMTP authentication errors, invalid email addresses.

---

#### 2.6 Data Update & Reporting

- **Overview:**  
Updates the Google Sheet with the latest ranking data to maintain historical records and generates a summary report of the day's monitoring results.

- **Nodes Involved:**  
  - Update Rankings in Google Sheet  
  - Generate SEO Monitoring Summary

- **Node Details:**  
  - **Update Rankings in Google Sheet:**  
    - Type: Google Sheets (Update Operation)  
    - Updates the sheet with current rankings, ranking changes, status, found URLs, and timestamps.  
    - Uses Service Account authentication.  
    - Input from ranking analysis node.  
    - Output to summary node.  
    - Edge cases: Concurrent sheet edits, API quota issues, data mapping errors.

  - **Generate SEO Monitoring Summary:**  
    - Type: Code (JavaScript)  
    - Aggregates all processed keyword data.  
    - Calculates total keywords processed, alerts sent, counts of improved/dropped/stable/not found keywords, and average ranking.  
    - Logs a detailed summary to console.  
    - Outputs an execution summary object.  
    - Input from update node and alert nodes.  
    - Edge cases: Empty input data, division by zero when calculating averages.

---

### 3. Summary Table

| Node Name                     | Node Type                | Functional Role                              | Input Node(s)               | Output Node(s)                          | Sticky Note                                                                                                           |
|-------------------------------|--------------------------|----------------------------------------------|-----------------------------|----------------------------------------|-----------------------------------------------------------------------------------------------------------------------|
| Daily SEO Check Trigger        | Cron Trigger             | Initiates workflow daily at 8 AM             | —                           | Get Keywords Database                  | 🕐 DAILY SCHEDULER: Runs daily at 8 AM for consistent monitoring                                                       |
| Get Keywords Database          | Google Sheets (Read)     | Retrieves keyword list and previous rankings | Daily SEO Check Trigger      | Filter Active Keywords Only             | 📊 KEYWORD DATABASE: Central source of keywords and ranking data                                                      |
| Filter Active Keywords Only    | Filter                   | Filters only active keywords                  | Get Keywords Database        | Fetch Google Rankings via SERP API     | 🔍 KEYWORD FILTER: Saves API calls by processing only active keywords                                                  |
| Fetch Google Rankings via SERP API | HTTP Request          | Fetches current Google rankings via SERP API | Filter Active Keywords Only  | Wait For Response                      | 🌐 GOOGLE RANKING FETCHER: Retrieves real-time keyword rankings                                                        |
| Wait For Response             | Wait                     | Ensures complete API response before next step | Fetch Google Rankings via SERP API | Parse Rankings & Detect Changes          |                                                                                                                       |
| Parse Rankings & Detect Changes| Code                     | Parses SERP API response and detects ranking changes | Wait For Response           | Filter Significant Ranking Drops, Update Rankings in Google Sheet | ⚙️ RANKING ANALYZER: Processes ranking data, detects drops/changes                                                   |
| Filter Significant Ranking Drops | Filter                 | Filters keywords with significant drops      | Parse Rankings & Detect Changes | Send Slack Ranking Alert, Send Email Ranking Alert | 🚨 ALERT FILTER: Alerts only for drops > 2 or missing in top 100                                                     |
| Send Slack Ranking Alert       | Slack                    | Sends Slack alerts to SEO team                | Filter Significant Ranking Drops | Generate SEO Monitoring Summary       | 💬 SLACK NOTIFICATION: Instant team alerts with detailed ranking info                                                 |
| Send Email Ranking Alert       | Email Send               | Sends email alerts to stakeholders            | Filter Significant Ranking Drops | Generate SEO Monitoring Summary       | 📧 EMAIL NOTIFICATION: Detailed email reports for management and clients                                              |
| Update Rankings in Google Sheet| Google Sheets (Update)   | Updates ranking data for history and tracking | Parse Rankings & Detect Changes | Generate SEO Monitoring Summary       | 📈 RANKING HISTORY TRACKER: Maintains audit trail and data for trend analysis                                         |
| Generate SEO Monitoring Summary| Code                     | Summarizes workflow execution and results     | Send Slack Ranking Alert, Send Email Ranking Alert, Update Rankings in Google Sheet | —                                      | 📊 EXECUTION SUMMARY: Logs daily SEO monitoring summary                                                              |
| Sticky Note                   | Sticky Note              | Workflow explanation and block overview       | —                           | —                                      | ## How it works: Explains workflow steps and logic                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Cron Trigger Node**  
   - Name: "Daily SEO Check Trigger"  
   - Type: Cron Trigger  
   - Set Cron Expression: `0 8 * * *` (Daily at 8 AM)  

2. **Add Google Sheets Node to read keywords**  
   - Name: "Get Keywords Database"  
   - Operation: Read  
   - Authentication: Service Account  
   - Document ID: `<your-google-sheet-id>`  
   - Sheet Name/ID: `<your-keywords-sheet-id>`  
   - Configure to read columns including `keyword`, `url`, `previous_rank`, `monitor_active`, etc.  
   - Connect output of "Daily SEO Check Trigger" to this node.

3. **Add Filter Node "Filter Active Keywords Only"**  
   - Filter conditions:  
     - `keyword` is not empty  
     - `monitor_active` equals `"true"` (case sensitive)  
   - Connect output of Get Keywords Database to this filter node.

4. **Add HTTP Request Node "Fetch Google Rankings via SERP API"**  
   - HTTP Method: GET  
   - URL: `https://serpapi.com/search`  
   - Query Parameters:  
     - `q`: `={{ $json.keyword }}`  
     - `location`: `United States`  
     - `hl`: `en`  
     - `gl`: `us`  
     - `google_domain`: `google.com`  
     - `api_key`: `YOUR_SERPAPI_KEY` (set your actual API key in credentials)  
     - `num`: `100`  
   - Authentication: HTTP Header Auth with your SERP API key  
   - Connect output of filter node to this HTTP Request node.

5. **Add Wait Node "Wait For Response"**  
   - No parameters needed (default)  
   - Connect output of HTTP Request node to this node.

6. **Add Code Node "Parse Rankings & Detect Changes"**  
   - Paste provided JavaScript code that:  
     - Iterates over SERP API results  
     - Matches target URL domain in organic results  
     - Calculates ranking change and status  
     - Flags if alert is needed  
   - Connect output of "Wait For Response" to this node.

7. **Add Filter Node "Filter Significant Ranking Drops"**  
   - Filter condition: `needs_alert` equals `true`  
   - Connect output of code node to this filter.

8. **Add Slack Node "Send Slack Ranking Alert"**  
   - Configure Slack API credentials (OAuth2)  
   - Channel: your alert channel ID  
   - Message text: use template with keyword, ranks, changes, URLs, status  
   - Connect filter node output to this Slack node.

9. **Add Email Send Node "Send Email Ranking Alert"**  
   - Configure SMTP credentials  
   - Set "To" to your SEO team email (e.g. seo-team@yourcompany.com)  
   - Set "From" email (e.g. seo-alerts@yourcompany.com)  
   - Subject: `🚨 SEO Alert: {{ $json.keyword }} ranking dropped`  
   - Body: Use HTML format with ranking details and recommendations  
   - Connect filter node output to this email node.

10. **Add Google Sheets Node "Update Rankings in Google Sheet"**  
    - Operation: Update  
    - Authentication: Service Account  
    - Document ID and Sheet Name for storing updated rankings  
    - Map columns to update current ranking, previous rank, status, URLs, date  
    - Connect output of "Parse Rankings & Detect Changes" to this update node.

11. **Add Code Node "Generate SEO Monitoring Summary"**  
    - Paste provided JavaScript code to:  
      - Aggregate processed keywords count, alerts, statuses, average rank  
      - Log a detailed summary  
    - Connect outputs of "Send Slack Ranking Alert", "Send Email Ranking Alert", and "Update Rankings in Google Sheet" to this summary node.

12. **Connect all nodes according to the following order:**  
    - Daily SEO Check Trigger → Get Keywords Database → Filter Active Keywords Only → Fetch Google Rankings via SERP API → Wait For Response → Parse Rankings & Detect Changes → (to two branches)  
      - 1) Filter Significant Ranking Drops → Send Slack Ranking Alert & Send Email Ranking Alert → Generate SEO Monitoring Summary  
      - 2) Update Rankings in Google Sheet → Generate SEO Monitoring Summary

13. **Test workflow execution** with sample data to verify all nodes execute as expected.

14. **Optional:** Add sticky notes or comments in n8n UI to clarify workflow steps.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                 | Context or Link                                                                                          |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Workflow automatically runs daily at 8 AM to maintain fresh SEO keyword ranking data and timely alerts.                                                                                                                                      | See "Daily SEO Check Trigger" node configuration                                                       |
| SERP API costs approximately $0.001 per search; monitor API usage to control expenses.                                                                                                                                                        | SERP API Pricing: https://serpapi.com/pricing                                                          |
| Slack alerts use rich formatting and can be customized to mention specific team members or channels.                                                                                                                                         | Slack API Documentation: https://api.slack.com/messaging                                               |
| Google Sheets integration uses service account authentication for secure, automated access without manual login.                                                                                                                            | Google Sheets API Docs: https://developers.google.com/sheets/api                                        |
| Email alerts provide an audit trail and can be forwarded to stakeholders who are not on Slack.                                                                                                                                               | Use SMTP credentials appropriate for your email provider                                               |
| The workflow is designed to handle cases where keywords drop out of the top 100 search results by flagging them for alerting and further investigation.                                                                                    | Ranking analysis code handles "not_found" status                                                       |
| To avoid alert fatigue, alerts are only sent for drops greater than 2 positions or disappearance, ignoring minor fluctuations.                                                                                                              | Filter Significant Ranking Drops node logic                                                           |
| The workflow logs detailed execution summaries to the n8n console for debugging and performance monitoring purposes.                                                                                                                        | See "Generate SEO Monitoring Summary" node                                                            |
| Modify cron expression in the trigger node to adjust monitoring frequency (e.g., multiple times per day).                                                                                                                                    | Cron syntax reference: https://crontab.guru                                                            |
| The sticky note in the workflow summarizes the process and explains each step clearly inside the n8n editor for collaborator clarity.                                                                                                       | Included in the workflow JSON as "Sticky Note" node                                                    |

---

**Disclaimer:** The provided workflow originates exclusively from an automated n8n workflow integration. It complies fully with current content policies and contains no illegal, offensive, or protected elements. All data processed is lawful and publicly accessible.