Zendesk Pending Ticket Follow-up System with Gmail, Google Sheets & ClickUp

https://n8nworkflows.xyz/workflows/zendesk-pending-ticket-follow-up-system-with-gmail--google-sheets---clickup-8743


# Zendesk Pending Ticket Follow-up System with Gmail, Google Sheets & ClickUp

### 1. Workflow Overview

This workflow automates the management and follow-up process for pending support tickets in Zendesk. It is designed for customer support teams to efficiently track unresolved tickets, send personalized follow-up emails, and create actionable tasks for team members to ensure timely responses.

The workflow runs on a scheduled basis (Monday to Friday at 8 PM) and logically divides into the following blocks:

- **1.1 Input Reception & Data Retrieval:** Scheduled trigger initiates the workflow and fetches all Zendesk tickets with a "pending" status via API.
- **1.2 Filtering Logic:** Ensures only tickets with the exact "pending" status are processed, reducing noise.
- **1.3 Data Processing:** Formats and enriches ticket data with computed fields such as ticket age, priority level, and attention flags.
- **1.4 Data Logging:** Logs or updates ticket entries in a Google Sheet for historical tracking and analytics.
- **1.5 Task Management:** Creates ClickUp tasks for each pending ticket to assign and remind team members.
- **1.6 Email Generation:** Generates personalized HTML follow-up emails grouped by requester email, summarizing pending tickets.
- **1.7 Email Delivery:** Sends the generated follow-up emails via Gmail with proper branding and formatting.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Data Retrieval

**Overview:**  
This block triggers the workflow on a fixed schedule and retrieves all Zendesk tickets currently pending.

**Nodes Involved:**  
- Schedule Trigger  
- Get Pending Tickets  
- Data Retrieval Info (sticky note)

**Node Details:**

- **Schedule Trigger**  
  - Type: Cron Trigger  
  - Configuration: Fires at 20:00 (8 PM) Monday to Friday (weekdays).  
  - Input: None (time-based trigger)  
  - Output: Starts the workflow every scheduled time.  
  - Failure cases: Cron expression errors or scheduler downtime.

- **Get Pending Tickets**  
  - Type: Zendesk node (getAll operation)  
  - Configuration: Filters tickets with status "pending", returns all matching tickets without pagination limits.  
  - Credentials: Uses Zendesk API credentials named "Zendesk account vivek".  
  - Input: Trigger from Schedule Trigger.  
  - Output: Array of pending tickets.  
  - Failure cases: API authentication errors, rate limits, network timeouts, or Zendesk API downtime.

- **Data Retrieval Info (Sticky Note)**  
  - Contains explanations about data fetching, required credentials, and filter criteria.

---

#### 2.2 Filtering Logic

**Overview:**  
Ensures that only tickets with the status "pending" proceed further, adding an extra layer of filtering.

**Nodes Involved:**  
- Filter Pending Tickets (If node)  
- Filtering Logic (sticky note)

**Node Details:**

- **Filter Pending Tickets**  
  - Type: If node  
  - Configuration: Checks if the ticket's `status` field strictly equals "pending".  
  - Input: Output from Get Pending Tickets.  
  - Output: Passes tickets where status === "pending"; discards others.  
  - Failure cases: Expression evaluation errors if `status` is missing or malformed.

- **Filtering Logic (Sticky Note)**  
  - Explains the purpose of filtering to reduce noise and process only actionable tickets.

---

#### 2.3 Data Processing

**Overview:**  
Transforms raw Zendesk ticket data into a structured and enriched JSON object, adding computed properties like ticket age in hours, priority level, and attention flags.

**Nodes Involved:**  
- Format Ticket Data (Code node)  
- Data Processing (sticky note)

**Node Details:**

