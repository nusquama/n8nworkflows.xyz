Automated YouTube Publishing from Drive with GPT & Gemini Metadata Generation

https://n8nworkflows.xyz/workflows/automated-youtube-publishing-from-drive-with-gpt---gemini-metadata-generation-10619


# Automated YouTube Publishing from Drive with GPT & Gemini Metadata Generation

### 1. Workflow Overview

This workflow automates the full cycle of publishing motivational or economics-related videos to YouTube from a monitored Google Drive folder, enhanced by AI-generated metadata using OpenAI GPT and Google Gemini models. It also includes additional social media posting to Instagram and Facebook, plus notifications via Telegram.

Logical blocks:

- **1.1 Input Reception:** Detect new videos uploaded to a specific Google Drive folder.
- **1.2 Video Download:** Download the detected video file from Google Drive.
- **1.3 Metadata Generation:** Use AI models (OpenAI GPT-4.1-nano and Google Gemini) to create YouTube title, description, and SEO tags.
- **1.4 YouTube Upload:** Upload the video to YouTube with the generated metadata.
- **1.5 Post-Upload Cleanup:** Delete the uploaded video from Google Drive.
- **1.6 Notifications:** Send a Telegram message confirming the upload.
- **1.7 Social Media Posting:** Post related content to Instagram and Facebook using Facebook Graph API.
- **1.8 Token Management:** Fetch and manage permanent system user tokens for Facebook API usage.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Watches a specific Google Drive folder for newly created video files; triggers workflow execution on file creation.
- **Nodes Involved:**  
  - `New Video?`
- **Node Details:**

  - **New Video?**  
    - Type: Google Drive Trigger  
    - Role: Watches Google Drive folder for new files (event: fileCreated)  
    - Configuration: Polls every minute on folder ID `1vAZ6zUlbqqaFv2_sPZXx_yeNQcjrfcvy` (Motivational videos folder)  
    - Inputs: None (trigger node)  
    - Outputs: Trigger data includes new file metadata (e.g., file ID)  
    - Edge Cases: Potential delay from polling frequency; permission errors if Drive OAuth2 token invalid; folder access restricted  
    - Credentials: Google Drive OAuth2, account named "Google Drive account 2"

#### 2.2 Video Download

- **Overview:** Downloads the newly detected video file from Google Drive to be processed further.
- **Nodes Involved:**  
  - `Download New Video`
- **Node Details:**

  - **Download New Video**  
    - Type: Google Drive node (operation: download)  
    - Role: Downloads the video file using file ID from `New Video?` node  
    - Configuration: File ID dynamically set from trigger output (`={{ $('New Video?').item.json.id }}`)  
    - Inputs: Output from `YT Title` node (linked after title generation)  
    - Outputs: Binary file data of the video  
    - Edge Cases: Download failures due to file not found, permission errors, or network issues  
    - Credentials: Same Google Drive OAuth2 account as trigger node

#### 2.3 Metadata Generation

- **Overview:** Generates a professional, SEO-optimized YouTube title, description, and tags based on the video transcript or content, using GPT-4.1-nano and Gemini AI models.
- **Nodes Involved:**  
  - `Create Description` (OpenAI GPT-4.1-nano)  
  - `2.5FlashPrev` (Google Gemini)  
  - `YT Tags` (LangChain agent)  
  - `YT Title` (OpenAI GPT-4.1-nano)
