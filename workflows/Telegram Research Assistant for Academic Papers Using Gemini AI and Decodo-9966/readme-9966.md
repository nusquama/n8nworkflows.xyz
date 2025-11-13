Telegram Research Assistant for Academic Papers Using Gemini AI and Decodo

https://n8nworkflows.xyz/workflows/telegram-research-assistant-for-academic-papers-using-gemini-ai-and-decodo-9966


# Telegram Research Assistant for Academic Papers Using Gemini AI and Decodo

### 1. Workflow Overview

This workflow implements a **Telegram Research Assistant for Academic Papers** leveraging **Google Gemini AI** and **Decodo** scraping tool. It is designed to assist researchers, students, and AI enthusiasts by enabling search, analysis, and summarization of academic papers directly through Telegram messages.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Message Type Detection**: Captures Telegram messages (text, photo, voice), determines their type, and handles unsupported formats.
- **1.2 Media Download and Preprocessing**: Downloads photo or voice files if applicable and processes them (image analysis or voice transcription).
- **1.3 Text Formatting and Preparation**: Converts processed content into a text format suitable for AI processing and prepares chat metadata.
- **1.4 Academic URL Insight Generation**: Analyzes and explains academic search URLs to guide the research agent in query construction.
- **1.5 Research Summary Agent (AI Processing)**: Uses Gemini AI and Decodo to interpret user intent, generate search queries, scrape academic data, and summarize research papers.
- **1.6 Output Handling and Telegram Response**: Checks the length of the generated summary, sends it as a Telegram message or a text file if too long.
- **1.7 Auxiliary Nodes**: Includes sticky notes for documentation and instructions and simple memory for session management.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Message Type Detection

- **Overview:**  
  Captures incoming Telegram messages and classifies them into Photo, Text, Voice, or unsupported types to route for appropriate processing or fallback.

- **Nodes Involved:**  
  - Start Telegram Bot  
  - Detect Message Type  
  - Send Fallback Text

- **Node Details:**

  - **Start Telegram Bot**  
    - *Type:* Telegram Trigger  
    - *Role:* Entry trigger for Telegram messages (updates of type "message")  
    - *Configuration:* Uses Telegram API credentials of the research bot; listens for all incoming messages.  
    - *Input/Output:* No input; outputs raw Telegram message JSON.  
    - *Failures:* Telegram API connectivity or webhook misconfiguration.

  - **Detect Message Type**  
    - *Type:* Switch Node  
    - *Role:* Determines if the message contains a photo, text, voice, or other unsupported content.  
    - *Configuration:*  
      - Checks for non-empty `message.photo` array → Photo  
      - Checks for non-empty `message.text` → Text  
      - Checks for non-empty `message.voice` → Voice  
      - Fallback → "extra" (unsupported)  
    - *Input:* Output of Start Telegram Bot  
    - *Output:* Routes to corresponding downstream nodes.  
    - *Edge Cases:* Messages with multiple content types; fallback for unsupported media.

  - **Send Fallback Text**  
    - *Type:* Telegram Node  
    - *Role:* Sends a fixed fallback message when unsupported message types are received.  
    - *Configuration:* Text: "Sorry, i can only support text, photo, and voice message"  
    - *Input:* Routed from Detect Message Type fallback  
    - *Output:* Sends response to original chat ID  
    - *Failures:* Telegram API errors.

---

#### 2.2 Media Download and Preprocessing

- **Overview:**  
  Downloads photos or voice messages from Telegram and processes them via Gemini AI for content analysis or transcription.

- **Nodes Involved:**  
  - Download Telegram Photo  
  - Analyze Image Content  
  - Format Image Text  
  - Download Telegram Voice  
  - Transcribe Voice Message  
  - Format Voice Text

