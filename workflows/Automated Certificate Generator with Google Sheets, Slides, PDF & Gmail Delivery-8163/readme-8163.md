Automated Certificate Generator with Google Sheets, Slides, PDF & Gmail Delivery

https://n8nworkflows.xyz/workflows/automated-certificate-generator-with-google-sheets--slides--pdf---gmail-delivery-8163


# Automated Certificate Generator with Google Sheets, Slides, PDF & Gmail Delivery

### 1. Workflow Overview

This workflow automates the generation and distribution of personalized certificates based on new data entries in a Google Sheet. It is designed for scenarios such as course completions, event participations, or any context where certificates need to be issued automatically.

**Logical Blocks:**

- **1.1 Input Reception:** Detects new rows added to a Google Sheet containing participant data.
- **1.2 Certificate Creation:** Copies a Google Slides certificate template and replaces placeholders with participant-specific information.
- **1.3 File Conversion and Storage:** Converts the personalized Slides file into a PDF, stores it in Google Drive, and cleans up temporary files.
- **1.4 Email Delivery:** Sends the generated PDF certificate via Gmail to the participant’s email address.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Watches a Google Sheet for new rows added, triggering the workflow with each new participant's data (name and email).

- **Nodes Involved:**  
  - Google Sheets Trigger

- **Node Details:**  
  - **Google Sheets Trigger**  
    - *Type & Role:* Watches for new rows in a specified Google Sheet to initiate the workflow.  
    - *Configuration:*  
      - Event set to "rowAdded" to trigger when a new row is appended.  
      - Polling interval: every minute.  
      - Sheet specified by URL and sheet name (gid=0, typically the first sheet).  
    - *Expressions:* Uses URL and sheet name references, ensuring dynamic linkage to the correct Google Sheet.  
    - *Input/Output:* No input; output is the newly added row data with fields like "Name" and "Email".  
    - *Failure Modes:*  
      - Authentication errors if OAuth credentials expire.  
      - Timeout or rate-limiting from Google Sheets API.  
      - Misconfiguration if Sheet URL or Sheet ID is incorrect.  
    - *Credentials:* Requires OAuth2 credentials linked to Google Sheets API.

#### 2.2 Certificate Creation

- **Overview:**  
  Copies a Google Slides certificate template file, renames it per participant, then replaces placeholder text with the participant’s name.

- **Nodes Involved:**  
  - Copy file (Google Drive)  
  - Replace text in a presentation (Google Slides)

- **Node Details:**  
  - **Copy file**  
    - *Type & Role:* Copies an existing Google Slides file (certificate template).  
    - *Configuration:*  
      - Source file specified by Google Slides template ID.  
      - Destination folder specified by URL.  
      - New file named as `<Participant Name> Certificate`.  
    - *Input:* Triggered by new row data from Google Sheets Trigger.  
    - *Output:* JSON containing the new file ID.  
    - *Failure Modes:*  
      - Google Drive API errors (auth, permissions).  
      - Invalid template or folder ID.  
    - *Credentials:* Google Drive OAuth2 credentials required.

  - **Replace text in a presentation**  
    - *Type & Role:* Modifies copied Slides file by replacing placeholder text `[NAME]` with actual participant name.  
    - *Configuration:*  
      - Text replacement uses expression to inject participant’s name from Google Sheets Trigger output.  
      - Operates on specified page(s) in the presentation (pageObjectIds: ["p1"]).  
    - *Input:* Takes copied file ID from previous node.  
    - *Output:* Returns updated presentation ID.  
    - *Failure Modes:*  
      - Text replacement fails if placeholder not found or pageObjectId invalid.  
      - API errors or permission issues.  
    - *Credentials:* Google Slides OAuth2 credentials needed.

#### 2.3 File Conversion and Storage

- **Overview:**  
  Converts the updated Google Slides file to PDF, uploads it to a target Drive folder, then deletes the temporary Slides copy to keep storage clean.

- **Nodes Involved:**  
  - Download file (Google Drive)  
  - Upload file (Google Drive)  
  - Delete a file (Google Drive)

