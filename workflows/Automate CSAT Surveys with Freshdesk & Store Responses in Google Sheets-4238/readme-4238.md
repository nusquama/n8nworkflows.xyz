Automate CSAT Surveys with Freshdesk & Store Responses in Google Sheets

https://n8nworkflows.xyz/workflows/automate-csat-surveys-with-freshdesk---store-responses-in-google-sheets-4238


# Automate CSAT Surveys with Freshdesk & Store Responses in Google Sheets

---

### 1. Workflow Overview

This workflow automates the collection of Customer Satisfaction (CSAT) survey responses following ticket resolution in Freshdesk, and stores those responses in Google Sheets. It further automates emailing customers a feedback survey link when their support ticket is closed.

The workflow logically divides into these blocks:

- **1.1 Initialization & Scheduling:** Trigger the workflow manually or at scheduled intervals to start the process.
- **1.2 Data Preparation:** Set static metadata such as sender details and survey link.
- **1.3 Ticket Synchronization:** Retrieve existing ticket data from Google Sheets and fetch updated ticket statuses from Freshdesk.
- **1.4 Ticket Status Evaluation:** Identify tickets that have transitioned to "Resolved" status.
- **1.5 Update Ticket Status in Google Sheets:** Reflect latest ticket statuses in the Google Sheet.
- **1.6 Retrieve Client Information:** Fetch customer contact details from Freshdesk for resolved tickets.
- **1.7 Email Composition and Sending:** Create and send a personalized email with a survey link to the client.
- **1.8 Survey Reception & Storage:** Capture survey responses submitted via an n8n Form Trigger node and append them to a designated Google Sheet.

---

### 2. Block-by-Block Analysis

#### 2.1 Initialization & Scheduling

- **Overview:** Starts the workflow either manually or automatically on an hourly schedule.
- **Nodes Involved:**
  - When clicking ‘Test workflow’
  - Schedule Trigger

**Node Details:**

- **When clicking ‘Test workflow’**  
  - *Type:* Manual Trigger  
  - *Role:* Allows manual initiation of the workflow for testing or on-demand execution.  
  - *Connections:* Outputs to "Set your data".  
  - *Edge cases:* Manual start may cause data duplication if run alongside scheduled runs.

- **Schedule Trigger**  
  - *Type:* Schedule Trigger  
  - *Role:* Automatically triggers the workflow every hour based on configured interval.  
  - *Parameters:* Interval set to every 1 hour.  
  - *Connections:* Outputs to "Set your data".  
  - *Edge cases:* Overlapping runs if a previous run takes longer than the interval.

#### 2.2 Data Preparation

- **Overview:** Sets static information such as sender’s name, email, company, position, and the survey link URL, which are used later in email composition.
- **Nodes Involved:**
  - Set your data
  - Sticky Note (contextual note for this block)

**Node Details:**

- **Set your data**  
  - *Type:* Set node  
  - *Role:* Assigns constant values for personal and company details and the survey link.  
  - *Key fields set:*  
    - your name: "Thomas Vié"  
    - Your email: "thomas@pollup.net"  
    - your company name: "Pollup Data services"  
    - your position: "C.T.O."  
    - survey link: URL of the form trigger webhook (production URL).  
  - *Connections:* Outputs to "get existing tickets".  
  - *Edge cases:* Survey link must match the Form Trigger webhook URL; if changed, email links may break.

- **Sticky Note (Set your data here)**  
  - *Content:* Advises to put the production URL of the Survey node as the survey link and to activate the workflow.

#### 2.3 Ticket Synchronization

- **Overview:** Loads the list of tickets already stored in Google Sheets, then fetches all current tickets from Freshdesk for comparison.
- **Nodes Involved:**
  - get existing tickets
  - get tickets
  - Sticky Notes (contextual for these nodes)

**Node Details:**

- **get existing tickets**  
  - *Type:* Google Sheets (read)  
  - *Role:* Reads the existing ticket data from a Google Sheet where ticket info and statuses are stored.  
  - *Parameters:* Reads from Sheet1 of the specified Google Sheets document (Freshdesk Tickets spreadsheet).  
  - *Connections:* Outputs to "get tickets".  
  - *Edge cases:* Sheet must exist and be accessible with proper OAuth2 credentials. Empty or malformed data may cause logic errors.

- **get tickets**  
  - *Type:* Freshdesk (getAll tickets)  
  - *Role:* Retrieves all tickets from Freshdesk to compare current statuses against stored data.  
  - *Parameters:* No filters; fetches all tickets.  
  - *Connections:* Outputs to "If ticket resolved".  
  - *Edge cases:* API rate limits, authentication failures, or network timeouts can interrupt. Large ticket volumes may impact performance.

