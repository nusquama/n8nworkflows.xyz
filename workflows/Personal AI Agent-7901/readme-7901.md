Personal AI Agent

https://n8nworkflows.xyz/workflows/personal-ai-agent-7901


# Personal AI Agent

### 1. Workflow Overview

This n8n workflow, titled **Personal AI Agent** (named internally as "All in One Agent"), serves as a comprehensive personal assistant integrating Gmail, Google Calendar, Google Sheets, Telegram, and Google Gemini AI to automate email management, calendar scheduling, contact handling, and messaging. Its target use cases include managing incoming emails, categorizing and replying automatically, scheduling calendar events with conflict checks, maintaining a contacts database, and interacting via Telegram commands.

The workflow is logically divided into these main functional blocks:

- **1.1 Input Reception**: Triggers capturing incoming Gmail emails and Telegram messages.
- **1.2 Email Processing & Categorization**: Processing incoming emails using AI for classification and automated replies, labeling, or ignoring.
- **1.3 AI Agent Processing**: Central AI node interpreting user commands or email content and directing actions.
- **1.4 Gmail Operations**: Nodes for sending, replying, deleting, and retrieving emails.
- **1.5 Google Calendar Operations**: Scheduling, updating, deleting, and fetching calendar events with conflict management.
- **1.6 Contact Management**: Reading and updating contacts from a Google Sheet.
- **1.7 Auxiliary Components**: Utilities like date/time parsing, session memory, and supportive AI chains.
- **1.8 Messaging Outputs**: Sending outputs back to Telegram chat or notifications.
- **1.9 Documentation & Setup Guidance**: Sticky notes providing setup instructions and tutorial links.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
Captures input events from Gmail (new emails) and Telegram (messages/commands) to kickstart processing.

**Nodes involved:**  
- Gmail Trigger  
- Telegram Trigger

**Node Details:**

- **Gmail Trigger**  
  - Type: Gmail trigger node (polling every minute)  
  - Configuration: No filters (listens to all incoming emails)  
  - Inputs: External Gmail inbox activity  
  - Outputs: New email JSON objects including labels, sender info, message ID, snippet  
  - Edge Cases: Possible API rate limits, authentication token expiry, or no new emails detected  
  - Version: 1.2  

- **Telegram Trigger**  
  - Type: Telegram trigger node  
  - Configuration: Listens for new messages only  
  - Inputs: Telegram messages from user  
  - Outputs: Telegram message JSON including chat ID, text, sender details  
  - Edge Cases: Bot revoked, Telegram API downtime, message format issues  
  - Version: 1.2  

---

#### 1.2 Email Processing & Categorization

**Overview:**  
Processes incoming Gmail messages, classifies them into categories (Client, Sponsorship Request, Not Business), and applies appropriate labels or replies accordingly.

**Nodes involved:**  
- If  
- Basic LLM Chain  
- Text Classifier  
- Client (labeling)  
- Sponsorship Request (labeling)  
- Not Business (labeling)  
- Reply to a message  
- Reply to a message1  
- No Operation, do nothing  
- Send a text message1  

**Node Details:**

- **If**  
  - Type: Conditional node  
  - Configuration: Checks if the first label of the email is "INBOX" (filter for unread inbox emails)  
  - Input: Gmail Trigger output  
  - Outputs: True branch leads to Basic LLM Chain, False branch to No Operation  
  - Edge Cases: Emails without labels or unexpected label format  

- **Basic LLM Chain**  
  - Type: LangChain text LLM chain  
  - Configuration: Parses the email sender, snippet, and current time to generate a formatted message describing mail type, sender, and time  
  - Input: Output of If node  
  - Output: Structured text for classification  
  - Edge Cases: LLM response delays or errors  

- **Text Classifier**  
  - Type: LangChain text classification node  
  - Configuration: Classifies email snippet into one of three categories (Client, Sponsorship Request, Not Business) using custom categories and descriptions  
  - Input: Output from Basic LLM Chain  
  - Output: Classified category  
  - Edge Cases: Misclassification, ambiguous emails  

- **Client, Sponsorship Request, Not Business (Gmail nodes)**  
  - Type: Gmail node (add labels operation)  
  - Configuration: Each applies a specific label to the email based on classification  
  - Input: Output from Text Classifier node  
  - Edge Cases: Invalid label IDs, Gmail API failures  

