Generate Images with Settyan Flash V2.0.0 AI Model via Replicate

https://n8nworkflows.xyz/workflows/generate-images-with-settyan-flash-v2-0-0-ai-model-via-replicate-7106


# Generate Images with Settyan Flash V2.0.0 AI Model via Replicate

### 1. Workflow Overview

This workflow automates image generation using the **Settyan Flash V2.0.0 Beta.7 AI model** hosted on Replicate. It is designed to receive a manual trigger, send a generation request with specified input parameters to the Replicate API, poll for the prediction status until completion, and then process and output the resulting images and metadata.  

The workflow’s logical structure consists of these blocks:  
- **1.1 Input and Initialization:** Manual trigger and API key setup for authentication.  
- **1.2 Prediction Creation:** Sending a POST request to Replicate to initiate image generation.  
- **1.3 Prediction Monitoring:** Extracting the prediction ID, waiting between polls, and checking the prediction status repeatedly.  
- **1.4 Result Processing:** Once the prediction completes successfully, processing and formatting the output data.

---

### 2. Block-by-Block Analysis

#### 1.1 Input and Initialization

**Overview:**  
Starts the workflow manually and sets the Replicate API key needed for authentication in subsequent API requests.

**Nodes Involved:**  
- On clicking 'execute'  
- Set API Key  

**Node Details:**  

- **On clicking 'execute'**  
  - Type: Manual Trigger  
  - Role: Entry point; initiates execution upon manual user action.  
  - Configuration: Default, no parameters.  
  - Inputs: None  
  - Outputs: Connects to "Set API Key" node.  
  - Edge Cases: Workflow will not start without manual trigger.  

- **Set API Key**  
  - Type: Set  
  - Role: Defines and stores the Replicate API key string in workflow data for authentication.  
  - Configuration: Assigns a string variable named `replicate_api_key` with placeholder `"YOUR_REPLICATE_API_KEY"`.  
  - Expressions: None, static assignment.  
  - Inputs: From Manual Trigger node.  
  - Outputs: To "Create Prediction" node.  
  - Edge Cases: Failing to replace `"YOUR_REPLICATE_API_KEY"` with a valid key will cause authentication errors downstream.

---

#### 1.2 Prediction Creation

**Overview:**  
Makes an authenticated HTTP POST request to Replicate’s prediction API to start the image generation task with input parameters.

**Nodes Involved:**  
- Create Prediction  

**Node Details:**  

- **Create Prediction**  
  - Type: HTTP Request  
  - Role: Sends request to Replicate API `/v1/predictions` endpoint to create a new prediction job.  
  - Configuration:  
    - URL: `https://api.replicate.com/v1/predictions`  
    - Method: POST  
    - Authentication: HTTP Header Auth using `Authorization: Bearer <replicate_api_key>`  
    - Headers: `Content-Type: application/json`  
    - Body (JSON):  
      ```json
      {
        "version": "054a842ef89d949dd0dbead5df7607cd43adedc7319aabd56fe6fdfeac8a8b5e",
        "input": {
          "prompt": "prompt value",
          "seed": 1,
          "width": 1,
          "height": 1,
          "lora_scale": 1
        }
      }
      ```  
    - Timeout: 60 seconds.  
  - Expressions:  
    - Authorization header dynamically set using API key from "Set API Key".  
  - Inputs: From "Set API Key".  
  - Outputs: To "Extract Prediction ID".  
  - Edge Cases:  
    - Invalid API key → 401 Unauthorized.  
    - API timeout or network issues.  
    - Invalid input parameters causing 4xx errors.  
  - Version Specific: Uses n8n HTTP Request node version 4.2.

---

#### 1.3 Prediction Monitoring

**Overview:**  
Extracts the prediction ID from the creation response, then periodically checks the prediction status until it is marked as succeeded.

**Nodes Involved:**  
- Extract Prediction ID  
- Wait  
- Check Prediction Status  
- Check If Complete  

**Node Details:**  

- **Extract Prediction ID**  
  - Type: Code  
  - Role: Parses the response from the creation API call to extract prediction ID and initial status, constructs the URL for status polling.  
  - Configuration: JavaScript code runs once per item, returning:  
    - `predictionId` (string)  
    - `status` (string)  
    - `predictionUrl` (string) to check status later.  
  - Inputs: From "Create Prediction".  
  - Outputs: To "Wait".  
  - Edge Cases: If response JSON structure changes or is malformed → code failure.  