- **Node Details:**  
  - **Download file**  
    - *Type & Role:* Downloads the modified Slides file and converts it to PDF format.  
    - *Configuration:*  
      - File ID from replaced Slides node.  
      - Conversion option set to export Slides as PDF.  
      - Output file named `<Participant Name> Certificate`.  
    - *Input:* Presentation ID from Replace text in a presentation node.  
    - *Output:* Binary PDF data for further processing.  
    - *Failure Modes:*  
      - Conversion errors if Slides file corrupted or API limits reached.  
    - *Credentials:* Uses Google Drive OAuth2 credentials.

  - **Upload file**  
    - *Type & Role:* Uploads the generated PDF to a designated Google Drive folder for storage.  
    - *Configuration:*  
      - Target folder specified by URL.  
      - File named `<Participant Name> Certificate.pdf`.  
    - *Input:* Binary PDF data from Download file node.  
    - *Output:* Metadata about the uploaded file.  
    - *Failure Modes:*  
      - Upload failures due to permissions or storage limits.  
    - *Credentials:* Google Drive OAuth2 credentials.

  - **Delete a file**  
    - *Type & Role:* Deletes the temporary Google Slides copy to prevent clutter.  
    - *Configuration:*  
      - File ID taken from Replace text in a presentation output.  
      - Operation is deleteFile.  
    - *Input:* Receives file ID of copied Slides file.  
    - *Output:* Confirmation of deletion.  
    - *Failure Modes:*  
      - File not found or permission denied errors.  
    - *Credentials:* Google Drive OAuth2 credentials.

#### 2.4 Email Delivery

- **Overview:**  
  Sends the generated PDF certificate as an email attachment to the participant’s email address.

- **Nodes Involved:**  
  - Send a message (Gmail)

- **Node Details:**  
  - **Send a message**  
    - *Type & Role:* Sends email through Gmail with personalized message and attached certificate PDF.  
    - *Configuration:*  
      - Recipient email dynamically set from Google Sheets Trigger row.  
      - Email subject: "Your Certificate".  
      - Message body includes participant’s name and congratulatory text.  
      - Attachment: the PDF from Download file node.  
    - *Input:* PDF binary data and participant email/name.  
    - *Output:* Email send confirmation.  
    - *Failure Modes:*  
      - Gmail API authentication or quota limits.  
      - Invalid email addresses causing send failure.  
    - *Credentials:* Gmail OAuth2 credentials required.

---

### 3. Summary Table

| Node Name                  | Node Type             | Functional Role                      | Input Node(s)                | Output Node(s)                     | Sticky Note                                                                                          |
|----------------------------|-----------------------|------------------------------------|-----------------------------|-----------------------------------|----------------------------------------------------------------------------------------------------|
| Google Sheets Trigger       | googleSheetsTrigger   | Detect new participant entries     | —                           | Copy file                         | Getting data from sheets                                                                            |
| Copy file                  | googleDrive           | Copy certificate template          | Google Sheets Trigger        | Replace text in a presentation    | Copy certificate template                                                                          |
| Replace text in a presentation | googleSlides       | Replace placeholder with name      | Copy file                   | Download file                     | Change text                                                                                         |
| Download file              | googleDrive           | Convert Slides to PDF              | Replace text in a presentation | Send a message, Upload file      |                                                                                                    |
| Upload file                | googleDrive           | Save PDF in Drive folder           | Download file               | Delete a file                     | Saving PDFs                                                                                        |
| Delete a file              | googleDrive           | Remove temporary Slides copy       | Upload file                 | —                                 | Deleting temporary PPT                                                                             |
| Send a message             | gmail                 | Email certificate to participant   | Download file               | —                                 |                                                                                                    |
| Sticky Note                | stickyNote            | Workflow guidance and instructions | —                           | —                                 | ## Workflow Guide: Auto-Certificate Generator 🎓 (full content in notes)                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Sheets Trigger Node**  
   - Type: Google Sheets Trigger  
   - Set event to "rowAdded"  
   - Configure polling interval to every minute  
   - Specify the Google Sheet URL and sheet name (gid=0) where new participant data (Name, Email) will be added  
   - Add Google Sheets OAuth2 credentials  

