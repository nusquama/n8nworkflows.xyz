fluidX THE EYE — Create & invite via SMS for live camera session

https://n8nworkflows.xyz/workflows/fluidx-the-eye---create---invite-via-sms-for-live-camera-session-10724


# fluidX THE EYE — Create & invite via SMS for live camera session

### 1. Workflow Overview

This workflow automates the creation and management of a live camera session using fluidX THE EYE service. It targets scenarios where a customer and a service agent need to join a live video session, sharing their cameras, with automated media handling, notifications, and AI-powered image analysis.

Logical blocks are grouped as follows:

- **1.1 Input Reception:** Receives user phone number and agent email via an n8n form trigger.
- **1.2 Configuration Setup:** Sets static and dynamic session configuration variables.
- **1.3 Session Creation and Folder Setup:** Creates a fluidX THE EYE session and a corresponding Google Drive folder for media storage.
- **1.4 Notification Dispatch:** Sends SMS to the user and email to the agent with session links.
- **1.5 Session Monitoring and Media Polling:** Waits for session activation, polls session status and media references, deduplicates media, and downloads new images.
- **1.6 AI-Based Image Analysis:** Applies OpenAI vision model to analyze each new image and generate descriptive metadata.
- **1.7 Media Upload and Summary Management:** Uploads images and AI-generated captions to Google Drive, generates session summary text files, and uploads summaries while avoiding duplicates.
- **1.8 Cleanup:** Clears cached media IDs to prepare for the next session.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
Collects the phone number of the user (customer) and the email address of the agent via a form submission trigger.

**Nodes Involved:**  
- On form submission

**Node Details:**  
- **On form submission**  
  - Type: Form Trigger  
  - Configuration: Form titled "fluidX digital The Eye Session Invitation" with required fields: "User Phone Number" and "Agent Email Address".  
  - Inputs: HTTP webhook form submission  
  - Outputs: JSON containing user phone and agent email  
  - Edge Cases: Missing required fields; invalid phone/email format handled externally or by form validation  
  - Version: 2.3  

---

#### 1.2 Configuration Setup

**Overview:**  
Sets static and dynamic configuration variables for the session including API base URL, user phone, agent email, language, company/project info, and correlation ID.

**Nodes Involved:**  
- Set Config

**Node Details:**  
- **Set Config**  
  - Type: Set  
  - Role: Defines all session parameters including environment and form inputs.  
  - Key Variables:  
    - `BASE_URL`: fluidX API base URL  
    - `phoneNumberUser`: from form input  
    - `emailAgent`: from form input  
    - `message`: SMS intro text  
    - `language`: for AI captions (default DE)  
    - `company`, `project`, `billingcode`, `sku`: production identifiers  
    - `correlationId`: current timestamp  
  - Input: Form submission data  
  - Output: Configured JSON for downstream nodes  
  - Edge Cases: Missing or malformed form data; time format issues  
  - Version: 2  

---

#### 1.3 Session Creation and Folder Setup

**Overview:**  
Creates a new fluidX session using the API and prepares a dedicated Google Drive folder to store session media.

**Nodes Involved:**  
- fluidX API - Create Session  
- Set Session Vars  
- Create Session Folder

**Node Details:**  
- **fluidX API - Create Session**  
  - Type: HTTP Request (POST)  
  - Role: Creates a live session with fluidX using configured parameters.  
  - URL: `${BASE_URL}/api/fx/ext/session/create`  
  - Body: JSON with company, project, billing code, correlation ID, SKU  
  - Auth: HTTP header using fluidX API key  
  - Timeout: 30s  
  - Inputs: Config JSON  
  - Outputs: Session details (sessionId, URLs, status)  
  - Edge Cases: API errors, timeout, invalid credentials  
  - Version: 3  

- **Set Session Vars**  
  - Type: Set  
  - Role: Extracts and sets session-specific variables from API response for later use, including sessionId, URLs, and folder name with timestamp.  
  - Key Expressions:  
    - `sessionId` from `revXRSession.id`  
    - Folder name format: `"THEEYE_" + timestamp + sessionId`  
  - Inputs: Session creation output  
  - Outputs: Session variables JSON  
  - Version: 2  

- **Create Session Folder**  
  - Type: Google Drive Folder Creation  
  - Role: Creates a folder in Google Drive under a pre-specified parent folder (`THEEYE` root folder) to store session files.  
  - Folder Name: from `Set Session Vars` node  
  - Drive: "My Drive"  
  - Parent Folder ID: fixed ID for THEEYE folder (replace for production)  
  - Credentials: Google Drive OAuth2  
  - Inputs: Session vars  
  - Outputs: Folder metadata (including folder ID)  
  - Edge Cases: Permission issues, API limits  
  - Version: 3  