- **Format Ticket Data**  
  - Type: Code (JavaScript) node  
  - Configuration:  
    - Parses dates to ISO strings.  
    - Calculates ticket age in hours.  
    - Maps priority to numeric levels (low=1, normal=2, high=3, urgent=4).  
    - Cleans descriptions for preview (strips HTML, truncates to 100 chars).  
    - Flags tickets needing attention based on age > 24h, urgent tags, or high priority.  
    - Builds ticket URL based on a fixed Zendesk domain (placeholder `your-domain.zendesk.com`).  
    - Uses safe defaults for missing fields.  
  - Input: Tickets from Filter Pending Tickets.  
  - Output: Array of formatted ticket JSON objects.  
  - Failure cases:  
    - Missing or malformed date fields.  
    - Unexpected data types causing runtime errors.  
    - Unhandled null or undefined fields.

- **Data Processing (Sticky Note)**  
  - Describes data enrichment and preparation for downstream nodes.

---

#### 2.4 Data Logging

**Overview:**  
Appends or updates the ticket records in a Google Sheet to maintain a historical log and enable reporting.

**Nodes Involved:**  
- Log to Google Sheets  
- Data Logging (sticky note)

**Node Details:**

- **Log to Google Sheets**  
  - Type: Google Sheets node  
  - Configuration:  
    - Operation: appendOrUpdate with "Ticket ID" as matching column.  
    - Sheet: Uses sheet with gid=0 named "Pending Tickets".  
    - Document ID: Specific Google Sheet ID (visible in configuration).  
    - Maps ticket fields such as tags, status, subject, priority, age, etc., to corresponding columns.  
    - Mapping mode: autoMapInputData with explicit matching columns for update.  
  - Credentials: Google Sheets OAuth2 with account "automations@techdome.ai".  
  - Input: Formatted ticket data from Format Ticket Data.  
  - Output: Updated sheet rows confirmation.  
  - Failure cases: Authentication failures, permissions issues, Google Sheets API rate limits, document or sheet not found.

- **Data Logging (Sticky Note)**  
  - Explains the logging purpose and update logic keyed on Ticket ID.

---

#### 2.5 Task Management

**Overview:**  
Creates or updates ClickUp tasks for each pending ticket to remind the support team and track follow-up actions.

**Nodes Involved:**  
- Create ClickUp Task  
- Task Management (sticky note)

**Node Details:**

- **Create ClickUp Task**  
  - Type: ClickUp node (task creation)  
  - Configuration:  
    - Creates task in list ID "901412904902" under team "9014871666" and space "90143687238".  
    - Task name set to ticket subject.  
    - Folderless tasks (no folder specified).  
    - Additional fields include tags (from Zendesk), status set to "to do", due date set to ticket creation timestamp, and priority based on numeric priority level.  
  - Credentials: ClickUp API named "ClickUp account vivek".  
  - Input: Formatted ticket data from Log to Google Sheets.  
  - Output: Confirmation of task creation.  
  - Failure cases: API authentication errors, invalid list/team/space IDs, rate limiting.

- **Task Management (Sticky Note)**  
  - Describes task creation details and rationale.

---

#### 2.6 Email Generation

**Overview:**  
Generates personalized, professional HTML follow-up emails for each customer with pending tickets, grouping tickets by requester email.

**Nodes Involved:**  
- Generate Follow-up Emails (Code node)  
- Email Generation (sticky note)

**Node Details:**

- **Generate Follow-up Emails**  
  - Type: Code (JavaScript) node  
  - Configuration:  
    - Filters input for only tickets with status "pending".  
    - Groups tickets by extracted email address (regex applied on ticket description fallback if missing).  
    - Generates an HTML email per requester including:  
      - Personalized greeting extracted from description or fallback "Valued Customer".  
      - Ticket details with IDs, age, subject, and description snippets.  
      - Call-to-action linking to support portal.  
      - Contact info and branding footer.  
    - Email subject includes ticket count and urgency.  
    - Uses inline CSS for consistent email client rendering.  
    - If no pending tickets found, returns a message item indicating so.  
  - Input: Formatted ticket data array from Log to Google Sheets.  
  - Output: Array of email objects with `to`, `subject`, `html`, and metadata fields.  
  - Failure cases:  
    - Regex failures extracting emails or names.  
    - Malformed HTML generation.  
    - Empty ticket arrays.  
    - Missing description fields.

- **Email Generation (Sticky Note)**  
  - Explains email content and personalization strategy.

---

#### 2.7 Email Delivery

