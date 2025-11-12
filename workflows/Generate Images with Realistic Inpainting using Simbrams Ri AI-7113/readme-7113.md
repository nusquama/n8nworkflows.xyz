Generate Images with Realistic Inpainting using Simbrams Ri AI

https://n8nworkflows.xyz/workflows/generate-images-with-realistic-inpainting-using-simbrams-ri-ai-7113


# Generate Images with Realistic Inpainting using Simbrams Ri AI

### 1. Workflow Overview

This workflow, titled **Simbrams Ri Image Generator**, is designed to generate images with realistic inpainting by leveraging the **simbrams/ri** AI model hosted on Replicate. It is tailored for users who want to submit an initial image and mask to the AI model and retrieve the processed image output after the model completes its asynchronous prediction task.

The workflow is logically divided into the following blocks:

- **1.1 Input Trigger and API Key Setup:** Manual initiation and setting of the Replicate API key needed for authenticated requests.

- **1.2 Prediction Creation:** Sending a POST request to Replicate API to create a prediction job with parameters including image URL, mask URL, and generation settings.

- **1.3 Prediction Monitoring:** Polling the Replicate API repeatedly to check the status of the prediction until it completes successfully.

- **1.4 Result Processing:** Extracting and formatting the final output once the prediction is completed.

- **1.5 Informational Documentation:** A sticky note node provides details on the model and instructions for setup and usage.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Trigger and API Key Setup

- **Overview:**  
This block initiates the workflow manually and sets the Replicate API key as a workflow variable for subsequent authenticated API calls.

- **Nodes Involved:**  
  - On clicking 'execute' (Manual Trigger)  
  - Set API Key (Set Node)

- **Node Details:**

  - **On clicking 'execute'**  
    - Type: Manual Trigger  
    - Role: Entry point of the workflow, allows manual execution  
    - Configuration: No parameters; triggers workflow on user command  
    - Inputs: None  
    - Outputs: Connected to "Set API Key" node  
    - Edge Cases: User forgetting to trigger manually results in no execution  

  - **Set API Key**  
    - Type: Set Node  
    - Role: Stores Replicate API key as a variable for later HTTP requests  
    - Configuration: Assigns string variable `replicate_api_key` with placeholder `"YOUR_REPLICATE_API_KEY"`  
    - Inputs: From Manual Trigger  
    - Outputs: To "Create Prediction"  
    - Edge Cases: Missing or invalid API key will cause authentication failure in later HTTP requests  

#### 2.2 Prediction Creation

- **Overview:**  
Sends a POST request to the Replicate API to create a new prediction job using the simbrams/ri model and configured input parameters including image, mask, seed, steps, and other generation settings.

- **Nodes Involved:**  
  - Create Prediction (HTTP Request)  
  - Extract Prediction ID (Code)

- **Node Details:**

  - **Create Prediction**  
    - Type: HTTP Request  
    - Role: Initiates prediction job on Replicate API  
    - Configuration:  
      - Method: POST  
      - URL: `https://api.replicate.com/v1/predictions`  
      - Headers: Authorization Bearer token from `replicate_api_key`, Content-Type application/json  
      - Body: JSON containing:  
        - `version`: fixed model version string  
        - `input`: object with keys: `image`, `mask`, `seed`, `steps`, `strength`, `blur_mask`, `merge_m_s`  
      - Timeout: 60 seconds  
      - Authentication: HTTP header with Bearer token from Set API Key node  
    - Inputs: From "Set API Key"  
    - Outputs: To "Extract Prediction ID"  
    - Edge Cases:  
      - Network timeouts or slow API responses  
      - Invalid input URLs for image or mask causing API errors  
      - Unauthorized errors due to missing/invalid API key  

  - **Extract Prediction ID**  
    - Type: Code  
    - Role: Parses the prediction ID and initial status from API response for polling  
    - Configuration: Runs JavaScript code once per item to extract:  
      - `predictionId` from response JSON `id` field  
      - `status` from response JSON `status` field  
      - Constructs `predictionUrl` for further GET requests  
    - Inputs: From "Create Prediction"  
    - Outputs: To "Wait" node  
    - Edge Cases:  
      - Missing or malformed response JSON  
      - Prediction ID missing causing downstream failures  

#### 2.3 Prediction Monitoring

- **Overview:**  
Repeatedly polls the Replicate API prediction endpoint at short intervals to check if the prediction has completed successfully.

- **Nodes Involved:**  
  - Wait (Wait Node)  
  - Check Prediction Status (HTTP Request)  
  - Check If Complete (IF Node)

