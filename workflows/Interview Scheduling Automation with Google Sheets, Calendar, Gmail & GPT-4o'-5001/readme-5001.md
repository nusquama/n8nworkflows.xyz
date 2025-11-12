Interview Scheduling Automation with Google Sheets, Calendar, Gmail & GPT-4o'

https://n8nworkflows.xyz/workflows/interview-scheduling-automation-with-google-sheets--calendar--gmail---gpt-4o--5001


# Interview Scheduling Automation with Google Sheets, Calendar, Gmail & GPT-4o'

### 1. Workflow Overview

This workflow automates the scheduling of interviews by integrating Google Sheets, Google Calendar, Gmail, and the Azure OpenAI GPT-4o language model. It listens for new entries in a Google Sheet containing candidate information, calculates the next available interview slot, creates a corresponding Google Calendar event, generates a personalized email invitation using AI, and sends it via Gmail.

The workflow is structured into the following logical blocks:

- **1.1 Input Reception:** Detects new candidate data entries from Google Sheets.
- **1.2 Slot Calculation:** Computes the next available interview time slot based on predefined weekly schedules.
- **1.3 Calendar Event Creation:** Adds the interview event to Google Calendar with the calculated slot.
- **1.4 AI Email Generation:** Utilizes Azure OpenAI GPT-4o to craft a personalized interview invitation email.
- **1.5 Email Delivery:** Sends the generated email to the candidate’s email address via Gmail.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block triggers the workflow when a new candidate entry is added or modified in a specified Google Sheet.

- **Nodes Involved:**  
  - Google Sheets Trigger

- **Node Details:**

  - **Google Sheets Trigger**  
    - *Type & Role:* Trigger node that listens for changes in a Google Sheets document.  
    - *Configuration:*  
      - Polling mode set to run every minute.  
      - Monitors the sheet named "Foglio1" (gid=0) in the specified Google Sheets document.  
      - Uses OAuth2 credentials linked to a Google Sheets account for authentication.  
    - *Expressions/Variables:* None beyond the document and sheet identifiers.  
    - *Input/Output:* No input (trigger); outputs the changed row data as JSON.  
    - *Edge Cases:*  
      - Missing or revoked OAuth2 credentials can cause trigger failure.  
      - Changes outside the monitored sheet or document do not trigger the workflow.  
      - Potential delays due to polling frequency (max once per minute).  
    - *Sub-workflow:* None.

#### 1.2 Slot Calculation

- **Overview:**  
  Calculates the next available interview slot based on a fixed schedule (Monday, Wednesday, and Friday at 3 PM), ensuring the slot is always in the future.

- **Nodes Involved:**  
  - Code

- **Node Details:**

  - **Code**  
    - *Type & Role:* JavaScript code node to compute the next interview time slot.  
    - *Configuration:*  
      - Defines fixed interview slots on Monday, Wednesday, and Friday at 3 PM.  
      - Skips the current day even if the slot time hasn't passed.  
      - Formats start and end times in "YYYY-MM-DD HH:mm:ss" format; end time is always one hour after start.  
    - *Expressions/Variables:*  
      - Outputs two variables: `nextSlot` and `endSlot` for the event timing.  
    - *Input/Output:* Receives candidate data from Google Sheets Trigger; outputs JSON with slot times.  
    - *Edge Cases:*  
      - Timezone differences might affect slot calculations if not synchronized with Google Calendar/Sheets timezone.  
      - Code assumes server time is accurate.  
      - Unexpected JavaScript exceptions could halt the workflow.  
    - *Sub-workflow:* None.

#### 1.3 Calendar Event Creation

- **Overview:**  
  Creates a Google Calendar event for the interview using the calculated time slot.

- **Nodes Involved:**  
  - Google Calendar

- **Node Details:**

  - **Google Calendar**  
    - *Type & Role:* Creates an event on the user's Google Calendar.  
    - *Configuration:*  
      - Uses `nextSlot` as the event start and `endSlot` as the event end.  
      - Adds the event to the calendar associated with the email `itf.jyothiswarup@gmail.com`.  
      - No additional fields specified (e.g., description, attendees).  
    - *Expressions/Variables:* Uses dynamic expressions to set start/end times from Code node output.  
    - *Input/Output:* Inputs slot timings; outputs event details including `htmlLink`.  
    - *Edge Cases:*  
      - OAuth2 token expiration or revocation can block event creation.  
      - Calendar API quota limits or connectivity issues.  
      - Conflicts with existing events are not checked.  
    - *Sub-workflow:* None.

