Voice-to-Ideas: Transcribe Telegram Voice Notes with OpenAI Whisper to Google Sheets

https://n8nworkflows.xyz/workflows/voice-to-ideas--transcribe-telegram-voice-notes-with-openai-whisper-to-google-sheets-10806


# Voice-to-Ideas: Transcribe Telegram Voice Notes with OpenAI Whisper to Google Sheets

### 1. Workflow Overview

This workflow, **Voice-to-Ideas: Transcribe Telegram Voice Notes with OpenAI Whisper to Google Sheets**, is designed to automatically capture voice notes sent to a Telegram bot, transcribe them using OpenAI Whisper, and store the raw transcription in a Google Sheet along with the date. It also handles plain text messages sent to the same Telegram bot, saving them directly to the Google Sheet with timestamps.

**Target Use Cases:**  
- Creators, entrepreneurs, writers, or anyone who wants to quickly record ideas via voice on Telegram without typing.  
- Storing unformatted voice note transcriptions or text messages as raw data for later reference or processing.

**Logical Blocks:**  
- **1.1 Input Reception:** Triggered by new Telegram messages to the bot (voice or text).  
- **1.2 Message Type Detection:** Differentiates voice notes from text messages.  
- **1.3 Voice Note Processing:** Downloads voice files and transcribes them via OpenAI Whisper.  
- **1.4 Text Message Processing:** Prepares and saves plain text messages.  
- **1.5 Data Storage:** Appends transcriptions or text messages with timestamps to Google Sheets.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Listens for new messages from the Telegram bot, including both voice notes and text messages, to initiate the workflow.

- **Nodes Involved:**  
  - Receive Telegram Message

- **Node Details:**  
  - **Receive Telegram Message**  
    - *Type:* Telegram Trigger  
    - *Role:* Entry point, triggers when the Telegram bot receives a message.  
    - *Configuration:* Watches for `"message"` update type only. Uses the Telegram API credentials linked to the bot.  
    - *Input:* Incoming Telegram update webhook.  
    - *Output:* JSON containing Telegram message data (voice or text).  
    - *Edge Cases:* Telegram API downtime, invalid bot token, webhook misconfiguration.  
    - *Sticky Note:* "Triggers the workflow whenever your Telegram bot receives a new message. Captures both text messages and voice notes."

#### 1.2 Message Type Detection

- **Overview:**  
  Checks whether the incoming Telegram message contains a voice note or plain text and routes the workflow accordingly.

- **Nodes Involved:**  
  - Detect Message Type

- **Node Details:**  
  - **Detect Message Type**  
    - *Type:* Switch  
    - *Role:* Branching logic based on message content.  
    - *Configuration:*  
      - Checks if `message.voice` exists → routes to voice branch.  
      - Checks if `message.text` exists → routes to text branch.  
    - *Input:* JSON from Telegram trigger node.  
    - *Output:* Two outputs — one for voice messages, one for text messages.  
    - *Edge Cases:* Messages without voice or text (e.g., stickers, photos) are not explicitly handled and will be dropped.  
    - *Sticky Note:* "Checks whether the incoming Telegram message is a voice note or plain text, and routes the workflow down the correct path."

#### 1.3 Voice Note Processing

- **Overview:**  
  Downloads the voice message file from Telegram and transcribes it into text using OpenAI Whisper.

- **Nodes Involved:**  
  - Download Voice File  
  - Transcribe Voice Note

- **Node Details:**  
  - **Download Voice File**  
    - *Type:* Telegram node (File download)  
    - *Role:* Fetches the actual audio file from Telegram servers using the `file_id` from the voice message.  
    - *Configuration:* Uses `message.voice.file_id` to download the file. Credentials use the same Telegram bot API.  
    - *Input:* JSON from Switch node voice output.  
    - *Output:* Binary audio file data for transcription.  
    - *Edge Cases:* File not found on Telegram, API errors, large file size limitations.  
    - *Sticky Note:* "Downloads the actual audio file from Telegram so it can be sent to OpenAI for transcription."

  - **Transcribe Voice Note**  
    - *Type:* OpenAI (LangChain node)  
    - *Role:* Sends the downloaded audio file to OpenAI Whisper for transcription.  
    - *Configuration:*  
      - Resource: Audio  
      - Operation: Translate (used here for transcription)  
      - Credentials: OpenAI API key with Whisper enabled.  
    - *Input:* Binary audio file from previous node.  
    - *Output:* Text transcription of the voice note.  
    - *Edge Cases:* API key invalid/expired, transcription errors, audio format unsupported, timeouts.  
    - *Sticky Note:* "Uses OpenAI to convert the voice recording into plain text. Produces raw transcription without rewriting or formatting."

#### 1.4 Text Message Processing

- **Overview:**  
  Takes plain text messages from Telegram and prepares them for storage.

- **Nodes Involved:**  
  - Prepare Text Message

