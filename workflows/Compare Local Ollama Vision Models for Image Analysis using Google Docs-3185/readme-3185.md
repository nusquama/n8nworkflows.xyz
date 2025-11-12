Compare Local Ollama Vision Models for Image Analysis using Google Docs

https://n8nworkflows.xyz/workflows/compare-local-ollama-vision-models-for-image-analysis-using-google-docs-3185


# Compare Local Ollama Vision Models for Image Analysis using Google Docs

### 1. Workflow Overview

This workflow is designed to analyze images by leveraging multiple locally hosted Ollama Vision Language Models and then save the detailed analysis results into a Google Docs document. It is targeted at developers, data analysts, marketers, and AI enthusiasts who require comprehensive image descriptions, contextual insights, and structured data extraction from images.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Image Retrieval**: Triggering the workflow manually and downloading the target image from Google Drive.
- **1.2 Image Preparation**: Converting the downloaded image into a Base64 string suitable for sending to the Ollama models.
- **1.3 Model List Setup**: Defining and splitting the list of local Ollama Vision Models to be used for image analysis.
- **1.4 Prompt Preparation**: Creating detailed prompts that instruct the Ollama models on how to analyze the image.
- **1.5 Request Construction & Execution**: Building the HTTP request body for each model and sending the request to the local Ollama API.
- **1.6 Result Processing & Storage**: Formatting the responses and saving the markdown-formatted image descriptions into a Google Docs file.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Image Retrieval

**Overview:**  
This block initiates the workflow manually and downloads the image file from Google Drive based on a provided file ID.

**Nodes Involved:**  
- When clicking ‘Test workflow’  
- Google Doc Image Id  
- Download Image File from Google Drive  
- Get Base64 String

**Node Details:**

- **When clicking ‘Test workflow’**  
  - *Type:* Manual Trigger  
  - *Role:* Starts the workflow on user command.  
  - *Connections:* Outputs to "Google Doc Image Id".  
  - *Failure Modes:* None typical; manual trigger.  

- **Google Doc Image Id**  
  - *Type:* Set  
  - *Role:* Stores the Google Drive file ID of the image to download.  
  - *Configuration:* Static string assignment with placeholder `[your-google-id]` to be replaced by the user.  
  - *Connections:* Outputs to "Download Image File from Google Drive".  
  - *Failure Modes:* Missing or incorrect file ID will cause download failure.  

- **Download Image File from Google Drive**  
  - *Type:* Google Drive node (download operation)  
  - *Role:* Downloads the image file binary data from Google Drive using the provided file ID.  
  - *Configuration:* Uses OAuth2 credentials for Google Drive access.  
  - *Connections:* Outputs binary data to "Get Base64 String".  
  - *Failure Modes:* Authentication errors, file not found, permission issues, or network errors.  

- **Get Base64 String**  
  - *Type:* Extract From File  
  - *Role:* Converts the downloaded binary image file into a Base64-encoded string stored in JSON property for further use.  
  - *Configuration:* Operation set to "binaryToProperty".  
  - *Connections:* Outputs to "List of Vision Models".  
  - *Failure Modes:* Binary data missing or corrupt input.

---

#### 1.2 Model List Setup

**Overview:**  
Defines the list of local Ollama Vision Models to be used for image analysis and splits this list to process each model individually.

**Nodes Involved:**  
- List of Vision Models  
- Split List of Vision Models for Looping

**Node Details:**

- **List of Vision Models**  
  - *Type:* Set  
  - *Role:* Defines an array of model names (e.g., "granite3.2-vision", "llama3.2-vision", "gemma3:27b").  
  - *Configuration:* Static array assignment.  
  - *Connections:* Outputs to "Split List of Vision Models for Looping".  
  - *Failure Modes:* Empty or invalid model names will cause request failures downstream.  

- **Split List of Vision Models for Looping**  
  - *Type:* Split Out  
  - *Role:* Splits the array of models into individual items for batch processing.  
  - *Configuration:* Splits on the "models" field.  
  - *Connections:* Outputs each model item to "Loop Over Ollama Models".  
  - *Failure Modes:* Empty input array leads to no processing.

