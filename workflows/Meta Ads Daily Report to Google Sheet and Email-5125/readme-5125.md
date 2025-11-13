Meta Ads Daily Report to Google Sheet and Email

https://n8nworkflows.xyz/workflows/meta-ads-daily-report-to-google-sheet-and-email-5125


# Meta Ads Daily Report to Google Sheet and Email

### 1. Workflow Overview

This workflow automates the process of fetching daily Meta (Facebook) Ads campaign performance data, storing it in a Google Sheet, and emailing a formatted report each day at 9 AM. It is designed for digital marketers or analysts who want to automate daily retrieval, archival, and reporting of advertising metrics without manual intervention.

Logical Blocks:

- **1.1 Scheduled Trigger and Date Setup:** Automatically triggers the workflow daily at 9 AM and calculates the date for "yesterday" to use as the reporting period.
- **1.2 Data Retrieval from Meta Ads API:** Fetches campaign-level insights for the previous day using the Facebook Graph API.
- **1.3 Data Transformation:** Processes raw API data into a clean, structured format suitable for Google Sheets and email presentation.
- **1.4 Data Storage:** Appends the transformed data to a specific Google Sheet, maintaining a daily log.
- **1.5 Email Report Generation and Sending:** Creates an HTML table summary of the report data and sends it via Gmail.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger and Date Setup

**Overview:**  
This block schedules the workflow to run daily at 9 AM and computes the date string for the previous day ("yesterday"), which is used as the time range for fetching ads data.

**Nodes Involved:**  
- Trigger - Daily at 9AM  
- Set Yesterday's Date

**Node Details:**

- **Trigger - Daily at 9AM**  
  - Type: Schedule Trigger  
  - Configuration: Set to trigger once daily at 9:00 AM local time.  
  - Inputs: None (entry point)  
  - Outputs: Connects to "Set Yesterday's Date"  
  - Failures: Possible scheduler misconfiguration or timezone mismatch.  
  - Version: 1.2

- **Set Yesterday's Date**  
  - Type: Set Node  
  - Configuration: Uses a JavaScript expression to calculate yesterday’s date in ISO format (YYYY-MM-DD).  
  - Expression:  
    ```js
    (() => {   
      const date = new Date();   
      date.setDate(date.getDate() - 1);   
      return date.toISOString().split('T')[0]; 
    })()
    ```  
  - Outputs a JSON property named `yesterday` for downstream use.  
  - Inputs: From the schedule trigger  
  - Outputs: Connects to "Fetch Ads Data"  
  - Failures: If system time is incorrect, the date calculation will be off.  
  - Version: 3.4

#### 1.2 Data Retrieval from Meta Ads API

**Overview:**  
This block calls the Facebook Graph API to fetch campaign-level insights for the date computed previously, specifically the prior day's metrics.

**Nodes Involved:**  
- Fetch Ads Data

**Node Details:**

- **Fetch Ads Data**  
  - Type: HTTP Request  
  - Configuration:  
    - URL: Facebook Graph API endpoint for ads insights at campaign level.  
    - Query Parameters:  
      - `fields`: campaign_name, spend, results, cost_per_result, cpm  
      - `level`: campaign  
      - `time_range`: JSON string with since and until set to `yesterday` date from previous node.  
    - Authentication: Uses predefined Facebook Graph API credentials.  
  - Inputs: From "Set Yesterday's Date" node, using `yesterday` value as query parameter.  
  - Outputs: Raw JSON response containing campaign insights.  
  - Failures:  
    - Authentication errors (expired token, invalid credentials).  
    - API rate limits or network timeouts.  
    - Invalid Ad Account ID placeholder (`<Ad_Account_ID>` must be replaced).  
  - Version: 4.2

#### 1.3 Data Transformation

**Overview:**  
Transforms the raw API response into a structured, flat JSON object with only the required fields and handles missing or nested data.

**Nodes Involved:**  
- Transform Ads Data

**Node Details:**

- **Transform Ads Data**  
  - Type: Code (JavaScript)  
  - Configuration:  
    - Extracts `data` array from API response.  
    - Maps each campaign's data into an object with keys: date, campaign_name, spend, results, cost_per_result, cpm.  
    - Uses safe navigation and default values ('0' or 'N/A') when fields are missing.  
    - Example snippet:  
      ```js
      const rawData = $json.data || [];
      return rawData.map(campaign => {
        const results = campaign.results?.[0]?.values?.[0]?.value || '0';
        const costPerResult = campaign.cost_per_result?.[0]?.values?.[0]?.value || '0';
        return {
          json: {
            date: campaign.date_start || '',
            campaign_name: campaign.campaign_name || 'N/A',
            spend: campaign.spend || '0',
            results,
            cost_per_result: costPerResult,
            cpm: campaign.cpm || '0'
          }
        };
      });
      ```  
  - Inputs: From "Fetch Ads Data" node  
  - Outputs: Processed array of campaign report objects  
  - Failures: Expression errors if raw data format changes; missing expected nested properties.  
  - Version: 2

