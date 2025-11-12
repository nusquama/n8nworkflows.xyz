Convert YouTube Videos to MP3 with RapidAPI, Google Drive Storage & Sheets Logging

https://n8nworkflows.xyz/workflows/convert-youtube-videos-to-mp3-with-rapidapi--google-drive-storage---sheets-logging-6722


# Convert YouTube Videos to MP3 with RapidAPI, Google Drive Storage & Sheets Logging

---
### 1. Workflow Overview

This workflow automates the conversion of YouTube videos to MP3 format using a RapidAPI service. It accepts a YouTube URL through a user-submitted form, sends the URL to a conversion API, downloads the resulting MP3, uploads it to Google Drive, and logs the process details in Google Sheets. The workflow manages asynchronous conversion status by polling and conditionally proceeding once the conversion is complete. It also computes and records the MP3 file size for user reference.

Logical blocks:

- **1.1 Input Reception:** Captures YouTube URLs from a user form.
- **1.2 Conversion Request:** Sends the URL to a YouTube-to-MP3 conversion API.
- **1.3 Status Handling and Polling:** Checks if the conversion is done; if not, waits before retrying.
- **1.4 MP3 Download:** Downloads the converted MP3 once ready.
- **1.5 Google Drive Upload:** Uploads the MP3 file to a specified Google Drive folder.
- **1.6 File Size Computation:** Calculates the MP3 file size in megabytes.
- **1.7 Logging to Google Sheets:** Records URL, status, download links, size, and date in a Google Sheet.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Captures the YouTube video URL submitted by the user via a custom form trigger, initiating the workflow.

- **Nodes Involved:**  
  - On form submission

- **Node Details:**

  - **On form submission**  
    - Type: Form Trigger  
    - Role: Workflow entry point triggered by user form submission.  
    - Configuration: Form titled "Youtube to MP3" with a single required field "URL" (placeholder: "https://youtu.be/abcdefg"). Description: "Youtube to MP3 Converter".  
    - Expressions: N/A  
    - Inputs: None (trigger node).  
    - Outputs: Passes form data (YouTube URL) to subsequent nodes.  
    - Edge cases: Missing or invalid URLs will not trigger form submission; no explicit URL validation beyond required field.  
    - Sticky Note: Explains trigger role and field requirement.

#### 2.2 Conversion Request

- **Overview:**  
  Sends a POST request to the RapidAPI YouTube-to-MP3 conversion endpoint with the submitted URL to initiate conversion.

- **Nodes Involved:**  
  - HTTP Request

- **Node Details:**

  - **HTTP Request**  
    - Type: HTTP Request  
    - Role: Calls external RapidAPI service to convert YouTube video to MP3.  
    - Configuration:  
      - URL: `https://youtube-to-mp3-downloader1.p.rapidapi.com/output.php`  
      - Method: POST  
      - Content-Type: multipart/form-data  
      - Headers: Includes `x-rapidapi-host` and `x-rapidapi-key` (API key placeholder "your key").  
      - Body: Contains parameter `url` set dynamically to the submitted YouTube URL (`{{$json.URL}}`).  
    - Expressions: URL dynamically inserted from the form data.  
    - Inputs: Receives form submission data.  
    - Outputs: Response expected to contain conversion status and MP3 URL.  
    - OnError: Configured to continue workflow even if request fails, preventing full stop.  
    - Edge cases: Authentication failure (invalid API key), network timeouts, malformed URL input, API downtime.  
    - Sticky Note: Describes API call, headers, and URL parameter usage.

#### 2.3 Status Handling and Polling

- **Overview:**  
  Evaluates the conversion status from the API response. If conversion is "done," proceeds to download; otherwise, waits and retries.

- **Nodes Involved:**  
  - If  
  - Wait  
  - Google Sheets1 (logs interim status)

