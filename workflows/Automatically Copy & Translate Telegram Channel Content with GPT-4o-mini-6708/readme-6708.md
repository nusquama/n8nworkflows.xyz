Automatically Copy & Translate Telegram Channel Content with GPT-4o-mini

https://n8nworkflows.xyz/workflows/automatically-copy---translate-telegram-channel-content-with-gpt-4o-mini-6708


# Automatically Copy & Translate Telegram Channel Content with GPT-4o-mini

### 1. Workflow Overview

This workflow automates the process of copying and translating the latest post from a specified Telegram channel and reposting it into a target Telegram channel. It is designed for daily execution and supports text-only posts, as well as posts containing images or videos. The workflow fetches the HTML content of the source Telegram channel, extracts the latest post’s content and media, cleans and translates the text using OpenAI’s GPT-4o-mini model, handles media URLs, downloads any media files, and sends the translated post to the target channel with proper attribution.

The workflow is logically divided into the following blocks:

- **1.1 Scheduling and Configuration:** Triggers the workflow daily and sets up channel parameters.
- **1.2 Content Retrieval and Extraction:** Fetches HTML content and extracts post text and media.
- **1.3 Text Cleaning and Translation:** Cleans the extracted text and translates it via OpenAI.
- **1.4 Media Processing:** Parses media URLs, identifies media type, and downloads media files.
- **1.5 Posting Content:** Sends the translated post with or without media to the target Telegram channel.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduling and Configuration

- **Overview:**  
  This block schedules the workflow to run daily at 9 AM and defines the source channel, target channel, translation language, and channel signature. It initializes the settings used throughout the workflow.

- **Nodes Involved:**  
  - Daily Schedule  
  - Set Telegram Channels

- **Node Details:**

  - **Daily Schedule**  
    - Type: Schedule Trigger  
    - Role: Triggers the workflow daily at 9 AM (configurable).  
    - Configuration: Time set to trigger at hour 9 daily.  
    - Input: None (trigger node).  
    - Output: Connects to Set Telegram Channels node.  
    - Potential Failures: Timezone mismatches, scheduler disabled.

  - **Set Telegram Channels**  
    - Type: Set  
    - Role: Defines key workflow parameters: source channel username, target channel username, target translation language, and channel signature to remove from posts.  
    - Configuration:  
      - sourceChannel: e.g., "channel_username"  
      - targetChannel: e.g., "@channel_username"  
      - targetLanguage: e.g., "Persian"  
      - channelSignature: e.g., "@channel_username" (used for cleaning post text)  
    - Input: From Daily Schedule node.  
    - Output: Connects to Fetch Channel Page node.  
    - Edge Cases: Incorrect channel usernames or missing parameters can cause failures downstream.

---

#### 1.2 Content Retrieval and Extraction

- **Overview:**  
  Fetches the latest post's HTML content from the source Telegram channel and extracts post text, post ID, video URL, and photo URL from the HTML.

- **Nodes Involved:**  
  - Fetch Channel Page  
  - Extract Post Content  
  - Clean Post Content

- **Node Details:**

  - **Fetch Channel Page**  
    - Type: HTTP Request  
    - Role: Retrieves the public HTML content of the Telegram source channel's posts page.  
    - Configuration:  
      - URL dynamically constructed as `https://t.me/s/{{sourceChannel}}`  
      - User-Agent header set to mimic a desktop browser for reliable fetching.  
    - Input: From Set Telegram Channels.  
    - Output: Connects to Extract Post Content.  
    - Edge Cases: HTTP errors, channel privacy settings blocking access, network timeouts.

  - **Extract Post Content**  
    - Type: HTML Extract  
    - Role: Parses the fetched HTML to extract the last (most recent) Telegram post's text, post ID, video source URL, and photo style attribute.  
    - Configuration: CSS selectors used to precisely locate content elements in the HTML structure of Telegram posts.  
    - Input: From Fetch Channel Page.  
    - Output: Connects to Clean Post Content.  
    - Edge Cases: Changes in Telegram page structure breaking CSS selectors, missing elements if the post lacks media.

  - **Clean Post Content**  
    - Type: Code (JavaScript)  
    - Role: Cleans the post text by removing the original channel signature to avoid attribution duplication and trims whitespace.  
    - Key Expression: Replaces all occurrences of the `channelSignature` string from the post text.  
    - Input: From Extract Post Content.  
    - Output: Connects to Translate Text.  
    - Edge Cases: Missing post text, signature not found, or irregular formatting.

---

#### 1.3 Text Cleaning and Translation

- **Overview:**  
  Translates the cleaned post text into the target language using OpenAI GPT-4o-mini model.

- **Nodes Involved:**  
  - Translate Text

