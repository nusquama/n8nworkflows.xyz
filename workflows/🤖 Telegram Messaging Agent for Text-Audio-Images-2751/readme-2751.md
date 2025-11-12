🤖 Telegram Messaging Agent for Text/Audio/Images

https://n8nworkflows.xyz/workflows/---telegram-messaging-agent-for-text-audio-images-2751


# 🤖 Telegram Messaging Agent for Text/Audio/Images

---

### 1. Workflow Overview

This n8n workflow implements an intelligent Telegram bot designed to receive and process multiple message types—text, voice recordings, and images with captions—using AI-powered classification and response generation. It acts as a personal assistant capable of understanding user inputs, validating sender identity, and providing contextual replies.

The workflow is logically divided into the following blocks:

- **1.1 Webhook Management and Initialization**  
  Handles Telegram webhook setup, status monitoring, and reception of incoming messages via webhook endpoints.

- **1.2 User Validation**  
  Verifies the identity of the message sender against predefined user credentials to restrict access.

- **1.3 Message Routing and Classification**  
  Routes incoming messages by type (audio, text, image, or others) and classifies text content into task-related or other categories using AI.

- **1.4 Message Processing and AI Integration**  
  Processes each message type with specialized nodes: transcribes audio, analyzes images, and generates AI-driven responses using OpenAI GPT-4 models.

- **1.5 Response Dispatch**  
  Sends back appropriate Telegram messages based on classification results, including error messages for unauthorized users or unprocessable inputs.

---

### 2. Block-by-Block Analysis

#### 1.1 Webhook Management and Initialization

**Overview:**  
This block manages Telegram webhook configuration for both testing and production environments, monitors webhook status, and receives incoming Telegram updates via webhook.

**Nodes Involved:**  
- Telegram Token & Webhooks (Set node)  
- Set Webhook Test URL (HTTP Request)  
- Test Webhook Status (Telegram)  
- Set Webhook Production URL (HTTP Request)  
- Production Webhook Status (Telegram)  
- Get Telegram Webhook Info (HTTP Request)  
- Get Webhook Status (Telegram)  
- Listen for Telegram Events (Webhook)  
- Sticky Notes: Sticky Note, Sticky Note1, Sticky Note2, Sticky Note7, Sticky Note8, Sticky Note9

**Node Details:**

- **Telegram Token & Webhooks**  
  - Type: Set  
  - Role: Stores Telegram bot token and webhook URLs for test and production environments.  
  - Configuration:  
    - `token`: Telegram bot token prefixed with "bot"  
    - `test_url`: HTTPS URL for webhook testing endpoint  
    - `production_url`: HTTPS URL for production webhook endpoint  
  - Inputs: None (starting point for webhook setup)  
  - Outputs: Feeds into webhook setup HTTP requests  
  - Edge Cases: Incorrect token or URLs will cause webhook setup failures.

- **Set Webhook Test URL**  
  - Type: HTTP Request  
  - Role: Sets Telegram webhook to the test URL using Telegram Bot API `setWebhook` method.  
  - Configuration: Uses token and test_url from previous node to construct API call.  
  - Inputs: Telegram Token & Webhooks output  
  - Outputs: Triggers Test Webhook Status node  
  - Failure Modes: Network errors, invalid token, or invalid URL.

- **Test Webhook Status**  
  - Type: Telegram  
  - Role: Sends a Telegram message to a fixed chat ID to confirm webhook test status.  
  - Configuration: Sends text from HTTP response description.  
  - Inputs: Set Webhook Test URL output  
  - Outputs: None  
  - Edge Cases: Telegram API errors or invalid chat ID.

- **Set Webhook Production URL**  
  - Type: HTTP Request  
  - Role: Sets Telegram webhook to the production URL similarly to the test webhook.  
  - Inputs: Telegram Token & Webhooks output  
  - Outputs: Triggers Production Webhook Status node  
  - Failure Modes: Same as test webhook.

- **Production Webhook Status**  
  - Type: Telegram  
  - Role: Sends a Telegram message to confirm production webhook status.  
  - Inputs: Set Webhook Production URL output  
  - Outputs: None

- **Get Telegram Webhook Info**  
  - Type: HTTP Request  
  - Role: Retrieves current webhook info from Telegram API for monitoring.  
  - Inputs: Telegram Token & Webhooks output  
  - Outputs: Triggers Get Webhook Status node  
  - Failure Modes: API errors, invalid token.