- **Reply to a message & Reply to a message1**  
  - Type: Gmail node (reply operation)  
  - Configuration: Sends pre-defined reply messages for Client and Sponsorship Request emails respectively  
  - Input: Output from Client or Sponsorship Request nodes  
  - Edge Cases: Reply failure, message ID invalidation  

- **No Operation, do nothing**  
  - Type: NoOp node, does nothing for emails not in INBOX  
  - Input: False branch of If node  

- **Send a text message1**  
  - Type: Telegram node (send message)  
  - Configuration: Sends a notification message to a fixed Telegram chat ID (likely for alerts)  
  - Input: From Basic LLM Chain output  
  - Edge Cases: Telegram API token invalid or chat ID blocked  

---

#### 1.3 AI Agent Processing

**Overview:**  
Core AI agent node using LangChain’s agent with Google Gemini LLM integration to interpret user input from Telegram or email and orchestrate corresponding actions (email, calendar, contacts).

**Nodes involved:**  
- AI Agent  
- Gemini  
- Gemini1  
- Simple Memory  

**Node Details:**

- **AI Agent**  
  - Type: LangChain Agent node  
  - Configuration:  
    - Input text is the user’s message (from Telegram or email JSON path `$json.message.text`)  
    - System message defines assistant behavior with detailed responsibilities and instructions for Gmail tasks, calendar scheduling, contact handling, and behavior guidelines  
    - Uses various Gmail and Calendar tools  
  - Inputs: Telegram Trigger, Gmail-related nodes, and linked AI language models  
  - Outputs: Commands or text responses that trigger Gmail, Calendar, Contacts, or Telegram send nodes  
  - Edge Cases: Misinterpretation of instructions, missing input data, API authorization errors  

- **Gemini & Gemini1**  
  - Type: LangChain Google Gemini chat LLM nodes  
  - Configuration: Connects to Google Gemini AI for natural language processing  
  - Gemini1 is used in sequence with Basic LLM Chain and Text Classifier for enhanced classification  
  - Edge Cases: API quota, latency, or authentication issues  

- **Simple Memory**  
  - Type: LangChain memory buffer (windowed)  
  - Configuration: Stores conversational context keyed by Telegram chat ID for session continuity  
  - Input: Telegram chat ID from Telegram Trigger  
  - Output: Memory context to AI Agent node  
  - Edge Cases: Memory overflow or loss, session key mismanagement  

---

#### 1.4 Gmail Operations

**Overview:**  
Nodes performing direct Gmail actions like sending emails, replying, deleting, and retrieving multiple messages, invoked by AI Agent outputs.

**Nodes involved:**  
- Send  
- Reply  
- Delete  
- Get Many  

**Node Details:**

- **Send**  
  - Type: Gmail Tool (send email)  
  - Configuration: Sends email with parameters (To, Subject, Message) dynamically set from AI output overrides  
  - Inputs: AI Agent output commands  
  - Edge Cases: Invalid email address, SMTP errors, quota limits  

- **Reply**  
  - Type: Gmail Tool (reply to email)  
  - Configuration: Replies to specific email message IDs with AI-generated text  
  - Inputs: AI Agent output, message ID from email triggers  
  - Edge Cases: Message ID invalid or reply failures  

- **Delete**  
  - Type: Gmail Tool (delete email)  
  - Configuration: Deletes email by message ID as instructed by AI Agent  
  - Edge Cases: Message ID invalid, insufficient permissions  

- **Get Many**  
  - Type: Gmail Tool (retrieve emails)  
  - Configuration: Retrieves multiple emails with options set dynamically (filters, return all) via AI overrides  
  - Edge Cases: Large data volume, rate limits  

---

#### 1.5 Google Calendar Operations

**Overview:**  
Handles calendar event creation, updates, deletions, and queries with conflict detection and scheduling logic.

**Nodes involved:**  
- Create Event  
- Update  
- Delete2  
- Get Many2  

**Node Details:**

- **Create Event**  
  - Type: Google Calendar Tool (create event)  
  - Configuration: Creates event with parameters (start, end time, summary, attendees, description) dynamically from AI Agent output  
  - Inputs: AI Agent commands  
  - Edge Cases: Scheduling conflicts, invalid time formats, permission errors  

