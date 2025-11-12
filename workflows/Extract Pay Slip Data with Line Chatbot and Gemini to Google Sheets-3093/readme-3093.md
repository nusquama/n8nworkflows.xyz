Extract Pay Slip Data with Line Chatbot and Gemini to Google Sheets

https://n8nworkflows.xyz/workflows/extract-pay-slip-data-with-line-chatbot-and-gemini-to-google-sheets-3093


# Extract Pay Slip Data with Line Chatbot and Gemini to Google Sheets

### 1. Workflow Overview

This workflow is designed to automate the extraction of key data from pay slip images or text messages sent by users via the Line Messaging API chatbot. It leverages Google Gemini 2.0 Flash experimental AI models to analyze images or text without requiring custom coding or complex condition handling. Extracted data fields (Status, From, To, Date, Amount) are then appended to a Google Sheets document for record-keeping and verification.

The workflow logically divides into the following blocks:

- **1.1 Input Reception and Classification:** Receives incoming messages from Line, extracts message metadata, and classifies the message type (text, image, or sticker).
- **1.2 Image Retrieval and AI Processing:** For image messages, retrieves the image content and uses Google Gemini AI to extract structured pay slip data.
- **1.3 Text Message AI Processing:** For text messages, processes the text through Google Gemini AI as an assistant.
- **1.4 Response to User:** Sends back AI-generated responses to the user via Line Messaging API.
- **1.5 Data Insertion into Google Sheets:** Inserts extracted pay slip data into a predefined Google Sheets document.
- **1.6 Memory Management:** Maintains conversational context for text messages using a window buffer memory node.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Classification

**Overview:**  
This block receives incoming webhook requests from the Line Messaging API, extracts relevant message details, and classifies the message type to route processing accordingly.

**Nodes Involved:**  
- Line: Messaging API (Webhook)  
- Message Type (Set)  
- Message Classification (Switch)  
- Sticky Note4 (Documentation)

**Node Details:**

- **Line: Messaging API**  
  - Type: Webhook  
  - Role: Entry point for incoming Line messages via POST requests.  
  - Configuration: Path set to a unique webhook ID; listens for POST HTTP method.  
  - Inputs: External HTTP POST from Line platform.  
  - Outputs: JSON payload containing message event data.  
  - Edge Cases: Webhook misconfiguration, invalid payloads, or unauthorized requests may cause failures.

- **Message Type**  
  - Type: Set  
  - Role: Extracts and assigns key message properties (text, message ID, user ID, message type) from the webhook JSON for easier access downstream.  
  - Configuration: Assigns values from JSON paths such as `body.events[0].message.text` and `body.events[0].message.type`.  
  - Inputs: Output from Line: Messaging API.  
  - Outputs: Enriched JSON with extracted fields.  
  - Edge Cases: Missing fields in incoming payload may cause undefined values.

- **Message Classification**  
  - Type: Switch  
  - Role: Routes workflow based on message type: "text", "image", or specific sticker ID (150).  
  - Configuration: Conditions check the message type field from the Line webhook JSON.  
  - Inputs: Output from Message Type node.  
  - Outputs: Three branches: text processing, image processing, and a third unused branch for sticker ID 150.  
  - Edge Cases: Unexpected message types or missing fields may cause routing errors.

- **Sticky Note4**  
  - Type: Sticky Note  
  - Content: Explains that the workflow supports both text and image messages from users, enabling a single AI assistant chatbot.

---

#### 2.2 Image Retrieval and AI Processing

**Overview:**  
For image messages, this block retrieves the image content from Line, sends it to Google Gemini AI for analysis, and extracts structured pay slip data.

**Nodes Involved:**  
- Line: Get Image (HTTP Request)  
- Google Gemini for Image (Google Gemini AI Node)  
- Image Message Processing (LangChain Chain LLM)  
- Line: Response to User (HTTP Request)  
- Text from Slip Result (Google Sheets)  
- Sticky Note (Prompt for Gemini)  
- Sticky Note3 (Google Sheets insertion explanation)

**Node Details:**

- **Line: Get Image**  
  - Type: HTTP Request  
  - Role: Downloads the image content from Line's message content API using the message ID.  
  - Configuration: URL dynamically constructed using the message ID from the webhook JSON; uses HTTP Header Authentication with Line credentials.  
  - Inputs: Triggered after message classification identifies an image message.  
  - Outputs: Binary image data for AI processing.  
  - Edge Cases: Network errors, expired tokens, or invalid message IDs may cause failures.

- **Google Gemini for Image**  
  - Type: LangChain Google Gemini AI Chat Node  
  - Role: Runs the Gemini 2.0 Flash experimental model to analyze the image and extract pay slip data.  
  - Configuration: Model set to "models/gemini-2.0-flash-exp"; uses Google Palm API credentials.  
  - Inputs: Receives image binary from Line: Get Image node.  
  - Outputs: AI-generated JSON text containing pay slip fields.  
  - Edge Cases: API quota limits, authentication errors, or malformed images may cause failures.