- **Get Webhook Status**  
  - Type: Telegram  
  - Role: Sends the webhook info JSON as a message to a fixed chat ID for review.  
  - Inputs: Get Telegram Webhook Info output  
  - Outputs: None

- **Listen for Telegram Events**  
  - Type: Webhook  
  - Role: Receives incoming Telegram updates via HTTP POST at configured endpoint.  
  - Configuration: HTTP POST method, binary property named "data" for attachments.  
  - Inputs: External Telegram updates  
  - Outputs: Triggers Validation node  
  - Edge Cases: Network issues, malformed requests.

- **Sticky Notes**  
  Provide documentation and instructions on webhook setup, usage, and status monitoring.

---

#### 1.2 User Validation

**Overview:**  
Validates the Telegram message sender's identity by comparing the incoming message's first name, last name, and user ID against predefined authorized user data.

**Nodes Involved:**  
- Validation (Set)  
- Check User & Chat ID (If)  
- Error message (Telegram)  
- Sticky Note4

**Node Details:**

- **Validation**  
  - Type: Set  
  - Role: Defines authorized user credentials (first name, last name, user ID).  
  - Configuration: Hardcoded values for authorized user.  
  - Inputs: Output from Listen for Telegram Events  
  - Outputs: Feeds into Check User & Chat ID node.

- **Check User & Chat ID**  
  - Type: If  
  - Role: Compares incoming message sender's first name, last name, and ID with authorized credentials.  
  - Configuration: Uses expressions to compare incoming JSON fields with Validation node values.  
  - Inputs: Validation node output  
  - Outputs:  
    - True: Passes to Message Router  
    - False: Sends to Error message node  
  - Edge Cases: Missing fields, case sensitivity, or multiple users not supported.

- **Error message**  
  - Type: Telegram  
  - Role: Sends a failure message "Unable to process your message." to unauthorized users.  
  - Inputs: Check User & Chat ID false output  
  - Outputs: None

- **Sticky Note4**  
  - Provides a brief note on user validation purpose.

---

#### 1.3 Message Routing and Classification

**Overview:**  
Routes incoming messages by type (audio, text, image, or others) and classifies text content into "task" or "other" categories using AI text classifiers.

**Nodes Involved:**  
- Message Router (Switch)  
- Edit Fields (Set)  
- Text Classifier (Langchain Text Classifier)  
- Text Classifier Audio (Langchain Text Classifier)  
- Sticky Note5, Sticky Note6, Sticky Note3

**Node Details:**

- **Message Router**  
  - Type: Switch  
  - Role: Detects message type by checking existence of voice, text, or photo fields in incoming Telegram message JSON.  
  - Configuration:  
    - Routes to outputs named "audio", "text", "image", or fallback "extra".  
  - Inputs: Check User & Chat ID true output  
  - Outputs:  
    - audio → Get Audio File  
    - text → Edit Fields  
    - image → Image Schema  
    - extra → Error message (fallback)  
  - Edge Cases: Messages without recognized types go to fallback.

- **Edit Fields**  
  - Type: Set  
  - Role: Extracts and assigns the text message content to a field named "text".  
  - Inputs: Message Router "text" output  
  - Outputs: Text Classifier node

- **Text Classifier**  
  - Type: Langchain Text Classifier  
  - Role: Classifies text messages into "task" or "other" categories using AI.  
  - Configuration: Input text from "text" field; categories defined as "task" (for task/todo messages) and "other".  
  - Inputs: Edit Fields output  
  - Outputs:  
    - "task" → Text Task Message node  
    - "other" → Text Other Message node  
  - Edge Cases: Classification errors or ambiguous text.

- **Text Classifier Audio**  
  - Type: Langchain Text Classifier  
  - Role: Classifies transcribed audio text similarly into "task" or "other".  
  - Inputs: Transcribe Recording output  
  - Outputs:  
    - "task" → Audio Task Message node  
    - "other" → Audio Other Message node

- **Sticky Notes**  
  - Provide labels for processing text, audio, and image blocks.

---

#### 1.4 Message Processing and AI Integration

**Overview:**  
Processes each message type with specialized nodes: audio messages are downloaded and transcribed; images are downloaded, converted, and analyzed; text messages are classified and responded to using OpenAI GPT-4 models.

