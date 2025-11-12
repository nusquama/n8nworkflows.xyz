Automatically Save & Organize LINE Message Files in Google Drive with Sheets Logging

https://n8nworkflows.xyz/workflows/automatically-save---organize-line-message-files-in-google-drive-with-sheets-logging-3191


# Automatically Save & Organize LINE Message Files in Google Drive with Sheets Logging

### 1. Workflow Overview

This workflow automates the process of saving files received via the LINE Messaging API into Google Drive, organizing them into folders based on configurable criteria, and logging file metadata into a Google Sheet. It also provides immediate feedback to the LINE user by sending a reply message with either the file URL or an error notification if the file type is disallowed.

The workflow is structured into the following logical blocks:

- **1.1 Workflow Entry & Configuration:** Listens for incoming LINE webhook events and retrieves configuration settings from Google Sheets. Merges event data with configuration and determines folder naming conventions.

- **1.2 Folder Search & Creation:** Searches for existing folders in Google Drive based on date and file type criteria. Creates folders if they do not exist and consolidates folder IDs to determine the final upload destination.

- **1.3 File Retrieval & Validation:** Downloads the binary content of the file from LINE and validates the file type against allowed types specified in the configuration.

- **1.4 File Upload, Logging & Reply:** Uploads the validated file to Google Drive, logs file details into Google Sheets, checks if reply messages are enabled, and sends a reply message back to the LINE user with success or error information.

---

### 2. Block-by-Block Analysis

#### 2.1 Workflow Entry & Configuration

**Overview:**  
This block initiates the workflow by receiving POST requests from LINE containing file messages. It retrieves configuration parameters from a Google Sheet and merges these with the event data. It then determines folder names based on configuration settings such as storing by date or file type.

**Nodes Involved:**  
- LINE Webhook Listener  
- Get Config  
- Merge Event and Config Data  
- Process Event and Config Data  
- Determine Folder Info  

**Node Details:**

- **LINE Webhook Listener**  
  - *Type:* Webhook  
  - *Role:* Receives POST requests from LINE Messaging API containing file upload events and metadata.  
  - *Configuration:* HTTP method POST, path "line-webhook".  
  - *Inputs:* External HTTP POST request.  
  - *Outputs:* Event JSON data with message details.  
  - *Edge Cases:* Invalid or malformed webhook requests; missing event data.  

- **Get Config**  
  - *Type:* Google Sheets  
  - *Role:* Reads configuration data from a predefined Google Sheet range ("config!A1:H2").  
  - *Configuration:* Uses OAuth2 credentials for Google Sheets; reads specific columns including Parent Folder ID, allowed file types, folder organization flags, current date, reply enabled flag, and LINE channel access token.  
  - *Inputs:* Triggered after webhook.  
  - *Outputs:* Configuration JSON object.  
  - *Edge Cases:* Sheet not found, permission errors, empty or malformed config data.  

- **Merge Event and Config Data**  
  - *Type:* Merge  
  - *Role:* Combines the event data from LINE and the configuration data from Google Sheets by index.  
  - *Configuration:* Mode set to "mergeByIndex".  
  - *Inputs:* Event data and config data.  
  - *Outputs:* Single merged JSON object containing both event and config.  
  - *Edge Cases:* Mismatched data lengths or missing inputs.  

- **Process Event and Config Data**  
  - *Type:* Function (Code)  
  - *Role:* Structures and validates merged data, ensuring config is present and properly formatted. Throws error if config is missing.  
  - *Key Expressions:* Extracts config fields from either second item or event data fallback.  
  - *Inputs:* Merged event and config data.  
  - *Outputs:* JSON object with separated event and config properties.  
  - *Edge Cases:* Missing config fields, malformed input data.  

