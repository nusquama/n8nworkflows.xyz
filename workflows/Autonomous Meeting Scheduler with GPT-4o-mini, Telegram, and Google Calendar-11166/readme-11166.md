Autonomous Meeting Scheduler with GPT-4o-mini, Telegram, and Google Calendar

https://n8nworkflows.xyz/workflows/autonomous-meeting-scheduler-with-gpt-4o-mini--telegram--and-google-calendar-11166


# Autonomous Meeting Scheduler with GPT-4o-mini, Telegram, and Google Calendar

### 1. Workflow Overview

This workflow implements an **Autonomous Meeting Scheduler** integrating **Telegram**, **GPT-4o-mini (OpenAI)**, **Google Sheets (CRM)**, **Google Calendar**, and **Gmail**. It enables users to schedule meetings through natural language messages on Telegram. The AI agent autonomously parses requests, looks up contacts, checks calendar availability, proposes times, and upon confirmation, creates calendar events and sends email invitations.

**Key Use Cases:**
- Scheduling meetings via chat without manual calendar management
- Automating contact lookup and availability checks
- Sending professional confirmations via email
- Logging and managing conversations and meetings centrally

**Logical Blocks:**

- **1.1 Trigger Reception:** Receives Telegram messages from users.
- **1.2 Data Preparation:** Extracts relevant data from the Telegram message.
- **1.3 CRM Data Loading:** Retrieves contact information from Google Sheets.
- **1.4 AI Processing:** Uses GPT-4o-mini to interpret requests, find contacts, check availability, and propose meeting times or create events.
- **1.5 Response Dispatch:** Sends messages back to Telegram with proposals or confirmations.
- **1.6 Event Creation & Email:** Creates Google Calendar events and sends confirmation emails on user approval.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger Reception

- **Overview:**  
  Captures incoming user messages from Telegram to initiate the scheduling process.

- **Nodes Involved:**  
  - Telegram Trigger

- **Node Details:**

  - **Telegram Trigger**  
    - *Type:* Telegram Trigger (n8n native)  
    - *Role:* Listens for new Telegram messages (updates of type "message")  
    - *Configuration:*  
      - Listens to "message" updates only  
      - No additional filters applied  
    - *Connections:* Output to "Prepare Data" node  
    - *Potential Failures:* Telegram API connectivity, authentication errors, message format changes  
    - *Version:* v1.2

#### 1.2 Data Preparation

- **Overview:**  
  Extracts essential data fields from the Telegram message JSON to simplify downstream processing.

- **Nodes Involved:**  
  - Prepare Data

- **Node Details:**

  - **Prepare Data**  
    - *Type:* Set Node  
    - *Role:* Maps user message text, chat ID, and user first name into named variables  
    - *Configuration:*  
      - Assigns:  
        - `user_message`: extracted from `message.text`  
        - `chat_id`: extracted from `message.chat.id`  
        - `user_name`: extracted from `message.from.first_name`  
    - *Connections:* Output to "Load CRM Data"  
    - *Edge Cases:* Missing fields in Telegram message JSON, empty messages  
    - *Version:* v3.4

#### 1.3 CRM Data Loading

- **Overview:**  
  Loads the contact database from a Google Sheets spreadsheet to enable contact lookup.

- **Nodes Involved:**  
  - Load CRM Data

- **Node Details:**

  - **Load CRM Data**  
    - *Type:* Google Sheets Node  
    - *Role:* Reads entire "CRM" sheet from a specified Google Sheet document  
    - *Configuration:*  
      - Authentication via Service Account credentials  
      - Document ID: `YOUR_GOOGLE_SHEET_ID` (replace with actual ID)  
      - Sheet Name: "CRM"  
      - Expected format: Columns for Name, Email, Phone  
    - *Connections:* Output to "AI Agent"  
    - *Edge Cases:* Invalid credentials, missing sheet or columns, API quota limits  
    - *Version:* v4.6

#### 1.4 AI Processing

- **Overview:**  
  Central logic block where the AI assistant processes the user's request, accesses CRM and calendar tools, and decides next steps.

- **Nodes Involved:**  
  - AI Agent  
  - OpenAI Model  
  - Google Calendar Tool  
  - CRM Search Tool