- **Node Details:**

  - **Create Description**  
    - Type: OpenAI GPT node  
    - Role: Creates a detailed, engaging YouTube description in first-person style, with emojis and hashtags  
    - Configuration: GPT-4.1-nano model; system prompt instructs professional copywriting with specific style rules; input references original filename  
    - Inputs: Trigger output containing video metadata  
    - Outputs: AI-generated description text (`message.content`)  
    - Edge Cases: API quota limits, network timeouts, prompt misinterpretation  
    - Credentials: OpenAI API

  - **2.5FlashPrev**  
    - Type: Google Gemini AI node  
    - Role: Intermediate AI processing, presumably to enhance or validate content before tags generation  
    - Configuration: Uses model `models/gemini-2.0-flash`  
    - Inputs: Output of `Create Description` node  
    - Outputs: Processed AI content forwarded to `YT Tags`  
    - Edge Cases: API rate limits, invalid responses  
    - Credentials: Google Palm API

  - **YT Tags**  
    - Type: LangChain agent node  
    - Role: Generates comma-separated YouTube tags optimized for SEO, based on input transcript or video topic  
    - Configuration: Prompt instructs output of 20-30 tags related to motivation, economics, success etc., no hashtags or extra text  
    - Inputs: Content from `2.5FlashPrev` (or `Create Description`)  
    - Outputs: Tag list string  
    - Edge Cases: Tag relevance, API failures  
    - Credentials: None (LangChain agent using prior AI outputs)

  - **YT Title**  
    - Type: OpenAI GPT node  
    - Role: Creates a concise SEO-friendly video title (max 40 characters)  
    - Configuration: GPT-4.1-nano; prompt demands only title, no extra text  
    - Inputs: Tags output, chained from `YT Tags`  
    - Outputs: Video title text (`message.content`)  
    - Edge Cases: Title length issues, API errors  
    - Credentials: OpenAI API

#### 2.4 YouTube Upload

- **Overview:** Uploads the video file with generated metadata to YouTube under category "Education" (categoryId 22), marks video as public.
- **Nodes Involved:**  
  - `Upload to youtube`
- **Node Details:**

  - **Upload to youtube**  
    - Type: YouTube node (upload video)  
    - Role: Uploads video binary with title, description, tags, privacy status public  
    - Configuration:  
      - Title: from `YT Title` output  
      - Description: enriched by `Create Description` output plus static hashtags and tags from `YT Tags`  
      - Privacy: public  
      - Category: 22 (Education)  
      - Region code: IN (India)  
      - Binary property: video data from `Download New Video`  
    - Inputs: Video binary and metadata from prior nodes  
    - Outputs: Upload response including upload ID  
    - Edge Cases: Upload failures (network, quota), invalid metadata, binary data issues  
    - Credentials: YouTube OAuth2 (named "Eval info")

#### 2.5 Post-Upload Cleanup

- **Overview:** Deletes the source video file from Google Drive after successful upload to YouTube.
- **Nodes Involved:**  
  - `Delete video from Upload Folder1`
- **Node Details:**

  - **Delete video from Upload Folder1**  
    - Type: Google Drive node (deleteFile operation)  
    - Role: Deletes the original video file using file ID from `New Video?` trigger  
    - Inputs: File ID from trigger  
    - Outputs: Deletion confirmation JSON  
    - Edge Cases: Permission denied, file not found (e.g., if already deleted)  
    - Credentials: Same Google Drive OAuth2 account

#### 2.6 Notifications

- **Overview:** Sends a Telegram message to notify about the successful YouTube upload and deletion status.
- **Nodes Involved:**  
  - `Send a text message`
- **Node Details:**

  - **Send a text message**  
    - Type: Telegram node (send message)  
    - Role: Sends formatted message with upload ID, deletion confirmation, and video title  
    - Configuration:  
      - Chat ID hardcoded (`6727168479`)  
      - Text includes dynamic fields from upload and deletion nodes  
    - Inputs: Data from upload and delete nodes  
    - Edge Cases: Telegram API failures, unauthorized chat ID  
    - Credentials: Telegram API token

#### 2.7 Social Media Posting

- **Overview:** Posts related image content to Instagram and Facebook pages using Facebook Graph API with permanent system user tokens.
- **Nodes Involved:**  
  - `Post to Instagram`  
  - `Wait 2 Second`  
  - `Publish instagram post`  
  - `Use System token to get page token`  
  - `Get Correct Page Token`  
  - `Post to Facebook`
