Generate Short-form Clips from YouTube Videos with GPT-4o, Grok & Airtable

https://n8nworkflows.xyz/workflows/generate-short-form-clips-from-youtube-videos-with-gpt-4o--grok---airtable-11645


# Generate Short-form Clips from YouTube Videos with GPT-4o, Grok & Airtable

### 1. Workflow Overview

This workflow automates the process of repurposing long-form YouTube videos into optimized short-form clips, titles, thumbnails, and descriptions using AI models and integrations. It is designed for content creators and marketers focused on AI automation, no-code workflows, and business automation to efficiently generate engaging YouTube content assets and manage them via Airtable.

The workflow is logically divided into the following blocks:

- **1.1 Video Upload Detection & Airtable Record Creation:** Detects newly added videos in a Google Drive folder and creates a corresponding Airtable record.
- **1.2 Title Generation:** Uses GPT-4o-mini model to generate three optimized YouTube title options based on the video summary.
- **1.3 Thumbnail Generation:** Manually triggered subsystem to generate AI-optimized YouTube thumbnails using user-input thumbnail ideas and Kie.ai Nano Banana Pro with reference images from Airtable.
- **1.4 Transcript Retrieval & Formatting:** Retrieves YouTube video transcript via Apify and formats it with timestamps.
- **1.5 Clip Identification:** Uses Grok AI model to detect elite short-form clips from the full transcript with detailed captions and timestamps.
- **1.6 Description Generation:** Generates a YouTube video description with chapter timestamps based on the formatted transcript.
- **1.7 YouTube Video Metadata Update:** Updates YouTube video with generated titles and description.
- **1.8 Airtable Updates:** Updates Airtable records with generated titles, clips, thumbnails, and status.
- **1.9 Polling Loop for YouTube Captions:** Manages polling for transcript availability after YouTube upload to avoid race conditions.

---

### 2. Block-by-Block Analysis

#### 1.1 Video Upload Detection & Airtable Record Creation

**Overview:**  
Automatically triggers when a new video file is added to a specific Google Drive folder. It creates a new record in Airtable with the video file’s name and link.

**Nodes Involved:**  
- `When video is added to drive folder`  
- `Create a record`

**Node Details:**

- **When video is added to drive folder**  
  - Type: Google Drive Trigger  
  - Configuration: Triggered on file creation in a specific Google Drive folder ("Edited videos (pre upload)"), polling every minute.  
  - Input: New video file metadata.  
  - Output: Video metadata JSON.  
  - Potential failures: Google Drive API auth issues, folder permission errors, network timeouts.

- **Create a record**  
  - Type: Airtable node (create operation)  
  - Configuration: Creates a record in Airtable base “Content Factory” in the specified table, mapping “Video file name” and “Longform edited video” fields from the file metadata.  
  - Input: File name and webViewLink from previous node.  
  - Output: Airtable record creation confirmation.  
  - Failures: Airtable API quota/auth errors.

---

#### 1.2 Title Generation

**Overview:**  
Triggered when a video summary is added, this block generates three SEO-optimized YouTube titles using GPT-4o-mini model via Langchain OpenAI node.

**Nodes Involved:**  
- `Webhook`  
- `Get a record`  
- `Generate 3 titles`  
- `Add titles to record`

**Node Details:**

- **Webhook**  
  - Type: Webhook (HTTP endpoint)  
  - Configuration: Receives HTTP requests to trigger this block.  
  - Input: Contains query parameter with Airtable record ID.  
  - Output: Passes query to next node.

- **Get a record**  
  - Type: Airtable (get operation)  
  - Configuration: Retrieves record by ID from Airtable base/table containing video summary.  
  - Input: recordId from webhook query.  
  - Output: Video summary and other metadata.  
  - Failures: Record not found, Airtable auth errors.

- **Generate 3 titles**  
  - Type: Langchain OpenAI node (GPT-4o-mini)  
  - Configuration: Uses a detailed system prompt specifying niche, audience, title SEO rules, and three title formulas to create 3 title options based on video summary. Output is JSON with keys: title_1, title_2, title_3.  
  - Input: Video summary text.  
  - Output: AI-generated titles.  
  - Failures: API limits, malformed inputs, model response errors.

