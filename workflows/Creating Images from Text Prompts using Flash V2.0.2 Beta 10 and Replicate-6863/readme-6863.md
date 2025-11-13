Creating Images from Text Prompts using Flash V2.0.2 Beta 10 and Replicate

https://n8nworkflows.xyz/workflows/creating-images-from-text-prompts-using-flash-v2-0-2-beta-10-and-replicate-6863


# Creating Images from Text Prompts using Flash V2.0.2 Beta 10 and Replicate

### 1. Workflow Overview

This n8n workflow automates the process of creating images from text prompts using the AI model **flash-v2.0.2-beta.10** hosted on the Replicate platform. It is designed for users who want to generate AI-driven images (referred to as "Other" generation) based on descriptive prompts, optionally including parameters to influence style, resolution, and generation specifics.

**Target Use Cases:**
- AI-based image generation from textual descriptions.
- Automated batch or single image creation workflows.
- Integration of Replicate’s AI models into custom automation pipelines.
- Use in digital content creation, marketing, prototyping, or creative applications.

**Logical Blocks:**

- **1.1 Input Reception & Initialization:** Manual trigger and setting up API tokens and input parameters.
- **1.2 Prediction Request Submission:** Sending the generation request to the Replicate API.
- **1.3 Monitoring & Status Checking:** Periodic polling of the generation status until completion or failure.
- **1.4 Response Handling:** Differentiating between success or failure, preparing structured responses.
- **1.5 Logging and Final Output:** Logging request details for monitoring and outputting final results.
- **1.6 Documentation & Support:** Embedded sticky notes containing usage instructions, parameter references, and contact info.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Initialization

- **Overview:**  
This block initiates the workflow and prepares all necessary input parameters and authentication tokens required for the generation request.

- **Nodes Involved:**  
  - Manual Trigger  
  - Set API Token  
  - Set Other Parameters

- **Node Details:**

  - **Manual Trigger**  
    - Type: Trigger node to manually start the workflow.  
    - Configuration: No parameters; user manually activates the workflow.  
    - Inputs: None  
    - Outputs: Connected to "Set API Token".  
    - Edge Cases: None.  
    - Notes: Starting point for manual testing or operation.

  - **Set API Token**  
    - Type: Set node used to store the Replicate API token.  
    - Configuration: Hardcoded string field "api_token" with placeholder `"YOUR_REPLICATE_API_TOKEN"` to be replaced by the actual token.  
    - Inputs: From "Manual Trigger".  
    - Outputs: Connects to "Set Other Parameters".  
    - Edge Cases: If token is invalid or missing, API calls will fail authorization.  
    - Notes: Critical for authentication.

  - **Set Other Parameters**  
    - Type: Set node to define all input parameters for the AI model generation.  
    - Configuration: Copies API token from previous node and sets multiple parameters with default values such as:  
      - mask: URL placeholder image  
      - seed: -1 (random)  
      - image: Sample image URL  
      - model: "dev"  
      - width, height: 512 each  
      - prompt: "Create something amazing"  
      - go_fast: false  
      - extra_lora, lora_scale, megapixels, num_outputs, aspect_ratio, output_format, guidance_scale, output_quality, prompt_strength, extra_lora_scale, replicate_weights, num_inference_steps, disable_safety_checker — set to defaults or empty strings  
    - Inputs: From "Set API Token".  
    - Outputs: Connects to "Create Other Prediction".  
    - Edge Cases: Parameter mismatch or invalid values might cause API errors or unexpected results.  
    - Notes: All model parameters are centralized here for easy adjustment.

---

#### 1.2 Prediction Request Submission

- **Overview:**  
This block sends a POST request to the Replicate API to initiate image generation with the configured parameters.

- **Nodes Involved:**  
  - Create Other Prediction

- **Node Details:**

  - **Create Other Prediction**  
    - Type: HTTP Request node (POST).  
    - Configuration:  
      - URL: `https://api.replicate.com/v1/predictions`  
      - Headers: Authorization with bearer token from API token, and `Prefer: wait` to wait for result if possible.  
      - JSON Body: Includes model version, and all parameters set in previous node via expressions referencing current JSON.  
      - Response: Configured never to error even on HTTP errors, expects JSON response.  
    - Inputs: From "Set Other Parameters".  
    - Outputs: Connects to "Log Request".  
    - Edge Cases:  
      - API token invalid or expired → 401 Unauthorized  
      - Network issues or timeouts  
      - API rate limits or quota exceeded  
      - Malformed parameters causing 400 errors  
    - Notes: Initiates the asynchronous generation process.