---

#### 1.4 Notification Dispatch

**Overview:**  
Sends SMS invitation to the user and email invitation to the agent with session access links.

**Nodes Involved:**  
- fluidX API - Send SMS User  
- Send email to Agent  
- Wait1

**Node Details:**  
- **fluidX API - Send SMS User**  
  - Type: HTTP Request (POST)  
  - Role: Sends SMS with session link to the user’s phone number.  
  - URL: `${BASE_URL}/api/fx/ext/sms/send`  
  - Body: JSON with phoneNumber, message (SMS text + session URL), sessionId  
  - Auth: fluidX API key header  
  - Timeout: 30s  
  - Inputs: Config and Session Vars  
  - Outputs: SMS send result  
  - Edge Cases: SMS gateway errors, invalid phone number, API rate limits  
  - Version: 3  

- **Send email to Agent**  
  - Type: Email Send  
  - Role: Sends plaintext email to the agent with the session link.  
  - To: agent email from config  
  - From: fixed sender email `fluidx@titel.ai` (adjust for production)  
  - Subject: fluidX.digital View TheEye Link  
  - Body: Includes session URL from session vars  
  - Credentials: SMTP configured in n8n  
  - Inputs: Config and Session Vars  
  - Outputs: Email send status  
  - Edge Cases: SMTP auth failure, invalid email, spam filtering  
  - Version: 2.1  

- **Wait1**  
  - Type: Wait  
  - Role: Waits 30 seconds after email send to allow session to initialize before polling status.  
  - Inputs: Email send output  
  - Outputs: Trigger for next steps  
  - Version: 1.1  

---

#### 1.5 Session Monitoring and Media Polling

**Overview:**  
Waits for the session to be active, polls session info and media references, filters new photos, and downloads image files.

**Nodes Involved:**  
- fluidX API - Get Session Info  
- If Session Active  
- Extract Media Refs photos  
- If photos  
- Split photoRefs (Item Lists)  
- If Media Id  
- fluidX API - Media Info HTTP Request GET  
- Merge photoRefs  
- If new Image  
- Check Already Uploaded  
- fluidX API - Download Photo THEEYE Session  
- Carry URL and Id

**Node Details:**  
- **fluidX API - Get Session Info**  
  - Type: HTTP Request (GET)  
  - Role: Retrieves current session status and media references.  
  - URL: `${BASE_URL}/api/revxr/ext/session` with sessionId query  
  - Auth: fluidX API key  
  - Timeout: 30s  
  - Inputs: Wait or Wait1 outputs  
  - Outputs: Session info JSON  
  - Edge Cases: API errors, network timeout  
  - Version: 3  

- **If Session Active**  
  - Type: If  
  - Role: Checks if session status is `"ACTIVE"` to continue processing.  
  - Condition: `revXRSession.status == "ACTIVE"`  
  - Inputs: Session info  
  - Outputs: Branch 1 (active) → Extract Media Refs photos; Branch 2 (inactive) → fluidX API - Media Summary (for closed session)  
  - Version: 2.2  

- **Extract Media Refs photos**  
  - Type: Set  
  - Role: Extracts `photoRefs` array from session media info for processing.  
  - Inputs: Session info  
  - Outputs: JSON with `photoRefs` array  
  - Version: 2  

- **If photos**  
  - Type: If  
  - Role: Checks if `photoRefs` array is empty or not.  
  - Inputs: Extract Media Refs photos output  
  - Outputs: Branch 1 (empty) → Wait node, Branch 2 (non-empty) → Split photoRefs (Item Lists)  
  - Version: 2.2  

- **Split photoRefs (Item Lists)**  
  - Type: Item Lists (Splitter)  
  - Role: Splits array of photos to process each photo individually.  
  - Field to split out: `photoRefs`  
  - Inputs: If photos (non-empty) output  
  - Outputs: Individual photo items for further processing  
  - Version: 3  

- **If Media Id**  
  - Type: If  
  - Role: Checks if each photo item contains a valid `id` field (indicates media metadata available).  
  - Condition: `id` field is not empty  
  - Inputs: Split photoRefs output  
  - Outputs: Branch 1 (has id) → fluidX API - Media Info HTTP Request GET; Branch 2 (no id) → Wait  
  - Version: 2.2  

