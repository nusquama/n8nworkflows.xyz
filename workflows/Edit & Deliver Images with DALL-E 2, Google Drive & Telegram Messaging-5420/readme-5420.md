Edit & Deliver Images with DALL-E 2, Google Drive & Telegram Messaging

https://n8nworkflows.xyz/workflows/edit---deliver-images-with-dall-e-2--google-drive---telegram-messaging-5420


# Edit & Deliver Images with DALL-E 2, Google Drive & Telegram Messaging

### 1. Workflow Overview

This n8n workflow, titled **"Edit & Deliver Images with DALL-E 2, Google Drive & Telegram Messaging"**, automates an end-to-end AI image editing pipeline. It is designed for users who want to upload an image and specify desired edits via a simple web form, then receive the edited image delivered instantly via Telegram. The workflow leverages OpenAI’s DALL-E 2 image editing API, Google Drive for secure storage of originals, and Telegram for notification and delivery.

The workflow logic is organized into the following functional blocks:

- **1.1 Input Reception:** Captures user-uploaded images and editing instructions via a web form.
- **1.2 Configuration Setup:** Defines key variables such as Google Drive folder ID, Telegram chat ID, image size, and AI model.
- **1.3 Original Image Backup:** Uploads the original user image to Google Drive for backup and organization.
- **1.4 Image Preparation:** Downloads the stored image from Drive to prepare it for AI processing.
- **1.5 AI Processing:** Sends the downloaded image and user prompt to OpenAI’s image editing API (DALL-E 2).
- **1.6 Result Conversion:** Converts the AI’s HTTP response into a usable image file.
- **1.7 Delivery:** Sends the final edited image to a specified Telegram chat.

Each block builds upon the previous, forming a linear flow from user input to final delivery.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block initiates the workflow when a user submits a form containing an image file and an editing prompt.

**Nodes Involved:**  
- 🚀 Form Submission Trigger

**Node Details:**

- **🚀 Form Submission Trigger**  
  - Type: Form Trigger  
  - Role: Entry point; listens for HTTP form submissions with file upload and text input.  
  - Configuration:  
    - Form titled "AI Image Editor"  
    - Two required fields:  
      - `image` (single file upload)  
      - `prompt` (text input describing desired edits)  
    - Form description explains user instructions.  
  - Expressions/Variables: Accesses uploaded file as binary data; prompt string as JSON property.  
  - Input: HTTP webhook call from form submission.  
  - Output: Passes form data to next node with file in binary and prompt in JSON.  
  - Edge Cases:  
    - Missing required fields (image or prompt) will prevent submission.  
    - File type validation is implicit; non-image files may cause errors downstream.  
  - Version-specific: Requires n8n version supporting Form Trigger node v2.2.

---

#### 1.2 Configuration Setup

**Overview:**  
Sets essential workflow variables that users must customize before running.

**Nodes Involved:**  
- ⚙️ Configuration Variables

**Node Details:**

- **⚙️ Configuration Variables**  
  - Type: Set Node  
  - Role: Defines static variables used throughout the workflow.  
  - Configuration:  
    - `google_drive_folder_id`: Placeholder string for Google Drive folder ID to save original images.  
    - `telegram_chat_id`: Placeholder string for Telegram chat to send images.  
    - `image_size`: Output image size, default "1024x1024".  
    - `openai_model`: AI model name, default "dall-e-2".  
  - Expressions/Variables: Variables are referenced in downstream nodes via expressions.  
  - Input: Receives data from form trigger node.  
  - Output: Data enriched with configuration variables.  
  - Edge Cases:  
    - Invalid or missing IDs will cause Google Drive or Telegram nodes to fail.  
    - Model name must be supported by OpenAI API.  
  - Version-specific: Uses Set node v3.4 for improved assignment capabilities.

---

#### 1.3 Original Image Backup

**Overview:**  
Uploads the original user image to Google Drive with a timestamped filename for backup and organization.

**Nodes Involved:**  
- 📁 Upload Original to Drive

