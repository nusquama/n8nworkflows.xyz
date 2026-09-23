Transcribe Telegram voice messages with Palatine Speech

https://n8nworkflows.xyz/workflows/transcribe-telegram-voice-messages-with-palatine-speech-17726


# Transcribe Telegram voice messages with Palatine Speech

### 1. Workflow Overview

The **Transcribe Telegram Voice Messages with Palatine Speech** workflow automates the handling of incoming messages sent to a Telegram bot. Its primary objective is to make voice communication accessible by automatically transcribing voice notes into text using the Palatine Speech engine and replying to the sender with the transcription. Additionally, it ensures standard text messages are echoed back to maintain a responsive interactive experience.

The logical flow is grouped into three functional blocks:
- **1.1 Input Reception & Routing:** Listens for incoming webhook events from Telegram and uses conditional logic to split the flow based on whether the message is a voice note or plain text.
- **1.2 Audio Processing:** Downloads the raw binary audio file from Telegram's servers and sends it to the Palatine Speech API for asynchronous transcription.
- **1.3 Delivery:** Handles the outgoing communication back to the user, either by replying with the generated text transcription or by echoing the original text message.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Routing
- **Overview:** This block acts as the entry point for the automation, capturing user interactions from Telegram and inspecting the payload type to route messages appropriately.
- **Nodes Involved:** 
  - `Telegram Trigger`
  - `Voice or Text?`
- **Node Details:**
  - **Telegram Trigger**
    - *Type and Technical Role:* `n8n-nodes-base.telegramTrigger` (Trigger Node). Listens for incoming updates from the Telegram Bot API via webhook.
    - *Configuration Choices:* Configured to listen for `message` updates with default additional fields.
    - *Key Expressions or Variables:* None (generates the initial execution context).
    - *Input/Output Connections:* Output connects to `Voice or Text?`.
    - *Version-Specific Requirements:* Version 1.2.
    - *Edge Cases/Failure Types:* Webhook registration failure if credentials or network configurations are invalid; rate limiting by Telegram API.
  - **Voice or Text?**
    - *Type and Technical Role:* `n8n-nodes-base.switch` (Flow Control Node). Evaluates the incoming message structure to route execution down different paths.
    - *Configuration Choices:* Contains two output rules. Output 0 (“Voice”) checks if `={{ $json.message.voice?.file_id }}` exists. Output 1 (“Text”) checks if `={{ $json.message.text }}` exists.
    - *Key Expressions or Variables:* 
      - Voice condition: `={{ $json.message.voice?.file_id }}` (Operator: Exists)
      - Text condition: `={{ $json.message.text }}` (Operator: Exists)
    - *Input/Output Connections:* Input connected from `Telegram Trigger`. Output 0 connects to `Download Voice File`; Output 1 connects to `Echo Text Message`.
    - *Version-Specific Requirements:* Version 3.2.
    - *Edge Cases/Failure Types:* Unhandled message types (e.g., photos, stickers) will not match either rule and will cause the execution to halt at this node.

#### 2.2 Audio Processing
- **Overview:** This block retrieves the binary audio payload associated with a voice note from Telegram and leverages an external AI speech-to-text service to convert the speech into readable text.
- **Nodes Involved:**
  - `Download Voice File`
  - `Palatine Speech Transcription`
- **Node Details:**
  - **Download Voice File**
    - *Type and Technical Role:* `n8n-nodes-base.telegram` (Action Node). Downloads a file from Telegram servers using its unique file identifier.
    - *Configuration Choices:* Resource set to `file`, with the `fileId` parameter mapped dynamically to the incoming Telegram voice object.
    - *Key Expressions or Variables:* 
      - `={{ $json.message.voice.file_id }}`
    - *Input/Output Connections:* Input connected from `Voice or Text?` (Output 0); output connects to `Palatine Speech Transcription`.
    - *Version-Specific Requirements:* Version 1.2.
    - *Edge Cases/Failure Types:* Expired file identifiers or Telegram API downtime preventing binary download.
  - **Palatine Speech Transcription**
    - *Type and Technical Role:* `n8n-nodes-palatine-speech.palatineSpeech` (Community Action Node). Transcribes incoming audio binaries into text via the Palatine Speech API.
    - *Configuration Choices:* Model selected as `palatine_large_highspeed`, with default configurations for polling and request options.
    - *Key Expressions or Variables:* Processes the binary data stream passed automatically from the preceding download step.
    - *Input/Output Connections:* Input connected from `Download Voice File`; output connects to `Send Transcription Reply`.
    - *Version-Specific Requirements:* Version 3. Requires the `n8n-nodes-palatine-speech` community package.
    - *Edge Cases/Failure Types:* Unsupported audio formats, API authentication failures, or processing timeouts on large audio files.

