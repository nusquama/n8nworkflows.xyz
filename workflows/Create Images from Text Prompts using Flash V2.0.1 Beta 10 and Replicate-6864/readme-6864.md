Create Images from Text Prompts using Flash V2.0.1 Beta 10 and Replicate

https://n8nworkflows.xyz/workflows/create-images-from-text-prompts-using-flash-v2-0-1-beta-10-and-replicate-6864


# Create Images from Text Prompts using Flash V2.0.1 Beta 10 and Replicate

### 1. Workflow Overview

This n8n workflow automates the generation of images based on text prompts using the AI model "flash-v2.0.1-beta.10" hosted on Replicate. It is designed for creative or automated image generation use cases where users input textual prompts and receive AI-generated images in response.

The workflow is logically divided into the following blocks:

- **1.1 Input Initialization:** Receives manual trigger and sets essential API credentials and generation parameters.
- **1.2 Prediction Request:** Sends a generation request to the Replicate API using the configured parameters.
- **1.3 Status Polling & Control Loop:** Waits and repeatedly checks the prediction status until completion or failure.
- **1.4 Success and Error Handling:** Processes results on success or failure and prepares structured responses.
- **1.5 Logging and Output:** Logs the request details for monitoring and outputs the final result.
- **1.6 Documentation and User Guidance:** Sticky notes provide detailed instructions, parameter references, and support information.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Initialization

**Overview:**  
This block begins the workflow execution via manual trigger, sets the Replicate API token, and configures all parameters needed for the image generation model.

**Nodes Involved:**  
- Manual Trigger  
- Set API Token  
- Set Other Parameters  

**Node Details:**

- **Manual Trigger**  
  - Type: Trigger node to start workflow manually.  
  - Configuration: Default, no parameters.  
  - Inputs: None  
  - Outputs: Connects to Set API Token  
  - Edge Cases: None, but user must manually start workflow.

- **Set API Token**  
  - Type: Set node  
  - Role: Stores the Replicate API token as a string variable named `api_token`.  
  - Configuration: User must replace `"YOUR_REPLICATE_API_TOKEN"` with a valid token.  
  - Inputs: From Manual Trigger  
  - Outputs: To Set Other Parameters  
  - Edge Cases: Missing or invalid token leads to authentication failure downstream.

- **Set Other Parameters**  
  - Type: Set node  
  - Role: Defines all parameters required to generate the image, including prompt, image size, model options, and advanced flags.  
  - Configuration Highlights:  
    - `prompt`: Text prompt for image generation, default "Create something amazing"  
    - `image` and `mask`: URLs for input image and mask for inpainting  
    - `seed`, `width`, `height`, `model`, `num_outputs`, `aspect_ratio`, and others for fine control  
    - Boolean flags like `go_fast` and `disable_safety_checker`  
  - Uses expression to get API token from previous node (`Set API Token`).  
  - Inputs: From Set API Token  
  - Outputs: To Create Other Prediction  
  - Edge Cases: Invalid parameter values (e.g., negative sizes, malformed URLs) may cause API errors.

---

#### 2.2 Prediction Request

**Overview:**  
Sends a POST request to the Replicate API to initiate an image generation prediction with the configured parameters and waits for immediate response.

**Nodes Involved:**  
- Create Other Prediction  

**Node Details:**

- **Create Other Prediction**  
  - Type: HTTP Request  
  - Role: Submits the image generation request to the Replicate API endpoint: `https://api.replicate.com/v1/predictions`  
  - Configuration:  
    - HTTP Method: POST  
    - Auth Header: `Authorization: Bearer {api_token}`  
    - Request Body: JSON including model version and all input parameters from "Set Other Parameters" node.  
    - Header `Prefer: wait` to wait for initial processing.  
    - Response expected in JSON format.  
  - Inputs: From Set Other Parameters  
  - Outputs: To Log Request  
  - Edge Cases: Network errors, API rate limiting, invalid tokens, or malformed request body can cause failures.

---

#### 2.3 Status Polling & Control Loop

**Overview:**  
After initiating the prediction, this block implements a polling loop to check the prediction status until it is either succeeded or failed, incorporating waiting periods to avoid excessive requests.

**Nodes Involved:**  
- Log Request  
- Wait 5s  
- Check Status  
- Is Complete?  
- Has Failed?  
- Wait 10s  

**Node Details:**

- **Log Request**  
  - Type: Code node  
  - Role: Logs prediction initiation details including timestamp and prediction ID for monitoring.  
  - Code: Logs to console relevant JSON data from the prediction response.  
  - Inputs: From Create Other Prediction  
  - Outputs: To Wait 5s  
  - Edge Cases: Logging failure does not stop workflow.

