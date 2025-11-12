Automated 4-Stage Email Follow-Up System with AI Personalization and Database Tracking

https://n8nworkflows.xyz/workflows/automated-4-stage-email-follow-up-system-with-ai-personalization-and-database-tracking-9108


# Automated 4-Stage Email Follow-Up System with AI Personalization and Database Tracking

### 1. Workflow Overview

This workflow automates a **4-stage email follow-up system** using AI to personalize emails and a database to track lead status and follow-up scheduling. The primary use case is for sales teams, recruiters, or agencies managing cold outreach and nurturing leads by sending personalized follow-ups at scheduled intervals.

The workflow is logically organized into these blocks:

- **1.1 Input Reception & Lead Filtering**: Scheduled daily trigger fetches all leads from a NocoDB database, filters out those not needing follow-up today or marked "Not Interested".
- **1.2 Follow-up Stage Routing**: Uses a Switch node to determine which follow-up stage (1-4) applies based on database flags indicating which follow-ups were already sent.
- **1.3 AI Email Personalization**: For each follow-up stage, a Groq AI model personalizes the email template to insert the recipient's first name without altering any other content.
- **1.4 Email Sending & Database Update**: Sends the personalized email via SMTP and updates the database to mark the follow-up as sent and schedule the next follow-up date.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Lead Filtering

- **Overview:**  
  This block is triggered daily at 10 AM. It fetches all lead records from NocoDB and filters for those who have a follow-up scheduled for today or earlier, excluding leads marked as "Not Interested".

- **Nodes Involved:**  
  - Schedule Trigger  
  - Get many rows (NocoDB)  
  - Filter  
  - Filter1  

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Configured to trigger daily at 10:00 AM.  
    - Starts the workflow automatically each day.  
    - No input connections; output connects to “Get many rows”.  
    - Possible failures: Workflow not triggering if n8n instance down.

  - **Get many rows** (NocoDB)  
    - Type: NocoDB node (getAll operation)  
    - Retrieves all records from the "mznkwe3wz0dofde" table in project "psinm4yvd05vnvy".  
    - Uses NocoDB API Token credentials.  
    - Returns all records without limit.  
    - Inputs: Trigger from Schedule Trigger. Outputs: all rows to Filter.  
    - Possible failures: API token invalid, database unreachable.

  - **Filter**  
    - Type: Filter node  
    - Filters rows where "Next Follow up/Contact" is equal to today’s date or earlier.  
    - Uses an OR condition for equality or before comparison with current date formatted as M/dd/yy.  
    - Inputs: Leads from Get many rows. Outputs: matched leads to Filter1.  
    - Edge cases: Date format mismatches could cause filtering errors.

  - **Filter1**  
    - Type: Filter node  
    - Filters out leads where "Not Interested" field is NOT empty (i.e., excludes leads marked not interested).  
    - Strict string empty check on "Not Interested".  
    - Inputs: Filter output. Outputs: leads needing follow-up to Switch node.  
    - Edge cases: Null or unexpected values in "Not Interested" field.

---

#### 1.2 Follow-up Stage Routing

- **Overview:**  
  Routes each lead to the appropriate follow-up stage based on which follow-up emails have already been sent (tracked by fields "Follow Up 1 (Day 3) Y/N", etc.).

- **Nodes Involved:**  
  - Switch  

- **Node Details:**

  - **Switch**  
    - Type: Switch node  
    - Has 4 outputs corresponding to Follow Up 1 through Follow Up 4.  
    - Conditions check if the respective "Follow Up X (Day Y) Y/N" field is empty (null or "").  
    - Routes lead to first follow-up stage not yet completed.  
    - Inputs: Filter1 output. Outputs: to Follow Up Message nodes.  
    - Edge cases: If all follow-ups marked “Y”, lead is not routed further (no default output). Fields must match exactly including case.

---

#### 1.3 AI Email Personalization

- **Overview:**  
  For each follow-up stage, the workflow uses a Groq AI model to personalize the email template by replacing only the recipient’s name, preserving all other content exactly.

