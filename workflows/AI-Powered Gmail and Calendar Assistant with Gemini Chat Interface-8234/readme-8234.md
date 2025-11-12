AI-Powered Gmail and Calendar Assistant with Gemini Chat Interface

https://n8nworkflows.xyz/workflows/ai-powered-gmail-and-calendar-assistant-with-gemini-chat-interface-8234


# AI-Powered Gmail and Calendar Assistant with Gemini Chat Interface

---
### 1. Workflow Overview

This workflow implements an AI-powered assistant designed to interact via a webhook interface, processing user requests related to Gmail and Google Calendar through a conversational AI model (Google Gemini). Its primary purpose is to serve as a personal productivity assistant that can understand user intents, execute email and calendar operations, and respond with meaningful information or confirmations.

**Target Use Cases:**
- Scheduling meetings and sending invitations
- Checking and summarizing emails within specified time frames
- Retrieving calendar events for specified date ranges
- Sending emails on the user’s behalf
- Converting natural language date/time expressions into precise ISO formats

**Logical Blocks:**

- **1.1 Input Reception and Preprocessing:** Receives user input through a webhook, normalizes incoming text for processing.
- **1.2 AI Language Model Processing:** Uses Google Gemini Chat Model to interpret and generate responses.
- **1.3 AI Agent (Core Logic):** Acts as the cognitive engine translating user requests into specific tool calls, managing context and decision logic.
- **1.4 Tool Execution Nodes:** A set of nodes interfacing with Gmail and Google Calendar APIs to perform actions such as reading emails, scheduling events, checking availability, and sending messages.
- **1.5 Memory Management:** Maintains short-term conversation context to support coherent multi-turn interactions.
- **1.6 Response Preparation and Webhook Reply:** Formats AI-generated output and sends it back to the requester via the webhook response.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Preprocessing

- **Overview:**  
Captures incoming HTTP POST requests on the `/chat` endpoint, extracts user textual input, and prepares it for AI processing.

- **Nodes Involved:**  
  - Webhook  
  - Set  

- **Node Details:**  

  - **Webhook**  
    - *Type:* HTTP Webhook  
    - *Role:* Entry point accepting POST requests at `/chat` path, allowing requests from any origin (`allowedOrigins: *`).  
    - *Config:* HTTP method POST, response mode set to wait for a response node.  
    - *Connections:* Outputs to the Set node.  
    - *Failures/Edge Cases:* Malformed requests without expected text fields may cause empty input downstream.

  - **Set**  
    - *Type:* Data manipulation (Set node)  
    - *Role:* Extracts and normalizes the user input text from various possible JSON fields (`body.text`, `text`, `body.input`) into a unified `input` field for AI Agent.  
    - *Config:* Expression used to assign `input` field with prioritization of available text keys.  
    - *Connections:* Outputs to the AI Agent node.  
    - *Failures/Edge Cases:* If input text is missing or empty, AI Agent receives empty input, potentially causing response issues.

#### 1.2 AI Language Model Processing

- **Overview:**  
Processes input text with Google Gemini Chat Model to generate AI language understanding and response content.

- **Nodes Involved:**  
  - Google Gemini Chat Model  

- **Node Details:**  

  - **Google Gemini Chat Model**  
    - *Type:* Language Model node (Google Gemini)  
    - *Role:* Provides natural language understanding and generation capabilities, powering the assistant’s conversational intelligence.  
    - *Config:* Uses configured Google Palm API credentials; no additional parameters defined, relies on AI Agent node to handle chaining.  
    - *Connections:* Connected as the AI language model input to AI Agent node.  
    - *Failures/Edge Cases:* API authentication errors, quota limits, or network issues may cause failures. Model response latency can affect workflow throughput.

#### 1.3 AI Agent (Core Logic)

- **Overview:**  
Central cognitive engine interpreting user input, managing conversation context, choosing tools to call, and generating actionable outputs including clarifications and confirmations.

- **Nodes Involved:**  
  - AI Agent  

- **Node Details:**  

  - **AI Agent**  
    - *Type:* Langchain Agent node  
    - *Role:* Translates user requests into precise tool calls according to a detailed "Tool Manifest" and operational protocol embedded in the system message. Manages interaction flow including asking for missing parameters and confirming actions before execution.  
    - *Config:*  
      - Input text bound to normalized `input` field.  
      - System prompt defines strict operating principles: tool-centric operation, clarification mandate, single-task focus, user confirmation before data-changing actions.  
      - Tool manifest includes Gmail and Google Calendar operations plus date/time utility.  
      - Defined workflow for scheduling meetings with steps for availability checking, event creation, attendee notification, and final confirmation.  
    - *Connections:*  
      - Receives AI language model output and memory context via `ai_languageModel` and `ai_memory` inputs.  
      - Calls external tool nodes via `ai_tool` connections.  
      - Sends processed output to Prepare Reply node.  
    - *Versions:* Requires n8n v2.2+ for Langchain agent node.  
    - *Failures/Edge Cases:*  
      - Expression errors if input variables are missing or malformed.  
      - Potential API call failures in downstream tool nodes must be handled gracefully.  
      - Ambiguous user requests requiring clarification loop.  
      - Authentication and permission errors for Gmail/Calendar APIs.