- **fluidX API - Media Info HTTP Request GET**  
  - Type: HTTP Request (GET)  
  - Role: Retrieves detailed media information for a photo by ID.  
  - URL: `${BASE_URL}/api/revxr/ext/media/info?type=image&id=photoId`  
  - Auth: fluidX API key  
  - Timeout: 30s  
  - Inputs: If Media Id branch  
  - Outputs: Media Info JSON  
  - Edge Cases: Missing media info, API errors  
  - Version: 4.2  

- **Merge photoRefs**  
  - Type: Merge  
  - Role: Combines results from media info fetch and items without media info for unified downstream processing.  
  - Mode: Combine by position  
  - Inputs: fluidX API - Media Info HTTP Request GET, If Media Id (no id branch)  
  - Outputs: Combined photoRefs array  
  - Version: 3.2  

- **If new Image**  
  - Type: If  
  - Role: Filters photos by checking if media description exists and other conditions to identify new images needing processing.  
  - Conditions: mediaInfo.description does not exist  
  - Inputs: Merge photoRefs output  
  - Outputs: Branch 1 (new image) → Check Already Uploaded; Branch 2 (existing) → Wait  
  - Version: 2.2  

- **Check Already Uploaded**  
  - Type: Code  
  - Role: Deduplicates images by checking a global static cache of already processed photo IDs and filters only new photos.  
  - Logic: Maintains an array of uploaded photo IDs in workflow static data; skips duplicates  
  - Inputs: If new Image output  
  - Outputs: New photos only  
  - Edge Cases: Cache loss on reboot, concurrency issues  
  - Version: 2  

- **fluidX API - Download Photo THEEYE Session**  
  - Type: HTTP Request (GET)  
  - Role: Downloads the actual image binary data using photoRefs download URL.  
  - URL: photo download URL from JSON  
  - Auth: fluidX API key  
  - Timeout: 60s  
  - Inputs: Check Already Uploaded output  
  - Outputs: Image binary data  
  - Edge Cases: Download failures, timeouts  
  - Version: 3  

- **Carry URL and Id**  
  - Type: Set  
  - Role: Attaches the photo ID and URL to the image data for tracking in downstream analysis and upload.  
  - Inputs: Downloaded photo data  
  - Outputs: Enhanced image data JSON with `_srcId` and `_srcUrl`  
  - Version: 2  

---

#### 1.6 AI-Based Image Analysis

**Overview:**  
Analyzes images using OpenAI vision model to generate captions and metadata describing the image content.

**Nodes Involved:**  
- Analyze image  
- Merge Analyze + URL  
- Build MediaInfo

**Node Details:**  
- **Analyze image**  
  - Type: OpenAI Image Analysis (Langchain node)  
  - Role: Sends base64 image data to OpenAI model for descriptive caption generation in configured language.  
  - Model: chatgpt-4o-latest  
  - Input Type: Base64 image  
  - Prompt: "What's in this image? Output language is {language}"  
  - Credentials: OpenAI API key  
  - OnError: continue output to avoid blocking workflow  
  - Inputs: Carry URL and Id output  
  - Outputs: Analysis text output  
  - Edge Cases: API rate limits, model errors, malformed images  
  - Version: 2  

- **Merge Analyze + URL**  
  - Type: Merge  
  - Role: Combines OpenAI analysis results with image URL and ID information for metadata building.  
  - Mode: Combine by position  
  - Inputs: Analyze image output and Carry URL and Id output  
  - Outputs: Combined analysis + URL data  
  - Version: 3.2  

- **Build MediaInfo**  
  - Type: Set  
  - Role: Constructs media metadata JSON including type, id, AI-generated title, and description.  
  - Title Generation: Extracts meaningful first phrase from AI text, trims to 10 words, capitalizes first letter, with fallbacks.  
  - Inputs: Merge Analyze + URL output  
  - Outputs: MediaInfo JSON for upload  
  - Version: 2  

---

#### 1.7 Media Upload and Summary Management

**Overview:**  
Uploads images and AI metadata to Google Drive, generates textual summaries per photo and session, uploads summary files, and tracks uploaded files to avoid duplicates.

**Nodes Involved:**  
- fluidX API - Media Info HTTP Request POST  
- Generate Photo Summary  
- Convert Text to Binary  
- Upload Photo Summary  
- Upload file  
- Merge  
- Generate Photo Summary  
- Convert Text to Binary (summary)  
- Upload Session Summary  
- Remember Session Summary File  
- Convert Summary Text to Binary  

**Node Details:**  
- **fluidX API - Media Info HTTP Request POST**  
  - Type: HTTP Request (POST)  
  - Role: Posts AI-generated media metadata back to fluidX API to document the session.  
  - URL: `${BASE_URL}/api/revxr/ext/media/info`  
  - Auth: fluidX API key  
  - Inputs: MediaInfo from Build MediaInfo  
  - Outputs: Confirmation JSON  
  - Version: 4.2  

