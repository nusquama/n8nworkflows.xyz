Capture and Store Website Screenshots from Google Sheets to Drive using Dumpling AI

https://n8nworkflows.xyz/workflows/capture-and-store-website-screenshots-from-google-sheets-to-drive-using-dumpling-ai-6949


# Capture and Store Website Screenshots from Google Sheets to Drive using Dumpling AI

### 1. Workflow Overview

This workflow automates the process of capturing full-page screenshots of websites listed in a Google Sheets document, then storing these screenshots into a designated Google Drive folder using Dumpling AI as the screenshot service. It is designed for users who maintain a list of URLs in a Google Sheet and want to automatically generate and archive visual snapshots of these sites without manual intervention.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Watches for new rows added to a specific Google Sheet containing website URLs.
- **1.2 Screenshot Request:** Sends the captured URL to Dumpling AI to request a full-page screenshot.
- **1.3 Screenshot Retrieval:** Downloads the screenshot image from the URL returned by Dumpling AI.
- **1.4 Storage:** Uploads the downloaded screenshot image file to a specified Google Drive folder.
- **1.5 Documentation:** A sticky note summarizes the workflow purpose and steps for clarity.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block monitors a Google Sheet for new rows added, each containing a website URL. It triggers the workflow execution upon detecting a new entry.

**Nodes Involved:**  
- Watch New Row in Google Sheets

**Node Details:**

- **Watch New Row in Google Sheets**  
  - *Type & Role:* Google Sheets Trigger node; triggers workflow on new row additions.  
  - *Configuration:*  
    - Monitors "Sheet1" in a Google Sheets document identified by its ID.  
    - Polls every minute for new rows.  
  - *Key Expressions:* None directly; the URL is extracted from the new row’s "Website" column.  
  - *Input:* External event (new row in Google Sheet).  
  - *Output:* JSON object containing new row data, including the "Website" URL field.  
  - *Version Requirements:* Compatible with n8n version supporting Google Sheets Trigger v1.  
  - *Edge Cases:*  
    - Authentication failure with Google Sheets OAuth2 credentials.  
    - No new rows detected (workflow idle).  
    - Incorrect sheet name or document ID causing trigger failure.  
  - *Sub-workflow:* None

---

#### 1.2 Screenshot Request

**Overview:**  
This block sends the website URL obtained from the trigger node to Dumpling AI’s API to request a full-page screenshot.

**Nodes Involved:**  
- Request Screenshot from Dumpling AI

**Node Details:**

- **Request Screenshot from Dumpling AI**  
  - *Type & Role:* HTTP Request node; sends POST request to Dumpling AI API.  
  - *Configuration:*  
    - URL: `https://app.dumplingai.com/api/v1/screenshot`  
    - Method: POST  
    - Body: JSON containing `"url"` set dynamically from the `"Website"` field of the trigger node, and `"fullPage": true` to capture entire page.  
    - Authentication: HTTP Header Auth via Dumpling AI API key credential.  
  - *Key Expressions:*  
    - `{{ $json.Website }}` used to insert the website URL dynamically.  
  - *Input:* JSON with website URL from previous node.  
  - *Output:* JSON including the screenshot URL under `screenshotUrl`.  
  - *Version Requirements:* HTTP Request v4.2 or higher to support JSON body and authentication.  
  - *Edge Cases:*  
    - API authentication errors (invalid/expired API key).  
    - Network timeouts or Dumpling AI service downtime.  
    - Invalid URL format causing API rejection.  
  - *Sub-workflow:* None

---

#### 1.3 Screenshot Retrieval

**Overview:**  
Downloads the screenshot file using the URL provided by Dumpling AI’s response.

**Nodes Involved:**  
- Download Screenshot

**Node Details:**

- **Download Screenshot**  
  - *Type & Role:* HTTP Request node; performs a GET request to retrieve the screenshot image.  
  - *Configuration:*  
    - URL dynamically set to `{{$json.screenshotUrl}}` from the prior node’s response.  
  - *Key Expressions:*  
    - Uses the dynamic URL string from previous node output.  
  - *Input:* JSON containing `screenshotUrl`.  
  - *Output:* Binary data of the screenshot image.  
  - *Version Requirements:* HTTP Request v4.2 or higher for binary data handling.  
  - *Edge Cases:*  
    - Broken or expired screenshot URL.  
    - Download failure due to network issues.  
    - Non-image response or corrupted file content.  
  - *Sub-workflow:* None

---

#### 1.4 Storage

**Overview:**  
Uploads the downloaded screenshot image to a designated folder in Google Drive for archival and access.

**Nodes Involved:**  
- Upload Screenshot to Google Drive

**Node Details:**

- **Upload Screenshot to Google Drive**  
  - *Type & Role:* Google Drive node; uploads binary file to Google Drive.  
  - *Configuration:*  
    - Filename is set statically (`temp-488dda43-93de-4fa6-acfc-f3977c584ab1.png`).  
    - Destination folder specified by folder ID (Google Drive folder named "pdf").  
    - Target drive: "My Drive".  
  - *Key Expressions:* None dynamic; filename is static.  
  - *Input:* Binary data from the HTTP download node.  
  - *Output:* Metadata of the uploaded file in Google Drive.  
  - *Version Requirements:* Google Drive node v3 for OAuth2 support and folder upload.  
  - *Edge Cases:*  
    - Google Drive OAuth2 authentication failure.  
    - Insufficient storage quota or folder permission issues.  
    - Filename conflicts or invalid folder ID.  
  - *Sub-workflow:* None