#### 1.4 Data Storage

**Overview:**  
Appends the transformed data to a Google Sheet, preserving a daily record of campaign performance metrics.

**Nodes Involved:**  
- Update Google Sheet

**Node Details:**

- **Update Google Sheet**  
  - Type: Google Sheets  
  - Configuration:  
    - Operation: append rows  
    - Sheet: Sheet1 (gid=0) of the Google Sheet with document ID `1tnd0mehy0ZEWwmVExDAu3danPPt1YZKe1SSdh-0ZjFg`  
    - Columns mapped explicitly: Date, Campaign Name, Spend, Results, Cost/Result, CPM  
    - Mapping mode: defineBelow (explicit column mapping)  
  - Credentials: OAuth2 credential for Google Sheets  
  - Inputs: From "Transform Ads Data" node  
  - Outputs: None (terminal operation)  
  - Failures:  
    - Authentication errors with Google OAuth token.  
    - API quota or rate limits.  
    - Sheet ID or permissions misconfiguration.  
  - Version: 4.6

#### 1.5 Email Report Generation and Sending

**Overview:**  
Generates an HTML email body summarizing the same campaign data in a tabular format and sends it via Gmail to recipients.

**Nodes Involved:**  
- Generate Email HTML  
- Send Email Report

**Node Details:**

- **Generate Email HTML**  
  - Type: Code (JavaScript)  
  - Configuration:  
    - Iterates over all input items to build HTML table rows.  
    - Constructs a complete HTML string including table headers and campaign data rows.  
    - Uses fallback values for missing data.  
  - Inputs: From "Transform Ads Data" node  
  - Outputs: JSON with `htmlBody` property containing the email content  
  - Failures: HTML formatting issues if unexpected data occurs.  
  - Version: 2

- **Send Email Report**  
  - Type: Gmail  
  - Configuration:  
    - Uses OAuth2 credentials for Gmail  
    - Subject: "Facebook Ads Report for Yesterday"  
    - Message body: uses the `htmlBody` from previous node  
  - Inputs: From "Generate Email HTML" node  
  - Outputs: None (final action)  
  - Failures: Authentication errors, Gmail API quotas, invalid recipient configuration.  
  - Version: 2.1

---

### 3. Summary Table

| Node Name              | Node Type         | Functional Role                   | Input Node(s)          | Output Node(s)                   | Sticky Note                                      |
|------------------------|-------------------|---------------------------------|-----------------------|---------------------------------|-------------------------------------------------|
| Trigger - Daily at 9AM | Schedule Trigger  | Initiates workflow daily at 9AM | None                  | Set Yesterday's Date            |                                                 |
| Set Yesterday's Date    | Set               | Calculates previous day's date   | Trigger - Daily at 9AM | Fetch Ads Data                  |                                                 |
| Fetch Ads Data          | HTTP Request      | Retrieves Meta Ads insights      | Set Yesterday's Date   | Transform Ads Data              | Replace `<Ad_Account_ID>` with your actual ID.  |
| Transform Ads Data      | Code              | Formats raw API data             | Fetch Ads Data         | Update Google Sheet, Generate Email HTML |                                              |
| Update Google Sheet     | Google Sheets     | Appends data to Google Sheet    | Transform Ads Data     | None                           | Ensure Google Sheets OAuth2 credential is valid. |
| Generate Email HTML     | Code              | Creates HTML email body          | Transform Ads Data     | Send Email Report              |                                                 |
| Send Email Report       | Gmail             | Sends the email report           | Generate Email HTML    | None                           | Configure Gmail OAuth2 credentials properly.    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Name: "Trigger - Daily at 9AM"  
   - Type: Schedule Trigger  
   - Set to run daily at 9:00 AM local time.

2. **Add a Set node to calculate yesterday’s date:**  
   - Name: "Set Yesterday's Date"  
   - Add a string field named `yesterday` with the following expression:  
     ```js
     (() => {   
       const date = new Date();   
       date.setDate(date.getDate() - 1);   
       return date.toISOString().split('T')[0]; 
     })()
     ```  
   - Connect "Trigger - Daily at 9AM" output to this node.

