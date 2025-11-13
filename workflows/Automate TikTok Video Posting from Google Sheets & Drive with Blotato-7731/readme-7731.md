Automate TikTok Video Posting from Google Sheets & Drive with Blotato

https://n8nworkflows.xyz/workflows/automate-tiktok-video-posting-from-google-sheets---drive-with-blotato-7731


# Automate TikTok Video Posting from Google Sheets & Drive with Blotato

### 1. Workflow Overview

This workflow automates the process of posting videos to TikTok by integrating Google Sheets, Google Drive, and the Blotato platform. It is designed for creators or social media managers who maintain a list of videos and captions in Google Sheets, store video files in Google Drive, and want to schedule and automate TikTok posts efficiently.

The workflow is logically divided into three main blocks:

- **1.1 Data Retrieval & Filtering:** Triggered on a schedule, it fetches video entries from Google Sheets and filters only those with a status marked as "pending" to process one video at a time.

- **1.2 Video Preparation & Upload:** Extracts the Google Drive file ID from the video URL, shares the file publicly on Drive, and uploads the video to the Blotato platform for TikTok posting.

- **1.3 TikTok Posting & Status Update:** Uses Blotato to post the video on TikTok with the appropriate caption, then updates the Google Sheets status for the video entry to "posted."

---

### 2. Block-by-Block Analysis

#### 2.1 Data Retrieval & Filtering

**Overview:**  
This block is responsible for initiating the workflow on a timed schedule, retrieving video data from Google Sheets, and filtering for entries that are pending posting. It limits processing to one video per execution to prevent rate limiting or overload.

**Nodes Involved:**  
- Schedule Trigger  
- URL VIDEO to Post (Google Sheets)  
- If  
- Limit

**Node Details:**

- **Schedule Trigger**  
  - *Type:* Trigger  
  - *Role:* Starts workflow execution every hour.  
  - *Configuration:* Interval set to every 1 hour.  
  - *Inputs:* None (trigger).  
  - *Outputs:* Connects to "URL VIDEO to Post".  
  - *Edge Cases:* Failure if n8n instance downtime; no retries enabled.  
  - *Notes:* Ensures regular polling of new video entries.

- **URL VIDEO to Post**  
  - *Type:* Google Sheets  
  - *Role:* Retrieves rows from the specified Google Sheets document and sheet (named "Sheet1").  
  - *Configuration:* Uses OAuth2 credentials for Google Sheets; pulls all rows from the first sheet (gid=0) of the document with ID `1TgAOn_dRUwEQFlWtNXFxXj72SVqPuMIMZ85dwsjcuG0`.  
  - *Inputs:* Trigger from Schedule Trigger.  
  - *Outputs:* Flows into "If" node.  
  - *Edge Cases:* OAuth token expiration, API limits, empty or malformed data.  
  - *Notes:* Requires the sheet to have columns: ID, Title, Media URL, Caption, Status.

- **If**  
  - *Type:* Conditional  
  - *Role:* Filters for rows where the "Status" column equals "pending".  
  - *Configuration:* Checks `$json.Status == "pending"` with strict type and case sensitivity.  
  - *Inputs:* Rows from Google Sheets node.  
  - *Outputs:* Only passes rows with "pending" status to the next node.  
  - *Edge Cases:* If no rows are pending, workflow stops here.  
  - *Notes:* Ensures only unposted videos are processed.

- **Limit**  
  - *Type:* Limit  
  - *Role:* Restricts the number of items passed downstream to 1 per execution.  
  - *Configuration:* Default limit of 1.  
  - *Inputs:* Filtered rows from "If".  
  - *Outputs:* Passes one item to the next block.  
  - *Edge Cases:* If multiple pending rows exist, processes one per hour, which may delay posting.  
  - *Notes:* Prevents hitting API rate limits or excessive postings at once.

---

#### 2.2 Video Preparation & Upload

**Overview:**  
This block extracts the Google Drive file ID from the video URL, sets the file sharing permissions to public, and uploads the video file to the Blotato platform to prepare it for TikTok posting.

**Nodes Involved:**  
- Get Google Drive ID (Set)  
- Share file (Google Drive)  
- Upload Video to BLOTATO (Blotato)

**Node Details:**

- **Get Google Drive ID**  
  - *Type:* Set  
  - *Role:* Extracts the file ID from the "Media URL" field of the Google Sheets row using regex.  
  - *Configuration:* Uses the expression:  
    `={{ $json['Media URL'].match(/https:\/\/drive\.google\.com\/file\/d\/([A-Za-z0-9_-]+)/i)[1] }}`  
  - *Inputs:* One row from "Limit".  
  - *Outputs:* Passes extracted Drive file ID as `final_google_drive_url` in JSON.  
  - *Edge Cases:* If URL format differs or no match found, the node will fail or produce undefined ID.  
  - *Notes:* Assumes Google Drive URL format `https://drive.google.com/file/d/<fileId>`.

