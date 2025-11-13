Auto-Post Instagram Carousels using Google Sheets, Drive, Cloudinary & Graph API

https://n8nworkflows.xyz/workflows/auto-post-instagram-carousels-using-google-sheets--drive--cloudinary---graph-api-8989


# Auto-Post Instagram Carousels using Google Sheets, Drive, Cloudinary & Graph API

### 1. Workflow Overview

This workflow automates the process of publishing Instagram carousel posts using assets managed in Google Drive and metadata from Google Sheets. It is designed for social media managers or marketers who want to schedule and batch post Instagram carousels without manual uploads. The workflow orchestrates asset retrieval, hosting images on Cloudinary, Instagram Graph API media container creation, carousel assembly, publishing, and status updates.

The workflow is logically divided into four main blocks:

- **1.1 Schedule & Job Selection:** Periodic trigger fetches carousel post jobs from Google Sheets filtered by status and type.
- **1.2 Asset Collection & Hosting:** Retrieves images from a specified Google Drive folder, downloads them, and uploads to Cloudinary for hosting.
- **1.3 Instagram Media Container Creation:** Sets up Instagram API credentials and variables, creates individual media containers for each image, aggregates them, then creates the carousel container.
- **1.4 Publishing & Logging:** Publishes the carousel post on Instagram and updates the Google Sheets job status to "Processed".

---

### 2. Block-by-Block Analysis

#### 1.1 Schedule & Job Selection

**Overview:**  
This block triggers the workflow periodically and queries Google Sheets to find pending carousel posts that need processing.

**Nodes Involved:**  
- Schedule Trigger  
- Get Execution for Carousel

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Initiates workflow every X minutes (configured to run on a minutes interval).  
  - Configuration: Default interval trigger, runs periodically to check for new jobs.  
  - Inputs: None (trigger node)  
  - Outputs: Triggers "Get Execution for Carousel" node.  
  - Edge cases: Workflow idle if no matching rows; ensure interval is appropriate to avoid API quota issues.

- **Get Execution for Carousel**  
  - Type: Google Sheets  
  - Role: Filters execution jobs from a Google Sheet named "Execute" to find rows where `Status=ToDo` and `Type=Carousel`.  
  - Configuration: Uses filter UI with lookup values on columns `Status` and `Type`. Document and sheet IDs are set to a specific master Google Sheet.  
  - Inputs: Trigger from Schedule Trigger.  
  - Outputs: Sends filtered rows to "Get image list from a Google Drive folder1" node.  
  - Expressions: Filtering criteria are hardcoded in parameters.  
  - Edge cases: No matching rows may cause workflow to do nothing; invalid credentials or sheet access may cause failure.  
  - Sticky Note: Suggests using `ExecuteId` as a unique key for update operations.

#### 1.2 Asset Collection & Hosting

**Overview:**  
This block retrieves image files listed in the Google Sheet job row’s folder, downloads each image from Google Drive, then uploads them to Cloudinary for public hosting.

**Nodes Involved:**  
- Get image list from a Google Drive folder1  
- Download Image from Google Drive (Carousel)  
- Upload images to Cloudinary

**Node Details:**

- **Get image list from a Google Drive folder1**  
  - Type: Google Drive  
  - Role: Lists all files in the Google Drive folder specified in the current job’s `Folder` field.  
  - Configuration: Uses folder ID dynamically from the "Get Execution for Carousel" node data. Returns all files with fields including id, name, thumbnailLink, and webViewLink.  
  - Inputs: Output of "Get Execution for Carousel".  
  - Outputs: Passes file list to "Download Image from Google Drive (Carousel)".  
  - Edge cases: Folder ID must be valid and accessible by OAuth credentials; private files require appropriate access rights.

- **Download Image from Google Drive (Carousel)**  
  - Type: HTTP Request  
  - Role: Downloads each image file binary from Google Drive using a constructed URL with file ID.  
  - Configuration: URL uses Google Drive’s public file download endpoint with file ID from previous node’s JSON data. Response set to full file binary.  
  - Inputs: Files from Drive listing node.  
  - Outputs: Passes binary data to "Upload images to Cloudinary".  
  - Edge cases: File permissions may block download; large files may cause timeouts; URL construction must be correct.

