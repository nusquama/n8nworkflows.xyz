Generate AI Images from Text Prompts with Gemini 2.0, Google Sheets & Drive

https://n8nworkflows.xyz/workflows/generate-ai-images-from-text-prompts-with-gemini-2-0--google-sheets---drive-7544


# Generate AI Images from Text Prompts with Gemini 2.0, Google Sheets & Drive

### 1. Workflow Overview

This workflow automates the generation of AI-created images from text prompts using Google Gemini 2.0, integrated with Google Sheets and Google Drive. It targets scenarios where users maintain a list of image generation requests in a Google Sheet, and the workflow processes pending requests, generates detailed AI prompts, creates images, uploads them to Google Drive, shares the files publicly, and updates the sheet with the image URLs and status.

The workflow logic is grouped into the following blocks:

- **1.1 Input Reception:** Periodically triggers the workflow and fetches rows from a Google Sheet containing image generation requests.
- **1.2 Filtering Pending Requests:** Filters rows whose status is "pending" to process only new or unprocessed prompts.
- **1.3 AI Prompt Generation:** Uses Google Gemini AI models via LangChain to transform simple titles into detailed, vivid image generation prompts following specific artistic and formatting rules.
- **1.4 Image Generation:** Generates images based on the detailed AI prompts using Google Gemini's image generation model.
- **1.5 Upload & Share:** Uploads the generated image files to a specified Google Drive folder and sets public sharing permissions.
- **1.6 Sheet Update:** Updates the original Google Sheet row with the new image URL and changes the status to "posted."

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Periodically triggers the workflow to start processing and fetches all rows from the specified Google Sheet.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Get row(s) in sheet

- **Node Details:**

  - **Schedule Trigger**  
    - *Type & Role:* Time-based trigger node that initiates workflow execution every minute.  
    - *Configuration:* Interval set to 1 minute.  
    - *Input/Output:* No input; outputs trigger signal to "Get row(s) in sheet".  
    - *Edge Cases:* Workflow may trigger when no new data exists, causing no rows to process.

  - **Get row(s) in sheet**  
    - *Type & Role:* Reads data rows from a Google Sheet.  
    - *Configuration:*  
      - Document ID: Points to the "n8n wallpaper" Google Sheet.  
      - Sheet Name: "Sheet1" (gid=0).  
    - *Credentials:* Uses Google Sheets OAuth2.  
    - *Input/Output:* Takes trigger from Schedule Trigger; outputs all rows to the "If" node.  
    - *Edge Cases:* Google Sheets API quota limits; sheet changes may affect data structure.

#### 1.2 Filtering Pending Requests

- **Overview:**  
  Filters out rows that are not marked as "pending" in their status field, ensuring only new requests proceed.

- **Nodes Involved:**  
  - If  
  - Limit

- **Node Details:**

  - **If**  
    - *Type & Role:* Conditional filtering node that passes only rows where `status == "pending"`.  
    - *Configuration:*  
      - Condition: `$json.status` equals "pending" (case-sensitive, strict type).  
    - *Input/Output:* Input from "Get row(s) in sheet"; outputs filtered rows to "Limit".  
    - *Edge Cases:* Rows without a status field or with unexpected values are excluded.

  - **Limit**  
    - *Type & Role:* Limits the number of items passed downstream; defaults to no limit (empty configuration).  
    - *Configuration:* No explicit limit set, so all filtered rows pass through.  
    - *Input/Output:* Input from "If"; outputs to "AI Agent".  
    - *Edge Cases:* If a limit were set or needed, could prevent excessive processing.

#### 1.3 AI Prompt Generation

- **Overview:**  
  Converts simple input titles to detailed, photorealistic image generation prompts using a custom AI agent powered by Google Gemini via LangChain integration.

- **Nodes Involved:**  
  - AI Agent  
  - Structured Output Parser  
  - Google Gemini Chat Model  
  - Code