#### 2.3 Delivery
- **Overview:** This block formats and sends the final response back to the correct chat identifier on Telegram, completing the conversation loop.
- **Nodes Involved:**
  - `Echo Text Message`
  - `Send Transcription Reply`
- **Node Details:**
  - **Echo Text Message**
    - *Type and Technical Role:* `n8n-nodes-base.telegram` (Action Node). Sends a text message back to the user via the Telegram Bot API.
    - *Configuration Choices:* Resource set to send text messages. Uses dynamic expressions to target the originating chat ID and repeat the input text.
    - *Key Expressions or Variables:*
      - Message text: `={{ $json.message.text }}`
      - Chat ID: `={{ $('Telegram Trigger').item.json.message.chat.id }}`
    - *Input/Output Connections:* Input connected from `Voice or Text?` (Output 1); no outgoing connections.
    - *Version-Specific Requirements:* Version 1.2.
    - *Edge Cases/Failure Types:* Bot blocked by the user or invalid chat ID.
  - **Send Transcription Reply**
    - *Type and Technical Role:* `n8n-nodes-base.telegram` (Action Node). Delivers the generated transcript text back to the originating chat.
    - *Configuration Choices:* Sends the transcription output from the AI model back to the user's chat ID.
    - *Key Expressions or Variables:*
      - Message text: `={{ $json.transcription }}`
      - Chat ID: `={{ $('Telegram Trigger').item.json.message.chat.id }}`
    - *Input/Output Connections:* Input connected from `Palatine Speech Transcription`; no outgoing connections.
    - *Version-Specific Requirements:* Version 1.2.
    - *Edge Cases/Failure Types:* Empty transcription output or network transmission errors.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Telegram Trigger | n8n-nodes-base.telegramTrigger | Listens for new messages sent to the Telegram bot | None | Voice or Text? | ## Transcribe Telegram Voice Messages with Palatine Speech<br><br>**Who's it for:** Anyone who would rather send a voice note to their Telegram bot than type.<br><br>Transcribes Telegram voice messages to text and replies with the transcript (text messages are echoed back).<br><br>### How it works<br>1. Telegram Trigger receives a new message<br>2. Switch routes voice vs. text<br>3. Download the voice file from Telegram<br>4. Palatine Speech transcribes the audio<br>5. Bot replies with the transcript<br><br>### Setup<br>1. Install `n8n-nodes-palatine-speech` in **Settings → Community Nodes**.<br>2. Add credentials: Telegram Bot API · Palatine Speech API.<br>3. Set the folder / chat / database IDs on the trigger and destination nodes.<br>4. Activate the workflow.<br><br>### Customization<br>- Reply with an AI summary instead of the raw transcript<br>- Log every transcript to Google Sheets<br>- Restrict the bot to specific chat IDs<br><br>**Requirements:** Telegram Bot API · Palatine Speech API<br><br>_Uses the community node **Palatine Speech** (`n8n-nodes-palatine-speech`). Install it from the nodes panel: press **+**, search `Palatine`, open the node and click **Install node**._<br>## 1. Receive the audio<br>Trigger and fetch the file that needs transcribing. |
