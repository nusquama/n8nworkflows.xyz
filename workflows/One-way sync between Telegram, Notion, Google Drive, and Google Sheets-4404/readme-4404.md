One-way sync between Telegram, Notion, Google Drive, and Google Sheets

https://n8nworkflows.xyz/workflows/one-way-sync-between-telegram--notion--google-drive--and-google-sheets-4404


# One-way sync between Telegram, Notion, Google Drive, and Google Sheets

---

### 1. Workflow Overview

This workflow acts as a **one-way synchronization and content management assistant** integrating Telegram with Notion, Google Drive, and Google Sheets. It automates processing of different content types received from Telegram messages, facilitating organized storage and tracking across multiple platforms. The workflow is designed primarily for productivity and content archival use cases, especially for users who forward images, text, or files via Telegram and want them systematically saved and logged without manual intervention.

The workflow consists of the following logical blocks:

- **1.1 Input Reception:** Listens for incoming Telegram messages (images, texts, files).
- **1.2 Content Type Routing:** Routes the message data based on content type (Image, Text, File).
- **1.3 Image Processing Path:** Downloads, converts, uploads images, then adds them with captions to Notion.
- **1.4 Text Processing Path:** Adds received text messages as headings to a Notion page.
- **1.5 File Processing Path:** Downloads files from Telegram, uploads to Google Drive, then logs file metadata in Google Sheets.
- **1.6 Completion Notification:** Sends a confirmation message back to the Telegram sender upon successful processing.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception

- **Overview:** Captures all incoming Telegram messages to trigger the workflow.
- **Nodes Involved:**  
  - 📱 Telegram Message Trigger

- **Node Details:**

  - **Node:** 📱 Telegram Message Trigger  
    - *Type:* Telegram Trigger (Webhook-based)  
    - *Role:* Entry point; listens for new messages on Telegram account  
    - *Configuration:*  
      - Listens for "message" update events only  
      - Uses Telegram API credentials labeled "AI Employee"  
    - *Key Expressions:* None (triggers on any message)  
    - *Connections:* Outputs to the Content Type Router  
    - *Edge Cases:*  
      - Telegram API downtime or invalid credentials may cause missed triggers  
      - Messages without supported content types will not proceed past routing  
    - *Version:* 1.1  

---

#### 2.2 Content Type Routing

- **Overview:** Determines the type of content in the Telegram message to route processing accordingly.
- **Nodes Involved:**  
  - 🔀 Content Type Router

- **Node Details:**

  - **Node:** 🔀 Content Type Router  
    - *Type:* Switch node  
    - *Role:* Inspects the message JSON to detect if the content is Image, Text, or File  
    - *Configuration:*  
      - Routes to "Image" if `message.photo` exists  
      - Routes to "Text" if `message.text` exists  
      - Routes to "File" if `message.document` exists  
      - Uses strict type validation and case sensitivity  
    - *Key Expressions:*  
      - `={{ $json.message.photo }}` for images  
      - `={{ $json.message.text }}` for text  
      - `={{ $json.message.document }}` for files  
    - *Connections:*  
      - To Image processing path (Download Telegram Image)  
      - To Text processing path (Add Text to Notion)  
      - To File processing path (Download Telegram File)  
    - *Edge Cases:*  
      - Messages with unsupported or multiple content types might not route correctly  
      - Missing expected fields may cause routing failure  
    - *Version:* 3.2  

---

#### 2.3 Image Processing Path

- **Overview:** Handles images sent via Telegram by downloading, converting, uploading to ImgBB, then adding them to Notion with captions.
- **Nodes Involved:**  
  - 📥 Download Telegram Image  
  - 🔄 Convert to Base64  
  - 🌐 Upload to ImgBB  
  - 📝 Add Image to Notion  
  - Image Path Description (Sticky Note)