- **Wait 5s**  
  - Type: Wait node  
  - Role: Pauses execution for 5 seconds before checking status.  
  - Inputs: From Log Request  
  - Outputs: To Check Status  
  - Edge Cases: None

- **Check Status**  
  - Type: HTTP Request  
  - Role: Fetches current status of the prediction from the Replicate API using prediction ID.  
  - Configuration:  
    - GET request to `https://api.replicate.com/v1/predictions/{prediction_id}`  
    - Auth header with API token.  
  - Inputs: From Wait 5s and Wait 10s (loop)  
  - Outputs: To Is Complete?  
  - Edge Cases: Network failures, expired prediction ID, or auth errors.

- **Is Complete?**  
  - Type: If node  
  - Role: Checks if the prediction status equals "succeeded" to determine completion.  
  - Inputs: From Check Status  
  - Outputs:  
    - True branch: Success Response  
    - False branch: Has Failed?  
  - Edge Cases: Status other than succeeded or failed requires further polling.

- **Has Failed?**  
  - Type: If node  
  - Role: Checks if prediction status equals "failed" to handle errors.  
  - Inputs: From Is Complete?  
  - Outputs:  
    - True branch: Error Response  
    - False branch: Wait 10s (retry loop)  
  - Edge Cases: Unknown statuses, indefinite pending.

- **Wait 10s**  
  - Type: Wait node  
  - Role: Pauses 10 seconds before retrying status check.  
  - Inputs: From Has Failed? false branch  
  - Outputs: To Check Status  
  - Edge Cases: None

---

#### 2.4 Success and Error Handling

**Overview:**  
Processes the final outcome of the prediction. On success, it packages the output URL and metadata. On failure, it generates an error response with details.

**Nodes Involved:**  
- Success Response  
- Error Response  
- Display Result  

**Node Details:**

- **Success Response**  
  - Type: Set node  
  - Role: Constructs a JSON object indicating success, including the generated image URL, prediction ID, status, and a success message.  
  - Inputs: From Is Complete? true branch  
  - Outputs: To Display Result

- **Error Response**  
  - Type: Set node  
  - Role: Constructs a JSON object indicating failure, including error details, prediction ID, status, and failure message.  
  - Inputs: From Has Failed? true branch  
  - Outputs: To Display Result

- **Display Result**  
  - Type: Set node  
  - Role: Assigns the final response object (success or error) to a single variable `final_result` for consistent output.  
  - Inputs: From Success Response and Error Response  
  - Outputs: None (end node)  
  - Edge Cases: None

---

#### 2.5 Logging and Output

**Overview:**  
Handles logging of request details for audit and debugging purposes.

**Nodes Involved:**  
- Log Request (also part of polling loop)

**Node Details:**  
- Explored previously in 2.3.  
- Logs timestamp, prediction ID, and model type to console.

---

#### 2.6 Documentation and User Guidance

**Overview:**  
Sticky notes provide detailed instructions, parameter descriptions, and contact/support information embedded within the workflow canvas.

**Nodes Involved:**  
- Sticky Note9  
- Sticky Note4  

**Node Details:**

- **Sticky Note9**  
  - Content: Branding and contact info for support, including YouTube and LinkedIn links.  
  - Role: Provides developer contact and resources.

- **Sticky Note4**  
  - Content: Extensive documentation covering model overview, parameters, workflow explanation, quick start instructions, troubleshooting, and links to external documentation.  
  - Role: Acts as embedded manual and best practice guide for users and developers.

---

### 3. Summary Table