- **Update**  
  - Type: Google Calendar Tool (update event)  
  - Configuration: Updates event fields, primarily handling reminders or metadata, based on AI commands  
  - Edge Cases: Invalid event ID, update conflicts  

- **Delete2**  
  - Type: Google Calendar Tool (delete event)  
  - Configuration: Deletes event by event ID from AI commands  
  - Edge Cases: Event not found, permission denied  

- **Get Many2**  
  - Type: Google Calendar Tool (get events)  
  - Configuration: Retrieves calendar events in a time window (timeMin, timeMax) to check for conflicts  
  - Edge Cases: Large result sets, API errors  

---

#### 1.6 Contact Management

**Overview:**  
Reads and updates contacts stored in Google Sheets to resolve email addresses by name and save new contacts.

**Nodes involved:**  
- Get Contacts  
- Add Contacts  

**Node Details:**

- **Get Contacts**  
  - Type: Google Sheets Tool (read sheet)  
  - Configuration: Reads "Sheet1" of a Google Sheet document containing Name and Email Address columns  
  - Inputs: AI Agent queries for contact lookup  
  - Edge Cases: Sheet access errors, sheet structure changes  

- **Add Contacts**  
  - Type: Google Sheets Tool (append row)  
  - Configuration: Adds new contact entries (Name and Email Address) based on AI Agent output after user confirmation  
  - Edge Cases: Duplicate entries, append failures  

---

#### 1.7 Auxiliary Components

**Overview:**  
Supportive nodes for date/time formatting, session memory, and intermediate AI processing chains.

**Nodes involved:**  
- Date & Time  
- Basic LLM Chain (for email parsing)  
- Simple Memory  

**Node Details:**

- **Date & Time**  
  - Type: DateTime Tool  
  - Configuration: Parses and formats date/time inputs with timezone Asia/Dhaka and dynamic output field names  
  - Used by AI Agent for interpreting meeting times  
  - Edge Cases: Incorrect input format, timezone mismatches  

- **Basic LLM Chain**  
  - As described in 1.2 for email parsing  

- **Simple Memory**  
  - As described in 1.3 for session context  

---

#### 1.8 Messaging Outputs

**Overview:**  
Sends processed text responses back to Telegram users.

**Nodes involved:**  
- Send a text message  
- Send a text message1  

**Node Details:**

- **Send a text message**  
  - Type: Telegram node (send message)  
  - Configuration: Sends AI Agent output text back to the same Telegram chat ID as the trigger message originated from  
  - Edge Cases: Telegram API failures, chat ID invalid  

- **Send a text message1**  
  - As described in 1.2 for external notifications  

---

#### 1.9 Documentation & Setup Guidance

**Overview:**  
Sticky notes provide detailed setup instructions, author credits, and tutorial links.

**Nodes involved:**  
- Sticky Note  
- Sticky Note3  

**Node Details:**

- **Sticky Note**  
  - Large setup guide covering OAuth2 credential creation for Gmail, Calendar, Google Sheets, Telegram bot setup, Google Gemini API key, AI agent customization, and testing tips  
  - Provides YouTube link to author Rakin Jakaria’s channel: https://www.youtube.com/@rakinjakaria  

- **Sticky Note3**  
  - Contains a clickable YouTube tutorial link with embedded thumbnail for step-by-step guidance on building an AI agent: https://youtu.be/WO3mM7XZRLg  

---

### 3. Summary Table