- **Add titles to record**  
  - Type: Airtable (update operation)  
  - Configuration: Updates the same record in Airtable, saving the 3 generated title options under respective fields, matching by a "Calculation" column.  
  - Input: Titles from previous node, existing record calculation field.  
  - Output: Updated Airtable record.  
  - Failures: Airtable write errors, mismatched record keys.

---

#### 1.3 Thumbnail Generation

**Overview:**  
Manually triggered via webhook; optimizes thumbnail prompt text with GPT-5-mini, sends prompt and reference images to Kie.ai Nano Banana Pro API to generate thumbnails, then uploads the thumbnail back to Airtable.

**Nodes Involved:**  
- `Webhook2`  
- `Get a record2`  
- `Thumbnail prompt optimizer`  
- `clean up prompt`  
- `Code in JavaScript`  
- `Create thumbnail`  
- `Wait 2 mins`  
- `Get image1`  
- `Switch4`  
- `Download image`  
- `Extract from File5`  
- `Add thumbnail to airtable`

**Node Details:**

- **Webhook2**  
  - Type: Webhook  
  - Triggered manually via Airtable button.  
  - Input: Contains recordId query parameter.

- **Get a record2**  
  - Airtable get node retrieving thumbnail idea and reference images fields.

- **Thumbnail prompt optimizer**  
  - Uses GPT-5-mini with a system prompt to optimize raw thumbnail concept into a detailed, high-converting image generation prompt tailored for Nano Banana Pro.

- **clean up prompt**  
  - Cleans and formats the optimized prompt to a single-line string without extra whitespace or line breaks.

- **Code in JavaScript**  
  - Extracts reference image URLs from Airtable record fields, cleans prompt string, builds JSON payload with prompt, images, aspect ratio, resolution, and output format for the Kie.ai API.

- **Create thumbnail**  
  - HTTP POST to Kie.ai API endpoint to create thumbnail generation job with the prepared JSON payload, using HTTP header authentication.

- **Wait 2 mins**  
  - Wait node to allow sufficient time for thumbnail generation.

- **Get image1**  
  - HTTP GET request polls Kie.ai API for job status and retrieves result JSON containing thumbnail URL.

- **Switch4**  
  - Conditional node checking if the generation result is ready or if an error/status requires waiting.

- **Download image**  
  - Downloads actual thumbnail image from Kie.ai result URL.

- **Extract from File5**  
  - Extracts binary image data from HTTP response for upload.

- **Add thumbnail to airtable**  
  - Uploads generated thumbnail image as an attachment to Airtable record, using Airtable API with authentication.  
  - Note: Requires user to replace `YOUR_BASE_ID` with actual Airtable base ID.

- Failures: Missing reference images, API rate limits, authentication failures, invalid prompt formatting, thumbnail generation errors.

---

#### 1.4 Transcript Retrieval & Formatting

**Overview:**  
After video upload to YouTube and marking in Airtable, this block retrieves the YouTube video transcript via Apify, formats it with timestamps into readable text.

**Nodes Involved:**  
- `Webhook3`  
- `Get a record4`  
- `Get a video1`  
- `Check to see if transcript is ready`  
- `Is transcript ready?`  
- `Wait 5 mins`  
- `Get transcript from video1`  
- `Formatt transcript1`

**Node Details:**

- **Webhook3**  
  - Manual trigger with recordId query parameter to start transcript retrieval.

- **Get a record4**  
  - Airtable get node to fetch video metadata.

- **Get a video1**  
  - YouTube node fetching video details by ID.

- **Check to see if transcript is ready**  
  - Calls YouTube Captions API to check if auto-generated captions are available.

- **Is transcript ready?**  
  - Switch node branching based on presence of captions: proceeds if ready, otherwise waits.

- **Wait 5 mins**  
  - Wait node to delay rechecking captions, prevents API rate limit.

- **Get transcript from video1**  
  - HTTP POST to Apify YouTube transcript scraper actor with video URL, fetches transcript dataset.

- **Formatt transcript1**  
  - JavaScript node formats transcript items into HH:MM:SS.mmm timestamps with clean text, concatenated for downstream use.

- Failures: YouTube OAuth errors, missing captions, API limits, malformed transcript data.

---

#### 1.5 Clip Identification

**Overview:**  
Analyzes the formatted transcript using Grok 4.1 Fast AI model to identify elite short-form clips with captions and timestamps suitable for TikTok, Instagram, and YouTube Shorts.