- **Node Details:**

  - **Wait**  
    - Type: Wait  
    - Role: Pauses workflow for 2 seconds between polling attempts  
    - Configuration: Wait 2 seconds  
    - Inputs: From "Extract Prediction ID" and "Check If Complete" (when prediction incomplete)  
    - Outputs: To "Check Prediction Status"  
    - Edge Cases:  
      - Too short wait time may cause API rate limiting  
      - Long wait time slows workflow responsiveness  

  - **Check Prediction Status**  
    - Type: HTTP Request  
    - Role: GET request to prediction URL to fetch current status and result data  
    - Configuration:  
      - URL: dynamic, from incoming JSON `predictionUrl`  
      - Headers: Authorization Bearer token from `replicate_api_key`  
      - Method: GET (default)  
    - Inputs: From "Wait"  
    - Outputs: To "Check If Complete"  
    - Edge Cases:  
      - Network errors or API unavailability  
      - Authorization errors if token expired or invalid  

  - **Check If Complete**  
    - Type: IF  
    - Role: Checks if prediction status is `succeeded`  
    - Configuration: Boolean condition: `$json.status == "succeeded"`  
    - Inputs: From "Check Prediction Status"  
    - Outputs:  
      - True branch: To "Process Result"  
      - False branch: Back to "Wait" to continue polling  
    - Edge Cases:  
      - Prediction may enter failed or error states not handled explicitly here (could cause infinite polling)  

#### 2.4 Result Processing

- **Overview:**  
Processes the final prediction response, extracting relevant metadata and the generated image URL for further use or output.

- **Nodes Involved:**  
  - Process Result (Code)

- **Node Details:**

  - **Process Result**  
    - Type: Code  
    - Role: Extracts and formats key information from completed prediction  
    - Configuration: JavaScript code returning an object with:  
      - `status`  
      - `output` (likely image URL array)  
      - `metrics`  
      - `created_at`  
      - `completed_at`  
      - `model` set as "simbrams/ri"  
      - `image_url` set as `result.output` (for easy access)  
    - Inputs: From "Check If Complete" (true branch)  
    - Outputs: Terminal node (no further nodes connected)  
    - Edge Cases:  
      - Empty or unexpected output data structure  
      - Multiple output URLs requiring further extraction (not handled)  

#### 2.5 Informational Documentation

- **Overview:**  
Provides user-facing details about the workflow, including instructions and model description.

- **Nodes Involved:**  
  - Sticky Note

- **Node Details:**

  - **Sticky Note**  
    - Type: Sticky Note  
    - Role: Documentation block within the workflow editor  
    - Content:  
      - Workflow title and purpose  
      - Setup instructions (adding API key, configuring inputs, running)  
      - Model details (type, provider, required fields)  
    - Inputs/Outputs: None  
    - Edge Cases: None (purely informational)

---

### 3. Summary Table

| Node Name              | Node Type          | Functional Role                             | Input Node(s)          | Output Node(s)           | Sticky Note                                                                                |
|------------------------|--------------------|--------------------------------------------|-----------------------|--------------------------|--------------------------------------------------------------------------------------------|
| On clicking 'execute'   | Manual Trigger     | Initiates workflow manually                 | None                  | Set API Key              |                                                                                            |
| Set API Key            | Set                | Sets Replicate API key for authentication  | On clicking 'execute' | Create Prediction        |                                                                                            |
| Create Prediction      | HTTP Request       | Sends POST request to create prediction    | Set API Key           | Extract Prediction ID    |                                                                                            |
| Extract Prediction ID  | Code               | Extracts prediction ID and status           | Create Prediction     | Wait                     |                                                                                            |
| Wait                   | Wait               | Waits 2 seconds between polling attempts   | Extract Prediction ID, Check If Complete (false branch) | Check Prediction Status |                                                                                            |
| Check Prediction Status | HTTP Request       | Polls prediction status from Replicate API | Wait                  | Check If Complete        |                                                                                            |
| Check If Complete      | IF                 | Checks if prediction completed successfully | Check Prediction Status | Process Result (true), Wait (false) |                                                                                            |
| Process Result         | Code               | Processes final prediction output           | Check If Complete (true) | None                    |                                                                                            |
| Sticky Note            | Sticky Note        | Provides workflow info and usage instructions | None                  | None                     | ## Simbrams Ri Image Generator  This workflow uses the **simbrams/ri** model from Replicate to generate image content.  Setup: 1. Add your Replicate API key 2. Configure the input parameters 3. Run the workflow  Model Details: - **Type**: Image Generation - **Provider**: simbrams - **Required Fields**: image, mask |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**  
   - Name: `On clicking 'execute'`  
   - Purpose: Entry point for manual workflow execution  

