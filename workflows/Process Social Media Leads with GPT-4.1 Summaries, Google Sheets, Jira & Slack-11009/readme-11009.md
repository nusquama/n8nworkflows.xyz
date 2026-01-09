Process Social Media Leads with GPT-4.1 Summaries, Google Sheets, Jira & Slack

https://n8nworkflows.xyz/workflows/process-social-media-leads-with-gpt-4-1-summaries--google-sheets--jira---slack-11009


# Process Social Media Leads with GPT-4.1 Summaries, Google Sheets, Jira & Slack

### 1. Workflow Overview

This workflow automates the processing of social media marketing leads by integrating AI summarization, data storage, task management, and team notifications. It targets marketing teams and digital agencies who want to capture, classify, and track inbound social media inquiries related to ads, promotions, partnerships, and collaborations efficiently.

The workflow is logically divided into two main functional blocks:

- **1.1 Lead Reception and Processing:**  
  Captures incoming social media messages via webhook, filters relevant leads based on keywords, enriches the message using GPT-4.1 to classify and summarize the lead, and stores the results into Google Sheets.

- **1.2 Task Creation and Notifications:**  
  Creates Jira tasks for team follow-up based on AI-processed leads and sends Slack notifications alerting the team about new leads.

Additionally, there is a **1.3 Scheduled Reporting Block** that periodically retrieves stored leads from Google Sheets, filters them by date, summarizes key metrics, and distributes weekly reports via Slack.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Lead Reception and Processing

**Overview:**  
This block listens for incoming social media messages (via webhook), filters messages containing marketing-related keywords, uses OpenAI GPT-4.1 to generate a structured summary and classification, then parses and prepares the AI output for storage.

**Nodes Involved:**  
- Get DM (Webhook)  
- Lead Keyword Filter (Code)  
- AI Lead Classifier (OpenAI)  
- AI Output Parser (Code)  
- Sticky Note2 (Documentation)

**Node Details:**

- **Get DM**  
  - Type: Webhook  
  - Role: Entry point to receive POST requests with social media messages (e.g., DMs).  
  - Config: Path "marketing-lead", POST method, no additional options.  
  - Inputs: External HTTP POST request  
  - Outputs: JSON array of incoming messages  
  - Potential Failures: Webhook misconfiguration, malformed JSON payloads, unexpected input structures.

- **Lead Keyword Filter**  
  - Type: Code (JavaScript)  
  - Role: Filters incoming messages, retaining only those containing relevant marketing keywords like "social media", "ad request", "partnership".  
  - Config: Static list of keywords; converts messages to lowercase for case-insensitive matching.  
  - Key Variables: Filters input array, outputs array of cleaned lead objects with username, source, message, and receivedAt timestamp.  
  - Inputs: Data from Get DM node  
  - Outputs: Filtered and cleaned lead items  
  - Edge Cases: Empty input or input not an array; items missing message field; no matches found resulting in empty output.

- **AI Lead Classifier**  
  - Type: OpenAI (Language Model, GPT-4.1)  
  - Role: Processes each filtered lead message to generate a JSON response with a one-sentence summary and a categorized lead type.  
  - Config: Model set to GPT-4.1, prompt instructs to return only JSON with "summary" and "category" fields.  
  - Key Expression: Injects message text from Lead Keyword Filter node into prompt dynamically.  
  - Inputs: Filtered leads with messages  
  - Outputs: Raw GPT-4.1 output (JSON text in string format)  
  - Failures: API authentication or quota errors, malformed AI output, unexpected response format.

- **AI Output Parser**  
  - Type: Code (JavaScript)  
  - Role: Parses the AI JSON output string into usable JSON objects; merges with original metadata (username, source, timestamp).  
  - Config: Parses string response using JSON.parse, constructs new JSON objects with summary, category, username, source, and current timestamp.  
  - Inputs: AI Lead Classifier output  
  - Outputs: Cleaned lead JSON objects ready for storage and downstream processing  
  - Edge Cases: JSON parse errors if AI response is malformed, missing fields.