**Nodes Involved:**  
- `Clipping agent1`  
- `OpenRouter Chat Model1` (AI model node)  
- `Key value pair clips`  
- `cleanly formatt clips`  
- `Update record`

**Node Details:**

- **OpenRouter Chat Model1**  
  - Uses Grok 4.1 Fast model from OpenRouter API to analyze transcript and video metadata, applying a detailed prompt to identify clips meeting strict quality and length criteria.

- **Clipping agent1**  
  - Langchain chain node passing transcript and metadata into the AI model; includes a comprehensive prompt specifying clip criteria.

- **Key value pair clips**  
  - JavaScript node parses AI JSON output into individual clip objects for Airtable.

- **cleanly formatt clips**  
  - Formats clips into readable text summary listing start/end times, titles, captions, and total count.

- **Update record**  
  - Airtable update node saves the formatted clips and marks the record status as "Complete."

- Failures: API throttling, malformed AI output, parsing errors, Airtable update errors.

---

#### 1.6 Description Generation

**Overview:**  
Generates a YouTube video description with properly formatted chapter timestamps, based on the transcript, using GPT-5-mini.

**Nodes Involved:**  
- `Description agent1`  
- `Update a video1`

**Node Details:**

- **Description agent1**  
  - Langchain OpenAI node with a system prompt instructing to fill a YouTube description template with chapter timestamps derived from transcript, ensuring formatting adheres to YouTube requirements.

- **Update a video1**  
  - YouTube node updating the uploaded video’s metadata (title and description) using OAuth credentials.

- Failures: API limits, invalid timestamp formatting, YouTube API errors.

---

#### 1.7 YouTube Video Metadata Update & Cleanup

**Overview:**  
Updates the YouTube video metadata with generated titles and description, then deletes the source video file from Google Drive to clean up storage.

**Nodes Involved:**  
- `Update a video1`  
- `Delete a file1`

**Node Details:**

- **Update a video1**  
  - Updates YouTube video title and description with AI-generated content.

- **Delete a file1**  
  - Deletes the video file from Google Drive using the file ID stored in Airtable to save space.

- Failures: YouTube API errors, Google Drive permissions.

---

#### 1.8 Polling Loop for YouTube Captions

**Overview:**  
Handles the delay between video upload and availability of captions on YouTube by polling the Captions API every 5 minutes until captions become available, then continuing the process.

**Nodes Involved:**  
- `Check to see if transcript is ready`  
- `Is transcript ready?`  
- `Wait 5 mins`

**Node Details:**  
Described above under transcript retrieval.

---

### 3. Summary Table

