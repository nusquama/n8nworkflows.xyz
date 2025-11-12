Automated Law Firm Lead Management & Scheduling with AI, JotForm, WhatsApp & Calendar

https://n8nworkflows.xyz/workflows/automated-law-firm-lead-management---scheduling-with-ai--jotform--whatsapp---calendar-9383


# Automated Law Firm Lead Management & Scheduling with AI, JotForm, WhatsApp & Calendar

### 1. Workflow Overview

This workflow automates lead management and appointment scheduling for a law firm using AI, JotForm, WhatsApp, Google Sheets, and Google Calendar. It is divided into two main logical blocks:

**1.1 New Lead Intake & Welcome Message**  
Triggered by new form submissions via JotForm, this block captures potential client data, stores or updates it in a Google Sheet, and uses AI to generate a personalized WhatsApp welcome message sent directly to the client.

**1.2 AI-Powered Appointment Scheduling**  
Activated when the client replies on WhatsApp, this block uses AI conversational agents to handle scheduling requests. It accesses the client’s data, checks calendar availability, books consultations, and maintains chat memory for context-aware interactions.

---

### 2. Block-by-Block Analysis

---

#### 2.1 New Lead Intake & Welcome Message

**Overview:**  
This block listens for new submissions from a JotForm legal inquiry form, stores or updates client data in Google Sheets, generates a tailored welcome message using AI, and sends it to the client via WhatsApp.

**Nodes Involved:**  
- JotForm Trigger  
- Append or update row in sheet (Google Sheets)  
- AI Agent (Langchain Agent)  
- Google Gemini Chat Model (AI Language Model)  
- Send message (WhatsApp)

**Node Details:**

- **JotForm Trigger**  
  - Type: JotForm Trigger Node  
  - Role: Listens for new submissions on a specific JotForm (ID: 252801824783057).  
  - Configuration: Uses JotForm API credentials; triggers webhook on form submission.  
  - Inputs: None (trigger node)  
  - Outputs: Form data JSON with fields such as Full Name, Phone Number, Email, Legal Service of Interest, Brief Message.  
  - Potential failures: API connection/authentication issues, webhook downtime, form ID mismatch.

- **Append or update row in sheet**  
  - Type: Google Sheets Node  
  - Role: Inserts or updates a row in the "Law Client Enquiries" Google Sheet based on the Email Address as a unique key.  
  - Configuration: Uses Google Service Account credentials; maps form fields to sheet columns; matching on "Email Address" to update existing entries.  
  - Inputs: Data from JotForm Trigger  
  - Outputs: Confirmation with updated or appended row data.  
  - Edge cases: Duplicate emails, API quota limits, schema mismatches in the sheet, permission errors on Google Sheets.

- **AI Agent**  
  - Type: Langchain Agent Node  
  - Role: Generates a personalized WhatsApp welcome message using input client data and a structured prompt.  
  - Configuration: Defines persona "Alex," legal intake assistant; includes client info and detailed instructions with message structure, output constraints (e.g., use of bold via asterisks, compact message).  
  - Inputs: Data from Google Sheets node (client info)  
  - Outputs: Generated WhatsApp message text.  
  - Failures: AI service downtime, prompt syntax errors, unexpected input data formats.

- **Google Gemini Chat Model**  
  - Type: Langchain Google Gemini Chat Model Node  
  - Role: Executes the AI language generation for the AI Agent's prompt.  
  - Configuration: Uses Google PaLM API credentials; no special parameters set.  
  - Inputs: Prompt text from AI Agent  
  - Outputs: Generated message text for WhatsApp  
  - Failures: API quota, network issues, model unavailability.

- **Send message (WhatsApp)**  
  - Type: WhatsApp Node  
  - Role: Sends the generated welcome message to the client’s phone number.  
  - Configuration: Uses WhatsApp API credentials; sends message to phone number from form data; message text from AI Agent output.  
  - Inputs: AI-generated message and client phone number  
  - Outputs: Message send status  
  - Edge cases: Invalid phone number formatting, API authentication failures, message delivery failures.

---

#### 2.2 AI-Powered Appointment Scheduling

**Overview:**  
When a client replies via WhatsApp, this block engages an AI scheduling assistant to manage booking a legal consultation. It retrieves client info, checks calendar availability, books appointments, and sends conversational messages to the client, preserving chat context with a Postgres memory.

**Nodes Involved:**  
- WhatsApp Trigger  
- If (condition check)  
- AI Agent1 (Langchain Agent)  
- Postgres Chat Memory  
- Know about the user enquiry (Google Sheets Tool)  
- GET MANY EVENTS OF DAY THE USER ASKED (Google Calendar Tool)  
- Create an event (Google Calendar Tool)  
- Google Gemini Chat Model1 (AI Language Model)  
- Send message1 (WhatsApp)