- **Generate Photo Summary**  
  - Type: Code  
  - Role: Creates a textual summary for each photo metadata with title and description in a text file format.  
  - Inputs: MediaInfo from POST response  
  - Outputs: JSON with filename and text content  
  - Version: 2  

- **Convert Text to Binary**  
  - Type: Code  
  - Role: Converts photo summary text into base64-encoded binary for Google Drive upload.  
  - Inputs: Generate Photo Summary output  
  - Outputs: Binary file JSON  
  - Version: 2  

- **Upload Photo Summary**  
  - Type: Google Drive Upload  
  - Role: Uploads photo summary text files into the session folder on Google Drive.  
  - Folder ID: from Create Session Folder node  
  - Inputs: Convert Text to Binary output  
  - Outputs: Upload metadata including file ID  
  - Version: 3  

- **Upload file**  
  - Type: Google Drive Upload  
  - Role: Uploads image files to the session folder.  
  - Folder ID: from Create Session Folder node  
  - Inputs: Carry URL and Id (image binary)  
  - Outputs: Upload metadata  
  - Version: 3  

- **Merge**  
  - Type: Merge  
  - Role: Chooses branch between photo summary upload and image upload outputs for synchronization and further steps.  
  - Mode: Choose Branch  
  - Inputs: Upload file and Upload Photo Summary outputs  
  - Outputs: Unified output stream  
  - Version: 3.2  

- **Convert Summary Text to Binary**  
  - Type: Code  
  - Role: Converts the overall session media summary text (retrieved from API) to a binary file for Google Drive upload.  
  - Logic: Checks global static cache to avoid duplicate summary uploads per session  
  - Inputs: fluidX API - Media Summary output  
  - Outputs: Binary file JSON for upload  
  - Version: 2  

- **Upload Session Summary**  
  - Type: Google Drive Upload  
  - Role: Uploads session summary text file to Google Drive session folder.  
  - Inputs: Convert Summary Text to Binary output  
  - Outputs: Upload metadata  
  - OnError: continue output (non-blocking)  
  - Version: 3  

- **Remember Session Summary File**  
  - Type: Code  
  - Role: Stores the uploaded session summary file ID in global static data per session to prevent re-upload.  
  - Inputs: Upload Session Summary output  
  - Outputs: Pass-through  
  - Version: 2  

---

#### 1.8 Cleanup

**Overview:**  
Clears the cached uploaded photo IDs after session completion to prevent cross-session contamination.

**Nodes Involved:**  
- Clear Uploaded Photo Store  
- The End

**Node Details:**  
- **Clear Uploaded Photo Store**  
  - Type: Code  
  - Role: Empties the global static array storing uploaded photo IDs to reset state for the next session.  
  - Inputs: Remember Session Summary File output  
  - Outputs: Pass-through  
  - Version: 2  

- **The End**  
  - Type: No Operation / Workflow terminator  
  - Role: Marks end of workflow execution  
  - Inputs: Clear Uploaded Photo Store output  
  - Outputs: None  
  - Version: 1  

---

### 3. Summary Table

