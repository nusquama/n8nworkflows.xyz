Repurpose YouTube Videos Across Social Media with Gemini AI and ContentDrips

https://n8nworkflows.xyz/workflows/repurpose-youtube-videos-across-social-media-with-gemini-ai-and-contentdrips-8338


# Repurpose YouTube Videos Across Social Media with Gemini AI and ContentDrips

---

# Comprehensive Reference Document  
**Workflow Title:** Repurpose YouTube Videos Across Social Media with Gemini AI and ContentDrips

---

### 1. Workflow Overview

This workflow automates the repurposing of YouTube videos by detecting new uploads on a specified YouTube channel and distributing content updates across multiple social media platforms using SocialBu. It integrates AI-powered content generation with Google Gemini to create descriptions and ContentDrips to generate promotional images. The workflow also manages YouTube webhook subscriptions via PubSubHubbub, validates webhook challenges, logs video data, uploads media, posts updates, and sends success or error notifications to Discord.

**Target Use Cases:**  
- Content creators or marketers wanting to automate the distribution of new YouTube video announcements across social platforms.  
- Teams seeking to maintain synchronized social updates without manual content writing or image creation.  
- Users wanting integration of AI-generated content descriptions and branded images into social media marketing workflows.

**Logical Blocks:**  
- **1.1 YouTube Subscription and Webhook Management**: Setup and validate YouTube channel notification subscriptions using PubSubHubbub.  
- **1.2 YouTube Video Fetching & New Video Detection**: Periodically read channel RSS feed, extract latest video metadata, and detect new uploads.  
- **1.3 AI Content Generation (Google Gemini)**: Generate short video descriptions and social media post text from video metadata using Google Gemini AI.  
- **1.4 Image Generation (ContentDrips)**: Create branded promotional images based on AI-generated descriptions and video thumbnails.  
- **1.5 Social Media Posting via SocialBu API**: Authenticate with SocialBu, upload images, create scheduled posts across connected social accounts.  
- **1.6 Notifications and Error Handling**: Send success and error messages to Discord channels, handle API errors and retries.

---

### 2. Block-by-Block Analysis

---

#### 2.1 YouTube Subscription and Webhook Management

- **Overview:**  
Manages subscription to YouTube channel notifications via PubSubHubbub, handles verification challenges from YouTube, and maintains subscription validity by re-subscribing periodically.

- **Nodes Involved:**  
  - 📡 Subscribe to YouTube Notifications (HTTP Request)  
  - 🎣 YouTube Webhook Listener (Webhook)  
  - 🔐 Validate Security Token (Code)  
  - ✅ Send Verification Response (Respond to Webhook)  
  - Run every 5 days (Schedule Trigger)  
  - When clicking ‘Execute workflow’ (Manual Trigger, disabled)  
  - Sticky Notes: 🔐 Subscription Setup, 🎯 Processing Pipeline, Sticky Note, etc.

- **Node Details:**

  1. **📡 Subscribe to YouTube Notifications**  
     - Type: HTTP Request (POST)  
     - Sends subscription request to Google's PubSubHubbub endpoint with parameters: callback URL (n8n webhook), topic URL (YouTube channel feed), verify token, secret, lease seconds.  
     - Content-Type: application/x-www-form-urlencoded  
     - Disabled by default; intended for manual or scheduled execution every ~5 days to maintain subscription.  
     - Potential failures: network errors, invalid callback URLs, token mismatches.

  2. **🎣 YouTube Webhook Listener**  
     - Type: Webhook node (receives HTTP requests from YouTube)  
     - Path: youtube-trigger  
     - Accepts both GET (verification challenge) and POST (notifications) methods.  
     - Responds to verification requests via downstream nodes.  
     - Potential failures: incorrect webhook path configuration, invalid requests.

  3. **🔐 Validate Security Token**  
     - Type: Code node (JavaScript)  
     - Reads query parameters from webhook GET requests: hub.verify_token, hub.challenge, hub.mode  
     - Validates that verify_token matches expected ("shayan_ali_bakhsh") and mode is "subscribe"  
     - Returns JSON containing challenge for valid verification or "Forbidden" message with status 403 otherwise.  
     - Edge case: Invalid or missing tokens cause verification failure.

  4. **✅ Send Verification Response**  
     - Type: Respond to Webhook  
     - Sends back the challenge string as plain text response to verify webhook ownership.  
     - Works only after successful token validation.

  5. **Run every 5 days**  
     - Type: Schedule Trigger  
     - Triggers subscription request node every 5 days to renew YouTube PubSubHubbub subscription.