| Node Name                     | Node Type                     | Functional Role                                           | Input Node(s)                  | Output Node(s)               | Sticky Note                                                                                                    |
|-------------------------------|-------------------------------|-----------------------------------------------------------|-------------------------------|------------------------------|----------------------------------------------------------------------------------------------------------------|
| When video is added to drive folder | Google Drive Trigger           | Detect new video file upload in Drive                      |                               | Create a record               | ## When edited video is added to google drive folder create the record with our video in airtable              |
| Create a record                | Airtable (create)              | Create Airtable record for new video                       | When video is added to drive folder | Generate 3 titles             | ## When edited video is added to google drive folder create the record with our video in airtable              |
| Webhook                       | Webhook                       | Trigger title generation on video summary addition        |                               | Get a record                  | ## When Video summary is added this system is triggered to then generate 3 optimized titles to A/B test        |
| Get a record                  | Airtable (get)                | Retrieve video summary record                               | Webhook                       | Generate 3 titles             | ## When Video summary is added this system is triggered to then generate 3 optimized titles to A/B test        |
| Generate 3 titles             | Langchain OpenAI (GPT-4o-mini) | Generate 3 YouTube title options based on video summary    | Get a record                  | Add titles to record          | ## When Video summary is added this system is triggered to then generate 3 optimized titles to A/B test        |
| Add titles to record          | Airtable (update)             | Save generated titles to Airtable record                   | Generate 3 titles             |                              | ## When Video summary is added this system is triggered to then generate 3 optimized titles to A/B test        |
| Webhook2                      | Webhook                       | Manual trigger for thumbnail generation                    |                               | Get a record2                 | ## This system is manually triggered once you want to generate the thumbnail based on the idea you typed in Airtable "thumbnail idea" field. This system then sends that idea into our image prompt optimizer that then sends the image prompt to kie.ai to access nano banana pro |
| Get a record2                 | Airtable (get)                | Get thumbnail idea and reference images                    | Webhook2                      | Thumbnail prompt optimizer    | ## This system is manually triggered once you want to generate the thumbnail based on the idea you typed in Airtable "thumbnail idea" field. This system then sends that idea into our image prompt optimizer that then sends the image prompt to kie.ai to access nano banana pro |
| Thumbnail prompt optimizer    | Langchain OpenAI (GPT-5-mini) | Optimize raw thumbnail prompt                               | Get a record2                 | clean up prompt              | ## This system is manually triggered once you want to generate the thumbnail based on the idea you typed in Airtable "thumbnail idea" field. This system then sends that idea into our image prompt optimizer that then sends the image prompt to kie.ai to access nano banana pro |
| clean up prompt              | Set node                      | Clean optimized prompt string                              | Thumbnail prompt optimizer    | Code in JavaScript           | ## This system is manually triggered once you want to generate the thumbnail based on the idea you typed in Airtable "thumbnail idea" field. This system then sends that idea into our image prompt optimizer that then sends the image prompt to kie.ai to access nano banana pro |
| Code in JavaScript           | Code                         | Build JSON payload for Kie.ai API with prompt and images  | clean up prompt              | Create thumbnail             | ## This system is manually triggered once you want to generate the thumbnail based on the idea you typed in Airtable "thumbnail idea" field. This system then sends that idea into our image prompt optimizer that then sends the image prompt to kie.ai to access nano banana pro |
| Create thumbnail             | HTTP Request                 | Request thumbnail generation job from Kie.ai               | Code in JavaScript           | Wait 2 mins                 | ## This system is manually triggered once you want to generate the thumbnail based on the idea you typed in Airtable "thumbnail idea" field. This system then sends that idea into our image prompt optimizer that then sends the image prompt to kie.ai to access nano banana pro |
| Wait 2 mins                 | Wait                         | Wait for thumbnail generation                              | Create thumbnail             | Get image1                  | ## This system is manually triggered once you want to generate the thumbnail based on the idea you typed in Airtable "thumbnail idea" field. This system then sends that idea into our image prompt optimizer that then sends the image prompt to kie.ai to access nano banana pro |
| Get image1                  | HTTP Request                 | Poll Kie.ai API for thumbnail generation result            | Wait 2 mins                 | Switch4                     | ## This system is manually triggered once you want to generate the thumbnail based on the idea you typed in Airtable "thumbnail idea" field. This system then sends that idea into our image prompt optimizer that then sends the image prompt to kie.ai to access nano banana pro |
| Switch4                     | Switch                       | Check Kie.ai thumbnail generation status                   | Get image1                  | Download image / Wait10      | ## This system is manually triggered once you want to generate the thumbnail based on the idea you typed in Airtable "thumbnail idea" field. This system then sends that idea into our image prompt optimizer that then sends the image prompt to kie.ai to access nano banana pro |
| Download image              | HTTP Request                 | Download generated thumbnail image                         | Switch4                     | Extract from File5           | ## This system is manually triggered once you want to generate the thumbnail based on the idea you typed in Airtable "thumbnail idea" field. This system then sends that idea into our image prompt optimizer that then sends the image prompt to kie.ai to access nano banana pro |
| Extract from File5          | Extract from File            | Extract image binary for upload                            | Download image              | Add thumbnail to airtable    | ## This system is manually triggered once you want to generate the thumbnail based on the idea you typed in Airtable "thumbnail idea" field. This system then sends that idea into our image prompt optimizer that then sends the image prompt to kie.ai to access nano banana pro |
| Add thumbnail to airtable   | HTTP Request                 | Upload generated thumbnail image to Airtable               | Extract from File5           |                              | ## This system is manually triggered once you want to generate the thumbnail based on the idea you typed in Airtable "thumbnail idea" field. This system then sends that idea into our image prompt optimizer that then sends the image prompt to kie.ai to access nano banana pro |
| Webhook3                    | Webhook                     | Trigger transcript and clip generation after video upload  |                               | Get a record4               | ## Once you mark the video as "uploaded to youtube" by checking the check box in airtable this system is then triggered. This system then gets the video you uploaded with title 1 (thats on private for now) and uses and apify actor to get the transcript of the video. That transcript then gets fed down 2 different paths. 1st path is our clipping path that identifies clips and outputs the timestamps of each clip and the timestamp of the main CTA while also outputting the caption of the video. Path 2 is designed with filling in the description of the video (using the transcript to fill in the timestamp part of the description) |
| Get a record4               | Airtable (get)              | Retrieve video metadata for transcript and clip generation | Webhook3                    | Get a video1                | ## Once you mark the video as "uploaded to youtube" by checking the check box in airtable this system is then triggered. This system then gets the video you uploaded with title 1 (thats on private for now) and uses and apify actor to get the transcript of the video. That transcript then gets fed down 2 different paths. 1st path is our clipping path that identifies clips and outputs the timestamps of each clip and the timestamp of the main CTA while also outputting the caption of the video. Path 2 is designed with filling in the description of the video (using the transcript to fill in the timestamp part of the description) |
| Get a video1               | YouTube                     | Retrieve YouTube video details                             | Get a record4               | Check to see if transcript is ready | ## Once you mark the video as "uploaded to youtube" by checking the check box in airtable this system is then triggered. This system then gets the video you uploaded with title 1 (thats on private for now) and uses and apify actor to get the transcript of the video. That transcript then gets fed down 2 different paths. 1st path is our clipping path that identifies clips and outputs the timestamps of each clip and the timestamp of the main CTA while also outputting the caption of the video. Path 2 is designed with filling in the description of the video (using the transcript to fill in the timestamp part of the description) |
| Check to see if transcript is ready | HTTP Request                 | Check YouTube Captions API for transcript availability     | Get a video1               | Is transcript ready?         | # ⚠️ YouTube Captions Polling Loop (Not in Video) See sticky note for details                                    |
| Is transcript ready?        | Switch                      | Branch based on whether captions exist                     | Check to see if transcript is ready | Get transcript from video1 / Wait 5 mins | # ⚠️ YouTube Captions Polling Loop (Not in Video) See sticky note for details                                    |
| Wait 5 mins                | Wait                         | Wait 5 minutes before retrying caption check               | Is transcript ready?        | Check to see if transcript is ready | # ⚠️ YouTube Captions Polling Loop (Not in Video) See sticky note for details                                    |
| Get transcript from video1 | HTTP Request                 | Retrieve transcript from Apify                             | Is transcript ready? (true) | Formatt transcript1          | # ⚠️ YouTube Captions Polling Loop (Not in Video) See sticky note for details                                    |
| Formatt transcript1        | Code                        | Format transcript with timestamps and clean text           | Get transcript from video1 | Description agent1 / Clipping agent1 | # Once you mark the video as "uploaded to youtube" by checking the check box in airtable this system is then triggered. This system then gets the video you uploaded with title 1 (thats on private for now) and uses and apify actor to get the transcript of the video. That transcript then gets fed down 2 different paths. 1st path is our clipping path that identifies clips and outputs the timestamps of each clip and the timestamp of the main CTA while also outputting the caption of the video. Path 2 is designed with filling in the description of the video (using the transcript to fill in the timestamp part of the description) |
| Description agent1         | Langchain OpenAI (GPT-5-mini) | Generate YouTube description with chapters using transcript | Formatt transcript1        | Update a video1              | # Once you mark the video as "uploaded to youtube" by checking the check box in airtable this system is then triggered. This system then gets the video you uploaded with title 1 (thats on private for now) and uses and apify actor to get the transcript of the video. That transcript then gets fed down 2 different paths. 1st path is our clipping path that identifies clips and outputs the timestamps of each clip and the timestamp of the main CTA while also outputting the caption of the video. Path 2 is designed with filling in the description of the video (using the transcript to fill in the timestamp part of the description) |
| Update a video1            | YouTube                     | Update YouTube video metadata (title, description)         | Description agent1          | Delete a file1              | # Once you mark the video as "uploaded to youtube" by checking the check box in airtable this system is then triggered. This system then gets the video you uploaded with title 1 (thats on private for now) and uses and apify actor to get the transcript of the video. That transcript then gets fed down 2 different paths. 1st path is our clipping path that identifies clips and outputs the timestamps of each clip and the timestamp of the main CTA while also outputting the caption of the video. Path 2 is designed with filling in the description of the video (using the transcript to fill in the timestamp part of the description) |
| Delete a file1             | Google Drive                | Delete the video file from Google Drive                     | Update a video1             |                              | # Once you mark the video as "uploaded to youtube" by checking the check box in airtable this system is then triggered. This system then gets the video you uploaded with title 1 (thats on private for now) and uses and apify actor to get the transcript of the video. That transcript then gets fed down 2 different paths. 1st path is our clipping path that identifies clips and outputs the timestamps of each clip and the timestamp of the main CTA while also outputting the caption of the video. Path 2 is designed with filling in the description of the video (using the transcript to fill in the timestamp part of the description) |
| Clipping agent1            | Langchain chain LLM          | Identify elite short-form clips from transcript             | Formatt transcript1 / OpenRouter Chat Model1 | Key value pair clips         | # Once you mark the video as "uploaded to youtube" by checking the check box in airtable this system is then triggered. This system then gets the video you uploaded with title 1 (thats on private for now) and uses and apify actor to get the transcript of the video. That transcript then gets fed down 2 different paths. 1st path is our clipping path that identifies clips and outputs the timestamps of each clip and the timestamp of the main CTA while also outputting the caption of the video. Path 2 is designed with filling in the description of the video (using the transcript to fill in the timestamp part of the description) |
| OpenRouter Chat Model1     | Langchain LLM Chat Model     | Grok 4.1 Fast AI model for clip identification              | Clipping agent1 (ai_languageModel input) | Clipping agent1             | # Once you mark the video as "uploaded to youtube" by checking the check box in airtable this system is then triggered. This system then gets the video you uploaded with title 1 (thats on private for now) and uses and apify actor to get the transcript of the video. That transcript then gets fed down 2 different paths. 1st path is our clipping path that identifies clips and outputs the timestamps of each clip and the timestamp of the main CTA while also outputting the caption of the video. Path 2 is designed with filling in the description of the video (using the transcript to fill in the timestamp part of the description) |
| Key value pair clips       | Code                        | Parse AI JSON response of clips into individual items       | Clipping agent1             | cleanly formatt clips        | # Once you mark the video as "uploaded to youtube" by checking the check box in airtable this system is then triggered. This system then gets the video you uploaded with title 1 (thats on private for now) and uses and apify actor to get the transcript of the video. That transcript then gets fed down 2 different paths. 1st path is our clipping path that identifies clips and outputs the timestamps of each clip and the timestamp of the main CTA while also outputting the caption of the video. Path 2 is designed with filling in the description of the video (using the transcript to fill in the timestamp part of the description) |
| cleanly formatt clips      | Code                        | Format clips into readable text summary                      | Key value pair clips        | Update record               | # Once you mark the video as "uploaded to youtube" by checking the check box in airtable this system is then triggered. This system then gets the video you uploaded with title 1 (thats on private for now) and uses and apify actor to get the transcript of the video. That transcript then gets fed down 2 different paths. 1st path is our clipping path that identifies clips and outputs the timestamps of each clip and the timestamp of the main CTA while also outputting the caption of the video. Path 2 is designed with filling in the description of the video (using the transcript to fill in the timestamp part of the description) |
| Update record              | Airtable (update)           | Save clips summary and mark record as complete               | cleanly formatt clips       |                              | # Once you mark the video as "uploaded to youtube" by checking the check box in airtable this system is then triggered. This system then gets the video you uploaded with title 1 (thats on private for now) and uses and apify actor to get the transcript of the video. That transcript then gets fed down 2 different paths. 1st path is our clipping path that identifies clips and outputs the timestamps of each clip and the timestamp of the main CTA while also outputting the caption of the video. Path 2 is designed with filling in the description of the video (using the transcript to fill in the timestamp part of the description) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Trigger Node**  
   - Type: Google Drive Trigger  
   - Configure to trigger on new file created in a specific folder (Edited videos (pre upload))  
   - Set polling interval to every minute  
   - Connect Google Drive OAuth2 credentials.

