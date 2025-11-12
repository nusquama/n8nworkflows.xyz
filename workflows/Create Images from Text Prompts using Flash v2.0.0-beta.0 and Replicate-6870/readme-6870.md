Create Images from Text Prompts using Flash v2.0.0-beta.0 and Replicate

https://n8nworkflows.xyz/workflows/create-images-from-text-prompts-using-flash-v2-0-0-beta-0-and-replicate-6870


# Create Images from Text Prompts using Flash v2.0.0-beta.0 and Replicate

### 1. Workflow Overview

This workflow automates the generation of images from text prompts using the Flash v2.0.0-beta.0 model hosted on Replicate’s API. It is designed for users who want to programmatically create AI-generated images based on descriptive text inputs, with extensive parameter customization and robust status monitoring.

The workflow logically divides into the following blocks:

- **1.1 Input Initialization**: Starts the workflow manually and sets up the API token and all model input parameters.
- **1.2 Prediction Request Submission**: Sends a generation request to Replicate API with specified parameters.
- **1.3 Status Monitoring Loop**: Periodically checks the prediction status until it succeeds or fails.
- **1.4 Result Handling**: Processes success or failure outcomes, preparing structured responses.
- **1.5 Logging and Output**: Logs prediction details and outputs the final result object.
- **1.6 Documentation and User Guidance**: Embedded sticky notes provide detailed instructions, parameter references, and support information.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Initialization

**Overview:**  
This block initiates workflow execution and sets all necessary parameters, including the Replicate API token and model-specific inputs.

**Nodes Involved:**  
- Manual Trigger  
- Set API Token  
- Set Other Parameters  

**Node Details:**

- **Manual Trigger**  
  - *Type*: Manual trigger node (initiates the workflow on user command)  
  - *Configuration*: No parameters, simply starts the workflow.  
  - *Inputs*: None  
  - *Outputs*: Connects to "Set API Token" node  
  - *Potential Failures*: None inherent; user must manually trigger.

- **Set API Token**  
  - *Type*: Set node  
  - *Role*: Assigns the Replicate API token as a string parameter named `api_token`.  
  - *Configuration*: Static value `"YOUR_REPLICATE_API_TOKEN"` placeholder to be replaced by the user.  
  - *Input*: From Manual Trigger  
  - *Output*: To "Set Other Parameters"  
  - *Failures*: User must replace placeholder with a valid token; missing or invalid token will cause API auth errors downstream.

- **Set Other Parameters**  
  - *Type*: Set node  
  - *Role*: Sets all input parameters for the image generation model, including the API token passed from previous node and all other model options.  
  - *Configuration*: Multiple parameters with default values, such as:  
    - `mask`: URL to a placeholder mask image  
    - `seed`: -1 (default for random seed)  
    - `image`: URL to input image (default is a random image from picsum.photos)  
    - `model`: "dev"  
    - `width` & `height`: 512 (image dimensions)  
    - `prompt`: "Create something amazing" (text prompt)  
    - Various other parameters controlling output format, quality, inference steps, safety checker, etc.  
  - *Inputs*: From "Set API Token"  
  - *Outputs*: To "Create Other Prediction"  
  - *Expressions*: Uses expression to carry over `api_token` from "Set API Token" node  
  - *Failures*: Parameter value types must be valid; invalid types could cause API request rejection.

---

#### 2.2 Prediction Request Submission

**Overview:**  
Submits a POST request to Replicate API to initiate image generation using the configured parameters.

**Nodes Involved:**  
- Create Other Prediction  

**Node Details:**

- **Create Other Prediction**  
  - *Type*: HTTP Request node  
  - *Role*: Sends a POST request to `https://api.replicate.com/v1/predictions` with JSON body containing the model version and all input parameters.  
  - *Configuration*:  
    - Method: POST  
    - URL: Replicate Prediction API endpoint  
    - Headers: Authorization Bearer token from the `api_token` field, and `Prefer: wait` header to request synchronous response  
    - Body: JSON constructed with dynamic expressions referencing all parameters set previously  
  - *Inputs*: From "Set Other Parameters"  
  - *Outputs*: To "Log Request"  
  - *Failures*:  
    - Authentication errors if token invalid or missing  
    - API rate limits or quota exceeded  
    - Malformed JSON or invalid parameters causing API errors  
  - *Version*: Uses HTTP Request node version 4.1