- **Node Details:**

  - **Translate Text**  
    - Type: OpenAI (Langchain)  
    - Role: Sends the cleaned post text to OpenAI to translate it into the target language in a friendly, natural style without additional commentary or formatting.  
    - Configuration:  
      - Model: GPT-4o-mini  
      - Prompt: Injects the `targetLanguage` and the cleaned `post` text dynamically.  
    - Credentials: Uses OpenAI API credentials.  
    - Input: From Clean Post Content.  
    - Output: Connects to Process Media URLs.  
    - Edge Cases: API authentication failures, rate limits, empty or too long text inputs, network timeouts.

---

#### 1.4 Media Processing

- **Overview:**  
  Extracts and cleans media URLs from CSS style attributes, identifies whether the post contains photo, video, or only text, and downloads media files when applicable.

- **Nodes Involved:**  
  - Process Media URLs  
  - Has Photo? (If)  
  - Has Video? (If)  
  - Text Only? (If)  
  - Download Photo  
  - Download Video

- **Node Details:**

  - **Process Media URLs**  
    - Type: Code (JavaScript)  
    - Role: Parses the photo URL out of a CSS style attribute string, extracting the actual image URL for download.  
    - Input: From Translate Text.  
    - Output: Connects to three conditional nodes: Has Photo?, Has Video?, Text Only?  
    - Edge Cases: Malformed style attribute, missing photo URL.

  - **Has Photo?**  
    - Type: If  
    - Role: Checks if the cleaned JSON contains a non-empty `photo` URL.  
    - Input: From Process Media URLs.  
    - Output: If true → Download Photo; else no direct output.  
    - Edge Cases: False positives if photo URL is invalid.

  - **Has Video?**  
    - Type: If  
    - Role: Checks if the original extracted JSON has a non-empty `video` URL.  
    - Input: From Process Media URLs.  
    - Output: If true → Download Video; else no direct output.  
    - Edge Cases: Video URL may be inaccessible or expired.

  - **Text Only?**  
    - Type: If  
    - Role: Determines if the post contains neither photo nor video (both fields empty).  
    - Input: From Process Media URLs.  
    - Output: If true → Send Text to Channel node.  
    - Edge Cases: Posts with unsupported media types.

  - **Download Photo**  
    - Type: HTTP Request  
    - Role: Downloads the photo media file from the extracted photo URL.  
    - Configuration: Timeout set to 30 seconds.  
    - Input: From Has Photo? (true branch).  
    - Output: Connects to Send Photo to Channel.  
    - Edge Cases: Download failures, invalid URLs, slow network.

  - **Download Video**  
    - Type: HTTP Request  
    - Role: Downloads the video media file from the video URL.  
    - Configuration: Timeout set to 60 seconds.  
    - Input: From Has Video? (true branch).  
    - Output: Connects to Send Video to Channel.  
    - Edge Cases: Large file sizes causing timeout, broken links.

---

#### 1.5 Posting Content

- **Overview:**  
  Sends the translated post content to the target Telegram channel. Handles three content paths: text-only, text + photo, or text + video.

- **Nodes Involved:**  
  - Send Text to Channel  
  - Send Photo to Channel  
  - Send Video to Channel

- **Node Details:**

  - **Send Text to Channel**  
    - Type: Telegram  
    - Role: Sends text-only posts to the target Telegram channel.  
    - Configuration:  
      - Text: Translated message content + channel signature appended  
      - Chat ID: targetChannel  
      - `appendAttribution` disabled to avoid Telegram’s default attribution, since custom signature is appended.  
    - Input: From Text Only? (true branch).  
    - Credentials: Telegram Bot API credentials.  
    - Edge Cases: Bot permissions, chat ID errors.

  - **Send Photo to Channel**  
    - Type: Telegram  
    - Role: Sends posts containing a photo and translated text caption to the target channel.  
    - Configuration:  
      - Chat ID: targetChannel  
      - Operation: sendPhoto  
      - Binary Data: true (photo file)  
      - Caption: translated text + channel signature appended  
    - Input: From Download Photo (downloaded image).  
    - Credentials: Telegram Bot API credentials (may be different bot).  
    - Edge Cases: Media upload failures, bot permissions.

  - **Send Video to Channel**  
    - Type: Telegram  
    - Role: Sends posts containing a video and translated text caption to the target channel.  
    - Configuration:  
      - Chat ID: targetChannel  
      - Operation: sendVideo  
      - Binary Data: true (video file)  
      - Caption: translated text + channel signature appended  
    - Input: From Download Video (downloaded video).  
    - Credentials: Telegram Bot API credentials (may be different bot).  
    - Edge Cases: Large video size, upload timeouts, bot permissions.

---

