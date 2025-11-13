Automate AI Video Creation & Multi-Platform Publishing with GPT-4, Veo 3.1 & Blotato

https://n8nworkflows.xyz/workflows/automate-ai-video-creation---multi-platform-publishing-with-gpt-4--veo-3-1---blotato-10358


# Automate AI Video Creation & Multi-Platform Publishing with GPT-4, Veo 3.1 & Blotato

### 1. Workflow Overview

This workflow automates the creation of AI-generated videos using GPT-4 and Veo 3.1, then publishes them across multiple social media platforms via Blotato. It targets content creators and marketers who want to streamline video content generation, storage, and multi-channel distribution with minimal manual intervention.

The workflow is structured into five logical blocks:

- **1.1 Input Reception and Validation:** Monitors a Google Sheet for new video requests, parses inputs, and validates data.
- **1.2 AI Content Generation:** Uses GPT-4 to create detailed video prompts, captions, and hashtags.
- **1.3 Video Creation:** Prepares the request and calls Veo 3.1 API to generate videos from prompts and reference images.
- **1.4 Storage and Tracking:** Downloads generated videos, uploads them to Google Drive, and updates Google Sheets with metadata and status.
- **1.5 Multi-Platform Publishing:** Publishes the final videos to multiple social media platforms using the Blotato integration and updates workflow status accordingly.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Validation

- **Overview:**  
  This block listens for new rows added to a Google Sheet named "Video_Requests" and parses each entry to extract relevant fields. It validates required fields, especially URLs and text lengths, to ensure the workflow only processes well-formed requests.

- **Nodes Involved:**  
  - Google Sheets Trigger  
  - Parse Sheet Input  
  - Workflow Configuration  

- **Node Details:**

  - **Google Sheets Trigger**  
    - Type: Trigger  
    - Role: Listens for new rows added in the "Video_Requests" Google Sheet every 30 minutes.  
    - Configuration: Monitors sheet by ID and GID with OAuth2 credentials.  
    - Inputs: None (trigger node)  
    - Outputs: New Google Sheet rows as JSON  
    - Failures: OAuth token expiration, sheet access errors, API rate limits.

  - **Parse Sheet Input**  
    - Type: Code  
    - Role: Parses and validates input fields: id_video, niche, idea, three image URLs, url_final, status, and row number.  
    - Configuration: Validates text length for niche (>2 chars), idea (>5 chars), and URL format (must start with http/https). Normalizes status field.  
    - Key Expressions: Custom JavaScript for parsing and validation.  
    - Inputs: Output from Google Sheets Trigger  
    - Outputs: JSON with parsed fields plus validation status and errors if any.  
    - Edge Cases: Missing fields, invalid URLs, unrecognized status values, fewer than 3 image URLs. Throws validation errors that downstream nodes should handle.

  - **Workflow Configuration**  
    - Type: Set  
    - Role: Stores static configuration variables like API keys (OpenAI, fal.ai), Google Drive folder ID, Google Sheets ID, minimum idea length, and required photos count.  
    - Configuration: Hardcoded placeholders that must be replaced with real credentials and IDs.  
    - Inputs: Output from Parse Sheet Input  
    - Outputs: JSON enriched with configuration data.  
    - Edge Cases: Missing or incorrect API keys or IDs can cause failures in API calls downstream.

---

#### 1.2 AI Content Generation

- **Overview:**  
  This block sends the validated idea and niche to GPT-4 to generate a cinematic video prompt, social media caption, and hashtags. It then parses the AI response into structured fields for further use.

- **Nodes Involved:**  
  - GPT-4 API Call  
  - Parse GPT Response  
  - Optimize Prompt for Veo  