2. **Create Airtable Create Record Node ("Create a record")**  
   - Select base: Content Factory  
   - Select table: Table 1  
   - Map fields:  
     - Video file name ← file name from Drive trigger  
     - Longform edited video ← webViewLink from Drive trigger  
   - Connect Airtable token credentials.  
   - Connect output of Google Drive trigger to this node.

3. **Create Webhook Node ("Webhook")**  
   - Set unique path (e.g., a03394e6-91e5-40a5-9104-98c025e14f69)  
   - No authentication  
   - This will be triggered by Airtable automation when video summary is added.

4. **Create Airtable Get Record Node ("Get a record")**  
   - Use recordId from webhook query parameter  
   - Fetch record from same base/table  
   - Connect to Webhook node.

5. **Create Langchain OpenAI Node ("Generate 3 titles")**  
   - Model: GPT-4o-mini  
   - Use system prompt specifying niche, audience, title rules, and output JSON schema for three titles  
   - Input: Video summary from Airtable record  
   - Connect from Get a record node.

6. **Create Airtable Update Record Node ("Add titles to record")**  
   - Match by Calculation field (from previous record)  
   - Update fields: Title 1, Title 2, Title 3 with outputs from AI node  
   - Connect from Generate 3 titles node.

7. **Create Second Webhook Node ("Webhook2")**  
   - Unique path (e.g., 8bcc5357-97bc-4816-982d-fee8a591f66f)  
   - This will be manually triggered from Airtable button for thumbnail generation.