**Overview:**  
Sends the generated follow-up emails via Gmail using OAuth2 credentials, preserving HTML formatting and branding.

**Nodes Involved:**  
- Send Follow-up Email  
- Email Delivery (sticky note)

**Node Details:**

- **Send Follow-up Email**  
  - Type: Gmail node (send message)  
  - Configuration:  
    - Recipient email dynamically set from generated emails (`to` field).  
    - Subject dynamically set per email.  
    - HTML formatted message body from generated content.  
    - Plain text fallback message included.  
  - Credentials: Gmail OAuth2 credentials named "Gmail credentials".  
  - Input: Emails from Generate Follow-up Emails.  
  - Output: Email sending status.  
  - Failure cases:  
    - Authentication errors.  
    - Gmail API rate limits or quota exceeded.  
    - Invalid email addresses.  
    - Network or service outages.

- **Email Delivery (Sticky Note)**  
  - Highlights requirement for Gmail OAuth2 and professional branding.

---

### 3. Summary Table

| Node Name              | Node Type             | Functional Role                            | Input Node(s)          | Output Node(s)           | Sticky Note                                                                                              |
|------------------------|-----------------------|-------------------------------------------|------------------------|--------------------------|--------------------------------------------------------------------------------------------------------|
| Workflow Overview      | Sticky Note           | High-level workflow purpose overview      | –                      | –                        | ## 🎯 Workflow Overview: Automates Zendesk ticket follow-ups; runs Mon-Fri 8 PM                        |
| Schedule Trigger       | Cron Trigger          | Starts workflow on schedule                | –                      | Get Pending Tickets       |                                                                                                        |
| Data Retrieval Info    | Sticky Note           | Explains Zendesk API data retrieval        | –                      | –                        | ## 📊 Data Retrieval: Fetch pending tickets; requires Zendesk API credentials                           |
| Get Pending Tickets    | Zendesk node          | Fetches all pending tickets                | Schedule Trigger        | Filter Pending Tickets    |                                                                                                        |
| Filtering Logic        | Sticky Note           | Explains ticket filtering rationale         | –                      | –                        | ## 🔍 Smart Filtering: Only process tickets with status 'pending'                                      |
| Filter Pending Tickets | If node               | Filters tickets strictly by status "pending" | Get Pending Tickets     | Format Ticket Data        |                                                                                                        |
| Data Processing        | Sticky Note           | Explains data enrichment and formatting    | –                      | –                        | ## 🛠️ Data Processing: Format and enrich ticket data with computed fields                              |
| Format Ticket Data     | Code node             | Formats raw tickets into enriched JSON     | Filter Pending Tickets  | Log to Google Sheets, Generate Follow-up Emails |                                                                                                        |
| Data Logging           | Sticky Note           | Explains Google Sheets logging             | –                      | –                        | ## 📊 Data Logging: Logs tickets for reporting; updates by Ticket ID                                   |
| Log to Google Sheets   | Google Sheets node    | Appends/updates tickets in Google Sheet   | Format Ticket Data      | Create ClickUp Task       |                                                                                                        |
| Task Management        | Sticky Note           | Explains ClickUp task creation             | –                      | –                        | ## ✅ Task Management: Creates ClickUp tasks per ticket with due dates and tags                        |
| Create ClickUp Task    | ClickUp node          | Creates task for each ticket in ClickUp   | Log to Google Sheets    | –                        |                                                                                                        |
| Email Generation       | Sticky Note           | Explains email generation logic            | –                      | –                        | ## 📧 Email Generation: Personalized emails grouped by customer email                                  |
| Generate Follow-up Emails | Code node           | Generates personalized HTML follow-up emails | Format Ticket Data      | Send Follow-up Email      |                                                                                                        |
| Email Delivery         | Sticky Note           | Explains email sending via Gmail           | –                      | –                        | ## 📤 Email Delivery: Sends follow-up emails; requires Gmail OAuth2 credentials                         |
| Send Follow-up Email   | Gmail node            | Sends personalized follow-up emails        | Generate Follow-up Emails | –                        |                                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add a Cron node (Schedule Trigger):**  
   - Type: Cron  
   - Set to trigger at 20:00 (8 PM) Monday to Friday using custom cron expression `0 0 20 * * 1-5`.

