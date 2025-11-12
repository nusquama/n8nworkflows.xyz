Voice-to-Email Response System with Telegram, OpenAI Whisper & Gmail

https://n8nworkflows.xyz/workflows/voice-to-email-response-system-with-telegram--openai-whisper---gmail-3930


# Voice-to-Email Response System with Telegram, OpenAI Whisper & Gmail

### 1. Workflow Overview

This workflow automates replying to emails using voice messages via Telegram, leveraging OpenAI Whisper for transcription and ChatGPT for crafting polished email drafts. It is designed for users who prefer speaking their replies rather than typing them out, streamlining email response processes.

The workflow consists of two main logical blocks:

- **1.1 Incoming Email Processing:** Triggered by new emails in the Gmail inbox, this block filters emails that require a response using AI-based analysis, then sends a formatted message to the user’s Telegram bot to prompt a voice reply.

- **1.2 Telegram Voice Reply Handling:** Triggered when the Telegram bot receives a voice message reply, this block validates the reply, downloads and transcribes the audio using OpenAI Whisper, then uses ChatGPT to generate a polished email draft and creates it in Gmail, notifying the user via Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 Incoming Email Processing

**Overview:**  
Handles new incoming emails, filters those needing a response using ChatGPT, and notifies the user on Telegram with email details to enable voice reply.

**Nodes Involved:**  
- New Email Received  
- In the Inbox?  
- OpenAI Chat Model  
- Structured Output Parser  
- Does Email Need a Response?  
- Needs a response?  
- Set Chat ID  
- Text Email

**Node Details:**

- **New Email Received**  
  - *Type:* Gmail Trigger  
  - *Role:* Triggers workflow on new email arrival every minute.  
  - *Configuration:* Polls Gmail inbox; fetches full email data (not simple).  
  - *Input:* Gmail inbox.  
  - *Output:* Email JSON including sender, subject, labels, etc.  
  - *Failures:* Authentication errors, Gmail API rate limits, network issues.

- **In the Inbox?**  
  - *Type:* If node  
  - *Role:* Filters emails to process only those labeled "INBOX" (excludes sent or archived).  
  - *Logic:* Checks if the email’s labelIds array contains "INBOX".  
  - *Input:* Output of New Email Received.  
  - *Output:* Continues only if email is in inbox.  
  - *Failures:* Expression errors if labelIds missing.

- **OpenAI Chat Model**  
  - *Type:* LangChain OpenAI Chat Model  
  - *Role:* Runs AI model to analyze if the email needs a reply.  
  - *Configuration:* Uses OpenAI API; base URL specified; credentials linked to OpenAI account.  
  - *Input:* Email content passed as prompt text.  
  - *Output:* AI textual response (JSON string indicating "Y" or "N").  
  - *Failures:* API key/auth errors, OpenAI service outages.

- **Structured Output Parser**  
  - *Type:* LangChain Structured Output Parser  
  - *Role:* Parses AI output strictly as JSON with a schema example expecting { "response": "Y" or "N" }.  
  - *Input:* Output from OpenAI Chat Model.  
  - *Output:* Structured object for downstream logic.  
  - *Failures:* Parsing errors if AI output malformed.

- **Does Email Need a Response?**  
  - *Type:* LangChain Chain LLM  
  - *Role:* Wraps the AI prompt and output parser into one chain node (combined with OpenAI and Structured Output Parser).  
  - *Input:* Email JSON.  
  - *Output:* Parsed response indicating whether to respond.  
  - *Failures:* Same as above for AI and parsing.

- **Needs a response?**  
  - *Type:* If node  
  - *Role:* Checks if AI response is "Y" (needs reply).  
  - *Input:* Output of Does Email Need a Response?  
  - *Output:* Proceeds if response is "Y".  
  - *Failures:* Expression errors if response field missing.

- **Set Chat ID**  
  - *Type:* Set node  
  - *Role:* Assigns a fixed Telegram chat ID variable for downstream Telegram communication.  
  - *Configuration:* Hardcoded chat_id string (user must edit to their chat ID).  
  - *Input:* None (starts new data).  
  - *Output:* Adds chat_id field for Telegram node.  
  - *Failures:* User must manually update chat_id or message won’t be sent correctly.