- **Sticky Notes**  
  - *Purpose:* Remind to create the Google Sheet and connect Freshdesk credentials appropriately.

#### 2.4 Ticket Status Evaluation

- **Overview:** Compares the status of each ticket fetched from Freshdesk with the stored status and checks if the current status is "Resolved" (status code 4).
- **Nodes Involved:**
  - If ticket resolved

**Node Details:**

- **If ticket resolved**  
  - *Type:* If node (conditional branching)  
  - *Role:* Evaluates two conditions combined with AND:  
    1. The ticket status from Freshdesk differs from the stored status in Google Sheets.  
    2. The ticket status equals 4 (which corresponds to "Resolved").  
  - *Expressions used:*  
    - Left value: `{{$('get tickets').item.json.status}}`  
    - Right value: `{{$('get existing tickets').item.json.status}}`  
  - *Connections:* True branch to "Updates status", False branch ends flow for non-resolved or unchanged tickets.  
  - *Edge cases:* If matching fails or data is missing, tickets may be skipped. Status codes assumed fixed.

#### 2.5 Update Ticket Status in Google Sheets

- **Overview:** Updates the ticket status in the Google Sheet for tickets that transitioned to "Resolved".
- **Nodes Involved:**
  - Updates status

**Node Details:**

- **Updates status**  
  - *Type:* Google Sheets (appendOrUpdate)  
  - *Role:* Updates the status column for the corresponding ticket row in the Google Sheet.  
  - *Matching column:* `row_number` to identify the correct row.  
  - *Data updated:* status field with current Freshdesk ticket status.  
  - *Connections:* Outputs to "get client".  
  - *Edge cases:* Update failures if row number is missing or sheet access revoked.

#### 2.6 Retrieve Client Information

- **Overview:** Retrieves the contact details of the ticket requester from Freshdesk to personalize the survey email.
- **Nodes Involved:**
  - get client

**Node Details:**

- **get client**  
  - *Type:* Freshdesk (contact get)  
  - *Role:* Fetches contact info using the ticket's requester_id.  
  - *Parameters:* Uses `contactId = {{$('get tickets').item.json.requester_id}}`.  
  - *Connections:* Outputs to "Create the email text".  
  - *Edge cases:* Missing or invalid requester_id may cause failure. API limits or auth errors possible.

- **Sticky Note**  
  - Notes to connect this node with Freshdesk credentials.

#### 2.7 Email Composition and Sending

- **Overview:** Constructs a personalized email body with the survey link and sends it to the client’s email.
- **Nodes Involved:**
  - Create the email text
  - Convert the email text to HTML
  - Send Email
  - Sticky Note (email send settings)

**Node Details:**

- **Create the email text**  
  - *Type:* Set node  
  - *Role:* Creates two key fields:  
    - subject: "Quick Feedback? Help Us Improve"  
    - body: A templated message referencing ticket subject, survey link, and sender details.  
  - *Expressions:*  
    - Subject uses static string.  
    - Body uses interpolation with `$('get tickets').item.json.subject` and `$('Set your data').item.json['survey link']` and sender info.  
  - *Connections:* Outputs to "Convert the email text to HTML".  
  - *Edge cases:* Missing data may produce broken emails.

- **Convert the email text to HTML**  
  - *Type:* Markdown node  
  - *Role:* Converts the plain text email body (which uses markdown link syntax) to HTML for email clients.  
  - *Connections:* Outputs to "Send Email".  
  - *Edge cases:* Markdown syntax errors may affect output.

- **Send Email**  
  - *Type:* Email Send  
  - *Role:* Sends the composed HTML email to the requester’s email address.  
  - *Parameters:*  
    - To: requester’s email from `$('get client').first().json.email`  
    - From: static sender email ("thomas@pollup.net")  
    - Subject and HTML body from previous nodes.  
  - *Credentials:* SMTP account credentials configured.  
  - *Edge cases:* SMTP failures, invalid addresses, or network issues may cause delivery failure.

- **Sticky Note**  
  - Suggests replacing this node with a Gmail node or other email sending nodes as needed.

#### 2.8 Survey Reception & Storage

- **Overview:** Captures survey submissions from users via a form trigger, then appends the responses to a Google Sheet.
- **Nodes Involved:**
  - Survey
  - Save survey to google sheet
  - Sticky Notes (modifying survey and saving survey)

**Node Details:**

- **Survey**  
  - *Type:* Form Trigger  
  - *Role:* Hosts a web form titled "Quick Feedback" with two required fields: a dropdown rating (1-5) and a textarea for comments.  
  - *WebhookId:* Matches the survey link used in email.  
  - *Connections:* Outputs to "Save survey to google sheet".  
  - *Edge cases:* Form URL must be public and workflow active; form field changes require workflow edit.