- **Node Details:**

  - **Google Gemini Chat Model**  
    - *Type & Role:* Language model node providing the AI backend (Google Gemini PaLM API).  
    - *Configuration:* Default options; credentials configured with Google Palm API account.  
    - *Input/Output:* Connected as AI language model to "AI Agent".  
    - *Edge Cases:* API rate limits, authentication errors, or service downtime.

  - **AI Agent**  
    - *Type & Role:* LangChain agent node that defines the prompt writing logic.  
    - *Configuration:*  
      - Prompt template instructs the AI to produce a JSON object containing "title" and "post" keys.  
      - Specific formatting rules include: single JSON object, escape sequences, no double quotes inside strings, no asterisks, and a fixed "Aspect Ratio: 9:16".  
      - Input: Simple title from the Google Sheet (`{{ $json['Input Title'] }}`).  
    - *Input/Output:* Input from "Limit", uses Google Gemini Chat Model as language model, outputs raw AI response.  
    - *Edge Cases:* AI output may not follow strict JSON formatting due to model variability; requires parsing.

  - **Structured Output Parser**  
    - *Type & Role:* Parses AI output to enforce adherence to the defined JSON schema with "title" and "post" string properties.  
    - *Configuration:* Uses manual JSON schema to validate AI output.  
    - *Input/Output:* Connected to "AI Agent" output parser input; outputs parsed JSON to "Code" node.  
    - *Edge Cases:* Parsing failures if AI output format is incorrect or malformed.

  - **Code**  
    - *Type & Role:* JavaScript node that processes the AI-generated prompt string to escape newline characters properly for downstream usage.  
    - *Configuration:*  
      - Replaces newline characters (`\n`) in the `post` field with escaped newlines (`\\n`).  
    - *Input/Output:* Input from "AI Agent"; outputs to "Generate Image with Gemini".  
    - *Edge Cases:* Unexpected data structure in AI output may cause runtime errors.

#### 1.4 Image Generation

- **Overview:**  
  Generates an image using the detailed prompt text from the AI agent via Google Gemini's image generation model.

- **Nodes Involved:**  
  - Generate Image with Gemini

- **Node Details:**

  - **Generate Image with Gemini**  
    - *Type & Role:* LangChain node for Google Gemini image generation.  
    - *Configuration:*  
      - Model ID: `models/gemini-2.0-flash-exp-image-generation`.  
      - Prompt: Uses the escaped `post` text from the AI output to create a high-quality social media image.  
      - Resource type: image generation.  
    - *Credentials:* Google Palm API account.  
    - *Input/Output:* Input from "Code" node; outputs binary image data to "Upload file".  
    - *Edge Cases:* API limits, prompt length constraints, or malformed prompts might cause generation failure.

#### 1.5 Upload & Share

- **Overview:**  
  Uploads the generated image to Google Drive inside a specific folder and sets the file permissions to public viewing.

- **Nodes Involved:**  
  - Upload file  
  - Share file

- **Node Details:**

  - **Upload file**  
    - *Type & Role:* Google Drive node to upload binary files.  
    - *Configuration:*  
      - File name: `w_{{ ID }}.extension` where ID is from the Google Sheet row, and file extension defaults to "png" if not detected.  
      - Drive: "My Drive".  
      - Folder ID: Specific folder "wallpaper images" identified by folder ID.  
    - *Credentials:* Google Drive OAuth2.  
    - *Input/Output:* Input from "Generate Image with Gemini"; outputs file metadata including ID and links to "Share file".  
    - *Edge Cases:* Upload failure due to quota, permissions, or file size limits.

  - **Share file**  
    - *Type & Role:* Google Drive node that sets file permissions.  
    - *Configuration:*  
      - Operation: Share file.  
      - Permissions: Role "reader", type "anyone" (publicly accessible).  
      - File ID: From uploaded file metadata.  
    - *Credentials:* Google Drive OAuth2.  
    - *Input/Output:* Input from "Upload file"; outputs to "update imageUrl".  
    - *Edge Cases:* Permission API errors, or sharing restrictions on the folder/account.