**Node Details:**

- **WhatsApp Trigger**  
  - Type: WhatsApp Trigger Node  
  - Role: Listens for incoming WhatsApp messages directed to the firm’s business number; triggers workflow on message receipt.  
  - Configuration: Uses WhatsApp OAuth credentials; listens for message updates including delivery status and new messages.  
  - Inputs: Incoming WhatsApp message JSON  
  - Outputs: Message and sender details  
  - Failures: OAuth token expiration, webhook issues, message format errors.

- **If**  
  - Type: IF Node  
  - Role: Checks if incoming WhatsApp message text is empty; if empty, stops workflow; else continues.  
  - Configuration: Condition tests if message text body is empty string.  
  - Inputs: WhatsApp Trigger node output.  
  - Outputs: Conditional branch to AI Agent1 if message text exists.  
  - Edge cases: Non-text messages (media), blank messages, message format anomalies.

- **AI Agent1**  
  - Type: Langchain Agent Node  
  - Role: Acts as a professional scheduling assistant “Alex” managing the conversation. It uses tools to query client data, calendar events, and create bookings.  
  - Configuration: Complex prompt defining persona, context with current timestamp, task flow steps, fallback messages, and output formatting rules for WhatsApp.  
  - Inputs: User message text from WhatsApp trigger, session chat memory, results from tools.  
  - Outputs: Response text for WhatsApp.  
  - Failures: AI response errors, tool invocation failures, prompt handling issues.

- **Postgres Chat Memory**  
  - Type: Langchain Postgres Chat Memory Node  
  - Role: Maintains conversation history keyed by WhatsApp user ID to provide context for AI Agent1.  
  - Configuration: Uses Postgres credentials; session key is WhatsApp user ID.  
  - Inputs: Conversation data from AI Agent1 and WhatsApp Trigger  
  - Outputs: Chat memory data used by AI Agent1  
  - Failures: Database connection issues, session key mismatches.

- **Know about the user enquiry**  
  - Type: Google Sheets Tool Node  
  - Role: Allows AI Agent1 to retrieve client details from the Google Sheet using phone number as lookup key.  
  - Configuration: Uses Google Service Account credentials; searches "Law Client Enquiries" sheet by phone number.  
  - Inputs: Phone number extracted from WhatsApp message context  
  - Outputs: Client data row matching phone number  
  - Failures: Missing data, API permission issues, lookup failure.

- **GET MANY EVENTS OF DAY THE USER ASKED**  
  - Type: Google Calendar Tool Node  
  - Role: Retrieves all calendar events on the date requested by the client to check availability.  
  - Configuration: Uses Google OAuth credentials; timeMin and timeMax dynamically set via AI overrides; returns all events.  
  - Inputs: Date/time parameters from AI Agent1 prompt context  
  - Outputs: List of existing events on requested date  
  - Failures: Calendar API quota limits, invalid date formats, permission errors.

- **Create an event**  
  - Type: Google Calendar Tool Node  
  - Role: Books a legal consultation event on the Google Calendar with client email as attendee.  
  - Configuration: Uses Google OAuth credentials; event start/end times, summary, description, attendees dynamically set by AI Agent1; sends update notifications.  
  - Inputs: Availability confirmation and booking data from AI Agent1  
  - Outputs: Created event confirmation  
  - Failures: Event conflicts, invalid attendee emails, API errors.

- **Google Gemini Chat Model1**  
  - Type: Langchain Google Gemini Chat Model Node  
  - Role: Executes AI language model calls for AI Agent1’s prompt processing.  
  - Configuration: Uses Google PaLM API credentials.  
  - Inputs: Prompt from AI Agent1 including conversation context and tools data  
  - Outputs: AI-generated conversational responses  
  - Failures: API unavailability, quota exhaustion.

- **Send message1 (WhatsApp)**  
  - Type: WhatsApp Node  
  - Role: Sends AI-generated replies back to the client on WhatsApp to continue the scheduling conversation.  
  - Configuration: Uses WhatsApp API credentials; recipient is the WhatsApp user ID from the trigger node; message body from AI Agent1 output.  
  - Inputs: AI Agent1 output message text  
  - Outputs: Message send status  
  - Failures: Message delivery failures, invalid phone numbers.

---

### 3. Summary Table