| Node Name                           | Node Type                     | Functional Role                          | Input Node(s)                        | Output Node(s)                      | Sticky Note                                                                                                  |
|-----------------------------------|-------------------------------|----------------------------------------|------------------------------------|------------------------------------|--------------------------------------------------------------------------------------------------------------|
| 📝 Setup & Instructions             | Sticky Note                   | Provides setup instructions             | -                                  | -                                  | Setup instructions: API key, credentials, form setup, usage tips, and project notes                           |
| On form submission                 | Form Trigger                 | Receives user phone and agent email    | -                                  | Set Config                        |                                                                                                              |
| Set Config                        | Set                          | Sets config variables and session params| On form submission                 | fluidX API - Create Session        |                                                                                                              |
| fluidX API - Create Session       | HTTP Request (POST)           | Creates fluidX THE EYE session          | Set Config                        | Set Session Vars                  |                                                                                                              |
| Set Session Vars                  | Set                          | Extracts session IDs and URLs           | fluidX API - Create Session       | Create Session Folder             |                                                                                                              |
| Create Session Folder             | Google Drive Folder           | Creates Google Drive folder for session | Set Session Vars                  | fluidX API - Send SMS User         |                                                                                                              |
| fluidX API - Send SMS User        | HTTP Request (POST)           | Sends SMS invite to user                | Create Session Folder             | Send email to Agent               |                                                                                                              |
| Send email to Agent               | Email Send                   | Sends email invite to service agent    | fluidX API - Send SMS User        | Wait1                            |                                                                                                              |
| Wait1                            | Wait                         | Waits 30 seconds before polling         | Send email to Agent               | fluidX API - Get Session Info     |                                                                                                              |
| fluidX API - Get Session Info     | HTTP Request (GET)            | Retrieves session status and media refs | Wait1 or Wait                    | If Session Active                 |                                                                                                              |
| If Session Active                | If                           | Checks if session is active              | fluidX API - Get Session Info     | Extract Media Refs photos / fluidX API - Media Summary |                                                                                                              |
| Extract Media Refs photos         | Set                          | Extracts photoRefs array                  | If Session Active                 | If photos                       |                                                                                                              |
| If photos                       | If                           | Checks if photoRefs is empty              | Extract Media Refs photos         | Wait / Split photoRefs (Item Lists) |                                                                                                              |
| Split photoRefs (Item Lists)      | Item Lists                   | Splits photoRefs array into items        | If photos                       | If Media Id / Merge photoRefs      |                                                                                                              |
| If Media Id                     | If                           | Checks if photo has media ID              | Split photoRefs (Item Lists)      | fluidX API - Media Info HTTP Request GET / Wait |                                                                                                              |
| fluidX API - Media Info HTTP Request GET | HTTP Request (GET)            | Gets detailed media info by photo ID    | If Media Id                      | Merge photoRefs                  |                                                                                                              |
| Merge photoRefs                 | Merge                        | Combines media info and no-id photoRefs  | fluidX API - Media Info GET, If Media Id (no id) | If new Image                    |                                                                                                              |
| If new Image                   | If                           | Filters new images without existing desc | Merge photoRefs                 | Check Already Uploaded / Wait      |                                                                                                              |
| Check Already Uploaded           | Code                         | Removes already processed photos          | If new Image                   | fluidX API - Download Photo THEEYE Session |                                                                                                              |
| fluidX API - Download Photo THEEYE Session | HTTP Request (GET)            | Downloads image file from URL             | Check Already Uploaded           | Carry URL and Id                |                                                                                                              |
| Carry URL and Id               | Set                          | Attaches photo ID and URL to image data   | fluidX API - Download Photo THEEYE Session | Analyze image / Upload file / Merge Analyze + URL |                                                                                                              |
| Analyze image                  | OpenAI Image Analysis         | Generates AI captions for images          | Carry URL and Id                | Merge Analyze + URL              |                                                                                                              |
| Merge Analyze + URL            | Merge                        | Combines AI analysis and URL information  | Analyze image / Carry URL and Id | Build MediaInfo                 |                                                                                                              |
| Build MediaInfo                | Set                          | Builds media metadata with AI captions    | Merge Analyze + URL             | fluidX API - Media Info HTTP Request POST |                                                                                                              |
| fluidX API - Media Info HTTP Request POST | HTTP Request (POST)           | Posts AI metadata back to fluidX API      | Build MediaInfo                | Generate Photo Summary          |                                                                                                              |
| Generate Photo Summary          | Code                         | Creates textual summaries for photos       | fluidX API - Media Info POST    | Convert Text to Binary          |                                                                                                              |
| Convert Text to Binary          | Code                         | Converts summary text to binary for upload| Generate Photo Summary          | Upload Photo Summary           |                                                                                                              |
| Upload Photo Summary           | Google Drive Upload           | Uploads photo summaries to Google Drive    | Convert Text to Binary          | Merge                          |                                                                                                              |
| Upload file                   | Google Drive Upload           | Uploads image files to Google Drive         | Carry URL and Id                | Merge                          |                                                                                                              |
| Merge                         | Merge                        | Chooses branch between photo and summary uploads | Upload file / Upload Photo Summary | Wait                           |                                                                                                              |
| Wait                          | Wait                         | Waits 15 seconds between polling cycles    | Merge                         | fluidX API - Get Session Info     |                                                                                                              |
| fluidX API - Media Summary     | HTTP Request (GET)            | Gets final media summary after session ends| If Session Active (inactive branch) | Convert Summary Text to Binary  |                                                                                                              |
| Convert Summary Text to Binary | Code                         | Converts session summary text to binary     | fluidX API - Media Summary     | Upload Session Summary         |                                                                                                              |
| Upload Session Summary        | Google Drive Upload           | Uploads session summary text file           | Convert Summary Text to Binary | Remember Session Summary File  |                                                                                                              |
| Remember Session Summary File | Code                         | Remembers uploaded session summary file ID | Upload Session Summary         | Clear Uploaded Photo Store     |                                                                                                              |
| Clear Uploaded Photo Store    | Code                         | Clears cached uploaded photo IDs at session end | Remember Session Summary File | The End                        |                                                                                                              |
| The End                      | No Operation (NoOp)           | Marks end of workflow execution             | Clear Uploaded Photo Store      | -                              |                                                                                                              |
| ℹ️ Viewer Tips (multiple nodes) | Sticky Note                   | Provides contextual operational tips        | -                              | -                              | Multiple notes provide API endpoint references, usage instructions, and best practices throughout workflow   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Form Trigger node ("On form submission"):**  
   - Type: Form Trigger  
   - Form Title: "fluidX digital The Eye Session Invitation"  
   - Fields:  
     - User Phone Number (required, placeholder "+4912345678")  
     - Agent Email Address (required, placeholder "agent@example.com")  

