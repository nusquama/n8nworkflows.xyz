Automated Slack-to-Jira Issue Creation with Attachments

https://n8nworkflows.xyz/workflows/automated-slack-to-jira-issue-creation-with-attachments-8813


# Automated Slack-to-Jira Issue Creation with Attachments

### 1. Workflow Overview

This workflow automates the process of creating Jira issues based on messages posted in a designated Slack channel. It extracts structured issue details from Slack message texts, creates Jira issues accordingly, and handles any attachments included in the Slack messages by downloading and uploading them to the corresponding Jira issue. Finally, it posts a confirmation message back in Slack with the Jira issue details.

The workflow is logically divided into the following blocks:

- **1.1 Slack Input Monitoring and Message Validation**  
  Listens for new messages in a specific Slack channel and filters messages containing issue report data.

- **1.2 Message Parsing and Data Extraction**  
  Parses the Slack message text (including rich text blocks) to extract issue title, description, priority, issue type, and attachments.

- **1.3 Jira Issue Creation**  
  Creates a new Jira issue with the extracted data, including priority and issue type mappings.

- **1.4 Attachment Processing**  
  If attachments exist, downloads each file from Slack and uploads them to the created Jira issue, handling multiple attachments sequentially.

- **1.5 Confirmation Notification**  
  Sends a summary confirmation message back to a designated Slack channel with Jira issue details and upload status.

---

### 2. Block-by-Block Analysis

#### 2.1 Slack Input Monitoring and Message Validation

**Overview:**  
This block triggers the workflow upon receiving a new message in a specific Slack channel and filters messages to proceed only if they contain the keyword "title" (indicating a structured issue report).

**Nodes Involved:**  
- Slack Trigger  
- checking message (If node)

**Node Details:**  

- **Slack Trigger**  
  - Type: Slack Trigger  
  - Role: Listens for new messages posted in Slack channel ID `C09CVLMSF3R` ("issue-smarteremr").  
  - Configuration: Trigger on message events only; uses Slack API credentials.  
  - Input: Slack message event data (including text, user, files).  
  - Output: Raw Slack message JSON.  
  - Potential Failures: Slack API authentication errors, webhook misconfiguration, channel ID errors.

- **checking message**  
  - Type: If (Condition)  
  - Role: Checks if the message text contains the string "title" (case-sensitive).  
  - Configuration: Condition: message text contains "title".  
  - Input: Slack message JSON from Slack Trigger.  
  - Output: Passes messages with "title" in text; discards others.  
  - Edge Cases: Messages without "title" keyword will not trigger further processing.

---

#### 2.2 Message Parsing and Data Extraction

**Overview:**  
Parses Slack message text and rich text blocks using regex to extract issue title, description, priority, and issue type. Also processes attachment metadata.

**Nodes Involved:**  
- Parse Message into Jira Format (Code node)

**Node Details:**  

- **Parse Message into Jira Format**  
  - Type: Code (JavaScript)  
  - Role: Extracts structured issue fields from Slack message text and blocks; processes attachments metadata.  
  - Configuration:  
    - Supports rich text extraction from Slack blocks for more complete text.  
    - Regex patterns to extract `title`, `description`, `priority`, and `type` from the message text.  
    - Cleans extracted text for Jira compatibility.  
    - Maps Slack priority values to Jira priority levels (critical, high, medium, low).  
    - Maps issue types to Jira issue type IDs.  
    - Prepares Jira description with metadata (reporter, channel, timestamp, attachment list).  
    - Processes attachment info including URL resolution, file type, size, and flags for video/large files.  
  - Input: Slack message JSON from checking message node.  
  - Output: JSON object with extracted fields: summary, description, priority, priorityId, issueType, issueTypeName, attachments array, and debugging info.  
  - Edge Cases:  
    - Messages with missing or malformed fields fallback to defaults.  
    - Rich text extraction handles multiple block formats.  
    - Attachments without valid URLs are ignored.  
    - Large or video files flagged but not blocked.  
  - Version: Uses n8n Code node v2.

---

#### 2.3 Jira Issue Creation

**Overview:**  
Creates a new Jira issue in a specified project with extracted title, description, priority, and reporter information.