- **Node Details:**

  - **GPT-4 API Call**  
    - Type: LangChain OpenAI node  
    - Role: Calls GPT-4 with system prompt to act as a viral video content creator and user prompt including niche and idea.  
    - Configuration: Uses "gpt-4o-mini" model, returns JSON output with prompt, caption, and hashtags array.  
    - Inputs: Workflow Configuration output  
    - Outputs: Raw GPT response JSON  
    - Failures: OpenAI API key invalid, rate limits, timeouts, malformed prompts.

  - **Parse GPT Response**  
    - Type: Code  
    - Role: Parses the GPT response JSON or string to extract prompt, caption, and hashtags, normalizes hashtags adding leading '#' and deduplication.  
    - Configuration: JavaScript code handles various response formats and gracefully manages parsing errors.  
    - Inputs: GPT-4 API Call output  
    - Outputs: Structured JSON with prompt, caption, hashtags array, and hashtags string.  
    - Edge Cases: Invalid JSON responses, empty or malformed fields.

  - **Optimize Prompt for Veo**  
    - Type: Set  
    - Role: Appends fixed descriptive enhancements to the AI-generated prompt to tailor it for Veo 3.1 video generation (e.g., consistent character, photorealistic, duration, aspect ratio).  
    - Inputs: Parse GPT Response output  
    - Outputs: Adds `veo_prompt` field to JSON for video generation.  
    - Edge Cases: Empty prompt fields cause downstream errors.

---

#### 1.3 Video Creation

- **Overview:**  
  This block prepares the Veo API request body with the optimized prompt and validated image URLs, calls the Veo video generation API, and extracts the resulting video metadata.

- **Nodes Involved:**  
  - Prepare Veo Request Body  
  - Veo Generation1  
  - Extract Video Data  

- **Node Details:**

  - **Prepare Veo Request Body**  
    - Type: Code  
    - Role: Constructs the JSON body for Veo 3.1 API, converts Google Drive URLs to direct download URLs, validates presence and format of prompt and exactly 3 image URLs.  
    - Inputs: Output from Optimize Prompt for Veo and Parse Sheet Input  
    - Outputs: JSON with `veo_request_body` ready for API consumption.  
    - Edge Cases: Missing or invalid prompt, image URLs, URL conversion failures throw errors that stop execution.

  - **Veo Generation1**  
    - Type: HTTP Request  
    - Role: Sends POST request to Veo 3.1 API endpoint with prepared JSON body and authorization header.  
    - Configuration: 10 minute timeout (600,000 ms) to accommodate video generation time.  
    - Inputs: Prepare Veo Request Body output  
    - Outputs: Veo API JSON response with video data.  
    - Failures: API key invalid, timeout, server errors, malformed requests.

  - **Extract Video Data**  
    - Type: Code  
    - Role: Extracts video URL, content type, file size, file name, and attaches metadata from prior nodes (niche, idea, caption, hashtags, etc.) into a unified JSON structure.  
    - Inputs: Veo Generation1 output  
    - Outputs: JSON with video metadata for downstream processing.  
    - Edge Cases: Missing expected video fields, partial data may cause incomplete outputs.

---

#### 1.4 Storage and Tracking

- **Overview:**  
  This block downloads the generated video file, uploads it to Google Drive, and updates the Google Sheet with the new video URL and processing status.

- **Nodes Involved:**  
  - Download Video  
  - Google Drive Upload  
  - Google Sheets Append  

- **Node Details:**

  - **Download Video**  
    - Type: HTTP Request  
    - Role: Downloads video file from Veo-generated video URL.  
    - Configuration: Response format set to file for binary data.  
    - Inputs: Extract Video Data output  
    - Outputs: Binary video file data stream with metadata.  
    - Failures: URL invalid, network errors, timeouts.

  - **Google Drive Upload**  
    - Type: Google Drive  
    - Role: Uploads the downloaded video file to Google Drive folder specified in configuration node.  
    - Configuration: Uses OAuth2 credentials, destination folder ID from configuration, names file with user ID and timestamp.  
    - Inputs: Download Video output  
    - Outputs: Google Drive file metadata including accessible URL.  
    - Failures: OAuth token expiration, insufficient permissions, quota errors.

  - **Google Sheets Append**  
    - Type: Google Sheets  
    - Role: Updates the original Google Sheet row with the final video URL and status ("Published").  
    - Configuration: Matches rows by `id_video`, writes new status and URL fields.  
    - Inputs: Google Drive Upload output  
    - Outputs: Confirmation of sheet update.  
    - Failures: OAuth issues, API quota, row matching errors.