- **Node Details:**

  - **AI Agent**  
    - *Type:* Langchain Agent Node  
    - *Role:* Orchestrates AI reasoning using GPT-4o-mini and integrated tools  
    - *Configuration Highlights:*  
      - Input text includes current date, user message, and CRM contacts serialized as JSON  
      - System prompt instructs the assistant to:  
        1. Find contact in CRM  
        2. Check calendar availability (9 AM - 4 PM)  
        3. Propose 3 available slots  
        4. Create event after confirmation  
      - Output JSON expected with fields: message, action (propose_slots/create_event/clarify), contact_email, meeting_date, meeting_time  
      - Max Iterations: 10  
    - *Connections:*  
      - Input tools:  
        - OpenAI Model (provides GPT-4o-mini)  
        - Google Calendar Tool (for calendar queries)  
        - CRM Search Tool (for contact lookup)  
      - Outputs to "Send Response" and "Should Create Event?" nodes  
    - *Potential Failures:*  
      - AI model rate limits or downtime  
      - Misinterpretation or malformed output JSON  
      - Tool API errors (Google Calendar, Sheets)  
    - *Version:* v1.9

  - **OpenAI Model**  
    - *Type:* Langchain LM Chat (OpenAI)  
    - *Role:* Provides GPT-4o-mini language model for AI Agent  
    - *Configuration:* Model set to "gpt-4o-mini"  
    - *Connections:* Output feeds AI Agent  
    - *Potential Failures:* Auth errors, model unavailability  
    - *Version:* v1.2

  - **Google Calendar Tool**  
    - *Type:* Google Calendar Tool  
    - *Role:* Enables AI Agent to query Google Calendar availability  
    - *Configuration:*  
      - Calendar ID: `YOUR_CALENDAR_ID` (replace with actual calendar)  
    - *Connections:* Input to AI Agent as a tool  
    - *Potential Failures:* API errors, permission issues  
    - *Version:* v1.3

  - **CRM Search Tool**  
    - *Type:* Google Sheets Tool  
    - *Role:* Enables AI Agent to search contacts in CRM sheet  
    - *Configuration:* Same Google Sheet ID and sheet name "CRM" as Load CRM Data  
    - *Connections:* Input to AI Agent as a tool  
    - *Potential Failures:* Same as Load CRM Data  
    - *Version:* v4.6

#### 1.5 Response Dispatch

- **Overview:**  
  Sends back messages to the Telegram user with AI-generated meeting proposals, clarifications, or confirmations.

- **Nodes Involved:**  
  - Send Response

- **Node Details:**

  - **Send Response**  
    - *Type:* Telegram Node  
    - *Role:* Sends text messages to Telegram chat  
    - *Configuration:*  
      - Message text taken from AI Agent output JSON field `output.message` (fallback to original message if absent)  
      - Chat ID from "Prepare Data" node  
      - Attribution disabled for cleaner messages  
    - *Connections:* Receives from AI Agent  
    - *Potential Failures:* Telegram API errors, invalid chat ID  
    - *Version:* v1.2

#### 1.6 Event Creation & Email

- **Overview:**  
  Upon user's meeting time confirmation, creates a Google Calendar event and sends a confirmation email with meeting details.

- **Nodes Involved:**  
  - Should Create Event?  
  - Create Calendar Event  
  - Send Confirmation Email

- **Node Details:**

  - **Should Create Event?**  
    - *Type:* If Node  
    - *Role:* Checks if AI Agent's action is "create_event" to trigger event creation  
    - *Configuration:*  
      - Condition: `output.action === "create_event"`  
    - *Connections:*  
      - True branch: to "Create Calendar Event"  
      - False branch: no connection (ends workflow)  
    - *Potential Failures:* Missing or malformed action field  
    - *Version:* v2.2

  - **Create Calendar Event**  
    - *Type:* Google Calendar Node  
    - *Role:* Creates an event on Google Calendar with specified date/time and attendees  
    - *Configuration:*  
      - Start time: ISO string constructed from AI output `meeting_date` and `meeting_time`  
      - End time: one hour later than start time  
      - Calendar ID: `YOUR_CALENDAR_ID`  
      - Event summary: "Meeting: <user_name> & <contact_name>"  
      - Attendees: email from AI output `contact_email`  
    - *Connections:* Output to "Send Confirmation Email"  
    - *Potential Failures:* API errors, invalid date/time format, permission issues  
    - *Version:* v1.3

  - **Send Confirmation Email**  
    - *Type:* Gmail Node  
    - *Role:* Sends a styled HTML email confirming the meeting to the attendee  
    - *Configuration:*  
      - Recipient: AI output `contact_email`  
      - Subject: "Meeting Scheduled: <date> <time>"  
      - HTML body with date, time, user name, and a calendar link (`htmlLink` from calendar event JSON)  
    - *Potential Failures:* Gmail API authentication, invalid email addresses  
    - *Version:* v2.1

---

### 3. Summary Table