**Nodes Involved:**  
- Create Jira Issue (Jira node)  
- Check Attachments (If node)

**Node Details:**  

- **Create Jira Issue**  
  - Type: Jira node  
  - Role: Submits a new issue to Jira project "Kanban" (project ID 10001) using the parsed data.  
  - Configuration:  
    - Summary from parsed message summary.  
    - Issue type fixed to type ID 10006 (Task) but overridden by parsed issue type if needed.  
    - Assignee and reporter set to user "Vivek Patidar" (user ID 712020).  
    - Description field populated by parsed description.  
    - Uses Jira Software Cloud API credentials.  
  - Input: Parsed message data from Parse Message node.  
  - Output: Jira issue details including unique issue key.  
  - Potential Failures: Jira API authentication issues, invalid project or issue type, network errors.

- **Check Attachments**  
  - Type: If (Condition)  
  - Role: Checks if there are any attachments to process (files.totalFiles > 0).  
  - Input: Jira issue creation output and parsed files info.  
  - Output:  
    - If attachments exist: proceeds to prepare files for processing.  
    - If no attachments: proceeds directly to Slack confirmation.  
  - Edge Cases: Avoids unnecessary file processing if no attachments.

---

#### 2.4 Attachment Processing

**Overview:**  
Processes each attachment by fetching file info from Slack, downloading the file using authenticated URLs, then uploading the file to the created Jira issue as an attachment. Handles multiple attachments by splitting into batches.

**Nodes Involved:**  
- Prepare Files for Processing (Code node)  
- Split Files into Batches (SplitInBatches node)  
- Get File Info (HTTP Request node)  
- Download Attachment (HTTP Request node)  
- Upload Attachment to Jira (Jira node)

**Node Details:**  

- **Prepare Files for Processing**  
  - Type: Code (JavaScript)  
  - Role: Extracts attachment info from parsed message and Jira issue key; creates individual items for each attachment with relevant metadata.  
  - Input: Parsed message data and Jira issue details.  
  - Output: Array of file objects, each with fileId, fileName, fileUrl, Jira issue key, file index, etc.  
  - Edge Cases: Handles empty attachment arrays gracefully.

- **Split Files into Batches**  
  - Type: SplitInBatches  
  - Role: Processes attachments one at a time (batch size = 1) for sequential handling.  
  - Input: Array of attachment items from previous node.  
  - Output: Single attachment item per execution cycle.  
  - Edge Cases: Ensures memory-efficient processing of multiple files.

- **Get File Info**  
  - Type: HTTP Request  
  - Role: Requests detailed Slack file metadata using Slack API and file ID.  
  - Configuration:  
    - URL: Slack API endpoint `files.info` with file ID parameter.  
    - Authorization: Bearer token from Slack credentials in headers.  
  - Input: Attachment item with file ID.  
  - Output: Slack file info JSON.  
  - Potential Failures: Slack API rate limiting, invalid file ID, auth errors.

- **Download Attachment**  
  - Type: HTTP Request  
  - Role: Downloads actual file binary data from Slack using private download URL.  
  - Configuration:  
    - URL: `url_private_download` from file info.  
    - Authorization: Bearer token in headers.  
    - Response Format: File binary data.  
  - Input: File info from previous node.  
  - Output: Binary file content for upload.  
  - Potential Failures: Download failures, expired URLs, large file timeouts.

- **Upload Attachment to Jira**  
  - Type: Jira node  
  - Role: Uploads binary file as an attachment to the Jira issue using the issue key.  
  - Configuration:  
    - Issue Key dynamically from batch item.  
    - Upload file from binary data of HTTP Request node.  
    - Uses Jira Software Cloud API credentials.  
  - Input: Binary file data and Jira issue key.  
  - Output: Confirmation of attachment upload.  
  - Edge Cases: API limits on attachment size or file types, upload failures.

- **Attachment Processing Loop**  
  - After upload, loops back to Split Files into Batches to process next attachment until all are uploaded.

---

#### 2.5 Confirmation Notification

**Overview:**  
Sends a confirmation message to a specified Slack channel summarizing the Jira issue creation and attachment upload status.

