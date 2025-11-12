Transcribe YouTube Videos with AI Enhancement via Chat Interface

https://n8nworkflows.xyz/workflows/transcribe-youtube-videos-with-ai-enhancement-via-chat-interface-3381


# Transcribe YouTube Videos with AI Enhancement via Chat Interface

### 1. Workflow Overview

This workflow automates the transcription of YouTube videos by accepting a video URL via a chat interface, validating it, retrieving the transcript using an external service (Supadata), enhancing the transcription with AI (OpenAI), and returning the refined text back to the user through the chat interface.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures user input (YouTube video URL) via a chat message trigger.
- **1.2 URL Validation:** Validates the format and correctness of the provided YouTube URL.
- **1.3 Conditional Routing:** Routes the flow based on URL validation success or failure.
- **1.4 Transcription Retrieval:** Calls the Supadata API to fetch the video transcript.
- **1.5 AI Enhancement:** Uses OpenAI to correct grammar, punctuation, and structure of the transcript.
- **1.6 Response Delivery:** Sends the final or error message back to the user via the chat interface.
- **1.7 Error Handling:** Manages errors gracefully at various stages, providing meaningful feedback.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block initiates the workflow by receiving a chat message containing a YouTube video URL from the user.

- **Nodes Involved:**  
  - When chat message received

- **Node Details:**

  - **When chat message received**  
    - *Type:* Chat Trigger (LangChain)  
    - *Role:* Entry point for the workflow; listens for chat messages publicly.  
    - *Configuration:*  
      - Public webhook enabled.  
      - UI options include a title ("Youtube Video Transcriber 🚀"), subtitle, and input placeholder prompting for a YouTube URL.  
      - Initial message prompts user to provide a YouTube video URL.  
    - *Input:* User chat message with video URL.  
    - *Output:* JSON containing the chat input text under `chatInput`.  
    - *Edge Cases:*  
      - No input or empty message (handled downstream).  
      - Malformed input (validated later).  
    - *Version:* 1.1

#### 2.2 URL Validation

- **Overview:**  
  Validates the YouTube URL format and extracts the video ID to ensure the input is a valid YouTube video link before proceeding.

- **Nodes Involved:**  
  - Code

- **Node Details:**

  - **Code**  
    - *Type:* Code (Python)  
    - *Role:* Performs regex-based validation of the YouTube URL and extracts the video ID.  
    - *Configuration:*  
      - Python code checks for presence, length, and pattern matching against known YouTube URL formats (standard watch URL, shortened youtu.be, embed, and /v/ URLs).  
      - Returns a JSON object with keys: `text` (original URL or error message) and `is_valid` (boolean).  
    - *Key Expressions:* Uses regex patterns for validation and extraction.  
    - *Input:* Receives chat input text from previous node.  
    - *Output:* Validation result JSON.  
    - *Edge Cases:*  
      - Empty or missing URL.  
      - URL too short or invalid format.  
      - Invalid or missing video ID.  
      - Unexpected exceptions propagate as errors.  
    - *Version:* 2

#### 2.3 Conditional Routing

- **Overview:**  
  Routes the workflow based on whether the URL validation succeeded or failed.

- **Nodes Involved:**  
  - If

- **Node Details:**

  - **If**  
    - *Type:* If (Conditional)  
    - *Role:* Checks the boolean `is_valid` field from the validation node.  
    - *Configuration:*  
      - Condition: `is_valid` equals `true`.  
    - *Input:* Output from Code node.  
    - *Output:*  
      - True branch: Valid URL → proceed to transcription retrieval.  
      - False branch: Invalid URL → respond with error message.  
    - *Edge Cases:*  
      - Missing `is_valid` field (treated as false).  
    - *Version:* 2.2

#### 2.4 Transcription Retrieval

- **Overview:**  
  Calls the Supadata API to retrieve the transcript of the YouTube video using the validated URL.

- **Nodes Involved:**  
  - HTTP Request

- **Node Details:**

  - **HTTP Request**  
    - *Type:* HTTP Request  
    - *Role:* Sends a GET request to Supadata’s YouTube transcription API.  
    - *Configuration:*  
      - URL constructed dynamically with the validated video URL as a query parameter.  
      - Query parameters: `url` (video URL), `text=true` (request transcript text), `lang=pt` (Portuguese language setting, customizable).  
      - Header includes `x-api-key` with Supadata API key credential.  
      - Timeout set to 300 seconds (5 minutes) to accommodate longer processing.  
      - On error: continues workflow with error output for handling.  
    - *Input:* Validated video URL from If node.  
    - *Output:* JSON response containing transcript data or error info.  
    - *Edge Cases:*  
      - API key missing or invalid → authentication error.  
      - Video inaccessible or no captions → empty or error response.  
      - Timeout or network issues.  
    - *Version:* 4.2

