Generate Urban Architecture Images with Lemaar Door AI via Replicate

https://n8nworkflows.xyz/workflows/generate-urban-architecture-images-with-lemaar-door-ai-via-replicate-6879


# Generate Urban Architecture Images with Lemaar Door AI via Replicate

### 1. Workflow Overview

This workflow, titled **Creativeathive Lemaar Door Urban AI Generator**, automates the generation of urban architecture images using the AI model **creativeathive/lemaar-door-urban** hosted on the Replicate platform. The workflow is designed to:

- Accept a manual trigger to start the process.
- Use a user-provided Replicate API key for authentication.
- Submit an image generation request with configurable input parameters to the Replicate API.
- Poll the API until the prediction completes.
- Process and output the generated image URLs and associated metadata.

The logical structure is divided into these blocks:

- **1.1 Input Initialization**: Manual trigger and API key setup.
- **1.2 Prediction Creation**: Submitting the generation request to Replicate.
- **1.3 Prediction Polling**: Periodically checking the status of the generation until complete.
- **1.4 Result Processing**: Handling the output once generation is successful.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Initialization

- **Overview:**  
  This block receives the manual execution trigger and sets the required API key to authenticate requests to Replicate.

- **Nodes Involved:**  
  - On clicking 'execute' (Manual Trigger)  
  - Set API Key

- **Node Details:**

  - **On clicking 'execute'**  
    - Type: Manual Trigger  
    - Role: Starts the workflow manually.  
    - Configuration: No special parameters; user triggers execution.  
    - Input: None  
    - Output: Triggers "Set API Key" node.  
    - Edge Cases: None (manual trigger).  

  - **Set API Key**  
    - Type: Set  
    - Role: Stores the Replicate API key as a workflow variable for later use.  
    - Configuration: A string variable named `replicate_api_key` with a placeholder value `"YOUR_REPLICATE_API_KEY"` (must be replaced by the user).  
    - Input: Triggered by manual trigger node.  
    - Output: Passes API key data to "Create Prediction" node.  
    - Edge Cases: Missing or invalid API key will cause authentication failures downstream.

#### 1.2 Prediction Creation

- **Overview:**  
  This block sends a POST request to Replicate to create a new prediction job for generating urban architecture images.

- **Nodes Involved:**  
  - Create Prediction  
  - Extract Prediction ID

- **Node Details:**

  - **Create Prediction**  
    - Type: HTTP Request  
    - Role: Submit a new prediction job to the Replicate API.  
    - Configuration:  
      - Method: POST  
      - URL: `https://api.replicate.com/v1/predictions`  
      - Headers: Authorization Bearer token using stored API key, Content-Type application/json  
      - Body: JSON specifying model version and input parameters.  
        - Inputs contain fields: `prompt` (string placeholder "prompt value"), `seed` (1), `width` (1), `height` (1), `lora_scale` (1). These should be customized before execution to affect generation.  
      - Timeout: 60 seconds  
      - Authentication: Generic HTTP Header with Bearer token.  
    - Input: API key data from "Set API Key" node.  
    - Output: Raw response JSON containing prediction ID and initial status.  
    - Edge Cases:  
      - API key invalid or missing → 401 Unauthorized error.  
      - Network issues or timeout → request failure.  
      - Invalid input parameters → API returns error.  

  - **Extract Prediction ID**  
    - Type: Code (JavaScript)  
    - Role: Parses the response from prediction creation to extract the prediction ID and initial status for polling.  
    - Configuration:  
      - Runs once per item, extracts `id` and `status` from JSON.  
      - Constructs prediction URL for subsequent status checks.  
    - Input: Response from "Create Prediction".  
    - Output: Object with keys: `predictionId`, `status`, `predictionUrl`.  
    - Edge Cases: If response format changes or missing fields, extraction fails.

#### 1.3 Prediction Polling

- **Overview:**  
  This block repeatedly checks the status of the prediction until it reaches a "succeeded" state.

- **Nodes Involved:**  
  - Wait  
  - Check Prediction Status  
  - Check If Complete