**Nodes Involved:**  
- Get Audio File (Telegram)  
- Transcribe Recording (Langchain OpenAI)  
- Audio Task Message (Telegram)  
- Audio Other Message (Telegram)  
- Image Schema (Set)  
- Get Image (Telegram)  
- Extract from File to Base64 (Extract from File)  
- Convert to Image File (Convert to File)  
- Analyze Image (Langchain OpenAI)  
- Image Message (Telegram)  
- gpt-4o-mini (Langchain LM Chat OpenAI)  
- gpt-4o-mini1 (Langchain LM Chat OpenAI)  
- Sticky Note5, Sticky Note3, Sticky Note6

**Node Details:**

- **Get Audio File**  
  - Type: Telegram  
  - Role: Downloads the voice message file using Telegram file ID.  
  - Inputs: Message Router "audio" output  
  - Outputs: Transcribe Recording node  
  - Edge Cases: File not found, network errors.

- **Transcribe Recording**  
  - Type: Langchain OpenAI (Audio Transcription)  
  - Role: Transcribes audio binary data to text using OpenAI's transcription capabilities.  
  - Inputs: Get Audio File output (binary data)  
  - Outputs: Text Classifier Audio node

- **Audio Task Message & Audio Other Message**  
  - Type: Telegram  
  - Role: Sends back task or other classification responses for audio messages.  
  - Inputs: Text Classifier Audio outputs  
  - Outputs: None  
  - Configuration: Sends formatted HTML messages with classification result.

- **Image Schema**  
  - Type: Set  
  - Role: Extracts photo file ID and caption from incoming message for image processing.  
  - Inputs: Message Router "image" output  
  - Outputs: Get Image node

- **Get Image**  
  - Type: Telegram  
  - Role: Downloads the image file using extracted file ID.  
  - Inputs: Image Schema output  
  - Outputs: Extract from File to Base64 node

- **Extract from File to Base64**  
  - Type: Extract from File  
  - Role: Converts downloaded binary image file to base64 string for AI analysis.  
  - Inputs: Get Image output  
  - Outputs: Convert to Image File node

- **Convert to Image File**  
  - Type: Convert to File  
  - Role: Converts base64 data back to binary file format with proper filename for AI analysis.  
  - Inputs: Extract from File to Base64 output  
  - Outputs: Analyze Image node

- **Analyze Image**  
  - Type: Langchain OpenAI (Image Analysis)  
  - Role: Uses GPT-4 Vision API to analyze image content and generate descriptive text.  
  - Inputs: Convert to Image File output  
  - Outputs: Image Message node

- **Image Message**  
  - Type: Telegram  
  - Role: Sends analyzed image content back to the user.  
  - Inputs: Analyze Image output  
  - Outputs: None

- **gpt-4o-mini & gpt-4o-mini1**  
  - Type: Langchain LM Chat OpenAI  
  - Role: AI language models used for text classification and processing.  
  - Inputs/Outputs: Integrated within classification nodes.

- **Sticky Notes**  
  - Label processing sections for clarity.

---

#### 1.5 Response Dispatch

**Overview:**  
Sends Telegram messages back to users based on classification results or error conditions.

**Nodes Involved:**  
- Audio Task Message  
- Audio Other Message  
- Text Task Message  
- Text Other Message  
- Image Message  
- Error message

**Node Details:**

- All Telegram nodes sending messages use the Telegram API credentials and send formatted text messages to the chat ID extracted from the incoming message.

- Messages differentiate between "task" and "other" categories, providing contextual feedback.

- The Error message node handles unauthorized users or unprocessable messages.

- Edge Cases: Telegram API rate limits, invalid chat IDs, or message formatting errors.

---

### 3. Summary Table