- **Save survey to google sheet**  
  - *Type:* Google Sheets (append)  
  - *Role:* Appends submitted survey data to a designated Google Sheet.  
  - *Parameters:*  
    - Auto-maps the form fields to columns in Sheet1 of the "Feedback freshdesk" spreadsheet.  
  - *Credentials:* Google Sheets OAuth2 credentials.  
  - *Edge cases:* Sheet must exist, have correct columns, and be accessible.

- **Sticky Notes**  
  - Suggest creating the Google Sheet before saving and modifying survey questions as needed.

---

### 3. Summary Table

| Node Name                | Node Type          | Functional Role                          | Input Node(s)               | Output Node(s)             | Sticky Note                                                                                      |
|--------------------------|--------------------|----------------------------------------|-----------------------------|----------------------------|------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger     | Manual start of workflow                | -                           | Set your data              |                                                                                                |
| Schedule Trigger          | Schedule Trigger   | Scheduled start every hour              | -                           | Set your data              |                                                                                                |
| Set your data            | Set                | Set static sender details and survey link | When clicking ‘Test workflow’, Schedule Trigger | get existing tickets       | ## Set your data here - Put as Survey link the Production URL of the "Survey" Node and set the workflow to "Active" |
| get existing tickets     | Google Sheets      | Read stored ticket data from Google Sheets | Set your data                | get tickets                | ## Save tickets - Create an empty google sheet and put it here                                  |
| get tickets              | Freshdesk          | Fetch all tickets from Freshdesk        | get existing tickets         | If ticket resolved         | ## Get tickets - Connect here to Freshdesk with your credentials                                |
| If ticket resolved       | If                 | Check if ticket status changed to resolved | get tickets                  | Updates status             |                                                                                                |
| Updates status           | Google Sheets      | Update ticket status in Google Sheets   | If ticket resolved           | get client                 |                                                                                                |
| get client               | Freshdesk          | Get contact info for requester          | Updates status               | Create the email text      | ## Get Client - Connect here to Freshdesk with your credentials                                |
| Create the email text    | Set                | Compose email subject and body          | get client                   | Convert the email text to HTML |                                                                                                |
| Convert the email text to HTML | Markdown           | Convert email body from markdown to HTML | Create the email text        | Send Email                 |                                                                                                |
| Send Email               | Email Send         | Send email to client                     | Convert the email text to HTML | -                          | ## Email send settings - Here you can change this node for a gmail node for example.            |
| Survey                   | Form Trigger       | Capture survey responses via form       | -                           | Save survey to google sheet |                                                                                                |
| Save survey to google sheet | Google Sheets      | Append survey responses to Google Sheet | Survey                      | -                          | ## Save survey - Create an empty google sheet and put it here                                  |
| Sticky Note              | Sticky Note        | Notes and instructions                   | -                           | -                          | ## Contact me - If you need any modification to this workflow, help, or other workflows contact thomas@pollup.net. See others [here](https://n8n.io/creators/zeerobug/) |
| Sticky Note1             | Sticky Note        | Notes for creating ticket storage sheet  | -                           | -                          | ## Save tickets - Create an empty google sheet and put it here                                  |
| Sticky Note2             | Sticky Note        | Notes for Freshdesk credentials setup    | -                           | -                          | ## Get tickets - Connect here to Freshdesk with your credentials                               |
| Sticky Note3             | Sticky Note        | Notes for email send node customization  | -                           | -                          | ## Email send settings - Here you can change this node for a gmail node for example.           |
| Sticky Note4             | Sticky Note        | Notes for Freshdesk client node setup    | -                           | -                          | ## Get Client - Connect here to Freshdesk with your credentials                               |
| Sticky Note5             | Sticky Note        | Notes for survey Google Sheet setup      | -                           | -                          | ## Save survey - Create an empty google sheet and put it here                                  |
| Sticky Note6             | Sticky Note        | Notes for modifying survey questions      | -                           | -                          | ## Modify Survey - Set all the survey's question you need here                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node:**  
   - Name: "When clicking ‘Test workflow’"  
   - Purpose: To allow manual workflow starts for testing.

2. **Create Schedule Trigger node:**  
   - Name: "Schedule Trigger"  
   - Set interval to trigger every 1 hour.

3. **Create Set node:**  
   - Name: "Set your data"  
   - Assign the following values:  
     - your name: "Thomas Vié"  
     - Your email: "thomas@pollup.net"  
     - your company name: "Pollup Data services"  
     - your position: "C.T.O."  
     - survey link: (set initially empty, to be updated with the Survey node webhook URL once created)