#### 2.5 AI Enhancement

- **Overview:**  
  Uses OpenAI to improve the transcript’s grammar, punctuation, and textual structure for readability and clarity.

- **Nodes Involved:**  
  - OpenAI

- **Node Details:**

  - **OpenAI**  
    - *Type:* OpenAI (LangChain)  
    - *Role:* Processes the raw transcript text to correct grammar and punctuation, and organizes content with markdown formatting.  
    - *Configuration:*  
      - Model: `gpt-4o-mini-2024-07-18` (latest GPT-4 variant).  
      - System prompt instructs the model to act as an expert in grammar correction and textual structuring, maintaining original wording but improving punctuation and formatting in Portuguese.  
      - User prompt dynamically includes the transcript content from the HTTP Request node.  
      - OpenAI API credentials configured.  
      - Retries enabled on failure.  
    - *Input:* Raw transcript text from Supadata API response.  
    - *Output:* Corrected and formatted transcript text.  
    - *Edge Cases:*  
      - API key missing or invalid → authentication error.  
      - Rate limits or timeouts.  
      - Empty or malformed transcript input.  
    - *Version:* 1.8

#### 2.6 Response Delivery

- **Overview:**  
  Sends the final processed transcript or error messages back to the user via the chat interface.

- **Nodes Involved:**  
  - Edit Fields - Respond to Chat Message 3  
  - Edit Fields - Respond to Chat Message 4  
  - Edit Fields - Respond to Chat Message 2  
  - Respond to Webhook - Chat Message

- **Node Details:**

  - **Edit Fields - Respond to Chat Message 3**  
    - *Type:* Set  
    - *Role:* Prepares the successful response text from OpenAI’s output for sending.  
    - *Configuration:* Sets `text` field to OpenAI’s `message.content`.  
    - *Input:* OpenAI node output.  
    - *Output:* JSON with `text` field for response.  
    - *Version:* 3.4

  - **Edit Fields - Respond to Chat Message 4**  
    - *Type:* Set  
    - *Role:* Prepares a generic error message if AI processing fails.  
    - *Configuration:* Sets `text` to "Something went wrong with the data structuring."  
    - *Input:* OpenAI error output.  
    - *Output:* JSON with error text.  
    - *Version:* 3.4

  - **Edit Fields - Respond to Chat Message 2**  
    - *Type:* Set  
    - *Role:* Prepares error message from HTTP Request failure (e.g., Supadata API failure).  
    - *Configuration:* Sets `text` to a concatenation of error fields: `error - message`.  
    - *Input:* HTTP Request error output.  
    - *Output:* JSON with error text.  
    - *Version:* 3.4

  - **Respond to Webhook - Chat Message**  
    - *Type:* Respond to Webhook  
    - *Role:* Sends the final text response back to the chat interface user.  
    - *Configuration:*  
      - Responds with plain text using the `text` field from input JSON.  
      - Retries enabled on failure.  
    - *Input:* Either error message from invalid URL or from HTTP Request failure.  
    - *Output:* HTTP response to chat client.  
    - *Version:* 1.1

#### 2.7 Error Handling

- **Overview:**  
  The workflow uses `continueErrorOutput` on HTTP Request and OpenAI nodes to catch errors and route them to appropriate error message preparation nodes, ensuring the user receives meaningful feedback instead of silent failures.

- **Nodes Involved:**  
  - HTTP Request (onError: continueErrorOutput)  
  - OpenAI (onError: continueErrorOutput)  
  - Edit Fields nodes for error messages  
  - Respond to Webhook - Chat Message

- **Edge Cases:**  
  - API failures, timeouts, invalid credentials, empty responses, or unexpected exceptions are caught and translated into user-friendly messages.

---

### 3. Summary Table