2. **Add Copy file Node (Google Drive)**  
   - Type: Google Drive  
   - Operation: copy  
   - Source file ID: your Google Slides certificate template ID  
   - Destination folder: specify Google Drive folder URL  
   - Name: Set to expression `={{ $json.Name }} Certificate` to personalize filename  
   - Connect input from the Google Sheets Trigger node  
   - Add Google Drive OAuth2 credentials  

3. **Add Replace text in a presentation Node (Google Slides)**  
   - Type: Google Slides  
   - Operation: replaceText  
   - Presentation ID: use expression `={{ $json.id }}` to reference copied file ID  
   - Text replacements: find `[NAME]`, replace with participant's name via expression `={{ $('Google Sheets Trigger').item.json.Name }}`  
   - Page Object IDs: `["p1"]` (adjust if your template uses different slide IDs)  
   - Connect input from Copy file node  
   - Add Google Slides OAuth2 credentials  

4. **Add Download file Node (Google Drive)**  
   - Type: Google Drive  
   - Operation: download  
   - File ID: expression `={{ $json.presentationId }}` (output from Replace text node)  
   - Enable conversion: Slides to PDF (`application/pdf`)  
   - File name: expression `={{ $('Google Sheets Trigger').item.json.Name }} Certificate`  
   - Connect input from Replace text in a presentation node  
   - Add Google Drive OAuth2 credentials  

5. **Add Upload file Node (Google Drive)**  
   - Type: Google Drive  
   - Operation: upload  
   - Destination folder: specify Google Drive folder URL for final PDFs  
   - File name: expression `={{ $('Google Sheets Trigger').item.json.Name }} Certificate.pdf`  
   - Connect input from Download file node  
   - Add Google Drive OAuth2 credentials  

6. **Add Delete a file Node (Google Drive)**  
   - Type: Google Drive  
   - Operation: deleteFile  
   - File ID: expression `={{ $('Replace text in a presentation').item.json.presentationId }}` (the temporary Slides copy)  
   - Connect input from Upload file node  
   - Add Google Drive OAuth2 credentials  

7. **Add Send a message Node (Gmail)**  
   - Type: Gmail  
   - Send To: expression `={{ $('Google Sheets Trigger').item.json.Email }}`  
   - Subject: "Your Certificate"  
   - Message:  
     ```
     Dear {{ $('Google Sheets Trigger').item.json.Name }},
     Congratulations on completing the program 🎉.
     Please find your certificate attached.

     Best regards,
     Your Team
     ```  
   - Attachments: select binary data from Download file node (`data` property)  
   - Connect input from Download file node  
   - Add Gmail OAuth2 credentials  

8. **Connect nodes in sequence:**  
   - Google Sheets Trigger → Copy file → Replace text in a presentation → Download file → (parallel) Send a message & Upload file → Delete a file  

9. **Credentials:**  
   - Ensure Google Sheets, Drive, Slides, and Gmail nodes have properly configured OAuth2 credentials with necessary scopes (read/write access).  

---

### 5. General Notes & Resources

| Note Content                                                                                                                      | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Update **Sheet ID**, **Slides template ID**, **Destination Folder ID**, and **Gmail credentials** with your own before running. | Workflow sticky note content (describes required user-specific configuration).                   |
| Ensure the Google Slides template contains the exact placeholder `[NAME]` where the participant's name should appear.            | Workflow sticky note content.                                                                    |
| Google Drive and Google Slides API permissions must allow file copying, editing, and deletion.                                    | General prerequisite for Google API operations.                                                 |
| Gmail API usage is subject to sending limits and OAuth2 token validity; monitor quota usage to avoid interruptions.               | Gmail node operational note.                                                                     |
| Polling frequency can be adjusted in the Google Sheets Trigger node if near real-time processing is required or to reduce API calls. | Performance tuning note.                                                                          |

---

**Disclaimer:** The provided description and workflow originate solely from an n8n automation workflow. The processing strictly complies with prevailing content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.