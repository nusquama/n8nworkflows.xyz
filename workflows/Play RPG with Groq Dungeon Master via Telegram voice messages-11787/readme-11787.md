Play RPG with Groq Dungeon Master via Telegram voice messages

https://n8nworkflows.xyz/workflows/play-rpg-with-groq-dungeon-master-via-telegram-voice-messages-11787


# Play RPG with Groq Dungeon Master via Telegram voice messages

### 1. Workflow Overview

This workflow enables playing a persistent Dungeons & Dragons 3.5 RPG campaign through Telegram voice messages, using AI powered by Groq. It acts as a Dungeon Master (DM) that understands player voice commands, manages game state, enforces game rules, and narrates the adventure using text-to-speech.  
The logical flow is structured into four main blocks:

- **1.1 Input Reception**: Receives voice messages from Telegram, downloads the audio, and prepares it for processing.
- **1.2 Speech-to-Text Conversion**: Converts the received audio message to text using Groq’s Whisper STT model.
- **1.3 RPG Agent and Memory Processing**: The core AI agent (Dungeon Master) processes the player’s input, accesses persistent campaign memory, applies D&D 3.5 rules, and generates the game narration and state updates.
- **1.4 Text-to-Speech and Response Delivery**: Converts the AI-generated narration back to audio and sends it as an audio message to the Telegram chat.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block captures voice messages sent by the player on Telegram, downloads the audio file, and prepares the data for transcription.

**Nodes Involved:**  
- Receive Voice Message  
- Download Audio File  
- Paste your Groq API key (prepares authorization for API calls)  

**Node Details:**  

- **Receive Voice Message**  
  - *Type:* Telegram Trigger  
  - *Role:* Listens for new messages, filtered for voice messages.  
  - *Configuration:* Triggers on "message" updates; uses Telegram Bot credentials.  
  - *Input/Output:* No input (trigger node), outputs message JSON including voice file ID.  
  - *Failure Modes:* Telegram API downtime, webhook misconfiguration, missing voice message file ID.  

- **Download Audio File**  
  - *Type:* Telegram Node (file resource)  
  - *Role:* Downloads the voice message audio file from Telegram servers using file ID.  
  - *Configuration:* Uses file ID from 'Receive Voice Message' node; download flag is false (file path retrieval).  
  - *Input:* Receives voice file ID from previous node.  
  - *Output:* Provides file path URL for further processing.  
  - *Edge Cases:* File not found, Telegram API limits.  

- **Paste your Groq API key**  
  - *Type:* Set  
  - *Role:* Stores Groq API key in workflow data for authorization in calls.  
  - *Configuration:* Hardcoded API key string.  
  - *Input:* Receives download file path info.  
  - *Output:* Exposes `groq_api_key` JSON field for headers.  
  - *Security Note:* API key is embedded here; recommend environment variables for production.  

---

#### 2.2 Speech-to-Text Conversion

**Overview:**  
Converts the downloaded audio file into a text transcript using the Groq Whisper model.

**Nodes Involved:**  
- Speech to Text  

**Node Details:**  

- **Speech to Text**  
  - *Type:* HTTP Request  
  - *Role:* Sends audio file URL to Groq’s audio transcription endpoint.  
  - *Configuration:*  
    - POST multipart/form-data with parameters:  
      - `url`: Telegram file URL (built dynamically)  
      - `model`: "whisper-large-v3" for speech recognition accuracy  
    - Authorization header uses Groq API key from previous node.  
  - *Input:* Receives Groq API key and file path URL.  
  - *Output:* Receives transcribed text in JSON response.  
  - *Edge Cases:*  
    - API rate limiting or authentication failure  
    - Audio file format unsupported or corrupt  
    - Network timeouts or malformed responses  

---

#### 2.3 RPG Agent and Memory Processing

**Overview:**  
Processes the transcribed player input within a persistent RPG campaign context, applies D&D 3.5 rules strictly, manages dice roll authorization and execution, and generates narration and state updates.