- **Node Details:**

  - **If**  
    - Type: Conditional Node  
    - Role: Checks if the conversion status equals "done."  
    - Configuration: Condition where `$json.status === 'done'`.  
    - Inputs: Receives API response from HTTP Request node.  
    - Outputs:  
      - True branch: proceeds to MP3 download.  
      - False branch: proceeds to Wait node.  
    - Edge cases: Missing or malformed status field; case sensitivity enforced.  
    - Sticky Note: Summarizes filtering logic.

  - **Wait**  
    - Type: Wait  
    - Role: Pauses workflow before retrying status check or updating status logs.  
    - Configuration: No custom wait duration configured; defaults apply (likely indefinite until resumed manually or by external trigger).  
    - Inputs: From If node false branch.  
    - Outputs: Connects to Google Sheets1 for logging.  
    - Edge cases: Potential indefinite wait if conversion never completes; no timeout or retry limit configured.  
    - Sticky Note: Explains pause and retry mechanism.

  - **Google Sheets1**  
    - Type: Google Sheets (appendOrUpdate)  
    - Role: Logs interim status "Status" and URL with placeholders for size and download links marked as "N/A".  
    - Configuration:  
      - Document ID and Sheet name set to specific Google Sheet.  
      - Mapping fields: URL (from form), Status (dynamic from JSON), Created at (current date), Size/Download Link/Web View Link set to "N/A".  
      - Authentication via Google Service Account.  
    - Inputs: From Wait node.  
    - Outputs: None further.  
    - Edge cases: Google Sheets API failures, authorization errors, rate limits.  
    - Sticky Note: Clarifies purpose of logging interim status.

#### 2.4 MP3 Download

- **Overview:**  
  Downloads the MP3 file from the URL provided once conversion is completed.

- **Nodes Involved:**  
  - Download mp3

- **Node Details:**

  - **Download mp3**  
    - Type: HTTP Request (GET)  
    - Role: Downloads MP3 file from the URL in the API response.  
    - Configuration: URL set dynamically to `{{$json.url}}` from the API response. No additional headers or authentication.  
    - Inputs: True branch output of If node.  
    - Outputs: Passes the downloaded file data to Google Drive upload.  
    - Edge cases: Invalid or expired URL, HTTP errors, download timeouts.  
    - Sticky Note: Describes MP3 download initiation.

#### 2.5 Google Drive Upload

- **Overview:**  
  Uploads the downloaded MP3 file to a specified folder in Google Drive.

- **Nodes Involved:**  
  - Google Drive

- **Node Details:**

  - **Google Drive**  
    - Type: Google Drive Upload  
    - Role: Saves the MP3 file into a designated Google Drive folder using OAuth2 credentials.  
    - Configuration:  
      - Drive ID and Folder ID set to a specific Google Drive folder ("youtube to mp3").  
      - No additional options specified.  
      - OAuth2 credentials linked to a Google Drive account.  
    - Inputs: Receives MP3 file binary data from Download mp3 node.  
    - Outputs: Provides file metadata including name, size, webContentLink, and webViewLink.  
    - Edge cases: OAuth token expiration, quota exceeded, upload failures.  
    - Sticky Note: States upload functionality.

#### 2.6 File Size Computation

- **Overview:**  
  Converts the uploaded file size from bytes to megabytes (MB) for logging purposes.

- **Nodes Involved:**  
  - Code

- **Node Details:**

  - **Code**  
    - Type: JavaScript Code  
    - Role: Processes Google Drive response to convert file size from bytes to MB.  
    - Configuration:  
      - JS code functions convert bytes → KB → MB.  
      - Extracts `size` and `name` from Google Drive node JSON input.  
      - Outputs JSON with `fileName` and `fileSizeInMb` (rounded to two decimals).  
    - Inputs: Receives Google Drive upload metadata.  
    - Outputs: Passes computed size to Google Sheets node for logging.  
    - Edge cases: Missing or invalid size field; numeric conversion errors.  
    - Sticky Note: Explains conversion logic.

#### 2.7 Logging to Google Sheets

- **Overview:**  
  Appends or updates a Google Sheet with final conversion details including URL, status, size, download, and web view links.

- **Nodes Involved:**  
  - Google Sheets

- **Node Details:**

  - **Google Sheets**  
    - Type: Google Sheets (appendOrUpdate)  
    - Role: Logs finalized conversion data.  
    - Configuration:  
      - Document ID and Sheet name targeting a specific Google Sheet.  
      - Fields mapped: URL, Size (in MB), Status ("Success"), Created at (current date), Download Link, Web View Link (from Google Drive node).  
      - Authentication via Google Service Account.  
    - Inputs: Receives data from Code node with size info and Google Drive metadata.  
    - Outputs: None further.  
    - Edge cases: Authentication failures, API rate limits, data conflicts in appendOrUpdate.  
    - Sticky Note: Describes logging purpose.