- **Node Details:**

  - **Download Telegram Photo**  
    - *Type:* Telegram Node  
    - *Role:* Downloads the highest resolution photo (4th element in photo array) sent by the user.  
    - *Configuration:* Uses `message.photo[3].file_id` to fetch file  
    - *Input:* From Detect Message Type (Photo output)  
    - *Output:* Binary file data for image  
    - *Edge Cases:* Photo array length less than 4, file download issues.

  - **Analyze Image Content**  
    - *Type:* Google Gemini AI Node (Image Analysis)  
    - *Role:* Analyzes downloaded photo content to extract textual description or metadata.  
    - *Configuration:* Uses Gemini 2.5 Flash model for image analysis  
    - *Input:* Binary image file from Download Telegram Photo  
    - *Output:* JSON with content parts describing image  
    - *Credentials:* Google Palm API  
    - *Failures:* API limits, unsupported image formats.

  - **Format Image Text**  
    - *Type:* Set Node  
    - *Role:* Formats analyzed image text and optional caption into a single text string for downstream processing.  
    - *Configuration:* Constructs `message.text` with caption and image analysis text  
    - *Input:* From Analyze Image Content and original message caption  
    - *Output:* JSON with formatted text  
    - *Edge Cases:* Missing caption or content parts.

  - **Download Telegram Voice**  
    - *Type:* Telegram Node  
    - *Role:* Downloads voice message file from Telegram.  
    - *Configuration:* Uses `message.voice.file_id`  
    - *Input:* From Detect Message Type (Voice output)  
    - *Output:* Binary audio data  
    - *Edge Cases:* File download errors.

  - **Transcribe Voice Message**  
    - *Type:* Google Gemini AI Node (Audio Processing)  
    - *Role:* Transcribes voice message audio into text using Gemini AI.  
    - *Configuration:* Gemini 2.5 Flash model, audio resource, binary input  
    - *Input:* Binary audio file from Download Telegram Voice  
    - *Output:* JSON with transcribed text parts  
    - *Credentials:* Google Palm API  
    - *Failures:* API errors, unclear audio.

  - **Format Voice Text**  
    - *Type:* Set Node  
    - *Role:* Extracts transcribed text and sets it as `message.text` for further processing.  
    - *Input:* Transcribe Voice Message output  
    - *Output:* JSON with formatted text  
    - *Edge Cases:* Empty transcription results.

---

#### 2.3 Text Formatting and Preparation

- **Overview:**  
  Prepares a unified chat input text and chat ID metadata for the research agent to process.

- **Nodes Involved:**  
  - Prepare Chat Data

- **Node Details:**

  - **Prepare Chat Data**  
    - *Type:* Set Node  
    - *Role:* Sets key JSON fields `chatInput` (the text to analyze) and `chatId` (Telegram chat ID) for downstream use.  
    - *Configuration:*  
      - `chatInput` = `message.text`  
      - `chatId` = chat ID from original Telegram message  
    - *Input:* From either Format Image Text, Format Voice Text, or directly from Detect Message Type (Text output)  
    - *Output:* JSON with `chatInput` and `chatId` fields  
    - *Edge Cases:* Missing or empty text input.

---

#### 2.4 Academic URL Insight Generation

- **Overview:**  
  Generates detailed, structured explanations of academic search URLs to inform the research agent how to construct search queries.

- **Nodes Involved:**  
  - Define Search URLs  
  - Gemini URL Interpreter  
  - Generate Search URL Insights

- **Node Details:**

  - **Define Search URLs**  
    - *Type:* Set Node  
    - *Role:* Provides an array of academic search URLs to analyze, e.g., Google Scholar and arXiv queries about artificial intelligence.  
    - *Configuration:* Hardcoded list of URLs as JSON array  
    - *Output:* JSON with `urls` array  
    - *Edge Cases:* URLs must be valid and relevant.

  - **Gemini URL Interpreter**  
    - *Type:* Google Gemini AI Chat Node  
    - *Role:* Processes the URLs array to extract, group, and explain query parameters semantically.  
    - *Configuration:* Uses Google Palm API, default options  
    - *Input:* From Define Search URLs  
    - *Output:* Markdown-formatted explanation of each URL and its parameters  
    - *Failures:* API errors, large input size.

  - **Generate Search URL Insights**  
    - *Type:* Chain LLM Node  
    - *Role:* Orchestrates a prompt chain that instructs Gemini AI to analyze the academic URLs and produce structured insight.  
    - *Configuration:* Complex prompt templates designed to produce detailed parameter analysis and notes.  
    - *Input:* Feeds from Gemini URL Interpreter and Define Search URLs outputs  
    - *Output:* Markdown insight used later by Research Summary Agent  
    - *Failures:* Prompt misalignment, API limits.