- **Share file**  
  - *Type:* Google Drive  
  - *Role:* Shares the extracted Google Drive file publicly by giving "reader" permission to "anyone".  
  - *Configuration:*  
    - Operation: Share  
    - File ID: `={{ $json.final_google_drive_url }}` (from previous node)  
    - Permissions: Role = "reader", Type = "anyone" (public read access)  
  - *Inputs:* Output from "Get Google Drive ID".  
  - *Outputs:* Continues to "Upload Video to BLOTATO".  
  - *Edge Cases:* Permission errors if OAuth token lacks scope or file ownership issues.  
  - *Notes:* Makes the video file accessible for upload by Blotato.

- **Upload Video to BLOTATO**  
  - *Type:* Blotato (Custom Integration)  
  - *Role:* Uploads the publicly shared video from Google Drive to Blotato media storage.  
  - *Configuration:*  
    - Media URL: Constructed as `https://drive.google.com/uc?export=download&id={{ final_google_drive_url }}` to download the file directly.  
    - Resource: "media" (upload media resource)  
  - *Credentials:* Blotato API account required.  
  - *Inputs:* Output from "Share file".  
  - *Outputs:* Provides uploaded media URL to next node.  
  - *Edge Cases:* Upload failures due to network, API errors, invalid URL, or credential issues.  
  - *Notes:* Critical step linking Google Drive and TikTok via Blotato.

---

#### 2.3 TikTok Posting & Status Update

**Overview:**  
Posts the uploaded video to TikTok via the Blotato platform using the uploaded media URL and caption from the sheet, then updates the Google Sheets row status to "posted."

**Nodes Involved:**  
- Tiktok (Blotato)  
- Update Status to "DONE" (Google Sheets)

**Node Details:**

- **Tiktok**  
  - *Type:* Blotato (Custom Integration)  
  - *Role:* Posts video to TikTok account on Blotato using uploaded media and caption.  
  - *Configuration:*  
    - Platform: "tiktok"  
    - Account ID: `13117` (predefined TikTok account in Blotato)  
    - Post Content Text: `={{ $('URL VIDEO to Post').item.json.Caption }}` (caption from original sheet row)  
    - Post Content Media URLs: `={{ $json.url }}` (uploaded media URL from previous node)  
  - *Credentials:* Blotato API account.  
  - *Inputs:* Output from "Upload Video to BLOTATO".  
  - *Outputs:* Connects to "Update Status to \"DONE\"".  
  - *Edge Cases:* API errors, invalid account ID, media URL issues, caption missing.  
  - *Notes:* Automates TikTok publishing.

- **Update Status to "DONE"**  
  - *Type:* Google Sheets  
  - *Role:* Updates the "Status" column of the original Google Sheets row to "posted" after successful TikTok post.  
  - *Configuration:*  
    - Operation: Update  
    - Sheet: Same as original data retrieval (gid=0, same doc ID)  
    - Matching Column: "ID" to locate the correct row  
    - Columns to Update: "Status" set to "posted"  
    - Data Source: Uses ID from original row to match  
  - *Credentials:* Google Sheets OAuth2.  
  - *Inputs:* Output from "Tiktok".  
  - *Outputs:* None (end of workflow).  
  - *Edge Cases:* Update failure if row not found, permissions error, API limits.  
  - *Notes:* Tracks posting status to avoid duplicates.

---

### 3. Summary Table

| Node Name                | Node Type                   | Functional Role                                  | Input Node(s)            | Output Node(s)                | Sticky Note                                                                                                                          |
|--------------------------|-----------------------------|-------------------------------------------------|--------------------------|------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger         | Schedule Trigger            | Starts workflow every hour                       | None                     | URL VIDEO to Post            | **Data Retrieval & Filtering Section:** Runs every hour to process videos from Google Sheets.                                        |
| URL VIDEO to Post        | Google Sheets               | Retrieves video data rows                        | Schedule Trigger         | If                          | Requires Google Sheets with columns: ID, Title, Media URL, Caption, Status.                                                          |
| If                       | If                         | Filters rows with Status == "pending"            | URL VIDEO to Post        | Limit                       | Filters videos to post only those marked "pending".                                                                                  |
| Limit                    | Limit                      | Limits processing to 1 video per run              | If                       | Get Google Drive ID          | Processes one video at a time to avoid hitting rate limits.                                                                          |
| Get Google Drive ID      | Set                        | Extracts Google Drive file ID from video URL     | Limit                    | Share file                  | Extracts file ID using regex from Google Drive URL.                                                                                  |
| Share file               | Google Drive               | Shares video file publicly on Google Drive       | Get Google Drive ID       | Upload Video to BLOTATO      | Shares Google Drive file publicly to allow Blotato download access.                                                                  |
| Upload Video to BLOTATO  | Blotato                    | Uploads video from Google Drive to Blotato       | Share file               | Tiktok                      | Uploads video media for TikTok posting via Blotato API.                                                                              |
| Tiktok                   | Blotato                    | Posts video with caption to TikTok                | Upload Video to BLOTATO  | Update Status to "DONE"      | Posts video using Blotato TikTok integration with caption from Google Sheets.                                                        |
| Update Status to "DONE"  | Google Sheets               | Updates Google Sheets row status to "posted"     | Tiktok                   | None                        | Marks video as posted to prevent re-processing.                                                                                      |
| 📋 DATA RETRIEVAL & FILTERING | Sticky Note                | Overview and instructions for entire workflow    | None                     | None                        | **Important:** Self-hosted n8n only; Google Sheets with required columns; Blotato API; Google OAuth2; videos in Google Drive; status must be "pending". |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Set interval to run every 1 hour.

