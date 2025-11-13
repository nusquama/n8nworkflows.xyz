Generate Images with Adamantiamable Lumi AI via Replicate

https://n8nworkflows.xyz/workflows/generate-images-with-adamantiamable-lumi-ai-via-replicate-6944


# Generate Images with Adamantiamable Lumi AI via Replicate

### 1. Workflow Overview

This workflow, named **Adamantiamable Lumi AI Generator**, is designed to generate images or other content by interacting with the **adamantiamable/lumi** AI model hosted on the Replicate platform. It is triggered manually and performs a sequence of API calls to create a prediction, poll the prediction status until completion, and then process the resulting output.

The workflow logic is divided into the following logical blocks:

- **1.1 Manual Trigger and Initialization:** Start the workflow manually and set the required API key.
- **1.2 Create Prediction:** Send a request to Replicate API to create a new prediction using the Lumi model with specific input parameters.
- **1.3 Polling and Status Checking:** Repeatedly check the status of the prediction by polling the API until the prediction is complete.
- **1.4 Result Processing:** After successful completion, extract and format the prediction results for downstream use.


### 2. Block-by-Block Analysis

#### 2.1 Manual Trigger and Initialization

- **Overview:**  
  This block initiates the workflow when manually triggered and sets the API key required for authentication with Replicate.

- **Nodes Involved:**  
  - On clicking 'execute'  
  - Set API Key

- **Node Details:**  

  - **On clicking 'execute'**  
    - Type: Manual Trigger  
    - Role: Entry point of the workflow, waits for manual execution.  
    - Configuration: No parameters; simply triggers the workflow on user action.  
    - Inputs: None  
    - Outputs: Connected to Set API Key node  
    - Edge Cases: None typical; manual trigger requires user interaction.

  - **Set API Key**  
    - Type: Set node  
    - Role: Assigns the Replicate API key as a workflow variable for authentication.  
    - Configuration: Sets a string variable `replicate_api_key` with the placeholder `"YOUR_REPLICATE_API_KEY"`.  
    - Inputs: From manual trigger  
    - Outputs: Passes the API key forward to Create Prediction node  
    - Edge Cases: Missing or invalid API key will cause authentication failures downstream.  
    - Notes: User must replace `"YOUR_REPLICATE_API_KEY"` with their actual API key from Replicate.


#### 2.2 Create Prediction

- **Overview:**  
  This block sends a POST request to Replicate’s prediction endpoint to start the image generation process with specified parameters.

- **Nodes Involved:**  
  - Create Prediction  
  - Extract Prediction ID

- **Node Details:**  

  - **Create Prediction**  
    - Type: HTTP Request  
    - Role: Calls Replicate API to create a new prediction job.  
    - Configuration:  
      - URL: `https://api.replicate.com/v1/predictions`  
      - Method: POST  
      - Headers: `Authorization: Bearer <replicate_api_key>`, `Content-Type: application/json`  
      - Body (JSON): Contains the model version ID and input parameters such as `prompt`, `seed`, `width`, `height`, and `lora_scale`.  
      - Input parameters are currently hardcoded placeholder values; these should be dynamically set for practical use.  
      - Timeout: 60 seconds  
    - Inputs: Receives API key from Set API Key node  
    - Outputs: JSON response with prediction details, including prediction ID and initial status  
    - Edge Cases:  
      - API key invalid or missing -> 401 Unauthorized  
      - Network timeout or Replicate API downtime  
      - Malformed JSON body or invalid input parameters causing API validation errors  

  - **Extract Prediction ID**  
    - Type: Code (JavaScript)  
    - Role: Extracts the prediction ID and initial status from the create prediction API response to facilitate polling.  
    - Configuration:  
      - Reads JSON response, extracts `id` and `status` fields.  
      - Constructs a URL for polling the prediction status: `https://api.replicate.com/v1/predictions/{predictionId}`  
    - Inputs: Output from Create Prediction node  
    - Outputs: JSON containing `predictionId`, `status`, and `predictionUrl`  
    - Edge Cases:  
      - Missing or malformed response JSON causing extraction errors  
      - Unexpected API response structure changes  


#### 2.3 Polling and Status Checking

- **Overview:**  
  This block repeatedly polls the Replicate prediction endpoint to check the status of the prediction until it completes successfully.

- **Nodes Involved:**  
  - Wait  
  - Check Prediction Status  
  - Check If Complete