- **Nodes Involved:**  
  - Groq Chat Model (Follow Up 1)  
  - Groq Chat Model1 (Follow Up 2)  
  - Groq Chat Model2 (Follow Up 3)  
  - Groq Chat Model3 (Follow Up 4)  
  - Follow Up Message 1  
  - Follow Up Message 2  
  - Follow Up Message 3  
  - Follow Up Message 4  

- **Node Details:**

  - **Groq Chat Model / Groq Chat Model1 / Groq Chat Model2 / Groq Chat Model3**  
    - Type: LangChain Groq Chat Model nodes  
    - Models used: "openai/gpt-oss-120b" for Follow Up 1 and 4, "openai/gpt-oss-20b" for Follow Up 2 and 3.  
    - Credentials: Groq API key.  
    - Inputs: Lead data from Switch node. Outputs: to respective Follow Up Message nodes.  
    - Edge cases: API key invalid, model limits, rate limits.

  - **Follow Up Message 1 / 2 / 3 / 4**  
    - Type: LangChain ChainLLM nodes  
    - Each configured with a prompt instructing AI to replace ONLY the recipient’s name in the template, maintaining all formatting and other text intact.  
    - Dynamic input: recipient’s first name from lead data.  
    - Output: personalized email text only.  
    - Inputs: Groq Chat Model outputs. Outputs: send email nodes.  
    - Potential failures: prompt errors, unexpected AI output format.

---

#### 1.4 Email Sending & Database Update

- **Overview:**  
  Sends the personalized email to the lead via SMTP, then updates the NocoDB database to mark the follow-up as sent and schedule the next follow-up date.

- **Nodes Involved:**  
  - Send email  
  - Send email1  
  - Send email2  
  - Send email3  
  - Update a row (for Follow Up 2)  
  - Update a row1 (for Follow Up 1)  
  - Update a row2 (for Follow Up 3)  
  - Update a row3 (for Follow Up 4)  

- **Node Details:**

  - **Send email / Send email1 / Send email2 / Send email3**  
    - Type: Email Send (SMTP) nodes  
    - Send personalized email text from Follow Up Message nodes.  
    - Subject fixed as "RE: Quick question about Lead Qualification" (minor variation in spacing).  
    - Recipient email from lead’s email field.  
    - Credentials: SMTP account.  
    - Inputs: Follow Up Message nodes. Outputs: respective Update a row nodes.  
    - Edge cases: SMTP credentials invalid, email delivery failures, invalid recipient email.

  - **Update a row / Update a row1 / Update a row2 / Update a row3**  
    - Type: NocoDB update operation nodes  
    - Each updates respective follow-up flag field to "Y" and sets the "Next Follow up/Contact" date for the next follow-up (except Update a row3 which does not schedule next date).  
    - Uses current lead’s ID for updating the correct row.  
    - Date increments: +4 or +5 days from now, depending on stage.  
    - Inputs: Send email nodes. Outputs: none (end of branch).  
    - Edge cases: Field name or ID mismatch, API token invalid, date formatting issues.

---

### 3. Summary Table