---

#### 2.5 Research Summary Agent (AI Processing)

- **Overview:**  
  The core AI agent interpreting user queries with context from chat, using Gemini AI and Decodo to scrape academic sources, generate search queries, and summarize papers succinctly.

- **Nodes Involved:**  
  - Simple Memory  
  - Decodo  
  - Gemini Research Model  
  - Research Summary Agent

- **Node Details:**

  - **Simple Memory**  
    - *Type:* Langchain Memory Buffer Window  
    - *Role:* Maintains a session-based memory buffer keyed by Telegram chat ID to allow conversational context.  
    - *Configuration:*  
      - `sessionKey` = Telegram chat ID from incoming JSON  
      - `sessionIdType` = customKey  
    - *Input:* From Prepare Chat Data (via Research Summary Agent)  
    - *Output:* Memory context used by Research Summary Agent  
    - *Edge Cases:* Memory overflow, session key mismatch.

  - **Decodo**  
    - *Type:* Decodo Tool Node  
    - *Role:* Scrapes academic web pages to extract paper metadata (titles, authors, abstracts, publication details).  
    - *Configuration:* Uses Decodo API credentials  
    - *Input:* Called internally by Research Summary Agent as a tool  
    - *Output:* Extracted structured academic data  
    - *Failures:* API quota, network issues.

  - **Gemini Research Model**  
    - *Type:* Google Gemini AI Chat Node  
    - *Role:* Provides language model capabilities for natural language understanding and generation within the agent.  
    - *Configuration:* Uses Google Palm API credentials  
    - *Input:* Message context and instructions from Research Summary Agent  
    - *Output:* AI-generated text responses or search queries  
    - *Failures:* API errors, model availability.

  - **Research Summary Agent**  
    - *Type:* Langchain Agent Node  
    - *Role:* Implements the research assistant logic: interprets user intent, extracts keywords, plans search queries, uses Decodo to scrape results, ranks and summarizes papers, and formats output.  
    - *Configuration:*  
      - System message includes detailed instructions on objectives, rules, and tools.  
      - Placeholder `{{INPUT_SEARCH_URL_INSIGHTS}}` must be replaced with insights generated in 2.4.  
      - Uses Simple Memory and Decodo as integrations  
    - *Input:* Prepared chat input and memory context  
    - *Output:* Structured summary text  
    - *Edge Cases:* Misinterpretation of queries, empty results, API limits.

---

#### 2.6 Output Handling and Telegram Response

- **Overview:**  
  Checks the length of the generated summary to decide whether to send as a Telegram message or as a text file attachment.

- **Nodes Involved:**  
  - Check Telegram Message Length  
  - Convert Output to Text File  
  - Send Research Summary Message  
  - Send Research Summary File

- **Node Details:**

  - **Check Telegram Message Length**  
    - *Type:* If Node  
    - *Role:* Tests if the summary output length exceeds 4000 characters (Telegram message limit).  
    - *Input:* Output text from Research Summary Agent  
    - *Output:* Routes to text message node or file conversion node  
    - *Edge Cases:* Message exactly at limit.

  - **Convert Output to Text File**  
    - *Type:* Convert To File Node  
    - *Role:* Converts long text output into a .txt file named "research_summary" for sending as a document.  
    - *Input:* Long summary text from Research Summary Agent  
    - *Output:* Binary file data  
    - *Edge Cases:* Encoding issues.

  - **Send Research Summary Message**  
    - *Type:* Telegram Node  
    - *Role:* Sends the research summary as a Telegram text message with HTML parsing enabled.  
    - *Input:* Short summary text and chat ID from Prepare Chat Data  
    - *Output:* Telegram message sent to user  
    - *Failures:* Telegram API errors.

  - **Send Research Summary File**  
    - *Type:* Telegram Node  
    - *Role:* Sends the converted text file as a document with a caption explaining the reason.  
    - *Input:* Binary file from Convert Output to Text File and chat ID  
    - *Output:* Telegram document sent to user  
    - *Failures:* Telegram API errors.