**Node Details:**

- **📁 Upload Original to Drive**  
  - Type: Google Drive Node  
  - Role: Uploads binary image data as a PNG file to configured Drive folder.  
  - Configuration:  
    - File name set dynamically: `original-<timestamp>.png` using current datetime.  
    - Drive: "My Drive" selected.  
    - Folder ID: Retrieved from configuration variables node.  
    - Input data field: Uses binary data from form upload.  
  - Input: Receives image and config variables.  
  - Output: Outputs file metadata including file ID for downstream download.  
  - Edge Cases:  
    - OAuth credential issues can block upload.  
    - Invalid folder ID or insufficient permissions cause failure.  
  - Version-specific: Google Drive node v3.

---

#### 1.4 Image Preparation

**Overview:**  
Downloads the original image from Google Drive to prepare it for sending to OpenAI’s API.

**Nodes Involved:**  
- 📥 Download for Processing

**Node Details:**

- **📥 Download for Processing**  
  - Type: Google Drive Node  
  - Role: Downloads the image file by ID to binary data for HTTP request.  
  - Configuration:  
    - File ID dynamically set from output of upload node.  
    - Operation: Download.  
  - Input: Receives file metadata with ID from upload node.  
  - Output: Binary image data for AI processing.  
  - Edge Cases:  
    - File not found or permission denied errors.  
    - Network timeouts.  
  - Version-specific: Google Drive node v3.

---

#### 1.5 AI Processing

**Overview:**  
Sends the downloaded image and user prompt to OpenAI’s image editing API (DALL-E 2) to generate edited images.

**Nodes Involved:**  
- 🎨 AI Image Editor

**Node Details:**

- **🎨 AI Image Editor**  
  - Type: HTTP Request Node  
  - Role: Calls OpenAI’s /v1/images/edits API endpoint with multipart form data.  
  - Configuration:  
    - URL: `https://api.openai.com/v1/images/edits`  
    - Method: POST  
    - Timeout: 60 seconds  
    - Content Type: multipart-form-data  
    - Authentication: HTTP Header Auth with OpenAI API key credential.  
    - Body Parameters:  
      - `image`: Binary data field "data" (downloaded file)  
      - `prompt`: User prompt from form trigger node  
      - `model`: From configuration variables, default "dall-e-2"  
      - `size`: From configuration variables, default "1024x1024"  
      - `n`: 1 (one image generated)  
  - Input: Binary image data and prompt.  
  - Output: JSON response containing edited image URL(s).  
  - Edge Cases:  
    - API rate limits, invalid API key, or malformed requests.  
    - Timeout or network errors.  
    - Handling empty or invalid prompt may cause errors.  
  - Version-specific: HTTP Request v4.2.

---

#### 1.6 Result Conversion

**Overview:**  
Converts the AI response URL into a binary PNG file for delivery.

**Nodes Involved:**  
- 🔄 Convert to File

**Node Details:**

- **🔄 Convert to File**  
  - Type: Convert To File Node  
  - Role: Downloads the edited image from returned URL, converts to a binary PNG file with proper naming.  
  - Configuration:  
    - Operation: toBinary  
    - Source Property: `data[0].url` (first AI image URL)  
    - File Name: `edited-<timestamp>.png` dynamically set  
    - MIME Type: image/png  
  - Input: JSON with URL from AI node.  
  - Output: Binary image file for Telegram node.  
  - Edge Cases:  
    - URL unreachable or HTTP errors.  
    - Invalid or missing URL in AI response.  
  - Version-specific: Convert To File v1.1.

---

#### 1.7 Delivery

**Overview:**  
Sends the final edited image as a photo to a Telegram chat specified in configuration.

**Nodes Involved:**  
- 📱 Send to Telegram

**Node Details:**