| Node Name             | Node Type                          | Functional Role                         | Input Node(s)           | Output Node(s)                                   | Sticky Note                                                                                                      |
|-----------------------|----------------------------------|---------------------------------------|------------------------|-------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| Main Instructions     | Sticky Note                      | Documentation overview                 |                        |                                                 | 🤖 PRIVATE AI VIRTUAL ASSISTANT: Production-ready AI assistant scheduling meetings via Telegram and Google APIs. |
| Sticky Note           | Sticky Note                      | Documentation                        |                        |                                                 | 1️⃣ TRIGGER: Receives messages from Telegram. User sends: "Schedule meeting with Anna on Friday".               |
| Sticky Note1          | Sticky Note                      | Documentation                        |                        |                                                 | 2️⃣ PREPARE DATA: Extracts user's message text, chat ID, and username.                                          |
| Sticky Note2          | Sticky Note                      | Documentation                        |                        |                                                 | 3️⃣ LOAD CRM DATA: Fetches contact database from Google Sheets (Name | Email | Phone).                          |
| Sticky Note3          | Sticky Note                      | Documentation                        |                        |                                                 | 4️⃣ AI AGENT: Finds contact, checks calendar, proposes slots, confirms, creates meeting.                         |
| Sticky Note4          | Sticky Note                      | Documentation                        |                        |                                                 | 5️⃣ SEND RESPONSE: Replies on Telegram with slots, confirmations, meeting details.                              |
| Sticky Note5          | Sticky Note                      | Documentation                        |                        |                                                 | 6️⃣ CREATE EVENT & EMAIL: Creates event, sends email, invites both parties.                                     |
| Telegram Trigger      | Telegram Trigger                 | Receives Telegram user messages       |                        | Prepare Data                                    | 1️⃣ TRIGGER                                                                                                      |
| Prepare Data          | Set                             | Extracts message, chat ID, username   | Telegram Trigger       | Load CRM Data                                   | 2️⃣ PREPARE DATA                                                                                                 |
| Load CRM Data         | Google Sheets                   | Loads CRM contacts                    | Prepare Data           | AI Agent                                        | 3️⃣ LOAD CRM DATA                                                                                                |
| CRM Search Tool       | Google Sheets Tool              | Provides CRM lookup for AI agent     |                        | AI Agent                                        | 4️⃣ AI AGENT                                                                                                     |
| Google Calendar Tool  | Google Calendar Tool            | Provides calendar access for AI      |                        | AI Agent                                        | 4️⃣ AI AGENT                                                                                                     |
| OpenAI Model          | Langchain LM Chat OpenAI        | Supplies GPT-4o-mini model            |                        | AI Agent                                        | 4️⃣ AI AGENT                                                                                                     |
| AI Agent              | Langchain Agent                | Core AI processing and decision-making | Load CRM Data, CRM Search Tool, Google Calendar Tool, OpenAI Model | Send Response, Should Create Event?                    | 4️⃣ AI AGENT                                                                                                     |
| Send Response         | Telegram                        | Sends message replies to Telegram     | AI Agent               | Should Create Event?                            | 5️⃣ SEND RESPONSE                                                                                                |
| Should Create Event?  | If                             | Decides if event creation is needed   | AI Agent               | Create Calendar Event                           | 6️⃣ CREATE EVENT & EMAIL                                                                                          |
| Create Calendar Event | Google Calendar                | Creates event with attendees          | Should Create Event?   | Send Confirmation Email                         | 6️⃣ CREATE EVENT & EMAIL                                                                                          |
| Send Confirmation Email| Gmail                         | Sends confirmation email              | Create Calendar Event  |                                                 | 6️⃣ CREATE EVENT & EMAIL                                                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node:**  
   - Type: Telegram Trigger  
   - Configure to listen for "message" updates only  
   - No filters  
   - Connect output to "Prepare Data"

2. **Create Prepare Data node:**  
   - Type: Set  
   - Assign variables:  
     - `user_message` = `{{$json.message.text}}`  
     - `chat_id` = `{{$json.message.chat.id}}`  
     - `user_name` = `{{$json.message.from.first_name}}`  
   - Connect output to "Load CRM Data"

3. **Create Load CRM Data node:**  
   - Type: Google Sheets  
   - Authentication: Service Account with access to the Google Sheet  
   - Document ID: Replace with your Google Sheet ID containing CRM data  
   - Sheet Name: "CRM"  
   - Connect output to "AI Agent"

4. **Create CRM Search Tool node:**  
   - Type: Google Sheets Tool  
   - Same document ID and sheet name "CRM"  
   - Provide as tool input to "AI Agent"

