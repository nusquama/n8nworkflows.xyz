Download TikTok Videos without Watermarks and Upload to Google Drive

https://n8nworkflows.xyz/workflows/download-tiktok-videos-without-watermarks-and-upload-to-google-drive-3146


# Download TikTok Videos without Watermarks and Upload to Google Drive

### 1. Workflow Overview

This workflow automates the process of downloading TikTok videos **without watermarks** and optionally uploading them to Google Drive with public sharing enabled. It is designed for content creators, social media managers, digital marketers, and researchers who require original TikTok videos for analysis, repurposing, or archiving.

The workflow is logically divided into four main blocks:

- **1.1 Input Reception and Trigger:** Manual trigger to start the workflow and provide the TikTok video URL.
- **1.2 Fetch TikTok Video Page Data:** HTTP request to retrieve the full HTML content of the TikTok video page, including session cookies.
- **1.3 Extract Raw Video URL:** Code node parses the HTML to extract the direct URL of the original video file without watermark.
- **1.4 Download and (Optional) Upload:** HTTP request downloads the video file using extracted URL and cookies; optionally, the video is uploaded to Google Drive with public sharing permissions set.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Trigger

- **Overview:**  
  This block initiates the workflow manually and provides the TikTok video URL to process.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the workflow on user command.  
    - Configuration: No parameters; simply triggers the workflow.  
    - Inputs: None  
    - Outputs: Triggers the next node to fetch TikTok page data.  
    - Edge Cases: User must manually trigger; no URL input parameter here, URL is hardcoded in the HTTP Request node.  
    - Version: n8n v1+ compatible.

---

#### 2.2 Fetch TikTok Video Page Data

- **Overview:**  
  Retrieves the full HTML content of the TikTok video page, including HTTP headers and cookies, which are necessary for subsequent scraping and video download.

- **Nodes Involved:**  
  - Get TikTok Video Page Data  
  - Sticky Note (explaining usage)