- **Sticky Notes:**  
  - Explain manual subscription setup, parameters needed (callback URL, channel feed URL, verify token), and subscription duration limitations.  
  - Highlight that subscription must be run manually the first time and then periodically due to 10-day max lease.

---

#### 2.2 YouTube Video Fetching & New Video Detection

- **Overview:**  
Periodically fetches the YouTube channel’s RSS feed to retrieve video metadata, extracts key video information, and determines if a new video has been uploaded since the last check.

- **Nodes Involved:**  
  - Run every 10 minutes (Schedule Trigger)  
  - Fetch Youtube Channel Videos (RSS Feed Read)  
  - Organize Items (Code)  
  - If new video (If node)  
  - Stop (NoOp)

- **Node Details:**

  1. **Run every 10 minutes**  
     - Type: Schedule Trigger  
     - Triggers the workflow every 10 minutes.

  2. **Fetch Youtube Channel Videos**  
     - Type: RSS Feed Read  
     - Reads RSS feed URL of the YouTube channel (https://www.youtube.com/feeds/videos.xml?channel_id=UC6ZligwnOYKDjgKFHEPTXzg)  
     - Parses media:group custom fields to extract enriched video metadata.

  3. **Organize Items**  
     - Type: Code node  
     - Extracts the latest video item from RSS feed's first entry.  
     - Parses title, link, author, publish date, video ID (stripping "yt:video:" prefix), description, thumbnail URL (adjusts quality), views, rating and rating count.  
     - Compares current video ID against last stored video ID in global static data.  
     - If video ID differs or first run, marks `new: true` and updates stored video ID; otherwise `new: false`.  
     - Edge cases: missing fields, malformed data, static data persistence issues.

  4. **If new video**  
     - Type: If node  
     - Checks if `new` flag is true to determine whether to proceed with content generation or stop workflow.

  5. **Stop**  
     - Type: NoOp  
     - Stops workflow execution if no new video detected.

- **Sticky Notes:**  
  - Instructions on how to obtain RSS feed link for YouTube channel.  
  - Explanation of logic to detect the latest video based on stored last video ID.

---

#### 2.3 AI Content Generation (Google Gemini)

- **Overview:**  
Generates a concise video description and social media post text from the latest YouTube video’s metadata using the Google Gemini AI language model.

- **Nodes Involved:**  
  - Generate Description (Google Gemini node)  
  - Organaize Output (Code)  
  - Sticky Note4, Sticky Note9 (informational)

- **Node Details:**

  1. **Generate Description**  
     - Type: Google Gemini (LangChain) AI node  
     - Model: "models/gemini-2.5-flash"  
     - Input: JSON containing video title, description, and link extracted from RSS feed.  
     - Prompt instructs AI to produce:  
       - A short description (1 sentence, ~10 words) summarizing the video.  
       - A casual announcement paragraph mentioning "new video is out," summarizing content, and including the YouTube link.  
     - Output format: JSON with keys `short_description` and `post_text`.  
     - Retry enabled on failure.  
     - Potential failures: API errors, rate limits, parsing errors in output.

  2. **Organaize Output**  
     - Type: Code node  
     - Parses the AI’s raw text output, cleans markdown JSON fences, attempts JSON parse.  
     - On failure, returns an object with error info and raw text for debugging.  
     - Key variables: `short_description`, `post_text`.  
     - Edge cases: improperly formatted AI responses.

---

#### 2.4 Image Generation (ContentDrips)

- **Overview:**  
Creates a branded promotional image using ContentDrips API based on the AI-generated short description and the video thumbnail.

- **Nodes Involved:**  
  - Content drips Create Image (ContentDrips node)  
  - Download Image (HTTP Request)  
  - Upload PDF (HTTP Request)  
  - Sticky Note5, Sticky Note10 (informational)

- **Node Details:**

  1. **Content drips Create Image**  
     - Type: ContentDrips node  
     - Operation: Generate graphic synchronously using template ID 154452.  
     - Inputs:  
       - `description` from AI-generated `short_description`.  
       - `thumbnail` URL from YouTube video data.  
       - Branding info (bio, name, handle, website, avatar).  
     - Includes branding on image.  
     - Polls API with max wait 10 seconds, polls every 10 seconds.  
     - Credentials: ContentDrips API key.  
     - Error handling: Continue on error output.  
     - Potential failures: API key invalid, template issues, network timeouts.

  2. **Download Image**  
     - Type: HTTP Request  
     - Downloads generated image from ContentDrips export URL as binary file.  
     - Output stored as binary data named "image.png".  
     - Potential failures: URL broken, network errors.

  3. **Upload PDF**  
     - Type: HTTP Request (PUT)  
     - Uploads downloaded image binary to SocialBu media upload URL (signed_url).  
     - Sets Content-Type to image/png and ACL to private.  
     - Input: binary data "image.png".  
     - Potential failures: upload authorization errors, network issues.

---

#### 2.5 Social Media Posting via SocialBu API

- **Overview:**  
Authenticates with SocialBu API, retrieves connected social accounts, uploads media, creates scheduled posts with generated content, and logs out.

- **Nodes Involved:**  
  - SocialBu Access Token (HTTP Request)  
  - Get Accounts (HTTP Request)  
  - Get Upload URL (HTTP Request)  
  - Check if Image Uploaded (Code)  
  - Image Uploaded (If)  
  - Merge (Merge node)  
  - Organize Object (Code)  
  - Post to SocialBu Connected Accounts (HTTP Request)  
  - Destroy Access Token (HTTP Request)  
  - Success Message (Discord)  
  - Stop and Error (StopAndError)  
  - Error Message (Discord)  
  - Error Trigger (Error Trigger node)  
  - Sticky Note3, Sticky Note11, Sticky Note12 (informational)

- **Node Details:**

  1. **SocialBu Access Token**  
     - Type: HTTP Request (POST)  
     - Authenticates with SocialBu API using user email and password (stored securely).  
     - Retrieves bearer token used for subsequent API calls.  
     - Potential failures: invalid credentials, API downtime.

  2. **Get Accounts**  
     - Type: HTTP Request (GET)  
     - Retrieves list of connected social media accounts.  
     - Requires bearer token in Authorization header.  
     - Outputs JSON array of accounts with details like active status and supported attachment types.

  3. **Get Upload URL**  
     - Type: HTTP Request (POST)  
     - Requests a signed upload URL from SocialBu for media upload.  
     - Sends JSON body specifying file name and MIME type (image/png).  
     - Requires Authorization header.

  4. **Check if Image Uploaded**  
     - Type: Code node  
     - Polls SocialBu API up to 4 times with 5-second delay intervals to check upload status using returned key.  
     - Returns success boolean.  
     - Edge case: network timeouts, false negatives.

  5. **Image Uploaded**  
     - Type: If node  
     - Proceeds if upload successful; otherwise sends workflow to Stop and Error node.

  6. **Merge**  
     - Type: Merge node  
     - Combines data from media upload and account retrieval for post preparation.

  7. **Organize Object**  
     - Type: Code node  
     - Formats publish time (current UTC + 1 minute).  
     - Extracts upload_token from upload response.  
     - Filters active accounts that accept PNG attachments.  
     - Constructs JSON payload for posting.

  8. **Post to SocialBu Connected Accounts**  
     - Type: HTTP Request (POST)  
     - Sends post creation request with accounts list, scheduled publish time, post content (from AI), and attachments (upload token).  
     - Draft flag true for scheduling.  
     - Requires Authorization header.

  9. **Destroy Access Token**  
     - Type: HTTP Request (POST)  
     - Logs out from SocialBu API to revoke token.  
     - Requires Authorization header.

  10. **Success Message**  
      - Type: Discord node  
      - Sends a success notification message to a configured Discord server/channel via OAuth2 credentials.

  11. **Stop and Error**  
      - Type: Stop and Error node  
      - Halts workflow with error message if upload or posting fails after retries.

  12. **Error Message**  
      - Type: Discord node  
      - Sends error notifications to Discord on workflow errors such as API failures or exceptions.

  13. **Error Trigger**  
      - Listens for errors in workflow execution and triggers error message node.

- **Sticky Notes:**  
  - Explain benefits of using SocialBu for multiple platform posting and instructions to update credentials.  
  - Provide Discord OAuth2 credential setup link.

---

#### 2.6 Notifications and Error Handling

- **Overview:**  
Sends notifications to Discord regarding workflow success or failure, enabling real-time monitoring of automation status.

- **Nodes Involved:**  
  - Success Message (Discord)  
  - Error Trigger (Error Trigger)  
  - Error Message (Discord)

- **Node Details:**

  1. **Success Message**  
     - Sends a confirmation text to Discord on successful post scheduling.  
     - Uses OAuth2 authentication for Discord API.

  2. **Error Trigger**  
     - Captures any workflow errors and triggers downstream error notifications.

  3. **Error Message**  
     - Sends detailed error message including node name, execution mode, error message, workflow name and ID to Discord channel for alerting.

- **Edge Cases:**  
  - Discord API rate limits or credential issues can cause notification failures.

---

### 3. Summary Table

| Node Name                         | Node Type                   | Functional Role                                      | Input Node(s)                          | Output Node(s)                                   | Sticky Note                                                                                                   |
|----------------------------------|-----------------------------|-----------------------------------------------------|--------------------------------------|-------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger              | Manual start for subscription setup                 |                                      | 📡 Subscribe to YouTube Notifications            |                                                                                                              |
| 📋 Workflow Overview             | Sticky Note                 | Overview of YouTube Auto-Notification system        |                                      |                                                 |                                                                                                              |
| 🔐 Subscription Setup            | Sticky Note                 | Instructions for subscription setup                  |                                      |                                                 |                                                                                                              |
| 🎯 Processing Pipeline           | Sticky Note                 | Explains verification and notification flows        |                                      |                                                 |                                                                                                              |
| 📡 Subscribe to YouTube Notifications | HTTP Request           | Sends YouTube PubSubHubbub subscription request     | When clicking ‘Execute workflow’     |                                                 |                                                                                                              |
| 🎣 YouTube Webhook Listener      | Webhook                    | Receives YouTube webhook calls                       |                                      | 🔐 Validate Security Token, Log Response          |                                                                                                              |
| 🔐 Validate Security Token       | Code                       | Validates YouTube webhook verification token        | 🎣 YouTube Webhook Listener           | ✅ Send Verification Response                      |                                                                                                              |
| ✅ Send Verification Response    | Respond to Webhook         | Sends challenge response to YouTube                  | 🔐 Validate Security Token            |                                                 |                                                                                                              |
| Log Response                    | Code                       | Logs incoming webhook data                            | 🎣 YouTube Webhook Listener           |                                                 |                                                                                                              |
| Run every 5 days                 | Schedule Trigger           | Triggers subscription renewal every 5 days           |                                      | 📡 Subscribe to YouTube Notifications            |                                                                                                              |
| Stop                            | NoOp                       | Stops workflow if no new video                        | If new video                        |                                                 |                                                                                                              |
| Run every 10 minutes             | Schedule Trigger           | Periodically triggers video fetch                     |                                      | Fetch Youtube Channel Videos                      |                                                                                                              |
| Fetch Youtube Channel Videos     | RSS Feed Read              | Reads YouTube channel RSS feed                        | Run every 10 minutes                 | Organize Items                                    |                                                                                                              |
| Organize Items                  | Code                       | Extracts and compares latest video metadata           | Fetch Youtube Channel Videos         | If new video                                      |                                                                                                              |
| If new video                   | If                         | Branches workflow based on new video presence         | Organize Items                      | Generate Description, Stop                         |                                                                                                              |
| Generate Description             | Google Gemini AI           | Generates video description and post text             | If new video                       | Organaize Output                                  |                                                                                                              |
| Organaize Output                | Code                       | Parses AI output JSON                                  | Generate Description                | Content drips Create Image                         |                                                                                                              |
| Content drips Create Image       | ContentDrips               | Generates promotional image with branding             | Organaize Output                   | SocialBu Access Token                             |                                                                                                              |
| SocialBu Access Token           | HTTP Request               | Authenticates with SocialBu API                        | Content drips Create Image          | Get Accounts, Get Upload URL                      |                                                                                                              |
| Get Accounts                   | HTTP Request               | Retrieves connected social media accounts              | SocialBu Access Token              | Merge                                            |                                                                                                              |
| Get Upload URL                | HTTP Request               | Requests signed URL for image upload                    | SocialBu Access Token              | Download Image                                    |                                                                                                              |
| Download Image                | HTTP Request               | Downloads generated image                                | Get Upload URL                    | Upload PDF                                       |                                                                                                              |
| Upload PDF                   | HTTP Request               | Uploads image to SocialBu storage                        | Download Image                   | Check if Image Uploaded                           |                                                                                                              |
| Check if Image Uploaded        | Code                       | Polls SocialBu API for upload status                    | Upload PDF                      | Image Uploaded                                   |                                                                                                              |
| Image Uploaded                | If                         | Branches on upload success/failure                       | Check if Image Uploaded           | Merge or Stop and Error                           |                                                                                                              |
| Merge                          | Merge                      | Combines upload & account data                           | Image Uploaded, Get Accounts      | Organize Object                                  |                                                                                                              |
| Organize Object               | Code                       | Prepares post data payload                               | Merge                           | Post to SocialBu Connected Accounts               |                                                                                                              |
| Post to SocialBu Connected Accounts | HTTP Request           | Creates scheduled posts with media                       | Organize Object                 | Destroy Access Token                             |                                                                                                              |
| Destroy Access Token           | HTTP Request               | Revokes SocialBu API token                               | Post to SocialBu Connected Accounts | Success Message                                  |                                                                                                              |
| Success Message               | Discord                    | Sends success notification to Discord                    | Destroy Access Token             |                                                 |                                                                                                              |
| Stop and Error                | StopAndError               | Stops workflow on fatal errors                            | Image Uploaded                  |                                                 |                                                                                                              |
| Error Trigger                | Error Trigger              | Detects workflow errors                                  |                                      | Error Message                                    |                                                                                                              |
| Error Message               | Discord                    | Sends error notification to Discord                       | Error Trigger                  |                                                 |                                                                                                              |
| Log Response                | Code                       | Logs webhook notification data                            | 🎣 YouTube Webhook Listener       |                                                 |                                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node** named “Run every 10 minutes” configured to trigger every 10 minutes.

2. **Add RSS Feed Read node** “Fetch Youtube Channel Videos”  
   - URL: `https://www.youtube.com/feeds/videos.xml?channel_id=YOUR_CHANNEL_ID` (replace with target channel)  
   - Enable reading custom fields: `media:group`  

3. **Add Code node** “Organize Items” connected from RSS Feed node  
   - Extract latest video metadata (title, link, author, published date, video ID, description, thumbnail, views, ratings).  
   - Persist last video ID in global static data for comparison.  
   - Return JSON with `{ new: true/false, item: latestVideoData }` depending on whether video is new.

4. **Add If node** “If new video” checking if `$json.new === true`  
   - True branch continues; false branch connects to a NoOp “Stop” node to end workflow.

5. **Add Google Gemini node** “Generate Description” on true branch  
   - Model: `models/gemini-2.5-flash`  
   - Input messages passing video title, description, and link.  
   - Prompt instructs to generate JSON with `short_description` and `post_text`.  
   - Attach Google Gemini API credentials.

6. **Add Code node** “Organaize Output” connected from Gemini node  
   - Cleans AI output text, parses JSON, returns structured data.

7. **Add ContentDrips node** “Content drips Create Image”  
   - Operation: generate graphic sync  
   - Template ID: 154452  
   - Content updates: description (from AI), thumbnail (from video data), branding info.  
   - Attach ContentDrips API credentials.

8. **Add HTTP Request node** “SocialBu Access Token”  
   - POST to `https://socialbu.com/api/v1/auth/get_token` with JSON body containing SocialBu email and password.  
   - Save bearer token for downstream calls.

9. **Add HTTP Request node** “Get Accounts”  
   - GET to `https://socialbu.com/api/v1/accounts` with Authorization header using bearer token.

10. **Add HTTP Request node** “Get Upload URL”  
    - POST to `https://socialbu.com/api/v1/upload_media` with JSON body specifying file name and mime type.  
    - Use Authorization header.

11. **Download Image** node (HTTP Request)  
    - Download image binary from ContentDrips export URL.  
    - Set response format to file, output property “image.png”.

12. **Upload PDF** node (HTTP Request)  
    - PUT request to upload signed URL with binary image data.  
    - Set Content-Type to image/png and x-amz-acl to private.

13. **Code node** “Check if Image Uploaded”  
    - Poll SocialBu upload status API up to 4 times with delay to confirm upload success.

14. **If node** “Image Uploaded”  
    - Checks success flag; if false, sends to “Stop and Error” node with message.

15. **Merge node** combines upload and account data results.

16. **Code node** “Organize Object”  
    - Prepare JSON payload for post creation: accounts accepting PNG, upload token, publish time (current + 1 min).

17. **HTTP Request node** “Post to SocialBu Connected Accounts”  
    - POST to `https://socialbu.com/api/v1/posts` with JSON body including accounts, publish time, post content, attachments.  
    - Use bearer token in Authorization header.

18. **HTTP Request node** “Destroy Access Token”  
    - POST to `https://socialbu.com/api/v1/auth/logout` to revoke token.

19. **Discord node** “Success Message”  
    - Send success notification to configured Discord channel using OAuth2 credentials.

20. **Error Handling:**  
    - Add Error Trigger node connected to “Error Message” Discord node sending detailed error reports.

21. **YouTube PubSubHubbub subscription:**  
    - Add Schedule Trigger node “Run every 5 days”.  
    - Add HTTP Request node “📡 Subscribe to YouTube Notifications” POSTing to `https://pubsubhubbub.appspot.com/subscribe` with required form parameters (hub.callback, hub.topic, verify_token, lease_seconds).  
    - Manually trigger this node initially to register webhook.

22. **Webhook Listener:**  
    - Add Webhook node with path “youtube-trigger” accepting GET and POST.  
    - Connect to Code node “🔐 Validate Security Token” that reads query params, verifies token, returns challenge.  
    - Connect to “✅ Send Verification Response” node responding with challenge text.  
    - Also connect webhook to a logging Code node.

23. **Connect all nodes in correct execution order as per the workflow logic.**

24. **Set up all credentials:**  
    - Google Gemini API credentials for AI node.  
    - ContentDrips API key and branding info.  
    - SocialBu account email and password for API access.  
    - Discord OAuth2 credentials for notifications.

25. **Update all URLs, tokens, API keys and channel IDs as per your environment.**

---

### 5. General Notes & Resources

| Note Content                                                                                                           | Context or Link                                                                                                                   |
|------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| How to get YouTube channel RSS link: View page source, search “channel_id” to find feed URL.                           | [YouTube RSS Feed Tutorial](https://chuck.is/yt-rss/#:~:text=Hit%20CTRL%2BF%20to%20pull,preferred%20RSS%20reader%20and%20rejoice.) |
| SocialBu supports posting to Facebook, Instagram, YouTube, TikTok, Twitter (X), LinkedIn, Threads, Pinterest, and more. | [SocialBu Website](https://socialbu.com/)                                                                                        |
| ContentDrips offers many templates and image types for social media content creation with n8n integration.              | [ContentDrips Node on npm](https://www.npmjs.com/package/n8n-nodes-contentdrips)                                                 |
| Update Gemini API access token and SocialBu credentials regularly to avoid authentication failures.                     |                                                                                                                                |
| Discord OAuth2 credential setup instructions for notification nodes.                                                    | [Discord OAuth2 Setup](https://docs.n8n.io/integrations/builtin/credentials/discord/)                                              |
| Workflow author and contact for help: Shayan Ali Bakhsh on LinkedIn                                                     | [LinkedIn Profile](https://www.linkedin.com/in/shayan-khan20/)                                                                    |

---

**Disclaimer:**  
The text provided is exclusively sourced from an n8n automated workflow respecting all applicable content policies. No illegal, offensive, or protected content is included. All processed data is legal and publicly accessible.

---