---

#### 1.3 Prompt Preparation

**Overview:**  
Prepares the detailed user prompt instructing the Ollama models on how to analyze the image exhaustively.

**Nodes Involved:**  
- General Image Prompt

**Node Details:**

- **General Image Prompt**  
  - *Type:* Set  
  - *Role:* Sets a detailed markdown prompt covering comprehensive inventory, contextual analysis, spatial relationships, and textual elements extraction.  
  - *Configuration:* Static multi-line string with structured instructions for the AI model.  
  - *Connections:* Outputs to "Create Request Body".  
  - *Failure Modes:* Improper formatting or missing prompt text may reduce analysis quality.

---

#### 1.4 Request Construction & Execution

**Overview:**  
Constructs the JSON request body for the Ollama API and sends the HTTP POST request to the local Ollama instance for each model.

**Nodes Involved:**  
- Loop Over Ollama Models  
- Create Request Body  
- Ollama LLM Request

**Node Details:**

- **Loop Over Ollama Models**  
  - *Type:* Split In Batches  
  - *Role:* Iterates over each model from the split list to process sequentially or in batches.  
  - *Configuration:* Default batch size (1).  
  - *Connections:* Outputs to "Create Result Objects" and "General Image Prompt".  
  - *Failure Modes:* Batch processing errors or empty input.  

- **Create Request Body**  
  - *Type:* Set  
  - *Role:* Builds the JSON body for the Ollama API request, embedding the current model name, user prompt, and Base64 image string.  
  - *Configuration:* Uses expressions to dynamically insert model name (`{{ $json.models }}`), prompt (`{{ $json.user_prompt }}`), and image data (`{{ $('List of Vision Models').item.json.data }}`).  
  - *Connections:* Outputs to "Ollama LLM Request".  
  - *Failure Modes:* Expression errors, malformed JSON, or missing data cause request failures.  

- **Ollama LLM Request**  
  - *Type:* HTTP Request  
  - *Role:* Sends POST request to local Ollama API endpoint (`http://127.0.0.1:11434/api/chat`) with the constructed JSON body.  
  - *Configuration:* Content-Type header set to `application/json`, body sent as JSON, no streaming.  
  - *Connections:* Outputs to "Loop Over Ollama Models" (for next iteration).  
  - *Failure Modes:* Connection refused if Ollama server is down, timeout, invalid model name, or malformed request.

---

#### 1.5 Result Processing & Storage

**Overview:**  
Formats the response from each Ollama model and appends the markdown content into a Google Docs document for collaborative review.

**Nodes Involved:**  
- Create Result Objects  
- Save Image Descriptions to Google Docs

**Node Details:**

- **Create Result Objects**  
  - *Type:* Set  
  - *Role:* Wraps the response JSON into a "result" object for easier handling downstream.  
  - *Configuration:* Assigns the entire JSON response to a new property "result".  
  - *Connections:* Outputs to "Save Image Descriptions to Google Docs".  
  - *Failure Modes:* Missing or malformed response data.  

- **Save Image Descriptions to Google Docs**  
  - *Type:* Google Docs (update operation)  
  - *Role:* Inserts the markdown-formatted analysis text into a specified Google Docs document.  
  - *Configuration:* Uses OAuth2 credentials for Google Docs; inserts text with model name as a header and content below. Document ID must be specified by the user.  
  - *Connections:* Terminal node.  
  - *Failure Modes:* Authentication errors, invalid document ID, API quota limits, or malformed markdown.

---

### 3. Summary Table

