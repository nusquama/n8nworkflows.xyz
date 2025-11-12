Generate & Upload Images with Image-to-Image GPT, Google Sheets & Drive

https://n8nworkflows.xyz/workflows/generate---upload-images-with-image-to-image-gpt--google-sheets---drive-6215


# Generate & Upload Images with Image-to-Image GPT, Google Sheets & Drive

### 1. Workflow Overview

This workflow automates the generation and storage of AI-created images using prompts listed in a Google Sheet. It processes each prompt, generates an image via an AI image-to-image API, uploads the generated image to Google Drive, and updates the Google Sheet with the image's link or logs any errors. The workflow is designed for scenarios where users maintain a dynamic list of image prompts in Google Sheets and want seamless, automated image generation and storage without manual intervention.

**Logical Blocks:**

- **1.1 Input Reception:** Manual trigger and reading prompts from Google Sheets.
- **1.2 Batch Processing:** Looping over each prompt row to handle images one-by-one.
- **1.3 Validation:** Filtering valid prompts that require image generation.
- **1.4 AI Image Generation:** Calling the external AI image generation API.
- **1.5 Error Handling:** Catching and processing errors from the API or subsequent steps.
- **1.6 Image Upload:** Converting and uploading the generated image to Google Drive.
- **1.7 Sheet Update & Logging:** Updating the original Google Sheet with image data or logging errors.
- **1.8 Flow Control:** Managing timing and sequencing with wait steps.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Initiates workflow execution manually and reads the list of prompts from a Google Sheet.
- **Nodes Involved:** When clicking ‘Execute workflow’, Google Sheets2
- **Node Details:**
  - *When clicking ‘Execute workflow’*  
    - Type: Manual trigger  
    - Role: Starts workflow on user command  
    - Inputs: None  
    - Outputs: Triggers Google Sheets2  
    - Edge cases: User must manually trigger; no automatic scheduling.
  - *Google Sheets2*  
    - Type: Google Sheets node (read operation)  
    - Role: Reads all rows from specified sheet ("Sheet1")  
    - Configuration: Uses service account authentication, reads sheet by gid=0  
    - Inputs: Trigger from manual node  
    - Outputs: Passes rows to Loop Over Items  
    - Edge cases: Authentication failure, empty sheet, invalid document ID.

#### 1.2 Batch Processing

- **Overview:** Processes each spreadsheet row individually to handle image generation sequentially.
- **Nodes Involved:** Loop Over Items
- **Node Details:**
  - *Loop Over Items*  
    - Type: SplitInBatches  
    - Role: Iterates over each row from Google Sheets2 one at a time  
    - Configuration: Default options, no batch reset  
    - Inputs: Rows data from Google Sheets2  
    - Outputs: Each row passed separately for filtering  
    - Edge cases: Large dataset may slow workflow; no batch reset means no reprocessing of failed batches unless restarted.

#### 1.3 Validation

- **Overview:** Filters rows to continue processing only if a prompt exists and no image has been generated yet (drive path is empty).
- **Nodes Involved:** If2
- **Node Details:**
  - *If2*  
    - Type: If node  
    - Role: Checks conditions on each item:  
      1. Prompt column is not empty  
      2. drive path column is empty (no image generated yet)  
    - Inputs: Single row from Loop Over Items  
    - Outputs:  
      - True branch: passes valid rows to HTTP Request1  
      - False branch: loops back to Loop Over Items to continue processing next row  
    - Edge cases: Empty or missing fields, strict validation may skip rows unintentionally if column names mismatch.

#### 1.4 AI Image Generation

- **Overview:** Sends the prompt and reference image URL to an AI image generation API to create an image.
- **Nodes Involved:** HTTP Request1
- **Node Details:**
  - *HTTP Request1*  
    - Type: HTTP Request  
    - Role: Calls external AI image-to-image API (hosted on RapidAPI)  
    - Configuration:  
      - POST method with multipart/form-data  
      - Body parameters: prompt text and image URL from current row  
      - Headers: x-rapidapi-host and x-rapidapi-key (user must provide valid API key)  
      - On error: continues workflow without breaking  
    - Inputs: Valid prompt row from If2  
    - Outputs:  
      - Success: base64 image data to Code1  
      - Error: passes error to If1 for logging  
    - Edge cases: API timeout, invalid API key, rate limits, malformed responses.

