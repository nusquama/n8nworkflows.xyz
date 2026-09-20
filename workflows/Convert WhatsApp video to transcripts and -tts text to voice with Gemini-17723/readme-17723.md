Convert WhatsApp video to transcripts and /tts text to voice with Gemini

https://n8nworkflows.xyz/workflows/convert-whatsapp-video-to-transcripts-and--tts-text-to-voice-with-gemini-17723


# Convert WhatsApp video to transcripts and /tts text to voice with Gemini

### 1. Workflow Overview

This workflow automates audio-to-text transcription and text-to-speech generation via WhatsApp. It processes incoming WhatsApp messages to handle two distinct capabilities: converting incoming videos into text transcriptions using Google Gemini (via FreeConvert for format translation) and converting text commands starting with `/tts` into voice notes using Fish Audio.

The logic is grouped into the following functional blocks:
- **1.1 Input Reception & Normalization:** Listens for incoming WhatsApp events and normalizes message payloads (phone, media ID, text body, and message type).
- **1.2 Routing & Text-to-Speech (TTS):** Evaluates incoming message content to route `/tts` commands to a sub-workflow that generates and returns a voice note.
- **1.3 Video Conversion Pipeline:** Handles incoming video messages by downloading the media, initializing a FreeConvert job, uploading the file, and polling until conversion to MP3 completes.
- **1.4 Transcription & Response:** Adjusts audio metadata, transcribes the MP3 file using Google Gemini, and sends the resulting text back to the user on WhatsApp.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Normalization
This block captures incoming webhook events from WhatsApp, extracts essential fields, and prepares the data payload for routing.
- **Nodes Involved:** `When WhatsApp Message Received`, `Set Message Fields`
- **Node Details:**
  - **When WhatsApp Message Received**
    - *Type and Technical Role:* `n8n-nodes-base.whatsAppTrigger` (Webhook trigger)
    - *Configuration Choices:* Listens for incoming WhatsApp messages (`updates: ["messages"]`) and status updates.
    - *Key Expressions:* None (Webhook payload receiver).
    - *Input/Output Connections:* Output connects to `Set Message Fields`.
    - *Version-specific Requirements:* Requires valid WhatsApp Cloud API webhook configuration.
    - *Edge Cases / Potential Failure Types:* Webhook verification failures or invalid signature tokens.
  - **Set Message Fields**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data transformation)
    - *Configuration Choices:* Extracts sender phone number, message type combined with document MIME type, media URL/ID, and body text.
    - *Key Expressions:* 
      - `phone`: `={{ $json.messages[0].from }}`
      - `type`: `={{ $json.messages[0].type }} {{ $json.messages[0].document.mime_type }}`
      - `media_id`: `={{ $json.messages[0].document.url }}`
      - `text_body`: `={{ $json.messages[0].text?.body || '' }}`
    - *Input/Output Connections:* Input from `When WhatsApp Message Received`; output connects to `If Message Type Video`.
    - *Edge Cases / Potential Failure Types:* Missing array indices if payload structure changes unexpectedly.