- **Wait**  
  - Type: Wait  
  - Role: Delays execution for 2 seconds between polling attempts to avoid API rate limits.  
  - Configuration: 2 seconds wait time.  
  - Inputs: From "Extract Prediction ID" or from "Check If Complete" when status is incomplete.  
  - Outputs: To "Check Prediction Status".  
  - Edge Cases: None significant, but excessive retries can cause long workflow duration.  

- **Check Prediction Status**  
  - Type: HTTP Request  
  - Role: Polls the Replicate API for the current status of the prediction job.  
  - Configuration:  
    - URL: Dynamic, from expression `{{$json.predictionUrl}}` (set in previous node).  
    - Method: GET (default)  
    - Authentication: HTTP Header Auth with same API key.  
  - Inputs: From "Wait".  
  - Outputs: To "Check If Complete".  
  - Edge Cases:  
    - Network errors or API downtime.  
    - Authorization errors if token expired or invalid.  

- **Check If Complete**  
  - Type: If  
  - Role: Evaluates if the prediction status equals `"succeeded"`.  
  - Configuration: Boolean condition comparing `$json.status === "succeeded"`.  
  - Inputs: From "Check Prediction Status".  
  - Outputs:  
    - True branch → "Process Result"  
    - False branch → back to "Wait" for another polling cycle.  
  - Edge Cases: If status is `"failed"` or other unexpected status, this node does not explicitly handle failure cases, which may cause infinite loops.

---

#### 1.4 Result Processing

**Overview:**  
Upon successful completion of the prediction, processes the returned result to extract output URLs and metadata.

**Nodes Involved:**  
- Process Result  

**Node Details:**  

- **Process Result**  
  - Type: Code  
  - Role: Extracts and formats useful data from the completed prediction response.  
  - Configuration: JavaScript code runs once per item, returning an object with fields:  
    - `status` (e.g., "succeeded")  
    - `output` (image URLs or generated content)  
    - `metrics` (prediction usage data)  
    - `created_at` and `completed_at` timestamps  
    - `model` (hardcoded string "settyan/flash-v2.0.0-beta.7")  
    - `other_url` (same as output, likely image URLs)  
  - Inputs: From "Check If Complete" true branch.  
  - Outputs: None explicitly connected here; typically the workflow would end or continue with output handling.  
  - Edge Cases: If output is empty or prediction metadata missing → output may be partial or invalid.

---

### 3. Summary Table

| Node Name                | Node Type          | Functional Role                             | Input Node(s)            | Output Node(s)             | Sticky Note                                                                                         |
|--------------------------|--------------------|---------------------------------------------|--------------------------|----------------------------|---------------------------------------------------------------------------------------------------|
| On clicking 'execute'    | Manual Trigger     | Starts workflow manually                     | None                     | Set API Key                |                                                                                                   |
| Set API Key              | Set                | Stores Replicate API key for authentication | On clicking 'execute'     | Create Prediction          |                                                                                                   |
| Create Prediction        | HTTP Request       | Sends prediction creation request to Replicate API | Set API Key             | Extract Prediction ID      |                                                                                                   |
| Extract Prediction ID    | Code               | Extracts prediction ID and initial status   | Create Prediction        | Wait                       |                                                                                                   |
| Wait                     | Wait               | Pauses workflow between polls                | Extract Prediction ID, Check If Complete (false) | Check Prediction Status |                                                                                                   |
| Check Prediction Status  | HTTP Request       | Polls prediction status from Replicate API  | Wait                     | Check If Complete          |                                                                                                   |
| Check If Complete        | If                 | Checks if prediction finished successfully  | Check Prediction Status  | Process Result (true), Wait (false) |                                                                                                   |
| Process Result           | Code               | Processes and formats final prediction result | Check If Complete (true) | None                      |                                                                                                   |
| Sticky Note              | Sticky Note        | Workflow description and usage instructions | None                     | None                      | ## Settyan Flash V2.0.0 Beta.7 AI Generator\n\nThis workflow uses the **settyan/flash-v2.0.0-beta.7** model from Replicate to generate other content.\n\n### Setup\n1. Add your Replicate API key\n2. Configure the input parameters\n3. Run the workflow\n\n### Model Details\n- **Type**: Other Generation\n- **Provider**: settyan\n- **Required Fields**: prompt |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `On clicking 'execute'`  
   - Type: Manual Trigger  
   - Default settings.  