- **Node Details:**  

  - **Wait**  
    - Type: Wait  
    - Role: Pauses the workflow for a fixed duration before polling again to avoid rapid-fire API calls.  
    - Configuration: Waits for 2 seconds between polls.  
    - Inputs: From Extract Prediction ID or Check If Complete (on incomplete status)  
    - Outputs: Connects to Check Prediction Status  
    - Edge Cases: None significant; too short wait may cause rate limiting.

  - **Check Prediction Status**  
    - Type: HTTP Request  
    - Role: GET request to the prediction URL to retrieve the current status and output if available.  
    - Configuration:  
      - URL: Dynamic, uses expression to get `predictionUrl` from previous node.  
      - Headers: `Authorization: Bearer <replicate_api_key>`  
      - Method: GET (default)  
    - Inputs: From Wait node  
    - Outputs: JSON with updated prediction status  
    - Edge Cases:  
      - API key errors or expired tokens  
      - Network or downtime issues  
      - Rate limiting if polled too frequently  

  - **Check If Complete**  
    - Type: If  
    - Role: Conditional check on the status field to determine if prediction is complete (`status == 'succeeded'`).  
    - Configuration: Boolean condition comparing `$json.status` with `"succeeded"`.  
    - Inputs: From Check Prediction Status  
    - Outputs:  
      - True branch proceeds to Process Result node  
      - False branch loops back to Wait node for another poll  
    - Edge Cases:  
      - Other statuses (e.g., `failed`, `canceled`) not explicitly handled in this condition.  
      - Possible infinite loop if prediction never succeeds without a timeout or failure branch  


#### 2.4 Result Processing

- **Overview:**  
  Once the prediction is complete, this block processes and formats the results for further use or output.

- **Nodes Involved:**  
  - Process Result

- **Node Details:**  

  - **Process Result**  
    - Type: Code (JavaScript)  
    - Role: Extracts key fields from the completed prediction response and returns them in a structured format.  
    - Configuration:  
      - Reads fields such as `status`, `output`, `metrics`, `created_at`, `completed_at` from input JSON.  
      - Adds a fixed model identifier string `"adamantiamable/lumi"`.  
      - Duplicates output URL in `other_url` for convenience.  
    - Inputs: From Check If Complete node (true branch)  
    - Outputs: JSON with processed prediction metadata and results  
    - Edge Cases:  
      - Missing expected fields in the prediction output  
      - Unexpected output data formats  

#### 2.5 Documentation and Notes

- **Sticky Note** (not a functional node)  
  - Provides summary and usage instructions:  
    - Explains model and provider info  
    - Setup steps to add API key, configure inputs, and run  
    - Notes required fields and model type  


### 3. Summary Table

| Node Name              | Node Type        | Functional Role                      | Input Node(s)           | Output Node(s)           | Sticky Note                                                                                                              |
|------------------------|------------------|------------------------------------|------------------------|--------------------------|--------------------------------------------------------------------------------------------------------------------------|
| On clicking 'execute'   | Manual Trigger   | Manual start of the workflow       | —                      | Set API Key              |                                                                                                                          |
| Set API Key            | Set              | Assign Replicate API key            | On clicking 'execute'   | Create Prediction        |                                                                                                                          |
| Create Prediction      | HTTP Request     | Create prediction via Replicate API| Set API Key             | Extract Prediction ID    |                                                                                                                          |
| Extract Prediction ID  | Code             | Extract prediction ID and status   | Create Prediction       | Wait                     |                                                                                                                          |
| Wait                   | Wait             | Pause before polling again         | Extract Prediction ID, Check If Complete (false branch) | Check Prediction Status |                                                                                                                          |
| Check Prediction Status| HTTP Request     | Poll prediction status             | Wait                    | Check If Complete        |                                                                                                                          |
| Check If Complete      | If               | Check if prediction succeeded      | Check Prediction Status | Process Result (true), Wait (false) |                                                                                                                          |
| Process Result         | Code             | Format and return prediction output| Check If Complete (true)| —                        |                                                                                                                          |
| Sticky Note            | Sticky Note      | Documentation and instructions     | —                       | —                        | ## Adamantiamable Lumi AI Generator<br><br>This workflow uses the **adamantiamable/lumi** model from Replicate to generate other content.<br><br>### Setup<br>1. Add your Replicate API key<br>2. Configure the input parameters<br>3. Run the workflow<br><br>### Model Details<br>- **Type**: Other Generation<br>- **Provider**: adamantiamable<br>- **Required Fields**: prompt |


### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it "Adamantiamable Lumi AI Generator".

2. **Add a Manual Trigger node**  
   - Name: `On clicking 'execute'`  
   - No special configuration needed. This will be the workflow entry point.

3. **Add a Set node** to store the Replicate API key  
   - Name: `Set API Key`  
   - Add a string field named `replicate_api_key` with value `"YOUR_REPLICATE_API_KEY"` (replace this with your actual API key).  
   - Connect `On clicking 'execute'` output to this node input.

