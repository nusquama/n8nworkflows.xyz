Telegram Voice Assistant with OpenAI & Gmail: Manage & Draft Emails via Voice

https://n8nworkflows.xyz/workflows/telegram-voice-assistant-with-openai---gmail--manage---draft-emails-via-voice-9339


# Telegram Voice Assistant with OpenAI & Gmail: Manage & Draft Emails via Voice

### 1. Workflow Overview

This workflow implements a **Telegram Voice Assistant integrated with OpenAI and Gmail** to manage and draft emails via voice or text commands. It is designed for users who want to interact with their Gmail inbox and compose emails using natural language inputs sent through Telegram, including voice messages transcribed to text. The workflow handles input routing, audio transcription, AI-driven email management, and sending or drafting emails through Gmail OAuth2 integration.

Logical blocks:

- **1.1 Input Reception & Routing:** Receives incoming Telegram messages (voice, text, or image) and routes them accordingly.
- **1.2 Voice Processing:** Handles voice messages by downloading, transcribing, and converting AI responses back to audio.
- **1.3 Text Processing:** Processes textual input from Telegram users, passing it to the AI chat model for understanding and response.
- **1.4 AI Email Management:** Uses an AI agent to interpret user commands related to emails, such as retrieving, drafting, or sending emails.
- **1.5 Email Operations:** Executes Gmail actions including drafting, sending, and retrieving emails.
- **1.6 Session Memory Management:** Maintains conversation context per Telegram chat session using window buffer memory.
- **1.7 Response Delivery:** Sends back the AI-generated response via Telegram as text or audio.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Routing

- **Overview:**  
  This block listens for incoming Telegram messages and routes them based on message type (voice, text, or image/document).

- **Nodes Involved:**  
  - Telegram Trigger  
  - Switch  
  - Set 'Text'

- **Node Details:**

  - **Telegram Trigger**  
    - Type: Telegram Trigger node  
    - Role: Listens for new Telegram messages from a specific chat ID (6713681895)  
    - Config: Listens for "message" updates only; requires Telegram API credentials  
    - Input: External Telegram webhook  
    - Output: Raw Telegram message JSON  
    - Edge cases: Telegram API failures, webhook downtime, unauthorized chat IDs  

  - **Switch**  
    - Type: Switch node  
    - Role: Routes messages into Voice, Text, or Image branches based on message content presence  
    - Config:  
      - Voice branch if `message.voice.file_id` exists  
      - Text branch if `message.text` exists  
      - Image branch if `message.document.mime_type` exists  
    - Input: Output from Telegram Trigger  
    - Output: Routed to respective nodes for processing  
    - Edge cases: Messages without recognized content types, malformed JSON  

  - **Set 'Text'**  
    - Type: Set node  
    - Role: Extracts and assigns the text message content to a variable named `text`  
    - Config: Sets `text` = `{{$json.message.text}}`  
    - Input: Text branch from Switch node  
    - Output: Passes structured text JSON for AI processing  
    - Edge cases: Empty or null text messages

---

#### 1.2 Voice Processing

- **Overview:**  
  This block downloads Telegram voice messages, transcribes them via OpenAI, then passes the transcribed text for AI email management. It also converts AI-generated text responses back into audio format.

- **Nodes Involved:**  
  - Download File  
  - Transcribe  
  - Transcribe1  
  - Audio Response

- **Node Details:**

  - **Download File**  
    - Type: Telegram node  
    - Role: Downloads voice file from Telegram using the voice file ID  
    - Config: Takes `fileId` from `message.voice.file_id`, uses Telegram credentials  
    - Input: Voice branch from Switch node  
    - Output: Binary audio file for transcription  
    - Edge cases: File not found, download timeout, Telegram API errors  

  - **Transcribe**  
    - Type: LangChain OpenAI node (resource: audio, operation: transcribe)  
    - Role: Transcribes downloaded voice audio into text using OpenAI's transcription API  
    - Config: Default transcription options, requires OpenAI credentials  
    - Input: Audio file from Download File  
    - Output: Transcribed text JSON  
    - Edge cases: Audio format unsupported, API rate limits, transcription errors  

  - **Transcribe1**  
    - Type: LangChain OpenAI node (resource: audio)  
    - Role: Converts AI-generated text output back into an audio response with voice "onyx" and MP3 format  
    - Config: Input expression `{{$json.output}}`, voice: "onyx", response format: mp3  
    - Input: AI-generated text from Email Manager  
    - Output: Binary audio data for Telegram audio reply  
    - Edge cases: API voice conversion errors, unsupported voice parameters  

  - **Audio Response**  
    - Type: Telegram node  
    - Role: Sends back audio response to the Telegram chat  
    - Config: Sends audio binary data to the chat ID from original Telegram message  
    - Input: Binary audio from Transcribe1  
    - Output: Confirmation of sent audio message  
    - Edge cases: Telegram API send failures, large audio file size  