2. **Add a Set node ("Set Config"):**  
   - Set variables:  
     - `BASE_URL`: `"https://live.fluidx.digital"`  
     - `phoneNumberUser`: Expression `{{$json["User Phone Number"]}}`  
     - `message`: `"fluidX.digital TheEye. Your Link: "`  
     - `emailAgent`: Expression `{{$json["Agent Email Address"]}}`  
     - `language`: `"DE"` (default, adjust as needed)  
     - `company`: `"service berlin"`  
     - `project`: `"fluidX demo"`  
     - `billingcode`: `"demo-budget"`  
     - `correlationId`: Use current datetime `{{$now}}`  
     - `sku`: `"THEEYE"`  
   - Connect form trigger output to this node  

3. **Add HTTP Request node ("fluidX API - Create Session"):**  
   - URL: `{{$node["Set Config"].json.BASE_URL}}/api/fx/ext/session/create`  
   - Method: POST  
   - Authentication: HTTP Header Auth using fluidX API key credential (header `x-api-key`)  
   - Body (JSON):  
     ```json
     {
       "sessionDetails": {
         "company": "={{ $json.company }}",
         "project": "={{ $json.project }}",
         "billingcode": "={{ $json.billingcode }}",
         "correlationId": "2025-1234",
         "sku": "={{ $json.sku }}"
       }
     }
     ```  
   - Connect "Set Config" output to this node  

4. **Add Set node ("Set Session Vars"):**  
   - Set variables from previous node response:  
     - `sessionId`: `{{$json.revXRSession.id}}`  
     - `viewTheEyeUrl`: `{{$json.sessionInfo.viewTheEyeUrl}}`  
     - `theEyeUserUrl`: `{{$json.sessionInfo.theEyeUserUrl}}`  
     - `folderName`: expression: `"THEEYE_" + $now.toFormat("yyyyLLdd_HHmmss") + "_" + $json.revXRSession.id`  
   - Connect "fluidX API - Create Session" output to this node  

5. **Add Google Drive node ("Create Session Folder"):**  
   - Resource: Folder  
   - Operation: Create  
   - Folder Name: Expression from "Set Session Vars" node `folderName`  
   - Parent Folder ID: your THEEYE folder ID (replace with your Google Drive folder ID)  
   - Credentials: Google Drive OAuth2 credentials  
   - Connect "Set Session Vars" output to this node  

6. **Add HTTP Request node ("fluidX API - Send SMS User"):**  
   - URL: `${BASE_URL}/api/fx/ext/sms/send`  
   - Method: POST  
   - Authentication: fluidX API key (header)  
   - Body (JSON):  
     ```json
     {
       "phoneNumber": "={{ $node[\"Set Config\"].json.phoneNumberUser }}",
       "message": "={{ ($node[\"Set Config\"].json.message + $node[\"Set Session Vars\"].json.theEyeUserUrl).toJsonString() }}",
       "sessionId": "={{ $node[\"Set Session Vars\"].json.sessionId.toJsonString() }}"
     }
     ```  
   - Connect "Create Session Folder" output to this node  

7. **Add Email Send node ("Send email to Agent"):**  
   - To: `={{ $node["Set Config"].json.emailAgent }}`  
   - From: `fluidx@titel.ai` (adjust as needed)  
   - Subject: `fluidX.digital View TheEye Link`  
   - Body (text):  
     ```
     fluidx.digital View TheEye: Your Link: 
     {{ $node["Set Session Vars"].json.viewTheEyeUrl }}
     ```  
   - Credentials: SMTP credentials configured in n8n  
   - Connect "fluidX API - Send SMS User" output to this node  

