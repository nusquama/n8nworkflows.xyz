Customer Feedback Automation with Sentiment Analysis using GPT-4.1, Jira & Slack

https://n8nworkflows.xyz/workflows/customer-feedback-automation-with-sentiment-analysis-using-gpt-4-1--jira---slack-11008


# Customer Feedback Automation with Sentiment Analysis using GPT-4.1, Jira & Slack

### 1. Workflow Overview

This workflow automates the processing of customer feedback received via a webhook, performs sentiment analysis using GPT-4.1, creates Jira tasks for negative feedback or feature suggestions, and provides a weekly summary report of all feedback issues to a Slack channel.

**Target Use Cases:**  
- Collecting and validating customer feedback from external sources via HTTP POST.  
- Automatically analyzing feedback sentiment (positive, negative, suggestion, neutral) using OpenAI GPT-4.1.  
- Creating actionable Jira tasks for negative feedback or feature requests to streamline issue tracking.  
- Sending real-time error notifications to Slack when feedback payloads are invalid.  
- Generating and posting a weekly summary report of feedback issues to Slack for management review.

**Logical Blocks:**

- **1.1 Input Reception & Validation**  
  Receives customer feedback through a webhook and validates that required fields are present.

- **1.2 AI-Driven Feedback Sentiment Analysis & Issue Creation**  
  Uses OpenAI to determine sentiment, conditionally creates Jira tasks for negative or suggestion feedback, or sends Slack alerts on errors.

- **1.3 Weekly Summary Generation & Reporting**  
  Scheduled weekly trigger fetches Jira issues created during the week, summarizes them via GPT-4.1, and posts the summary to Slack.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Validation

- **Overview:**  
  This block sets up the webhook to receive customer feedback and validates the incoming payload to ensure critical data fields are present before further processing.

- **Nodes Involved:**  
  - Collect Feedback (Webhook)  
  - Validate Payload (If)  
  - Slack – Payload Error (Slack Notification)

- **Node Details:**

  - **Collect Feedback**  
    - Type: Webhook  
    - Role: Entry point for receiving customer feedback via HTTP POST at `/customer-feedback`.  
    - Configuration: HTTP method set to POST; webhook path is `customer-feedback`.  
    - Inputs: External HTTP POST requests.  
    - Outputs: JSON body containing feedback data forwarded to validation.  
    - Failures: Missing webhook registration or incorrect HTTP method can cause failure.  
    - Notes: Sticky Note attached: "Receives customer feedback data via POST webhook."

  - **Validate Payload**  
    - Type: If  
    - Role: Checks if either `feedback_text` or `sentiment` fields in the payload are non-empty.  
    - Configuration: Condition uses OR logic on `$json.body.feedback_text` and `$json.body.sentiment` to check non-empty strings.  
    - Inputs: Output from Collect Feedback.  
    - Outputs:  
      - True: Proceed to sentiment analysis.  
      - False: Trigger Slack error notification.  
    - Failures: Expression evaluation errors if fields are missing or malformed JSON.  
    - Edge Cases: Payloads with empty or missing critical fields trigger the error path.

  - **Slack – Payload Error**  
    - Type: Slack  
    - Role: Sends an alert message to a specific Slack channel when the payload is invalid.  
    - Configuration: Posts a formatted message including the full received payload JSON for review.  
    - Inputs: False branch of Validate Payload.  
    - Outputs: None (end of path).  
    - Credentials: Slack API credentials configured with OAuth2.  
    - Failures: Slack API rate limits or authentication failures.  
    - Notes: Sends detailed payload info for manual intervention.

#### 2.2 AI-Driven Feedback Sentiment Analysis & Issue Creation

- **Overview:**  
  Analyzes validated feedback using OpenAI GPT-4.1 to determine sentiment. If sentiment is negative or a suggestion, automatically creates a Jira task with detailed feedback data.

- **Nodes Involved:**  
  - Determine Sentiment (OpenAI GPT-4.1)  
  - Is Negative or Feature Request? (If)  
  - Create Jira Task (Jira)