| Node Name                          | Node Type               | Functional Role                                   | Input Node(s)                      | Output Node(s)                      | Sticky Note                                                                                      |
|-----------------------------------|-------------------------|-------------------------------------------------|----------------------------------|-----------------------------------|------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’      | Manual Trigger          | Starts the workflow manually                      |                                  | Google Doc Image Id                | ## 👍Try Me!                                                                                   |
| Google Doc Image Id                | Set                     | Stores Google Drive image file ID                 | When clicking ‘Test workflow’     | Download Image File from Google Drive |                                                                                                |
| Download Image File from Google Drive | Google Drive            | Downloads image file from Google Drive            | Google Doc Image Id               | Get Base64 String                 | ## ⬇️Download Image from Google Drive                                                        |
| Get Base64 String                 | Extract From File        | Converts binary image to Base64 string            | Download Image File from Google Drive | List of Vision Models             |                                                                                                |
| List of Vision Models             | Set                     | Defines array of Ollama Vision Models             | Get Base64 String                | Split List of Vision Models for Looping | ## 📜Create List of Local Ollama Vision Models                                               |
| Split List of Vision Models for Looping | Split Out               | Splits model list into individual items           | List of Vision Models            | Loop Over Ollama Models           |                                                                                                |
| Loop Over Ollama Models           | Split In Batches        | Iterates over each model for processing           | Split List of Vision Models for Looping | Create Result Objects, General Image Prompt | ## 🦙👁️👁️ Process Image with Ollama Vision Models and Save Results to Google Drive           |
| General Image Prompt              | Set                     | Sets detailed analysis prompt for image           | Loop Over Ollama Models          | Create Request Body               |                                                                                                |
| Create Request Body              | Set                     | Builds JSON request body for Ollama API           | General Image Prompt             | Ollama LLM Request               |                                                                                                |
| Ollama LLM Request               | HTTP Request            | Sends image analysis request to local Ollama API | Create Request Body              | Loop Over Ollama Models           | ## 👁️ Analyze Image with Local Ollama LLM                                                    |
| Create Result Objects            | Set                     | Wraps Ollama response into result object          | Loop Over Ollama Models          | Save Image Descriptions to Google Docs |                                                                                                |
| Save Image Descriptions to Google Docs | Google Docs             | Inserts markdown analysis into Google Docs        | Create Result Objects            |                                   |                                                                                                |
| Sticky Note1                    | Sticky Note             | Visual note: "👁️ Analyze Image with Local Ollama LLM" |                                  |                                   | ## 👁️ Analyze Image with Local Ollama LLM                                                    |
| Sticky Note2                    | Sticky Note             | Visual note: "👍Try Me!"                           |                                  |                                   | ## 👍Try Me!                                                                                   |
| Sticky Note3                    | Sticky Note             | Visual note: "📜Create List of Local Ollama Vision Models" |                                  |                                   | ## 📜Create List of Local Ollama Vision Models                                               |
| Sticky Note4                    | Sticky Note             | Visual note: "🦙👁️👁️ Process Image with Ollama Vision Models and Save Results to Google Drive" |                                  |                                   | ## 🦙👁️👁️ Process Image with Ollama Vision Models and Save Results to Google Drive           |
| Sticky Note5                    | Sticky Note             | Visual note: Full workflow description and instructions |                                  |                                   | ## 🦙👁️👁️ Find the Best Local Ollama Vision Models for Your Use Case (full workflow overview) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: To start the workflow manually.

2. **Create Set Node "Google Doc Image Id"**  
   - Type: Set  
   - Purpose: Store the Google Drive file ID of the image to analyze.  
   - Configuration: Add a string field named `id` with value `[your-google-id]` (replace with actual file ID).  
   - Connect output from Manual Trigger.

3. **Create Google Drive Node "Download Image File from Google Drive"**  
   - Type: Google Drive (Operation: Download)  
   - Purpose: Download the image file using the ID from previous node.  
   - Configuration: Set `fileId` to `={{ $json.id }}`.  
   - Credentials: Configure Google Drive OAuth2 credentials.  
   - Connect input from "Google Doc Image Id".

4. **Create Extract From File Node "Get Base64 String"**  
   - Type: Extract From File (Operation: binaryToProperty)  
   - Purpose: Convert downloaded binary image to Base64 string in JSON property.  
   - Connect input from "Download Image File from Google Drive".

5. **Create Set Node "List of Vision Models"**  
   - Type: Set  
   - Purpose: Define array of Ollama Vision Models to use.  
   - Configuration: Add array field `models` with values: `["granite3.2-vision","llama3.2-vision","gemma3:27b"]`.  
   - Connect input from "Get Base64 String".