- **Node Details:**

  - **Post to Instagram**  
    - Type: HTTP Request  
    - Role: Creates Instagram media object by uploading image URL and caption  
    - Configuration:  
      - URL placeholder for page ID and image URL — requires user input  
      - Uses Facebook Graph API auth (system user token)  
    - Edge Cases: Invalid token, API rate limits, invalid parameters

  - **Wait 2 Second**  
    - Type: Wait node  
    - Role: Delays 2 seconds before publishing Instagram post (to allow media processing)  
    - Edge Cases: None significant

  - **Publish instagram post**  
    - Type: HTTP Request  
    - Role: Publishes the previously created Instagram media object using creation ID  
    - Configuration: Uses Facebook Graph API token  
    - Inputs: Creation ID from `Post to Instagram` response

  - **Use System token to get page token**  
    - Type: HTTP Request  
    - Role: Fetches list of Facebook pages and tokens using permanent system user token  
    - Configuration:  
      - URL: `https://graph.facebook.com/v19.0/me/accounts`  
      - Auth: Facebook Graph API  
    - Edge Cases: Token expired, permission denied

  - **Get Correct Page Token**  
    - Type: Code node (JavaScript)  
    - Role: Extracts correct page token by filtering response for page ID `266271423823110`  
    - Inputs: Output JSON of `Use System token to get page token`  
    - Outputs: Access token JSON  
    - Edge Cases: Page ID not found

  - **Post to Facebook**  
    - Type: HTTP Request  
    - Role: Posts a photo to Facebook page with caption and access token  
    - Configuration:  
      - URL: Facebook Graph API photo upload endpoint for page ID `266271423823110`  
      - Requires access token dynamically injected  
    - Edge Cases: Token invalid, photo upload fails

#### 2.8 Sticky Notes (Contextual Documentation)

- Sticky notes provide contextual guides and instructions for users, such as how to create permanent system user tokens for Facebook API, and general workflow block descriptions.

---

### 3. Summary Table