#### 2.2 Routing & Text-to-Speech (TTS)
This block checks whether a message is a video or a text command, executing a sub-workflow to generate voice notes when a `/tts` prefix is detected.
- **Nodes Involved:** `If Message Type Video`, `If Text Conversion Requested`, `Execute Voice Note Workflow`, `When Called by Main Workflow`, `Generate Speech with FishAudio`, `Upload media`, `Send Voice Note Confirmation`
- **Node Details:**
  - **If Message Type Video**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Conditional routing)
    - *Configuration Choices:* Evaluates whether the message type contains `video`.
    - *Key Expressions:* Left value `={{ $json.type }}` contains `video`.
    - *Input/Output Connections:* Input from `Set Message Fields`. True branch goes to `Fetch Incoming Video`; false branch goes to `If Text Conversion Requested`.
  - **If Text Conversion Requested**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Conditional routing)
    - *Configuration Choices:* Checks if `text_body` contains `/tts`.
    - *Key Expressions:* Left value `={{ $json.text_body }}` contains `/tts`.
    - *Input/Output Connections:* Input from `If Message Type Video` (false branch); true branch goes to `Execute Voice Note Workflow`.
  - **Execute Voice Note Workflow**
    - *Type and Technical Role:* `n8n-nodes-base.executeWorkflow` (Sub-workflow execution)
    - *Configuration Choices:* Executes the current workflow (`UfGSlAVdkgxtNcTm`) as a sub-workflow, passing cleaned text and phone parameters.
    - *Key Expressions:* 
      - `phone`: `={{ $json.phone }}`
      - `text_body`: `={{ $json.text_body.replace('/tts ', '') }}`
    - *Input/Output Connections:* Input from `If Text Conversion Requested`.
  - **When Called by Main Workflow**
    - *Type and Technical Role:* `n8n-nodes-base.executeWorkflowTrigger` (Sub-workflow trigger)
    - *Configuration Choices:* Defines inputs `phone` and `text_body` received from the parent workflow invocation.
    - *Input/Output Connections:* Output connects to `Generate Speech with FishAudio`.
  - **Generate Speech with FishAudio**
    - *Type and Technical Role:* `n8n-nodes-fishaudio.fishaudio` (AI / Audio generation)
    - *Configuration Choices:* Uses model `s2.1-pro-free`, voice ID `92b9f644244f4571ba58d696a6ef5e67`, and speed `1`.
    - *Key Expressions:* Text: `={{ $('When Called by Main Workflow').item.json.text_body }}`.
    - *Input/Output Connections:* Input from sub-workflow trigger; output connects to `Upload media`.
    - *Credentials:* Fish Audio API.
    - *Edge Cases / Potential Failure Types:* API rate limits, invalid voice IDs, or text formatting errors.
  - **Upload media**
    - *Type and Technical Role:* `n8n-nodes-base.whatsApp` (API integration)
    - *Configuration Choices:* Uploads generated media using resource `media` and specified phone number ID.
    - *Input/Output Connections:* Input from Fish Audio; output connects to `Send Voice Note Confirmation`.
  - **Send Voice Note Confirmation**
    - *Type and Technical Role:* `n8n-nodes-base.whatsApp` (API integration)
    - *Configuration Choices:* Sends message type `audio` using media ID reference (`useMediaId`).
    - *Key Expressions:* 
      - `mediaId`: `={{ $json.id }}`
      - `recipientPhoneNumber`: `={{ $('When Called by Main Workflow').item.json.phone }}`
    - *Input/Output Connections:* Input from `Upload media`.

#### 2.3 Video Conversion Pipeline
Downloads incoming video binary data from WhatsApp, initializes a conversion job with FreeConvert, uploads the payload, and polls until processing completes.
- **Nodes Involved:** `Fetch Incoming Video`, `Post Conversion Job`, `Prepare File Upload`, `Upload File for Conversion`, `Wait for Conversion Status`, `Fetch Conversion Job Status`, `If Conversion Complete`
- **Node Details:**
  - **Fetch Incoming Video**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (API integration / file download)
    - *Configuration Choices:* HTTP GET request with predefined WhatsApp credential, downloading the file payload.
    - *Key Expressions:* URL: `={{ $json.media_id }}`. Response format set to `file`.
    - *Input/Output Connections:* Input from `If Message Type Video` (true branch); output connects to `Post Conversion Job`.
  - **Post Conversion Job**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (API integration)
    - *Configuration Choices:* POST request to FreeConvert API (`https://api.freeconvert.com/v1/process/jobs`) specifying tasks to import, convert to `mp3`, and export via URL.
    - *Key Expressions:* JSON body defines tasks (`import-1`, `convert-1`, `export-1`). Headers include Authorization Bearer token.
    - *Input/Output Connections:* Input from `Fetch Incoming Video`; output connects to `Prepare File Upload`.
  - **Prepare File Upload**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration Choices:* Validates the FreeConvert job response tasks and re-attaches the original binary video data from `Fetch Incoming Video`.
    - *Key Expressions:* References `$('Fetch Incoming Video').item.binary`. Throws descriptive errors if missing tasks or binary data.
    - *Input/Output Connections:* Input from `Post Conversion Job`; output connects to `Upload File for Conversion`.
  - **Upload File for Conversion**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (API integration / file upload)
    - *Configuration Choices:* POST request using `multipart/form-data` to the dynamic form upload URL returned by FreeConvert.
    - *Key Expressions:* 
      - URL: `={{ $json.tasks.find(t => t.name === 'import-1').result.form.url }}`
      - Signature: `={{ $json.tasks.find(t => t.name === 'import-1').result.form.parameters.signature }}`
    - *Input/Output Connections:* Input from `Prepare File Upload`; output connects to `Wait for Conversion Status`.
  - **Wait for Conversion Status**
    - *Type and Technical Role:* `n8n-nodes-base.wait` (Flow control)
    - *Configuration Choices:* Pauses execution momentarily before polling status.
    - *Input/Output Connections:* Input from `Upload File for Conversion` or `If Conversion Complete` (false branch); output connects to `Fetch Conversion Job Status`.
  - **Fetch Conversion Job Status**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (API integration)
    - *Configuration Choices:* GET request to check the status of the FreeConvert job ID.
    - *Key Expressions:* URL: `=https://api.freeconvert.com/v1/process/jobs/{{ $('Post Conversion Job').item.json.id }}`.
    - *Input/Output Connections:* Input from `Wait for Conversion Status`; output connects to `If Conversion Complete`.
  - **If Conversion Complete**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Conditional routing)
    - *Configuration Choices:* Evaluates whether the job status equals `completed`.
    - *Key Expressions:* Left value `={{ $json.status }}` equals `completed`.
    - *Input/Output Connections:* Input from `Fetch Conversion Job Status`. True branch goes to `Fetch Converted MP3`; false branch loops back to `Wait for Conversion Status`.