- **Determine Folder Info**  
  - *Type:* Function (Code)  
  - *Role:* Determines folder names for storing files based on config flags: date folder name (using current date or config date), file type folder name (based on message type). Also passes base folder ID.  
  - *Key Expressions:* Uses ISO date format without dashes for date folder; lowercases message type for file type folder.  
  - *Inputs:* Processed event and config data.  
  - *Outputs:* JSON with folder naming info and flags.  
  - *Edge Cases:* Missing event or config data, unexpected message types.  

---

#### 2.2 Folder Search & Creation

**Overview:**  
This block manages Google Drive folder structure. It searches for existing folders based on date and file type, creates them if missing, and consolidates folder IDs to determine the final upload folder.

**Nodes Involved:**  
- Search Date Folder  
- Check Existing Date Folder  
- Create Date Folder  
- Set Date Folder ID  
- Search File Type Folder  
- Check Existing File Type Folder  
- Create File Type Folder  
- Set File Type Folder ID  
- Merge Final Parent Folder Data  
- Configure Final Parent Folder ID  
- Check if Store by Date is Enabled  
- Check if Store by File Type is Enabled  

**Node Details:**

- **Search Date Folder**  
  - *Type:* Google Drive (Search)  
  - *Role:* Searches for a folder named as the date folder inside the base parent folder.  
  - *Configuration:* Query string set to date folder name; folderId set to base folder ID from config.  
  - *Inputs:* Folder info from previous block.  
  - *Outputs:* Search results with folder metadata if found.  
  - *Edge Cases:* Folder not found, API rate limits, permission errors.  

- **Check Existing Date Folder**  
  - *Type:* IF  
  - *Role:* Checks if the date folder search returned a folder ID (exists).  
  - *Configuration:* Condition checks if `id` field is defined.  
  - *Inputs:* Output of Search Date Folder.  
  - *Outputs:* Routes to create folder if not found or to set folder ID if found.  
  - *Edge Cases:* Empty search results.  

- **Create Date Folder**  
  - *Type:* Google Drive (Create Folder)  
  - *Role:* Creates a new folder named as the date folder inside the base folder if it does not exist.  
  - *Configuration:* Folder name from "Determine Folder Info"; parent folder ID from base folder ID.  
  - *Inputs:* Triggered if date folder missing.  
  - *Outputs:* New folder metadata including ID.  
  - *Edge Cases:* Creation failure due to permissions or quota.  

- **Set Date Folder ID**  
  - *Type:* Code  
  - *Role:* Consolidates folder ID from either existing folder or newly created folder and attaches it to the data for downstream use.  
  - *Key Expressions:* Checks multiple inputs for folder ID; sets `targetParentId`.  
  - *Inputs:* Outputs from Check Existing Date Folder and Create Date Folder.  
  - *Outputs:* JSON with `targetParentId` for date folder.  
  - *Edge Cases:* Missing folder IDs in inputs.  

- **Search File Type Folder**  
  - *Type:* Google Drive (Search)  
  - *Role:* Searches for a subfolder named after the file type inside the date folder or base folder depending on config.  
  - *Configuration:* Query string set to file type folder name; folderId dynamically set based on whether Store by Date is enabled and date folder ID availability.  
  - *Inputs:* Folder info and date folder ID.  
  - *Outputs:* Search results for file type folder.  
  - *Edge Cases:* Folder not found, permission issues.  

- **Check Existing File Type Folder**  
  - *Type:* IF  
  - *Role:* Checks if the file type folder exists by verifying presence of folder ID.  
  - *Configuration:* Condition checks if `id` field is defined.  
  - *Inputs:* Output of Search File Type Folder.  
  - *Outputs:* Routes to create folder if missing or to set folder ID if found.  
  - *Edge Cases:* Empty search results.  

- **Create File Type Folder**  
  - *Type:* Google Drive (Create Folder)  
  - *Role:* Creates a new folder named after the file type inside the date folder or base folder.  
  - *Configuration:* Folder name from "Determine Folder Info"; parent folder ID dynamically set based on config and date folder ID.  
  - *Inputs:* Triggered if file type folder missing.  
  - *Outputs:* New folder metadata including ID.  
  - *Edge Cases:* Creation failure due to permissions or quota.  