8. **Add Wait node ("Wait1"):**  
   - Duration: 30 seconds  
   - Connect "Send email to Agent" output to this node  

9. **Add HTTP Request node ("fluidX API - Get Session Info"):**  
   - URL: `${BASE_URL}/api/revxr/ext/session`  
   - Method: GET  
   - Query Param: `sessionId` from "Set Session Vars"  
   - Auth: fluidX API key  
   - Timeout: 30s  
   - Connect "Wait1" output to this node  

10. **Add If node ("If Session Active"):**  
    - Condition: `revXRSession.status == "ACTIVE"` (string equals)  
    - Connect "fluidX API - Get Session Info" output to this node  

11. **Add Set node ("Extract Media Refs photos"):**  
    - Set `photoRefs` to `={{$json.mediaInfo?.photoRefs || $json.photoRefs || []}}`  
    - Connect "If Session Active" True branch to this node  

12. **Add If node ("If photos"):**  
    - Condition: Check if `photoRefs` array is empty (array empty)  
    - Connect "Extract Media Refs photos" output to this node  

13. **Add Wait node ("Wait"):**  
    - Duration: 15 seconds  
    - Connect "If photos" True branch (empty) to this node  

14. **Add Item Lists node ("Split photoRefs (Item Lists)")**  
    - Field to split: `photoRefs`  
    - Connect "If photos" False branch (non-empty) to this node  

15. **Add If node ("If Media Id"):**  
    - Condition: `id` field exists and not empty  
    - Connect "Split photoRefs" output to this node  

16. **Add HTTP Request node ("fluidX API - Media Info HTTP Request GET"):**  
    - URL: `${BASE_URL}/api/revxr/ext/media/info?type=image&id={{$json.id}}`  
    - Method: GET  
    - Auth: fluidX API key  
    - Timeout: 30s  
    - Connect "If Media Id" True branch to this node  

17. **Add Merge node ("Merge photoRefs"):**  
    - Mode: Combine by position  
    - Connect "fluidX API - Media Info HTTP Request GET" and "If Media Id" False branch to this node  

18. **Add If node ("If new Image"):**  
    - Condition: `mediaInfo.description` not exists (string not exists)  
    - Connect "Merge photoRefs" output to this node  

19. **Add Code node ("Check Already Uploaded"):**  
    - JavaScript to maintain and check global static cache of uploaded photo IDs  
    - Connect "If new Image" True branch to this node  

20. **Add HTTP Request node ("fluidX API - Download Photo THEEYE Session"):**  
    - URL: `{{$json.url}}` (photo download URL)  
    - Method: GET  
    - Auth: fluidX API key  
    - Timeout: 60s  
    - Connect "Check Already Uploaded" output to this node  

21. **Add Set node ("Carry URL and Id"):**  
    - Set fields:  
      - `_srcId` = `{{$json.id}}`  
      - `_srcUrl` = `{{$json.url}}`  
    - Connect "fluidX API - Download Photo THEEYE Session" output to this node  

22. **Add OpenAI node ("Analyze image"):**  
    - Model: chatgpt-4o-latest  
    - Input type: base64 image  
    - Prompt: `"What's in this image? Output language is {{ $('Set Config').item.json.language }}"`  
    - Credentials: OpenAI API key  
    - On Error: continue output  
    - Connect "Carry URL and Id" output to this node  

23. **Add Merge node ("Merge Analyze + URL"):**  
    - Mode: Combine by position  
    - Connect "Analyze image" and "Carry URL and Id" outputs to this node  

24. **Add Set node ("Build MediaInfo"):**  
    - Build JSON with `type`, `id`, `title` (from AI text with fallback logic), `description` (AI text)  
    - Connect "Merge Analyze + URL" output to this node  

25. **Add HTTP Request node ("fluidX API - Media Info HTTP Request POST"):**  
    - URL: `${BASE_URL}/api/revxr/ext/media/info`  
    - Method: POST  
    - Body: MediaInfo JSON from previous node  
    - Auth: fluidX API key  
    - Connect "Build MediaInfo" output to this node  

26. **Add Code node ("Generate Photo Summary"):**  
    - Creates text summary for each photo from mediaInfo with filename and content  
    - Connect "fluidX API - Media Info HTTP Request POST" output to this node  

27. **Add Code node ("Convert Text to Binary"):**  
    - Converts photo summary text to base64 binary for upload  
    - Connect "Generate Photo Summary" output to this node  

28. **Add Google Drive Upload node ("Upload Photo Summary"):**  
    - Uploads photo summary text files to session folder  
    - Folder ID: from "Create Session Folder" output  
    - Credentials: Google Drive OAuth2  
    - Connect "Convert Text to Binary" output to this node  