2. **Add a Set node**  
   - Name: `Set API Key`  
   - Connect input from `On clicking 'execute'`  
   - Add a string field assignment:  
     - Name: `replicate_api_key`  
     - Value: `"YOUR_REPLICATE_API_KEY"` (replace with your actual API key)  

3. **Add an HTTP Request node**  
   - Name: `Create Prediction`  
   - Connect input from `Set API Key`  
   - Set Method: `POST`  
   - Set URL: `https://api.replicate.com/v1/predictions`  
   - Under Authentication, choose Generic Credential with HTTP Header Auth  
   - Add Header Parameters:  
     - `Authorization` set to expression: `'Bearer ' + $json["replicate_api_key"]` referencing the Set API Key node  
     - `Content-Type`: `application/json`  
   - Body Content Type: JSON  
   - Body:  
     ```json
     {
       "version": "1713fdec48707483a50e950ae63212c59293e6abff8abd1834c911b75b881d70",
       "input": {
         "image": "https://example.com/input.image",
         "mask": "https://example.com/input.image",
         "seed": 1,
         "steps": 20,
         "strength": 0.8,
         "blur_mask": true,
         "merge_m_s": true
       }
     }
     ```  
   - Set timeout to 60000 ms (60 seconds)  

4. **Add a Code node**  
   - Name: `Extract Prediction ID`  
   - Connect input from `Create Prediction`  
   - Use JavaScript code:  
     ```javascript
     const prediction = $input.item.json;
     const predictionId = prediction.id;
     const initialStatus = prediction.status;

     return {
       predictionId: predictionId,
       status: initialStatus,
       predictionUrl: `https://api.replicate.com/v1/predictions/${predictionId}`
     };
     ```  
   - Run mode: Run once for each item  

5. **Add a Wait node**  
   - Name: `Wait`  
   - Connect input from `Extract Prediction ID` (and later from "Check If Complete" false branch)  
   - Configure to wait 2 seconds  

6. **Add an HTTP Request node**  
   - Name: `Check Prediction Status`  
   - Connect input from `Wait`  
   - Method: GET (default)  
   - URL: Use expression `{{$json.predictionUrl}}`  
   - Authentication: Same as "Create Prediction" (Generic Credential HTTP Header Auth)  
   - Header: Authorization Bearer token from Set API Key node as before  

7. **Add an IF node**  
   - Name: `Check If Complete`  
   - Connect input from `Check Prediction Status`  
   - Condition: Boolean  
     - Expression: `$json.status == "succeeded"`  
   - True output: Connect to `Process Result` node  
   - False output: Connect back to `Wait` node for polling  

8. **Add a Code node**  
   - Name: `Process Result`  
   - Connect input from `Check If Complete` (true branch)  
   - JavaScript code:  
     ```javascript
     const result = $input.item.json;

     return {
       status: result.status,
       output: result.output,
       metrics: result.metrics,
       created_at: result.created_at,
       completed_at: result.completed_at,
       model: 'simbrams/ri',
       image_url: result.output
     };
     ```  
   - Run mode: Run once for each item  

9. **(Optional) Add a Sticky Note node**  
   - Name: `Sticky Note`  
   - No connections  
   - Content:  
     ```
     ## Simbrams Ri Image Generator

     This workflow uses the **simbrams/ri** model from Replicate to generate image content.

     ### Setup
     1. Add your Replicate API key
     2. Configure the input parameters
     3. Run the workflow

     ### Model Details
     - **Type**: Image Generation
     - **Provider**: simbrams
     - **Required Fields**: image, mask
     ```

---

### 5. General Notes & Resources

| Note Content                                                                                                                            | Context or Link                                  |
|-----------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| The workflow uses the Replicate API model version `1713fdec48707483a50e950ae63212c59293e6abff8abd1834c911b75b881d70` specifically.      | Important for API compatibility                  |
| Input image and mask URLs need to be publicly accessible URLs; private or local paths will cause errors.                                | Image and mask input requirements                 |
| Polling interval is fixed at 2 seconds to balance responsiveness and API rate limits. Adjust as needed based on API constraints.        | Polling control in "Wait" node                    |
| The workflow does not explicitly handle prediction failure status other than retrying; consider adding error handling for robustness.  | Potential improvement for production use         |
| Replicate API requires a valid API key with permissions to use the simbrams/ri model.                                                    | Authentication prerequisite                       |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly complies with current content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.