---

#### 2.7 Auxiliary Nodes

- **Sticky Note** (Main Documentation)  
  - Provides user-friendly documentation, setup instructions, use cases, and links for Decodo signup.

- **Sticky Note2** (Search URL Insights Instructions)  
  - Explains how to use the URL insight generation block to improve Research Summary Agent performance.

- **Sticky Note7** and **Sticky Note8**  
  - Provide setup tips for Telegram Bot credentials and configuring the system message placeholder.

---

### 3. Summary Table

| Node Name                | Node Type                             | Functional Role                                      | Input Node(s)             | Output Node(s)                        | Sticky Note                                                                                          |
|--------------------------|-------------------------------------|-----------------------------------------------------|---------------------------|-------------------------------------|----------------------------------------------------------------------------------------------------|
| Start Telegram Bot        | Telegram Trigger                    | Entry point; receives Telegram messages              | None                      | Detect Message Type                 |                                                                                                    |
| Detect Message Type       | Switch                             | Determines message media type                         | Start Telegram Bot         | Download Telegram Photo, Prepare Chat Data, Download Telegram Voice, Send Fallback Text |                                                                                                    |
| Send Fallback Text        | Telegram                           | Sends fallback for unsupported message types         | Detect Message Type        | None                               |                                                                                                    |
| Download Telegram Photo   | Telegram                           | Downloads photo file from Telegram                     | Detect Message Type        | Analyze Image Content               |                                                                                                    |
| Analyze Image Content     | Google Gemini AI (Image Analysis) | Extracts textual info from image                      | Download Telegram Photo    | Format Image Text                  |                                                                                                    |
| Format Image Text         | Set                               | Formats image analysis and caption into text         | Analyze Image Content      | Prepare Chat Data                  |                                                                                                    |
| Download Telegram Voice   | Telegram                           | Downloads voice message file                           | Detect Message Type        | Transcribe Voice Message           |                                                                                                    |
| Transcribe Voice Message  | Google Gemini AI (Audio)           | Transcribes voice message to text                      | Download Telegram Voice    | Format Voice Text                  |                                                                                                    |
| Format Voice Text         | Set                               | Formats transcribed voice text                         | Transcribe Voice Message   | Prepare Chat Data                  |                                                                                                    |
| Prepare Chat Data         | Set                               | Prepares unified text input and chat ID               | Format Image Text, Format Voice Text, Detect Message Type (Text) | Research Summary Agent             |                                                                                                    |
| Simple Memory             | Langchain Memory Buffer            | Maintains conversational context                      | Prepare Chat Data          | Research Summary Agent             |                                                                                                    |
| Decodo                   | Decodo Tool                       | Scrapes academic paper metadata                        | Research Summary Agent (ai_tool) | Research Summary Agent           |                                                                                                    |
| Gemini Research Model     | Google Gemini AI Chat             | Provides language model for text understanding         | Research Summary Agent (ai_languageModel) | Research Summary Agent      |                                                                                                    |
| Research Summary Agent    | Langchain Agent                   | Core AI agent for research intent interpretation, search, scraping, and summarization | Prepare Chat Data, Simple Memory, Decodo, Gemini Research Model | Check Telegram Message Length       | Sticky Note8: Replace _{{INPUT_SEARCH_URL_INSIGHTS}}_ placeholder in system message with insights. |
| Check Telegram Message Length | If                             | Checks if output length exceeds 4000 chars             | Research Summary Agent     | Convert Output to Text File, Send Research Summary Message |                                                                                                    |
| Convert Output to Text File | Convert To File                  | Converts long text summary to text file                | Check Telegram Message Length | Send Research Summary File       |                                                                                                    |
| Send Research Summary Message | Telegram                       | Sends summary as Telegram text message                 | Check Telegram Message Length | None                            |                                                                                                    |
| Send Research Summary File | Telegram                         | Sends summary as Telegram document (file)              | Convert Output to Text File | None                             |                                                                                                    |
| Define Search URLs        | Set                               | Provides list of academic search URLs                  | None                      | Generate Search URL Insights       |                                                                                                    |
| Gemini URL Interpreter    | Google Gemini AI Chat             | Analyzes and explains academic search URLs            | Define Search URLs         | Generate Search URL Insights       |                                                                                                    |
| Generate Search URL Insights | Chain LLM                      | Generates Markdown insights on academic search URLs   | Define Search URLs, Gemini URL Interpreter | Research Summary Agent (via manual copy-paste) | Sticky Note2: Instructions on generating and using URL insights.                                  |
| Sticky Note               | Sticky Note                      | Documentation and setup instructions                   | None                      | None                               | Contains detailed workflow overview, setup instructions, and Decodo signup link.                   |
| Sticky Note2              | Sticky Note                      | Instructions for URL insight generation                 | None                      | None                               | Explains how to generate and use search URL insights.                                              |
| Sticky Note7              | Sticky Note                      | Telegram bot creation instructions                      | None                      | None                               | Advises to create Telegram bot via BotFather and add credentials.                                 |
| Sticky Note8              | Sticky Note                      | Reminder to replace placeholder in system message       | None                      | None                               | Advises to replace _{{INPUT_SEARCH_URL_INSIGHTS}}_ in Research Summary Agent system message.       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Bot Trigger Node**  
   - Node Type: Telegram Trigger  
   - Configure credentials with your Telegram bot token.  
   - Set Update Types to "message" only.

