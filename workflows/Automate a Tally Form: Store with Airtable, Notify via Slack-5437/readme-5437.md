Automate a Tally Form: Store with Airtable, Notify via Slack

https://n8nworkflows.xyz/workflows/automate-a-tally-form--store-with-airtable--notify-via-slack-5437


# Automate a Tally Form: Store with Airtable, Notify via Slack

### 1. Workflow Overview

This workflow automates the processing of submissions from a Tally form by capturing the form data via a webhook, cleaning and restructuring it, storing it as records in Airtable, and finally sending notification emails. It is designed to streamline lead capture or client request handling by eliminating manual data entry and providing instant communication.

**Use Cases:**  
- Automatically capturing form submissions for CRM or client management  
- Synchronizing form data with Airtable databases  
- Sending confirmation emails to form respondents  

**Logical Blocks:**  
- **1.1 Input Reception:** Receiving Tally form submission via webhook  
- **1.2 Data Cleaning:** Transforming raw Tally form data into a structured format  
- **1.3 Data Storage:** Creating new records in Airtable with the cleaned data  
- **1.4 Post-Storage Processing:** Triggering follow-up actions after storing data  
- **1.5 Email Notification:** Sending a thank-you email to the submitter

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block receives incoming HTTP POST requests from Tally form submissions via a webhook node. It captures the raw form data payload to start the workflow.

- **Nodes Involved:**  
  - Webhook : Tally

- **Node Details:**  
  - **Webhook : Tally**  
    - Type: Webhook node  
    - Configuration: Listens on a unique path for POST requests  
    - Key expressions: Receives full Tally form response in JSON under `$json.body.data.fields`  
    - Input: External HTTP POST from Tally form integration  
    - Output: Raw form response JSON  
    - Edge cases: Failed webhook triggers if Tally integration is misconfigured or network issues occur; invalid payload structure may cause downstream failures

#### 1.2 Data Cleaning

- **Overview:**  
  This block restructures the array of question-answer pairs from Tally into a flat JSON object with named fields for easier consumption in later nodes.

- **Nodes Involved:**  
  - Edit Fields

- **Node Details:**  
  - **Edit Fields**  
    - Type: Set node  
    - Configuration: Assigns new string fields by extracting specific array entries from `$json.body.data.fields` via index access, mapping values to keys: full_name, company_name, job_title, email, phone_number  
    - Key expressions: Uses expressions like `={{ $json.body.data.fields[0].value }}` to map each field  
    - Input: Raw Tally form data from webhook node  
    - Output: Structured JSON with explicit keys for each form field  
    - Edge cases: If the array `fields` is missing entries or changes order, values may be misassigned or undefined; no error handling for missing fields

#### 1.3 Data Storage

- **Overview:**  
  This block creates a new record in an Airtable base with the cleaned form data, effectively storing form responses in a structured database.

- **Nodes Involved:**  
  - Airtable : Create a record

- **Node Details:**  
  - **Airtable : Create a record**  
    - Type: Airtable node  
    - Configuration: Operation set to “create,” targeting a specific Base and Table identified by IDs; fields mapped explicitly from cleaned JSON keys to Airtable columns: Email, Full Name, Job Title, Company Name, Phone Number  
    - Credentials: Airtable Personal Access Token required  
    - Input: Cleaned fields from Edit Fields node  
    - Output: API response confirming record creation  
    - Edge cases: API authentication failure, rate limiting, invalid field mapping, or table/field schema changes can cause failures

#### 1.4 Post-Storage Processing

- **Overview:**  
  This block waits for the Airtable record creation to complete before triggering the final email notification, ensuring data persistence before outreach.

- **Nodes Involved:**  
  - Wait

- **Node Details:**  
  - **Wait**  
    - Type: Wait node  
    - Configuration: Default wait with no delay parameters (effectively a pass-through or placeholder for synchronization)  
    - Input: Airtable record creation completion  
    - Output: Passes data to email notification node  
    - Edge cases: If configured with a delay, could cause slow workflows; here no delay means minimal risk

#### 1.5 Email Notification

- **Overview:**  
  Sends a thank-you email to the form submitter using Gmail, confirming receipt of their submission with a summary.

- **Nodes Involved:**  
  - GMAIL : Send a message

- **Node Details:**  
  - **GMAIL : Send a message**  
    - Type: Gmail node  
    - Configuration: Sends an email to the address in `$json.fields.Email` with a subject “Thanks for reaching out!” and an HTML-formatted message summarizing the submission (Full Name, Company Name, Job Title)  
    - Credentials: Gmail OAuth2 account required  
    - Input: Data passed from Wait node  
    - Output: Email send confirmation  
    - Edge cases: Authentication errors, quota limits, invalid email addresses, or HTML injection risks if input is not sanitized

---

### 3. Summary Table

