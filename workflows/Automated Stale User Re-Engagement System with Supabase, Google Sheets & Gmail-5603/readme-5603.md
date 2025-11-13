Automated Stale User Re-Engagement System with Supabase, Google Sheets & Gmail

https://n8nworkflows.xyz/workflows/automated-stale-user-re-engagement-system-with-supabase--google-sheets---gmail-5603


# Automated Stale User Re-Engagement System with Supabase, Google Sheets & Gmail

---

### 1. Workflow Overview

This workflow is designed as an **Automated Stale User Re-Engagement System** that targets users who have not logged into a service for over 30 days. It leverages **Supabase** to retrieve stale user data, processes and cleans the data, stores it in a **Google Sheet**, and sends personalized, branded **HTML emails** to these users using **Gmail**. The workflow runs automatically every night at midnight but can also be triggered manually.

The workflow is logically divided into the following blocks:

- **1.1 Scheduling and Triggering**: Defines when and how the workflow starts (daily schedule or manual trigger).
- **1.2 Data Preparation and Retrieval**: Calculates the 30-day cutoff date and fetches stale users from Supabase.
- **1.3 Data Cleaning and Sheet Management**: Removes duplicate users, clears old sheet data, and updates Google Sheets with fresh user info.
- **1.4 AI Email Generation**: Uses an AI agent to produce a dynamic HTML email template personalized with user data.
- **1.5 Email Sending**: Sends the generated emails individually to each stale user via Gmail.
- **1.6 Output Cleaning and Parsing**: Cleans and parses AI output JSON to extract usable email subject and body.
- **1.7 Documentation and Notes**: Includes sticky notes explaining workflow purpose and steps for users.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduling and Triggering

**Overview:**  
This block controls how the workflow is initiated — either automatically every day at midnight or manually via a button.

**Nodes Involved:**  
- Schedule Trigger  
- When clicking ‘Execute workflow’

**Node Details:**

- **Schedule Trigger**  
  - *Type:* Schedule Trigger  
  - *Role:* Automatically triggers the workflow daily at midnight (cron expression: `0 0 * * *`).  
  - *Configuration:* Set with a cron interval to run once per day at 00:00.  
  - *Inputs:* None (trigger node).  
  - *Outputs:* Starts the workflow by connecting to Clear Sheet and Edit Fields nodes.  
  - *Edge cases:* If n8n scheduler is disabled, the workflow won't run automatically. Cron misconfiguration may cause timing issues.

- **When clicking ‘Execute workflow’**  
  - *Type:* Manual Trigger  
  - *Role:* Allows manual execution of the workflow, primarily for testing or ad-hoc runs.  
  - *Configuration:* Default manual trigger, no parameters.  
  - *Inputs:* None (trigger node).  
  - *Outputs:* Connects directly to the AI Agent node for immediate email generation.  
  - *Edge cases:* No automatic scheduling; manual trigger requires user intervention.

---

#### 1.2 Data Preparation and Retrieval

**Overview:**  
This block calculates the cutoff date for stale users and fetches those users from Supabase.

**Nodes Involved:**  
- Edit Fields  
- HTTP Request

**Node Details:**

- **Edit Fields**  
  - *Type:* Set node  
  - *Role:* Computes the date 30 days ago to use as a filter for stale users.  
  - *Configuration:* Defines a string variable `thirtyDaysAgo` using JavaScript Date methods, subtracting 30 days and formatting as ISO date (YYYY-MM-DD).  
  - *Key Expression:* `={{ new Date(Date.now() - 30*24*60*6*1000).toISOString().slice(0,10) }}` (Note: there is a typo in the multiplier; correct would be `30*24*60*60*1000` for milliseconds).  
  - *Inputs:* Trigger from Schedule Trigger.  
  - *Outputs:* Passes `thirtyDaysAgo` to the HTTP Request node.  
  - *Edge cases:* Incorrect time calculation may result in wrong date filtering.