- **Node Details:**

  - **Node:** 📥 Download Telegram Image  
    - *Type:* Telegram node (file download)  
    - *Role:* Downloads the highest resolution Telegram photo (index 2)  
    - *Configuration:*  
      - `fileId` set from `message.photo[2].file_id`  
      - Uses Telegram API credentials "AI Employee"  
    - *Connections:* Outputs to Convert to Base64  
    - *Edge Cases:*  
      - Photos with fewer than 3 sizes may cause undefined index errors  
      - Telegram API limits or network issues can cause download failures  
    - *Version:* 1.2  

  - **Node:** 🔄 Convert to Base64  
    - *Type:* ExtractFromFile node  
    - *Role:* Converts downloaded binary image file to base64 string in JSON property  
    - *Configuration:* Binary to property conversion with default options  
    - *Connections:* Outputs to Upload to ImgBB  
    - *Edge Cases:* Corrupt file downloads may cause conversion errors  
    - *Version:* 1  

  - **Node:** 🌐 Upload to ImgBB  
    - *Type:* HTTP Request  
    - *Role:* Uploads base64 image to ImgBB image hosting service  
    - *Configuration:*  
      - POST to `https://api.imgbb.com/1/upload`  
      - Sends form-urlencoded body with parameter "image" containing base64 data  
      - Query parameters include expiration time (600 seconds) and API key placeholder `<api_key>`  
    - *Key Expressions:* `={{ $json.data }}` for image data  
    - *Connections:* Outputs to Add Image to Notion  
    - *Edge Cases:*  
      - Invalid or expired API key leads to auth errors  
      - Network issues or ImgBB downtime  
    - *Version:* 4.2  

  - **Node:** 📝 Add Image to Notion  
    - *Type:* HTTP Request (Notion API)  
    - *Role:* Adds a toggle block to a specific Notion parent block with image and caption  
    - *Configuration:*  
      - PATCH request to Notion API endpoint for children blocks  
      - Body includes toggle block with caption text combining Telegram message caption and sender's full name  
      - Toggle's child is an image block linking to ImgBB URL obtained from previous step  
      - Uses HTTP Header Auth with Notion API credentials "Lakshit Notion Account"  
      - Notion API version header set to "2022-06-28"  
    - *Key Expressions:*  
      - Caption: `{{ $('🔀 Content Type Router').item.json.message.caption }} - {{ $('📱 Telegram Message Trigger').item.json.message.from.first_name }} {{ $('📱 Telegram Message Trigger').item.json.message.from.last_name }}`  
      - Image URL: `{{ $json.data.url }}`  
    - *Connections:* Outputs to Completion Notification  
    - *Edge Cases:*  
      - Notion API rate limits or authorization failures  
      - Missing caption or malformed JSON may cause request failure  
    - *Version:* 4.2  

---

#### 2.4 Text Processing Path

- **Overview:** Processes textual messages by adding them as heading blocks in a Notion page.
- **Nodes Involved:**  
  - 📝 Add Text to Notion  
  - Text Path Description (Sticky Note)

- **Node Details:**

  - **Node:** 📝 Add Text to Notion  
    - *Type:* Notion node (block creation)  
    - *Role:* Adds a Heading 3 block to a fixed Notion page with the message text content  
    - *Configuration:*  
      - Target block ID is statically set (`1fdb06aa7d2e80b384a6eb99788f67fb`)  
      - Uses Notion credentials "Lakshit Notion Account"  
      - Heading 3 block content set from Telegram message text: `={{ $json.message.text }}`  
    - *Connections:* Outputs to Completion Notification  
    - *Edge Cases:*  
      - Notion API errors or invalid page ID  
      - Empty or unsupported text content  
    - *Version:* 2.2  

---

#### 2.5 File Processing Path

- **Overview:** Downloads document files from Telegram, uploads them to Google Drive, then logs metadata into Google Sheets.
- **Nodes Involved:**  
  - 📥 Download Telegram File  
  - ☁️ Upload to Google Drive  
  - 📊 Record in Google Sheets  
  - File Path Description (Sticky Note)