| Node Name             | Node Type         | Functional Role                    | Input Node(s)       | Output Node(s)           | Sticky Note                                                                                                              |
|-----------------------|-------------------|----------------------------------|---------------------|--------------------------|--------------------------------------------------------------------------------------------------------------------------|
| Webhook : Tally       | Webhook           | Receive Tally form submission    | —                   | Edit Fields              | ## 🧩 Step 1 – Connect Tally to n8n ... (explains webhook setup for Tally integration)                                     |
| Edit Fields           | Set               | Clean and restructure raw data   | Webhook : Tally     | Airtable : Create a record | ## 🛠 Step 2 – Clean the Tally response ... (details transforming fields array into key-value pairs)                      |
| Airtable : Create a record | Airtable      | Store cleaned data in Airtable   | Edit Fields          | Wait                     | ## 🧱 Step 3 – Send the cleaned data to Airtable ... (instructions to create Airtable records)                             |
| Wait                  | Wait              | Synchronize before next action   | Airtable : Create a record | GMAIL : Send a message |                                                                                                                          |
| GMAIL : Send a message | Gmail             | Send confirmation email to user  | Wait                 | —                        |                                                                                                                          |
| Sticky Note            | Sticky Note       | Workflow goal description        | —                    | —                        | ## 🎯 Workflow Goal ... (overview of automated flow: Tally → Airtable → Slack)                                            |
| Sticky Note1           | Sticky Note       | Instructions for webhook setup   | —                    | —                        | ## 🧩 Step 1 – Connect Tally to n8n ... (detailed webhook setup steps)                                                    |
| Sticky Note2           | Sticky Note       | Data cleaning explanation        | —                    | —                        | ## 🛠 Step 2 – Clean the Tally response ... (transform fields array to object)                                            |
| Sticky Note3           | Sticky Note       | Airtable integration instructions| —                    | —                        | ## 🧱 Step 3 – Send the cleaned data to Airtable ... (setting up Airtable node)                                           |
| Sticky Note4           | Sticky Note       | Slack notification instructions  | —                    | —                        | ## 📬 Step 4 – Send an automatic Slack message ... (note: Slack node is mentioned in note but not present in current workflow JSON) |
| Sticky Note5           | Sticky Note       | Workflow completion message      | —                    | —                        | ## ✅ Step 5 – Workflow complete ... (summary of automation benefits)                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Add a **Webhook** node named “Webhook : Tally.”  
   - Set HTTP Method to `POST`.  
   - Note the webhook URL generated.

2. **Configure Tally Integration**  
   - In your Tally form, go to **Settings > Integrations > Webhooks**.  
   - Paste the webhook URL from n8n and enable it to send submissions.

3. **Add Set Node to Clean Data**  
   - Add a **Set** node named “Edit Fields.”  
   - Use expressions to assign fields from Tally’s raw array:  
     - `full_name = {{$json.body.data.fields[0].value}}`  
     - `company_name = {{$json.body.data.fields[1].value}}`  
     - `job_title = {{$json.body.data.fields[2].value}}`  
     - `email = {{$json.body.data.fields[3].value}}`  
     - `phone_number = {{$json.body.data.fields[4].value}}`  
   - Connect the Webhook node output to this Set node.

4. **Set Up Airtable Create Record Node**  
   - Add an **Airtable** node named “Airtable : Create a record.”  
   - Set operation to **Create**.  
   - Connect Airtable credentials (Personal Access Token).  
   - Select your Airtable Base and Table.  
   - Map the fields from the Set node:  
     - Email → `{{$json.email}}`  
     - Full Name → `{{$json.full_name}}`  
     - Job Title → `{{$json.job_title}}`  
     - Company Name → `{{$json.company_name}}`  
     - Phone Number → `{{$json.phone_number}}`  
   - Connect “Edit Fields” node to this Airtable node.

5. **Add Wait Node (Optional Synchronization)**  
   - Add a **Wait** node named “Wait.”  
   - No delay parameters needed.  
   - Connect the Airtable node output to this Wait node.

6. **Add Gmail Node to Send Email**  
   - Add a **GMAIL : Send a message** node.  
   - Connect Gmail OAuth2 credentials.  
   - Set “Send To” to `{{$json.fields.Email}}`.  
   - Email subject: “Thanks for reaching out!”  
   - Message body (HTML):  
     ```
     <p>Hi {{$json.fields['Full Name']}},</p>
     <p>Thanks for reaching out! We’ve received your request and our team will get back to you as soon as possible.</p>
     <p><strong>Here’s a quick summary:</strong></p>
     <ul>
       <li><strong>Company:</strong> {{$json.fields['Company Name']}}</li>
       <li><strong>Job Title:</strong> {{$json.fields['Job Title']}}</li>
     </ul>
     <p>We’ll be in touch very soon!</p>
     <p>— The Team</p>
     ```
   - Connect the Wait node output to this Gmail node.

7. **Activate Workflow**  
   - Save and activate the workflow.  
   - Test with Tally form submission to verify end-to-end automation.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                 | Context or Link                                                                                                           |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------|
| **Workflow automates form response handling from Tally to Airtable and sends confirmation emails via Gmail.**                                                                                                                | Workflow Goal Sticky Note                                                                                                  |
| **Tally webhook setup requires copying n8n webhook URL into Tally integration settings for automatic triggers.**                                                                                                              | Sticky Note1                                                                                                              |
| **Data cleaning transforms Tally’s fields array into simple key-value pairs for better usability downstream.**                                                                                                               | Sticky Note2                                                                                                              |
| **Airtable node requires a personal access token and correct base/table selection to store data properly.**                                                                                                                  | Sticky Note3                                                                                                              |
| **Slack notification instructions are included as a sticky note but Slack node is not present in current workflow JSON; consider adding if Slack alerts are desired.**                                                      | Sticky Note4                                                                                                              |
| **Workflow completion note emphasizes elimination of manual data handling and ensures instant data availability and team notification.**                                                                                    | Sticky Note5                                                                                                              |

---

**Disclaimer:** The provided text stems exclusively from an n8n automation workflow. It complies fully with content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.