- **Set File Type Folder ID**  
  - *Type:* Code  
  - *Role:* Consolidates folder ID from existing or newly created file type folder and attaches it to data for downstream use.  
  - *Key Expressions:* Similar logic as Set Date Folder ID node.  
  - *Inputs:* Outputs from Check Existing File Type Folder and Create File Type Folder.  
  - *Outputs:* JSON with `targetParentId` for file type folder.  
  - *Edge Cases:* Missing folder IDs.  

- **Merge Final Parent Folder Data**  
  - *Type:* Merge  
  - *Role:* Merges the date folder ID and file type folder ID data streams for final folder determination.  
  - *Inputs:* Outputs from Set Date Folder ID and Set File Type Folder ID.  
  - *Outputs:* Combined data for final folder configuration.  
  - *Edge Cases:* Mismatched data lengths.  

- **Configure Final Parent Folder ID**  
  - *Type:* Code  
  - *Role:* Determines the final folder ID where the file will be uploaded based on config flags:  
    - Store by Date & File Type = true → file type folder ID  
    - Store by Date true & File Type false → date folder ID  
    - Store by Date false & File Type true → file type folder ID  
    - Both false → parent folder ID from config  
  - *Inputs:* Merged folder data and config.  
  - *Outputs:* JSON with `finalParentId` for upload.  
  - *Edge Cases:* Missing folder IDs, inconsistent config flags.  

- **Check if Store by Date is Enabled**  
  - *Type:* IF  
  - *Role:* Checks if "Store by Date" flag in config is true to decide whether to create or use date folder.  
  - *Inputs:* Config data.  
  - *Outputs:* Routes to create date folder or skip.  
  - *Edge Cases:* Missing or malformed config flag.  

- **Check if Store by File Type is Enabled**  
  - *Type:* IF  
  - *Role:* Checks if "Store by File Type" flag in config is true to decide whether to create or use file type folder.  
  - *Inputs:* Config data.  
  - *Outputs:* Routes to create file type folder or skip.  
  - *Edge Cases:* Missing or malformed config flag.  

---

#### 2.3 File Retrieval & Validation

**Overview:**  
This block retrieves the actual file binary content from the LINE Messaging API and validates the file type against allowed types defined in the configuration. If the file type is not allowed, it throws an error to be handled downstream.

**Nodes Involved:**  
- Get File Binary Content  
- Validate File Type  

**Node Details:**

- **Get File Binary Content**  
  - *Type:* HTTP Request  
  - *Role:* Downloads the binary content of the file from LINE API using the message ID.  
  - *Configuration:* URL constructed dynamically using message ID from webhook event; uses HTTP header authentication with LINE Channel Access Token.  
  - *Inputs:* Final folder ID configuration (to ensure proper sequencing).  
  - *Outputs:* Binary file data with file extension metadata.  
  - *Edge Cases:* Network errors, invalid message ID, authentication failure, timeouts.  

- **Validate File Type**  
  - *Type:* Function (Code)  
  - *Role:* Checks if the file type (message type) is included in the allowed types list from config. Throws an error if disallowed.  
  - *Key Expressions:* Splits allowed types string by "|" and compares lowercased file type.  
  - *Inputs:* Binary file data and config.  
  - *Outputs:* Passes items if valid; throws error otherwise.  
  - *Edge Cases:* Missing allowed types config, unexpected file types, error propagation for reply messaging.  

---

#### 2.4 File Upload, Logging & Reply

**Overview:**  
This block uploads the validated file to Google Drive in the determined folder, logs file metadata into a Google Sheet, checks if reply messages are enabled, and sends a reply message back to the LINE user with either the file URL or an error message.