2. **Add Switch Node to Detect Message Type**  
   - Node Type: Switch  
   - Add rules:  
     - Photo: Check if `message.photo` array is not empty  
     - Text: Check if `message.text` is not empty  
     - Voice: Check if `message.voice` is not empty  
   - Set fallback to handle unsupported types.

3. **Add Telegram Node to Send Fallback Text**  
   - Node Type: Telegram  
   - Configure with Telegram credentials.  
   - Text: "Sorry, i can only support text, photo, and voice message"  
   - Chat ID: `{{$json.message.chat.id}}`

4. **Add Telegram Node to Download Photo**  
   - Node Type: Telegram  
   - File ID: `{{$json.message.photo[3].file_id}}` (ensure array length)  
   - Resource: file  
   - Connect from Switch Photo output.

5. **Add Google Gemini AI Node to Analyze Image**  
   - Node Type: LM Chat Google Gemini (resource: image, operation: analyze)  
   - Model: "models/gemini-2.5-flash"  
   - Input Type: binary  
   - Connect from Download Telegram Photo.

6. **Add Set Node to Format Image Text**  
   - Create field `message.text` with value:  
     ```
     Caption: {{$json.message.caption ?? "[none]"}}
     ---
     Image: {{$json.content.parts[0].text}}
     ```  
   - Connect from Analyze Image Content.

7. **Add Telegram Node to Download Voice Message**  
   - Node Type: Telegram  
   - File ID: `{{$json.message.voice.file_id}}`  
   - Resource: file  
   - Connect from Switch Voice output.

8. **Add Google Gemini AI Node to Transcribe Voice**  
   - Node Type: LM Chat Google Gemini (resource: audio)  
   - Model: "models/gemini-2.5-flash"  
   - Input Type: binary  
   - Connect from Download Telegram Voice.

9. **Add Set Node to Format Voice Text**  
   - Create field `message.text` with value: `{{$json.content.parts[0].text}}`  
   - Connect from Transcribe Voice Message.

10. **Add Set Node to Prepare Chat Data**  
    - Fields:  
      - `chatInput` = `{{$json.message.text}}`  
      - `chatId` = `{{$json.message.chat.id}}` (from original message)  
    - Connect from:  
      - Format Image Text  
      - Format Voice Text  
      - Switch Text output (direct text message)

11. **Add Simple Memory Node**  
    - Type: Langchain Memory Buffer Window  
    - Session Key: `{{$json.chatId}}`  
    - Session ID Type: customKey  
    - Connect from Prepare Chat Data.