| Node Name           | Node Type           | Functional Role                              | Input Node(s)           | Output Node(s)                  | Sticky Note                                                                                                                  |
|---------------------|---------------------|----------------------------------------------|-------------------------|--------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| Manual Trigger      | Manual Trigger      | Starts workflow execution                     | None                    | Set API Token                  |                                                                                                                              |
| Set API Token       | Set                 | Sets Replicate API token                      | Manual Trigger          | Set Other Parameters           |                                                                                                                              |
| Set Other Parameters| Set                 | Sets all generation parameters                | Set API Token           | Create Other Prediction        |                                                                                                                              |
| Create Other Prediction| HTTP Request      | Sends image generation request to Replicate API | Set Other Parameters    | Log Request                   |                                                                                                                              |
| Log Request         | Code                | Logs prediction data for monitoring           | Create Other Prediction | Wait 5s                      |                                                                                                                              |
| Wait 5s             | Wait                | Waits 5 seconds before status check           | Log Request             | Check Status                  |                                                                                                                              |
| Check Status        | HTTP Request        | Checks prediction status                       | Wait 5s, Wait 10s       | Is Complete?                  |                                                                                                                              |
| Is Complete?        | If                  | Checks if prediction succeeded                 | Check Status            | Success Response, Has Failed? |                                                                                                                              |
| Has Failed?         | If                  | Checks if prediction failed                     | Is Complete?            | Error Response, Wait 10s      |                                                                                                                              |
| Wait 10s            | Wait                | Waits 10 seconds before retrying status check | Has Failed?             | Check Status                  |                                                                                                                              |
| Success Response    | Set                 | Creates success response object                 | Is Complete?            | Display Result                |                                                                                                                              |
| Error Response      | Set                 | Creates error response object                   | Has Failed?             | Display Result                |                                                                                                                              |
| Display Result      | Set                 | Assigns final response for output               | Success Response, Error Response | None                         |                                                                                                                              |
| Sticky Note9        | Sticky Note         | Branding and support contact info               | None                    | None                         | =======================================\nFLASH-V2.0.1-BETA.10 GENERATOR\nFor any questions or support, please contact:\nYaron@nofluff.online\nYouTube: https://www.youtube.com/@YaronBeen/videos\nLinkedIn: https://www.linkedin.com/in/yaronbeen/\n======================================= |
| Sticky Note4        | Sticky Note         | Comprehensive workflow and parameter documentation | None                    | None                         | ## 🤖 **SETTYAN/FLASH-V2.0.1-BETA.10 - OTHER GENERATION WORKFLOW**\nPowered by Replicate API and n8n Automation\n\nIncludes model overview, parameters, instructions, and troubleshooting.\nLinks: https://replicate.com/settyan/flash-v2.0.1-beta.10, https://replicate.com/docs, https://docs.n8n.io |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - No configuration needed.

2. **Add Set API Token Node**  
   - Type: Set  
   - Add a string variable `api_token` with value: `"YOUR_REPLICATE_API_TOKEN"` (replace with your actual token).  
   - Connect output of Manual Trigger to this node.

3. **Add Set Other Parameters Node**  
   - Type: Set  
   - Copy the `api_token` value from "Set API Token" node using expression: `={{ $('Set API Token').item.json.api_token }}`  
   - Add string and numeric variables for model parameters as follows (with default values):  
     - `mask` (string): `"https://via.placeholder.com/512x512/000000/FFFFFF.png"`  
     - `seed` (number): `-1`  
     - `image` (string): `"https://picsum.photos/512/512"`  
     - `model` (string): `"dev"`  
     - `width` (number): `512`  
     - `height` (number): `512`  
     - `prompt` (string): `"Create something amazing"`  
     - `go_fast` (boolean): `false`  
     - `extra_lora` (string): `""`  
     - `lora_scale` (number): `1`  
     - `megapixels` (string): `"1"`  
     - `num_outputs` (number): `1`  
     - `aspect_ratio` (string): `"1:1"`  
     - `output_format` (string): `"webp"`  
     - `guidance_scale` (number): `3`  
     - `output_quality` (number): `80`  
     - `prompt_strength` (number): `0.8`  
     - `extra_lora_scale` (number): `1`  
     - `replicate_weights` (string): `""`  
     - `num_inference_steps` (number): `28`  
     - `disable_safety_checker` (boolean): `false`  
   - Connect output of Set API Token to this node.

4. **Add Create Other Prediction Node**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Headers:  
     - Authorization: `Bearer {{ $json.api_token }}`  
     - Prefer: `wait`  
   - Body (raw JSON):  
     ```json
     {
       "version": "settyan/flash-v2.0.1-beta.10:92078fec46fc705fceb16d17f8654b3b9aeea8b0b8ccaad08740b080e202aa10",
       "input": {
         "mask": "{{ $json.mask }}",
         "seed": {{ $json.seed }},
         "image": "{{ $json.image }}",
         "model": "{{ $json.model }}",
         "width": {{ $json.width }},
         "height": {{ $json.height }},
         "prompt": "{{ $json.prompt }}",
         "go_fast": {{ $json.go_fast }},
         "extra_lora": "{{ $json.extra_lora }}",
         "lora_scale": {{ $json.lora_scale }},
         "megapixels": "{{ $json.megapixels }}",
         "num_outputs": {{ $json.num_outputs }},
         "aspect_ratio": "{{ $json.aspect_ratio }}",
         "output_format": "{{ $json.output_format }}",
         "guidance_scale": {{ $json.guidance_scale }},
         "output_quality": {{ $json.output_quality }},
         "prompt_strength": {{ $json.prompt_strength }},
         "extra_lora_scale": {{ $json.extra_lora_scale }},
         "replicate_weights": "{{ $json.replicate_weights }}",
         "num_inference_steps": {{ $json.num_inference_steps }},
         "disable_safety_checker": {{ $json.disable_safety_checker }}
       }
     }
     ```
   - Connect output of Set Other Parameters node to this node.