- **HTTP Request**  
  - *Type:* HTTP Request  
  - *Role:* Queries Supabase REST API to retrieve stale users whose last sign-in date is less or equal to `thirtyDaysAgo`.  
  - *Configuration:* URL uses dynamic expression with `thirtyDaysAgo` to filter view `stale_users_view`. Authentication uses predefined Supabase API credentials.  
  - *Key Expression:* URL parameter `last_sign_in_at=lte.{{ $json.thirtyDaysAgo }}`.  
  - *Inputs:* From Edit Fields node.  
  - *Outputs:* Sends fetched user data to Remove Duplicates node.  
  - *Edge cases:* Potential authentication failures with Supabase API, network timeouts, or empty data if no stale users found.

---

#### 1.3 Data Cleaning and Sheet Management

**Overview:**  
This block ensures data cleanliness by removing duplicate emails, clearing the Google Sheet, and appending updated user data.

**Nodes Involved:**  
- Remove Duplicates  
- Clear Sheet  
- Update Google Sheet with Leads

**Node Details:**

- **Remove Duplicates**  
  - *Type:* Remove Duplicates  
  - *Role:* Removes duplicate entries based on the `email` field to ensure unique stale user records.  
  - *Configuration:* Compares entries by the `email` field only.  
  - *Inputs:* From HTTP Request node.  
  - *Outputs:* Passes unique user records to Update Google Sheet with Leads.  
  - *Edge cases:* Case sensitivity in emails could cause duplicates not to be removed if not normalized.

- **Clear Sheet**  
  - *Type:* Google Sheets  
  - *Role:* Clears existing data from the target Google Sheet, preserving the first row (typically headers).  
  - *Configuration:* Operation set to “clear”; sheet and document IDs specified; uses Service Account authentication.  
  - *Inputs:* Triggered by Schedule Trigger (runs before data update).  
  - *Outputs:* No direct output connection, but ensures sheet is empty before appending new data.  
  - *Edge cases:* Google API quota limits, authentication errors, or sheet access permissions issues.

- **Update Google Sheet with Leads**  
  - *Type:* Google Sheets  
  - *Role:* Appends or updates rows in the Google Sheet with stale user data (Name, Email, Last Signed In date).  
  - *Configuration:* Defines columns with expressions to derive Name (from email prefix, capitalized), Email, and Last Signed In @; matching on “Last Signed In @” for updates; uses Service Account authentication.  
  - *Key Expressions:*  
    - Name: `={{ $json.email.split('@')[0].charAt(0).toUpperCase() + $json.email.split('@')[0].slice(1) }}`  
    - Email: `={{ $json.email }}`  
    - Last Signed In @: `={{ $json.last_sign_in_at }}`  
  - *Inputs:* From Remove Duplicates node.  
  - *Outputs:* Passes updated data to Send a message node.  
  - *Edge cases:* Data type mismatches, Google Sheets API limits, partial updates if errors occur.

---

#### 1.4 AI Email Generation

**Overview:**  
Generates a dynamic, branded, friendly HTML email template using AI based on the stale user data.

**Nodes Involved:**  
- AI Agent  
- Structured Output Parser  
- Code

**Node Details:**

- **AI Agent**  
  - *Type:* Langchain Agent (AI)  
  - *Role:* Produces JSON output with email subject and HTML message body using placeholders for user data, based on a detailed prompt describing content and brand guidelines.  
  - *Configuration:*  
    - Text prompt instructs the AI to generate one friendly HTML email template with placeholders for Name, Email, and Last Signed In @.  
    - Brand info and call-to-action included.  
    - Output format strictly JSON with fields `subject` and `message`.  
  - *Inputs:* Triggered manually or connected via Google Vertex Chat Model (optional).  
  - *Outputs:* AI output passed to Structured Output Parser.  
  - *Edge cases:* AI may return malformed JSON or unexpected output; prompt clarity is critical.