| Node Name             | Node Type                          | Functional Role                             | Input Node(s)           | Output Node(s)           | Sticky Note                                                                                                         |
|-----------------------|----------------------------------|--------------------------------------------|------------------------|--------------------------|---------------------------------------------------------------------------------------------------------------------|
| AI Agent              | LangChain Agent                  | Core AI interpreter and orchestrator       | Telegram Trigger, Gmail Tools, Gemini, Simple Memory | Send a text message, Gmail, Calendar, Contacts nodes |                                                                                                                     |
| Send                  | Gmail Tool                      | Sends emails                               | AI Agent                 | AI Agent                 |                                                                                                                     |
| Reply                 | Gmail Tool                      | Replies to emails                          | AI Agent                 | AI Agent                 |                                                                                                                     |
| Delete                | Gmail Tool                      | Deletes emails                            | AI Agent                 | AI Agent                 |                                                                                                                     |
| Get Many              | Gmail Tool                      | Retrieves multiple emails                  | AI Agent                 | AI Agent                 |                                                                                                                     |
| Gemini                | LangChain Google Gemini LLM     | Provides LLM capabilities                  |                         | AI Agent                 |                                                                                                                     |
| Telegram Trigger       | Telegram Trigger                | Receives Telegram messages                  | External Telegram        | AI Agent                 |                                                                                                                     |
| Send a text message    | Telegram Send                   | Sends text messages to Telegram chats      | AI Agent                 |                          |                                                                                                                     |
| Gmail Trigger          | Gmail Trigger                  | Receives new Gmail emails                   | External Gmail           | If                       |                                                                                                                     |
| Create Event           | Google Calendar Tool            | Creates calendar events                      | AI Agent                 | AI Agent                 |                                                                                                                     |
| Get Many2              | Google Calendar Tool            | Retrieves calendar events                    | AI Agent                 | AI Agent                 |                                                                                                                     |
| Delete2                | Google Calendar Tool            | Deletes calendar events                      | AI Agent                 | AI Agent                 |                                                                                                                     |
| Update                 | Google Calendar Tool            | Updates calendar events                      | AI Agent                 | AI Agent                 |                                                                                                                     |
| Sticky Note            | Sticky Note                    | Setup Guide and author credits               |                         |                          | # 🛠️ Setup Guide with detailed OAuth2, API keys, Telegram, and AI agent setup instructions with YouTube links        |
| Sticky Note3           | Sticky Note                    | YouTube tutorial link                         |                         |                          | ## Start here: Step by Step Youtube Tutorial :star: [Video](https://youtu.be/WO3mM7XZRLg) with embedded thumbnail     |
| If                    | If node                        | Filters emails labeled INBOX                 | Gmail Trigger            | Basic LLM Chain, No Operation |                                                                                                                     |
| No Operation, do nothing | NoOp                          | Skips processing for non-INBOX emails        | If                       |                          |                                                                                                                     |
| Basic LLM Chain        | LangChain LLM chain            | Parses email snippet for classification      | If                       | Text Classifier, Send a text message1 |                                                                                                                     |
| Text Classifier        | LangChain text classifier      | Categorizes email into Client/Sponsorship/Not Business | Basic LLM Chain          | Client, Sponsorship Request, Not Business |                                                                                                                     |
| Client                 | Gmail node (add label)          | Labels emails as Client                      | Text Classifier           | Reply to a message        |                                                                                                                     |
| Sponsorship Request    | Gmail node (add label)          | Labels emails as Sponsorship Request         | Text Classifier           | Reply to a message1       |                                                                                                                     |
| Not Business           | Gmail node (add label)          | Labels emails as Not Business                 | Text Classifier           |                          |                                                                                                                     |
| Reply to a message     | Gmail node (reply)              | Replies to Client emails                      | Client                    |                          |                                                                                                                     |
| Reply to a message1    | Gmail node (reply)              | Replies to Sponsorship Request emails         | Sponsorship Request       |                          |                                                                                                                     |
| Send a text message1   | Telegram Send                  | Sends notification messages to fixed chat    | Basic LLM Chain           |                          |                                                                                                                     |
| Simple Memory          | LangChain memory buffer        | Maintains session context for AI Agent       | Telegram Trigger          | AI Agent                 |                                                                                                                     |
| Gemini1                | LangChain Google Gemini LLM     | Secondary LLM for classification             |                          | Basic LLM Chain, Text Classifier |                                                                                                                     |
| Get Contacts           | Google Sheets Tool             | Reads contacts from Google Sheet              | AI Agent                 | AI Agent                 |                                                                                                                     |
| Add Contacts           | Google Sheets Tool             | Appends new contacts to Google Sheet          | AI Agent                 | AI Agent                 |                                                                                                                     |
| Date & Time            | DateTime Tool                 | Parses and formats date/time                    | AI Agent                 | AI Agent                 |                                                                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Credentials:**
   - Setup OAuth2 credentials in n8n for Gmail, Google Calendar, Google Sheets, and Telegram API.
   - Setup Google Gemini API key credential with the key from Google AI Studio.

2. **Input Nodes:**
   - Add a **Gmail Trigger** node configured to poll every minute without filters.
   - Add a **Telegram Trigger** node configured to listen for new messages.

3. **Email Filtering:**
   - Add an **If** node connected to the Gmail Trigger.
     - Condition: Check if the first label ID equals "INBOX".
   - Connect True branch to **Basic LLM Chain** node.
   - Connect False branch to **No Operation** node.