| Node Name                 | Node Type                          | Functional Role                              | Input Node(s)                 | Output Node(s)                           | Sticky Note                                                                                   |
|---------------------------|----------------------------------|----------------------------------------------|------------------------------|-----------------------------------------|-----------------------------------------------------------------------------------------------|
| Telegram Token & Webhooks  | Set                              | Stores bot token and webhook URLs             | None                         | Set Webhook Production URL, Set Webhook Test URL, Get Telegram Webhook Info |                                                                                               |
| Set Webhook Test URL       | HTTP Request                     | Sets Telegram webhook to test URL             | Telegram Token & Webhooks     | Test Webhook Status                      |                                                                                               |
| Test Webhook Status        | Telegram                         | Sends test webhook status message             | Set Webhook Test URL          | None                                    |                                                                                               |
| Set Webhook Production URL | HTTP Request                     | Sets Telegram webhook to production URL       | Telegram Token & Webhooks     | Production Webhook Status                |                                                                                               |
| Production Webhook Status  | Telegram                         | Sends production webhook status message       | Set Webhook Production URL    | None                                    |                                                                                               |
| Get Telegram Webhook Info  | HTTP Request                     | Retrieves webhook info from Telegram API      | Telegram Token & Webhooks     | Get Webhook Status                       |                                                                                               |
| Get Webhook Status         | Telegram                         | Sends webhook info message                     | Get Telegram Webhook Info     | None                                    |                                                                                               |
| Listen for Telegram Events | Webhook                         | Receives incoming Telegram updates            | External                     | Validation                              |                                                                                               |
| Validation                | Set                              | Defines authorized user credentials            | Listen for Telegram Events    | Check User & Chat ID                     | Sticky Note4: "## Validate Telegram User"                                                    |
| Check User & Chat ID       | If                               | Validates sender identity                       | Validation                   | Message Router (true), Error message (false) |                                                                                               |
| Error message             | Telegram                         | Sends error message to unauthorized users     | Check User & Chat ID (false), Message Router (fallback) | None                                    |                                                                                               |
| Message Router            | Switch                          | Routes messages by type (audio, text, image)  | Check User & Chat ID (true)   | Get Audio File, Edit Fields, Image Schema, Error message |                                                                                               |
| Edit Fields               | Set                              | Extracts text from message                      | Message Router (text)         | Text Classifier                         |                                                                                               |
| Text Classifier           | Langchain Text Classifier        | Classifies text messages as task or other     | Edit Fields                  | Text Task Message, Text Other Message   |                                                                                               |
| Text Classifier Audio     | Langchain Text Classifier        | Classifies transcribed audio text              | Transcribe Recording         | Audio Task Message, Audio Other Message |                                                                                               |
| Get Audio File            | Telegram                         | Downloads voice message file                    | Message Router (audio)        | Transcribe Recording                    |                                                                                               |
| Transcribe Recording      | Langchain OpenAI (Audio)         | Transcribes audio to text                       | Get Audio File               | Text Classifier Audio                   |                                                                                               |
| Audio Task Message        | Telegram                         | Sends task classification response for audio  | Text Classifier Audio (task)  | None                                    |                                                                                               |
| Audio Other Message       | Telegram                         | Sends other classification response for audio | Text Classifier Audio (other) | None                                    |                                                                                               |
| Image Schema              | Set                              | Extracts image file ID and caption              | Message Router (image)        | Get Image                              |                                                                                               |
| Get Image                 | Telegram                         | Downloads image file                            | Image Schema                 | Extract from File to Base64             |                                                                                               |
| Extract from File to Base64 | Extract from File               | Converts image binary to base64                  | Get Image                    | Convert to Image File                   |                                                                                               |
| Convert to Image File     | Convert to File                  | Converts base64 to binary file                   | Extract from File to Base64   | Analyze Image                          |                                                                                               |
| Analyze Image             | Langchain OpenAI (Image)         | Analyzes image content with GPT-4 Vision API   | Convert to Image File        | Image Message                         |                                                                                               |
| Image Message             | Telegram                         | Sends analyzed image description                 | Analyze Image                | None                                    |                                                                                               |
| Text Task Message         | Telegram                         | Sends task classification response for text    | Text Classifier (task)        | None                                    |                                                                                               |
| Text Other Message        | Telegram                         | Sends other classification response for text   | Text Classifier (other)       | None                                    |                                                                                               |
| Sticky Note               | Sticky Note                     | Documentation and labeling                      | None                         | None                                    | # Receive Telegram Message with Webhook                                                     |
| Sticky Note1              | Sticky Note                     | Documentation on webhook setup                   | None                         | None                                    | # How to set up a Telegram Bot WebHook (detailed instructions)                              |
| Sticky Note2              | Sticky Note                     | Telegram webhook tools overview                   | None                         | None                                    | # Telegram Webhook Tools - Setting your Telegram Bot WebHook the Easy Way                    |
| Sticky Note3              | Sticky Note                     | Label for image processing block                  | None                         | None                                    | # Process Image                                                                            |
| Sticky Note4              | Sticky Note                     | Label for user validation block                    | None                         | None                                    | ## Validate Telegram User                                                                 |
| Sticky Note5              | Sticky Note                     | Label for audio processing block                    | None                         | None                                    | # Process Audio                                                                           |
| Sticky Note6              | Sticky Note                     | Label for text processing block                     | None                         | None                                    | # Process Text                                                                           |
| Sticky Note7              | Sticky Note                     | Label for webhook status block                      | None                         | None                                    | ## Webhook Status                                                                        |
| Sticky Note8              | Sticky Note                     | Label for setting webhook for testing               | None                         | None                                    | ## Set Webhook for Testing                                                               |
| Sticky Note9              | Sticky Note                     | Label for setting webhook for production            | None                         | None                                    | ## Set Webhook for Production                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Set node named "Telegram Token & Webhooks"**  
   - Assign three string fields:  
     - `token`: Your Telegram bot token prefixed with "bot" (e.g., "bot123456:ABCDEF")  
     - `test_url`: HTTPS URL for your webhook test endpoint (e.g., "https://yourdomain.com/webhook-test/your-endpoint")  
     - `production_url`: HTTPS URL for your production webhook endpoint (e.g., "https://yourdomain.com/webhook/your-endpoint")