**Nodes Involved:**  
- Upload File to Google Drive  
- Log File Details to Google Sheet  
- Check Reply Enabled Flag  
- Send LINE Reply Message  

**Node Details:**

- **Upload File to Google Drive**  
  - *Type:* Google Drive (Upload)  
  - *Role:* Uploads the binary file to the final parent folder ID determined earlier.  
  - *Configuration:* File name constructed from event timestamp and file extension; folder ID set dynamically.  
  - *Inputs:* Validated binary file data and final folder ID.  
  - *Outputs:* Metadata of uploaded file including Google Drive URL.  
  - *Edge Cases:* Upload failures, quota limits, permission errors.  

- **Log File Details to Google Sheet**  
  - *Type:* Google Sheets (Append)  
  - *Role:* Appends a new row to the "fileList" sheet with file name, upload date, Google Drive URL, and file type.  
  - *Configuration:* Uses OAuth2 credentials; maps fields from upload metadata.  
  - *Inputs:* Upload metadata.  
  - *Outputs:* Confirmation of append operation.  
  - *Edge Cases:* Sheet access errors, malformed data.  

- **Check Reply Enabled Flag**  
  - *Type:* IF  
  - *Role:* Checks if the "Reply Enabled" flag in config is true to decide whether to send a reply message.  
  - *Inputs:* Config data.  
  - *Outputs:* Routes to send reply or skip.  
  - *Edge Cases:* Missing flag or malformed config.  

- **Send LINE Reply Message**  
  - *Type:* HTTP Request  
  - *Role:* Sends a reply message to the LINE user via LINE Messaging API. The message contains either the Google Drive file URL or an error message from validation.  
  - *Configuration:* POST to LINE reply API endpoint; uses HTTP header authentication with Channel Access Token; JSON body dynamically constructed with replyToken and message text.  
  - *Inputs:* Reply token from webhook and message content from validation or upload results.  
  - *Outputs:* API response from LINE.  
  - *Edge Cases:* Authentication failure, invalid reply token, network errors.  

---

### 3. Summary Table