- **Upload images to Cloudinary**  
  - Type: HTTP Request  
  - Role: Uploads the image binary to Cloudinary for hosting and returns the hosted image URL.  
  - Configuration: POST to Cloudinary upload API with multipart/form-data. Uses `file` field for binary data and `upload_preset` for unsigned upload preset.  
  - Inputs: Binary data of images from download node.  
  - Outputs: Passes Cloudinary response (including hosted URL) to "Setup for Instagram (access token, ig_business_id)".  
  - Edge cases: Cloudinary credentials and preset must be valid; unsigned presets should be locked down; network issues can cause upload failure.  
  - Sticky Note: Recommends using signed uploads for security and keeping upload presets locked down.

#### 1.3 Instagram Media Container Creation

**Overview:**  
This block prepares Instagram API parameters, creates media containers for each image URL, then aggregates media container IDs before creating the carousel container.

**Nodes Involved:**  
- Setup for Instagram (access token, ig_business_id)  
- Create Media Container (Image)  
- Combine Instagram media containers  
- Create Media Container (Carousel)

**Node Details:**

- **Setup for Instagram (access token, ig_business_id)**  
  - Type: Set  
  - Role: Assigns Instagram API variables including access token, Instagram user ID, image URL (from Cloudinary upload), and caption (from Google Sheets job).  
  - Configuration: Sets static strings for `access_token` and `ig_user_id` (to be replaced by user credentials). Dynamically sets `image_url` from Cloudinary's returned JSON and `caption` from Google Sheets row field `Expected content`.  
  - Inputs: Cloudinary upload response.  
  - Outputs: Passes variables to "Create Media Container (Image)".  
  - Edge cases: Access token and user ID must be valid; expired tokens cause API failures.

- **Create Media Container (Image)**  
  - Type: HTTP Request  
  - Role: Creates a media container on Instagram for each image URL with caption.  
  - Configuration: POST to Instagram Graph API endpoint `/v23.0/{ig_user_id}/media`. Sends form-urlencoded parameters: `image_url`, `caption`, and `access_token`.  
  - Inputs: Instagram variables from Set node.  
  - Outputs: Returns a container ID for each image, passed to aggregation node.  
  - Edge cases: API rate limits, invalid access tokens, or malformed URLs cause errors.

- **Combine Instagram media containers**  
  - Type: Aggregate  
  - Role: Aggregates the list of media container IDs from image containers into a single array for carousel creation.  
  - Configuration: Merges lists of `body.id` fields from previous node.  
  - Inputs: Media container creation outputs.  
  - Outputs: Aggregated IDs passed to "Create Media Container (Carousel)".  
  - Edge cases: Empty lists if image container creation failed.

- **Create Media Container (Carousel)**  
  - Type: HTTP Request  
  - Role: Creates a carousel media container using the aggregated media IDs and caption.  
  - Configuration: POST to Instagram API `/v23.0/{ig_user_id}/media` with parameters: `caption`, `media_type=CAROUSEL`, `children` (comma-separated media container IDs), and `access_token`.  
  - Inputs: Aggregated media IDs and Instagram setup data.  
  - Outputs: Returns carousel media container ID to publishing node.  
  - Edge cases: API errors if children list is empty or invalid token.

#### 1.4 Publishing & Logging

**Overview:**  
This block publishes the carousel post on Instagram and updates the Google Sheet row to mark the job as processed.

**Nodes Involved:**  
- Publish Instagram Carousel  
- Update Execute to "Processed"

**Node Details:**

- **Publish Instagram Carousel**  
  - Type: HTTP Request  
  - Role: Publishes the Instagram carousel post using the carousel container ID.  
  - Configuration: POST to `/v23.0/{ig_user_id}/media_publish` with parameters: `creation_id` (carousel container ID) and `access_token`. Response is JSON.  
  - Inputs: Carousel container JSON from previous step.  
  - Outputs: Passes success status to Google Sheets update.  
  - Edge cases: Invalid container ID or expired token causes failure.

- **Update Execute to "Processed"**  
  - Type: Google Sheets  
  - Role: Updates the status of the processed job in the Google Sheet from "ToDo" to "Processed" using the unique `ExecuteId` key.  
  - Configuration: Update operation on the same sheet and document as earlier. Matches on `ExecuteId` column and sets `Status=Processed`.  
  - Inputs: Triggered by successful publish.  
  - Outputs: None (end of workflow).  
  - Edge cases: Update may fail if `ExecuteId` is missing or sheet is locked.