- **Sticky Note2**  
  - Role: Documentation note explaining the Get DM node’s function as the webhook listener for new social media messages.

---

#### 2.2 Task Creation and Notifications

**Overview:**  
After processing and storing lead information, this block creates Jira tasks for team follow-up and sends Slack notifications to alert the marketing team of new leads.

**Nodes Involved:**  
- Store Lead (Google Sheets)  
- Create Task (Jira)  
- Send a Summary (Slack)  
- Sticky Note3 (Jira task creation explanation)  
- Sticky Note4 (Slack notification explanation)

**Node Details:**

- **Store Lead**  
  - Type: Google Sheets  
  - Role: Appends processed lead data to a Google Sheet for record keeping.  
  - Config: Targets specific Google Sheet by document ID and sheet gid=0; auto-mapping of fields including source, category, username, receivedAt timestamp.  
  - Inputs: Parsed lead data from AI Output Parser  
  - Outputs: Confirmation of data appended  
  - Failures: OAuth credential issues, Google API rate limits, sheet access permissions.

- **Create Task**  
  - Type: Jira  
  - Role: Automatically creates a Jira task in a configured project with summary and description fields populated from lead data.  
  - Config: Project ID 10000, issue type "Story" (ID 10004), description includes AI summary, timestamp, and category.  
  - Inputs: Stored lead data from Store Lead node  
  - Outputs: Jira issue creation confirmation  
  - Failures: Jira API auth failures, invalid project or issue type IDs, network errors.

- **Send a Summary**  
  - Type: Slack  
  - Role: Sends a formatted message to a Slack channel notifying the team of the new marketing lead with key details.  
  - Config: Posts to channel ID "C09S57E2JQ2" with message template including username, summary, and timestamp.  
  - Inputs: Output from Create Task node  
  - Outputs: Slack API confirmation  
  - Failures: Slack authentication, channel permissions, API limits.

- **Sticky Note3**  
  - Explains the automatic creation of Jira tasks using cleaned data.

- **Sticky Note4**  
  - Explains the Slack notification node’s role in alerting the team.

---

#### 2.3 Scheduled Reporting Block

**Overview:**  
This block runs on a schedule to extract lead data from Google Sheets, filter leads received within the last day, aggregate statistics, and send a weekly summary report to Slack.

**Nodes Involved:**  
- Schedule Trigger  
- Extrect Lead Data (Google Sheets)  
- Weekly lead Filter (Code)  
- Report Data Formatter (Code)  
- Weekly Report Slack (Slack)  
- Sticky Note5 (Lead cleaning and saving process explanation)  
- Sticky Note (Schedule and reporting overview)

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Triggers workflow execution periodically (default daily, configured interval unspecified).  
  - Inputs: None  
  - Outputs: Trigger signal to start data extraction

- **Extrect Lead Data**  
  - Type: Google Sheets  
  - Role: Reads all lead entries from Google Sheets for reporting purposes.  
  - Config: Reads from same document and sheet as Store Lead.  
  - Inputs: Trigger from Schedule Trigger  
  - Outputs: Array of all stored lead records  
  - Failures: Google Sheet read errors, credential issues.

- **Weekly lead Filter**  
  - Type: Code (JavaScript)  
  - Role: Filters the retrieved lead records to only include those received during the current day (midnight to midnight).  
  - Config: Compares lead timestamp or receivedAt date fields with current date bounds.  
  - Inputs: Lead data from Extrect Lead Data  
  - Outputs: Leads filtered by date range  
  - Edge Cases: Missing or malformed date fields, timezone considerations.

- **Report Data Formatter**  
  - Type: Code (JavaScript)  
  - Role: Aggregates metrics such as total leads, counts by source and category, and selects sample lead data for the report.  
  - Inputs: Filtered leads from Weekly lead Filter  
  - Outputs: JSON object with summary statistics