**Nodes Involved:**  
- Reply to Channel On Slack (Slack node)

**Node Details:**  

- **Reply to Channel On Slack**  
  - Type: Slack node (Post message)  
  - Role: Posts a formatted message in Slack channel ID `C09D7N4MXFF` ("tickets") confirming issue creation.  
  - Configuration:  
    - Message includes Jira issue link, title, description, issue type, priority, and number of attachments uploaded.  
    - Uses Slack API credentials.  
  - Input: Jira issue data, parsed message data, attachment count.  
  - Output: Slack message confirmation.  
  - Potential Failures: Slack API auth errors, channel permission errors.

---

### 3. Summary Table

| Node Name                 | Node Type           | Functional Role                                  | Input Node(s)                 | Output Node(s)                | Sticky Note                                                                                                       |
|---------------------------|---------------------|-------------------------------------------------|------------------------------|------------------------------|------------------------------------------------------------------------------------------------------------------|
| Slack Trigger             | Slack Trigger       | Listens for new messages in Slack channel       | —                            | checking message              | Purpose: Monitors specific Slack channel for new messages; triggers workflow on message events; outputs raw data |
| checking message          | If                  | Filters messages containing "title" keyword     | Slack Trigger                | Parse Message into Jira Format | Purpose: Extracts structured issue details from Slack message text using regex; maps priorities                  |
| Parse Message into Jira Format | Code               | Parses Slack message to extract issue fields     | checking message             | Create Jira Issue             | Purpose: Extracts structured issue details, attachments metadata, and prepares Jira description                  |
| Create Jira Issue         | Jira                | Creates new Jira issue with parsed data          | Parse Message into Jira Format | Check Attachments             | Purpose: Creates new Jira issue with parsed data, outputs issue key                                             |
| Check Attachments         | If                  | Checks if attachments exist to continue processing | Create Jira Issue            | Prepare Files for Processing / Reply to Channel On Slack | —                                                                                                                |
| Prepare Files for Processing | Code               | Prepares attachment files for batch processing   | Check Attachments            | Split Files into Batches      | Purpose: Prepares file data array for batch processing                                                          |
| Split Files into Batches  | SplitInBatches      | Splits attachments into single batches            | Prepare Files for Processing | Reply to Channel On Slack / Get File Info | Purpose: Processes each attachment separately                                                                    |
| Get File Info             | HTTP Request        | Fetches detailed Slack file info                  | Split Files into Batches     | Download Attachment           | Purpose: Downloads files from Slack using private URLs with auth                                                 |
| Download Attachment       | HTTP Request        | Downloads file binary from Slack                   | Get File Info               | Upload Attachment to Jira     | —                                                                                                                |
| Upload Attachment to Jira | Jira                | Uploads attachment file to Jira issue              | Download Attachment          | Split Files into Batches      | Purpose: Uploads binary file data to Jira issue; preserves screenshots/documentation                              |
| Reply to Channel On Slack | Slack               | Sends confirmation message back to Slack          | Check Attachments / Split Files into Batches | —                            | Purpose: Sends confirmation message to Slack channel with Jira issue details                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Slack Trigger node**  
   - Type: Slack Trigger  
   - Configure to listen on `message` event type.  
   - Set channel to ID `C09CVLMSF3R` ("issue-smarteremr").  
   - Connect Slack API credentials.  

2. **Add If node named "checking message"**  
   - Condition: `$json.text` contains substring `"title"` (case sensitive).  
   - Input: Slack Trigger output.  
   - Output: Proceed only if condition met.

3. **Add Code node named "Parse Message into Jira Format"**  
   - Paste the provided JavaScript code for parsing Slack message text including rich text blocks, extracting title, description, priority, issue type, and attachments metadata.  
   - Input: Output from "checking message".  
   - Set node version to 2.

4. **Add Jira node named "Create Jira Issue"**  
   - Configure to create issue in project ID `10001` ("Kanban").  
   - Set summary to `={{ $json.summary }}` from parse node output.  
   - Set issue type to ID `10006` (Task) or dynamic if desired.  
   - Set assignee and reporter to user ID `712020` ("Vivek Patidar").  
   - Set description field to `={{ $json.extractedData.descriptionMatch }}`.  
   - Connect Jira Software Cloud API credentials.  
   - Input: Output from "Parse Message into Jira Format".