#### 2.4 Transcription & Response
Downloads the resulting MP3 file, enforces the correct audio MIME type, transcribes the speech via Google Gemini, and sends the text transcription back to the user.
- **Nodes Involved:** `Fetch Converted MP3`, `Process Audio Metadata`, `Transcribe Audio with Gemini`, `Send Transcription to WhatsApp`
- **Node Details:**
  - **Fetch Converted MP3**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (API integration / file download)
    - *Configuration Choices:* Downloads the converted file from the FreeConvert export URL.
    - *Key Expressions:* URL: `={{ $json.tasks.find(t => t.name === 'export-1').result.url }}`.
    - *Input/Output Connections:* Input from `If Conversion Complete` (true branch); output connects to `Process Audio Metadata`.
  - **Process Audio Metadata**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration Choices:* Iterates through incoming items and explicitly sets the MIME type of binary data to `audio/mpeg`.
    - *Input/Output Connections:* Input from `Fetch Converted MP3`; output connects to `Transcribe Audio with Gemini`.
  - **Transcribe Audio with Gemini**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.googleGemini` (AI integration)
    - *Configuration Choices:* Uses model `models/gemini-3.1-flash-lite`, resource `audio`, input type `binary`.
    - *Input/Output Connections:* Input from `Process Audio Metadata`; output connects to `Send Transcription to WhatsApp`.
    - *Credentials:* Google Palm / Gemini API.
    - *Edge Cases / Potential Failure Types:* Unsupported audio duration limits or rate limiting.
  - **Send Transcription to WhatsApp**
    - *Type and Technical Role:* `n8n-nodes-base.whatsApp` (API integration)
    - *Configuration Choices:* Sends a WhatsApp text message containing the AI transcription result to the originating contact.
    - *Key Expressions:* 
      - `textBody`: `={{ $json.candidates[0].content.parts[0].text }}`
      - `recipientPhoneNumber`: `={{ $('When WhatsApp Message Received').item.json.contacts[0].wa_id }}`
    - *Input/Output Connections:* Input from `Transcribe Audio with Gemini`.
    - *Credentials:* WhatsApp Cloud API.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | n8n-nodes-base.stickyNote | Overview documentation | None | None | STT & TTS in whatsapp<br><br>### How v works...<br><br>@[youtube](GFD8sBwHheg) |
| Sticky Note1 | n8n-nodes-base.stickyNote | Receive and extract documentation | None | None | ## Receive and extract WhatsApp message<br><br>Receives incoming messages from WhatsApp and extracts message parameters like phone, type, media ID, and body text. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Determine media type documentation | None | None | ## Determine media type route<br><br>Evaluates whether the incoming WhatsApp media is a video file or text/audio message. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Download video documentation | None | None | ## Download video and create job<br><br>Downloads the video file from WhatsApp and initializes a conversion job with FreeConvert. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Prepare upload documentation | None | None | ## Prepare and upload video file<br><br>Prepares binary data and uploads the video file to the conversion service. |
| Sticky Note5 | n8n-nodes-base.stickyNote | Monitor job status documentation | None | None | ## Monitor conversion job status<br><br>Waits and repeatedly checks the conversion job status until processing is complete. |
| Sticky Note6 | n8n-nodes-base.stickyNote | Download MP3 documentation | None | None | ## Download MP3 and process data<br><br>Downloads the converted MP3 audio and prepares it for transcription. |
| Sticky Note7 | n8n-nodes-base.stickyNote | Transcribe and send documentation | None | None | ## Transcribe audio and send response<br><br>Transcribes the audio recording using Google Gemini and sends the result back to the user via WhatsApp. |
| Sticky Note8 | n8n-nodes-base.stickyNote | Generate voice note documentation | None | None | ## Generate voice note from text<br><br>Invokes a sub-workflow to generate voice notes from text messages. |
| Sticky Note9 | n8n-nodes-base.stickyNote | Generate speech documentation | None | None | ## Generate speech and send media<br><br>Handles sub-workflow execution triggered by the main flow to convert text into speech and upload media back to WhatsApp. |
| When WhatsApp Message Received | n8n-nodes-base.whatsAppTrigger | Trigger on incoming WhatsApp message | None | Set Message Fields | |
| Set Message Fields | n8n-nodes-base.set | Extract message attributes | When WhatsApp Message Received | If Message Type Video | |
| Execute Voice Note Workflow | n8n-nodes-base.executeWorkflow | Trigger text-to-speech sub-workflow | If Text Conversion Requested | None | |
| When Called by Main Workflow | n8n-nodes-base.executeWorkflowTrigger | Sub-workflow entry point | None | Generate Speech with FishAudio | |
| Post Conversion Job | n8n-nodes-base.httpRequest | Initialize FreeConvert job | Fetch Incoming Video | Prepare File Upload | |
| Upload File for Conversion | n8n-nodes-base.httpRequest | Upload video to FreeConvert | Prepare File Upload | Wait for Conversion Status | |
| Wait for Conversion Status | n8n-nodes-base.wait | Delay before status check | Upload File for Conversion, If Conversion Complete | Fetch Conversion Job Status | |
| Fetch Conversion Job Status | n8n-nodes-base.httpRequest | Check FreeConvert job status | Wait for Conversion Status | If Conversion Complete | |
| If Conversion Complete | n8n-nodes-base.if | Check if conversion finished | Fetch Conversion Job Status | Fetch Converted MP3, Wait for Conversion Status | |
| Fetch Converted MP3 | n8n-nodes-base.httpRequest | Download converted MP3 file | If Conversion Complete | Process Audio Metadata | |
| Generate Speech with FishAudio | n8n-nodes-fishaudio.fishaudio | Generate audio from text | When Called by Main Workflow | Upload media | |
| Upload media | n8n-nodes-base.whatsApp | Upload generated media to WhatsApp | Generate Speech with FishAudio | Send Voice Note Confirmation | |
| Send Voice Note Confirmation | n8n-nodes-base.whatsApp | Send audio message to user | Upload media | None | |
| If Text Conversion Requested | n8n-nodes-base.if | Route text to TTS sub-workflow | If Message Type Video | Execute Voice Note Workflow | |
| Prepare File Upload | n8n-nodes-base.code | Attach binary data to API response | Post Conversion Job | Upload File for Conversion | |
| Transcribe Audio with Gemini | @n8n/n8n-nodes-langchain.googleGemini | Transcribe audio to text | Process Audio Metadata | Send Transcription to WhatsApp | |
| Process Audio Metadata | n8n-nodes-base.code | Enforce audio/mpeg MIME type | Fetch Converted MP3 | Transcribe Audio with Gemini | |
| Send Transcription to WhatsApp | n8n-nodes-base.whatsApp | Send transcription text to user | Transcribe Audio with Gemini | None | |
| If Message Type Video | n8n-nodes-base.if | Branch based on video media type | Set Message Fields | Fetch Incoming Video, If Text Conversion Requested | |
| Fetch Incoming Video | n8n-nodes-base.httpRequest | Download video from WhatsApp | If Message Type Video | Post Conversion Job | |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Entry Point:**
   - Add a **When WhatsApp Message Received** trigger node (`n8n-nodes-base.whatsAppTrigger`). Configure credentials for WhatsApp Cloud API.
   - Add a **Set** node named `Set Message Fields` (`n8n-nodes-base.set`). Map string assignments: `phone` (`={{ $json.messages[0].from }}`), `type` (`={{ $json.messages[0].type }} {{ $json.messages[0].document.mime_type }}`), `media_id` (`={{ $json.messages[0].document.url }}`), and `text_body` (`={{ $json.messages[0].text?.body || '' }}`).
   - Connect **When WhatsApp Message Received** to **Set Message Fields**.

2. **Set Up Routing & Sub-Workflow Logic:**
   - Add an **If** node named `If Message Type Video` (`n8n-nodes-base.if`). Set condition to check if `type` contains `video`. Connect **Set Message Fields** to this node.
   - Add an **If** node named `If Text Conversion Requested` (`n8n-nodes-base.if`). Set condition to check if `text_body` contains `/tts`. Connect the *false* output of `If Message Type Video` to this node.
   - Add an **Execute Workflow** node named `Execute Voice Note Workflow` (`n8n-nodes-base.executeWorkflow`). Set mode to `each`, select the current workflow ID, and map inputs `phone` (`={{ $json.phone }}`) and `text_body` (`={{ $json.text_body.replace('/tts ', '') }}`). Connect the *true* output of `If Text Conversion Requested` to this node.
   - Add an **Execute Workflow Trigger** node named `When Called by Main Workflow` (`n8n-nodes-base.executeWorkflowTrigger`). Define workflow inputs `phone` and `text_body`.
   - Add a **Fish Audio** node named `Generate Speech with FishAudio` (`n8n-nodes-fishaudio.fishaudio`). Configure Fish Audio credentials, model `s2.1-pro-free`, voice ID (`92b9f644244f4571ba58d696a6ef5e67`), speed `1`, and set text expression to `={{ $('When Called by Main Workflow').item.json.text_body }}`. Connect **When Called by Main Workflow** to this node.
   - Add a WhatsApp node named `Upload media` (`n8n-nodes-base.whatsApp`). Set resource to `media`, specify your phone number ID, and connect **Generate Speech with FishAudio** to it.
   - Add a WhatsApp node named `Send Voice Note Confirmation` (`n8n-nodes-base.whatsApp`). Set operation to `send`, message type to `audio`, `mediaId` to `={{ $json.id }}`, and recipient phone number to `={{ $('When Called by Main Workflow').item.json.phone }}`. Connect **Upload media** to this node.

3. **Build the Video Conversion Pipeline:**
   - Add an HTTP Request node named `Fetch Incoming Video` (`n8n-nodes-base.httpRequest`). Set method to `GET`, URL to `={{ $json.media_id }}`, response format to `file`, and use predefined WhatsApp credentials. Connect the *true* output of `If Message Type Video` to this node.
   - Add an HTTP Request node named `Post Conversion Job` (`n8n-nodes-base.httpRequest`). Set method to `POST`, URL to `https://api.freeconvert.com/v1/process/jobs`, body to JSON containing tasks (`import-1`, `convert-1` to mp3, `export-1`), and add authorization headers (`Bearer YOUR_TOKEN_HERE`). Connect **Fetch Incoming Video** to this node.
   - Add a **Code** node named `Prepare File Upload` (`n8n-nodes-base.code`). Insert JavaScript to validate job tasks and pull binary data from `Fetch Incoming Video`. Connect **Post Conversion Job** to this node.
   - Add an HTTP Request node named `Upload File for Conversion` (`n8n-nodes-base.httpRequest`). Set method to `POST`, content type to `multipart/form-data`, URL to the import form URL expression, and add form parameters for signature and binary file data. Connect **Prepare File Upload** to this node.
   - Add a **Wait** node named `Wait for Conversion Status` (`n8n-nodes-base.wait`). Connect **Upload File for Conversion** (and the *false* path of `If Conversion Complete`) to this node.
   - Add an HTTP Request node named `Fetch Conversion Job Status` (`n8n-nodes-base.httpRequest`). Set method to `GET` with URL `=https://api.freeconvert.com/v1/process/jobs/{{ $('Post Conversion Job').item.json.id }}` and authorization headers. Connect **Wait for Conversion Status** to this node.
   - Add an **If** node named `If Conversion Complete` (`n8n-nodes-base.if`). Set condition to check if `{{ $json.status }}` equals `completed`. Connect **Fetch Conversion Job Status** to this node. Connect the *false* branch back to **Wait for Conversion Status**.