---

### 3. Summary Table

| Node Name           | Node Type           | Functional Role                                   | Input Node(s)         | Output Node(s)              | Sticky Note                                                                                                                     |
|---------------------|---------------------|-------------------------------------------------|-----------------------|-----------------------------|--------------------------------------------------------------------------------------------------------------------------------|
| On form submission  | Form Trigger        | Captures YouTube URL via user-submitted form     | None                  | HTTP Request                | Triggered when the user submits a form with a YouTube video URL, required for processing.                                       |
| HTTP Request        | HTTP Request        | Sends YouTube URL to RapidAPI for MP3 conversion | On form submission    | If                         | Sends a POST request to the API endpoint `https://youtube-to-mp3-downloader1.p.rapidapi.com/output.php` to convert the video. |
| If                  | If Condition        | Checks if conversion status is "done"             | HTTP Request          | Download mp3 (true), Wait (false) | Filters the workflow to continue only if the status of the conversion is "done."                                               |
| Wait                | Wait                | Pauses before retrying or logging interim status | If (false branch)     | Google Sheets1              | Pauses the process until the conversion is done, and then moves to the next step.                                              |
| Google Sheets1      | Google Sheets       | Logs interim status and URL with placeholders     | Wait                  | None                       | Logs the initial status and URL information in the same Google Sheet before the download link is available.                    |
| Download mp3        | HTTP Request        | Downloads the converted MP3 file                   | If (true branch)      | Google Drive                | Initiates the download of the MP3 file once it is processed and ready.                                                         |
| Google Drive        | Google Drive Upload | Uploads the MP3 file to Google Drive folder       | Download mp3          | Code                       | Uploads the converted MP3 file to Google Drive.                                                                                |
| Code                | JavaScript Code     | Converts file size from bytes to MB                | Google Drive          | Google Sheets               | Converts the file size from bytes to megabytes (MB) to be included in the data logged in Google Sheets.                        |
| Google Sheets       | Google Sheets       | Logs final conversion details including size       | Code                  | None                       | Logs the successful conversion details (URL, download link, size, etc.) in a Google Sheet.                                     |
| Sticky Note         | Sticky Note         | Documentation and explanations                      | None                  | None                       | # Youtube to MP3 Converter workflow description and node-by-node explanations.                                                 |
| Sticky Note1        | Sticky Note         | On form submission explanation                      | None                  | None                       | Explains the form trigger node.                                                                                                |
| Sticky Note2        | Sticky Note         | HTTP Request node explanation                       | None                  | None                       | Describes API call, headers, and URL parameter usage.                                                                          |
| Sticky Note3        | Sticky Note         | If node explanation                                 | None                  | None                       | Explains filtering condition for conversion completion.                                                                       |
| Sticky Note4        | Sticky Note         | Download mp3 node explanation                       | None                  | None                       | Describes MP3 download initiation.                                                                                            |
| Sticky Note5        | Sticky Note         | Google Drive node explanation                       | None                  | None                       | Describes upload to Google Drive.                                                                                             |
| Sticky Note6        | Sticky Note         | Code node explanation                               | None                  | None                       | Explains size conversion logic.                                                                                               |
| Sticky Note7        | Sticky Note         | Google Sheets node explanation                      | None                  | None                       | Describes final logging in Google Sheets.                                                                                     |
| Sticky Note8        | Sticky Note         | Google Sheets1 node explanation                     | None                  | None                       | Describes interim logging in Google Sheets.                                                                                   |
| Sticky Note9        | Sticky Note         | Wait node explanation                               | None                  | None                       | Explains wait step for asynchronous process handling.                                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node named "On form submission":**  
   - Type: Form Trigger  
   - Configure form title: "Youtube to MP3"  
   - Add one form field:  
     - Label: URL  
     - Placeholder: "https://youtu.be/abcdefg"  
     - Required: true  
   - Description: "Youtube to MP3 Converter"