#### 1.6 Sheet Update

- **Overview:**  
  Updates the status and image URL in the original Google Sheet row corresponding to the processed request.

- **Nodes Involved:**  
  - update imageUrl

- **Node Details:**

  - **update imageUrl**  
    - *Type & Role:* Google Sheets node to update a specific row.  
    - *Configuration:*  
      - Document ID and Sheet Name same as input sheet.  
      - Matching on "ID" column to find the correct row.  
      - Updates columns:  
        - `status` to "posted"  
        - `imageUrl` to the Google Drive webViewLink of uploaded image.  
    - *Credentials:* Google Sheets OAuth2.  
    - *Input/Output:* Input from "Share file". No downstream nodes.  
    - *Edge Cases:* Matching failures if ID is missing or changed; API quota limits.

---

### 3. Summary Table

| Node Name                | Node Type                         | Functional Role                           | Input Node(s)             | Output Node(s)            | Sticky Note                                      |
|--------------------------|----------------------------------|-----------------------------------------|---------------------------|---------------------------|-------------------------------------------------|
| Schedule Trigger         | Schedule Trigger                  | Periodic workflow initiation            |                           | Get row(s) in sheet       |                                                 |
| Get row(s) in sheet      | Google Sheets                    | Fetches rows from Google Sheet           | Schedule Trigger           | If                        |                                                 |
| If                       | If                              | Filters rows with status "pending"       | Get row(s) in sheet        | Limit                     |                                                 |
| Limit                    | Limit                           | Limits number of rows to process         | If                        | AI Agent                  |                                                 |
| AI Agent                 | LangChain Agent                 | Generates detailed AI prompt from title | Limit, Google Gemini Chat Model, Structured Output Parser | Code                      |                                                 |
| Google Gemini Chat Model | LangChain LM Chat Model          | Provides AI language model                |                           | AI Agent (as languageModel) |                                                 |
| Structured Output Parser | LangChain Output Parser Structured | Parses AI output JSON                     | AI Agent                   | AI Agent (outputParser)   |                                                 |
| Code                     | Code                            | Escapes newlines in AI prompt string     | AI Agent                   | Generate Image with Gemini |                                                 |
| Generate Image with Gemini| LangChain Google Gemini Image    | Generates images from prompt              | Code                      | Upload file               |                                                 |
| Upload file              | Google Drive                    | Uploads generated image to Drive          | Generate Image with Gemini | Share file                |                                                 |
| Share file               | Google Drive                    | Shares uploaded file publicly             | Upload file                | update imageUrl           |                                                 |
| update imageUrl          | Google Sheets                   | Updates row status and image URL          | Share file                 |                           |                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Schedule Trigger" node:**  
   - Type: Schedule Trigger  
   - Configure to run every 1 minute (interval: minutes=1)  
   - No credentials needed.

2. **Create "Get row(s) in sheet" node:**  
   - Type: Google Sheets  
   - Credentials: Google Sheets OAuth2 account  
   - Document ID: Select your target Google Sheet (e.g., "n8n wallpaper")  
   - Sheet Name: "Sheet1" (gid=0)  
   - Connect output of Schedule Trigger to this node.

3. **Create "If" node:**  
   - Type: If  
   - Condition: `$json.status` equals string "pending" (case sensitive, strict)  
   - Connect output of "Get row(s) in sheet" to this node.

4. **Create "Limit" node:**  
   - Type: Limit  
   - Leave configuration empty (no limit)  
   - Connect output of "If" (true branch) to this node.

5. **Create "Google Gemini Chat Model" node:**  
   - Type: LangChain LM Chat Model (Google Gemini)  
   - Credentials: Google Palm API account  
   - Default options, no extra config.  
   - No input connection; this node serves as language model backend.

