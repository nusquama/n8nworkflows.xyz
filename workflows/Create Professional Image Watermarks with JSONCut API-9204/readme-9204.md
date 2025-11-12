Create Professional Image Watermarks with JSONCut API

https://n8nworkflows.xyz/workflows/create-professional-image-watermarks-with-jsoncut-api-9204


# Create Professional Image Watermarks with JSONCut API

### 1. Workflow Overview

This workflow automates the creation of professional image watermarks by leveraging the JsonCut API. It is designed for use cases where users want to upload a main image and a watermark image via a form, have these images processed by the JsonCut service to composite a watermarked image, and then download the resulting image automatically.

The workflow is organized into the following logical blocks:

- **1.1 Input Reception:** Handles user input via a form trigger where users upload the main image and watermark image files.
- **1.2 File Upload to JsonCut API:** Uploads the received files asynchronously to the JsonCut file storage service.
- **1.3 Job Creation and Processing:** Creates an image composition job on JsonCut with the uploaded files and waits for the job to complete.
- **1.4 Job Status Polling and Error Handling:** Periodically checks the job status, branching logic for success or failure scenarios.
- **1.5 Download and Output:** Downloads the final watermarked image file once the job completes successfully.
- **1.6 Failure Management:** Stops the workflow with an error message if the job fails or is cancelled.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block initiates the workflow by capturing user inputs through a web form. Users must upload two files: the main image and the watermark image. This input triggers the entire process.

- **Nodes Involved:**  
  - Form Trigger

- **Node Details:**  
  - **Form Trigger**  
    - Type: Form Trigger (Webhook-based input trigger)  
    - Configuration:  
      - Path: Unique webhook path set for form submissions.  
      - Form Title: "Image Watermark Generator"  
      - Form Description: "Upload your main image and watermark to create a watermarked image"  
      - Fields: Two required file upload fields ("Main Image" and "Watermark Image").  
    - Input: External form submission with files  
    - Output: JSON containing the uploaded files as binary data  
    - Edge Cases: Missing file uploads will block submission due to required fields. Improper file types might cause upload failures downstream.  
    - Version: typeVersion 2

---

#### 2.2 File Upload to JsonCut API

- **Overview:**  
  This block uploads the user-uploaded main and watermark images to the JsonCut API file storage. It uses two parallel HTTP requests, one for each file, and then merges their responses for use in job creation.

- **Nodes Involved:**  
  - Upload Main Image  
  - Upload Watermark  
  - Merge Uploads  
  - Aggregate

- **Node Details:**  
  - **Upload Main Image**  
    - Type: HTTP Request  
    - Role: Upload main image file to JsonCut API endpoint `/files/upload`  
    - Configuration:  
      - Method: POST  
      - Content-Type: multipart/form-data  
      - Body: Uses binary data from form field "Main Image"  
      - Headers: Accept application/json  
      - Authentication: HTTP Header with JsonCut API Key credential  
    - Input: Binary file data from Form Trigger  
    - Output: JSON response with upload metadata (including storage URL)  
    - Edge Cases: API key invalid, file upload failures, network issues, timeout  
    - Version: typeVersion 4.1

  - **Upload Watermark**  
    - Same configuration as Upload Main Image, but uploads the watermark file from "Watermark Image" field.  
    - Input: Binary watermark file from Form Trigger  
    - Output: JSON response similar to main image upload  
    - Edge Cases: Same as above

  - **Merge Uploads**  
    - Type: Merge  
    - Role: Joins the outputs of the two upload HTTP requests into a single data stream  
    - Configuration: Default settings, merges two streams from Upload Main Image and Upload Watermark  
    - Input: From both upload nodes  
    - Output: Combined JSON array containing both upload responses  
    - Edge Cases: Mismatched or missing inputs if one upload fails or is delayed

  - **Aggregate**  
    - Type: Aggregate  
    - Role: Aggregates the uploaded files’ storage URLs into a single JSON object with a list for job creation  
    - Configuration: Aggregates field `data.storageUrl` from the merged output  
    - Input: From Merge Uploads  
    - Output: JSON with aggregated storage URLs array accessible as `storageUrl`  
    - Edge Cases: Missing or malformed URLs cause failure in job creation

---

#### 2.3 Job Creation and Processing

- **Overview:**  
  This block creates a new job on the JsonCut API to composite the watermarked image using the uploaded files. It then initiates a wait period before checking job status.

- **Nodes Involved:**  
  - Create JsonCut Job  
  - Wait

- **Node Details:**  
  - **Create JsonCut Job**  
    - Type: HTTP Request  
    - Role: Submits a job to JsonCut to compose the watermark on the main image  
    - Configuration:  
      - Method: POST  
      - URL: `/jobs` endpoint  
      - Body (JSON):  
        - Specifies image dimensions 800x600  
        - Layers array:  
          - Base layer uses first uploaded image URL (main image) with fit "cover"  
          - Watermark layer uses second uploaded image URL positioned bottom-right with opacity 0.5 and fit "contain"  
      - Headers: Content-Type and Accept as application/json  
      - Authentication: HTTP Header with JsonCut API Key  
    - Input: Aggregated storage URLs from previous block  
    - Output: JSON including job ID and initial job status  
    - Edge Cases: Malformed JSON, missing URLs, API key errors, rate limits

  - **Wait**  
    - Type: Wait  
    - Role: Pauses workflow for 3 seconds to allow job processing on JsonCut side before status check  
    - Configuration: Wait for 3 seconds  
    - Input: Triggered after job creation  
    - Output: Continues to next node after delay  
    - Edge Cases: Time too short might cause premature status check; too long delays workflow