- Sticky Note: Summarizes the Instagram posting steps and final sheet update.

---

### 3. Summary Table

| Node Name                            | Node Type          | Functional Role                      | Input Node(s)                 | Output Node(s)                         | Sticky Note                                                                                          |
|------------------------------------|--------------------|------------------------------------|------------------------------|--------------------------------------|----------------------------------------------------------------------------------------------------|
| Schedule Trigger                   | Schedule Trigger   | Periodic trigger to start workflow | None                         | Get Execution for Carousel            | ## STEP 1 · Schedule & Pick Jobs<br>Runs every X minutes; filters rows in Sheets for ToDo carousels |
| Get Execution for Carousel         | Google Sheets      | Filters jobs with Status=ToDo/Type=Carousel | Schedule Trigger             | Get image list from a Google Drive folder1 | See above                                                                                           |
| Get image list from a Google Drive folder1 | Google Drive      | Lists files in Drive folder for job | Get Execution for Carousel    | Download Image from Google Drive (Carousel) | ## STEP 2 · Collect Assets<br>Reads images from folder; private files require OAuth or public access |
| Download Image from Google Drive (Carousel) | HTTP Request      | Downloads images by file ID         | Get image list from a Google Drive folder1 | Upload images to Cloudinary           | See above                                                                                           |
| Upload images to Cloudinary        | HTTP Request       | Uploads images to Cloudinary        | Download Image from Google Drive (Carousel) | Setup for Instagram (access token, ig_business_id) | ## STEP 3 · Host Images<br>Receives binary, returns URL; prefer signed uploads and locked presets    |
| Setup for Instagram (access token, ig_business_id) | Set               | Sets Instagram API params and caption | Upload images to Cloudinary    | Create Media Container (Image)         | See above                                                                                           |
| Create Media Container (Image)     | HTTP Request       | Creates media container per image   | Setup for Instagram            | Combine Instagram media containers     | See above                                                                                           |
| Combine Instagram media containers | Aggregate          | Aggregates media container IDs      | Create Media Container (Image) | Create Media Container (Carousel)      | See above                                                                                           |
| Create Media Container (Carousel)  | HTTP Request       | Creates carousel container using aggregated IDs | Combine Instagram media containers | Publish Instagram Carousel             | See above                                                                                           |
| Publish Instagram Carousel         | HTTP Request       | Publishes the carousel post          | Create Media Container (Carousel) | Update Execute to "Processed"           | ## STEP 4 · IG Carousel & Log<br>Creates media containers, aggregates, creates carousel, publishes, updates Sheets |
| Update Execute to "Processed"      | Google Sheets      | Marks job as processed in Sheets    | Publish Instagram Carousel     | None                                 | See above                                                                                           |
| Sticky Note                       | Sticky Note        | Explains scheduling and job selection | None                         | None                                 | See step 1 note                                                                                     |
| Sticky Note1                      | Sticky Note        | Explains asset collection            | None                         | None                                 | See step 2 note                                                                                     |
| Sticky Note2                      | Sticky Note        | Explains image hosting on Cloudinary | None                         | None                                 | See step 3 note                                                                                     |
| Sticky Note3                      | Sticky Note        | Explains Instagram publishing & logging | None                         | None                                 | See step 4 note                                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Set to trigger every X minutes as needed to check for new jobs.  
   - No credentials needed.

2. **Create a Google Sheets node ("Get Execution for Carousel"):**  
   - Connect input from Schedule Trigger.  
   - Set document ID to your Google Sheet containing the "Execute" sheet.  
   - Set sheet name to "Execute" (or appropriate name).  
   - Apply filters: `Status = ToDo` and `Type = Carousel`.  
   - Use Google Sheets OAuth2 credentials with read access.

3. **Create a Google Drive node ("Get image list from a Google Drive folder1"):**  
   - Connect input from "Get Execution for Carousel".  
   - Configure to list all files in the folder given by the `Folder` field from the current Google Sheets row (use expression to reference).  
   - Select fields: id, name, thumbnailLink, webViewLink.  
   - Use Google Drive OAuth2 credentials with read access.