- **Weekly Report Slack**  
  - Type: Slack  
  - Role: Posts the weekly lead report summary to the same Slack channel as new lead notifications.  
  - Config: Message template includes total leads, breakdown by source and category.  
  - Inputs: Aggregated report data from Report Data Formatter  
  - Outputs: Slack message confirmation  
  - Failures: Slack API or permission issues.

- **Sticky Note5**  
  - Describes the lead cleaning and saving steps related to Google Sheets.

- **Sticky Note**  
  - Summarizes the scheduled reporting nodes and their roles.

---

### 3. Summary Table

| Node Name           | Node Type                  | Functional Role                          | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                  |
|---------------------|----------------------------|----------------------------------------|-------------------------|-------------------------|----------------------------------------------------------------------------------------------|
| Get DM              | Webhook                    | Entry point for incoming social media leads | —                       | Lead Keyword Filter      | This node listens for new social media messages and starts the workflow whenever a new DM arrives. |
| Lead Keyword Filter  | Code                       | Filters messages by marketing keywords | Get DM                  | AI Lead Classifier       | ## Lead Cleaning and Saving Process: These steps check the message for important keywords, use AI to understand what the lead wants, pull out the useful details, and then save everything into your Google Sheet. |
| AI Lead Classifier   | OpenAI GPT-4.1             | Generates summary and category for leads | Lead Keyword Filter      | AI Output Parser         |                                                                                              |
| AI Output Parser     | Code                       | Parses AI JSON output, merges metadata | AI Lead Classifier       | Store Lead               |                                                                                              |
| Store Lead           | Google Sheets              | Appends processed lead data to sheet  | AI Output Parser         | Create Task              |                                                                                              |
| Create Task          | Jira                       | Creates Jira issue for lead follow-up | Store Lead               | Send a Summary           | This node automatically creates a follow-up task in Jira for your team using the cleaned message and summary. |
| Send a Summary       | Slack                      | Sends Slack notification for new lead | Create Task              | —                       | This node sends a simple lead notification to Slack so your team instantly knows a new lead has arrived. |
| Schedule Trigger     | Schedule Trigger           | Triggers scheduled reporting workflow  | —                       | Extrect Lead Data        | **Schedule Trigger:** Triggers the weekly reporting workflow on a schedule.                  |
| Extrect Lead Data    | Google Sheets              | Reads all leads from sheet for reports | Schedule Trigger         | Weekly lead Filter       |                                                                                              |
| Weekly lead Filter   | Code                       | Filters sheet data for leads from current day | Extrect Lead Data        | Report Data Formatter    |                                                                                              |
| Report Data Formatter| Code                       | Aggregates lead counts and example data | Weekly lead Filter       | Weekly Report Slack      |                                                                                              |
| Weekly Report Slack  | Slack                      | Sends weekly summary report to Slack  | Report Data Formatter    | —                       |                                                                                              |
| Sticky Note          | Sticky Note                | Documentation for schedule/reporting  | —                       | —                       | **Schedule Trigger:** Triggers the weekly reporting workflow on a schedule. **Extrect Lead Data:** Fetches all logged leads from the Google Sheet for reporting. **Weekly lead Filter:** Filters the sheet data to include leads from a Week. **Report Data Formatter:** Calculates total leads, category counts, source counts, and sample data. **Weekly Report Slack:** Sends the weekly report to Slack. |
| Sticky Note1         | Sticky Note                | Workflow overview and setup instructions | —                       | —                       | ## How It Works: This workflow collects messages from social media, filters, summarizes with AI, stores, creates Jira tasks, and sends Slack alerts. Setup steps listed. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node ("Get DM")**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: "marketing-lead"  
   - Purpose: Receive social media inbound messages as JSON payload.

2. **Add Code Node ("Lead Keyword Filter")**  
   - Type: Code  
   - JavaScript: Filter incoming messages array for keywords ["social media", "send info", "ad request", "promo request", "collaboration", "partnership", "work together"].  
   - Output: Clean array of lead objects with username, source, message, receivedAt timestamp.