| Node Name                      | Node Type          | Functional Role                                | Input Node(s)                       | Output Node(s)                          | Sticky Note                                                                                                      |
|--------------------------------|--------------------|-----------------------------------------------|-----------------------------------|---------------------------------------|------------------------------------------------------------------------------------------------------------------|
| LINE Webhook Listener           | Webhook            | Receives LINE webhook POST requests            | -                                 | Merge Event and Config Data, Get Config | ## Workflow Entry & Configuration: Receives POST requests from LINE.                                            |
| Get Config                     | Google Sheets      | Reads configuration data from Google Sheet     | LINE Webhook Listener             | Merge Event and Config Data            | ## Workflow Entry & Configuration: Reads config data from Google Sheets.                                        |
| Merge Event and Config Data     | Merge              | Merges event and config data                    | LINE Webhook Listener, Get Config | Process Event and Config Data          | ## Workflow Entry & Configuration: Combines event and config data.                                              |
| Process Event and Config Data   | Function (Code)    | Structures and validates merged data            | Merge Event and Config Data       | Determine Folder Info                  | ## Workflow Entry & Configuration: Processes merged data.                                                       |
| Determine Folder Info           | Function (Code)    | Determines folder names and flags                | Process Event and Config Data     | Search Date Folder                    | ## Workflow Entry & Configuration: Calculates folder names based on config.                                     |
| Search Date Folder             | Google Drive (Search) | Searches for date folder in Google Drive        | Determine Folder Info             | Check Existing Date Folder             | ## Folder Search & Creation: Searches for date folder.                                                          |
| Check Existing Date Folder      | IF                 | Checks if date folder exists                      | Search Date Folder                | Set Date Folder ID, Check if Store by Date is Enabled | ## Folder Search & Creation: Checks existence of date folder.                                                   |
| Check if Store by Date is Enabled | IF              | Checks if storing by date is enabled             | Check Existing Date Folder        | Create Date Folder, Set Date Folder ID | ## Folder Search & Creation: Conditional on Store by Date flag.                                                |
| Create Date Folder             | Google Drive (Create Folder) | Creates date folder if missing                    | Check if Store by Date is Enabled | Set Date Folder ID                    | ## Folder Search & Creation: Creates date folder if needed.                                                     |
| Set Date Folder ID             | Function (Code)    | Consolidates date folder ID                       | Check Existing Date Folder, Create Date Folder | Search File Type Folder, Merge Final Parent Folder Data | ## Folder Search & Creation: Sets date folder ID for downstream use.                                            |
| Search File Type Folder        | Google Drive (Search) | Searches for file type folder                      | Set Date Folder ID               | Check Existing File Type Folder        | ## Folder Search & Creation: Searches for file type folder.                                                     |
| Check Existing File Type Folder | IF                | Checks if file type folder exists                  | Search File Type Folder          | Set File Type Folder ID, Check if Store by File Type is Enabled | ## Folder Search & Creation: Checks existence of file type folder.                                              |
| Check if Store by File Type is Enabled | IF          | Checks if storing by file type is enabled          | Check Existing File Type Folder  | Create File Type Folder, Set File Type Folder ID | ## Folder Search & Creation: Conditional on Store by File Type flag.                                            |
| Create File Type Folder        | Google Drive (Create Folder) | Creates file type folder if missing                 | Check if Store by File Type is Enabled | Set File Type Folder ID              | ## Folder Search & Creation: Creates file type folder if needed.                                                |
| Set File Type Folder ID        | Function (Code)    | Consolidates file type folder ID                   | Check Existing File Type Folder, Create File Type Folder | Merge Final Parent Folder Data       | ## Folder Search & Creation: Sets file type folder ID for downstream use.                                       |
| Merge Final Parent Folder Data | Merge              | Merges date and file type folder ID data          | Set Date Folder ID, Set File Type Folder ID | Configure Final Parent Folder ID     | ## Folder Search & Creation: Merges folder IDs for final decision.                                              |
| Configure Final Parent Folder ID | Function (Code)  | Determines final upload folder ID based on config | Merge Final Parent Folder Data   | Get File Binary Content               | ## Folder Search & Creation: Determines final folder ID for upload.                                             |
| Get File Binary Content        | HTTP Request       | Downloads file binary content from LINE API       | Configure Final Parent Folder ID | Validate File Type                    | ## File Retrieval & Validation: Retrieves file binary data.                                                     |
| Validate File Type             | Function (Code)    | Validates file type against allowed types          | Get File Binary Content          | Upload File to Google Drive           | ## File Retrieval & Validation: Validates file type; throws error if disallowed.                                |
| Upload File to Google Drive    | Google Drive (Upload) | Uploads validated file to Google Drive             | Validate File Type               | Log File Details to Google Sheet      | ## Upload, Log, & Reply: Uploads file to Google Drive.                                                          |
| Log File Details to Google Sheet | Google Sheets (Append) | Logs file metadata into Google Sheet                 | Upload File to Google Drive      | Check Reply Enabled Flag              | ## Upload, Log, & Reply: Logs file details in Google Sheets.                                                    |
| Check Reply Enabled Flag       | IF                 | Checks if reply messages are enabled                | Log File Details to Google Sheet | Send LINE Reply Message               | ## Upload, Log, & Reply: Checks reply enabled flag.                                                             |
| Send LINE Reply Message        | HTTP Request       | Sends reply message to LINE user                     | Check Reply Enabled Flag         | -                                   | ## Upload, Log, & Reply: Sends reply message with file URL or error.                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node:**  
   - Name: "LINE Webhook Listener"  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: "line-webhook"  
   - Purpose: Receive incoming POST requests from LINE Messaging API.  