8. **Create Airtable Get Record Node ("Get a record2")**  
   - Use recordId from webhook2 query  
   - Retrieve thumbnail idea and up to 2 reference images  
   - Connect from Webhook2.

9. **Create Langchain OpenAI Node ("Thumbnail prompt optimizer")**  
   - Model: GPT-5-mini  
   - System prompt: Optimize raw thumbnail prompt into detailed Nano Banana Pro prompt with YouTube thumbnail essentials  
   - Input: Thumbnail idea field from Airtable  
   - Connect from Get a record2.

10. **Create Set Node ("clean up prompt")**  
    - Remove newlines and extra spaces from optimized prompt string to a single line  
    - Connect from Thumbnail prompt optimizer.

11. **Create JavaScript Code Node ("Code in JavaScript")**  
    - Extract 2 reference image URLs from Airtable record  
    - Build JSON payload with prompt, image_input, aspect_ratio, resolution, output_format, and callback URL  
    - Connect from clean up prompt.

12. **Create HTTP Request Node ("Create thumbnail")**  
    - POST to `https://api.kie.ai/api/v1/jobs/createTask`  
    - Use JSON body from previous node  
    - Authenticate with Kie.ai HTTP header credentials  
    - Connect from Code in JavaScript.

13. **Create Wait Node ("Wait 2 mins")**  
    - Wait 2 minutes for thumbnail generation  
    - Connect from Create thumbnail.