- **📱 Send to Telegram**  
  - Type: Telegram Node  
  - Role: Sends photo message with binary image data to Telegram chat.  
  - Configuration:  
    - Chat ID: from configuration variables  
    - Operation: sendPhoto  
    - Binary Data: true (sends the image file)  
    - Additional Fields: none specified  
    - Requires Telegram Bot OAuth credentials.  
  - Input: Binary image from Convert to File node.  
  - Output: Telegram API response confirming message delivery.  
  - Edge Cases:  
    - Invalid chat ID or bot permissions.  
    - Network or Telegram API rate-limits/errors.  
  - Version-specific: Telegram node v1.2.

---

### 3. Summary Table

| Node Name                 | Node Type               | Functional Role                 | Input Node(s)                 | Output Node(s)               | Sticky Note                                                                                                 |
|---------------------------|-------------------------|--------------------------------|------------------------------|-----------------------------|------------------------------------------------------------------------------------------------------------|
| 🚀 Form Submission Trigger | Form Trigger            | User input reception via form  | —                            | ⚙️ Configuration Variables   | 💡 **STEP 1: Form Submission** User uploads image and prompt via web form; fields required.                |
| ⚙️ Configuration Variables | Set                     | Define workflow variables      | 🚀 Form Submission Trigger     | 📁 Upload Original to Drive  | ⚙️ **STEP 2: Configuration** Set customizable variables before first use.                                  |
| 📁 Upload Original to Drive| Google Drive            | Upload original image to Drive | ⚙️ Configuration Variables    | 📥 Download for Processing   | 📁 **STEP 3: Backup Storage** Uploads image with timestamp to Drive folder.                                |
| 📥 Download for Processing | Google Drive            | Download image for AI processing| 📁 Upload Original to Drive   | 🎨 AI Image Editor           | 📥 **STEP 4: Prepare for AI** Downloads image for OpenAI input.                                            |
| 🎨 AI Image Editor         | HTTP Request            | Call OpenAI image editing API  | 📥 Download for Processing    | 🔄 Convert to File           | 🎨 **STEP 5: AI Magic** Sends image and prompt to OpenAI DALL-E 2 API.                                    |
| 🔄 Convert to File         | Convert To File         | Convert AI response URL to file| 🎨 AI Image Editor            | 📱 Send to Telegram          | 🔄 **STEP 6: File Conversion** Converts AI URL to PNG binary file with timestamped name.                   |
| 📱 Send to Telegram        | Telegram                | Deliver edited image to user   | 🔄 Convert to File            | —                           | 📱 **STEP 7: Instant Delivery** Sends edited image to Telegram chat with timestamp.                        |
| Sticky Note               | Sticky Note             | Documentation and instructions | —                            | —                           | # 🎨 AI Image Editor Workflow... Full detailed description and setup instructions.                         |
| Sticky Note1              | Sticky Note             | Step 1 description             | —                            | —                           | 💡 **STEP 1: Form Submission** User uploads image and prompt with validation.                             |
| Sticky Note2              | Sticky Note             | Step 2 description             | —                            | —                           | ⚙️ **STEP 2: Configuration** Variables to update before use.                                               |
| Sticky Note3              | Sticky Note             | Step 3 description             | —                            | —                           | 📁 **STEP 3: Backup Storage** Upload original image to Drive.                                              |
| Sticky Note4              | Sticky Note             | Step 4 description             | —                            | —                           | 📥 **STEP 4: Prepare for AI** Download image for AI processing.                                           |
| Sticky Note5              | Sticky Note             | Step 5 description             | —                            | —                           | 🎨 **STEP 5: AI Magic** Sends image and prompt to OpenAI API.                                             |
| Sticky Note6              | Sticky Note             | Step 6 description             | —                            | —                           | 🔄 **STEP 6: File Conversion** Converts AI response to PNG file.                                          |
| Sticky Note7              | Sticky Note             | Step 7 description             | —                            | —                           | 📱 **STEP 7: Instant Delivery** Sends edited image to Telegram.                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node**  
   - Type: Form Trigger (v2.2)  
   - Configure Form Title: "AI Image Editor"  
   - Add two fields:  
     - File upload field labeled "image" (required, single file)  
     - Text input field labeled "prompt" with placeholder and required  
   - Save to get webhook URL.  