- **Structured Output Parser**  
  - *Type:* Langchain Structured Output Parser  
  - *Role:* Parses AI output JSON to ensure it matches the schema `{ subject: "", message: "" }`.  
  - *Configuration:* Manual schema definition for subject and message strings.  
  - *Inputs:* From AI Agent.  
  - *Outputs:* Passes structured data to Code node.  
  - *Edge cases:* Parsing failure if AI output deviates from schema.

- **Code**  
  - *Type:* Code node (JavaScript)  
  - *Role:* Cleans AI output by trimming code block markdown, parses JSON, and adds `subject` and `message` fields to the item JSON for downstream use.  
  - *Configuration:* Custom JS code that removes ```json or ``` wrappers and parses JSON.  
  - *Inputs:* From AI Agent via Structured Output Parser.  
  - *Outputs:* Passes cleaned email content to downstream nodes or manual trigger path.  
  - *Edge cases:* Parsing errors if input is malformed or empty.

---

#### 1.5 Email Sending

**Overview:**  
Sends personalized HTML emails to each stale user using Gmail with the generated content.

**Nodes Involved:**  
- Send a message

**Node Details:**

- **Send a message**  
  - *Type:* Gmail node  
  - *Role:* Sends personalized emails to each user with dynamic HTML content and subject line.  
  - *Configuration:*  
    - Recipient email: dynamic from `{{ $json.Email }}`.  
    - Subject: dynamic from `{{ $json.Name }}` in “We miss you, {{ $json.Name }}!”  
    - HTML message body uses a static template with embedded placeholders for Name, Email, Last Signed In @, and brand info.  
    - Reply-to address is set to a specific email (`venkibvb5192@gmail.com`).  
    - Authentication via Google Service Account credentials.  
  - *Inputs:* From Update Google Sheet with Leads node.  
  - *Outputs:* None (terminal node).  
  - *Edge cases:* Gmail API rate limits, authentication errors, invalid email addresses.

---

#### 1.6 Documentation and Notes

**Overview:**  
Provides contextual information and explanations for users reviewing or maintaining the workflow.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note1

**Node Details:**

- **Sticky Note**  
  - *Type:* Sticky Note  
  - *Role:* Explains the email generation purpose, AI usage, and dynamic placeholders for the email content.  
  - *Content Highlights:*  
    - Workflow generates friendly HTML email templates with placeholders.  
    - Emphasizes dynamic content with no hardcoded data.  
  - *Position:* Near AI nodes for contextual clarity.

- **Sticky Note1**  
  - *Type:* Sticky Note  
  - *Role:* Describes the overall workflow steps, scheduling, data sources, and email content strategy.  
  - *Content Highlights:*  
    - Daily automatic execution.  
    - Data flow summary from Supabase to Google Sheets to Gmail.  
    - Email content overview.  
  - *Position:* Near the start of the workflow.

---

### 3. Summary Table