- **Image Message Processing**  
  - Type: LangChain Chain LLM  
  - Role: Defines the prompt for the AI to analyze the image and return a JSON response with specific fields (Status, From, To, Date, Amount).  
  - Configuration: Prompt explicitly instructs AI to extract only the listed fields; uses HumanMessagePromptTemplate for image binary input.  
  - Inputs: Image binary from Line: Get Image; AI model from Google Gemini for Image node.  
  - Outputs: Text output with JSON-formatted extracted data.  
  - Edge Cases: AI misinterpretation or unexpected image formats.

- **Line: Response to User**  
  - Type: HTTP Request  
  - Role: Sends the AI-extracted JSON response back to the user via Line Messaging API reply endpoint.  
  - Configuration: POST request with JSON body containing replyToken and text message; uses Line HTTP Header Authentication.  
  - Inputs: AI JSON text from Image Message Processing.  
  - Outputs: Confirmation of message sent.  
  - Edge Cases: API rate limits, invalid reply tokens.

- **Text from Slip Result**  
  - Type: Google Sheets  
  - Role: Appends extracted pay slip data into a Google Sheets document with columns: Status, From, To, Date, Amount.  
  - Configuration: Document ID and sheet name set to a specific Google Sheets URL; mapping uses parsed JSON fields from AI output.  
  - Inputs: AI JSON text from Image Message Processing node.  
  - Outputs: Confirmation of row appended.  
  - Edge Cases: Google Sheets API quota, invalid document ID, or schema mismatch.

- **Sticky Note**  
  - Content: Provides the Gemini prompt used for image analysis: "Analyze image and then return in JSON Response that has the only following value: Status, From, To, Date, Amount."

- **Sticky Note3**  
  - Content: Explains the insertion of extracted pay slip data into Google Sheets matching the AI prompt fields.

---

#### 2.3 Text Message AI Processing

**Overview:**  
Processes text messages sent by users through the Gemini AI assistant model and replies with AI-generated responses.

**Nodes Involved:**  
- Google Gemini for Text (Google Gemini AI Node)  
- Window Buffer Memory (LangChain Memory Buffer)  
- Text Message Processing (LangChain Agent)  
- Line: Text Response to User (HTTP Request)  
- Sticky Note1 (Gemini AI Assistant explanation)  
- Sticky Note2 (Reply to User explanation)

**Node Details:**

- **Google Gemini for Text**  
  - Type: LangChain Google Gemini AI Chat Node  
  - Role: Uses Gemini 2.0 Flash experimental model to process user text messages as an AI assistant.  
  - Configuration: Model set to "models/gemini-2.0-flash-exp"; uses Google Palm API credentials.  
  - Inputs: Text from Message Classification node routed to text branch.  
  - Outputs: AI-generated text response.  
  - Edge Cases: API limits, authentication errors.

- **Window Buffer Memory**  
  - Type: LangChain Memory Buffer Window  
  - Role: Maintains conversational context per user session using userId as session key.  
  - Configuration: Session key set to userId extracted from webhook JSON; sessionIdType is customKey.  
  - Inputs: Text messages from Message Classification node.  
  - Outputs: Contextual memory for AI agent.  
  - Edge Cases: Memory overflow or session key errors.

- **Text Message Processing**  
  - Type: LangChain Agent  
  - Role: Conversational AI agent that processes user text input with memory context and generates responses.  
  - Configuration: Uses conversationalAgent agent type; prompt includes user message text dynamically.  
  - Inputs: Text message and memory from Window Buffer Memory; AI model from Google Gemini for Text.  
  - Outputs: AI response text.  
  - Edge Cases: Prompt formatting errors, memory sync issues.

- **Line: Text Response to User**  
  - Type: HTTP Request  
  - Role: Sends AI-generated text response back to the user via Line Messaging API reply endpoint.  
  - Configuration: POST request with JSON body containing replyToken and AI output text; uses Line HTTP Header Authentication.  
  - Inputs: AI response text from Text Message Processing.  
  - Outputs: Confirmation of message sent.  
  - Edge Cases: API limits, invalid reply tokens.

- **Sticky Note1**  
  - Content: Describes Gemini AI Assistant capabilities with memory, reasoning, and planning.

- **Sticky Note2**  
  - Content: Explains that the workflow replies to users without coding or OCR processing.

---

### 3. Summary Table