6. **Create Split Out Node "Split List of Vision Models for Looping"**  
   - Type: Split Out  
   - Purpose: Split the models array into individual items for looping.  
   - Configuration: Field to split out: `models`.  
   - Connect input from "List of Vision Models".

7. **Create Split In Batches Node "Loop Over Ollama Models"**  
   - Type: Split In Batches  
   - Purpose: Process each model one by one.  
   - Connect input from "Split List of Vision Models for Looping".

8. **Create Set Node "General Image Prompt"**  
   - Type: Set  
   - Purpose: Define detailed user prompt for image analysis.  
   - Configuration: Add string field `user_prompt` with the multi-line markdown prompt instructing exhaustive image analysis (copy from workflow description).  
   - Connect input from "Loop Over Ollama Models".

9. **Create Set Node "Create Request Body"**  
   - Type: Set  
   - Purpose: Build JSON request body for Ollama API.  
   - Configuration: Add string field `body` with JSON structure embedding:  
     - `"model": "{{ $json.models }}"`  
     - `"messages": [{"role": "user", "content": "{{ $json.user_prompt }}", "images": ["{{ $('List of Vision Models').item.json.data }}"]}]`  
     - `"stream": false`  
   - Connect input from "General Image Prompt".

10. **Create HTTP Request Node "Ollama LLM Request"**  
    - Type: HTTP Request  
    - Purpose: Send POST request to local Ollama API.  
    - Configuration:  
      - URL: `http://127.0.0.1:11434/api/chat`  
      - Method: POST  
      - Headers: Content-Type: application/json  
      - Body: Use expression `={{ $json.body }}` as JSON  
      - Send body and headers enabled  
    - Connect input from "Create Request Body".

11. **Connect "Ollama LLM Request" output back to "Loop Over Ollama Models"**  
    - To continue batch processing.

12. **Create Set Node "Create Result Objects"**  
    - Type: Set  
    - Purpose: Wrap the response JSON into a `result` object.  
    - Configuration: Assign `result` = `={{ $json }}`.  
    - Connect input from "Loop Over Ollama Models".

13. **Create Google Docs Node "Save Image Descriptions to Google Docs"**  
    - Type: Google Docs (Operation: Update)  
    - Purpose: Insert markdown analysis into Google Docs document.  
    - Configuration:  
      - Document URL or ID: `[your-google-doc-id]` (replace with actual document ID)  
      - Action: Insert text with expression:  
        ```
        <{{ $json.result.model }}>
        {{ $json.result.message.content }}
        </{{ $json.result.model }}>

        ```  
    - Credentials: Configure Google Docs OAuth2 credentials.  
    - Connect input from "Create Result Objects".

14. **Add Sticky Notes** (optional for clarity)  
    - Add notes describing each major block or node as per workflow.

15. **Test the Workflow**  
    - Replace placeholders `[your-google-id]` and `[your-google-doc-id]` with actual IDs.  
    - Ensure local Ollama server is running with the specified models pulled.  
    - Run the workflow manually and verify Google Docs updates.

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                  |
|-------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| Ensure you have access to a local instance of Ollama Vision Models: https://ollama.com/                                             | Official Ollama website                          |
| Pull the Ollama vision models locally before running the workflow to avoid request failures.                                        | Ollama CLI or UI                                 |
| Configure Google Drive and Google Docs OAuth2 credentials in n8n prior to execution.                                                | n8n credential setup documentation               |
| Customize the prompt in "General Image Prompt" node to tailor image analysis to your specific domain or use case.                  | Prompt engineering best practices                |
| Replace Google Drive image source with other providers (e.g., AWS S3, Dropbox) by modifying the download node accordingly.         | Integration flexibility                           |
| Add post-processing or integration nodes (e.g., Slack, HubSpot) after Google Docs node for extended automation.                    | Workflow extensibility                            |
| The workflow outputs markdown-formatted text for easy readability and documentation in Google Docs.                                | Markdown formatting benefits                      |

---

This documentation provides a comprehensive understanding of the workflow structure, node configurations, and instructions to reproduce or modify it. It also highlights potential failure points and integration considerations for robust operation.