---

#### 1.3 Text Processing

- **Overview:**  
  Processes text messages — extracts the text, sends it to the AI chat model for interpretation, then delivers the response back to Telegram.

- **Nodes Involved:**  
  - Set 'Text'  
  - OpenAI Chat Model1  
  - Text Response

- **Node Details:**

  - **Set 'Text'** (detailed above in 1.1)  
    - Prepares text for AI processing.  

  - **OpenAI Chat Model1**  
    - Type: LangChain OpenAI Chat Model node  
    - Role: Sends user text input to GPT-4.1-mini model for AI understanding and response generation  
    - Config: Model set to "gpt-4.1-mini", default options  
    - Input: Text from Set 'Text' node  
    - Output: AI-generated response text in JSON  
    - Edge cases: API rate limits, prompt formatting errors, model unavailability  

  - **Text Response**  
    - Type: Telegram node  
    - Role: Sends AI-generated text response back to Telegram chat  
    - Config: Text message is `{{$json.output}}`, chatId from Telegram Trigger  
    - Input: AI response from Email Manager node (which integrates AI chat output)  
    - Output: Confirmation of sent text message  
    - Edge cases: Telegram API failures, message length limits  

---

#### 1.4 AI Email Management

- **Overview:**  
  The core AI agent that interprets user commands related to emails. It routes queries to specific email-related tools (get, draft, send emails) and manages session context.

- **Nodes Involved:**  
  - Email Manager  
  - Window Buffer Memory  

- **Node Details:**

  - **Email Manager**  
    - Type: LangChain AI Agent node  
    - Role: Natural language agent that receives user input text and decides which Gmail tool to invoke (get emails, draft emails, send emails)  
    - Config:  
      - Uses a system message defining roles and tools (Get Emails, Draft Emails, Send Emails)  
      - Enforces rules for contact info lookup before sending or drafting emails  
      - Current date/time dynamically inserted for contextual understanding  
    - Input: Text (from Set 'Text'), transcribed voice text, or AI chat model output  
    - Output: AI-generated instructions, email drafts, or responses  
    - Edge cases: Ambiguous user commands, API timeouts, logic errors in routing commands  

  - **Window Buffer Memory**  
    - Type: LangChain Memory Buffer Window node  
    - Role: Maintains chat session memory based on Telegram chat ID for context-aware responses  
    - Config: Uses `sessionKey` set to Telegram chat ID to isolate session data  
    - Input: Session-specific context from Telegram Trigger  
    - Output: Provides memory context to Email Manager AI Agent  
    - Edge cases: Memory overflow, session ID mismatches  

---

#### 1.5 Email Operations

- **Overview:**  
  Handles Gmail operations such as retrieving recent emails, drafting new emails, and sending emails based on AI agent instructions.

- **Nodes Involved:**  
  - Get Emails  
  - Draft Email  
  - Send Email  

- **Node Details:**

  - **Get Emails**  
    - Type: Gmail Tool node  
    - Role: Retrieves up to 3 emails from the Gmail inbox based on AI-defined date filters  
    - Config:  
      - Limit 3 messages  
      - Filter by labels: INBOX  
      - Filters `receivedAfter` and `receivedBefore` dynamically set by AI agent outputs  
      - Uses Gmail OAuth2 credentials  
    - Input: Invoked by Email Manager agent  
    - Output: Email data for AI processing  
    - Edge cases: OAuth token expiration, Gmail API quota errors  

  - **Draft Email**  
    - Type: Gmail Tool node  
    - Role: Saves an email draft with subject and body provided by AI agent  
    - Config:  
      - Message and subject populated from AI outputs (`$fromAI("emailBody")`, `$fromAI("subject")`)  
      - Uses Gmail OAuth2 credentials  
    - Input: Invoked by Email Manager agent  
    - Output: Draft email confirmation  
    - Edge cases: Draft save errors, invalid email content  

  - **Send Email**  
    - Type: Gmail Tool node  
    - Role: Sends an email to recipient(s) with message and subject from AI agent  
    - Config:  
      - SendTo, Message, Subject dynamically populated via AI overrides  
      - Uses Gmail OAuth2 credentials  
    - Input: Invoked by Email Manager agent  
    - Output: Email send confirmation  
    - Edge cases: Invalid recipient, send failure, OAuth authorization errors  