- **Node Details:**

  - **Node:** 📥 Download Telegram File  
    - *Type:* Telegram node (file download)  
    - *Role:* Downloads the document file attached to the Telegram message  
    - *Configuration:*  
      - `fileId` from `message.document.file_id`  
      - Uses Telegram API credentials "AI Employee"  
    - *Connections:* Outputs to Upload to Google Drive  
    - *Edge Cases:*  
      - Large files or unsupported formats may cause download issues  
      - Telegram API limits or network errors  
    - *Version:* 1.2  

  - **Node:** ☁️ Upload to Google Drive  
    - *Type:* Google Drive node  
    - *Role:* Uploads downloaded file to a specific Google Drive folder  
    - *Configuration:*  
      - File name from Telegram message document file name  
      - Uploads to folder with ID `1Y9p3y0P7X39ZnMCH16iFpJbRofLnmR9u` in "My Drive"  
      - Uses OAuth2 credentials "Google Drive Lakshit77 account"  
    - *Connections:* Outputs to Record in Google Sheets  
    - *Edge Cases:*  
      - Insufficient Drive storage or permission errors  
      - Filename collisions or forbidden characters  
    - *Version:* 3  

  - **Node:** 📊 Record in Google Sheets  
    - *Type:* Google Sheets node  
    - *Role:* Appends a new row with file metadata to a Google Sheets spreadsheet  
    - *Configuration:*  
      - Targets Sheet1 (`gid=0`) in document ID `1RZs7yzvmJZrlHMonYhHNrgmyNIBlYaZt3Lp5XY5fen0`  
      - Columns recorded: Person Uploaded (sender name), File Name, Created At (formatted date/time), File Type, File Size (MB), Drive Link  
      - Uses OAuth2 credentials "Freelance Account"  
      - Data expressions calculate file size in MB and format creation date  
    - *Connections:* Outputs to Completion Notification  
    - *Edge Cases:*  
      - Google Sheets API quota limits or permission errors  
      - Date formatting errors if timestamps missing  
    - *Version:* 4.5  

---

#### 2.6 Completion Notification

- **Overview:** Sends a confirmation message back to the Telegram user when content processing completes successfully.
- **Nodes Involved:**  
  - ✅ Send Completion Message  
  - Completion Description (Sticky Note)

- **Node Details:**

  - **Node:** ✅ Send Completion Message  
    - *Type:* Telegram node (send message)  
    - *Role:* Sends a fixed success message to the Telegram chat from which content was received  
    - *Configuration:*  
      - Message text: "✅ Task Completed Successfully! Your content has been processed and saved."  
      - Chat ID dynamically set from incoming message chat: `={{ $('📱 Telegram Message Trigger').item.json.message.chat.id }}`  
      - Uses Telegram API credentials "AI Employee"  
      - Attribution disabled in message  
    - *Connections:* Terminal node (no outputs)  
    - *Edge Cases:*  
      - Telegram API blocking messages or invalid chat IDs  
      - Network failures  
    - *Version:* 1.2  

---

### 3. Summary Table

