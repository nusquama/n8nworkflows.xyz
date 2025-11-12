Summarize YouTube Transcripts in Any Language with Google Gemini & Google Docs

https://n8nworkflows.xyz/workflows/summarize-youtube-transcripts-in-any-language-with-google-gemini---google-docs-5866


# Summarize YouTube Transcripts in Any Language with Google Gemini & Google Docs

### 1. Workflow Overview

This workflow automates the process of summarizing YouTube video transcripts in any language using Google Gemini (PaLM model) and updating the summary in a Google Document. It is designed for users who want concise summaries of YouTube videos for social media or content sharing, supporting multilingual input and output.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Collects user input via a form containing the YouTube video URL and desired summary language.
- **1.2 Data Mapping:** Assigns form inputs to workflow variables for consistent reference.
- **1.3 Transcript Retrieval:** Calls the YouTube Transcript API to fetch the raw transcript of the video.
- **1.4 Transcript Formatting:** Decodes and cleans the transcript data to prepare it for AI processing.
- **1.5 AI Summary Generation:** Uses Google Gemini through an AI agent node to generate a concise summary in the specified language.
- **1.6 Summary Extraction & Optimization:** Extracts and cleans the summary text from the AI's response.
- **1.7 Google Docs Update:** Inserts the final summary into a specified Google Document for sharing or further editing.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Collects user inputs (`videoUrl` and `language`) via a form trigger node.

- **Nodes Involved:**  
  - On form submission

- **Node Details:**  
  - **On form submission**  
    - *Type:* Form Trigger (webhook-based)  
    - *Configuration:* Form titled "summarize youtube videos from transcript for social media" with two required fields: `videoUrl` (YouTube video URL) and `language` (e.g., English).  
    - *Input/Output:* Receives form data; outputs JSON with form fields to the Mapper node.  
    - *Edge Cases:* Missing or malformed URLs; unsupported language input.  
    - *Sticky Note:* Explains purpose and details of the form submission (Sticky Note1).

#### 2.2 Data Mapping

- **Overview:**  
  Maps and standardizes form inputs for downstream use.

- **Nodes Involved:**  
  - Mapper

- **Node Details:**  
  - *Type:* Set node  
  - *Configuration:* Assigns `videoUrl` and `language` from the form JSON input to node-level variables for ease of access.  
  - *Input/Output:* Receives form data, outputs mapped variables to YouTube Transcript AI node.  
  - *Edge Cases:* Missing values propagate; no validation here.  
  - *Sticky Note:* Clarifies data mapping role (Sticky Note2).

#### 2.3 Transcript Retrieval

- **Overview:**  
  Obtains the YouTube video transcript by invoking a third-party API.

- **Nodes Involved:**  
  - YouTube Transcript AI

- **Node Details:**  
  - *Type:* HTTP Request node  
  - *Configuration:*  
    - POST request to `https://youtube-transcriptor-pro.p.rapidapi.com/yt/index.php`  
    - Sends `videoUrl` as multipart form data  
    - Headers include RapidAPI host and API key (secured credential)  
  - *Input/Output:* Receives `videoUrl`, outputs raw transcript JSON to Formator node.  
  - *Edge Cases:* API failures, invalid video URLs, rate limits, incorrect/missing API key.  
  - *Sticky Note:* Details API call and response (Sticky Note3).

#### 2.4 Transcript Formatting

- **Overview:**  
  Parses, decodes, and validates the raw transcript data for AI consumption.

- **Nodes Involved:**  
  - Formator (code node)

- **Node Details:**  
  - *Type:* Code node (JavaScript)  
  - *Configuration:*  
    - Parses the outer JSON string from API response  
    - Extracts the `data` field containing transcript text  
    - Decodes Unicode escape sequences  
    - Checks for empty or invalid transcripts, returns fallback message if invalid  
  - *Input/Output:* Receives raw API transcript; outputs clean `chatInput` text for AI Agent1.  
  - *Edge Cases:* JSON parse errors, empty transcripts, malformed Unicode sequences.  
  - *Sticky Note:* Explains decoding and validation logic (Sticky Note4).

#### 2.5 AI Summary Generation

- **Overview:**  
  Uses Google Gemini model to generate a language-matched summary of the transcript.

- **Nodes Involved:**  
  - AI Agent1  
  - Google Gemini Chat Model (language model node)