| Node Name         | Node Type                 | Functional Role                          | Input Node(s)          | Output Node(s)           | Sticky Note                                                                                                                        |
|-------------------|---------------------------|----------------------------------------|-----------------------|--------------------------|------------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger  | Schedule Trigger           | Daily workflow trigger at 10 AM        | -                     | Get many rows            |                                                                                                                                     |
| Get many rows     | NocoDB getAll             | Fetch all leads from database          | Schedule Trigger       | Filter                   |                                                                                                                                     |
| Filter            | Filter                    | Filter leads with follow-up due today  | Get many rows          | Filter1                  |                                                                                                                                     |
| Filter1           | Filter                    | Exclude leads marked "Not Interested"  | Filter                 | Switch                   |                                                                                                                                     |
| Switch            | Switch                    | Route leads to correct follow-up stage | Filter1                | Follow Up Message 1-4    |                                                                                                                                     |
| Groq Chat Model   | LangChain Chat Model (Groq)| Personalize Follow Up 1 email           | Switch (Follow Up 1)   | Follow Up Message 1      |                                                                                                                                     |
| Groq Chat Model1  | LangChain Chat Model (Groq)| Personalize Follow Up 2 email           | Switch (Follow Up 2)   | Follow Up Message 2      |                                                                                                                                     |
| Groq Chat Model2  | LangChain Chat Model (Groq)| Personalize Follow Up 3 email           | Switch (Follow Up 3)   | Follow Up Message 3      |                                                                                                                                     |
| Groq Chat Model3  | LangChain Chat Model (Groq)| Personalize Follow Up 4 email           | Switch (Follow Up 4)   | Follow Up Message 4      |                                                                                                                                     |
| Follow Up Message 1| Chain LLM (LangChain)     | Insert recipient's name in Follow Up 1 | Groq Chat Model        | Send email               |                                                                                                                                     |
| Follow Up Message 2| Chain LLM (LangChain)     | Insert recipient's name in Follow Up 2 | Groq Chat Model1       | Send email1              |                                                                                                                                     |
| Follow Up Message 3| Chain LLM (LangChain)     | Insert recipient's name in Follow Up 3 | Groq Chat Model2       | Send email2              |                                                                                                                                     |
| Follow Up Message 4| Chain LLM (LangChain)     | Insert recipient's name in Follow Up 4 | Groq Chat Model3       | Send email3              |                                                                                                                                     |
| Send email        | Email Send (SMTP)         | Send Follow Up 1 email                  | Follow Up Message 1    | Update a row1            |                                                                                                                                     |
| Send email1       | Email Send (SMTP)         | Send Follow Up 2 email                  | Follow Up Message 2    | Update a row             |                                                                                                                                     |
| Send email2       | Email Send (SMTP)         | Send Follow Up 3 email                  | Follow Up Message 3    | Update a row2            |                                                                                                                                     |
| Send email3       | Email Send (SMTP)         | Send Follow Up 4 email                  | Follow Up Message 4    | Update a row3            |                                                                                                                                     |
| Update a row1     | NocoDB update             | Mark Follow Up 1 sent, schedule next   | Send email             | -                        |                                                                                                                                     |
| Update a row      | NocoDB update             | Mark Follow Up 2 sent, schedule next   | Send email1            | -                        |                                                                                                                                     |
| Update a row2     | NocoDB update             | Mark Follow Up 3 sent, schedule next   | Send email2            | -                        |                                                                                                                                     |
| Update a row3     | NocoDB update             | Mark Follow Up 4 sent, no further date | Send email3            | -                        |                                                                                                                                     |
| Sticky Note       | Sticky Note               | Overview and detailed explanation      | -                     | -                        | # Automated Email Follow-Up Workflow ... (full content in workflow)                                                                |
| Sticky Note1      | Sticky Note               | Customization and setup instructions   | -                     | -                        | ## Customizing this workflow ... (full content in workflow)                                                                        |
| Sticky Note2      | Sticky Note               | Troubleshooting guide                   | -                     | -                        | ## Troubleshooting ... (full content in workflow)                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

**Step 1: Create Database and Credentials**

1. Setup a NocoDB project and table with columns:  
   - `first_name`, `email`, `Initial Contact Date`, `Next Follow up/Contact`  
   - `Follow Up 1 (Day 3) Y/N`, `Follow Up 2 (Day 7) Y/N`, `Follow Up 3 (Day 12) Y/N`, `Follow Up 4 (Day 16) Y/N`  
   - `Not Interested` (optional for exclusion)  
2. Populate or import leads with required data.  
3. Obtain NocoDB API token for authentication.

---

**Step 2: Add Credentials in n8n**

1. Add NocoDB API credentials using the API token.  
2. Add Groq API credentials for AI personalization.  
3. Add SMTP credentials for sending emails (or Gmail OAuth if preferred).

---

**Step 3: Build Workflow Nodes**

1. **Schedule Trigger**  
   - Create a Schedule Trigger node set to daily at 10:00 AM.  
   - Connect output to next node.

2. **Get many rows (NocoDB)**  
   - Create NocoDB node with operation `getAll`.  
   - Configure with your project ID, table name, and credentials.  
   - Set to return all records.  
   - Connect input from Schedule Trigger.