---

#### 1.5 Multi-Platform Publishing

- **Overview:**  
  Using the Blotato community node, this block publishes the video across multiple social media platforms (TikTok, Facebook, Instagram, LinkedIn, Twitter, YouTube) with captions and media URLs taken from the Google Sheets data.

- **Nodes Involved:**  
  - Upload Video to BLOTATO  
  - Tiktok  
  - Facebook  
  - Instagram  
  - Linkedin  
  - Twitter (X)  
  - Youtube  
  - Merge1  
  - Google Sheets Append1  

- **Node Details:**

  - **Upload Video to BLOTATO**  
    - Type: Blotato API node  
    - Role: Uploads final video URL to Blotato media resource for publishing.  
    - Inputs: Google Sheets Append (with updated URL)  
    - Outputs: Media upload confirmation and media URL for posting.  
    - Failures: API key issues, network errors.

  - **Platform Nodes (Tiktok, Facebook, Instagram, Linkedin, Twitter (X), Youtube)**  
    - Type: Blotato API node  
    - Role: Each node posts the video to a specific platform using the Blotato API, with captions and media URLs derived from Google Sheets data.  
    - Configuration: Each node uses platform-specific account IDs and parameters for privacy, notification, and content text.  
    - Inputs: Upload Video to BLOTATO output  
    - Outputs: Posting confirmation per platform.  
    - Failures: Account authentication errors, API limits, posting errors.

  - **Merge1**  
    - Type: Merge  
    - Role: Merges the outputs of all platform posting nodes to synchronize and handle parallel posting results.  
    - Configuration: "Choose Branch" mode with 6 inputs (one per platform).  
    - Inputs: All platform nodes  
    - Outputs: Combined execution result.  
    - Edge Cases: Partial failure handling and synchronization.

  - **Google Sheets Append1**  
    - Type: Google Sheets  
    - Role: Updates the Google Sheet row to "Published" status after successful posting to all platforms.  
    - Inputs: Merge1 output  
    - Outputs: Confirmation of final status update.  
    - Failures: Same as other Google Sheets nodes.

---

### 3. Summary Table