- **Node Details:**  
  - *AI Agent1*  
    - *Type:* Langchain Agent node  
    - *Configuration:*  
      - System message prompt instructs the AI to summarize the transcript in the same language as provided by `language` from Mapper node.  
      - Output format includes a `🎬 **Summary**` section with main points and optional tone/style.  
    - *Input/Output:* Takes formatted transcript (`chatInput`) and language; outputs AI response text.  
    - *Edge Cases:* AI timeout, malformed prompt, language mismatch, rate limits.  
    - *Sub-workflow:* Uses Google Gemini Chat Model node as language model backend.  
    - *Sticky Note:* Describes AI summarization purpose and prompt (Sticky Note5).

  - *Google Gemini Chat Model*  
    - *Type:* Langchain Google Gemini LM node  
    - *Configuration:* Uses Gemini 2.0 Flash model with configured Google PaLM API credentials.  
    - *Input/Output:* Provides language model inference for AI Agent1.  
    - *Edge Cases:* API auth errors, quota limits, model version requirements.

#### 2.6 Summary Extraction & Optimization

- **Overview:**  
  Parses the AI's response to extract and clean the summary text.

- **Nodes Involved:**  
  - Optimizer (code node)

- **Node Details:**  
  - *Type:* Code node (JavaScript)  
  - *Configuration:*  
    - Uses regex to find the section labeled `🎬 **Summary**` in AI output  
    - Extracts summary text stopping at the next delimiter or end of string  
  - *Input/Output:* Takes AI raw output, outputs clean summary text for Google Docs node.  
  - *Edge Cases:* Missing summary label, unexpected AI output format, empty summary.  
  - *Sticky Note:* Explains summary extraction logic (Sticky Note6).

#### 2.7 Google Docs Update

- **Overview:**  
  Updates a Google Document with the extracted summary for user access.

- **Nodes Involved:**  
  - Google Docs

- **Node Details:**  
  - *Type:* Google Docs node  
  - *Configuration:*  
    - Operation: Update document  
    - Inserts the `summary` text into the document (specific URL must be configured)  
    - Uses Google Service Account credentials for authentication  
  - *Input/Output:* Receives clean summary text, updates Google Doc accordingly.  
  - *Edge Cases:* Incorrect document URL, authentication errors, API quota limits.  
  - *Sticky Note:* Details document update process (Sticky Note7).

---

### 3. Summary Table

| Node Name              | Node Type                           | Functional Role                      | Input Node(s)          | Output Node(s)          | Sticky Note                                                                                                                        |
|------------------------|-----------------------------------|------------------------------------|------------------------|-------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| On form submission      | Form Trigger                      | Input reception                    | —                      | Mapper                  | Collects `videoUrl` and `language` from user via form.                                                                            |
| Mapper                 | Set                               | Data mapping                      | On form submission      | YouTube Transcript AI   | Maps form inputs to variables for consistent access.                                                                              |
| YouTube Transcript AI  | HTTP Request                      | Transcript retrieval              | Mapper                 | Formator                | Calls YouTube Transcript API via RapidAPI to fetch video transcript.                                                              |
| Formator               | Code                             | Transcript formatting             | YouTube Transcript AI  | AI Agent1               | Decodes and validates transcript JSON data for AI input.                                                                          |
| AI Agent1              | Langchain Agent                  | AI summary generation             | Formator               | Optimizer               | Uses Google Gemini to generate a summary in the specified language.                                                               |
| Google Gemini Chat Model| Langchain LM Google Gemini       | AI language model backend         | —                      | AI Agent1 (ai_languageModel) | Provides Google Gemini model inference for AI Agent1 node.                                                                        |
| Optimizer              | Code                             | Summary extraction & optimization | AI Agent1               | Google Docs             | Extracts clean summary text from AI response using regex.                                                                          |
| Google Docs            | Google Docs                      | Google Document update            | Optimizer               | —                       | Inserts final summary into configured Google Document.                                                                             |
| Sticky Note            | Sticky Note                      | Documentation                    | —                      | —                       | Provides comprehensive workflow description and overview.                                                                         |
| Sticky Note1           | Sticky Note                      | Documentation                    | —                      | —                       | Details the form submission node.                                                                                                 |
| Sticky Note2           | Sticky Note                      | Documentation                    | —                      | —                       | Details the data mapping node.                                                                                                    |
| Sticky Note3           | Sticky Note                      | Documentation                    | —                      | —                       | Details the YouTube transcript API call node.                                                                                     |
| Sticky Note4           | Sticky Note                      | Documentation                    | —                      | —                       | Details transcript decoding and formatting logic.                                                                                 |
| Sticky Note5           | Sticky Note                      | Documentation                    | —                      | —                       | Details AI summarization node.                                                                                                    |
| Sticky Note6           | Sticky Note                      | Documentation                    | —                      | —                       | Details summary extraction from AI response.                                                                                      |
| Sticky Note7           | Sticky Note                      | Documentation                    | —                      | —                       | Details Google Docs update node.                                                                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node**  
   - Type: Form Trigger  
   - Name: `On form submission`  
   - Configure form title: "summarize youtube videos from transcript for social media"  
   - Add two required fields:  
     - `videoUrl` (placeholder: "full video url")  
     - `language` (placeholder: "English")  
   - This node will start the workflow on form submission.