2. **Create an HTTP Request node "Set Webhook Test URL"**  
   - Method: GET  
   - URL: `https://api.telegram.org/{{ $json.token }}/setWebhook`  
   - Query Parameter: `url` = `{{ $json.test_url }}`  
   - Connect input from "Telegram Token & Webhooks"

3. **Create a Telegram node "Test Webhook Status"**  
   - Text: `={{ $json.description }} for Testing`  
   - Chat ID: Your Telegram chat ID for status messages  
   - Connect input from "Set Webhook Test URL"

4. **Create an HTTP Request node "Set Webhook Production URL"**  
   - Method: GET  
   - URL: `https://api.telegram.org/{{ $json.token }}/setWebhook`  
   - Query Parameter: `url` = `{{ $json.production_url }}`  
   - Connect input from "Telegram Token & Webhooks"

5. **Create a Telegram node "Production Webhook Status"**  
   - Text: `={{ $json.description }} for Production`  
   - Chat ID: Your Telegram chat ID  
   - Connect input from "Set Webhook Production URL"

6. **Create an HTTP Request node "Get Telegram Webhook Info"**  
   - Method: GET  
   - URL: `https://api.telegram.org/{{ $json.token }}/getWebhookInfo`  
   - Connect input from "Telegram Token & Webhooks"

7. **Create a Telegram node "Get Webhook Status"**  
   - Text: `={{ JSON.stringify($json.result, null, 2) }}`  
   - Chat ID: Your Telegram chat ID  
   - Connect input from "Get Telegram Webhook Info"

8. **Create a Webhook node "Listen for Telegram Events"**  
   - HTTP Method: POST  
   - Path: Your webhook endpoint path (e.g., "your-endpoint")  
   - Binary Property Name: "data"  
   - This node receives incoming Telegram updates.

9. **Create a Set node "Validation"**  
   - Assign fields:  
     - `first_name`: Authorized user's first name  
     - `last_name`: Authorized user's last name  
     - `id`: Authorized user's Telegram ID (number)  
   - Connect input from "Listen for Telegram Events"

10. **Create an If node "Check User & Chat ID"**  
    - Condition: All of the following must be true:  
      - Incoming message sender's first name equals `{{ $json.first_name }}` from Validation node  
      - Last name equals `{{ $json.last_name }}`  
      - ID equals `{{ $json.id }}`  
    - Connect input from "Validation"  
    - True output: Connect to "Message Router"  
    - False output: Connect to "Error message"

11. **Create a Telegram node "Error message"**  
    - Text: "Unable to process your message."  
    - Chat ID: Extract from incoming message JSON path `$json.body.message.chat.id`  
    - Connect input from "Check User & Chat ID" false output and "Message Router" fallback output

12. **Create a Switch node "Message Router"**  
    - Rules:  
      - Output "audio": if `body.message.voice` exists  
      - Output "text": if `body.message.text` exists  
      - Output "image": if `body.message.photo` exists  
      - Fallback output: "extra"  
    - Connect input from "Check User & Chat ID" true output

13. **Create a Telegram node "Get Audio File"**  
    - Resource: File  
    - File ID: `={{ $json.body.message.voice.file_id }}`  
    - Connect input from "Message Router" audio output