#### 1.4 Tool Execution Nodes

- **Overview:**  
Nodes directly interfacing with external Gmail and Google Calendar APIs to perform requested actions such as retrieving emails/events, checking availability, creating or updating events, and sending emails.

- **Nodes Involved:**  
  - Send a message in Gmail  
  - Get many messages in Gmail  
  - Get many events in Google Calendar  
  - Date & Time  
  - Get availability in a calendar in Google Calendar  
  - Create an event in Google Calendar  
  - Update an event in Google Calendar  

- **Node Details:**  

  - **Send a message in Gmail**  
    - *Type:* Gmail Tool - send message  
    - *Role:* Sends an email using Gmail OAuth2 credentials.  
    - *Config:* Recipient, subject, and message body dynamically populated from AI Agent variables. Attribution appended automatically.  
    - *Connections:* Called as a tool by AI Agent node.  
    - *Failures/Edge Cases:* Authentication failures, invalid email addresses, Gmail API limits.

  - **Get many messages in Gmail**  
    - *Type:* Gmail Tool - get messages  
    - *Role:* Retrieves emails filtered by user-specified time frames or senders, providing detailed summaries.  
    - *Config:* Limit and query filters set from AI Agent parameters; supports timeframe filtering.  
    - *Connections:* Tool call from AI Agent node.  
    - *Failures/Edge Cases:* Filtering errors, API rate limits, authentication issues.

  - **Get many events in Google Calendar**  
    - *Type:* Google Calendar Tool - get events  
    - *Role:* Fetches calendar events within specified date ranges.  
    - *Config:* `timeMin` and `timeMax` dynamically set by AI Agent; calendar identified by user email.  
    - *Connections:* Tool call from AI Agent node.  
    - *Failures/Edge Cases:* Permission errors, invalid date ranges.

  - **Date & Time**  
    - *Type:* Date/Time Tool  
    - *Role:* Converts natural language date/time inputs into ISO 8601 formatted timestamps, respecting user’s timezone.  
    - *Config:* Timezone and output field name passed from AI Agent.  
    - *Connections:* Tool call from AI Agent node.  
    - *Failures/Edge Cases:* Incorrect or ambiguous timezone inputs; parsing failures.

  - **Get availability in a calendar in Google Calendar**  
    - *Type:* Google Calendar Tool - availability check  
    - *Role:* Checks if user is free during a proposed time slot to avoid scheduling conflicts.  
    - *Config:* `timeMin` and `timeMax` set dynamically; calendar specified.  
    - *Connections:* Tool call from AI Agent node.  
    - *Failures/Edge Cases:* API permission issues, invalid time slots.

  - **Create an event in Google Calendar**  
    - *Type:* Google Calendar Tool - create event  
    - *Role:* Schedules new events with title, time, attendees, description, and Google Meet link.  
    - *Config:* Event details dynamically populated; attendees list included if provided. Default reminders optionally used.  
    - *Connections:* Tool call from AI Agent node.  
    - *Failures/Edge Cases:* Missing mandatory fields, attendees with invalid emails, API errors.

  - **Update an event in Google Calendar**  
    - *Type:* Google Calendar Tool - update event  
    - *Role:* Edits existing events based on event ID and updated fields.  
    - *Config:* Event ID and update fields passed dynamically.  
    - *Connections:* Tool call from AI Agent node.  
    - *Failures/Edge Cases:* Invalid event ID, permission errors.

#### 1.5 Memory Management

- **Overview:**  
Maintains a short-term context buffer of recent interactions to enable coherent conversations.

- **Nodes Involved:**  
  - Simple Memory  

- **Node Details:**  

  - **Simple Memory**  
    - *Type:* Langchain memory buffer window  
    - *Role:* Stores last ~10 interaction pairs (`input` and `reply`) keyed by a custom session ID to maintain conversation context.  
    - *Config:* Context window length set to 10 interactions. Session key uses a custom expression.  
    - *Connections:* Outputs memory context to AI Agent node.  
    - *Failures/Edge Cases:* Loss of context if session keys are inconsistent; memory overflow mitigated by window size.

#### 1.6 Response Preparation and Webhook Reply

- **Overview:**  
Formats the AI Agent’s final output into a response JSON and returns it to the original webhook caller.

- **Nodes Involved:**  
  - Prepare Reply  
  - Respond to Webhook  