3. **Add OpenAI Node ("AI Lead Classifier")**  
   - Credentials: OpenAI API key with GPT-4.1 access  
   - Model: GPT-4.1  
   - Prompt: Return JSON with fields summary and category (one of Sales, Support, Partnership, Influencer Inquiry, General Lead) based on message content from Lead Keyword Filter.  
   - Input: message field from filtered leads.

4. **Add Code Node ("AI Output Parser")**  
   - JavaScript: Parse AI JSON string output, merge with username, source, timestamp from Lead Keyword Filter.  
   - Output: Clean lead JSON objects.

5. **Add Google Sheets Node ("Store Lead")**  
   - Credentials: Google OAuth2 with access to target spreadsheet  
   - Operation: Append row to sheet (gid=0) in spreadsheet by ID  
   - Mapped fields: username, source, summary, category, receivedAt timestamp.

6. **Add Jira Node ("Create Task")**  
   - Credentials: Jira Cloud API OAuth or API token  
   - Project: Select appropriate project (ID 10000 in original)  
   - Issue Type: Story (ID 10004)  
   - Summary: "New lead from: {{username}}"  
   - Description: Include AI summary, received timestamp, category.

7. **Add Slack Node ("Send a Summary")**  
   - Credentials: Slack OAuth token with chat:write scope  
   - Channel: Select marketing team channel (e.g., C09S57E2JQ2)  
   - Message: Template including username, summary, receivedAt timestamp.

8. **Connect Nodes Sequentially:**  
   Get DM → Lead Keyword Filter → AI Lead Classifier → AI Output Parser → Store Lead → Create Task → Send a Summary

9. **Scheduled Reporting Setup:**  
   - Add Schedule Trigger node (e.g., daily or weekly interval).  
   - Add Google Sheets node ("Extrect Lead Data") configured to read all rows from the same spreadsheet and sheet used in Store Lead.  
   - Add Code node ("Weekly lead Filter") to filter leads by today's date using timestamp or receivedAt field.  
   - Add Code node ("Report Data Formatter") to aggregate total leads, counts by source and category, and select first 5 leads as examples.  
   - Add Slack node ("Weekly Report Slack") to send formatted report message to the Slack marketing channel.

10. **Connect Scheduled Nodes:**  
    Schedule Trigger → Extrect Lead Data → Weekly lead Filter → Report Data Formatter → Weekly Report Slack

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                  | Context or Link                                                                                                                   |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| This workflow collects messages from social media or the website, filters for marketing-relevant inquiries, uses AI to summarize and categorize leads, saves data in Google Sheets, creates Jira tasks for follow-up, and notifies the team via Slack. A scheduled job provides weekly reporting. Setup instructions are included in Sticky Note1. | Sticky Note1 in workflow                                                                                                         |
| Slack channel used for notifications: ID C09S57E2JQ2. Adjust this to your own Slack workspace and channel as needed.                                                                                                                                                                                         | Slack nodes configuration                                                                                                        |
| OpenAI GPT-4.1 model is required for high-quality summarization and classification. Ensure API key has access and usage quota.                                                                                                                                                                               | AI Lead Classifier node                                                                                                          |
| Google Sheets must have columns matching field names exactly for auto-mapping: username, source, summary, category, recievdAt, timestamp.                                                                                                                                                                    | Store Lead and Extrect Lead Data nodes                                                                                            |
| Jira project and issue types must be pre-configured. The workflow uses project ID 10000 and issue type ID 10004 (Story) by default; adjust accordingly.                                                                                                                                                       | Create Task node                                                                                                                  |
| Scheduled trigger interval is flexible; adjust to daily or weekly based on reporting needs.                                                                                                                                                                                                                  | Schedule Trigger node                                                                                                            |
| Potential error points include API authentication failures (OpenAI, Google Sheets, Jira, Slack), malformed input data, and rate limiting. Build in retry or error handling as needed.                                                                                                                        | General operational considerations                                                                                              |

---

**Disclaimer:** The provided text and workflow originate exclusively from an automated workflow created with n8n, complying fully with current content policies and containing no illegal, offensive, or protected material. All processed data are legal and publicly accessible.