29. **Add Google Drive Upload node ("Upload file"):**  
    - Uploads actual image files to session folder  
    - Folder ID: from "Create Session Folder" output  
    - Credentials: Google Drive OAuth2  
    - Connect "Carry URL and Id" output to this node  

30. **Add Merge node ("Merge"):**  
    - Mode: Choose Branch  
    - Connect "Upload file" and "Upload Photo Summary" outputs to this node  

31. **Connect "Merge" output to "Wait" node** (step 13) for polling cycle continuation.

32. **Add HTTP Request node ("fluidX API - Media Summary"):**  
    - URL: `${BASE_URL}/api/fx/ext/media/summary?sessionId={{ sessionId }}`  
    - Method: GET  
    - Auth: fluidX API key  
    - Connect "If Session Active" False branch to this node  

33. **Add Code node ("Convert Summary Text to Binary"):**  
    - Converts session media summary text from API response to binary for upload  
    - Checks and prevents duplicate uploads by caching session summary files in static data  
    - Connect "fluidX API - Media Summary" output to this node  

34. **Add Google Drive Upload node ("Upload Session Summary"):**  
    - Uploads session summary file to session folder  
    - Folder ID: from "Create Session Folder" output  
    - Credentials: Google Drive OAuth2  
    - OnError: continue (non-blocking)  
    - Connect "Convert Summary Text to Binary" output to this node  

35. **Add Code node ("Remember Session Summary File"):**  
    - Stores uploaded summary file ID in global static data keyed by sessionId to prevent future duplicate uploads  
    - Connect "Upload Session Summary" output to this node  

36. **Add Code node ("Clear Uploaded Photo Store"):**  
    - Clears global static array of uploaded photo IDs after session ends  
    - Connect "Remember Session Summary File" output to this node  

37. **Add NoOp node ("The End"):**  
    - Marks end of workflow  
    - Connect "Clear Uploaded Photo Store" output to this node  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Context or Link                                                                                         |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| Setup instructions emphasize creating a fluidX developer account and API key at https://live.fluidx.digital with TEST plan free for development use. All HTTP nodes use centralized fluidX API key credentials for easy rotation. Google Drive folder ID must be replaced with your own shared workspace folder before publishing. Keep sensitive credentials stored securely in n8n credentials or environment variables. Phone and email are collected dynamically via form trigger to avoid personal data storage.                                                   | Setup & Instructions Node content                                                                       |
| fluidX API reference documentation is at https://live.fluidx.digital/api/swagger.html, providing endpoint details used in this workflow.                                                                                                                                                                                                                                                                                                                                                                                                | API documentation link                                                                                   |
| The workflow includes multiple sticky notes providing operational tips such as API endpoint usage, waiting strategies, and cleanup best practices. These notes are integral for understanding the flow especially for maintenance and troubleshooting.                                                                                                                                                                                                                                                                                       | Sticky Notes throughout the workflow                                                                     |
| AI image analysis is powered by OpenAI's GPT-4o vision-capable model, configured to output captions in the language set in the "Set Config" node (`DE` by default). This enables automatic generation of descriptive metadata for session images.                                                                                                                                                                                                                                                                                           | AI Analysis block                                                                                        |
| The workflow demonstrates robust deduplication using global static data storage in n8n to avoid reprocessing or reuploading the same media or session summaries across runs, a key design to prevent API quota wastage and storage clutter.                                                                                                                                                                                                                                                                                                | Deduplication logic in "Check Already Uploaded" and "Remember Session Summary File" nodes               |
| SMS and email notifications ensure both customer and agent receive proper session invites with direct links, supporting seamless session joining and real-time collaboration.                                                                                                                                                                                                                                                                                                                                                             | Notification block                                                                                       |
| Google Drive integration is used to store all media assets and session summaries, enabling shared access and archival within a dedicated THEEYE folder structure.                                                                                                                                                                                                                                                                                                                                                                          | Google Drive nodes                                                                                       |
| The workflow uses wait nodes strategically to allow asynchronous session activation and media availability before polling, balancing API call frequency and responsiveness.                                                                                                                                                                                                                                                                                                                                                              | Wait nodes usage                                                                                        |
| For testing, users are advised to log out of live.fluidx.digital in the agent’s browser to ensure the invite flow opens a clean session.                                                                                                                                                                                                                                                                                                                                                                                                      | Setup instructions note                                                                                  |

---

**Disclaimer:** The provided text stems exclusively from an automated workflow built with n8n, an integration and automation tool. The processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.