- **Text Email**  
  - *Type:* Telegram node (Send Message)  
  - *Role:* Sends email details (ID, thread, from, subject, snippet) formatted as markdown to Telegram chat via VoicerEmailer bot.  
  - *Configuration:* Escapes special characters in email text to prevent markdown issues; truncated to first 100 characters with ellipsis.  
  - *Input:* Chat ID from Set Chat ID; email data from New Email Received.  
  - *Output:* Telegram message sent.  
  - *Failures:* Telegram API auth errors; malformed chat ID.

---

#### 2.2 Telegram Voice Reply Handling

**Overview:**  
Triggered by Telegram bot receiving a message, validates that the message is a voice reply to the email notification, downloads and transcribes the audio, then uses AI to compose a polished reply and drafts an email in Gmail, finally notifying the user.

**Nodes Involved:**  
- Telegram Bot Message Received  
- Is Type Audio Message + Reply?  
- It needs to be an audio message + a reply!  
- Get Audio File  
- OpenAI (Whisper Transcription)  
- Write Polished Reply  
- Create Email Draft  
- Direct to Draft

**Node Details:**

- **Telegram Bot Message Received**  
  - *Type:* Telegram Trigger  
  - *Role:* Listens for any new message sent to the Telegram bot.  
  - *Configuration:* Triggers on "message" update type; credentials linked to Telegram bot.  
  - *Input:* Telegram messages JSON.  
  - *Output:* Message data for validation.  
  - *Failures:* Telegram webhook setup errors, bot permissions.

- **Is Type Audio Message + Reply?**  
  - *Type:* If node  
  - *Role:* Checks that the Telegram message is both a reply to a previous message and contains a voice audio message.  
  - *Logic:* Validates existence of .reply_to_message and .voice nodes in the message.  
  - *Input:* Telegram Bot Message Received node.  
  - *Output:* True branch if message is a voice reply; False branch otherwise.  
  - *Failures:* Expression errors if message structure unexpected.

- **It needs to be an audio message + a reply!**  
  - *Type:* Telegram node (Send Message)  
  - *Role:* Sends error message to user if message is not a voice reply, instructing correct usage.  
  - *Input:* False branch from If node.  
  - *Output:* Telegram message back to user.  
  - *Failures:* Telegram API errors.

- **Get Audio File**  
  - *Type:* Telegram node (Get File)  
  - *Role:* Downloads the voice audio file from Telegram servers using the file_id.  
  - *Configuration:* Extracts file_id from incoming voice message JSON.  
  - *Input:* True branch from Is Type Audio Message + Reply?  
  - *Output:* File binary data for transcription.  
  - *Failures:* Telegram file download errors, invalid file_id.

- **OpenAI (Whisper Transcription)**  
  - *Type:* LangChain OpenAI Audio Node  
  - *Role:* Sends audio file to OpenAI Whisper API for transcription.  
  - *Configuration:* Uses OpenAI credentials; operation set to transcribe audio.  
  - *Input:* Binary audio file from Get Audio File node.  
  - *Output:* Text transcription of voice message.  
  - *Failures:* Audio file format issues, API errors, timeout.

- **Write Polished Reply**  
  - *Type:* LangChain Chain LLM  
  - *Role:* Uses ChatGPT to write a polished email reply based on the original email and the transcribed voice note.  
  - *Configuration:* Prompt instructs to rephrase voice note into a professionally toned email reply without subject line, includes sign-off. Examples provided. Output is parsed JSON.  
  - *Input:* Original email text from Telegram Bot Message Received (reply_to_message.text) and transcription from OpenAI node.  
  - *Output:* Polished email reply text.  
  - *Failures:* AI service errors, malformed input, prompt misinterpretation.

- **Create Email Draft**  
  - *Type:* Gmail node (Create Draft)  
  - *Role:* Creates a draft email in Gmail with the polished reply, in the same thread as the original email.  
  - *Configuration:* Uses Gmail OAuth2 credentials; extracts recipient email, threadId, and subject via regex from Telegram message text; message body from Write Polished Reply output.  
  - *Input:* Polished reply text and Telegram message metadata.  
  - *Output:* Gmail draft email created.  
  - *Failures:* Gmail API authorization issues, regex failures on Telegram message text, thread ID mismatch.

- **Direct to Draft**  
  - *Type:* Telegram node (Send Message)  
  - *Role:* Sends confirmation message to Telegram user including the AI-generated reply and a link to the Gmail draft thread.  
  - *Configuration:* Replies to the original Telegram message; formats URL to Gmail web with thread ID.  
  - *Input:* Output of Create Email Draft and Telegram message data.  
  - *Output:* Telegram message sent to user.  
  - *Failures:* Telegram API errors, broken link formatting.