14. **Create HTTP Request Node ("Get image1")**  
    - Poll `https://api.kie.ai/api/v1/jobs/recordInfo` with taskId query parameter returned from createTask  
    - Authenticate with Kie.ai credentials  
    - Connect from Wait 2 mins.

15. **Create Switch Node ("Switch4")**  
    - Route:  
      - If result JSON present → Download image  
      - Else → Wait 10 seconds and retry  
    - Connect from Get image1.

16. **Create HTTP Request Node ("Download image")**  
    - Download image from Kie.ai result URL  
    - Connect from Switch4 (success path).

17. **Create Extract from File Node ("Extract from File5")**  
    - Extract binary data for image upload  
    - Connect from Download image.

18. **Create HTTP Request Node ("Add thumbnail to airtable")**  
    - POST to Airtable attachment endpoint for the specific record ID  
    - Upload binary image as PNG  
    - Authenticate with Airtable token or API key  
    - Replace placeholder for base ID with actual Airtable base ID  
    - Connect from Extract from File5.

19. **Create Third Webhook Node ("Webhook3")**  
    - Unique path (e.g., 5df46517-cec1-4563-93f3-dc51948f3551)  
    - Triggered when video is marked as uploaded to YouTube.

20. **Create Airtable Get Record Node ("Get a record4")**  
    - Retrieve record by recordId from webhook3 query  
    - Connect from Webhook3.

21. **Create YouTube Node ("Get a video1")**  
    - Get video details by YouTube video ID stored in Airtable  
    - Connect from Get a record4.

22. **Create HTTP Request Node ("Check to see if transcript is ready")**  
    - Call YouTube Captions API to check for auto-caption availability  
    - Use OAuth2 YouTube credentials  
    - Connect from Get a video1.