- **Node Details:**  

  - **Prepare Reply**  
    - *Type:* Set node  
    - *Role:* Assigns AI Agent’s `output` field to a new `reply` field to structure response content.  
    - *Config:* Simple assignment of `reply = $json.output`, passes through other fields unchanged.  
    - *Connections:* Outputs to Respond to Webhook node.  
    - *Failures/Edge Cases:* If output field missing, response may be empty.

  - **Respond to Webhook**  
    - *Type:* Respond to Webhook node  
    - *Role:* Sends HTTP 200 response with prepared reply, completing the request cycle.  
    - *Config:* Response code 200.  
    - *Connections:* Final node in main execution chain.  
    - *Failures/Edge Cases:* None typical; network or client-side errors outside node scope.

---

### 3. Summary Table

| Node Name                         | Node Type                             | Functional Role                                | Input Node(s)          | Output Node(s)          | Sticky Note                                                                                                                     |
|----------------------------------|-------------------------------------|-----------------------------------------------|-----------------------|------------------------|--------------------------------------------------------------------------------------------------------------------------------|
| Webhook                          | HTTP Webhook                        | Entry point for incoming chat requests        |                       | Set                    |                                                                                                                                |
| Set                              | Set                                 | Normalizes user input text                      | Webhook                | AI Agent               |                                                                                                                                |
| AI Agent                        | Langchain Agent                     | Core AI logic, interprets requests, calls tools | Set, Google Gemini Chat Model, Simple Memory | Prepare Reply            | "🤖 AI Agent: The “brain” of the workflow. Interprets your requests and chooses the right tool..."                              |
| Google Gemini Chat Model          | Language Model (Google Gemini)      | AI language understanding and generation       |                       | AI Agent               | "🧠 Google Gemini Chat Model: The AI language model that powers the assistant... [AI API Key link](https://aistudio.google.com/apikey)" |
| Simple Memory                    | Langchain Memory Buffer Window      | Maintains short-term conversational context    |                       | AI Agent               | "📌 Simple Memory: Keeps short-term context of the last ~10 interactions..."                                                    |
| Send a message in Gmail           | Gmail Tool (send message)            | Sends emails via Gmail                          | AI Agent               |                        | "📧 Send a message in Gmail: Sends emails from your Gmail account..."                                                          |
| Get many messages in Gmail        | Gmail Tool (get messages)            | Retrieves and summarizes emails                 | AI Agent               |                        | "📥 Get many messages in Gmail: Checks your inbox. Can filter by timeframe or sender..."                                         |
| Get many events in Google Calendar| Google Calendar Tool (get events)   | Fetches calendar events for date ranges        | AI Agent               |                        | "📅 Get many events in Google Calendar: Lists your events for a chosen date range..."                                           |
| Date & Time                      | DateTime Tool                      | Converts natural language date/time expressions | AI Agent               | AI Agent               | "⏰ Date & Time: Converts natural phrases like 'tomorrow at 3 PM' into ISO date-time..."                                         |
| Get availability in a calendar... | Google Calendar Tool (availability) | Checks free/busy status to avoid conflicts      | AI Agent               |                        | "✅ Get availability in Google Calendar: Checks if you’re free during a specific time slot..."                                  |
| Create an event in Google Calendar| Google Calendar Tool (create event) | Schedules new calendar events                   | AI Agent               |                        | "📝 Create an event in Google Calendar: Schedules a new meeting..."                                                             |
| Update an event in Google Calendar| Google Calendar Tool (update event) | Edits existing calendar events                  | AI Agent               |                        | "✏️ Update an event in Google Calendar: Edits an existing meeting..."                                                          |
| Prepare Reply                    | Set                                 | Formats AI output for webhook response          | AI Agent               | Respond to Webhook      |                                                                                                                                |
| Respond to Webhook               | Respond to Webhook                  | Sends HTTP 200 response with AI-generated reply | Prepare Reply           |                        |                                                                                                                                |
| Sticky Note                     | Sticky Note                        | Notes about Messenger alternatives              |                       |                        | "🗣️ Chat Node: If you wish to, you can switch to a different Messenger app like discord, Whatsapp and Telegram"                 |
| Sticky Note1                    | Sticky Note                        | Notes about AI Agent                             |                       |                        | "🤖 AI Agent: The “brain” of the workflow..."                                                                                   |
| Sticky Note2                    | Sticky Note                        | Notes about Google Gemini Chat Model             |                       |                        | "🧠 Google Gemini Chat Model: The AI language model that powers the assistant... [AI API Key link]"                              |
| Sticky Note3                    | Sticky Note                        | Notes about Simple Memory                        |                       |                        | "📌 Simple Memory: Keeps short-term context of the last ~10 interactions..."                                                    |
| Sticky Note4                    | Sticky Note                        | Notes describing all tools available            |                       |                        | "## Tools: The tools consist of the following... (email, calendar, date/time utilities, etc.)"                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook Node**
   - Type: HTTP Webhook  
   - Configuration:  
     - Path: `chat`  
     - HTTP Method: POST  
     - Allowed Origins: `*` (allow all)  
     - Response Mode: `Response Node`  
   - Purpose: Entry point for user requests.