| Node Name                | Node Type                      | Functional Role                                  | Input Node(s)                 | Output Node(s)                 | Sticky Note                                                                                                 |
|--------------------------|--------------------------------|-------------------------------------------------|------------------------------|-------------------------------|-------------------------------------------------------------------------------------------------------------|
| Workflow Configuration   | Set                            | Stores API keys and workflow parameters         | Parse Sheet Input             | GPT-4 API Call                |                                                                                                             |
| Google Sheets Trigger    | Google Sheets Trigger          | Triggers on new rows in Video_Requests sheet    | None                         | Parse Sheet Input             |                                                                                                             |
| Parse Sheet Input        | Code                           | Parses and validates Google Sheets input        | Google Sheets Trigger         | Workflow Configuration        |                                                                                                             |
| GPT-4 API Call           | LangChain OpenAI Node          | Generates video prompt, caption, hashtags       | Workflow Configuration        | Parse GPT Response            |                                                                                                             |
| Parse GPT Response       | Code                           | Parses GPT JSON response into structured fields | GPT-4 API Call                | Optimize Prompt for Veo       |                                                                                                             |
| Optimize Prompt for Veo  | Set                            | Adds cinematic enhancements to prompt            | Parse GPT Response            | Prepare Veo Request Body      |                                                                                                             |
| Prepare Veo Request Body | Code                           | Builds Veo API request body, validates URLs     | Optimize Prompt for Veo       | Veo Generation1              |                                                                                                             |
| Veo Generation1          | HTTP Request                   | Calls Veo 3.1 API to generate video              | Prepare Veo Request Body      | Extract Video Data            | Timeout set to 10 minutes to accommodate video generation delays                                            |
| Extract Video Data       | Code                           | Extracts video metadata from Veo response        | Veo Generation1              | Download Video                |                                                                                                             |
| Download Video           | HTTP Request                   | Downloads generated video file                    | Extract Video Data            | Google Drive Upload           |                                                                                                             |
| Google Drive Upload      | Google Drive                   | Uploads video to Google Drive                      | Download Video                | Google Sheets Append          |                                                                                                             |
| Google Sheets Append     | Google Sheets                  | Updates sheet with video URL and status           | Google Drive Upload           | Upload Video to BLOTATO       |                                                                                                             |
| Upload Video to BLOTATO  | Blotato API Node               | Uploads video media for publishing                 | Google Sheets Append          | Tiktok, Linkedin, Facebook, Instagram, Twitter (X), Youtube |                                                                                                             |
| Tiktok                   | Blotato API Node               | Posts video to TikTok                              | Upload Video to BLOTATO       | Merge1                       |                                                                                                             |
| Linkedin                 | Blotato API Node               | Posts video to LinkedIn                            | Upload Video to BLOTATO       | Merge1                       |                                                                                                             |
| Facebook                 | Blotato API Node               | Posts video to Facebook                            | Upload Video to BLOTATO       | Merge1                       |                                                                                                             |
| Instagram                | Blotato API Node               | Posts video to Instagram                           | Upload Video to BLOTATO       | Merge1                       |                                                                                                             |
| Twitter (X)              | Blotato API Node               | Posts video to Twitter                             | Upload Video to BLOTATO       | Merge1                       |                                                                                                             |
| Youtube                  | Blotato API Node               | Posts video to YouTube                             | Upload Video to BLOTATO       | Merge1                       |                                                                                                             |
| Merge1                   | Merge Node                    | Synchronizes platform posting results             | Tiktok, Linkedin, Facebook, Instagram, Twitter (X), Youtube | Google Sheets Append1        |                                                                                                             |
| Google Sheets Append1    | Google Sheets                  | Updates sheet status to "Published"                | Merge1                       | None                         |                                                                                                             |
| Step 1 - Prerequisites   | Sticky Note                   | Describes setup instructions and prerequisites    | None                         | None                         | See detailed setup instructions including API keys and Google Sheets setup                                 |
| Step 3 - API Keys Configuration | Sticky Note             | Notes on API usage, costs, and timeout warnings   | None                         | None                         | Warns about Veo video generation timeout (10 min) and API cost estimates                                   |
| Step 4 - Workflow Activation | Sticky Note                | Pre-activation checklist and testing instructions | None                         | None                         | Checklist to verify credentials and test workflow with sample data                                         |
| Step 5 - Publishing      | Sticky Note                   | Instructions to install Blotato node and monitor workflow | None                     | None                         | Installation and activation instructions for Blotato community node in n8n with workflow monitoring advice |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Sheets:**
   - Create a sheet named **Video_Requests** with columns: `id_video`, `niche`, `idea`, `url_1`, `url_2`, `url_3`, `url_final`, `status`.
   - Populate with video requests; ensure URLs start with `http://` or `https://`.

2. **Add Google Sheets Trigger Node:**
   - Trigger on "row added" event in the "Video_Requests" sheet.
   - Set polling interval to every 30 minutes.
   - Authenticate with Google Sheets OAuth2.

3. **Add Code Node "Parse Sheet Input":**
   - Parse incoming sheet row data.
   - Validate `niche` (min 3 chars), `idea` (min 6 chars), and 3 image URLs.
   - Normalize status field.
   - Output JSON with validation flag and errors.

4. **Add Set Node "Workflow Configuration":**
   - Define variables for OpenAI API key, fal.ai API key, Google Drive folder ID, Google Sheets ID, minimum idea length, and required photos.
   - Replace placeholder values with actual credentials and IDs.

5. **Add GPT-4 API Call Node:**
   - Use LangChain OpenAI node with model `gpt-4o-mini`.
   - System prompt: "You are a viral video content creator".
   - User prompt: "Create viral video content for {{niche}} about {{idea}}. Return JSON with prompt (150-200 words), caption (50-100 words), hashtags (array of 8-10)."
   - Authenticate with OpenAI API key from credentials.

6. **Add Code Node "Parse GPT Response":**
   - Parse OpenAI output to extract prompt, caption, hashtags.
   - Normalize hashtags with leading '#', dedupe, and join as string.