23. **Create Switch Node ("Is transcript ready?")**  
    - Branch if captions exist (items.length > 0)  
    - True → Get transcript from video1  
    - False → Wait 5 mins

24. **Create Wait Node ("Wait 5 mins")**  
    - Pause workflow for 5 minutes before retrying caption check  
    - Connect back to Check to see if transcript is ready.

25. **Create HTTP Request Node ("Get transcript from video1")**  
    - POST to Apify YouTube transcript scraper actor with video URL  
    - Include API key/token in headers  
    - Connect from Switch (true branch).

26. **Create Code Node ("Formatt transcript1")**  
    - Convert transcript data array to formatted string with HH:MM:SS.mmm timestamps and text  
    - Connect from Get transcript from video1.

27. **Create Langchain OpenAI Node ("Description agent1")**  
    - Use GPT-5-mini with system prompt to fill YouTube description template with chapter timestamps based on transcript  
    - Connect from Formatt transcript1.

28. **Create YouTube Update Node ("Update a video1")**  
    - Update YouTube video metadata: set title and description  
    - Use OAuth2 credentials  
    - Connect from Description agent1.

29. **Create Google Drive Node ("Delete a file1")**  
    - Delete original video file from Google Drive using stored file ID  
    - Connect from Update a video1.

30. **Create Langchain Chain LLM Node ("Clipping agent1")**  
    - Use Grok 4.1 Fast model from OpenRouter to analyze transcript and identify elite clips  
    - Use detailed prompt specifying clip criteria  
    - Connect from Formatt transcript1.

31. **Create Langchain OpenRouter Chat Model Node ("OpenRouter Chat Model1")**  
    - AI language model node providing Grok 4.1 Fast model  
    - Connect to Clipping agent1 as AI model input.

32. **Create Code Node ("Key value pair clips")**  
    - Parse AI JSON array output into individual clip JSON objects  
    - Connect from Clipping agent1.

33. **Create Code Node ("cleanly formatt clips")**  
    - Format clip objects into readable text summary with timestamps and captions  
    - Connect from Key value pair clips.

34. **Create Airtable Update Node ("Update record")**  
    - Save formatted clips text and mark record status as "Complete"  
    - Connect from cleanly formatt clips.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| 🎬 AI REPURPOSING SYSTEM overview: Fully automated pipeline for YouTube content repurposing including transcription, title generation, thumbnail creation, clipping, and description generation with ~10-15 min processing time per video. | Sticky Note at workflow start |
| Mandatory setup steps: Duplicate Airtable base, update webhook URLs in Airtable automations and buttons, connect all required credentials, create Google Drive folder, replace API keys, update Airtable base ID in thumbnail upload node. | Sticky Note at workflow start |
| Required API services and credentials: Airtable, Google Drive, Apify, OpenAI, OpenRouter (Grok), Kie.ai Nano Banana Pro, YouTube OAuth2 | Sticky Note at workflow start |
| YouTube captions polling loop added to handle delay in caption availability (10-60 min after upload). Automatically retries every 5 min until captions ready. | Sticky Note6 |
| Clip identification strict criteria ensure only high-quality, standalone, engaging clips are selected with proper context, length, and CTA inclusion. | Clipping agent1 prompt details |
| YouTube title generation uses 3 proven formula styles for SEO, curiosity, and hybrid titles with strict length and keyword rules. | Generate 3 titles system prompt |
| Thumbnail prompts optimized for Kie.ai Nano Banana Pro with detailed guidance on text overlays, colors, effects, and image usage. | Thumbnail prompt optimizer system prompt |
| YouTube description generation fills in chapter timestamps formatted exactly for YouTube, preserving template text and enhancing viewer experience. | Description agent1 system prompt |
| Workflow creator: Jordan Austin — https://www.skool.com/ai-growth-lab-4394/about?ref=5065ab28874b4325b44ef3ead8105722 | Sticky Note at workflow start |
| Video overview and setup instructions video: https://youtube.com/shorts/FBRQNmCN0F8?feature=share | Sticky Note at workflow start |

---

**Disclaimer:** This document is an analysis and detailed reference of an n8n workflow designed for legal, public, and ethical use in content automation. All APIs and credentials must be used according to their terms of service.