#### 1.5 Error Handling

- **Overview:** Detects if the API returned an error and routes error data for logging.
- **Nodes Involved:** If1, Google Sheets4
- **Node Details:**
  - *If1*  
    - Type: If node  
    - Role: Checks if the HTTP Request returned an error field  
    - Inputs: Response from HTTP Request1  
    - Outputs:  
      - True branch: sends error info to Google Sheets4  
      - False branch: continues normal flow to Code1  
    - Edge cases: False negatives if error format changes.
  - *Google Sheets4*  
    - Type: Google Sheets node (append or update)  
    - Role: Logs errors into the Google Sheet (same or separate tab)  
    - Configuration: Matches rows by Prompt, appends error text into Base64 column  
    - Inputs: Error data from If1  
    - Outputs: Returns to Wait node  
    - Edge cases: Sheet write failure, authentication issues.

#### 1.6 Image Upload

- **Overview:** Decodes the base64 image string, converts it to binary, and uploads the image file to Google Drive.
- **Nodes Involved:** Code1, Google Drive1
- **Node Details:**
  - *Code1*  
    - Type: Code (JavaScript)  
    - Role: Extracts and cleans base64 image string, converts it to binary file format  
    - Key expressions: Splits base64 string on comma to strip metadata prefix if present  
    - Outputs: Binary image data with filename (output.jpg) and mimeType (image/jpeg)  
    - Inputs: Successful API response with base64 image  
    - Outputs: Passes binary data to Google Drive1  
    - Edge cases: Invalid base64 strings, incorrect mimeType assumptions.
  - *Google Drive1*  
    - Type: Google Drive node (upload)  
    - Role: Uploads binary image file to specified Drive folder  
    - Configuration: Uses service account credentials, folderId and driveId provided via URL mode  
    - Inputs: Binary image data from Code1  
    - Outputs: File upload metadata (including file ID) passed to Wait node  
    - Edge cases: Permission denied, quota exceeded, invalid folder ID.

#### 1.7 Sheet Update & Logging

- **Overview:** Updates the original Google Sheet row with the base64 data or image URL and logs generation timestamps; also appends logs for success or failure.
- **Nodes Involved:** Google Sheets1, Google Sheets4 (for errors), Wait
- **Node Details:**
  - *Google Sheets1*  
    - Type: Google Sheets (append)  
    - Role: Adds a new row or updates with prompt, base64 image, and timestamp  
    - Configuration: Uses service account, sheet by gid=0  
    - Inputs: Binary data from Google Drive1 converted back to base64 in HTTP Request response (via references)  
    - Outputs: Passes to Wait for pacing  
    - Edge cases: Sheet row duplication, network errors.
  - *Google Sheets4* (also used for error logging)  
    - Appends or updates error information as needed.
  - *Wait*  
    - Type: Wait node  
    - Role: Delays workflow by 10 seconds before processing next item  
    - Inputs: From Google Sheets1 or Google Sheets4  
    - Outputs: Loops back to Loop Over Items for next batch  
    - Edge cases: Wait duration too short or long may affect API rate limits or workflow responsiveness.

#### 1.8 Flow Control

- **Overview:** Uses wait steps to pace the processing and loops the batch processing until all items are handled.
- **Nodes Involved:** Wait, Loop Over Items
- **Node Details:**
  - *Wait*  
    - See above.
  - *Loop Over Items*  
    - Receives control to continue or finish based on batch status.

---

### 3. Summary Table