| Node Name                   | Node Type                           | Functional Role                                    | Input Node(s)                      | Output Node(s)                   | Sticky Note                                                                                                           |
|-----------------------------|-----------------------------------|---------------------------------------------------|----------------------------------|---------------------------------|-----------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger            | Schedule Trigger                  | Automatically triggers workflow daily at midnight | None                             | Clear Sheet, Edit Fields         | ## 🛠 Automated Stale Users Re‑engagement Workflow  (sticky note covers multiple nodes including Schedule Trigger)     |
| Clear Sheet                | Google Sheets                    | Clears old data in Google Sheet (except first row) | Schedule Trigger                 | None                            | ## 🛠 Automated Stale Users Re‑engagement Workflow                                                                   |
| Edit Fields                | Set                             | Calculates 30-day cutoff date for stale users      | Schedule Trigger                 | HTTP Request                    | ## 🛠 Automated Stale Users Re‑engagement Workflow                                                                   |
| HTTP Request               | HTTP Request                    | Queries Supabase to get stale users                 | Edit Fields                     | Remove Duplicates               | ## 🛠 Automated Stale Users Re‑engagement Workflow                                                                   |
| Remove Duplicates          | Remove Duplicates                | Removes duplicate emails                            | HTTP Request                    | Update Google Sheet with Leads  | ## 🛠 Automated Stale Users Re‑engagement Workflow                                                                   |
| Update Google Sheet with Leads | Google Sheets                    | Appends or updates stale user data in Google Sheet | Remove Duplicates               | Send a message                 | ## 🛠 Automated Stale Users Re‑engagement Workflow                                                                   |
| Send a message             | Gmail                           | Sends personalized HTML emails to users            | Update Google Sheet with Leads  | None                           | ## 🛠 Automated Stale Users Re‑engagement Workflow                                                                   |
| When clicking ‘Execute workflow’ | Manual Trigger                  | Manual execution trigger                            | None                            | AI Agent                       |                                                                                                                       |
| AI Agent                  | Langchain Agent                 | Generates HTML email template JSON                  | When clicking ‘Execute workflow’ | Code                          | ## 📨  Generate Email Description for Stale Leads (sticky note near AI nodes)                                         |
| Structured Output Parser   | Langchain Structured Output Parser | Parses AI output into structured JSON               | AI Agent                       | AI Agent                      | ## 📨  Generate Email Description for Stale Leads                                                                     |
| Code                      | Code                            | Cleans and parses AI output JSON                     | AI Agent                       | None                          | ## 📨  Generate Email Description for Stale Leads                                                                     |
| Google Vertex Chat Model   | Langchain LM Chat Google Vertex | Optional AI model interface (not connected in main flow) | None                         | AI Agent                      |                                                                                                                       |
| Sticky Note                | Sticky Note                     | Explains AI email generation block                   | None                           | None                          | ## 📨  Generate Email Description for Stale Leads                                                                     |
| Sticky Note1               | Sticky Note                     | Explains overall workflow purpose and steps          | None                           | None                          | ## 🛠 Automated Stale Users Re‑engagement Workflow                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Set cron expression to `0 0 * * *` (runs daily at midnight).  
   - Connect its output to the next nodes: Clear Sheet and Edit Fields.

2. **Create a Clear Sheet node**  
   - Type: Google Sheets  
   - Operation: Clear  
   - Document ID: Use your Google Sheets document ID (e.g., `1HcTc5dD05NfCbLjOPH3UBERhrrGX41EpGmW-2HvadtE`).  
   - Sheet Name: `gid=0` (Sheet1)  
   - Keep First Row: Enabled (to preserve headers).  
   - Authentication: Connect a Google Service Account credential with Sheet edit permissions.  
   - Connect input from Schedule Trigger.

3. **Create an Edit Fields (Set) node**  
   - Type: Set  
   - Add a string field named `thirtyDaysAgo`.  
   - Set its value with the expression:  
     `={{ new Date(Date.now() - 30*24*60*60*1000).toISOString().slice(0,10) }}`  
   - Connect input from Schedule Trigger.

4. **Create an HTTP Request node**  
   - Type: HTTP Request  
   - Method: GET (default)  
   - URL:  
     `https://[YOUR_SUPABASE_PROJECT].supabase.co/rest/v1/stale_users_view?last_sign_in_at=lte.{{ $json.thirtyDaysAgo }}`  
   - Authentication: Use Supabase API credentials (predefined credential with API key).  
   - Connect input from Edit Fields node.

5. **Create a Remove Duplicates node**  
   - Type: Remove Duplicates  
   - Compare: selected fields  
   - Fields to Compare: `email`  
   - Connect input from HTTP Request node.

6. **Create an Update Google Sheet with Leads node**  
   - Type: Google Sheets  
   - Operation: Append Or Update  
   - Document ID: Same Google Sheet as Clear Sheet  
   - Sheet Name: `gid=0` (Sheet1)  
   - Columns: Define schema with these columns:  
     - Name (string)  
     - Email (string)  
     - Last Signed In @ (string)  
   - Mapping:  
     - Name: `={{ $json.email.split('@')[0].charAt(0).toUpperCase() + $json.email.split('@')[0].slice(1) }}`  
     - Email: `={{ $json.email }}`  
     - Last Signed In @: `={{ $json.last_sign_in_at }}`  
   - Matching Columns: `Last Signed In @`  
   - Authentication: Google Service Account credential.  
   - Connect input from Remove Duplicates node.