2. **Create a Google Sheets node named "URL VIDEO to Post"**  
   - Type: Google Sheets  
   - Operation: Read Rows  
   - Document ID: `1TgAOn_dRUwEQFlWtNXFxXj72SVqPuMIMZ85dwsjcuG0`  
   - Sheet Name: gid=0 (Sheet1)  
   - Credentials: Configure Google Sheets OAuth2 with proper scopes.  
   - Connect Schedule Trigger output to this node.

3. **Add an If node named "If"**  
   - Condition: `$json.Status` equals "pending" (string, case-sensitive)  
   - Connect output of "URL VIDEO to Post" to this node.

4. **Add a Limit node named "Limit"**  
   - Set limit to 1 item.  
   - Connect the "true" output of the "If" node to this Limit node.

5. **Add a Set node named "Get Google Drive ID"**  
   - Add a new string field `final_google_drive_url`  
   - Value: `={{ $json['Media URL'].match(/https:\/\/drive\.google\.com\/file\/d\/([A-Za-z0-9_-]+)/i)[1] }}`  
   - Connect output of "Limit" to this node.

6. **Add a Google Drive node named "Share file"**  
   - Operation: Share  
   - File ID: `={{ $json.final_google_drive_url }}`  
   - Permissions: Role = reader, Type = anyone (public)  
   - Credentials: Google Drive OAuth2 with permission to share files.  
   - Connect "Get Google Drive ID" output here.

7. **Add a Blotato node named "Upload Video to BLOTATO"**  
   - Resource: media  
   - Media URL: `=https://drive.google.com/uc?export=download&id={{ $json.final_google_drive_url }}`  
   - Credentials: Blotato API with valid account.  
   - Connect output of "Share file" to this node.

8. **Add a Blotato node named "Tiktok"**  
   - Platform: "tiktok"  
   - Account ID: `13117` (or your TikTok account ID in Blotato)  
   - Post Content Text: `={{ $('URL VIDEO to Post').item.json.Caption }}`  
   - Post Content Media URLs: `={{ $json.url }}` (output URL from Blotato upload)  
   - Credentials: Blotato API account.  
   - Connect output from "Upload Video to BLOTATO".

9. **Add a Google Sheets node named "Update Status to \"DONE\""**  
   - Operation: Update  
   - Document ID and Sheet Name same as "URL VIDEO to Post"  
   - Mapping Mode: Define below  
   - Matching Columns: "ID" (to find the correct row)  
   - Columns to update: "Status" = "posted"  
   - Data:  
     - ID: `={{ $('URL VIDEO to Post').item.json.ID }}`  
     - Status: "posted"  
   - Credentials: Google Sheets OAuth2.  
   - Connect output of "Tiktok" node here.

10. **Connect nodes as follows:**  
    - Schedule Trigger → URL VIDEO to Post → If → Limit → Get Google Drive ID → Share file → Upload Video to BLOTATO → Tiktok → Update Status to "DONE"

11. **Credential Setup:**  
    - Ensure Google Sheets OAuth2 credentials have read/write access to the target spreadsheet.  
    - Ensure Google Drive OAuth2 credentials allow file sharing permissions.  
    - Configure Blotato API credentials with access to media upload and TikTok posting scopes.

12. **Testing & Validation:**  
    - Confirm Google Sheets data columns: ID, Title, Media URL, Caption, Status  
    - Videos must reside on Google Drive with URLs matching the expected pattern.  
    - Status must be "pending" to be processed.  
    - Check logs for errors on regex extraction, permission issues, or API failures.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                         | Context or Link                                                                 |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------|
| This workflow requires a self-hosted n8n instance due to usage of community nodes and custom Blotato integration.                                                 | Sticky Note content in workflow.                                                |
| Blotato platform is used as an intermediary for TikTok posting, requiring a valid API account and configured TikTok accounts inside Blotato.                     | Blotato documentation (not included here).                                     |
| Ensure Google Sheets columns are correctly named and populated: ID, Title, Media URL, Caption, Status. Status drives the workflow logic.                           | Workflow prerequisites.                                                         |
| Video URLs must be in the format: https://drive.google.com/file/d/FILE_ID and accessible by the OAuth2 Google Drive credentials.                                  | Regex extraction logic in "Get Google Drive ID".                               |
| Public sharing of Google Drive files is essential to allow Blotato to download the video files for posting.                                                       | Google Drive "Share file" node setup.                                          |
| The workflow processes only one video per hour to avoid API limits and possible TikTok posting restrictions.                                                      | Limit node configuration.                                                       |

---

**Disclaimer:** The provided description and analysis are derived solely from an automated n8n workflow export. The workflow complies with all relevant content policies and contains no illegal, offensive, or protected material. All data processed is legal and public.