- **Node Details:**  
  - **Prepare Text Message**  
    - *Type:* Set node  
    - *Role:* Extracts the plain text from the Telegram message and formats it into a single field named `message` for Google Sheets.  
    - *Configuration:* Assigns `message` = `{{$json.message.text}}`  
    - *Input:* JSON from Switch node text output.  
    - *Output:* JSON with a single `message` field.  
    - *Edge Cases:* Empty or malformed text messages (unlikely due to Switch node conditions).  
    - *Sticky Note:* "Formats the text from Telegram into a structure suitable for Google Sheets so it can be saved just like voice notes."

#### 1.5 Data Storage

- **Overview:**  
  Appends either the transcribed voice note or the plain text message to a Google Sheet with a timestamp.

- **Nodes Involved:**  
  - Save Transcribed Note  
  - Save Text Message

- **Node Details:**  
  - **Save Transcribed Note**  
    - *Type:* Google Sheets  
    - *Role:* Appends a new row with the transcription text and current date/time to the configured Google Sheet.  
    - *Configuration:*  
      - Operation: Append  
      - Document ID and Sheet Name: configured to target the correct Google Sheet and worksheet (not shown in JSON).  
      - Data: transcription text and timestamp.  
    - *Input:* Text transcription from OpenAI node.  
    - *Output:* Confirmation of row append.  
    - *Credentials:* Google Sheets OAuth2.  
    - *Edge Cases:* Credential expiration, sheet access permissions, document ID or sheet name misconfiguration.  
    - *Sticky Note:* "Appends the transcribed text to your Google Sheet. Creates a new row containing your voice note and a timestamp."

  - **Save Text Message**  
    - *Type:* Google Sheets  
    - *Role:* Appends a new row with the original text message and current date/time to the same Google Sheet.  
    - *Configuration:* Same as above, adapted for text message input.  
    - *Input:* Prepared text message from Set node.  
    - *Output:* Confirmation of row append.  
    - *Credentials:* Same Google Sheets OAuth2.  
    - *Edge Cases:* Same as above.  
    - *Sticky Note:* "Appends the text message to your Google Sheet. Saves the original message along with the current date."

---

### 3. Summary Table

| Node Name             | Node Type                  | Functional Role                               | Input Node(s)           | Output Node(s)             | Sticky Note                                                                                                                                                     |
|-----------------------|----------------------------|-----------------------------------------------|-------------------------|----------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Sticky Note           | Sticky Note                | Workflow overview and instructions            | —                       | —                          | ## Voice-to-Ideas: Auto-Transcribe Telegram Voice Notes to Google Sheets... (full content as given)                                                             |
| Receive Telegram Message | Telegram Trigger          | Entry point: triggers on new Telegram message | —                       | Detect Message Type         | Triggers the workflow whenever your Telegram bot receives a new message. Captures both text messages and voice notes.                                          |
| Detect Message Type    | Switch                    | Routes messages by type (voice or text)       | Receive Telegram Message | Download Voice File, Prepare Text Message | Checks whether the incoming Telegram message is a voice note or plain text, and routes the workflow down the correct path.                                        |
| Download Voice File    | Telegram (File)            | Downloads voice message audio from Telegram   | Detect Message Type (voice) | Transcribe Voice Note        | Downloads the actual audio file from Telegram so it can be sent to OpenAI for transcription.                                                                    |
| Transcribe Voice Note  | OpenAI (LangChain)         | Transcribes audio to text with OpenAI Whisper | Download Voice File      | Save Transcribed Note       | Uses OpenAI to convert the voice recording into plain text. Produces raw transcription without rewriting or formatting.                                         |
| Save Transcribed Note  | Google Sheets              | Saves transcribed text and date to Google Sheets | Transcribe Voice Note    | —                          | Appends the transcribed text to your Google Sheet. Creates a new row containing your voice note and a timestamp.                                                |
| Prepare Text Message   | Set                       | Formats plain text message for saving          | Detect Message Type (text) | Save Text Message            | Formats the text from Telegram into a structure suitable for Google Sheets so it can be saved just like voice notes.                                            |
| Save Text Message      | Google Sheets              | Saves text messages and date to Google Sheets  | Prepare Text Message     | —                          | Appends the text message to your Google Sheet. Saves the original message along with the current date.                                                          |
| Sticky Note1          | Sticky Note                | Telegram trigger explanation                    | —                       | —                          | Triggers the workflow whenever your Telegram bot receives a new message. Captures both text messages and voice notes.                                          |
| Sticky Note2          | Sticky Note                | Switch node explanation                         | —                       | —                          | Checks whether the incoming Telegram message is a voice note or plain text, and routes the workflow down the correct path.                                      |
| Sticky Note3          | Sticky Note                | Download audio explanation                       | —                       | —                          | Downloads the actual audio file from Telegram so it can be sent to OpenAI for transcription.                                                                    |
| Sticky Note4          | Sticky Note                | OpenAI transcription explanation                | —                       | —                          | Uses OpenAI to convert the voice recording into plain text. Produces raw transcription without rewriting or formatting.                                         |
| Sticky Note5          | Sticky Note                | Google Sheets append explanation                 | —                       | —                          | Appends the transcribed text to your Google Sheet. Creates a new row containing your voice note and a timestamp.                                                |
| Sticky Note6          | Sticky Note                | Text preparation explanation                      | —                       | —                          | Formats the text from Telegram into a structure suitable for Google Sheets so it can be saved just like voice notes.                                            |
| Sticky Note7          | Sticky Note                | Text message saving explanation                   | —                       | —                          | Appends the text message to your Google Sheet. Saves the original message along with the current date.                                                          |
| Sticky Note8          | Sticky Note                | Input reception block explanation                  | —                       | —                          | Receives new Telegram messages from your bot, including voice notes and text. The message type is detected so the workflow can follow the correct branch.       |
| Sticky Note9          | Sticky Note                | Voice message processing explanation              | —                       | —                          | Handles audio messages. Downloads the voice file from Telegram, sends it to OpenAI for transcription, and saves the transcribed text to Google Sheets.          |
| Sticky Note10         | Sticky Note                | Text message processing explanation                | —                       | —                          | Processes regular text messages. Formats the incoming text and saves it directly to your Google Sheet with the current timestamp.                               |
| Sticky Note11         | Sticky Note                | Data storage explanation                            | —                       | —                          | Both branches end here. Each message transcribed or typed is appended as a new row in Google Sheets under the Stories and Date columns.                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Bot and Credentials:**  
   - Use BotFather on Telegram to create a bot and get the Bot Token.  
   - In n8n, create Telegram API credentials using this token.