| Node Name                        | Node Type                     | Functional Role                      | Input Node(s)                  | Output Node(s)                         | Sticky Note                                                                                                     |
|---------------------------------|-------------------------------|------------------------------------|--------------------------------|---------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| When chat message received       | Chat Trigger (LangChain)       | Entry point: receives YouTube URL  | -                              | Code                                  | ## Entry Point The workflow entry point is the  node chat message.                                              |
| Code                            | Code (Python)                  | Validates YouTube URL format       | When chat message received      | If                                    | ## Validation - URL This node ensures that only a valid youtube video url goes forward.                         |
| If                              | If (Conditional)               | Routes flow based on URL validity  | Code                           | HTTP Request (true), Respond to Webhook (false) |                                                                                                                 |
| HTTP Request                   | HTTP Request                   | Calls Supadata API for transcript  | If (true)                     | OpenAI (success), Edit Fields - Respond to Chat Message 2 (error) | ## Supadata Supadata is a powerful tool that converts YouTube video URLs into structured data via a simple API. It efficiently extracts transcriptions, making it ideal for AI training, data analysis, or text-based applications. |
| OpenAI                         | OpenAI (LangChain)             | Enhances transcript with AI        | HTTP Request (success)          | Edit Fields - Respond to Chat Message 3 (success), Edit Fields - Respond to Chat Message 4 (error) | ## Data Structuring Here is the core of the workflow, where structuring is done to get the right format to answer. **NOTE:** 1. Users implementing this template must modify the language in the OpenAI prompt to suit their desired output. 2. An OpenAI API key is essential and must be properly configured to support data structuring and processing. |
| Edit Fields - Respond to Chat Message 3 | Set                      | Prepares successful response text  | OpenAI (success)                | Respond to Webhook - Chat Message      |                                                                                                                 |
| Edit Fields - Respond to Chat Message 4 | Set                      | Prepares AI error message           | OpenAI (error)                  | Respond to Webhook - Chat Message      |                                                                                                                 |
| Edit Fields - Respond to Chat Message 2 | Set                      | Prepares HTTP Request error message| HTTP Request (error)            | Respond to Webhook - Chat Message      |                                                                                                                 |
| Respond to Webhook - Chat Message | Respond to Webhook            | Sends final response to user       | If (false), Edit Fields nodes   | -                                     |                                                                                                                 |
| Sticky Note                     | Sticky Note                   | Documentation                      | -                              | -                                     | ## Description This workflow simplifies access to YouTube video content converting into clear and concise transcriptions, ideal for users seeking practicality. It transcribes YouTube videos directly and returns the text, eliminating the need to watch the full video. |
| Sticky Note1                    | Sticky Note                   | Documentation                      | -                              | -                                     | ## Validation - URL This node ensures that only a valid youtube video url goes forward.                         |
| Sticky Note2                    | Sticky Note                   | Documentation                      | -                              | -                                     | ## Data Structuring Here is the core of the workflow, where structuring is done to get the right format to answer. **NOTE:** 1. Users implementing this template must modify the language in the OpenAI prompt to suit their desired output. 2. An OpenAI API key is essential and must be properly configured to support data structuring and processing. |
| Sticky Note4                    | Sticky Note                   | Documentation                      | -                              | -                                     | ## Supadata Supadata is a powerful tool that converts YouTube video URLs into structured data via a simple API. It efficiently extracts transcriptions, making it ideal for AI training, data analysis, or text-based applications. |
| Sticky Note5                    | Sticky Note                   | Documentation                      | -                              | -                                     | ## Description This workflow simplifies access to YouTube video content converting into clear and concise transcriptions, ideal for users seeking practicality. It transcribes YouTube videos directly and returns the text, eliminating the need to watch the full video. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Chat Trigger Node**  
   - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
   - Configure as public webhook.  
   - Set UI options:  
     - Title: "Youtube Video Transcriber 🚀"  
     - Subtitle: "Have a great transcription!  📖"  
     - Input placeholder: "Insert a URL of a YouTube video.  💻"  
   - Initial message: "Give me a URL of a video from YouTube to start! 👍"  
   - Position: e.g., (-100, -60)

2. **Add a Code Node for URL Validation**  
   - Type: `Code` (Python)  
   - Paste the provided Python code that validates YouTube URLs using regex patterns.  
   - Input: Connect from Chat Trigger node.  
   - Output: JSON with `text` (URL or error message) and `is_valid` (boolean).  
   - Position: e.g., (280, -60)

3. **Add an If Node for Conditional Routing**  
   - Type: `If`  
   - Condition: Check if `is_valid` equals `true`.  
   - Input: Connect from Code node.  
   - Position: e.g., (600, -240)