4. **Create an HTTP Request node ("Download Image from Google Drive (Carousel)"):**  
   - Connect input from Google Drive list node.  
   - Set method to GET.  
   - Use URL: `https://drive.google.com/uc?export=download&id={{ $json.id }}` (dynamic file ID).  
   - Set response to full binary file.  
   - No credentials needed if files are publicly accessible; otherwise, configure OAuth or public sharing.

5. **Create an HTTP Request node ("Upload images to Cloudinary"):**  
   - Connect input from the previous node (binary image).  
   - Set method to POST.  
   - URL: `https://api.cloudinary.com/v1_1/<your-cloud-name>/image/upload`.  
   - Set content type to multipart/form-data.  
   - Add form parameters:  
     - `file` (binary data from previous node)  
     - `upload_preset` set to your Cloudinary unsigned preset.  
   - No authentication if unsigned preset is used; otherwise configure API key as needed.

6. **Create a Set node ("Setup for Instagram (access token, ig_business_id)"):**  
   - Connect input from Cloudinary upload response.  
   - Assign variables:  
     - `access_token`: your Instagram access token (string).  
     - `ig_user_id`: your Instagram Business User ID (string).  
     - `image_url`: set to Cloudinary `secure_url` field from upload response.  
     - `caption`: extract from the original Google Sheets row field `Expected content`.  
   - No credentials needed.

7. **Create an HTTP Request node ("Create Media Container (Image)"):**  
   - Connect input from Set node.  
   - Set method to POST.  
   - URL: `https://graph.instagram.com/v23.0/{{ $json.ig_user_id }}/media`.  
   - Content type: application/x-www-form-urlencoded.  
   - Body parameters:  
     - `image_url`: from Set node.  
     - `caption`: from Set node.  
     - `access_token`: from Set node.  
   - Use Instagram API credentials embedded in variables.

8. **Create an Aggregate node ("Combine Instagram media containers"):**  
   - Connect input from "Create Media Container (Image)".  
   - Configure to merge lists of `body.id` fields into a single array.  
   - No credentials needed.

9. **Create an HTTP Request node ("Create Media Container (Carousel)"):**  
   - Connect input from Aggregate node.  
   - Set method to POST.  
   - URL: `https://graph.instagram.com/v23.0/{{ $json.ig_user_id }}/media`.  
   - Content type: application/x-www-form-urlencoded.  
   - Body parameters:  
     - `caption`: from Google Sheets row `Expected content`.  
     - `media_type`: "CAROUSEL".  
     - `children`: aggregated media container IDs as comma-separated string.  
     - `access_token`: from Set node.  

10. **Create an HTTP Request node ("Publish Instagram Carousel"):**  
    - Connect input from Carousel container creation node.  
    - Set method to POST.  
    - URL: `https://graph.instagram.com/v23.0/{{ $json.ig_user_id }}/media_publish`.  
    - Body parameters:  
      - `creation_id`: carousel container ID (from previous node’s response).  
      - `access_token`: from Set node.

11. **Create a Google Sheets node ("Update Execute to \"Processed\""):**  
    - Connect input from Publish Instagram Carousel node.  
    - Configure update operation on the same Google Sheet and sheet as in step 2.  
    - Use `ExecuteId` as matching column.  
    - Update `Status` column to "Processed" for the matching row.  
    - Use Google Sheets OAuth2 credentials with write access.

12. **Optional:** Add Sticky Note nodes at relevant workflow points to provide inline documentation.

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                                                                         |
|----------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| Use `ExecuteId` as a unique identifier for rows to ensure accurate updates in Google Sheets.                   | Referenced in filtering and updating Google Sheets rows.                                              |
| Cloudinary upload should preferably use signed uploads with locked down presets for security.                   | https://cloudinary.com/documentation/upload_presets                                                    |
| Instagram Graph API version v23.0 is used; verify API compatibility and token scopes before deployment.        | https://developers.facebook.com/docs/instagram-api/                                                    |
| Google Drive files must be accessible for download; either public or accessible via OAuth credentials.          | https://developers.google.com/drive/api/v3/about-files                                                 |
| Instagram Business User and Access Token require Facebook App setup with appropriate permissions and tokens.   | https://developers.facebook.com/docs/instagram-api/getting-started                                     |
| Scheduling interval should balance between timely posting and API quota limits.                                |                                                                                                        |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated workflow built with n8n, an integration and automation tool. This processing strictly adheres to content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.