### 3. Summary Table

| Node Name           | Node Type              | Functional Role                                  | Input Node(s)           | Output Node(s)                  | Sticky Note                                                                                                                                                |
|---------------------|------------------------|-------------------------------------------------|-------------------------|-------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Daily Schedule      | Schedule Trigger       | Triggers workflow daily at 9 AM                   | None                    | Set Telegram Channels          | Triggers the workflow daily at 9 AM. Adjust the time according to your needs.                                                                             |
| Set Telegram Channels | Set                    | Configures source/target channels and settings    | Daily Schedule           | Fetch Channel Page             | Configure your source channel, target channel, and other settings here                                                                                   |
| Fetch Channel Page  | HTTP Request           | Fetches HTML content of source Telegram channel   | Set Telegram Channels    | Extract Post Content           | ## 1. Get the html content of the source Telegram channel In this part of the workflow we fetch the html content of source channel and then extract text, images, and videos from the latest Telegram post. Removes original channel signatures and cleans up the text |
| Extract Post Content | HTML Extract           | Extracts latest post text, media URLs, and IDs    | Fetch Channel Page       | Clean Post Content             | (Covered by above note)                                                                                                                                    |
| Clean Post Content  | Code (JavaScript)      | Cleans post text by removing original channel signature | Extract Post Content   | Translate Text                | (Covered by above note)                                                                                                                                    |
| Translate Text      | OpenAI (Langchain)     | Translates cleaned post text to target language   | Clean Post Content       | Process Media URLs             | ## 2. Use AI for translation Uses OpenAI to translate the post content to the target language                                                             |
| Process Media URLs  | Code (JavaScript)      | Extracts clean image URLs and routes flow         | Translate Text           | Has Photo?, Has Video?, Text Only? | ## 3. Extract Media URLs and Identify Media Type Telegram uses different methods for handling image and video content. In this step, we identify the type of media included in the post. If the post contains an image or video, the corresponding media file is downloaded. |
| Has Photo?          | If                     | Checks if post has photo URL                       | Process Media URLs       | Download Photo                | (Covered by above note)                                                                                                                                    |
| Has Video?          | If                     | Checks if post has video URL                       | Process Media URLs       | Download Video                | (Covered by above note)                                                                                                                                    |
| Text Only?          | If                     | Checks if post has only text (no photo or video)  | Process Media URLs       | Send Text to Channel          | (Covered by above note)                                                                                                                                    |
| Download Photo      | HTTP Request           | Downloads photo media file                         | Has Photo?               | Send Photo to Channel          | (Covered by above note)                                                                                                                                    |
| Download Video      | HTTP Request           | Downloads video media file                         | Has Video?               | Send Video to Channel          | (Covered by above note)                                                                                                                                    |
| Send Text to Channel | Telegram               | Sends translated text-only post to target channel | Text Only?               | None                         | (Covered by above note)                                                                                                                                    |
| Send Photo to Channel | Telegram               | Sends translated post with photo to target channel | Download Photo           | None                         | (Covered by above note)                                                                                                                                    |
| Send Video to Channel | Telegram               | Sends translated post with video to target channel | Download Video           | None                         | (Covered by above note)                                                                                                                                    |
| 📖 Setup Instructions | Sticky Note            | Workflow description and setup instructions       | None                    | None                         | ## Telegram Channel Content Copier & Translator Automatically copies and translates content from one Telegram channel to another... [Documentation & Support](https://n8n.io/integrations/telegram/) |
| Sticky Note         | Sticky Note            | Explanation of block 1 (content fetch and extract)| None                    | None                         | ## 1. Get the html content of the source Telegram channel ...                                                                                            |
| Sticky Note1        | Sticky Note            | Explanation of block 2 (AI translation)            | None                    | None                         | ## 2. Use AI for translation Uses OpenAI to translate the post content to the target language                                                              |
| Sticky Note2        | Sticky Note            | Explanation of block 3 (media extraction and handling) | None                 | None                         | ## 3. Extract Media URLs and Identify Media Type Telegram uses different methods for handling image and video content...                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Daily Schedule" node**  
   - Type: Schedule Trigger  
   - Set to trigger daily at 9:00 AM (adjustable).  
   - No credentials needed.

2. **Create "Set Telegram Channels" node**  
   - Type: Set  
   - Add string fields:  
     - sourceChannel: e.g., `"channel_username"` (without @)  
     - targetChannel: e.g., `"@channel_username"` (with @)  
     - targetLanguage: e.g., `"Persian"`  
     - channelSignature: e.g., `"@channel_username"` (used to remove signature from posts)  
   - Connect output of Daily Schedule → Set Telegram Channels.