- **Node Details:**

  - **Get TikTok Video Page Data**  
    - Type: HTTP Request  
    - Role: Fetches TikTok video page HTML and headers.  
    - Configuration:  
      - URL: Hardcoded TikTok video URL (e.g., https://www.tiktok.com/@randomspamvideos25/video/7251387037834595630)  
      - Method: GET (default)  
      - Response: Full HTTP response as text  
      - Headers: Custom User-Agent mimicking Chrome browser on Windows  
      - Sends headers to receive cookies and session data  
    - Inputs: Trigger from manual node  
    - Outputs: Full HTML page content and HTTP headers (including cookies)  
    - Edge Cases:  
      - URL must be valid TikTok video URL format  
      - TikTok may block requests if User-Agent or headers are missing or changed  
      - Network errors or rate limiting possible  
    - Version: HTTP Request node v4.2

  - **Sticky Note (1)**  
    - Content: Explains how to replace the URL with the desired TikTok video URL and describes output (HTML + cookies).  
    - Context: Guides user on input URL format and node purpose.

---

#### 2.3 Extract Raw Video URL

- **Overview:**  
  Parses the fetched HTML to locate the embedded JSON data containing the direct URL to the original TikTok video without watermark.

- **Nodes Involved:**  
  - Scrape raw video URL  
  - Sticky Note (explaining logic)

- **Node Details:**

  - **Scrape raw video URL**  
    - Type: Code (JavaScript)  
    - Role: Extracts the raw video URL and cookies from HTML response.  
    - Configuration:  
      - JavaScript code uses regex to find `<script id="__UNIVERSAL_DATA_FOR_REHYDRATION__" type="application/json">` tag  
      - Parses JSON inside this script tag  
      - Navigates JSON structure to find `playAddr` URL (raw video URL)  
      - Extracts cookies from HTTP headers for authenticated video download  
      - Throws errors if HTML or JSON data is missing or malformed  
    - Inputs: Output from HTTP Request node (HTML + headers)  
    - Outputs: JSON object containing `videoUrl` and concatenated `cookies` string  
    - Key Expressions:  
      - Regex to extract JSON string  
      - JSON path: `__DEFAULT_SCOPE__["webapp.video-detail"].itemInfo.itemStruct.video.playAddr`  
    - Edge Cases:  
      - HTML structure changes on TikTok may break regex or JSON path  
      - Missing or malformed JSON causes errors  
      - Cookies may be absent or incomplete, causing download failures  
    - Version: Code node v2

  - **Sticky Note (2)**  
    - Content: Explains that this node parses HTML to find the raw video URL before watermark application.  
    - Context: Helps user understand the extraction logic.

---

#### 2.4 Download and (Optional) Upload

- **Overview:**  
  Downloads the original TikTok video file using the extracted URL and cookies, then optionally uploads the video to Google Drive and sets public sharing permissions.

- **Nodes Involved:**  
  - Output video file without watermark  
  - Upload to Google Drive  
  - Set file permissions to public with link  
  - Sticky Note (explaining upload step)

- **Node Details:**

  - **Output video file without watermark**  
    - Type: HTTP Request  
    - Role: Downloads the video file as a binary file stream.  
    - Configuration:  
      - URL: Dynamic expression referencing extracted `videoUrl`  
      - Method: GET  
      - Response: File (binary)  
      - Headers:  
        - User-Agent mimics Chrome browser  
        - Referer set to TikTok homepage  
        - Accept headers set for video content types  
        - Cookie header set with extracted cookies from previous node  
      - Allows unauthorized certificates (for HTTPS)  
    - Inputs: Output from code node (videoUrl + cookies)  
    - Outputs: Binary video file data  
    - Edge Cases:  
      - Invalid or expired cookies cause download failure  
      - Network errors or TikTok blocking requests possible  
      - Video URL may expire or be invalidated  
    - Version: HTTP Request node v4.2

  - **Upload to Google Drive**  
    - Type: Google Drive node  
    - Role: Uploads the downloaded video file to Google Drive.  
    - Configuration:  
      - File name: Extracted from TikTok video ID in URL, e.g., `Video_ID.mp4`  
      - Drive: "My Drive" (default)  
      - Folder: Root folder by default  
      - Credentials: Google Drive OAuth2 credentials required  
      - Input: Binary video file from previous node  
    - Inputs: Binary video file from HTTP Request node  
    - Outputs: Metadata of uploaded file including file ID  
    - Edge Cases:  
      - Missing or invalid Google Drive credentials cause auth errors  
      - API quota limits or network errors possible  
      - Folder permissions or drive access restrictions  
    - Version: Google Drive node v3

  - **Set file permissions to public with link**  
    - Type: Google Drive node  
    - Role: Sets sharing permissions on uploaded file to allow public access via link.  
    - Configuration:  
      - Operation: Share  
      - File ID: Dynamic from uploaded file metadata  
      - Permissions: Role = writer, Type = anyone, Allow file discovery = true  
      - Credentials: Same Google Drive OAuth2 credentials  
    - Inputs: Output from Upload to Google Drive node  
    - Outputs: Confirmation of permission change  
    - Edge Cases:  
      - Permission errors if file ID invalid or insufficient rights  
      - Google Drive API limits or errors  
    - Version: Google Drive node v3

  - **Sticky Note (3)**  
    - Content: Explains optional upload to Google Drive, naming convention, and prerequisite Google Drive API setup with OAuth credentials.  
    - Context: Guides user on enabling Google Drive API and configuring credentials.

---

### 3. Summary Table

| Node Name                       | Node Type           | Functional Role                              | Input Node(s)                  | Output Node(s)                      | Sticky Note                                                                                              |
|--------------------------------|---------------------|----------------------------------------------|-------------------------------|-----------------------------------|--------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’   | Manual Trigger      | Starts workflow manually                      | None                          | Get TikTok Video Page Data         |                                                                                                        |
| Get TikTok Video Page Data      | HTTP Request        | Fetches TikTok video page HTML and cookies   | When clicking ‘Test workflow’ | Scrape raw video URL               | ## 1. Load the video page: Replace URL with desired TikTok video URL. Outputs HTML + cookies.          |
| Scrape raw video URL            | Code                | Extracts raw video URL and cookies from HTML | Get TikTok Video Page Data     | Output video file without watermark | ## 2. Find the raw video URL: Parses HTML to find video URL before watermark.                           |
| Output video file without watermark | HTTP Request    | Downloads original video file without watermark | Scrape raw video URL           | Upload to Google Drive             | ## 3. Output video file without watermark: Uses cookies to access original video file.                 |
| Upload to Google Drive          | Google Drive        | Uploads video file to Google Drive            | Output video file without watermark | Set file permissions to public with link | ## (Optional) Upload video to Google Drive: Save as Video_ID.mp4. Requires Google Drive API & OAuth.   |
| Set file permissions to public with link | Google Drive | Sets public sharing permissions on uploaded file | Upload to Google Drive         | None                              |                                                                                                        |
| Sticky Note                    | Sticky Note         | Explains step 1 (Load video page)             | None                          | None                             | ## 1. Load the video page: Replace URL with desired TikTok video URL. Outputs HTML + cookies.          |
| Sticky Note1                   | Sticky Note         | Explains step 2 (Find raw video URL)           | None                          | None                             | ## 2. Find the raw video URL: Parses HTML to find video URL before watermark.                           |
| Sticky Note2                   | Sticky Note         | Explains step 3 (Output video file)            | None                          | None                             | ## 3. Output video file without watermark: Uses cookies to access original video file.                 |
| Sticky Note3                   | Sticky Note         | Explains optional upload to Google Drive       | None                          | None                             | ## (Optional) Upload video to Google Drive: Save as Video_ID.mp4. Requires Google Drive API & OAuth.   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a **Manual Trigger** node named `When clicking ‘Test workflow’`.  
   - No parameters needed.

2. **Create HTTP Request Node to Fetch TikTok Video Page**  
   - Add an **HTTP Request** node named `Get TikTok Video Page Data`.  
   - Set **HTTP Method** to `GET`.  
   - Set **URL** to the TikTok video URL you want to download, e.g., `https://www.tiktok.com/@Username/video/Video_ID`.  
   - Under **Options > Response**, enable **Full Response** and set **Response Format** to `Text`.  
   - Under **Headers**, add a header:  
     - Name: `User-Agent`  
     - Value: `Mozilla/5.0 (Windows NT 10.0; Win64; x64) Chrome/91.0.4472.124`  
   - Connect output of Manual Trigger node to this node.

3. **Create Code Node to Extract Raw Video URL**  
   - Add a **Code** node named `Scrape raw video URL`.  
   - Use JavaScript with the following logic:  
     - Extract the HTML content from the previous node’s response.  
     - Use regex to find the `<script id="__UNIVERSAL_DATA_FOR_REHYDRATION__" type="application/json">` tag.  
     - Parse the JSON inside this script tag.  
     - Navigate to `__DEFAULT_SCOPE__["webapp.video-detail"].itemInfo.itemStruct.video.playAddr` to get the raw video URL.  
     - Extract cookies from HTTP headers for authenticated requests.  
     - Return JSON with `videoUrl` and `cookies`.  
   - Connect output of HTTP Request node to this node.

4. **Create HTTP Request Node to Download Video File**  
   - Add an **HTTP Request** node named `Output video file without watermark`.  
   - Set **HTTP Method** to `GET`.  
   - Set **URL** to expression: `{{$json["videoUrl"]}}` (dynamic URL from previous node).  
   - Under **Options > Response**, set **Response Format** to `File`.  
   - Under **Headers**, add the following headers:  
     - `User-Agent`: `Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36`  
     - `Referer`: `https://www.tiktok.com/`  
     - `Accept`: `video/mp4,video/webm,video/*;q=0.9,application/octet-stream;q=0.8`  
     - `Accept-Language`: `en-US,en;q=0.5`  
     - `Connection`: `keep-alive`  
     - `Cookie`: Expression `{{$json["cookies"]}}` (cookies from previous node)  
   - Enable **Allow Unauthorized Certificates** if needed.  
   - Connect output of Code node to this node.

5. **(Optional) Create Google Drive Upload Node**  
   - Add a **Google Drive** node named `Upload to Google Drive`.  
   - Set **Operation** to `Upload`.  
   - Set **File Name** to expression extracting video ID from URL, e.g.:  
     ```js
     {{$node["Get TikTok Video Page Data"].parameter.url.match(/\/video\/(\d+)/)[1] + ".mp4"}}
     ```  
   - Set **Drive** to `My Drive`.  
   - Set **Folder** to `root` or desired folder ID.  
   - Connect binary data input from previous HTTP Request node (video file).  
   - Configure Google Drive OAuth2 credentials with Client ID and Secret from Google Cloud Console.  
   - Connect output of HTTP Request node to this node.

6. **(Optional) Create Google Drive Permission Node**  
   - Add a **Google Drive** node named `Set file permissions to public with link`.  
   - Set **Operation** to `Share`.  
   - Set **File ID** to expression: `{{$json["id"]}}` (from upload node output).  
   - Set **Permissions**: Role = `writer`, Type = `anyone`, Allow File Discovery = `true`.  
   - Use same Google Drive OAuth2 credentials.  
   - Connect output of Upload to Google Drive node to this node.

7. **Connect Nodes in Sequence:**  
   - Manual Trigger → Get TikTok Video Page Data → Scrape raw video URL → Output video file without watermark → Upload to Google Drive → Set file permissions to public with link.

8. **Test Workflow:**  
   - Trigger manually and verify video downloads without watermark and uploads to Google Drive with public access.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                      |
|---------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Workflow requires replacing the example TikTok URL with the actual video URL you want to download.            | Input URL in `Get TikTok Video Page Data` HTTP Request node.                                        |
| Google Drive upload is optional and requires enabling Google Drive API in Google Cloud Console.                | https://console.cloud.google.com/apis/api/drive.googleapis.com/overview                             |
| Google Drive OAuth2 credentials (Client ID and Secret) must be configured in n8n credentials for upload nodes. | n8n credential setup for Google Drive OAuth2                                                        |
| TikTok page structure or video URL extraction method may change, requiring updates to the regex or JSON path. | Maintenance note for the `Scrape raw video URL` code node.                                         |
| User-Agent header mimics Chrome browser to avoid TikTok blocking requests.                                     | Important for HTTP Request nodes to succeed.                                                       |
| Video download depends on valid cookies extracted from TikTok page response headers.                           | Critical for successful video file retrieval without watermark.                                    |
| This workflow can be extended with webhooks or scheduling nodes for automation and integration with other apps.| Suggested customization ideas for advanced users.                                                 |

---

This documentation provides a complete, structured understanding of the TikTok video downloader workflow, enabling reproduction, modification, and troubleshooting.