| Node Name                     | Node Type                  | Functional Role                          | Input Node(s)             | Output Node(s)                  | Sticky Note                                                                                                 |
|-------------------------------|----------------------------|----------------------------------------|---------------------------|---------------------------------|-------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger             | Starts workflow manually                | None                      | Google Sheets2                  | ### 1. **Manual Trigger** Manually starts the workflow for testing or controlled execution.                 |
| Google Sheets2                | Google Sheets              | Reads prompts from Google Sheet         | When clicking ‘Execute workflow’ | Loop Over Items                 | ### 2. **Google Sheets2 – Fetch Prompts** - Connects to sheet, reads prompts.                              |
| Loop Over Items               | Split In Batches           | Processes rows one-by-one                | Google Sheets2             | If2 (true branch), Loop Over Items (false branch) | ### 3. **Loop Over Items** - Loops through each row sequentially.                                          |
| If2                          | If                         | Filters valid prompts (Prompt exists & no drive path) | Loop Over Items            | HTTP Request1 (true), Loop Over Items (false) | ### 4. **If2 – Filter Valid Rows** - Checks prompt presence and unprocessed rows.                           |
| HTTP Request1                | HTTP Request               | Calls AI image generation API            | If2 (true)                 | Code1 (success), If1 (error)    | ### 5. **HTTP Request – Call Image API** - Sends prompt and image URL to AI service.                        |
| If1                          | If                         | Detects API error response               | HTTP Request1 (error)      | Google Sheets4 (true), Code1 (false) | ### 6. **Try Catch – Error Handling** - Handles errors from API call or upload.                             |
| Google Sheets4               | Google Sheets              | Logs errors or updates rows               | If1 (true)                 | Wait                           | ### 10. **Google Sheets3 – Log Status** - Logs errors or status info.                                       |
| Code1                        | Code                       | Converts base64 image string to binary   | HTTP Request1 (success)    | Google Drive1                  | ### 7. **Google Drive – Upload Image** - Converts and prepares image for upload.                            |
| Google Drive1                | Google Drive               | Uploads image to Drive                    | Code1                      | Wait                          | ### 7. **Google Drive – Upload Image** - Uploads image file to Drive.                                       |
| Google Sheets1               | Google Sheets              | Updates sheet with base64 image and timestamp | Code1/Google Drive1        | Wait                          | ### 9. **Google Sheets1 – Update Sheet** - Updates the sheet with image data.                              |
| Wait                        | Wait                       | Delays next iteration by 10 seconds      | Google Sheets1, Google Sheets4, Google Drive1 | Loop Over Items                 | This node waits for 10 seconds before next processing iteration.                                           |
| Sticky Note (various)        | Sticky Note                | Documentation and explanations            | None                      | None                          | Multiple sticky notes explain each node/block in detail.                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Start workflow execution manually.

2. **Add Google Sheets Node (Google Sheets2) to Read Prompts**  
   - Type: Google Sheets (Read operation)  
   - Parameters:  
     - Document ID: URL or ID of Google Sheet containing prompts  
     - Sheet Name: Use gid=0 or "Sheet1"  
     - Authentication: Service Account (set up Google API credentials)  
   - Connect output of Manual Trigger to this node.

3. **Add SplitInBatches Node (Loop Over Items)**  
   - Type: SplitInBatches  
   - No special parameters needed, default batch size of 1 (process one row at a time)  
   - Connect output of Google Sheets2 to this node.

4. **Add If Node (If2) to Filter Rows**  
   - Type: If  
   - Conditions:  
     - Check if "Prompt" field is not empty  
     - Check if "drive path" field is empty  
   - Connect Loop Over Items output to this node.  
   - True branch: proceeds to image generation  
   - False branch: connect back to Loop Over Items to process next row.

5. **Add HTTP Request Node (HTTP Request1) to Call AI API**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://image-to-image-gpt.p.rapidapi.com/productgpt/index.php`  
   - Content Type: multipart/form-data  
   - Body Parameters:  
     - "prompt": `={{ $json.Prompt }}`  
     - "image": `={{ $json['Image url'] }}`  
   - Headers:  
     - `x-rapidapi-host`: `image-to-image-gpt.p.rapidapi.com`  
     - `x-rapidapi-key`: Your RapidAPI key (must be set)  
   - Error Handling: On error, continue workflow  
   - Connect If2 true output to this node.