- **Node Details:**

  - **Wait**  
    - Type: Wait  
    - Role: Pauses workflow execution for 2 seconds between polling attempts to avoid rate limits.  
    - Configuration: Wait time set to 2 seconds.  
    - Input: Triggered after prediction ID extraction or if prediction is not yet complete.  
    - Output: Triggers status check.  
    - Edge Cases: Insufficient wait time may cause API rate limiting.

  - **Check Prediction Status**  
    - Type: HTTP Request  
    - Role: Sends GET request to the stored prediction URL to fetch current status.  
    - Configuration:  
      - URL dynamically set to `predictionUrl` from previous output.  
      - Headers include Authorization Bearer token.  
      - Method: GET (default).  
    - Input: From "Wait" node.  
    - Output: JSON containing current status and possibly output if ready.  
    - Edge Cases:  
      - Network errors.  
      - API rate limiting.  
      - Invalid prediction URL or expired job.  

  - **Check If Complete**  
    - Type: If  
    - Role: Branches workflow based on whether prediction status is "succeeded".  
    - Configuration: Boolean condition comparing `$json.status` to `"succeeded"`.  
    - Input: From "Check Prediction Status".  
    - Output:  
      - True branch leads to "Process Result".  
      - False branch loops back to "Wait" to continue polling.  
    - Edge Cases:  
      - Status values other than "succeeded" or "failed" are not explicitly handled; the loop continues indefinitely if prediction never completes or fails silently.

#### 1.4 Result Processing

- **Overview:**  
  Once the prediction is successful, this block extracts relevant output data and metadata for further use.

- **Nodes Involved:**  
  - Process Result

- **Node Details:**

  - **Process Result**  
    - Type: Code (JavaScript)  
    - Role: Extracts useful information from the completed prediction JSON.  
    - Configuration:  
      - Returns an object with:  
        - `status` (prediction status),  
        - `output` (array of image URLs),  
        - `metrics` (prediction performance data),  
        - `created_at` and `completed_at` timestamps,  
        - `model` name hardcoded as `'creativeathive/lemaar-door-urban'`,  
        - `other_url` duplicating the output URLs.  
    - Input: Output from "Check If Complete" true branch.  
    - Output: Simplified structured data ready for downstream consumption or display.  
    - Edge Cases: If the prediction output is empty or malformed, result processing may fail or return incomplete data.

---

### 3. Summary Table

| Node Name              | Node Type          | Functional Role                                  | Input Node(s)                 | Output Node(s)               | Sticky Note                                                                                                                        |
|------------------------|--------------------|-------------------------------------------------|------------------------------|-----------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| On clicking 'execute'   | Manual Trigger     | Initiates workflow manually                      | None                         | Set API Key                 |                                                                                                                                   |
| Set API Key            | Set                | Stores Replicate API key for authentication     | On clicking 'execute'         | Create Prediction           |                                                                                                                                   |
| Create Prediction      | HTTP Request       | Sends prediction creation request to Replicate  | Set API Key                   | Extract Prediction ID       |                                                                                                                                   |
| Extract Prediction ID  | Code               | Extracts prediction ID and status from response | Create Prediction             | Wait                       |                                                                                                                                   |
| Wait                   | Wait               | Pauses between polling requests                  | Extract Prediction ID, Check If Complete (False) | Check Prediction Status    |                                                                                                                                   |
| Check Prediction Status| HTTP Request       | Polls Replicate API for prediction status       | Wait                         | Check If Complete           |                                                                                                                                   |
| Check If Complete      | If                 | Determines if prediction is complete             | Check Prediction Status       | Process Result (True), Wait (False) |                                                                                                                                   |
| Process Result         | Code               | Processes completed prediction output            | Check If Complete (True)      | None                       |                                                                                                                                   |
| Sticky Note            | Sticky Note        | Provides setup and model information             | None                         | None                       | ## Creativeathive Lemaar Door Urban AI Generator\n\nThis workflow uses the **creativeathive/lemaar-door-urban** model from Replicate to generate other content.\n\n### Setup\n1. Add your Replicate API key\n2. Configure the input parameters\n3. Run the workflow\n\n### Model Details\n- **Type**: Other Generation\n- **Provider**: creativeathive\n- **Required Fields**: prompt |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Name: "On clicking 'execute'"  
   - No parameters needed.  