2. **Create Google Sheets Node for Config:**  
   - Name: "Get Config"  
   - Type: Google Sheets  
   - Operation: Read  
   - Sheet ID: Use your Google Sheet ID containing config data.  
   - Range: "config!A1:H2"  
   - Credentials: Google Sheets OAuth2 credentials configured.  
   - Purpose: Retrieve configuration parameters such as Parent Folder ID, allowed file types, flags, and tokens.  

3. **Create Merge Node:**  
   - Name: "Merge Event and Config Data"  
   - Type: Merge  
   - Mode: mergeByIndex  
   - Inputs: Connect from "LINE Webhook Listener" and "Get Config".  
   - Purpose: Combine event and config data into a single item.  

4. **Create Function Node to Process Data:**  
   - Name: "Process Event and Config Data"  
   - Type: Function  
   - Code: Extract config from second input or fallback; throw error if missing.  
   - Input: Output of Merge node.  
   - Output: JSON with separated event and config objects.  

5. **Create Function Node to Determine Folder Info:**  
   - Name: "Determine Folder Info"  
   - Type: Function  
   - Code:  
     - Extract base folder ID from config.  
     - If "Store by Date" enabled, set date folder name (use config date or current date in YYYYMMDD format).  
     - If "Store by File Type" enabled, set file type folder name from event message type (lowercase).  
   - Input: Output of Process Event and Config Data.  

6. **Create Google Drive Search Node for Date Folder:**  
   - Name: "Search Date Folder"  
   - Type: Google Drive (Search)  
   - Query: Date folder name from previous node.  
   - Folder ID: Base folder ID from config.  

7. **Create IF Node to Check Date Folder Exists:**  
   - Name: "Check Existing Date Folder"  
   - Condition: Check if `id` field exists in search result.  

8. **Create IF Node to Check if Store by Date Enabled:**  
   - Name: "Check if Store by Date is Enabled"  
   - Condition: Check if config "Store by Date" is true.  

9. **Create Google Drive Create Folder Node:**  
   - Name: "Create Date Folder"  
   - Type: Google Drive (Create Folder)  
   - Folder Name: Date folder name from "Determine Folder Info".  
   - Parent Folder ID: Base folder ID from config.  

10. **Create Function Node to Set Date Folder ID:**  
    - Name: "Set Date Folder ID"  
    - Type: Function  
    - Code: Consolidate folder ID from existing or newly created folder; set as `targetParentId`.  

11. **Create Google Drive Search Node for File Type Folder:**  
    - Name: "Search File Type Folder"  
    - Type: Google Drive (Search)  
    - Query: File type folder name from "Determine Folder Info".  
    - Folder ID: Use date folder ID if "Store by Date" enabled; else base folder ID.  

12. **Create IF Node to Check File Type Folder Exists:**  
    - Name: "Check Existing File Type Folder"  
    - Condition: Check if `id` field exists in search result.  

13. **Create IF Node to Check if Store by File Type Enabled:**  
    - Name: "Check if Store by File Type is Enabled"  
    - Condition: Check if config "Store by File Type" is true.  

14. **Create Google Drive Create Folder Node:**  
    - Name: "Create File Type Folder"  
    - Type: Google Drive (Create Folder)  
    - Folder Name: File type folder name from "Determine Folder Info".  
    - Parent Folder ID: Date folder ID if "Store by Date" enabled; else base folder ID.  

15. **Create Function Node to Set File Type Folder ID:**  
    - Name: "Set File Type Folder ID"  
    - Type: Function  
    - Code: Consolidate folder ID from existing or newly created folder; set as `targetParentId`.  

16. **Create Merge Node to Combine Folder IDs:**  
    - Name: "Merge Final Parent Folder Data"  
    - Type: Merge  
    - Inputs: From "Set Date Folder ID" and "Set File Type Folder ID".  