3. **Add a Zendesk node (Get Pending Tickets):**  
   - Operation: `getAll` tickets  
   - Filter: status = "pending"  
   - Return All: true  
   - Connect input from Schedule Trigger.  
   - Set Zendesk credentials (API token or OAuth) under "Zendesk account vivek" or your own.

4. **Add an If node (Filter Pending Tickets):**  
   - Condition: Check if `{{$json.status}}` equals `pending` (case sensitive, strict).  
   - Connect input from Get Pending Tickets node.

5. **Add a Code node (Format Ticket Data):**  
   - Paste the JavaScript code that formats tickets, computes age, cleans description, etc.  
   - Input from the "true" output of the Filter Pending Tickets node.

6. **Add a Google Sheets node (Log to Google Sheets):**  
   - Operation: `appendOrUpdate`  
   - Document ID: Your Google Sheet ID (use the provided or your own).  
   - Sheet Name or GID: e.g., `gid=0` or "Pending Tickets".  
   - Matching Columns: "Ticket ID"  
   - Map ticket fields such as Subject, Status, Priority, Tags, Age, etc., to sheet columns.  
   - Connect input from Format Ticket Data node.  
   - Set Google Sheets OAuth2 credentials (e.g., "automations@techdome.ai").

7. **Add a ClickUp node (Create ClickUp Task):**  
   - Operation: Create task  
   - Configure team, space, list IDs as per your ClickUp setup.  
   - Set task name from ticket subject, status "to do", due date from created timestamp, tags from Zendesk tags, and priority level.  
   - Connect input from Log to Google Sheets node.  
   - Set ClickUp API credentials.

8. **Add a Code node (Generate Follow-up Emails):**  
   - Paste the provided JavaScript code to group tickets by requester email and generate HTML emails.  
   - Input from Format Ticket Data node.

9. **Add a Gmail node (Send Follow-up Email):**  
   - Operation: Send email message  
   - To: `{{$json.to}}` dynamically from generated emails  
   - Subject: `{{$json.subject}}`  
   - HTML message body: `{{$json.html}}`  
   - Include plain text fallback message.  
   - Connect input from Generate Follow-up Emails node.  
   - Set Gmail OAuth2 credentials.

10. **Add sticky notes** for documentation and clarity at each logical block, reflecting the content described in Section 2.

11. **Activate and test the workflow.** Use test data or live Zendesk tickets to verify data retrieval, filtering, formatting, logging, task creation, email generation, and delivery.

---

### 5. General Notes & Resources

| Note Content                                                                                                           | Context or Link                                                                                      |
|------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Workflow runs Monday to Friday at 8 PM to avoid weekend inactivity in support tickets.                                  | Workflow Overview sticky note                                                                        |
| Zendesk domain placeholder in ticket URL (`your-domain.zendesk.com`) must be replaced with your actual Zendesk domain. | Format Ticket Data node code comments                                                               |
| Gmail OAuth2 credentials require proper setup and consent for scope to send emails on behalf of your domain/email.      | Email Delivery sticky note                                                                           |
| ClickUp list, team, and space IDs must be configured to match your workspace.                                          | Task Management sticky note                                                                          |
| Google Sheets must have columns matching the mapped fields, with "Ticket ID" as unique key for updates.                | Data Logging sticky note                                                                             |
| For email personalization, the code attempts to extract customer names and emails from ticket descriptions.            | Email Generation code comments                                                                       |
| Support portal URL in email template should be updated to your actual Zendesk help center URL.                          | Generate Follow-up Emails node HTML template                                                         |
| Rate limiting on Zendesk, Google Sheets, ClickUp, and Gmail APIs may require retry or error handling improvements.     | General caution for all API nodes                                                                    |
| This workflow assumes well-formed ticket data and proper authentication credentials for all integrations.               | Overall workflow precondition                                                                         |

---

**Disclaimer:**  
The provided documentation is based exclusively on an automated workflow built with n8n, respecting all applicable content policies. It does not contain illegal or offensive content, and all data handled is legal and public.