**Nodes Involved:**  
- The track of your adventure (memoryBufferWindow)  
- Groq Chat Model  
- Dungeon Master  

**Node Details:**  

- **The track of your adventure**  
  - *Type:* LangChain Memory Buffer Window  
  - *Role:* Maintains and provides a sliding window of recent conversation state and campaign memory for context.  
  - *Configuration:*  
    - Uses a global session key for persistence  
    - Context window length of 30 messages for balancing memory and performance  
  - *Input:* None directly; used as memory input to Dungeon Master.  
  - *Output:* Supplies memory context to the Dungeon Master agent.  
  - *Edge Cases:* Memory overflow, session key conflicts.  

- **Groq Chat Model**  
  - *Type:* LangChain Groq Chat Language Model  
  - *Role:* Provides the foundational language model for the agent to generate text.  
  - *Configuration:*  
    - Model: "groq/compound-mini"  
    - Empty options object (default settings)  
    - Uses Groq API credentials.  
  - *Input:* None directly (used as AI language model in agent).  
  - *Output:* Generates text completions based on prompts.  
  - *Edge Cases:* API quota limits, model latency or downtime.  

- **Dungeon Master**  
  - *Type:* LangChain Agent  
  - *Role:* Core AI agent managing the RPG logic.  
  - *Configuration:*  
    - Prompt includes detailed system message describing:  
      - Stateful campaign with external memory persistence  
      - Strict D&D 3.5 rules enforcement  
      - Dice roll authorization and execution rules  
      - Output constrained to plain text narration and explicit state update serialization  
      - Narration style suited for TTS  
      - Structured output format with scene description and options  
    - Input prompt dynamically includes player action text from transcription.  
    - Uses the Groq Chat Model as language model.  
    - Uses The track of your adventure as memory buffer.  
  - *Input:* Receives text from speech-to-text node and memory context.  
  - *Output:* Produces narration text and explicit state updates.  
  - *Edge Cases:*  
    - Agent misunderstanding roll authorization phrases  
    - Failure to serialize state changes properly  
    - Unexpected input text or commands  
    - API failures or rate limits  

---

#### 2.4 Text-to-Speech and Response Delivery

**Overview:**  
Converts the AI-generated narration text into an audio file using Groq TTS and sends it back to the player in the Telegram chat.

**Nodes Involved:**  
- Text to Speech  
- Send an audio file  

**Node Details:**  

- **Text to Speech**  
  - *Type:* HTTP Request  
  - *Role:* Sends narration text to Groq TTS endpoint to receive speech audio in WAV format.  
  - *Configuration:*  
    - POST to Groq speech endpoint with JSON body specifying:  
      - model: "playai-tts"  
      - voice: "Fritz-PlayAI"  
      - input: narration text (escaped for newlines)  
      - response_format: "wav"  
    - Authorization header uses Groq API key from the paste node.  
  - *Input:* Receives narration text from Dungeon Master.  
  - *Output:* Binary audio data (WAV).  
  - *Edge Cases:*  
    - API authorization failure  
    - Large input text size  
    - Network issues or malformed response  

- **Send an audio file**  
  - *Type:* Telegram Node  
  - *Role:* Sends the generated audio file to the Telegram chat where the original voice message came from.  
  - *Configuration:*  
    - chatId dynamically derived from the original message chat ID  
    - operation: sendAudio with binary data enabled  
    - Uses Telegram credentials  
  - *Input:* Binary audio from Text to Speech node.  
  - *Output:* Confirmation of sent message.  
  - *Edge Cases:* Telegram API failures, file size limits, incorrect chat ID.  

---

### 3. Summary Table