| Node Name               | Node Type                            | Functional Role                          | Input Node(s)              | Output Node(s)                   | Sticky Note                                                                                     |
|-------------------------|------------------------------------|----------------------------------------|----------------------------|---------------------------------|------------------------------------------------------------------------------------------------|
| Line: Messaging API      | Webhook                            | Entry point for Line messages           | External HTTP POST          | Message Type                    | ## Get Line Message from User User can send message in both text and Pay Slip image then classify the message type in text or image so we could have single workflow for AI Assistant that support anything. |
| Message Type            | Set                                | Extracts key message fields             | Line: Messaging API         | Message Classification          |                                                                                                |
| Message Classification  | Switch                             | Routes based on message type             | Message Type                | Text Message Processing, Line: Get Image |                                                                                                |
| Line: Get Image         | HTTP Request                      | Downloads image content from Line       | Message Classification (image branch) | Image Message Processing        |                                                                                                |
| Google Gemini for Image | LangChain Google Gemini AI Chat    | AI model for image analysis              | Image Message Processing    | Image Message Processing        | ## Extract text from image **Prompt for Gemini** Analyze image and then return in JSON Response that has the only following value: Status, From, To, Date, Amount |
| Image Message Processing | LangChain Chain LLM               | Defines prompt and processes image AI   | Line: Get Image, Google Gemini for Image | Line: Response to User, Text from Slip Result |                                                                                                |
| Line: Response to User  | HTTP Request                      | Sends AI JSON response to user          | Image Message Processing    | Text from Slip Result           | ## Reply to User Reply the processing result to the user without coding or OCR processing.       |
| Text from Slip Result   | Google Sheets                    | Appends extracted data to Google Sheets | Line: Response to User      | -                             | ## Insert result to Google Sheet Get all important information from the Pay Slip and insert into Google Sheet in the same format that we have provided in our prompt. |
| Google Gemini for Text  | LangChain Google Gemini AI Chat    | AI model for text message processing     | Text Message Processing     | Text Message Processing         |                                                                                                |
| Window Buffer Memory    | LangChain Memory Buffer Window     | Maintains conversational context        | Text Message Processing     | Text Message Processing         |                                                                                                |
| Text Message Processing | LangChain Agent                   | Conversational AI agent for text         | Message Classification (text branch), Window Buffer Memory, Google Gemini for Text | Line: Text Response to User | ## Gemini AI Assistant AI Assistant using Gemini 2.0 Flash Experiment unlocks new possibilities for AI agents - intelligent systems that can use memory, reasoning, and planning to complete tasks for you. |
| Line: Text Response to User | HTTP Request                  | Sends AI text response to user           | Text Message Processing     | -                             |                                                                                                |
| Sticky Note             | Sticky Note                      | Documentation for Gemini prompt          | -                          | -                             | ## Extract text from image **Prompt for Gemini** Analyze image and then return in JSON Response that has the only following value: Status, From, To, Date, Amount |
| Sticky Note1            | Sticky Note                      | Documentation for Gemini AI Assistant    | -                          | -                             | ## Gemini AI Assistant AI Assistant using Gemini 2.0 Flash Experiment unlocks new possibilities for AI agents - intelligent systems that can use memory, reasoning, and planning to complete tasks for you. |
| Sticky Note2            | Sticky Note                      | Documentation for reply to user          | -                          | -                             | ## Reply to User Reply the processing result to the user without coding or OCR processing.       |
| Sticky Note3            | Sticky Note                      | Documentation for Google Sheets insertion | -                          | -                             | ## Insert result to Google Sheet Get all important information from the Pay Slip and insert into Google Sheet in the same format that we have provided in our prompt. |
| Sticky Note4            | Sticky Note                      | Documentation for message reception      | -                          | -                             | ## Get Line Message from User User can send message in both text and Pay Slip image then classify the message type in text or image so we could have single workflow for AI Assistant that support anything. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node: "Line: Messaging API"**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: Unique identifier (e.g., "4c0de537-2889-47d2-ac44-3a9cda89c9f3")  
   - Purpose: Receive incoming messages from Line Messaging API.

2. **Create Set Node: "Message Type"**  
   - Type: Set  
   - Assign the following fields from webhook JSON:  
     - `body.events[0].message.text` → text  
     - `body.events[0].message.id` → message ID  
     - `body.events[0].source.userId` → user ID  
     - `body.events[0].message.type` → message type  
   - Connect input from "Line: Messaging API".

3. **Create Switch Node: "Message Classification"**  
   - Type: Switch  
   - Add rules to check `message.type` field:  
     - Equals "text" → route to text processing branch  
     - Equals "image" → route to image processing branch  
     - Equals stickerId 150 (optional) → unused branch  
   - Connect input from "Message Type".