2. **Create Set Node for Mapping**  
   - Type: Set  
   - Name: `Mapper`  
   - Add two string assignments:  
     - `videoUrl` = `{{$json.videoUrl}}`  
     - `language` = `{{$json.language}}`  
   - Connect `On form submission` → `Mapper`.

3. **Create HTTP Request Node for Transcript Retrieval**  
   - Type: HTTP Request  
   - Name: `YouTube Transcript AI`  
   - Method: POST  
   - URL: `https://youtube-transcriptor-pro.p.rapidapi.com/yt/index.php`  
   - Content-Type: `multipart/form-data`  
   - Body Parameters:  
     - Name: `videoUrl`, Value: `{{$json.videoUrl}}`  
   - Header Parameters:  
     - `x-rapidapi-host`: `youtube-transcriptor-pro.p.rapidapi.com`  
     - `x-rapidapi-key`: set your RapidAPI key in credentials  
   - Connect `Mapper` → `YouTube Transcript AI`.

4. **Create Code Node for Transcript Formatting**  
   - Type: Code (JavaScript)  
   - Name: `Formator`  
   - Paste the provided JS code that:  
     - Parses JSON string response  
     - Extracts `data` field  
     - Decodes Unicode escape sequences  
     - Returns `chatInput` with cleaned transcript or error message if invalid  
   - Connect `YouTube Transcript AI` → `Formator`.

5. **Create Langchain Agent Node for AI Summary Generation**  
   - Type: Langchain Agent  
   - Name: `AI Agent1`  
   - System message prompt: instruct AI to summarize transcript in `{{$node["Mapper"].json["language"]}}` with requested format (including `🎬 **Summary**` section).  
   - Connect `Formator` → `AI Agent1`.

6. **Create Google Gemini Chat Model Node as AI Backend**  
   - Type: Langchain Google Gemini LM  
   - Name: `Google Gemini Chat Model`  
   - Model: `models/gemini-2.0-flash`  
   - Set Google PaLM API credentials (Service Account or API key)  
   - Connect `Google Gemini Chat Model` as AI language model backend to `AI Agent1` (via ai_languageModel input).

7. **Create Code Node for Summary Extraction & Optimization**  
   - Type: Code (JavaScript)  
   - Name: `Optimizer`  
   - Paste JS code to extract text labeled `🎬 **Summary**` using regex from AI output text.  
   - Connect `AI Agent1` → `Optimizer`.

8. **Create Google Docs Node to Update Document**  
   - Type: Google Docs  
   - Name: `Google Docs`  
   - Operation: Update Document  
   - Authentication: Service Account with Google Docs API enabled  
   - Configure `documentURL` parameter with your target Google Doc URL  
   - Action: Insert text = `{{$json.summary}}`  
   - Connect `Optimizer` → `Google Docs`.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                   | Context or Link                                                                                                 |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| This workflow supports language flexibility, allowing summaries in any language provided in the form.                                                                                                                                           | Core feature                                                                                                  |
| The YouTube Transcript API used is a third-party service via RapidAPI; ensure you have a valid API key and quota.                                                                                                                             | https://rapidapi.com/youtube-transcriptor-pro/api/youtube-transcriptor-pro                                      |
| Google Gemini (PaLM) requires valid Google Cloud credentials with PaLM API access. Setup of Google Service Account credentials is necessary for both Gemini and Google Docs nodes.                                                              | https://cloud.google.com/palm                                                                                  |
| The Google Docs node requires the document URL to be preconfigured. The document should have permissions set to allow the service account to update it.                                                                                        | Google Docs API documentation                                                                                  |
| The AI agent prompt is designed to produce a specific structured summary format with a `🎬 **Summary**` marker to facilitate extraction downstream.                                                                                            | Prompt design best practice                                                                                     |
| The code nodes include robust error handling for JSON parsing and transcript validity, but any upstream API failure may cause incomplete or missing transcripts affecting summary quality.                                                      | Error handling considerations                                                                                   |
| The workflow is disabled by default (`active:false`); enable it after configuring credentials and document URLs.                                                                                                                              | n8n execution setting                                                                                           |
| Sticky notes within the workflow provide detailed insights into each step, useful for debugging and extending the workflow.                                                                                                                    | Embedded within workflow                                                                                         |

---

**Disclaimer:** The content above is derived exclusively from an automated n8n workflow. It complies with all applicable content policies and contains no illegal or protected data. All data handled is public and legal.