| Node Name               | Node Type                             | Functional Role                         | Input Node(s)         | Output Node(s)       | Sticky Note                                                                                                      |
|-------------------------|-------------------------------------|---------------------------------------|-----------------------|----------------------|------------------------------------------------------------------------------------------------------------------|
| Receive Voice Message    | Telegram Trigger                    | Receive voice messages from Telegram  | None                  | Download Audio File   | **1. Bot receives voice message and downloads it**                                                               |
| Download Audio File      | Telegram                           | Retrieve audio file path from Telegram| Receive Voice Message  | Paste your Groq API key | **1. Bot receives voice message and downloads it**                                                               |
| Paste your Groq API key  | Set                                | Store Groq API key for authorization  | Download Audio File    | Speech to Text       | **2. Using your entered API key Groq parses speech to text**                                                     |
| Speech to Text           | HTTP Request                      | Convert audio to text via Groq Whisper| Paste your Groq API key| Dungeon Master       | **2. Using your entered API key Groq parses speech to text**                                                     |
| The track of your adventure| LangChain Memory Buffer Window    | Manage persistent campaign memory     | None                  | Dungeon Master       | **3. Agent processes your input using saved data**                                                               |
| Groq Chat Model          | LangChain Groq Language Model      | Provide language model for agent      | None                  | Dungeon Master       | **3. Agent processes your input using saved data**                                                               |
| Dungeon Master           | LangChain Agent                   | RPG logic processing and narration    | Speech to Text, Memory, Groq Chat Model | Text to Speech | **3. Agent processes your input using saved data**                                                               |
| Text to Speech           | HTTP Request                      | Convert narration text to audio       | Dungeon Master        | Send an audio file   | **4. Using your entered API key Groq generates speech from text and sends it via the Bot**                       |
| Send an audio file       | Telegram                         | Send audio narration back to Telegram | Text to Speech        | None                 | **4. Using your entered API key Groq generates speech from text and sends it via the Bot**                       |
| Sticky Note              | Sticky Note                      | Documentation notes                    | None                  | None                 | **Dungeons & Goblins AI Dungeon Master (with voice in Telegram)** Workflow overview and usage instructions       |
| Sticky Note1             | Sticky Note                      | Documentation note                    | None                  | None                 | **3. Agent processes your input using saved data**                                                               |
| Sticky Note2             | Sticky Note                      | Documentation note                    | None                  | None                 | **1. Bot receives voice message and downloads it**                                                               |
| Sticky Note3             | Sticky Note                      | Documentation note                    | None                  | None                 | **2. Using your entered API key Groq parses speech to text**                                                     |
| Sticky Note4             | Sticky Note                      | Documentation note                    | None                  | None                 | **4. Using your entered API key Groq generates speech from text and sends it via the Bot**                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Name: "Receive Voice Message"  
   - Type: Telegram Trigger  
   - Configure to listen to "message" updates  
   - Connect Telegram Bot credentials  
   - Position: start of workflow

2. **Create Telegram Node to Download Audio**  
   - Name: "Download Audio File"  
   - Type: Telegram (file resource)  
   - Set `fileId` parameter dynamically from `Receive Voice Message` node’s voice message file ID path  
   - Set `download` to false (to get file path)  
   - Use same Telegram credentials  
   - Connect output of "Receive Voice Message" to this node

3. **Create Set Node for Groq API Key**  
   - Name: "Paste your Groq API key"  
   - Type: Set  
   - Add JSON field `groq_api_key` with your Groq API token string  
   - Connect output of "Download Audio File" to this node

4. **Create HTTP Request Node for Speech-to-Text**  
   - Name: "Speech to Text"  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.groq.com/openai/v1/audio/transcriptions`  
   - Content-Type: multipart/form-data  
   - Body parameters:  
     - `url` = `https://api.telegram.org/file/bot<YOUR_BOT_TOKEN>/{{ $json["result"]["file_path"] }}` (build dynamically from "Download Audio File")  
     - `model` = "whisper-large-v3"  
   - Headers: Authorization Bearer with Groq API key from "Paste your Groq API key" node  
   - Connect output of "Paste your Groq API key" to this node

5. **Create LangChain Memory Buffer Window Node**  
   - Name: "The track of your adventure"  
   - Type: LangChain Memory Buffer Window  
   - Session Key: "global_session"  
   - Session Id Type: customKey  
   - Context Window Length: 30  
   - No input connection needed; will be linked as memory input