17. **Create Function Node to Configure Final Parent Folder ID:**  
    - Name: "Configure Final Parent Folder ID"  
    - Type: Function  
    - Code:  
      - Based on config flags, select final folder ID:  
        - Both true → file type folder ID  
        - Store by Date true only → date folder ID  
        - Store by File Type true only → file type folder ID  
        - Both false → base folder ID from config  
      - Attach `finalParentId` to output.  

18. **Create HTTP Request Node to Get File Binary Content:**  
    - Name: "Get File Binary Content"  
    - Type: HTTP Request  
    - URL: `https://api-data.line.me/v2/bot/message/{{messageId}}/content` (messageId from webhook event)  
    - Authentication: HTTP Header Auth with LINE Channel Access Token from config.  
    - Purpose: Download file binary data.  

19. **Create Function Node to Validate File Type:**  
    - Name: "Validate File Type"  
    - Type: Function  
    - Code:  
      - Extract allowed file types from config (split by "|").  
      - Extract file type from webhook event message.  
      - Throw error if file type not in allowed list.  

20. **Create Google Drive Upload Node:**  
    - Name: "Upload File to Google Drive"  
    - Type: Google Drive (Upload)  
    - File Name: Use event timestamp + file extension.  
    - Folder ID: `finalParentId` from previous node.  

21. **Create Google Sheets Append Node:**  
    - Name: "Log File Details to Google Sheet"  
    - Type: Google Sheets (Append)  
    - Sheet: "fileList" in config Google Sheet.  
    - Columns: File Name, Date Uploaded, Google Drive URL, File Type.  

22. **Create IF Node to Check Reply Enabled Flag:**  
    - Name: "Check Reply Enabled Flag"  
    - Condition: Check if config "Reply Enabled" is true.  

23. **Create HTTP Request Node to Send LINE Reply Message:**  
    - Name: "Send LINE Reply Message"  
    - Type: HTTP Request  
    - URL: `https://api.line.me/v2/bot/message/reply`  
    - Method: POST  
    - Authentication: HTTP Header Auth with LINE Channel Access Token.  
    - Body: JSON with replyToken from webhook and message text containing either file URL or error message.  

24. **Connect Nodes According to Workflow Logic:**  
    - Follow the connections as described in the workflow overview and JSON connections.  
    - Ensure error handling is configured to continue or capture errors as needed.  

25. **Configure Credentials:**  
    - Google Sheets OAuth2 credentials with access to the config and fileList sheets.  
    - Google Drive OAuth2 credentials with permissions to create folders and upload files.  
    - HTTP Header Auth credentials for LINE Messaging API with valid Channel Access Token.  

26. **Test Workflow:**  
    - Send a file message via LINE to the configured webhook URL.  
    - Verify file is uploaded to Google Drive in correct folder.  
    - Confirm file metadata is logged in Google Sheets.  
    - Check that reply message is sent back to LINE user with correct info.  

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                                  |
|-----------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Google Sheets template for config and fileList sheets with required columns and formatting.               | [Google Sheet Template](https://docs.google.com/spreadsheets/d/1iO4ZHU7s0fe1Jn8jcScNDce7rFXQlkRBqsO8IFHbcSc/edit?usp=sharing) |
| LINE Developer Console webhook must be configured to point to the n8n webhook URL for this workflow.      | LINE Messaging API documentation and developer console.                                                        |
| Google Drive and Google Sheets OAuth2 credentials must be authorized with appropriate scopes.             | Google API Console credentials setup.                                                                            |
| The workflow uses dynamic expressions extensively; ensure all referenced nodes and fields exist and are correctly named. |                                                                                                                 |
| Error handling is configured to continue on upload and reply nodes to avoid workflow interruption.        |                                                                                                                 |

---

This documentation provides a detailed, structured reference for understanding, reproducing, and modifying the "Automatically Save & Organize LINE Message Files in Google Drive with Sheets Logging" workflow. It covers all nodes, their roles, configurations, and interconnections, enabling advanced users and automation agents to work effectively with this workflow.