---

#### 2.3 Status Monitoring Loop

**Overview:**  
Implements a polling mechanism to check the prediction status until it completes or fails.

**Nodes Involved:**  
- Log Request  
- Wait 5s  
- Check Status  
- Is Complete?  
- Has Failed?  
- Wait 10s  

**Node Details:**

- **Log Request**  
  - *Type*: Code node (JavaScript)  
  - *Role*: Logs prediction request details (timestamp, prediction ID, model type) to console for monitoring and debugging.  
  - *Inputs*: From "Create Other Prediction"  
  - *Outputs*: To "Wait 5s"  
  - *Failures*: Console logging only; no critical failures.

- **Wait 5s**  
  - *Type*: Wait node  
  - *Role*: Pauses the workflow for 5 seconds before next status check to avoid rapid polling.  
  - *Inputs*: From "Log Request"  
  - *Outputs*: To "Check Status"  
  - *Failures*: Minimal; only timing issues if workflow halted.

- **Check Status**  
  - *Type*: HTTP Request node  
  - *Role*: Sends GET request to `https://api.replicate.com/v1/predictions/{prediction_id}` to retrieve current status and result.  
  - *Configuration*:  
    - URL dynamically constructed using `id` from "Create Other Prediction" response  
    - Authorization header with Bearer token from "Set API Token"  
  - *Inputs*: From "Wait 5s" and from "Wait 10s" (retry loop)  
  - *Outputs*: To "Is Complete?"  
  - *Failures*: API errors, auth errors, network timeouts.

- **Is Complete?**  
  - *Type*: If node  
  - *Role*: Checks if the status field equals "succeeded".  
  - *Inputs*: From "Check Status"  
  - *Outputs*:  
    - True branch to "Success Response"  
    - False branch to "Has Failed?"  
  - *Failures*: Expression errors if JSON structure unexpected.

- **Has Failed?**  
  - *Type*: If node  
  - *Role*: Checks if status equals "failed".  
  - *Inputs*: From "Is Complete?"  
  - *Outputs*:  
    - True branch to "Error Response"  
    - False branch to "Wait 10s" (continue polling)  
  - *Failures*: Same as above.

- **Wait 10s**  
  - *Type*: Wait node  
  - *Role*: Pauses 10 seconds before retrying status check, implementing backoff for prolonged generation times.  
  - *Inputs*: From "Has Failed?" false branch  
  - *Outputs*: To "Check Status"  
  - *Failures*: Minimal.

---

#### 2.4 Result Handling

**Overview:**  
Prepares structured success or error responses based on prediction outcome.

**Nodes Involved:**  
- Success Response  
- Error Response  
- Display Result  

**Node Details:**

- **Success Response**  
  - *Type*: Set node  
  - *Role*: Constructs a JSON response object with fields:  
    - `success`: true  
    - `result_url`: URL(s) of generated images from prediction output  
    - `prediction_id`, `status`, and a success message  
  - *Inputs*: From "Is Complete?" true branch  
  - *Outputs*: To "Display Result"  
  - *Failures*: None expected; relies on valid prediction output.

- **Error Response**  
  - *Type*: Set node  
  - *Role*: Constructs a JSON response object with fields:  
    - `success`: false  
    - `error`: error message from prediction or default message  
    - `prediction_id`, `status`, and failure message  
  - *Inputs*: From "Has Failed?" true branch  
  - *Outputs*: To "Display Result"  
  - *Failures*: None expected.

- **Display Result**  
  - *Type*: Set node  
  - *Role*: Sets the final result object to a single field `final_result` for downstream consumption or output.  
  - *Inputs*: From both "Success Response" and "Error Response"  
  - *Outputs*: Terminal node (workflow output)  
  - *Failures*: None.

---

#### 2.5 Logging and Output

**Overview:**  
Handles runtime logging of requests and final output packaging. This overlaps partly with 2.3 and 2.4 but is explicitly separated for clarity.

**Nodes Involved:**  
- Log Request (previously described)  
- Display Result (previously described)  

---

#### 2.6 Documentation and User Guidance

**Overview:**  
Provides embedded documentation, usage instructions, parameter explanations, and support contacts within sticky note nodes.

**Nodes Involved:**  
- Sticky Note9  
- Sticky Note4  

**Node Details:**

