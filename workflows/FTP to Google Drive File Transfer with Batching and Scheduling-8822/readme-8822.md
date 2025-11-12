FTP to Google Drive File Transfer with Batching and Scheduling

https://n8nworkflows.xyz/workflows/ftp-to-google-drive-file-transfer-with-batching-and-scheduling-8822


# FTP to Google Drive File Transfer with Batching and Scheduling

### 1. Workflow Overview

This workflow automates the transfer of files from an FTP server to Google Drive, orchestrated with batching and scheduling to handle potentially large file sets efficiently and reliably.

It consists of four logical blocks:

- **1.1 Trigger & File Listing:** Automatically starts on a schedule, connects to the FTP server, and lists files in a specified directory.

- **1.2 Batch Processing Setup:** Splits the list of files into batches to process them sequentially, preventing resource overload.

- **1.3 File Handling:** Downloads each file from the FTP server one by one from the current batch.

- **1.4 Cloud Upload:** Uploads each downloaded file to Google Drive, preserving the original filename.


---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & File Listing

- **Overview:**  
  This block initiates the workflow on a periodic schedule and fetches the list of files from a defined FTP directory.

- **Nodes Involved:**  
  - ⏯️ Schedule Trigger  
  - 📂 List Files from FTP

- **Node Details:**

  - **⏯️ Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Starts the workflow automatically at defined intervals.  
    - Configuration: Runs on a default interval (unspecified in detail, defaults to every minute or as configured in n8n).  
    - Inputs: None (trigger node).  
    - Outputs: Triggers the "List Files from FTP" node.  
    - Edge Cases: Misconfigured schedule intervals could cause too frequent or infrequent runs; ensure correct interval for use case.

  - **📂 List Files from FTP**  
    - Type: FTP Node (List Operation)  
    - Role: Connects to the FTP server and lists files in the remote directory `/path/to/your/files`.  
    - Configuration:  
      - Operation: List  
      - Path: `/path/to/your/files` (customizable FTP path)  
      - Credentials: FTP credentials configured under the credential name `ftp_credentials`.  
    - Inputs: Trigger from Schedule Trigger node.  
    - Outputs: Sends file list to the Batch Files node.  
    - Edge Cases:  
      - FTP connection errors (network issues, auth failures)  
      - Empty directory (resulting in no files to process)  
      - Permission denied on FTP path  
    - Version: Uses FTP node version 1.

---

#### 1.2 Batch Processing Setup

- **Overview:**  
  Splits the file list into manageable batches to process files sequentially, avoiding system overload and ensuring robust handling.

- **Nodes Involved:**  
  - 🔀 Batch Files

- **Node Details:**

  - **🔀 Batch Files**  
    - Type: SplitInBatches  
    - Role: Divides the list of files into batches for sequential processing.  
    - Configuration: Default batch size (not explicitly set, so defaults to 1, processing files one at a time).  
    - Inputs: Receives the file list from the FTP list node.  
    - Outputs: Sends each batch item (individual file) first to "Download File from FTP" and also to "Upload to Google Drive" (the latter is connected but only triggered after download).  
    - Edge Cases:  
      - If batch size is not set, default is 1; batch size can be customized for parallel processing but may risk rate limits or overload.  
      - Errors in splitting or empty input data will result in no downstream processing.  
    - Version: Version 3 of SplitInBatches node.

---

#### 1.3 File Handling

- **Overview:**  
  Downloads each file from FTP one at a time as dictated by the batch processing.

- **Nodes Involved:**  
  - ⬇️ Download File from FTP

- **Node Details:**

  - **⬇️ Download File from FTP**  
    - Type: FTP Node (Download Operation)  
    - Role: Downloads the current batch file from the FTP server.  
    - Configuration:  
      - Operation: Download (implied by node type and parameter context)  
      - Path: Dynamic, set by expression `={{ $json.name }}`, meaning it uses the filename from the batch item JSON.  
      - Credentials: Uses `ftp_credentials` (same as listing node).  
    - Inputs: Receives file info from "Batch Files" node.  
    - Outputs: Passes the downloaded file data to "Upload to Google Drive".  
    - Edge Cases:  
      - Missing or incorrect filename in batch item JSON leads to download failure.  
      - FTP connection or permission errors.  
      - Large file size causing timeouts or memory issues.  
    - Version: FTP node version 1.

---

#### 1.4 Cloud Upload

- **Overview:**  
  Uploads the downloaded file to Google Drive, preserving its original filename and placing it in the root folder.

- **Nodes Involved:**  
  - ☁️ Upload to Google Drive