| Voice or Text? | n8n-nodes-base.switch | Routes execution based on whether the message contains a voice note or text | Telegram Trigger | Download Voice File, Echo Text Message | ## Transcribe Telegram Voice Messages with Palatine Speech<br><br>**Who's it for:** Anyone who would rather send a voice note to their Telegram bot than type.<br><br>Transcribes Telegram voice messages to text and replies with the transcript (text messages are echoed back).<br><br>### How it works<br>1. Telegram Trigger receives a new message<br>2. Switch routes voice vs. text<br>3. Download the voice file from Telegram<br>4. Palatine Speech transcribes the audio<br>5. Bot replies with the transcript<br><br>### Setup<br>1. Install `n8n-nodes-palatine-speech` in **Settings → Community Nodes**.<br>2. Add credentials: Telegram Bot API · Palatine Speech API.<br>3. Set the folder / chat / database IDs on the trigger and destination nodes.<br>4. Activate the workflow.<br><br>### Customization<br>- Reply with an AI summary instead of the raw transcript<br>- Log every transcript to Google Sheets<br>- Restrict the bot to specific chat IDs<br><br>**Requirements:** Telegram Bot API · Palatine Speech API<br><br>_Uses the community node **Palatine Speech** (`n8n-nodes-palatine-speech`). Install it from the nodes panel: press **+**, search `Palatine`, open the node and click **Install node**._<br>## 1. Receive the audio<br>Trigger and fetch the file that needs transcribing. |
| Download Voice File | n8n-nodes-base.telegram | Downloads the binary voice recording using its file ID | Voice or Text? | Palatine Speech Transcription | ## Transcribe Telegram Voice Messages with Palatine Speech<br><br>**Who's it for:** Anyone who would rather send a voice note to their Telegram bot than type.<br><br>Transcribes Telegram voice messages to text and replies with the transcript (text messages are echoed back).<br><br>### How it works<br>1. Telegram Trigger receives a new message<br>2. Switch routes voice vs. text<br>3. Download the voice file from Telegram<br>4. Palatine Speech transcribes the audio<br>5. Bot replies with the transcript<br><br>### Setup<br>1. Install `n8n-nodes-palatine-speech` in **Settings → Community Nodes**.<br>2. Add credentials: Telegram Bot API · Palatine Speech API.<br>3. Set the folder / chat / database IDs on the trigger and destination nodes.<br>4. Activate the workflow.<br><br>### Customization<br>- Reply with an AI summary instead of the raw transcript<br>- Log every transcript to Google Sheets<br>- Restrict the bot to specific chat IDs<br><br>**Requirements:** Telegram Bot API · Palatine Speech API<br><br>_Uses the community node **Palatine Speech** (`n8n-nodes-palatine-speech`). Install it from the nodes panel: press **+**, search `Palatine`, open the node and click **Install node**._<br>## 1. Receive the audio<br>Trigger and fetch the file that needs transcribing. |
| Echo Text Message | n8n-nodes-base.telegram | Echoes incoming text messages back to the user | Voice or Text? | None | ## Transcribe Telegram Voice Messages with Palatine Speech<br><br>**Who's it for:** Anyone who would rather send a voice note to their Telegram bot than type.<br><br>Transcribes Telegram voice messages to text and replies with the transcript (text messages are echoed back).<br><br>### How it works<br>1. Telegram Trigger receives a new message<br>2. Switch routes voice vs. text<br>3. Download the voice file from Telegram<br>4. Palatine Speech transcribes the audio<br>5. Bot replies with the transcript<br><br>### Setup<br>1. Install `n8n-nodes-palatine-speech` in **Settings → Community Nodes**.<br>2. Add credentials: Telegram Bot API · Palatine Speech API.<br>3. Set the folder / chat / database IDs on the trigger and destination nodes.<br>4. Activate the workflow.<br><br>### Customization<br>- Reply with an AI summary instead of the raw transcript<br>- Log every transcript to Google Sheets<br>- Restrict the bot to specific chat IDs<br><br>**Requirements:** Telegram Bot API · Palatine Speech API<br><br>_Uses the community node **Palatine Speech** (`n8n-nodes-palatine-speech`). Install it from the nodes panel: press **+**, search `Palatine`, open the node and click **Install node**._<br>## 3. Deliver the result |
| Palatine Speech Transcription | n8n-nodes-palatine-speech.palatineSpeech | Transcribes the audio file into text | Download Voice File | Send Transcription Reply | ## Transcribe Telegram Voice Messages with Palatine Speech<br><br>**Who's it for:** Anyone who would rather send a voice note to their Telegram bot than type.<br><br>Transcribes Telegram voice messages to text and replies with the transcript (text messages are echoed back).<br><br>### How it works<br>1. Telegram Trigger receives a new message<br>2. Switch routes voice vs. text<br>3. Download the voice file from Telegram<br>4. Palatine Speech transcribes the audio<br>5. Bot replies with the transcript<br><br>### Setup<br>1. Install `n8n-nodes-palatine-speech` in **Settings → Community Nodes**.<br>2. Add credentials: Telegram Bot API · Palatine Speech API.<br>3. Set the folder / chat / database IDs on the trigger and destination nodes.<br>4. Activate the workflow.<br><br>### Customization<br>- Reply with an AI summary instead of the raw transcript<br>- Log every transcript to Google Sheets<br>- Restrict the bot to specific chat IDs<br><br>**Requirements:** Telegram Bot API · Palatine Speech API<br><br>_Uses the community node **Palatine Speech** (`n8n-nodes-palatine-speech`). Install it from the nodes panel: press **+**, search `Palatine`, open the node and click **Install node**._<br>## 2. Process with Palatine Speech |
| Send Transcription Reply | n8n-nodes-base.telegram | Sends the generated text transcription back to the user | Palatine Speech Transcription | None | ## Transcribe Telegram Voice Messages with Palatine Speech<br><br>**Who's it for:** Anyone who would rather send a voice note to their Telegram bot than type.<br><br>Transcribes Telegram voice messages to text and replies with the transcript (text messages are echoed back).<br><br>### How it works<br>1. Telegram Trigger receives a new message<br>2. Switch routes voice vs. text<br>3. Download the voice file from Telegram<br>4. Palatine Speech transcribes the audio<br>5. Bot replies with the transcript<br><br>### Setup<br>1. Install `n8n-nodes-palatine-speech` in **Settings → Community Nodes**.<br>2. Add credentials: Telegram Bot API · Palatine Speech API.<br>3. Set the folder / chat / database IDs on the trigger and destination nodes.<br>4. Activate the workflow.<br><br>### Customization<br>- Reply with an AI summary instead of the raw transcript<br>- Log every transcript to Google Sheets<br>- Restrict the bot to specific chat IDs<br><br>**Requirements:** Telegram Bot API · Palatine Speech API<br><br>_Uses the community node **Palatine Speech** (`n8n-nodes-palatine-speech`). Install it from the nodes panel: press **+**, search `Palatine`, open the node and click **Install node**._<br>## 3. Deliver the result |