4. **Email Parsing & Classification:**
   - Add **Basic LLM Chain** with a prompt to extract sender, snippet, and time from the email.
   - Connect to **Text Classifier** node.
     - Define categories: Client, Sponsorship Request, Not Business.
   - Connect Text Classifier outputs to three Gmail nodes:
     - **Client**: Add label operation with Client label ID.
     - **Sponsorship Request**: Add label operation with Sponsorship label ID.
     - **Not Business**: Add label operation with Not Business label ID.

5. **Auto Replies:**
   - Connect Client node to **Reply to a message** node with custom reply text.
   - Connect Sponsorship node to **Reply to a message1** node with custom reply text.

6. **Telegram Notification:**
   - Connect Basic LLM Chain to **Send a text message1** node.
     - Configure fixed chat ID for notifications.

7. **AI Agent Setup:**
   - Add **AI Agent** node using LangChain agent type.
     - Set input to user messages (from Telegram or AI overrides).
     - Paste the system message describing assistant responsibilities (email, calendar, contacts).
     - Link AI Agent to:
       - Gmail nodes (Send, Reply, Delete, Get Many)
       - Google Calendar nodes (Create Event, Update, Delete2, Get Many2)
       - Google Sheets nodes (Get Contacts, Add Contacts)
       - Date & Time node for time parsing
       - Simple Memory node for session context
       - Gemini nodes for LLM integration

8. **Gmail Action Nodes:**
   - Add Gmail Tool nodes for Send, Reply, Delete, Get Many.
   - Configure inputs from AI Agent overrides for fields like To, Subject, Message, Message ID.

9. **Google Calendar Nodes:**
   - Add Google Calendar Tool nodes for Create Event, Update Event, Delete Event, Get Many Events.
   - Configure calendar ID for your calendar.
   - Input parameters dynamically from AI Agent overrides.

10. **Google Sheets Contact Nodes:**
    - Add Google Sheets Tool nodes for reading (Get Contacts) and appending (Add Contacts).
    - Configure spreadsheet document ID and sheet name.
    - Map columns "Name" and "Email Address".

11. **Date & Time Node:**
    - Add DateTime Tool node.
    - Configure timezone (e.g., Asia/Dhaka).
    - Allow dynamic control of including current time and output field name.

12. **Session Memory Setup:**
    - Add LangChain Memory Buffer Window node.
    - Configure session key as Telegram chat ID.

13. **Output Messaging:**
    - Add Telegram Send node to send AI Agent outputs back to the user’s chat ID.

14. **Connect Workflow:**
    - Connect Gmail Trigger > If > Basic LLM Chain > Text Classifier > appropriate label and reply nodes.
    - Connect Telegram Trigger to AI Agent.
    - Connect AI Agent to all Gmail, Calendar, Sheets, Date & Time, and Send Telegram nodes.

15. **Sticky Notes:**
    - Add sticky notes with setup instructions and tutorial video links for documentation and onboarding.

---

### 5. General Notes & Resources

| Note Content                                                                                                                       | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Setup Guide for OAuth2 credentials for Gmail, Google Calendar, Google Sheets, Telegram API, and Google Gemini API key.            | Provided in the large Sticky Note node; details stepwise connection and configuration advice.      |
| Author: Rakin Jakaria, YouTube Channel with tutorials and guidance on building AI agents.                                         | https://www.youtube.com/@rakinjakaria                                                              |
| Step-by-step YouTube tutorial video titled "I Built an Auto Lead Finder AI Agent" with embedded thumbnail link.                   | https://youtu.be/WO3mM7XZRLg                                                                       |
| Important: Gmail label IDs must match your actual Gmail account label IDs for correct labeling.                                   | Mentioned in setup instructions.                                                                   |
| Adjust Date & Time node timezone to your local region to ensure correct scheduling and time parsing.                              | Asia/Dhaka used in example; change as needed.                                                      |
| AI Agent system message defines detailed assistant behavior to ensure accurate and helpful task execution with Gmail and Calendar.| Customized system prompt inside AI Agent node parameters.                                          |

---

**Disclaimer:**  
The text above stems exclusively from an automated n8n workflow. All content respects applicable policies and contains no illegal, offensive, or protected material. Only legal and public data are processed.