2. **Create a Set Node to Normalize Input**
   - Connect from Webhook node.  
   - Assign field `input` with expression:  
     `= {{$json["body"]?.text || $json["text"] || $json["body"]?.input || $json["text"]}}`  
   - Purpose: Extract user input text in a consistent field.

3. **Add Google Gemini Chat Model Node**
   - Type: Langchain LM Chat Google Gemini  
   - Provide Google Palm API credentials.  
   - No additional parameters required.  
   - Purpose: AI language understanding and response generation.

4. **Add Simple Memory Node**
   - Type: Langchain Memory Buffer Window  
   - Parameters:  
     - Session Key: Custom expression as per session management needs (e.g., `"parameters": {"memoryKey": "history", "inputKey": "input", "outputKey": "reply"}`)  
     - Context Window Length: 10 interactions  
   - Purpose: Maintain recent conversation history.

5. **Add AI Agent Node**
   - Type: Langchain Agent  
   - Parameters:  
     - Input Text: Bind to `input` field from Set node.  
     - System Message: Insert detailed instructions with tool manifest and operational protocol (as per workflow description).  
     - Connect AI Language Model input to Google Gemini Chat Model node.  
     - Connect AI Memory input to Simple Memory node.  
     - Configure `ai_tool` connections to all tool nodes below.  
   - Purpose: Core logic processing user requests and delegating tasks.

6. **Create Tool Nodes with Credentials**
   - **Send a message in Gmail:**  
     - Type: Gmail Tool (send message)  
     - Connect `ai_tool` input from AI Agent.  
     - Configure Gmail OAuth2 credentials.  
     - Parameters for recipient, subject, and message body to be dynamically populated from AI Agent outputs.

   - **Get many messages in Gmail:**  
     - Type: Gmail Tool (get messages)  
     - Connect `ai_tool` input from AI Agent.  
     - Configure Gmail OAuth2 credentials.

   - **Get many events in Google Calendar:**  
     - Type: Google Calendar Tool (get events)  
     - Connect `ai_tool` input from AI Agent.  
     - Configure Google Calendar OAuth2 credentials.

   - **Date & Time Tool:**  
     - Type: Date & Time Tool  
     - Connect `ai_tool` input from AI Agent.

   - **Get availability in a calendar in Google Calendar:**  
     - Type: Google Calendar Tool (availability check)  
     - Connect `ai_tool` input from AI Agent.  
     - Configure Google Calendar OAuth2 credentials.

   - **Create an event in Google Calendar:**  
     - Type: Google Calendar Tool (create event)  
     - Connect `ai_tool` input from AI Agent.  
     - Configure Google Calendar OAuth2 credentials.

   - **Update an event in Google Calendar:**  
     - Type: Google Calendar Tool (update event)  
     - Connect `ai_tool` input from AI Agent.  
     - Configure Google Calendar OAuth2 credentials.

7. **Add Prepare Reply Node**
   - Type: Set  
   - Connect from AI Agent node.  
   - Assign `reply = $json.output` to format AI Agent output for webhook response.

8. **Add Respond to Webhook Node**
   - Type: Respond to Webhook  
   - Connect from Prepare Reply node.  
   - Response code: 200  
   - Purpose: Return the final AI-generated reply to the webhook caller.

9. **Set Up Sticky Notes (Optional)**
   - Add visual notes with content describing functional blocks, tools, and usage tips as per original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                         | Context or Link                                                                                     |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| You can switch the AI language model to alternatives like OpenAI or Claude; Gemini was chosen here for free and easy setup.                                        | AI Model Node configuration                                                                        |
| AI API Key for Google Gemini can be obtained here: https://aistudio.google.com/apikey                                                                             | Google Gemini Chat Model Sticky Note                                                              |
| The workflow strictly enforces user confirmation before creating or sending data-changing operations, enhancing safety and user control.                         | AI Agent System Message / Operational Protocol                                                    |
| The assistant supports natural language time expressions, converting them robustly into ISO 8601 timestamps using the Date & Time tool node.                    | Date & Time Tool Node                                                                              |
| The defined workflow for scheduling meetings ensures no double booking by checking availability before event creation and sending invitation emails on success.   | AI Agent System Message / Tool Manifest                                                           |
| Sticky note suggests the possibility to extend the chat interface to other messaging platforms (Discord, WhatsApp, Telegram) with additional nodes and connectors. | Sticky Note on Chat Node                                                                           |

---

**Disclaimer:** The text provided originates solely from an automated n8n workflow. It adheres strictly to content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.