4. **Text Message Processing Branch:**  
   a. **Create LangChain Memory Buffer Node: "Window Buffer Memory"**  
      - Type: Memory Buffer Window  
      - Session Key: `{{$json.body.events[0].source.userId}}`  
      - Session ID Type: customKey  
      - Connect input from "Message Classification" (text branch).  
   
   b. **Create LangChain Google Gemini AI Chat Node: "Google Gemini for Text"**  
      - Model Name: "models/gemini-2.0-flash-exp"  
      - Credentials: Google Palm API (configure with your API key)  
      - Connect input from "Message Classification" (text branch).  
   
   c. **Create LangChain Agent Node: "Text Message Processing"**  
      - Agent: conversationalAgent  
      - Text: `This is the message from User: {{$json.body.events[0].message.text}}`  
      - Connect AI model input from "Google Gemini for Text"  
      - Connect memory input from "Window Buffer Memory"  
      - Connect main input from "Message Classification" (text branch).  
   
   d. **Create HTTP Request Node: "Line: Text Response to User"**  
      - Method: POST  
      - URL: `https://api.line.me/v2/bot/message/reply`  
      - Authentication: HTTP Header Auth with Line credentials  
      - JSON Body:  
        ```json
        {
          "replyToken": "{{ $json.body.events[0].replyToken }}",
          "messages": [
            {
              "type": "text",
              "text": {{ JSON.stringify($json.output) }}
            }
          ]
        }
        ```  
      - Connect input from "Text Message Processing".

5. **Image Message Processing Branch:**  
   a. **Create HTTP Request Node: "Line: Get Image"**  
      - Method: GET  
      - URL: `https://api-data.line.me/v2/bot/message/{{ $json.body.events[0].message.id }}/content`  
      - Authentication: HTTP Header Auth with Line credentials  
      - Connect input from "Message Classification" (image branch).  
   
   b. **Create LangChain Google Gemini AI Chat Node: "Google Gemini for Image"**  
      - Model Name: "models/gemini-2.0-flash-exp"  
      - Credentials: Google Palm API  
      - Connect input from "Line: Get Image" (binary image data).  
   
   c. **Create LangChain Chain LLM Node: "Image Message Processing"**  
      - Prompt:  
        ```
        Analyze image and then return in JSON Response that has the only following Value:
        Status, From, To, Date, Amount
        ```  
      - Messages:  
        - System: "You are the image analyzer. You can analyze image and extract the important information from image."  
        - HumanMessagePromptTemplate: imageBinary  
      - Connect AI model input from "Google Gemini for Image"  
      - Connect main input from "Line: Get Image".  
   
   d. **Create HTTP Request Node: "Line: Response to User"**  
      - Method: POST  
      - URL: `https://api.line.me/v2/bot/message/reply`  
      - Authentication: HTTP Header Auth with Line credentials  
      - JSON Body:  
        ```json
        {
          "replyToken": "{{ $('Line: Messaging API').item.json.body.events[0].replyToken }}",
          "messages": [
            {
              "type": "text",
              "text": {{ JSON.stringify($json.text.replace(/^```(?:json|markdown)?\n?/, "").replace(/\n?```$/, "")) }}
            }
          ]
        }
        ```  
      - Connect input from "Image Message Processing".  
   
   e. **Create Google Sheets Node: "Text from Slip Result"**  
      - Operation: Append  
      - Document ID: URL of your Google Sheets document  
      - Sheet Name: "gid=0" or your sheet name  
      - Columns mapping: Parse JSON text from "Image Message Processing" output and map fields:  
        - Status  
        - From  
        - To  
        - Date  
        - Amount  
      - Credentials: Google Sheets OAuth2 API  
      - Connect input from "Line: Response to User".

6. **Add Sticky Notes for Documentation:**  
   - Add notes describing Gemini prompt, AI assistant capabilities, reply process, Google Sheets insertion, and message reception as per the original workflow.

7. **Credential Setup:**  
   - Configure HTTP Header Authentication credentials for Line Messaging API with required tokens.  
   - Configure Google Palm API credentials with your Google AI Studio API key.  
   - Configure Google Sheets OAuth2 credentials with access to your target spreadsheet.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Create Line Business ID for chatbot integration.                                                         | [Line Business](https://account.line.biz/login)                                                    |
| Google AI Studio API Key required for Google Gemini AI.                                                  | [Google AI Studio API Key](https://aistudio.google.com/apikey)                                     |
| Google Sheets example template with columns: Status, From, To, Date, Amount.                             | [Google Sheets](http://docs.google.com/spreadsheets)                                               |
| Workflow enables no-code text extraction from images using Google Gemini 2.0 Flash experimental model.  |                                                                                                |
| Multipurpose chatbot supports both text and image messages in a single workflow.                         |                                                                                                |
| Reduces human error by automating pay slip data extraction and verification.                            |                                                                                                |

---

This documentation provides a comprehensive reference for understanding, reproducing, and maintaining the "Extract Pay Slip Data with Line Chatbot and Gemini to Google Sheets" workflow. It covers all nodes, logic branches, configurations, and integration points with external services.