4. **Add HTTP Request Node to Call Supadata API**  
   - Type: `HTTP Request`  
   - Method: GET  
   - URL: `https://api.supadata.ai/v1/youtube/transcript?url={{ $json.text }}&text=true&lang=pt`  
     - Note: Replace `lang=pt` with desired language code if needed.  
   - Headers: Add header `x-api-key` with value referencing Supadata API credential.  
   - Timeout: 300000 ms (5 minutes)  
   - On Error: Set to `continueErrorOutput` to handle errors gracefully.  
   - Input: Connect from If node’s true branch.  
   - Position: e.g., (1000, -100)

5. **Add OpenAI Node for AI Enhancement**  
   - Type: `@n8n/n8n-nodes-langchain.openAi`  
   - Model: `gpt-4o-mini-2024-07-18` (or latest GPT-4 variant)  
   - Messages:  
     - System role: Instructions to correct grammar, punctuation, and structure in Portuguese, preserving original wording and formatting, outputting markdown.  
     - User role: Pass transcript content from HTTP Request node (`{{$json.content}}`).  
   - Credentials: Configure OpenAI API key.  
   - On Error: Set to `continueErrorOutput`.  
   - Input: Connect from HTTP Request node success output.  
   - Position: e.g., (1460, 20)

6. **Add Set Node to Prepare Success Response**  
   - Type: `Set`  
   - Assign field `text` to `{{$json.message.content}}` (OpenAI output).  
   - Input: Connect from OpenAI node success output.  
   - Position: e.g., (1900, -320)

7. **Add Set Node to Prepare AI Error Response**  
   - Type: `Set`  
   - Assign field `text` to static string: "Something went wrong with the data structuring."  
   - Input: Connect from OpenAI node error output.  
   - Position: e.g., (1900, 100)

8. **Add Set Node to Prepare HTTP Request Error Response**  
   - Type: `Set`  
   - Assign field `text` to concatenation: `{{$json.error}} - {{$json.message}}`  
   - Input: Connect from HTTP Request node error output.  
   - Position: e.g., (1000, 180)

9. **Add Respond to Webhook Node to Send Final Response**  
   - Type: `Respond to Webhook`  
   - Respond with: Text  
   - Response body: `{{$json.text}}`  
   - Retry on fail: Enabled  
   - Input:  
     - Connect from If node false branch (invalid URL).  
     - Connect from all Set nodes (error or success responses).  
   - Position: e.g., (600, 60)

10. **Connect Nodes According to Logic:**  
    - Chat Trigger → Code → If  
    - If (true) → HTTP Request → OpenAI → Set (success) → Respond to Webhook  
    - HTTP Request (error) → Set (HTTP error) → Respond to Webhook  
    - OpenAI (error) → Set (AI error) → Respond to Webhook  
    - If (false) → Respond to Webhook (invalid URL message)

11. **Configure Credentials:**  
    - Supadata API key credential for HTTP Request node header.  
    - OpenAI API key credential for OpenAI node.

12. **Optional:** Add Sticky Notes for documentation and clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                            | Context or Link                                                                                  |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow is designed to simplify access to YouTube video content by converting it into clear and concise transcriptions, ideal for users seeking practicality and efficiency in studying or research.                                               | See Sticky Note5 content in workflow.                                                          |
| Supadata is a powerful transcription API that extracts captions from YouTube videos, enabling structured data retrieval for AI training or text-based applications. API key and language customization are required for proper use.                      | https://supadata.ai                                                                             |
| OpenAI is used here to enhance the raw transcript by correcting grammar and punctuation and formatting the text in markdown, maintaining the original language (Portuguese by default). API key configuration is mandatory.                             | https://platform.openai.com/api-keys                                                           |
| Users should customize the language parameter in both the Supadata API call and the OpenAI prompt to support transcription in other languages as needed.                                                                                                | Workflow description and Sticky Notes.                                                         |
| Limitations include dependency on video captions availability, video length constraints, access restrictions (private or region-locked videos), and processing time variability.                                                                         | Workflow description.                                                                           |
| For more information on n8n nodes and LangChain integration, refer to official n8n documentation and LangChain resources.                                                                                                                              | https://docs.n8n.io/ and https://js.langchain.com/docs/                                        |
| The workflow uses retry mechanisms and error continuation to provide robust user feedback and avoid silent failures during API calls.                                                                                                                  | Error handling nodes configuration.                                                            |

---

This completes the comprehensive reference documentation for the "YouTube Video Transcriber" workflow.