5. **Add Log Request Node (Code Node)**  
   - Type: Code  
   - JavaScript code:  
     ```javascript
     const data = $input.all()[0].json;
     console.log('settyan/flash-v2.0.1-beta.10 Request:', {
       timestamp: new Date().toISOString(),
       prediction_id: data.id,
       model_type: 'other'
     });
     return $input.all();
     ```  
   - Connect output of Create Other Prediction node to this node.

6. **Add Wait 5s Node**  
   - Type: Wait  
   - Unit: Seconds  
   - Amount: 5  
   - Connect output of Log Request to this node.

7. **Add Check Status Node (HTTP Request)**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.replicate.com/v1/predictions/{{ $('Create Other Prediction').item.json.id }}`  
   - Headers:  
     - Authorization: `Bearer {{ $('Set API Token').item.json.api_token }}`  
   - Connect output of Wait 5s to this node.

8. **Add Is Complete? Node (If Node)**  
   - Type: If  
   - Condition: Check if `status` field equals `"succeeded"` (case sensitive = false)  
   - Connect output of Check Status to this node.

9. **Add Success Response Node (Set)**  
   - Type: Set  
   - Set one field:  
     - Name: `response`  
     - Type: Object  
     - Value (expression):  
       ```json
       {
         "success": true,
         "result_url": {{$json.output}},
         "prediction_id": {{$json.id}},
         "status": {{$json.status}},
         "message": "Other generated successfully"
       }
       ```  
   - Connect true output of Is Complete? to this node.

10. **Add Has Failed? Node (If Node)**  
    - Type: If  
    - Condition: Check if `status` field equals `"failed"`  
    - Connect false output of Is Complete? to this node.

11. **Add Error Response Node (Set)**  
    - Type: Set  
    - Set one field:  
      - Name: `response`  
      - Type: Object  
      - Value (expression):  
        ```json
        {
          "success": false,
          "error": {{$json.error || "Other generation failed"}},
          "prediction_id": {{$json.id}},
          "status": {{$json.status}},
          "message": "Failed to generate other"
        }
        ```  
    - Connect true output of Has Failed? to this node.

12. **Add Wait 10s Node**  
    - Type: Wait  
    - Unit: Seconds  
    - Amount: 10  
    - Connect false output of Has Failed? to this node.

13. **Complete the Polling Loop**  
    - Connect output of Wait 10s back into Check Status node to continue polling.

14. **Add Display Result Node (Set)**  
    - Type: Set  
    - Set one field:  
      - Name: `final_result`  
      - Type: Object  
      - Value: `={{ $json.response }}`  
    - Connect outputs of Success Response and Error Response nodes to this node.

15. **Add Sticky Note Nodes (optional for documentation)**  
    - Add two Sticky Note nodes with the content as described in section 2.6 for user guidance and branding.

---

### 5. General Notes & Resources

| Note Content                                                                                                                              | Context or Link                                                                                             |
|-------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| For any questions or support, please contact: Yaron@nofluff.online                                                                        | Sticky Note9 in workflow                                                                                     |
| Explore more tips and tutorials here: YouTube: https://www.youtube.com/@YaronBeen/videos LinkedIn: https://www.linkedin.com/in/yaronbeen/ | Sticky Note9                                                                                                 |
| Model Documentation and API details available at: https://replicate.com/settyan/flash-v2.0.1-beta.10                                      | Sticky Note4 and official Replicate docs                                                                     |
| Detailed parameter reference and troubleshooting guide embedded in workflow sticky notes                                                  | Sticky Note4                                                                                                 |
| Replace the placeholder API token with a valid Replicate API token to avoid authentication errors                                         | Workflow usage note                                                                                           |
| Recommended to monitor API usage and billing on Replicate platform                                                                         | Troubleshooting advice                                                                                        |
| Use valid URLs for `image` and `mask` parameters to prevent request errors                                                                | Parameter best practice                                                                                       |
| The workflow uses "wait" headers and polling with delay nodes to avoid excessive API calls and respect API limits                         | Workflow design note                                                                                          |

---

**Disclaimer:**  
The provided text is extracted exclusively from an automated workflow created with n8n, a tool for integration and automation. This workflow complies strictly with content policies and contains no illegal, offensive, or protected material. All data processed is legal and publicly accessible.