3. **Filter**  
   - Create Filter node.  
   - Add condition: `(Next Follow up/Contact == today) OR (Next Follow up/Contact before today)` using expression:  
     `{{$json["Next Follow up/Contact"]}}` compared to `{{$now.format("M/dd/yy")}}`.  
   - Connect input from Get many rows.

4. **Filter1**  
   - Create another Filter node.  
   - Add condition: `Not Interested` is empty string.  
   - Connect input from Filter.

5. **Switch**  
   - Create a Switch node with 4 outputs: Follow Up 1 to 4.  
   - Add conditions for each output: check if respective `"Follow Up X (Day Y) Y/N"` field is empty string (null).  
   - Connect input from Filter1.

6. **Groq Chat Model Nodes for Personalization**  
   - For each follow-up (1-4), create a LangChain Groq Chat Model node:  
     - Use models "openai/gpt-oss-120b" for Follow Up 1 & 4, "openai/gpt-oss-20b" for Follow Up 2 & 3.  
     - Attach Groq API credentials.  
   - Connect each output of Switch node to corresponding Groq model node.

7. **Follow Up Message Nodes**  
   - For each Groq Chat Model, add a ChainLLM node with prompt to replace ONLY recipient’s name in a predefined email template.  
   - Use expressions to insert recipient’s `first_name`.  
   - Connect Groq Chat Model outputs to these nodes.

8. **Send Email Nodes**  
   - For each personalized message, create an Email Send node (SMTP).  
   - Set `toEmail` dynamically from lead’s `email` field.  
   - Use fixed subject lines as per workflow.  
   - Connect each Follow Up Message node to respective Send Email node.

9. **Update a row Nodes**  
   - For each Send Email node, create a NocoDB update node to:  
     - Mark respective Follow Up flag as "Y".  
     - Set "Next Follow up/Contact" to current date plus appropriate days (+4 or +5 days) for next stage, except last stage which does not set next date.  
     - Use lead’s ID to update correct database row.  
   - Connect Send Email nodes to these update nodes.

---

**Step 4: Finalizing**

- Activate the workflow.  
- Test with a lead having today's follow-up date.  
- Monitor emails sent and database updates.  
- Customize email templates in the ChainLLM nodes as needed.  
- Adjust follow-up intervals by editing date increments in update nodes.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                       | Context or Link                                                                                                      |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| This workflow automates personalized cold email follow-ups with AI and database tracking, ideal for sales, recruiting, and outreach.                                             | Sticky Note overview in workflow                                                                                     |
| Follow-up intervals are Day 3, 7, 12, 16 by default; adjust by changing `$now.plus(X, 'Days')` in update nodes.                                                                    | Sticky Note1 customization instructions                                                                              |
| AI personalization uses Groq API; can be swapped for OpenAI, Anthropic, or Ollama models as they use the same LangChain interface.                                               | Sticky Note1 customization instructions                                                                              |
| Replace SMTP nodes with Gmail nodes using OAuth2 for better deliverability if desired.                                                                                            | Sticky Note1 customization instructions                                                                              |
| Database fields are case-sensitive and critical; missing or misnamed fields cause workflow failure.                                                                                | Sticky Note overview and Sticky Note2 troubleshooting                                                                |
| For troubleshooting, check API keys, credentials, database project/table IDs, and email delivery issues.                                                                           | Sticky Note2 troubleshooting guide                                                                                   |
| AI personalization cost is approximately $0.001 per request with Groq; free tier available at https://console.groq.com/pricing                                                  | Sticky Note overview                                                                                                 |
| Leads marked "Not Interested" are excluded from follow-ups automatically.                                                                                                         | Sticky Note overview                                                                                                 |
| The workflow only processes leads where "Initial Contact Date" is set manually to start the follow-up sequence.                                                                   | Sticky Note overview                                                                                                 |

---

**Disclaimer:**  
This document is generated from an automated n8n workflow export. All data processed are legal and public. The workflow respects content policies and contains no illegal or offensive elements.