- **Node Details:**

  - **Determine Sentiment**  
    - Type: OpenAI (LangChain)  
    - Role: Sends feedback text to GPT-4.1, requesting a one-word sentiment classification — `positive`, `negative`, `suggestion`, or `neutral`.  
    - Configuration: Prompt instructs GPT-4.1 to return only one lowercase word representing sentiment, embedding feedback text dynamically.  
    - Inputs: Validated feedback JSON.  
    - Outputs: GPT-4.1 response with sentiment classification.  
    - Credentials: OpenAI API key configured.  
    - Failures: API rate limits, timeouts, or malformed input could cause errors.  
    - Edge Cases: Ambiguous feedback might yield inconsistent sentiment; fallback or manual review may be needed.

  - **Is Negative or Feature Request?**  
    - Type: If  
    - Role: Checks if GPT sentiment output is `negative` or `suggestion` (case-insensitive).  
    - Configuration: Condition compares the first text response from GPT output to these values.  
    - Inputs: Output from Determine Sentiment.  
    - Outputs: True branch leads to Jira task creation; False branch is terminal (no action).  
    - Failures: Expression errors if GPT output is missing or malformed.

  - **Create Jira Task**  
    - Type: Jira Issue Create  
    - Role: Creates a new Jira task in project ID `10000` with issue type `Task` (ID `10003`).  
    - Configuration:  
      - Summary includes user name from feedback.  
      - Description composed of user details, sentiment, feedback text, and timestamp.  
    - Inputs: True branch of Is Negative or Feature Request.  
    - Outputs: Jira API response confirming issue creation.  
    - Credentials: Jira Cloud API credentials configured.  
    - Failures: Jira API authentication errors, project or issue type misconfiguration, or network issues.  
    - Edge Cases: Jira project or issue type changes require update; invalid user info could affect summary.

- **Sticky Note Content:**  
  "AI-Driven Feedback Analysis and Ticket Generation: Validates feedback completeness, alerts on errors, analyzes sentiment, and creates Jira tasks for negative or suggestion feedback."

#### 2.3 Weekly Summary Generation & Reporting

- **Overview:**  
  On a weekly schedule, fetches Jira issues created in the feedback project, summarizes them using GPT-4.1, and posts the summary report to Slack for team visibility.

- **Nodes Involved:**  
  - Weekly Summary Trigger (Schedule)  
  - Search Weekly Issues (Jira)  
  - Create Weekly Summary (OpenAI GPT-4.1)  
  - Send Weekly Slack Summary (Slack)

- **Node Details:**

  - **Weekly Summary Trigger**  
    - Type: Schedule Trigger  
    - Role: Automatically triggers workflow execution once per week (default interval: weekly).  
    - Configuration: Uses default interval scheduling (weekly).  
    - Inputs: None (trigger only).  
    - Outputs: Trigger event to start issue search process.  
    - Failures: Scheduling misconfiguration or n8n instance downtime could delay triggers.

  - **Search Weekly Issues**  
    - Type: Jira Issue Search  
    - Role: Retrieves all Jira issues created since start of day (configured for weekly reporting timeframe).  
    - Configuration: JQL query `"project = KAN AND created >= startOfDay()"` (should be adjusted for weekly window if needed).  
    - Inputs: Trigger event from Weekly Summary Trigger.  
    - Outputs: Array of Jira issues matching query.  
    - Credentials: Jira API credentials configured.  
    - Failures: Jira API limits, JQL syntax errors, or incorrect project key.

  - **Create Weekly Summary**  
    - Type: OpenAI (LangChain)  
    - Role: Sends the list of issues to GPT-4.1, requesting a formal weekly summary report including sentiment distribution, patterns, and priority concerns.  
    - Configuration:  
      - Prompt instructs GPT to produce a clear, concise, official style report.  
      - Input embeds Jira issues’ descriptions dynamically.  
    - Inputs: Output array of issues from Search Weekly Issues.  
    - Outputs: Text summary report.  
    - Credentials: OpenAI API key configured.  
    - Failures: Large data payloads may cause token limits; API errors possible.

  - **Send Weekly Slack Summary**  
    - Type: Slack  
    - Role: Posts the generated weekly summary report to a designated Slack channel.  
    - Configuration: Posts formatted message; channel ID specified.  
    - Inputs: Summary text from Create Weekly Summary.  
    - Outputs: None (end of flow).  
    - Credentials: Slack API configured.  
    - Failures: Slack API rate limits or invalid channel ID.