---

#### 1.3 Monitoring & Status Checking

- **Overview:**  
This block continually polls the Replicate API to check the status of the image generation until it succeeds or fails, incorporating wait times and retry logic.

- **Nodes Involved:**  
  - Log Request  
  - Wait 5s  
  - Check Status  
  - Is Complete?  
  - Has Failed?  
  - Wait 10s

- **Node Details:**

  - **Log Request**  
    - Type: Code node (JavaScript).  
    - Configuration: Logs timestamp, prediction ID, and model type to the console for monitoring.  
    - Inputs: From "Create Other Prediction".  
    - Outputs: Connects to "Wait 5s".  
    - Edge Cases: Errors in code execution could stop flow; unlikely here as code is simple.  
    - Notes: Useful for debugging and audit trail.

  - **Wait 5s**  
    - Type: Wait node.  
    - Configuration: Waits 5 seconds before next action.  
    - Inputs: From "Log Request".  
    - Outputs: Connects to "Check Status".  
    - Edge Cases: None.  

  - **Check Status**  
    - Type: HTTP Request node (GET).  
    - Configuration:  
      - URL dynamically constructed from prediction ID returned by "Create Other Prediction".  
      - Authorization header with API token.  
      - Response JSON, never error.  
    - Inputs: From "Wait 5s" and "Wait 10s".  
    - Outputs: Connects to "Is Complete?" node.  
    - Edge Cases:  
      - Network issues or API downtime  
      - Prediction ID missing or invalid  
      - API rate limits

  - **Is Complete?**  
    - Type: If node.  
    - Configuration: Checks if `status` field equals "succeeded".  
    - Inputs: From "Check Status".  
    - Outputs:  
      - If true: connects to "Success Response".  
      - If false: connects to "Has Failed?".  
    - Edge Cases: Status field missing or unexpected values.

  - **Has Failed?**  
    - Type: If node.  
    - Configuration: Checks if `status` equals "failed".  
    - Inputs: From "Is Complete?" (false branch).  
    - Outputs:  
      - If true: connects to "Error Response".  
      - If false: connects to "Wait 10s" (retry).  
    - Edge Cases: Other statuses (e.g., "processing") keep retrying.

  - **Wait 10s**  
    - Type: Wait node.  
    - Configuration: Waits 10 seconds before retrying status check.  
    - Inputs: From "Has Failed?" (false branch).  
    - Outputs: Connects back to "Check Status".  
    - Edge Cases: None.

---

#### 1.4 Response Handling

- **Overview:**  
Based on the success or failure of the image generation, this block prepares structured JSON responses for downstream consumption or user feedback.

- **Nodes Involved:**  
  - Success Response  
  - Error Response  
  - Display Result

- **Node Details:**

  - **Success Response**  
    - Type: Set node.  
    - Configuration: Sets a JSON object property `response` containing success status, output URL from prediction, prediction ID, status, and a success message.  
    - Inputs: From "Is Complete?" (success branch).  
    - Outputs: Connects to "Display Result".  
    - Edge Cases: Output URL missing or malformed.

  - **Error Response**  
    - Type: Set node.  
    - Configuration: Sets a JSON object property `response` containing failure status, error message (fallback to 'Other generation failed'), prediction ID, status, and error message.  
    - Inputs: From "Has Failed?" (failure branch).  
    - Outputs: Connects to "Display Result".  
    - Edge Cases: Error message absent or unclear.

  - **Display Result**  
    - Type: Set node.  
    - Configuration: Copies the `response` object from prior nodes into `final_result` for final output or webhook response.  
    - Inputs: From both "Success Response" and "Error Response".  
    - Outputs: Terminal node.  
    - Edge Cases: None.

---

#### 1.5 Logging and Final Output

- **Overview:**  
This block provides logging of generation requests for operational monitoring and ensures the final structured output is available.

- **Nodes Involved:**  
  - Log Request (already included in 1.3)  
  - Display Result (already included in 1.4)

- **Node Details:**  
See nodes above.

---

#### 1.6 Documentation & Support

- **Overview:**  
Sticky Notes nodes provide embedded instructions, parameter details, troubleshooting tips, and contact information for ease of use and maintenance.

- **Nodes Involved:**  
  - Sticky Note9  
  - Sticky Note4