---

### 3. Summary Table

| Node Name                     | Node Type                                   | Functional Role                             | Input Node(s)               | Output Node(s)                  | Sticky Note                                                                                              |
|-------------------------------|---------------------------------------------|---------------------------------------------|-----------------------------|--------------------------------|---------------------------------------------------------------------------------------------------------|
| New Email Received             | Gmail Trigger                              | Triggers on new email arrival               | —                           | In the Inbox?                  | ## 1. New Email Received: Triggered only on inbox emails, excluding sent folder.                         |
| In the Inbox?                  | If                                          | Filters emails labeled INBOX                 | New Email Received           | Does Email Need a Response?     | ## 1. New Email Received: Only process emails in the inbox.                                             |
| OpenAI Chat Model              | LangChain OpenAI Chat Model                  | Runs AI to determine if reply is needed     | Does Email Need a Response?  | Does Email Need a Response?     | ## 2. Check If Email Needs a Response: Uses AI to filter emails needing reply, ignoring newsletters.     |
| Structured Output Parser       | LangChain Structured Output Parser           | Parses AI JSON output                        | OpenAI Chat Model            | Does Email Need a Response?     | ## 2. Check If Email Needs a Response: Parses AI output strictly as JSON.                                |
| Does Email Need a Response?    | LangChain Chain LLM                          | Chains AI prompt and output parsing          | In the Inbox?, OpenAI nodes  | Needs a response?               | ## 2. Check If Email Needs a Response: Combines AI prompt and parsing.                                   |
| Needs a response?              | If                                          | Branches workflow if reply needed           | Does Email Need a Response?  | Set Chat ID                    | ## 2. Check If Email Needs a Response: Checks AI response Y/N.                                          |
| Set Chat ID                   | Set                                          | Sets Telegram chat ID variable               | Needs a response?            | Text Email                    | ## Edit here: User must set their Telegram chat ID here.                                                |
| Text Email                   | Telegram                                    | Sends email details message to Telegram     | Set Chat ID                  | —                              | ## 3. Send Email to Telegram: Sends email info to user via Telegram bot.                                |
| Telegram Bot Message Received  | Telegram Trigger                            | Triggers on incoming Telegram bot messages  | —                           | Is Type Audio Message + Reply? | ## 4. Telegram Reply Received: Triggered by Telegram bot messages.                                     |
| Is Type Audio Message + Reply? | If                                          | Checks message is voice reply to bot message| Telegram Bot Message Received| Get Audio File, It needs to be an audio message + a reply! | ## 4. Telegram Reply Received: Validates audio reply format.                                            |
| It needs to be an audio message + a reply! | Telegram                             | Sends error message if message invalid      | Is Type Audio Message + Reply? (false branch) | —                              | ## 4. Telegram Reply Received: Explains correct usage to user.                                         |
| Get Audio File               | Telegram                                    | Downloads Telegram voice message audio file | Is Type Audio Message + Reply? (true branch) | OpenAI (Whisper)              | ## 5. Audio Transcription: Downloads audio for transcription.                                          |
| OpenAI                      | LangChain OpenAI Audio                      | Transcribes audio to text using Whisper API | Get Audio File               | Write Polished Reply           | ## 5. Audio Transcription: Uses Whisper API for transcription.                                         |
| Write Polished Reply          | LangChain Chain LLM                          | Uses ChatGPT to polish voice note reply     | OpenAI, Telegram Bot Message Received | Create Email Draft            | ## 5. Create Email Draft: AI crafts professional email reply text.                                     |
| Create Email Draft            | Gmail                                       | Creates Gmail draft reply in original thread| Write Polished Reply         | Direct to Draft                | ## 5. Create Email Draft: Creates draft email and links it back to original thread.                     |
| Direct to Draft              | Telegram                                    | Sends confirmation and draft link to Telegram user | Create Email Draft         | —                              | ## 5. Create Email Draft: Notifies user with draft and link.                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger Node**  
   - Type: Gmail Trigger  
   - Trigger on new emails every minute  
   - Set `simple` to false to get full email details  
   - Connect Gmail OAuth2 credentials

2. **Add If Node "In the Inbox?"**  
   - Condition: Check if `labelIds` array contains "INBOX"  
   - Input: Output from Gmail Trigger  
   - Proceed only if true

3. **Add LangChain OpenAI Chat Model Node**  
   - Use OpenAI API with your OpenAI credentials  
   - Prompt: Provide full email content asking if response is needed (Y/N)  
   - Base URL: https://api.openai.com/v1