---

#### 2.4 Job Status Polling and Error Handling

- **Overview:**  
  This block polls the JsonCut API to check the job processing status. It branches workflow logic based on whether the job is completed, failed, or cancelled, handling errors accordingly.

- **Nodes Involved:**  
  - Check JsonCut job Status  
  - If Success  
  - If Error  
  - Error Stop

- **Node Details:**  
  - **Check JsonCut job Status**  
    - Type: HTTP Request  
    - Role: Retrieves job status using job ID  
    - Configuration:  
      - Method: GET  
      - URL: `/jobs/{{jobId}}` (using expression to inject job ID)  
      - Headers: Accept application/json  
      - Authentication: HTTP Header with JsonCut API Key  
    - Input: From Wait node  
    - Output: Job status JSON including `data.status` field  
    - Edge Cases: Network errors, invalid job ID, API rate limits

  - **If Success**  
    - Type: If  
    - Role: Checks if job status equals "COMPLETED"  
    - Configuration: Condition: `$json.data.status == "COMPLETED"`  
    - Input: From job status check  
    - Output:  
      - True branch proceeds to download the image  
      - False branch proceeds to If Error node  
    - Edge Cases: Status field missing or different casing

  - **If Error**  
    - Type: If  
    - Role: Checks if job status is either "FAILED" or "CANCELLED"  
    - Configuration: Conditions: `$json.data.status == "FAILED"` or `$json.data.status == "CANCELLED"`  
    - Input: From If Success node (false branch)  
    - Output:  
      - True branch triggers error stop  
      - False branch loops back to Wait node for another polling cycle  
    - Edge Cases: Infinite looping if job never completes or fails; consider adding max retries (not present)

  - **Error Stop**  
    - Type: Stop and Error  
    - Role: Terminates workflow with error message if job failed or cancelled  
    - Configuration: Error message "Failed to generate image"  
    - Input: From If Error node  
    - Output: Workflow terminates  
    - Edge Cases: None

---

#### 2.5 Download and Output

- **Overview:**  
  This block downloads the final watermarked image file once the job is successfully completed.

- **Nodes Involved:**  
  - Download Image

- **Node Details:**  
  - **Download Image**  
    - Type: HTTP Request  
    - Role: Downloads the composited image file from JsonCut API  
    - Configuration:  
      - Method: GET  
      - URL: `/files/{{outputFileId}}/download` (uses expression to get output file ID from job status response)  
      - Headers: Accept application/octet-stream  
      - Response format: File (binary)  
      - Authentication: HTTP Header with JsonCut API Key  
    - Input: From If Success node (true branch)  
    - Output: Binary file data containing the watermarked image  
    - Edge Cases: File missing, download failures, invalid file ID

---

### 3. Summary Table

| Node Name              | Node Type           | Functional Role                            | Input Node(s)                 | Output Node(s)                 | Sticky Note                                      |
|------------------------|---------------------|------------------------------------------|------------------------------|-------------------------------|-------------------------------------------------|
| Form Trigger           | Form Trigger        | Receives user image and watermark files | -                            | Upload Main Image, Upload Watermark |                                                 |
| Upload Main Image      | HTTP Request        | Uploads main image to JsonCut API        | Form Trigger                 | Merge Uploads                 | ### Upload Files to JsonCut API                  |
| Upload Watermark       | HTTP Request        | Uploads watermark image to JsonCut API   | Form Trigger                 | Merge Uploads                 | ### Upload Files to JsonCut API                  |
| Merge Uploads          | Merge               | Merges upload responses                   | Upload Main Image, Upload Watermark | Aggregate                   | ### Upload Files to JsonCut API                  |
| Aggregate              | Aggregate           | Aggregates storage URLs for job creation | Merge Uploads                | Create JsonCut Job            | ### Upload Files to JsonCut API                  |
| Create JsonCut Job     | HTTP Request        | Creates image composition job             | Aggregate                   | Wait                        | ### Create Job with Jsoncut API and wait for the result |
| Wait                   | Wait                | Pauses before status check                 | Create JsonCut Job           | Check JsonCut job Status      | ### Create Job with Jsoncut API and wait for the result |
| Check JsonCut job Status | HTTP Request      | Checks job status                          | Wait                        | If Success                   | ### Create Job with Jsoncut API and wait for the result |
| If Success             | If                  | Branches on job completion status          | Check JsonCut job Status     | Download Image, If Error      | ### Create Job with Jsoncut API and wait for the result |
| If Error               | If                  | Branches on job failure/cancellation       | If Success                  | Error Stop, Wait              | ### Create Job with Jsoncut API and wait for the result |
| Error Stop             | Stop and Error      | Stops workflow on failure                   | If Error                    | -                           | ### Create Job with Jsoncut API and wait for the result |
| Download Image         | HTTP Request        | Downloads final watermarked image           | If Success                  | -                           | ### Create Job with Jsoncut API and wait for the result |
| Sticky Note            | Sticky Note         | Annotation for upload section               | -                           | -                           | ### Upload Files to JsonCut API                  |
| Sticky Note1           | Sticky Note         | Annotation for job creation/wait section   | -                           | -                           | ### Create Job with Jsoncut API and wait for the result |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Form Trigger node**  
   - Type: Form Trigger  
   - Set webhook path (unique, e.g., `b355dccf-d9fa-46e0-9c3c-8d3c743aa037`)  
   - Configure form with two required file upload fields: "Main Image" and "Watermark Image"  
   - Set form title: "Image Watermark Generator"  
   - Set form description accordingly  