6. **Create "Structured Output Parser" node:**  
   - Type: LangChain Output Parser Structured  
   - Schema Type: Manual  
   - Input Schema:  
     ```json
     {
       "type": "object",
       "properties": {
         "title": { "type": "string" },
         "post": { "type": "string" }
       },
       "required": ["title", "post"],
       "additionalProperties": false
     }
     ```  
   - No input connection; used to parse AI Agent output.

7. **Create "AI Agent" node:**  
   - Type: LangChain Agent  
   - Prompt: Use the supplied detailed prompt instructions:
     - Input variable: `{{ $json['Input Title'] }}` from Google Sheet row  
     - Output: JSON with keys "title" and "post"  
     - Formatting rules: single JSON object, escape sequences, no double quotes inside strings, no asterisks, end with "Aspect Ratio: 9:16"  
   - Connect input from "Limit" node (main input).  
   - Link "Google Gemini Chat Model" node as AI language model input.  
   - Link "Structured Output Parser" node as AI output parser.  
   - No other special options.

8. **Create "Code" node:**  
   - Type: Code (JavaScript)  
   - Code:
     ```javascript
     const items = $input.all();
     const updatedItems = items.map((item) => {
       if (
         item.json &&
         item.json.output &&
         typeof item.json.output.post === "string"
       ) {
         item.json.output.post = item.json.output.post.replace(/\n/g, "\\n");
       }
       return item;
     });
     return updatedItems;
     ```
   - Connect input from "AI Agent" node.

9. **Create "Generate Image with Gemini" node:**  
   - Type: LangChain Google Gemini Image  
   - Credentials: Google Palm API account  
   - Model ID: `models/gemini-2.0-flash-exp-image-generation`  
   - Prompt:  
     ```
     Create a high-quality, visually engaging image for a social media post based on the following text:

     "{{ $json.output.post }}"
     ```  
   - Connect input from "Code" node.

10. **Create "Upload file" node:**  
    - Type: Google Drive  
    - Credentials: Google Drive OAuth2 account  
    - Operation: Upload file (default)  
    - File Name:  
      ```
      w_{{ $('Get row(s) in sheet').item.json.ID }}.{{$binary.data.fileExtension || 'png'}}
      ```  
    - Drive ID: "My Drive"  
    - Folder ID: Select or enter folder ID for "wallpaper images" folder  
    - Connect input from "Generate Image with Gemini" node.

11. **Create "Share file" node:**  
    - Type: Google Drive  
    - Credentials: Google Drive OAuth2 account  
    - Operation: Share file  
    - File ID: Set to `={{ $json.id }}` from "Upload file" output  
    - Permissions: Role - "reader", Type - "anyone" (public)  
    - Connect input from "Upload file" node.

12. **Create "update imageUrl" node:**  
    - Type: Google Sheets  
    - Credentials: Google Sheets OAuth2 account  
    - Operation: Update  
    - Document ID and Sheet Name same as in the input sheet.  
    - Matching Columns: `ID`  
    - Columns to update:  
      - `status`: set to `"posted"`  
      - `imageUrl`: set to `{{$node["Upload file"].json.webViewLink}}`  
    - Connect input from "Share file" node.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                   |
|---------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| The workflow uses Google Gemini 2.0 for both chat and image generation via LangChain nodes.                   | See https://developers.generativeai.google/ for API details.                                    |
| Google Sheets and Drive credentials must be configured with OAuth2 and appropriate scopes for reading/writing. | OAuth2 setup is required for Google services.                                                   |
| The AI prompt includes strict formatting to ensure JSON compliance and proper escaping to avoid parser errors. | Important to avoid double quotes inside JSON string values and no asterisks usage.               |
| Automating image generation with detailed artistic prompts can be adapted for various domains beyond nail polish. | The prompt template can be customized in the AI Agent node.                                     |
| Public sharing permission on Google Drive files enables embedding or direct linking of generated images.      | Google Drive folder must allow file sharing; check organizational policy if restrictions apply. |

---

**Disclaimer:** The content provided is derived exclusively from an automated workflow created with n8n, an integration and automation tool. The processing strictly abides by current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.