- **Node Details:**

  - **Sticky Note9**  
    - Content: Title block with contact email (Yaron@nofluff.online) and links to YouTube and LinkedIn for support and tutorials.  
    - Position: Top-left corner as a header.  
    - Usage: Reference for users needing help.

  - **Sticky Note4**  
    - Content:  
      - Detailed description of model, parameters, workflow components, quick start guide, troubleshooting, and resource links.  
      - Extensive notes on parameters, expected behaviors, and best practices.  
    - Position: Left side spanning vertically.  
    - Usage: Comprehensive embedded documentation.

---

### 3. Summary Table

| Node Name              | Node Type          | Functional Role                        | Input Node(s)             | Output Node(s)             | Sticky Note                                                                                                            |
|------------------------|--------------------|-------------------------------------|---------------------------|----------------------------|------------------------------------------------------------------------------------------------------------------------|
| Manual Trigger         | manualTrigger      | Start workflow                      | —                         | Set API Token              |                                                                                                                        |
| Set API Token          | set                | Store Replicate API token           | Manual Trigger            | Set Other Parameters       |                                                                                                                        |
| Set Other Parameters   | set                | Define AI model input parameters    | Set API Token             | Create Other Prediction    |                                                                                                                        |
| Create Other Prediction| httpRequest        | Send generation request to Replicate API | Set Other Parameters    | Log Request                |                                                                                                                        |
| Log Request            | code               | Log request details for monitoring  | Create Other Prediction   | Wait 5s                    |                                                                                                                        |
| Wait 5s                | wait               | Pause before status check            | Log Request               | Check Status               |                                                                                                                        |
| Check Status           | httpRequest        | Poll Replicate API for prediction status | Wait 5s, Wait 10s        | Is Complete?               |                                                                                                                        |
| Is Complete?           | if                 | Check if prediction succeeded       | Check Status              | Success Response, Has Failed? |                                                                                                                     |
| Has Failed?            | if                 | Check if prediction failed          | Is Complete?              | Error Response, Wait 10s   |                                                                                                                        |
| Wait 10s               | wait               | Pause before retrying status check  | Has Failed?               | Check Status               |                                                                                                                        |
| Success Response       | set                | Prepare success response JSON       | Is Complete?              | Display Result             |                                                                                                                        |
| Error Response         | set                | Prepare error response JSON         | Has Failed?               | Display Result             |                                                                                                                        |
| Display Result         | set                | Output final structured response    | Success Response, Error Response | —                    |                                                                                                                        |
| Sticky Note9           | stickyNote         | Contact info and support links      | —                         | —                          | =======================================\nFLASH-V2.0.2-BETA.10 GENERATOR\nFor support contact: Yaron@nofluff.online\nYouTube: https://www.youtube.com/@YaronBeen/videos\nLinkedIn: https://www.linkedin.com/in/yaronbeen/ |
| Sticky Note4           | stickyNote         | Full documentation and parameter guide | —                      | —                          | Contains detailed model overview, parameter reference, workflow explanation, quick start instructions, and troubleshooting guide |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Manual Trigger node:**  
   - Name: Manual Trigger  
   - No parameters needed.

3. **Add a Set node for API token:**  
   - Name: Set API Token  
   - Add a string field named `api_token` with value `"YOUR_REPLICATE_API_TOKEN"` (replace with your actual token).  
   - Connect Manual Trigger → Set API Token.

4. **Add a Set node for model parameters:**  
   - Name: Set Other Parameters  
   - Assign fields with the following key-value pairs:  
     - `api_token`: `={{ $('Set API Token').item.json.api_token }}` (expression to copy token)  
     - `mask`: `"https://via.placeholder.com/512x512/000000/FFFFFF.png"`  
     - `seed`: `-1`  
     - `image`: `"https://picsum.photos/512/512"`  
     - `model`: `"dev"`  
     - `width`: `512`  
     - `height`: `512`  
     - `prompt`: `"Create something amazing"`  
     - `go_fast`: `false`  
     - `extra_lora`: `""`  
     - `lora_scale`: `1`  
     - `megapixels`: `"1"`  
     - `num_outputs`: `1`  
     - `aspect_ratio`: `"1:1"`  
     - `output_format`: `"webp"`  
     - `guidance_scale`: `3`  
     - `output_quality`: `80`  
     - `prompt_strength`: `0.8`  
     - `extra_lora_scale`: `1`  
     - `replicate_weights`: `""`  
     - `num_inference_steps`: `28`  
     - `disable_safety_checker`: `false`  
   - Connect Set API Token → Set Other Parameters.