6. **Create LangChain Groq Chat Model Node**  
   - Name: "Groq Chat Model"  
   - Type: LangChain Groq Chat Language Model  
   - Model: "groq/compound-mini"  
   - Credentials: Groq API account  
   - No direct input; used internally by agent

7. **Create LangChain Agent Node for Dungeon Master**  
   - Name: "Dungeon Master"  
   - Type: LangChain Agent  
   - Prompt Type: define  
   - Text input: `=Player action:\n{{ $json.text }}` (use transcription text from "Speech to Text")  
   - System message: Use the provided detailed campaign rules and behavior constraints (copy exact prompt from workflow)  
   - Options: System message as per the workflow  
   - Connect AI language model input to "Groq Chat Model"  
   - Connect AI memory input to "The track of your adventure"  
   - Connect main input to "Speech to Text" output

8. **Create HTTP Request Node for Text-to-Speech**  
   - Name: "Text to Speech"  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.groq.com/openai/v1/audio/speech`  
   - Content-Type: application/json  
   - JSON Body:  
     ```json
     {
       "model": "playai-tts",
       "input": "{{ $json.output.replace(/\\n/g, '\\n') }}",
       "voice": "Fritz-PlayAI",
       "response_format": "wav"
     }
     ```  
   - Headers: Authorization Bearer with Groq API key from "Paste your Groq API key" node  
   - Connect output of "Dungeon Master" to this node

9. **Create Telegram Node to Send Audio**  
   - Name: "Send an audio file"  
   - Type: Telegram  
   - Operation: sendAudio  
   - Chat ID: `={{ $('Receive Voice Message').item.json.message.chat.id }}` (dynamic from original message)  
   - Enable binary data sending  
   - Use Telegram credentials  
   - Connect output of "Text to Speech" to this node

10. **Connect all nodes in the order:**  
    Receive Voice Message → Download Audio File → Paste your Groq API key → Speech to Text → Dungeon Master → Text to Speech → Send an audio file

11. **Add credentials:**  
    - Telegram Bot token with permissions to receive messages and send audio  
    - Groq API token for both transcription and TTS

12. **Optional:** Add sticky notes to explain each block for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                           | Context or Link                                                                                             |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| This workflow strictly enforces D&D 3.5 rules, including dice roll authorization and automated roll execution, enhancing game integrity.                                                                                              | Workflow logic, Dungeon Master node system message                                                           |
| Dice rolls are only executed by the AI after explicit player authorization phrases to prevent cheating or inconsistencies.                                                                                                           | Dungeon Master node behavior constraints                                                                     |
| Campaign state is persistently stored outside the AI and passed back each turn to maintain continuity.                                                                                                                               | The track of your adventure (memory buffer) node                                                             |
| Telegram voice messages enable natural, hands-free interaction with the RPG DM.                                                                                                                                                       | Workflow use case                                                                                            |
| Groq Whisper model used for high-quality speech-to-text transcription; Groq TTS model synthesizes narration audio.                                                                                                                   | Speech to Text and Text to Speech nodes                                                                      |
| API keys are embedded in Set node for simplicity but consider using environment variables or encrypted credentials for production.                                                                                                    | Paste your Groq API key node                                                                                  |
| For detailed D&D 3.5 rules and game design, consult official D&D manuals and community resources.                                                                                                                                     | External reference (not included in workflow)                                                                |
| Workflow designed for long-running campaigns with consistent, machine-readable state serialization to enable save/load and debugging.                                                                                                | Overall workflow design                                                                                       |
| Telegram Bot setup must include webhook configuration for n8n and appropriate permissions.                                                                                                                                             | Telegram Trigger node configuration                                                                           |
| Workflow tested with n8n version supporting LangChain nodes and Groq integrations (version 1.3+ recommended).                                                                                                                          | Version-specific requirements                                                                                 |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.