5. **Add If node named "Check Attachments"**  
   - Condition: Number of total files `{{ $json.files.totalFiles }}` > 0.  
   - Input: Output from "Create Jira Issue".  
   - If yes, proceed to prepare files; if no, proceed to Slack reply.

6. **Add Code node named "Prepare Files for Processing"**  
   - Use provided JavaScript code extracting attachments and Jira issue key, creating individual file objects for batch processing.  
   - Input: Output from "Parse Message into Jira Format" and "Create Jira Issue".

7. **Add SplitInBatches node named "Split Files into Batches"**  
   - Set batch size to 1 to process files sequentially.  
   - Input: Output from "Prepare Files for Processing".

8. **Add HTTP Request node named "Get File Info"**  
   - Set URL: `https://slack.com/api/files.info?file={{ $json.fileId }}`.  
   - Add HTTP header: `Authorization: Bearer {{$credentials.slackApi.accessToken}}`.  
   - Input: Output from "Split Files into Batches".

9. **Add HTTP Request node named "Download Attachment"**  
   - Set URL: `={{ $json.file.url_private_download }}` (from Get File Info output).  
   - Add HTTP header: `Authorization: Bearer {{$credentials.slackApi.accessToken}}`.  
   - Set response format to file/binary.  
   - Input: Output from "Get File Info".

10. **Add Jira node named "Upload Attachment to Jira"**  
    - Set resource to `issueAttachment`.  
    - Set issueKey to `={{ $json.jiraIssueKey }}` from batch item.  
    - Attach binary data from "Download Attachment" node.  
    - Connect Jira Software Cloud API credentials.  
    - Input: Output from "Download Attachment".

11. **Connect "Upload Attachment to Jira" output back to "Split Files into Batches" input**  
    - This creates a loop to process all attachments sequentially.

12. **Add Slack node named "Reply to Channel On Slack"**  
    - Set to post message in channel ID `C09D7N4MXFF` ("tickets").  
    - Compose message text with Jira issue key link, title, description, issue type, priority, and number of attachments uploaded using expressions referencing previous nodes.  
    - Connect Slack API credentials.  
    - Input: From either "Check Attachments" node (if no attachments) or "Split Files into Batches" node (after all attachments processed).

---

### 5. General Notes & Resources

| Note Content                                                                                                       | Context or Link                                                                                              |
|--------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| This workflow preserves screenshots and documentation from Slack messages by attaching files directly to Jira issues. | Sticky Note7 (Attachment upload purpose)                                                                    |
| Files are downloaded from Slack using private URLs with bearer token authentication to ensure security and access. | Sticky Note8 (Slack attachment download details)                                                            |
| The workflow uses advanced regex and rich text parsing to robustly extract issue details from Slack messages.      | Sticky Note10 (Parsing logic explanation)                                                                    |
| It supports handling multiple attachments by splitting them into batches to avoid memory or API limits.            | Sticky Note13 and Sticky Note - Split Files (Batch processing explanation)                                   |
| Confirmation messages include clickable Jira issue keys and summary details for easy tracking in Slack.             | Sticky Note12 (Slack confirmation purpose)                                                                   |
| Project and user IDs like Jira project (10001) and user (712020) must be adapted to your Jira instance configuration. | Configuration note                                                                                             |
| Slack channel IDs (e.g., `C09CVLMSF3R` and `C09D7N4MXFF`) must be set according to your Slack workspace setup.     | Configuration note                                                                                             |
| Slack API token scopes must include permissions to read messages, files, and post messages in relevant channels.    | Slack API permissions requirement                                                                              |
| Jira API credentials require permissions to create issues and upload attachments in the specified project.           | Jira API permission requirement                                                                                |

---

This reference document fully describes the "Automated Slack-to-Jira Issue Creation with Attachments" workflow, enabling reproduction, modification, and troubleshooting by advanced users and automation agents.