---

#### 1.5 Documentation

**Overview:**  
A sticky note node provides a summary and explanation of the workflow’s purpose and steps for user clarity.

**Nodes Involved:**  
- Sticky Note

**Node Details:**

- **Sticky Note**  
  - *Type & Role:* Visual annotation node; no execution impact.  
  - *Configuration:* Contains multi-line markdown text summarizing the workflow logic and purpose.  
  - *Input / Output:* None (informational only).  
  - *Version Requirements:* None.  
  - *Edge Cases:* N/A  
  - *Sub-workflow:* None

---

### 3. Summary Table

| Node Name                      | Node Type                  | Functional Role                    | Input Node(s)                    | Output Node(s)                     | Sticky Note                                                                                                 |
|-------------------------------|----------------------------|----------------------------------|---------------------------------|----------------------------------|-------------------------------------------------------------------------------------------------------------|
| Watch New Row in Google Sheets | Google Sheets Trigger      | Detects new website URLs added   | -                               | Request Screenshot from Dumpling AI |                                                                                                             |
| Request Screenshot from Dumpling AI | HTTP Request           | Sends URL to Dumpling AI API     | Watch New Row in Google Sheets  | Download Screenshot               |                                                                                                             |
| Download Screenshot            | HTTP Request               | Downloads screenshot binary      | Request Screenshot from Dumpling AI | Upload Screenshot to Google Drive |                                                                                                             |
| Upload Screenshot to Google Drive | Google Drive             | Uploads screenshot to Drive      | Download Screenshot             | -                                |                                                                                                             |
| Sticky Note                   | Sticky Note                | Documents workflow purpose       | -                               | -                                | ### 🖼️ Website Screenshot Capture from Google Sheets\n\nThis workflow watches for new rows in a Google Sheet. When a new website URL is added:\n\n1. It sends the URL to **Dumpling AI** to capture a full-page screenshot  \n2. Downloads the screenshot file using the returned URL  \n3. Uploads the screenshot to a specified **Google Drive folder** |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the "Watch New Row in Google Sheets" node**  
   - Add a **Google Sheets Trigger** node.  
   - Configure credentials with OAuth2 for Google Sheets.  
   - Set the event to trigger on "rowAdded".  
   - Specify the Google Sheets document ID corresponding to your spreadsheet.  
   - Set the sheet name to "Sheet1" (or your actual sheet name).  
   - Set polling interval to every minute (or as desired).  
   - No additional expressions needed here.

2. **Create the "Request Screenshot from Dumpling AI" node**  
   - Add an **HTTP Request** node.  
   - Connect it to the Google Sheets Trigger node.  
   - Set method to POST.  
   - URL: `https://app.dumplingai.com/api/v1/screenshot`.  
   - Set "Send Body" to true with JSON content:  
     ```json
     {
       "url": "{{ $json.Website }}",
       "fullPage": true
     }
     ```  
   - Configure authentication using HTTP Header Auth with your Dumpling AI API key credential.  
   - Ensure HTTP Request node version supports JSON body and header auth (v4.2+).

3. **Create the "Download Screenshot" node**  
   - Add another **HTTP Request** node.  
   - Connect it to the "Request Screenshot from Dumpling AI" node.  
   - Set method to GET.  
   - Set URL to `={{ $json.screenshotUrl }}` (dynamic URL from previous response).  
   - No authentication needed.  
   - Confirm HTTP Request node supports binary data handling (v4.2+).

4. **Create the "Upload Screenshot to Google Drive" node**  
   - Add a **Google Drive** node.  
   - Connect it to the "Download Screenshot" node.  
   - Configure Google Drive OAuth2 credentials.  
   - Action: Upload file.  
   - Set the file name statically, e.g., `temp-488dda43-93de-4fa6-acfc-f3977c584ab1.png`.  
   - Specify the folder ID of the target Google Drive folder where screenshots will be stored.  
   - Set the drive to "My Drive" or your specific drive location.  
   - Ensure the node version supports folder uploads and OAuth2 (v3+).

5. **Add a Sticky Note for documentation (optional)**  
   - Add a **Sticky Note** node anywhere on the canvas.  
   - Write a brief explanation describing the workflow steps and purpose.  
   - Format with markdown if desired.

6. **Connect the nodes in sequence:**  
   - Google Sheets Trigger → Dumpling AI Request → Download Screenshot → Upload to Google Drive.

7. **Activate the workflow** and test by adding a new URL row to the monitored Google Sheet.

---

### 5. General Notes & Resources

| Note Content                                                                                                      | Context or Link                                                                                               |
|------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Dumpling AI API documentation: https://app.dumplingai.com/docs/api                                               | Reference for API usage and authentication details                                                           |
| Google Sheets API OAuth2 setup required for trigger node                                                          | Required for Google Sheets Trigger node to function correctly                                                 |
| Google Drive API OAuth2 setup required for upload operations                                                      | Required for Google Drive node to upload files                                                                |
| Polling interval can be adjusted to reduce API call frequency or latency                                          | Currently set to 1 minute for near real-time capture                                                          |
| Filename in upload node is fixed; to avoid overwrites, consider dynamic naming (e.g., timestamp or URL hash)      | Enhances file management and prevents overwriting previous screenshots                                        |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.