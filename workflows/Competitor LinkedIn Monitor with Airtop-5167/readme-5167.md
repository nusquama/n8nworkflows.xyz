Competitor LinkedIn Monitor with Airtop

https://n8nworkflows.xyz/workflows/competitor-linkedin-monitor-with-airtop-5167


# Competitor LinkedIn Monitor with Airtop

### 1. Workflow Overview

This workflow automates the monitoring of competitor LinkedIn activity and shares summarized insights on Slack. It is designed for organizations needing regular updates on competitors’ recent LinkedIn posts to stay informed about their strategies, topics of focus, and audience engagement.

The workflow’s logic is divided into the following blocks:

- **1.1 Scheduled Trigger:** Initiates the workflow on a weekly schedule.
- **1.2 Data Retrieval:** Fetches competitor LinkedIn profile URLs from a Google Sheet.
- **1.3 AI-Powered Post Analysis:** Uses Airtop to extract and summarize recent LinkedIn posts from competitors.
- **1.4 Notification Dispatch:** Sends the summarized insights to a Slack channel.
- **1.5 Documentation & User Guidance:** Provides setup instructions and use case information via sticky notes.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  Starts the workflow automatically once per week at 18:00 hours to ensure regular monitoring without manual intervention.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**  
  - **Name:** Schedule Trigger  
  - **Type:** Schedule Trigger  
  - **Configuration:** Set to trigger weekly at 18:00 (6 PM).  
  - **Inputs:** None (trigger node).  
  - **Outputs:** Connected to "Get Competitors" node.  
  - **Edge Cases:**  
    - Time zone discrepancies may cause unexpected trigger times if not aligned with server settings.  
    - If the node is disabled or misconfigured, the workflow will not run automatically.

#### 1.2 Data Retrieval

- **Overview:**  
  Retrieves the list of competitor LinkedIn profile URLs from a Google Sheets document, which acts as the source of truth for monitored profiles.

- **Nodes Involved:**  
  - Get Competitors

- **Node Details:**  
  - **Name:** Get Competitors  
  - **Type:** Google Sheets  
  - **Configuration:**  
    - Reads from the spreadsheet with ID `1TknyHS8ie0KONgF4-KFGR76I74egHLt7-M9sIxjQX`.  
    - Reads from the first sheet (`gid=0`).  
    - Uses OAuth2 credentials for authorized access.  
  - **Key Expressions:** None dynamic; static document and sheet reference.  
  - **Inputs:** From Schedule Trigger.  
  - **Outputs:** To Analyze Posts node.  
  - **Edge Cases:**  
    - OAuth token expiry or permission issues may lead to failures accessing the sheet.  
    - Changes in sheet structure (e.g., columns renamed or removed) may cause data retrieval errors.  
    - Empty or malformed data rows in the sheet can cause downstream failures.

#### 1.3 AI-Powered Post Analysis

- **Overview:**  
  This block processes each competitor’s LinkedIn post URLs using Airtop to extract up to 5 recent posts (published within the last week), summarize them, and highlight key insights.

- **Nodes Involved:**  
  - Analyze Posts

- **Node Details:**  
  - **Name:** Analyze Posts  
  - **Type:** Airtop node (browser automation + AI processing)  
  - **Configuration:**  
    - Uses the URL field from the Google Sheet data (`$json['Post lists to watch']`) to browse LinkedIn posts.  
    - Airtop profile name must be entered manually for browser authentication to LinkedIn.  
    - The prompt instructs extraction of up to 5 posts no older than 1 week, summarizing post count, main topics, and engagement levels.  
    - Session mode is set to "new" for each run.  
    - Uses Airtop API credentials for authentication.  
  - **Inputs:** From Get Competitors node.  
  - **Outputs:** To Send Summary node.  
  - **Edge Cases:**  
    - Missing or incorrect Airtop profile name will cause authentication or navigation failures.  
    - Network or LinkedIn layout changes may cause scraping errors.  
    - If no posts meet the criteria (e.g., no posts in last week), summaries may be empty or misleading.  
    - Airtop API rate limits or downtime could interrupt processing.

#### 1.4 Notification Dispatch

- **Overview:**  
  Sends the summarized LinkedIn post insights to a specific Slack channel to keep teams informed.

- **Nodes Involved:**  
  - Send Summary

- **Node Details:**  
  - **Name:** Send Summary  
  - **Type:** Slack node (message sending)  
  - **Configuration:**  
    - Sends to Slack channel with ID `C092P1RRX4L` (named "competitive-monitoring").  
    - Message text dynamically composes a header with competitor name and the AI-generated summary.  
    - Uses markdown formatting and displays an emoji icon (`❗`).  
    - Requires Slack Bot credentials with appropriate channel permissions.  
  - **Inputs:** From Analyze Posts node.  
  - **Outputs:** None (end node).  
  - **Edge Cases:**  
    - Slack API authentication failure or revoked token will prevent message delivery.  
    - Channel ID mismatch or bot lacking channel membership will cause errors.  
    - Message length or markdown formatting issues could cause delivery failure.

#### 1.5 Documentation & User Guidance

- **Overview:**  
  Provides instructions, setup guidance, and use case explanation directly in the workflow interface via sticky notes.

- **Nodes Involved:**  
  - Sticky Note (near Analyze Posts)  
  - Sticky Note1 (large readme note)

- **Node Details:**  
  - **Name:** Sticky Note  
    - Provides a link and reminder to enter the Airtop Profile authenticated to LinkedIn.  
  - **Name:** Sticky Note1  
    - Contains a detailed README covering use case, automation steps, setup requirements, and suggestions for future enhancements.  
  - **Type:** Sticky Note  
  - **Configuration:** Text content with markdown format and multiple URLs for references.  
  - **Inputs/Outputs:** None (informational only).  
  - **Edge Cases:** Not applicable.