12. **Add Decodo Node**  
    - Type: Decodo Tool Node  
    - Configure Decodo API credentials  
    - This node will be used as an AI tool integration by the Research Summary Agent.

13. **Add Google Gemini AI Node for Research Model**  
    - Type: LM Chat Google Gemini  
    - Configure Google Palm API credentials  
    - Used as language model integration by the Research Summary Agent.

14. **Add Research Summary Agent Node**  
    - Type: Langchain Agent  
    - System Message: Paste detailed instructions including objectives, rules, and tools.  
    - Replace `{{INPUT_SEARCH_URL_INSIGHTS}}` placeholder with the output from URL insights generation.  
    - Configure integrations:  
      - AI tool: Decodo node  
      - AI memory: Simple Memory node  
      - AI language model: Gemini Research Model node  
    - Connect from Prepare Chat Data.

15. **Add If Node to Check Output Length**  
    - Condition: Check if `{{$json.output.length}} > 4000` (Telegram message character limit)  
    - Connect from Research Summary Agent output.

16. **Add Convert to File Node**  
    - Operation: toText  
    - File Name: "research_summary"  
    - Source Property: `output` (text from agent)  
    - Connect from If Node (true branch: output too long).

17. **Add Telegram Node to Send Research Summary File**  
    - Operation: sendDocument  
    - Binary Data: true  
    - Caption: "I've compiled the search result into a txt file because the content is too long for a single telegram message"  
    - Chat ID: from Prepare Chat Data `chatId`  
    - Connect from Convert Output to Text File.

18. **Add Telegram Node to Send Research Summary Message**  
    - Text: `{{$json.output}}`  
    - Chat ID: from Prepare Chat Data `chatId`  
    - Parse Mode: HTML  
    - Connect from If Node (false branch: output short enough).

19. **Add Define Search URLs Node**  
    - Type: Set  
    - Assign field: `urls` with array of academic URLs (e.g., Google Scholar, arXiv).  
    - This is input for URL insights generation.

20. **Add Gemini URL Interpreter Node**  
    - Type: LM Chat Google Gemini  
    - Connect from Define Search URLs.

21. **Add Generate Search URL Insights Node**  
    - Type: Chain LLM  
    - Use prompt templates to analyze URLs and extract parameter explanations.  
    - Connect from Define Search URLs and Gemini URL Interpreter.

22. **Manual Step:**  
    - Copy the Markdown output from Generate Search URL Insights node.  
    - Paste it into the `{{INPUT_SEARCH_URL_INSIGHTS}}` placeholder in the Research Summary Agent system message.

23. **Add Sticky Notes** for documentation and setup instructions as per original workflow (optional but recommended for users).

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                      | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| AI Research Assistant Using Gemini AI and Decodo. Sign up for Decodo [HERE](https://visit.decodo.com/discount) for Discount. This workflow transforms your Telegram bot into a smart academic research assistant powered by Gemini AI and Decodo. It analyzes queries, interprets URLs, scrapes scholarly data, and returns concise summaries of research papers directly in chat. | Main Sticky Note with branding and signup link                                                  |
| Search URL Insights: Use this workflow section to generate clear explanations of academic search URLs (Google Scholar, arXiv, PubMed, etc.) so that the Research Summary Agent can construct smarter search queries. Steps to use: input your URLs, run the workflow, and copy-paste results into the agent’s system message.                                                  | Sticky Note2 with instructions for URL insight generation                                      |
| Create your Telegram Bot using BotFather and paste its API key into n8n credentials to enable Telegram integration.                                                                                                                                                                                                                                            | Sticky Note7                                                                                     |
| Replace the `{{INPUT_SEARCH_URL_INSIGHTS}}` placeholder in the Research Summary Agent system message with generated URL insights for proper operation.                                                                                                                                                                                                           | Sticky Note8                                                                                     |

---

**Disclaimer:** The provided text is extracted exclusively from an automated workflow created with n8n, adhering strictly to current content policies and containing no illegal or protected elements. All processed data is legal and public.