- **Node Details:**

  - **☁️ Upload to Google Drive**  
    - Type: Google Drive Node (Upload File)  
    - Role: Uploads the file received from the FTP download step to Google Drive.  
    - Configuration:  
      - Filename: Dynamic, uses expression `={{ $json.name }}` to retain FTP filename.  
      - Drive ID: Set to "My Drive" (default Google Drive).  
      - Folder ID: Set to "root" folder (Google Drive root directory).  
      - Credentials: Uses OAuth2 credentials configured as `google_drive_credentials`.  
    - Inputs: Receives file binary data from "Download File from FTP".  
    - Outputs: None (end of processing chain).  
    - Edge Cases:  
      - Google Drive API quota limits or authentication failures.  
      - Filename conflicts (overwriting existing files with same name).  
      - Upload failures due to file size or network issues.  
    - Version: Google Drive node version 3.

---

### 3. Summary Table

| Node Name                  | Node Type            | Functional Role                    | Input Node(s)           | Output Node(s)             | Sticky Note                                                                                                 |
|----------------------------|----------------------|----------------------------------|------------------------|----------------------------|-------------------------------------------------------------------------------------------------------------|
| ⏯️ Schedule Trigger         | Schedule Trigger     | Initiates workflow periodically  | None                   | 📂 List Files from FTP      | ## 1. Trigger & File Listing: *The workflow starts automatically using a Schedule trigger.*                 |
| 📂 List Files from FTP      | FTP (List)           | Lists files on FTP server        | ⏯️ Schedule Trigger     | 🔀 Batch Files              | *It connects to the FTP server and retrieves a list of files from the given remote folder path.*             |
| 🔀 Batch Files              | SplitInBatches       | Splits files into batches        | 📂 List Files from FTP  | ⬇️ Download File from FTP, ☁️ Upload to Google Drive | ## 2. Batch Processing Setup: *Files are split into manageable batches to process sequentially.*           |
| ⬇️ Download File from FTP   | FTP (Download)       | Downloads each file from FTP     | 🔀 Batch Files          | ☁️ Upload to Google Drive   | ## 3. File Handling: *Each batch item (file) is downloaded from the FTP server.*                            |
| ☁️ Upload to Google Drive   | Google Drive Upload  | Uploads file to Google Drive     | ⬇️ Download File from FTP| None                      | ## 4. Cloud Upload: *Files are uploaded to Google Drive, retaining the original filename.*                   |
| Sticky Note                | Sticky Note          | Informational comments           | None                   | None                       | Covers multiple nodes with descriptive block-level commentary.                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n**, name it e.g., "FTP to Cloud Sync".

2. **Add a Schedule Trigger node:**  
   - Set it to trigger at desired intervals (e.g., every hour or daily).  
   - Leave default configuration or customize interval as required.

3. **Add an FTP node configured to list files:**  
   - Set operation to "List".  
   - Set Path to the FTP folder you want to monitor, e.g., `/path/to/your/files`.  
   - Configure and attach FTP credentials (`ftp_credentials`) with access to your FTP server.

4. **Connect Schedule Trigger node output to FTP List node input.**

5. **Add a SplitInBatches node:**  
   - Connect the FTP List node output to this node.  
   - Optionally set batch size (default is 1 for sequential processing).

6. **Add an FTP node configured to download files:**  
   - Set operation to Download (default for FTP node if path provided).  
   - Set Path parameter to `={{ $json.name }}` to dynamically download each file by name.  
   - Use the same FTP credentials as the listing FTP node.

7. **Connect SplitInBatches node output (main output 2) to Download node input.**

8. **Add a Google Drive node configured to upload files:**  
   - Set the operation to Upload (default).  
   - For the file name, use expression `={{ $json.name }}` to keep original filename.  
   - Set Drive ID to "My Drive" or your specific drive.  
   - Set Folder ID to "root" or specify target folder ID if different.  
   - Attach OAuth2 Google Drive credentials (`google_drive_credentials`).

9. **Connect the Download FTP node output to Google Drive upload node input.**

10. **Verify credentials for FTP and Google Drive:**  
    - FTP: username, password, host, port as per your FTP server.  
    - Google Drive: OAuth2 credentials with scopes for file upload.

11. **Set workflow to active and test by running manually or wait for scheduled trigger.**

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                             |
|-----------------------------------------------------------------------------------------------------------|---------------------------------------------|
| Workflow designed to handle large file sets efficiently by processing files one at a time in batches.    | Important for avoiding API rate limits and system overload. |
| Uses standard FTP and Google Drive nodes; credentials must be securely configured in n8n.                 | FTP and Google Drive credential setup.      |
| For Google Drive upload, filename conflicts might overwrite existing files; consider adding versioning.  | Potential enhancement for production use.   |
| Sticky notes included in the workflow provide detailed explanation of each logical block and their roles. | Helpful for maintainers and collaborators.  |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected materials. All data handled is legal and public.