4. **Add an HTTP Request node** to create a prediction  
   - Name: `Create Prediction`  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Authentication: Generic Header Authentication  
     - Header name: `Authorization`  
     - Header value expression: `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`  
   - Additional header: `Content-Type: application/json`  
   - Body Content Type: JSON  
   - JSON Body (example template):  
     ```json
     {
       "version": "33ec1160f84c9657c8238cabc9dc8ed6bb335eb3168cdedf02df3850f8f37239",
       "input": {
         "prompt": "prompt value",
         "seed": 1,
         "width": 1,
         "height": 1,
         "lora_scale": 1
       }
     }
     ```  
   - Timeout: 60000 ms (60 seconds)  
   - Connect `Set API Key` output to this node input.

5. **Add a Code node** to extract prediction ID and initial status  
   - Name: `Extract Prediction ID`  
   - Mode: Run Once For Each Item  
   - JavaScript code:
     ```js
     const prediction = $input.item.json;
     const predictionId = prediction.id;
     const initialStatus = prediction.status;

     return {
       predictionId: predictionId,
       status: initialStatus,
       predictionUrl: `https://api.replicate.com/v1/predictions/${predictionId}`
     };
     ```
   - Connect `Create Prediction` output to this node input.

6. **Add a Wait node**  
   - Name: `Wait`  
   - Duration: 2 seconds  
   - Connect `Extract Prediction ID` output to this node input.

7. **Add an HTTP Request node** to check prediction status  
   - Name: `Check Prediction Status`  
   - Method: GET (default)  
   - URL expression: `{{$json["predictionUrl"]}}`  
   - Authentication: Generic Header Authentication  
     - Header name: `Authorization`  
     - Header value expression: `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`  
   - Connect `Wait` output to this node input.

8. **Add an If node** to check if prediction is complete  
   - Name: `Check If Complete`  
   - Condition Type: Boolean  
   - Condition: `$json.status` equals `"succeeded"`  
   - Connect `Check Prediction Status` output to this node input.

9. **Add a Code node** to process the final result  
   - Name: `Process Result`  
   - Mode: Run Once For Each Item  
   - JavaScript code:
     ```js
     const result = $input.item.json;

     return {
       status: result.status,
       output: result.output,
       metrics: result.metrics,
       created_at: result.created_at,
       completed_at: result.completed_at,
       model: 'adamantiamable/lumi',
       other_url: result.output
     };
     ```
   - Connect the **true** output of `Check If Complete` to this node input.

10. **Loop back for polling**  
    - Connect the **false** output of `Check If Complete` back to the `Wait` node to continue polling until completion.

11. **Add a Sticky Note** for documentation (optional but recommended)  
    - Content:
      ```
      ## Adamantiamable Lumi AI Generator

      This workflow uses the **adamantiamable/lumi** model from Replicate to generate other content.

      ### Setup
      1. Add your Replicate API key
      2. Configure the input parameters
      3. Run the workflow

      ### Model Details
      - Type: Other Generation
      - Provider: adamantiamable
      - Required Fields: prompt
      ```

12. **Save and activate the workflow.**

13. **To run:**  
    - Replace `"YOUR_REPLICATE_API_KEY"` with a valid Replicate API key.  
    - Modify the `prompt` and other input parameters in the `Create Prediction` HTTP Request node as needed.  
    - Execute the workflow manually by clicking the trigger node.  
    - The workflow will poll until the prediction completes, then output the result.


### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                  | Context or Link                                                                                 |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| The workflow uses the Replicate API version `v1` and the model version `33ec1160f84c9657c8238cabc9dc8ed6bb335eb3168cdedf02df3850f8f37239` of the `adamantiamable/lumi` model. Ensure this version is current and update if necessary.      | Replicate API documentation: https://replicate.com/docs/api-reference/predictions             |
| The polling interval is set to 2 seconds to balance timely updates without excessive API calls. Adjust if API rate limits or latency issues occur.                                                                           |                                                                                                 |
| This workflow does not handle prediction failures explicitly (e.g., status `failed` or `canceled`). Consider adding error handling or timeouts for robustness in production environments.                                    |                                                                                                 |
| Replace placeholder values in the `Create Prediction` node's JSON body (especially `prompt`) with dynamic input or parameters as needed for flexible use.                                                                    |                                                                                                 |
| The API key should be securely stored and managed; avoid hardcoding in production. Use n8n credentials or environment variables if possible.                                                                                  |                                                                                                 |
| Sticky note content provides useful summary instructions for users to set up and run the workflow correctly.                                                                                                                 |                                                                                                 |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.