| Node Name                 | Node Type                | Functional Role                      | Input Node(s)                | Output Node(s)              | Sticky Note                                                                                                 |
|---------------------------|--------------------------|------------------------------------|-----------------------------|-----------------------------|-------------------------------------------------------------------------------------------------------------|
| Main Description          | Sticky Note              | Workflow general description       |                             |                             | # 📱 Telegram Productivity Assistant: Images to Notion, Text as headings, Files to Google Drive & Sheets   |
| 📱 Telegram Message Trigger | Telegram Trigger         | Entry point for incoming messages  |                             | 🔀 Content Type Router       |                                                                                                             |
| 🔀 Content Type Router     | Switch                   | Routes message by content type     | 📱 Telegram Message Trigger  | 📥 Download Telegram Image, 📝 Add Text to Notion, 📥 Download Telegram File |                                                                                                             |
| Image Path Description     | Sticky Note              | Describes image processing path    |                             |                             | ## 📸 IMAGE PROCESSING PATH: Download → Base64 → ImgBB → Notion                                             |
| 📥 Download Telegram Image | Telegram (file)          | Downloads Telegram image            | 🔀 Content Type Router       | 🔄 Convert to Base64         |                                                                                                             |
| 🔄 Convert to Base64       | ExtractFromFile          | Converts image to base64            | 📥 Download Telegram Image   | 🌐 Upload to ImgBB           |                                                                                                             |
| 🌐 Upload to ImgBB         | HTTP Request             | Uploads image to ImgBB hosting      | 🔄 Convert to Base64         | 📝 Add Image to Notion       |                                                                                                             |
| 📝 Add Image to Notion     | HTTP Request             | Adds image with caption to Notion  | 🌐 Upload to ImgBB           | ✅ Send Completion Message   |                                                                                                             |
| Text Path Description      | Sticky Note              | Describes text processing path     |                             |                             | ## 📝 TEXT PROCESSING PATH: Adds text as heading in Notion page                                            |
| 📝 Add Text to Notion      | Notion                   | Adds text heading to Notion page   | 🔀 Content Type Router       | ✅ Send Completion Message   |                                                                                                             |
| File Path Description      | Sticky Note              | Describes file processing path     |                             |                             | ## 📁 FILE PROCESSING PATH: Download → Google Drive → Sheets logging                                         |
| 📥 Download Telegram File  | Telegram (file)          | Downloads file from Telegram        | 🔀 Content Type Router       | ☁️ Upload to Google Drive     |                                                                                                             |
| ☁️ Upload to Google Drive  | Google Drive             | Uploads file to Drive folder        | 📥 Download Telegram File    | 📊 Record in Google Sheets   |                                                                                                             |
| 📊 Record in Google Sheets | Google Sheets            | Logs file metadata in spreadsheet   | ☁️ Upload to Google Drive     | ✅ Send Completion Message   |                                                                                                             |
| Completion Description     | Sticky Note              | Describes completion notification  |                             |                             | ## ✅ COMPLETION NOTIFICATION: Sends confirmation back to Telegram user                                      |
| ✅ Send Completion Message | Telegram (send message)  | Sends success message to user       | 📝 Add Text to Notion, 📝 Add Image to Notion, 📊 Record in Google Sheets |                             |                                                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node:**  
   - Type: Telegram Trigger  
   - Listen for "message" updates only  
   - Use Telegram API credentials (OAuth token for your bot)  
   - Name: "📱 Telegram Message Trigger"

2. **Create Switch Node to Route Content Type:**  
   - Type: Switch  
   - Add three outputs: "Image", "Text", "File"  
   - Conditions:  
     - Image: check if `message.photo` array exists and is non-empty  
     - Text: check if `message.text` exists  
     - File: check if `message.document` exists  
   - Connect output of Telegram Trigger to this Switch  
   - Name: "🔀 Content Type Router"

3. **Image Processing Path:**  

   3.1. **Download Telegram Image:**  
        - Type: Telegram (file) node  
        - Resource: file  
        - File ID: `={{ $json.message.photo[2].file_id }}` (highest resolution)  
        - Use Telegram credentials  
        - Connect "Image" output of Switch to this node  
        - Name: "📥 Download Telegram Image"

   3.2. **Convert Image to Base64:**  
        - Type: ExtractFromFile  
        - Operation: Binary to property (base64)  
        - Connect Download Telegram Image output here  
        - Name: "🔄 Convert to Base64"

   3.3. **Upload to ImgBB:**  
        - Type: HTTP Request  
        - Method: POST  
        - URL: `https://api.imgbb.com/1/upload`  
        - Query Parameters: `key=<your_imgbb_api_key>`, `expiration=600`  
        - Body Parameters (form-urlencoded): `image` set to base64 property from previous step  
        - Connect Convert to Base64 output here  
        - Name: "🌐 Upload to ImgBB"

   3.4. **Add Image to Notion:**  
        - Type: HTTP Request  
        - Method: PATCH  
        - URL: Notion API endpoint for adding children blocks to your target block  
        - Authentication: HTTP Header Auth with Notion API token  
        - Headers: `Notion-Version: 2022-06-28`  
        - Body JSON: Create a toggle block with caption text from Telegram message caption + sender’s name, and child block as image with URL from ImgBB response  
        - Connect Upload to ImgBB output here  
        - Name: "📝 Add Image to Notion"