---

### 3. Summary Table

| Node Name       | Node Type           | Functional Role                 | Input Node(s)       | Output Node(s)   | Sticky Note                                                                                             |
|-----------------|---------------------|--------------------------------|---------------------|------------------|-------------------------------------------------------------------------------------------------------|
| Schedule Trigger| Schedule Trigger     | Starts workflow weekly          | None                | Get Competitors  |                                                                                                       |
| Get Competitors | Google Sheets       | Retrieves competitor URLs       | Schedule Trigger    | Analyze Posts    |                                                                                                       |
| Analyze Posts   | Airtop              | Extracts and summarizes posts  | Get Competitors     | Send Summary     | "Enter [Airtop Profile](https://portal.airtop.ai/browser-profiles) authenticated to LinkedIn in the 'Profile Name' field"|
| Send Summary    | Slack               | Sends summary to Slack channel  | Analyze Posts       | None             |                                                                                                       |
| Sticky Note     | Sticky Note          | Reminder about Airtop Profile  | None                | None             | "## Enter [Airtop Profile](https://portal.airtop.ai/browser-profiles) authenticated to LinkedIn in the 'Profile Name' field"|
| Sticky Note1    | Sticky Note          | Detailed README & setup guide  | None                | None             | Comprehensive README with use case, setup links, and next steps. See https://portal.airtop.ai/browser-profiles and Google Sheet link.|

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger:**  
   - Add a "Schedule Trigger" node.  
   - Set it to trigger weekly at 18:00 (6 PM).  
   - No credentials required.

2. **Add Google Sheets Node (Get Competitors):**  
   - Create a "Google Sheets" node.  
   - Set "Operation" to "Read Rows".  
   - Enter Spreadsheet ID: `1TknyHS8ie0KONgF4-KFGR76I74egHLt7-M9sIxjQX`.  
   - Select Sheet Name or GID: `gid=0`.  
   - Configure OAuth2 credentials with access to the Google Sheet.  
   - Connect Schedule Trigger's output to this node’s input.

3. **Add Airtop Node (Analyze Posts):**  
   - Create an "Airtop" node.  
   - Choose the "extraction" resource.  
   - Set the URL parameter to `={{ $json['Post lists to watch'] }}` to dynamically use URLs from Google Sheets data.  
   - Enter the prompt text:  
     ```
     This is a list of posts.

     Perform the following steps:
     1. Extract the text of up to 5 posts that were published no more than 1 week ago.
     2. Summarize the selected posts and highlight:
        Number of posts published in the last week
        Main topics covered in the posts
        Level of user engagement with the posts
     ```  
   - Enter the Airtop Profile Name authenticated to LinkedIn (must be pre-created on Airtop portal).  
   - Set session mode to "new".  
   - Use Airtop API credentials.  
   - Connect Get Competitors node output to this node input.

4. **Add Slack Node (Send Summary):**  
   - Create a "Slack" node.  
   - Set operation to "Send Message".  
   - Select or enter Slack channel ID: `C092P1RRX4L` (ensure the bot has permission).  
   - Compose message text:  
     ```
     =SUMMARY OF POSTS - {{ $('Get Competitors').item.json['Name and Title '] }}
     {{ $json.data.modelResponse }}
     ```  
   - Enable Markdown formatting.  
   - Set bot profile icon emoji to `❗`.  
   - Connect Analyze Posts node output to this node input.  
   - Configure Slack OAuth2 credentials for the bot user.

5. **Add Sticky Notes for Documentation:**  
   - Add a sticky note near the Airtop node reminding users to enter the Airtop Profile authenticated to LinkedIn, linking to https://portal.airtop.ai/browser-profiles.  
   - Add a large sticky note anywhere convenient with the detailed README content including setup instructions, use cases, and helpful links.

6. **Connect Nodes in Sequence:**  
   - Schedule Trigger → Get Competitors → Analyze Posts → Send Summary.

7. **Verify Credentials:**  
   - Ensure Google Sheets OAuth2 credentials have read access to the specified document.  
   - Ensure Airtop API credentials and profile exist and are valid.  
   - Validate Slack OAuth2 credentials have permission to post in target channel.

8. **Test the Workflow:**  
   - Run manually or wait for scheduled trigger to verify end-to-end execution.  
   - Check Slack channel for message delivery.  
   - Troubleshoot any API errors, credential issues, or data mismatches.

---

### 5. General Notes & Resources

| Note Content                                                                                                                         | Context or Link                                                                                           |
|--------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| Airtop Profile must be authenticated to LinkedIn for browser automation to work properly.                                            | https://portal.airtop.ai/browser-profiles                                                                |
| Google Sheet template provided contains competitor LinkedIn profile URLs to monitor.                                                 | https://docs.google.com/spreadsheets/d/1TknyHS8ie0KONgF4-KFGR76I74egHLt7-M9sIxjQXGY/edit?usp=sharing       |
| Slack bot requires channel membership and messaging permissions to post updates.                                                    | Slack API documentation                                                                                   |
| README sticky note includes guidance on expanding monitoring and integrating insights with CRM tools.                                | Visible in workflow’s Sticky Note1                                                                        |
| Automating LinkedIn competitive monitoring saves time and provides structured competitor insights regularly to teams.                | Summary based on workflow objective                                                                       |

---

**Disclaimer:**  
The provided content is extracted exclusively from an n8n automated workflow. It complies fully with content policies and does not contain any illegal, offensive, or protected elements. All data handled is legal and publicly accessible.