2. **Create two HTTP Request nodes for file upload**  
   - First node: "Upload Main Image"  
     - Method: POST  
     - URL: `https://api.jsoncut.com/api/v1/files/upload`  
     - Content-Type: multipart/form-data  
     - Body: Send binary data from Form Trigger field "Main Image" as form field "file"  
     - Header: Accept: application/json  
     - Authentication: HTTP Header Auth with JsonCut API Key credential  
   - Second node: "Upload Watermark" (same as above but uses "Watermark Image" field)  

3. **Create a Merge node**  
   - Type: Merge  
   - Merge the outputs of the two upload nodes (use default settings to combine the data)  

4. **Create an Aggregate node**  
   - Type: Aggregate  
   - Aggregate the field `data.storageUrl` from the merged data into a list, accessible as `storageUrl`  

5. **Create an HTTP Request node "Create JsonCut Job"**  
   - Method: POST  
   - URL: `https://api.jsoncut.com/api/v1/jobs`  
   - Body: JSON specifying the job configuration:  
     - Image size: 800x600  
     - Layers:  
       - Base image: `{{ $json.storageUrl[0] }}`, fit "cover"  
       - Watermark: `{{ $json.storageUrl[1] }}`, positioned bottom-right with opacity 0.5, fit "contain"  
   - Headers: Content-Type: application/json, Accept: application/json  
   - Authentication: HTTP Header Auth with JsonCut API Key  

6. **Create a Wait node**  
   - Wait for 3 seconds  

7. **Create an HTTP Request node "Check JsonCut job Status"**  
   - Method: GET  
   - URL: `https://api.jsoncut.com/api/v1/jobs/{{ $json.data.jobId }}` (use expression)  
   - Header: Accept: application/json  
   - Authentication: HTTP Header Auth with JsonCut API Key  

8. **Create an If node "If Success"**  
   - Condition: `$json.data.status == "COMPLETED"`  

9. **Create an If node "If Error"**  
   - Condition: `$json.data.status == "FAILED"` OR `$json.data.status == "CANCELLED"`  

10. **Create a Stop and Error node "Error Stop"**  
    - Set error message: "Failed to generate image"  

11. **Create an HTTP Request node "Download Image"**  
    - Method: GET  
    - URL: `https://api.jsoncut.com/api/v1/files/{{ $json.data.outputFileId }}/download` (expression)  
    - Header: Accept: application/octet-stream  
    - Response format: file (binary)  
    - Authentication: HTTP Header Auth with JsonCut API Key  

12. **Connect nodes in this sequence:**  
    - Form Trigger → Upload Main Image and Upload Watermark (parallel)  
    - Upload Main Image & Upload Watermark → Merge Uploads  
    - Merge Uploads → Aggregate  
    - Aggregate → Create JsonCut Job  
    - Create JsonCut Job → Wait  
    - Wait → Check JsonCut job Status  
    - Check JsonCut job Status → If Success  
    - If Success (true) → Download Image  
    - If Success (false) → If Error  
    - If Error (true) → Error Stop  
    - If Error (false) → Wait (loop back for polling)  

13. **Configure credentials:**  
    - Create and assign the JsonCut API Key credential for all HTTP Request nodes requiring authentication  

14. **Add sticky notes as annotations:**  
    - For upload-related nodes: "### Upload Files to JsonCut API"  
    - For job creation, waiting, and status polling nodes: "### Create Job with Jsoncut API and wait for the result"  

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                                             |
|------------------------------------------------------------------------------|-------------------------------------------------------------|
| Workflow provides asynchronous polling to handle JsonCut job processing time | Important to prevent premature status check and errors      |
| Consider adding maximum retry logic or timeout in polling loop to avoid infinite loops | Enhancement suggestion for robustness                       |
| JsonCut API documentation: https://docs.jsoncut.com/                         | Official API reference for further customization             |
| File uploads use multipart/form-data content type with binary data mapped correctly | Critical for successful file transfer                        |
| Opacity and positioning parameters in job configuration support watermark transparency and placement | Enables professional watermark appearance                    |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.