14. **Create a Langchain OpenAI node "Transcribe Recording"**  
    - Resource: Audio  
    - Operation: Transcribe  
    - Binary Property Name: "data"  
    - Connect input from "Get Audio File"

15. **Create a Langchain Text Classifier node "Text Classifier Audio"**  
    - Input Text: `={{ $json.text }}`  
    - Categories:  
      - "task": For task/todo messages  
      - "other": For other messages  
    - Connect input from "Transcribe Recording"

16. **Create two Telegram nodes "Audio Task Message" and "Audio Other Message"**  
    - Text:  
      - Task: `Task message: <i>{{ $json.text }}</i>`  
      - Other: `Other message: <i>{{ $json.text }}</i>`  
    - Chat ID: Extract from incoming message JSON path `$json.body.message.chat.id`  
    - Connect "Audio Task Message" to "Text Classifier Audio" task output  
    - Connect "Audio Other Message" to "Text Classifier Audio" other output

17. **Create a Set node "Edit Fields"**  
    - Assign field "text" = `={{ $json.body.message.text }}`  
    - Connect input from "Message Router" text output

18. **Create a Langchain Text Classifier node "Text Classifier"**  
    - Input Text: `={{ $json.text }}`  
    - Categories same as audio classifier  
    - Connect input from "Edit Fields"

19. **Create two Telegram nodes "Text Task Message" and "Text Other Message"**  
    - Text:  
      - Task: `Task message: <i>{{ $json.text }}</i>`  
      - Other: `Other message: <i>{{ $json.text }}</i>`  
    - Chat ID: Extract from incoming message JSON path `$json.body.message.chat.id`  
    - Connect "Text Task Message" to "Text Classifier" task output  
    - Connect "Text Other Message" to "Text Classifier" other output

20. **Create a Set node "Image Schema"**  
    - Assign fields:  
      - `image_file_id` = `={{ $json.body.message.photo.last().file_id }}`  
      - `caption` = `={{ $json.body.message.caption }}`  
    - Connect input from "Message Router" image output

21. **Create a Telegram node "Get Image"**  
    - Resource: File  
    - File ID: `={{ $json.image_file_id }}`  
    - Connect input from "Image Schema"

22. **Create an Extract from File node "Extract from File to Base64"**  
    - Operation: binaryToProperty  
    - Connect input from "Get Image"

23. **Create a Convert to File node "Convert to Image File"**  
    - Operation: toBinary  
    - File Name: `={{ $json.result.file_path }}`  
    - Source Property: "data"  
    - Connect input from "Extract from File to Base64"

24. **Create a Langchain OpenAI node "Analyze Image"**  
    - Model: GPT-4o-mini (or GPT-4 Vision API)  
    - Resource: Image  
    - Operation: Analyze  
    - Input Type: base64  
    - Connect input from "Convert to Image File"

25. **Create a Telegram node "Image Message"**  
    - Text: `={{ $json.content }}` (analysis result)  
    - Chat ID: Extract from incoming message JSON path `$json.body.message.chat.id`  
    - Connect input from "Analyze Image"

26. **Create Telegram credentials**  
    - Configure Telegram API credentials with your bot token.

27. **Create OpenAI credentials**  
    - Configure OpenAI API credentials with your API key.

28. **Test the workflow**  
    - Deploy webhook URLs to Telegram BotFather  
    - Send test messages of each type (text, voice, image) from authorized user  
    - Verify correct classification and responses.

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                                                                  |
|------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| How to set up a Telegram Bot WebHook: detailed explanation with example URLs and verification commands                             | See Sticky Note1 content in the workflow                                                       |
| Telegram Webhook Tools: Simplified webhook setup instructions                                                                       | See Sticky Note2 content                                                                        |
| Workflow uses OpenAI GPT-4 and GPT-4 Vision API for advanced AI processing                                                         | Requires valid OpenAI API credentials                                                           |
| User validation is strict: only one authorized user defined by first name, last name, and Telegram user ID                         | Modify Validation node to add more users or relax conditions if needed                          |
| Error handling includes fallback for unrecognized message types and unauthorized users                                              | Error message node sends feedback to users                                                     |
| Workflow supports both testing and production webhook endpoints for development flexibility                                        | Use Telegram Token & Webhooks node to configure URLs                                           |

---

This document provides a complete, structured reference for the "🤖 Telegram Messaging Agent for Text/Audio/Images" workflow, enabling advanced users and AI agents to understand, reproduce, and modify the workflow confidently.