| Node Name                        | Node Type                          | Functional Role                              | Input Node(s)                     | Output Node(s)                   | Sticky Note                                                                                                           |
|---------------------------------|----------------------------------|----------------------------------------------|----------------------------------|---------------------------------|-----------------------------------------------------------------------------------------------------------------------|
| JotForm Trigger                 | JotForm Trigger                  | Triggers workflow on new form submission    | None                             | Append or update row in sheet    | Part A: New Lead Intake & Welcome Message - Starting point for new lead intake.                                        |
| Append or update row in sheet   | Google Sheets                    | Stores/updates client data in Google Sheet  | JotForm Trigger                  | AI Agent                        | Part A: New Lead Intake & Welcome Message - Records client info.                                                      |
| AI Agent                       | Langchain Agent                   | Generates personalized WhatsApp welcome msg | Append or update row in sheet    | Send message                   | Part A: New Lead Intake & Welcome Message - Creates client welcome message using AI.                                   |
| Google Gemini Chat Model        | Langchain Google Gemini Model    | AI text generation engine                    | AI Agent                        | AI Agent                       | Part A: New Lead Intake & Welcome Message - AI model powering message creation.                                        |
| Send message                   | WhatsApp                        | Sends WhatsApp welcome message               | AI Agent                        | None                           | Part A: New Lead Intake & Welcome Message - Sends welcome message to client.                                           |
| WhatsApp Trigger               | WhatsApp Trigger                | Triggers on incoming WhatsApp messages       | None                             | If                             | Part B: AI-Powered Appointment Scheduling - Starts scheduling conversation.                                           |
| If                            | If                             | Checks if incoming message text is empty     | WhatsApp Trigger                | AI Agent1                      | Part B: AI-Powered Appointment Scheduling - Filters out empty or non-text messages.                                   |
| AI Agent1                     | Langchain Agent                   | Conversational AI scheduling assistant       | If                             | Send message1                  | Part B: AI-Powered Appointment Scheduling - Manages booking conversation.                                              |
| Postgres Chat Memory           | Postgres Chat Memory             | Maintains chat context per user              | WhatsApp Trigger                | AI Agent1                     | Part B: AI-Powered Appointment Scheduling - Enables AI memory of chat history.                                        |
| Know about the user enquiry    | Google Sheets Tool               | Retrieves client enquiry details by phone    | AI Agent1                      | AI Agent1                     | Part B: AI-Powered Appointment Scheduling - AI tool to fetch client data.                                             |
| GET MANY EVENTS OF DAY THE USER ASKED | Google Calendar Tool             | Checks calendar events on requested day      | AI Agent1                      | AI Agent1                     | Part B: AI-Powered Appointment Scheduling - AI tool to check calendar availability.                                   |
| Create an event               | Google Calendar Tool             | Books consultation event in calendar         | AI Agent1                      | AI Agent1                     | Part B: AI-Powered Appointment Scheduling - AI tool to create calendar event.                                         |
| Google Gemini Chat Model1      | Langchain Google Gemini Model    | AI text generation engine for scheduling     | AI Agent1                      | AI Agent1                     | Part B: AI-Powered Appointment Scheduling - AI model powering scheduling conversation.                                |
| Send message1                 | WhatsApp                        | Sends scheduling conversation replies        | AI Agent1                      | None                           | Part B: AI-Powered Appointment Scheduling - Sends chat messages to client during scheduling.                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the JotForm Trigger Node**  
   - Type: JotForm Trigger  
   - Set form ID to your legal inquiry form (e.g., 252801824783057)  
   - Configure credentials with your JotForm API key  
   - This node listens to new form submissions.

2. **Create the Google Sheets Node "Append or update row in sheet"**  
   - Type: Google Sheets  
   - Operation: Append or update  
   - Document: Select or input your Google Sheet document ID (e.g., "Law Client Enquiries")  
   - Sheet: Use Sheet1 or appropriate tab  
   - Map form fields to columns: Full Name (combine first and last), Email Address, Phone Number, client type, Legal Service of Interest, Brief Message, How Did You Hear About Us?  
   - Set matching column to "Email Address" to update existing rows  
   - Use Google Service Account credentials for authentication  
   - Connect from JotForm Trigger node.

3. **Create the Langchain Agent Node "AI Agent"**  
   - Type: Langchain Agent  
   - Input prompt: Include persona "Alex," client's full name, legal service interest, message, and detailed instructions to draft a concise WhatsApp welcome message with formatting rules (bold with asterisks, no personal contact info)  
   - Ensure output is a single WhatsApp message text  
   - Connect input from Google Sheets node's output (client data).

4. **Create the Langchain Google Gemini Chat Model Node**  
   - Type: Langchain Google Gemini Chat Model  
   - Credentials: Google PaLM API  
   - Connect input from AI Agent node (prompt)  
   - Output feeds back into AI Agent node for message generation.