4. **Build the Transcription & Response Pipeline:**
   - Add an HTTP Request node named `Fetch Converted MP3` (`n8n-nodes-base.httpRequest`). Set method to `GET`, URL to the export task result URL, and response format to file. Connect the *true* branch of `If Conversion Complete` to this node.
   - Add a **Code** node named `Process Audio Metadata` (`n8n-nodes-base.code`). Insert JavaScript to assign `audio/mpeg` to `item.binary.data.mimeType`. Connect **Fetch Converted MP3** to this node.
   - Add a Google Gemini node named `Transcribe Audio with Gemini` (`@n8n/n8n-nodes-langchain.googleGemini`). Configure Google Palm/Gemini credentials, resource `audio`, input type `binary`, and model `models/gemini-3.1-flash-lite`. Connect **Process Audio Metadata** to this node.
   - Add a WhatsApp node named `Send Transcription to WhatsApp` (`n8n-nodes-base.whatsApp`). Set operation to `send`, `textBody` to `={{ $json.candidates[0].content.parts[0].text }}`, and recipient phone number to `={{ $('When WhatsApp Message Received').item.json.contacts[0].wa_id }}`. Connect **Transcribe Audio with Gemini** to this node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Video walkthrough reference for the STT & TTS WhatsApp integration workflow | [YouTube Video Guide](https://www.youtube.com/watch?v=GFD8sBwHheg) |