2. **Add an HTTP Request node named "HTTP Request":**  
   - Connect input from "On form submission" (main output).  
   - Set HTTP method to POST.  
   - URL: `https://youtube-to-mp3-downloader1.p.rapidapi.com/output.php`  
   - Content-Type: multipart/form-data  
   - In Body Parameters, add parameter "url" with value expression: `{{$json.URL}}`  
   - In Headers, add:  
     - `x-rapidapi-host`: `youtube-to-mp3-downloader1.p.rapidapi.co`  
     - `x-rapidapi-key`: your RapidAPI key (configure credential or enter key)  
   - On error: set to "Continue" to prevent workflow stop on failure.

3. **Add an If node named "If":**  
   - Connect input from "HTTP Request" (main output).  
   - Configure condition:  
     - Type: String  
     - Operation: Equals  
     - Left value: `{{$json.status}}`  
     - Right value: `done`  
   - This node splits flow based on conversion completion.

4. **Add a Wait node named "Wait":**  
   - Connect input from "If" false branch (conversion not done).  
   - Configure default wait settings (optional: set delay duration if desired).

5. **Add a Google Sheets node named "Google Sheets1" (interim logging):**  
   - Connect input from "Wait" node.  
   - Operation: Append or Update Row  
   - Provide Google Sheets credentials (Service Account recommended).  
   - Select target Spreadsheet (Document ID) and Sheet (gid=0 or "Sheet1").  
   - Map columns:  
     - URL: `{{$json.URL}}` from form data  
     - Status: `{{$json.status}}` from current data  
     - Size: `N/A`  
     - Download Link: `N/A`  
     - Web View Link: `N/A`  
     - Created at: `{{$now.format('dd-MM-yyyy')}}`

6. **Add an HTTP Request node named "Download mp3":**  
   - Connect input from "If" true branch (conversion done).  
   - Configure HTTP method: GET (default).  
   - URL: `{{$json.url}}` (MP3 download link from API response).  
   - No additional headers required.

7. **Add a Google Drive node named "Google Drive":**  
   - Connect input from "Download mp3".  
   - Operation: Upload File  
   - Configure Google Drive OAuth2 credentials with proper access.  
   - Set Drive ID and Folder ID to your target folder ("youtube to mp3" folder).  
   - Upload the binary MP3 data received from the HTTP Request node.

8. **Add a Code node named "Code":**  
   - Connect input from "Google Drive".  
   - Add JavaScript code to convert file size from bytes to MB:  
     ```js
     function kbToMb(kb) {
         return (kb / 1024).toFixed(2);
     }
     function bytesToKb(bytes) {
         return bytes / 1024;
     }
     let fileSizeInBytes = $input.first().json.size;
     let fileSizeInKb = bytesToKb(fileSizeInBytes);
     let fileSizeInMb = kbToMb(fileSizeInKb);
     return [{ json: {
         fileName: $input.first().json.name,
         fileSizeInMb: fileSizeInMb
     }}];
     ```
9. **Add a Google Sheets node named "Google Sheets":**  
   - Connect input from "Code".  
   - Operation: Append or Update Row  
   - Use same Google Sheets credentials and document as "Google Sheets1".  
   - Map columns:  
     - URL: `{{$('On form submission').item.json.URL}}`  
     - Size: `{{$json.fileSizeInMb}} MB`  
     - Status: `"Success"`  
     - Created at: `{{$now.format('dd-MM-yyyy')}}`  
     - Download Link: `{{$('Google Drive').item.json.webContentLink}}`  
     - Web View Link: `{{$('Google Drive').item.json.webViewLink}}`

---

### 5. General Notes & Resources

| Note Content                                                                                                        | Context or Link                                  |
|---------------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| This workflow uses the RapidAPI YouTube to MP3 downloader API; ensure your API key is valid and authorized.         | RapidAPI service for YouTube to MP3 conversion  |
| Google Drive and Sheets credentials use OAuth2 and Service Account authentication respectively; configure before use.| Credential setup for Google APIs                 |
| The workflow includes asynchronous polling via the Wait node to handle conversion delays without stopping workflow. | Best practice for handling external process time|
| File size is converted from bytes to MB with precision to two decimals for user-friendly display in logs.           | Size conversion code in the Code node            |
| The form trigger allows easy user input without external triggers or API calls.                                      | User-friendly input method                        |
| Ensure folder IDs and Google Sheet IDs are updated to your environment before deployment.                            | Google Drive & Sheets configuration              |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.