5. **Create Google Calendar Tool node:**  
   - Type: Google Calendar Tool  
   - Calendar ID: Specify your calendar to check availability  
   - Provide as tool input to "AI Agent"

6. **Create OpenAI Model node:**  
   - Type: Langchain LM Chat OpenAI  
   - Model: "gpt-4o-mini"  
   - Provide as language model to "AI Agent"

7. **Create AI Agent node:**  
   - Type: Langchain Agent  
   - Configure input text template:  
     ```
     Current date: {{ $now.setZone('Europe/Warsaw').toFormat('yyyy-MM-dd') }}
     User message: {{ $('Prepare Data').item.json.user_message }}
     CRM contacts: {{ JSON.stringify($('Load CRM Data').all().map(item => ({name: item.json.Name, email: item.json.Email}))) }}
     ```
   - System prompt:  
     - Scheduling assistant instructions: find contact, check calendar (9AM-4PM), propose 3 slots, confirm, create event  
     - Output JSON format with message, action, contact_email, meeting_date, meeting_time  
   - Max iterations: 10  
   - Connect tools: OpenAI Model, Google Calendar Tool, CRM Search Tool  
   - Connect output to "Send Response" and "Should Create Event?"

8. **Create Send Response node:**  
   - Type: Telegram  
   - Text: `{{$json.output.message || $json.message}}`  
   - Chat ID: `{{$('Prepare Data').item.json.chat_id}}`  
   - Disable attribution  
   - Connect input from AI Agent

9. **Create Should Create Event? node:**  
   - Type: If  
   - Condition: Check if `{{$json.output.action}}` equals `"create_event"`  
   - True branch connects to "Create Calendar Event"  
   - False branch ends workflow

10. **Create Create Calendar Event node:**  
    - Type: Google Calendar  
    - Calendar ID: Your calendar ID  
    - Start time: Construct from `{{$json.output.meeting_date}}T{{$json.output.meeting_time}}:00`  
    - End time: One hour after start time  
    - Summary: "Meeting: {{$('Prepare Data').item.json.user_name}} & {{$json.output.contact_name}}"  
    - Attendees: `[{{$json.output.contact_email}}]`  
    - Connect output to "Send Confirmation Email"

11. **Create Send Confirmation Email node:**  
    - Type: Gmail  
    - To: `{{$('AI Agent').item.json.output.contact_email}}`  
    - Subject: "Meeting Scheduled: {{$('AI Agent').item.json.output.meeting_date}} {{$('AI Agent').item.json.output.meeting_time}}"  
    - HTML body: Include date, time, user name, and a clickable link to the calendar event (`{{$json.htmlLink}}`)  
    - Connect input from "Create Calendar Event"

12. **Credentials Setup:**  
    - Telegram API credentials with bot token  
    - Google Service Account credentials for Sheets and Calendar with appropriate permissions  
    - OpenAI API key with access to GPT-4o-mini  
    - Gmail OAuth2 credentials for sending emails

13. **Testing & Validation:**  
    - Test Telegram interactions for message receipt and reply  
    - Validate CRM data loading and correct parsing  
    - Confirm AI Agent produces expected JSON outputs  
    - Verify calendar event creation and email delivery

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                    | Context or Link                                                                                                     |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| This workflow demonstrates the future of personal business automation with a privacy-first approach, keeping data within Google Sheets and Calendar.                                                                                                                            | Sticky Note "Main Instructions"                                                                                     |
| To customize, edit the AI Agent system prompt to add new business capabilities or connect additional APIs without changing workflow code.                                                                                                                                        | Sticky Note "Main Instructions"                                                                                     |
| For detailed concepts on using GPT-powered autonomous agents with n8n, see the n8n blog and Langchain documentation.                                                                                                                                                            | n8n blog: https://n8n.io/blog, Langchain docs: https://docs.langchain.com/                                         |
| Replace all placeholders like `YOUR_GOOGLE_SHEET_ID` and `YOUR_CALENDAR_ID` with your actual Google resource identifiers before running the workflow.                                                                                                                             | Inline notes in Google Sheets and Calendar node parameters                                                         |
| Ensure Google Service Account has delegated domain-wide authority if necessary, and Gmail OAuth2 consent screen is properly configured for email sending.                                                                                                                        | Google Cloud Platform documentation                                                                                 |
| The AI agent’s output JSON must be validated to prevent malformed actions that could disrupt flow; consider adding error handling nodes for robustness in production.                                                                                                           | Best practice recommendation                                                                                        |

---

*Disclaimer:*  
The text provided originates exclusively from an automated n8n workflow that complies strictly with current content policies and contains no illegal or offensive elements. All data handled is legal and public.