2. **Create Set Node for API Key**  
   - Name: `Set API Key`  
   - Type: Set  
   - Add a string field named `replicate_api_key` with value `YOUR_REPLICATE_API_KEY` (replace with your actual Replicate API key).  
   - Connect output of Manual Trigger to this node.  

3. **Create HTTP Request Node to Initiate Prediction**  
   - Name: `Create Prediction`  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Authentication: Generic HTTP Header Auth  
     - Add header `Authorization` with value expression: `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`  
   - Add header `Content-Type: application/json`  
   - Body (Raw JSON):  
     ```json
     {
       "version": "054a842ef89d949dd0dbead5df7607cd43adedc7319aabd56fe6fdfeac8a8b5e",
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
   - Connect output of `Set API Key` to this node.  

4. **Create Code Node to Extract Prediction ID**  
   - Name: `Extract Prediction ID`  
   - Type: Code (JavaScript)  
   - Mode: Run once per item  
   - Code:  
     ```javascript
     const prediction = $input.item.json;
     const predictionId = prediction.id;
     const initialStatus = prediction.status;

     return {
       predictionId,
       status: initialStatus,
       predictionUrl: `https://api.replicate.com/v1/predictions/${predictionId}`
     };
     ```  
   - Connect output of `Create Prediction` to this node.  

5. **Create Wait Node**  
   - Name: `Wait`  
   - Type: Wait  
   - Set to 2 seconds  
   - Connect output of `Extract Prediction ID` to this node.  

6. **Create HTTP Request Node to Check Status**  
   - Name: `Check Prediction Status`  
   - Type: HTTP Request  
   - URL: Expression `{{$json["predictionUrl"]}}`  
   - Authentication: Generic HTTP Header Auth, same header as before (`Authorization: Bearer <API_KEY>`)  
   - Method: GET (default)  
   - Connect output of `Wait` to this node.  

7. **Create If Node to Check Completion**  
   - Name: `Check If Complete`  
   - Type: If  
   - Condition: Boolean  
   - Check if `$json.status === "succeeded"`  
   - Connect output of `Check Prediction Status` to this node.  

8. **Create Code Node to Process Result**  
   - Name: `Process Result`  
   - Type: Code (JavaScript)  
   - Mode: Run once per item  
   - Code:  
     ```javascript
     const result = $input.item.json;

     return {
       status: result.status,
       output: result.output,
       metrics: result.metrics,
       created_at: result.created_at,
       completed_at: result.completed_at,
       model: 'settyan/flash-v2.0.0-beta.7',
       other_url: result.output
     };
     ```  
   - Connect the "true" output of `Check If Complete` to this node.  

9. **Loop Back for Polling**  
   - Connect the "false" output of `Check If Complete` back to the `Wait` node to continue polling until completion.  

10. **Add Sticky Note for Documentation** (optional)  
    - Content:  
      ```
      ## Settyan Flash V2.0.0 Beta.7 AI Generator

      This workflow uses the **settyan/flash-v2.0.0-beta.7** model from Replicate to generate other content.

      ### Setup
      1. Add your Replicate API key
      2. Configure the input parameters
      3. Run the workflow

      ### Model Details
      - **Type**: Other Generation
      - **Provider**: settyan
      - **Required Fields**: prompt
      ```  

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                                                                  |
|------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| The workflow requires a valid Replicate API key to authenticate requests. Ensure your API key has sufficient permissions.          | https://replicate.com/docs/apis                                                                 |
| The model version used is fixed by the version hash `"054a842ef89d949dd0dbead5df7607cd43adedc7319aabd56fe6fdfeac8a8b5e"`.          | Replicate model versioning documentation                                                        |
| Input parameters such as `prompt`, `seed`, `width`, `height`, and `lora_scale` must be set appropriately for meaningful results.  | Model-specific input schema on Replicate                                                        |
| The workflow currently does not handle prediction failure states explicitly; consider enhancing error handling for production use. | Consider adding an IF node to check for `"failed"` status and branch accordingly.                |
| Polling interval is set to 2 seconds; adjust if needed based on API rate limits and expected prediction times.                    | API rate-limiting guidelines: https://replicate.com/docs/apis#rate-limits                        |

---

**Disclaimer:** The provided text is derived exclusively from an n8n automated workflow. It adheres strictly to applicable content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.