4. **Text Processing Path:**  

   4.1. **Add Text to Notion:**  
        - Type: Notion node (block)  
        - Resource: block  
        - Operation: append or create heading 3 block in target page (use your Notion block ID)  
        - Text content: Telegram message text `={{ $json.message.text }}`  
        - Use Notion credentials  
        - Connect "Text" output of Switch here  
        - Name: "📝 Add Text to Notion"

5. **File Processing Path:**  

   5.1. **Download Telegram File:**  
        - Type: Telegram (file) node  
        - Resource: file  
        - File ID: `={{ $json.message.document.file_id }}`  
        - Use Telegram credentials  
        - Connect "File" output of Switch here  
        - Name: "📥 Download Telegram File"

   5.2. **Upload to Google Drive:**  
        - Type: Google Drive node  
        - Operation: Upload file  
        - Folder ID: Target folder where you want to save files  
        - File Name: `={{ $json.message.document.file_name }}`  
        - Use Google Drive OAuth2 credentials  
        - Connect Download Telegram File output here  
        - Name: "☁️ Upload to Google Drive"

   5.3. **Record File Info in Google Sheets:**  
        - Type: Google Sheets node  
        - Operation: Append row  
        - Spreadsheet ID: Your Google Sheets document ID  
        - Sheet Name: e.g., "Sheet1" or `gid=0`  
        - Map columns:  
          - Person Uploaded: `={{ $json.message.from.first_name + " " + $json.message.from.last_name }}`  
          - File Name: `={{ $json.message.document.file_name }}`  
          - Created At: `={{ $json.createdTime.toDateTime().toFormat("dd-MM-yyyy HH:mm") }}` (format as per your timezone)  
          - File Type: `={{ $json.fileExtension }}`  
          - File Size: `={{ ($json.size / (1024 * 1024)).toFixed(2) }} mb`  
          - Drive Link: `={{ $json.webViewLink }}`  
        - Use Google Sheets OAuth2 credentials  
        - Connect Upload to Google Drive output here  
        - Name: "📊 Record in Google Sheets"

6. **Completion Notification:**  

   - Create Telegram node (send message)  
   - Text: "✅ Task Completed Successfully! Your content has been processed and saved."  
   - Chat ID: `={{ $('📱 Telegram Message Trigger').item.json.message.chat.id }}`  
   - Use Telegram credentials  
   - Connect outputs from:  
     - 📝 Add Text to Notion  
     - 📝 Add Image to Notion  
     - 📊 Record in Google Sheets  
   - Name: "✅ Send Completion Message"

7. **Optionally Add Sticky Notes:**  
   - For each main logical block, add descriptive sticky notes to document purpose and flow.

8. **Activate the workflow** and test by sending image, text, and file messages to your Telegram bot.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                              | Context or Link                                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| Workflow integrates Telegram content with Notion, Google Drive, and Google Sheets in a unified productivity assistant setup.                             | Workflow purpose                                                                                                    |
| ImgBB used as image hosting to avoid storing images directly in Notion, improving performance and reliability.                                            | ImgBB API documentation: https://api.imgbb.com/                                                                    |
| Notion API version pinned to "2022-06-28" to ensure compatibility with toggle block and image child blocks.                                               | Notion API docs: https://developers.notion.com/reference                                                             |
| Telegram API credentials must have permission to read messages and send messages for notification to succeed.                                             | Telegram Bot API docs: https://core.telegram.org/bots/api                                                             |
| Google Drive and Google Sheets OAuth2 credentials require appropriate scopes for file upload and sheet modification respectively.                         | Google API docs: https://developers.google.com/drive/api and https://developers.google.com/sheets/api                 |
| Date/time formatting assumes timezone awareness; configure n8n timezone settings accordingly to match your requirements.                                  | n8n documentation on date functions                                                                                  |

---

*Disclaimer: The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.*