- **Sticky Note Content:**  
  "**Weekly Summary Trigger:** Runs automatically once per week to generate the summary.\n\n**Search Weekly Issues:** Fetches all Jira issues created today (or within defined time) for weekly reporting.\n\n**Create Weekly Summary:** Uses GPT to prepare a formal summary of all weekly issues and feedback patterns.\n\n**Send Weekly Slack Summary:** Sends the generated weekly summary report to a Slack channel."

---

### 3. Summary Table

| Node Name                  | Node Type                  | Functional Role                                  | Input Node(s)          | Output Node(s)               | Sticky Note                                                                                                      |
|----------------------------|----------------------------|-------------------------------------------------|------------------------|-----------------------------|-----------------------------------------------------------------------------------------------------------------|
| Collect Feedback           | Webhook                    | Receive customer feedback via POST request       | External HTTP request  | Validate Payload             | Receives customer feedback data via POST webhook.                                                               |
| Validate Payload           | If                         | Validate presence of feedback text or sentiment  | Collect Feedback       | Determine Sentiment, Slack – Payload Error | AI-Driven Feedback Analysis and Ticket Generation: Validates feedback completeness, alerts on errors, analyzes sentiment, creates Jira tasks. |
| Slack – Payload Error      | Slack                      | Send alert on invalid or incomplete payload      | Validate Payload       | None                        | Sends detailed payload info for manual intervention.                                                            |
| Determine Sentiment        | OpenAI (LangChain)         | Analyze feedback text to determine sentiment     | Validate Payload       | Is Negative or Feature Request? | AI-Driven Feedback Analysis and Ticket Generation: Validates feedback completeness, alerts on errors, analyzes sentiment, creates Jira tasks. |
| Is Negative or Feature Request? | If                    | Check if sentiment is negative or suggestion      | Determine Sentiment    | Create Jira Task            | AI-Driven Feedback Analysis and Ticket Generation: Validates feedback completeness, alerts on errors, analyzes sentiment, creates Jira tasks. |
| Create Jira Task           | Jira Issue Create          | Create Jira task for negative/suggestion feedback| Is Negative or Feature Request? | None                        | AI-Driven Feedback Analysis and Ticket Generation: Validates feedback completeness, alerts on errors, analyzes sentiment, creates Jira tasks. |
| Weekly Summary Trigger     | Schedule Trigger           | Trigger weekly summary generation                 | None                   | Search Weekly Issues        | **Weekly Summary Trigger:** Runs automatically once per week to generate the summary.                            |
| Search Weekly Issues       | Jira Issue Search          | Fetch Jira issues created recently                 | Weekly Summary Trigger | Create Weekly Summary       | **Weekly Summary Trigger:** Runs automatically once per week to generate the summary.                            |
| Create Weekly Summary      | OpenAI (LangChain)         | Generate formal weekly feedback summary           | Search Weekly Issues   | Send Weekly Slack Summary   | **Weekly Summary Trigger:** Runs automatically once per week to generate the summary.                            |
| Send Weekly Slack Summary  | Slack                      | Post weekly feedback summary to Slack channel     | Create Weekly Summary  | None                       | **Weekly Summary Trigger:** Runs automatically once per week to generate the summary.                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node "Collect Feedback"**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `customer-feedback`  
   - Purpose: Receive incoming customer feedback as JSON payload.

2. **Create If Node "Validate Payload"**  
   - Type: If  
   - Condition: Check if either `{{$json.body.feedback_text}}` or `{{$json.body.sentiment}}` is not empty string.  
   - Logic: OR condition on non-empty strings of these fields.  
   - Connect input from "Collect Feedback".

3. **Create Slack Node "Slack – Payload Error"**  
   - Type: Slack  
   - Purpose: Notify Slack channel on invalid payload.  
   - Channel: Use channel ID `C09S57E2JQ2` or appropriate channel.  
   - Message: Include a warning and a JSON-formatted string of the received payload using expression `{{ JSON.stringify($json.body, null, 2) }}`.  
   - Credentials: Configure Slack OAuth2 credentials.  
   - Connect input from False output of "Validate Payload".

4. **Create OpenAI Node "Determine Sentiment"**  
   - Type: OpenAI (LangChain)  
   - Model: GPT-4.1  
   - Prompt:  
     ```
     Return only one lowercase word: "positive", "negative", "suggestion", or "neutral".
     Feedback: {{ $json.body.feedback_text }}
     ```  
   - Credentials: Configure OpenAI API key.  
   - Connect input from True output of "Validate Payload".