5. **Add HTTP Request node to create prediction:**  
   - Name: Create Other Prediction  
   - HTTP Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Headers:  
     - `Authorization`: `=Bearer {{ $json.api_token }}`  
     - `Prefer`: `wait`  
   - Body Content Type: JSON  
   - Body: Use JSON with expressions referencing all parameters from Set Other Parameters, matching the structure:  
     ```json
     {
       "version": "settyan/flash-v2.0.2-beta.10:<version_id>",
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
   - Connect Set Other Parameters → Create Other Prediction.

6. **Add a Code node to log requests:**  
   - Name: Log Request  
   - JavaScript code:  
     ```js
     const data = $input.all()[0].json;
     console.log('settyan/flash-v2.0.2-beta.10 Request:', {
       timestamp: new Date().toISOString(),
       prediction_id: data.id,
       model_type: 'other'
     });
     return $input.all();
     ```  
   - Connect Create Other Prediction → Log Request.

7. **Add a Wait node (5 seconds):**  
   - Name: Wait 5s  
   - Time unit: seconds  
   - Amount: 5  
   - Connect Log Request → Wait 5s.

8. **Add HTTP Request node to check status:**  
   - Name: Check Status  
   - HTTP Method: GET  
   - URL: `=https://api.replicate.com/v1/predictions/{{ $('Create Other Prediction').item.json.id }}`  
   - Headers:  
     - `Authorization`: `=Bearer {{ $('Set API Token').item.json.api_token }}`  
   - Configure response to never error and to parse JSON.  
   - Connect Wait 5s → Check Status.  
   - Connect Wait 10s → Check Status (to be created later).

9. **Add If node to check if generation is complete:**  
   - Name: Is Complete?  
   - Condition: Check if `status` equals `"succeeded"` (case sensitive).  
   - Connect Check Status → Is Complete?.

10. **Add If node to check if generation failed:**  
    - Name: Has Failed?  
    - Condition: Check if `status` equals `"failed"`.  
    - Connect Is Complete? (false branch) → Has Failed?.

11. **Add Wait node (10 seconds) for retry delay:**  
    - Name: Wait 10s  
    - Time unit: seconds  
    - Amount: 10  
    - Connect Has Failed? (false branch) → Wait 10s.

12. **Connect Wait 10s → Check Status** (loop back for retry).

13. **Add Set node for success response:**  
    - Name: Success Response  
    - Set JSON property `response` to:  
      ```js
      {
        success: true,
        result_url: $json.output,
        prediction_id: $json.id,
        status: $json.status,
        message: 'Other generated successfully'
      }
      ```  
    - Connect Is Complete? (true branch) → Success Response.

14. **Add Set node for error response:**  
    - Name: Error Response  
    - Set JSON property `response` to:  
      ```js
      {
        success: false,
        error: $json.error || 'Other generation failed',
        prediction_id: $json.id,
        status: $json.status,
        message: 'Failed to generate other'
      }
      ```  
    - Connect Has Failed? (true branch) → Error Response.

15. **Add Set node for final display:**  
    - Name: Display Result  
    - Set property `final_result` equal to the `response` from previous nodes.  
    - Connect Success Response → Display Result.  
    - Connect Error Response → Display Result.

16. **(Optional) Add Sticky Note nodes** with detailed instructions, parameter references, and contact info as per the documentation.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| For any questions or support, please contact: Yaron@nofluff.online | Contact email from Sticky Note9 |
| Explore more tips and tutorials here: YouTube: https://www.youtube.com/@YaronBeen/videos, LinkedIn: https://www.linkedin.com/in/yaronbeen/ | Support and tutorial links from Sticky Note9 |
| Model Documentation and API reference: https://replicate.com/settyan/flash-v2.0.2-beta.10 | Official model page |
| Replicate API Documentation: https://replicate.com/docs | Documentation for API usage |
| n8n Documentation: https://docs.n8n.io | Official n8n docs for node usage and workflow design |
| Replace `"YOUR_REPLICATE_API_TOKEN"` with a valid Replicate API token for authentication | Critical for API calls |
| Recommended to test with default parameters before customization | Best practice |
| Monitor API usage and billing on replicate.com to avoid quota issues | Operational advice |
| Keep API tokens secure and never expose them publicly | Security advice |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected material. All manipulated data is legal and public.