- **Sticky Note9**  
  - *Type*: Sticky Note  
  - *Content*: Brief header with contact email (Yaron@nofluff.online) and links to YouTube and LinkedIn channels for support and tutorials.  
  - *Purpose*: Quick support reference.

- **Sticky Note4**  
  - *Type*: Sticky Note  
  - *Content*:  
    - Detailed model overview, parameter explanations, workflow component descriptions, key benefits, quick start instructions, troubleshooting guide, and resource links (Replicate and n8n docs).  
  - *Purpose*: Comprehensive user and developer reference embedded in workflow canvas.

---

### 3. Summary Table

| Node Name             | Node Type           | Functional Role                         | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                                                          |
|-----------------------|---------------------|---------------------------------------|-------------------------|-------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| Manual Trigger        | Manual Trigger       | Starts workflow execution              | None                    | Set API Token            |                                                                                                                                      |
| Set API Token         | Set                 | Sets Replicate API token               | Manual Trigger          | Set Other Parameters     |                                                                                                                                      |
| Set Other Parameters  | Set                 | Sets all model input parameters        | Set API Token           | Create Other Prediction  |                                                                                                                                      |
| Create Other Prediction| HTTP Request         | Sends generation request to Replicate | Set Other Parameters    | Log Request              |                                                                                                                                      |
| Log Request           | Code                 | Logs request details                   | Create Other Prediction | Wait 5s                  |                                                                                                                                      |
| Wait 5s               | Wait                 | Waits 5 seconds before status check    | Log Request             | Check Status             |                                                                                                                                      |
| Check Status          | HTTP Request         | Checks prediction status               | Wait 5s, Wait 10s       | Is Complete?             |                                                                                                                                      |
| Is Complete?          | If                   | Checks if prediction succeeded         | Check Status            | Success Response, Has Failed? |                                                                                                                                  |
| Has Failed?           | If                   | Checks if prediction failed            | Is Complete?            | Error Response, Wait 10s |                                                                                                                                      |
| Wait 10s              | Wait                 | Waits 10 seconds before retrying       | Has Failed?             | Check Status             |                                                                                                                                      |
| Success Response      | Set                  | Prepares success JSON response         | Is Complete?            | Display Result           |                                                                                                                                      |
| Error Response        | Set                  | Prepares error JSON response           | Has Failed?             | Display Result           |                                                                                                                                      |
| Display Result        | Set                  | Sets final result object                | Success Response, Error Response | None              |                                                                                                                                      |
| Sticky Note9          | Sticky Note          | Support contact info and video links   | None                    | None                    | =======================================<br>For support contact: Yaron@nofluff.online<br>YouTube: https://www.youtube.com/@YaronBeen/videos<br>LinkedIn: https://www.linkedin.com/in/yaronbeen/ |
| Sticky Note4          | Sticky Note          | Detailed workflow and model documentation | None                 | None                    | Extensive multi-section content with parameter details, troubleshooting, quick start, and resource links (see full content in analysis) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger**:  
   - Add a Manual Trigger node as the workflow entry point.

2. **Add Set API Token node**:  
   - Type: Set  
   - Add string field named `api_token`  
   - Value: Set to `"YOUR_REPLICATE_API_TOKEN"` (replace with your actual token after creation)  
   - Connect Manual Trigger output to this node.

3. **Add Set Other Parameters node**:  
   - Type: Set  
   - Add all required and optional parameters as fields:  
     - `api_token`: expression referencing `$('Set API Token').item.json.api_token`  
     - `mask`: `"https://via.placeholder.com/512x512/000000/FFFFFF.png"`  
     - `seed`: -1 (number)  
     - `image`: `"https://picsum.photos/512/512"`  
     - `model`: `"dev"`  
     - `width`: 512  
     - `height`: 512  
     - `prompt`: `"Create something amazing"`  
     - `go_fast`: false (boolean)  
     - `extra_lora`: `""`  
     - `lora_scale`: 1  
     - `megapixels`: `"1"`  
     - `num_outputs`: 1  
     - `aspect_ratio`: `"1:1"`  
     - `output_format`: `"webp"`  
     - `guidance_scale`: 3  
     - `output_quality`: 80  
     - `prompt_strength`: 0.8  
     - `extra_lora_scale`: 1  
     - `replicate_weights`: `""`  
     - `num_inference_steps`: 28  
     - `disable_safety_checker`: false  
   - Connect Set API Token output to this node.