4. **Add Structured Output Parser Node**  
   - JSON schema expects `{ "response": "Y" | "N" }`  
   - Connect output from OpenAI Chat Model

5. **Add LangChain Chain LLM Node "Does Email Need a Response?"**  
   - Chain OpenAI Chat Model and Structured Output Parser for integrated processing  
   - Input: Email JSON from Inbox check  
   - Output: Parsed AI response

6. **Add If Node "Needs a response?"**  
   - Condition: Check if parsed response equals "Y"  
   - Input: Output of Chain LLM node

7. **Add Set Node "Set Chat ID"**  
   - Assign a string variable `chat_id` with your Telegram chat ID (find from execution data)  
   - Input: True branch from "Needs a response?"

8. **Add Telegram Node "Text Email" (Send Message)**  
   - Use Telegram API credentials (bot token)  
   - Chat ID: Use `chat_id` from Set node  
   - Text: Format email details (ID, thread ID, from name/email, subject, snippet) with escaped markdown special characters  
   - Input: Output from Set Chat ID node

9. **Create Telegram Trigger Node "Telegram Bot Message Received"**  
   - Listen for "message" updates from your Telegram bot  
   - Use same Telegram credentials

10. **Add If Node "Is Type Audio Message + Reply?"**  
    - Conditions:  
      - Message has a `reply_to_message` object  
      - Message has a `voice` object  
    - Input: Telegram Bot Trigger

11. **Add Telegram Node "It needs to be an audio message + a reply!"**  
    - Send message back to user instructing correct usage  
    - Chat ID: `message.chat.id` from Telegram trigger  
    - Input: False branch from previous If node

12. **Add Telegram Node "Get Audio File"**  
    - Operation: Get file  
    - File ID: Extract from `message.voice.file_id`  
    - Input: True branch from If node

13. **Add OpenAI Audio Node**  
    - Operation: Transcribe audio  
    - Input: Binary of audio file from Telegram Get Audio File node  
    - Use OpenAI credentials

14. **Add LangChain Chain LLM Node "Write Polished Reply"**  
    - Prompt includes original email (from `reply_to_message.text`) and transcription text  
    - Instruct AI to produce a polished reply without subject line, matching voice tone, includes sign-off  
    - Use OpenAI credentials

15. **Add Gmail Node "Create Email Draft"**  
    - Operation: Create draft  
    - Use Gmail OAuth2 credentials  
    - Extract recipient email, thread ID, subject from Telegram reply message text via regex  
    - Message body: Output from Write Polished Reply node

16. **Add Telegram Node "Direct to Draft"**  
    - Send message confirming draft creation  
    - Include AI-generated reply and link to Gmail thread (formatted URL)  
    - Reply to original Telegram message  
    - Chat ID: From Telegram Bot Message Received reply_to_message chat id

17. **Wire the nodes as per the logical flow described in section 1 and 2.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                         | Context or Link                                                                                  |
|--------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Create a Telegram bot following https://core.telegram.org/bots/tutorial to enable bot messaging capabilities.                        | Telegram Bot creation instructions                                                             |
| Add your Gmail and OpenAI credentials in n8n before enabling the workflow.                                                           | n8n credentials configuration                                                                   |
| Set your Telegram Chat ID by sending a message to your bot and retrieving the chat_id from the workflow executions tab.             | Instructions in Sticky Note2 and workflow setup                                                |
| The workflow escapes markdown special characters before sending email text to Telegram to avoid formatting issues.                  | See node "Text Email" message formatting code                                                 |
| Voice transcription is performed using OpenAI Whisper API via LangChain OpenAI Audio node.                                            | OpenAI Whisper API documentation                                                                |
| AI prompt for filtering emails includes criteria to ignore newsletters, no-reply senders, and informational emails.                 | See Sticky Note on "Check If Email Needs a Response"                                           |
| AI prompt for polishing replies instructs to keep tone and content consistent with voice note, including a sign-off, no subject line.| See "Write Polished Reply" node parameters                                                    |
| The final Telegram message includes a clickable link to the Gmail draft thread for quick access.                                      | Link format: https://mail.google.com/mail/#all/{{ threadId }}                                  |

---

This document fully describes the workflow titled **Voice-to-Email Response System with Telegram, OpenAI Whisper & Gmail**, capturing its complete structure, logic, nodes, and setup instructions for advanced users and AI agents.