4. **Connect both triggers ("When clicking ‘Test workflow’" and "Schedule Trigger") to "Set your data" node.**

5. **Create Google Sheets node:**  
   - Name: "get existing tickets"  
   - Operation: Read rows from Google Sheets  
   - DocumentId: Your "Freshdesk Tickets" Google Sheets document ID  
   - SheetName: "Sheet1" (or corresponding sheet)  
   - Connect "Set your data" to this node.

6. **Create Freshdesk node:**  
   - Name: "get tickets"  
   - Operation: Get All Tickets  
   - Credentials: Connect your Freshdesk API credentials  
   - Connect "get existing tickets" to this node.

7. **Create If node:**  
   - Name: "If ticket resolved"  
   - Condition (AND):  
     - Left: `{{$('get tickets').item.json.status}}`  
     - Operator: Not Equals  
     - Right: `{{$('get existing tickets').item.json.status}}`  
   - AND  
     - Left: `{{$('get tickets').item.json.status}}`  
     - Operator: Equals  
     - Right: 4 (Resolved status)  
   - Connect "get tickets" to this node.

8. **Create Google Sheets node:**  
   - Name: "Updates status"  
   - Operation: Append or update rows  
   - Matching column: "row_number"  
   - Update the "status" column with the latest ticket status  
   - Connect "If ticket resolved" (true branch) to this node.

9. **Create Freshdesk node:**  
   - Name: "get client"  
   - Operation: Get Contact  
   - ContactId: `{{$('get tickets').item.json.requester_id}}`  
   - Credentials: Freshdesk API credentials  
   - Connect "Updates status" to this node.

10. **Create Set node:**  
    - Name: "Create the email text"  
    - Assign two fields:  
      - subject: "Quick Feedback? Help Us Improve"  
      - body: Template text including ticket subject, survey link from "Set your data", and sender info.  
    - Connect "get client" to this node.

11. **Create Markdown node:**  
    - Name: "Convert the email text to HTML"  
    - Mode: markdownToHtml  
    - Input markdown from "body" field of previous node.  
    - Connect "Create the email text" to this node.

12. **Create Email Send node:**  
    - Name: "Send Email"  
    - To Email: `{{$('get client').first().json.email}}`  
    - From Email: static sender email (e.g., "thomas@pollup.net")  
    - Subject: from "Create the email text"  
    - HTML Body: from "Convert the email text to HTML"  
    - Credentials: SMTP or email provider credentials  
    - Connect "Convert the email text to HTML" to this node.

13. **Create Form Trigger node:**  
    - Name: "Survey"  
    - Form Title: "Quick Feedback"  
    - Fields:  
      - Dropdown (1-5) for satisfaction rating (required)  
      - Textarea for comments (required)  
    - Activate webhook and note the webhook URL.

14. **Create Google Sheets node:**  
    - Name: "Save survey to google sheet"  
    - Operation: Append  
    - DocumentId: Your "Feedback freshdesk" Google Sheets document ID  
    - SheetName: "Sheet1" (or corresponding sheet)  
    - Auto-map incoming survey data fields to columns  
    - Connect "Survey" form trigger to this node.

15. **Update "Set your data" node:**  
    - Set the "survey link" field to the production webhook URL of the "Survey" node.

16. **Add Sticky Note nodes (optional):**  
    - Add notes to remind about Google Sheets creation, credential setup, and customization instructions as per the original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                              | Context or Link                                                                                       |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| Contact me if you need modifications, help with this workflow, or other n8n, Make, Langchain, or Langgraph workflows: thomas@pollup.net                   | Contact email                                                                                         |
| Take a look at my other workflows here: https://n8n.io/creators/zeerobug/                                                                                  | Creator portfolio                                                                                     |
| Set your survey link to the production URL of the "Survey" node and activate the workflow to enable survey collection                                     | Workflow activation instruction                                                                       |
| Create empty Google Sheets for ticket storage and survey response storage before running workflow                                                          | Setup instruction                                                                                    |
| Connect Freshdesk nodes with proper API credentials; ensure rate limits and API permissions are respected                                                | Integration note                                                                                      |
| Email node can be replaced with Gmail node or any other email provider node as needed                                                                     | Customization suggestion                                                                             |
| Survey questions can be modified in the Form Trigger node to suit specific feedback requirements                                                          | Survey customization instruction                                                                     |

---

This documentation enables clear understanding, modification, and re-creation of the "Automate CSAT Surveys with Freshdesk & Store Responses in Google Sheets" workflow. It highlights key configurations, dependencies, and integration points critical for operational success and troubleshooting.