2. **Create Google Sheets Credentials:**  
   - In Google Cloud Console, create OAuth2 credentials for Google Sheets API.  
   - Connect these credentials in n8n under Google Sheets OAuth2.

3. **Prepare Google Sheet:**  
   - Create a Google Sheet with two columns:  
     - `Notes` (for storing transcription/text)  
     - `Date` (for timestamp)

4. **Create Telegram Trigger Node:**  
   - Add a Telegram Trigger node named “Receive Telegram Message.”  
   - Set it to listen for `"message"` updates.  
   - Attach the Telegram API credentials.

5. **Add Switch Node to Detect Message Type:**  
   - Add a Switch node named “Detect Message Type.”  
   - Set two outputs:  
     - Output 1 named “voice”: Condition checks if `{{$json.message.voice}}` exists.  
     - Output 2 named “text”: Condition checks if `{{$json.message.text}}` exists.

6. **Voice Branch:**  
   - Create a Telegram node named “Download Voice File.”  
   - Set resource to `file`.  
   - Set `fileId` to `{{$json.message.voice.file_id}}`.  
   - Use the Telegram API credentials.

   - Add an OpenAI node named “Transcribe Voice Note.”  
   - Use LangChain integration or standard OpenAI node with:  
     - Resource: Audio  
     - Operation: Translate (for transcription)  
   - Attach OpenAI API credentials with Whisper enabled.

   - Add a Google Sheets node named “Save Transcribed Note.”  
   - Operation: Append  
   - Set Document ID and Sheet Name to your Google Sheet.  
   - Map data fields:  
     - `Notes`: transcription text from OpenAI node output  
     - `Date`: current timestamp (use expression like `{{ $now }}`)

7. **Text Branch:**  
   - Add a Set node named “Prepare Text Message.”  
   - Create a field `message` with value `{{$json.message.text}}`.

   - Add a Google Sheets node named “Save Text Message.”  
   - Operation: Append  
   - Set Document ID and Sheet Name to your Google Sheet.  
   - Map data fields:  
     - `Notes`: `{{ $json.message }}` (from Set node)  
     - `Date`: current timestamp (`{{ $now }}`)

8. **Connect Nodes:**  
   - Connect Telegram Trigger to Switch node.  
   - Connect Switch node voice output to “Download Voice File.”  
   - Connect “Download Voice File” to “Transcribe Voice Note.”  
   - Connect “Transcribe Voice Note” to “Save Transcribed Note.”  
   - Connect Switch node text output to “Prepare Text Message.”  
   - Connect “Prepare Text Message” to “Save Text Message.”

9. **Test Workflow:**  
   - Send a voice note and a text message to your Telegram bot.  
   - Confirm transcriptions and messages appear in your Google Sheet with timestamps.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                        | Context or Link                                                                                                     |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| This workflow is ideal for creators and entrepreneurs wanting to quickly capture spoken ideas via Telegram without typing.                                        | Sticky Note overview content                                                                                        |
| Requires a Telegram bot token created with BotFather and credentials setup in n8n.                                                                                | Setup steps in Sticky Note                                                                                          |
| OpenAI API key must have Whisper transcription enabled to transcribe voice notes.                                                                                 | Workflow requirements                                                                                              |
| Google Sheet must have exactly two columns: Notes and Date, for proper data storage.                                                                              | Workflow requirements                                                                                              |
| For more on Telegram bot creation, see https://core.telegram.org/bots                                                                                             | External documentation reference                                                                                   |
| For OpenAI Whisper API details, see https://platform.openai.com/docs/guides/speech-to-text                                                                       | OpenAI API documentation                                                                                           |
| Google Sheets OAuth2 credentials configuration details are available at https://developers.google.com/sheets/api/quickstart/js                                        | Google Sheets API documentation                                                                                    |

---

**Disclaimer:** The provided text is derived exclusively from an n8n automated workflow. It strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.