7. **Add Set Node "Optimize Prompt for Veo":**
   - Append to prompt: "consistent character throughout, photorealistic quality, professional cinematography, 8 seconds duration, 9:16 aspect ratio, 24fps".
   - Output as `veo_prompt`.

8. **Add Code Node "Prepare Veo Request Body":**
   - Build JSON with `prompt` (optimized), and 3 validated image URLs (convert Google Drive URLs to direct links).
   - Set video duration to 8 seconds, aspect ratio 9:16.
   - Throw error if validation fails.

9. **Add HTTP Request Node "Veo Generation1":**
   - POST to Veo API URL: `https://fal.run/fal-ai/veo3.1/reference-to-video`.
   - Send JSON body from previous node.
   - Set headers: Authorization with fal.ai API key, Content-Type: application/json.
   - Timeout: 600,000 ms (10 minutes).

10. **Add Code Node "Extract Video Data":**
    - Extract video URL, content type, size, name.
    - Add metadata from previous steps.

11. **Add HTTP Request Node "Download Video":**
    - Download video file from extracted video URL.
    - Configure response to file/binary.

12. **Add Google Drive Node "Google Drive Upload":**
    - Upload downloaded video to configured Google Drive folder.
    - Name file using pattern: `AI_Video_{user_id}_{timestamp}.mp4`.
    - Authenticate via Google Drive OAuth2.

13. **Add Google Sheets Node "Google Sheets Append":**
    - Update corresponding row with new video URL and "Published" status.
    - Match rows by `id_video`.
    - Authenticate with Google Sheets OAuth2.

14. **Add Blotato Node "Upload Video to BLOTATO":**
    - Use Blotato API credentials.
    - Upload video media using final video URL.

15. **Add Blotato Nodes for each platform: TikTok, Facebook, Instagram, LinkedIn, Twitter (X), YouTube:**
    - For each, configure platform, accountId, and post content:
      - Text: Caption from Google Sheets.
      - Media URL: Final video URL.
    - Authenticate with Blotato API.

16. **Add Merge Node "Merge1":**
    - Combine outputs of all platform posting nodes.
    - Choose branch mode with 6 inputs.

17. **Add Google Sheets Node "Google Sheets Append1":**
    - Update Google Sheet row status to "Published" after posting.
    - Authenticate with Google Sheets OAuth2.

18. **Add Sticky Notes:**
    - Add notes describing prerequisites, API keys configuration, workflow activation, and publishing instructions.

19. **Activate Workflow:**
    - Enable all credentials.
    - Activate workflow.
    - Test with sample row in Google Sheets.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                            | Context or Link                                                                                 |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow integrates GPT-4, Veo 3.1 video AI, Google Sheets, Google Drive, and Blotato for multi-platform publishing.                                                  | Workflow purpose overview                                                                       |
| Veo video generation API may take 5-10 minutes; HTTP request timeout set accordingly (10 min).                                                                        | API call timeout warning                                                                        |
| Estimated costs per video: GPT-4 (~$0.01-0.05), Veo 3.1 (~$0.50-2.00), total ~$0.51-1.05 per video.                                                                    | Cost estimation notes                                                                           |
| Google Sheets setup requires columns: id_video, niche, idea, url_1, url_2, url_3, url_final, status. Image URLs must be valid and start with http/https.               | Input data preparation                                                                          |
| Google Drive folder ID and Google Sheets document ID must be set in Workflow Configuration for uploads and tracking.                                                  | Configuration prerequisites                                                                    |
| Blotato node installation requires enabling Community Nodes in n8n and installing `@blotato/n8n-nodes-blotato`. API keys from Blotato must be configured in credentials. | Publishing setup instructions and links (https://blotato.com/?ref=firas)                        |
| Users should regularly monitor workflow execution logs and error messages in n8n to handle failures or API limit issues.                                             | Maintenance advice                                                                             |

---

**Disclaimer:** The provided text is extracted exclusively from an automated workflow created with n8n, respecting all applicable content policies. All data processed is legal and public.