6. **Add If Node (If1) to Detect API Errors**  
   - Type: If  
   - Condition: Check if response contains `.error` field (exists)  
   - Connect HTTP Request1 error output to this node.  
   - True branch: sends error data to logging node  
   - False branch: proceeds to image processing.

7. **Add Google Sheets Node (Google Sheets4) for Error Logging**  
   - Type: Google Sheets (Append or Update)  
   - Sheet: Same or separate tab for logs  
   - Columns: Map "Prompt" and "Base64" (for error text)  
   - Authentication: Service Account  
   - Connect If1 true output here.

8. **Add Code Node (Code1) to Convert Base64 to Binary**  
   - Type: Code (JavaScript)  
   - Code:  
     ```javascript
     const base64String = $input.first().json.image_base64;
     const cleanedBase64 = base64String.includes(",") ? base64String.split(",")[1] : base64String;
     return [{
       binary: {
         data: {
           data: Buffer.from(cleanedBase64, 'base64'),
           mimeType: 'image/jpeg',
           fileName: 'output.jpg'
         }
       }
     }];
     ```  
   - Connect If1 false output to this node.

9. **Add Google Drive Node (Google Drive1) to Upload Image**  
   - Type: Google Drive (Upload)  
   - Parameters:  
     - Name: `={{ $binary.data.fileName }}`  
     - Folder ID and Drive ID set via URL mode (must be configured)  
   - Authentication: Service Account  
   - Connect Code1 output to this node.

10. **Add Google Sheets Node (Google Sheets1) to Update Sheet**  
    - Type: Google Sheets (Append)  
    - Map columns:  
      - Prompt: `={{ $('If2').item.json.Prompt }}`  
      - Base64: `={{ $('HTTP Request1').item.json.image_base64 }}`  
      - Generated Date: `={{ $now }}`  
    - Authentication: Service Account  
    - Connect Google Drive1 output to this node.

11. **Add Wait Node to Delay Next Iteration**  
    - Type: Wait  
    - Parameters: Wait for 10 seconds  
    - Connect outputs of Google Sheets1 and Google Sheets4 here.

12. **Connect Wait Node back to Loop Over Items Node**  
    - This creates the loop for processing the next batch item.

13. **Set up Credentials**  
    - Google API Service Account with permissions for Google Sheets and Drive  
    - RapidAPI key with access to image-to-image-gpt API

14. **Configure Google Sheet**  
    - Sheet with columns at least: Prompt, drive path, Image url (optional), Base64, Generated Date  
    - Ensure sheet sharing allows service account access.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                         | Context or Link                                                                                                   |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| This workflow centralizes prompt management in Google Sheets, enabling automated AI image generation with minimal manual input.                                    | Sticky Note overview (node d268cfe7-3ca2-4d96-9965-263fb1d8146a)                                                |
| Image generation uses the RapidAPI-hosted "image-to-image-gpt" endpoint; users must supply their own API key for the service.                                     | HTTP Request1 node parameters                                                                                     |
| The workflow includes built-in error handling to log API failures without stopping the batch process, ensuring robustness for large prompt sets.                  | Nodes If1 and Google Sheets4                                                                                      |
| The 10-second wait between image uploads helps mitigate API rate limits and Google Drive upload throttling.                                                        | Wait node sticky note (30397d7d-206e-4a64-b262-05858e226f1a)                                                     |
| Service account credentials must be correctly configured with Google Drive and Sheets API enabled for seamless operation.                                         | Google Sheets and Google Drive nodes credentials                                                                  |
| The base64 image data is converted to a binary file with a fixed JPEG mime type; if the API returns other formats, this code may need adjustment.                 | Code1 node JavaScript                                                                                              |
| The workflow demonstrates a scalable pattern for integrating AI generation services with cloud storage and spreadsheet tracking, which can be adapted to other APIs. | General architecture note                                                                                          |