3. **Create "Fetch Channel Page" node**  
   - Type: HTTP Request  
   - URL: `https://t.me/s/{{ $json.sourceChannel }}` (expression)  
   - Add header: `User-Agent` with value mimicking a modern browser (e.g., Chrome on Windows).  
   - Connect Set Telegram Channels → Fetch Channel Page.

4. **Create "Extract Post Content" node**  
   - Type: HTML Extract  
   - Operation: Extract HTML content by CSS selectors.  
   - Add extraction keys:  
     - `post`: CSS selector for last post text  
     - `postId`: CSS selector for post ID  
     - `video`: CSS selector for video src attribute  
     - `photo`: CSS selector for photo style attribute  
   - Connect Fetch Channel Page → Extract Post Content.

5. **Create "Clean Post Content" node**  
   - Type: Code (JavaScript)  
   - Code: Remove the `channelSignature` string from `post` text and trim whitespace.  
   - Connect Extract Post Content → Clean Post Content.

6. **Create "Translate Text" node**  
   - Type: OpenAI (Langchain)  
   - Model: GPT-4o-mini  
   - Message content: Dynamic prompt to translate `post` to `targetLanguage` with instructions to output only translated text.  
   - Credentials: Configure OpenAI API credentials.  
   - Connect Clean Post Content → Translate Text.

7. **Create "Process Media URLs" node**  
   - Type: Code (JavaScript)  
   - Code: Parse photo URL from CSS style attribute string to extract direct URL.  
   - Connect Translate Text → Process Media URLs.

8. **Create conditional nodes:**  
   - "Has Photo?" (If node) → condition: `photo` is non-empty string.  
   - "Has Video?" (If node) → condition: `video` is non-empty string.  
   - "Text Only?" (If node) → condition: both `photo` and `video` are empty strings.  
   - Connect Process Media URLs → Has Photo?, Has Video?, Text Only? (all three in parallel).

9. **Create "Download Photo" node**  
   - Type: HTTP Request  
   - URL: Use extracted `photo` URL.  
   - Timeout: 30 seconds.  
   - Connect Has Photo? (true) → Download Photo.

10. **Create "Download Video" node**  
    - Type: HTTP Request  
    - URL: Use extracted `video` URL.  
    - Timeout: 60 seconds.  
    - Connect Has Video? (true) → Download Video.

11. **Create "Send Text to Channel" node**  
    - Type: Telegram  
    - Operation: sendMessage (text only)  
    - Text: Concatenate translated message + channel signature.  
    - Chat ID: Use `targetChannel` from settings.  
    - Disable Telegram’s append attribution.  
    - Connect Text Only? (true) → Send Text to Channel.

12. **Create "Send Photo to Channel" node**  
    - Type: Telegram  
    - Operation: sendPhoto  
    - Chat ID: `targetChannel`  
    - Use binary data from Download Photo as photo.  
    - Caption: translated message + channel signature.  
    - Credentials: Telegram Bot API (ensure bot has permissions).  
    - Connect Download Photo → Send Photo to Channel.

13. **Create "Send Video to Channel" node**  
    - Type: Telegram  
    - Operation: sendVideo  
    - Chat ID: `targetChannel`  
    - Use binary data from Download Video as video.  
    - Caption: translated message + channel signature.  
    - Credentials: Telegram Bot API (ensure bot has permissions).  
    - Connect Download Video → Send Video to Channel.

14. **Add Sticky Notes** (optional but recommended)  
    - Add notes describing each block’s purpose and setup instructions. Include a general setup sticky note with required credentials and links.

15. **Configure Credentials**  
    - Telegram Bot API credentials for sending messages/media.  
    - OpenAI API credentials for translation.

16. **Test Workflow**  
    - Trigger manually or wait for scheduled run.  
    - Verify that posts are fetched, translated, media downloaded, and reposted correctly.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                              | Context or Link                                          |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------|
| Automatically copies and translates content from one Telegram channel to another, supporting text, images, and videos.                                                   | Workflow purpose                                         |
| Requires Telegram Bot API credentials with permissions to send messages and media in target channel.                                                                      | Telegram integration                                     |
| Requires OpenAI API credentials for GPT-4o-mini model usage.                                                                                                             | OpenAI integration                                       |
| Schedule Trigger can be adjusted to different times depending on desired update frequency.                                                                                | Scheduling flexibility                                   |
| Telegram channel HTML structure may change, requiring CSS selector updates in Extract Post Content node.                                                                  | Maintenance consideration                               |
| Workflow includes clear setup instructions and documentation link for Telegram integration: https://n8n.io/integrations/telegram/                                        | Documentation & support                                  |

---

This detailed reference document enables advanced users and AI agents to fully understand, reproduce, and maintain the Telegram content copying and translation workflow.