2. **Add a Set Node for Configuration Variables**  
   - Type: Set (v3.4)  
   - Add string variables:  
     - `google_drive_folder_id`: your Google Drive folder ID where images will be saved  
     - `telegram_chat_id`: your Telegram chat ID (numeric or username)  
     - `image_size`: e.g., "1024x1024"  
     - `openai_model`: e.g., "dall-e-2"  
   - Connect Form Trigger node output to this node input.

3. **Add Google Drive Node to Upload Original Image**  
   - Type: Google Drive (v3)  
   - Operation: Upload  
   - Drive: My Drive  
   - Folder ID: Use expression referencing `google_drive_folder_id` from Set node  
   - File Name: Use expression to create dynamic name like `original-{{ $now.format('yyyy-MM-dd-HHmmss') }}.png`  
   - Binary Property: "image" (from form upload)  
   - Connect Set node output to this node.

4. **Add Google Drive Node to Download Image**  
   - Type: Google Drive (v3)  
   - Operation: Download  
   - File ID: Use expression referencing uploaded file ID from previous node (`{{$json["id"]}}`)  
   - Connect Upload node output to this node.

5. **Add HTTP Request Node for OpenAI Image Editing**  
   - Type: HTTP Request (v4.2)  
   - Method: POST  
   - URL: `https://api.openai.com/v1/images/edits`  
   - Authentication: HTTP Header Auth with OpenAI API Key credential  
   - Content-Type: multipart/form-data  
   - Body Parameters:  
     - `image`: Binary form data from downloaded file ("data")  
     - `prompt`: Expression referencing form prompt  
     - `model`: Expression referencing `openai_model` variable  
     - `size`: Expression referencing `image_size` variable  
     - `n`: 1  
   - Timeout: 60000 ms (60 seconds)  
   - Connect Download node output to this node.

6. **Add Convert To File Node**  
   - Type: Convert To File (v1.1)  
   - Operation: toBinary  
   - Source Property: `data[0].url` (the URL of the edited image from OpenAI response)  
   - File Name: dynamic like `edited-{{ $now.format('yyyy-MM-dd-HHmmss') }}.png`  
   - MIME Type: image/png  
   - Connect HTTP Request node output to this node.

7. **Add Telegram Node to Send Photo**  
   - Type: Telegram (v1.2)  
   - Operation: sendPhoto  
   - Chat ID: Expression referencing `telegram_chat_id` variable  
   - Enable Binary Data sending  
   - Connect Convert To File node output to this node.

8. **Set up Credentials**:  
   - Google Drive OAuth credentials for both Google Drive nodes.  
   - Telegram Bot OAuth credentials for Telegram node.  
   - OpenAI API key stored as HTTP Header Auth credential for HTTP Request node.

9. **Test Workflow:**  
   - Activate workflow.  
   - Use form URL from Form Trigger node.  
   - Upload test image and enter prompt.  
   - Check Telegram for edited image delivery.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                          | Context or Link                                         |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------|
| Workflow created by David Olusola.                                                                                                                                                                                                                    | Branding                                               |
| Form provides user-friendly interface for uploading images and entering editing instructions.                                                                                                                                                        | User instructions                                      |
| Requires setup of Google Drive OAuth, Telegram Bot, and OpenAI API key credentials.                                                                                                                                                                   | Credentials setup                                      |
| Example prompts include: “Remove the background”, “Add sunglasses to the person”, “Change the sky to sunset”, “Make it look like a cartoon”.                                                                                                         | User guidance                                          |
| Customization options: change image size, file naming, add error handling, batch processing, multiple delivery channels.                                                                                                                              | Extendability notes                                   |
| This workflow utilizes DALL-E 2 model for AI image editing leveraging OpenAI’s API.                                                                                                                                                                   | AI model info                                          |

---

**Disclaimer:** The text provided originates exclusively from an automated n8n workflow. It strictly complies with content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.