---

### 4. Reproducing the Workflow from Scratch

1. **Install Prerequisites:** Ensure the `n8n-nodes-palatine-speech` community node is installed via **Settings → Community Nodes** (or by clicking the `+` button on the canvas, searching for `Palatine`, and installing).
2. **Configure Credentials:** 
   - Create a **Telegram Bot API** credential using your token from BotFather.
   - Create a **Palatine Speech API** credential using your API key from the Palatine Speech dashboard.
3. **Create the Telegram Trigger Node:**
   - Add a **Telegram Trigger** node.
   - Set parameters: Updates set to `message`.
   - Assign the Telegram Bot API credential.
4. **Create the Switch Node (Voice or Text?):**
   - Add a **Switch** node and connect it to the Telegram Trigger.
   - Define Output 0 (“Voice”) with a condition where `={{ $json.message.voice?.file_id }}` exists.
   - Define Output 1 (“Text”) with a condition where `={{ $json.message.text }}` exists.
5. **Set up the Voice Branch:**
   - **Download Voice File:** Add a **Telegram** node connected to Output 0 of the Switch. Set resource to `file` and configure `fileId` to `={{ $json.message.voice.file_id }}` using the Telegram credential.
   - **Palatine Speech Transcription:** Add a **Palatine Speech** node connected to the Download Voice File node. Set the model to `palatine_large_highspeed` and assign the Palatine Speech credential.
   - **Send Transcription Reply:** Add a **Telegram** node connected to the Palatine Speech node. Set text to `={{ $json.transcription }}` and `chatId` to `={{ $('Telegram Trigger').item.json.message.chat.id }}` using the Telegram credential.
6. **Set up the Text Branch:**
   - **Echo Text Message:** Add a **Telegram** node connected to Output 1 of the Switch. Set text to `={{ $json.message.text }}` and `chatId` to `={{ $('Telegram Trigger').item.json.message.chat.id }}` using the Telegram credential.
7. **Finalize:** Save the workflow and activate it.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Uses the community node **Palatine Speech** (`n8n-nodes-palatine-speech`). Install it from the nodes panel: press **+**, search `Palatine`, open the node and click **Install node**. | Required Community Node Integration |