7. **Create a Send a message node**  
   - Type: Gmail  
   - Send To: `={{ $json.Email }}`  
   - Subject: `=We miss you, {{ $json.Name }}!`  
   - Message: Paste the full HTML email template (as in the original), using placeholders: `{{ $json.Name }}`, `{{ $json.Email }}`, `{{ $json['Last Signed In @'] }}`.  
   - Reply-To: set as desired (e.g., `venkibvb5192@gmail.com`).  
   - Authentication: Google Service Account or OAuth2 Gmail credentials.  
   - Connect input from Update Google Sheet with Leads node.

8. **Create Manual Trigger node (optional)**  
   - Type: Manual Trigger  
   - Connect output to AI Agent node (below) for manual runs.

9. **Create AI Agent node**  
   - Type: Langchain Agent  
   - Text prompt: Use the provided detailed prompt instructing the AI to generate a JSON with subject and HTML message placeholders.  
   - Set to execute once per trigger.  
   - Connect output to Structured Output Parser node.

10. **Create Structured Output Parser node**  
    - Type: Langchain Structured Output Parser  
    - Input Schema: `{ "subject": "", "message": "" }`  
    - Connect input from AI Agent node.  
    - Connect output to Code node.

11. **Create Code node**  
    - Type: Code (JavaScript)  
    - Paste JS code that trims code block markdown and parses JSON to extract `subject` and `message`.  
    - Connect input from Structured Output Parser node.

12. **(Optional) Create Google Vertex Chat Model node**  
    - Type: Langchain LM Chat Google Vertex  
    - Configure with project ID and model name.  
    - Connect output to AI Agent node if integrating advanced AI model.  

13. **Create Sticky Notes**  
    - Add notes near relevant nodes describing the workflow’s purpose and details for maintainers.

14. **Credential Setup**  
    - Supabase API Credential: API key with access to the Supabase project.  
    - Google Service Account Credential: With permissions for Google Sheets and Gmail API.  
    - OAuth2 Gmail Credential: Alternative for Gmail node if desired.  

15. **Test the workflow**  
    - Run manual trigger to validate AI email generation and sending.  
    - Confirm data retrieval, sheet updates, and email delivery.  
    - Schedule trigger will run automatically at midnight.

---

### 5. General Notes & Resources

| Note Content                                                                                                                           | Context or Link                                                                                       |
|----------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| Workflow runs automatically every night at midnight to re-engage users inactive for 30+ days.                                          | Scheduling section                                                                                    |
| Uses AI generation via Langchain Agent to produce fully dynamic, branded HTML email templates with placeholders.                      | AI Email Generation block                                                                            |
| Email template includes user-friendly intro, dynamic table with user data, and call-to-action linking to https://thalathikka.com.     | Email Sending node and Sticky Note                                                                   |
| Google Sheets used as intermediate data store for stale user data, cleared and updated each run.                                       | Data Cleaning and Sheet Management block                                                             |
| Supabase REST API used to query PostgreSQL view `stale_users_view` filtered by last sign-in date ≤ 30 days ago.                       | Data Preparation and Retrieval block                                                                 |
| Careful credential management required: Supabase API key, Google Service Account for Sheets and Gmail.                                 | Credential Setup                                                                                      |
| Possible improvements: fix time calculation typo in `Edit Fields` node, normalize email case before duplicate removal.                | Edge cases and troubleshooting                                                                       |
| Documentation notes embedded as sticky notes for maintainers and future developers.                                                    | Sticky Notes nodes                                                                                   |
| For Gmail node, watch out for API quota and authentication issues when sending bulk emails.                                            | Email Sending node edge cases                                                                         |

---

**Disclaimer:**  
The provided content is exclusively derived from an automated workflow created using n8n, an integration and automation platform. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.

---