---

#### 1.6 Session Memory Management

- **Overview:**  
  Manages conversation context on a per-chat basis to enable memory-aware AI responses.

- **Nodes Involved:**  
  - Window Buffer Memory (also connected to AI Email Manager)

- **Node Details:**

  - **Window Buffer Memory** (already detailed above)  
    - Stores recent conversation snippets, enabling context retention across messages within the same Telegram chat session.

---

#### 1.7 Response Delivery

- **Overview:**  
  Sends the AI-generated responses back to the user in Telegram either as text or audio, depending on input type.

- **Nodes Involved:**  
  - Text Response  
  - Audio Response

- **Node Details:**

  - **Text Response** (detailed above)  
  - **Audio Response** (detailed above)

---

### 3. Summary Table

| Node Name           | Node Type                              | Functional Role                                  | Input Node(s)         | Output Node(s)                     | Sticky Note                                                                                      |
|---------------------|--------------------------------------|-------------------------------------------------|-----------------------|----------------------------------|------------------------------------------------------------------------------------------------|
| Telegram Trigger     | Telegram Trigger                     | Receive incoming Telegram messages               | -                     | Switch                           | 1. Setup Telegram bot and credentials. Activate workflow to start chat with bot.               |
| Switch              | Switch                              | Route messages by type: Voice, Text, Image       | Telegram Trigger       | Download File, Set 'Text'         | Modify routing logic to customize message handling.                                            |
| Set 'Text'           | Set                                 | Extract text content from Telegram message        | Switch (Text branch)   | Email Manager                    |                                                                                               |
| Download File       | Telegram (file download)             | Download voice message audio file                  | Switch (Voice branch)  | Transcribe                      |                                                                                               |
| Transcribe          | LangChain OpenAI (audio transcribe) | Transcribe downloaded voice to text                | Download File          | Email Manager                   | Use OpenAI API key; beware of audio format and API limits.                                    |
| OpenAI Chat Model1  | LangChain OpenAI Chat Model          | Generate AI response from text input               | Set 'Text'              | Email Manager                   | GPT-4.1-mini model used for AI understanding.                                                 |
| Email Manager       | LangChain AI Agent                   | Handle email commands: get, draft, send via AI    | Set 'Text', Transcribe, OpenAI Chat Model1, Window Buffer Memory | Text Response, Transcribe1         | Central AI agent managing email-related tools; uses session memory.                            |
| Window Buffer Memory| LangChain Memory Buffer Window       | Maintain session context per Telegram chat ID     | Telegram Trigger        | Email Manager                   |                                                                                               |
| Get Emails          | Gmail Tool                         | Retrieve emails from Gmail inbox                    | Email Manager (ai_tool) | Email Manager                  | Gmail OAuth2 required; filters emails dynamically based on AI input.                          |
| Draft Email         | Gmail Tool                         | Save email drafts                                  | Email Manager (ai_tool) | Email Manager                  |                                                                                               |
| Send Email          | Gmail Tool                         | Send emails                                        | Email Manager (ai_tool) | Email Manager                  |                                                                                               |
| Transcribe1         | LangChain OpenAI (text to audio)   | Convert AI text response to audio (voice onyx)     | Email Manager           | Audio Response                 | Generates audio reply for Telegram voice response.                                            |
| Text Response       | Telegram (send message)              | Send text response to Telegram chat                 | Email Manager           | -                              |                                                                                               |
| Audio Response      | Telegram (send audio)                | Send audio response to Telegram chat                | Transcribe1             | -                              |                                                                                               |
| Sticky Note8        | Sticky Note                        | Documentation, setup instructions, and usage tips  | -                       | -                              | Contains detailed setup, customization tips, and node flow explanation with links to Youtube and contact. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Bot and Credentials:**
   - Use @BotFather on Telegram to create a bot.
   - Copy the Bot Token.
   - In n8n, create Telegram API credentials with this token.

2. **Create Telegram Trigger Node:**
   - Node Type: Telegram Trigger
   - Configure to listen for "message" updates.
   - Restrict to chat ID `6713681895` or your own.
   - Assign Telegram credentials.

3. **Add Switch Node:**
   - Node Type: Switch
   - Create three outputs: Voice, Text, Image.
   - Voice condition: check if `message.voice.file_id` exists.
   - Text condition: check if `message.text` exists.
   - Image condition: check if `message.document.mime_type` exists.
   - Connect Telegram Trigger output to Switch input.