5. **Create the WhatsApp Node "Send message"**  
   - Type: WhatsApp  
   - Operation: Send  
   - Phone Number ID: your WhatsApp business phone number ID  
   - Recipient Phone Number: reference client phone number from Google Sheets node  
   - Message Body: bind to AI Agent’s generated output  
   - Use WhatsApp API credentials  
   - Connect from AI Agent output.

6. **Create the WhatsApp Trigger Node**  
   - Type: WhatsApp Trigger  
   - Configure to listen for incoming messages on your WhatsApp business number  
   - Use WhatsApp OAuth2 credentials  
   - This node starts the scheduling flow upon receiving client replies.

7. **Create the If Node**  
   - Type: If  
   - Condition: Check if incoming message text body is empty  
   - If yes: stop workflow (no output)  
   - If no: continue to AI Agent1

8. **Create the Langchain Agent Node "AI Agent1"**  
   - Type: Langchain Agent  
   - Persona: "Alex," professional legal scheduling assistant  
   - Include current timestamp context for interpreting relative times  
   - Instructions: Follow a multi-step scheduling procedure using tools; handle greetings naturally; produce WhatsApp-styled conversational responses with bullet lists, bold, and italics  
   - Include fallback message if unable to answer  
   - Connect input from the If node (when message text exists).

9. **Create the Postgres Chat Memory Node**  
   - Type: Postgres Chat Memory  
   - Session Key: WhatsApp user ID extracted from incoming message  
   - Connect as AI memory to AI Agent1 node  
   - Use Postgres credentials for database access.

10. **Create the Google Sheets Tool Node "Know about the user enquiry"**  
    - Type: Google Sheets Tool  
    - Operation: Lookup row by "Phone Number" in the "Law Client Enquiries" sheet  
    - Use Google Service Account credentials  
    - Connect as a tool to AI Agent1 to enable client info retrieval.

11. **Create the Google Calendar Tool Node "GET MANY EVENTS OF DAY THE USER ASKED"**  
    - Type: Google Calendar Tool  
    - Operation: Get all events between dynamic timeMin and timeMax (dates extracted by AI)  
    - Use Google OAuth2 credentials  
    - Connect as a tool to AI Agent1 to check calendar availability.

12. **Create the Google Calendar Tool Node "Create an event"**  
    - Type: Google Calendar Tool  
    - Operation: Create event with details (start, end, summary, attendees, description) dynamically set by AI Agent1  
    - Send update notifications to guests  
    - Use Google OAuth2 credentials  
    - Connect as a tool to AI Agent1 to book appointments.

13. **Create the Langchain Google Gemini Chat Model Node "Google Gemini Chat Model1"**  
    - Type: Langchain Google Gemini Chat Model  
    - Use Google PaLM API credentials  
    - Connect as AI language model to AI Agent1 for response generation.

14. **Create the WhatsApp Node "Send message1"**  
    - Type: WhatsApp  
    - Operation: Send  
    - Phone Number ID: your WhatsApp business phone number ID  
    - Recipient Phone Number: WhatsApp user ID from WhatsApp Trigger node  
    - Message Body: AI Agent1 output message text  
    - Use WhatsApp API credentials  
    - Connect from AI Agent1 output.

---

### 5. General Notes & Resources

| Note Content                                                                                                                | Context or Link                                                                                           |
|-----------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| The workflow uses a professional persona "Alex" to humanize AI communications both for welcoming leads and scheduling.     | Internal branding and user experience enhancement                                                       |
| Calendly link for scheduling is included in the welcome message: https://calendly.com/vtr-law-firm/consultation              | Invoke client action with clear call to action in welcome message                                        |
| Google Sheets document "Law Client Enquiries" is the central repository for client data, with email as the unique key       | Sheet URL: https://docs.google.com/spreadsheets/d/1invngp2z_3ZMe_Qcs5XioDkAs50DSXT-Pl4ibPLXyA0             |
| Google Calendar is used for appointment availability and event creation, linked to email attendees                         | Calendar account: venkibvb5192@gmail.com                                                                 |
| Postgres is used for chat memory to maintain context across WhatsApp conversations                                          | Database connection keeps AI conversations coherent                                                     |
| Workflow split into two parts: Part A handles lead intake and initial messages; Part B manages AI-driven scheduling chats. | Sticky notes within the workflow clarify these logical separations                                        |

---

This documentation provides a comprehensive understanding of the automated law firm lead management and scheduling workflow, enabling reproduction, modification, and troubleshooting by developers and AI agents alike.

---

**Disclaimer:** The provided text is generated exclusively from an automated n8n workflow. It complies strictly with content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.