#### 1.4 AI Email Generation

- **Overview:**  
  Generates a personalized email message inviting the candidate for the interview, incorporating candidate details and the calendar event link, using Azure OpenAI GPT-4o.

- **Nodes Involved:**  
  - Basic LLM Chain  
  - Azure OpenAI Chat Model  
  - Structured Output Parser

- **Node Details:**

  - **Basic LLM Chain**  
    - *Type & Role:* Langchain node that orchestrates prompt creation, language model invocation, and output parsing.  
    - *Configuration:*  
      - Input text includes candidate’s name, educational background (from Google Sheets), calendar event link (from Google Calendar), and a fixed signature introduction.  
      - Uses a prompt instructing the AI to write a concise, personalized email starting with a 15-20 word opening statement, followed by calendar details and a signature.  
      - Output is parsed to extract a structured JSON containing the email body.  
    - *Expressions/Variables:*  
      - References fields from Google Sheets Trigger and Google Calendar nodes.  
    - *Input/Output:*  
      - Receives calendar event info and candidate data.  
      - Outputs the parsed email body for sending.  
    - *Edge Cases:*  
      - GPT model latency or API errors.  
      - Parsing failures if AI output deviates from expected schema.  
      - Azure OpenAI API quota or credential issues.  
    - *Sub-workflow:* None.

  - **Azure OpenAI Chat Model**  
    - *Type & Role:* Executes the GPT-4o model hosted on Azure OpenAI.  
    - *Configuration:*  
      - Model set explicitly to "gpt-4o".  
      - No additional options configured.  
      - Uses Azure OpenAI credentials.  
    - *Input/Output:*  
      - Receives prompt from Basic LLM Chain.  
      - Outputs AI-generated text to Basic LLM Chain for parsing.  
    - *Edge Cases:*  
      - Network or Azure service outages.  
      - Model request limits or billing issues.  
    - *Sub-workflow:* None.

  - **Structured Output Parser**  
    - *Type & Role:* Parses AI output into a defined JSON structure.  
    - *Configuration:*  
      - Schema expects one field: `body` (string) containing email text.  
    - *Input/Output:*  
      - Receives raw AI output text.  
      - Outputs parsed JSON to Basic LLM Chain.  
    - *Edge Cases:*  
      - Malformed AI output causing parser failure.  
    - *Sub-workflow:* None.

#### 1.5 Email Delivery

- **Overview:**  
  Sends the personalized interview invitation email to the candidate’s email address extracted from Google Sheets.

- **Nodes Involved:**  
  - Gmail

- **Node Details:**

  - **Gmail**  
    - *Type & Role:* Sends an email via Gmail SMTP API.  
    - *Configuration:*  
      - Recipient address dynamically set from the candidate’s `EMAIL` field in Google Sheets.  
      - Email subject fixed as "Interview Details".  
      - Email message set from the AI-generated email body (`output.body`).  
      - Email type set to plain text.  
      - Uses OAuth2 credentials for Gmail account authentication.  
    - *Expressions/Variables:*  
      - Pulls recipient and message dynamically from prior nodes.  
    - *Input/Output:*  
      - Inputs email content and recipient address; no output connected further.  
    - *Edge Cases:*  
      - OAuth2 token expiration or invalid credentials.  
      - Gmail sending limits or spam filtering.  
      - Invalid recipient email addresses.  
    - *Sub-workflow:* None.

---

### 3. Summary Table

| Node Name             | Node Type                               | Functional Role                 | Input Node(s)          | Output Node(s)        | Sticky Note                                           |
|-----------------------|---------------------------------------|--------------------------------|-----------------------|-----------------------|-------------------------------------------------------|
| Google Sheets Trigger  | n8n-nodes-base.googleSheetsTrigger    | Input reception from candidate sheet | None                  | Code                  |                                                       |
| Code                  | n8n-nodes-base.code                    | Calculate next interview slot  | Google Sheets Trigger  | Google Calendar       |                                                       |
| Google Calendar       | n8n-nodes-base.googleCalendar          | Create calendar event          | Code                  | Basic LLM Chain       |                                                       |
| Basic LLM Chain        | @n8n/n8n-nodes-langchain.chainLlm     | Generate personalized email    | Google Calendar, Azure OpenAI Chat Model, Structured Output Parser | Gmail                  |                                                       |
| Azure OpenAI Chat Model| @n8n/n8n-nodes-langchain.lmChatAzureOpenAi | AI model execution (GPT-4o)    | Basic LLM Chain (ai_languageModel) | Basic LLM Chain (ai_languageModel) |                                                       |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Parse AI output JSON           | Azure OpenAI Chat Model | Basic LLM Chain (ai_outputParser) |                                                       |
| Gmail                  | n8n-nodes-base.gmail                   | Send email invitation          | Basic LLM Chain        | None                  |                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node: Google Sheets Trigger**  
   - Set node type to "Google Sheets Trigger".  
   - Configure OAuth2 credentials for Google Sheets access.  
   - Set document ID to your Google Sheet containing candidate data.  
   - Set sheet name to the relevant sheet (e.g., "Foglio1").  
   - Set polling to run every minute to detect new rows or changes.