2. **Create Set Node**  
   - Type: Set  
   - Name: "Set API Key"  
   - Add a string field named `replicate_api_key`.  
   - Set its value to your Replicate API key (replace `"YOUR_REPLICATE_API_KEY"`).  
   - Connect "On clicking 'execute'" output to "Set API Key" input.  

3. **Create HTTP Request Node for Prediction Creation**  
   - Type: HTTP Request  
   - Name: "Create Prediction"  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Authentication: Generic HTTP Header authentication  
     - Header name: `Authorization`  
     - Header value: `Bearer {{ $json.replicate_api_key }}` (use expression referencing "Set API Key")  
   - Headers: Add `Content-Type: application/json`  
   - Body content type: JSON  
   - Body:  
     ```json
     {
       "version": "d7eff9d576b4f25f674adfab16f4dbdd061a0a87041daf5edf0273130c84477a",
       "input": {
         "prompt": "prompt value",
         "seed": 1,
         "width": 1,
         "height": 1,
         "lora_scale": 1
       }
     }
     ```  
   - Timeout: 60 seconds  
   - Connect output of "Set API Key" to input of "Create Prediction".  

4. **Create Code Node to Extract Prediction ID**  
   - Type: Code  
   - Name: "Extract Prediction ID"  
   - Mode: Run once per item  
   - JavaScript code:  
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
   - Connect output of "Create Prediction" to input of "Extract Prediction ID".  

5. **Create Wait Node**  
   - Type: Wait  
   - Name: "Wait"  
   - Wait time: 2 seconds  
   - Connect output of "Extract Prediction ID" to input of "Wait".  

6. **Create HTTP Request Node to Check Prediction Status**  
   - Type: HTTP Request  
   - Name: "Check Prediction Status"  
   - Method: GET (default)  
   - URL: Expression: `{{ $json.predictionUrl }}` (dynamic URL from previous node)  
   - Authentication: Generic HTTP Header authentication  
     - Header name: `Authorization`  
     - Header value: `Bearer {{ $json.replicate_api_key }}` (reference API key from "Set API Key" node)  
   - Connect output of "Wait" to input of "Check Prediction Status".  

7. **Create If Node to Check Completion**  
   - Type: If  
   - Name: "Check If Complete"  
   - Condition: Boolean  
     - Expression: `$json.status === "succeeded"`  
   - Connect output of "Check Prediction Status" to input of "Check If Complete".  

8. **Create Code Node to Process Final Result**  
   - Type: Code  
   - Name: "Process Result"  
   - Mode: Run once per item  
   - JavaScript code:  
     ```javascript
     const result = $input.item.json;
     return {
       status: result.status,
       output: result.output,
       metrics: result.metrics,
       created_at: result.created_at,
       completed_at: result.completed_at,
       model: 'creativeathive/lemaar-door-urban',
       other_url: result.output
     };
     ```  
   - Connect "Check If Complete" true output to "Process Result".  

9. **Loop Back If Not Complete**  
   - Connect "Check If Complete" false output back to "Wait" node to continue the polling loop.  

10. **Add Sticky Note (Optional)**  
    - Content:  
      ```
      ## Creativeathive Lemaar Door Urban AI Generator

      This workflow uses the **creativeathive/lemaar-door-urban** model from Replicate to generate other content.

      ### Setup
      1. Add your Replicate API key
      2. Configure the input parameters
      3. Run the workflow

      ### Model Details
      - **Type**: Other Generation
      - **Provider**: creativeathive
      - **Required Fields**: prompt
      ```  
    - Position near the start node for guidance.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                       | Context or Link                           |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------|
| This workflow interacts with the Replicate API using a specific model version identified by a SHA hash `d7eff9d576b4f25f674adfab16f4dbdd061a0a87041daf5edf0273130c84477a`. | Model version identifier in API request. |
| Prompt and other input parameters (`seed`, `width`, `height`, `lora_scale`) should be customized to control image generation results.                           | Model input parameters configuration.     |
| Polling interval is set to 2 seconds to balance responsiveness and API rate limiting.                                                                              | Wait node configuration note.             |
| The workflow requires a valid Replicate API key with sufficient quota and permissions.                                                                             | API key management.                       |
| In case of failure or non-completion, the workflow loops indefinitely; consider adding error handling or maximum retries for robustness.                          | Suggested workflow improvement.           |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated n8n workflow. It adheres strictly to content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.