3. **Add an HTTP Request node to fetch Meta Ads data:**  
   - Name: "Fetch Ads Data"  
   - Set HTTP Method to GET  
   - URL: `https://graph.facebook.com/v18.0/act_<Ad_Account_ID>/insights` (replace `<Ad_Account_ID>` with your actual account ID)  
   - Add Query Parameters:  
     - `fields`: `campaign_name,spend,results,cost_per_result,cpm`  
     - `level`: `campaign`  
     - `time_range`: `={"since":"{{$json.yesterday}}","until":"{{$json.yesterday}}"}` (use expression referencing `yesterday`)  
   - Authentication: Use Facebook Graph API OAuth2 credentials  
   - Connect "Set Yesterday's Date" output to this node.

4. **Add a Code node to transform raw data:**  
   - Name: "Transform Ads Data"  
   - Use the following JavaScript code:  
     ```js
     const rawData = $json.data || [];
     return rawData.map(campaign => {
       const results = campaign.results?.[0]?.values?.[0]?.value || '0';
       const costPerResult = campaign.cost_per_result?.[0]?.values?.[0]?.value || '0';
       return {
         json: {
           date: campaign.date_start || '',
           campaign_name: campaign.campaign_name || 'N/A',
           spend: campaign.spend || '0',
           results,
           cost_per_result: costPerResult,
           cpm: campaign.cpm || '0'
         }
       };
     });
     ```  
   - Connect output of "Fetch Ads Data" here.

5. **Add a Google Sheets node to append data:**  
   - Name: "Update Google Sheet"  
   - Operation: Append  
   - Document ID: Your Google Sheet ID (e.g., `1tnd0mehy0ZEWwmVExDAu3danPPt1YZKe1SSdh-0ZjFg`)  
   - Sheet Name: Use `gid=0` or actual sheet name (e.g., "Sheet1")  
   - Define columns explicitly: Date, Campaign Name, Spend, Results, Cost/Result, CPM  
   - Map each column to corresponding JSON keys from the code node output.  
   - Use Google Sheets OAuth2 credentials with write access.  
   - Connect from "Transform Ads Data".

6. **Add a Code node to generate HTML for the email:**  
   - Name: "Generate Email HTML"  
   - JavaScript code:  
     ```js
     const items = $input.all();
     const rows = items.map(item => {
       const data = item.json;
       return `
         <tr>
           <td>${data.campaign_name || '-'}</td>
           <td>${data.spend || '0'}</td>
           <td>${data.results || '0'}</td>
           <td>${data.cost_per_result || '0'}</td>
           <td>${data.cpm || '0'}</td>
         </tr>`;
     }).join('');
     return [{
       json: {
         htmlBody: `
           <h3>Facebook Ads Report</h3>
           <table border="1" cellpadding="5" cellspacing="0">
             <thead>
               <tr>
                 <th>Campaign</th><th>Spend</th><th>Results</th><th>Cost/Result</th><th>CPM</th>
               </tr>
             </thead>
             <tbody>${rows}</tbody>
           </table>
         `
       }
     }];
     ```  
   - Connect from "Transform Ads Data".

7. **Add a Gmail node to send the email:**  
   - Name: "Send Email Report"  
   - Use Gmail OAuth2 credentials with send mail permission  
   - Subject: "Facebook Ads Report for Yesterday"  
   - Message: Use the expression `{{$json.htmlBody}}` to send the generated HTML body  
   - Connect from "Generate Email HTML".

8. **Establish connections:**  
   - "Trigger - Daily at 9AM" → "Set Yesterday's Date"  
   - "Set Yesterday's Date" → "Fetch Ads Data"  
   - "Fetch Ads Data" → "Transform Ads Data"  
   - "Transform Ads Data" → "Update Google Sheet"  
   - "Transform Ads Data" → "Generate Email HTML"  
   - "Generate Email HTML" → "Send Email Report"

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                                                                       |
|------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| Replace `<Ad_Account_ID>` in the Facebook API URL with your actual Ads account ID.             | Important for correct API requests.                                                                 |
| Ensure Google Sheets OAuth2 credentials have write permissions to the target spreadsheet.      | OAuth2 setup in n8n credentials management.                                                         |
| Gmail OAuth2 credentials must have "Send Email" permission enabled and be authorized properly. | https://developers.google.com/gmail/api/auth/about-auth                                            |
| Facebook Graph API may apply rate limits; consider error handling or retry mechanisms if needed.| Facebook Graph API documentation: https://developers.facebook.com/docs/graph-api                   |
| The date calculation uses system time; timezone differences may affect "yesterday" definition. | Adjust system timezone or modify the date logic if required for your locale.                         |
| The Google Sheet ID and sheet name (gid) must match your actual spreadsheet.                   | Found in the Google Sheets URL after `/d/` and `#gid=` respectively.                                |

---

This detailed description and stepwise reconstruction guide will allow users and automated agents to fully understand, reproduce, and maintain this Meta Ads daily reporting workflow using n8n.