4. **Voice Processing Branch:**
   - Add Telegram node "Download File":
     - Configure to download voice file using `message.voice.file_id`.
     - Assign Telegram credentials.
     - Connect Switch "Voice" output to Download File.
   - Add LangChain OpenAI node "Transcribe":
     - Resource: Audio
     - Operation: Transcribe
     - Assign OpenAI credentials.
     - Connect Download File output to Transcribe.
   - Add LangChain OpenAI node "Transcribe1":
     - Resource: Audio
     - Input: `{{$json.output}}`
     - Voice: "onyx"
     - Response format: "mp3"
     - Assign OpenAI credentials.
     - Connect Email Manager output (see below) to Transcribe1.
   - Add Telegram node "Audio Response":
     - Operation: sendAudio
     - ChatId: from Telegram Trigger message chat id
     - Enable binary data sending.
     - Assign Telegram credentials.
     - Connect Transcribe1 output to Audio Response.

5. **Text Processing Branch:**
   - Add Set node "Set 'Text'":
     - Assign variable `text` = `{{$json.message.text}}`.
     - Connect Switch "Text" output to Set 'Text'.

6. **AI Chat Model Node:**
   - Add LangChain OpenAI Chat Model node:
     - Model: gpt-4.1-mini
     - Assign OpenAI credentials.
     - Connect Set 'Text' output to OpenAI Chat Model.

7. **Window Buffer Memory Node:**
   - Add LangChain Memory Buffer Window node:
     - Session Key: `{{$('Telegram Trigger').item.json.message.chat.id}}`
     - Session ID Type: customKey
     - Connect Telegram Trigger output to Window Buffer Memory.

8. **Email Manager AI Agent Node:**
   - Add LangChain AI Agent node:
     - Text input: `{{$json.text}}`
     - Options/System message: Define tools (Get Emails, Draft Emails, Send Emails), rules for contact lookup, current date/time.
     - Prompt Type: define
     - Connect outputs:
       - From Set 'Text' (text input)
       - From Transcribe (voice input)
       - From OpenAI Chat Model (text AI response)
       - From Window Buffer Memory (memory context)
     - Connect Email Manager output to Text Response and Transcribe1.

9. **Gmail Tool Nodes:**
   - Create and configure Gmail OAuth2 credentials.
   - Add "Get Emails" node:
     - Operation: getAll
     - Limit: 3 emails
     - Filters: Label INBOX, dates (receivedAfter, receivedBefore) from AI input
     - Connect Email Manager ai_tool output to Get Emails.
   - Add "Draft Email" node:
     - Operation: draft
     - Subject: from AI output "subject"
     - Message: from AI output "emailBody"
     - Connect Email Manager ai_tool output to Draft Email.
   - Add "Send Email" node:
     - Operation: send
     - To: from AI output "To"
     - Subject: from AI output "Subject"
     - Message: from AI output "Message"
     - Connect Email Manager ai_tool output to Send Email.

10. **Response Nodes:**
    - Add "Text Response" Telegram node:
      - Send text `{{$json.output}}`
      - Chat ID from Telegram Trigger
      - Assign Telegram credentials.
      - Connect Email Manager main output to Text Response.

11. **Activate Workflow:**  
    - Validate all connections and credentials.  
    - Activate workflow.  
    - Interact with Telegram bot via voice or text messages.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                     | Context or Link                                                                                          |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| For more tutorials visit our [Youtube](https://www.youtube.com/@AIBIZElevate77)                                                                                                                                                  | Video tutorials for setup and customization                                                            |
| Setup instructions: Create Telegram bot, add credentials (Telegram, OpenAI, Gmail OAuth2), assign credentials to nodes, activate workflow, start chat.                                                                           | Sticky Note in workflow                                                                                  |
| Customize routing logic, add new integrations (e.g., Notion, Slack), adjust email drafting and sending logic, update transcription and AI model settings.                                                                       | Sticky Note in workflow                                                                                  |
| Session state managed with Memory Buffer Window node per Telegram chat session for context-aware AI responses.                                                                                                                   | Workflow design note                                                                                      |
| For further help or to customize, book appointment at [aibizelevate.com](https://aibizelevate.com/) or connect on [LinkedIn](https://www.linkedin.com/in/barbora-svobodova-461b92285/)                                           | Support and consultancy                                                                                   |

---

This document provides a detailed, stepwise, and comprehensive reference for the "Telegram Email Assistant Talking Voice" workflow integrating Telegram, OpenAI, and Gmail for voice- and text-driven email management.