4. **Add Create Other Prediction node**:  
   - Type: HTTP Request (version 4.1)  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Headers:  
     - `Authorization`: `Bearer {{ $json.api_token }}`  
     - `Prefer`: `wait`  
   - Body: JSON with fields:  
     - `version`: `"settyan/flash-v2.0.0-beta.0:babd558e31787a18bd9add7bbcd7a8913345df389b9470f4d7ff3d84f8fb79c4"`  
     - `input`: all parameters as per "Set Other Parameters" node, using expressions like `"{{ $json.mask }}"`, `{{ $json.seed }}`, etc.  
   - Send body as JSON  
   - Connect Set Other Parameters output to this node.

5. **Add Log Request node**:  
   - Type: Code  
   - JavaScript code:  
     ```javascript
     const data = $input.all()[0].json;
     console.log('settyan/flash-v2.0.0-beta.0 Request:', {
       timestamp: new Date().toISOString(),
       prediction_id: data.id,
       model_type: 'other'
     });
     return $input.all();
     ```  
   - Connect Create Other Prediction output to this node.

6. **Add Wait 5s node**:  
   - Type: Wait  
   - Amount: 5 seconds  
   - Connect Log Request output to this node.

7. **Add Check Status node**:  
   - Type: HTTP Request (version 4.1)  
   - Method: GET  
   - URL: `https://api.replicate.com/v1/predictions/{{ $('Create Other Prediction').item.json.id }}`  
   - Headers:  
     - `Authorization`: `Bearer {{ $('Set API Token').item.json.api_token }}`  
   - Connect Wait 5s output to this node.  
   - Also connect Wait 10s output (to be created later) to this node to enable retry.

8. **Add Is Complete? node**:  
   - Type: If  
   - Condition: `$json.status` equals `"succeeded"` (string, loose typing)  
   - Connect Check Status output to this node.

9. **Add Has Failed? node**:  
   - Type: If  
   - Condition: `$json.status` equals `"failed"`  
   - Connect Is Complete? false branch to this node.

10. **Add Success Response node**:  
    - Type: Set  
    - Assign field `response` (object) with:  
      ```json
      {
        "success": true,
        "result_url": $json.output,
        "prediction_id": $json.id,
        "status": $json.status,
        "message": "Other generated successfully"
      }
      ```  
    - Connect Is Complete? true branch to this node.

11. **Add Error Response node**:  
    - Type: Set  
    - Assign field `response` (object) with:  
      ```json
      {
        "success": false,
        "error": $json.error || "Other generation failed",
        "prediction_id": $json.id,
        "status": $json.status,
        "message": "Failed to generate other"
      }
      ```  
    - Connect Has Failed? true branch to this node.

12. **Add Wait 10s node**:  
    - Type: Wait  
    - Amount: 10 seconds  
    - Connect Has Failed? false branch to this node.

13. **Connect Wait 10s output back to Check Status** to continue polling.

14. **Add Display Result node**:  
    - Type: Set  
    - Assign field `final_result` to the incoming `response` object  
    - Connect both Success Response and Error Response outputs to this node.

15. **Add Sticky Note nodes (optional for documentation)**:  
    - Add two sticky notes with content as per the original workflow for user guidance and support information.  
    - Place them visibly on the canvas.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| Contact for support: Yaron@nofluff.online | Support email provided in Sticky Note9 |
| YouTube channel with tutorials: https://www.youtube.com/@YaronBeen/videos | Linked in Sticky Note9 |
| LinkedIn profile: https://www.linkedin.com/in/yaronbeen/ | Linked in Sticky Note9 |
| Model Documentation: https://replicate.com/settyan/flash-v2.0.0-beta.0 | Referenced in Sticky Note4 |
| Replicate API Documentation: https://replicate.com/docs | Referenced in Sticky Note4 |
| n8n Documentation: https://docs.n8n.io | Referenced in Sticky Note4 |
| Troubleshooting tips: Check API token validity, parameter correctness, monitor for generation timeouts | Provided in Sticky Note4 |
| Workflow designed for robust polling with adaptive wait times (5s initial, 10s retry) | Implementation detail described in analysis |

---

**Disclaimer:** The provided text and workflow originate exclusively from an automated n8n workflow designed for legal and public data processing. It complies strictly with content policies and contains no illegal or offensive material.