5. **Create If Node "Is Negative or Feature Request?"**  
   - Type: If  
   - Condition: Check if OpenAI response text equals `negative` or `suggestion` (case-insensitive).  
   - Use expression on `{{ $json.output[0].content[0].text }}`.  
   - Connect input from "Determine Sentiment".

6. **Create Jira Node "Create Jira Task"**  
   - Type: Jira Issue Create  
   - Project ID: `10000`  
   - Issue Type: `Task` (ID `10003`)  
   - Summary: `Customer Feedback From – {{ $('Collect Feedback').item.json.body.user_name }}`  
   - Description:  
     ```
     **Customer Feedback**  
     **User:** {{ $('Collect Feedback').item.json.body.user_name }}  
     **Email:** {{ $('Collect Feedback').item.json.body.email }}  
     **Sentiment:** {{ $json.output[0].content[0].text }}  
     ---  
     {{ $('Collect Feedback').item.json.body.feedback_text }}  
     ---  
     Received: {{$now}}  
     ```  
   - Credentials: Configure Jira Cloud API credentials.  
   - Connect input from True output of "Is Negative or Feature Request?".

7. **Create Schedule Trigger "Weekly Summary Trigger"**  
   - Type: Schedule Trigger  
   - Set interval to weekly (default weekly interval).  

8. **Create Jira Node "Search Weekly Issues"**  
   - Type: Jira Issue Search  
   - JQL: `project = KAN AND created >= startOfDay()` (consider adjusting to weekly range if needed)  
   - Return All: Enabled  
   - Credentials: Use Jira Cloud API credentials.  
   - Connect input from "Weekly Summary Trigger".

9. **Create OpenAI Node "Create Weekly Summary"**  
   - Type: OpenAI (LangChain)  
   - Model: GPT-4.1  
   - Prompt:  
     ```
     You are an assistant summarizing customer feedback for management.
     Write a clear, concise weekly report based only on the provided issues.
     Include:
     - Overall sentiment distribution
     - Positive patterns
     - Negative patterns / issues
     - Feature requests
     - High-priority problems
     
     Format the output in official formal message
     
     Here are the feedback issues:
     {{ $json.fields.description }}
     ```  
   - Credentials: OpenAI API key configured.  
   - Connect input from "Search Weekly Issues".

10. **Create Slack Node "Send Weekly Slack Summary"**  
    - Type: Slack  
    - Channel: Use channel ID `C09S57E2JQ2` or appropriate.  
    - Message:  
      ```
      📊 *Weekly Customer Feedback Summary*

      {{ $json.output[0].content[0].text }}
      ```  
    - Credentials: Slack OAuth2 credentials.  
    - Connect input from "Create Weekly Summary".

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Context or Link                                      |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| This workflow automates customer feedback handling end-to-end, reducing manual triage effort by integrating webhook input, AI sentiment analysis, Jira issue creation, and Slack notifications. It is designed to improve customer support responsiveness and management visibility.                                                                                                                                                                                                                                                                                                                                                                                   | Workflow purpose and benefits                         |
| Slack channels and Jira project keys/IDs are hardcoded and should be adapted to your environment. Ensure all API credentials (Slack, Jira, OpenAI) are properly configured with sufficient permissions.                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Credential and environment setup                      |
| The OpenAI node uses GPT-4.1 model for sentiment classification and report generation. Prompt design is critical to ensure consistent responses; test with various feedback samples to tune accuracy.                                                                                                                                                                                                                                                                                                                                                                                                                                                                | OpenAI prompt design importance                       |
| The weekly Jira issue search uses a JQL query with `startOfDay()` which may need adjustment to cover the entire week (e.g., `startOfWeek()`) depending on your reporting period. Confirm time zones and JQL syntax accordingly.                                                                                                                                                                                                                                                                                                                                                                                                                                         | Jira JQL configuration for weekly reporting          |
| Slack API rate limits and Jira API quotas should be monitored to avoid workflow disruptions during high feedback volume periods.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | API usage considerations                              |
| Sticky notes in the workflow serve as documentation blocks explaining the architecture and logic flow. They should be reviewed for onboarding and maintenance purposes.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Internal workflow documentation                       |

---

**Disclaimer:** The provided text is sourced exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.