2. **Add Code Node for Slot Calculation**  
   - Connect the Google Sheets Trigger output to this node’s input.  
   - Use the provided JavaScript code to calculate the next interview slot (Monday, Wednesday, Friday at 3 PM), skipping the current day.  
   - Output JSON with `nextSlot` and `endSlot` formatted as "YYYY-MM-DD HH:mm:ss".

3. **Add Google Calendar Node**  
   - Connect the Code node output to the Google Calendar node input.  
   - Authenticate using Google Calendar OAuth2 credentials.  
   - Set the calendar to the desired email (e.g., your own or company calendar).  
   - Configure event start and end times using expressions referencing `nextSlot` and `endSlot` from the Code node.  
   - Leave additional fields empty unless event description or attendees are required.

4. **Add Basic LLM Chain Node**  
   - Connect Google Calendar node output to the Basic LLM Chain node input.  
   - In the Basic LLM Chain, configure the prompt with:  
     - Candidate name and educational background from Google Sheets Trigger data.  
     - Calendar event link (use `htmlLink` from Google Calendar node output).  
     - A signature introduction fixed text.  
   - Set prompt instructions to generate a professional, concise email starting with a short opening statement, followed by calendar details and a signature.  
   - Enable output parsing with a JSON schema expecting a `body` string.

5. **Add Azure OpenAI Chat Model Node**  
   - Connect Basic LLM Chain’s AI language model input to this node’s output.  
   - Authenticate with Azure OpenAI credentials.  
   - Select model "gpt-4o".  
   - No special options needed.

6. **Add Structured Output Parser Node**  
   - Connect Azure OpenAI Chat Model output to this node’s input.  
   - Configure parser with JSON schema expecting a single field `body` (string).  
   - Output parsed JSON back into Basic LLM Chain to complete email generation.

7. **Connect Basic LLM Chain Output to Gmail Node**  
   - Add Gmail node and connect Basic LLM Chain output to Gmail input.  
   - Authenticate Gmail node with OAuth2 credentials.  
   - Set recipient email dynamically using the candidate’s email from Google Sheets Trigger data.  
   - Set email subject to "Interview Details".  
   - Set email message body from the parsed AI output `body`.  
   - Use plain text email format.

8. **Finalize Connections**  
   - Ensure the flow is: Google Sheets Trigger → Code → Google Calendar → Basic LLM Chain → Gmail.  
   - Within the Basic LLM Chain, link Azure OpenAI Chat Model and Structured Output Parser nodes accordingly.

9. **Test the Workflow**  
   - Add a test candidate row with columns for NAME, EDUCATIONAL, and EMAIL in Google Sheets.  
   - Wait for the trigger or run manually.  
   - Verify calendar event creation, email generation, and email delivery.

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                                  |
|-----------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| The scheduling slots are fixed: Monday, Wednesday, Friday at 3 PM; adjust in the Code node as needed. | Modify JavaScript in the Code node to customize interview slots. |
| Azure OpenAI GPT-4o model is used for email generation; ensure your Azure subscription supports this model. | https://azure.microsoft.com/en-us/services/cognitive-services/openai-service/ |
| Gmail and Google OAuth2 credentials require proper scopes for sending emails and accessing calendars. | Refer to Google OAuth2 setup documentation for n8n.             |
| Workflow uses polling on Google Sheets; expect up to a 1-minute delay after sheet updates.    | Consider webhook alternatives for faster trigger if supported.  |
| For complex email formatting (HTML, attachments), extend the Gmail node configuration.         | n8n documentation for Gmail node: https://docs.n8n.io/nodes/n8n-nodes-base.gmail/ |

---

**Disclaimer:** The provided content originates exclusively from an automated n8n workflow. It respects all applicable content policies and contains no illegal, offensive, or protected material. Only legal and public data is processed.