| Node Name                    | Node Type                          | Functional Role                     | Input Node(s)           | Output Node(s)                 | Sticky Note                                                                                                                                                                  |
|------------------------------|----------------------------------|-----------------------------------|------------------------|------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| New Video?                   | Google Drive Trigger             | Detect new video file in Drive    | None                   | Create Description            | Get video from Drive Folder [Guide](https://docs.n8n.io/workflows/sticky-notes/)                                                                                             |
| Create Description           | OpenAI GPT-4.1-nano             | Generate YouTube description      | New Video?              | 2.5FlashPrev, YT Tags         | Generate Meta data [Guide](https://docs.n8n.io/workflows/sticky-notes/)                                                                                                     |
| 2.5FlashPrev                 | Google Gemini AI                 | AI intermediate processing         | Create Description      | YT Tags                       | Generate Meta data [Guide](https://docs.n8n.io/workflows/sticky-notes/)                                                                                                     |
| YT Tags                     | LangChain agent                 | Generate YouTube SEO tags          | Create Description, 2.5FlashPrev | YT Title                      | Generate Meta data [Guide](https://docs.n8n.io/workflows/sticky-notes/)                                                                                                     |
| YT Title                    | OpenAI GPT-4.1-nano             | Generate SEO YouTube title         | YT Tags                 | Download New Video            | Generate Meta data [Guide](https://docs.n8n.io/workflows/sticky-notes/)                                                                                                     |
| Download New Video           | Google Drive                    | Download video file                | YT Title                | Upload to youtube             | Download Video [Guide](https://docs.n8n.io/workflows/sticky-notes/)                                                                                                         |
| Upload to youtube            | YouTube                        | Upload video with metadata         | Download New Video       | Delete video from Upload Folder1 | Upload Video [Guide](https://docs.n8n.io/workflows/sticky-notes/)                                                                                                           |
| Delete video from Upload Folder1 | Google Drive                    | Delete original video from Drive   | Upload to youtube       | Send a text message           | Delete Video [Guide](https://docs.n8n.io/workflows/sticky-notes/)                                                                                                           |
| Send a text message          | Telegram                       | Notify upload completion           | Delete video from Upload Folder1 | None                         | Send Notification [Guide](https://docs.n8n.io/workflows/sticky-notes/)                                                                                                      |
| Post to Instagram            | HTTP Request (Facebook Graph API) | Create Instagram media object      | None                   | Wait 2 Second                | Using the permanent never expire token                                                                                                                                       |
| Wait 2 Second               | Wait                          | Delay before publishing Instagram post | Post to Instagram       | Publish instagram post        | Using the permanent never expire token                                                                                                                                       |
| Publish instagram post       | HTTP Request (Facebook Graph API) | Publish Instagram media object     | Wait 2 Second           | Use System token to get page token | Using the permanent never expire token                                                                                                                                       |
| Use System token to get page token | HTTP Request (Facebook Graph API) | Fetch Facebook page tokens         | Publish instagram post  | Get Correct Page Token        | Using the permanent never expire token                                                                                                                                       |
| Get Correct Page Token       | Code                          | Extract correct Facebook page token | Use System token to get page token | Post to Facebook             | Using the permanent never expire token                                                                                                                                       |
| Post to Facebook             | HTTP Request (Facebook Graph API) | Post photo to Facebook page        | Get Correct Page Token   | None                         | Using the permanent never expire token                                                                                                                                       |
| Sticky Note                 | Sticky Note                   | Workflow documentation             | None                   | None                         | See individual sticky note contents                                                                                                                                          |
| Sticky Note1                | Sticky Note                   | Workflow documentation             | None                   | None                         | See individual sticky note contents                                                                                                                                          |
| Sticky Note2                | Sticky Note                   | Workflow documentation             | None                   | None                         | See individual sticky note contents                                                                                                                                          |
| Sticky Note3                | Sticky Note                   | Workflow documentation             | None                   | None                         | See individual sticky note contents                                                                                                                                          |
| Sticky Note4                | Sticky Note                   | Workflow documentation             | None                   | None                         | See individual sticky note contents                                                                                                                                          |
| Sticky Note5                | Sticky Note                   | Workflow documentation             | None                   | None                         | See individual sticky note contents                                                                                                                                          |
| Sticky Note8                | Sticky Note                   | Workflow documentation             | None                   | None                         | Using the permanent never expire token                                                                                                                                        |
| Sticky Note9                | Sticky Note                   | Workflow documentation             | None                   | None                         | How to Create a Permanent System User Token (Recommended) [Detailed Instructions Inside]                                                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**  
   - Add a Google Drive Trigger node named `New Video?`.  
   - Set event to `fileCreated`.  
   - Set polling interval to every 1 minute.  
   - Set folder to watch by folder ID: `1vAZ6zUlbqqaFv2_sPZXx_yeNQcjrfcvy`.  
   - Connect Google Drive OAuth2 credentials ("Google Drive account 2").

2. **Create Description Generation:**  
   - Add an OpenAI node named `Create Description`.  
   - Use model `gpt-4.1-nano`.  
   - System prompt: professional copywriter with style rules as per workflow (see node details).  
   - Input message: reference the original filename from `New Video?`.  
   - Connect input from `New Video?` node.  
   - Attach OpenAI API credentials.

3. **Add Google Gemini AI Node:**  
   - Add Gemini AI node named `2.5FlashPrev`.  
   - Set model to `models/gemini-2.0-flash`.  
   - Connect input from `Create Description`.  
   - Attach Google Palm API credentials.

4. **Add YouTube Tags Generator:**  
   - Add LangChain Agent node named `YT Tags`.  
   - Prompt instructs generation of 20-30 comma-separated tags, related to motivational/economics topics.  
   - Connect input from `2.5FlashPrev`.  
   - No specific credentials needed.

5. **Add YouTube Title Generator:**  
   - Add OpenAI node named `YT Title`.  
   - Use model `gpt-4.1-nano`.  
   - Prompt for a maximum 40-character SEO title.  
   - Connect input from `YT Tags`.  
   - Attach OpenAI API credentials.

6. **Add Video Download Node:**  
   - Add Google Drive node named `Download New Video`.  
   - Operation: Download file.  
   - File ID: use expression to get ID from `New Video?` trigger (`={{ $('New Video?').item.json.id }}`).  
   - Connect input from `YT Title`.  
   - Use Google Drive OAuth2 credentials ("Google Drive account 2").

7. **Add YouTube Upload Node:**  
   - Add YouTube node named `Upload to youtube`.  
   - Operation: Upload video.  
   - Set title from `YT Title` output.  
   - Set description combining static text, description from `Create Description`, and tags from `YT Tags`.  
   - Set privacy status to "public".  
   - Set category ID to `22` (Education).  
   - Set region code to `IN`.  
   - Binary property: from `Download New Video`.  
   - Connect input from `Download New Video`.  
   - Attach YouTube OAuth2 credentials ("Eval info").

8. **Add Google Drive Delete Node:**  
   - Add Google Drive node named `Delete video from Upload Folder1`.  
   - Operation: Delete file.  
   - File ID: same as in `New Video?` trigger.  
   - Connect input from `Upload to youtube`.  
   - Use Google Drive OAuth2 credentials.

9. **Add Telegram Notification Node:**  
   - Add Telegram node named `Send a text message`.  
   - Chat ID: `6727168479`.  
   - Message: Include upload ID, deletion success, and title from previous nodes using expressions.  
   - Connect input from `Delete video from Upload Folder1`.  
   - Attach Telegram API credentials.

10. **Set Up Facebook Graph API Token Retrieval:**  
    - Add HTTP Request node `Use System token to get page token`.  
    - GET request to `https://graph.facebook.com/v19.0/me/accounts`.  
    - Use Facebook Graph API credentials (permanent system user token).  
    - Connect input from `Publish instagram post`.

11. **Code Node to Extract Page Token:**  
    - Add Code node `Get Correct Page Token`.  
    - JavaScript to find page with ID `266271423823110` and return its access token.  
    - Connect input from `Use System token to get page token`.

12. **Facebook Photo Post Node:**  
    - Add HTTP Request node `Post to Facebook`.  
    - POST to `https://graph.facebook.com/v19.0/266271423823110/photos`.  
    - Body parameters: photo URL, caption, and dynamic access token from previous node.  
    - Connect input from `Get Correct Page Token`.

13. **Instagram Post Creation:**  
    - Add HTTP Request node `Post to Instagram`.  
    - POST to `https://graph.facebook.com/v19.0/<YOUR PAGE ID HERE>/media`.  
    - Body params: image_url and caption (to be customized).  
    - Connect input to `Wait 2 Second`.

14. **Wait Node:**  
    - Add Wait node `Wait 2 Second` with 2-second delay.  
    - Connect input from `Post to Instagram`.

15. **Publish Instagram Post:**  
    - Add HTTP Request node `Publish instagram post`.  
    - POST to `https://graph.facebook.com/v19.0/17841404935066235/media_publish`.  
    - Body param `creation_id` from `Post to Instagram` output.  
    - Connect input from `Wait 2 Second`.

16. **Connect workflow start to `New Video?` trigger and chain nodes as above.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                             | Context or Link                                         |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------|
| Guide to create permanent Facebook System User Token for automation: includes steps such as Business verification, System user creation, assigning assets, generating never-expiring token with required permissions (`pages_show_list`, `pages_read_engagement`, `pages_manage_posts`, `instagram_basic`, `instagram_content_publish`). Recommended for stable Facebook API access. | See Sticky Note9 content in workflow                    |
| Workflow uses multiple AI models: OpenAI GPT-4.1-nano for copywriting tasks, Google Gemini (PaLM) model for intermediate processing, LangChain agent for SEO tag generation. Credentials for all must be properly configured.                                                                                                                                                                         | n8n AI node documentation                              |
| All Google Drive nodes use the same OAuth2 credentials for consistent access.                                                                                                                                                                                                                                                                                                                            | Credential setup in workflow                            |
| Telegram notification node requires chat ID and bot token configured.                                                                                                                                                                                                                                                                                                                                   | Telegram Bot API documentation                          |
| YouTube upload uses categoryId 22 (Education) and sets video privacy to public. Change if needed.                                                                                                                                                                                                                                                                                                         | YouTube API documentation                               |
| Social media posting nodes require valid Facebook Page IDs and permanent system user tokens with appropriate permissions.                                                                                                                                                                                                                                                                                 | Facebook Graph API docs, see sticky note for token setup |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, respecting all relevant content policies. No illegal, offensive, or protected content is included. All data processed is legal and publicly accessible.